# Systemkarte — SCUM Server Management Suite

Gesamtübersicht des Systems. Stand: 2026-03-30

---

## Architektur auf einen Blick

```
                           start.ahk
                        (Einstiegspunkt)
                              |
          +-------------------+-------------------+
          |                   |                   |
    systemstart.py     log_separator.ahk    streamtochat.ahk
    (Bot-Watchdog)     (Log-Parsing)        (Chat-Injection)
          |                   |                   |
          |            serverlogs\download\   gamechat\*.tochat
          |              ↓ 15+ Ordner             ↑
          |            log_bridge.py /        streamtochat.py
          |            inselkom.py (Cogs)     (*.txt → *.tochat)
          |                                       ↑
          |                                  gamechat\*.txt
          |                                       ↑
          |                              chat_bridge.py, timer.py,
          |                              mapevent.py, eventmanager.ahk,
          |                              bank.py, aircraft.py, ...
          |
          +---> bot_infobot.py (Haupt-Bot)
          |         ↓ lädt _cogs/ (50+)
          |         ↓ nutzt _modules/
          |
          +---> bot_game-manager.py ──→ streamtochat.py
          +---> bot_lottery.py
          +---> bot_purgecounter.py
          +---> bot_squadmanager.py
          +---> bot_voicechannel.py
          +---> bot_titles.py
          +---> bot_mapevent_challenge.py
          +---> bot_bunker_functions.py
          +---> payment_mail_watcher.py
          +---> weitere Standalone-Bots...

    Parallel laufende AHK-Skripte (nicht von start.ahk gestartet):
    ┌─────────────────────────────────────────────────────┐
    │ onclipactions.ahk    Clipboard → ini/*.txt          │
    │ remote.ahk           remote/*.txt → Aktionen        │
    │ log_mover_sab.ahk    SAB_Logs → serverlogs\download │
    │ eventmanager.ahk     Event-Orchestrierung           │
    │ purge_checker.ahk    Punkte → Purge-Trigger         │
    │ reboot.ahk           System-Neustart                │
    │ get_flags.ahk        Flaggen-Refresh                │
    └─────────────────────────────────────────────────────┘

    Externe Toolsuiten (eigene Prozesse):
    ┌─────────────────────────────────────────────────────┐
    │ EconomyBob/          Wirtschafts-Log-Monitoring      │
    │ PP-Live-Logcrawler/  Konsolen-Crawler                │
    │ monitor/             Web-Dashboard                   │
    └─────────────────────────────────────────────────────┘

    === NUC (Ubuntu, Werkstattrack) ===
    ┌─────────────────────────────────────────────────────┐
    │ UNIX-PC/update-manager/                              │
    │   steamcmd.py        Update-Poller + Dispatcher      │
    │   pp_updater.py      PP-Panel-Automatisierung        │
    └─────────────────────────────────────────────────────┘

    === Bot-PC (Windows, Autostart) ===
    ┌─────────────────────────────────────────────────────┐
    │ watchdog/                                            │
    │   watchdog_update-manager_linux.py                   │
    │                      Überwacht NUC-Heartbeat,         │
    │                      Discord-Alarm bei Ausfall       │
    └─────────────────────────────────────────────────────┘
```

## Startkette im Detail

```
start.ahk
  │
  ├─ 1. SYSTEM-CLEANUP (perfekter Ausgangszustand):
  │     ├─ Löscht Lock-Files (Eventsperre.txt, Banksperre.txt)
  │     ├─ Löscht alte Logs und nicht-übergebene Befehle
  │     └─ Sperren und temporäre Dateien werden zurückgesetzt
  │
  ├─ 2. [55s Delay] systemstart.py
  │     └─ Startet alle Python-Bots als Subprozesse
  │        └─ bot_infobot.py (Haupt-Bot → lädt _cogs/ via loader.py)
  │        └─ bot_lottery.py, bot_purgecounter.py, ...
  │
  ├─ 3. [15s Delay] log_separator.ahk
  │     └─ Endlos-Loop: serverlogs\download\*.log → 15+ kategorisierte Ordner
  │
  ├─ 4. [10s Delay] bot_game-manager.py
  │     └─ Startet auch streamtochat.py
  │
  └─ 5. [2.5s Delay] streamtochat.ahk
        └─ Endlos-Loop: gamechat\*.tochat → SCUM-Fenster (Tastatur-Simulation)
```

## AHK-Skripte: Wer startet wen?

```
start.ahk ──────→ systemstart.py
                → log_separator.ahk
                → bot_game-manager.py
                → streamtochat.ahk + streamtochat.py

remote.ahk ─────→ eventmanager.ahk (Codes 9, 11-13)
                → reboot.ahk (Code 99)
                → timer.ahk (Code 1, Restart)

eventmanager.ahk → events\{eventArt}\{eventArt}.ahk (dynamisch)

purge_checker.ahk → events\purge\purge.py

log_separator.ahk → events\stuckport\stuck.ahk (bei Trigger)

get_flags.ahk ───→ scripts\build_flag_db.pyw
```

## StreamToChat-Pipeline (Discord → Spielchat)

```
Cog/Skript schreibt Nachricht
    ↓
gamechat\{timestamp}.txt         ← chat_bridge.py, timer.py, mapevent.py,
    ↓                               eventmanager.ahk, bank.py, aircraft.py, ...
streamtochat.py (watchdog)
    ↓ parst Timing-Direktiven (<<<DELAY>>>)
    ↓ konvertiert .txt → .tochat
gamechat\{timestamp}.tochat
    ↓
streamtochat.ahk (650ms Polling)
    ↓ aktiviert SCUM-Fenster
    ↓ T → Ctrl+A → Delete → Ctrl+V → Enter
Spielchat im Spiel
```

**Zwei Komponenten, klare Arbeitsteilung:**
- `streamtochat.py` — Queue-Processor: überwacht `gamechat/`, parst Timing, erzeugt `.tochat`
- `streamtochat.ahk` — Tastatur-Roboter: liest `.tochat`, tippt Text ins Spielfenster

Beide werden von `start.ahk` gestartet (Schritt 4 + 5). `streamtochat.py` wird zusätzlich von `bot_game-manager.py` als Subprocess mitgestartet.

---

## 1. Kern-Infrastruktur

| Datei / Ordner | Rolle |
|---|---|
| `start.ahk` | **Einstiegspunkt** — startet alle Prozesse |
| `systemstart.py` | Bot-Prozess-Management, Watchdog |
| `_modules/app_config.py` | Zentrale Konfiguration (Pfade, IDs, DB-Pfade, Feature-Flags) |
| `_modules/config.py` | Bot-spezifische Config |
| `_modules/loader.py` | Cog-Loader, lädt `_cogs/` dynamisch |
| `_modules/logging_utils.py` | Zentrales Logging (`printlog`) |
| `_modules/http_monitor.py` | HTTP-Monitoring für alle Requests |
| `_modules/heartbeat.py` | Heartbeat-Metriken (CPU, RAM, PID) -> `heartbeat.db` |
| `_modules/health_controls.py` | Health-Check-Infrastruktur |
| `_modules/process_utils.py` | Prozess-Management |
| `_modules/utils.py` | Allgemeine Hilfsfunktionen |

---

## 2. Haupt-Bot (`bot_infobot.py`) + Cogs

Der Haupt-Bot lädt alle Cogs aus `_cogs/`. Gruppiert nach Funktion:

### Spieler & Identität
| Cog | Funktion |
|---|---|
| `identity.py` | Steam-Discord-Verknüpfung |
| `profile.py` | Spielerprofile |
| `char.py` / `chars.py` | Charakterdaten |
| `history.py` | Spielerhistorie |
| `link_translator.py` | Link-Übersetzung / Verknüpfungen |
| `team_sync.py` | Team-Synchronisation |

### Wirtschaft & Handel
| Cog | Funktion |
|---|---|
| `bank.py` | Bankensystem (Konten, Transaktionen) -> `bank.db` |
| `handelshaus.py` | Handelshaus / Marktplatz -> `handelshaus.db` |
| `trade_stats.py` | Handelsstatistiken -> `trades.db` |
| `donations.py` | Spendensystem -> `donations.db` |
| `insurance.py` | Versicherungssystem |
| `check_insurance.py` | Versicherungsprüfung |
| `lizenzen.py` | Lizenzverwaltung |

### Fahrzeuge
| Cog | Funktion |
|---|---|
| `carservice.py` | Fahrzeugdienst: `act_cars.txt` -> `carservice.db` |
| `cars_uploader.py` | Fahrzeugdaten-Upload |

### Events & Gameplay
| Cog | Funktion |
|---|---|
| `game_events.py` | Spielevents |
| `map_event_challenges.py` | Map-Event-Challenges -> `mapevent_challenge.db` |
| `events_stats.py` | Event-Statistiken -> `event_stats.db` |
| `purge.py` | Purge-System |
| `bunker.py` | Bunker-Funktionen |
| `fishing.py` | Angelsystem |
| `awards.py` | Auszeichnungen |
| `titles.py` | Titelsystem |
| `npczones.py` | NPC-Zonen-Verwaltung |
| `aircraft.py` | Flugzeugsystem |

### Tracking & Monitoring
| Cog | Funktion |
|---|---|
| `trackingstation.py` | Phrasen-Tracking (Kanalnamen als Trigger), New-Player-Tracking (VAC/Ban → Auto-Kanal), Flaggen-Checks (Bauverbotszonen via MapTools) + `flags.db`-Sync |
| `statbot.py` | Spielerstatistiken -> `statbot.db`, `userstats.db` |
| `kill_feed.py` | Kill-Feed-Verarbeitung |
| `server_status.py` | Serverstatus-Anzeige |
| `trader_status.py` | Händlerstatus |
| `steam_watch.py` | Steam-Überwachung |
| `voice_rewards.py` | Voice-Channel-Belohnungen |

### Kommunikation & Logs
| Cog | Funktion |
|---|---|
| `chat_bridge.py` | Chat-Bridge (Ingame <-> Discord) |
| `log_bridge.py` | Log-Weiterleitung |
| `inselkom.py` | Insel-Kommunikation |
| `router.py` / `router_v2.py` | Nachrichtenrouting |
| `announcements` (Modul) | Ankündigungen |

### Administration
| Cog | Funktion |
|---|---|
| `core.py` | Kern-Admin-Befehle |
| `basic.py` | Basis-Befehle |
| `help.py` | Hilfesystem |
| `rules.py` | Regeln-Anzeige |
| `squad.py` | Squad-Verwaltung |
| `audit.py` | Audit-Log |
| `hooks.py` | Hook-System |
| `health.py` | Health-Check-Anzeige |
| `bootstrap.py` | Boot-Sequenz |
| `maintenance.py` | Wartungsmodus |
| `cleanup_messages.py` | Nachrichten-Bereinigung |

---

## 3. Standalone-Bots (eigene Prozesse)

| Bot | Funktion |
|---|---|
| `bot_game-manager.py` | Spielserver-Verwaltung, Flaggen-Export (`flags.txt`), StreamToChat |
| `bot_lottery.py` | Lotteriesystem -> `lottery.db` |
| `bot_purgecounter.py` | Purge-Countdown |
| `bot_squadmanager.py` | Squad-Management |
| `bot_voicechannel.py` | Voice-Channel-Automatik |
| `bot_titles.py` | Titel-Vergabe |
| `bot_mapevent_challenge.py` | Map-Event-Challenges |
| `bot_bunker_functions.py` | Bunker-Steuerung |
| `airdrop.py` | Airdrop-System |
| `mapevent.py` | Map-Event-System |
| `handelsbericht.py` | Handelsberichte |
| `payment_mail_watcher.py` | Payment-E-Mail-Überwachung -> `payment.db` |

---

## 4. Externe Toolsuiten

### EconomyBob/
Eigenständiges Wirtschafts-Monitoring: liest Server-Logs, pflegt `economy.db`, filtert Events.

### PP-Live-Logcrawler/
PingPerfect-Console-Crawler: liest Live-Konsole des Gameservers, verarbeitet Events, Player-Tracking.

### monitor/
Web-basiertes Monitoring-Dashboard: `web_server.py` + `monitor_gui.py`, eigene DB, JSON-Bridge.

### UNIX-PC/update-manager/ (läuft auf i3, Ubuntu)
Automatisiertes Server-Update-System: `steamcmd.py` pollt Steam API, `pp_updater.py` steuert PP-Panel per Selenium (Stop → Update → Start). Multi-Server (nacheinander). Kommuniziert per SMB (heartbeat.json → Bot-PC) und Discord-Webhooks.

### watchdog/ (läuft auf Bot-PC, Windows, Autostart)
Heartbeat-Monitor für den i3: liest `_pid/heartbeat.json` (vom i3 per SMB abgelegt), prüft Timestamp-Alter, sendet Discord-Alarm bei Ausfall. Grace Period (1h) verhindert Spam. Config: `watchdog/ini/update-manager/watchdog.ini`.

---

## 5. AutoHotkey-Skripte (Automatisierung)

| Skript | Funktion |
|---|---|
| `start.ahk` | System-Start-Automation |
| `reboot.ahk` | Server-Reboot-Steuerung |
| `get_flags.ahk` | Flaggen-Daten aus Spielkonsole holen |
| `log_separator.ahk` | Log-Dateien aufteilen |
| `streamtochat.ahk` | Text in Spielchat streamen |
| `remote.ahk` | Remote-Steuerung |
| `eventmanager.ahk` | Event-Steuerung |
| `purge_checker.ahk` | Purge-Zustand prüfen |
| `onclipactions.ahk` | Clipboard-Aktionen |

---

## 6. Datenbanken (`ini/database/`)

| Datenbank | Hauptnutzer | Zweck |
|---|---|---|
| `userdata.db` | identity, profile, char | Spieler-Stammdaten |
| `bank.db` | bank Cog | Kontostände, Transaktionen |
| `flags.db` | trackingstation Cog | Aktive Flaggen (Positionen, Besitzer) |
| `userstats.db` | statbot Cog | Spielerstatistiken |
| `statbot.db` | statbot Cog | Stat-Bot-Daten |
| `event_stats.db` | events_stats Cog | Event-Statistiken |
| `lottery.db` | lottery Bot | Lotteriedaten |
| `carservice.db` | carservice Cog | Fahrzeughistorie |
| `handelshaus.db` | handelshaus Cog | Handelshaus-Einträge |
| `trades.db` | trade_stats Cog | Handelsstatistiken |
| `donations.db` | donations Cog | Spenden |
| `payment.db` | payment_mail_watcher | Zahlungen |
| `heartbeat.db` | heartbeat Modul | System-Health-Metriken |
| `persistent_stats.db` | statbot Cog | Persistente Statistiken |
| `userhistory.db` | history Cog | Spielerhistorie |
| `historical_stats.db` | — | Historische Snapshots |
| `konten_snapshots.db` | bank Cog | Konten-Snapshots |
| `mapevent_challenge.db` | map_event_challenges | Challenge-Daten |
| `map_event_contributions.db` | — | Event-Beiträge |

---

## 7. Wichtige Datenflüsse

```
Spielserver (SCUM)
    |
    +-- Konsole --> get_flags.ahk --> ini/flags.txt --> trackingstation.py --> flags.db
    |
    +-- Logs --> PP-Live-Logcrawler --> serverlogs/ --> log_bridge.py --> Discord
    |                                              --> trackingstation.py (*.log)
    |
    +-- Logs --> EconomyBob --> economy.db
    |
    +-- FTP --> ftp_sync.py --> lokale Kopien
    |
    +-- act_cars.txt --> carservice.py --> carservice.db
    |
    +-- Spielchat <-- streamtochat.py/ahk <-- chat_bridge.py <-- Discord

ini/database/*.db (lokal)
    |
    +-- rsync_push.py (alle 5s, SQLite Backup-API, Delta-Transfer)
    |
    +-- SSH/rsync --> Webserver (scumsaecke.de/dash/includes/database/)
                      --> Dashboard (PHP, Read-Only)
```

---

## 8. Konfigurations-Dateien (`ini/`)

| Datei | Steuert |
|---|---|
| `game-manager.ini` | Game-Manager-Bot (Flags-Export, StreamToChat, Speech) |
| `bot_tracker.ini` | Tracking-Konfiguration (Legacy) |
| `bunker_config.ini` | Bunker-Events |
| `map_event.ini/json` | Map-Events |
| `mapevent_challenge.ini` | Map-Event-Challenges |
| `serveranzeige.ini` | Serveranzeige |
| `statbot.ini` | Statistik-Bot |
| `timer.ini` | Timer-Konfiguration |
| `trader.ini` | Händler-Konfiguration |
| `npczones.ini` | NPC-Zonen |
| `MapTools.ini` | Karten-Konfiguration (Sektoren, Bauzonen) |
| `ServerSettings.ini` | Server-Einstellungen |
| `special_roles.ini` | Spezial-Rollen |
| `team.ini` | Team-Konfiguration |

---

## 9. Hilfsskripte & Tools

| Datei | Zweck |
|---|---|
| `MapTools.py` | Kartenberechnung (Sektoren, Bauzonen, Koordinaten) |
| `ftp_sync.py` | FTP-Synchronisation mit Gameserver |
| `steamcmd.py` | Steam-Version-Monitor (Legacy, nur Discord-Webhook bei neuem Build — Update-Automatisierung jetzt in `UNIX-PC/update-manager/`) |
| `config_changer.py` | Server-Config-Änderungen |
| `npczones_spawner.py` | NPC-Zonen-Spawner |
| `timer.py` | Timer-Verwaltung |
| `purge_files.py` | Purge-Datei-Verwaltung |
| `scripts/build_flag_db.pyw` | Manueller flags.db-Aufbau |
| `scripts/data_to_db.py` | Daten-Import in DBs |
| `tools/sync/rsync_push.py` | DB-Sync zum Dashboard-Webserver (SSH/rsync, alle 5s) |

---

## 10. Deaktiviert / Archiv

| Ordner | Inhalt |
|---|---|
| `xxdeaktiviert/` | Deaktivierte Bots (rcv, quests, trigger, online-manager, start-manager) |
| `xpro_pve/` | PvE-Variante: LogBoB (analog zu EconomyBob) für den ProPVE Server |

---

*Nächste Etappe: Verbindungen & Abhängigkeiten zwischen den Komponenten im Detail.*
