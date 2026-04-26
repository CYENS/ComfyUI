# Installation Guide

This project has three parts that work together:

- **ComfyUI** — the AI engine that runs image/audio/video workflows
- **Backend** — a FastAPI server that manages users, workflows, jobs, and assets
- **Frontend** — the Next.js web app you interact with in your browser

---

## What you need before you start

| Tool | Version | How to get it |
|------|---------|---------------|
| Python | 3.13+ | [python.org](https://www.python.org/downloads/) |
| Node.js | 18+ | [nodejs.org](https://nodejs.org/) |
| Git | any | [git-scm.com](https://git-scm.com/) |
| uv | latest | see below |
| NVIDIA GPU | CUDA-capable | Required to run AI workflows |

> **GPU note:** ComfyUI requires a CUDA-capable NVIDIA GPU for image/audio/video generation. CPU-only mode is possible but very slow and not officially supported here.

### Installing uv (Python package manager)

**Mac / Linux:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows:**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

After installing, restart your terminal and confirm it works:
```bash
uv --version
```

> **Note:** uv automatically downloads and manages the correct Python version for the backend — you don't need to install Python 3.14 manually.

### SSH access

All three repositories are private and use SSH. Make sure your GitHub SSH key is set up before cloning:
- [GitHub docs: generating an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

---

## Step 1 — Clone the repositories

The project uses three git repos. Clone them in order:

```bash
# Clone the main repo (includes the backend as a submodule)
git clone --recurse-submodules git@github.com:CYENS/ComfyUI.git
cd ComfyUI

# Clone the frontend into the expected location
git clone git@github.com:cchadj/loomaxr-api-platform-frontend.git frontend
```

> `--recurse-submodules` automatically clones the backend repo into `backend/`. The frontend must be cloned separately.

---

## Step 2 — Install ComfyUI dependencies

From the project root:

```bash
pip install -r requirements.txt
```

This installs PyTorch (with CUDA), numpy, and the other packages ComfyUI needs.

---

## Step 3 — Set up the Backend

```bash
cd backend
uv sync
cp .env.example .env
```

Open `.env` in a text editor and update these values for local development:

| Key | Default in .env.example | What to set it to locally |
|-----|--------------------------|---------------------------|
| `COMFY_BASE_URL` | `http://127.0.0.1:8188` | leave as-is |
| `DATABASE_URL` | `sqlite:///./backend.db` | leave as-is |
| `COMFY_MODELS_DIR` | `/app/models` | absolute path to your `models/` folder, e.g. `/home/yourname/ComfyUI/models` |
| `WORKER_LOG_FILE` | `/home/tom/…/worker.log` | any writable path, e.g. `./logs/worker.log` |

Now seed the database with sample users and workflows:

```bash
uv run python -m app.seed
```

This creates the following test accounts:

| Username | Password | Role |
|----------|----------|------|
| `admin` | `admin123` | Admin (full access) |
| `workflow_creator` | `workflow123` | Can create & edit workflows |
| `job_creator` | `job123` | Can run workflows |
| `viewer` | `viewer123` | Can view approved outputs |
| `moderator` | `moderator123` | Can approve/reject outputs |

Go back to the project root when done:
```bash
cd ..
```

---

## Step 4 — Set up the Frontend

```bash
cd frontend
npm install
cd ..
```

---

## Running everything

You need **four terminal windows** open at the same time.

### Terminal 1 — ComfyUI (the AI engine)
```bash
# from the project root
python main.py --listen 0.0.0.0 --port 8188
```

### Terminal 2 — Backend API
```bash
cd backend
uv run uvicorn app.main:app --reload --port 8000
```

### Terminal 3 — Background Worker
```bash
cd backend
uv run python -m app.worker
```
The worker picks up submitted jobs and sends them to ComfyUI.

### Terminal 4 — Frontend
```bash
cd frontend
npm run dev
```

---

## Open in your browser

Once all four are running, go to:

```
http://localhost:3000
```

Log in with any of the test accounts from Step 3. The `admin` account can do everything.

---

## If something goes wrong

**Port already in use** — another process is using 8188, 8000, or 3000. Stop it, or change the port in the relevant command and `.env`.

**Backend fails to start with a database error** — the database schema is out of date. Delete it and re-seed:
```bash
cd backend
rm -f backend.db
uv run python -m app.seed
```

**Workflows fail with "missing model"** — the AI model files need to be present in `COMFY_MODELS_DIR`. Models are large files (1–20 GB each) and are not included in the repo. Ask the project admin to share them or check the workflow's requirements page for download links.

**`uv run pip install` fails with permission error** — use `uv pip install` instead (without `run`).
