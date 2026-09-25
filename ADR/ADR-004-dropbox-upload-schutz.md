# ADR-004: Upload-Schutz für Dropbox-Sync statt Merge-vor-Upload (Zwischenschritt)

**Datum:** 2026-09-25
**Status:** ersetzt durch ADR-005 (2026-09-25 – Merge vor jedem Upload, das Flag `vt_dbx_pulled` entfällt)
**Projekt:** Vokabeltrainer

## Problem

Der Dropbox-Upload (`_uploadPayload()`) schreibt immer `mode: 'overwrite'` – ein Gerät ersetzt damit die komplette Cloud-Datei durch seinen lokalen Stand. Der Download (`syncFromDropbox()`) arbeitet umgekehrt rein additiv.

Daraus folgt eine scharfe Asymmetrie: Wer zuletzt hochlädt, gewinnt vollständig. Besonders gefährlich ist der Erstkontakt eines frisch installierten Geräts – dessen `localStorage` ist leer, und die vier Auto-Sync-Auslöser (Vokabeln per Foto oder manuell speichern, Wort aus dem Nachschlagen übernehmen, Klasse einer Unité ändern) feuern ohne Rückfrage. Eine einzige neue Vokabel auf einem neuen Gerät löscht so den gesamten Cloud-Stand.

Das ist am 2026-09-24 real eingetreten: `Apps/Vokabeltrainer-Jule-Franze/vokabeln.json` enthielt danach nur noch eine Unité („Unité ?", 43 Wörter), ohne `stars`, `errors` und `uniteClasses` (Todo #295).

## Entscheidung

Zweistufig absichern, ohne das Sync-Modell selbst umzubauen:

1. Freigabe-Flag `vt_dbx_pulled` – der Auto-Upload läuft erst, wenn das Gerät einmal erfolgreich von Dropbox geladen hat.
2. Automatisches `syncFromDropbox(true)` direkt nach erfolgreichem OAuth-Login, damit ein neues Gerät ohne Zutun auf den Cloud-Stand kommt und das Flag gesetzt wird.
3. Manueller Upload ohne gesetztes Flag nur nach expliziter `confirm()`-Rückfrage mit Nennung der betroffenen Vokabelzahl.

Der eigentliche Fix – vor jedem Upload laden, mergen, hochladen – bleibt offen (Todo #295).

## Begründung

- Der gefährlichste Fall ist der unbeaufsichtigte: ein Kind richtet ein neues Gerät ein und verliert durch eine beiläufige Eingabe alles. Genau dieser Pfad wird geschlossen, und zwar ohne dass jemand eine Bedienregel kennen muss.
- Der Eingriff ist klein und lokal (ein Flag, drei Prüfstellen), ändert das Dateiformat nicht und ist mit älteren App-Versionen verträglich.
- Das automatische Laden nach dem Verbinden ist risikolos, weil der Download nie löscht.
- Ein sofortiger Umbau auf Merge-vor-Upload wirft eigene Fragen auf (Löschungen, Umbenennungen, Konfliktauflösung), die eine eigene Runde verdienen – der Schutz sollte nicht darauf warten.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Merge-vor-Upload sofort umsetzen | Der richtige Zielzustand, aber größerer Eingriff mit offenen Teilfragen (Löschen/Umbenennen werden bis heute nicht synchronisiert). Als Todo #295 eingeplant, statt den Schutz zu verzögern |
| Auto-Sync generell abschalten, nur manueller Upload | Nimmt den Komfort, für den der Auto-Sync gebaut wurde, und verlagert die Gefahr nur auf den Knopf |
| Upload blockieren, wenn lokal weniger Vokabeln als in der Cloud | Braucht ohnehin einen Download vor dem Upload – dann kann direkt gemergt werden. Zudem heuristisch: weniger Vokabeln können legitim sein (Löschen) |
| `mode: 'add'`/`update` mit Dropbox-Revision statt `overwrite` | Löst nur den Schreibkonflikt, nicht den Datenverlust – die App müsste den Konflikt trotzdem inhaltlich auflösen |
| Warnung nur im Hilfetext („erst Laden") | War faktisch schon so dokumentiert und hat den Verlust nicht verhindert |

## Gilt unter

- Der Upload bleibt `overwrite` und die Datei bleibt die einzige Cloud-Quelle.
- Es sind wenige, einander vertrauende Geräte im Spiel (Jules iPhone/iPad, ein Dropbox-App-Ordner).
- Wird Merge-vor-Upload umgesetzt (Todo #295), verliert die Flag-Logik ihre Schutzfunktion und kann entfallen – diese ADR ist dann durch die neue zu ersetzen.

## Konsequenzen

**Positiv**
- Ein neu installiertes Gerät kann den Cloud-Stand nicht mehr beiläufig leeren.
- Neue Geräte haben nach dem Verbinden sofort alle Vokabeln, ohne dass jemand `☁️↓` kennen muss.
- Ein versehentlicher manueller Upload vom leeren Gerät verlangt eine bewusste Bestätigung.

**Negativ**
- Der Restfehler bleibt: ein Gerät mit veraltetem, aber nicht leerem Stand überschreibt beim nächsten Auto-Sync weiterhin neuere Ergänzungen anderer Geräte.
- Ein zusätzlicher localStorage-Key, der bei einem Storage-Reset (Schul-iPad/MDM) verloren geht – dann greift der Schutz erneut, was gewollt, aber beim ersten Speichern als Toast sichtbar ist.
- Geräte mit App-Version < v56 verhalten sich unverändert; der Schutz wirkt erst, wenn alle Geräte aktualisiert sind.
