# TypeScript Type Generation

Automatically generate TypeScript types from PocketBase collection schemas using `pocketbase-typegen`.

---

## Installation

```bash
npm install -D pocketbase-typegen
```

---

## Type Generation Commands

### URL Mode (Recommended — fetches from a running PocketBase)

```bash
npx pocketbase-typegen --url http://127.0.0.1:8090 \
  --email admin@example.com \
  --password yourpassword \
  --out src/types/pocketbase-types.ts
```

### DB File Mode (can be used even when PocketBase is stopped)

```bash
npx pocketbase-typegen --db ../backend/pb_data/data.db \
  --out src/types/pocketbase-types.ts
```

> DB file mode directly accesses the PocketBase data file. Only usable when the file is available on the local machine.

---

## Integration with npm Scripts

`package.json`:

```json
{
  "scripts": {
    "typegen": "pocketbase-typegen --url http://127.0.0.1:8090 --email admin@example.com --password yourpassword --out src/types/pocketbase-types.ts"
  }
}
```

```bash
npm run typegen
```

> Confirm the superuser email and password with the user.

---

## Structure of Generated Types

`pocketbase-typegen` generates the following types:

```ts
// Collection name enum
export enum Collections {
  Posts = "posts",
  Users = "users",
}

// Record type (for input — used in create/update)
export type PostsRecord = {
  title: string;
  content?: string;
  status?: "draft" | "published";
  author?: string;  // relation → ID string
};

// Response type (for output — includes system fields from the API response)
export type PostsResponse<Texpand = unknown> = PostsRecord & {
  id: string;
  collectionId: string;
  collectionName: string;
  created: string;
  updated: string;
  expand?: Texpand;
};

// Typed PocketBase client
export type TypedPocketBase = PocketBase & {
  collection(idOrName: "posts"): RecordService<PostsResponse>;
  collection(idOrName: "users"): RecordService<UsersResponse>;
};
```

---

## Using the Types

### Typing the PB Client

```ts
// src/lib/pocketbase.ts
import PocketBase from "pocketbase";
import type { TypedPocketBase } from "@/types/pocketbase-types";

export const pb = new PocketBase(
  import.meta.env.VITE_PB_URL,
) as TypedPocketBase;
```

The return value of `pb.collection("posts")` automatically becomes `RecordService<PostsResponse>`.

### Usage in Queries

```ts
// Type inference works — result is ListResult<PostsResponse>
const result = await pb.collection("posts").getList(1, 50);

// When explicitly specifying the type parameter
const result = await pb.collection("posts").getList<PostsResponse>(1, 50);
```

### Typing expand

```ts
// Define a type with expand
type PostWithAuthor = PostsResponse<{ author: UsersResponse }>;

const post = await pb.collection("posts").getOne<PostWithAuthor>(id, {
  expand: "author",
});

// post.expand?.author is of type UsersResponse
```

### Usage in Mutations

```ts
import type { PostsRecord } from "@/types/pocketbase-types";

// PostsRecord is used for create/update payloads
const newPost: PostsRecord = {
  title: "Hello",
  content: "World",
  status: "draft",
};
await pb.collection("posts").create(newPost);
```

---

## Regenerating After Collection Changes

Regenerate types after modifying PocketBase collection schemas:

```bash
npm run typegen
```

After regeneration, the TypeScript compiler will detect type mismatches.

**Workflow:**
1. Modify schema via PocketBase Admin UI or `pb_collections.py`
2. Regenerate types with `npm run typegen`
3. Check and fix TypeScript errors
4. Commit
