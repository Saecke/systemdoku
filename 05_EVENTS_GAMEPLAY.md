# Bereich: Events & Gameplay

## Übersicht

Events und Gameplay umfassen alle spielinhaltlichen Systeme: Live-Statistik-Erfassung aus Serverlogs, Zombie-Purge, Bunker-Monitoring, Map-Event-Challenges, Angeln, Titel/Auszeichnungen, NPC-Zonen und Flugzeugkauf. Der zentrale Einstiegspunkt für Echtzeit-Daten ist `game_events.py`, das Serverlogs parst und Counter in `userdata.db` hochzählt.

---

## Komponenten

### game_events.py (Cog)
**Echtzeit-Log-Parser** — parst Serverlogs und aktualisiert Spieler-/Squad-Statistiken.

- **Listener:** `on_message` auf 4 Log-Kanälen:
  - **Login-Kanal:** `[Login]`-Nachrichten → `player_name`, `last_login`, `gamer_id`, Counter `C_1` (Logins)
  - **Lockpicking-Kanal:** Erfolgreiche Lockpicks → Counter `C_2`
  - **Airdrop-Kanal:** Airdrop-Erfolge → Counter `C_3`
  - **Killbox-Kanal:** Killbox-Erfolge → Counter `C_4`
- **Trade-Parsing:** Erkennt `[Trade]` in beliebigen Nachrichten → `ST_1` (Einkaufswert), `ST_2` (Verkaufswert)
- **Squad-Updates:** Aktualisiert auch Squad-Counter und Handelswerte in `squaddata`

| DB | Tabellen | Zugriff |
|---|---|---|
| `userdata.db` | userdata (C_1–C_4, ST_1–ST_2, player_name, last_login) | Schreiben |
| `userdata.db` | squaddata (Squad-Counter, Handelswerte) | Schreiben |

**Schlüsselrolle:** Speist Echtzeit-Daten in userdata, von denen char, awards, titles, profile abhängen.

---

### map_event_challenges.py (Cog)
**Map-Event-Challenges** — dynamische Item-Sammel-Herausforderungen mit Schwierigkeitssteigerung.

- **Mechanik:** Zufällige Items sammeln, Fortschritt wird aus Trade-Logs erfasst
- **Difficulty Ramping:** Automatische Schwierigkeitssteigerung über Runden
- **Beitrags-Tracking:** Pro Spieler (Steam-ID) und pro Runde
- **Befehle:** `/mapevent-start`, `/mapevent-reroll` (Supervisor-only)
- **Game-Integration:** Schreibt in Gamechat-Verzeichnis, triggert Event-Skripte bei Abschluss

| DB / Datei | Zugriff |
|---|---|
| `map_event_contributions.db` (rounds, contributions, item_total_uses, round_tasks) | Lesen + Schreiben |
| `ini/map_event.ini` + `ini/map_event.json` | Lesen (Konfiguration, Item-Aliases) |
| Economy-Logs (Tail) | Lesen (Live-Trade-Erkennung) |

---

### purge.py (Cog)
**Zombie-Purge-Registrierung** — Squad-Flaggen für Purge-Events anmelden.

- **Befehl:** `/purge` (Aliases: /zombie) — Squad-Leader können:
  - Squad registrieren (kostet 5 Gold aus Squad-Tresor)
  - Aktive Flagge wechseln
  - Abmelden
- **Validierung:** Prüft Squad-Mitgliedschaft, Leader-Rang, Flaggen-Besitz
- **Startup-Check:** Validiert Konsistenz der Registrierungsdateien beim Bot-Start

| DB / Datei | Zugriff |
|---|---|
| `userdata.db` (squad_id, member_rank, squad_name) | Lesen |
| `userdata.db` (s_tresor) | Schreiben (Gold abziehen) |
| `flags.db` | Lesen (Flaggen-Liste des Spielers) |
| `events/purge/ok_flags/` (Dateien pro steam_id) | Lesen + Schreiben |
| `events/purge/ok_flags/registered_flags/` (Koordinaten) | Lesen + Schreiben |

---

### bunker.py (Cog)
**Bunker-Status-Monitor** — verfolgt Lock/Unlock-Zustände verlassener Bunker.

- **Listener:** `on_message` auf Bunkerlock-Kanal → parst `[LogBunkerLock]`-Zeilen
- **Task:** `update_bunker_status` (60s) — generiert Status-Embed
- **Anzeige:** Lock-Status, verbleibende Lock-Zeit (2h pro Aktivierung)
- **Persistierung:** Status in `bunker_config.ini`, Message-ID in `bunker_message_id.txt`

Nutzt `bunker_func`-Modul für Parsing und Datenaufbereitung.

---

### fishing.py (Cog)
**Angel-Statistiken & Leaderboard** — persönliche und server-weite Angelwerte.

- **Befehl:** `/fisch` (Aliases: /fischen, /fish, /fishing) — Text-Command via `on_message`
- **14 Fischarten:** Bass, Catfish, Pike, Carp, Amur, Bleak, Chub, Ruffe, u.v.m.
- **Anzeige:** Persönliche Stats vs. Server-Top, Rang-Platzierung, Kronen (🎣) für Bestplatzierte
- **Admin-Ausschluss:** Steam-ID <ADMIN_STEAM_ID> wird aus Rankings ausgeschlossen

| DB | Tabellen | Zugriff |
|---|---|---|
| `persistent_stats.db` | fishing_stats (steam_id + 14 Fischarten-Spalten) | Lesen |
| `userdata.db` | Steam-ID-Mapping | Lesen |

---

### awards.py (Cog)
**Kombiniertes Leaderboard** — Top-10 aus Kronen, Angel-Awards und Event-Trophäen.

- **Befehl:** `/awards` — Text-Command via `on_message`
- **Drei Quellen:**
  - 👑 **Kronen** aus `survival_stats` (44+ Kategorien: Fame, Kills, Crafting, Movement, etc.)
  - 🎣 **Angel-Awards** aus `fishing_stats`
  - 🏆 **Trophäen** aus `events_stats` (Events gewonnen, Kills, CTF-Captures)
- **Gleichstände:** Werden korrekt behandelt (mehrere Spieler auf gleicher Platzierung)

| DB | Tabellen | Zugriff |
|---|---|---|
| `persistent_stats.db` | survival_stats, fishing_stats, events_stats | Lesen |
| `userdata.db` | Spielernamen | Lesen |

---

### titles.py (Cog)
**Titel- & Rang-System** — Progression, Discord-Rollen, Leaderboard.

- **Befehle:** `/titel` (persönlicher Fortschritt), `/todo` (fehlende Titel, DM), `/rangliste` (Server-Leaderboard)
- **Titel-Daten:** Aus `ini/titles.ini` (Name, Punkte, Kategorie, Icon, Beschreibung, Thumbnail)
- **Rollen-Sync:**
  - `on_member_update` — verkündet neue Titel/Rollen
  - `on_member_join` — synchronisiert Titel beim Beitritt
  - `on_member_remove` — räumt auf beim Verlassen
- **Special:** "Gott unter Sterblichen" ab 95% Titel-Abschluss (auto-granted)
- **role_info:** Wird in den Bot-Kontext geschrieben und von `profile.py` gelesen

| DB / Datei | Zugriff |
|---|---|
| `persistent_stats.db` (survival_stats) | Lesen (Punkte-Berechnung) |
| `userdata.db` | Lesen (Steam-ID-Mapping) |
| `ini/titles.ini` | Lesen (Titel-Metadaten) |
| `ini/special_roles.ini` | Lesen (persistente Spezial-Rollen) |

---

### npczones.py (Cog)
**NPC-Aktionszonen-Status** — Echtzeit-Anzeige aktiver NPC-Zonen.

- **Task:** `update_message` (120s) — aktualisiert persistentes Embed
- **Anzeige:** Aktive Zone, Zeitfenster, Spawn-Frequenz, Countdown (`<t:epoch:R>`)
- **Status-Checks:** System-Online via PID-Datei (< 30s Alter), Event-Sperren via `*sperre*`-Dateien

| Datei | Zugriff |
|---|---|
| `ini/npczones.ini` | Lesen (Zonen-Status) |
| `events/npczones/[zone]/config.ini` | Lesen (Zone-Settings) |
| `_pid/sys_online.pid` | Lesen (Online-Check) |
| `ini/npczones_message_id.txt` | Lesen + Schreiben (Message-ID) |

---

### aircraft.py (Cog)
**Flugzeugkauf** — Spieler können Flugzeuge kaufen und spawnen lassen.

- **Befehl:** `/flugzeug` (Aliases: /plane, /flieger) — DM-Interaktion:
  1. Standort-Prüfung (Spieler-Position)
  2. Flugzeug-Auswahl mit Preisen (Standard / Wasserflugzeug)
  3. Kontostand-Prüfung (Tresor > Spielkonto priorisiert)
  4. Gold abziehen
  5. Gamechat-Befehle: Teleport → Warten → Spawn
- **Concurrency:** Lock-File-Mechanismus gegen Doppelkäufe

| DB / Datei | Zugriff |
|---|---|
| `userdata.db` | Lesen (Verlinkung) |
| `bank.db` (via bank_db) | Schreiben (Gold abziehen) |
| `ini/aircraft/aircraft.ini` | Lesen (Preise, Spawn-Befehle, Koordinaten) |
| `cfg.players_file_path` | Lesen (Spieler-Position + Kontostand) |
| `cfg.GAMECHAT_DIR` | Schreiben (Spielbefehle) |

---

## Datenflüsse

```
Spielserver-Logs
    │
    ├── Login/Lockpick/Airdrop/Killbox → game_events.py → userdata.db (C_1–C_4, ST_1–ST_2)
    │                                                        ↓
    │                                              titles.py (Punkte berechnen)
    │                                              awards.py (Kronen vergeben)
    │                                              profile.py (Profil anzeigen)
    │
    ├── Trade-Logs → map_event_challenges.py → map_event_contributions.db
    │
    ├── BunkerLock-Logs → bunker.py → bunker_config.ini → Status-Embed
    │
    └── NPC-Zonen-Config → npczones.py → Status-Embed

Spieler-Aktionen (Discord)
    │
    ├── /purge → purge.py → flags.db + Squad-Tresor + Registrierungsdateien
    │
    ├── /flugzeug → aircraft.py → bank.db + Gamechat-Befehle
    │
    └── /awards, /titel, /fisch → Lesen aus persistent_stats.db
```

## Stat-Pipeline

```
Spielserver → Logs → Discord-Kanäle
    → game_events.py schreibt C_1–C_4, ST_1–ST_2 in userdata.db
    → persistent_stats.db wird extern befüllt (data_to_db.py / PP-Logcrawler)
        → char.py liest survival_stats (100+ Werte)
        → fishing.py liest fishing_stats (14 Fischarten)
        → awards.py liest survival_stats + fishing_stats + events_stats
        → titles.py liest survival_stats → berechnet Punkte → vergibt Discord-Rollen
```
