# Bereich: Externe Toolsuiten

## Übersicht

Drei eigenständige Toolsuiten, die als separate Daemons neben dem Haupt-System laufen. Jede hat eigene Config, eigene DB und kommuniziert nur über Dateien oder Webhooks mit dem Rest.

---

## EconomyBob/

**Wirtschafts-Log-Monitor** — liest Economy-Logs vom FTP und postet sie in Discord.

### Architektur
Zwei-Thread-Design:
1. **Poller-Thread:** FTP → SQLite (neue Log-Einträge)
2. **Poster-Thread:** SQLite → Discord-Webhook (ungesendete Einträge)

### Config: `economy_config.ini`

| Abschnitt | Keys |
|---|---|
| `[FTP]` | host, user, password, port (<FTP_PORT>), remote_dir |
| `[Discord]` | webhook (Discord-Webhook-URL) |
| `[General]` | db_path (economy.db), poll_interval (3s), message_limit (1900), rate_limit_delay (2s) |

### Datenbank: `economy.db` (SQLite, WAL-Modus)

| Tabelle | Spalten | Zweck |
|---|---|---|
| economy_logs | hash PK (SHA-256), timestamp, content, sent (0/1) | Log-Einträge |
| ftp_state | filename PK, byte_offset | Lese-Position pro Datei |

**Index:** `idx_sent` auf (sent, timestamp) — schnelles Abrufen ungesendeter Einträge.

### Log-Pipeline
1. FTP: Findet `economy_*.log` per Regex
2. SIZE-Prüfung: Nur neue Bytes herunterladen (REST+RETR)
3. Rotation-Erkennung: Wenn Dateigröße schrumpft → von Anfang lesen
4. Parsing: UTF-16-LE, Split auf `\x0a\x00`, Timestamp-Regex
5. Hash: SHA-256(filename|byte_offset|content) → Deduplizierung
6. Batch-Insert mit `INSERT OR IGNORE`
7. Poster: SELECT unsent, GROUP BY 1900 chars, POST als Discord-Embeds
8. Cleanup: Gesendete Einträge > 2 Tage werden gelöscht

### Filter: `filter.txt`
48 Filtermuster — zeilen die diese Patterns enthalten werden ignoriert (z.B. "Game version", "Bunker is Locked", "Pingperfect").

### Dateien

| Datei | Zugriff |
|---|---|
| `economy.db` + `.db-wal` + `.db-shm` | Lesen + Schreiben |
| `logs/economy_*.log` | Schreiben (lokale Kopie) |
| FTP: `economy_*.log` | Lesen (remote) |

→ Keine Verbindung zum Haupt-System außer Discord-Webhook

---

## PP-Live-Logcrawler/

**Konsolen-Crawler** — liest Live-Logs aus dem PingPerfect-Webpanel via Browser-Automatisierung.

### Architektur
Browser-Automation (Selenium/Playwright) → Log-Parsing → 17+ Kategorien → Dateien/DBs + INI-Export

### Config: `config/config.ini` (300+ Zeilen)

| Abschnitt | Keys |
|---|---|
| `[SERVER]` | server_id, base_url (PingPerfect-Panel) |
| `[AUTH]` | username, password, auto_login |
| `[BROWSER]` | profile_dir, headless, profile_cleanup |
| `[SYSTEM]` | poll/retry/error sleep, heartbeat, health intervals, timeouts, debug |
| `[SCUM SERVER HEALTH MONITORING]` | scum_health_enabled, interval (15s), crash_detection, crash_threshold (15s) |
| `[PARSING]` | time_offset_hours (1), steam_id_pattern |
| `[FILTERS]` | scum_preserve_patterns, js/html/css/custom_patterns |

### Log-Typen (17+)

Jeder Typ als eigener `[LOG_TYPE_xxx]`-Abschnitt mit: enabled, name, keywords, output_file, database, table_name, priority.

| Log-Typ | Ausgabe-Datei | Datenbank |
|---|---|---|
| chat | chat.log | chat.db |
| admin | admin.log | admin.db |
| economy | economy.log | economy.db |
| events | events.log | events.db |
| scum | SCUM.log | SCUM.db |
| login | login.log | login.db |
| gameplay | gameplay.log | gameplay.db |
| quests | quests.log | quests.db |
| violations | violations.log | violations.db |
| famepoints | famepoints.log | famepoints.db |
| kill | kill.log | kill.db |
| loot | loot.log | loot.db |
| vehicle_destruction | vehicle_destruction.log | vehicle_destruction.db |
| sentry | sentry.log | sentry.db |
| base_building_destruction | base_building_destruction.log | base_building_destruction.db |
| ... | ... | ... |

### Health-Monitoring

Parst `[LogSCUM: Global Stats]`-Zeilen:
```
FPS_MAX (FPS_AVG), FPS_MIN | C: Chars (Active), P: Players, Z: Zombies, ...
```

### INI-Export (gelesen vom Haupt-System!)

| Datei | Inhalt | Gelesen von |
|---|---|---|
| `ini/playercount.ini` | Aktuelle Spielerzahl (eine Zahl) | `server_status.py` Cog |
| `ini/serverhealth.ini` | TPS, Characters, Zombies, Vehicles, Items, etc. | `server_status.py` Cog |

**serverhealth.ini Felder:**
timestamp, fps_max, fps_avg, fps_min, fps_count, characters_total/active, prisoners_total/active, zombies_total/active, robots_total/active, structures_total/active, animals_total/active, vehicles, iv_total/active (Items), no_total/active (Nodes)

### Crash-Detection
- Threshold: 15s ohne neue Global-Stats → Crash erkannt
- Auto-Restart konfigurierbar (max 2 Crashes pro 5-Min-Fenster)

→ **Kritische Verbindung:** `server_status.py` Cog ([03_TRACKING_MONITORING.md](03_TRACKING_MONITORING.md)) liest `playercount.ini` und `serverhealth.ini`
→ Chat-Logs in `logs/chat*.log` werden von `link_translator.py` Cog gelesen ([04_SPIELER_IDENTITAET.md](04_SPIELER_IDENTITAET.md))

---

## monitor/

**Bot-Monitoring-System** — überwacht alle Discord-Bots und Konsolen-Prozesse, alarmiert bei Ausfällen.

### Architektur
Discord-Bot + JSON-Bridge + Web-Server + Desktop-GUI — alles in einem Paket.

### Config: `config/settings.yaml`

| Key | Wert |
|---|---|
| alert_channel_id | <CHANNEL_MONITOR_ALERT> |
| check_interval | 60s |
| admin_user_id | <ADMIN_DISCORD_ID> |
| latency.warning_ms | 1000 |
| latency.critical_ms | 2000 |
| maintenance.hours | [3, 9, 15, 21] (UTC) |
| maintenance.duration_minutes | 20 |
| grace_period_minutes | 30 |

### Überwachte Bots (in `bots[]`)
Jeder Bot hat: name, discord_id, script_path, console_title, working_dir, checks[].

**Check-Typen:**

| Typ | Funktion |
|---|---|
| `presence` | Bot online? Alert nach X Min offline |
| `channel_watch` | Einzelner Kanal: Letzte Nachricht älter als X Min? |
| `multi_channel` | Mehrere Kanäle: Alert nur wenn ALLE inaktiv |
| `activity` | Status/Game/Custom-Status tracken (nur Logging) |

### Überwachte Konsolen-Prozesse (`console_watches[]`)
PP_Console_Crawler, EconomyBoB, ftp_sync, streamtochat, mokkamaster, payment_mail_watcher, airdrop, timer, npczones_spawner, LogBoB.exe

### Datenbank: `data/monitor.db` (aiosqlite)

| Tabelle | Spalten | Zweck |
|---|---|---|
| events | id PK, bot_id, bot_name, event_type, channel_id, message_id, details (JSON), timestamp | Event-Historie |
| check_results | id PK, bot_id, bot_name, check_type, status (ok/warning/critical), value, message, timestamp | Check-Ergebnisse |
| alerts | id PK, bot_id, check_type, alert_type, message, timestamp | Gesendete Alerts (Cooldown-Tracking) |
| bot_status | bot_id PK, bot_name, is_online, last_seen, current_status, current_activity, last_message_channel, last_message_time, updated_at | Aktueller Status |

### JSON-Bridge (für externe Apps)

| Datei | Richtung | Inhalt |
|---|---|---|
| `data/status.json` | Lesen | System-Status, Bot-Status, Latenz, Konsolen-Prozesse |
| `data/commands.json` | Schreiben | Befehls-Queue (status, check, start, stop, restart, reload, reboot) |
| `data/command_results.json` | Lesen | Befehlsergebnisse |

### Web-Server: `web_server.py` (Flask)
- Port: 5000
- Auth: "pass1" (Basic Auth, optional)
- Routes: `GET /`, `POST /command/<action>`, `GET /api/status`, `POST /api/command`

### Desktop-GUI: `monitor_gui.py` (CustomTkinter)
- Liest `data/status.json` (lokal oder Netzwerkpfad)
- Refresh: 5s
- Dark Theme mit Farbkodierung (Online=grün, Idle=orange, DND=rot, Offline=grau)
- Buttons: Status, Check, Start, Stop, Restart, Reload

### DM-Befehle (nur Admin)
```
status    — Status aller Bots
check     — Sofort alle Checks ausführen
ping      — Discord-API-Latenz
history <bot> [stunden] — Event-Verlauf
checks <bot> [typ] — Check-Ergebnisse
bots      — Liste überwachter Bots
stop <bot> — Bot beenden
start <bot> — Bot starten
restart <bot> — Bot neustarten
reload    — Config neu laden
reboot    — Reboot-Script ausführen
help      — Hilfe
```

### Singleton-Schutz: `singleton.py`
- PID-basierte Lock-Files in `data/locks/`
- Cross-Platform PID-Prüfung (Windows: tasklist)
- Auto-Release via atexit-Handler

### Prozess-Steuerung
- **Start:** `cmd.exe /c python <script_path>` mit working_dir
- **Stop:** Windows PowerShell `Get-CimInstance` → PID finden → `taskkill`
- **Restart:** Stop + 3s Pause + Start

→ Überwacht alle Bots aus [09_STANDALONE_BOTS.md](09_STANDALONE_BOTS.md) und Prozesse aus [08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md)

---

## Integration ins Gesamtsystem

```
PP-Live-Logcrawler
    ├── ini/playercount.ini ──→ server_status.py Cog (Bot-Presence)
    ├── ini/serverhealth.ini ──→ server_status.py Cog (TPS/Entity-Embed)
    └── logs/chat*.log ──→ link_translator.py Cog (Kartenlinks)

EconomyBob
    └── Discord-Webhook ──→ Economy-Kanal ──→ statbot.py / trader_status.py Cogs

Monitor
    ├── Überwacht alle Discord-Bots (Presence + Channel-Aktivität)
    ├── Überwacht alle Konsolen-Prozesse (Titel-Check)
    ├── Alarmiert in Alert-Channel bei Ausfällen
    └── Steuert Bots remote (Start/Stop/Restart)
```
