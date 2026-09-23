# ADR-001: GitHub Pages als Hosting-Plattform

**Datum:** 2026-06-23
**Status:** aktiv
**Projekt:** Vokabeltrainer

## Problem
Der Vokabeltrainer muss öffentlich erreichbar sein (für mehrere Nutzer) und als PWA installierbar sein (HTTPS erforderlich). Hosting-Lösung muss wartungsarm sein.

## Entscheidung
GitHub Pages: Statisches Hosting direkt aus dem `main`-Branch des Repos `sEppofaz/Vokabeltrainer`. Deployment via `git push`.

## Begründung
- Kostenlos, kein monatlicher Server-Aufwand
- HTTPS out-of-the-box (PWA-Voraussetzung erfüllt)
- Deployment = `git push`: kein SSH, kein rsync, kein Restart
- Keine Backend-Logik nötig (API-Calls gehen direkt vom Browser an Anthropic/Dropbox)
- GitHub-Repo ist ohnehin vorhanden

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Hetzner Server (eigenes nginx) | Kosten (bereits ausgelastet), Wartungsaufwand, kein Mehrwert für rein statische App |
| Netlify / Vercel | Unnötige Komplexität, weiterer Account, kein Vorteil gegenüber GitHub Pages für diesen Anwendungsfall |
| Dropbox direkt (file://) | Blockiert CORS; kein HTTPS; PWA-Installation nicht möglich |
| Lokaler Server | Nicht für andere Nutzer erreichbar |

## Gilt unter
- App bleibt rein statisch (kein Backend, kein SSR)
- GitHub-Account bleibt aktiv
- API-Calls (Anthropic, Dropbox) laufen direkt vom Browser (CORS-freigegeben)

## Konsequenzen
+ Zero-Cost-Hosting
+ Deployment in einem Befehl
+ Automatische HTTPS-Zertifikatsverwaltung
- GitHub Pages hat Limits (100 GB/Monat Bandwidth – für diese App irrelevant)
- Keine serverseitige Logik möglich (Backend-Features würden anderen Ansatz erfordern)
- Deployment-Lag: GitHub Pages braucht ~1 Minute nach push bis live
