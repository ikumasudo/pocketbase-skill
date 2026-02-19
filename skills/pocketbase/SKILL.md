---
name: pocketbase
description: >-
  Skill for operating PocketBase backend REST API. Provides collection CRUD,
  record CRUD, superuser/user authentication, backup & restore,
  and migration file generation. Use for requests related to PocketBase,
  pb_migrations, collection management, and record operations.
license: MIT
metadata:
  version: "1.0.0"
allowed-tools: Read Write Edit Bash Grep Glob
---

# PocketBase Skill

Skill for operating a PocketBase v0.23+ backend via REST API. Uses Python scripts (standard library only) to perform authentication, collection CRUD, record CRUD, backup, and migration file generation.

## 1. Prerequisites and Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PB_URL` | No | `http://127.0.0.1:8090` | PocketBase base URL |
| `PB_SUPERUSER_EMAIL` | Yes* | — | Superuser email address |
| `PB_SUPERUSER_PASSWORD` | Yes* | — | Superuser password |

*Required when performing superuser operations.

If environment variables are not set, the `.env` file in the project root will be loaded.

```env
PB_URL=http://127.0.0.1:8090
PB_SUPERUSER_EMAIL=admin@example.com
PB_SUPERUSER_PASSWORD=your-password
```

**Important:** If environment variables are not set, confirm the values with the user before executing operations. When writing credentials to a `.env` file, ensure that `.env` is included in `.gitignore`.

### Connection Check

```bash
python ~/.claude/skills/pocketbase/scripts/pb_health.py
```

Runs a health check and (if credentials are available) a superuser authentication test.

## 2. Authentication

### Superuser Authentication

```bash
python ~/.claude/skills/pocketbase/scripts/pb_auth.py
```

Authenticates using `PB_SUPERUSER_EMAIL` and `PB_SUPERUSER_PASSWORD` via `POST /api/collections/_superusers/auth-with-password`. Returns a token.

### User Authentication

```bash
python ~/.claude/skills/pocketbase/scripts/pb_auth.py --collection users --identity user@example.com --password secret
```

Authenticates against any auth collection.

### Token Usage

Each script internally auto-acquires and caches the superuser token. On a 401 error, it retries authentication once.

**Details:** `Read references/auth-api.md` — OAuth2, impersonate, password reset, etc.

## 3. Collection Management

### List

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py list
python ~/.claude/skills/pocketbase/scripts/pb_collections.py list --filter "name~'user'" --sort "-created"
```

### Get

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py get posts
python ~/.claude/skills/pocketbase/scripts/pb_collections.py get pbc_1234567890
```

### Create

```bash
# Inline JSON
python ~/.claude/skills/pocketbase/scripts/pb_collections.py create '{"name":"posts","type":"base","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"}]}'

# From file
python ~/.claude/skills/pocketbase/scripts/pb_collections.py create --file schema.json
```

Collection types:
- `base` — Standard data collection
- `auth` — Auth collection (automatically adds email, password, username, etc.)
- `view` — Read-only SQL view (requires `viewQuery`)

### Update

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py update posts '{"listRule":"@request.auth.id != '\'''\''","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"},{"name":"status","type":"select","values":["draft","published"]}]}'
```

### Delete

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py delete posts
```

### Import

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py import --file collections.json
```

`collections.json` is a collections array, or `{"collections": [...], "deleteMissing": false}` format.

**Details:** `Read references/collections-api.md` — API rule syntax, all parameters.

## 4. Record Management

### List

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py list posts
python ~/.claude/skills/pocketbase/scripts/pb_records.py list posts --filter 'status="published"' --sort "-created" --expand "author" --page 1 --perPage 50
```

### Get

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789
python ~/.claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789 --expand "author,comments"
```

### Create

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py create posts '{"title":"Hello World","content":"<p>My first post</p>","status":"draft"}'
python ~/.claude/skills/pocketbase/scripts/pb_records.py create posts --file record.json
```

### Update

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py update posts abc123def456789 '{"status":"published"}'
```

### Delete

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py delete posts abc123def456789
```

### Filter Syntax Quick Reference

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equal | `status = "published"` |
| `!=` | Not equal | `status != "draft"` |
| `>`, `>=`, `<`, `<=` | Comparison | `count > 10` |
| `~` | Contains (LIKE) | `title ~ "hello"` |
| `!~` | Does not contain | `title !~ "test"` |
| `?=`, `?~` etc. | Array/multi-value field | `tags ?= "news"` |

Grouping: `(expr1 && expr2) || expr3`

### Sort

`-created` (DESC), `+title` (ASC), `@random`. Comma-separated for multiple fields.

### Expand (Relation Expansion)

`--expand "author"` — Direct relation.
`--expand "author.profile"` — Nested relation (up to 6 levels).
`--expand "author,category"` — Multiple relations.

**Details:** `Read references/records-api.md` — Batch operations, field selection, all operators.

## 5. Backup & Restore

```bash
# List
python ~/.claude/skills/pocketbase/scripts/pb_backups.py list

# Create (omit name for auto-generated timestamp name)
python ~/.claude/skills/pocketbase/scripts/pb_backups.py create
python ~/.claude/skills/pocketbase/scripts/pb_backups.py create my_backup.zip

# Restore (caution: replaces all data; server restart involved)
python ~/.claude/skills/pocketbase/scripts/pb_backups.py restore pb_backup_20240101120000.zip

# Delete
python ~/.claude/skills/pocketbase/scripts/pb_backups.py delete pb_backup_20240101120000.zip
```

**Notes:**
- Restore replaces all data (no merge)
- Server becomes temporarily unavailable during restore
- Always create a backup of current data before restoring

**Details:** `Read references/backups-api.md`

## 6. Migrations

### File Generation

```bash
python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "create_posts_collection"
python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "add_status_field" --dir ./pb_migrations
```

Generates a file in `{timestamp}_{description}.js` format. Write migration logic in the `// === UP ===` and `// === DOWN ===` sections of the template.

### Common Patterns

| Pattern | UP | DOWN |
|---------|-----|------|
| Create collection | `new Collection({...})` + `app.save()` | `app.findCollectionByNameOrId()` + `app.delete()` |
| Add field | `collection.fields.add(new Field({...}))` | `collection.fields.removeByName()` |
| Remove field | `collection.fields.removeByName()` | `collection.fields.add(new Field({...}))` |
| Change rules | `collection.listRule = "..."` | Revert to original rule |
| Execute SQL | `app.db().newQuery("...").execute()` | Reverse SQL |
| Seed data | `new Record(collection)` + `app.save()` | Delete records |

**Details:** `Read references/migrations.md` — Code examples for all patterns.
**Field types:** `Read references/field-types.md` — All field types and configuration options.

## 7. Error Handling

All scripts output structured JSON:

```json
{
  "success": true,
  "status": 200,
  "data": { ... }
}
```

### Common Error Codes

| HTTP Status | Meaning | Resolution |
|-------------|---------|------------|
| 400 | Bad Request | Check request body. Validation error details in `data` field |
| 401 | Unauthorized | Token expired. Scripts auto-retry |
| 403 | Forbidden | Operation denied by API rules. Check rules |
| 404 | Not Found | Collection or record does not exist |

Validation error example:
```json
{
  "success": false,
  "status": 400,
  "data": {
    "status": 400,
    "message": "Failed to create record.",
    "data": {
      "title": {
        "code": "validation_required",
        "message": "Missing required value."
      }
    }
  }
}
```

## 8. Quick Reference

| Task | Script | Detail Reference |
|------|--------|-----------------|
| Connection check | `python ~/.claude/skills/pocketbase/scripts/pb_health.py` | — |
| Superuser auth | `python ~/.claude/skills/pocketbase/scripts/pb_auth.py` | `references/auth-api.md` |
| User auth | `python ~/.claude/skills/pocketbase/scripts/pb_auth.py --collection <name> --identity <email> --password <pw>` | `references/auth-api.md` |
| List collections | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py list` | `references/collections-api.md` |
| Get collection | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py get <name>` | `references/collections-api.md` |
| Create collection | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py create '<json>'` | `references/collections-api.md`, `references/field-types.md` |
| Update collection | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py update <name> '<json>'` | `references/collections-api.md` |
| Delete collection | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py delete <name>` | `references/collections-api.md` |
| Import collections | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py import --file <file>` | `references/collections-api.md` |
| List records | `python ~/.claude/skills/pocketbase/scripts/pb_records.py list <collection>` | `references/records-api.md` |
| Get record | `python ~/.claude/skills/pocketbase/scripts/pb_records.py get <collection> <id>` | `references/records-api.md` |
| Create record | `python ~/.claude/skills/pocketbase/scripts/pb_records.py create <collection> '<json>'` | `references/records-api.md` |
| Update record | `python ~/.claude/skills/pocketbase/scripts/pb_records.py update <collection> <id> '<json>'` | `references/records-api.md` |
| Delete record | `python ~/.claude/skills/pocketbase/scripts/pb_records.py delete <collection> <id>` | `references/records-api.md` |
| List backups | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py list` | `references/backups-api.md` |
| Create backup | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py create [name]` | `references/backups-api.md` |
| Restore backup | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py restore <key>` | `references/backups-api.md` |
| Delete backup | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py delete <key>` | `references/backups-api.md` |
| Generate migration | `python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "<description>"` | `references/migrations.md` |
