---
name: pocketbase
description: >-
  PocketBase バックエンドの REST API を操作するスキル。コレクション CRUD、
  レコード CRUD、superuser/ユーザー認証、バックアップ・リストア、
  マイグレーションファイル生成を提供。PocketBase、pb_migrations、
  コレクション管理、レコード操作に関する要求で使用する。
version: 1.0.0
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# PocketBase Skill

PocketBase v0.23+ バックエンドを REST API 経由で操作するスキル。Python スクリプト（標準ライブラリのみ）を使用して認証、コレクション CRUD、レコード CRUD、バックアップ、マイグレーションファイル生成を行う。

## 1. 前提条件と設定

### 環境変数

| 変数名 | 必須 | デフォルト | 説明 |
|--------|------|-----------|------|
| `PB_URL` | No | `http://127.0.0.1:8090` | PocketBase のベース URL |
| `PB_SUPERUSER_EMAIL` | Yes* | — | superuser メールアドレス |
| `PB_SUPERUSER_PASSWORD` | Yes* | — | superuser パスワード |

*superuser 操作を行う場合は必須。

環境変数が未設定の場合、プロジェクトルートの `.env` ファイルを読み込む。

```env
PB_URL=http://127.0.0.1:8090
PB_SUPERUSER_EMAIL=admin@example.com
PB_SUPERUSER_PASSWORD=your-password
```

**重要:** 環境変数が未設定の場合、ユーザーに値を確認してから操作を実行すること。`.env` ファイルにクレデンシャルを書き込む場合は `.gitignore` に `.env` が含まれていることを確認すること。

### 接続確認

```bash
python ~/.claude/skills/pocketbase/scripts/pb_health.py
```

ヘルスチェックと（クレデンシャルがある場合は）superuser 認証テストを実行する。

## 2. 認証

### Superuser 認証

```bash
python ~/.claude/skills/pocketbase/scripts/pb_auth.py
```

`PB_SUPERUSER_EMAIL` と `PB_SUPERUSER_PASSWORD` を使って `POST /api/collections/_superusers/auth-with-password` で認証。トークンを返す。

### ユーザー認証

```bash
python ~/.claude/skills/pocketbase/scripts/pb_auth.py --collection users --identity user@example.com --password secret
```

任意の auth コレクションに対して認証を実行。

### トークンの使い方

各スクリプトは内部的に superuser トークンを自動取得・キャッシュする。401 エラー時は再認証を 1 回リトライする。

**詳細:** `Read references/auth-api.md` — OAuth2、impersonate、パスワードリセット等。

## 3. コレクション管理

### 一覧

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py list
python ~/.claude/skills/pocketbase/scripts/pb_collections.py list --filter "name~'user'" --sort "-created"
```

### 取得

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py get posts
python ~/.claude/skills/pocketbase/scripts/pb_collections.py get pbc_1234567890
```

### 作成

```bash
# インラインJSON
python ~/.claude/skills/pocketbase/scripts/pb_collections.py create '{"name":"posts","type":"base","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"}]}'

# ファイルから
python ~/.claude/skills/pocketbase/scripts/pb_collections.py create --file schema.json
```

コレクション型:
- `base` — 標準データコレクション
- `auth` — 認証コレクション（email, password, username 等を自動追加）
- `view` — 読み取り専用 SQL ビュー（`viewQuery` が必要）

### 更新

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py update posts '{"listRule":"@request.auth.id != '\'''\''","fields":[{"name":"title","type":"text","required":true},{"name":"content","type":"editor"},{"name":"status","type":"select","values":["draft","published"]}]}'
```

### 削除

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py delete posts
```

### インポート

```bash
python ~/.claude/skills/pocketbase/scripts/pb_collections.py import --file collections.json
```

`collections.json` はコレクション配列、または `{"collections": [...], "deleteMissing": false}` 形式。

**詳細:** `Read references/collections-api.md` — API ルール構文、全パラメータ。

## 4. レコード管理

### 一覧

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py list posts
python ~/.claude/skills/pocketbase/scripts/pb_records.py list posts --filter 'status="published"' --sort "-created" --expand "author" --page 1 --perPage 50
```

### 取得

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789
python ~/.claude/skills/pocketbase/scripts/pb_records.py get posts abc123def456789 --expand "author,comments"
```

### 作成

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py create posts '{"title":"Hello World","content":"<p>My first post</p>","status":"draft"}'
python ~/.claude/skills/pocketbase/scripts/pb_records.py create posts --file record.json
```

### 更新

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py update posts abc123def456789 '{"status":"published"}'
```

### 削除

```bash
python ~/.claude/skills/pocketbase/scripts/pb_records.py delete posts abc123def456789
```

### フィルター構文クイックリファレンス

| 演算子 | 説明 | 例 |
|--------|------|-----|
| `=` | 等しい | `status = "published"` |
| `!=` | 等しくない | `status != "draft"` |
| `>`, `>=`, `<`, `<=` | 比較 | `count > 10` |
| `~` | 含む（LIKE） | `title ~ "hello"` |
| `!~` | 含まない | `title !~ "test"` |
| `?=`, `?~` 等 | 配列/多値フィールド | `tags ?= "news"` |

グルーピング: `(expr1 && expr2) || expr3`

### ソート

`-created` (DESC), `+title` (ASC), `@random`。カンマ区切りで複数指定可。

### Expand（リレーション展開）

`--expand "author"` — 直接リレーション。
`--expand "author.profile"` — ネストリレーション（最大 6 階層）。
`--expand "author,category"` — 複数リレーション。

**詳細:** `Read references/records-api.md` — バッチ操作、フィールド選択、全演算子。

## 5. バックアップ・リストア

```bash
# 一覧
python ~/.claude/skills/pocketbase/scripts/pb_backups.py list

# 作成（名前省略でタイムスタンプ付き自動命名）
python ~/.claude/skills/pocketbase/scripts/pb_backups.py create
python ~/.claude/skills/pocketbase/scripts/pb_backups.py create my_backup.zip

# リストア（注意: 全データが置換される。サーバー再起動あり）
python ~/.claude/skills/pocketbase/scripts/pb_backups.py restore pb_backup_20240101120000.zip

# 削除
python ~/.claude/skills/pocketbase/scripts/pb_backups.py delete pb_backup_20240101120000.zip
```

**注意事項:**
- リストアは全データを置換する（マージ不可）
- リストア中はサーバーが一時的に利用不可になる
- リストア前に必ず現在のバックアップを作成すること

**詳細:** `Read references/backups-api.md`

## 6. マイグレーション

### ファイル生成

```bash
python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "create_posts_collection"
python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "add_status_field" --dir ./pb_migrations
```

`{timestamp}_{description}.js` 形式のファイルを生成する。テンプレートの `// === UP ===` と `// === DOWN ===` セクションにマイグレーションロジックを記述する。

### 一般的なパターン

| パターン | UP | DOWN |
|----------|-----|------|
| コレクション作成 | `new Collection({...})` + `app.save()` | `app.findCollectionByNameOrId()` + `app.delete()` |
| フィールド追加 | `collection.fields.add(new Field({...}))` | `collection.fields.removeByName()` |
| フィールド削除 | `collection.fields.removeByName()` | `collection.fields.add(new Field({...}))` |
| ルール変更 | `collection.listRule = "..."` | 元のルールに戻す |
| SQL 実行 | `app.db().newQuery("...").execute()` | 逆の SQL |
| シードデータ | `new Record(collection)` + `app.save()` | レコード削除 |

**詳細:** `Read references/migrations.md` — 全パターンのコード例。
**フィールド型:** `Read references/field-types.md` — 全フィールド型と設定オプション。

## 7. エラーハンドリング

全スクリプトは構造化 JSON を出力する:

```json
{
  "success": true,
  "status": 200,
  "data": { ... }
}
```

### 共通エラーコード

| HTTP ステータス | 意味 | 対処 |
|----------------|------|------|
| 400 | Bad Request | リクエストボディを確認。`data` フィールドにバリデーションエラー詳細あり |
| 401 | Unauthorized | トークン期限切れ。スクリプトは自動リトライする |
| 403 | Forbidden | API ルールで操作が禁止されている。ルールを確認 |
| 404 | Not Found | コレクションまたはレコードが存在しない |

バリデーションエラーの例:
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

## 8. クイックリファレンス

| タスク | スクリプト | 詳細リファレンス |
|--------|-----------|----------------|
| 接続確認 | `python ~/.claude/skills/pocketbase/scripts/pb_health.py` | — |
| superuser 認証 | `python ~/.claude/skills/pocketbase/scripts/pb_auth.py` | `references/auth-api.md` |
| ユーザー認証 | `python ~/.claude/skills/pocketbase/scripts/pb_auth.py --collection <name> --identity <email> --password <pw>` | `references/auth-api.md` |
| コレクション一覧 | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py list` | `references/collections-api.md` |
| コレクション詳細 | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py get <name>` | `references/collections-api.md` |
| コレクション作成 | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py create '<json>'` | `references/collections-api.md`, `references/field-types.md` |
| コレクション更新 | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py update <name> '<json>'` | `references/collections-api.md` |
| コレクション削除 | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py delete <name>` | `references/collections-api.md` |
| コレクションインポート | `python ~/.claude/skills/pocketbase/scripts/pb_collections.py import --file <file>` | `references/collections-api.md` |
| レコード一覧 | `python ~/.claude/skills/pocketbase/scripts/pb_records.py list <collection>` | `references/records-api.md` |
| レコード取得 | `python ~/.claude/skills/pocketbase/scripts/pb_records.py get <collection> <id>` | `references/records-api.md` |
| レコード作成 | `python ~/.claude/skills/pocketbase/scripts/pb_records.py create <collection> '<json>'` | `references/records-api.md` |
| レコード更新 | `python ~/.claude/skills/pocketbase/scripts/pb_records.py update <collection> <id> '<json>'` | `references/records-api.md` |
| レコード削除 | `python ~/.claude/skills/pocketbase/scripts/pb_records.py delete <collection> <id>` | `references/records-api.md` |
| バックアップ一覧 | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py list` | `references/backups-api.md` |
| バックアップ作成 | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py create [name]` | `references/backups-api.md` |
| バックアップ復元 | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py restore <key>` | `references/backups-api.md` |
| バックアップ削除 | `python ~/.claude/skills/pocketbase/scripts/pb_backups.py delete <key>` | `references/backups-api.md` |
| マイグレーション生成 | `python ~/.claude/skills/pocketbase/scripts/pb_create_migration.py "<description>"` | `references/migrations.md` |
