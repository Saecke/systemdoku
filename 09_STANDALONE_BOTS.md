# Bereich: Standalone-Bots

## Übersicht

Jeder Standalone-Bot ist ein **eigenständiger Python-Prozess** mit eigenem Discord-Token. Fällt einer aus, laufen alle anderen weiter. Gestartet werden sie von `systemstart.py` (→ [08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md)).

---

## bot_squadmanager.py
**Squad-Channel-Verwaltung** — erstellt/löscht automatisch Squad-Kanäle und Voice-Channels.

### Config
- `ini/squadmanager/config.ini` + `ini/system.ini`
- `[Misc]`: OLD_SQUADS_HOURS, OLD_MESSAGE_HOURS, MAX_CHANNELS_PER_CATEGORY, ACTIVE_WINDOW_HOURS, MIN_LINKED_MEMBERS, MIN_ACTIVE_MEMBERS

### Datenbank: `ini/squadmanager/squadmanager_database.db` (aiosqlite)

| Tabelle | Spalten | Zweck |
|---|---|---|
| squad_channels | squad_id PK, channel_id, last_activity, voice_channel_id, member_hash | Squad↔Channel-Zuordnung |
| squad_members | discord_id PK, squad_id, steam_id, player_name | Mitglieder-Sync |
| voice_guests | squad_id, member_id, expires_at | Temporäre Voice-Gäste |
| voice_guests_history | squad_id, member_id, expired_at, duration_seconds | Gäste-Archiv |
| squad_voice_channels | voice_channel_id PK, squad_id, created_at, empty_since | Voice-Channel-Tracking |
| orphaned_channels | channel_id PK | Channels zur Löschung vorgemerkt |
| channel_missing | channel_id PK, first_missing | Grace-Period für fehlende Channels |
| channel_warnings | channel_id PK, last_warning | Inaktivitäts-Warnungen |
| bot_meta | key, value | Metadaten-Store |

### Tasks

| Task | Intervall | Funktion |
|---|---|---|
| manage_squad_channels | 6 Min | Haupt-Lifecycle: Channels erstellen/updaten/löschen |
| cleanup_database | 13 Min | PRAGMA optimize |

### Verhalten
- Synchronisiert Mitglieder aus `userdata.db` (Haupt-DB, Read-Only)
- Erstellt Text- + Voice-Channels pro Squad mit passenden Permissions
- Grace-Period vor Channel-Löschung (channel_missing)
- Retry-Mechanismus: `with_retry()` — 3 Versuche, exponentielles Backoff (1–8s)
- Jitter auf allen Operationen gegen Race-Conditions

→ Siehe auch: `squad.py` Cog ([06_LOGS_KOMMUNIKATION.md](06_LOGS_KOMMUNIKATION.md)) — liest/schreibt `squaddata`

---

## bot_titles.py
**Skill-basierte Discord-Rollen** — lädt Game-DB vom FTP, erkennt Skill-Meilensteine, vergibt Rollen.

### Config
- `ini/ftp.ini` — FTP-Credentials + `scum_db_folder`
- `ini/system.ini` — Token, Bot-Name

### Datenfluss

```
FTP-Server (SCUM.db-backup)
    ↓ download_db()
ini/database/scum_working.db (temporär)
    ↓ parse + vergleiche
ini/database/titles_work.db (Tracking)
    ↓ archiviere
ini/database/persistent_stats.db (historisch)
    ↓ steam_id → discord_id via userdata.db
Discord-Rollen vergeben
```

### Datenbank: `ini/database/titles_work.db`

| Tabelle | Spalten | Zweck |
|---|---|---|
| awarded_skills | steam_id, skill_name, discord_role_id, awarded_at | Bereits vergebene Skills |
| last_update | event PK, updated_at | Letzter Update-Zeitpunkt |

### Datenbank: `ini/database/persistent_stats.db` (Archiv)

| Tabelle | Spalten | Zweck |
|---|---|---|
| survival_stats | steam_id PK, level, exp, 100+ Werte | Überlebensstatistiken |
| prisoner_bases | user_profile_id PK, BaseStrength/Dexterity/Constitution/Intelligence | Basisattribute |
| fishing_stats | steam_id PK, dynamische Spalten | Angel-Statistiken |
| events_stats | steam_id PK, event_name, completion_time, ... | Event-Ergebnisse |
| skills | steam_id, name, level, experience, xml | Skill-Level + XP |
| event_rankings_cached | steam_id PK, ranking_name, rank | Event-Rankings |
| vehicle_locks | vehicle_entity_id, user_profile_id, vehicle_class, lock_item_class, source | Fahrzeugschlösser |
| item_history | item_name, date, sold, purchased, avg_price | Item-Handelshistorie |

### Skill-Rollen-Mapping

| Kategorie | Skills |
|---|---|
| Kampf | Boxing, MeleeWeapons, Archery, Handgun, Rifles |
| Mobilität | Running, Endurance |
| Stealth | Thievery, Demolition, Motorcycle, Aviation, Stealth, Driving |
| Survival | Awareness, Engineering, Camouflage, Sniping, Survival, Medical, Farming, Cooking |

### Kombi-Rollen
- **Soldier** = bestimmte Kampf-Skills kombiniert
- **Scout** = bestimmte Mobilitäts-Skills
- **Operative** = bestimmte Stealth-Skills
- **Survivor** = bestimmte Survival-Skills

### Spezial
- `FAME_ROLE_ID = <ROLE_FAME>` — wird bei hoher Aktivität vergeben
- FTP-Download mit Rate-Limiting
- Logs: `botlog/discord_titelvergabe_log_*.log`

→ Befüllt `persistent_stats.db` — wird von `char.py`, `fishing.py`, `awards.py`, `titles.py` Cogs gelesen ([05_EVENTS_GAMEPLAY.md](05_EVENTS_GAMEPLAY.md))

---

## bot_mapevent_challenge.py
**Community-Challenges** — verfolgt Quests/Trades/Lockpicks und triggert Events bei Zielerreichung.

### Config: `ini/mapevent_challenge.ini`

| Abschnitt | Keys |
|---|---|
| `[bot]` | token, logserver_guild_id, public_guild_id |
| `[channels]` | economy_channel_ids, gameplay_channel_ids, quests_channel_ids, embed_channel_id |
| `[timing]` | active_start/end_hour, random_delay_min/max, embed_update_interval, daily_reset_hour |
| `[goals]` | Dynamische Zieldefinitionen (quest, trade, lockpick, ...) |
| `[scoring]` | Punktwerte pro Beitragstyp |
| `[phrases]` | Suchmuster in Logs (trade_phrase, quest_phrase, lockpick_phrase) |
| `[display]` | embed_title/color/thumbnail, progress_bar_length/filled/empty, top_contributors_limit |
| `[mapevent]` | script_path, auto_start_enabled, start_message |
| `[adaptive]` | enabled, target_events_per_day, max_adjustment_percent, history_events |
| `[server_restarts]` | restart_times, buffer_before/after_restart |
| `[item_challenge]` | Dynamische Item-Ziele (z.B. item_VendingMachine_01=100) |

### Datenbank: `ini/database/mapevent_challenge.db` (aiosqlite)

| Tabelle | Spalten | Zweck |
|---|---|---|
| progress | type PK, current, goal | Fortschritt pro Kategorie |
| contributions | steam_id, name, type, points | Spieler-Beiträge |
| event_history | id PK, started_at, completed_at, duration_minutes, goals_json | Event-Archiv |
| kv_store | key PK, value | Beliebiger State |

### Verhalten
- **on_message:** Überwacht Log-Kanäle, parst Quest/Lockpick/Trade-Nachrichten
- **Adaptive Ziele:** Skaliert Ziele basierend auf Event-Historie
- **Restart-Erkennung:** Pausiert während Server-Neustarts (konfigurierbare Buffer)
- **Zielerreichung:** Startet `mapevent.py` als Subprocess
- **Daily Reset:** Setzt Fortschritt zu konfigurierter Stunde zurück
- **DM-Befehle:** status, deletedatabase, start, reset, reload
- **State Recovery:** Erkennt ausstehende Events nach Bot-Neustart

→ Startet `mapevent.py` ([08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md)) bei Zielerreichung

---

## airdrop.py
**Airdrop-Scheduler** — plant und platziert Airdrops in erlaubten Zonen.

### Config: `ini/airdrop/airdrop.ini`

| Abschnitt | Keys |
|---|---|
| `[AIRDROP]` | zyklen, lastdrop, nextdrop, dropcount, aktiv, instant_dropcount |
| `[Timer_1..N]` | time_start, time_end (HH:MM:SS), minuten_min, minuten_max, sperre |
| `[SperrZonen]` | zone_name=x1,y1;x2,y2 (rechteckige Sperrzonen) |
| `[Restarts]` | Restart-Zeiten (HH:MM:SS) |
| `[SperrZeiten]` | drop_sperre_min_bevor, drop_sperre_min_after |

### Verhalten
1. Wählt aktives Zeitfenster aus Timer-Konfigs
2. Generiert zufällige Koordinaten
3. Prüft gegen Sperrzonen, Bauverbotszonen (MapTools), Spieler-Flaggen-Radius (10km)
4. Schreibt Spawn-Befehl in `gamechat/*.txt`
5. Loggt in `log_public/*.txt`

### Trigger-Dateien

| Datei | Auslöser | Wirkung |
|---|---|---|
| `ini/airdrop/drop.go` | Extern angelegt | Sofort-Drop |
| `ini/airdrop/drop.override` | `purge_checker.ahk` | Trost-Airdrop nach gescheitertem Purge |

→ Nutzt `MapTools.py` ([08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md)) für Zonen-Prüfung
→ Liest `ini/flags.txt` ([07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md)) für Flaggen-Proximity

---

## handelsbericht.py
**Täglicher Handelsbericht** — analysiert Trades und postet Zusammenfassung.

### Config: `ini/handelsbericht.ini` (auto-erstellt)

| Abschnitt | Keys |
|---|---|
| `[Discord]` | webhook_url, main_embed_title, sales_embed_title, purchases_embed_title |
| `[Database]` | trade_db, stats_db, cleanup_trades |
| `[Settings]` | top_items_limit (10), log_level |

### Datenbanken

| DB | Tabelle | Zweck |
|---|---|---|
| `trades.db` | trades (timestamp, item_name, quantity, price, transaction_type) | Rohdaten (gelesen) |
| `historical_stats.db` | historical_stats (date PK, total_volume, total_sold, total_purchased) | Tagesaggregation |
| `historical_stats.db` | item_history (item_name, date, sold, purchased, avg_price) | Item-Historie |

### Verhalten
- Liest Trades der letzten 24h
- Berechnet Top-10 Items nach Gesamtwert (gewichteter Durchschnittspreis)
- Erkennt Inflation/Deflation
- Archiviert Tagesstatistiken in `historical_stats.db`
- Optional: Löscht verarbeitete Trades (`cleanup_trades`)
- Postet 2–3 Embeds via Discord-Webhook (Verkäufe, Käufe, Zusammenfassung)

→ Wird von `timer.py` ([08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md)) gestartet
→ Liest `trades.db` — befüllt von `trade_stats.py` Cog ([03_TRACKING_MONITORING.md](03_TRACKING_MONITORING.md))

---

## bot_bunker_functions.py
**Bunker-Utility** — parst Bunker-Lock-Logs (keine eigene Bot-Instanz, Hilfsbibliothek).

Wird als Modul von `bunker.py` Cog importiert. Details siehe [08_SUPPORT_SKRIPTE.md](08_SUPPORT_SKRIPTE.md) → bunker_func.py.

---

---

## bot_game-manager.py
**Master-Automatisierung** — überwacht Logins, holt Serverdaten, steuert Chat-Relay, RCV-Workflow.

### Config: `ini/game-manager.ini` (15+ Abschnitte)

| Abschnitt | Keys |
|---|---|
| `[game-manager]` | trigger_channel_id, steam_id_channel_id |
| `[autostart]` | scripts (Komma-getrennt, beim Start ausführen) |
| `[speech]` | welcome_fakename, welcome_delay, namechange_fakename, namechange_delay |
| `[logwatch]` | logfile (login.log Pfad), steamid (Filter oder "ALLE") |
| `[playercount]` | scum_logfile (SCUM.log), enabled, max_players |
| `[rcv]` | rcv_file, chatlog, verify_timeout_sec (15), login_timeout_sec (300), diverse wait_*_ms |
| `[vehicle_data]` | enabled, output_file (ini/act_cars.txt) |
| `[player_data]` | enabled, output_file (ini/players.txt) |
| `[squad_data]` | enabled, output_file (ini/squads.txt) |
| `[flag_data]` | enabled, output_file (ini/flags.txt) |
| `[streamtochat]` | enabled, delay_ms |
| `[window]` | title_contains (SCUM) |
| `[window_hotkey]` | hotkey (f11), size1, size2 |
| `[drone_mode]` | key (ctrl+d) |
| `[cleanup]` | directories (Ordner die beim Start geleert werden) |

### Keine Datenbank — rein dateibasiert

### Tasks & Threads

| Task/Thread | Funktion |
|---|---|
| Log-Listener (Daemon) | Überwacht `login.log` für Login-Events |
| Logout-Listener (Daemon) | Erkennt Logout-Events |
| Vehicle-Data | Sendet `#ListSpawnedVehicles` → parst → `ini/act_cars.txt` |
| Player-Data | Sendet Spieler-Befehl → `ini/players.txt` |
| Squad-Data | Sendet Squad-Befehl → `ini/squads.txt` |
| Flag-Data | Sendet Flag-Befehl → `ini/flags.txt` |
| Playercount | Tailt `SCUM.log` → extrahiert Spielerzahl |
| Window-Hotkey | F11 → Fenstergröße umschalten (650x380 ↔ 1280x720) |

### RCV-Workflow (Remote Chat Verification)
Nach Login des Bots im Spiel:
1. Wartet auf Login-Event (Timeout: 300s)
2. Prüft auf schnellen Logout (→ Reboot)
3. Öffnet Spielchat, wechselt Kanal, schließt (T → TAB → ESC)
4. Lädt zufällige Nachricht aus `speech/rcv.txt`
5. Schreibt in Chat-Bridge
6. Überwacht `chat.log` auf Verifikation (Timeout: 15s)
7. Bei Fehler: `force_reboot()` — bei Erfolg: weiter mit Daten-Fetches

### Command-Injection
Via `write_chat_command()`: Erstellt `.txt`-Datei in `gamechat_dir` → `streamtochat.py` → `streamtochat.ahk` → Spielfenster.

### Error-Handling
- Login-Timeout (300s) → `force_reboot()`
- RCV-Fehler → `force_reboot()`
- Schneller Logout nach Login → `force_reboot()`

→ Erzeugt die Dateien die von `onclipactions.ahk` ([07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md)) und zahlreichen Cogs gelesen werden
→ Liest Logs aus `PP-Live-Logcrawler/logs/` ([10_EXTERNE_TOOLS.md](10_EXTERNE_TOOLS.md))

---

## bot_lottery.py
**Lotteriesystem** — Ticket-basierte Lotterie mit Item-Tracking, Leaderboard und automatischer Ziehung.

### Config: `ini/lottery/config.ini`

| Abschnitt | Keys |
|---|---|
| `[Settings]` | batch_window_ms (1000), max_tickets_per_player, log_channel, game_log_channel, embed_channel, embed_channel_result, trade_channel, saecke_chat, trigger_tickets (Ziehungs-Schwelle) |
| `[Items]` | Item-Multiplikatoren |
| `[ItemName]` | Anzeigenamen pro Item |
| `[ItemName_Plural]` | Pluralformen |

### Datenbank: `ini/database/lottery.db`

| Tabelle | Spalten | Zweck |
|---|---|---|
| players | steamid PK, username, tickets | Spieler-Tickets |
| lotteries | lottery_id PK, draw_time, total_tickets, winner_info | Ziehungs-Archiv |
| meta | steamid PK, ticket_count, over_ticket_count, is_title_1/2/4_set | Metadaten & Titel-Status |
| tickets | ticket_id PK, steamid, ticket_code (8-stellig), is_new | Einzelne Tickets |
| processed_messages | message_id PK, created_at | Deduplizierung (14 Tage Retention) |

### Verhalten
- **on_message:** Überwacht Trade-Kanal, parst Format `STEAMID USERNAME ITEM QUANTITY`
- **Batch-Processing:** Sammelt Trades für `batch_window_ms`, verarbeitet dann gebündelt
- **Leaderboard:** Aktualisiert Embed jede Minute
- **Ziehung:** Automatisch bei `trigger_tickets` Schwelle — Top 3 Gewinner + dramatische Reveal-Sequenz
- **Overflow:** Überschüssige Tickets werden an andere Spieler umverteilt

### Dateien

| Datei | Zugriff |
|---|---|
| `ini/lottery/phrases.txt` + `phrases_{item}.txt` | Lesen (Zufalls-Nachrichten) |
| `ini/lottery/active_lottery_msg_id.txt` | Lesen + Schreiben (Embed-ID) |

→ Titel-Status wird von `profile.py` Cog gelesen ([04_SPIELER_IDENTITAET.md](04_SPIELER_IDENTITAET.md))

---

## bot_purgecounter.py
**Purge-Aktivitätszähler** — misst Server-Aktivität aus 5 Kanälen und berechnet Zombie-Angriffs-Wahrscheinlichkeit.

### Keine Datenbank — rein dateibasiert

### Überwachte Kanäle (5 Stück)

| Kanal-ID | Typ | Multiplikator |
|---|---|---|
| <CHANNEL_LOGIN> | Login | 0.9 |
| <CHANNEL_TRADES> | Handel/Trades | 0.5 |
| <CHANNEL_GLOBALCHAT> | Global-Chat | 0.9 |
| <CHANNEL_LOCKPICK> | Lockpicking | 0.7 |
| <CHANNEL_QUESTS> | Quests | 0.6 |

### Wahrscheinlichkeitsberechnung
- 0–10.000 Punkte → 0%
- 10.000–15.000 → Linear 50%–100%
- 15.000+ → 100%

### Tasks

| Task | Intervall | Funktion |
|---|---|---|
| save_counts | 5 Min | Counter speichern, Embed aktualisieren |
| post_status_update | 60s | Bot-Presence mit aktuellem Punktestand |

### Counter-Dateien
- `events/purge/counter/count_login.txt`
- `events/purge/counter/count_trades.txt`
- `events/purge/counter/count_chat.txt`
- `events/purge/counter/count_lockpick.txt`
- `events/purge/counter/count_event.txt`
- `events/purge/counter/count_total.txt`

**Ausgabe:** Status-Embed in Kanal <CHANNEL_PURGE_LOG> mit Aufschlüsselung, Wahrscheinlichkeit und letztem Angriff.

→ Counter werden von `purge_checker.ahk` ([07_AHK_AUTOMATISIERUNG.md](07_AHK_AUTOMATISIERUNG.md)) gelesen und ausgewertet

---

## bot_voicechannel.py
**Temporäre Voice-Channels** — Spieler joinen einen Creator-Channel und bekommen einen eigenen Raum.

### Config: `ini/squadmanager/config.ini`

| Abschnitt | Keys |
|---|---|
| `[Discord]` | GUILD_ID, CHATLOG_CHANNEL_ID, OWNER_ID, CREATOR_VOICE_CHANNEL_ID, VOICE_CATEGORY_ID, MEMBER_ROLE_ID, VIEWER_ROLE_ID, VOICE_DEFAULT_ROLE_IDS, TEXT_PRIVILEGED_ROLE_IDS, AFK_VOICE_CHANNEL_ID |
| `[Misc]` | OLD_MESSAGE_HOURS (12), VOICE_EMPTY_DELETE_MINUTES (5) |

### Datenbank: `ini/squadmanager/user_channels.db` (SQLite, WAL)

| Tabelle | Spalten | Zweck |
|---|---|---|
| user_channels | user_id PK, voice_channel_id, created_at, last_activity | User → Voice-Channel |
| channel_members | channel_owner_id + member_id PK, added_at | Permanente Mitglieder |
| voice_guests | channel_owner_id + user_id PK, expires_at, created_at | Temporäre Gäste |
| voice_guests_history | channel_owner_id, user_id, expired_at, created_at | Gäste-Archiv |
| voice_empty_tracking | voice_channel_id PK, empty_since, owner_id | Leere Kanäle tracken |

### Text-Befehle (im eigenen Voice-Channel-Text)

| Befehl | Funktion |
|---|---|
| `!add @User` | Mitglied hinzufügen (permanent) |
| `!remove @User` | Mitglied entfernen |
| `!guest @User [stunden]` | Temporärer Zugang (Standard: 1h) |
| `!unguest @User` | Gastzugang widerrufen |
| `!list` | Mitglieder anzeigen |
| `!kick @User` | In AFK-Channel verschieben |
| `!name NeuerName` | Channel umbenennen |

### Permissions-System
- `@everyone` → hidden (view=False, connect=False)
- **Owner** → view, connect, speak, manage_channels
- **DEFAULT_ROLE_IDS** → view, connect (immer erlaubt)
- **Mitglieder/Gäste** → view, connect, speak
- **VIEWER_ROLE** → view only (connect=False)

### Tasks

| Task | Intervall | Funktion |
|---|---|---|
| expire_voice_guests | 5 Min | Abgelaufene Gäste entfernen, in AFK verschieben |
| cleanup_empty_voices | 2 Min | Leere Channels nach X Min löschen |

### Events
- **on_voice_state_update:** Join Creator-Channel → eigenen Channel erstellen / Leave → empty_since setzen
- **on_message:** Text-Befehle verarbeiten
