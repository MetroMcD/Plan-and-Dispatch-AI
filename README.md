<img src="https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/assets/plan-and-dispatch-ai-icon.png" alt="Plan and Dispatch AI" width="120" align="right">

# Plan and Dispatch AI

*[English version](README.en.md)*

Aufgaben-Dispatching-Board für Beratungs- und Serviceteams — als Docker-Container
für das eigene Netz. Kein Cloud-Zwang: die Daten bleiben in einem SQLite-Volume
auf Ihrem Server, die KI-Anbindung ist optional und kann lokal laufen.

## Schnellstart

```bash
mkdir plan-and-dispatch-ai && cd plan-and-dispatch-ai
curl -O https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/Caddyfile
curl -o .env https://raw.githubusercontent.com/MetroMcD/Plan-and-Dispatch-AI/main/.env.example

# ADMIN_PASSWORD in der .env setzen — ohne das startet der Stack nicht
docker compose up -d
```

Danach `http://<server>:5050` bzw. `https://<server>:5443` öffnen und anmelden.
Vollständige Anleitung inklusive Offline-Installation, KI-Anbindung, Sicherung
und Updates: **[INSTALL.md](INSTALL.md)**

## Was es kann

- **Kanban-Board** mit einer Spalte je Mitarbeiter, Mehrbenutzer-Login,
  Team-Filter und persönlichem Spalten-Layout
- **KI-Dispatching:** neue Aufgaben werden klassifiziert, im Aufwand geschätzt und
  dem passenden Mitarbeiter zugeordnet — mit Kapazitätsprüfung, damit niemand
  überbucht wird
- **Auslastungssicht** als Matrix Mitarbeiter × Woche
- **Projektplan** mit Meilensteinen und **Soll/Ist-Vergleich** gegen die erfasste Zeit
- **Zeiterfassung** je Ticket mit Budgetprüfung
- **Anbindungen:** Push-API für ERP-/Ticketsysteme, HubSpot, Outlook-Add-in
  (E-Mail → Aufgabe), Excel-/CSV-Import, MCP-Server für KI-Clients
- **Betrieb:** nächtliche Sicherung, Sicherung und Rücksicherung im Admin-Bereich,
  HTTPS über Caddy, Healthcheck, Login-Sperre nach Fehlversuchen

## KI ist optional

Ohne KI-Anbindung funktioniert das Board vollständig; die Klassifizierung fällt
dann auf eine Stichwortlogik zurück. Wer KI nutzt, hat die Wahl zwischen einem
lokalen Modell (LM Studio, Ollama — die Daten verlassen den Rechner nicht) und
einem Anbieter (OpenAI, Azure, Anthropic).

## Lizenz und Status

Freeware für beliebige Nutzung, auch kommerziell. Der Quellcode ist **nicht**
Open Source und die Software wird nicht als Fork oder Ableitung freigegeben —
Einzelheiten in **[LICENSE.md](LICENSE.md)**.

Dies ist ein Nebenprojekt einer Privatperson, kein Unternehmen. Es gibt keine
Zusicherung von Verfügbarkeit oder Unterstützung. Fehlerberichte sind willkommen
(siehe [CONTRIBUTING.md](CONTRIBUTING.md)), Code-Beiträge nehme ich nicht an.

Änderungen je Version: **[CHANGELOG.md](CHANGELOG.md)**
