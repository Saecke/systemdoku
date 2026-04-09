# Bereich: Logs, Kommunikation & Administration

## Übersicht

Dieses Bereich umfasst drei Säulen:
1. **Kommunikation** — Brücken zwischen Discord und Spielchat, Log-Relay-Systeme
2. **UI & Navigation** — Router (persistente Befehlszentrale), Regeln, Hilfe
3. **Administration** — Audit, Cleanup, Maintenance, Bootstrap, Squad-Sync

---

## Kommunikation

### chat_bridge.py (Cog)
**Discord → Spielchat** — leitet Nachrichten aus Discord ins Spiel und umgekehrt.

- **Inselkom-Handler:** Überwacht Inselkom-Kanal, schreibt Nachrichten als `.txt` in `GAMECHAT_DIR`
- **Remote-Admin-Handler:** Überwacht Remote-Admin-Kanal mit Bestätigungs-Workflow (✅/❌, 30s Timeout)
- **Sanitierung:** Erlaubte Zeichen (`ALLOWED_CHARS`), @mentions werden abgelehnt
- **Eventsperre:** Sperrt Weiterleitung während Events (`eventsperre`-Datei)
- **Feedback:** ⏳ → ✅ Reaktionen, Auto-Delete nach Verarbeitung

| DB / Datei | Zugriff |
|---|---|
| `userdata.db` | Lesen (Discord→Spielername-Mapping) |
| `cfg.GAMECHAT_DIR/*.txt` | Schreiben (Spielchat-Befehle) |
| `cfg.eventsperre` | Lesen (Sperr-Flag) |

---

### log_bridge.py (Cog)
**Leichtgewichtiger Log-Relay** — liest lokale Dateien, postet in Discord-Kanäle.

- **Task:** Pollt Ordner aus `cfg.logs.folders` im konfigurierten Intervall
- **Encoding:** UTF-16 BOM → UTF-8-SIG → chardet → Latin-1 (Fallback-Kette)
- **Chunking:** Teilt große Logs in 1900-Zeichen-Blöcke
- **Nachbearbeitung:** `rename_phrases` (Textersetzung), `skip_phrases` (Datei überspringen)
- **Aufräumen:** Quelldateien werden nach erfolgreichem Post gelöscht

---

### inselkom.py (Cog)
**Erweiterter Log-Relay** — wie log_bridge, aber mit Rate-Limiting und Diagnose.

- **Per-Channel-Rate-Limiting:** Sliding-Window-Deque, respektiert Discords 5 req/5s Limit
- **Burst-Cap:** Konfigurierbar via `cfg.logs.inselkom_burst_cap`
- **Quarantäne:** Dateien für ungültige Kanäle werden verschoben statt gelöscht
- **Diagnose:** Detailliertes Logging (Chunk-Index, Retry-After, Channel-Name)
- **429-Handling:** Parst `retry_after`, Minimum 1.5–2.0s Pause

Parallel zu log_bridge — feature-reicher für Hochlast-Szenarien.

---

## UI & Navigation

### router.py (Cog) — Befehlszentrale v1
**Persistentes UI** — Embed mit Buttons und Dropdown im Link-Kanal.

- **Buttons:** Link, Status, Profil, Hilfe, Dashboard, Spenden
- **Dropdown:** Char, CharX, KFZ, Lizenzen, Squad, Tresor, Rangliste, Titel, Todo, Fischen, Events, Awards
- **Antworten:** Immer ephemeral (privat)
- **Delegation:** Ruft Methoden anderer Cogs auf (Profile, Fishing, Awards, etc.)
- **Persistenz:** Message-ID in `ini/router_message_id.txt`, wird beim Start geladen/erstellt

**Verbindung:** Schwere Cross-Cog-Abhängigkeit — benötigt Profile, Fishing, EventsStats, Awards Cogs.

---

### router_v2.py (Cog) — Befehlszentrale v2 (Test)
**Alternative UI** — Discord Components v2 mit per-Button-Callbacks.

- **Feature-Flag:** `cfg.features.router_v2.channel_id`
- **Untermenüs:** Account, Stats, Titles, Economy (kategorisiert)
- **Unlink-Bestätigung:** Dialog mit 60s Timeout
- Nutzt intern Router-v1-Methoden für Konsistenz.

---

### rules.py (Cog)
**Regelakzeptanz** — persistente Regelnachricht mit Rollen-Vergabe.

- **UI:** Components v2 (mit Fallback auf Plaintext + Button)
- **Button:** "Regeln akzeptieren" → Rolle wird zugewiesen
- **Idempotent:** Prüft ob Rolle bereits vorhanden
- **Persistenz:** Message-ID in `ini/rules_message_id.txt`, auto-pinned

---

### help.py (Cog)
**Hilfe-Übersicht** — `/hilfe` listet alle verfügbaren Befehle.

- **Befehl:** `/hilfe` (Aliases: h, H) — DM-only
- **Dynamisch:** `/log`-Eintrag zeigt Dashboard-Redirect wenn deaktiviert
- Keine DB-Zugriffe.

---

### basic.py (Cog)
**Status-Befehl** — `/status` zeigt Verlinkung und Sponsor-Info.

- **Befehl:** `/status` (Aliases: state) — DM-only
- **Anzeige:** Discord-Name, Spielername, Steam-ID, Verlinkungszeitpunkt, Rollen-Badges
- **Unverlinkt:** Bietet Reaktions-basierte Verlinkung an (✅/❌ → `/link`)

---

## Administration

### audit.py (Cog)
**Befehls-Logging** — protokolliert alle ausgeführten Befehle.

- **Listener:** `on_command` (jeder Befehl), `on_command_error` (CommandNotFound)
- **Kanal-Posts:** Optional in `cfg.input_log_channel_id` (abschaltbar)
- **Pacing:** 2s Pause zwischen Channel-Posts gegen Rate-Limits

---

### cleanup_messages.py (Cog)
**Nachrichten-Bereinigung** — löscht alte Nachrichten aus konfigurierten Kanälen.

- **Task:** 2-Min-Loop, Round-Robin über konfigurierte Kanäle
- **Config:** `cfg.cleanup.purge_channels` — pro Kanal: `max_messages` + `time` (Stunden)
- **Schutz:** Gepinnte Nachrichten werden übersprungen
- **Bulk-Delete:** Chunks mit konfigurierbarer Größe und Pause
- **Rate-Limiting:** 429-Erkennung mit 2.5s Backoff
- **Counter:** Gesamtzahl gelöschter Nachrichten in `cfg.cleanup.deleted_messages_file`

---

### maintenance.py (Cog)
**Wartungs-Koordinator** — orchestriert periodische Updates anderer Cogs.

- **Task:** 5-Min-Loop, ruft sequentiell auf:
  1. `Squad.update_member_counts()`
  2. `Titles.check_and_update_roles()`
  3. `History.cleanup_old_logs()`
  4. `Titles.assign_lottery_title_roles()`
  5. `Titles.assign_mapevent_title_roles()`
- **Defensiv:** Prüft ob Ziel-Cog geladen ist, 1s Pause zwischen Aufrufen

---

### bootstrap.py (Cog)
**Start & Überwachung** — Whitelist, Datei-Watcher, Auto-Restart.

- **on_ready:** Postet Serverliste in Hi-Kanal, verlässt nicht-gewhitelistete Guilds
- **File-Watcher:** 10s-Loop, prüft mtime von bootstrap.py
- **Auto-Restart:** Bei Dateiänderung → `os.execv()` (vollständiger Prozess-Neustart)
- **Fallback:** Wenn execv fehlschlägt → `bot.close()`

---

### hooks.py (Cog)
**Webhook-Lifecycle** — verwaltet temporäre Webhooks mit Ablaufdatum.

- **API:** `add_persistent_hook(webhook_id, expire_at)` — plant Löschung
- **Persistenz:** JSON in `cfg.hooks_file` mit Webhook-IDs und Timestamps
- **Startup:** Reschedule aller offenen Webhooks aus vorheriger Session
- **Ablauf:** `asyncio.sleep()` bis Expiry → `webhook.delete()`

---

### squad.py (Cog)
**Squad-Sync & Info** — synchronisiert Squad-Roster aus Spieldatei.

- **Task:** `squads_to_database` (15 Min) — parst `act_squads_file_path`
  - Regex extrahiert: SquadId, SquadName, SteamId, CharacterName, MemberRank
  - Aktualisiert `userdata` (squad_id, squad_name, member_rank)
  - Erstellt/aktualisiert `squaddata` (Mitglieder, Handelswerte, Tresor)
  - Entfernt Squad-Felder für Spieler die nicht mehr in Squads sind
- **Befehl:** `/squad` (Aliases: team, s) — Squad-Info mit Mitgliedern nach Rang
  - 💸 Gold-Transfer (Spieler-Tresor → Squad-Tresor)
  - 🎯 Purge-Status

| DB | Tabellen | Zugriff |
|---|---|---|
| `userdata.db` | userdata (squad_id, squad_name, member_rank) | Lesen + Schreiben |
| `userdata.db` | squaddata (members, volumes, s_tresor) | Lesen + Schreiben |
| `bank.db` | Konten (Gold-Transfer) | Lesen + Schreiben |

---

### core.py (Cog)
**Minimaler Startup** — loggt Bot-Identität, synct Slash-Commands.

- **on_ready:** Bot-Info loggen + `bot.tree.sync()`
- Early-Load Cog, keine Fachlogik.

---

## Datenflüsse

```
Spieler-Nachricht in Discord
    → chat_bridge.py (Sanitierung, Bestätigung)
    → GAMECHAT_DIR/*.txt
    → Spielserver zeigt Nachricht im Chat

Server-Logs (lokal)
    → log_bridge.py / inselkom.py (Polling, Encoding, Chunking)
    → Discord-Kanäle (nach cfg.logs.folders Mapping)

Spieler klickt Button im Router
    → router.py / router_v2.py
    → Delegation an Ziel-Cog (Profile, Fishing, Awards, etc.)
    → Ephemeral-Antwort an Spieler

Alle 5 Minuten
    → maintenance.py ruft Squad, Titles, History auf
    → Squad-Member-Counts aktualisiert
    → Titel/Rollen geprüft und synchronisiert
    → Alte Logs bereinigt

Alle 2 Minuten
    → cleanup_messages.py löscht alte Nachrichten
```
