# Rechnungstool Cloud

Rechnungen schreiben mit Dashboard, Schritt-für-Schritt-Assistent und eigener Datenbank – von jedem Gerät erreichbar. Laufende Kosten: **0 €/Monat**.

## Architektur

```
Browser (jedes Gerät)
   │  lädt statische Datei
   ▼
index.html  ──────────────  gehostet auf eigenem Webspace,
   │                        Cloudflare Pages oder GitHub Pages (kostenlos)
   │  HTTPS + anon key
   ▼
Supabase (Free Tier)
   ├─ Auth        E-Mail + Passwort, Session-Verwaltung
   └─ PostgreSQL  Tabellen profiles / customers / invoices
                  Row Level Security: jeder sieht nur seine Daten
```

Die App ist bewusst eine einzige HTML-Datei ohne Build-Schritt. Ohne eingetragene Supabase-Zugangsdaten läuft sie im lokalen Modus (Speicherung nur im Browser) – so kannst du sie sofort testen, bevor die Cloud steht.

## Kosten

| Baustein | Anbieter | Kosten |
|---|---|---|
| Datenbank + Auth | Supabase Free | 0 € |
| Hosting | eigener Webspace / Cloudflare Pages / GitHub Pages | 0 € |
| Bibliothek | supabase-js via jsDelivr-CDN (SRI-gepinnt) | 0 € |

Grenzen des Supabase Free Tiers, die du kennen solltest:

- **500 MB Datenbank** – für Rechnungsdaten praktisch unerschöpflich (eine Rechnung ≈ 1–2 KB).
- **Pause nach ~7 Tagen Inaktivität**: Das Projekt schläft ein und wird im Dashboard mit einem Klick wieder geweckt. Bei regelmäßiger Nutzung passiert das nicht.
- **Keine automatischen Backups** im Free Tier → nutze regelmäßig den JSON-Export der App („Daten") und archiviere Rechnungs-PDFs ohnehin 8 Jahre (GoBD).
- Max. 2 kostenlose Projekte pro Organisation; Auth-Mails sind ratenlimitiert (für Solo-Nutzung irrelevant).

## Einrichtung (einmalig, ~15 Minuten)

### Schritt 1 – Supabase-Projekt anlegen

1. Auf [supabase.com](https://supabase.com) registrieren (GitHub-Login geht am schnellsten).
2. **New project** → Name z. B. `rechnungstool`, Region **eu-central-1 (Frankfurt)** (Daten bleiben in der EU), Datenbank-Passwort generieren lassen und im Passwortmanager ablegen (du brauchst es selten, aber verliere es nicht).
3. Warten, bis das Projekt provisioniert ist (~2 Minuten).

### Schritt 2 – Datenbankschema einspielen

1. Links im Dashboard **SQL Editor** öffnen.
2. Den kompletten Inhalt von `schema.sql` einfügen → **Run**.
3. Erwartete Ausgabe: „Success. No rows returned." Damit existieren die drei Tabellen inklusive Row Level Security.

### Schritt 3 – Auth konfigurieren

1. **Authentication → Sign In / Providers**: E-Mail ist standardmäßig aktiv – so lassen.
2. Optional für Solo-Nutzung: **Confirm email** deaktivieren, dann entfällt der Bestätigungslink bei der Registrierung.
3. **Authentication → URL Configuration**: als *Site URL* die Adresse eintragen, unter der die App später läuft (z. B. `https://rechnungen.cloudwerk.tech`). Wichtig, falls Bestätigungs-/Reset-Mails im Spiel sind.

### Schritt 4 – Zugangsdaten in die App eintragen

1. **Project Settings → API** öffnen.
2. `Project URL` und den **anon public** Key kopieren.
3. In `index.html` ganz oben im `<script>`-Block eintragen:

```js
var SUPABASE_URL = "https://xxxxxxxx.supabase.co";
var SUPABASE_ANON_KEY = "eyJhbGciOi...";
```

Der anon key darf öffentlich im Quelltext stehen – er ist dafür gemacht. Die Sicherheit kommt aus der Row Level Security in der Datenbank: Ohne gültigen Login gibt der Key keinerlei Daten frei, und mit Login ausschließlich die eigenen Zeilen.

### Schritt 5 – Deployen (eine Option wählen)

**Option A – eigener Webspace (z. B. Subdomain auf cloudwerk.tech):** `index.html` per SFTP/Panel hochladen. Fertig. HTTPS vorausgesetzt.

**Option B – Cloudflare Pages:** [pages.cloudflare.com](https://pages.cloudflare.com) → *Upload assets* → Ordner mit `index.html` hochladen → kostenlose `*.pages.dev`-URL, eigene Domain optional anbindbar.

**Option C – GitHub Pages:** Repo anlegen, `index.html` pushen, unter *Settings → Pages* den Branch veröffentlichen.

In allen Fällen: die veröffentlichte URL als Site URL in Schritt 3 nachtragen, falls noch nicht geschehen.

### Schritt 6 – Erster Start

1. App öffnen → **Neu registrieren** mit deiner E-Mail.
2. **Danach empfohlen:** Im Supabase-Dashboard unter **Authentication → Sign In / Providers** die Option **Allow new users to sign up** deaktivieren. Ab jetzt kann niemand außer dir ein Konto anlegen – dein privates Tool bleibt privat, auch wenn die URL bekannt wird.
3. Erstes Profil anlegen, Testrechnung durch den Assistenten schicken, auf einem zweiten Gerät anmelden und dieselben Daten sehen. Das war's.

## Wie die App speichert

- Jede Änderung (Profil speichern, Rechnung abschließen, „Bezahlt" markieren, Löschen) schreibt sofort gezielt in die Datenbank; oben rechts erscheint kurz „Gespeichert ✓" bzw. eine Fehlermeldung bei Verbindungsproblemen.
- Der JSON-Import ersetzt den kompletten Datenbestand (lokal und in der Cloud) – gedacht für Umzug oder Restore aus einem Backup.
- Rechnungsnummern zählen pro Profil und Jahr fortlaufend; manuell vergebene Nummern werden übernommen und weitergezählt.

## Troubleshooting

| Symptom | Ursache / Lösung |
|---|---|
| „Cloud nicht erreichbar – vorübergehend lokaler Modus" | Projekt pausiert (Free Tier) → im Supabase-Dashboard *Restore/Resume* klicken. Oder Tippfehler in URL/Key. |
| Login schlägt fehl mit „Email not confirmed" | Bestätigungslink in der Mail öffnen oder *Confirm email* deaktivieren (Schritt 3). |
| Registrierung geht nicht mehr | Gewollt, falls du *Allow new users to sign up* deaktiviert hast (Schritt 6). |
| „Invalid API key" | anon key erneut aus *Project Settings → API* kopieren (kompletten Key, keine Zeilenumbrüche). |
| Daten fehlen auf zweitem Gerät | Mit demselben Konto angemeldet? Jedes Konto sieht nur seine eigenen Zeilen (RLS). |

## Später aufrüsten

Wenn das Tool wächst: Supabase Pro (25 $/Monat) bringt tägliche Backups, kein Pausieren und mehr Ressourcen. Alternativ lässt sich das Schema 1:1 auf selbst gehostetes Supabase oder PocketBase auf einem kleinen VPS (~5 €/Monat) umziehen – die App spricht nur mit einer klar gekapselten Datenschicht (`dbWrite`/`dbDelete`/`cloudLoadAll` in `index.html`).