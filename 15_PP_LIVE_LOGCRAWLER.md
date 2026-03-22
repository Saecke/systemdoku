# PP-Live-Logcrawler — Detaildokumentation

## Übersicht

Der PP-Live-Logcrawler ist ein eigenständiges, multi-threaded Python-System, das Live-Logs von der PingPerfect-Webkonsole liest, in Echtzeit verarbeitet und für das Hauptsystem bereitstellt. Er nutzt Selenium-Browser-Automatisierung + XHR-Streaming für die Datenerfassung.

**Hauptskript:** `PP_Console_Crawler.py` (3.466 Zeilen)
**Abhängigkeiten:** Selenium, httpx, sqlite3, rich, psutil, matplotlib, tkinter

---

## Architektur

```
PingPerfect Game Panel (HTTPS)
    ↓ Selenium Chrome (headless)
    ↓ Login + Cookie/Token-Extraktion
    ↓
XHR POST: GetGameLogsRead/{server_id}
    ↓ Liste verfügbarer Log-Dateien
    ↓
XHR GET: RemoteTail/{server_id}?fullName={log}
    ↓ Proxy-URL oder Direct-Content
    ↓
Chunked HTTP Streaming (pro Log-Typ eigener Thread)
    ↓
Parse → Filter → Deduplizieren → Speichern
    ↓                    ↓                ↓
Log-Dateien        SQLite-DBs      INI-Export
(logs/*.log)       (logs/*.db)     (ini/playercount.ini
                                    ini/serverhealth.ini)
```

---

## Threading-Modell

| Thread | Funktion |
|---|---|
| **Main** | Config laden, Browser starten, Auth, Log-Discovery, Health-Status |
| **Log-Monitor** (1 pro Typ) | Streamt + parst + filtert + schreibt für einen Log-Typ |
| **Crash-Detection-Daemon** | Prüft alle 5s ob SCUM noch Global Stats liefert |
| **Startup-Watchdog** | Erzwingt Neustart wenn innerhalb 30s keine Stats kommen |
| **Config-Reload** | Überwacht config.ini auf Änderungen (Hash-basiert) |

---

## Konfiguration: `config/config.ini` (311 Zeilen)

### Grundeinstellungen

| Abschnitt | Key | Wert | Beschreibung |
|---|---|---|---|
| `[SERVER]` | server_id | `<SERVER_ID>` | PingPerfect Server-ID |
| `[SERVER]` | base_url | `https://gamepanel.pingperfect.com` | Panel-URL |
| `[AUTH]` | username | `<PP_USERNAME>` | Login |
| `[AUTH]` | password | `<PP_PASSWORD>` | Passwort |
| `[AUTH]` | auto_login | true | Automatisch einloggen |
| `[BROWSER]` | headless | true | Kein Browser-Fenster |
| `[BROWSER]` | profile_dir | ./profile | Chrome-Profil (Cookies) |
| `[BROWSER]` | profile_cleanup | true | Profil vor Start löschen |

### Timing & Netzwerk

| Key | Wert | Beschreibung |
|---|---|---|
| sleep_poll_seconds | 3 | Polling-Intervall (idle) |
| sleep_retry_seconds | 5 | Retry-Intervall |
| sleep_error_seconds | 10 | Warten nach Fehler |
| heartbeat_interval | 30 | Heartbeat-Meldung |
| http_connect_timeout | 10.0 | Verbindungs-Timeout |
| http_read_timeout | 60.0 | Lese-Timeout |

### Health-Monitoring

| Key | Wert | Beschreibung |
|---|---|---|
| scum_health_enabled | true | Health-Monitoring aktiv |
| scum_health_interval | 15 | Max. Sekunden zwischen Global Stats |
| scum_crash_detection | true | Crash-Erkennung aktiv |
| scum_crash_threshold | 15.0 | Crash-Schwelle (Sekunden) |
| health_report_interval | 60 | Report-Intervall |
| health_min_samples | 3 | Min. Samples für Analyse |

### Restart-Tracking

| Key | Wert | Beschreibung |
|---|---|---|
| restart_on_first_crash | false | Beim ersten Crash neustarten |
| restart_max_crashes_per_window | 2 | Max. Crashes im Zeitfenster |
| restart_time_window_minutes | 5 | Zeitfenster für Crash-Zählung |

---

## 17+ Log-Typen

Jeder Typ als eigener `[LOG_TYPE_*]`-Abschnitt mit: enabled, name, keywords, output_file, database, table_name, priority.

| Log-Typ | Enabled | Datei | Beschreibung |
|---|---|---|---|
| CHAT | ja | chat.log | Spieler-Chat |
| ECONOMY | ja | economy.log | Wirtschafts-Transaktionen |
| SCUM | ja | SCUM.log | Server-Performance (Global Stats) |
| LOGIN | ja | login.log | Login/Logout |
| GAMEPLAY | ja | gameplay.log | Gameplay-Events |
| QUESTS | ja | quests.log | Quest-Abschlüsse |
| VIOLATIONS | ja | violations.log | Regelverstöße |
| KILL | ja | kill.log | Spieler-Kills |
| CHEST_OWNERSHIP | ja | chest_ownership.log | Truhen-Besitzwechsel |
| VEHICLE_DESTRUCTION | ja | vehicle_destruction.log | Fahrzeug-Zerstörung |
| EVENT_KILL | ja | event_kill.log | Event-Kills |
| ADMIN | nein | admin.log | Admin-Befehle |
| EVENTS | nein | events.log | Server-Events |
| FAMEPOINTS | nein | famepoints.log | Fame-Punkte |
| LOOT | nein | loot.log | Loot-Drops |
| ARMOR_ABSORPTION | nein | armor_absorption.log | Rüstungs-Absorption |
| SERVER_NOTIFICATIONS | nein | server_notifications.log | Server-Meldungen |
| RAID_PROTECTION | nein | raid_protection.log | Raid-Schutz |
| SENTRY | nein | sentry.log | Sentry-System |
| BASE_BUILDING_DESTRUCTION | nein | base_building_destruction.log | Basis-Zerstörung |

---

## Authentifizierungs-Flow

```
1. Selenium öffnet Chrome (headless, mit Profil)
2. Navigiert zu gamepanel.pingperfect.com
3. Login-Seite erkannt? → Auto-Submit Credentials
   Sonst: Manueller Login-Prompt
4. get_auth_context(driver, server_id):
   - Cookies aus Browser extrahieren
   - User-Agent + Headers lesen
   - Verification-Token aus DOM holen
5. httpx-Client mit Cookies/Headers initialisieren
6. Bei Reconnect: get_auth_context() erneut (Token-Refresh)
```

---

## XHR-Streaming-Pipeline

### Endpunkt-Kette

```
POST /Service/LogViewer/GetGameLogsRead/{server_id}
    → Liste: Name, FullName, Size, LastWriteTimeUtc

GET /Service/LogViewer/RemoteTail/{server_id}?fullName={encoded}
    → JSON mit 'url' (Proxy-Endpoint) oder direktem Content

GET {proxy_url}
    → Chunked HTML mit <pre>-Tags oder Plain-Text
    → Streaming bis Verbindung schließt (tail-like)
```

### Pro-Typ-Thread-Verhalten

- **Endlos-Streaming:** Stoppt nie (bis thread_stop_event)
- **Proxy-URL-Persistenz:** Wiederverwendung über Requests
- **Chunked Reading:** Verarbeitet Daten Chunk für Chunk
- **Zeilen-Buffering:** Sammelt vollständige Zeilen vor Verarbeitung
- **Deduplizierung:** MD5-Hash-Check gegen `cache.db`
- **Heartbeat:** Alle 30s wenn keine Daten
- **Auto-Reconnect:** Exponentielles Backoff (5s → 10s → 20s → 40s → 60s, +Jitter)
- **Trigger:** Nach 5 aufeinanderfolgenden Fehlern

---

## SCUM Server Health Monitoring

### Global Stats Parsing

**Eingabe (alle ~5 Sekunden):**
```
[2026.03.22-00.32.13:756][572]LogSCUM: Global Stats:  35.7ms ( 28.0FPS),  41.1ms ( 24.4FPS),  87.2ms ( 11.5FPS) |
C: 680 (646), P:  57 ( 57), Z: 544 (521), R:   0 (  0), S:  23 ( 23), A:  24 ( 24), V: 175 | IV: 7883 (8916) | NO: 12983 (12983)
```

**Extrahierte Metriken:**

| Metrik | Beispiel | Beschreibung |
|---|---|---|
| fps_max / fps_avg / fps_min | 28.0 / 24.4 / 11.5 | Server-Framerate (3 Samples) |
| characters_total / active | 680 / 646 | Alle Charaktere |
| prisoners_total / active | 57 / 57 | Spieler online |
| zombies_total / active | 544 / 521 | Zombies |
| robots_total / active | 0 / 0 | Roboter |
| structures_total / active | 23 / 23 | Strukturen |
| animals_total / active | 24 / 24 | Tiere |
| vehicles | 175 | Fahrzeuge |
| iv_total / active | 7883 / 8916 | Inventar-Items |
| no_total / active | 12983 / 12983 | Netzwerk-Objekte |

### Crash-Detection-Algorithmus

```
SCUM postet Global Stats alle ~5 Sekunden (erwartet)

Arming: Erst nach 3 aufeinanderfolgenden Stats aktiv

if zeit_seit_letzten_stats > 15s AND armed:
    → CRASH erkannt
    → Diagnostik loggen
    → Crash zur History hinzufügen
    → Crashes im 5-Min-Fenster zählen
    → Wenn > 2 Crashes: trigger_external_restart()

if zeit_seit_letzten_stats > 60s:
    → SUSTAINED FREEZE → Force Hard Reboot

if stats_wieder_da AND crash_detected:
    → Recovery, Crash-Flag zurücksetzen
```

### Startup-Watchdog (30s Grace-Window)

```
1. Stream-Threads starten
2. Watchdog armed mit aktuellem Timestamp
3. Pollt jede Sekunde:
   - first_stat_seen? → Startup erfolgreich, Watchdog beendet
   - 30s abgelaufen? → Force Restart via trigger_external_restart()
```

---

## Output-Dateien

### Log-Dateien (Separate Mode)
```
logs/
├── chat.log
├── economy.log
├── SCUM.log
├── login.log
├── gameplay.log
├── quests.log
├── kill.log
└── ...
```

### Datenbanken (Optional)
```
logs/
├── chat.db (Tabelle: chat_logs)
├── economy.db (Tabelle: economy_logs)
├── cache.db (Deduplizierung: MD5-Hash + Grace-Period)
└── FULL_LIVE_LOGS.db (Live-Mirror aller Typen)
```

**DB-Schema pro Typ:**
| Spalte | Typ | Beschreibung |
|---|---|---|
| id | INTEGER PK | Auto-Increment |
| timestamp | TEXT | Log-Zeitstempel |
| player_name | TEXT | Extrahierter Spielername |
| message | TEXT | Nachrichteninhalt |
| raw_content | TEXT | Original-Zeile |
| session_id | TEXT | Session-ID |
| line_num | INTEGER | Zeilennummer |
| log_level | TEXT | Log-Level |
| source | TEXT | Quelldatei |
| created_at | DATETIME | Einfüge-Zeitpunkt |

### INI-Export (gelesen vom Hauptsystem!)

| Datei | Inhalt | Aktualisierung | Gelesen von |
|---|---|---|---|
| `ini/playercount.ini` | Aktive Spielerzahl (eine Zahl) | Bei jedem Global Stats | `server_status.py` Cog |
| `ini/serverhealth.ini` | TPS, Entities, Timestamps | Alle 60 Sekunden | `server_status.py` Cog |

**playercount.ini:** Einfach nur `57` (aktuelle Spielerzahl)

**serverhealth.ini:**
```ini
timestamp=2026.03.22-00.32.13:756
fps_max=28.0
fps_avg=24.4
fps_min=11.5
characters_total=680
characters_active=646
prisoners_total=57
prisoners_active=57
zombies_total=544
zombies_active=521
vehicles=175
iv_total=7883
no_total=12983
```

### Archiv-System (täglich)
```
archive/
├── 2026-01-21/
│   ├── .archived.ok (Marker)
│   ├── chat_1768960916822.log
│   └── economy_1768960916822.log
├── 2026-03-22/
│   └── ...
```

---

## Filter-System

### Noise-Filter (Blacklist)
100+ Patterns die aus allen Logs gefiltert werden:
- HTML: `DOCTYPE`, `<html>`, `<head>`, `<body>`
- CSS: `font-family`, `color`, `margin`
- Custom: Game-Version-Meldungen, Loot-Tree-Fehler, etc.

### Preserve-Patterns (SCUM.log Whitelist)
Diese Patterns werden **nie** gefiltert:
```
LogSCUM: Global Stats:, FPS, TPS, Characters, Prisoners,
C:, P:, Z:, R:, S:, A:, V:, IV:, NO:
```

### Timestamp-Korrektur
- `time_offset_hours: 1` — wird auf alle Timestamps addiert
- `scum_time_offset_hours: 1` — separater Offset für SCUM.log

---

## Error-Handling & Recovery

| Fehler | Reaktion |
|---|---|
| XHR Timeout | Retry nach `sleep_retry_seconds` (5s) |
| Connection Reset (10054) | Freundliche Meldung + Auto-Reconnect |
| 5 aufeinanderfolgende Fehler | Exponentielles Backoff (5→10→20→40→60s +Jitter) |
| HTTP 404 | Log-Meldung, Error-Timeout abwarten |
| Browser-Absturz | Sofort `trigger_external_restart()` |
| SCUM Crash (15s keine Stats) | Crash loggen, History tracken |
| >2 Crashes in 5 Min | Externer Restart |
| 60s Sustained Freeze | Force Hard Reboot |
| Driver-Build-Fehler | 5 Retries mit Backoff, Chrome cleanup |

---

## console_restart.py — Restart-Manager

Wird bei Crashes aufgerufen:
1. Wartet 10s auf graceful Shutdown
2. Killt verbleibende Prozesse (Script + Chrome/Chromium/Edge)
3. Wartet 3s
4. Startet Script neu via `subprocess.Popen`

---

## GUI-Tools (tools/)

| Tool | Zweck |
|---|---|
| `config_gui.py` | Moderne tkinter-Config-UI (Dark Theme, Tabs) |
| `config_ui.py` | Terminal-basierte Config-UI |
| `health_gui.py` | Health-Dashboard mit matplotlib-Charts |
| `health_gui_simple.py` | Vereinfachte Health-GUI |
| `start_health_gui.bat` | Batch-Starter |

### Health-GUI Farbkodierung
- 🟢 Exzellent: TPS ≥ 30
- 🟡 Gut: TPS ≥ 20
- 🟠 Fair: TPS ≥ 15
- 🔴 Schlecht: TPS < 15

---

## Verbindungen zum Hauptsystem

```
PP-Live-Logcrawler
    │
    ├── ini/playercount.ini ──→ server_status.py Cog (Bot-Presence + Embed)
    ├── ini/serverhealth.ini ──→ server_status.py Cog (TPS/Entity-Anzeige)
    │
    ├── logs/chat*.log ──→ link_translator.py Cog (Koordinaten → Kartenlinks)
    ├── logs/chat.log ──→ bot_game-manager.py (RCV-Verifikation)
    ├── logs/login.log ──→ bot_game-manager.py (Login-Erkennung)
    ├── logs/SCUM.log ──→ bot_game-manager.py (Playercount-Tail)
    │
    └── logs/*.log ──→ llm_admin (Log-Monitor via SMB)
```

→ Siehe auch: [03_TRACKING_MONITORING.md](03_TRACKING_MONITORING.md) (server_status.py)
→ Siehe auch: [09_STANDALONE_BOTS.md](09_STANDALONE_BOTS.md) (bot_game-manager.py)
→ Siehe auch: [13_LLM_ADMIN.md](13_LLM_ADMIN.md) (Log-Monitor)

---

## Ressourcen-Verbrauch

| Ressource | Geschätzt |
|---|---|
| RAM (10 Log-Typen + Browser) | 300–600 MB |
| Disk (pro Woche) | 500+ MB |
| Netzwerk | Variabel (Streaming, Chunk-basiert) |
| CPU | Niedrig (I/O-bound, Polling) |
