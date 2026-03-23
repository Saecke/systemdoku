# Bereich: Wirtschaft & Handel

## Übersicht

Das Wirtschaftssystem basiert auf einem Gold-Tresor (Bank), Lizenzen, einem Handelshaus (Auktionen), Fahrzeugversicherungen und einem Spendensystem. Alle Systeme sind voneinander unabhängig und teilen sich nur die zentrale Spieler-Zuordnung (`userdata.db`: Discord-ID <-> Steam-ID).

---

## Komponenten

### bank.py (Cog) + bank_db.py (Modul)
**Tresor-System** — Spieler können Gold einzahlen/abheben.

- **Befehl:** `/tresor` (Aliases: Bank, t, T) — DM-Flow
- **Standort-Prüfung:** Spieler muss sich physisch im Spiel an der VIP-Lounge befinden (Koordinaten-Abgleich mit Toleranz)
- **Cooldown:** 24h zwischen Ein-/Auszahlungen
- **Banksperre:** Lock-File-Mechanismus gegen gleichzeitige Zugriffe
- **Lizenzen:** Banker's License (l1) erlaubt bis zu 5 Transfers/Tag
- **Voice-Belohnungen:** Zeigt Status der Voice-Channel-Goldbelohnung an
- **Schreibt:** Gold-Befehle via `cfg.GAMECHAT_DIR` ins Spiel

| DB | Tabellen | Zugriff |
|---|---|---|
| `bank.db` | Konten, Aktionen, Benutzeraktionen, Lizenzen | Lesen + Schreiben |
| `userdata.db` | Spieler-Stammdaten | Lesen |
| `userstats.db` | Voice-Time | Lesen |

---

### handelshaus.py (Cog)
**Auktionshaus & Marktplatz** — Forum-basiertes Handelssystem.

- **Auktionen:** Startpreis, Sofortkauf, Snipe-Schutz (Verlängerung bei Gebot kurz vor Ende)
- **Festpreisverkauf:** Multi-Quantity-Käufe
- **Gebots-Staffelung:** 5% (niedrig), 3% (mittel), 1% (hoch)
- **Angebotslimits nach Rolle:** Sponsor=10, Booster=5, Stammtisch=4, Linked=3, Unlinked=1
- **Bilder-Upload:** Größenlimit, WebP-Kompression

| DB | Tabellen | Zugriff |
|---|---|---|
| `handelshaus.db` | auctions, bids, purchases | Lesen + Schreiben (aiosqlite) |

Eigenständig — keine direkte Verbindung zu bank.db oder Gold-System.

---

### trade_stats.py (Cog)
**Handelsstatistiken** — Parst Ingame-Trade-Logs aus Discord.

- **Listener:** `on_message` auf dem Trade-Logs-Kanal
- **Parsing:** Erkennt `[Trade]`-Zeilen, extrahiert Item, Menge, Preis, Kauf/Verkauf
- **Aggregation:** Pro-Item-Statistiken (total_purchased, total_sold, total_value)

| DB | Tabellen | Zugriff |
|---|---|---|
| `trades.db` | trades, item_stats | Lesen + Schreiben |

Reine Analytik — kein Einfluss auf Kontostände.

---

### insurance.py + check_insurance.py (Cogs)
**Fahrzeugversicherung & Fahrzeugübersicht** — KFZ-Verwaltung mit Despawn-Countdown.

- **Befehl:** `/kfz` (Aliases: auto, car) — zeigt Fahrzeuginfos (DM)
  - **Versicherte:** Versicherungsdetails + Despawn-Countdown + alle Fahrzeuge auf ihren Namen
  - **Nicht versichert, aber verlinkt:** Alle abgeschlossenen Fahrzeuge (via owner_steam_id/owner_db_id aus carservice.db)
  - **Fahrzeuglimit-Warnungen:** 2 ohne Lizenz, 3 mit Fuhrpark-Lizenz (l2), 5+ = schwerer Regelverstoß
  - **Hinweise:** Abgelaufene Lizenz → `/lizenzen` erneuern; Keine Versicherung → `/tresor` kaufen
- **Admin:** `/vers add/rem/list` — Versicherungsnummern (V#####) zuweisen/entfernen
- **Background:** Pollt `act_cars.txt` alle 5 Min + File-Monitor alle 5 Sek
- **Features:** Despawn-Countdown, Kartenlinks (scum-map.com) pro Fahrzeug, Fahrzeughistorie
- **Fahrzeug-Mappings:** Rager, WolfsWagen, Laika, Dirtbike, Cruiser, RIS, Traktor, Citybike, Mountainbike, Barba, Dinghy, Floß, Sidecar, Kinglet Duster, Kinglet Mariner

| DB | Tabellen | Zugriff |
|---|---|---|
| `userdata.db` | insurance_id, steam_id, gamer_id, vehicle_* Spalten | Lesen + Schreiben |
| `carservice.db` | vehicle_history, events, vehicles (owner_steam_id, owner_db_id) | Lesen (Fahrzeuge + Historie) |
| `bank.db` | Lizenzen (l2 = Fuhrpark-Lizenz) | Lesen (Lizenz-Check) |
| `act_cars.txt` | Aktive Fahrzeuge (Datei, deutsche Lokalzeit trotz Z-Suffix) | Lesen |

### vehicle_monitor.py (Cog)
**Fahrzeuglimit-Monitor** — postet Verstöße automatisch in einen Admin-Channel.

- **Trigger:** Änderung an `act_cars.txt`, max. 1x pro Stunde
- **Logik:** Gruppiert alle Fahrzeuge pro Owner aus carservice.db, prüft Fuhrpark-Lizenz (l2) aus bank.db
- **Verstöße:** >2 Fahrzeuge ohne Lizenz, >3 mit Lizenz, 5+ = schwerer Regelverstoß
- **Ausgabe:** Embed in Admin-Channel mit Spielername, SteamID, Lizenzstatus, Fahrzeugliste
- **Channel:** `cfg.discord_conf.channels.vehicle_monitor`

---

### lizenzen.py (Cog)
**Lizenzverwaltung** — 7-Tage-Lizenzen kaufen/verlängern.

- **Befehl:** `/lizenzen` (Aliases: license, l) — Lizenz-Kaufmenü (DM)
- **Auswahl per Reaktion:** 💳 Banker (l1), 🚗 Fleet (l2), 🏗️ Building (l3)
- **Kosten:** `cfg.gold_cost_per_license` Gold pro Lizenz
- **Laufzeit:** `cfg.license_duration_days` (Standard: 7 Tage)

| DB | Tabellen | Zugriff |
|---|---|---|
| `bank.db` | Konten (Gold abziehen), Lizenzen (Ablauf setzen), Aktionen (Log) | Lesen + Schreiben |
| `userdata.db` | Steam-ID-Lookup | Lesen |

---

### donations.py (Cog)
**Spenden & Sponsor-System** — PayPal-Integration, Sponsor-Rolle.

- **Befehl:** `/spende` — Spendenguide oder Transaktion einlösen (DM)
- **Admin:** `/sponsor` — aktive Sponsor-Zeiträume anzeigen
- **Umrechnung:** 30 Cent = 1 Tag Sponsor
- **Rollen:** Sponsor-Rolle (zeitlich begrenzt) + Stammtisch (permanent)
- **Background:** Prune-Task entfernt abgelaufene Sponsor-Rollen

| DB | Tabellen | Zugriff |
|---|---|---|
| `payment.db` | Transaktionen (von payment_mail_watcher) | Lesen |
| `donations.db` | sponsor_roles | Lesen + Schreiben |

---

## Datenflüsse

```
Spieler tippt /tresor
    → bank.py prüft Standort (players.txt)
    → bank.py prüft Cooldown (Benutzeraktionen)
    → bank.py prüft Lizenz (Lizenzen)
    → bank_db: Kontostand ändern + Aktion loggen
    → Spielchat-Befehl via GAMECHAT_DIR

Spieler tippt /lizenzen
    → lizenzen.py prüft Verlinkung (userdata.db)
    → Reaktions-Menü → Gold abziehen (bank.db)
    → Lizenz-Ablauf setzen (bank.db Lizenzen)

Trade im Spiel
    → Log landet in Discord-Kanal
    → trade_stats.py parst → trades.db
    → statbot.py parst → statbot.db (Wirtschafts-Übersicht)

PayPal-Zahlung
    → payment_mail_watcher.py → payment.db
    → Spieler löst mit /spende ein → donations.py
    → Sponsor-Rolle vergeben + donations.db
```

## Zentrale Verknüpfung

```
userdata.db (discord_id ↔ steam_id)
    ├── bank.db (Konten, Lizenzen via steam_id)
    ├── handelshaus.db (Angebote via discord_id)
    ├── carservice.db (Fahrzeuge via steam_id)
    ├── trades.db (Analytik, keine User-Referenz)
    ├── donations.db (Sponsor via discord_id)
    └── payment.db (Transaktionen, extern)
```
