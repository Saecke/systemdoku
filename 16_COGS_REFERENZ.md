# Cogs-Referenz (bot_infobot.py)

Technische Referenz fuer das Cog-System des Haupt-Bots. Fuer die fachliche Beschreibung einzelner Cogs siehe die Domaenen-Dokumente [02](02_WIRTSCHAFT.md)--[06](06_LOGS_KOMMUNIKATION.md), fuer die tabellarische Uebersicht siehe [00_ARCHITEKTURPRINZIP.md](00_ARCHITEKTURPRINZIP.md#cog-uebersicht-bot_infobotpy--_cogs).

---

## Lade-Architektur

```
bot_infobot.py
  --> _modules/config.py      Token aus ini/system.ini laden
  --> _modules/app_config.py   Zentrale Konfiguration (Single Source of Truth)
  --> _modules/loader.py       bind_setup() erzeugt den setup_hook
  --> _modules/http_monitor.py HTTP-Request-Monitoring installieren

Beim Start:
  bot.setup_hook = bind_setup(bot)
  bot.run(TOKEN)

bind_setup(bot) gibt eine async-Funktion zurueck:
  1. Optional: Ausgabe der extensions_meta (formatierte Tabelle in Konsole)
  2. Iteration ueber cfg.startup_cogs (Liste von Strings)
  3. Fuer jede Extension: bot.load_extension(ext)
  4. Ergebnis: bot._loaded_extensions_order (geladen) + bot._failed_extensions (fehlgeschlagen)
```

### Reihenfolge

Die Ladereihenfolge in `cfg.startup_cogs` ist relevant. Einige Cogs setzen Bot-Attribute, die von spaeter geladenen Cogs gelesen werden (siehe [Implizite Abhaengigkeiten](#implizite-abhaengigkeiten-zwischen-cogs)).

### Fehlerbehandlung

Fehlgeschlagene Cogs blockieren nicht den Start - sie werden geloggt und uebersprungen. Der Health-Cog kann den Ladezustand ueber `bot._loaded_extensions_order` und `bot._failed_extensions` inspizieren.

---

## Ladezustand (Stand: Maerz 2026)

### Aktive Cogs (in Ladereihenfolge)

| # | Extension | Kategorie |
|---|-----------|-----------|
| 01 | trackingstation | Tracking |
| 02 | core | Admin |
| 03 | basic | Admin |
| 04 | help | Admin |
| 05 | identity | Spieler |
| 06 | steam_watch | Tracking |
| 07 | insurance | Wirtschaft |
| 08 | check_insurance | Wirtschaft |
| 09 | titles | Events |
| 10 | profile | Spieler |
| 11 | char | Spieler |
| 12 | history | Spieler |
| 13 | squad | Kommunikation |
| 14 | lizenzen | Wirtschaft |
| 15 | bank | Wirtschaft |
| 16 | carservice | Wirtschaft |
| 17 | purge | Events |
| 18 | game_events | Events |
| 19 | hooks | Kommunikation |
| 20 | voice_rewards | Tracking |
| 21 | health | Admin |
| 22 | bootstrap | Admin |
| 23 | audit | Admin |
| 24 | maintenance | Admin |
| 25 | statbot | Tracking |
| 26 | cleanup_messages | Admin |
| 27 | server_status | Tracking |
| 28 | chat_bridge | Kommunikation |
| 29 | cars_uploader | Wirtschaft |
| 30 | trader_status | Tracking |
| 31 | trade_stats | Wirtschaft |
| 32 | team_sync | Admin |
| 33 | inselkom | Kommunikation |
| 34 | kill_feed | Tracking |
| 35 | npczones | Events |
| 36 | fishing | Events |
| 37 | events_stats | Events |
| 38 | awards | Events |
| 39 | donations | Wirtschaft |
| 40 | router | Kommunikation |
| 41 | handelshaus | Wirtschaft |
| 42 | link_translator | Spieler |
| 43 | vehicle_monitor | Wirtschaft |

### Deaktivierte Cogs (auskommentiert in startup_cogs)

| Extension | Grund |
|-----------|-------|
| aircraft | Derzeit nicht im Einsatz |
| map_event_challenges | Derzeit nicht im Einsatz |
| bunker | Derzeit nicht im Einsatz |

### Nicht eingebundene Cogs (nur im Entwicklungsordner)

Die folgenden Dateien existieren nur lokal im Entwicklungsordner und sind **nicht** auf dem Live-System vorhanden. Sie sind weder in `startup_cogs` eingetragen noch werden sie deployed.

| Extension | Status |
|-----------|--------|
| chars | Platzhalter (Leaderboards, nicht implementiert) |
| log_bridge | Durch inselkom ersetzt (erweiterter Log-Relay) |
| router_v2 | Components v2 Variante, per Feature-Flag steuerbar |
| rules | In extensions_meta gelistet, aber nicht in startup_cogs |

### Backup-Dateien (nur im Entwicklungsordner)

Lokale Sicherungskopien, existieren nicht auf dem Live-System.

| Datei | Original |
|-------|----------|
| donations.py_bu | donations.py |
| insurance.py_bu | insurance.py |
| statbot.py_bu | statbot.py |

---

## Gemeinsame Cog-Patterns

### Grundstruktur jeder Cog

```python
from discord.ext import commands
from _modules import app_config as cfg
from _modules.logging_utils import printlog

class MeineCog(commands.Cog):
    def __init__(self, bot):
        self.bot = bot

    # Slash-Commands via @app_commands oder Text via @commands.command
    # Listener via @commands.Cog.listener()
    # Background-Tasks via @tasks.loop()

async def setup(bot):
    await bot.add_cog(MeineCog(bot))
```

### Konfigurations-Zugriff

Alle Cogs importieren `_modules.app_config as cfg` und lesen daraus:
- **Pfade**: `cfg.database_file`, `cfg.GAMECHAT_DIR`, etc. (oder ueber `cfg.paths.*` Namespace)
- **Discord-IDs**: `cfg.main_server_id`, `cfg.discord_conf.channels.*`, `cfg.discord_conf.roles.*`
- **Tuning**: `cfg.tunables.*` (Reward-Rate, Cooldowns, Limits)
- **Feature-Flags**: `cfg.features.*` (z.B. `cfg.features.router_v2.enabled`)

### Datenbank-Zugriff

- Standard: `sqlite3` (synchron) oder `aiosqlite` (asynchron)
- DB-Pfade kommen aus `cfg` (z.B. `cfg.database_file`, `cfg.database_bank`)
- Kein ORM - direkte SQL-Statements
- Mehrere Cogs koennen dieselbe DB lesen, aber Schreibzugriffe sind in der Regel auf eine Cog konzentriert

### Logging

- `printlog()` aus `_modules.logging_utils` - einheitliches Format mit Zeitstempel
- Datei-Logs in `botlog/`

### Background-Tasks

Viele Cogs nutzen `@tasks.loop()` fuer periodische Aufgaben:
- `voice_rewards`: 60s/5min Tasks (Sprachzeit zaehlen, Gold verteilen)
- `cleanup_messages`: Nachrichten-Bereinigung in Intervallen
- `server_status`: BattleMetrics-Polling
- `maintenance`: 5-Min-Koordinator fuer mehrere Wartungsaufgaben
- `npczones`: 120s Status-Update-Loop
- `insurance`: Despawn-Countdown-Aktualisierung

### on_message-Listener

Mehrere Cogs reagieren auf Discord-Nachrichten in bestimmten Kanaelen:
- `game_events`: Parsed Log-Kanaele (Login, Lockpick, Airdrop, Killbox) -> Counter
- `trackingstation`: Flags-Parsing, Phasen-Tracking
- `statbot`: Nachrichten-Zaehler, Trade-Parsing
- `trade_stats`: Trade-Log-Parsing -> trades.db
- `kill_feed`: Kill-Logs -> formatierte Embeds
- `link_translator`: Koordinaten -> Kartenlinks
- `steam_watch`: Login-Parsing -> Spieler-/Namensaenderungen

### Persistente Embeds

Einige Cogs verwalten ein einzelnes, staendig aktualisiertes Embed:
- `identity`: Link-Embed mit Buttons (Message-ID in `ini/link_message_id.txt`)
- `bunker`: Bunker-Status (Message-ID in `ini/bunker_message_id.txt`)
- `npczones`: NPC-Zonen-Status (Message-ID in `ini/npczones_message_id.txt`)
- `router`: Persistentes UI mit Buttons + Dropdown
- `router_v2`: Components v2 Variante (Message-ID in `ini/router_v2_message_id.txt`)
- `rules`: Regelakzeptanz-Embed (Message-ID in `ini/rules_message_id.txt`)

---

## Implizite Abhaengigkeiten zwischen Cogs

Obwohl die Entkopplungsphilosophie direkte Abhaengigkeiten vermeidet, gibt es implizite Verbindungen:

### Shared Bot-Attribute

| Attribut | Gesetzt von | Gelesen von |
|----------|-------------|-------------|
| `bot.role_info` | titles | profile, awards |
| `bot.title_categories` | titles | profile |
| `bot._loaded_extensions_order` | loader.py | health |
| `bot._failed_extensions` | loader.py | health |

**Konsequenz**: `titles` muss vor `profile` geladen werden (ist in startup_cogs so konfiguriert).

### Geteilte Datenbanken

| Datenbank | Writer | Reader |
|-----------|--------|--------|
| userdata.db | identity, squad, game_events | basic, profile, char, chars, titles, health, insurance, history, u.a. |
| bank.db | bank, lizenzen, voice_rewards | health, audit |
| userstats.db | voice_rewards | health |
| persistent_stats.db | extern (Spiel/Importer) | char, chars, fishing, events_stats, health |
| event_stats.db | game_events | titles, events_stats, health |
| flags.db | trackingstation | purge |
| userhistory.db | history | health |
| payments.db | extern (payment_mail_watcher) | donations |

### on_message-Ketten

Wenn eine Nachricht in einem Log-Kanal erscheint, reagieren potenziell mehrere Cogs:
- `cfg.monitored_channel_ids` -> trackingstation, statbot (beide zaehlen/parsen)
- Trade-Kanaele -> game_events (Counter), trade_stats (DB), statbot (Dashboard)
- Login-Kanal -> game_events (C_1), steam_watch (neue Spieler)

Die Cogs operieren unabhaengig auf denselben Nachrichten - keine Cog wartet auf die andere.

### Datei-basierte Kommunikation

| Datei/Verzeichnis | Writer | Reader |
|-------------------|--------|--------|
| `gamechat/*.txt` | chat_bridge, bank, aircraft | Extern: streamtochat.py -> streamtochat.ahk |
| `ini/act_cars.txt` | Extern (onclipactions.ahk) | insurance, cars_uploader, vehicle_monitor |
| `ini/squads.txt` | Extern (onclipactions.ahk) | squad |
| `ini/players.txt` | Extern (onclipactions.ahk) | bank |
| `ini/flags.txt` | Extern (get_flags.ahk) | trackingstation |
| `events/Eventsperre.txt` | Extern (timer.py) | game_events, chat_bridge, health |
| `ini/Banksperre.txt` | bank (intern) | bank (intern) |
| `ini/temphooks.json` | hooks | hooks |

---

## Config-Namespaces (app_config.py)

Die zentrale Config ist in thematische Namespaces organisiert:

| Namespace | Inhalt | Beispiel |
|-----------|--------|---------|
| `cfg.paths.databases.*` | DB-Pfade | `cfg.paths.databases.userdata` |
| `cfg.paths.files.*` | Datei-Pfade | `cfg.paths.files.act_cars` |
| `cfg.discord_conf.channels.*` | Kanal-IDs | `cfg.discord_conf.channels.debug` |
| `cfg.discord_conf.roles.*` | Rollen-IDs | `cfg.discord_conf.roles.sponsor` |
| `cfg.discord_conf.squads.*` | Squad-Konfig | `cfg.discord_conf.squads.category_id` |
| `cfg.discord_conf.guild.*` | Guild-Konfig | `cfg.discord_conf.guild.whitelist` |
| `cfg.tunables.*` | Tuning-Parameter | `cfg.tunables.reward_rate` |
| `cfg.features.*` | Feature-Flags | `cfg.features.router_v2.enabled` |
| `cfg.urls.*` | Thumbnail-URLs | `cfg.urls.standard` |
| `cfg.logs.*` | Log-Relay-Konfig | `cfg.logs.folders`, `cfg.logs.skip_phrases` |
| `cfg.cleanup.*` | Message-Cleanup | `cfg.cleanup.purge_channels` |
| `cfg.tracking.*` | Tracking-Konfig | `cfg.tracking.trackchannels` |
| `cfg.audit.*` | Audit-Konfig | `cfg.audit.channel_posts_enabled` |
| `cfg.health.*` | Health-Konfig | `cfg.health.link_check_enabled` |
| `cfg.commands.*` | Befehl-Redirects | `cfg.commands.todo.enabled` |
| `cfg.team_sync.*` | Team-Sync-Konfig | `cfg.team_sync.channel_id` |

**Hinweis**: Aeltere Cogs nutzen teils noch die flachen Variablen (z.B. `cfg.database_file` statt `cfg.paths.databases.userdata`). Beide Formen sind verfuegbar - die Namespace-Variante ist die bevorzugte.

---

## Cog hinzufuegen oder entfernen

### Neue Cog aktivieren

1. Python-Datei in `_cogs/` erstellen (Grundstruktur siehe [Gemeinsame Patterns](#gemeinsame-cog-patterns))
2. In `_modules/app_config.py`:
   - Extension zu `startup_cogs` hinzufuegen (Position beachten bei Abhaengigkeiten)
   - Optional: Eintrag in `extensions_meta` fuer Konsolen-Uebersicht
3. Bot neu starten

### Cog deaktivieren

Zeile in `cfg.startup_cogs` auskommentieren (`#`). Die Cog-Datei bleibt erhalten, der Bot laedt sie einfach nicht mehr.

### Cog zur Laufzeit steuern

Der `health`-Cog bietet Supervisor-only Befehle zum Nachladen/Entladen:
- Reload einzelner Cogs
- Shutdown/Restart des gesamten Bots
- Diagnostik (welche Cogs geladen/fehlgeschlagen)
- Rate Limit Monitoring
- Fehlerberichte
