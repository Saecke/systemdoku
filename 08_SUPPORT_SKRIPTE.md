# Bereich: Support-Skripte & Hilfsbibliotheken

## Übersicht

Diese Skripte und Module bilden die **Infrastruktur-Schicht** zwischen den AHK-Skripten und den Discord-Bots. Scheduler, FTP-Sync, Payment-Überwachung, Map-Events, Gamechat-Queue und Hilfsbibliotheken.

---

## Prozess-Management

### systemstart.py
**Bot-Launcher** — startet alle Python-Bots als Subprozesse.

- Wird von `start.ahk` aufgerufen
- Liest hardcodierte Skriptliste mit Delay-Angaben (Millisekunden)
- Startet jeden Bot in neuem Konsolenfenster via `cmd.exe /c python`
- 0.5s Pause zwischen Launches

**Gestartete Skripte (in Reihenfolge):**
1. `payment_mail_watcher.py`
2. `tools/sync/rsync_push.py`
3. `bot_voicechannel.py`
4. `bot_titles.py`
5. `bot_infobot.py`
6. `ftp_del.py`
7. `ftp_sync.py`
8. `hook_licence.py`

→ Siehe auch: [07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md) (Startkette)

---

### timer.py
**Zentraler Scheduler** — steuert tägliche Zeitpläne.

- **Config:** `ini/timer.ini` — Abschnitte `[Zeiten]` und `[TimerSettings]`
- **Library:** `schedule` (cron-ähnlich)
- **Adaptiver Sleep:** Berechnet Zeit bis nächsten Job, schläft bis 2 Min davor

**Geplante Jobs:**

| Job | Funktion |
|---|---|
| `neustart_3m_warnung` | 3-Min-Restart-Warnung → `gamechat/`, `log/`, `log_public/` |
| `neustart_30s_warnung` | 30-Sek-Restart-Warnung → `gamechat/` |
| `neustart_woechentlich_warnung` | Wöchentlicher Root-Reboot (Samstags) |
| `purge_checker_ausfuehren` | Startet `purge_checker.ahk` |
| `mapevent_ausfuehren` | Startet `mapevent.py` |
| `handelsbericht_ausfuehren` | Startet `handelsbericht.py` |
| `daily_message_ausfuehren` | Startet `hook_daily.pyw` |
| `daily_event_leaderboard_ausfuehren` | Startet `hook__mapevent_leaderboard.pyw` |
| `get_server_data` | Schreibt `#ListSpawnedVehicles` + `#DumpAllSquadsInfoList` in `gamechat/` |

**Lock-Prüfungen:** Prüft `events/Eventsperre.txt` und `ini/banksperre.txt` vor Updates.
**Speech-Dateien:** `speech/restart_3m.txt`, `speech/restart_30s.txt`, `speech/root-reboot.txt` (zufällige Nachrichtenauswahl)

→ Siehe auch: [07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md) (eventmanager, purge_checker)

---

## Datensynchronisation

### ftp_sync.py
**FTP-Log-Synchronisation** — lädt neue Serverlogs vom FTP-Server.

- **Config:** `ini/ftp.ini` (host, port, user, password, remote_folder, local_folder)
- **Adaptive Polling:** 2s (aktiv) bis 10s (idle), +1s pro Leerlauf-Zyklus
- **Skip:** Ignoriert Dateien mit "economy" im Namen
- **Offset-Tracking:** Merkt sich Byte-Position pro Datei, lädt nur neue Inhalte
- **Log-Rotation:** Erkennt Dateigröße-Rückgang und Timestamp-Änderungen
- **Reboot-Erkennung:** Wenn Timestamp sich um >5 Einheiten ändert → startet `reboot.ahk`
- **Auto-Restart:** Überwacht eigene mtime, startet bei Änderung neu
- **Ausgabe:** `botlog/bot_ftp_work_*.log`

→ Daten landen in `serverlogs/download/` → verarbeitet von `log_separator.ahk` ([07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md))

---

### config_changer.py
**ServerSettings-Synchronisation** — steuert Event-Toggle per FTP-Upload.

- **Config:** `ini/ftp.ini` (Credentials + `serversettings_folder`)
- **Lokal:** `ini/ServerSettings.ini`
- **Schlüsselfeld:** `scum.AllowEvents` (0 oder 1)

**Zeitplan:**

| Uhrzeit | Aktion |
|---|---|
| 13:58 | ServerSettings.ini herunterladen |
| 14:00 | Upload mit `AllowEvents=1` |
| 01:58 | ServerSettings.ini herunterladen |
| 02:00 | Upload mit `AllowEvents=0` |

---

### streamtochat.py
**Gamechat-Queue-Processor** — wandelt Text-Dateien in Spielchat-Befehle.

- **Überwacht:** `gamechat/*.txt` (via watchdog)
- **Erzeugt:** `gamechat/*.tochat` (wird von `streamtochat.ahk` gelesen)
- **Timing-Syntax:** `<<<DELAY>>>command` (Pre-Delay) / `command<<<DELAY>>>` (Post-Delay)
  - Beispiel: `<<<5>>>say hello<<<2>>>` = 5s warten, senden, 2s warten
- **Encoding:** Auto-Detection via chardet
- **Threading:** Observer-Thread + Processing-Queue

→ Siehe auch: [07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md) (streamtochat.ahk liest .tochat-Dateien)

---

## Payment & Wirtschaft

### payment_mail_watcher.py
**E-Mail-Payment-Monitor** — überwacht IMAP-Postfach für Zahlungseingänge.

- **Config:** `ini/pingperfect/mail_watcher.ini`
- **State:** `ini/pingperfect/mail_watcher_state.json` (letzte UID pro Mailbox)

**Config-Abschnitte:**

| Abschnitt | Inhalt |
|---|---|
| `[imap]` | host, port, username, password, folder, tls, mark_seen, poll_interval_sec |
| `[labels]` | Feldnamen für Parsing (date, payer, method, amount) |
| `[from_contains]` / `[subject_contains]` | E-Mail-Filter |
| `[behavior]` | delete_after_store, delete_duplicates, use_state |

**Datenbank:** `ini/database/payment.db`
**Tabelle:** `Payments`

| Spalte | Typ |
|---|---|
| id | INTEGER PRIMARY KEY |
| payer_name | TEXT |
| method | TEXT |
| amount_raw | TEXT |
| amount_eur_cents | INTEGER |
| transaction_id | TEXT (dedupliziert) |
| pay_date | TEXT |
| claimed | INTEGER (0/1) |
| logtime | TEXT |

**Parsing:** HTML/Text-Body → Regex für Key-Value-Paare → Fallback auf Tabellen und Freitext
**CLI-Befehle:** `--list-folders`, `--peek`, `--tx-show/claim/unclaim`, `--reset-state`, `--once`, `--dry-run`

→ Wird von `donations.py` Cog gelesen ([02_WIRTSCHAFT.md](02_WIRTSCHAFT.md))

---

## Events & Karten

### mapevent.py
**Map-Event-Orchestrator** — spawnt NPCs/Zombies, trackt Teilnahme, loggt Ergebnisse.

- **Config:** `ini/mapevent.ini` (global) + `events/mapevents/[event_name]/config.ini` (pro Event)
- **Koordinaten:** `events/mapevents/[event_name]/coords.txt` oder `center.txt`
- **Spielerliste:** `ini/players.txt`
- **Lock:** `events/Eventsperre.txt` + pro Event `BLOCK_EVENT.txt`

**Config-Parameter (pro Event):**

| Key | Beschreibung |
|---|---|
| event_name, machine_name | Event-Identifikation |
| thumbnail_url | Bild für Discord-Embed |
| use_center_coordinates | true = center.txt, false = coords.txt |
| spawn_delay, start_delay | Timing |
| event_run_time | Gesamtdauer |
| max_distance_player_event | Proximity-Check für Teilnahme |
| NPC_SPAWN_CMD, ZOMBIE_SPAWN_CMD, NPC_SPAWN_RARE_CMD | Spawn-Befehle |
| announcement_schedule | Ankündigungszeitpunkte (20,15,10,5,2,1 Min vor Ende) |

**Ablauf:**
1. Prüft Locks (Eventsperre, Banksperre, BLOCK_EVENT)
2. Wählt zufälliges Event
3. Baut Spawn-Befehle aus Koordinaten
4. Startet Countdown-Ankündigungen
5. Trackt Spieler-Proximity für Teilnahme
6. Speichert Ergebnisse als JSON + in `map_event_contributions.db`

**Ausgabe:**
- `gamechat/*.txt` (Spawn-Befehle + Ankündigungen)
- `eventlogs/*.log` (Event-Details)
- `log_public/*.txt` (öffentliche Logs)
- `events/mapevents/[event]/results_*.json` (Ergebnisse)

→ Wird von `timer.py` gestartet, nutzt `streamtochat.py` als Queue
→ DB wird von `map_event_challenges.py` Cog gelesen ([05_EVENTS_GAMEPLAY.md](05_EVENTS_GAMEPLAY.md))

---

### npczones_spawner.py
**NPC-Zonen-Spawner** — automatisiertes NPC-Spawning an definierten Zonen.

- **Config:** `ini/npczones.ini` (global) + `events/npczones/[zone]/config.ini` (pro Zone)
- **Koordinaten:** `events/npczones/[zone]/coords.txt` oder `center.txt`
- **PID-Check:** `_pid/sys_online.pid` (muss <30s alt sein)
- **Block:** `ini/npczones_block.txt` (deaktiviert Spawner)

**Features:**
- Zonen-Rotation mit Wiederholungsvermeidung
- Zeitfenster-Locks (z.B. nur 10:00–22:00)
- Active-Window-Prüfung
- Spieler-Proximity-Tracking
- State wird in `ini/npczones.ini` persistiert

→ Status wird von `npczones.py` Cog gelesen und angezeigt ([05_EVENTS_GAMEPLAY.md](05_EVENTS_GAMEPLAY.md))

---

## Monitoring

### steamcmd.py (Windows, Legacy)
**Steam-Version-Monitor** — erkennt Server-Updates via Steam API.

- **Config:** `ini/serverversion.ini` (webhook_url, interval, appid, mention_roles)
- **State:** `ini/serverversion.ini` [BUILD] section (steam_buildid)
- **API:** `https://api.steamcmd.net/v1/info/{appid}`
- **Aktion bei Update:** Discord-Webhook mit alter → neuer BuildID + Rollen-Mentions

> **Hinweis:** Die vollständige Update-Automatisierung (Poller + PP-Panel-Steuerung + Ingame-Warnungen) läuft jetzt auf dem i3 unter `UNIX-PC/update-manager/`. Dieses Skript auf dem Bot-PC ist die ältere, reine Webhook-Variante. → Siehe [10_EXTERNE_TOOLS.md](10_EXTERNE_TOOLS.md#update-manager-unix-pcupdate-manager--watchdog-watchdog)

---

### purge_files.py
**Log-Bereinigung** — löscht alte `.log.txt`-Dateien rekursiv.

- **Extension:** `.log.txt`
- **Max Alter:** 7 Tage
- **Dry-Run:** Möglich (`DRY_RUN = True`)
- **Ausgabe:** Zusammenfassung (gelöschte Dateien, freigegebener Speicher)

---

## Hilfsbibliotheken (_modules/)

### loader.py
**Cog-Loader** — lädt alle Cogs in den Haupt-Bot.

- Funktion `bind_setup(bot)` → gibt async setup-Funktion zurück
- Iteriert über `cfg.startup_cogs` Liste
- Druckt Extension-Übersicht aus `cfg.extensions_meta`
- Trackt geladene/fehlgeschlagene Cogs als Bot-Attribute

→ Genutzt von `bot_infobot.py` beim Start

---

### MapTools.py
**Karten-Hilfsbibliothek** — Koordinatenberechnung, Zonen-Prüfung.

- **Config:** `ini/MapTools.ini` (Grid-Koordinaten, Farbwerte, ImageMagick-Pfad)
- **Karten:** `Maps/BauVerbotsZonen`, `Maps/AirDropZonen` (Bilddateien)

**Funktionen:**

| Funktion | Beschreibung |
|---|---|
| `istPositionInBauverbotszone(x, y)` | Prüft ob Koordinate in Bauverbotszone (Pixel-Farbcheck) |
| `istAirDropPositionErlaubt(x, y)` | Prüft ob Airdrop-Position erlaubt |
| `getSektor(x, y, numberpad)` | Konvertiert Koordinaten zu Sektor (z.B. "B1" oder "B1#7") |

**Abhängigkeit:** ImageMagick CLI für Bild-Pixel-Analyse.

→ Genutzt von `trackingstation.py` Cog ([03_TRACKING_MONITORING.md](03_TRACKING_MONITORING.md)) für Bauverbotszonen-Check

---

### bunker_func.py
**Bunker-Status-Tracking** — parst Bunker-Lock-Logs, persistiert Zustände.

- **Config:** `ini/bunker_config.ini` (Bunker-State-Persistierung)
- **Erlaubte Bunker:** D1, A3, C4, A1 (hardcoded)
- **Zeitfenster:** BUNKER_CARD_ACTIVATION_TIME=2h, BUNKER_CYCLE_ACTIVATION_TIME=2h

**Log-Parsing (Regex):**
- `"X Bunker is Locked. Locked Xh Xm Xs ago, next Activation in Xh Xm Xs"`
- `"X Bunker Activated via Keycard X hours ago"`
- `"X Bunker is Active. Activated X hours ago via Keycard"`
- `"X Bunker Deactivated"`

**BunkerActivationData-Klasse:**
- Properties: id, status, activation_time_unix_cycle, activation_time_unix_card, withkey
- Methoden: isActive(), isActiveWithKey(), getActivationTime()
- Persistierung: save() / load() zu/von INI

→ Genutzt von `bunker.py` Cog ([05_EVENTS_GAMEPLAY.md](05_EVENTS_GAMEPLAY.md))

---

### announcements.py
**Ankündigungs-Modul** — Stub/minimales Modul. Keine substantielle Logik erkennbar.

---

## Dashboard-Sync

### rsync_push.py (`tools/sync/`)
**DB-Sync zum Webserver** — hält das Dashboard aktuell.

- **Config:** `tools/sync/sync_config.json`
- **Intervall:** 5 Sekunden (konfigurierbar via `interval_seconds`)
- **Methode:** SQLite Backup-API → konsistenter Snapshot → rsync-over-SSH
- **Transport:** rsync (bevorzugt) → scp (Fallback) via WSL
- **SSH:** ControlPersist=7200s (Connection-Reuse über 2h)
- **Change-Detection:** Prüft mtime + size inkl. WAL/SHM/Journal — synct nur geänderte DBs
- **Bulk-Mode:** Ein rsync-Aufruf für das gesamte Snapshot-Verzeichnis
- **State:** `.sync_state.json` im Snapshot-Ordner (Signatur pro DB)

**Gesynchte Datenbanken (10 Stück):**

| DB | Quelle |
|---|---|
| userdata.db | identity, game_events, squad |
| persistent_stats.db | bot_titles |
| lottery.db | bot_lottery |
| event_stats.db | events_stats Cog |
| statbot.db | statbot Cog |
| userstats.db | voice_rewards Cog |
| userhistory.db | history Cog |
| flags.db | trackingstation Cog |
| carservice.db | carservice Cog |
| historical_stats.db | handelsbericht, bot_titles |

**Ziel:** `<WEBSERVER>:/www/htdocs/.../dash/includes/database/`

→ Gestartet von `systemstart.py` ([08_SUPPORT_SKRIPTE.md](#systemstartpy))
→ Dashboard liest die DBs read-only ([11_DASHBOARD.md](11_DASHBOARD.md))

---

## Dateibasierte Kommunikation

Alle Skripte kommunizieren über **Dateien** — keine direkte Prozesskommunikation:

```
gamechat/*.txt  ← timer.py, mapevent.py, npczones_spawner.py, chat_bridge.py
    ↓
streamtochat.py → gamechat/*.tochat
    ↓
streamtochat.ahk → SCUM-Fenster (Tastatur-Simulation)

serverlogs/download/ ← ftp_sync.py (vom FTP-Server)
    ↓
log_separator.ahk → serverlogs/chat-*/, serverlogs/economy/, etc.
    ↓
log_bridge.py / inselkom.py → Discord-Kanäle

events/Eventsperre.txt ← eventmanager.ahk, mapevent.py
    → gelesen von timer.py, bank.py, chat_bridge.py (Sperr-Prüfung)

_pid/sys_online.pid ← Heartbeat (mtime-basiert)
    → gelesen von npczones_spawner.py, npczones.py Cog
```
