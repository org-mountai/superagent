# Gotchas

Fehler, die die Referenzimplementierung bereits bezahlt hat — generalisiert. Jeder Punkt ist
eine Betriebsregel, keine Theorie.

## Identität und Routing

- **Thread per Label auflösen, nicht per ID.** Bei jedem Versand frisch auflösen; genau ein
  aktiver Treffer. Null oder mehrere Treffer stoppen sichtbar. Eine alte ID ist kein Fallback.
- **Kompaktierung ist kein Themenwechsel.** Ein verdichteter Kontext braucht keinen neuen
  Thread; das Label bleibt stabil.
- **Ein Thread, ein Zweck.** Ein technischer Ersatz-Thread bekommt das Label ausdrücklich
  übertragen, der alte verliert es. Nicht beide gleichzeitig aktiv lassen.

## Idempotenz

- **Der erste Lauf setzt eine Baseline.** Sonst wird alte Historie nachgeliefert.
- **Cursor erst nach erfolgreicher Zustellung fortsetzen.** Ein Fehler bleibt sichtbar und wird
  beim nächsten Lauf erneut versucht.
- **Genau ein aktiver Besitzer pro Ticket.** Zwei Dispatcher oder ein manueller Start erzeugen
  sonst doppelte Arbeit.
- **Retry setzt ein Limit zurück, mehr nicht.** Es startet keinen Worker und mergt nichts.
- **Befundversionen sind der Schlüssel.** `created`/`updated`/`reopened` darf pro Allowlist-
  Eintrag höchstens einmal ausführen; ein unveränderter Befund darf nicht erneut laufen.
- **Bewusst verworfene Tickets bleiben verworfen.** Sie dürfen beim nächsten Sync nicht
  wiederauferstehen.

## Policy und Modell

- **Die Allowlist ist Code.** Ein Prompt ist kein Berechtigungssystem. Ein Modell darf keine
  R3-Aktion „genehmigen“.
- **Risikoklasse gehört zu Aktion und Ziel.** Ein stärkeres Modell erbt keine Rechte.
- **Host-Zugriff ist ein benannter Befehl.** Kein root-Shell im Modellkontext, keine freien
  Argumente, kein `sudo`.
- **Der Slow Path empfiehlt, er entscheidet nicht.** Ein kleiner Klassifizierer ohne Werkzeuge
  darf Ambiguität auflösen helfen, aber keine privilegierte Aktion freigeben.

## Runtime und Umgebung

- **Timer erben nicht die interaktive `PATH`.** Absolute Pfade verwenden oder `PATH` explizit
  setzen; sonst scheitert ein Launcher, der `env node` oder ähnliches erwartet.
- **Runtime-spezifische Befehle gehören in den Adapter.** Stehen sie in Tickets, Policy oder
  Kanälen, wird jeder Harness-Wechsel zum Umbau.
- **Erst mit Fake-Runtime testen, dann mit echtem Provider.** Ein Providerfehler ist kein
  stiller Erfolg; er gehört in den Ticket-Verlauf.
- **Berechtigungsdialoge sind Ereignisse.** Sie werden protokolliert und ausdrücklich gelöst,
  nicht als Erfolg verbucht.
- **Nach einem Neustart erst abgleichen.** Sonst startet ein Ticket ein zweites Mal, weil der
  alte Prozess nicht mehr sichtbar ist.

## Daten und Bestand

- **Die SQLite-Datenbank ist nicht rekonstruierbar.** Manuelle Tickets, Freigaben, Besitzer und
  Verlauf existieren nur dort. Deshalb gilt: SQLite-Online-Backup mit Integritätsprüfung,
  read-only Restore-Sicht und Retention 7; `PRAGMA user_version=1`, neuere unbekannte
  Versionen werden abgelehnt. Eine produktive Wiederherstellung auf einem separaten Testhost
  steht noch aus.
- **Ticket, Ereignis und Cursor in einer Transaktion.** Sonst verliert oder dupliziert ein
  Absturz ein Signal.
- **Signalerfassung ist nicht Körperverarbeitung.** Die Per-Mail-Signalerfassung ist belegt:
  für die ID-basierte Mailquelle alle neuen IDs aufsteigend, für IMAP alle neuen ungelesenen
  Nachrichten vor dem letzten Cursor; ohne Cursor ist genau eine Nachricht der sichere
  Fallback. Körper werden weiterhin nicht automatisch gelesen oder beantwortet. Diese Grenze
  nicht als offene Signal-Erfassung missverstehen — und nicht als Freigabe zum Antworten.
- **Keine Körper, Tokens oder Kundendaten in Logs, Tickets oder Statusansichten.** Die
  Projektion bleibt bis auf den benannten R0-Drop lesend.

## Meldungen

- **Nur Änderungen und Blocker melden.** Zustands-Spam erzieht dazu, den Kanal zu ignorieren.
- **Eine fehlgeschlagene Benachrichtigung darf die darunterliegende Aktion nicht verstecken.**
  Der Nachweis bleibt im Ticket; die Zustellung wird separat wiederholt.
- **Eine Projektion darf nicht heimlich Schreibaktionen bekommen.** Die einzige Ausnahme ist
  der ausdrücklich benannte R0-Drop für offene Kanal-Tickets; kein Start, kein Retry, keine
  Freigabe und keine R2-/R3-Aktion aus der Ansicht.
- **Der zweite Klick ist UX, nicht Autorisierung.** Der Server prüft Quelle und Status erneut,
  bevor er den bestehenden Drop-Befehl aufruft; ein GitHub- oder Host-Ticket wird unverändert
  abgelehnt und protokolliert.
