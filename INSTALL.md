# Plan & Dispatch — Installations-Handout

*[English version](INSTALL.en.md)*

**Die Änderungen je Version stehen in `CHANGELOG.md`.**

Aufgaben-Dispatching-Board für Sage-100-Beratungsteams: Kanban-Board pro
Mitarbeiter, KI-gestützte Klassifizierung und Aufwandsschätzung,
kapazitätsbewusstes Auto-Dispatching, Auslastungssicht, Projektplan,
Soll/Ist-Vergleich mit Zeiterfassung, Unteraufgaben, interner
Ticket-Nummernkreis, ERP-/Ticketsystem-Anbindung (Push-API + HubSpot-Konnektor).

Dieses Dokument beschreibt die Installation als Docker-Container im
Firmennetz — von null bis zum ersten Login in ca. 10 Minuten.

---

## 1. Voraussetzungen

| Was | Details |
|---|---|
| Server | Linux, Windows (WSL2) oder macOS; 1 CPU-Kern, 1 GB RAM, 2 GB Disk genügen |
| Docker | Docker Engine **24+** mit Compose-Plugin (`docker compose version` muss funktionieren) |
| Netzwerk | Zwei freie Ports, Standard: **5050** (HTTP) und **5443** (HTTPS) |
| Optional | Ein KI-Endpunkt: LM Studio / Ollama / OpenAI / Azure / Anthropic — die App läuft auch komplett ohne KI (Keyword-Fallback) |

Alle Daten (SQLite-Datenbank, Session-Schlüssel, Backups) liegen im
Docker-Volume `dispatcher-data` — kein externer Datenbankserver nötig.

> **Wichtig:** Das Daten-Volume nicht auf NFS-/SMB-Freigaben legen
> (SQLite-WAL-Locking funktioniert dort nicht zuverlässig).

---

## 2. Installation

### Variante A — fertiges Image (empfohlen, Server hat Internet)

Die Anwendung wird als fertiges Docker-Image ausgeliefert; sie wird **nicht**
selbst aus dem Quellcode gebaut. Benötigt werden nur vier Dateien aus dem
Release-Paket: `docker-compose.yml`, `docker-compose.prod.yml`, `Caddyfile` und
`.env.example`.

```bash
mkdir plan-and-dispatch-ai && cd plan-and-dispatch-ai
# die vier Dateien aus dem Release-Paket hierher kopieren

# Konfiguration anlegen und anpassen — ADMIN_PASSWORD ist PFLICHT,
# ohne den Eintrag verweigert docker compose den Start
cp .env.example .env
nano .env

# Image ziehen und starten
docker compose up -d
```

Das Image bringt beide gängigen Architekturen mit (x86-64 und ARM64), läuft
also auch auf Apple Silicon, Raspberry Pi oder ARM-Servern.

### Variante B — Offline mit Image-Datei (Server ohne Internet)

Auf einem Rechner **mit** Internet das Image herunterladen und als Datei sichern:

```bash
docker pull plananddispatchai/plan-and-dispatch-ai:latest
docker save plananddispatchai/plan-and-dispatch-ai:latest | gzip > plananddispatch.tar.gz
```

Diese Datei zusammen mit den vier Konfigurationsdateien auf den Zielserver
kopieren, dann dort:

```bash
docker load -i plananddispatch.tar.gz
cp .env.example .env
nano .env
docker compose up -d --no-build
```

> Das Caddy-Image (`caddy:2-alpine`) muss offline ebenfalls per
> `docker save`/`docker load` mitgebracht werden — oder man lässt den
> Caddy-Service weg (siehe Abschnitt 5) und nutzt nur HTTP.

### Danach laufen drei Container

| Service | Aufgabe |
|---|---|
| `dispatcher` | Die App (gunicorn, Port 5050) |
| `backup` | Nächtlicher SQLite-Dump um 02:00 nach `/data/backups/`, 14 Tage Aufbewahrung |
| `caddy` | HTTPS-Reverse-Proxy auf Port 5443 (selbstsigniertes Zertifikat) |

Kurz prüfen:

```bash
docker compose ps                          # alle drei "running", dispatcher "healthy"
curl http://localhost:5050/healthz         # → {"status":"ok","version":"…"}
```

---

## 3. Erster Login & Grundeinrichtung

1. Browser: `http://<server>:5050` (oder `https://<server>:5443`).
2. Anmelden mit den Werten aus der `.env` (`ADMIN_USER` / `ADMIN_PASSWORD`,
   Default `admin` / `admin`). Beim Default erzwingt die App sofort einen
   Passwortwechsel — auch serverseitig, das lässt sich nicht überspringen.
3. **Admin-Bereich** (☰-Menü → Admin-Bereich) durchgehen, sinnvolle Reihenfolge:
   1. **Mitarbeiter** — Spalten des Boards: Name, E-Mail, Farbe, Skills (Freitext
      für die KI), Skill-Tags (kommagetrennt, fürs Auto-Dispatching),
      Wochenstunden + Puffer % (für Kapazitätsprüfung und Auslastungssicht),
      CRM/ERP-Benutzername (für die automatische Zuordnung beim Ticket-Import).
   2. **Benutzer** — Logins für das Team anlegen (Admin-Haken nur wo nötig).
      Neue Benutzer müssen ihr Startpasswort beim ersten Login ändern.
      Optional je Benutzer: verknüpfter Mitarbeiter („Meine Aufgaben" in der
      iOS-App) und Sichtbarkeit (Teams/Standorte).
   3. **Teams & Standorte** — optional: Team-Filter im Board, Teamfarben/-logos,
      Erscheinungsbild (Firmenfarbe + Logo für Header und Excel-Exporte),
      Budgetprüfungs-Schwelle der Zeiterfassung.
   4. **KI-Einstellungen** — siehe Abschnitt 4.
   5. **Meilenstein-Vorlage** — das Phasenmodell, das jedes neue Projekt
      automatisch bekommt (Namen, Beschreibungen, KI-Stichwörter) — Defaults
      passen für den Start.
   6. **Einstellungen** — interner Ticket-Nummernkreis für den Betrieb **ohne**
      externes Ticketsystem: eigene fortlaufende Ticketnummern mit Startnummer,
      optionalem Präfix (z.B. `SV-`) und Nullauffüllung.
   7. **Schnittstellen** — nur bei Anbindung nötig; drei Bereiche in einem Tab:
      **1·API** (Schnittstellen-Schalter + API-Schlüssel für den Ticket-Push aus
      ERP/Sage — Doku: `API_INGEST.md`, VM-Skripte: Ordner `sage-vm/`),
      **2·HubSpot** (Pull-Konnektor mit Private-App-Token),
      **3·CSV-Vorlage** für den manuellen Aufgabenimport (mit 5 Demo-Zeilen).

---

## 4. KI anbinden (optional, aber der eigentliche Mehrwert)

Admin-Bereich → **KI-Einstellungen**. Die Konfiguration liegt in der
Datenbank und gilt global — keine Container-Neustarts nötig.

| Einstellung | Beispiele |
|---|---|
| Provider `openai` (OpenAI-kompatibel) | LM Studio, Ollama, vLLM, OpenAI, Azure OpenAI |
| Provider `anthropic` | Claude-API (Modellname ist dann Pflicht) |
| Provider `disabled` | Keine KI — App nutzt Keyword-Fallback |

Typische Setups:

- **LM Studio auf dem Docker-Host** (lokal, keine Cloud):
  Base-URL `http://host.docker.internal:1234/v1`, Modell leer lassen
  (Auto-Detect). In LM Studio muss „Serve on Local Network" aktiviert sein.
- **OpenAI:** Base-URL `https://api.openai.com/v1` + API-Key + Modell.
- **Anthropic:** Base-URL leer lassen + API-Key + Modell (z. B. `claude-sonnet-5`).

Mit **„Verbindung testen"** prüfen. Die KI übernimmt dann: Klassifizierung
neuer Aufgaben (Mitarbeiter-Zuordnung mit Kapazitätsprüfung),
Aufwandsschätzung, Meilenstein-Zuordnung, Vorschläge für nächste Schritte
und das Wochenbriefing.

---

## 5. HTTP / HTTPS

- Standard: App parallel über **HTTP :5050** und **HTTPS :5443** erreichbar.
- Das HTTPS-Zertifikat ist selbstsigniert (Caddy-interne CA) — Browser
  warnen einmalig. Für vertrauenswürdige Zertifikate: Hostnamen im
  `Caddyfile` eintragen und die Caddy-Root-CA im Firmennetz verteilen
  (liegt im Volume `caddy-data` unter `pki/authorities/local/root.crt`).
- Wer nur HTTP im LAN möchte, kann den `caddy`-Service aus der
  `docker-compose.yml` entfernen — die App selbst braucht ihn nicht.

---

## 6. Backup & Wiederherstellung

**Automatisch:** Der `backup`-Service schreibt jede Nacht (Standard 02:00)
einen konsistenten SQLite-Dump nach `/data/backups/dispatcher-JJJJ-MM-TT.db`
im Volume und behält 14 Tages-Stände (einstellbar in der `.env`).

### Der Normalweg: Admin-Bereich → Tab „Datenbank"

Sichern, herunterladen und **zurücksichern** läuft ohne Server-Zugang direkt in
der App (Abschnitt 1 „Sicherungen"):

- **Sicherung jetzt erstellen** — legt sofort einen Stand an (`dispatcher-manuell-…`).
- **⬇ Herunterladen** — holt eine Sicherung aus dem Volume auf den eigenen Rechner.
- **Zurücksichern** — aus der Liste oder aus einer hochgeladenen `.db`-Datei.
  Vorher wird der aktuelle Stand automatisch als `dispatcher-vor-restore-…`
  gesichert, der Schritt ist also umkehrbar. Danach wird der Admin abgemeldet.

Zurückgespielt wird über die **SQLite-Backup-API** direkt in die laufende
Datenbank — deshalb braucht es weder einen Container-Neustart noch das
Hantieren mit `-wal`/`-shm`-Dateien (siehe die Warnung weiter unten).

**Aufbewahrung:** `BACKUP_KEEP_DAYS` (Standard 14) gilt für die nächtlichen
Tagesstände. Für die benannten Sicherungen (*manuell*, *vor Rücksicherung*)
gilt stattdessen `BACKUP_KEEP_MANUAL` (Standard 10) — je Art die neuesten N,
bewusst anzahl- statt altersbasiert, damit eine Sicherheitskopie nicht nach
14 Tagen verschwindet.

⚠️ Heruntergeladene Sicherungen enthalten den **kompletten** Datenbestand
inklusive KI-/HubSpot-Tokens im Klartext — entsprechend behandeln (§9).

### Rückfall: Kommandozeile

**Backup aus dem Volume herausholen** (z. B. zur Sicherung auf ein anderes System):

```bash
docker compose cp dispatcher:/data/backups ./backups-kopie
```

**Wiederherstellen:**

```bash
docker compose stop dispatcher
docker compose cp ./backups-kopie/dispatcher-2026-07-06.db dispatcher:/data/dispatcher.db
docker compose start dispatcher
```

**Sofort-Backup von Hand:**

```bash
docker compose exec backup python -c "import backup; backup.make_backup('manuell')"
```

### Datenbank einsehen & Datensätze bereinigen (nur Admin)

**Einsehen — direkt in der App:** Admin-Bereich → Tab **„Datenbank"**, Abschnitt 3
**„Rohdaten (nur Lesen)"** zeigt alle Tabellen mit Zeilenzahl und lässt jede
Tabelle paginiert durchsuchen. Geheimnisse (Passwort-Hashes, API-/HubSpot-Tokens)
sind maskiert. Es gibt bewusst **kein** freies SQL und keine Schreibfunktion in
der Oberfläche — Sicherung und Rücksicherung stehen in Abschnitt 1 desselben Tabs.

**Konfiguration übertragen:** Abschnitt 2 **„Konfiguration"** exportiert Benutzer
(Rechte, Sichtbarkeiten, Sprache, Board-Layout) und die globalen App-Einstellungen
als lesbares JSON — **ohne** Passwörter, Tokens, API-Schlüssel, Firmenlogo und
Ticketnummern-Zähler. Gedacht für eine zweite Installation: Teams und Standorte
werden dabei über ihre **Namen** zugeordnet, nicht über interne IDs. Beim Import
wird kein Benutzer gelöscht; fehlende werden mit Zufallspasswort und erzwungenem
Wechsel angelegt.

**Bereinigen/Korrigieren — in dieser Reihenfolge:**

1. **Normalfall:** über die Fach-Tabs (Benutzer, Mitarbeiter, Teams & Standorte,
   im Board: Kunden/Aufgabentypen/Beteiligte/Projekte, Aufgaben löschen als Admin).
2. **Rohzugriff** nur, wenn ein Fall über die Masken nicht lösbar ist —
   **vorher ein Backup sichern** (siehe oben). Zugriff über das App-Image
   (gleiche `db.get_conn()`, WAL-sicher), analog zum Passwort-Reset unten:

   ```bash
   docker compose exec dispatcher python3 -c "
   import db
   with db.get_conn() as c:
       # Beispiel: verwaiste Kommentare eines gelöschten Tickets entfernen
       c.execute(\"DELETE FROM comments WHERE task_id=?\", ('<task-id>',))
       c.commit()
   print('bereinigt')"
   ```

   ⚠️ Keine Roh-Edits an der laufenden DB von **außerhalb** des Containers
   (kein `docker cp` der `.db` zurück, solange die App läuft) — WAL-Locking.
   Schreibzugriffe immer über eine Verbindung **im** Container wie oben.

---

## 7. Update auf eine neue Version

```bash
cd plan-and-dispatch-ai
docker compose pull             # neue Fassung des Images holen
docker compose up -d            # Container mit dem neuen Image neu starten
```

Variante B (offline): das neue Image auf dem Rechner mit Internet als Datei
sichern, kopieren, dann `docker load -i …` und `docker compose up -d --no-build`.

Wer eine bestimmte Version festhalten will, setzt sie in der `.env`:
`IMAGE=plananddispatchai/plan-and-dispatch-ai:2.116`. Ohne Angabe gilt `latest`.

Die Datenbank wird beim Start automatisch migriert (nur additive
Schema-Änderungen); Daten, Session-Schlüssel und Backups bleiben im Volume
erhalten. Ein Rollback ist über das nächtliche Backup möglich.

**Empfehlung: vor dem Update eine Sicherung anlegen** — Admin → Datenbank →
„Sicherung jetzt erstellen". Das dauert einen Klick und macht den Schritt
umkehrbar. Ein Rückschritt auf ein **älteres** Image, nachdem die Datenbank
bereits migriert wurde, ist nicht vorgesehen; in diesem Fall die vor dem Update
erstellte Sicherung zurückspielen.

**Einmalig beim Update auf 2.113 oder neuer:** Die Anwendung läuft im Container
seit dieser Version nicht mehr als root, sondern als eingeschränkter Benutzer
(UID 10001). Beim ersten Start übergibt der Container die Besitzrechte am
Datenverzeichnis selbst — im Log erscheint dann einmal
`entrypoint: Besitzrechte an /data werden einmalig auf app übergeben ...`.
**Es ist nichts zu tun.** Nur wer statt des Docker-Volumes einen Ordner vom Host
einbindet, muss ihm einmal den passenden Besitzer geben:
`chown -R 10001:10001 <ordner>`.

---

## 8. Fehlerbehebung

| Problem | Lösung |
|---|---|
| Port 5050/5443 belegt | In der `.env` `HTTP_PORT`/`HTTPS_PORT` ändern, `docker compose up -d` |
| `docker compose ps` zeigt `unhealthy` | `docker compose logs dispatcher --tail 50` — häufigste Ursache: beschädigtes Volume oder voller Datenträger |
| KI-Test schlägt fehl (LM Studio) | LM Studio: „Serve on Local Network" aktivieren; vom Host testen: `curl http://localhost:1234/v1/models` |
| Login gesperrt („Zu viele Fehlversuche") | 5 Minuten warten — oder `docker compose restart dispatcher` (setzt die Sperren zurück) |
| Admin-Passwort vergessen | Siehe Kasten unten |
| Browser warnt vor Zertifikat | Erwartet bei `tls internal` — Ausnahme bestätigen oder Caddy-Root-CA verteilen (Abschnitt 5) |
| Alle Sessions nach Neustart weg | Nur wenn das Volume gelöscht wurde — `SECRET_KEY` in der `.env` setzen macht den Schlüssel unabhängig vom Volume |

**Admin-Passwort zurücksetzen** (direkt in der Datenbank, Benutzername ggf. anpassen):

```bash
docker compose exec dispatcher python3 -c "
from werkzeug.security import generate_password_hash
import db
with db.get_conn() as c:
    c.execute(\"UPDATE users SET password_hash=?, must_change_password=1 WHERE username='admin'\",
              (generate_password_hash('Neustart123'),))
    c.commit()
print('Passwort: Neustart123 (Wechsel wird beim Login erzwungen)')"
```

---

## 9. Sicherheit — Kurzüberblick

- Session-Cookie-Login (HttpOnly, SameSite=Lax), Passwörter gehasht
  (werkzeug/scrypt mit Salt)
- Login-Lockout: 5 Fehlversuche → 5 Minuten Sperre
- Erzwungener Passwortwechsel bei Start- und zurückgesetzten Passwörtern (serverseitig)
- Ingest-API nur mit API-Key (SHA-256-gehasht gespeichert, Klartext nur einmal sichtbar)
- Board-/Admin-Oberflächen werden nur an angemeldete Benutzer ausgeliefert
- Excel-Exporte mit Formula-Injection-Schutz (Zellwerte mit führendem `=`
  werden nie als Formel gespeichert)
- Abhängigkeiten in `requirements.txt` versionsgepinnt (reproduzierbare Builds)
- Die Anwendung läuft im Container als unprivilegierter Benutzer (UID 10001),
  nicht als root; der Programmcode im Container ist für sie nur lesbar
- Empfehlung fürs Firmennetz: HTTPS (Port 5443) verwenden; HTTP nur, wenn
  das LAN als vertrauenswürdig gilt. Die nächtlichen Backups im Volume
  enthalten die komplette Datenbank (inkl. KI-/HubSpot-Token) — Kopien
  entsprechend schützen.
- **`SESSION_COOKIE_SECURE=1`** in der `.env` setzen, sobald die App
  ausschließlich über https erreichbar ist (Reverse-Proxy, Cloudflare Tunnel).
  Das Session-Cookie wird dann nie über eine unverschlüsselte Verbindung
  gesendet. Standard ist `0`, weil ein Secure-Cookie beim Zugriff über
  `http://<host>:5050` gar nicht erst ankäme — **niemand könnte sich anmelden**.
  Wer parallel http und https anbietet, lässt den Schalter auf `0`.

---

*Fragen und Fehlerberichte: siehe `CONTRIBUTING.md` im Release-Paket.
Sicherheitslücken bitte nicht öffentlich melden, sondern an info@dispatcher-ai.de.*
