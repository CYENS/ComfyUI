# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Summary

This repository is a thin application layer around ComfyUI, not a fork that should heavily modify ComfyUI itself.

- **ComfyUI core** at the repo root remains the execution engine for node graphs.
- **`backend/`** is the actual product logic: authentication, workflow storage/versioning, job submission, asset storage, moderation, and model requirement management.
- **`backend/` is its own git repository/submodule**. When making commits for backend work, run git commands inside `backend/` and do not mix root-repo changes into backend commits.
- **`backend/app/routers/ui.py`** is a deliberately simple test frontend served by the backend. It is useful as a behavior reference for a real frontend, but it is not the intended final UX.
- **`apps/web/`** is legacy and should be ignored.

Conceptually, the platform turns raw ComfyUI workflow JSON into a role-based, multi-user workflow service:

1. A **workflow creator** exports a workflow from ComfyUI and pastes in:
   - the **API JSON** used for execution
   - optionally the **UI JSON** used to infer model download requirements
2. The backend parses the API JSON and identifies candidate node inputs that could be exposed to end users.
3. The workflow creator chooses which inputs should become the public contract of the workflow, and defines metadata such as label, type, default value, and whether each input is required.
4. The backend stores:
   - the raw `prompt_json` graph
   - an `inputs_schema_json` describing the exposed inputs
   - workflow/version metadata and any inferred model requirements
5. A **job creator** runs the workflow by supplying only those exposed inputs, not the full ComfyUI graph.
6. The worker loop takes queued jobs, injects submitted values into the stored graph using the configured input mappings, sends the final prompt to ComfyUI, and records outputs as assets.
7. A **moderator** reviews generated assets and decides which ones are approved.
8. **Viewers** and other non-privileged users should mainly consume approved assets, while admins and moderators have broader visibility.

This is the key product abstraction: users do not directly operate ComfyUI graphs. They operate saved, versioned workflow templates whose exposed inputs are curated by workflow creators.

## End-to-End Mental Model

Think about the system in four layers:

- **Execution layer**: ComfyUI runs prompts and generates files.
- **Control plane**: FastAPI stores workflows, users, jobs, requirements, and moderation state.
- **Worker layer**: a polling worker bridges database jobs to ComfyUI HTTP calls.
- **Presentation layer**: the sample `/ui` pages demonstrate how a real frontend can authenticate, build workflows, launch jobs, browse assets, and moderate outputs.

The backend is therefore both:

- an API gateway in front of ComfyUI
- a persistence/authorization layer that gives ComfyUI multi-user product behavior it does not have by itself

## Workflow Lifecycle

### 1. Workflow authoring

A workflow creator uses the builder flow to transform a raw ComfyUI export into an application workflow:

- `POST /api/workflows/parse` inspects the API JSON and returns `candidate_inputs`.
- The creator chooses which candidates to expose.
- Each exposed input becomes an item in `inputs_schema_json` with:
  - `id`
  - `label`
  - `type`
  - `required`
  - `default`
  - `mapping: [{node_id, path}]`

Those mappings are the critical link between the user-facing form and the internal ComfyUI graph. They tell the worker exactly where each submitted value should be inserted inside `prompt_json`.

### 2. Versioning

Workflows are versioned. A workflow has stable identity and metadata, while versions hold the actual executable graph and input schema. Updating `prompt_json` or `inputs_schema_json` creates a new workflow version rather than mutating execution history in place.

### 3. Model requirements

When a workflow is saved, the backend tries to extract model requirements from `ui_json` if available, or falls back to `prompt_json`.

Each requirement can carry:

- the model name
- the destination folder/type
- an optional download URL
- approval metadata

This matters because a workflow is only truly runnable when its required models are present in ComfyUI's models directory.

## Job Lifecycle

### 1. Submission

A job creator submits a job against a workflow and passes `params` keyed by exposed input IDs.

For image inputs, the frontend first uploads the file through `POST /api/jobs/upload-image`, then uses the returned server-side filename in the job payload. The sample UI in `backend/app/routers/ui.py` demonstrates this exact pattern.

### 2. Validation

Before accepting or executing the job, the backend can reject it if required models are missing. This prevents queueing jobs that ComfyUI cannot successfully run.

### 3. Execution

The worker loop polls the database for pending jobs, reconstructs the effective ComfyUI prompt by merging:

- the stored workflow `prompt_json`
- the submitted job parameters
- defaults from `inputs_schema_json` when no override is provided

It then submits the finished graph to ComfyUI over HTTP and tracks state transitions such as queued, submitted, running, generated, failed, or cancelled.

### 4. Output persistence

Generated files are written into backend-managed storage and represented as `Asset` records linked back to the job and workflow.

## Asset Moderation Model

Generated assets are not treated as automatically public content.

- The worker creates `Asset` rows when jobs complete.
- Moderators review assets through `POST /api/assets/{id}/review`.
- Review history is stored, and the latest state is also denormalized into a current-status table.
- Approved assets become generally visible; non-approved assets remain restricted to owners and elevated roles.

There is also a separate `is_public` flag handled by admins for public download endpoints. In practice, approval status and public visibility are related but not identical concepts.

## Model Download Approval Model

Model acquisition is intentionally gated because downloading arbitrary remote files is a privileged action.

The intended flow is:

1. Workflow creator proposes a model URL on a workflow requirement.
2. Moderator approves or rejects that URL.
3. Admin triggers the actual download.
4. The backend stores the model in the shared ComfyUI models directory.

Only certain file types from HTTPS HuggingFace/Civitai URLs are allowed. This keeps workflow creation flexible without making model downloads completely uncontrolled.

## Roles and Product Intent

The roles are not cosmetic; they define the core collaboration model:

- **WORKFLOW_CREATOR** defines reusable workflow templates and their exposed inputs.
- **JOB_CREATOR** consumes those templates to run jobs.
- **MODERATOR** governs safety/quality of both generated assets and model URLs.
- **ADMIN** has operational control, including downloads and global visibility controls.
- **VIEWER** is the read-oriented role for consuming approved outputs.

So this is not just "ComfyUI with auth." It is a small workflow marketplace / execution platform with distinct authoring, execution, and moderation responsibilities.

## What `backend/app/routers/ui.py` Shows

The sample UI is useful because it reveals the intended frontend behavior very directly:

- `/ui/auth` shows JWT login, refresh, logout, and `me` flows.
- `/ui/workflows` shows workflow browsing, detail loading, duplication, deletion, version creation, and inline job execution from exposed inputs.
- `/ui/jobs` lists submitted jobs.
- `/ui/assets` shows asset browsing, moderator approval toggles, admin public-visibility toggles, and authenticated downloads.
- `/ui/admin` shows the approval queue for model URLs and per-workflow model availability/download actions.

It also includes a shared JS helper that demonstrates the expected token handling model:

- store access and refresh tokens client-side
- attach bearer auth on API calls
- on `401`, perform a single in-flight refresh and retry

That file should be treated as a reference implementation of backend usage patterns, not as a production frontend architecture.

## Important Engineering Constraints

- Keep changes to ComfyUI core minimal; most product work belongs in `backend/`.
- Use `uv` for backend dependency and runtime commands.
- For backend Python commands, prefer `uv run ...`. For module execution, prefer `uv run -m module_name` over calling `python` directly.
- The backend schema is created directly from SQLAlchemy metadata on startup; there are no migrations.
- SQLite is the default dev database, but the schema is intended to remain Postgres-compatible.
- The frontend should talk only to the backend API, not directly to ComfyUI.
- `apps/web/` is legacy and should not influence new work.

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
