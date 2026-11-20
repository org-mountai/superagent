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
  R0-Tickets ohne Nachrichtenkörper im selben Ticketbestand geführt.
- **Eine read-only Statusprojektion:** Das Paseo-Plugin
  [org-mountai/paseo-superagent](https://github.com/org-mountai/paseo-superagent) zeigt
  Watchdog, Brücke, offene Tickets, Worker und Blocker, ohne Schreibaktionen.

Das ist der belegte Stand. Die Architektur ist bewusst größer als die Implementierung.

## Bewusst offen

- **Backup und Schema-Härtung** für den Ticketbestand. Die SQLite-Datenbank ist nicht beliebig
  rekonstruierbar; ein Backup- und Migrationskonzept ist noch Arbeit, kein fertiger Zustand.
- **Vollständige Per-Mail-Erfassung.** Der aktuelle Kanal verarbeitet Ereignisse und Metadaten,
  nicht jede Mail und nicht ihre Körper. Das ist ausdrücklich noch nicht umgesetzt.
- Freigabe- und Steuerungsoberflächen, Kosten- und Parallelitätsbudgets, weitere
  Runtime-Adapter und das Skalieren auf mehrere Worker-Hosts.
- Eine Lizenz ist noch nicht gewählt; dieses Repository enthält bewusst keine `LICENSE`-Datei.

## Dokumente

- [Architektur](docs/architecture.md) — Bausteine, Ticketfluss, Risikoklassen, Verträge
- [Deployment](docs/deployment.md) — generische Schritte für einen neuen Kundenstack
- [Gotchas](docs/gotchas.md) — Fehler, die die Referenzimplementierung bereits bezahlt hat
- [Paseo-Statusplugin](https://github.com/org-mountai/paseo-superagent) — read-only Projektion

## Abgrenzung

Kein Produktivcode. Keine Kundendaten. Hosts, Konten, Ticketnummern und Zugangsdaten sind
Platzhalter. Die Dokumente beschreiben die Form, nicht eine bestimmte Installation.
