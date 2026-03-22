# Bereich: Web-Dashboard (scumsaecke.de)

## Übersicht

Das Dashboard unter `scumsaecke.de/dash/` ist eine **rein lesende Visualisierung** aller Spieldaten. Spieler loggen sich per Discord OAuth ein und sehen ihre Statistiken, Titel, Squad-Infos und Logs. Alle Datenänderungen passieren ausschließlich über die Discord-Bots — das Dashboard schreibt keine Spieldaten.

**Stack:** PHP 7+ · SQLite · Bootstrap 5.3 · Vanilla JS · Discord OAuth 2.0 (PKCE)

---

## Authentifizierung

### Discord OAuth 2.0 mit PKCE
1. Spieler besucht `/login.php`
2. Redirect zu Discord OAuth (PKCE-Flow, kein Client-Secret nötig)
3. Callback: `/dash/includes/handlers/callback.php`
   - Tauscht Auth-Code gegen Token
   - Holt User-Info von Discord
   - Speichert `discord_id`, `discord_username`, `discord_avatar` in Session
   - Loggt Login in `ops.db` (IP anonymisiert, User-Agent)
4. Redirect zum Dashboard

### Session-Sicherheit
- `cookie_secure = 1` (nur HTTPS)
- `cookie_httponly = 1`
- `cookie_samesite = Lax`
- Session-Regeneration nach Login
- Logout löscht Session + Cookie

---

## Dashboard-Tabs (Spieler-Ansicht)

### /profil — Spielerprofil
**Was der Spieler sieht:**
- Spielername, Steam-ID
- Login-Counter (C_1)
- Handelsvolumen (ST_1: Verkäufe, ST_2: Einkäufe, Netto-Loot)
- Lotterie-Tickets (aktuell + gesamt)
- Event-Punkte
- Charakter-Basisattribute (Stärke, Konstitution, Geschicklichkeit, Intelligenz)
- Charakter-Extras (Fame-Points, Größe, Gewicht, Alter)

| Datenquelle | Tabelle | Felder |
|---|---|---|
| `userdata.db` | userdata | player_name, steam_id, C_1–C_4, ST_1–ST_2 |
| `lottery.db` | meta, players | ticket_count, tickets |
| `event_stats.db` | event_stats | event_points_count_overall |
| `persistent_stats.db` | prisoner_bases | BaseStrength/Constitution/Dexterity/Intelligence, fame_points, height, weight, age |

---

### /squad — Squad-Info
**Was der Spieler sieht:**
- Squad-Name, Mitgliederliste mit Rängen (Anführer=4, Unteroffizier=3, Mitglied=2, Anwärter=1)
- Squad-Handelsvolumen, Squad-Tresor
- Purge-Registrierungsstatus
- Eigener Kontostand

| Datenquelle | Tabelle | Felder |
|---|---|---|
| `userdata.db` | userdata | squad_id, squad_name, member_rank |
| `userdata.db` | squaddata | purch_1, sold_1, s_tresor |
| `bank.db` | Konten | kontostand |

---

### /kfz — Fahrzeugversicherung
**Was der Spieler sieht:**
- Fahrzeugtyp (Rager, WolfsWagen, Laika, etc.)
- Fahrzeug-ID, Koordinaten
- Letzte Synchronisation (Datum/Uhrzeit)
- Despawn-Countdown (14 Tage ab Sync)
- Versicherungsstatus

| Datenquelle | Tabelle | Felder |
|---|---|---|
| `userdata.db` | userdata | insurance_id, vehicle_id, vehicle_type, vehicle_date/time, vehicle_coord_x/y/z |

---

### /char — Charakter-Statistiken
**Was der Spieler sieht:**
- 7 Karten-Kategorien: Allgemein, Handwerk, Körper, Bewegung, Kampf, Tiere, Kronen
- Detaillierte Werte (bis 10 Dezimalstellen)
- Kronen-System (👑) wenn der Spieler den Server-Bestwert hält
- **Anpassbar:** Karten-Reihenfolge und Sichtbarkeit per Modal

| Datenquelle | Tabelle | Felder |
|---|---|---|
| `persistent_stats.db` | survival_stats | 50+ Spalten (kills, deaths, minutes_survived, animals_killed, ...) |

---

### /chars — Server-Leaderboards
**Was der Spieler sieht:**
- Server-weite Rankings für alle Stat-Kategorien
- Top-Spieler mit Namen und Werten
- Ausschluss konfigurierter Test-Steam-IDs

| Datenquelle | Tabelle |
|---|---|
| `persistent_stats.db` | survival_stats (alle Spieler) |

---

### /charx — Charakter-Vergleich
**Was der Spieler sieht:**
- Eigene Werte vs. Server-Top
- Rang-Perzentil

| Datenquelle | Tabelle |
|---|---|
| `persistent_stats.db` | survival_stats |
| `userdata.db` | userdata (Steam-ID-Mapping) |

---

### /fischen — Angel-Statistiken
**Was der Spieler sieht:**
- Fänge nach Fischart (14 Arten)
- Persönlicher Rekord (schwerstes, längstes)
- Angel-Leaderboard

| Datenquelle | Tabelle |
|---|---|
| `persistent_stats.db` | fishing_stats |

---

### /events — Event-Teilnahme
**Was der Spieler sieht:**
- Events gewonnen / verloren
- Gegner-Kills in Events
- Event-Trophäen (🏆)

| Datenquelle | Tabelle |
|---|---|
| `persistent_stats.db` | events_stats |

---

### /titel — Discord-Titel & Rollen
**Was der Spieler sieht:**
- 50+ freischaltbare Discord-Rollen
- Kategorien: Stärke, Konstitution, Geschicklichkeit, Intelligenz, Kampf, Sonstige
- Freischalt-Bedingungen (todo_description)
- Titel-Anzahl und Server-weite Erfolge
- Spezial-Titel: "GOTT UNTER STERBLICHEN" (≥95% aller Titel)

| Datenquelle | Tabelle / Datei |
|---|---|
| `persistent_stats.db` | discord_title_roles |
| `titles.ini` | Titel-Definitionen (Icons, Beschreibungen, Kategorien) |
| `special_roles.ini` | Discord-Rollen-IDs |

---

### /logs — Aktivitäts-Logs
**Was der Spieler sieht:**
- Letzte 100 Log-Einträge (gefiltert auf eigene Steam-ID)
- Echtzeit-Suche/Filter
- Fremde Steam-IDs werden als "Verschleiert" angezeigt

| Datenquelle | Tabelle |
|---|---|
| `userhistory.db` | user_logs (content, steam_id, timestamp) |

---

## Spieler-Eingaben im Dashboard

Das Dashboard ist **fast vollständig read-only**. Spieler können nur folgendes anpassen:

### 1. Widget-Anpassung (Char-Karten)
- Karten-Reihenfolge ändern (Hoch/Runter-Buttons)
- Karten-Kategorien ein-/ausblenden
- Gespeichert via POST → `/dash/includes/handlers/save_prefs.php`
- Persistiert in `user_prefs.db` als JSON

### 2. Restart-Alert-Einstellungen
- Server-Restart-Benachrichtigungen ein-/ausschalten
- Sound und Lautstärke wählen (0.0–1.0)
- Warnzeit einstellen (1–15 Min vor Restart)
- Persistiert in `user_prefs.db` als JSON

### 3. Log-Suche
- Client-seitige Echtzeit-Filterung (JavaScript)
- Keine Server-Anfrage — reine UI-Filterung

---

## Developer-Tools (nur Owner)

**Zugang:** Nur Discord-ID `<ADMIN_DISCORD_ID>`

### API-Endpoints

| Endpoint | Methode | Funktion |
|---|---|---|
| `/dash/api/impersonate.php` | POST | Dashboard eines anderen Spielers anzeigen (Discord-ID oder Steam-ID) |
| `/dash/api/toggle_sim.php` | POST | Simulationsflags setzen (not_linked, no_insurance, mastered, sponsor, etc.) |
| `/dash/api/ops_logs.php` | GET/POST | Login- & Audit-Logs durchsuchen (Filter, Datumsbereich, CSV-Export) |
| `/dash/api/ops_health.php` | GET | DB-Integritätsprüfung (PRAGMA integrity_check, Dateigrößen) |

### Simulations-Flags
Zum Testen der UI mit verschiedenen Zuständen: `not_linked`, `no_insurance`, `no_squad`, `no_vehicle`, `mastered`, `sponsor`, `stammtisch`, `god`, `affect_badges`

---

## Datenbanken im Dashboard

13 SQLite-Datenbanken in `/dash/includes/database/`:

| Datenbank | Größe | Hauptzweck | Befüllt von |
|---|---|---|---|
| `userdata.db` | 1.2 MB | Spieler-Stammdaten, Squads, Versicherung | identity.py, game_events.py, squad.py |
| `persistent_stats.db` | 5.7 MB | Survival/Fishing/Event-Stats, Titel | bot_titles.py |
| `bank.db` | 104 KB | Kontostände | bank.py, voice_rewards.py, lizenzen.py |
| `carservice.db` | 91 MB | Fahrzeughistorie | carservice.py |
| `event_stats.db` | 28 KB | Event-Teilnahme | events_stats.py |
| `flags.db` | 40 KB | Flaggen-Positionen | trackingstation.py |
| `historical_stats.db` | 20 MB | Historische Statistiken | handelsbericht.py, bot_titles.py |
| `lottery.db` | 388 KB | Lotterie-Tickets | bot_lottery.py |
| `ops.db` | 172 KB | Login-/Audit-Logs | Dashboard selbst |
| `statbot.db` | 772 KB | Bot-Statistiken | statbot.py |
| `userstats.db` | 100 KB | Voice-Time etc. | voice_rewards.py |
| `userhistory.db` | 8.0 MB | Spieler-Logs | history.py |
| `user_prefs.db` | 24 KB | Widget-Einstellungen | Dashboard selbst |

**Sync:** Die DBs werden von `tools/sync/rsync_push.py` per SSH/rsync auf den Webserver synchronisiert:
- **Intervall:** Alle 5 Sekunden (konfigurierbar)
- **Methode:** SQLite Backup-API → konsistenter Snapshot → rsync-over-SSH (Delta-Transfer, atomarer Rename)
- **Change-Detection:** Nur geänderte DBs werden gesynct (mtime/size-Vergleich inkl. WAL/SHM/Journal)
- **Fallback:** rsync → scp wenn rsync nicht verfügbar
- **SSH-Connection-Reuse:** ControlPersist=7200s (2h) — kein Handshake pro Sync
- **Ziel:** `<WEBSERVER>:/www/htdocs/.../dash/includes/database/`
- Das Dashboard öffnet die DBs read-only.

---

## Rollen & Badges

| Badge | Discord-Rolle-ID | Erkennung |
|---|---|---|
| Sponsor | <ROLE_SPONSOR> | Discord-Rollen-Check |
| Stammtisch | <ROLE_STAMMTISCH> | Discord-Rollen-Check |
| Mastered Character | — | Auto-Erkennung: alle Attribute/Skills auf 100% |
| Gott unter Sterblichen | <ROLE_GOTT> | ≥95% aller Titel freigeschaltet |

---

## Sicherheit

- **PKCE-OAuth:** Kein Client-Secret im Frontend
- **CSRF:** AJAX-Header + Same-Origin-Check auf APIs
- **SQL:** Prepared Statements (PDO)
- **IP-Anonymisierung:** Letztes Oktett in Logs entfernt
- **Impersonation:** Wird auditiert
- **Read-Only:** Keine Spieldaten-Modifikation möglich

---

## Datenfluss

```
Discord-Bots & AHK-Skripte
    ↓ schreiben in
ini/database/*.db (Originale, lokal)
    ↓ rsync_push.py (alle 5s, SQLite Backup-API → SSH/rsync)
    ↓ nur geänderte DBs, Delta-Transfer, atomarer Rename
scumsaecke.de/dash/includes/database/*.db (Webserver)
    ↓ gelesen von
Dashboard (PHP, Read-Only)
    ↓ angezeigt für
Spieler (Browser, nach OAuth-Login)
```

**Gesynchte Datenbanken (10 Stück):**
userdata.db, persistent_stats.db, lottery.db, event_stats.db, statbot.db, userstats.db, userhistory.db, flags.db, carservice.db, historical_stats.db

→ Datenquellen: Alle Cogs und Bots aus [02_WIRTSCHAFT.md](02_WIRTSCHAFT.md) bis [09_STANDALONE_BOTS.md](09_STANDALONE_BOTS.md)
→ `ops.db` und `user_prefs.db` sind die einzigen DBs die das Dashboard selbst schreibt
