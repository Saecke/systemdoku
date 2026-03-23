# Architekturprinzip & Dokumentationsindex

## Inhaltsverzeichnis

### 00 — Dieses Dokument
Architekturprinzip, Entkopplungsphilosophie, Dokumentationsindex

---

### 01 — [Systemkarte](01_SYSTEMKARTE.md)
Gesamtstruktur des Systems auf einen Blick.
- Architekturdiagramm (ASCII): alle Prozesse und ihre Beziehungen
- **Startkette:** `start.ahk` → `systemstart.py` → `log_separator.ahk` → `bot_game-manager.py` → `streamtochat.ahk`
- **StreamToChat-Pipeline:** `.txt` → `streamtochat.py` → `.tochat` → `streamtochat.ahk` → Spielfenster
- **AHK-Abhängigkeiten:** Welches Skript startet welches
- Kern-Infrastruktur (app_config, loader, logging, heartbeat, http_monitor)
- Haupt-Bot (`bot_infobot.py`) + 50 Cogs, gruppiert nach Funktion
- Standalone-Bots (8 eigene Prozesse)
- Alle 20+ Datenbanken mit Hauptnutzer und Zweck
- Datenfluss-Diagramm (Spielserver → Logs → Discord)
- Konfigurations-Dateien (`ini/`) mit Zuordnung

---

### 02 — [Wirtschaft & Handel](02_WIRTSCHAFT.md)
Alles rund um Gold, Handel und Zahlungen.
- **bank.py + bank_db.py:** Tresor-System, Ein-/Auszahlung, Standortprüfung, 24h-Cooldown, Banksperre (Lock-File)
- **handelshaus.py:** Forum-basiertes Auktionshaus, Snipe-Schutz, Gebots-Staffelung, Rollen-basierte Limits
- **trade_stats.py:** Parst `[Trade]`-Logs → `trades.db` (Analytik, kein Gold-Einfluss)
- **insurance.py + check_insurance.py:** KFZ-Versicherung & Fahrzeugübersicht, V-Nummern, Despawn-Countdown, Fahrzeughistorie, Owner-Fahrzeuge (auch für Nicht-Versicherte), Fuhrpark-Lizenz-Warnungen
- **lizenzen.py:** 3 Lizenztypen (Banker/Fleet/Bau), 7-Tage-Laufzeit, Gold-Abzug
- **donations.py:** PayPal→payment.db, Sponsor-Rolle (30 Cent = 1 Tag), Stammtisch permanent
- DB-Schemas: bank.db (Konten, Aktionen, Lizenzen, Benutzeraktionen), handelshaus.db, trades.db, donations.db
- Datenfluss-Diagramme: Einzahlung, Lizenzkauf, Trades, PayPal-Claim
- Zentrale Verknüpfung: userdata.db als Hub

---

### 03 — [Tracking & Monitoring](03_TRACKING_MONITORING.md)
Echtzeit-Überwachung und Statistiken.
- **statbot.py:** Nachrichten-Zähler, Trade-/Währungs-Parsing, Live-Dashboard mit Deltas, Inflation/Deflation
- **kill_feed.py:** NPC-Tötungen → formatierte Embeds, Hash-Deduplizierung, Rollen-Farbkodierung
- **server_status.py:** BattleMetrics-API (10-Min-Cache), TPS/Entity-Counts aus PP-Logcrawler INIs, Sparklines, Bot-Presence
- **trader_status.py:** `[Trade] After`-Logs → Händler-Fondsstände nach Gebiet, Einzelnes Embed
- **steam_watch.py:** Login-Parsing → neue Spieler/Namensänderungen erkennen, Steam-API (Bans/Profile)
- **voice_rewards.py:** Voice-Zeit → Gold (Formel: time/60 * rate), min. 2 aktive Mitglieder, 5-Min-Auszahlungstask
- **health.py:** Supervisor-only Diagnose, alle DBs prüfen, Task-Status, HTTP-Monitor, Reboot/Shutdown/Reload
- Datenflüsse: Login→steam_watch, Trade→statbot+trader_status, Voice→userstats→bank

---

### 04 — [Spieler & Identität](04_SPIELER_IDENTITAET.md)
Die Brücke zwischen Discord und Spielserver.
- **identity.py:** Steam↔Discord-Verlinkung, 20-Zeichen-UUID (5 Min gültig), persistentes Embed mit Buttons
- **profile.py:** Aggregiert aus 3+ DBs (userdata, lottery, event_stats), zeigt Titel/Punkte/Squad
- **char.py:** 100+ Survival-Stats in 6 Kategorien, mehrseitige Embeds, `/charx` Vergleichsansicht
- **chars.py:** Leere Datei (Platzhalter für Leaderboards)
- **history.py:** Game-Logs erfassen (on_message), max 100 pro Spieler (FIFO), Fremd-IDs verschleiert, 7-Tage Auto-Purge
- **link_translator.py:** Tail-Read auf `chat.log`, Koordinaten → scum-map.com Links, 5s Cooldown
- **team_sync.py:** JSON aus Discord-Nachricht → `team.ini` + `admins.txt` + `moderators.txt` (UTF-16), 5-Min-Sync
- **db.py:** `get_steam_id_from_discord_id()` — Read-Only, Typ-Mismatch-Handling
- userdata.db als zentraler Knotenpunkt: alle Cogs die steam_id brauchen lesen hier

---

### 05 — [Events & Gameplay](05_EVENTS_GAMEPLAY.md)
Spielinhaltliche Systeme und Progression.
- **game_events.py:** Echtzeit-Log-Parser auf 4 Kanälen (Login/Lockpick/Airdrop/Killbox), Counter C_1–C_4, Trade-Parsing ST_1/ST_2
- **map_event_challenges.py:** Dynamische Item-Sammel-Challenges, Difficulty Ramping, Beitrags-Tracking pro Spieler
- **purge.py:** Squad-Flaggen-Registrierung für Zombie-Purge, kostet 5 Gold aus Squad-Tresor, Leader-only
- **bunker.py:** Bunker-Lock/Unlock-Tracking via `[LogBunkerLock]`, 2h-Aktivierungsfenster, Status-Embed
- **fishing.py:** 14 Fischarten, Leaderboard, Kronen (🎣) für Bestplatzierte
- **awards.py:** Kombiniertes Top-10: Kronen (👑, 44+ Kategorien) + Angel-Awards + Event-Trophäen (🏆)
- **titles.py:** 50+ Discord-Rollen, Punkte-Berechnung aus survival_stats, Rollen-Sync, "Gott unter Sterblichen" ab 95%
- **npczones.py:** Aktive NPC-Zonen Status-Embed (120s-Loop), Zeitfenster, PID-Online-Check
- **aircraft.py:** Flugzeugkauf per DM, Standortprüfung, Gold abziehen, Gamechat-Befehle (Teleport→Spawn)
- Stat-Pipeline: Serverlogs → game_events → userdata → titles → Discord-Rollen

---

### 06 — [Logs & Kommunikation](06_LOGS_KOMMUNIKATION.md)
Nachrichtenbrücken, UI und Administration.
- **chat_bridge.py:** Discord→Spielchat (GAMECHAT_DIR), Remote-Admin mit Bestätigung (✅/❌, 30s), Eventsperre-Check
- **log_bridge.py:** Leichtgewichtiger Log-Relay, Encoding-Kette, Chunking (1900 Zeichen), rename/skip Phrases
- **inselkom.py:** Erweiterter Log-Relay mit Per-Channel Rate-Limiting, Quarantäne, Burst-Cap, Diagnose-Logging
- **router.py:** Persistentes UI (Buttons + Dropdown, 14 Optionen), ephemerale Antworten, Cross-Cog-Delegation
- **router_v2.py:** Components v2 mit Untermenüs (Account/Stats/Titles/Economy), Feature-Flag
- **rules.py:** Regelakzeptanz-Embed mit Rollen-Vergabe, Components v2 Fallback
- **squad.py:** Squad-Roster-Sync aus Spieldatei (15 Min), `/squad` mit Gold-Transfer und Purge-Status
- **audit.py:** Befehls-Logging in Kanal, 2s Pacing
- **cleanup_messages.py:** Auto-Purge alter Nachrichten, Round-Robin, Bulk-Delete, Gepinnte geschützt
- **maintenance.py:** 5-Min-Koordinator: Squad→Titles→History→Lottery/Mapevent-Rollen
- **bootstrap.py:** Guild-Whitelist, File-Watcher (10s), Auto-Restart via `os.execv()`
- **hooks.py:** Webhook-Lifecycle mit Expiry, JSON-Persistierung
- **core.py:** Minimaler Startup, `bot.tree.sync()`
- **basic.py:** `/status` — Verlinkung + Sponsor-Info
- **help.py:** `/hilfe` — Befehlsübersicht, dynamischer `/log`-Eintrag

---

### 07 — [AHK-Automatisierung](07_AHK_AUTOMATISIERUNG.md)
Hardware-nahe Automatisierung zwischen Spielfenster und System.
- **start.ahk:** Einstiegspunkt — löscht Locks, startet systemstart.py → log_separator → game-manager → streamtochat
- **log_separator.ahk:** Endlos-Loop, parst `serverlogs\download\` in 15+ Ordner, Rollen-Farbkodierung (ANSI), TZ-Korrektur
- **streamtochat.ahk:** 650ms Polling auf `gamechat\*.tochat`, Tastatur-Simulation ins SCUM-Fenster
- **onclipactions.ahk:** Clipboard-Hook → sortiert Daten in ini/*.txt (cars, squads, players, flags), klickt Sponsored-Dialog weg
- **get_flags.ahk:** Löscht flags.txt, schreibt `#ListFlags` in Gamechat, startet `build_flag_db.pyw`
- **eventmanager.ahk:** Blackout-Fenster (4x täglich), 5-Min-Cooldown, Zufalls-Event, Speech-Dateien, startet Event-Handler
- **purge_checker.ahk:** 6 Punkt-Kategorien, Schwelle 10.000, Wahrscheinlichkeit 50%–100%, Trost-Airdrop
- **remote.ahk:** Polling `remote\` (10s), Codes: 99=Reboot, 9=Event, 10=Cancel, 11-13=Horde-Varianten
- **reboot.ahk:** Loggt + `Shutdown, 6` (30s Delay)
- **log_mover_sab.ahk:** SAB_Logs → UTF-16 → `serverlogs\download\`
- Kommunikation AHK↔Python: ausschließlich über Dateien (gamechat/, serverlogs/, ini/, events/)

---

### 08 — [Support-Skripte](08_SUPPORT_SKRIPTE.md)
Infrastruktur-Skripte und Hilfsbibliotheken.
- **systemstart.py:** Startet alle Python-Bots sequentiell mit Delays (payment→rsync→voice→titles→infobot→ftp)
- **timer.py:** Zentraler Scheduler (schedule-Library), 10+ Jobs (Restart-Warnungen, Purge, Mapevent, Handelsbericht, Daily)
- **ftp_sync.py:** Adaptives Polling (2–10s), Offset-Tracking, Log-Rotation-Erkennung, Reboot-Trigger bei Timestamp-Änderung
- **config_changer.py:** Zeitgesteuert AllowEvents=0/1 per FTP-Upload (02:00/14:00 Uhr)
- **streamtochat.py:** Watchdog auf gamechat/, Timing-Syntax `<<<DELAY>>>`, .txt→.tochat Konvertierung
- **payment_mail_watcher.py:** IMAP-Monitor, HTML-Parsing, Deduplizierung, payment.db, CLI-Tools
- **mapevent.py:** Event-Orchestrator, NPC-Spawning, Spieler-Proximity-Tracking, Ergebnis-JSON + DB
- **npczones_spawner.py:** NPC-Zonen-Rotation, PID-Check, Zeitfenster, Participation-Tracking
- **steamcmd.py:** Steam-API-Polling → Discord-Webhook bei Version-Update
- **purge_files.py:** Rekursive Log-Bereinigung (7 Tage, .log.txt)
- **MapTools.py:** Koordinaten→Sektor, Bauverbotszonen-Check (ImageMagick Pixel-Analyse)
- **bunker_func.py:** Bunker-Log-Parsing, BunkerActivationData-Klasse, INI-Persistierung
- **loader.py:** Cog-Loader für bot_infobot, iteriert `cfg.startup_cogs`

---

### 09 — [Standalone-Bots](09_STANDALONE_BOTS.md)
8 eigenständige Bot-Prozesse mit eigenen Tokens.
- **bot_game-manager.py:** Master-Automatisierung — Login-Überwachung, RCV-Workflow (Chat-Verifikation nach Login), Daten-Fetch (Vehicles/Players/Squads/Flags), StreamToChat-Bridge, force_reboot() bei Fehlern
- **bot_squadmanager.py:** Automatische Squad-Channels (Text+Voice), Guest-System mit Expiry, 6-Min-Lifecycle, aiosqlite, Retry+Jitter
- **bot_titles.py:** FTP-Download SCUM.db → Skill-Erkennung → Discord-Rollen vergeben, Kombi-Rollen, persistent_stats.db befüllen
- **bot_mapevent_challenge.py:** Community-Challenges (Quest/Trade/Lockpick-Zähler), adaptive Ziele, Server-Restart-Erkennung, auto-Event bei Zielerreichung
- **bot_lottery.py:** Ticket-Lotterie aus Item-Trades, Batch-Processing, Leaderboard, automatische Ziehung, Overflow-Redistribution
- **bot_purgecounter.py:** 5-Kanal-Aktivitätszähler (Login/Handel/Chat/Lockpick/Quests) mit Multiplikatoren, Purge-Wahrscheinlichkeit 0–100%
- **bot_voicechannel.py:** Creator-Channel → eigener Voice-Raum, !add/!guest/!kick/!name Befehle, Guest-Expiry, Auto-Cleanup leerer Channels
- **airdrop.py:** Airdrop-Scheduler mit Zeitfenstern, Sperrzonen, Flaggen-Proximity-Check, Trigger-Dateien (drop.go, drop.override)
- **handelsbericht.py:** Täglicher Trade-Report, Top-10 Items, Inflation/Deflation, Archivierung in historical_stats.db, Discord-Webhook

---

### 10 — [Externe Tools](10_EXTERNE_TOOLS.md)
3 eigenständige Toolsuiten als separate Daemons.
- **EconomyBob/:** 2-Thread-Design (Poller+Poster), FTP→economy.db (WAL)→Discord-Webhook, SHA-256 Deduplizierung, UTF-16-LE Parsing, 48 Filtermuster
- **PP-Live-Logcrawler/:** Browser-Automation (Selenium) auf PingPerfect-Panel, 17+ Log-Typen mit eigenen DBs, Health-Monitoring (TPS/FPS/Entities), Crash-Detection (15s Threshold), **exportiert playercount.ini + serverhealth.ini** (gelesen von server_status.py Cog)
- **monitor/:** Discord-Bot-Watchdog, 4 Check-Typen (presence/channel_watch/multi_channel/activity), 10 Konsolen-Prozesse überwacht, SQLite mit Events/Alerts/Status, Flask Web-Server (:5000), CustomTkinter Desktop-GUI, JSON-Bridge, DM-Steuerung (start/stop/restart Bots), Singleton-Lock

---

### 11 — [Dashboard](11_DASHBOARD.md)
Web-Dashboard für Spieler unter scumsaecke.de/dash/.
- **Stack:** PHP 7+ / SQLite / Bootstrap 5.3 / Discord OAuth 2.0 (PKCE)
- **10 Tabs:** Profil, Squad, KFZ, Char (7 Kategorien), Chars (Leaderboards), CharX (Vergleich), Fischen, Events, Titel (50+), Logs
- **Spieler-Eingaben:** Widget-Reihenfolge, Karten ein/aus, Restart-Alert-Settings, Log-Suche
- **Kronen-System:** 👑 für Server-Bestwerte (54+ Metriken), 🎣 Angeln, 🏆 Events
- **13 SQLite-DBs** (Read-Only-Kopien der Originale aus ini/database/)
- **Dev-Tools:** Owner-only Impersonation, Simulations-Flags, Audit-Logs, DB-Health-Check
- **Sicherheit:** PKCE (kein Client-Secret), CSRF, Prepared Statements, IP-Anonymisierung

---

### 12 — [Spieler-Befehle](12_SPIELER_BEFEHLE.md)
Komplette Übersicht aller Spieler-Interaktionen.
- **18 Discord-Befehle:** /link, /status, /profil, /char, /charx, /chars, /tresor, /lizenzen, /kfz, /flugzeug, /squad, /purge, /titel, /todo, /rangliste, /fischen, /awards, /history, /hilfe, /spende
- **Router-UI:** 6 Buttons + 14-Option-Dropdown (ephemeral)
- **Identity-Buttons:** Verlinkung starten/prüfen (persistent)
- **Voice-Channel:** !add, !remove, !guest, !unguest, !list, !kick, !name
- **Dashboard:** Widget-Anpassung, Restart-Alerts, Log-Suche
- **Datenquellen pro Feature:** Welcher Befehl welche DB/Tabelle liest
- **Was Spieler NICHT können:** Keine Stat-Änderungen, keine fremden Logs, keine Admin-Befehle

---

### 15 — [PP-Live-Logcrawler](15_PP_LIVE_LOGCRAWLER.md)
Eigenständiges Log-Monitoring-System für PingPerfect-Konsole.
- **Browser-Automation:** Selenium Chrome → XHR-Streaming pro Log-Typ (17+ Typen, je eigener Thread)
- **SCUM Health:** Global Stats parsen (TPS, Entities), Crash-Detection (15s Threshold), Startup-Watchdog (30s Grace)
- **INI-Export:** `playercount.ini` + `serverhealth.ini` → gelesen von `server_status.py` Cog
- **Crash-Recovery:** Exponentielles Backoff, Auto-Restart bei >2 Crashes in 5 Min, 60s Freeze → Hard Reboot
- **Archiv:** Tägliche Log-Rotation nach `archive/{YYYY-MM-DD}/`
- **GUI-Tools:** Config-Editor + Health-Dashboard mit TPS-Farbkodierung

---

### 14 — [RCON-Zukunft](14_RCON_ZUKUNFT.md) *(geplant)*
Vorbereitung auf RCON-Support durch die SCUM-Entwickler.
- RCON als alternativer Transport-Layer, liest dieselben `gamechat/*.txt`-Dateien
- Am bestehenden System ändert sich **nichts** — gleiche Dateien, anderer Verarbeiter
- StreamToChat bleibt als Fallback, Log-Pipeline bleibt, Daten-Lesen bleibt
- Entscheidung StreamToChat vs. RCON per Config/Switch

---

### 13 — [LLM Admin Bot](13_LLM_ADMIN.md) *(in Entwicklung)*
KI-gestützter Server-Assistent mit Anomalie-Erkennung.
- **Chat-Assistent:** Reaktiv auf Admin-Fragen, LM Studio lokal (Qwen 3.5 9B), 12 Tools (DB-Abfragen, Log-Suche, Regeln)
- **Log-Monitor:** Proaktiv, 10+ Log-Quellen, Event-Buffer mit TTL, Baseline-Learning (EMA, 24h Lernphase)
- **Anomalie-Erkennung:** 3.0x über Baseline = Anomalie, immer alarmieren bei BattlEye/DDoS/Ban/Cheat
- **Normale Muster (kein Alarm):** Häufige Logins (SCUM crasht), Bulk-Trades, hohe Kontostände
- **Verdächtige Muster (Alarm):** Arbitrage, BattlEye-Kicks, DDoS, Spam/Beleidigungen
- **Sicherheit:** 3-Schichten (Pfad-Whitelist locked nach Start, URI mode=ro, SQL-Keyword-Blocking)
- **Daten via SMB:** bank.db, userdata.db, userstats.db, flags.db + PP-Logcrawler Logs + 5 Regel-Dateien
- **Config:** config.yaml mit Discord/LLM/DB/Log/Monitor-Sektionen
- **Admin-Korrekturen:** llm_notes.txt (wird an jeden Prompt angehängt)

---

## Kernphilosophie: Vollständige Entkopplung

Jede Komponente ist eine **autarke Einheit**. Kein Bot hängt von einem anderen ab, kein Cog vom anderen.

### Regeln
- **Bots**: Jeder Standalone-Bot ist eigenständig lauffähig. Fällt einer aus, läuft der Rest weiter.
- **Cogs**: Viele Cogs im Haupt-Bot (`bot_infobot.py`) kann einzeln in der Config deaktiviert werden, ohne Seiteneffekte.
- **Kommunikation**: Komponenten kommunizieren ausschließlich über:
  - Zentrale Datenpunkte (Dateien, Datenbanken)
  - `on_message`-Events (Discord-Nachrichten als Bus)
  - Postjobs / Hooks
- **Keine direkten Abhängigkeiten** zwischen Cogs oder zwischen Bots.

### Konsequenzen
- System ist extrem resilient (Teilausfälle bleiben lokal)
- Neue Features können als isolierte Cogs/Bots hinzugefügt werden
- Reihenfolge von Start/Stop spielt keine Rolle
- Trade-off: Gemeinsame Logik wird teils dupliziert statt geteilt

---

## Cog-Übersicht (bot_infobot.py → _cogs/)

Alle Cogs werden dynamisch über `_modules/loader.py` geladen und können einzeln in der Config deaktiviert werden.

### Spieler & Identität
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| Identity | identity.py | Steam↔Discord-Verlinkung (UUID-Flow) | [04](04_SPIELER_IDENTITAET.md) |
| Profile | profile.py | Spielerprofil (aggregiert 3+ DBs) | [04](04_SPIELER_IDENTITAET.md) |
| Char | char.py | Charakter-Stats (100+ Werte, 6 Kategorien) | [04](04_SPIELER_IDENTITAET.md) |
| Chars | chars.py | Platzhalter (Leaderboards, nicht implementiert) | [04](04_SPIELER_IDENTITAET.md) |
| History | history.py | Spieler-Logs mit ID-Verschleierung | [04](04_SPIELER_IDENTITAET.md) |
| LinkTranslator | link_translator.py | Koordinaten aus Globalchat → Kartenlinks | [04](04_SPIELER_IDENTITAET.md) |
| TeamSync | team_sync.py | Team-Roster aus Discord → INI-Dateien | [04](04_SPIELER_IDENTITAET.md) |

### Wirtschaft & Handel
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| Bank | bank.py | Gold-Tresor, Ein-/Auszahlung, Standortprüfung | [02](02_WIRTSCHAFT.md) |
| Handelshaus | handelshaus.py | Auktionshaus mit Snipe-Schutz | [02](02_WIRTSCHAFT.md) |
| TradeStats | trade_stats.py | Trade-Log-Parsing → Analytik-DB | [02](02_WIRTSCHAFT.md) |
| Insurance | insurance.py | KFZ-Versicherung & Fahrzeugübersicht, Despawn, Owner-Fahrzeuge, Lizenz-Warnungen | [02](02_WIRTSCHAFT.md) |
| CheckInsurance | check_insurance.py | Admin: V-Nummern zuweisen/entfernen | [02](02_WIRTSCHAFT.md) |
| VehicleMonitor | vehicle_monitor.py | Automatischer Fahrzeuglimit-Monitor → Admin-Channel (max 1x/h) | [02](02_WIRTSCHAFT.md) |
| Lizenzen | lizenzen.py | Banker/Fleet/Bau-Lizenzen kaufen | [02](02_WIRTSCHAFT.md) |
| Donations | donations.py | PayPal-Spenden, Sponsor-Rollen | [02](02_WIRTSCHAFT.md) |

### Tracking & Monitoring
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| TrackingStation | trackingstation.py | flags.txt → flags.db + Event-Log-Verarbeitung | [03](03_TRACKING_MONITORING.md) |
| Statbot | statbot.py | Nachrichten/Trade-Zähler, Live-Dashboard | [03](03_TRACKING_MONITORING.md) |
| KillFeed | kill_feed.py | NPC-Tötungen → formatierte Embeds | [03](03_TRACKING_MONITORING.md) |
| ServerStatus | server_status.py | BattleMetrics-API + TPS/Entity-Anzeige | [03](03_TRACKING_MONITORING.md) |
| TraderStatus | trader_status.py | Händler-Fondsstände nach Gebiet | [03](03_TRACKING_MONITORING.md) |
| SteamWatch | steam_watch.py | Neue Spieler + Namensänderungen erkennen | [03](03_TRACKING_MONITORING.md) |
| VoiceRewards | voice_rewards.py | Voice-Zeit → Gold-Belohnung | [03](03_TRACKING_MONITORING.md) |
| Health | health.py | Supervisor-Diagnose + Bot-Steuerung | [03](03_TRACKING_MONITORING.md) |

### Events & Gameplay
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| GameEvents | game_events.py | Echtzeit-Log-Parser → Counter C_1–C_4, ST_1–ST_2 | [05](05_EVENTS_GAMEPLAY.md) |
| MapEventChallenges | map_event_challenges.py | Item-Sammel-Challenges mit Difficulty Ramping | [05](05_EVENTS_GAMEPLAY.md) |
| Purge | purge.py | Zombie-Purge Squad-Registrierung | [05](05_EVENTS_GAMEPLAY.md) |
| Bunker | bunker.py | Bunker-Lock/Unlock-Status-Monitor | [05](05_EVENTS_GAMEPLAY.md) |
| Fishing | fishing.py | Angel-Stats & Leaderboard (14 Arten) | [05](05_EVENTS_GAMEPLAY.md) |
| Awards | awards.py | Kombiniertes Top-10 (Kronen+Angel+Trophäen) | [05](05_EVENTS_GAMEPLAY.md) |
| Titles | titles.py | 50+ Discord-Rollen, Punkte-System | [05](05_EVENTS_GAMEPLAY.md) |
| NPCZones | npczones.py | NPC-Aktionszonen Status-Embed | [05](05_EVENTS_GAMEPLAY.md) |
| Aircraft | aircraft.py | Flugzeugkauf mit Gamechat-Integration | [05](05_EVENTS_GAMEPLAY.md) |
| Carservice | carservice.py | act_cars.txt → carservice.db | [02](02_WIRTSCHAFT.md) |
| CarsUploader | cars_uploader.py | Fahrzeugdaten-Upload | [02](02_WIRTSCHAFT.md) |

### Kommunikation & Logs
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| ChatBridge | chat_bridge.py | Discord → Spielchat + Remote-Admin | [06](06_LOGS_KOMMUNIKATION.md) |
| LogBridge | log_bridge.py | Lokale Logs → Discord-Kanäle (leichtgewichtig) | [06](06_LOGS_KOMMUNIKATION.md) |
| Inselkom | inselkom.py | Erweiterter Log-Relay mit Rate-Limiting | [06](06_LOGS_KOMMUNIKATION.md) |
| Router | router.py | Persistentes UI (Buttons + Dropdown) | [06](06_LOGS_KOMMUNIKATION.md) |
| RouterV2 | router_v2.py | Components v2 UI mit Untermenüs | [06](06_LOGS_KOMMUNIKATION.md) |

### Administration
| Cog | Datei | Kurzbeschreibung | Doku |
|---|---|---|---|
| Core | core.py | Startup, Slash-Command-Sync | [06](06_LOGS_KOMMUNIKATION.md) |
| Bootstrap | bootstrap.py | Guild-Whitelist, File-Watcher, Auto-Restart | [06](06_LOGS_KOMMUNIKATION.md) |
| Audit | audit.py | Befehls-Logging | [06](06_LOGS_KOMMUNIKATION.md) |
| Maintenance | maintenance.py | 5-Min-Koordinator (Squad→Titles→History) | [06](06_LOGS_KOMMUNIKATION.md) |
| CleanupMessages | cleanup_messages.py | Auto-Purge alter Nachrichten | [06](06_LOGS_KOMMUNIKATION.md) |
| Hooks | hooks.py | Webhook-Lifecycle mit Expiry | [06](06_LOGS_KOMMUNIKATION.md) |
| Squad | squad.py | Squad-Roster-Sync + /squad Befehl | [06](06_LOGS_KOMMUNIKATION.md) |
| Rules | rules.py | Regelakzeptanz-Embed → Rollen-Vergabe | [06](06_LOGS_KOMMUNIKATION.md) |
| Basic | basic.py | /status Befehl | [06](06_LOGS_KOMMUNIKATION.md) |
| Help | help.py | /hilfe Befehlsübersicht | [06](06_LOGS_KOMMUNIKATION.md) |

---

## Modul-Übersicht (_modules/)

Shared Libraries die von allen Cogs und teilweise von Standalone-Bots genutzt werden.

| Modul | Datei | Kurzbeschreibung | Genutzt von |
|---|---|---|---|
| AppConfig | app_config.py | Zentrale Config (Pfade, IDs, DB-Pfade, Feature-Flags) | Allen Cogs + Bots |
| Config | config.py | Bot-spezifische Config | bot_infobot.py |
| Loader | loader.py | Cog-Loader, iteriert `cfg.startup_cogs` | bot_infobot.py |
| LoggingUtils | logging_utils.py | Zentrales Logging (`printlog`) | Allen Cogs |
| HTTPMonitor | http_monitor.py | HTTP-Request-Monitoring + Rate-Limit-Tracking | statbot, server_status, trader_status, audit |
| Heartbeat | heartbeat.py | CPU/RAM/PID-Metriken → heartbeat.db | Kern-Infrastruktur |
| HealthControls | health_controls.py | Health-Check-Infrastruktur | health.py Cog |
| DB | db.py | `get_steam_id_from_discord_id()` (Read-Only) | history, voice_rewards, u.a. |
| BankDB | bank_db.py | Kontostand lesen/ändern, Aktionen loggen | bank, lizenzen, voice_rewards, aircraft |
| BunkerFunc | bunker_func.py | Bunker-Log-Parsing, State-Persistierung | bunker.py Cog |
| Utils | utils.py | Allgemeine Hilfsfunktionen | Diverse Cogs |
| Announcements | announcements.py | Ankündigungs-Modul (minimal) | — |
| PurgeIO | purge_io.py | Purge-Datei-Operationen | purge.py Cog |
| PurgeStatus | purge_status.py | Purge-Status-Abfragen | purge.py Cog |
| LinkState | link_state.py | UUID-Tracking für Verlinkung | identity.py Cog |
| ProcessUtils | process_utils.py | Prozess-Management | Kern-Infrastruktur |
| AutoMinimize | auto_minimize.py | Fenster-Minimierung | Standalone-Bots |
