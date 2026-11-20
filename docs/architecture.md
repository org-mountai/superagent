# Architektur

Stand: 20.11.2026. Dieses Dokument beschreibt die Architektur, die die private
Referenzimplementierung belegt hat — generalisiert und ohne Installationsdetails.

## 1. Invarianten

1. **Ein Gesicht.** Nur der First Mate spricht mit der verantwortlichen Person. Worker liefern
   Ergebnisse, Belege und offene Risiken an die Outbox.
2. **Code vor Modell.** Eingang, Zulassung, Routing und Wiederholung sind deterministisch. Ein
   Modell entscheidet nur, wo echte Ambiguität bleibt.
3. **Keine Hintergrundarbeit ohne Ticket.** Jede Aktion hat eine sichtbare Arbeitseinheit mit
   Quelle, Ziel, Risikoklasse und Nachweis.
4. **Genau ein aktiver Besitzer pro Ticket.** Doppelstart ist ein Fehler, kein Wettlauf.
5. **Jede externe Wirkung hat eine Risikoklasse und einen Nachweis.**
6. **Modell ist nicht Berechtigung.** Ein stärkeres Modell bekommt nicht automatisch mehr Rechte.
7. **Neustart gleicht zuerst ab, dann startet er.** Kein blinder Wiederanlauf.
8. **Mergen bleibt menschlich.** Kein Auto-Merge.

## 2. Bausteine

```text
Chat · Code · Mail · pm · Host · Zeitplan
                    │
                    ▼
        [ Eingang: normalisieren, deduplizieren ]
                    │
                    ▼
        [ Admission: Allowlist + Risikoklasse ]
             │                    │
      eindeutig              unklar/verboten
             │                    │
             ▼                    ├──► kleiner Klassifizierer (ohne Werkzeuge)
        [ Ticket + Ereignislog ]  │
             │                    └──► First Mate (entscheidet)
             ▼
        [ Dispatcher: genau ein Besitzer ]
             │
      ┌──────┼──────────┐
      ▼      ▼          ▼
   Repo   Doku/Host   (später weitere)
      │      │
      └──────┴──► [ Outbox: PR, Bericht, Meldung ]
                        │
                        ▼
                 menschliches Review / Merge

        Watchdog ──► liest Tickets, meldet Blocker, startet nichts neu
```

| Baustein | Aufgabe | Deterministisch? |
|---|---|---|
| Eingang | Quellen normalisieren, nach externer Ereignis-ID deduplizieren | ja |
| Admission | Allowlist, Risikoklasse, Ambiguität entscheiden | ja; Modell nur im Slow Path |
| Ticketstore | Ticket, Ereignisse und Cursor transaktional führen | ja |
| Dispatcher | Start, Stop, Retry, genau ein Besitzer, Budgets | ja |
| Runtime-Adapter | Laufzeit hinter einem festen Vertrag kapseln | ja |
| Worker | eine begrenzte Aufgabe in isolierter Umgebung | Modell |
| Outbox | PR, Bericht, Kommentar oder Meldung genau einmal zustellen | ja |
| Watchdog | Zustand abgleichen, Blocker und Hänger melden | ja |
| Kanal-Adapter | Quelle in ein gemeinsames Ereignis übersetzen | ja |
| Statusprojektion | read-only Ansicht auf `status --json` | ja |

## 3. Ticketfluss

1. Eine Quelle erzeugt ein Ereignis: Chat, Issue/PR, Mail, Projektmanagement-Fälligkeit,
   Host-Befund oder Zeitplan.
2. Der Eingang normalisiert Quelle, externe ID, Absender, Thread, Inhalt und Anhänge und
   dedupliziert gegen den bestehenden Bestand.
3. Die Admission prüft die Allowlist und ordnet eine Risikoklasse zu. Eindeutig zulässige
   Arbeit wird sofort ein Ticket. Verbotene oder unklare Arbeit geht an den First Mate; nur bei
   echter Ambiguität darf ein kleiner, werkzeugloser Klassifizierer eine Empfehlung geben. Er
   darf keine R2-/R3-Aktion freigeben.
4. Das Ticket hält Quelle, Ziel, Risikoklasse, Modell-/Werkzeugbudget, Zustand, Besitzer,
   Ergebnis und Nachweis.
5. Der Dispatcher startet genau einen passenden Worker, wenn das Ticket an der Reihe ist.
6. Der Worker arbeitet in einem eigenen Worktree oder einer eigenen Umgebung und liefert Diff,
   PR, Bericht oder Ausführungsnachweis.
7. Die Outbox stellt das Ergebnis genau einmal zu: PR-Kommentar, Bericht oder Meldung an den
   First-Mate-Thread.
8. Review und Merge bleiben menschlich. Das Ticket wird erst danach `done`.
9. Der Watchdog gleicht laufende Tickets ab, erkennt Hänger, doppelte Besitzer und erreichte
   Retry-Grenzen und meldet nur Änderungen.

### Zustände

```text
candidate → admitted → queued → running → review → done
      │          │         │         │
      └──────────┴─────────┴─────────┴──► blocked / failed / cancelled
```

Jeder Übergang ist ein Ereignis mit Zeitpunkt und Ursache. Nach einem Neustart wird zuerst der
Zustand vereinheitlicht; erst danach darf ein Worker starten.

## 4. Risikoklassen

| Klasse | Bedeutung | Standardbehandlung |
|---|---|---|
| **R0** | lesen, prüfen, Signale erfassen | automatisch; erzeugt Ticket und Meldung |
| **R1** | reversible Repo-/Doku-Änderung | automatisch bis PR/Diff; Merge menschlich |
| **R2** | ausdrücklich erlaubte, reversible Host-Aktion | automatisch mit Audit und engem Allowlist-Eintrag |
| **R3** | externe Kommunikation, Reboot, Firewall/SSH, Löschen, Ausfallzeit, Merge | immer ausdrückliche Freigabe der verantwortlichen Person |

Die Risikoklasse gehört zur Aktion und zum Ziel, nicht zum Modell. Ein starkes Modell erbt keine
Rechte. R2 ist kein allgemeines root: erlaubt sind einzelne, benannte Befehle ohne freie
Argumente und ohne Rechteausweitung.

## 5. Verträge

### Kanal-Adapter

```text
normalize(source_event) → event {
  source, external_id, sender, thread, content?, attachments?, capabilities
}
```

Der Adapter ist idempotent, setzt beim ersten Lauf eine Baseline und liefert nur die Felder,
die die Datenpolitik erlaubt. Für Mail werden Signale und Metadaten erfasst; Körper werden
nicht automatisch gelesen oder beantwortet. Die Signalerfassung ist belegt: für die ID-basierte
Mailquelle alle neuen IDs aufsteigend, für IMAP alle neuen ungelesenen Nachrichten vor dem
letzten Cursor; fehlt der Cursor, ist genau eine Nachricht der sichere Fallback. R0-Signale aus
Mail, Projektmanagement und Zeitplan werden zu Tickets, aber nicht automatisch an Repo-Worker
übergeben: zuerst ordnet der First Mate ein, der Dispatcher verweigert sie ausdrücklich.

### Runtime-Adapter

```text
start(ticket) · status(id) · send(id, message) · stop(id) · result(id) · logs(id) · capabilities()
```

Tickets, Policy und Kanäle enthalten keine runtime-spezifischen Befehle oder IDs. Ein Fake-Adapter
muss dieselben Konformitätsproben bestehen wie der echte; ein Harness-Wechsel ist ein Adapter
plus Tests, kein Neubau.

### Host-Broker

Ein Allowlist-Eintrag verbindet genau einen benannten Befund mit genau einem Befehl. Es gibt
keine freien Argumente, kein `sudo` und kein allgemeines Abarbeiten fremder Host-Tickets. Der
Broker protokolliert Entscheidung, Ausführung und Exit-Code, dedupliziert nach Befundversion
und wiederholt einen Fehler nicht still. Erfolg schließt das Ticket nicht sofort; erst ein
späterer Check ohne Befund schließt es. Host-Tickets laufen nur über den Broker, nie über einen
Repo-Worker.

### Statusprojektion

`status --json` ist das gemeinsame, maschinenlesbare Lesemodell. Chat, UI und Dashboard sind
austauschbare Projektionen darauf. Eine Projektion ist read-only und nie der Ticketbestand.

## 6. Haltbarkeit und Wahrheit

- **Ticketstore:** SQLite; Ticket, Ereignis und Cursor werden in einer Transaktion geschrieben.
  Der Bestand enthält manuelle Tickets, Freigaben und Verlauf und ist nicht vollständig aus dem
  Code-Host rekonstruierbar. Er ist nicht beliebig verzichtbar.
- **Policy:** versionierte Konfiguration im Repo; sie ist die Wahrheit, kein Prompt.
- **Code:** Repositories und Pull Requests bleiben die Wahrheit für Code.
- **Langzeitentscheidungen:** versionierte Dokumentation, nicht der Ticketstore.
- **Backup und Schema:** belegt. SQLite-Online-Backup mit Integritätsprüfung, read-only
  Restore-Sicht und Retention 7; `PRAGMA user_version=1`, neuere unbekannte Versionen werden
  abgelehnt. Zwei echte Backups mit realem Bestand waren `ok`, ein täglicher Timer läuft und
  ist aktiviert. Eine produktive Wiederherstellung auf einem separaten Testhost bleibt offen.

## 7. Bewusste Nichtziele

Kein zweiter Companion, kein Auto-Merge, kein freier root-Prompt für Worker, keine Big-bang-
Migration, keine neue Orchestrierungs-Cloud, keine generelle Überwachung von Entwickler-Laptops
und keine Skalierung auf mehrere Hosts, bevor ein gemessenes Problem sie verlangt.
