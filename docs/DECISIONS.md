# Decisions

Architecture decision records (ADRs), tuned values, and the dependency log.

**Rules**

- Every structural change approved under AGENTS.md R1 gets an ADR here, in the same change that updates AGENTS.md §3.
- Every new package gets a row in the dependency log (AGENTS.md R6) **before** it is installed.
- Every tuned value gets its final value and evidence (eval run, date) in the tuned values table.
- ADRs are never deleted. To change one, add a new ADR that supersedes it and mark the old one `Superseded by ADR-xxx`.

Format: **Context → Decision → Consequences**. Status is `Accepted`, `Proposed`, or `Superseded`.

---

## ADR-001 — Modular monolith with ports and adapters

- **Status:** Accepted
- **Context:** Four people, about 24 hours, laptops with 8 GB RAM. We need clear boundaries so four people (and their AI agents) can work in parallel without conflicts, and we need to swap LLM, vector store and storage implementations easily.
- **Decision:** One FastAPI process with strict layers (api → services → core → adapters). Adapters implement `Protocol` interfaces in `adapters/*/base.py`. Only `container.py` constructs concrete adapters.
- **Consequences:** One command to run, one log stream, one place to warm models. Boundaries are enforced by import rules and `/check-structure`, not by the network. Any adapter can later be moved behind a network call (SYSTEM_DESIGN §17). The cost is discipline: agents must follow R3.

## ADR-002 — Deterministic eligibility; the LLM never decides

- **Status:** Accepted
- **Context:** Eligibility mistakes harm users. LLMs are inconsistent on numeric thresholds and exclusions and can't explain their decisions reliably.
- **Decision:** Eligibility is computed only by `core/eligibility/engine.py` from `data/schemes/eligibility_rules.json`, with three-valued logic (`eligible` / `not_eligible` / `need_more_info`) and a reason for each condition. The LLM may route a question to the eligibility flow, but never produces a verdict.
- **Consequences:** Same input → same output; 100% unit-testable; reasons are explainable. Rules must be written by hand from the official documents (CS work). Schemes with fuzzy criteria get `need_more_info` or a note pointing to the official office rather than a guess.

## ADR-003 — No LangChain or LlamaIndex in the request path

- **Status:** Accepted
- **Context:** Our differentiator is strict grounding: a hard score gate, citation mapping and numeric grounding, with per-stage timings in a debug panel. Frameworks hide these steps and add many dependencies.
- **Decision:** Write the pipeline directly (about five small modules in `core/rag` and `core/understanding`). Frameworks may be used in throwaway experiments, never imported from `backend/app` or `backend/ingestion`.
- **Consequences:** Full control and easy debugging; smaller install on Windows and 8 GB laptops. We write a little more code ourselves (prompt loading, JSON repair), which is cheap with Claude Code.

## ADR-004 — English-pivot retrieval, direct target-language answers

- **Status:** Accepted
- **Context:** Most official PDFs are in English. Users ask in Hindi, Tamil, Hinglish or English. Translating answers after generation adds latency and loses fluency.
- **Decision:** The query understanding step rewrites the user's question into a standalone English query (`query_en`) for retrieval. The generator receives English sources and writes the answer directly in the user's language. Scheme cards add localised names and aliases so non-English queries still match.
- **Consequences:** Reliable retrieval and one fewer call per turn. Answer quality in a language depends on the LLM's ability in that language, which is mitigated by per-language routing (`LLM_PROVIDER_BY_LANG`) and optional Bhashini (ADR-005). Numeric grounding must handle localised numerals.

## ADR-005 — Bhashini optional, behind an adapter, off by default

- **Status:** Accepted
- **Context:** Bhashini offers strong Indian-language translation and speech, but needs registration, adds latency and is an external dependency during a live demo.
- **Decision:** `adapters/translation/bhashini_translator.py` implements the translation `Protocol`; the default is `noop_translator` (`TRANSLATION_PROVIDER=none`). Building it is a stretch task (TR-01).
- **Consequences:** The demo doesn't depend on Bhashini being available. Turning it on is a config change. It is the natural path to more languages (SYSTEM_DESIGN §17).

## ADR-006 — Browser Web Speech API for voice

- **Status:** Accepted
- **Context:** Voice is essential for low-literacy users. Server-side speech models are too heavy for an 8 GB CPU laptop and paid APIs violate the ₹0 constraint.
- **Decision:** Use the browser's `SpeechRecognition` and `speechSynthesis` with BCP-47 tags from the language registry (`hi-IN`, `ta-IN`, `en-IN`). Feature-detect: if unsupported, hide the mic and show a hint.
- **Consequences:** Zero cost and zero backend load. Works best in Chrome/Edge; Firefox lacks recognition; Chrome sends audio to Google (disclosed, SYSTEM_DESIGN §13). Voice quality varies by device.

## ADR-007 — ChromaDB + `intfloat/multilingual-e5-small`

- **Status:** Accepted
- **Context:** We need local, free, CPU-friendly vector search with metadata filters (by `scheme_id`) and multilingual embeddings.
- **Decision:** Persistent ChromaDB in `data/vectorstore/` (cosine space). Embeddings with `multilingual-e5-small` (384 dimensions, ~120 MB), using the required `query:` and `passage:` prefixes.
- **Consequences:** No server to run; ~900 chunks fit easily. e5 scores cluster high, so `RAG_MIN_SCORE` needs tuning on the golden set (RAG-08). `sentence-transformers` pulls in PyTorch — install the CPU wheel to save disk and RAM (see dependency log).

## ADR-008 — Groq primary, Gemini secondary, mock for tests; model names only in `.env`

- **Status:** Accepted
- **Context:** We need fast, free LLM inference with a backup, and deterministic tests. Model names on free tiers change often.
- **Decision:** `groq_llm.py` primary (fast inference), `gemini_llm.py` secondary, `fallback_llm.py` composes them and applies `LLM_PROVIDER_BY_LANG`. `mock_llm.py` is deterministic and used by all tests. Model names live only in `.env` (`LLM_MODEL`, `LLM_FAST_MODEL`, `GEMINI_MODEL`).
- **Consequences:** Swapping a model is a config change. Rate limits on one provider fall over to the other. Tests never hit real APIs. Someone must check the current model names on the provider consoles before the demo.

## ADR-009 — Schemes and rules as versioned JSON

- **Status:** Accepted
- **Context:** Scheme metadata, documents checklists, application steps and eligibility rules must be reviewed by humans against the official documents and changed without code changes.
- **Decision:** Store them as JSON in `data/schemes/` (`schemes.json`, `eligibility_rules.json`, `profile_fields.json`), formats defined in `docs/DATA_SCHEMAS.md`, validated by loaders at startup.
- **Consequences:** Diffable in Git, editable by the non-coding team member, testable. Invalid JSON fails fast at startup. No admin UI is needed.

## ADR-010 — No profile persistence

- **Status:** Accepted
- **Context:** Eligibility profiles contain sensitive data (income, caste category, disability, land). We have no accounts and no need to keep them.
- **Decision:** The profile exists only in the request and in the browser's form state. It is never written to SQLite, logs, caches or analytics.
- **Consequences:** Strong privacy story and less compliance risk. Users re-enter details per session (the form is short). `/check-structure` and code review check this.

---

## Tuned values

Start values come from `SYSTEM_DESIGN.md`. RAG-08 fills in **Final** and **Evidence** (eval run + date + metric).

| Setting | Env key(s) | Start | Final | Evidence |
|---|---|---|---|---|
| Minimum retrieval score | `RAG_MIN_SCORE` | 0.78 | | |
| Chunk size / overlap (tokens) | (ingestion config) | 450 / 80 | | |
| Retrieval top_k (filtered + unfiltered) | `RAG_TOP_K_FILTERED`, `RAG_TOP_K_UNFILTERED` | 6 + 4 | | |
| Context chunks | `RAG_CONTEXT_CHUNKS` | 4 | | |

---

## Dependency log

One row per package (AGENTS.md R6). Planned packages are pre-approved for the listed task; anything else needs a new row first.

| Package | Side | Reason (one line) | Added in task | Free? |
|---|---|---|---|---|
| fastapi | backend | Async web framework with Pydantic integration | BE-01 | Yes |
| uvicorn[standard] | backend | ASGI server for FastAPI | BE-01 | Yes |
| pydantic (v2) | backend | Schemas and LLM output validation | BE-01 | Yes |
| pydantic-settings | backend | Config from env (R5) | BE-01 | Yes |
| pytest, pytest-asyncio | backend dev | Unit and integration tests | BE-01 | Yes |
| ruff | backend dev | Lint and format | BE-01 | Yes |
| groq | backend | Primary LLM SDK | LLM-01 | Yes (free tier) |
| google-genai | backend | Secondary LLM SDK (Gemini) | LLM-01 | Yes (free tier) |
| pymupdf | backend | Page-wise PDF text extraction | ING-01 | Yes |
| sentence-transformers | backend | e5 embeddings on CPU (install CPU-only torch first to save ~2 GB) | ING-03 | Yes |
| chromadb | backend | Local persistent vector store | ING-03 | Yes |
| httpx | backend | HTTP client for Bhashini adapter and smoke test | TR-01 / QA-03 | Yes |
| react, react-dom (18) | frontend | UI library | FE-01 | Yes |
| vite, typescript | frontend | Build tool and type checking | FE-01 | Yes |
| tailwindcss | frontend | Styling (only styling approach allowed) | FE-01 | Yes |
| eslint (+ TS/React plugins) | frontend | Lint | FE-01 | Yes |
| @tanstack/react-query | frontend | Server state, retries | FE-02 | Yes |
| react-router-dom | frontend | Routing between pages | FE-01 | Yes |
| i18next, react-i18next | frontend | Static UI translations | FE-04 | Yes |
| vite-plugin-pwa | frontend | Installable PWA, service worker | FE-08 | Yes |
| python-telegram-bot | bot | Telegram client | BOT-01 | Yes |
