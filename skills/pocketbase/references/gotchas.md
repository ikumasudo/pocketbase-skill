# PocketBase Gotchas & Common Mistakes

Version pitfalls, rule traps, and data behavior surprises that cause hard-to-debug bugs.

---

## CRITICAL: v0.22 to v0.23 Breaking Changes

If your training data or examples reference v0.22 patterns, they will fail on v0.23+.

### Admin Collection Renamed

| v0.22 | v0.23+ |
|-------|--------|
| `_admins` collection | `_superusers` collection |
| `POST /api/admins/auth-with-password` | `POST /api/collections/_superusers/auth-with-password` |
| `app.findAdminByEmail()` | `app.findAuthRecordByEmail("_superusers", email)` |
| `Admin{}` type | Standard `Record{}` in `_superusers` |

### Schema Field API Renamed

| v0.22 | v0.23+ |
|-------|--------|
| `schema` property | `fields` property |
| `SchemaField{Type: "text", ...}` | `TextField{Name: "title", ...}` (typed constructors) |
| Generic `SchemaField` struct | Type-specific: `TextField`, `NumberField`, `BoolField`, `EmailField`, `URLField`, `DateField`, `SelectField`, `JSONField`, `FileField`, `RelationField`, `EditorField`, `AutodateField` |

Migration example (v0.23+ JS):
```js
// v0.22 style — WRONG on v0.23+
collection.schema.addField(new SchemaField({type: "text", name: "title"}))

// v0.23+ style — CORRECT
collection.fields.add(new TextField({name: "title", required: true}))
```

### Field Properties Flattened (No `options` Wrapper)

In v0.22, many field types wrapped their properties in an `options` object. In v0.23+, **all properties are flat (top-level)**. The `options` key does not exist.

| Field Type | v0.22 wrapped property | v0.23+ flat property |
|------------|------------------------|----------------------|
| select | `options: {values: [...], maxSelect: 1}` | `values: [...], maxSelect: 1` |
| file | `options: {maxSelect: 1, maxSize: N, mimeTypes: [...]}` | `maxSelect: 1, maxSize: N, mimeTypes: [...]` |
| relation | `options: {collectionId: "...", maxSelect: 1}` | `collectionId: "...", maxSelect: 1` |
| text | `options: {min: 1, max: 500, pattern: "..."}` | `min: 1, max: 500, pattern: "..."` |

```json
// WRONG (v0.22) — options wrapper
{"name": "status", "type": "select", "options": {"values": ["draft", "published"], "maxSelect": 1}}
{"name": "avatar", "type": "file", "options": {"maxSelect": 1, "maxSize": 5242880}}

// CORRECT (v0.23+) — flat properties
{"name": "status", "type": "select", "values": ["draft", "published"], "maxSelect": 1}
{"name": "avatar", "type": "file", "maxSelect": 1, "maxSize": 5242880}
```

### `dao` Removed — Use `$app` Directly

| v0.22 | v0.23+ |
|-------|--------|
| `$app.dao().findRecordById(...)` | `$app.findRecordById(...)` |
| `$app.dao().saveRecord(...)` | `$app.save(record)` |
| `$app.dao().deleteRecord(...)` | `$app.delete(record)` |
| `$app.dao().db()` | `$app.db()` |

### Router / HTTP Handler Changes

| v0.22 | v0.23+ |
|-------|--------|
| Path param: `:paramName` | Path param: `{paramName}` |
| `e.PathParam("name")` | `e.Request.PathValue("name")` |
| echo framework | standard `net/http` |
| `e.String(200, "...")` | `e.String("...")` or `e.JSON(obj)` |

```js
// v0.22 style — WRONG
routerAdd("GET", "/api/hello/:name", (e) => {
    let name = e.pathParam("name")
    return e.string(200, "Hello " + name)
})

// v0.23+ style — CORRECT
routerAdd("GET", "/api/hello/{name}", (e) => {
    let name = e.request.pathValue("name")
    return e.string("Hello " + name)
})
```

### Hook Event Renames and `e.next()` Required

| v0.22 | v0.23+ |
|-------|--------|
| `onModelBeforeCreate` | `onRecordCreate` |
| `onModelAfterCreate` | `onRecordCreateExecute` (before DB) / `onRecordAfterCreateSuccess` |
| `onModelBeforeUpdate` | `onRecordUpdate` |
| `onModelBeforeDelete` | `onRecordDelete` |

**`e.next()` is now required** in hooks to continue the execution chain:

```js
// v0.22 style — WRONG on v0.23+
onModelBeforeCreate((e) => {
    // handler runs but no e.next()
})

// v0.23+ style — CORRECT
onRecordCreate((e) => {
    // do something before save
    e.next()  // REQUIRED — continues to DB write
    // do something after save (within same hook)
})
```

### File Upload: Replaces Instead of Appends

In v0.23+, assigning a file field **replaces all existing files**. To append or remove:

```js
// v0.23+ — Append files to existing
{"documents+": [newFile]}

// v0.23+ — Remove specific file
{"documents-": ["filename.pdf"]}

// v0.23+ — Replace all (overwrites existing)
{"documents": [newFile]}
```

---

## Rule Evaluation Traps

**This is the most common source of security bugs.**

### `""` vs `null` — Critical Difference

| Rule Value | Meaning | Who Can Access |
|------------|---------|----------------|
| `null` | Locked (deny all) | Superusers only |
| `""` | Open (allow all) | Anyone, including unauthenticated guests |
| `"@request.auth.id != ''"` | Auth required | Any authenticated user |
| `"@request.auth.id = id"` | Owner only | Record owner only |

**`""` (empty string) does NOT mean "no rule" or "deny all" — it means "allow everyone."**

When creating collections, all rules default to `null` (deny all). Always set rules explicitly and default to the most restrictive option.

### listRule Returns 200, Not 403

When `listRule` denies access, the API returns HTTP 200 with an **empty items array**, not 403:

```json
{"page": 1, "perPage": 30, "totalItems": 0, "totalPages": 0, "items": []}
```

This means you cannot distinguish "no records exist" from "access denied" via HTTP status. Test by checking record count with a superuser versus a regular user.

### Superusers Bypass ALL Rules

Superusers are exempt from all API rules. **Never test rules while authenticated as a superuser** — you will always see data even if rules deny regular users.

Test rule behavior with:
```bash
# Get a regular user token
python ~/.claude/skills/pocketbase/scripts/pb_auth.py \
  --collection users --identity user@example.com --password secret
```

Then use that token for testing instead of superuser credentials.

---

## Data Behavior Surprises

### `fields` in Update is a Full Replacement

When updating a collection via `PATCH /api/collections/{id}`, the `fields` property **replaces all existing fields** — it does not merge. If you send only the new fields, existing fields will be deleted.

**WRONG — deletes all existing fields:**
```json
// Attempting to "add" a relation field
{"fields": [{"name": "author", "type": "relation", "collectionId": "users"}]}
```

**CORRECT — include ALL fields (existing + new):**
```json
{"fields": [
  {"name": "title", "type": "text", "required": true},
  {"name": "content", "type": "editor"},
  {"name": "author", "type": "relation", "collectionId": "users"}
]}
```

**Tip:** Use `pb_collections.py get <name>` first to retrieve the current fields, then append your new field to that list.

**Best practice:** For multi-collection setups, use `import` instead of individual create+update — it handles everything in one call.

### Zero Defaults, Not Null

PocketBase stores **zero values** for missing fields, not `null` (except JSON fields):

| Field Type | Missing Value Stored As |
|------------|------------------------|
| text | `""` |
| number | `0` |
| bool | `false` |
| select | `""` (single) or `[]` (multi) |
| relation | `""` (single) or `[]` (multi) |
| json | `null` |
| date | `""` |

This matters for filter conditions: `price = 0` will match records where price was never set.

### Raw SQL Bypasses `created`/`updated`

Queries via `app.db()` (raw SQL) **do not trigger** autodate updates. Use `app.save(record)` to ensure timestamps are managed:

```js
// WRONG — updated timestamp will not change
app.db().newQuery("UPDATE posts SET title='New' WHERE id='abc'").execute()

// CORRECT — updated timestamp is managed by PocketBase
const record = $app.findRecordById("posts", "abc")
record.set("title", "New")
$app.save(record)
```

### Auth Records Require `email` in v0.23+

In v0.23+, the `email` field in auth collections is **required by default**. Creating auth records without email via migrations or hooks will fail unless you explicitly set `emailVisibility` and provide an email.

### `_externalAuths` Field Rename

In v0.23+, the field `recordId` in `_externalAuths` was renamed to `recordRef`:

```js
// v0.22 — WRONG
app.findFirstRecordByFilter("_externalAuths", "recordId = {:id}", {id: userId})

// v0.23+ — CORRECT
app.findFirstRecordByFilter("_externalAuths", "recordRef = {:id}", {id: userId})
```

---

## Filter & Query Pitfalls

### Default `=` vs `?=` for Multi-Value Fields

For **select**, **relation**, and **file** fields with multiple values:

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | **ALL** values must match (AND) | `tags = "news"` — record must have ONLY `"news"` |
| `?=` | **ANY** value matches (OR) | `tags ?= "news"` — record has `"news"` among its tags |

**Almost always use `?=` when checking multi-value fields.** Using `=` on a multi-value field for "contains" checks is a very common mistake.

```
// WRONG — checks if tags is exactly ["news"] (all elements equal "news")
tags = "news"

// CORRECT — checks if "news" is among the tags
tags ?= "news"
```

### `@collection` Cross-Collection References Always Require `?=`

When using `@collection.X.field` in API rules, the operator semantics change:

| Operator | Meaning in `@collection` context |
|----------|----------------------------------|
| `=` | **ALL rows** in the joined collection must satisfy the condition (universal quantification) |
| `?=` | **At least one row** exists satisfying the condition (existential quantification) |

With `=`, PocketBase generates a correlated subquery that requires EVERY row in the collection to match. This works when there is exactly one matching row, but **silently denies access as soon as there are 2+ rows** (e.g., a user with multiple memberships, multiple subscriptions, multiple roles).

**WRONG — breaks with 2+ membership rows:**
```
@collection.memberships.userId = @request.auth.id && @collection.memberships.orgId = orgId
```

**CORRECT — EXISTS semantics, works with any number of rows:**
```
@collection.memberships.userId ?= @request.auth.id && @collection.memberships.orgId ?= orgId
```

**With aliases (multiple joins to the same collection):**
```
// WRONG
@collection.memberships:m.userId = @request.auth.id && @collection.memberships:m.orgId = orgId

// CORRECT — alias ties the two conditions to the same row
@collection.memberships:m.userId ?= @request.auth.id && @collection.memberships:m.orgId ?= orgId
```

> **Note:** When multiple `?=` conditions reference the same `@collection.X` (without an alias), PocketBase applies them to the same row — i.e., a single row must satisfy ALL conditions. Use aliases (`:name` suffix) to create independent joins when needed.

**Rule: always use `?=` (never `=`) for `@collection` references in API rules.**

### Date Format in Filters

Date values in filter expressions must be **quoted strings** with timezone:

```
// WRONG — no quotes, wrong format
created > 2024-01-01

// CORRECT
created > "2024-01-01 00:00:00.000Z"
```

Use `@now`, `@todayStart`, `@todayEnd`, etc. for relative dates instead of hardcoded strings.

### Back-Relation Expand Limitations

When expanding back-relations (reverse side of a relation):

- Maximum **1000 items** returned in expand
- **No sorting** within expanded back-relation items
- **No filtering** within expanded back-relation items
- Workaround: query the related collection directly with a filter

```bash
# Instead of expanding back-relations on a user, query posts directly
python ~/.claude/skills/pocketbase/scripts/pb_records.py list posts \
  --filter 'author = "USER_ID"' --sort "-created"
```

### `@request.body.*` Excluded from File Uploads

When a request is `multipart/form-data` (file upload), `@request.body.*` references in rules **cannot access** the submitted file fields. Only non-file fields are available via `@request.body.*` in that context.

---

## Naming Conventions

### Initialism Normalization

PocketBase normalizes initialisms in field/type names to uppercase:

| Input | Stored As |
|-------|-----------|
| `Json` | `JSON` |
| `Url` | `URL` |
| `Smtp` | `SMTP` |
| `Api` | `API` |

This affects Go type names in migrations (use `JSONField`, `URLField`, not `JsonField`, `UrlField`).

### System Collections

System collections have underscore-prefixed names:

| Collection | Description |
|------------|-------------|
| `_superusers` | Admin/superuser accounts (v0.23+) |
| `_externalAuths` | OAuth2 external auth links |
| `_authOrigins` | Auth origin tracking |
| `_mfas` | Multi-factor auth records |
| `_otps` | One-time password records |

### Reserved Field Names

These field names are reserved and cannot be used in custom collections:

`id`, `created`, `updated`, `collectionId`, `collectionName`, `expand`

Attempting to create fields with these names will result in a validation error.
