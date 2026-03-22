# Bereich: Spieler & Identität

## Übersicht

Das Identitätssystem bildet die Brücke zwischen Discord und Spielserver. Kernstück ist die Steam-Discord-Verlinkung in `userdata.db` — fast alle anderen Systeme setzen diese Verlinkung voraus. Darauf aufbauend: Profil, Charakter-Statistiken, Log-Historie und Team-Verwaltung.

---

## Komponenten

### identity.py (Cog)
**Steam-Discord-Verlinkung** — das Fundament des gesamten Systems.

- **Persistentes UI:** Embed mit Buttons im Link-Kanal (wird beim Start sichergestellt)
- **Verlinkung:** Generiert 20-Zeichen-Verifizierungscode (5 Min gültig)
- **Ablauf:** Spieler gibt Code im Spiel ein → Server sendet Bestätigung in Discord-Kanal → identity.py verknüpft
- **Befehle:** `/link` (manuell), `/unlink` (mit Bestätigung)
- **Status:** Zeigt Verlinkungszeitpunkt, Steam-ID, Name, Rollen-Badges (Stammtisch, Sponsor)
- **UUID-Expiry-Loop:** Codes laufen nach 5 Min ab, mit 60s Vorwarnung

| DB / Datei | Zugriff |
|---|---|
| `userdata.db` (discord_id, steam_id, discord_username, linking_time) | Lesen + Schreiben |
| `cfg.LINK_MESSAGE_ID_FILE` (Message-ID für Embed) | Lesen + Schreiben |

---

### profile.py (Cog)
**Spielerprofil** — aggregiert Daten aus mehreren Quellen.

- **Befehl:** `/profil` (Aliases: profile, p, P)
- **Zeigt:** Punkte, erreichte/fehlende Titel, Squad, Lottery-Tickets, Event-Punkte, Sponsor-Status
- **Statistiken:** C_1–C_4 Counters, ST_1–ST_2 Handelswerte aus userdata.db
- **Fallback:** Bei unverknüpftem Account → Verweis auf `/link`

| DB | Tabellen | Zugriff |
|---|---|---|
| `userdata.db` | Spieler-Stammdaten, Counter, Squad-Info | Lesen |
| `lottery.db` | meta, players (Ticket-Anzahl) | Lesen |
| `event_stats.db` | event_stats (Event-Punkte) | Lesen |

Nutzt `role_info` aus dem Bot-Kontext (wird vom Titles-Cog befüllt).

---

### char.py (Cog)
**Charakter-Statistiken** — detaillierte Überlebenswerte.

- **Befehle:** `/char` (vollständig, mehrseitig), `/charx` (kompakte Vergleichsansicht)
- **Kategorien:** Allgemein, Kampfdetails, Tiere/Jagen, Bewegung, Handwerk/Loot, Körper/Überleben
- **100+ Statistiken** mit Einheiten (km, Stunden, kg, Headshots, etc.)
- **Mehrseitig:** Automatische Aufteilung bei >25 Feldern pro Embed

| DB | Tabellen | Zugriff |
|---|---|---|
| `userdata.db` | steam_id, player_name | Lesen |
| `persistent_stats.db` | survival_stats (100+ Spalten) | Lesen |

Rein lesend — keine Schreiboperationen.

---

### chars.py
**Leere Datei** — Platzhalter, vermutlich für server-weite Charakter-Leaderboards vorgesehen.

---

### history.py (Cog)
**Spieler-Log-Historie** — Ingame-Logs für Spieler abrufbar.

- **Listener:** `on_message` auf überwachten Kanälen — erfasst Logs die Steam-IDs enthalten
- **Speicherung:** Max 100 Logs pro Spieler (FIFO), auto-Löschung nach 7 Tagen
- **Datenschutz:** Fremde Steam-IDs werden verschleiert ("ID verschleiert")
- **Übersetzung:** Englische Game-Logs → Deutsch mit Formatierung
- **Befehl:** `/history` (Aliases: historie, log)

| DB | Tabellen | Zugriff |
|---|---|---|
| `userhistory.db` | user_logs (id, content, steam_id, timestamp) | Lesen + Schreiben |
| `userdata.db` | Steam-ID-Lookup | Lesen |

---

### link_translator.py (Cog)
**Koordinaten-Kartenlinks** — postet Kartenlinks aus dem Globalchat.

- **Background-Task:** Liest `chat.log` per Tail (kein Zurückspulen → keine Duplikate)
- **Erkennung:** Koordinaten aus "Global:"-Zeilen oder scum-map.com-Links
- **Ausgabe:** Standardisierter Kartenlink (scum-map.com) mit 5s Cooldown
- **Keine DB-Zugriffe** — rein dateibasiert

| Datei | Zugriff |
|---|---|
| `PP-Live-Logcrawler/logs/chat*.log` | Lesen (Tail-Only) |

---

### team_sync.py (Cog)
**Team-Roster-Verwaltung** — synchronisiert Team aus Discord-Nachricht.

- **Task:** 5-Minuten-Loop
- **Quelle:** JSON aus einer bestimmten Discord-Nachricht (gepinnt)
- **Format:** `{"admins": [{"name": "...", "id": "..."}], "moderators": [...]}`
- **Ausgabe:** Schreibt `team.ini`, `admins.txt` (UTF-16), `moderators.txt` (UTF-16)

Eigenständig — keine Verbindung zum Spieler-Identity-System.

---

### db.py (Modul)
**DB-Hilfsfunktion** — sichere Steam-ID-Abfrage.

- `get_steam_id_from_discord_id(database_path, discord_id)` — Read-Only-Zugriff
- Handles Typ-Mismatches (TEXT vs INTEGER in Discord-ID-Spalten)
- Einmalige Warnung pro DB-Pfad bei Fehlern

---

## Datenflüsse

```
Spieler klickt "Verlinken" im Discord
    → identity.py generiert UUID (5 Min gültig)
    → Spieler gibt UUID im Spiel ein
    → Server sendet Bestätigung → Discord-Kanal
    → identity.py schreibt in userdata.db (discord_id ↔ steam_id)

Spieler tippt /profil
    → profile.py liest userdata.db (Stammdaten + Counter)
    → profile.py liest lottery.db (Tickets)
    → profile.py liest event_stats.db (Event-Punkte)
    → profile.py liest role_info (Titel/Punkte vom Titles-Cog)
    → Aggregiertes Embed an Spieler

Spieler tippt /char
    → char.py liest userdata.db (steam_id)
    → char.py liest persistent_stats.db (100+ Werte)
    → Formatierte mehrseitige Embeds

Game-Log erscheint in Discord
    → history.py erkennt Steam-ID
    → Speichert in userhistory.db (max 100, FIFO)
    → Spieler fragt /history ab → Logs mit maskierten Fremd-IDs
```

## userdata.db als zentraler Knotenpunkt

```
                    userdata.db
                   (discord_id ↔ steam_id)
                        |
        +-------+-------+-------+-------+
        |       |       |       |       |
    identity  profile  char   history  bank
    (Schreib) (Lesen)  (Lesen) (Lesen) (Lesen)
        |       |
        |   lottery.db, event_stats.db
        |   persistent_stats.db
        |
    Alle weiteren Cogs die steam_id brauchen
```
