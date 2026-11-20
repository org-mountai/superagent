# Deployment

Generische Schritte, um die Architektur für einen neuen Kundenstack aufzusetzen. Das ist eine
Checkliste, kein Skript. Namen in spitzen Klammern sind Platzhalter und werden pro Installation
ersetzt.

## 0. Vor der Installation entscheiden

- **Runtime:** Welcher Harness führt Worker aus? Der Adapter muss den Runtime-Vertrag erfüllen.
- **Kanal:** Wo spricht der First Mate mit der verantwortlichen Person? Chat, Ticketsystem oder
  Mail.
- **Quellen:** Welche Eingänge speisen den Bestand? Code-Host, Mail, Projektmanagement,
  Zeitpläne, Host-Checks.
- **Host:** Welcher Host führt Worker aus? Ein eigener Host oder ein eigenes Konto pro
  Vertrauensgrenze; keine Vermischung mit dem Arbeitsplatz.
- **Datengrenze:** Was darf die Kundenumgebung verlassen? Welche Ereignisse bleiben
  metadaten-only?
- **Freigabe:** Wer gibt R3 frei, und wo wird die Freigabe dokumentiert?
- **Modelle:** schnelles Modell für Worker und Klassifizierung, stärkeres Modell für den First
  Mate; beide austauschbar. Die Modellwahl ist Konfiguration, keine Identität.

## 1. Voraussetzungen

- Python 3 in der von der Implementierung unterstützten Version, SQLite, Git.
- Ein Scheduler: systemd-User-Timer oder ein äquivalenter Dienst.
- Eine Worker-Runtime mit funktionierendem Adapter und einem Fake-Adapter für Offline-Proben.
- Ein dediziertes Dienstkonto ohne interaktiven Login.
- Secrets außerhalb des Repositories, Dateien nur für den Dienst lesbar; keine Werte in Tickets,
  Logs oder Prompts.
- Ein Code-Host-Konto mit den minimal nötigen Rechten für Issues und Pull Requests.

## 2. Verzeichnisse

```text
<app-dir>       Code und versionierte Konfiguration
<state-dir>     SQLite, Cursor, Logs (gesichert, nicht in Git)
<config-dir>    Policy, Runtime-Definitionen, Watchdog-Grenzen
```

`<state-dir>` und `<config-dir>` gehören nicht ins Repository. Secrets liegen getrennt und
werden nur über Umgebungsvariablen oder geschützte Dateien geladen.

## 3. Konfiguration

- **Policy-Allowlist:** wer Arbeit auslösen darf. Kanonisch, versioniert, wie Code reviewt.
- **Runtime-Konfiguration:** Adapter, Modell, Provider, Grenzen. Keine Identität, keine Policy.
- **Watchdog-Grenzen:** Hänger-Schwellen, Retry-Limits, Outbox-Versuche.
- **Kanal-Konfiguration:** erlaubte Ereignistypen, Ziel-Label, Dry-run-Schalter.
- **Host-Broker-Allowlist:** exakte Paare aus Befund und Befehl; standardmäßig leer.
- **Secrets:** nur der Ort wird dokumentiert, nie der Wert.

## 4. Zustand initialisieren

1. Datenbank und Schema-Versionstabelle anlegen.
2. Sofort nach der Initialisierung und vor jeder Migration eine Kopie ziehen.
3. Für jeden Kanal und jede Brücke eine Baseline setzen.
4. Prüfen, dass der erste Lauf keine alte Historie versendet oder Tickets anlegt.

## 5. Offline prüfen

Vor dem ersten echten Lauf:

- Fake-Runtime: Start, Status, Ergebnis, Stop.
- Doppelstart und Doppel-Sync: keine Doppelwirkung.
- Kanal-Sync im Dry-run: keine Tickets, keine Meldungen.
- Host-Broker: Allowlist-Treffer, Nichttreffer, Audit, Fehlerstopp, manueller Retry.
- Neustart: Zustand wird vor dem nächsten Start vereinheitlicht.
- Brücke: Baseline, Cursor, fehlender oder mehrdeutiger Thread stoppt sichtbar.

## 6. Kanal verbinden

1. Den dauerhaften Thread der verantwortlichen Person mit einem Rollen-Label versehen.
2. Das Label bei jedem Versand frisch auflösen; genau ein aktiver Treffer ist Pflicht.
3. Bei null oder mehreren Treffern sichtbar stoppen, niemals eine alte ID als Fallback raten.
4. Einen ausdrücklichen Empfangstest senden und die Antwort abwarten.
5. Erst danach die Baseline setzen und den Timer aktivieren.

## 7. Zeitpläne aktivieren

Ein 15-Minuten-Lauf ruft in dieser Reihenfolge auf:

1. Ticket-Eingang prüfen.
2. Kanal-Sync (Mail, Projektmanagement, Zeitplan) → R0-Tickets.
3. Host-Broker für allowlistete Befunde.
4. Event-Brücke → First-Mate-Thread.
5. Watchdog: Zustand abgleichen, Blocker melden.

Der Takt ist Konfiguration. Der Host-Schnellcheck läuft täglich, der Sicherheitscheck wöchentlich
und der Tiefcheck monatlich, wenn der Kunde diese Rhythmen will. Ein Lauf meldet nur Änderungen
und Blocker, keine Zustände.

## 8. Abnahme

Ein harmloses End-to-End-Ticket muss zeigen:

- genau einen Worker-Start,
- genau einen PR/Diff oder einen Bericht,
- genau eine Outbox-/Kanalmeldung,
- einen zweiten Sync ohne Wirkung,
- Audit-Einträge mit Zeitpunkt und Ursache,
- keine versteckte Rechtevergabe.

Für die Host-Schleife:

- Befund → Entscheidung → Ausführung → Exit-Code/Nachweis,
- ein späterer Check ohne Befund schließt das Ticket,
- dieselbe Befundversion führt nicht erneut aus,
- ein Fehler bleibt sichtbar und wird nicht still wiederholt.

Für die Statusprojektion:

- Offene R0-Kanal-Tickets aus Mail, Projektmanagement und Zeitplan lassen sich nach einer
  zweiten Bestätigung verwerfen.
- Der Server prüft Quelle und Status erneut und schreibt den Audit-Eintrag.
- GitHub-/Host-Tickets und alle R2-/R3-Aktionen werden unverändert abgelehnt.
- Ein Folge-Sync bleibt `quiet`.

## 9. Betrieb

- **Täglich:** `status --json`, Blocker, fehlgeschlagene Timer, Alter des letzten Backups.
- **Wöchentlich:** hängende Tickets, erreichte Retry-Limits, offene R3-Freigaben.
- **Monatlich:** Restore-Test, Schema-Review, Rechte-Review, Archivierung alter Ereignisse.
- **Meldungen:** nur Änderungen und Blocker; kein Fortschritts-Spam.

## 10. Rollback

1. Timer stoppen, dann Worker stoppen.
2. Datenbankkopie zusammen mit der passenden Code-/Konfigurationsversion zurückrollen.
3. Worker- und Host-Zugangsdaten widerrufen.
4. Das Zustandsverzeichnis aufbewahren; ein „Reset“ durch Löschen ist kein Rollback.

## 11. Für jeden neuen Stack offen

- Produktive Wiederherstellung auf einem separaten Testhost; Backup, Integritätsprüfung,
  read-only Restore-Sicht und Schema-Versionierung sind belegt.
- Automatische Mailkörper-Verarbeitung und -Beantwortung; die Signalerfassung ist belegt
  (siehe [Gotchas](gotchas.md)).
- Freigabeoberfläche und Aufbewahrungsfristen für Audits.
- Kosten- und Parallelitätsbudgets.
