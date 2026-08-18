# Changelog

*[Deutsche Fassung](CHANGELOG.md)*

All notable changes to Plan and Dispatch AI. The version number lives in
`config.APP_VERSION` and is shown in the admin area under "About this app".

Versioning: `Major.Minor` with a running minor (2.110 → 2.111 → … → 2.199).
A major bump marks a larger rebuild.

---

## 2.118

### Changed
- **The image now lives at `plananddispatchai/plan-and-dispatch-ai`.** The
  previous publisher name was a personal handle; Docker accounts cannot be
  renamed, hence the move to an account with a fitting name. If you use the
  compose file from the release package, the change arrives with your next
  update — please adjust any hard-coded references to the old name, as Docker
  Hub does not redirect.

## 2.117

### Changed
- The image now points to its project repository (the standard `source`
  metadata), so registries and tooling can link from there to documentation and
  issue reporting.

## 2.116

### Fixed
- **Ten known vulnerabilities in the Pillow imaging library** (12.2.0 → 12.3.0).
  They affected the handling of crafted image files; Pillow is used here to place
  the company logo into the Excel exports.
- **Distribution security updates** are now applied while the image is built, so a
  new release no longer ships known holes that have long since been patched.

### Changed
- The image now identifies itself (name, version, vendor, licence, build date as
  standard metadata) — registries and tooling display this.
- The underlying Python image is pinned to a fixed state, so rebuilding the same
  version produces the same result.
- Development files that are not part of the product stay out reliably. An
  internal developer note about the Outlook add-in had been included by mistake.

## 2.115

### Changed
- **A freshly set up container no longer starts with a black bar at the top,**
  but with a petrol blue (`#00718f`). The shade is dark enough for white text on
  it to meet the WCAG AA readability requirement (5.6:1). The colour can be
  changed as before under Admin → Teams & locations → Appearance; the button next
  to it switches to black.
- **Existing installations are unaffected** — the preset only applies where no
  colour has ever been stored. Anyone who deliberately chose black keeps black.
- The button used to read "Reset (black)" and now reads "Set to black" — black is
  no longer the default, it is a choice.

## 2.114

### Fixed
- **"Status", "Team" and "Sorting" were invisible in the filter dialog.** The three
  dropdowns still carried the styling of the dark header bar they used to sit in —
  white text on a white background. They were operable, but you could see neither
  the field nor your selection. They now use the dialog's normal form styling and
  fill its width.

### Changed
- **The filter button shows more clearly that filters are set.** There used to be
  only a small blue dot that was easy to miss on the funnel's tip; now the whole
  button is tinted. The tooltip additionally names *which* fields deviate
  ("Filters active: Status, Team"), so you do not have to open the dialog.

## 2.113

### Changed
- **The application no longer runs as the administrator (root) inside the
  container,** but as a restricted user. Should a hole be found in the
  application, the consequences are far more contained. **Nothing to do on your
  side:** on the first start after the update, the container hands over ownership
  of the data directory by itself — existing installations keep running untouched.
  If you bind-mount a host directory instead of using a Docker volume, assign it
  to user `10001` once.
- The program code inside the container is now read-only for the application.

### Fixed
- **The container could not be stopped cleanly.** The stop signal never reached
  the web server, so it was killed after ten seconds — in-flight requests were
  lost and every restart cost those ten seconds. Both are fixed; shutdown is now
  orderly.

## 2.112

### Added
- **This changelog.** From now on every version is recorded here.
- **`SESSION_COOKIE_SECURE`** as an environment variable: if you run the app over
  HTTPS only, you can restrict the session cookie to encrypted connections without
  touching the image. It stays off by default so that access via
  `http://<host>:5050` keeps working.

### Changed
- **The blanket beta notice above the "Integrations" tab is gone.** Instead, the
  two genuinely young areas — HubSpot and the Outlook add-in — carry an "In trial"
  tag in their section heading. The push API and the CSV template are considered
  stable and are no longer flagged.
- **Licence wording clarified:** the "License Grant" section spoke of the "compiled
  Docker image". The image contains parts of the software in readable form; the
  text now states plainly that this grants no rights to the source code whatsoever.
  The rights granted are unchanged.

## 2.111

### Added
- **Database backups in the admin area** ("Database" tab): list, create on demand,
  download and delete backups. Until now this was only possible from the command
  line on the server.
- **Restore** from the server-side list or from an uploaded `.db` file — the latter
  is the way back if the Docker volume itself is lost. Before every restore, a
  safety copy of the current state is created automatically.
- **Configuration export as JSON**: users (permissions, visibility, language, board
  layout) and global application settings — deliberately **without** passwords, API
  keys and tokens. Can be transferred to a second installation.

### Changed
- Named backups (manual / pre-restore) are retained by count
  (`BACKUP_KEEP_MANUAL`, default 10) rather than by age — a pre-restore copy should
  not expire just because nothing happened for a while.

### Fixed
- Backup files inherited the WAL journal mode of the source database and could not
  even be opened read-only without their companion file. This affected the existing
  nightly backups too.
- The application container did not know about the `BACKUP_*` variables and would
  have pruned using the defaults instead of the configured values.

## 2.109

### Added
- **Installation package for the Outlook add-in** straight from the admin area: the
  ZIP is generated per installation with that installation's own server address.

## 2.108

### Fixed
- AI calls failed with reasoning models because the response came back empty.

## 2.107

### Added
- **Outlook add-in**: turn an email into a task with one click — either in your own
  column or in the inbox. Recognises the customer from the sender's domain and
  learns the mapping when it is corrected on the board.

## 2.106

### Fixed
- After a forced password change the board stayed empty until the page was reloaded
  by hand.

## 2.105

### Changed
- The title spellcheck now also applies to the AI suggestion in the task dialog.

## 2.103 – 2.104

### Added
- **Login history** in the admin area: who is currently signed in, plus a log per
  user (time, success, IP, browser). Located in the "Users" tab.

## 2.99 – 2.102

### Added
- **Remaining budget on the task card**, plus a budget breakdown and a budget
  history in the task dialog.

### Fixed
- The "maintain master data" permission was not remembered in the admin dialog.

## 2.98

### Added
- **MCP server**: AI clients such as Claude Desktop can read and maintain the board.
- **Database browser** in the admin area (read-only, secrets masked).
- **Project owner** per project, and time tracking directly on the work step.

## 2.90 – 2.94

### Added
- **Manual and interactive onboarding inside the container** — reachable from the
  menu, bilingual.

### Fixed
- Board crash caused by a name collision in the follow-up bar.

## 2.82

### Added
- **Protection against prompt injection**: data from external systems (ticket
  titles, descriptions) is passed to the language model in a fenced form, and the
  response is validated and truncated before being stored.

## 2.71

### Added
- **System language English/German** per user, including the admin area, plus dark
  mode.

---

## Earlier versions

Older states are documented in the development repository's Git history. The
substantial building blocks created there: the Kanban board with one column per
team member, AI dispatching with capacity checks, the workload view, the project
plan with milestones, the planned/actual comparison, time tracking with budget
checks, Excel exports, the ingest interface for ERP and ticketing systems, the
HubSpot connector and the Excel/CSV import.
