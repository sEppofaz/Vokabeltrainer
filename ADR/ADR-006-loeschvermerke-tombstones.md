# ADR-006: Löschvermerke (Tombstones) für den Dropbox-Sync

**Datum:** 2026-09-25
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem

Nach ADR-005 mergt jeder Upload den Cloud-Stand, bevor er schreibt. Damit gehen Vokabeln nicht mehr verloren – aber Löschen wurde dadurch systematisch wirkungslos:

- `delItem()`, `delUnite()`, `renameUnite()`, `moveVocab()` und `saveItem()` arbeiteten rein lokal und lösten gar keinen Sync aus.
- Der Merge holt eine lokal gelöschte Vokabel aus der Cloud zurück, weil „fehlt lokal" nicht von „wurde nie geladen" zu unterscheiden ist.

Vor v57 nahm ein Upload die Löschung wenigstens zufällig mit (er überschrieb alles) – um den Preis, dass er dabei auch fremde Ergänzungen verwarf. Beides zusammen ist unbrauchbar: Entweder verschwinden fremde Vokabeln oder Löschen funktioniert nicht. Josefs Einwand dazu war knapp und richtig: „was bringt es sonst".

## Entscheidung

Die Datei führt zusätzlich zu den Vokabeln eine Liste der bewusst entfernten Einträge:

```json
"deleted": {
  "Unité 4||der hund|le chien": 1758800000000,
  "Unité 4||*": 1758800000000
}
```

- Schlüssel: `<Unité>||<vocabKey>` für eine Vokabel, `<Unité>||*` für eine ganze Unité. Wert: Zeitpunkt des Löschens.
- Vokabeln bekommen beim Anlegen ein Feld `ts`. **Ein Vermerk gewinnt nur gegen Vokabeln, die älter sind als er** – eine danach neu angelegte (`v.ts > Vermerk`) überlebt. Bestandsvokabeln ohne `ts` gelten als alt, ein Löschen wirkt also auch auf sie.
- `mergeRemoteData()` führt die Vermerke beider Seiten zusammen (jüngster gewinnt), übernimmt fremde Vokabeln nur ohne gültigen Vermerk und entfernt lokale Vokabeln, für die ein fremder Vermerk vorliegt. Rückgabe `{added, removed}`.
- Die fünf verändernden Aktionen hinterlassen einen Vermerk und lösen Auto-Sync aus. Umbenennen = Vermerk für den alten Unité-Namen; Bearbeiten = Vermerk für die alte Fassung; Verschieben = Vermerk in der Quell-Unité plus frischer Zeitstempel im Ziel.
- Vermerke verfallen nach 90 Tagen (`TOMB_TTL`), damit die Datei nicht endlos wächst.
- Der Import nutzt dieselbe Merge-Funktion statt einer eigenen Kopie.

Sterne und Fehlerzähler bleiben bewusst additiv: Ein abgewählter Stern wird nicht synchronisiert (Josefs Entscheidung, Aufwand/Nutzen).

## Begründung

- „Fehlt lokal" und „wurde gelöscht" sind ohne Vermerk nicht unterscheidbar – jede Lösung ohne Tombstones muss raten.
- Der Zeitstempelvergleich löst den einzigen wirklich heiklen Konflikt (löschen auf dem einen, neu anlegen auf dem anderen Gerät) ohne Nutzerdialog, und zwar zugunsten der jüngeren Absicht.
- Umbenennen, Verschieben und Bearbeiten sind aus Sicht der Daten Löschen + Anlegen und brauchen deshalb keine eigene Mechanik.
- Die Verfallsfrist hält die Datei klein, ohne im realen Nutzungsrhythmus (mehrmals pro Woche) je zu greifen.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Löschen gar nicht synchronisieren (Stand v57) | Halb synchronisiert ist kein Zustand: Korrekturen und Aufräumen wirken nie, falsche Vokabeln bleiben ewig |
| Zurück zu `overwrite` ohne Merge | Das war v56 und hat am 2026-09-24 real Vokabeln vernichtet |
| Volles CRDT / OT | Für zwei Geräte und eine 23-KB-Datei weit überdimensioniert |
| Tombstones ohne Zeitstempel (Vermerk gewinnt immer) | Eine nach dem Löschen neu angelegte gleiche Vokabel würde still wieder verschwinden – genau der Fall beim Korrigieren |
| Zeitstempel pro Vokabel statt Vermerke, jüngster gewinnt | Erkennt Änderungen, aber nicht Löschungen – ohne Vermerk bleibt „fehlt" mehrdeutig |
| Vermerke nie verfallen lassen | Datei wächst unbegrenzt; 90 Tage decken den realen Rhythmus um ein Vielfaches ab |
| Sterne/Fehler ebenfalls mit Vermerken | Dieselbe Mechanik nochmals pro Markierung; von Josef bewusst abgelehnt |

## Gilt unter

- **Alle Geräte laufen auf v58 oder neuer.** Ein Gerät auf v57 schreibt `deleted` beim Upload nicht mit und löscht damit alle Vermerke – Gelöschtes käme zurück.
- Es sind wenige, einander vertrauende Geräte im Spiel; niemand löscht böswillig.
- Ein Gerät synchronisiert mindestens alle 90 Tage. Wer länger pausiert, kann Gelöschtes zurückbringen.
- Die Uhren der Geräte gehen halbwegs richtig (iOS/Netzwerkzeit). Eine grob falsch gestellte Uhr kann einen Vermerk oder eine Neuanlage fälschlich gewinnen lassen.

## Konsequenzen

**Positiv**
- Löschen, Umbenennen, Verschieben und Korrigieren wirken auf allen Geräten.
- Diese fünf Aktionen lösen erstmals überhaupt einen Sync aus – sie blieben bisher lokal.
- Die Merge-Logik liegt an einer Stelle und wird von Dropbox-Sync **und** Import genutzt.
- Die Meldung nennt jetzt beide Richtungen („… übernommen, … auf anderem Gerät gelöscht").

**Negativ**
- Datenformat-Erweiterung: `deleted` und `ts`. Ältere App-Versionen zerstören die Vermerke beim Hochladen.
- Die Datei wächst um die Vermerke (~60 Byte je Eintrag, verfallen nach 90 Tagen).
- Der Zeitstempelvergleich hängt an der Gerätezeit.
- Abgewählte Sterne und zurückgesetzte Fehlerzähler werden weiterhin nicht synchronisiert.
