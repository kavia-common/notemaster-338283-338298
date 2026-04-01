# Notes Database Schema (PostgreSQL)

This schema is applied by connecting using the command in `db_connection.txt` and executing each statement **one-by-one** via:

- `psql ... -c "SQL_STATEMENT"`

## Connection

- `db_connection.txt` contains the canonical CLI command, for example:
  - `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions

Enabled (idempotently):

- `citext` (case-insensitive text; used for `users.email` and `tags.name`)
- `pg_trgm` (trigram index support for fast `ILIKE` search)
- `pgcrypto` (used for `gen_random_uuid()` UUID defaults)

## Shared Trigger Function

A shared trigger function keeps `updated_at` current:

- `set_updated_at()`:
  - `BEFORE UPDATE` sets `NEW.updated_at = now()`

## Tables

### `users`

Stores authentication and profile info.

Columns:
- `id` UUID PK (default `gen_random_uuid()`)
- `email` CITEXT UNIQUE NOT NULL
- `password_hash` TEXT NOT NULL
- `display_name` TEXT NULL
- `is_active` BOOLEAN NOT NULL DEFAULT true
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `last_login_at` TIMESTAMPTZ NULL

Constraints / Keys (as currently present in DB):
- primary key: `users_pkey` on `(id)`
- unique index: `users_email_uq` on `(email)` (present as a UNIQUE INDEX)
  - Note: in the running DB this uniqueness is enforced via an index named `users_email_uq` (not a `UNIQUE CONSTRAINT` entry).

Indexes (as currently present in DB):
- `idx_users_created_at_desc` on `(created_at desc)`
- `users_created_at_idx` on `(created_at desc)` (duplicate/legacy index currently present)

Triggers:
- `trg_users_updated_at` (`BEFORE UPDATE` → `set_updated_at()`)

### `notes`

Stores note content.

Columns:
- `id` UUID PK (default `gen_random_uuid()`)
- `user_id` UUID NOT NULL → `users(id)` ON DELETE CASCADE
- `title` TEXT NOT NULL DEFAULT ''
- `content` TEXT NOT NULL DEFAULT ''
- `is_pinned` BOOLEAN NOT NULL DEFAULT false
- `is_archived` BOOLEAN NOT NULL DEFAULT false
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `pinned_at` TIMESTAMPTZ NULL

Constraints / Keys (as currently present in DB):
- primary key: `notes_pkey` on `(id)`
- foreign key: `notes_user_id_fkey` on `(user_id)` → `users(id)` ON DELETE CASCADE

Indexes (as currently present in DB):
- `idx_notes_user_updated_at_desc` on `(user_id, updated_at desc)`
- `notes_user_updated_idx` on `(user_id, updated_at desc)` (duplicate/legacy index currently present)
- `idx_notes_user_pinned_pinned_at_desc` on `(user_id, is_pinned, pinned_at desc)`
- `notes_user_pinned_idx` on `(user_id, is_pinned, pinned_at desc)` (duplicate/legacy index currently present)
- trigram GIN (search):
  - `idx_notes_title_trgm` on `title gin_trgm_ops`
  - `notes_title_trgm_idx` on `title gin_trgm_ops` (duplicate/legacy index currently present)
  - `idx_notes_content_trgm` on `content gin_trgm_ops`
  - `notes_content_trgm_idx` on `content gin_trgm_ops` (duplicate/legacy index currently present)

Triggers:
- `trg_notes_updated_at` (`BEFORE UPDATE` → `set_updated_at()`)

### `tags`

User-scoped tags.

Columns:
- `id` UUID PK (default `gen_random_uuid()`)
- `user_id` UUID NOT NULL → `users(id)` ON DELETE CASCADE
- `name` CITEXT NOT NULL
- `color` TEXT NULL
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()

Constraints / Keys (as currently present in DB):
- primary key: `tags_pkey` on `(id)`
- foreign key: `tags_user_id_fkey` on `(user_id)` → `users(id)` ON DELETE CASCADE
- unique index: `tags_user_name_uq` on `(user_id, name)` (present as a UNIQUE INDEX)
  - Note: in the running DB this uniqueness is enforced via an index named `tags_user_name_uq` (not a `UNIQUE CONSTRAINT` entry).

Indexes (as currently present in DB):
- `idx_tags_user_created_at_desc` on `(user_id, created_at desc)`
- `tags_user_created_idx` on `(user_id, created_at desc)` (duplicate/legacy index currently present)
- `idx_tags_name_trgm` trigram GIN on `name gin_trgm_ops` (autocomplete/search)

### `note_tags`

Many-to-many between notes and tags.

Columns:
- `note_id` UUID NOT NULL → `notes(id)` ON DELETE CASCADE
- `tag_id` UUID NOT NULL → `tags(id)` ON DELETE CASCADE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()

Constraints / Keys (as currently present in DB):
- primary key: `note_tags_pkey` on `(note_id, tag_id)`
- foreign key: `note_tags_note_id_fkey` on `(note_id)` → `notes(id)` ON DELETE CASCADE
- foreign key: `note_tags_tag_id_fkey` on `(tag_id)` → `tags(id)` ON DELETE CASCADE

Indexes (as currently present in DB):
- `idx_note_tags_tag_id` on `(tag_id)`
- `note_tags_tag_id_idx` on `(tag_id)` (duplicate/legacy index currently present)

### `settings`

Per-user settings.

Columns:
- `user_id` UUID PK → `users(id)` ON DELETE CASCADE
- `theme` TEXT NOT NULL DEFAULT 'light'
- `created_at`, `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()

Constraints / Keys (as currently present in DB):
- primary key: `settings_pkey` on `(user_id)`
- foreign key: `settings_user_id_fkey` on `(user_id)` → `users(id)` ON DELETE CASCADE
- check constraint: `settings_theme_chk` CHECK `theme in ('light','dark')`

Indexes (as currently present in DB):
- `idx_settings_updated_at_desc` on `(updated_at desc)`
- `settings_updated_at_idx` on `(updated_at desc)` (duplicate/legacy index currently present)

Triggers:
- `trg_settings_updated_at` (`BEFORE UPDATE` → `set_updated_at()`)

## View

- `v_notes_with_tag_names`: convenience view returning one row per note plus `tags` as a text array.

Definition summary:
- `notes` LEFT JOIN `note_tags` LEFT JOIN `tags`
- `tags` = `array_agg(tags.name order by tags.name)` with a `FILTER` to exclude nulls
- empty array when no tags: `ARRAY[]::text[]`

Intended usage:
- Simplify list endpoints (notes with tag names) without requiring multiple client-side queries.
