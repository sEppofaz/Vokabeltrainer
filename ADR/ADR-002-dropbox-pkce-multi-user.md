# ADR-002: Dropbox PKCE OAuth2 für Multi-User-Sync (statt zentralem Backend)

**Datum:** 2026-06-23
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem
Mehrere Nutzer (z.B. Lea und Josef) sollen die App teilen, aber jeweils eigene Vokabeldaten haben – geräteübergreifend synchronisiert. Ohne Backend.

## Entscheidung
Jeder Nutzer bringt seinen eigenen Dropbox Developer-Account und App-Key mit. OAuth2 PKCE-Flow direkt im Browser – kein Client Secret nötig. Vokabeln werden als `vokabeln.json` im jeweiligen App-Folder gespeichert.

## Begründung
- Kein zentrales Backend nötig: jeder Nutzer hat seine eigene Dropbox-Instanz
- PKCE funktioniert ohne Server (kein Client Secret im Code oder auf einem Server)
- Dropbox API ist von GitHub Pages aus direkt aufrufbar (CORS-freigegeben)
- Datenschutz by Design: Vokabeldaten liegen im Dropbox-Account des jeweiligen Nutzers, nicht auf einem gemeinsamen Server
- Refresh Token wird automatisch erneuert (ab v32: kein manueller Token-Ablauf mehr)

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Zentrales Backend mit User-Auth | Hosting-Kosten, Datenschutzverantwortung, erheblicher Entwicklungsaufwand |
| Firebase / Supabase | Abhängigkeit von weiterem Service; Kosten bei Skalierung; Komplexität |
| localStorage ohne Sync | Kein Geräte-Wechsel möglich; Datenverlust bei Browser-Reset |
| Geteilter Dropbox-Account | Datenvermischung; ein Nutzer kann Daten des anderen überschreiben |
| iCloud / Google Drive | Keine direkte Browser-API; komplexerer OAuth-Flow |

## Gilt unter
- Nutzer sind bereit, einen eigenen Dropbox Developer-Account anzulegen (einmalig, kostenlos)
- Anthropic API Key wird ebenfalls pro Nutzer selbst mitgebracht
- App bleibt für kleine Nutzergruppe (<10 Personen)

## Konsequenzen
+ Zero-Cost: kein Backend, kein Hosting
+ Datenschutz: jeder Nutzer kontrolliert seine eigenen Daten
+ Keine gemeinsame User-Datenbank zu verwalten
- Einrichtungsaufwand für neue Nutzer (Dropbox App anlegen, App Key eintragen)
- Kein gemeinsames Vokabular möglich (jeder hat sein eigenes JSON)
- Bei Dropbox-API-Änderungen muss OAuth-Flow angepasst werden
