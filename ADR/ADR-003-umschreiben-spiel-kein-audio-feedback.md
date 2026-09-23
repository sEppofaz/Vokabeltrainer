# ADR-003: Umschreiben-Spiel nutzt reines Text-Feedback statt Sprachausgabe

**Datum:** 2026-09-23
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem

Für das neue Sprech-Spiel "Umschreiben" (Grammatik-Tab-Ersatz) sollte der Nutzer ein Wort per Mikrofon umschreiben, Claude die Umschreibung bewerten. Ursprünglich sollte das Feedback wahlweise gesprochen (`speechSynthesis`) oder geschrieben ausgegeben werden. Bei der Recherche stellte sich heraus: WebKit (Safari/iOS) hat einen bekannten Audiosession-Deadlock – wenn nach einer Audiowiedergabe (egal ob `<audio>` oder `speechSynthesis`) das Mikrofon erneut geöffnet wird, gibt WebKit die Audiosession nicht zuverlässig frei, `SpeechRecognition.start()` hängt dann ohne jedes Error-Event.

## Entscheidung

Gesprochenes Feedback wurde komplett gestrichen. Claudes Bewertung wird ausschließlich als Text angezeigt (`.speak-feedback`-Box). Zusätzlich wurde der ursprünglich geplante Plattform-Block per `navigator.standalone`-Erkennung (Diktieren pauschal deaktivieren auf installierten iOS-Home-Bildschirm-Apps) verworfen und durch ein generisches Timeout-Sicherheitsnetz ersetzt (8s ohne `onresult`/`onerror`/`onend` → automatischer Reset + Fehlermeldung mit Retry).

## Begründung

- Ohne jede Audiowiedergabe im Spiel-Flow tritt das WebKit-Deadlock-Szenario gar nicht erst auf – auf iPhone und iPad gleichermaßen, unabhängig davon ob installierte App oder Browser-Tab.
- Die Quellenlage zu "Diktieren funktioniert nicht in installierten Home-Bildschirm-Apps" erwies sich bei genauerem Hinsehen als unpräzise/widersprüchlich (SEO-Blog-Aggregate ohne belastbare Erstquelle) – ein pauschaler `navigator.standalone`-Block hätte das Feature möglicherweise dort deaktiviert, wo es tatsächlich funktioniert.
- Schriftliches Feedback ist zudem robuster (funktioniert ohne gute Französisch-TTS-Stimme) und im Klassenzimmer unauffälliger als automatisch abgespieltes Audio.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Feedback-Modus-Toggle (Text/Audio wählbar, Default Text) | Hätte das WebKit-Deadlock-Risiko im Audio-Modus weiter bestehen lassen; zusätzlicher State/UI-Aufwand für einen Modus mit bekanntem Bug-Risiko |
| Plattform-Block via `navigator.standalone` | Unsichere Erkennungsgrundlage; hätte das Feature evtl. dort gesperrt, wo es funktioniert |
| Text-Eingabe-Fallback für Diktieren | Von Josef bewusst nicht gewünscht – Diktieren ist der Kernmechanismus des Spiels, ein Text-Fallback würde das Sprechen entwerten |

## Gilt unter

Gilt solange das Umschreiben-Spiel keine Sprachausgabe verwendet. Falls künftig doch gesprochenes Feedback gewünscht wird, muss vorher geprüft werden, ob Apple den WebKit-Audiosession-Bug inzwischen behoben hat (Stand 2026-09: noch nicht bestätigt behoben).

## Konsequenzen

- Positiv: Spiel funktioniert zuverlässig auf allen getesteten Plattformen (iPhone/iPad, installiert oder Browser-Tab), keine bekannten Blocker.
- Negativ: Kein Hörverständnis-Training über das Feedback selbst (nur über die angezeigte Karte, die ohnehin nicht vorgelesen wird). Kann bei Bedarf später als eigenständiges, sauber gegen den WebKit-Bug abgesichertes Feature nachgerüstet werden (Timeout+Retry-Muster ist bereits vorhanden).
