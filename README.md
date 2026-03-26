# Systemdokumentation — SCUM Server Management Suite - Die Alten Säcke Community

Vollständige technische Dokumentation eines über 6 Jahre organisch gewachsenen Server-Management-Systems für einen SCUM-Gameserver. Entwickelt und erdacht von FMJ. Administrator der Alten Säcke Community

## Was ist das?

Ein Komplex aus **50+ Discord-Bot-Modulen**, **8 Standalone-Bots**, **10 AutoHotkey-Skripten**, **3 externen Toolsuiten**, einem **Web-Dashboard** und einem **KI-gestützten Admin-Assistenten** — alles entkoppelt, alles dateibasiert oder Datenbank-kommunizierend, alles eigenständig lauffähig.

## Dokumentstruktur

| # | Dokument | Beschreibung |
|---|---|---|
| 00 | [Architekturprinzip](00_ARCHITEKTURPRINZIP.md) | Entkopplungsphilosophie + Inhaltsverzeichnis aller Dokumente |
| 01 | [Systemkarte](01_SYSTEMKARTE.md) | Gesamtstruktur, Startkette, Datenflüsse |
| 02 | [Wirtschaft & Handel](02_WIRTSCHAFT.md) | Bank, Handelshaus, Lizenzen, Versicherung, Spenden |
| 03 | [Tracking & Monitoring](03_TRACKING_MONITORING.md) | Statbot, Kill-Feed, Serverstatus, Voice-Rewards |
| 04 | [Spieler & Identität](04_SPIELER_IDENTITAET.md) | Steam-Discord-Verlinkung, Profile, Charakter-Stats |
| 05 | [Events & Gameplay](05_EVENTS_GAMEPLAY.md) | Purge, Bunker, Fishing, Titles, NPC-Zonen, Aircraft |
| 06 | [Logs & Kommunikation](06_LOGS_KOMMUNIKATION.md) | Chat-Bridge, Log-Relay, Router-UI, Admin-Tools |
| 07 | [AHK-Automatisierung](07_AHK_AUTOMATISIERUNG.md) | Alle AHK-Skripte, Startkette, Tastatur-Simulation |
| 08 | [Support-Skripte](08_SUPPORT_SKRIPTE.md) | Timer, FTP-Sync, MapTools, Payment, Mapevent |
| 09 | [Standalone-Bots](09_STANDALONE_BOTS.md) | Game-Manager, Lottery, Purgecounter, Voicechannel, u.a. |
| 10 | [Externe Tools](10_EXTERNE_TOOLS.md) | EconomyBob, PP-Live-Logcrawler, Monitor-Dashboard |
| 11 | [Dashboard](11_DASHBOARD.md) | Web-Dashboard mit OAuth, 10 Tabs, Kronen-System |
| 12 | [Spieler-Befehle](12_SPIELER_BEFEHLE.md) | Alle Spieler-Interaktionen (Discord + Dashboard) |
| 13 | [LLM Admin Bot](13_LLM_ADMIN.md) | KI-Assistent mit Anomalie-Erkennung (in Entwicklung) |
| 14 | [RCON-Zukunft](14_RCON_ZUKUNFT.md) | Vorbereitung auf RCON-Integration |
| 15 | [PP-Live-Logcrawler](15_PP_LIVE_LOGCRAWLER.md) | Konsolen-Crawler, Health-Monitoring, Crash-Detection |
| 16 | [Cogs-Referenz](16_COGS_REFERENZ.md) | Lade-Architektur, Ladezustand, Patterns, Abhaengigkeiten |

## Einstieg

Starte mit der [00_ARCHITEKTURPRINZIP.md](00_ARCHITEKTURPRINZIP.md) — dort findest du das vollständige Inhaltsverzeichnis mit Unterpunkten zu jedem Dokument.

## Hinweis

IDs, IPs und Credentials in dieser Dokumentation sind durch Platzhalter ersetzt (z.B. `<ADMIN_DISCORD_ID>`, `<LAN_IP>`).

---

*Die Dokumentation wurde erstellt mit Unterstützung von Claude (Anthropic).*
