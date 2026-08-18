# Änderungsprotokoll

*[English version](CHANGELOG.en.md)*

Alle nennenswerten Änderungen an Plan and Dispatch AI. Die Versionsnummer steht in
`config.APP_VERSION` und wird im Admin-Bereich unter „Über diese App" angezeigt.

Versionsschema: `Major.Minor` mit fortlaufendem Minor (2.110 → 2.111 → … → 2.199).
Ein Major-Sprung kennzeichnet einen größeren Umbau.

---

## 2.118

### Geändert
- **Das Abbild liegt jetzt unter `plananddispatchai/plan-and-dispatch-ai`.** Der
  bisherige Veröffentlichungsname war ein Benutzerkürzel; Docker-Konten lassen
  sich nicht umbenennen, daher der Wechsel auf ein Konto mit passendem Namen.
  Wer die Compose-Datei aus dem Release-Paket nutzt, bekommt die Änderung beim
  nächsten Aktualisieren automatisch — feste Verweise auf den alten Namen bitte
  einmal anpassen, Docker Hub leitet nicht weiter.

## 2.117

### Geändert
- Das Abbild verweist jetzt auf sein Projekt-Repository (Standard-Metadatum
  `source`) — Registries und Werkzeuge verlinken von dort auf Dokumentation und
  Fehlermeldungen.

## 2.116

### Behoben
- **Zehn bekannte Schwachstellen in der Bildbibliothek Pillow** (Version 12.2.0 →
  12.3.0). Betroffen war das Verarbeiten manipulierter Bilddateien; im Dispatcher
  wird Pillow für das Firmenlogo in den Excel-Exporten genutzt.
- **Sicherheitsaktualisierungen der Distribution** werden beim Bauen des Images
  eingespielt. Damit schleppt eine neue Version keine Lücken mehr mit, für die es
  längst einen Fix gibt.

### Geändert
- Das Image weist sich jetzt selbst aus (Name, Version, Hersteller, Lizenz,
  Erstellungsdatum als Standard-Metadaten) — Registries und Werkzeuge zeigen das an.
- Das zugrunde liegende Python-Image ist auf einen festen Stand genagelt, damit ein
  Neubau derselben Version dasselbe Ergebnis liefert.
- Nicht ausgelieferte Entwicklungsdateien bleiben zuverlässig draußen. Eine interne
  Entwickler-Notiz zum Outlook-Add-in war bislang versehentlich im Image enthalten.

## 2.115

### Geändert
- **Ein frisch aufgesetzter Container startet nicht mehr mit einem schwarzen
  Balken oben,** sondern mit einem Petrolblau (`#00718f`) als Vorbelegung. Der Ton
  ist dunkel genug, dass weiße Schrift darauf die Lesbarkeitsanforderung WCAG AA
  erfüllt (5,6:1). Die Farbe lässt sich wie bisher unter Admin → Teams & Standorte
  → Erscheinungsbild ändern; die Schaltfläche daneben stellt auf Schwarz.
- **Bestehende Installationen sind nicht betroffen** — die Vorbelegung greift nur
  dort, wo noch nie eine Farbe hinterlegt wurde. Wer bewusst Schwarz eingestellt
  hat, behält Schwarz.
- Die Schaltfläche hieß „Zurücksetzen (Schwarz)" und heißt jetzt „Auf Schwarz
  setzen" — Schwarz ist nicht mehr der Standard, sondern eine Auswahl.

## 2.114

### Behoben
- **Im Filter-Dialog waren „Status", „Team" und „Sortierung" unsichtbar.** Die drei
  Auswahlfelder trugen noch das Styling aus der dunklen Kopfleiste, in der sie
  früher saßen — weiße Schrift auf weißem Grund. Sie waren bedienbar, aber man sah
  weder das Feld noch die getroffene Auswahl. Jetzt nutzen sie das normale
  Formular-Styling des Dialogs und füllen dessen Breite.

### Geändert
- **Der Filter-Knopf zeigt deutlicher, dass Filter gesetzt sind.** Bisher gab es nur
  einen kleinen blauen Punkt, der auf der Trichterspitze kaum auffiel; jetzt ist der
  ganze Knopf eingefärbt. Der Tooltip nennt zusätzlich, *welche* Felder abweichen
  („Filter aktiv: Status, Team") — ohne dass man den Dialog öffnen muss.

## 2.113

### Geändert
- **Die Anwendung läuft im Container nicht mehr als Administrator (root),** sondern
  als eingeschränkter Benutzer. Findet eine Lücke in der Anwendung, sind die Folgen
  damit deutlich begrenzter. **Es ist nichts zu tun:** beim ersten Start nach dem
  Update übergibt der Container die Besitzrechte am Datenverzeichnis einmalig
  selbst — bestehende Installationen laufen ohne Handgriff weiter. Wer das
  Datenverzeichnis als Ordner vom Host einbindet statt als Docker-Volume, muss ihm
  einmalig den Benutzer `10001` zuweisen.
- Der Programmcode im Container ist für die Anwendung nur noch lesbar, nicht mehr
  beschreibbar.

### Behoben
- **Der Container ließ sich nicht sauber stoppen.** Das Stopp-Signal erreichte den
  Webserver nicht, sodass er nach zehn Sekunden hart abgebrochen wurde — laufende
  Anfragen gingen dabei verloren und jeder Neustart kostete diese zehn Sekunden.
  Beides ist behoben, das Herunterfahren erfolgt jetzt geordnet.

## 2.112

### Neu
- **Dieses Änderungsprotokoll.** Ab sofort wird jede Version hier festgehalten.
- **`SESSION_COOKIE_SECURE`** als Umgebungsvariable: Wer die App ausschließlich über
  https betreibt, kann das Session-Cookie auf verschlüsselte Verbindungen beschränken,
  ohne das Image anzupassen. Standard bleibt aus, damit der Zugriff über
  `http://<host>:5050` weiter funktioniert.

### Geändert
- **Der pauschale Beta-Hinweis über dem Tab „Schnittstellen" ist weg.** Stattdessen
  tragen die beiden tatsächlich jungen Bereiche — HubSpot und Outlook-Add-in — ein
  Kennzeichen „In Erprobung" im Abschnittskopf. Push-API und CSV-Vorlage gelten als
  stabil und sind nicht mehr mitgekennzeichnet.
- **Lizenztext präzisiert:** Der Abschnitt „License Grant" sprach vom „compiled Docker
  image". Das Image enthält Teile der Software in lesbarer Form; der Text sagt jetzt
  klar, dass daraus keinerlei Rechte am Quellcode entstehen. Die eingeräumten Rechte
  ändern sich nicht.

## 2.111

### Neu
- **Datenbanksicherung im Admin-Bereich** (Tab „Datenbank"): Sicherungen auflisten,
  auf Knopfdruck erstellen, herunterladen und löschen. Bis dahin ging das nur über
  die Kommandozeile auf dem Server.
- **Rücksicherung** aus der Serverliste oder aus einer hochgeladenen `.db`-Datei —
  letzteres ist der Weg zurück, wenn das Docker-Volume selbst verloren geht. Vor
  jeder Rücksicherung wird automatisch eine Sicherheitskopie des aktuellen Standes
  angelegt.
- **Konfigurations-Export als JSON**: Benutzer (Rechte, Sichtbarkeiten, Sprache,
  Board-Layout) und globale App-Einstellungen — bewusst **ohne** Passwörter, API-
  Schlüssel und Token. Lässt sich auf eine zweite Installation übertragen.

### Geändert
- Benannte Sicherungen (manuell / vor Rücksicherung) werden anzahlbasiert
  aufbewahrt (`BACKUP_KEEP_MANUAL`, Standard 10) statt nach Alter — eine
  Vor-Restore-Kopie soll nicht verfallen, nur weil länger nichts passiert ist.

### Behoben
- Sicherungsdateien übernahmen den WAL-Journalmodus der Quelldatenbank und ließen
  sich ohne ihre Begleitdatei nicht einmal lesend öffnen. Betraf auch die bisherigen
  nächtlichen Sicherungen.
- Der App-Container kannte die `BACKUP_*`-Variablen nicht und hätte beim Aufräumen
  die Standardwerte statt der eingestellten verwendet.

## 2.109

### Neu
- **Installationspaket für das Outlook-Add-in** direkt aus dem Admin-Bereich: das
  ZIP wird je Installation mit der eigenen Serveradresse erzeugt.

## 2.108

### Behoben
- KI-Aufrufe schlugen bei Reasoning-Modellen fehl, weil die Antwort leer zurückkam.

## 2.107

### Neu
- **Outlook-Add-in**: E-Mail per Klick als Aufgabe anlegen — wahlweise in der eigenen
  Spalte oder im Eingang. Erkennt den Kunden aus der Absenderdomain und lernt die
  Zuordnung, wenn sie im Board korrigiert wird.

## 2.106

### Behoben
- Nach einem erzwungenen Passwortwechsel blieb das Board leer, bis man die Seite von
  Hand neu lud.

## 2.105

### Geändert
- Die Titel-Rechtschreibprüfung greift jetzt auch beim KI-Vorschlag im Aufgabendialog.

## 2.103 – 2.104

### Neu
- **Login-Historie** im Admin-Bereich: wer gerade angemeldet ist und ein Protokoll je
  Benutzer (Zeit, Erfolg, IP, Browser). Liegt im Reiter „Benutzer".

## 2.99 – 2.102

### Neu
- **Restbudget auf der Aufgabenkachel** sowie eine Budget-Aufschlüsselung und ein
  Budgetverlauf im Aufgabendialog.

### Behoben
- Die Berechtigung „Stammdaten pflegen" wurde im Admin-Dialog nicht gemerkt.

## 2.98

### Neu
- **MCP-Server**: KI-Clients wie Claude Desktop können das Board lesen und pflegen.
- **Datenbank-Browser** im Admin-Bereich (nur lesend, Geheimnisse maskiert).
- **Projekt-Owner** je Projekt und Zeiterfassung direkt am Arbeitsschritt.

## 2.90 – 2.94

### Neu
- **Handbuch und interaktives Onboarding im Container** — erreichbar aus dem Menü,
  zweisprachig.

### Behoben
- Board-Absturz durch eine Namenskollision im Wiedervorlage-Balken.

## 2.82

### Neu
- **Schutz gegen Prompt-Injection**: Daten aus Fremdsystemen (Ticket-Titel,
  Beschreibungen) werden gekapselt an das Sprachmodell übergeben, die Antwort vor dem
  Speichern geprüft und gekürzt.

## 2.71

### Neu
- **Systemsprache Deutsch/Englisch** pro Benutzer, auch im Admin-Bereich, samt
  Nachtmodus.

---

## Frühere Versionen

Ältere Stände sind in der Git-Historie des Entwicklungsrepositorys dokumentiert.
Wesentliche Bausteine, die dort entstanden sind: Kanban-Board mit einer Spalte je
Mitarbeiter, KI-Dispatching mit Kapazitätsprüfung, Auslastungssicht, Projektplan mit
Meilensteinen, Soll/Ist-Vergleich, Zeiterfassung mit Budgetprüfung, Excel-Exporte,
Ingest-Schnittstelle für ERP-/Ticketsysteme, HubSpot-Konnektor und der Import aus
Excel/CSV.
