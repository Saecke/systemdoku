# Zukunft: RCON-Integration

**Status: Geplant — wartet auf RCON-Support durch die SCUM-Entwickler**

## Kernentscheidung

Das bestehende System bleibt **vollständig unverändert**. RCON wird als alternativer Transport-Layer eingeführt, der exakt die gleichen `gamechat/*.txt`-Dateien verarbeitet wie das bisherige StreamToChat-System.

## Heutiger Weg (bleibt als Option erhalten)

```
Cog/Skript → gamechat/*.txt → streamtochat.py → .tochat → streamtochat.ahk → Spielfenster
```

## Zukünftiger RCON-Weg (zusätzlich)

```
Cog/Skript → gamechat/*.txt → rcon_engine → RCON-Socket → Spielserver
```

## Prinzip

- **Gleiche Dateien, anderer Verarbeiter.** Entweder StreamToChat oder RCON liest `gamechat/*.txt` — nicht beide gleichzeitig.
- Am bestehenden System ändert sich **nichts**. Kein Cog, kein Bot, kein Skript muss angepasst werden.
- Die Entscheidung ob StreamToChat oder RCON aktiv ist, liegt beim Betreiber (Config/Switch).
- AHK bleibt als **Fallback** verfügbar falls RCON ausfällt.

## Was NICHT umgestellt wird

- **Log-Pipeline bleibt:** FTP → Logcrawler → log_separator → Discord (funktioniert zuverlässig)
- **Daten-Lesen bleibt:** Clipboard/Dateien für flags.txt, act_cars.txt, squads.txt, players.txt
- **Alle Cogs bleiben:** Schreiben weiterhin in `gamechat/*.txt`

## Timing & Rate-Limiting

Kein neues Problem — ist bereits gelöst:
- `streamtochat.py` hat `<<<DELAY>>>`-Direktiven und ein 1x-pro-Sekunde-Limit
- RCON-Engine übernimmt exakt die gleichen Timings
- Falls RCON-API latent: kurz Pause — identisch zum heutigen Verhalten wenn das Spiel nicht reagiert

## Offline-Verhalten

Ebenfalls bereits gelöst:
- Heute: Game offline → Dateien stauen sich in `gamechat/`, Servicebots laufen weiter ohne Spieleingabe
- Mit RCON: RCON nicht erreichbar → gleiche Queue, gleicher Minimalbetrieb
- Kein neuer Fehlerfall — das System kennt "kein Spiel da" bereits

## Implementierung: `rcon_bridge/`

Das System ist gebaut und kann sofort im Shadow-Mode mitlaufen.

### Dateien

| Datei | Zweck |
|---|---|
| `rcon_bridge/main.py` | Entry-Point, FileWatcher, Queue, Processing-Loop, Fallback-Logik |
| `rcon_bridge/rcon_client.py` | MockRCONClient (Shadow) + RCONClient-Stub (für später) |
| `rcon_bridge/config.yaml` | Mode, RCON-Credentials, Timing, Fallback, Logging |

### Modi

| Mode | Verhalten | streamtochat aktiv? |
|---|---|---|
| `shadow` | Liest `.txt`, loggt was RCON senden würde, fasst Dateien nicht an | Ja (verarbeitet + löscht) |
| `active` | Liest `.txt`, sendet via RCON, löscht Dateien | Nein (nicht nötig) |
| `disabled` | Bridge aus, alles wie bisher | Ja |

### Fallback-Kette (Active-Mode)

```
Befehl aus gamechat/*.txt
    ↓
RCON verbunden? → Ja → RCON send()
                → Nein + Fallback enabled → .tochat schreiben → streamtochat.ahk
                → Nein + Fallback disabled → Befehl verloren (geloggt)
```

### Shadow-Mode Ablauf

```
gamechat/*.txt wird erstellt
    ↓
streamtochat.py liest + verarbeitet + löscht (wie bisher)
    ↓ parallel
rcon_bridge liest + loggt was es senden würde (MockRCONClient)
    → Log: Timestamp, Command, Pre/Post-Delay, Mock-Response
    → Datei wird NICHT gelöscht (streamtochat.py macht das)
```

### Starten

```bash
# Shadow-Mode (parallel zu streamtochat)
python rcon_bridge/main.py

# Mit anderer Config
python rcon_bridge/main.py --config pfad/zur/config.yaml
```

### Später: Echter RCON-Client

Wenn SCUM RCON unterstützt, wird nur `RCONClient` in `rcon_client.py` implementiert:
- TCP-Socket öffnen (vermutlich Valve Source RCON Protokoll)
- Auth mit Passwort
- Commands senden, Responses empfangen
- Alles andere bleibt identisch

---

## Warum das funktioniert

Die dateibasierte Entkopplung ([00_ARCHITEKTURPRINZIP.md](00_ARCHITEKTURPRINZIP.md)) macht den Transport-Layer austauschbar, ohne dass die darüberliegenden Systeme davon wissen müssen. `gamechat/*.txt` ist die stabile Schnittstelle — was dahinter passiert, ist dem Rest des Systems egal. Das System ist im Grunde seit 6 Jahren RCON-ready, ohne es zu wissen.
