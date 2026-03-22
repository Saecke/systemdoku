# Bereich: LLM Admin Bot (llm_admin/)

**Status: In Entwicklung — geht bald live**

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
    ├── Quick Heuristic (Regex: Fragezeichen, "wer/wo/wann/wie...")
    ├── LLM Relevance Check ("ja"/"nein")
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
    → Danach: nur Anomalien + kritische Events
    ↓
Log-Analyzer (LLM mit MONITOR_PROMPT + Tools)
    → "KEINE_AUFFAELLIGKEITEN" oder Befund
    ↓
Discord-Alert im Alert-Kanal
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

### Monitor-Quellen (10 Stück)

| Datei | TTL | Keep-Patterns |
|---|---|---|
| SCUM.log | 300s | battleye, ddos, ban, kick, cheat, hack, exploit |
| economy.log | 900s | — (alles) |
| login.log | 600s | — |
| chat.log | 600s | — |
| gameplay.log | 600s | — |
| kill.log | 600s | — |
| quests.log | 600s | — |
| chest_ownership.log | 600s | — |
| event_kill.log | 600s | — |
| vehicle_destruction.log | 600s | — |

---

## 12 Tools (Function Calling)

| Tool | Parameter | Funktion |
|---|---|---|
| `get_player_profile` | identifier (Name/SteamID/GamerID) | Komplettes Profil aus allen DBs |
| `run_query` | db_name, sql | Custom SELECT (Read-Only, validiert) |
| `get_db_schema` | — | Alle Tabellen-Strukturen |
| `get_server_stats` | — | Schnellübersicht (Spieler, Konten, Squads, etc.) |
| `preview_logs` | — | Alle Log-Dateien + erste Zeilen |
| `search_logs` | keywords[] | Keyword-Suche über alle Logs |
| `read_log_file` | filename | Tail einer Log-Datei (500 Zeilen) |
| `search_log_file` | filename, keywords[] | Keyword-Suche in einer Datei |
| `search_rules` | keywords[] | Suche in Regeldateien mit Kontext |
| `get_file_content` | name | Komplette Regeldatei lesen |
| `count_lines` | name | Zeilen zählen |
| `count_matches` | name, keyword | Treffer zählen |

---

## Sicherheit (3 Schichten, hardcoded)

### Schicht 1: Pfad-Whitelist
- Beim Start: alle DB-Pfade, Log-Ordner, Regel-Dateien registrieren
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

---

## Monitor: Anomalie-Erkennung

### Lernphase (24h)
- Alle Events durchlassen
- EMA (α=0.1) lernt Durchschnittswerte pro Spieler pro Log-Quelle
- Baseline wird in `monitor_baseline.json` persistiert

### Nach der Lernphase
- Vergleich: aktuelle Aktivität vs. gelernte Baseline
- Anomalie wenn: `count / avg >= anomaly_threshold` (3.0x)
- Unbekannte Spieler mit ≥8 Events = auch Anomalie
- **Immer alarmieren:** battleye, ban, ddos, kick, cheat, hack, exploit

### Was KEIN Alarm ist (im MONITOR_PROMPT definiert)
- Häufige Logins/Logouts (SCUM crasht oft)
- Bulk-Käufe/Verkäufe (jedes Item = 1 Zeile)
- Hohe Kontostände
- Spieler handeln im Chat (keine Spam-Meldung)

### Was ein Alarm ist
- Arbitrage: Kaufen bei Händler A, Verkaufen bei Händler B, wiederholen
- BattlEye-Kicks/Bans
- DDoS-Warnungen
- Beleidigungen/Spam/Serverwerbung im Chat

---

## Baseline-Daten (`monitor_baseline.json`)

```json
{
  "meta": { "started": "...", "updated": "..." },
  "sources": {
    "economy.log": { "avg": 91.5, "samples": 5 },
    "chat.log": { "avg": 72.7, "samples": 5 },
    "SCUM.log": { "avg": 757.3, "samples": 5 }
  },
  "players": {
    "<PLAYER_STEAM_ID>": {
      "last_seen": "...",
      "activity": { "chat.log": { "avg": 4.6, "samples": 2 } }
    }
  }
}
```

- Spieler ohne Aktivität seit 14 Tagen werden entfernt
- EMA-Flush alle 10 Minuten

---

## Admin-Korrekturen: `llm_notes.txt`

Wird bei jedem Chat-Aufruf an den System-Prompt angehängt:
- Fahrzeuge: `act_cars.txt` ist primäre Quelle, nicht DB
- Fahrzeugtypen beginnen mit "BPC_"
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
    └── SCHREIBT: Nichts (rein lesend)
```

→ Datenquellen: [02_WIRTSCHAFT.md](02_WIRTSCHAFT.md) (bank.db), [04_SPIELER_IDENTITAET.md](04_SPIELER_IDENTITAET.md) (userdata.db), [10_EXTERNE_TOOLS.md](10_EXTERNE_TOOLS.md) (PP-Logcrawler Logs)
→ Läuft auf separatem Rechner, greift via SMB-Shares auf die DBs zu
