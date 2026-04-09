# Bereich: AutoHotkey-Automatisierung

## Übersicht

Die AHK-Skripte bilden die **Hardware-nahe Automatisierungsschicht** des Systems. Sie steuern Prozesse, parsen Roh-Logs, injizieren Chat-Nachrichten ins Spiel, orchestrieren Events und erfassen Spielserver-Daten über die Zwischenablage. Zusammen bilden sie die Brücke zwischen dem Spielserver-Fenster und dem Rest des Systems.

---

## Komponenten

### start.ahk — Einstiegspunkt des Gesamtsystems
**Master-Startup-Orchestrator** — startet alle Dienste in definierter Reihenfolge.

- Löscht Lock-Files beim Start (`Eventsperre.txt`, `Banksperre.txt`)
- Startet sequentiell mit Zeitverzögerungen:
  1. `systemstart.py` (55s Delay)
  2. `log_separator.ahk` (15s Delay)
  3. `bot_game-manager.py` (10s Delay)
  4. `streamtochat.ahk` (2.5s Delay)
  5. `streamtochat.py`

Die Reihenfolge ist bewusst: erst Bots, dann Log-Verarbeitung, dann Chat-Injection.

---

### log_separator.ahk — Zentraler Log-Parser
**Kern-Log-Verarbeitung** — parst und kategorisiert Roh-Serverlogs in 15+ Kanäle.

- **Endlos-Loop:** Überwacht `serverlogs\download\` für eingehende `.log`-Dateien
- **Kategorisierung:**

| Ausgabeordner | Inhalt |
|---|---|
| `chat-full/` | Vollständiger Chat mit Steam-IDs und Character-IDs |
| `chat-global/`, `chat-local/`, `chat-squad/`, `chat-admin/` | Chat nach Kanal |
| `chat-link/` | Steam-Linking-Integration |
| `chat-trigger/` | Spezifische Chat-Trigger |
| `login-solo/`, `logout-solo/` | Login/Logout-Events |
| `gameplay/` | Quests, Basebuilding, etc. |
| `economy/` | Trades, Banktransaktionen |
| `flagtracking/` | Flaggen-Events (Created/Destroyed/Overtaken) |
| `bunkerlock/` | Bunker-Lock/Unlock |
| `killbox/`, `airdrops/` | Gameplay-Events |
| `quests-ok/`, `tobacco/`, `bank/`, `basebuilding/`, `autokick/` | Spezialisierte Logs |

- **Features:**
  - Zeitzone-Korrektur (+1h Offset)
  - Rollen-basierte Farbkodierung (ANSI-Format für Discord) — liest `admins.txt` und `moderators.txt`
  - Trigger-Erkennung: z.B. "ich-stecke-fest" → startet `events\stuckport\stuck.ahk`

**Schlüsselrolle:** Speist alle Log-basierten Cogs (log_bridge, inselkom, trackingstation, kill_feed, game_events, statbot, etc.)

---

### streamtochat.ahk — Chat-Injection ins Spiel
**Discord → Spielchat** — tippt Nachrichten physisch ins Spielfenster.

- **Polling:** Prüft `gamechat\` alle 650ms auf neue `.tochat`-Dateien
- **Ablauf:**
  1. Aktiviert SCUM-Fenster
  2. Klickt an Position (800, 450)
  3. Öffnet Chat mit `{t}`
  4. Ctrl+A → Delete → Ctrl+V → Enter
- **Audit:** Loggt alle Nachrichten in `botlog\stc_*.log`
- **Aufräumen:** Löscht `.tochat`-Dateien nach Verarbeitung

---

### onclipactions.ahk — Clipboard-Datenerfassung
**Passive Datenerfassung** — überwacht Zwischenablage und sortiert Spieldaten.

- **OnClipboardChange-Hook:** Reagiert auf Clipboard-Änderungen
- **Routing nach Inhalt:**

| Clipboard enthält | Zieldatei | Modus |
|---|---|---|
| "Owner not online" | `data\cars.txt` + `ini\act_cars.txt` | Überschreiben |
| "MemberRank" | `ini\squads.txt` | Überschreiben |
| "Gold balance" | `ini\players.txt` | Überschreiben |
| "Flag ID" | `ini\flags.txt` | **Anhängen** (akkumuliert) |

- **Nebenaufgabe:** Klickt alle 10s "Gesponserte Sitzung"-Dialog weg

**Wichtig für Tracking:** Dies ist die Quelle der `flags.txt` — Flaggen werden über die Zwischenablage gesammelt und angehängt, daher das Mixed-Encoding und die Duplikate.

---

### get_flags.ahk — Flaggen-Refresh
**Flaggen-DB neu aufbauen** — löscht alte Daten und triggert Neuerstellung.

- Löscht `ini\flags.txt`
- Schreibt `#ListFlags` in Gamechat (fordert Flaggen-Liste vom Server an)
- Startet `scripts\build_flag_db.pyw` (Python: flags.txt → flags.db)

---

### eventmanager.ahk — Event-Orchestrator
**Spiel-Events steuern** — Timing, Zufallsauswahl, Ankündigungen, Start.

- **Blackout-Fenster:** 4x täglich (01:45–02:00, 05:45–06:00, etc.) — keine Events
- **Cooldown:** 5 Minuten zwischen Events
- **Ablauf:**
  1. Prüft Eventsperre und Cooldown
  2. Wählt zufälligen Event-Typ (keine Wiederholung des letzten)
  3. Wählt zufälligen Event-Namen
  4. Generiert Chat-Nachrichten aus Speech-Dateien (Begrüßung, Ankündigung, Countdown)
  5. Startet Event-Handler: `events\{eventArt}\{eventArt}.ahk`
- **Remote-Override:** Liest `eingang\event_art.txt` wenn von `remote.ahk` gesetzt

| Datei | Zweck |
|---|---|
| `events\Eventsperre.txt` | Lock während Event läuft |
| `events\last_event_time.txt` | Zeitstempel letztes Event |
| `events\last_event_art.txt` | Typ letztes Event |
| `events\eventarten.txt` | Verfügbare Event-Typen |
| `events\eventnamen.txt` | Event-Namen-Pool |
| `events\speech\*.txt` | Sprach-/Textbausteine |

---

### purge_checker.ahk — Purge-Auslöser
**Probabilistischer Event-Trigger** — prüft Punkte und würfelt Purge.

- **Punktesystem:** 6 Kategorien (Chat, Event, Lockpick, Login, Total, Trades) aus `events\purge\counter\count_*.txt`
- **Schwelle:** 10.000 Punkte
- **Wahrscheinlichkeit:** 50% Basis, -10% pro 1.000 Punkte über Schwelle
- **Bei Purge:** Startet `events\purge\purge.py`
- **Kein Purge:** Vergabe eines Trost-Airdrops via `ini\airdrop\drop.override`
- **Chat:** Ankündigung der Gesamtpunkte, Countdown, Ergebnis

---

### remote.ahk — Fernsteuerung
**Command-Dispatcher** — liest Befehlsdateien aus `remote\` alle 10s.

| Code | Aktion |
|---|---|
| 99 | System-Reboot |
| 1 | Restart `timer.ahk` |
| 2 | Kill + Restart `SAB.exe` |
| 9 | Zufälliges Event starten |
| 10 | Alle laufenden Events abbrechen |
| 11 | Event-Horde starten |
| 12 | Horde starten |
| 13 | Hordedrop starten |

- Löscht Befehlsdateien nach Verarbeitung
- Loggt alle Befehle in `log\`

---

### reboot.ahk — System-Neustart
**Graceful Reboot** — loggt Event und startet Windows nach 30s neu.

- Schreibt Reboot-Nachricht in `log\`
- `Shutdown, 6` (Windows-Restart)

---

### log_mover_sab.ahk — Log-Archivierung
**SAB-Log-Umzug** — verschiebt SAB-Logs in den zentralen Download-Ordner.

- Liest `.txt` aus `SAB_Logs\`
- Konvertiert zu UTF-16
- Schreibt nach `serverlogs\download\sab_*.log`
- Löscht Originale

Wird dann von `log_separator.ahk` weiterverarbeitet.

---

## Gesamtbild: AHK im System

```
                         start.ahk
                        (Einstiegspunkt)
                             |
              +--------------+--------------+
              |              |              |
        systemstart.py  log_separator.ahk  streamtochat.ahk
        (Python-Bots)   (Log-Parsing)      (Chat-Injection)
              |              |              |
              |              v              |
              |     serverlogs\download\    |
              |        ↓ Kategorisiert      |
              |     15+ Ausgabeordner       |
              |        ↓                    |
              |     log_bridge / inselkom   |
              |     (Cogs posten in Discord)|
              |                             |
              +-------- gamechat\ ----------+
                    (Dateien = Nachrichten)

onclipactions.ahk (passiv)          remote.ahk (10s Polling)
    ↓ Clipboard-Daten                   ↓ Befehlsdateien
    ini\flags.txt                       eventmanager.ahk
    ini\act_cars.txt                    purge_checker.ahk
    ini\squads.txt                      reboot.ahk
    ini\players.txt

log_mover_sab.ahk → serverlogs\download\ → log_separator.ahk
get_flags.ahk → gamechat\ (#ListFlags) → onclipactions.ahk → ini\flags.txt
```

## Kommunikation zwischen AHK und Python

Die Brücke zwischen den AHK-Skripten und den Python-Bots/Cogs läuft ausschließlich über **Dateien**:

| Richtung | Mechanismus |
|---|---|
| AHK → Python | Dateien in `serverlogs\*`, `ini\*.txt`, `events\*` |
| Python → AHK | Dateien in `gamechat\*.txt` (werden von streamtochat.ahk gelesen) |
| AHK → AHK | `eingang\event_art.txt`, `events\Eventsperre.txt`, `remote\*.txt` |

Keine direkte Prozesskommunikation — alles dateibasiert, bewusst entkoppelt.
