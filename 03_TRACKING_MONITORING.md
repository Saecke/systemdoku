# Bereich: Tracking & Monitoring

## Übersicht

Tracking und Monitoring decken die Echtzeit-Überwachung des Servers ab: Spielerstatistiken, Flaggen-Tracking, Kill-Feed, Serverstatus, Händlerstände, Steam-Profile und Voice-Belohnungen. Der `health.py`-Cog dient als Diagnose-Hub für das gesamte System.

---

## Komponenten

### statbot.py (Cog)
**Statistik-Dashboard** — Zählt Nachrichten, parst Handels- und Währungsdaten, pflegt Live-Übersicht.

- **Listener:** `on_message` — zählt Nachrichten in konfigurierten Kanälen (mit optionalem Keyword-Filter)
- **Trade-Parsing:** Erkennt `[trade]`-Zeilen mit "sold by"/"bought by", extrahiert Preis und Menge
- **Währung:** Parst `[currency conversion]`-Zeilen (Gold/Credit-Umrechnung)
- **Task:** `overview_updater` (60s) — aktualisiert Discord-Embeds mit Live-Statistiken und Deltas
- **Inflation/Deflation:** Berechnet wirtschaftliche Trends

| DB | Tabellen | Zugriff |
|---|---|---|
| `statbot.db` | counters (date, counter, count), meta (Wirtschafts-Aggregation) | Lesen + Schreiben |

---

### kill_feed.py (Cog)
**Kill-Feed** — NPC-Tötungen als formatierte Meldungen.

- **Listener:** `on_message` auf Kill-Logs-Kanal
- **Parsing:** JSON-Format (bevorzugt) oder Text-Fallback; extrahiert Killer, Opfer, Waffe, Distanz
- **Deduplizierung:** Hash-basierter Kurzzeit-Cache (64 Einträge)
- **Ausgabe:** Farbkodierte Embeds nach NPC-Level, mit Sektor und Tageszeit
- **Anonymisierung:** Spielernamen werden nicht direkt gezeigt

| Datei | Zweck |
|---|---|
| `ini/kill_feed/templates.ini` | Nachrichtenvorlagen (5 Templates mit Platzhaltern) |
| `ini/kill_feed/renames_npc.ini` | NPC-Namens-Mapping |
| `ini/kill_feed/renames_weapon.ini` | Waffen-Namens-Mapping |
| `ini/kill_feed/titles.txt` | Witzige Überschriften für Embeds |

---

### server_status.py (Cog)
**Serverstatus-Anzeige** — BattleMetrics-API + lokale Health-Daten.

- **Task:** 60s-Loop — pollt BattleMetrics API (10-Min-Cache)
- **Quellen:** `playercount.ini` und `serverhealth.ini` aus dem PP-Live-Logcrawler
- **Anzeige:** Online/Offline, Spielerzahl, IP, Rang, Version, TPS-Health, Entity-Counts
- **Grafiken:** Sparklines aus TPS-History (5 Werte) und Player-History (30 Werte)
- **Bot-Presence:** Aktualisiert Bot-Aktivität mit Spielerzahl

| Datei | Quelle |
|---|---|
| `ini/serveranzeige.ini` | Server-ID, Channel-ID, Anzeigeoptionen |
| `ini/sa_m_id.txt` | Persistierte Message-ID für In-Place-Updates |
| `PP-Live-Logcrawler/ini/playercount.ini` | Spielerzahl (2-Min-Frische) |
| `PP-Live-Logcrawler/ini/serverhealth.ini` | TPS, Characters, Items, Objects |

---

### trader_status.py (Cog)
**Händler-Fondsstände** — Zeigt aktuelle Händler-Guthaben nach Standort.

- **Listener:** `on_message` auf Economy-Kanal — erkennt `[Trade] After`-Logs
- **Task:** 30s-Loop (mit exponentiellem Backoff) — aktualisiert Embed
- **Anzeige:** Händler gruppiert nach Gebiet (Location-Prefix)
- **Persistierung:** `ini/trader.ini` (trader_name=balance#timestamp)

---

### steam_watch.py (Cog)
**Neue Spieler & Namensänderungen** — Login-Überwachung + Steam-API.

- **Listener:** `on_message` auf Login-Kanal
- **Parsing:** `[Login] PlayerName (GamerID) (SteamID)`
- **Erkennung:** Neue Spieler (nie gesehen) oder Namensänderung (Vergleich mit userdata.db)
- **Steam-API:** GetPlayerSummaries + GetPlayerBans bei neuen Spielern
- **Cooldown:** 30s pro Steam-ID gegen Burst-Duplikate
- **Ausgabe:** JSON-Report mit Profil-URL, Erstelldatum, VAC/Game-Ban-Status

| DB | Zugriff |
|---|---|
| `userdata.db` | Lesen (Name-History-Vergleich) |

---

### voice_rewards.py (Cog)
**Voice-Belohnungen** — Sprachkanal-Zeit wird zu Gold.

- **Task 1:** `update_counters` (60s) — scannt alle Voice-Kanäle
  - Mindestens 2 aktive (nicht stummgeschaltete) Mitglieder nötig
  - AFK-Kanäle werden übersprungen
- **Task 2:** `update_gold_rewards` (5 Min) — verteilt Gold
  - Formel: `(voice_time / 60) * cfg.reward_rate`
  - Cross-Referenz: Discord-ID → Steam-ID (userdata.db) → Konto (bank.db)
  - Setzt voice_time nach Auszahlung zurück

| DB | Tabellen | Zugriff |
|---|---|---|
| `userstats.db` | user_voice_time (discord_id, voice_time, voice_time_overall) | Lesen + Schreiben |
| `userdata.db` | Discord→Steam Mapping | Lesen |
| `bank.db` | Konten (Gold-Einzahlung) | Schreiben |

---

### trackingstation.py (Cog)
**Spieler- & Flaggen-Tracking** — Phrasen-Weiterleitung, verdächtige Neuzugänge und Flaggenüberwachung.

- **Feature 1: Phrasen-Tracking** — Kanalnamen der Tracking-Kategorie dienen als Trigger-Phrasen. Nachrichten aus Quellkanälen, die eine Phrase enthalten, werden in den passenden Tracking-Kanal weitergeleitet.
- **Feature 2: New-Player-Tracking** — Reagiert auf "New player"-JSON. Bei VAC-Ban, Community-Ban, Economy-Ban oder Game-Bans wird automatisch ein Tracking-Kanal (SteamID als Name) in der Kategorie erstellt.
- **Feature 3: Flaggen-Checks** — Basebuilding-Logs ("Gebaut."/"Zerstört.") werden auf Bauverbotszonen geprüft (via MapTools, optional). Bei Verstoß: Reaktion + Ping an Moderatoren.
- **Feature 4: Flaggen-DB-Sync** — Verarbeitet `serverlogs/flagtracking/*.log` (Created/Destroyed/Overtaken) in `flags.db`. Zusätzlich: Snapshot-Sync aus `flags.txt` mit Plausibilitätsprüfung (>30%-Rückgang wird übersprungen).
- **Tasks:** `check_channel_names` (30s), `process_flagtracking` (30s)
- **Befehl:** `!tracking-diag` — Supervisor-only Diagnose (Kategorie, Phrasen, Loops, Pfade)
- **Konfiguration:** `cfg.tracking` mit Fallback auf `ini/bot_tracker.ini` (Legacy)
- **Kategorie-Autodiscovery:** Findet die Tracking-Kategorie automatisch, wenn keine ID konfiguriert ist (sucht nach "tracking"/"track"/"flagtracking")

| Ressource | Zugriff |
|---|---|
| `flags.db` (flags) | Lesen + Schreiben |
| `serverlogs/flagtracking/*.log` | Lesen + Löschen nach Verarbeitung |
| `ini/flags.txt` | Lesen (autoritativer Snapshot) |
| `ini/bot_tracker.ini` | Lesen (Legacy-Fallback) |

---

### health.py (Cog)
**System-Diagnose** — Supervisor-only Health-Check.

- **Befehl:** `/health` (Aliases: /diag, /hc, /syshealth) — nur für Supervisor
- **Prüft:** Alle DBs (Read-Only), Task/Loop-Status aller Cogs, HTTP-Monitor-Stats
- **Metriken:** Linked Users, Voice-Aktivität, Bankkonten, Squads, Titles
- **Admin-Befehle:** `/hc-rb` (Reboot), `/hc-stop` (Shutdown), `/hc-rc` (Reload Cogs), `/hc-rl <cog>` (Einzelnes Cog neu laden)

---

## Datenflüsse

```
Spielserver
    │
    ├── Login-Logs → Discord-Kanal → steam_watch.py → Steam-API → Meldung
    │
    ├── Kill-Logs → Discord-Kanal → kill_feed.py → Kill-Feed-Kanal
    │
    ├── Trade-Logs → Discord-Kanal ─┬→ statbot.py → statbot.db → Übersichts-Embed
    │                                └→ trader_status.py → trader.ini → Händler-Embed
    │
    ├── BattleMetrics API ──→ server_status.py → Status-Embed + Bot-Presence
    │   PP-Logcrawler INIs ─┘
    │
    ├── Voice-Kanäle → voice_rewards.py → userstats.db → bank.db (Gold)
    │
    ├── Log-Kanäle → trackingstation.py ─┬→ Phrasen-Tracking → Tracking-Kategorie
    │   New-Player-JSON ─┘               ├→ New-Player → Tracking-Kanal (SteamID)
    │   Basebuilding-Logs ─┘             └→ Flaggen-Check (MapTools) + flags.db
    │   flagtracking/*.log ─┘
    │   flags.txt ─┘

health.py ← introspiziert alle Cogs + DBs + HTTP-Monitor
```

## Verbindungen zu anderen Bereichen

- **voice_rewards.py** schreibt in `bank.db` (Wirtschaft)
- **steam_watch.py** liest aus `userdata.db` (Spieler-Identity)
- **server_status.py** liest aus dem `PP-Live-Logcrawler` (Externes Tool)
- **statbot.py** nutzt `http_monitor` (Kern-Infrastruktur)
- **trackingstation.py** nutzt `MapTools` (optional, für Bauverbotszonen-Prüfung)
