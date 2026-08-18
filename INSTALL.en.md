# Plan & Dispatch — Installation Guide

*[Deutsche Fassung](INSTALL.md)*

**Changes per version are listed in `CHANGELOG.en.md`.**

A task dispatching board for consulting teams: a Kanban column per team member,
AI-assisted classification and effort estimation, capacity-aware auto dispatching,
workload view, project plan, planned/actual comparison with time tracking,
subtasks, an internal ticket number range and integrations with ERP and ticketing
systems (push API + HubSpot connector).

This document covers installing the Docker container in your own network — from
nothing to the first login in about 10 minutes.

---

## 1. Requirements

| What | Details |
|---|---|
| Server | Linux, Windows (WSL2) or macOS; 1 CPU core, 1 GB RAM and 2 GB disk are enough |
| Docker | Docker Engine **24+** with the Compose plugin (`docker compose version` must work) |
| Network | Two free ports, by default **5050** (HTTP) and **5443** (HTTPS) |
| Optional | An AI endpoint: LM Studio / Ollama / OpenAI / Azure / Anthropic — the app also runs entirely without AI (keyword fallback) |

All data (SQLite database, session key, backups) lives in the Docker volume
`dispatcher-data` — no external database server is required.

> **Important:** do not place the data volume on an NFS or SMB share
> (SQLite's WAL locking is unreliable there).

---

## 2. Installation

### Option A — prebuilt image (recommended, server has internet)

The application ships as a prebuilt Docker image; it is **not** built from source.
You only need four files from the release package: `docker-compose.yml`,
`docker-compose.prod.yml`, `Caddyfile` and `.env.example`.

```bash
mkdir plan-and-dispatch-ai && cd plan-and-dispatch-ai
# copy the four files from the release package into this directory

# Create and edit the configuration — ADMIN_PASSWORD is REQUIRED,
# without it docker compose refuses to start
cp .env.example .env
nano .env

# Pull the image and start
docker compose up -d
```

The image ships both common architectures (x86-64 and ARM64), so it runs on
Apple Silicon, a Raspberry Pi or ARM servers as well.

### Option B — offline with an image file (server without internet)

On a machine **with** internet, download the image and save it to a file:

```bash
docker pull metromcd/plan-and-dispatch-ai:latest
docker save metromcd/plan-and-dispatch-ai:latest | gzip > plananddispatch.tar.gz
```

Copy that file together with the four configuration files to the target server,
then run there:

```bash
docker load -i plananddispatch.tar.gz
cp .env.example .env
nano .env
docker compose up -d --no-build
```

> The Caddy image (`caddy:2-alpine`) has to be brought along offline via
> `docker save`/`docker load` as well — or leave the Caddy service out (see
> section 5) and use HTTP only.

### Three containers will then be running

| Service | Purpose |
|---|---|
| `dispatcher` | The app (gunicorn, port 5050) |
| `backup` | Nightly SQLite dump at 02:00 into `/data/backups/`, kept for 14 days |
| `caddy` | HTTPS reverse proxy on port 5443 (self-signed certificate) |

Quick check:

```bash
docker compose ps                          # all three "running", dispatcher "healthy"
curl http://localhost:5050/healthz         # → {"status":"ok","version":"…"}
```

---

## 3. First login and basic setup

1. Browser: `http://<server>:5050` (or `https://<server>:5443`).
2. Sign in with the values from your `.env` (`ADMIN_USER` / `ADMIN_PASSWORD`).
   If you left the default `admin` / `admin`, the app forces a password change
   immediately — enforced on the server side, so it cannot be skipped.
3. Work through the **admin area** (☰ menu → Admin area) in this order:
   1. **Employees** — these become the board's columns: name, email, colour,
      skills (free text for the AI), skill tags (comma separated, for auto
      dispatching), weekly hours + buffer % (for capacity checks and the workload
      view), CRM/ERP username (for automatic assignment on ticket import).
   2. **Users** — create logins for the team (tick "admin" only where needed).
      New users must change their initial password at first login. Optional per
      user: a linked employee ("My tasks" in the iOS app) and visibility
      (teams/locations).
   3. **Teams & locations** — optional: team filter on the board, team colours and
      logos, appearance (company colour and logo for the header and Excel
      exports), the budget check threshold for time tracking.
   4. **AI settings** — see section 4.
   5. **Milestone template** — the phase model every new project inherits
      automatically (names, descriptions, AI keywords). The defaults are fine to
      start with.
   6. **Settings** — the internal ticket number range for running **without** an
      external ticketing system: your own sequential ticket numbers with a start
      value, an optional prefix (e.g. `SV-`) and zero padding.
   7. **Integrations** — only needed if you connect something; three areas in one
      tab: **1·API** (integration switches and API keys for pushing tickets from
      an ERP — documented in `API_INGEST.md`), **2·HubSpot** (pull connector with
      a private app token), **3·CSV template** for the manual task import
      (including five demo rows).

---

## 4. Connecting AI (optional, but where the value is)

Admin area → **AI settings**. The configuration lives in the database and applies
globally — no container restarts needed.

| Setting | Examples |
|---|---|
| Provider `openai` (OpenAI-compatible) | LM Studio, Ollama, vLLM, OpenAI, Azure OpenAI |
| Provider `anthropic` | Claude API (a model name is then required) |
| Provider `disabled` | No AI — the app uses its keyword fallback |

Typical setups:

- **LM Studio on the Docker host** (local, no cloud): base URL
  `http://host.docker.internal:1234/v1`, leave the model empty (auto-detect).
  "Serve on Local Network" must be enabled in LM Studio.
- **OpenAI:** base URL `https://api.openai.com/v1` + API key + model.
- **Anthropic:** leave the base URL empty + API key + model.

Verify with **"Test connection"**. The AI then handles: classification of new
tasks (assignment with a capacity check), effort estimation, milestone
assignment, suggestions for next steps and the weekly briefing.

---

## 5. HTTP / HTTPS

- By default the app is reachable over **HTTP :5050** and **HTTPS :5443** in
  parallel.
- The HTTPS certificate is self-signed (Caddy's internal CA), so browsers warn
  once. For trusted certificates, enter your hostname in the `Caddyfile` and
  distribute Caddy's root CA across your network (it sits in the `caddy-data`
  volume under `pki/authorities/local/root.crt`).
- If you only want HTTP on the LAN, remove the `caddy` service from
  `docker-compose.yml` — the app itself does not need it.

---

## 6. Backup and restore

**Automatic:** the `backup` service writes a consistent SQLite dump every night
(02:00 by default) to `/data/backups/dispatcher-YYYY-MM-DD.db` in the volume and
keeps 14 daily states (configurable in the `.env`).

### The normal route: admin area → "Database" tab

Creating, downloading and **restoring** backups works inside the app without
server access (section 1, "Backups"):

- **Create backup now** — writes a state immediately (`dispatcher-manuell-…`).
- **⬇ Download** — fetches a backup from the volume to your own machine.
- **Restore** — from the list or from an uploaded `.db` file. The current state
  is saved automatically as `dispatcher-vor-restore-…` beforehand, so the step is
  reversible. You are signed out afterwards.

Restoring goes through the **SQLite backup API** straight into the running
database, which is why it needs neither a container restart nor any juggling of
`-wal`/`-shm` files (see the warning below).

**Retention:** `BACKUP_KEEP_DAYS` (default 14) applies to the nightly states. For
named backups (*manual*, *pre-restore*), `BACKUP_KEEP_MANUAL` (default 10)
applies instead — the newest N per kind, deliberately count-based rather than
age-based so a safety copy does not vanish after 14 days.

⚠️ Downloaded backups contain the **complete** dataset including AI and HubSpot
tokens in clear text — treat them accordingly (section 9).

### Fallback: the command line

**Get backups out of the volume** (e.g. to archive them elsewhere):

```bash
docker compose cp dispatcher:/data/backups ./backups-copy
```

**Restore:**

```bash
docker compose stop dispatcher
docker compose cp ./backups-copy/dispatcher-2026-07-06.db dispatcher:/data/dispatcher.db
docker compose start dispatcher
```

**Immediate backup by hand:**

```bash
docker compose exec backup python -c "import backup; backup.make_backup('manuell')"
```

### Inspecting the database and cleaning up records (admins only)

**Inspecting — inside the app:** admin area → **"Database"** tab, section 3
**"Raw data (read only)"** lists every table with its row count and lets you page
and search through each one. Secrets (password hashes, API and HubSpot tokens)
are masked. There is deliberately **no** free-form SQL and no write function in
the interface — backup and restore live in section 1 of the same tab.

**Transferring configuration:** section 2 **"Configuration"** exports users
(permissions, visibility, language, board layout) and the global application
settings as readable JSON — **without** passwords, tokens, API keys, company logo
and the ticket number counter. It is meant for a second installation: teams and
locations are matched by **name**, not by internal ID. On import no user is
deleted; missing ones are created with a random password and a forced change.

**Cleaning up or correcting — in this order:**

1. **Normally:** through the dedicated tabs (users, employees, teams & locations;
   on the board: customers, task types, participants, projects; deleting tasks as
   an admin).
2. **Raw access** only when a case cannot be solved through the forms — and
   **take a backup first** (see above). Access through the app image (same
   `db.get_conn()`, WAL-safe):

   ```bash
   docker compose exec dispatcher python3 -c "
   import db
   with db.get_conn() as c:
       # Example: remove orphaned comments of a deleted ticket
       c.execute(\"DELETE FROM comments WHERE task_id=?\", ('<task-id>',))
       c.commit()
   print('cleaned')"
   ```

   ⚠️ No raw edits to the running database from **outside** the container (no
   `docker cp` of the `.db` back while the app is running) — WAL locking. Always
   write through a connection **inside** the container, as above.

---

## 7. Updating to a new version

```bash
cd plan-and-dispatch-ai
docker compose pull             # fetch the new image
docker compose up -d            # restart the containers on the new image
```

Option B (offline): save the new image to a file on a machine with internet, copy
it over, then `docker load -i …` and `docker compose up -d --no-build`.

To pin a specific version, set it in the `.env`:
`IMAGE=metromcd/plan-and-dispatch-ai:2.116`. Without it, `latest` applies.

The database is migrated automatically on start (additive schema changes only);
data, session key and backups stay in the volume.

**Recommended: take a backup before updating** — Admin → Database → "Create
backup now". It takes one click and makes the step reversible. Going back to an
**older** image after the database has already been migrated is not supported; in
that case restore the backup you took before the update.

**One-off when updating to 2.113 or newer:** since that version the application
no longer runs as root inside the container but as a restricted user (UID 10001).
On first start the container hands over ownership of the data directory itself —
the log then shows a single line reading
`entrypoint: Besitzrechte an /data werden einmalig auf app übergeben ...`.
**There is nothing to do.** Only if you bind-mount a host directory instead of
using the Docker volume do you need to set its owner once:
`chown -R 10001:10001 <directory>`.

---

## 8. Troubleshooting

| Problem | Solution |
|---|---|
| Port 5050/5443 already in use | Change `HTTP_PORT`/`HTTPS_PORT` in the `.env`, then `docker compose up -d` |
| `docker compose ps` shows `unhealthy` | `docker compose logs dispatcher --tail 50` — most common causes: damaged volume or a full disk |
| AI test fails (LM Studio) | Enable "Serve on Local Network" in LM Studio; test from the host: `curl http://localhost:1234/v1/models` |
| Login locked ("too many failed attempts") | Wait 5 minutes — or `docker compose restart dispatcher` (clears the lockouts) |
| Admin password forgotten | See the box below |
| Browser warns about the certificate | Expected with `tls internal` — accept the exception or distribute Caddy's root CA (section 5) |
| All sessions gone after a restart | Only happens if the volume was deleted — setting `SECRET_KEY` in the `.env` makes the key independent of the volume |

**Resetting the admin password** (directly in the database, adjust the username if
needed):

```bash
docker compose exec dispatcher python3 -c "
from werkzeug.security import generate_password_hash
import db
with db.get_conn() as c:
    c.execute(\"UPDATE users SET password_hash=?, must_change_password=1 WHERE username='admin'\",
              (generate_password_hash('Restart123'),))
    c.commit()
print('Password: Restart123 (a change is enforced at login)')"
```

---

## 9. Security — a short overview

- Session cookie login (HttpOnly, SameSite=Lax), passwords hashed
  (werkzeug/scrypt with salt)
- Login lockout: 5 failed attempts → 5 minute block
- Enforced password change for initial and reset passwords (server side)
- The ingest API requires an API key (stored SHA-256 hashed, shown in clear text
  only once)
- Board and admin interfaces are only served to signed-in users
- Excel exports are protected against formula injection (cell values starting
  with `=` are never stored as formulas)
- Dependencies in `requirements.txt` are version pinned (reproducible builds)
- The application runs as an unprivileged user (UID 10001) inside the container,
  not as root; the program code is read-only for it
- Recommended on a company network: use HTTPS (port 5443); use HTTP only if you
  consider the LAN trustworthy. The nightly backups in the volume contain the
  complete database (including AI and HubSpot tokens) — protect copies
  accordingly.
- **Set `SESSION_COOKIE_SECURE=1`** in the `.env` as soon as the app is reachable
  over HTTPS only (reverse proxy, Cloudflare Tunnel). The session cookie is then
  never sent over an unencrypted connection. The default is `0`, because a secure
  cookie would never arrive when accessed via `http://<host>:5050` — **nobody
  could sign in**. If you offer both HTTP and HTTPS, leave it at `0`.

---

*Questions and bug reports: see `CONTRIBUTING.en.md` in the release package.
Please do not report security issues publicly — write to info@dispatcher-ai.de.*
