# Bestandskunden-Import (SAP → Shopware) & Erst-Login mit PLZ+Kundennummer

## Ausgangslage / Problem
- SAP-Debitoren werden nach Shopware importiert.
- In Shopware sind Pflichtfelder:
  - Kundennummer (Pflicht)
  - Vorname (Pflicht)
  - Nachname (Pflicht)
  - E-Mail (Pflicht)
  - Passwort (Pflicht)
  - Kundengruppe (Pflicht)
- In SAP fehlen bei Bestandskunden häufig E-Mail-Adressen.
- Bestandskunden sollen sich trotzdem anmelden können, **ohne** dass beim Claim/Onboarding ein Passwort neu gesetzt werden muss.

Ziel: Import darf **nie** scheitern (Fallbacks), Bestandskunden können sich **einmalig** mit `PLZ+Kundennummer` anmelden und werden danach gezwungen/angeleitet, Passwort (und optional E-Mail) zu ändern.

---

## 1) Import-Regeln (Fallbacks für Pflichtfelder)

### 1.1 Kundennummer (Shopware `customerNumber`)
- **Shopware Kundennummer = SAP Debitorennummer (KUNNR)**
- Führende Nullen: **konsequent beibehalten** (oder konsequent normalisieren – aber nicht mischen).
- Muss eindeutig sein.

### 1.2 E-Mail (Pflicht) – Placeholder
Wenn keine echte E-Mail vorhanden ist, wird eine Placeholder-Mail generiert:

- **Format:** `<KUNNR>@noemail.com`

Beispiele:
- `0000000990@noemail.com`
- `2460001234@noemail.com`

Hinweis:
- Placeholder dient nur zur Pflichtvalidierung.
- Kunde kann später eine echte E-Mail setzen.

### 1.3 Vorname / Nachname (Pflicht) – Fallback
Wenn keine zuverlässig splitbare Person aus SAP-Namen ableitbar ist:

- `firstName = "Kunde"`
- `lastName  = "<KUNNR>"`

Beispiel:
- `firstName = "Kunde"`
- `lastName  = "2460001234"`

### 1.4 Passwort (Pflicht) – Initialpasswort für Bestandskunden
Für importierte Bestandskunden wird ein initiales Passwort gesetzt:

- **Initialpasswort:** `PLZ + KUNNR`
- Beispiel: PLZ `76189`, KUNNR `2460001234` → `761892460001234`

Wichtig:
- Dieses Passwort ist **nur für den ersten Login** gedacht.
- Nach dem ersten Login muss der Kunde es selbst ändern (siehe Abschnitt 3).

### 1.5 Kundengruppe (Pflicht)
- Mapping gemäß Business-Regel (z.B. immer B2B Standardgruppe).
- Muss beim Import immer gesetzt werden.

---

## 2) Erst-Login ohne E-Mail (Bestandskunde)

### 2.1 UX / Einstieg
Im Login/Registrieren wird eine Option angeboten:

**„Ich bin bereits Kunde, habe aber noch nicht online bestellt.“**

Eingaben:
- **Kundennummer** (Pflicht)
- **PLZ** (Pflicht)
- **E-Mail** (optional: kann nach Login gesetzt werden)

### 2.2 Prozess-Logik (Backend)

#### Schritt 1: Kunde finden
- Lookup über `customerNumber == <KUNNR>` (oder Custom Field, falls genutzt)

Wenn nicht gefunden:
- generische Fehlermeldung (keine Information leaken)

#### Schritt 2: PLZ verifizieren
- Vergleiche eingegebene PLZ mit gespeicherter PLZ des Kunden (z.B. Default Billing Address):
  - `inputPLZ == customer.billingAddress.zipcode`

Wenn PLZ mismatch:
- generische Fehlermeldung

#### Schritt 3: Login durchführen
- Das System verwendet das initiale Passwort:
  - `password = inputPLZ + inputKUNNR`
- Authentifizierung erfolgt danach wie üblich (Session/Token).

Hinweis:
- Alternativ kann der Kunde auch ganz normal über die Login-Maske mit E-Mail + Passwort arbeiten,
  sobald er eine echte E-Mail gesetzt hat.

---

## 3) Pflicht nach erstem Login: Passwort ändern (und optional E-Mail setzen)

### 3.1 Flagging / Status für “Initial Login”
Beim Import wird ein Flag am Customer gespeichert, z.B.:
- `customFields.customer_isInitialPassword = true`

Nach erfolgreichem Login:
- Wenn Flag `true`:
  - Nutzer wird direkt auf „Passwort ändern“ geleitet oder bekommt einen erzwungenen Dialog/Blocker.
  - Optional zusätzlich: „E-Mail-Adresse hinterlegen“ (empfohlen, wenn Placeholder vorhanden).

Nach erfolgreicher Passwortänderung:
- Flag wird auf `false` gesetzt.

### 3.2 Optional: E-Mail nachziehen
Wenn die E-Mail noch Placeholder ist (`<KUNNR>@noemail.com`):
- UI zeigt Hinweis: „Bitte E-Mail-Adresse hinterlegen, damit Sie Ihr Passwort zurücksetzen können.“
- Änderung erfolgt im Kundenkonto-Profil.

---

## 4) Sicherheits- und Abuse-Hinweise (pragmatisch)
- Fehlertexte immer generisch:
  - „Daten stimmen nicht überein.“
- Rate-Limit / Lockout:
  - Limit pro Kundennummer/IP (z.B. X Versuche pro 10 Minuten)
- Logging:
  - fehlgeschlagene Versuche (KUNNR, IP, timestamp)

---

## 5) Zusammenfassung (Kurz)
- Import:
  - `email = <KUNNR>@noemail.com` (Fallback)
  - `firstName = "Kunde"`, `lastName = "<KUNNR>"` (Fallback)
  - initiales Passwort: `PLZ+KUNNR`
  - Flag `isInitialPassword = true`
- Erst-Login:
  - Kunde gibt `KUNNR + PLZ` ein
  - System verifiziert PLZ und loggt mit `PLZ+KUNNR` ein
- Danach:
  - Kunde muss Passwort ändern (und idealerweise E-Mail setzen)
