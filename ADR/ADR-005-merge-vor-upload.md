# ADR-005: Merge vor jedem Dropbox-Upload

**Datum:** 2026-09-25
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem

Der Upload schreibt `mode: 'overwrite'`, der Download ergänzt nur. Wer zuletzt hochlädt, gewinnt vollständig – auch mit einem veralteten Stand.

ADR-004 hat mit dem Flag `vt_dbx_pulled` nur den Erstkontakt eines Geräts abgesichert. Der Rest des Problems blieb: Sobald ein Gerät einmal geladen hatte, überschrieb es beim nächsten Auto-Sync weiterhin alles, was andere Geräte seither ergänzt hatten. Bei zwei aktiv genutzten Geräten (iPhone + iPad) ist das kein Randfall, sondern der Normalbetrieb – die Vokabeln des jeweils anderen Geräts verschwanden still, ohne Meldung.

## Entscheidung

Jeder Upload lädt zuerst die Dropbox-Datei, merged sie additiv in den lokalen Stand und schreibt erst dann:

- `_downloadPayload(token)` – Download-Helfer, liefert `{status:'ok', data}` oder `{status:'empty'}` (HTTP 409, Datei existiert noch nicht), wirft bei jedem anderen Fehler.
- `mergeRemoteData(data)` – die bisher in `syncFromDropbox()` eingebettete Merge-Logik als eigene Funktion, Rückgabe: Anzahl neu übernommener Vokabeln.
- `_mergeAndUpload(token)` – Laden → Mergen → Hochladen. **Schlägt das Laden fehl, wird nicht hochgeladen.**
- `_queueDbx(fn)` – alle Dropbox-Zugriffe laufen nacheinander, damit sich zwei schnell aufeinanderfolgende Auto-Syncs nicht überholen.

`autoSyncToDropbox()` und `syncToDropbox()` gehen über `_mergeAndUpload()`, `syncFromDropbox()` über dieselben Helfer. Das Flag `vt_dbx_pulled` aus ADR-004 entfällt und wird beim Start aufgeräumt.

## Begründung

- Kein Gerät kann die Arbeit eines anderen mehr verwerfen, unabhängig von Reihenfolge und Aktualität. Die Bedienregel „vor dem Erfassen erst Laden" entfällt – und Regeln, die ein Kind im Alltag einhalten muss, sind ohnehin die schwächste Absicherung.
- Der Merge-Code existiert nur noch einmal; vorher hätte jede Änderung an der Zusammenführung an zwei Stellen gepflegt werden müssen.
- Der Abbruch bei fehlgeschlagenem Download ist die konservative Richtung: lieber nicht hochladen (lokal ist alles gespeichert) als blind überschreiben. Der Nutzer erfährt das per Toast.
- Die Datei ist klein (~23 KB); der zusätzliche Download pro Upload fällt nicht ins Gewicht.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Beim Flag-Ansatz aus ADR-004 bleiben | Schützt nur den Erstkontakt, nicht den laufenden Zwei-Geräte-Betrieb – der eigentliche Verlustfall blieb offen |
| Bei Download-Fehler trotzdem hochladen | Genau der Fall, in dem der Overwrite blind wird. Lokal ist nichts verloren, also kann der Upload warten |
| Dropbox-Revision (`mode: {'.tag':'update', rev}`) für optimistisches Sperren | Erkennt den Konflikt, löst ihn aber nicht – die App müsste danach ohnehin mergen. Zusätzliche Fehlerpfade ohne Gewinn |
| Zeitstempel je Vokabel, jüngster gewinnt | Braucht ein Datenformat-Upgrade und hilft nur bei Änderungen, nicht bei Ergänzungen – der Merge löst den realen Fall bereits vollständig |
| Tombstones für gelöschte Vokabeln | Würde Löschen endlich synchronisierbar machen, ist aber ein eigener Entwurf mit Datenformat-Änderung. Bewusst nicht mit diesem Fix vermischt |

## Gilt unter

- Additives Zusammenführen ist die gewünschte Semantik: Vokabeln sollen sich sammeln, nicht verschwinden.
- Die Datei bleibt klein genug, dass ein Download vor jedem Upload nicht stört.
- Wird Löschen jemals synchronisierbar (Tombstones), ist diese Entscheidung neu zu bewerten – dann darf „fehlt in der Cloud" nicht mehr automatisch „ergänzen" heißen.

## Konsequenzen

**Positiv**
- Kein Vokabelverlust mehr durch Reihenfolge oder veralteten Gerätestand.
- Das Gerät bekommt beim Speichern automatisch die Vokabeln des anderen Geräts mit; ein Toast nennt die Zahl.
- Merge-Logik nur noch an einer Stelle; `syncFromDropbox()` ist deutlich kürzer.
- ADR-004 wird dadurch abgelöst, ein localStorage-Key weniger.

**Negativ**
- Jeder Upload kostet einen zusätzlichen Download (bei ~23 KB unkritisch).
- Ohne Netz wird gar nicht mehr hochgeladen – der Nutzer muss den Toast beachten und später erneut hochladen. Lokal bleibt alles erhalten.
- Löschen und Umbenennen werden weiterhin nicht synchronisiert; gelöschte Vokabeln kommen beim nächsten Merge zurück. Das ist jetzt systematisch, nicht mehr zufällig – und bleibt offen.
