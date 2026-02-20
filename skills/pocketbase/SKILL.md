---
name: pocketbase
description: >-
  Skill for operating PocketBase backend REST API. Provides collection CRUD,
  record CRUD, superuser/user authentication, backup & restore,
  migration file generation, and design guidance for API rules, relations,
  and security patterns. Use for requests related to PocketBase,
  pb_migrations, collection management, record operations, and backend design.
license: MIT
metadata:
  version: "1.0.0"
allowed-tools: Read Write Edit Bash Grep Glob
---

# PocketBase Skill

Skill for operating a PocketBase v0.23+ backend via REST API. Uses Python scripts (standard library only) to perform authentication, collection CRUD, record CRUD, backup, migration file generation, and includes design guidance for API rules, relations, and security patterns.

## Skill Resources

All resources for this skill are bundled in the skill directory at `.claude/skills/pocketbase/`:

- **Scripts**: `.claude/skills/pocketbase/scripts/` — Python scripts for API operations
- **References**: `.claude/skills/pocketbase/references/` — Detailed docs loaded on demand
- **Assets**: `.claude/skills/pocketbase/assets/` — Templates and static files

When you need to look up PocketBase details or find skill-related files, check this directory first — everything you need is already here. There is no need to search the user's home directory or other projects.

## 0. Design Workflow & Decision Making

**Read `.claude/skills/pocketbase/references/gotchas.md` FIRST** before writing any PocketBase code.
Your training data contains outdated v0.22 patterns that will fail on v0.23+.
Check field JSON: ensure properties are **flat** (no `options` wrapper) and collection key is `fields` (not `schema`).

### ⚠️ v0.22 Anti-Patterns — DO NOT USE

**Field properties must be FLAT — `options` wrapper was removed in v0.23+:**

```json
// WRONG (v0.22) — "options" wrapper does not exist in v0.23+
{"name": "status", "type": "select", "options": {"values": ["draft", "published"], "maxSelect": 1}}
{"name": "avatar", "type": "file", "options": {"maxSelect": 1, "maxSize": 5242880}}
{"name": "author", "type": "relation", "options": {"collectionId": "...", "maxSelect": 1}}

// CORRECT (v0.23+) — all properties are top-level (flat)
{"name": "status", "type": "select", "values": ["draft", "published"], "maxSelect": 1}
{"name": "avatar", "type": "file", "maxSelect": 1, "maxSize": 5242880}
{"name": "author", "type": "relation", "collectionId": "...", "maxSelect": 1}
```

Applies to ALL field types: `select` (values, maxSelect), `file` (maxSelect, maxSize, mimeTypes, thumbs), `relation` (collectionId, maxSelect), `text` (min, max, pattern).

**Collection JSON: use `fields` key, not `schema`:**

```json
// WRONG: {"name": "posts", "type": "base", "schema": [...]}
// CORRECT: {"name": "posts", "type": "base", "fields": [...]}
```

**Migration JS: use typed constructors, not `SchemaField`:**

```js
// WRONG:   collection.schema.addField(new SchemaField({type: "select", options: {values: ["a"]}}))
// CORRECT: collection.fields.add(new SelectField({name: "status", values: ["a"]}))
```

**Pre-Generation Checklist** — verify before writing any PocketBase code:

- [ ] Field properties are **flat** (no `options` wrapper)
- [ ] Collection JSON uses `fields` key (not `schema`)
- [ ] Migrations use typed constructors (`SelectField`, `TextField`, `RelationField`, etc.)
- [ ] Hooks use `e.next()` and `$app.findRecordById()` (not `$app.dao()`)
- [ ] Routes use `{paramName}` syntax (not `:paramName`)
- [ ] `@collection` references in API rules use `?=` (not `=`) — `=` breaks with 2+ rows

### Design Decision Tree

When building a PocketBase application, follow this sequence:

1. **Requirements** — Identify entities, relationships, and access patterns
2. **Collection types** — Choose `base`, `auth`, or `view` for each entity
3. **Fields** — Design fields per collection (`Read .claude/skills/pocketbase/references/field-types.md`)
4. **Relations** — Design relations (`Read .claude/skills/pocketbase/references/relation-patterns.md`)
5. **API rules** — Set security rules (`Read .claude/skills/pocketbase/references/api-rules-guide.md`)
   - **Default to `null` (deny all). Open only what is needed.**
   - `null` = superuser only, `""` = anyone including guests
6. **Create** — Use scripts or migrations to create collections
7. **Verify** — Run self-tests (see below)

### Self-Test Verification

After creating or modifying collections:

1. Confirm schema: `pb_collections.py get <name>`
2. CRUD smoke test: create → list → get → update → delete
3. Rule verification: test as non-superuser
   - Use `pb_auth.py --collection users --identity ... --password ...`
   - Verify denied access returns expected behavior

### Reference Index

| Topic | Reference |
|-------|-----------|
| Gotchas & pitfalls | `Read .claude/skills/pocketbase/references/gotchas.md` |
| API rules design | `Read .claude/skills/pocketbase/references/api-rules-guide.md` |
| Relation patterns | `Read .claude/skills/pocketbase/references/relation-patterns.md` |
| JS SDK (frontend) | `Read .claude/skills/pocketbase/references/js-sdk.md` |
| JSVM hooks (server) | `Read .claude/skills/pocketbase/references/jsvm-hooks.md` |
| File handling | `Read .claude/skills/pocketbase/references/file-handling.md` |

## 1. Prerequisites and Configuration

### Getting and Starting PocketBase

If PocketBase is not yet installed, guide the user to download the latest version:

**Check the latest version:**
```bash
curl -s https://api.github.com/repos/pocketbase/pocketbase/releases/latest | python3 -c "import sys,json; print(json.load(sys.stdin)['tag_name'])"
```

**Download URL pattern:**
```
https://github.com/pocketbase/pocketbase/releases/download/v{VERSION}/pocketbase_{VERSION}_{OS}_{ARCH}.zip
```

**Platform asset names:**

| Platform | Asset name |
|----------|------------|
| Linux amd64 | `pocketbase_{VERSION}_linux_amd64.zip` |
| Linux arm64 | `pocketbase_{VERSION}_linux_arm64.zip` |
| macOS amd64 | `pocketbase_{VERSION}_darwin_amd64.zip` |
| macOS arm64 (Apple Silicon) | `pocketbase_{VERSION}_darwin_arm64.zip` |
| Windows amd64 | `pocketbase_{VERSION}_windows_amd64.zip` |

**Download, extract, and start:**
```bash
# Example for Linux amd64 (replace VERSION with the actual version number, e.g. 0.28.0)
VERSION=0.28.0
curl -L -o pocketbase.zip "https://github.com/pocketbase/pocketbase/releases/download/v${VERSION}/pocketbase_${VERSION}_linux_amd64.zip"
unzip pocketbase.zip
./pocketbase serve
```

**Create a superuser:**
```bash
./pocketbase superuser create admin@example.com yourpassword
```

> **Agent instruction:** If the user's PocketBase is not running or not installed, always recommend downloading the latest version using the GitHub API one-liner above to determine the current version number.

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
python .claude/skills/pocketbase/scripts/pb_health.py
```

Runs a health check and (if credentials are available) a superuser authentication test.

## 2. Authentication

### Superuser Authentication

```bash
python .claude/skills/pocketbase/scripts/pb_auth.py
```

Authenticates using `PB_SUPERUSER_EMAIL` and `PB_SUPERUSER_PASSWORD` via `POST /api/collections/_superusers/auth-with-password`. Returns a token.

### User Authentication

```bash
python .claude/skills/pocketbase/scripts/pb_auth.py --collection users --identity user@example.com --password secret
```

Authenticates against any auth collection.

### Token Usage

Each script internally auto-acquires and caches the superuser token. On a 401 error, it retries authentication once.

**Details:** `Read .claude/skills/pocketbase/references/auth-api.md` — OAuth2, impersonate, password reset, etc.

## 3. Collection Management

### List

```bash
python .claude/skills/pocketbase/scripts/pb_collections.py list
python .claude/skills/pocketbase/scripts/pb_collections.py list --filter "name~'user'" --sort "-created"
```

### Get

```bash
python .claude/skills/pocketbase/scripts/pb_collections.py get posts
python .claude/skills/pocketbase/scripts/pb_collections.py get pbc_1234567890
```

### Create

```bash
# Inline JSON
python .claude/skills/pocketbase/scripts/pb_collections.py create '{"name":"posts","type":"base","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"}]}'

# From file
python .claude/skills/pocketbase/scripts/pb_collections.py create --file schema.json
```

Collection types:
- `base` — Standard data collection
- `auth` — Auth collection (automatically adds email, password, username, etc.)
- `view` — Read-only SQL view (requires `viewQuery`)

### Update

```bash
python .claude/skills/pocketbase/scripts/pb_collections.py update posts '{"listRule":"@request.auth.id != '\'''\''","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"},{"name":"status","type":"select","values":["draft","published"]}]}'
```

### Delete

```bash
python .claude/skills/pocketbase/scripts/pb_collections.py delete posts
```

### Import

```bash
python .claude/skills/pocketbase/scripts/pb_collections.py import --file collections.json
```

`collections.json` is a collections array, or `{"collections": [...], "deleteMissing": false}` format.

**Details:** `Read .claude/skills/pocketbase/references/collections-api.md` — API rule syntax, all parameters.

## 4. Record Management

### List

```bash
python .claude/skills/pocketbase/scripts/pb_records.py list posts
python .claude/skills/pocketbase/scripts/pb_records.py list posts --filter 'status="published"' --sort "-created" --expand "author" --page 1 --perPage 50
```

### Get

```bash
python .claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789
python .claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789 --expand "author,comments"
```

### Create

```bash
python .claude/skills/pocketbase/scripts/pb_records.py create posts '{"title":"Hello World","content":"<p>My first post</p>","status":"draft"}'
python .claude/skills/pocketbase/scripts/pb_records.py create posts --file record.json
```

### Update

```bash
python .claude/skills/pocketbase/scripts/pb_records.py update posts abc123def456789 '{"status":"published"}'
```

### Delete

```bash
python .claude/skills/pocketbase/scripts/pb_records.py delete posts abc123def456789
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

**Details:** `Read .claude/skills/pocketbase/references/records-api.md` — Batch operations, field selection, all operators.

## 5. Backup & Restore

```bash
# List
python .claude/skills/pocketbase/scripts/pb_backups.py list

# Create (omit name for auto-generated timestamp name)
python .claude/skills/pocketbase/scripts/pb_backups.py create
python .claude/skills/pocketbase/scripts/pb_backups.py create my_backup.zip

# Restore (caution: replaces all data; server restart involved)
python .claude/skills/pocketbase/scripts/pb_backups.py restore pb_backup_20240101120000.zip

# Delete
python .claude/skills/pocketbase/scripts/pb_backups.py delete pb_backup_20240101120000.zip
```

**Notes:**
- Restore replaces all data (no merge)
- Server becomes temporarily unavailable during restore
- Always create a backup of current data before restoring

**Details:** `Read .claude/skills/pocketbase/references/backups-api.md`

## 6. Migrations

### Auto-Migration (Primary Workflow)

PocketBase **automatically generates migration files** whenever you change a collection via the Admin UI or the API (e.g., `pb_collections.py create/update`). The generated files are placed in `pb_migrations/` and applied automatically on next startup.

**Typical workflow:**
1. Make schema changes via Admin UI or `pb_collections.py`
2. PocketBase writes a timestamped `.js` file to `pb_migrations/`
3. Commit the generated file to git
4. On deploy, PocketBase runs pending migrations automatically at startup

You do **not** need to create migration files manually for collection structure changes — they are already generated for you.

### Manual Migration (for operations not auto-generated)

Use `pb_create_migration.py` to generate an empty template when you need to write migration logic that the Admin UI cannot produce:

- Data transformation (copy/reformat existing field values)
- Raw SQL operations
- Seed data insertion
- Complex multi-step schema changes

```bash
python .claude/skills/pocketbase/scripts/pb_create_migration.py "backfill_user_slugs"
python .claude/skills/pocketbase/scripts/pb_create_migration.py "seed_categories" --dir ./pb_migrations
```

Generates a file in `{timestamp}_{description}.js` format. Write migration logic in the `// === UP ===` and `// === DOWN ===` sections.

### Common Patterns

| Pattern | UP | DOWN |
|---------|-----|------|
| Create collection | `new Collection({...})` + `app.save()` | `app.findCollectionByNameOrId()` + `app.delete()` |
| Add field | `collection.fields.add(new Field({...}))` | `collection.fields.removeByName()` |
| Remove field | `collection.fields.removeByName()` | `collection.fields.add(new Field({...}))` |
| Change rules | `collection.listRule = "..."` | Revert to original rule |
| Execute SQL | `app.db().newQuery("...").execute()` | Reverse SQL |
| Seed data | `new Record(collection)` + `app.save()` | Delete records |

**Details:** `Read .claude/skills/pocketbase/references/migrations.md` — Code examples for all patterns.
**Field types:** `Read .claude/skills/pocketbase/references/field-types.md` — All field types and configuration options.

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
| Connection check | `python .claude/skills/pocketbase/scripts/pb_health.py` | — |
| Superuser auth | `python .claude/skills/pocketbase/scripts/pb_auth.py` | `.claude/skills/pocketbase/references/auth-api.md` |
| User auth | `python .claude/skills/pocketbase/scripts/pb_auth.py --collection <name> --identity <email> --password <pw>` | `.claude/skills/pocketbase/references/auth-api.md` |
| List collections | `python .claude/skills/pocketbase/scripts/pb_collections.py list` | `.claude/skills/pocketbase/references/collections-api.md` |
| Get collection | `python .claude/skills/pocketbase/scripts/pb_collections.py get <name>` | `.claude/skills/pocketbase/references/collections-api.md` |
| Create collection | `python .claude/skills/pocketbase/scripts/pb_collections.py create '<json>'` | `.claude/skills/pocketbase/references/collections-api.md`, `.claude/skills/pocketbase/references/field-types.md` |
| Update collection | `python .claude/skills/pocketbase/scripts/pb_collections.py update <name> '<json>'` | `.claude/skills/pocketbase/references/collections-api.md` |
| Delete collection | `python .claude/skills/pocketbase/scripts/pb_collections.py delete <name>` | `.claude/skills/pocketbase/references/collections-api.md` |
| Import collections | `python .claude/skills/pocketbase/scripts/pb_collections.py import --file <file>` | `.claude/skills/pocketbase/references/collections-api.md` |
| List records | `python .claude/skills/pocketbase/scripts/pb_records.py list <collection>` | `.claude/skills/pocketbase/references/records-api.md` |
| Get record | `python .claude/skills/pocketbase/scripts/pb_records.py get <collection> <id>` | `.claude/skills/pocketbase/references/records-api.md` |
| Create record | `python .claude/skills/pocketbase/scripts/pb_records.py create <collection> '<json>'` | `.claude/skills/pocketbase/references/records-api.md` |
| Update record | `python .claude/skills/pocketbase/scripts/pb_records.py update <collection> <id> '<json>'` | `.claude/skills/pocketbase/references/records-api.md` |
| Delete record | `python .claude/skills/pocketbase/scripts/pb_records.py delete <collection> <id>` | `.claude/skills/pocketbase/references/records-api.md` |
| List backups | `python .claude/skills/pocketbase/scripts/pb_backups.py list` | `.claude/skills/pocketbase/references/backups-api.md` |
| Create backup | `python .claude/skills/pocketbase/scripts/pb_backups.py create [name]` | `.claude/skills/pocketbase/references/backups-api.md` |
| Restore backup | `python .claude/skills/pocketbase/scripts/pb_backups.py restore <key>` | `.claude/skills/pocketbase/references/backups-api.md` |
| Delete backup | `python .claude/skills/pocketbase/scripts/pb_backups.py delete <key>` | `.claude/skills/pocketbase/references/backups-api.md` |
| Generate migration | `python .claude/skills/pocketbase/scripts/pb_create_migration.py "<description>"` | `.claude/skills/pocketbase/references/migrations.md` |
| API rules design     | — | `.claude/skills/pocketbase/references/api-rules-guide.md`   |
| Common pitfalls      | — | `.claude/skills/pocketbase/references/gotchas.md`           |
| Relation patterns    | — | `.claude/skills/pocketbase/references/relation-patterns.md` |
| JS SDK reference     | — | `.claude/skills/pocketbase/references/js-sdk.md`            |
| JSVM hooks           | — | `.claude/skills/pocketbase/references/jsvm-hooks.md`        |
| File handling        | — | `.claude/skills/pocketbase/references/file-handling.md`     |
