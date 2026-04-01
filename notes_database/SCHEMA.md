# Notes Database Schema (PostgreSQL)

This schema was applied by connecting using the command in `db_connection.txt` and executing each statement one-by-one via `psql -c`.

## Connection

- `db_connection.txt` contains the canonical CLI command, for example:
  - `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions

- `citext` (case-insensitive text; used for `users.email` and `tags.name`)
- `pg_trgm` (trigram index support for fast `ILIKE` search)

## Tables

### `users`
Stores authentication and profile info.

Columns:
- `id` UUID PK
- `email` CITEXT UNIQUE NOT NULL
- `password_hash` TEXT NOT NULL
- `display_name` TEXT NULL
- `is_active` BOOLEAN NOT NULL DEFAULT true
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL
- `last_login_at` TIMESTAMPTZ NULL

Indexes:
- unique: `(email)`
- `(created_at desc)`

### `notes`
Stores note content.

Columns:
- `id` UUID PK
- `user_id` UUID NOT NULL → `users(id)` ON DELETE CASCADE
- `title` TEXT NOT NULL DEFAULT ''
- `content` TEXT NOT NULL DEFAULT ''
- `is_pinned` BOOLEAN NOT NULL DEFAULT false
- `is_archived` BOOLEAN NOT NULL DEFAULT false
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL
- `pinned_at` TIMESTAMPTZ NULL

Indexes:
- `(user_id, updated_at desc)` for listing
- `(user_id, is_pinned, pinned_at desc)` for pinned filter
- trigram GIN:
  - `title gin_trgm_ops`
  - `content gin_trgm_ops`

### `tags`
User-scoped tags.

Columns:
- `id` UUID PK
- `user_id` UUID NOT NULL → `users(id)` ON DELETE CASCADE
- `name` CITEXT NOT NULL
- `color` TEXT NULL
- `created_at` TIMESTAMPTZ NOT NULL

Indexes:
- unique: `(user_id, name)`
- `(user_id, created_at desc)`

### `note_tags`
Many-to-many between notes and tags.

Columns:
- `note_id` UUID NOT NULL → `notes(id)` ON DELETE CASCADE
- `tag_id` UUID NOT NULL → `tags(id)` ON DELETE CASCADE
- `created_at` TIMESTAMPTZ NOT NULL
- primary key: `(note_id, tag_id)`
- index: `(tag_id)` for reverse lookup (all notes for a tag)

### `settings`
Per-user settings.

Columns:
- `user_id` UUID PK → `users(id)` ON DELETE CASCADE
- `theme` TEXT NOT NULL DEFAULT 'light' CHECK in ('light','dark')
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL

Indexes:
- `(updated_at desc)`

## Triggers

A shared trigger function keeps `updated_at` current:

- `set_updated_at()` and triggers:
  - `trg_users_updated_at` on `users`
  - `trg_notes_updated_at` on `notes`
  - `trg_settings_updated_at` on `settings`

## View

- `v_notes_with_tag_names`: convenience view returning one row per note plus `tags` as a text array.

Intended usage: simplify list endpoints (notes with tag names) without requiring multiple queries client-side.
