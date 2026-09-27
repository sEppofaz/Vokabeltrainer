# ADR-007: Umschaltbare Oberflächensprache Deutsch / Français

**Datum:** 2026-09-27
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem

Die App war durchgehend deutschsprachig, obwohl sie zum Französischlernen dient. Jule soll die
Oberfläche auf Französisch bedienen können – gelebte Immersion statt nur Vokabelpaare. Gefordert
war ein Umschalter im Info-Bereich direkt unter dem Dunkelmodus-Schalter, und die Wahl muss bis
zum nächsten aktiven Umstellen erhalten bleiben (keine Automatik, kein Zurückfallen).

Technische Randbedingung: `index.html` ist eine einzige Datei mit ~3.400 Zeilen, rund 200
oberflächenwirksamen Texten, verteilt auf statisches HTML und JS-Template-Strings. Die
Einrichtungs-Sektionen (API-Key, Dropbox) liegen mitten im Hilfe-Sheet und enthalten
Eingabefelder mit festen IDs.

## Entscheidung

Ein `I18N`-Objekt mit je einem Block pro Sprache (`de`, `fr`, 212 Schlüssel), Zugriff über
`tr(key)` mit Fallback auf `de` und `trf(key, {…})` für Texte mit Platzhaltern. Statische Texte
werden über `data-i18n` / `data-i18n-ph` / `data-i18n-title` / `data-i18n-aria` ausgezeichnet und
von `applyLangStatic()` gesetzt; `applyLang()` zieht anschließend die dynamischen Renderer nach.
Persistenz in `localStorage` unter `vt_lang`, gerätelokal wie das Theme.

**Startsprache ohne gespeicherte Wahl ist Französisch** (Josef-Vorgabe am Umsetzungstag, nachdem
die Mechanik stand).

Die Umsetzung erfolgt zweistufig: v59 = komplettes Bedien-UI, v60 = Hilfe-Sheet-Körper
(~1.200 Wörter, überwiegend technische Einrichtung).

## Begründung

- Das Muster (ein Objekt je Sprache + `t()`-Zugriff mit Fallback) ist im eigenen Bestand erprobt:
  QG TS Defaulter Report, dreisprachig. Kein neues Konzept nötig.
- `data-i18n` statt Einzelzuweisung per ID: der Defaulter Report setzt ~40 Labels einzeln
  (`document.getElementById(...).textContent = t(...)`). Bei ~200 Texten wäre diese Liste selbst
  die Fehlerquelle – ein neuer Text, der dort vergessen wird, bleibt stumm in der alten Sprache.
  Mit Attribut-Durchlauf kann das nicht passieren, und die Vollständigkeit ist statisch prüfbar.
- Die Funktion heißt `tr()` und nicht `t()`, weil `t` im Bestand mehrfach als lokaler Parameter
  belegt ist (`setOcrType(t)`, `setUploadTab(t)`, `showToast`, `showMain`). Ein globales `t()`
  wäre dort unbemerkt verdeckt worden.
- `trf()` mit `{platzhalter}` statt Stringkonkatenation, weil sich die Wortstellung unterscheidet:
  „12 Vokabeln in „Unité 3" gespeichert" vs. « 12 mots enregistrés dans « Unité 3 » ».

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Zwei parallele HTML-Bäume (deutsch/französisch), per `display` umgeschaltet | Die Einrichtungs-Sektionen im Hilfe-Sheet enthalten Eingabefelder mit festen IDs (`apikey-input`, `dbx-appkey-input`, …). Duplizieren erzeugt doppelte IDs und bricht jeden `getElementById`-Zugriff. |
| Einzelzuweisung je Element per ID (Muster Defaulter Report) | Bei ~200 Texten unpflegbar; jeder neue Text muss zusätzlich in der Zuweisungsliste landen, sonst bleibt er stumm. |
| Browsersprache automatisch erkennen (`navigator.language`) | Josef-Vorgabe: die Einstellung bleibt bis zum nächsten aktiven Umstellen. Eine Automatik würde die Wahl auf einem Gerät mit deutschem System immer wieder überstimmen. |
| Sprachwahl über Dropbox synchronisieren | Wie das Theme eine Geräteeinstellung. Eine Synchronisierung würde auf dem Schul-iPad die Wahl vom iPhone erzwingen und wäre zusätzlich ein Feld mehr im Sync-Format. |
| Übersetzung der KI-Prompts | Die Prompts sind interne Anweisungen. Das Umschreiben-Feedback richtet sich bewusst nach der *gesprochenen* Sprache (`langName`), Nachschlagen liefert die deutsche Übersetzung – das ist die Funktion der Features, nicht Oberflächentext. |

## Gilt unter

- Zwei Sprachen. Bei einer dritten wäre der Schalter (`role="switch"`, zwei Zustände) auf einen
  Zyklus-Button umzustellen, wie im Defaulter Report (DE/EN/ES).
- Die App bleibt eine Einzeldatei. Bei einem Build-Schritt wären ausgelagerte Sprachdateien
  sinnvoller.
- `initStaticIcons()` ersetzt `data-icon`-Spans weiterhin einmalig durch SVG – daraus folgt die
  Regel, dass `data-i18n` nur auf textführenden Blattelementen stehen darf.

## Konsequenzen

**Positiv**
- Jule bedient die App auf Französisch, ohne dass ein zweiter Codezweig entsteht.
- Vollständigkeit ist statisch prüfbar (Schlüssel in beiden Blöcken, gleiche Platzhalter, kein
  `data-i18n` über einem Icon).
- Zahlenformate folgen der Sprache (`trNum()`), ohne dass Aufrufstellen es einzeln wissen müssen.

**Negativ**
- Jede künftige Textänderung ist eine Doppelpflege – beide Sprachblöcke.
- Bis v60 ist der Hilfe-Sheet-Körper deutsch, während die Oberfläche standardmäßig französisch
  ist. Das ist bewusst in Kauf genommen, aber sichtbar.
- `applyLang()` muss bei jedem neuen dynamischen Renderer mitgepflegt werden, sonst bleibt dessen
  Inhalt bis zum nächsten Tab-Wechsel in der alten Sprache.
