<img src="https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/assets/logo.png" alt="Plan and Dispatch AI" width="120" align="right">

# Plan and Dispatch AI

*[Deutsche Fassung](README.md)*

A task dispatching board for consulting and service teams, packaged as a Docker
container for your own network. No cloud dependency: your data stays in a SQLite
volume on your server, and the AI features are optional and can run locally.

## Quick start

```bash
mkdir plan-and-dispatch-ai && cd plan-and-dispatch-ai
curl -O https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/Caddyfile
curl -o .env https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/.env.example

# Set ADMIN_PASSWORD in .env — the stack refuses to start without it
docker compose up -d
```

Then open `http://<server>:5050` or `https://<server>:5443` and sign in. Full
instructions, including offline installation, AI setup, backups and updates:
**[INSTALL.en.md](INSTALL.en.md)**

Available for **linux/amd64** and **linux/arm64**.

## What it does

- **Kanban board** with one column per team member, multi-user login, team
  filtering and a per-account column layout
- **AI dispatching:** incoming tasks get classified, estimated and assigned to
  the right person — with a capacity check, so nobody gets overbooked
- **Workload view** as a matrix of people against weeks
- **Project plan** with milestones and a **planned/actual comparison** against
  recorded time
- **Time tracking** per ticket, with budget checks
- **Integrations:** push API for ERP and ticketing systems, HubSpot connector,
  Outlook add-in (turn an email into a task), Excel/CSV import, MCP server for
  AI clients
- **Operations:** nightly backups, backup and restore from the admin area,
  HTTPS via Caddy, health check, login lockout after failed attempts

The interface is available in English and German, switchable per user.

## AI is optional

The board works fully without an AI endpoint; classification then falls back to
keyword matching. If you do use AI, you choose the provider: a local model
(LM Studio, Ollama — nothing leaves your machine) or a hosted one (OpenAI, Azure,
Anthropic).

## Licence and status

Freeware for any use, commercial included. The source code is **not** open source
and the software is not released for forking or derivative works — details in
**[LICENSE.md](LICENSE.md)**.

This is a side project by a private individual, not a company. There is no
guarantee of availability or support. Bug reports are welcome (see
[CONTRIBUTING.en.md](CONTRIBUTING.en.md)); code contributions are not accepted.

Changes per version: **[CHANGELOG.en.md](CHANGELOG.en.md)**
