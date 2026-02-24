# Authentication Patterns

Integration patterns for PocketBase AuthStore with React. Protected routes with TanStack Router, login/logout, and OAuth2 flows.

---

## Auth Context Provider

`frontend/src/lib/auth.tsx`:

```tsx
import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
} from "react";
import type { RecordModel } from "pocketbase";
import { pb } from "./pocketbase";

interface AuthContext {
  user: RecordModel | null;
  isLoggedIn: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContext | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<RecordModel | null>(
    pb.authStore.record,
  );

  useEffect(() => {
    // Watch AuthStore changes and sync state
    const unsubscribe = pb.authStore.onChange((_token, record) => {
      setUser(record);
    });
    return () => unsubscribe();
  }, []);

  const login = useCallback(async (email: string, password: string) => {
    await pb.collection("users").authWithPassword(email, password);
    // onChange calls setUser, so no manual update needed
  }, []);

  const logout = useCallback(() => {
    pb.authStore.clear();
  }, []);

  const value = useMemo(
    () => ({
      user,
      isLoggedIn: pb.authStore.isValid,
      login,
      logout,
    }),
    [user, login, logout],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

### AuthProvider Placement

Place alongside QueryClientProvider in `__root.tsx`:

```tsx
import { AuthProvider } from "@/lib/auth";

function RootComponent() {
  return (
    <AuthProvider>
      <Outlet />
    </AuthProvider>
  );
}
```

---

## Protected Routes with TanStack Router

### Authentication Check via beforeLoad

```ts
// routes/_authenticated.tsx (layout route)
import { createFileRoute, redirect, Outlet } from "@tanstack/react-router";
import { pb } from "@/lib/pocketbase";

export const Route = createFileRoute("/_authenticated")({
  beforeLoad: () => {
    if (!pb.authStore.isValid) {
      throw redirect({ to: "/login" });
    }
  },
  component: () => <Outlet />,
});
```

All routes under this `_authenticated` layout require authentication:

```
routes/
├── __root.tsx
├── _authenticated.tsx          ← Auth check (layout route)
├── _authenticated/
│   ├── dashboard.tsx           ← Auth required
│   ├── settings.tsx            ← Auth required
│   └── posts/
│       ├── index.tsx           ← Auth required
│       └── $postId.tsx         ← Auth required
├── login.tsx                   ← No auth required (public)
└── index.tsx                   ← No auth required (public)
```

### Authentication Check with Token Refresh

```ts
export const Route = createFileRoute("/_authenticated")({
  beforeLoad: async () => {
    if (!pb.authStore.isValid) {
      throw redirect({ to: "/login" });
    }
    // Refresh if token is valid but close to expiration
    try {
      await pb.collection("users").authRefresh();
    } catch {
      pb.authStore.clear();
      throw redirect({ to: "/login" });
    }
  },
  component: () => <Outlet />,
});
```

---

## Login Component

```tsx
// routes/login.tsx
import { createFileRoute, useNavigate } from "@tanstack/react-router";
import { useState } from "react";
import { useAuth } from "@/lib/auth";

export const Route = createFileRoute("/login")({
  component: LoginPage,
});

function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError("");
    try {
      await login(email, password);
      navigate({ to: "/dashboard" });
    } catch {
      setError("Invalid email or password");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      {error && <p>{error}</p>}
      <button type="submit">Log in</button>
    </form>
  );
}
```

---

## Logout

```tsx
import { useAuth } from "@/lib/auth";
import { useNavigate } from "@tanstack/react-router";

function LogoutButton() {
  const { logout } = useAuth();
  const navigate = useNavigate();

  const handleLogout = () => {
    logout();
    navigate({ to: "/login" });
  };

  return <button onClick={handleLogout}>Log out</button>;
}
```

---

## OAuth2 Flow

```tsx
import { pb } from "@/lib/pocketbase";

async function loginWithGoogle() {
  // Start OAuth2 authentication flow via browser popup
  const authData = await pb.collection("users").authWithOAuth2({
    provider: "google",
  });
  // On success, automatically saved to pb.authStore
  // AuthProvider's onChange also auto-updates the user state
}
```

> OAuth2 providers must be configured in advance via the PocketBase Admin UI (Settings → Auth providers).

---

## Integration with TanStack Query

Control `enabled` for queries that depend on authentication state:

```ts
import { pb } from "@/lib/pocketbase";

export const myPostsQueryOptions = queryOptions({
  queryKey: ["posts", "mine"],
  queryFn: () =>
    pb.collection("posts").getFullList({
      filter: pb.filter("author = {:userId}", {
        userId: pb.authStore.record?.id,
      }),
    }),
  enabled: pb.authStore.isValid,
});
```

Clear cache on logout:

```ts
const logout = useCallback(() => {
  pb.authStore.clear();
  queryClient.clear(); // Clear all cache
}, [queryClient]);
```
