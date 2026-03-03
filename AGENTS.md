# AGENTS Guidelines

## Schema Change Rule

Whenever a database schema change is made, regenerate the ER diagram PDF so documentation stays in sync.

Command (from `backend/`):

```bash
docker compose -f docker-compose.eralchemy.yml run --rm eralchemy-pdf
```

Expected output:

- `backend/schema-docs/erd.pdf`
