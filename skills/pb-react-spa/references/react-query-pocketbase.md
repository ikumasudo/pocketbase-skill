# TanStack Query + PocketBase Integration Patterns

A collection of data fetching and mutation patterns combining TanStack Query (React Query) with the PocketBase JS SDK.

> For the basic PocketBase JS SDK API, see `references/js-sdk.md` in the PocketBase skill.

---

## Importing the PB Client

```ts
import { pb } from "@/lib/pocketbase";
```

Share a singleton instance across all components. See SKILL.md section 2-2 for how to create it.

---

## queryOptions Pattern (Recommended)

Use `queryOptions` to separate type-safe query definitions as functions:

```ts
import { queryOptions } from "@tanstack/react-query";
import { pb } from "@/lib/pocketbase";
import type { PostsResponse } from "@/types/pocketbase-types";

// Query definition (defined outside components)
export const postsQueryOptions = queryOptions({
  queryKey: ["posts"],
  queryFn: () =>
    pb.collection("posts").getList<PostsResponse>(1, 50, {
      sort: "-created",
    }),
});

export const postQueryOptions = (id: string) =>
  queryOptions({
    queryKey: ["posts", id],
    queryFn: () => pb.collection("posts").getOne<PostsResponse>(id),
  });
```

Usage in components:

```tsx
import { useSuspenseQuery } from "@tanstack/react-query";
import { postsQueryOptions } from "@/lib/queries/posts";

function PostList() {
  const { data } = useSuspenseQuery(postsQueryOptions);
  return (
    <ul>
      {data.items.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

Integration with TanStack Router's `loader`:

```ts
// routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: ({ context }) =>
    context.queryClient.ensureQueryData(postsQueryOptions),
  component: PostList,
});
```

---

## CRUD Query Patterns

### List with Pagination

```ts
export const paginatedPostsQueryOptions = (page: number, perPage = 20) =>
  queryOptions({
    queryKey: ["posts", "list", { page, perPage }],
    queryFn: () =>
      pb.collection("posts").getList<PostsResponse>(page, perPage, {
        sort: "-created",
        filter: 'status = "published"',
      }),
  });
```

### Single Record

```ts
export const postQueryOptions = (id: string) =>
  queryOptions({
    queryKey: ["posts", id],
    queryFn: () =>
      pb.collection("posts").getOne<PostsResponse>(id, {
        expand: "author",
      }),
    enabled: !!id,
  });
```

### Filtered List

```ts
export const filteredPostsQueryOptions = (status: string) =>
  queryOptions({
    queryKey: ["posts", "list", { status }],
    queryFn: () =>
      pb.collection("posts").getFullList<PostsResponse>({
        filter: pb.filter("status = {:status}", { status }),
        sort: "-created",
      }),
  });
```

> **Important:** Always use `pb.filter()` when including user input in filters (prevents injection).

---

## Mutation Patterns

### Create Record

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: { title: string; content: string }) =>
      pb.collection("posts").create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["posts"] });
    },
  });
}
```

### Update Record

```ts
function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<PostsRecord> }) =>
      pb.collection("posts").update(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: ["posts"] });
      queryClient.invalidateQueries({ queryKey: ["posts", id] });
    },
  });
}
```

### Delete Record

```ts
function useDeletePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => pb.collection("posts").delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["posts"] });
    },
  });
}
```

---

## Optimistic Updates

```ts
function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<PostsRecord> }) =>
      pb.collection("posts").update(id, data),
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: ["posts", id] });
      const previous = queryClient.getQueryData(["posts", id]);
      queryClient.setQueryData(["posts", id], (old: PostsResponse) => ({
        ...old,
        ...data,
      }));
      return { previous };
    },
    onError: (_err, { id }, context) => {
      if (context?.previous) {
        queryClient.setQueryData(["posts", id], context.previous);
      }
    },
    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: ["posts", id] });
    },
  });
}
```

---

## Realtime Subscription + Cache Invalidation

Automatically update TanStack Query cache with PocketBase realtime events:

```ts
import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { pb } from "@/lib/pocketbase";

function useRealtimeInvalidation(collection: string) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const unsubscribe = pb.collection(collection).subscribe("*", (e) => {
      queryClient.invalidateQueries({ queryKey: [collection] });

      if (e.action === "update" || e.action === "delete") {
        queryClient.invalidateQueries({
          queryKey: [collection, e.record.id],
        });
      }
    });

    return () => {
      unsubscribe.then((fn) => fn());
    };
  }, [collection, queryClient]);
}
```

Usage in components:

```tsx
function PostList() {
  useRealtimeInvalidation("posts");
  const { data } = useSuspenseQuery(postsQueryOptions);
  // ... automatically updated in realtime
}
```

---

## File URL Retrieval

```ts
import { pb } from "@/lib/pocketbase";

// Image URL (with thumbnail)
const avatarUrl = pb.files.getURL(record, record.avatar, {
  thumb: "100x100",
});

// Usage in a React component
function Avatar({ record }: { record: UsersResponse }) {
  if (!record.avatar) return null;
  const url = pb.files.getURL(record, record.avatar, { thumb: "100x100" });
  return <img src={url} alt={record.name} />;
}
```

---

## Error Handling

```ts
import { ClientResponseError } from "pocketbase";

function useCreatePost() {
  return useMutation({
    mutationFn: (data: { title: string; content: string }) =>
      pb.collection("posts").create(data),
    onError: (error) => {
      if (error instanceof ClientResponseError) {
        if (error.status === 400 && error.response.data) {
          // Validation error (per field)
          const fieldErrors = error.response.data;
          // { title: { code: "validation_required", message: "..." } }
        }
        if (error.status === 403) {
          // Access denied by API rules
        }
      }
    },
  });
}
```

---

## Query Key Design Guidelines

| Pattern | Query Key | Use Case |
|---------|-----------|----------|
| Entire collection | `["posts"]` | List invalidation |
| Pagination | `["posts", "list", { page, perPage }]` | Specific page |
| Filtered | `["posts", "list", { status }]` | Per filter criteria |
| Single record | `["posts", id]` | Individual record |
| With relation expand | `["posts", id, { expand: "author" }]` | With expand |

`invalidateQueries({ queryKey: ["posts"] })` invalidates all queries with the `["posts"]` prefix.
