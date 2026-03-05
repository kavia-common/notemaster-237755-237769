# notes_database schema (NoteMaster)

This container uses a local PostgreSQL instance started by `startup.sh`.

**Connection string (authoritative):** see `db_connection.txt` (example: `psql postgresql://appuser:...@localhost:5000/myapp`)

## Tables

### `notes`
- `id` BIGSERIAL PRIMARY KEY
- `title` TEXT NOT NULL DEFAULT ''
- `content` TEXT NOT NULL DEFAULT ''
- `is_archived` BOOLEAN NOT NULL DEFAULT FALSE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()

A trigger updates `updated_at` on every update.

### `tags`
- `id` BIGSERIAL PRIMARY KEY
- `name` TEXT NOT NULL UNIQUE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()

### `note_tags` (join table)
- `note_id` BIGINT NOT NULL REFERENCES `notes(id)` ON DELETE CASCADE
- `tag_id` BIGINT NOT NULL REFERENCES `tags(id)` ON DELETE CASCADE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- PRIMARY KEY (`note_id`, `tag_id`)

## Indexes

### Sorting / listing
- `idx_notes_updated_at` on `notes(updated_at DESC)`
- `idx_notes_created_at` on `notes(created_at DESC)`
- `idx_tags_name` on `tags(name)`

### Tag filtering
- `idx_note_tags_tag_id` on `note_tags(tag_id)`
- `idx_note_tags_note_id` on `note_tags(note_id)`

### Search
To support fast substring search (`ILIKE '%term%'`) on title/content, trigram indexes are used:
- `pg_trgm` extension enabled
- `idx_notes_title_trgm` on `notes` using GIN trigram over `lower(title)`
- `idx_notes_content_trgm` on `notes` using GIN trigram over `lower(content)`

Note: an `unaccent(...)`-based expression index was attempted but not used because `unaccent` is not `IMMUTABLE` in PostgreSQL, which prevents it from being used directly in an index expression.

## Minimal seed data

The container seeds:
- Tags: `welcome`, `todo`, `ideas`
- A single note titled `Welcome to NoteMaster` tagged with `welcome`

All inserts are idempotent via `ON CONFLICT DO NOTHING`.

## How it was applied

Schema and seed were applied using the existing connection pattern:
- `psql postgresql://... -c "SQL_STATEMENT"`

This aligns with the repository's DB operational approach (see `startup.sh`, `backup_db.sh`, `restore_db.sh`).
