# Bereich: LLM Admin Bot (llm_admin/)

**Status: Live**

## Übersicht

Der LLM Admin Bot ist ein KI-gestützter Server-Assistent mit zwei Betriebsmodi:
1. **Chat-Assistent** (reaktiv) — beantwortet Admin-Fragen in Discord per lokalem LLM mit Tool-Zugriff auf DBs, Logs und Regeln
2. **Log-Monitor** (proaktiv) — überwacht 10+ Log-Quellen, lernt Aktivitätsmuster, erkennt Anomalien und alarmiert

**Stack:** Python 3.9+ · discord.py 2.3+ · LM Studio (lokal, Qwen 3.5 9B) · SQLite (Read-Only via SMB) · YAML-Config

---

## Architektur

```
Discord-Kanal (Admin-Chat)
    ↓ on_message
    ├── Reply auf Bot/Monitor-Alert? → Heuristik + Relevanz überspringen
    │     (Monitor-Alerts werden als Kontext einbezogen bei depth 0)
    ├── Bild-Attachment? → Pillow JPEG-Konvertierung + Vision-Format
    ├── Quick Heuristic (Regex: Fragezeichen, "wer/wo/wann/wie...")
    ├── LLM Relevance Check ("ja"/"nein")
    ├── Kontext:
    │   ├── message.reference → Reply-Chain (max 10 Nachrichten rückwärts,
    │   │     3s Timeout, nur gleicher User + Bot, kein eigener State)
    │   └── Kein Reference → Channel-History (Fallback)
    └── Tool-Call-Loop (bis max_tool_rounds):
        ├── LLM wählt Tool → ToolExecutor führt aus
        ├── Ergebnis zurück an LLM
        └── Nächste Runde oder finale Antwort
    ↓
Discord-Antwort (max 2000 Zeichen, Split bei Bedarf)

Parallel:
Log-Watcher (alle 10s)
    → Neue Zeilen aus 10+ Log-Dateien
    → Noise-Filter + Keep-Patterns
    ↓
Event-Buffer (TTL pro Quelle)
    ↓
Baseline-Filter (alle 120s)
    → Lernphase (24h): alles durchlassen
    → Event-Typ-Klassifizierung (buy/sell, login/logout etc.)
    → Danach: nur Anomalien + kritische Events
    → Source-Level: Quelle insgesamt >3x → alles durch
    → Restart-Grace: login/violations nach Restart toleriert
    ↓
Log-Analyzer (LLM mit MONITOR_PROMPT + Tools)
    → Deduplizierung: nur neue Events + max 25 Kontext-Zeilen
    → "KEINE_AUFFAELLIGKEITEN" → Status-Meldung
    → Befund → Discord-Alert im Alert-Kanal
```

---

## Konfiguration: `config.yaml`

### Discord

| Key | Wert |
|---|---|
| token | Bot-Token |
| watch_channel_ids | [<CHANNEL_LLM_ADMIN>] |
| cooldown_seconds | 2 |

### LM Studio

| Key | Wert |
|---|---|
| base_url | `http://localhost:1234/v1` |
| model | `qwen/qwen3.5-9b` |
| temperature | 0.4 |
| max_tokens | 5000 |
| tool_use | true |
| max_tool_rounds | 25 |

### Datenbanken (Read-Only via SMB)

| Name | Pfad | Zweck |
|---|---|---|
| bank | `\\<LAN_IP>\...\bank.db` | Konten, Transaktionen, Lizenzen |
| players | `\\<LAN_IP>\...\userdata.db` | Spieler, Squads, Versicherung |
| userstats | `\\<LAN_IP>\...\userstats.db` | Voice-Time, Statistiken |
| flags | `\\<LAN_IP>\...\flags.db` | Flaggen-Positionen |

### Log-Quellen

| Key | Wert |
|---|---|
| paths | `\\<LAN_IP>\...\PP-Live-Logcrawler\logs` |
| extensions | [".log"] |
| max_files | 20 |
| max_results | 250 |

### Regel-Dateien

| Name | Datei | Inhalt |
|---|---|---|
| regeln | `ini/llm/regeln.txt` | Serverregeln (§1–§5) |
| serversettings | `ini/ServerSettings.ini` | Server-Einstellungen |
| squads | `ini/squads.txt` | Squad-Listen |
| economy | `ini/EconomyOverride.json` | Händler-/Item-Overrides |
| vehicles | `ini/act_cars.txt` | Fahrzeugbestand |
| vehicle_types | `vehicle_types.txt` | BPC-/BP-Codes → deutsche Fahrzeugnamen |

### Wissensdatenbank

| Key | Wert |
|---|---|
| knowledge_path | `knowledge/` (lokales Verzeichnis) |

Verzeichnis mit .txt-Dateien zu SCUM-Spielwissen (Crafting, Items, Mechaniken, Händler etc.).
Wird NICHT in jeden Prompt geladen, sondern per Tool-Call nur bei Bedarf abgerufen.
Neue Dateien einfach ins Verzeichnis legen — werden automatisch erkannt, kein Neustart nötig.

### Gedächtnis

| Key | Wert |
|---|---|
| notes_file | `llm_notes.txt` (manuell, Hot-Reload) |
| memory_file | `llm_memory.txt` (vom Bot beschrieben, einzige Schreibdatei) |

- **llm_notes.txt**: Admin-Korrekturen/Hinweise, manuell editiert, in jedem Prompt geladen
- **llm_memory.txt**: Vom Bot selbst per `save_memory`/`delete_memory` Tools beschrieben.
  Format: `[YYYY-MM-DD HH:MM | Adminname] Notiz`. Max 50KB. Hot-Reload.

### Monitor

| Key | Wert |
|---|---|
| enabled | true |
| alert_channel_id | <CHANNEL_LLM_ADMIN> |
| check_interval_seconds | 10 (Polling) |
| analysis_interval_seconds | 120 (LLM-Analyse) |
| learning_hours | 24 |
| anomaly_threshold | 3.0 (3x über Baseline = Anomalie) |
| ignored_players | ["<BOT_STEAM_ID>"] (Inselkom-Bot) |
| restart_times | ["03:00", "09:00", "15:00", "21:00"] |
| restart_grace_minutes | 10 |
| restart_grace_sources | ["login.log", "violations.log"] |

### Monitor-Quellen (11 Stück)

| Datei | TTL | Keep-Patterns |
|---|---|---|
| SCUM.log | 300s | battleye, ddos, ban, kick, cheat, tps, fps, global stats, error, crash etc. |
| economy.log | 900s | — (alles) |
| login.log | 600s | — |
| kill.log | 1200s | — |
| chat.log | 900s | — |
| gameplay.log | 300s | — |
| chest_ownership.log | 600s | — |
| event_kill.log | 1200s | — |
| quests.log | 300s | — |
| vehicle_destruction.log | 900s | — |
| violations.log | 900s | — |

---

## 20 Tools (Function Calling)

Tool-Beschreibungen sind bewusst kompakt (1-2 Sätze) für das 9B-Kontextfenster.
Der System-Prompt enthält zusätzlich eine kompakte Tool-Übersicht mit "Wann welches Tool"-Zuordnung.

### Spieler & Squads

| Tool | Parameter | Funktion |
|---|---|---|
| `get_player_profile` | identifier (Name/SteamID/GamerID) | Komplettes Profil aus allen DBs |
| `get_squad_profiles` | identifier (Squad-Name/ID) | Alle Mitglieder eines Squads mit Profilen |
| `run_query` | db_name, sql | Custom SELECT (Read-Only, validiert) |
| `get_db_schema` | — | Alle Tabellen-Strukturen |
| `get_server_stats` | — | Schnellübersicht (Spieler, Konten, Squads, etc.) |
| `get_server_status` | — | Live-Status aus SCUM.log Healthline (Spieler, TPS, Zombies, Fahrzeuge) |

### Logs

| Tool | Parameter | Funktion |
|---|---|---|
| `preview_logs` | — | Alle Log-Dateien + erste Zeilen |
| `search_logs` | keywords[] | Keyword-Suche über alle Logs |
| `read_log_file` | filename | Tail einer Log-Datei (500 Zeilen) |
| `search_log_file` | filename, keywords[] | Keyword-Suche in einer Datei |
| `search_logs_timerange` | filename, start_time, end_time, keywords[]? | Zeitfenster-Suche ("18:00" bis "20:00") |
| `player_timeline` | identifier, start_time, end_time | Chronologische Timeline aus ALLEN Logs (Forensik) |

### Regeln & Wissen

| Tool | Parameter | Funktion |
|---|---|---|
| `search_rules` | keywords[] | Suche in Regeldateien mit Kontext |
| `get_file_content` | name | Komplette Regeldatei lesen |
| `count_lines` | name | Zeilen zählen |
| `count_matches` | name, keyword | Treffer zählen |
| `search_knowledge` | keywords[] | Keyword-Suche in der SCUM-Wissensdatenbank |
| `get_knowledge_topic` | name | Komplettes Wissensthema lesen |

### Monitor & Gedächtnis

| Tool | Parameter | Funktion |
|---|---|---|
| `get_baselines` | — | Gelernte Aktivitäts-Baselines (Durchschnitte, Top-Spieler) |
| `save_memory` | note, author | Notiz im Bot-Gedächtnis speichern |
| `delete_memory` | keyword | Notiz aus Gedächtnis löschen |

---

## Sicherheit (4 Schichten, hardcoded)

### Schicht 1: Pfad-Whitelist
- Beim Start: alle DB-Pfade, Log-Ordner, Regel-Dateien, Knowledge-Verzeichnis registrieren
- `lock_whitelist()` → danach unveränderlich
- Jeder Dateizugriff wird gegen Whitelist geprüft
- Subdirectories sind NICHT erlaubt (nur exakte Pfade + direkte Dateien)

### Schicht 2: SQLite Read-Only
- Alle Verbindungen via URI `mode=ro`
- `PRAGMA query_only = ON` erzwungen

### Schicht 3: SQL-Validierung
- `safe_execute()` blockiert: INSERT, UPDATE, DELETE, DROP, ALTER, CREATE, REPLACE, ATTACH, DETACH, REINDEX, VACUUM
- Nur SELECT, PRAGMA, EXPLAIN erlaubt
- Blockierte Versuche werden geloggt

### Schicht 4: Dateischreiben (nur Memory)
- GENAU EINE Datei darf beschrieben werden: `llm_memory.txt`
- Registriert per `register_writable_path()`, nach `lock_whitelist()` unveränderlich
- `safe_append_text()` / `safe_write_text()` prüfen strikt gegen diesen einen Pfad
- Max 50KB Größenlimit

---

## Monitor: Anomalie-Erkennung

### Lernphase (24h)
- Alle Events durchlassen
- EMA (α=0.1) lernt Durchschnittswerte pro Spieler pro Log-Quelle **und pro Event-Typ**
- Baseline wird in `monitor_baseline.json` persistiert

### Event-Typ-Tracking
Baseline klassifiziert Events automatisch:

| Quelle | Typen |
|--------|-------|
| economy.log | buy (purchased by), sell (sold by) |
| login.log | login, logout |
| chest_ownership.log | claim, change |
| quests.log | complete, abandon |
| vehicle_destruction.log | destroyed, disappeared |
| gameplay.log | lockpick, flag_create, flag_destroy |

Wird pro Spieler+Quelle+Typ als EMA getrackt → "Spieler hat 6x mehr Verkäufe als normal"

### Nach der Lernphase
- **Spieler-Level:** `count / avg >= anomaly_threshold` (3.0x) → Anomalie, mit Typ-Aufschlüsselung
- **Source-Level:** Quelle insgesamt >= threshold → ALLE Events durchlassen
- Unbekannte Spieler mit ≥8 Events = auch Anomalie
- **Immer alarmieren:** battleye, ban, ddos, kick, cheat, hack, exploit

### Restart-Grace-Period
- Konfigurierbare Serverneustart-Zeiten (03:00, 09:00, 15:00, 21:00)
- Für 10 Min danach: login.log + violations.log Spikes werden toleriert (SCUM lässt nur 6–10 Spieler/Min rein, bis 80 warten)
- **AUSSERHALB** der Restart-Zeiten: Login-Spike = besonders verdächtig → Hinweis auf Serverprobleme/Absturz

### Deduplizierung (Analyzer)
- **Event-Level:** Bereits analysierte Events werden nicht erneut gesendet, max 25 Kontext-Zeilen
- **Alert-Level:** Gemeldete Spieler bekommen 30 Min Cooldown mit **Grund + Uhrzeit**. LLM erhält: `- <steam_id>: um HH:MM gemeldet für "Grund"` — kann so zwischen bereits gemeldetem und neuem Verhalten unterscheiden. Events vor dem Meldezeitpunkt gelten als bekannt.
- "Keine Auffälligkeiten" → kurze Status-Meldung im Channel (statt Stille)

### Was KEIN Alarm ist (im MONITOR_PROMPT definiert)
- Häufige Logins/Logouts (SCUM crasht oft)
- Bulk-Käufe/Verkäufe (jedes Item = 1 Zeile)
- Verschiedene Items bei verschiedenen Händlern kaufen/verkaufen = normales Einkaufen
- Hohe Kontostände
- Spieler handeln im Chat (keine Spam-Meldung)

### Was ein Alarm ist
- Arbitrage: **Exakt gleiches Item** kaufen bei Händler A, verkaufen bei Händler B, wiederholt (alle 3 Bedingungen!)
- BattlEye-Kicks/Bans
- DDoS-Warnungen
- Beleidigungen/Spam/Serverwerbung im Chat

---

## Baseline-Daten (`monitor_baseline.json`)

```json
{
  "meta": { "started": "...", "updated": "..." },
  "sources": {
    "economy.log": {
      "avg": 91.5, "samples": 5,
      "types": { "buy": { "avg": 45.0, "samples": 5 }, "sell": { "avg": 30.0, "samples": 5 } }
    }
  },
  "players": {
    "<PLAYER_STEAM_ID>": {
      "last_seen": "...",
      "activity": {
        "economy.log": {
          "avg": 4.6, "samples": 10,
          "types": { "buy": { "avg": 2.5, "samples": 10 }, "sell": { "avg": 2.1, "samples": 10 } }
        }
      }
    }
  }
}
```

- Event-Typ-Durchschnitte auf Source- und Spieler-Ebene
- Spieler ohne Aktivität seit 14 Tagen werden entfernt
- EMA-Flush alle 10 Minuten
- Rückwärtskompatibel: fehlende `types`-Felder werden beim nächsten Flush ergänzt

---

## Admin-Korrekturen: `llm_notes.txt`

Hot-Reload: Wird bei jedem Aufruf (Chat + Monitor) frisch gelesen wenn geändert — kein Neustart nötig.
- Fahrzeuge: `act_cars.txt` ist primäre Quelle, nicht DB
- Fahrzeugtypen: ausgelagert in `vehicle_types.txt` (per Tool abrufbar), beginnen mit "BPC_"
- Performance-Daten in SCUM.log
- Lizenz-Daten in der Vergangenheit = abgelaufen
- TPS = 30 optimal (nicht "FPS" sagen)
- FMJ ist Server-Admin

---

## Domain-Wissen (im SYSTEM_PROMPT)

| Bereich | Detail |
|---|---|
| Währung | SD = ScumDollar (Ingame), Gold = Discord+Ingame (teuer) |
| Lizenzen | l1=Banker, l2=Fuhrpark (+1 Fahrzeug), l3=Baulizenz (+1 Base) |
| Versicherung | insurance_id vorhanden = versichert |
| EconomyOverride | "-1" = Game-Default verwenden (NICHT deaktiviert!) |
| Händler | 4 Sektoren (A_0, B_4, C_2, Z_3), je 6 Typen |
| Logs | Englische Begriffe (login, kill, Currency Conversion, ban) |

---

## Verbindungen zum Hauptsystem

```
llm_admin (LM Studio + Discord Bot)
    │
    ├── LIEST (Read-Only via SMB):
    │   ├── bank.db ← bank.py, voice_rewards.py, lizenzen.py
    │   ├── userdata.db ← identity.py, game_events.py, squad.py
    │   ├── userstats.db ← voice_rewards.py
    │   ├── flags.db ← trackingstation.py
    │   ├── PP-Live-Logcrawler/logs/*.log ← PP_Console_Crawler
    │   ├── ini/llm/regeln.txt (Serverregeln)
    │   ├── ini/ServerSettings.ini (Server-Config)
    │   ├── ini/squads.txt ← onclipactions.ahk
    │   ├── ini/EconomyOverride.json (Händler-Config)
    │   └── ini/act_cars.txt ← onclipactions.ahk
    │
    ├── LOKAL (im llm_admin/-Ordner):
    │   ├── llm_notes.txt (Admin-Hinweise, manuell gepflegt)
    │   ├── llm_memory.txt (Bot-Gedächtnis, EINZIGE Schreibdatei)
    │   ├── knowledge/*.txt (SCUM-Wissensdatenbank)
    │   └── monitor_baseline.json (gelernte Aktivitätsmuster)
    │
    └── SCHREIBT: Nur llm_memory.txt (max 50KB, per save_memory/delete_memory Tools)
```

→ Datenquellen: [02_WIRTSCHAFT.md](02_WIRTSCHAFT.md) (bank.db), [04_SPIELER_IDENTITAET.md](04_SPIELER_IDENTITAET.md) (userdata.db), [10_EXTERNE_TOOLS.md](10_EXTERNE_TOOLS.md) (PP-Logcrawler Logs)
→ Läuft auf separatem Rechner, greift via SMB-Shares auf die DBs zu
