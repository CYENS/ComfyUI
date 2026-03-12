# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

Multi-service project:
- **ComfyUI core** (root) — node-based AI workflow engine, run with `python main.py`. Keep changes here minimal.
- **Backend** (`backend/`) — FastAPI gateway, git submodule, run with `uv`
- **`apps/web/`** — legacy test UI, will be deleted, do not read or modify it
- **Deployment** — `docker-compose.yml` with services: comfyui, api, web, caddy

The backend wraps ComfyUI: workflows and their exposed inputs are stored in a DB, a worker loop polls for pending jobs and submits them to ComfyUI via HTTP, then stores generated assets back in the DB.

## Running locally (3 terminals)

```bash
# Terminal 1 — ComfyUI core
python main.py --listen 0.0.0.0 --port 8188

# Terminal 2 — Backend API
cd backend && uv run uvicorn app.main:app --reload --port 8000

# Terminal 3 — Background worker (polls DB, submits to ComfyUI)
cd backend && uv run python -m app.worker
```

First-time backend setup:
```bash
cd backend
uv sync
cp .env.example .env   # edit as needed
uv run python -m app.seed
```

Docker alternative: `docker-compose up` from root.

## Backend key facts

- **Package manager**: `uv` only — use `uv pip install` (not `uv run pip install`, which fails with permission error)
- **Schema**: `Base.metadata.create_all()` on startup — no Alembic, no migrations. This is intentional for local dev; just drop and recreate the DB when making schema changes.
- **Auth**: JWT + dev mode (`AUTH_DEV_MODE=true` bypasses JWT, uses `X-User-Id`/`X-User-Roles` headers)
- **Roles**: ADMIN, WORKFLOW_CREATOR, JOB_CREATOR, VIEWER, MODERATOR
- **DB**: SQLite default (`backend.db`), Postgres-compatible
- **Flask UI** (`/ui`): served alongside the FastAPI app via WSGIMiddleware, used for testing the backend

Key env vars in `backend/.env`:
- `COMFY_BASE_URL` — ComfyUI URL (default: `http://127.0.0.1:8188`)
- `COMFYUI_MODELS_DIR` — shared models dir (default: `/app/models`)
- `AUTH_DEV_MODE=true` — skip JWT for local dev
- `DATABASE_URL` — SQLite or Postgres connection string

## Roles and access control

| Role | Can do |
|------|--------|
| WORKFLOW_CREATOR | Create/edit workflows, expose inputs, set model download URLs |
| JOB_CREATOR | Create jobs from any active workflow |
| MODERATOR | Review/approve assets, approve/reject model download URLs |
| ADMIN | Everything above + trigger model downloads, access all assets |
| VIEWER | Access moderated (approved) assets |

## Workflow creation flow

1. Workflow creator opens `/ui/builder/` and pastes **API-format** JSON exported from ComfyUI
2. Frontend calls `POST /api/workflows/parse` → returns `candidate_inputs` (one per node input field): `{node_id, path, node_type, value_type, default}`
3. Creator selects which candidates to expose, sets label/required/default for each
4. Frontend saves to `POST /api/workflows` with:
   - `prompt_json` — the raw ComfyUI API JSON
   - `inputs_schema_json` — array of `{id, label, type, required, default, mapping: [{node_id, path}]}`
   - `ui_json` *(optional)* — ComfyUI UI-format export; used only to extract HuggingFace/Civitai model URLs for the model requirements feature
5. On save, model requirements are extracted from `ui_json` (or `prompt_json` fallback) and stored as `WorkflowModelRequirement` records

A workflow is valid for job creation when it exists and all its required models are present in ComfyUI's models directory.

The worker injects job creator input values into the `prompt_json` graph at runtime using the `mapping: [{node_id, path}]` entries in `inputs_schema_json`. Values fall back to the schema `default` if not supplied by the job creator.

## Model requirements / download flow

1. Workflow creator sets a HuggingFace/Civitai HTTPS URL on a `WorkflowModelRequirement` via `PATCH /api/workflows/{id}/requirements/{req_id}`
2. Moderator approves via `POST /api/admin/model-requirements/{req_id}/approve`
3. Admin triggers download via `POST /api/admin/model-requirements/{req_id}/download` (async background task)
4. Models downloaded to `COMFYUI_MODELS_DIR/{folder}/{model_name}` (shared volume with ComfyUI)
5. Job creation returns HTTP 422 if any required model is missing

Only `.safetensors/.sft/.gguf/.pt` extensions from HTTPS HuggingFace/Civitai URLs are accepted.

## Asset moderation flow

1. Worker completes a job → stores output file as `Asset` record
2. Moderator reviews via `POST /api/assets/{id}/review` → creates `AssetValidation` history + updates `AssetValidationCurrent`
3. Approved assets become visible to all authenticated users; unapproved assets only visible to owner and MODERATOR/ADMIN

**Future (low priority)**: Tighten visibility so VIEWER can only see approved assets, JOB_CREATOR sees only their own assets, WORKFLOW_CREATOR sees approved assets from their own workflows.

## Backend commands

```bash
cd backend

uv run uvicorn app.main:app --reload --port 8000   # API server
uv run python -m app.worker                         # background worker
uv run python -m app.seed                           # seed DB (roles + sample workflows)
uv run pytest tests/ -v                             # tests
uv run ruff check --fix . && uv run ruff format .   # lint + format
uv run mypy app/                                    # type check
```

## Router structure

`backend/app/routers/`: auth, workflows, jobs, assets, review, export, admin, ui
All registered in `backend/app/main.py` under `/api` prefix.

When adding a new router:
1. Create `backend/app/routers/my_feature.py`
2. In `main.py`: `from .routers import my_feature` then `app.include_router(my_feature.router, prefix="/api")`
