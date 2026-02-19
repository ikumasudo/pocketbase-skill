# pocketbase-skill

Agent Skills for operating a [PocketBase](https://pocketbase.io/) v0.23+ backend via REST API.

Compatible with the [Agent Skills](https://github.com/vercel-labs/skills) specification.

## Skills

| Skill | Description |
|-------|-------------|
| `pocketbase` | Collection CRUD, record CRUD, superuser/user authentication, backup & restore, and migration file generation |

## Installation

```bash
npx skills add ikumasudo/pocketbase-skill
```

Or install a specific skill:

```bash
npx skills add ikumasudo/pocketbase-skill@pocketbase
```

## Prerequisites

- Python 3 (standard library only, no external packages)
- A running PocketBase instance (v0.23+)
- Environment variables or `.env` file:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PB_URL` | No | `http://127.0.0.1:8090` | PocketBase base URL |
| `PB_SUPERUSER_EMAIL` | Yes* | - | Superuser email address |
| `PB_SUPERUSER_PASSWORD` | Yes* | - | Superuser password |

\*Required for superuser operations.

## License

[MIT](LICENSE.txt)
