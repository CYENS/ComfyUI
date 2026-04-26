# Installation Guide

This project has three parts that work together:

- **ComfyUI** — the AI engine that runs image/audio/video workflows
- **Backend** — a FastAPI server that manages users, workflows, jobs, and assets
- **Frontend** — the Next.js web app you interact with in your browser

---

## What you need before you start

| Tool | Version | How to get it |
|------|---------|---------------|
| Python | 3.14+ | [python.org](https://www.python.org/downloads/) |
| Node.js | 18+ | [nodejs.org](https://nodejs.org/) |
| Git | any | [git-scm.com](https://git-scm.com/) |
| uv | latest | see below |

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

---

## Step 1 — Clone the repository

```bash
git clone --recurse-submodules https://github.com/CYENS/ComfyUI.git
cd ComfyUI
```

> The `--recurse-submodules` flag is important — it also downloads the backend, which lives in a separate repo.

---

## Step 2 — Set up the Backend

```bash
cd backend
uv sync
cp .env.example .env
```

Open `.env` in a text editor. The defaults work for local development, but check these two lines:

- `COMFY_BASE_URL` — leave as `http://127.0.0.1:8188` (where ComfyUI will run)
- `DATABASE_URL` — leave as `sqlite:///./backend.db` (SQLite, no extra setup needed)

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

## Step 3 — Set up the Frontend

```bash
cd frontend
npm install
cd ..
```

---

## Step 4 — Install ComfyUI dependencies

From the project root:

```bash
pip install -r requirements.txt
```

> If you don't have a GPU or want a lighter install, use `requirements.no-torch.txt` instead.

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

Log in with any of the test accounts from Step 2. The `admin` account can do everything.

---

## If something goes wrong

**Port already in use** — another process is using 8188, 8000, or 3000. Stop it, or change the port in the relevant command and `.env`.

**Backend fails to start with a database error** — the database schema is out of date. Delete it and re-seed:
```bash
cd backend
rm -f backend.db
uv run python -m app.seed
```

**`uv run pip install` fails with permission error** — use `uv pip install` instead (without `run`).
