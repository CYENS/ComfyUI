# Installation Guide

The project is made up of three independent components. You can set each one up on its own, or run all three together.

| Component | Repository | Purpose |
|-----------|-----------|---------|
| **ComfyUI** | `git@github.com:CYENS/ComfyUI.git` | AI engine — runs the actual image/audio/video generation |
| **Backend** | `git@github.com:CYENS/comfyui-backend.git` | API server — manages users, workflows, jobs, and assets |
| **Frontend** | `git@github.com:cchadj/loomaxr-api-platform-frontend.git` | Web app — the interface you use in the browser |

> All repositories are private. Make sure your GitHub SSH key is configured before cloning.
> [GitHub docs: generating an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

---

## Prerequisites

Install these once, before anything else.

### uv (Python package manager)

**Mac / Linux:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows:**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Confirm it works:
```bash
uv --version
```

### Node.js

Download from [nodejs.org](https://nodejs.org/) — version 18 or higher.

### Python

Download from [python.org](https://www.python.org/downloads/) — version 3.13 or higher.

> uv automatically downloads the correct Python version for the backend, so you only need Python installed for ComfyUI itself.

### NVIDIA GPU (for ComfyUI)

ComfyUI requires a CUDA-capable NVIDIA GPU to run AI workflows. CPU-only mode is possible but very slow.

---

## ComfyUI

The AI engine. Runs on port **8188**.

```bash
git clone git@github.com:CYENS/ComfyUI.git
cd ComfyUI
pip install -r requirements.txt
python main.py --listen 0.0.0.0 --port 8188
```

> AI model files (`.safetensors`, `.gguf`, etc.) must be present in the `models/` folder for workflows to run. These are large files not included in the repo — ask the project admin for access.

---

## Backend

The API server. Runs on port **8000**.

```bash
git clone git@github.com:CYENS/comfyui-backend.git
cd comfyui-backend
uv sync
cp .env.example .env
```

Open `.env` and update these for local development:

| Key | What to set |
|-----|-------------|
| `COMFY_BASE_URL` | `http://127.0.0.1:8188` (where ComfyUI is running) |
| `COMFY_MODELS_DIR` | Absolute path to your ComfyUI `models/` folder, e.g. `/home/yourname/ComfyUI/models` |
| `WORKER_LOG_FILE` | Any writable path, e.g. `./logs/worker.log` |
| `DATABASE_URL` | Leave as `sqlite:///./backend.db` for local dev |

Seed the database with sample users and workflows:

```bash
uv run python -m app.seed
```

This creates the following accounts:

| Username | Password | Role |
|----------|----------|------|
| `admin` | `admin123` | Full access |
| `workflow_creator` | `workflow123` | Create & edit workflows |
| `job_creator` | `job123` | Run workflows |
| `viewer` | `viewer123` | View approved outputs |
| `moderator` | `moderator123` | Approve/reject outputs |

Start the API server:

```bash
uv run uvicorn app.main:app --reload --port 8000
```

Start the background worker (separate terminal — it submits jobs to ComfyUI):

```bash
uv run python -m app.worker
```

---

## Frontend

The web app. Runs on port **3000**.

```bash
git clone git@github.com:cchadj/loomaxr-api-platform-frontend.git
cd loomaxr-api-platform-frontend
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) and log in with any account from the Backend section above.

---

## Running everything together

You need **four terminals**:

| Terminal | Directory | Command |
|----------|-----------|---------|
| 1 | `ComfyUI/` | `python main.py --listen 0.0.0.0 --port 8188` |
| 2 | `comfyui-backend/` | `uv run uvicorn app.main:app --reload --port 8000` |
| 3 | `comfyui-backend/` | `uv run python -m app.worker` |
| 4 | `loomaxr-api-platform-frontend/` | `npm run dev` |

Then open [http://localhost:3000](http://localhost:3000).

---

## Troubleshooting

**Port already in use** — stop the conflicting process or change the port in the command and `.env`.

**Backend database error on startup** — drop and recreate the database:
```bash
rm -f backend.db
uv run python -m app.seed
```

**Workflow fails with "missing model"** — the required model file is not in `COMFY_MODELS_DIR`. Check the workflow's Requirements tab in the UI for download links, or ask the admin to trigger a download.

**`uv run pip install` fails with permission error** — use `uv pip install` (without `run`).
