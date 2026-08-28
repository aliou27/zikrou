# Database

## Overview

Zikrou uses **PostgreSQL 17** managed via **Flyway** migrations. The schema consists of 20+ tables across several functional domains.

---

## Entity Domains

### Reference Data

```
country     — countries (Senegal, Mauritania, Nigeria, ...)
genre       — audio genres (Qasida, Xassida, ...)
tag         — content tags
```

### Content

```
zakir       — Islamic scholars and reciters
daira       — community organizations
album       — groupings of Zikrs
zikr        — individual audio content
zikr_zakir  — many-to-many: Zikr ↔ Zakir
zikr_tag    — many-to-many: Zikr ↔ Tag
audio_file  — audio files per Zikr (quality variants)
zikr_stats  — aggregated stats (plays, likes, downloads)
```

### Users

```
user             — accounts
user_settings    — preferences (dark mode, language, quality)
refresh_token    — active refresh tokens
email_verification_token — email confirmation
```

### User Activity

```
favorite           — user ↔ zikr favorites (composite PK)
listening_history  — playback progress per user per zikr
zakir_follow       — user ↔ zakir follows (composite PK)
daira_follow       — user ↔ daira follows (composite PK)
download           — offline downloads
search_history     — user search queries
listening_session  — session tracking with source (HOME, SEARCH, PLAYLIST...)
```

### Playlists

```
playlist       — user-created playlists
playlist_zikr  — playlist ↔ zikr with position (composite PK)
```

### Community

```
comment    — comments on Zikrs (with parent for replies)
rating     — 1–5 star ratings (composite PK: user + zikr)
report     — content reports
suggestion — user content suggestions
```

### Monetization

```
subscription  — user subscription plans
payment       — payment records
```

### Platform

```
notification      — in-app notifications
device            — registered devices
push_token        — FCM tokens for push notifications
audit_log         — admin action tracking
app_config        — dynamic configuration (key/value)
```

---

## Key Design Patterns

### Composite Primary Keys

Several tables use composite primary keys instead of a surrogate ID. This enforces uniqueness at the database level:

| Table | Composite Key |
|---|---|
| `favorite` | (user_id, zikr_id) |
| `rating` | (user_id, zikr_id) |
| `zakir_follow` | (user_id, zakir_id) |
| `daira_follow` | (user_id, daira_id) |
| `playlist_zikr` | (playlist_id, zikr_id) |

### Soft Deletes

Comments use a `status` field (`VISIBLE`, `DELETED`) instead of physical deletion, preserving thread structure.

### Full-Text Search

The `zikr` table has a `search_vector tsvector` column maintained by a database trigger. A GIN index on this column enables fast full-text search.

```sql
-- Index
CREATE INDEX idx_zikr_search_vector ON zikr USING GIN(search_vector);

-- Query example
SELECT * FROM zikr
WHERE search_vector @@ plainto_tsquery('french', 'baye niass')
ORDER BY ts_rank(search_vector, plainto_tsquery('french', 'baye niass')) DESC;
```

### Migrations

All schema changes are managed by **Flyway** with versioned SQL scripts:

```
V1__init_tables.sql
V2__init_constraints.sql
V3__init_indexes.sql
V4__init_unique_constraints.sql
...
V22__min_app_build_config.sql
```

This ensures reproducible schema state across environments.

---

## Simplified Entity Relationships

```
Country ──────────── Zakir
  │                    │
  │               Album (per Zakir)
  │                    │
  ├── Daira        Zikr ←──────── Genre
  │     │           │
  │     └───────────┤ (via zikr_zakir)
  │                 │
User               AudioFile
  │                ZikrStats
  ├── Favorite → Zikr
  ├── ListeningHistory → Zikr
  ├── ZakirFollow → Zakir
  ├── DairaFollow → Daira
  ├── Playlist
  │     └── PlaylistZikr → Zikr
  ├── Comment → Zikr
  ├── Rating → Zikr
  ├── Download → Zikr
  ├── Subscription
  └── Payment → Subscription
```
