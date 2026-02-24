# Development Workflow & Deployment

Development environment setup, Vite proxy configuration, environment variable management, and production deployment procedures.

---

## Development Environment

### Vite Proxy Configuration Details

Use `server.proxy` in `vite.config.ts` to avoid CORS issues during development:

```ts
export default defineConfig({
  // ... plugins etc.
  server: {
    proxy: {
      // PocketBase REST API
      "/api": {
        target: "http://127.0.0.1:8090",
        changeOrigin: true,
      },
      // PocketBase internal endpoints (SSE realtime, etc.)
      "/_": {
        target: "http://127.0.0.1:8090",
        changeOrigin: true,
      },
    },
  },
});
```

With proxy configuration:
- Frontend (`http://localhost:5173/api/...`) → forwards to PocketBase (`http://127.0.0.1:8090/api/...`)
- From the browser's perspective, it's same-origin so no CORS headers are needed
- PocketBase's realtime SSE endpoints are also correctly forwarded

> **PB client configuration:** Since the proxy makes API requests same-origin, the PocketBase client needs no URL argument:
> ```ts
> const pb = new PocketBase();
> ```
> This works in both development (via Vite proxy) and production (PocketBase serves the SPA from `pb_public/`). No environment variables are needed.

### Environment Variable Management

No environment variables are needed for the PocketBase URL — the Vite proxy (development) and `pb_public/` serving (production) both provide same-origin access.

If you need environment variables for other purposes, use Vite's `.env` files:

| File | Purpose | Git |
|------|---------|-----|
| `.env` | Default values for local development | Add to `.gitignore` |
| `.env.example` | Template for team sharing | Commit |
| `.env.production` | For production builds | Add to `.gitignore` |

> Only environment variables with the `VITE_` prefix are available in client-side code. Never add the `VITE_` prefix to secret values.

### Running Simultaneously

**Method 1: Separate terminals (recommended)**

```bash
# Terminal 1
cd backend && ./pocketbase serve --http=127.0.0.1:8090

# Terminal 2
cd frontend && npm run dev
```

**Method 2: Within a Claude Code session**

```bash
# Start PocketBase in background
cd backend && nohup ./pocketbase serve --http=127.0.0.1:8090 > pb.log 2>&1 &

# Start Vite dev server
cd frontend && npm run dev
```

**Method 3: concurrently (npm script)**

```bash
npm install -D concurrently
```

Root `package.json`:

```json
{
  "scripts": {
    "dev": "concurrently \"cd backend && ./pocketbase serve\" \"cd frontend && npm run dev\""
  }
}
```

### Path Aliases

`create-tsrouter-app` pre-configures the `@/*` → `src/*` path alias in `tsconfig.json`:

```ts
// The following imports are available
import { pb } from "@/lib/pocketbase";
import type { PostsResponse } from "@/types/pocketbase-types";
```

If not configured, add to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## Deployment

### SPA Build → pb_public Placement

PocketBase automatically serves files in the `pb_public/` directory as static files. Simply placing the frontend build artifacts in `pb_public/` allows PocketBase alone to serve both the SPA and API.

```bash
# 1. Build the frontend
cd frontend && npm run build

# 2. Create pb_public (if it doesn't exist) and deploy
mkdir -p ../backend/pb_public
cp -r dist/* ../backend/pb_public/
```

To consolidate into an npm script (`frontend/package.json`):

```json
{
  "scripts": {
    "deploy": "npm run build && cp -r dist/* ../backend/pb_public/"
  }
}
```

### SPA Routing Support

PocketBase automatically handles SPA fallback for `pb_public/`. Even when the browser reloads on client-side routes like `/dashboard` or `/posts/123`, `index.html` is returned.

No additional configuration is needed.

### Production Deployment (Docker, Binary, Reverse Proxy)

For comprehensive production deployment options including Docker, Docker Compose, executable distribution, and reverse proxy configurations, see `Read references/deployment.md`.
