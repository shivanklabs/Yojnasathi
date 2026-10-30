# Yojana Sahayak

A multilingual government scheme assistant (Sparkathon AI #5). Ask by text or voice in your own language and get:

1. A simple answer grounded **only** in official scheme PDFs, with clickable citations (document + page).
2. A deterministic, rule-based eligibility check with reasons.
3. A documents checklist and the steps to apply.

Stack: React PWA + FastAPI, run locally. Free tiers only, CPU only, 8 GB RAM laptops.

---

## Read the docs in this order

1. **`AGENTS.md`** — rules, locked folder structure, ownership. Everyone reads this first.
2. **`docs/SYSTEM_DESIGN.md`** — how the system works and why.
3. **`docs/API_CONTRACT.md`** — the HTTP API (source of truth for backend and frontend).
4. **`docs/DATA_SCHEMAS.md`** — format of the JSON files in `data/schemes/`.
5. **`docs/BACKLOG.md`** — tasks, milestones, who does what.
6. **`docs/DECISIONS.md`** — architecture decisions, tuned values, dependency log.

Side-specific rules: `backend/CLAUDE.md`, `frontend/CLAUDE.md`.

---

## Prerequisites

- **Python 3.11**
- **Node.js 20** (with npm)
- **Git**
- **VS Code** (recommended editor)
- **Claude Code** (`claude` in the terminal)
- **API keys**: Groq (https://console.groq.com) and Gemini (https://aistudio.google.com) — both free tiers
- **Chrome or Edge** — needed for voice (Web Speech API)

## Setup

```bash
git clone <repo-url> yojana-sahayak
cd yojana-sahayak
cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env
```

Then fill in `.env` (API keys and model names — check current model names on the provider consoles). For frontend variables, create `frontend/.env.local` (see the bottom of `.env.example`).

> No keys yet? Set `LLM_PROVIDER=mock` in `.env` and `VITE_USE_MOCKS=true` in `frontend/.env.local` to work without them.

---

## Working with Claude Code

- Start every task with **`/implement <TASK-ID>`** (e.g. `/implement RAG-03`). Claude reads the rules, checks dependencies and posts a plan.
- **Review the plan before saying "ok"**: check the file paths match AGENTS.md §4 and that it only touches your files.
- Run **`/check-structure`** after a task (Claude does this automatically at the end of `/implement`) — it reports layering, config and structure violations.
- **One task per commit**, message `<TASK-ID>: <summary>`.
- After **`/compact`** (or when the context is summarised), tell Claude to **re-read `AGENTS.md`** before continuing.
- If Claude wants to add a folder or a package, it must stop and ask. Say no unless there's a good reason, and record it in `docs/DECISIONS.md`.

---

## Running locally (4 terminals)

| Terminal | Folder | Commands |
|---|---|---|
| 1 — Backend | `backend/` | `python -m venv .venv` → activate (`.venv\Scripts\activate` on Windows, `source .venv/bin/activate` on macOS/Linux) → `pip install -e ".[dev]"` → `uvicorn app.main:app --reload --port 8000` |
| 2 — Frontend | `frontend/` | `npm install` → `npm run dev` → open http://localhost:5173 |
| 3 — Ingestion / tests | `backend/` (venv active) | `python -m ingestion.run` (build the index once) · `pytest -q` · `ruff check .` · `python scripts/eval_rag.py` |
| 4 — Bot (optional) | `bots/telegram/` | `python bot.py` |

These commands start working as the backlog tasks that create them are completed (BE-01, FE-01, ING-03, RAG-06, BOT-01).

---

## Team start

| Role | Owns | First tasks |
|---|---|---|
| **AI-1** — RAG | ingestion, understanding, RAG core, LLM/embedding/vector adapters, chat service, eval | LLM-01 → ING-01 |
| **AI-2** — Eligibility, language, voice | eligibility engine, language registry, translation adapter, voice UI | LANG-01 → ELG-01 |
| **FSD** — Full-stack | app scaffold, API, schemas, storage, scheme/feedback services, frontend (except voice) | BE-01 → FE-01 → FE-02 |
| **CS** — Data, bot, QA, demo | PDFs, scheme/rule JSON, golden questions, integration + smoke tests, Telegram bot, demo | DATA-02 → DATA-01 |
