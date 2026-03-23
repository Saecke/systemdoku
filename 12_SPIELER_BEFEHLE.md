# Spieler-Eingaben & Befehle

## Übersicht

Alle Spieler-Interaktionen laufen über Discord (Befehle, Buttons, Dropdowns) oder das Web-Dashboard. Hier ist die komplette Übersicht dessen, was ein Spieler tun und sehen kann.

---

## Discord-Befehle (alle DM-only)

### Identität & Verlinkung

| Befehl | Aliases | Cog | Voraussetzung | Was passiert |
|---|---|---|---|---|
| `/link` | Link | identity.py | Jeder | Generiert 20-Zeichen-Code (5 Min gültig), Spieler gibt Code im Spiel ein |
| `/unlink` | Unlink | identity.py | Verlinkt | Entfernt Discord↔Steam-Verknüpfung (mit Bestätigung) |
| `/status` | state, State | basic.py | Jeder | Zeigt Verlinkungsstatus, Steam-ID, Spielername, Rollen-Badges |

### Profil & Statistiken

| Befehl | Aliases | Cog | Was der Spieler sieht |
|---|---|---|---|
| `/profil` | profile, p, P | profile.py | Punkte, erreichte/fehlende Titel, Squad, Lotterie-Tickets, Event-Punkte |
| `/char` | Stats, charakter, stats | char.py | Detaillierte Charakter-Statistiken in Kategorien (100+ Werte) |
| `/charx` | char+, compare | char.py | Eigene Werte vs. Server-Top mit Differenzen |
| `/chars` | Chars, topchars | char.py | Server-weite Bestwerte und Kronen-Topliste |
| `/fischen` | fish, fishing, fisch | fishing.py | Fänge nach Art (14 Arten), Rekorde, Angel-Leaderboard |
| `/awards` | — | awards.py | Top-10: Kronen (👑) + Angel-Awards (🎣) + Event-Trophäen (🏆) |

### Titel & Fortschritt

| Befehl | Aliases | Cog | Was der Spieler sieht |
|---|---|---|---|
| `/titel` | title, Titel | titles.py | Erreichte Titel mit Icons, fehlende Titel, Punkte |
| `/todo` | doto, Todo | titles.py | Fehlende Titel sortiert nach Punkten als Aufgabenliste |
| `/rangliste` | topliste, rang, top, rank | titles.py | Titel-Punkte-Leaderboard, eigene Position |

### Wirtschaft

| Befehl | Aliases | Cog | Was der Spieler sieht |
|---|---|---|---|
| `/tresor` | Bank, bank, t, T | bank.py | Gold-Kontostand, letzte 20 Transaktionen, Voice-Reward-Status |
| `/lizenzen` | license, lizenz, l, L | lizenzen.py | Aktive Lizenzen, Kaufmenü per Reaktion (💳🚗🏗️) |
| `/kfz` | auto, car, Car, KFZ | insurance.py | Fahrzeugtyp, ID, Kartenlinks, Despawn-Countdown, Versicherungsstatus, alle Fahrzeuge auf dem Namen, Fuhrpark-Lizenz-Warnungen (2/3/5+), `/tresor`-Hinweis für Nicht-Versicherte |
| `/flugzeug` | plane, flieger, Flieger | aircraft.py | Flugzeug-Auswahl, Preise, Kauf-Flow (Standort-Prüfung + Gold abziehen) |
| `/spende` | spenden | donations.py | Spendenguide, PayPal-Link, Transaktions-ID einlösen |

### Squad & Events

| Befehl | Aliases | Cog | Was der Spieler sieht |
|---|---|---|---|
| `/squad` | team, s, S | squad.py | Squad-Name, Mitglieder nach Rang, Handelsvolumen, Tresor, Purge-Status |
| `/purge` | zombie, Zombie | purge.py | Purge-Status + Squad-Leader: Registrierung, Flagge wechseln, Abmeldung |

### Logs & Hilfe

| Befehl | Aliases | Cog | Was der Spieler sieht |
|---|---|---|---|
| `/history` | historie, log, Log | history.py | Letzte 100 Game-Logs (fremde Steam-IDs verschleiert) |
| `/hilfe` | h, H, hilf | help.py | Befehlsübersicht mit Beschreibungen |

---

## Router — Persistentes UI (Buttons & Dropdown)

Im Link-Kanal steht ein permanentes Embed mit interaktiven Elementen. Alle Antworten sind **ephemeral** (nur der Spieler sieht sie).

### Buttons

| Button | Custom-ID | Funktion |
|---|---|---|
| Verlinkung starten | `router_btn_link` | Startet Link-Flow |
| Status | `router_btn_status` | Zeigt Verlinkungsstatus |
| Profil | `router_btn_profile` | Zeigt Profil |
| Hilfe | `router_btn_hilfe` | Zeigt Hilfe |
| Dashboard | `router_btn_dashboard` | Sendet Dashboard-Link |
| Spenden | `router_btn_spenden` | Sendet Spendeninfos per DM |

### Dropdown-Menü (14 Optionen)

Profil, Char, CharX, Chars, KFZ, Lizenzen, Squad, Tresor, Rangliste, Titel, Todo, Fischen, Events, Awards

→ Jede Option ruft intern die entsprechende Cog-Methode auf.

### Identity-Buttons (separates Embed)

| Button | Custom-ID | Funktion |
|---|---|---|
| Verlinkung starten | `persistent_link_button` | Generiert Verifizierungscode |
| Verlinkung prüfen | `persistent_status_button` | Zeigt Link-Status |

---

## Voice-Channel-Befehle (bot_voicechannel.py)

Text-Befehle im eigenen Voice-Channel-Text:

| Befehl | Funktion |
|---|---|
| `!add @User` | Mitglied permanent hinzufügen |
| `!remove @User` | Mitglied entfernen |
| `!guest @User [stunden]` | Temporärer Zugang (Standard: 1h) |
| `!unguest @User` | Gastzugang widerrufen |
| `!list` | Mitglieder anzeigen |
| `!kick @User` | In AFK-Channel verschieben |
| `!name NeuerName` | Channel umbenennen |

---

## Web-Dashboard (scumsaecke.de/dash/)

### Spieler-Eingaben

| Eingabe | Wo | Persistierung |
|---|---|---|
| Widget-Reihenfolge ändern | Char-Karten Modal | `user_prefs.db` (JSON) |
| Karten ein-/ausblenden | Char-Karten Modal | `user_prefs.db` (JSON) |
| Restart-Alert ein/aus | Einstellungen | `user_prefs.db` (JSON) |
| Alert-Sound & Lautstärke | Einstellungen | `user_prefs.db` (JSON) |
| Warnzeit (1–15 Min) | Einstellungen | `user_prefs.db` (JSON) |
| Log-Suche (Freitext) | Logs-Tab | Nur Client-seitig (JS) |

### Spieler-Ansichten (Tabs)

| Tab | Inhalt | Datenquellen |
|---|---|---|
| /profil | Name, Steam-ID, Counter, Handelsvolumen, Tickets, Basisattribute | userdata, lottery, event_stats, persistent_stats |
| /squad | Mitglieder, Ränge, Tresor, Purge-Status, Kontostand | userdata, squaddata, bank |
| /kfz | Fahrzeugtyp, Koordinaten, Kartenlinks, Despawn-Countdown, alle Fahrzeuge, Lizenz-Warnungen | userdata, carservice, bank |
| /char | 7 Stat-Kategorien, 50+ Werte, Kronen (👑) | persistent_stats |
| /chars | Server-Leaderboards für alle Kategorien | persistent_stats |
| /charx | Eigene Werte vs. Server-Top, Rang-Perzentil | persistent_stats, userdata |
| /fischen | 14 Fischarten, Rekorde, Leaderboard | persistent_stats |
| /events | Gewonnen/Verloren, Kills, Trophäen (🏆) | persistent_stats |
| /titel | 50+ Titel, Freischalt-Bedingungen, Fortschritt | persistent_stats, titles.ini |
| /logs | Letzte 100 Logs, Suche, Fremd-IDs verschleiert | userhistory |

→ Details siehe [11_DASHBOARD.md](11_DASHBOARD.md)

---

## Datenquellen pro Spieler-Feature

```
Spieler fragt /profil
    ← userdata.db (Name, Counter C_1–C_4, ST_1–ST_2)
    ← lottery.db (Tickets)
    ← event_stats.db (Event-Punkte)
    ← persistent_stats.db (Basisattribute)

Spieler fragt /char
    ← persistent_stats.db (survival_stats: 100+ Werte)

Spieler fragt /tresor
    ← bank.db (Kontostand, Transaktionen, Lizenzen)
    ← userstats.db (Voice-Time)
    ← userdata.db (Steam-ID)

Spieler fragt /squad
    ← userdata.db (Squad-Zuordnung, Mitglieder)
    ← bank.db (Squad-Tresor)
    ← flags.db (Purge-Registrierung)

Spieler fragt /kfz
    ← userdata.db (Versicherung, Fahrzeugdaten)
    ← carservice.db (Fahrzeughistorie)

Spieler fragt /history
    ← userhistory.db (Logs gefiltert auf eigene Steam-ID)

Dashboard /profil Tab
    ← userdata.db + lottery.db + event_stats.db + persistent_stats.db
    (gleiche Quellen wie Discord /profil, aber im Browser)
```

---

## Was Spieler NICHT können

- Keine direkten Stat-Änderungen (alles read-only aus dem Spiel)
- Keine Einsicht in andere Spieler-Logs (nur eigene)
- Keine Admin-Befehle (/health, /hc-*, /mapevent-*, /tracking-diag)
- Keine Versicherungsnummern-Verwaltung (/vers — nur Admins)
- Kein Zugriff auf Dev-Tools im Dashboard
- Keine Manipulation von Handelshausangeboten anderer Spieler
