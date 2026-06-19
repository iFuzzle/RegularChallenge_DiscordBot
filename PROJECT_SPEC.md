# RegularChallenge Discord Bot — Projektbeschreibung & Spezifikation

> **Zweck dieses Dokuments**
> Dies ist die maßgebliche, ausführliche Projektbeschreibung für den
> *RegularChallenge Discord Bot*. Sie dient (1) als Referenz, gegen die spätere
> Implementierungen abgeglichen werden, und (2) als Orientierung für eine oder
> mehrere KI-Agenten (z.B. Claude-Code-Instanzen), die an Teilbereichen arbeiten.
>
> Alles, was hier als **MUSS / SOLL / KANN** markiert ist, folgt der üblichen
> RFC-2119-Bedeutung. Punkte, die noch offen sind, stehen gesammelt unter
> [§16 Offene Punkte & Annahmen](#16-offene-punkte--annahmen).

---

## Inhaltsverzeichnis

1. [Vision & Kurzbeschreibung](#1-vision--kurzbeschreibung)
2. [Glossar / Domänenbegriffe](#2-glossar--domänenbegriffe)
3. [Technologie-Stack & Begründung](#3-technologie-stack--begründung)
4. [Architektur-Überblick (Docker-Stack)](#4-architektur-überblick-docker-stack)
5. [Repository-Struktur](#5-repository-struktur)
6. [Datenmodell](#6-datenmodell)
7. [Challenge-Generierungs-Engine](#7-challenge-generierungs-engine)
8. [Perioden, Scheduling & Zeitzonen](#8-perioden-scheduling--zeitzonen)
9. [Einreichungs- & Verifizierungs-Flow](#9-einreichungs--verifizierungs-flow)
10. [Punkte, Ranglisten & Streaks](#10-punkte-ranglisten--streaks)
11. [Discord-Bot — Features & Commands](#11-discord-bot--features--commands)
12. [Web-Interface — Features & Auth](#12-web-interface--features--auth)
13. [Multi-Game & Multi-Guild Design](#13-multi-game--multi-guild-design)
14. [Konfiguration, Secrets & Deployment](#14-konfiguration-secrets--deployment)
15. [Roadmap / Phasen](#15-roadmap--phasen)
16. [Offene Punkte & Annahmen](#16-offene-punkte--annahmen)
17. [Anhang A: SBK-1-Datenpool (Seed, zu verifizieren)](#anhang-a-sbk-1-datenpool-seed-zu-verifizieren)
18. [Anhang B: Beispiel-Challenges](#anhang-b-beispiel-challenges)

---

## 1. Vision & Kurzbeschreibung

Ein Discord-Bot, der für die *Snowboard Kids*-Community regelmäßige
**Challenges** generiert, postet und deren Ergebnisse trackt. Spieler reichen
ihre erfolgreichen Runs (Video) ein, Moderatoren bestätigen oder lehnen sie ab,
und der Bot pflegt **Ranglisten** und ein kumulatives **Punktesystem**.

Begleitend gibt es eine **Website** im selben Docker-Stack mit:

- einem **Mod-Panel** zur Review von Einreichungen (confirm/deny), zum Setzen
  von Wertungs-Werten und zum Umsortieren von Ranglisten,
- öffentlichen **Leaderboards** und Challenge-Übersichten,
- (später) Pool-Verwaltung und administrativen Funktionen (z.B. Spieler
  ausschließen).

**Leitidee der Challenges** (nach McGynas Vorschlag, inspiriert von den Daily
Wumpa Challenges aus *CTR Nitro-Fueled*): Challenges entstehen prozedural aus
Bausteinen — **Character + Board + Level + Condition1 (+ optionale Bonus-
Condition2)** — mit skalierbarem Schwierigkeitsgrad. Beispiele:

- *Slash · Alpine 2 · Sunset Rock · keine Tricks · (Bonus) keine blauen Items*
- *Nancy · Star · Ninja Land · keine Wipeouts · (Bonus) triff 1 CPU mit Eis*

**Aktueller Scope:** Start mit **Snowboard Kids 1 (N64)**. Das System wird aber
von Anfang an **multi-game** und **multi-guild** ausgelegt, damit später weitere
Spiele und weitere Server ohne Umbau der Basis unterstützt werden können.

**Community-Kontext:** Aktive Mitglieder u.a. aus **Australien und Deutschland**.
Der Server ist aktuell klein/wenig aktiv → Start mit **Weekly + Monthly**
Challenges (keine Dailies vorerst).

---

## 2. Glossar / Domänenbegriffe

| Begriff | Bedeutung |
|---|---|
| **Game** | Ein unterstütztes Spiel, z.B. `sbk1` (Snowboard Kids 1). Liefert eigene Pools (Characters, Boards, Levels, Conditions). |
| **Guild** | Ein Discord-Server. Jede Guild kann eigene Spiele/Challenges aktiv haben. |
| **Component / Baustein** | Ein Element einer prozeduralen Challenge: Character, Board, Level oder Condition. |
| **Condition** | Eine Spielregel/Auflage (z.B. „keine Tricks", „triff alle 3 Gegner"). Kann *pass/fail* oder *numerisch* (skalierbar) sein. |
| **Challenge** | Eine konkrete, an eine Periode gebundene Aufgabe (generiert oder admin-geschrieben) für eine Guild + ein Game. |
| **Period / Periode** | Zeitfenster einer Challenge: `weekly`, `monthly` (später `daily`). |
| **Run / Submission** | Eine Einreichung eines Spielers zu einer Challenge (Video + optionaler Kommentar). |
| **Confirm / Deny** | Mod-Entscheidung über eine Submission. |
| **Metric / Wertungswert** | Optionaler, vom Mod gesetzter Zahlenwert pro Run (Zeit oder Score), je nach Challenge. |
| **Points / Punkte** | Kumulative Teilnahme-Punkte: **1 Punkt pro bestätigtem Run**. |
| **Streak** | Anzahl aufeinanderfolgender Perioden mit ≥1 bestätigtem Run (nur Anzeige, kein Bonus). |
| **Mod** | Nutzer mit Moderationsrechten (per Discord-Rolle ermittelt). |

---

## 3. Technologie-Stack & Begründung

| Bereich | Technologie | Begründung |
|---|---|---|
| **Bot** | **Python 3.12+**, [`discord.py`](https://discordpy.readthedocs.io/) (oder `py-cord`/`nextcord` als Alternative) | Reife Lib, Slash-Commands, Attachments, Buttons/Modals. |
| **Web-Backend** | **Python 3.12+**, **FastAPI** | Eine Sprache mit dem Bot; teilt das `core`-Paket (DB-Modelle, Domänenlogik). Async, schnell, gute OpenAPI-Doku. |
| **Web-Frontend** | **Jinja2 + HTMX** (server-rendered), minimal JS, [Pico.css](https://picocss.com/) oder Tailwind | Schlankes Mod-Panel ohne SPA-Overhead. Schnelle Iteration, eine Codebasis. |
| **ORM / Migrations** | **SQLAlchemy 2.0** (async) + **Alembic** | Geteilte Modelle für Bot und Web; versionierte Schema-Migrationen. |
| **Scheduling** | **APScheduler** (im Bot-Prozess) oder dedizierter Worker | Posten/Abschließen von Perioden, UTC-basiert. |
| **Datenbank** | **PostgreSQL 16** | Robust, relational, Named Volume für Persistenz. |
| **Reverse Proxy / HTTPS** | **Caddy 2** | Automatisches Let's-Encrypt-HTTPS, minimale Konfiguration. |
| **Auth (Web)** | **Discord OAuth2** + Rollen-Abgleich gegen die Guild | Kein separates Passwort; Mod-Rolle im Guild = Zugang. |
| **Containerisierung** | **Docker** + **Docker Compose** | Reproduzierbarer Stack, einfaches Deployment. |
| **Video-Nachweis** | **Nur YouTube-Links** (keine Datei-Speicherung) | Discord-CDN-Links für hochgeladene Dateien laufen ab; es sollen **keine Videodateien** auf dem Server gehalten werden. |
| **Dev-Tooling** | `ruff` (lint+format), `pytest`, `mypy` (optional), `pre-commit` | Konsistente Codequalität. |

> **Hinweis zu Modellen/KI:** Aktuell **kein LLM** in der Challenge-Generierung
> (rein prozedural + Admin-Freitext). Falls später doch KI-gestützte
> Challenge-Texte gewünscht sind, sollten die aktuellsten Claude-Modelle
> (z.B. Opus/Sonnet der Claude-4.x-Familie) verwendet werden — als optionaler,
> klar gekapselter Generator-Provider.

---

## 4. Architektur-Überblick (Docker-Stack)

```
                       ┌─────────────────────────────┐
   Discord  ◀────────▶ │  bot       (Python)         │
                       │  - Slash-Commands            │
                       │  - Posting/Scheduler         │
                       └──────────────┬──────────────┘
                                      │  (shared core: models, domain)
   Browser ──HTTPS──▶ ┌──────────────┴──────────────┐
            ┌────────▶│  caddy     (reverse proxy)   │
            │         └──────────────┬──────────────┘
            │                        │
            │         ┌──────────────┴──────────────┐
            │         │  web       (FastAPI+HTMX)    │
            │         │  - Mod-Panel                 │
            │         │  - Public Leaderboards       │
            │         │  - Spieler-Profile (OAuth)   │
            │         │  - Discord OAuth2            │
            │         └──────────────┬──────────────┘
            │                        │
            │         ┌──────────────┴──────────────┐
            └────────▶│  db        (PostgreSQL)      │  ──▶ named volume: pgdata
                      └─────────────────────────────┘

                       Named Volumes: pgdata, caddy_data, caddy_config
```

**Services (docker-compose):**

| Service | Aufgabe | Abhängigkeiten | Volumes |
|---|---|---|---|
| `db` | PostgreSQL | – | `pgdata` |
| `bot` | Discord-Bot + Scheduler | `db` | – |
| `web` | FastAPI + HTMX | `db` | – |
| `caddy` | Reverse Proxy, Auto-HTTPS | `web` | `caddy_data`, `caddy_config` |

- `bot` und `web` teilen sich das **`core`-Paket** (gleiche SQLAlchemy-Modelle &
  Domänenlogik), laufen aber als **getrennte Prozesse/Container**.
- **Migrations** (`alembic upgrade head`) laufen als Init-Schritt vor `bot`/`web`
  (z.B. eigener `migrate`-Service oder Entrypoint-Hook). Doppelte Ausführung MUSS
  idempotent/abgesichert sein (advisory lock).
- **Keine Medien-Persistenz:** Es werden **keine Videodateien** gespeichert; als
  Nachweis dienen ausschließlich **YouTube-Links** (siehe §9).

---

## 5. Repository-Struktur

Monorepo mit gemeinsamem `core`-Paket:

```
RegularChallenge_DiscordBot/
├── PROJECT_SPEC.md            # dieses Dokument
├── README.md
├── pyproject.toml             # Workspace / Tooling (ruff, pytest, ...)
├── .env.example               # alle benötigten Env-Variablen (ohne Werte)
├── docker-compose.yml
├── docker-compose.override.yml# lokale Dev-Overrides (optional)
│
├── core/                      # geteiltes Paket (installierbar, z.B. "rc_core")
│   ├── models/                # SQLAlchemy-Modelle (siehe §6)
│   ├── db.py                  # Engine, Session-Factory
│   ├── domain/
│   │   ├── generation.py      # Challenge-Generator (§7)
│   │   ├── periods.py         # Perioden-Berechnung, UTC (§8)
│   │   ├── scoring.py         # Punkte/Streaks (§10)
│   │   └── submissions.py     # Submission-Parsing/Validierung (§9)
│   └── config.py              # zentrales Settings-Objekt (pydantic-settings)
│
├── bot/
│   ├── main.py                # Bot-Entrypoint
│   ├── cogs/                  # Command-Gruppen (submit, challenge, leaderboard, …)
│   ├── scheduler.py           # APScheduler-Jobs (posten/abschließen)
│   └── views/                 # Buttons/Modals (z.B. Submit-UI)
│
├── web/
│   ├── main.py                # FastAPI-App
│   ├── auth/                  # Discord OAuth2 + Session + Rollen-Check
│   ├── routes/                # mod_panel, public, api
│   ├── templates/             # Jinja2 (HTMX)
│   └── static/
│
├── migrations/                # Alembic
│   ├── env.py
│   └── versions/
│
├── deploy/
│   ├── Caddyfile
│   └── entrypoints/           # migrate.sh, wait-for-db.sh
│
├── data/
│   └── seeds/                 # Game-Daten (sbk1: characters/boards/levels/conditions)
│
└── tests/
    ├── unit/
    └── integration/
```

> **Paketierung:** `core` als installierbares Paket (`pip install -e ./core`),
> damit `bot` und `web` identische Modelle importieren. Keine Code-Duplizierung.

---

## 6. Datenmodell

> Alle Zeitstempel **MUSS** in **UTC** (`timestamptz`) gespeichert werden.
> IDs als `bigint`/`uuid` (Konvention im Repo festlegen — Empfehlung: `bigint`
> Surrogate-Keys + separate Discord-Snowflake-Spalten).

### 6.1 Kern-Entitäten

**`guilds`** — Discord-Server
- `id` (PK), `discord_guild_id` (unique), `name`
- `timezone` (IANA, nur für Anzeige/Anchor; Berechnung in UTC) — Default `UTC`
- `announce_channel_id`, `submit_channel_id` (nullable)
- `config` (JSONB: feature flags, posting-Zeit, Mod-Rollen-IDs …)
- `created_at`

**`games`** — unterstützte Spiele
- `id` (PK), `key` (z.B. `sbk1`, unique), `name`, `description`, `is_active`

**`guild_games`** — welche Spiele eine Guild fährt (n:m)
- `guild_id` (FK), `game_id` (FK), `enabled`
- PK = (`guild_id`, `game_id`)

**`users`** — globale Spieler (über Guilds hinweg)
- `id` (PK), `discord_user_id` (unique), `display_name`, `avatar_url`, `created_at`

**`guild_members`** — Mitgliedschaft + Rollenstatus (pro Guild)
- `guild_id` (FK), `user_id` (FK)
- `is_mod` (cache aus Discord-Rollen), `is_banned` (Default false; **Phase 3**)
- `roles_synced_at`
- PK = (`guild_id`, `user_id`)

### 6.2 Component-Pools (pro Game, **admin-pflegbar**)

**`characters`** — `id`, `game_id` (FK), `name`, `is_active`, `meta` (JSONB)

**`boards`** — `id`, `game_id` (FK), `name`, `is_active`, `meta` (JSONB)

**`levels`**
- `id`, `game_id` (FK), `name`, `is_active`
- `attributes` (JSONB): z.B. `{ "has_fall_off_zone": true, "lap_based": true }`
  → genutzt für Condition-Constraints (level-spezifische Conditions)

**`conditions`** — der wiederverwendbare Regel-Pool
- `id`, `game_id` (FK), `key` (slug), `is_active`
- `template_text` (z.B. `"Fall off the map {n} time(s)"`)
- `type`: `pass_fail` | `numeric`
- `numeric_min`, `numeric_max`, `numeric_unit` (nur bei `numeric`)
- `difficulty_scaling` (JSONB): Mapping `easy/medium/hard → Wert/Range`
- `bonus_eligible` (bool): darf als Condition2/Bonus gezogen werden
- `requires` (JSONB Constraints): z.B.
  `{ "level_attribute": "has_fall_off_zone" }`,
  `{ "excludes_conditions": ["no_jumping"] }`
- `weight` (int, Default 1): Ziehwahrscheinlichkeit
- `applies_to_metric` (optional Hinweis: `time` | `score` | `none`)

### 6.3 Challenges & Overrides

**`challenges`** — konkrete, periodengebundene Aufgabe
- `id` (PK), `guild_id` (FK), `game_id` (FK)
- `period_type`: `weekly` | `monthly` | `daily`
- `starts_at`, `ends_at` (UTC, inklusiv/exklusiv klar definieren)
- **Bausteine** (nullable bei reinem Freitext):
  `character_id`, `board_id`, `level_id`, `condition1_id`, `condition2_id`
- `difficulty`: `easy` | `medium` | `hard`
- `condition1_value`, `condition2_value` (konkretisierte numerische Werte)
- `source`: `generated` | `custom`
- `custom_text` (nullable; bei `source = custom` der vollständige Freitext)
- `rendered_text` (denormalisierter Anzeigetext, generiert beim Erstellen)
- `metric_type`: `time` | `score` | `none` (Wertungsdimension der Challenge)
- `default_sort`: `submitted_at_asc` (Default) | `metric_asc` | `metric_desc`
- `status`: `scheduled` | `active` | `closed`
- `created_by` (nullable; bei custom: Mod-User), `created_at`
- `announce_message_id` (nullable; Referenz auf Discord-Post)
- **Unique-Constraint:** (`guild_id`, `game_id`, `period_type`, `starts_at`)
  → maximal eine Challenge pro Game/Periode/Guild.

**`scheduled_custom_challenges`** — Admin-Overrides im Voraus
- `id`, `guild_id` (FK), `game_id` (FK), `period_type`
- `target_starts_at` (welche Periode überschrieben wird)
- Entweder Freitext (`custom_text`) **oder** fixierte Bausteine
- `created_by`, `created_at`
- Beim Generieren einer Periode: existiert hier ein Eintrag → **schlägt den
  Generator** (siehe §7.3).

### 6.4 Einreichungen & Wertung

**`submissions`** — Runs
- `id` (PK), `challenge_id` (FK), `user_id` (FK), `guild_id` (FK, denormalisiert)
- `video_url` (**MUSS**; YouTube-Link, RegEx-validiert) — einziger Nachweis
- `comment` (Text nach dem Link; wird Mods angezeigt)
- `status`: `pending` | `confirmed` | `denied`
- `metric_value` (numerisch, nullable; **vom Mod** gesetzt, je nach Challenge)
- `mod_id` (nullable, FK users), `mod_note` (nullable)
- `submitted_at`, `reviewed_at` (nullable)
- **Constraint:** `video_url` ist Pflicht und MUSS dem YouTube-Muster (§9.1)
  entsprechen. **Keine** Datei-/Medienspeicherung.
- **Mehrfach-Einreichungen:** erlaubt; Policy in §9 (welcher Run zählt für
  Punkte/Rangliste).

**`points_ledger`** — kumulative Punkte (Append-only Audit)
- `id` (PK), `user_id` (FK), `guild_id` (FK), `game_id` (FK)
- `season` (int, z.B. Jahr `2026`) — für jährlichen Reset / archivierte Seasons
- `points` (Default 1), `challenge_id` (FK)
- `awarded_at`
- **Regel:** Genau **1 Punkt pro Spieler pro Challenge** (Unique-Constraint über
  (`user_id`, `challenge_id`) erzwingt das, unabhängig von der Zahl der
  Einreichungen). Bei `deny`/Revert des wertenden Runs wird der Ledger-Eintrag
  entfernt/storniert (siehe §10).

**`mod_actions`** — Audit-Log (alle Mod-Entscheidungen)
- `id`, `guild_id`, `mod_id`, `action` (`confirm`/`deny`/`set_metric`/`resort`/
  `ban`/`unban`/`override` …), `target_type`, `target_id`, `payload` (JSONB),
  `created_at`

**`web_sessions`** — Web-Auth (oder über signierte Cookies / Server-Side Store)
- `id`, `user_id`, `expires_at`, `data` (JSONB)

### 6.5 Ableitungen (keine eigenen Tabellen, ggf. Views/Queries)

- **Per-Challenge-Rangliste:** bestätigte `submissions` einer Challenge,
  sortiert nach `default_sort` (Mod kann live umsortieren, §10).
- **Punkte-Leaderboard:** Summe `points_ledger` je `user` (× Guild × Game ×
  Season; Periodenfilter ableitbar).
- **Streak:** aus bestätigten Submissions je Spieler über aufeinanderfolgende
  Perioden berechnet.

---

## 7. Challenge-Generierungs-Engine

### 7.1 Ziel

Für eine `(guild, game, period_type, target_starts_at)`-Kombination genau **eine**
Challenge erzeugen — entweder aus einem Admin-Override (Vorrang) oder prozedural.

### 7.2 Prozedurales Schema (McGyna-Baukasten)

Eine generierte Challenge besteht aus:

```
Character  +  Board  +  Level  +  Condition1 (Pflicht)  [+ Condition2 (Bonus, optional)]
```

mit einem **Difficulty-Tier** (`easy` / `medium` / `hard`), das numerische
Conditions skaliert.

**Algorithmus (Pseudocode):**

```python
def generate_challenge(guild, game, period_type, starts_at, ends_at):
    # 1) Override-Vorrang
    override = find_scheduled_override(guild, game, period_type, starts_at)
    if override:
        return build_from_override(override)

    rng = seeded_rng(guild.id, game.id, period_type, starts_at)  # reproduzierbar

    # 2) Bausteine ziehen (nur is_active)
    character = rng.choice(active_characters(game))
    board     = rng.choice(active_boards(game))
    level     = rng.choice(active_levels(game))

    # 3) Difficulty wählen (gewichtet, z.B. easy 40 / medium 40 / hard 20)
    difficulty = weighted_choice(rng, DIFFICULTY_WEIGHTS)

    # 4) Condition1 ziehen, die zum Level passt (requires erfüllt)
    cond1 = weighted_choice(
        rng,
        eligible_conditions(game, level, exclude=[], bonus_only=False),
    )
    cond1_value = resolve_numeric(cond1, difficulty, rng)

    # 5) Optional Condition2 (Bonus), kompatibel mit cond1 & level
    cond2 = cond2_value = None
    if rng.random() < CONDITION2_PROBABILITY:
        cond2 = weighted_choice(
            rng,
            eligible_conditions(game, level, exclude=[cond1], bonus_only=True),
        )
        if cond2:
            cond2_value = resolve_numeric(cond2, difficulty, rng)

    # 6) Anti-Repeat: jüngste N Perioden meiden (Re-Roll bei Kollision)
    if collides_with_recent(...):
        return generate_challenge(...)  # bounded retries

    return assemble(character, board, level, cond1, cond1_value,
                    cond2, cond2_value, difficulty, metric_type, ...)
```

### 7.3 Constraints & Regeln (MUSS)

- **Level-Kompatibilität:** Eine Condition mit `requires.level_attribute` darf nur
  gezogen werden, wenn das Level dieses Attribut besitzt (z.B. „fall off the map"
  nur bei `has_fall_off_zone`).
- **Gegenseitiger Ausschluss:** `requires.excludes_conditions` verhindert
  widersprüchliche Kombis (z.B. „no jumping" + „2 unique special tricks").
- **Bonus-Eligibilität:** Condition2 nur aus `bonus_eligible = true`.
- **Difficulty-Scaling:** numerische Conditions ziehen ihren Wert aus
  `difficulty_scaling[difficulty]` (z.B. „hit every CPU {n}×": easy=1, med=2,
  hard=3).
- **Reproduzierbarkeit:** Generierung MUSS deterministisch aus einem Seed
  `(guild, game, period_type, starts_at)` ableitbar sein (Re-Runs erzeugen
  dasselbe Ergebnis, solange Pools unverändert sind).
- **Anti-Wiederholung:** Die letzten *N* Perioden (konfigurierbar) SOLLEN bei
  Level/Condition-Kombinationen gemieden werden.
- **`rendered_text`** wird beim Erstellen denormalisiert gespeichert (stabile
  Anzeige, auch wenn Pools sich später ändern).

### 7.4 Admin-Overrides (Freitext)

- Mods/Admins KÖNNEN über das Web-Panel für eine bestimmte zukünftige Periode
  eine Challenge **vorab festlegen** — entweder als **Freitext** oder mit fix
  gewählten Bausteinen.
- Beim Periodenstart hat der Override **Vorrang** vor dem Generator.
- `metric_type` und `default_sort` sind auch bei Custom-Challenges setzbar.

---

## 8. Perioden, Scheduling & Zeitzonen

### 8.1 Zeitzonen-Grundsatz (wichtig)

Community in **Australien + Deutschland** → Perioden MÜSSEN **global gleichzeitig**
beginnen/enden. Deshalb:

- **Alle Perioden-Grenzen werden in UTC berechnet und gespeichert.**
- Anzeige KANN pro Nutzer/Guild lokalisiert werden, die *Logik* bleibt UTC.

### 8.2 Periodendefinitionen (festgelegt)

**Anker-Uhrzeit = 20:00 UTC** (fairer Kompromiss USA / DE / AU, siehe §8.4).
Periodengrenze **und** Posting fallen auf diesen Anker zusammen, alles in UTC,
für alle Mitglieder identisch. Konfigurierbar pro Guild über `guilds.config`.

- **Weekly:** `[Montag 20:00 UTC, nächster Montag 20:00 UTC)`.
- **Monthly:** `[1. des Monats 20:00 UTC, 1. des Folgemonats 20:00 UTC)`.
- **Daily** (*später, Phase 3*): `[20:00 UTC, nächster Tag 20:00 UTC)`.
- **Season (jährlich, §10.1):** `[1. Jan 20:00 UTC, 1. Jan 20:00 UTC Folgejahr)`.

> Hinweis: Da Grenze = Posting, gibt es **kein** Zeitfenster, in dem eine
> Challenge aktiv aber noch nicht angekündigt ist.

### 8.3 Scheduler-Jobs (APScheduler, UTC)

1. **`open_period`** — zum Periodenstart: für jede `guild_game` mit passendem
   period_type die Challenge generieren (oder Override anwenden), Status `active`
   setzen und im `announce_channel` posten.
2. **`close_period`** — zum Periodenende: Challenge auf `closed`, finale
   Rangliste posten/„einfrieren".
3. **`sync_roles`** (periodisch) — Mod-/Ban-Status aus Discord cachen.
4. **`reconcile`** — Sicherheitsnetz: beim Bot-Start verpasste Perioden
   nachholen (z.B. nach Downtime), idempotent über die Unique-Constraint.

**Posting-Zeitpunkt:** = Periodenstart **20:00 UTC** (Grenze und Posting fallen
zusammen). Über `guilds.config` anpassbar; bleibt für alle Mitglieder identisch.

### 8.4 Wahl der Anker-Uhrzeit (20:00 UTC)

Es gibt keine Uhrzeit, die für USA, DE und AU gleichzeitig „Tag" ist. **20:00 UTC**
ist der fairste Kompromiss (niemand in der Tiefschlafphase):

| Region | Lokalzeit bei 20:00 UTC |
|---|---|
| US-Pazifik (UTC-8/-7) | 12:00–13:00 (Mittag) |
| US-Ostküste (UTC-5/-4) | 15:00–16:00 (Nachmittag) |
| Deutschland (UTC+1/+2) | 21:00–22:00 (Abend) |
| AU-Ostküste (UTC+10/+11) | 06:00–07:00 (früher Morgen) |

---

## 9. Einreichungs- & Verifizierungs-Flow

### 9.1 Einreichung (Spieler, via Discord)

- Command: **`/submit`** (siehe §11). Eingabe-Format:
  - **Erstes Token MUSS** ein **per RegEx validierter YouTube-Video-Link** sein.
  - **Alles nach dem ersten Token** = **Kommentar**, der dem Mod bei der Review
    angezeigt wird.
  - **Keine Datei-Uploads:** Es werden **ausschließlich YouTube-Links**
    akzeptiert. Discord-CDN-Links für Attachments laufen ab und es sollen keine
    Videodateien auf dem Server gehalten werden.
- **Validierung (MUSS):**
  - YouTube-RegEx akzeptiert die gängigen Formen:
    `youtu.be/<id>`, `youtube.com/watch?v=<id>`, `youtube.com/shorts/<id>`,
    `youtube.com/embed/<id>` (mit/ohne `www`, http/https, zusätzliche Query-Params).
    Empfohlenes Muster (in `core/domain/submissions.py` zentralisieren):
    ```
    ^(?:https?://)?(?:www\.)?
    (?:youtube\.com/(?:watch\?v=|shorts/|embed/)|youtu\.be/)
    (?P<id>[A-Za-z0-9_-]{11})(?:[?&#].*)?$
    ```
  - Eine Eingabe ohne gültigen YouTube-Link wird **abgelehnt** (klare
    Fehlermeldung, keine Submission angelegt).
- Es wird eine `submission` mit `status = pending` angelegt; gespeichert wird nur
  die `video_url`. Spieler erhält eine Bestätigung (ephemeral). Optional: Hinweis
  in einen Mod-Review-Channel posten.

### 9.2 Review (Mod, primär via Web-Panel)

- Mod sieht die **Pending-Queue** je Guild/Game/Challenge mit: Spieler, Video
  (YouTube-Embed), Kommentar, Einreichungszeit.
- Aktionen pro Submission:
  - **Confirm** → `status = confirmed`, `reviewed_at` setzen, **1 Punkt** in
    `points_ledger` (idempotent über `submission_id`).
  - **Deny** → `status = denied`, optionaler `mod_note`.
  - **Set Metric** → optionalen `metric_value` (Zeit/Score) setzen, je nach
    Challenge (`metric_type`). Auch nachträglich änderbar.
- Alle Aktionen werden in `mod_actions` protokolliert.
- **Discord-Fallback (KANN, Phase 2):** Confirm/Deny auch über Buttons unter dem
  Mod-Review-Post, mit identischer Backend-Logik.

### 9.3 Mehrfach-Einreichungen / Wertung (festgelegt)

- Ein Spieler DARF mehrfach einreichen (z.B. bessere Zeit).
- **Punkte:** **max. 1 Punkt pro Spieler pro Challenge** — Mehrfach-Einreichungen
  geben **keine** zusätzlichen Punkte (kein Spam-Anreiz).
- **Leaderboard-Anzeige:** In der **Per-Challenge-Rangliste** wird pro Spieler nur
  das **beste bestätigte Ergebnis** angezeigt (gemäß aktiver Sortierung/Metrik).
- **Weitere Einreichungen** des Spielers bleiben gespeichert und sind in dessen
  **privatem Profil** (§12.4) sichtbar, erscheinen aber nicht im öffentlichen
  Leaderboard.

---

## 10. Punkte, Ranglisten & Streaks

### 10.1 Kumulative Punkte & Seasons (festgelegt)

- **Max. 1 Punkt pro Spieler pro Challenge** (gemäß §9.3), kumulativ innerhalb
  einer Season.
- Gespeichert in `points_ledger` (append-only, idempotent). Deny/Revert eines
  zuvor bestätigten Runs entfernt den zugehörigen Punkt.
- **Seasons = jährlich:** Die kumulative Punktewertung läuft **pro Kalenderjahr**
  und wird zum **Jahreswechsel zurückgesetzt** (Season-Grenze konsistent mit dem
  Perioden-Anker, siehe §8 — Default **1. Januar 20:00 UTC** der jeweiligen
  Season; alternativ 1. Jan 00:00 UTC, beim Implementieren festlegen).
- Eine `season`-Dimension (z.B. Jahr `2026`) hängt an `points_ledger` →
  **aktuelle Season** + **archivierte Seasons** (Hall of Fame) sind abfragbar.
- Aggregation: pro Season je Spieler (× Guild × Game). Periodenbezogene
  Ranglisten (diese Woche / diesen Monat) per Zeitfilter ableitbar.

### 10.2 Per-Challenge-Rangliste

- Enthält die **bestätigten** Runs einer Challenge, **pro Spieler nur das beste
  Ergebnis** (gemäß aktiver Sortierung/Metrik). Weitere eigene Runs nur im
  privaten Profil (§9.3, §12.4).
- **Standard-Sortierung:** nach **Einreichungszeitpunkt aufsteigend**
  (= „zuerst geschafft").
- **Mod-Umsortierung (live, je Challenge/Periode):** Der Mod KANN die Liste
  umschalten nach:
  - **schnellste Zeit** (`metric_value` aufsteigend),
  - **längste Zeit** (`metric_value` absteigend),
  - **höchste Punktzahl** (`metric_value` absteigend),
  - **niedrigste Punktzahl** (`metric_value` aufsteigend).
- Die gewählte Sortierung ist eine Eigenschaft der Challenge (`default_sort`),
  die der Mod überschreiben/festlegen kann; die öffentliche Anzeige folgt ihr.

### 10.3 Streaks (nur Anzeige, kein Bonus)

- **Streak** = Anzahl aufeinanderfolgender Perioden (gleicher `period_type`) mit
  ≥1 bestätigtem Run des Spielers.
- Wird **angezeigt** (Profil/Leaderboard), gibt **keine** Extra-Punkte.
- Berechnung in `core/domain/scoring.py`.

### 10.4 Bonus-Condition

- Das Erfüllen der **Condition2 (Bonus)** ist **nur Auszeichnung/Filter** —
  z.B. ein „⭐ Bonus erfüllt"-Marker am Run, ggf. eigene Bonus-Rangliste/Filter.
- **Keine** zusätzlichen kumulativen Punkte.

---

## 11. Discord-Bot — Features & Commands

> Alle Commands als **Slash-Commands**, guild-scoped registriert. Antworten
> möglichst **ephemeral**, wo sinnvoll.

### 11.1 Spieler-Commands

| Command | Beschreibung |
|---|---|
| `/submit <youtube-link> [comment]` | Run einreichen. Erstes Token = gültiger YouTube-Link (Pflicht); Rest = Kommentar. Legt `submission (pending)` an. Keine Datei-Uploads. |
| `/challenge [game]` | Aktuelle Weekly + Monthly Challenge(s) anzeigen (rendered_text, Zeitfenster, Bonus). |
| `/leaderboard [period] [game]` | Rangliste: Season-Punkte (jährlich) oder Per-Challenge (Periode wählbar). |
| `/mystats [game]` | Eigene Punkte, Streak, eingereichte/bestätigte Runs. |
| `/help` | Kurzüberblick. |

### 11.2 Mod-/Admin-Commands (rollen-geschützt)

| Command | Beschreibung | Phase |
|---|---|---|
| `/review` | Link zum Web-Panel bzw. (später) Inline-Review starten. | 1 |
| `/challenge-preview [period]` | Vorschau der nächsten generierten Challenge. | 2 |
| `/setup` | Channels/Mod-Rollen/aktive Games für die Guild konfigurieren. | 1–2 |
| `/ban-player @user` / `/unban-player @user` | Spieler von Teilnahme ausschließen. | 3 |

### 11.3 Posting-Verhalten

- Bei Periodenstart postet der Bot die Challenge in den `announce_channel` mit:
  Titel, `rendered_text`, Bausteinen, Bonus, Zeitfenster (UTC + lokalisiert),
  Hinweis auf `/submit`.
- Bei Periodenende: finale (eingefrorene) Rangliste posten.

---

## 12. Web-Interface — Features & Auth

### 12.1 Authentifizierung & Zugriffsebenen

- **Discord OAuth2** Login (`identify`, `guilds`-Scopes; ggf. `guilds.members.read`
  bzw. Bot-seitiger Rollen-Lookup) — **für alle Nutzer**, nicht nur Mods.
- Nach Login wird der `users`-Eintrag verknüpft/angelegt und die Rolle bestimmt:
  - **Spieler (jeder eingeloggte Nutzer):** Zugriff auf das **eigene Profil**
    (§12.4).
  - **Mod (Mod-Rolle in der Ziel-Guild, IDs aus `guilds.config`):** zusätzlich
    Zugriff auf das **Mod-Panel** (§12.2).
- Öffentliche Leaderboards/Challenge-Übersichten (§12.3) sind **auch ohne Login**
  sichtbar; Login schaltet das persönliche Profil (und ggf. Mod-Funktionen) frei.
- Session via Server-Side Store oder signiertes Cookie (`SESSION_SECRET`).

### 12.2 Mod-Panel (geschützt)

- **Review-Queue:** pending Submissions je Guild/Game/Challenge; Video als
  YouTube-Embed abspielbar, Kommentar sichtbar.
- **Aktionen:** Confirm / Deny / Metric setzen / Sortierung der Per-Challenge-
  Rangliste festlegen.
- **Custom-Challenges planen:** Override (Freitext oder Bausteine) für künftige
  Perioden anlegen/bearbeiten.
- **Pool-Verwaltung (Phase 2):** Characters/Boards/Levels/Conditions je Game
  pflegen (CRUD, `is_active`, Constraints, Difficulty-Scaling).
- **Spieler-Verwaltung (Phase 3):** Ban/Unban, Punkte-Korrekturen (auditiert).
- **Audit-Log:** Einsicht in `mod_actions`.

### 12.3 Öffentlicher Bereich (ohne Login)

- Aktuelle Weekly/Monthly Challenges je Game.
- **Leaderboards:** Season-Punkte (jährlich) + Per-Challenge-Ranglisten (mit
  Streak-Anzeige); archivierte Seasons abrufbar (Hall of Fame).
- Challenge-Historie.
- Auf dem Leaderboard ist ein **„Login"**-Button präsent, mit dem sich Spieler per
  Discord-OAuth anmelden, um ihr persönliches Profil zu sehen.

### 12.4 Spieler-Profil (eingeloggt, jeder Nutzer)

Nach OAuth-Login sieht **jeder** Spieler sein eigenes Profil mit:

- **Übersicht:** kumulative Punkte, aktuelle Streak, Rang im Leaderboard.
- **Bisherige Challenges:** an denen teilgenommen wurde, inkl. Status der eigenen
  Einreichungen (`pending`/`confirmed`/`denied`) und ggf. `metric_value`/Platzierung.
- **Verpasste Challenges:** abgeschlossene Perioden **ohne** (bestätigte)
  Einreichung des Spielers.
- **Laufende & kommende Challenges:** aktuelle Challenge(s) mit eigenem
  Einreichungsstatus; bei kommenden/geplanten Perioden ein Ausblick (soweit
  bekannt/öffentlich) — Einreichungen sind erst ab Periodenstart möglich.
- **Eigene Einreichungs-Historie:** Liste aller Submissions (Link, Kommentar,
  Status, Mod-Notiz).
- Profil-Sicht ist **read-only**; das eigentliche Einreichen läuft über Discord
  (`/submit`). Optional (später) KANN das Web ein Einreichungsformular anbieten.

> **Sichtbarkeit:** Ein Nutzer sieht nur **sein eigenes** Profil im Detail.
> Fremde Profile zeigen höchstens öffentliche Aggregat-Daten (Punkte, Streak),
> wie sie ohnehin im Leaderboard erscheinen.

---

## 13. Multi-Game & Multi-Guild Design

Von Anfang an eingeplant (auch wenn es das Projekt vergrößert):

- **Multi-Game:** Alle Pools (`characters/boards/levels/conditions`) und
  Challenges hängen an `game_id`. Ein neues Spiel = neuer `games`-Eintrag +
  Seed-Daten, **kein** Schema-Umbau. Generator ist game-agnostisch (arbeitet nur
  mit den Pools des jeweiligen Games).
- **Multi-Guild:** Alles Guild-bezogene (Challenges, Submissions, Punkte, Config,
  Mod-Rollen, Channels) hängt an `guild_id`. Über `guild_games` legt jede Guild
  fest, **welche Spiele** sie fährt und **welche period_types** aktiv sind.
- **Isolation:** Ranglisten/Punkte sind standardmäßig **pro Guild** getrennt
  (× Game). (Cross-Guild-Aggregation optional, später.)
- **Skalierung:** Scheduler iteriert über alle aktiven `guild_game × period_type`.

> Konsequenz: Keine globalen Singletons/Hardcodings auf eine Guild oder „SBK1".
> Alles geht über die Konfig- und Pool-Tabellen.

---

## 14. Konfiguration, Secrets & Deployment

### 14.1 Environment-Variablen (`.env.example`)

```dotenv
# --- Discord ---
DISCORD_BOT_TOKEN=
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_PUBLIC_KEY=

# --- Datenbank ---
POSTGRES_USER=rc
POSTGRES_PASSWORD=
POSTGRES_DB=regularchallenge
DATABASE_URL=postgresql+asyncpg://rc:${POSTGRES_PASSWORD}@db:5432/regularchallenge

# --- Web ---
WEB_BASE_URL=https://challenges.example.com
SESSION_SECRET=
OAUTH_REDIRECT_URI=${WEB_BASE_URL}/auth/callback

# --- Betrieb ---
DEFAULT_TIMEZONE=UTC
LOG_LEVEL=INFO
```

> Secrets MÜSSEN über `.env` / Docker-Secrets injiziert werden, **niemals**
> committen. `.env` ist in `.gitignore`; nur `.env.example` wird versioniert.

### 14.2 Docker-Compose (Service-Skizze)

```yaml
services:
  db:
    image: postgres:16
    environment: [POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER"], ... }

  migrate:                      # läuft einmalig vor bot/web
    build: .
    command: ["alembic", "upgrade", "head"]
    depends_on: { db: { condition: service_healthy } }

  bot:
    build: .
    command: ["python", "-m", "bot.main"]
    env_file: [.env]
    depends_on: { db: { condition: service_healthy }, migrate: { condition: service_completed_successfully } }

  web:
    build: .
    command: ["uvicorn", "web.main:app", "--host", "0.0.0.0", "--port", "8000"]
    env_file: [.env]
    depends_on: { db: { condition: service_healthy }, migrate: { condition: service_completed_successfully } }

  caddy:
    image: caddy:2
    depends_on: [web]
    ports: ["80:80", "443:443"]
    volumes:
      - "./deploy/Caddyfile:/etc/caddy/Caddyfile"
      - "caddy_data:/data"
      - "caddy_config:/config"

volumes:
  pgdata: {}
  caddy_data: {}
  caddy_config: {}
```

### 14.3 Caddyfile (Skizze)

```
challenges.example.com {
    reverse_proxy web:8000
}
```

### 14.4 Betriebliche Hinweise

- **Backups:** regelmäßiger `pg_dump` des `pgdata`-Volumes (keine Medien zu
  sichern, da nur YouTube-Links gespeichert werden).
- **Migrations:** ausschließlich über Alembic; nie manuell am Schema.
- **Idempotenz:** Scheduler & Punktevergabe MÜSSEN gegen Doppelausführung
  abgesichert sein (Unique-Constraints, advisory locks).

---

## 15. Roadmap / Phasen

### Phase 0 — Fundament
- Repo-Struktur, `core`-Paket, Docker-Compose, Postgres, Alembic-Setup.
- Basis-Modelle (§6) + erste Migration. CI/Lint/Tests-Gerüst.

### Phase 1 — MVP
- SBK-1-Seed (Characters/Boards/Levels/Conditions; **zu verifizieren**, Anhang A).
- Prozedurale Generierung für **Weekly + Monthly** (UTC, §8).
- Discord: `/submit`, `/challenge`, `/leaderboard`, `/mystats`; Auto-Posting.
- Submission-Parsing (**nur YouTube-RegEx**, keine Datei-Speicherung).
- Web: Discord-OAuth (für **alle** Nutzer), **Spieler-Profil** (eigene
  Challenges/Einreichungen/verpasste), **Mod-Review (confirm/deny + Metric
  setzen)**, öffentliches Punkte-Leaderboard + Per-Challenge-Rangliste.
- Punkte-Ledger (1 Punkt/Spieler/Challenge) inkl. **jährlicher Season-Dimension**;
  Standard-Sortierung nach Einreichungszeit; im Leaderboard nur bestes Ergebnis.

### Phase 2 — Komfort & Pflege
- Mod-Umsortierung der Ranglisten (schnellste/längste/höchste/niedrigste).
- Streak-Anzeige.
- Admin-Custom-Challenges (Freitext-Overrides) planen.
- Pool-Verwaltung im Web (CRUD für Components).
- Optional: Confirm/Deny per Discord-Buttons.

### Phase 3 — Erweiterungen
- **Daily** Challenges (bei steigender Aktivität).
- Spieler-Ban/Ausschluss + weitere Mod-Aktionen.
- **Season-Archiv / Hall of Fame** (abgeschlossene Jahres-Seasons abrufbar).
- Weitere **Games** (Seed + aktivieren) und vollständige **Multi-Guild**-Konfig-UI.

### Phase 4 — Optional
- Statistiken/Analytics, ggf. optionaler LLM-Challenge-Generator.

---

## 16. Offene Punkte & Annahmen

> **Geklärt** (vom Auftraggeber bestätigt):
> - **Punkte-Policy:** max. **1 Punkt pro Spieler pro Challenge**; im Leaderboard
>   nur das **beste** Ergebnis, weitere Runs nur im privaten Profil. (§9.3/§10)
> - **Seasons:** **jährlich** (Reset zum Jahreswechsel). (§10.1)
> - **Perioden-/Season-Anker:** **UTC**, Anker-Uhrzeit **20:00 UTC** (fairer
>   USA/DE/AU-Kompromiss); Grenze = Posting. (§8)
> - **Nachweis:** ausschließlich **YouTube-Links**, keine Datei-Speicherung. (§9)
> - **OAuth:** für **alle** Nutzer; Spieler-Profil + Mod-Panel. (§12)
>
> **Noch offen / beim Implementieren festzulegen:**

1. **SBK-1-Datenpool:** Charaktere/Boards/Levels/Conditions in Anhang A sind ein
   **vorläufiger Seed** und MÜSSEN gegen das echte Spiel verifiziert werden.
   Pools sind ohnehin admin-pflegbar. (Hinweis: einige in den ursprünglichen
   Beispielen genannten Begriffe wie „Alpine 2", „Star", „Sunset Rock",
   „Ninja Land" stammen evtl. aus **SBK 2**, nicht SBK 1 — bitte beim Verifizieren
   beachten.) — **verifizieren** (Quellen siehe unten)
2. **Season-Grenze exakt:** 1. Jan **20:00 UTC** (Anker-konform) vs. 00:00 UTC —
   beim Implementieren final festlegen.
3. **Discord-Bibliothek:** `discord.py` angenommen (Alternativen `py-cord`/
   `nextcord`).
4. **ID-Strategie & Frontend-CSS** (Pico vs. Tailwind): Implementierungsdetail,
   beim Scaffolding festlegen.

### Quellen zur SBK-1-Datenverifikation

- **Snowboard Kids Wiki (Fandom):** <https://snowboardkids.fandom.com/> —
  Charaktere, Boards, Strecken pro Spiel.
- **StrategyWiki – Snowboard Kids:** <https://strategywiki.org/wiki/Snowboard_Kids>
- **GameFAQs (N64) – Snowboard Kids:** Guides/FAQs mit Shop-Boards & Strecken,
  <https://gamefaqs.gamespot.com/n64/198848-snowboard-kids>
- **MobyGames:** <https://www.mobygames.com/game/snowboard-kids/> (Release-Infos)
- **Original-Handbuch (N64)** als verlässlichste Quelle für exakte Boards/Namen.

---

## Anhang A: SBK-1-Datenpool (Seed, zu verifizieren)

> ⚠️ **Vorläufig.** Vor dem Seed gegen *Snowboard Kids 1 (N64, 1997)* prüfen.
> Endgültige Pflege erfolgt über das Web-Panel. Begriffe ohne Gewähr.

**Characters (SBK 1, zu prüfen):**
`Slash Kamei`, `Nancy Neil`, `Jam Kuehnemund`, `Linda Maltini`,
`Tommy Erikkson`, `Wendy Lane`, (unlockable) `Shinobin`, `Pumpkin/Snowman (?)`.

**Boards (SBK 1, zu prüfen):**
Im Shop kaufbare Boards — exakte Namen/Anzahl beim Verifizieren festlegen.

**Levels (SBK 1, zu prüfen):**
`Sunny Mountain`, `Big Snowman`, `Quicksand Valley`, `Night Highway`,
`Silver Mountain`, `Grass Valley`, `Dizzy Land`, … (vollständige Liste prüfen;
einige o.g. Beispielnamen gehören evtl. zu SBK 2).

**Conditions (Pool, McGyna-Ideen → als Daten modellieren):**

| key | template_text | type | bonus_eligible | requires (Beispiel) |
|---|---|---|---|---|
| `no_tricks` | „Keine Tricks" | pass_fail | ja | – |
| `no_blue_items` | „Keine blauen Items benutzen" | pass_fail | ja | – |
| `no_red_items` | „Keine roten Items benutzen" | pass_fail | ja | – |
| `no_items` | „Keine Items benutzen" | pass_fail | ja | – |
| `no_wipeouts` | „Keine Wipeouts/Stürze" | pass_fail | ja | – |
| `no_bonks` | „Keine Bonks" | pass_fail | ja | – |
| `no_jumping` | „Nicht springen" | pass_fail | ja | `excludes: [special_tricks]` |
| `hit_all_opponents` | „Triff alle 3 Gegner" | pass_fail | nein | – |
| `hit_cpu_with_ice` | „Triff {n} CPU mit Eis" | numeric | ja | – |
| `hit_every_cpu_n` | „Triff jeden CPU {n}×" | numeric | nein | – |
| `special_tricks` | „{n} unterschiedliche Spezial-Tricks" | numeric | nein | `excludes: [no_jumping, no_tricks]` |
| `spin_all_directions` | „Spin-Trick in jede Richtung" | pass_fail | ja | – |
| `fall_off_map_n` | „Falle {n}× von der Map" | numeric | ja | `level_attribute: has_fall_off_zone` |
| `button_restriction` | „Verzicht auf Eingabe: {button}" | pass_fail | ja | – |
| `win_from_4th_lap3` | „Gewinne, obwohl zu Beginn von Lap 3 auf Platz 4" | pass_fail | nein | `lap_based` |
| `zoolander` | „Keine Linkskurven (Zoolander)" | pass_fail | ja | – |

> **Difficulty-Scaling-Beispiele** (`difficulty_scaling`):
> - `hit_cpu_with_ice`: easy=1, medium=2, hard=3
> - `hit_every_cpu_n`: easy=1, medium=2, hard=3
> - `special_tricks`: easy=2, medium=3, hard=4
> - `fall_off_map_n`: easy=1, medium=2, hard=3

---

## Anhang B: Beispiel-Challenges

**Weekly (generated, medium, metric_type = time):**
> 🏂 **Weekly Challenge** — *Snowboard Kids*
> **Slash** · Board **Big Air** · **Sunny Mountain**
> **Auflage:** Keine Tricks
> **⭐ Bonus:** Keine blauen Items
> *Wertung: schnellste Zeit · Mo 20:00 UTC – Mo 20:00 UTC (1 Woche)*

**Monthly (generated, hard, metric_type = score):**
> 🏂 **Monthly Challenge** — *Snowboard Kids*
> **Nancy** · Board **Star** · **Ninja Land**
> **Auflage:** Triff jeden CPU 3×
> **⭐ Bonus:** Triff 1 CPU mit Eis
> *Wertung: höchste Punktzahl · 1. 20:00 UTC – 1. Folgemonat 20:00 UTC*

**Custom (admin freetext):**
> 🏂 **Weekly Challenge (Special)** — *Snowboard Kids*
> „Gewinne ein komplettes Rennen rückwärts gefahren — Video erforderlich."
> *Wertung: zuerst geschafft · Mo 20:00 UTC – Mo 20:00 UTC (1 Woche)*

---

*Ende der Spezifikation. Änderungen bitte als PR gegen dieses Dokument, damit der
„Soll-Zustand" für alle Agenten konsistent bleibt.*
