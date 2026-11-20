# Superagent

**Ein kleiner, deterministischer Eingang und eine Worker-Architektur für persönliche
Assistenz- und Agenten-Teams.**

Dieses Repository ist ein Dokumentations-Seed, nicht die Referenzimplementierung. Es beschreibt
Architektur, Verträge, Betriebsregeln und den Deployment-Rahmen, die in einem privaten Lab
belegt wurden. Der Code bleibt vorerst privat; diese Dokumente sind der öffentliche,
wiederverwendbare Teil.

## Die Idee

Ein Gesicht, viele austauschbare Arbeitskräfte, sichtbare Verantwortung.

Der Companion (der „First Mate“) spricht mit der verantwortlichen Person, zerlegt Arbeit, holt
Freigaben ein und fasst Ergebnisse zusammen. Worker erledigen je eine eng begrenzte Aufgabe in
einer isolierten Umgebung und sprechen nie direkt mit dem Menschen. Die zentrale Regel:

> Erst im Code nachsehen, mit einem Modell nur entscheiden, wenn es etwas zu entscheiden gibt.

Eingang, Policy und Routing sind deterministisch. Ein Modell wird nur bei echter Ambiguität
eingeschaltet. Jede Hintergrundarbeit hat ein Ticket; ohne Ticket keine Arbeit. Modelle,
Laufzeitumgebungen und Oberflächen sind austauschbare Achsen, keine Identität.

## Belegter Stand (20.11.2026)

Die private Referenzimplementierung hatte zu diesem Stand folgende Scheiben belegt:

- **Ein Repo-Durchlauf:** Ein Ticket startete genau einen isolierten Worker, der einen
  Pull Request lieferte. Ergebnis und Outbox-Meldung entstanden genau einmal; ein zweiter Start
  und ein zweiter Sync blieben ohne Doppelwirkung. Gemergt wurde menschlich.
- **Ein Watchdog:** Ein 15-Minuten-Timer glich aktive Tickets ab, zeigte Blocker, Hänger und
  Retry-Grenzen und startete selbst keinen Worker und keinen Merge.
- **Eine Host-Schleife:** Ein täglicher read-only Check führte Befunde idempotent als Tickets;
  eine eng allowlistete, auditierte R2-Host-Aktion lief vollständig durch
  Befund → Entscheidung → Ausführung → Nachweis.
- **Ein dauerhafter First-Mate-Thread:** Ausgewählte Ticket-Ereignisse wurden per Label an genau
  einen langlebigen Thread gesendet, mit Cursor und ohne Wiederholung alter Historie.
- **Ein Kanal-Adapter:** Mail-, Projektmanagement- und Zeitplan-Signale wurden als idempotente
  R0-Tickets ohne Nachrichtenkörper im selben Ticketbestand geführt. Die Mail-Signalerfassung
  ist vollständig: für die ID-basierte Mailquelle alle neuen IDs aufsteigend, für IMAP alle
  neuen ungelesenen Nachrichten vor dem letzten Cursor; ohne Cursor bleibt genau eine Nachricht
  der sichere Fallback. Vier Offline-Proben und ein echter Lauf mit drei getrennten Tickets und
  genau einer Brückenmeldung sind grün.
- **Eine Statusprojektion mit engem R0-Drop:** Das Paseo-Plugin
  [org-mountai/paseo-superagent](https://github.com/org-mountai/paseo-superagent) zeigt
  Watchdog, Brücke, offene Tickets, Worker und Blocker. Offene R0-Kanal-Tickets aus Mail,
  Projektmanagement und Zeitplan lassen sich nach einem zweiten Bestätigungsklick verwerfen;
  der Server prüft Quelle und Status erneut und nutzt den bestehenden Drop-Befehl samt Audit.
  GitHub-/Host-Tickets und alle R2-/R3-Aktionen sind ausgeschlossen. Eine Guard-Probe gegen ein
  GitHub-Ticket wurde unverändert abgelehnt, eine Zeitplan-Probe genau einmal verworfen; der
  Folge-Sync blieb `quiet`, Typecheck und Live-Reload sind grün. Weitere Steuerung gibt es nicht.
- **Backup und Schema:** Das Backup-Werkzeug nutzt SQLite-Online-Backup mit
  Integritätsprüfung, read-only Restore-Sicht und Retention 7. `PRAGMA user_version=1`
  versioniert das Schema; neuere unbekannte Versionen werden abgelehnt. Zwei echte Backups
  mit realem Bestand waren `ok`, ein täglicher Timer läuft und ist aktiviert. Zwei
  Offline-Proben und die komplette Suite sind grün.

Das ist der belegte Stand. Die Architektur ist bewusst größer als die Implementierung.

## Bewusst offen

- **Produktive Wiederherstellung.** Backup, Integritätsprüfung, read-only Restore-Sicht und
  Schema-Versionierung sind belegt; die echte produktive Wiederherstellung auf einem separaten
  Testhost bleibt offen.
- **Mailkörper-Verarbeitung.** Die Per-Mail-Signalerfassung ist belegt; Körper werden aber
  weiterhin nicht automatisch gelesen oder beantwortet. Diese Grenze bleibt offen.
- Freigabe- und Steuerungsoberflächen jenseits des R0-Drops, Kosten- und
  Parallelitätsbudgets, weitere Runtime-Adapter und das Skalieren auf mehrere Worker-Hosts.
- Eine Lizenz ist noch nicht gewählt; dieses Repository enthält bewusst keine `LICENSE`-Datei.

## Dokumente

- [Architektur](docs/architecture.md) — Bausteine, Ticketfluss, Risikoklassen, Verträge
- [Deployment](docs/deployment.md) — generische Schritte für einen neuen Kundenstack
- [Gotchas](docs/gotchas.md) — Fehler, die die Referenzimplementierung bereits bezahlt hat
- [Paseo-Statusplugin](https://github.com/org-mountai/paseo-superagent) — Statusprojektion plus eng begrenzter R0-Drop

## Abgrenzung

Kein Produktivcode. Keine Kundendaten. Hosts, Konten, Ticketnummern und Zugangsdaten sind
Platzhalter. Die Dokumente beschreiben die Form, nicht eine bestimmte Installation.
