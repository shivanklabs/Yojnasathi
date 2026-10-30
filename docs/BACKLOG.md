# Backlog

How to use this file:

- Pick a task whose deps are all ticked `[x]`, run `/implement <TASK-ID>`, and wait for the plan.
- **One task per commit**: `<TASK-ID>: <summary>` (AGENTS.md R7).
- Tick `[x]` only when the AGENTS.md §9 definition of done is met.
- Estimates assume one person working with Claude Code. "Deps" are task IDs that must be ticked first.
- Owners: **AI-1** (RAG), **AI-2** (eligibility + language + voice), **FSD** (full-stack), **CS** (data, bot, QA, demo) — see AGENTS.md §5.

---

## Milestones

| Milestone | Target | Goal | Exit check |
|---|---|---|---|
| **M0 — Setup** | Hour 0–1 | Repo, docs, env, keys, tools installed on every laptop | Everyone can run `claude`, Python 3.11, Node 20; `.env` filled; `/check-structure` OK |
| **M1 — Walking skeleton** | ~Hour 6 | End-to-end path works with mocks: PWA → `/api/v1/chat` → mock LLM → response; scheme list; health | Frontend shows a mocked and a real (mock-LLM) answer; `pytest -q` green |
| **M2 — Real RAG** | ~Hour 12 | Real RAG on **10 schemes** with citations, eligibility engine live, Hindi answers | Hindi question gets a cited answer; eligibility form returns reasons |
| **M3 — Complete** | ~Hour 18 | Voice, **20 schemes**, guardrails, debug panel, eval **hit@4 ≥ 85%** | All six demo-critical features (SYSTEM_DESIGN §18) work; eval report saved |
| **M4 — Freeze + demo** | Last 4 hours | Bug fixes only, demo script, backup video, slides | Smoke test passes; demo rehearsed twice |

---

## FSD — Full-stack

- [ ] **BE-01 — Backend scaffold** · M1 · Est 45m · Deps: —
  - `backend/pyproject.toml` with `[dev]` extras; `app/main.py` app factory with lifespan; `app/config.py` (pydantic-settings, reads root `.env`); `GET /api/v1/health` returning static `ok`; `utils/logging.py`.
  - AC: every backend command in AGENTS.md §7 that applies so far works exactly as written (`pip install -e ".[dev]"`, `uvicorn app.main:app --reload --port 8000`, `pytest -q`, `ruff check .`).
- [ ] **BE-02 — Middleware + error envelope** · M1 · Est 45m · Deps: BE-01
  - `api/middleware.py` (request_id, timing, rate limit), `api/errors.py` (domain errors → envelope), `utils/ids.py`, `utils/timing.py`.
  - AC: every response has `X-Request-ID`; a raised domain error returns the contract's error envelope; unit tests cover the mapping.
- [ ] **BE-03 — Schemas + domain** · M1 · Est 1h · Deps: BE-01
  - `app/schemas/` mirroring `docs/API_CONTRACT.md` exactly; `app/domain/` models and domain errors.
  - AC: field names and types match the contract; `domain/` imports nothing from the app; tests round-trip contract examples.
- [ ] **BE-04 — Container wiring** · M1 · Est 45m · Deps: BE-03, LLM-01
  - `app/container.py` selects adapters from config; services receive dependencies; FastAPI `deps.py` exposes them.
  - AC: `LLM_PROVIDER=mock` runs without any API key; no other module imports concrete adapters.
- [ ] **BE-05 — SQLite store** · M1 · Est 1h · Deps: BE-03
  - `adapters/storage/base.py` + `sqlite_store.py`: conversations, messages, citations, feedback, demo answer cache; schema created on startup.
  - AC: unit tests on a temp DB; blocking calls wrapped for async; no profile table or column exists.
- [ ] **BE-06 — Scheme service + routes** · M1 · Est 1h · Deps: BE-03, DATA-01
  - `services/scheme_service.py`, `api/v1/schemes.py`: list (filter by category, language) and detail.
  - AC: responses match the contract; unknown `scheme_id` → 404 envelope; localised names returned for `language=hi|ta`.
- [ ] **BE-07 — Chat + feedback routes** · M2 · Est 45m · Deps: BE-05, RAG-07
  - `api/v1/chat.py`, `api/v1/feedback.py`, `services/feedback_service.py`; routes call services only.
  - AC: `POST /chat` and `POST /feedback` match the contract; integration test with mock LLM passes.
- [ ] **BE-08 — Static PDFs + real health** · M2 · Est 45m · Deps: BE-06, RAG-03
  - Mount `/static/pdfs` → `data/raw/pdfs`; `/health` reports index chunk count, provider, models warm, `degraded` if index empty.
  - AC: a citation URL opens the right PDF page in the browser; health shows `degraded` with an empty vectorstore.
- [ ] **FE-01 — Frontend scaffold** · M1 · Est 45m · Deps: —
  - Vite + React 18 + TS strict + Tailwind + React Router + ESLint; Vite proxy `/api` and `/static` → `:8000`; `app/` providers and routes; empty pages.
  - AC: `npm install`, `npm run dev`, `npm run lint`, `npx tsc --noEmit` work exactly as in AGENTS.md §7.
- [ ] **FE-02 — Types + mocks + api-client** · M1 · Est 1h · Deps: FE-01
  - `types/api.ts` mirroring the contract; `lib/mocks/` for every endpoint; `lib/api-client.ts` (only `fetch`, switches to mocks on `VITE_USE_MOCKS=true`); `lib/config.ts`.
  - AC: with `VITE_USE_MOCKS=true` every page can load data with no backend; tsc passes.
- [ ] **FE-03 — Chat UI** · M1 · Est 2h · Deps: FE-02
  - `features/chat`: ChatWindow, MessageBubble, CitationChip, Composer, SuggestedQuestions, `useChat`; confidence badge; loading/error/retry states.
  - AC: citation chip opens `/static/pdfs/<file>#page=<n>` in a new tab; `none` confidence uses the not-found style; works at 360px.
- [ ] **FE-04 — Header + language switcher + i18n** · M1 · Est 1h · Deps: FE-01
  - `i18n/index.ts`, `en.json`, `hi.json`, `ta.json`; `components/ui/LanguageSwitcher` persisted in `localStorage` (try/catch).
  - AC: switching language changes all UI text; every key exists in all three files; reload keeps the language.
- [ ] **FE-05 — Eligibility page (dynamic form)** · M2 · Est 2h · Deps: FE-02, ELG-04
  - `features/eligibility`: ProfileForm generated from `GET /eligibility/questions`, ResultCard, ReasonList, `useEligibility`.
  - AC: adding a field in `profile_fields.json` shows up with no frontend change; results sorted as in SYSTEM_DESIGN §7; need-more-info highlights missing fields.
- [ ] **FE-06 — Schemes list/detail** · M2 · Est 1.5h · Deps: FE-02
  - `features/schemes`: SchemeCard, DocumentChecklist, ApplySteps; SchemesPage and SchemeDetailPage.
  - AC: list filters by category; detail shows checklist, steps, official link and source PDFs; localised names.
- [ ] **FE-07 — Debug panel** · M3 · Est 1h · Deps: FE-03
  - `features/chat/components/PipelineDebug`: toggle; intent, language, `query_en`, sources with scores (used vs. not), guardrail actions, timings.
  - AC: hidden by default; toggle state remembered; renders nothing when `debug` is absent.
- [ ] **FE-08 — PWA + mobile polish** · M3 · Est 1.5h · Deps: FE-03
  - vite-plugin-pwa manifest and icons in `public/`; 360px audit; tap targets ≥ 44px; aria labels; 16–18px base font.
  - AC: Chrome shows "Install app"; Lighthouse PWA installable; no horizontal scroll at 360px.

## AI-1 — RAG

- [ ] **LLM-01 — LLM adapters + fallback + mock** · M1 · Est 1.5h · Deps: BE-01
  - `adapters/llm/base.py` Protocol; `groq_llm.py`, `gemini_llm.py`, deterministic `mock_llm.py`, `fallback_llm.py` (chain + `LLM_PROVIDER_BY_LANG`); timeouts from config; domain errors.
  - AC: unit tests for fallback order and per-language routing using fakes; mock output identical across runs.
- [ ] **ING-01 — Extract + clean** · M2 · Est 1.5h · Deps: DATA-02
  - `ingestion/extract.py` (PyMuPDF page records with font info), `clean.py` (headers/footers, hyphenation, NFC).
  - AC: unit tests on a fixture page; repeated headers removed; page numbers are 1-based and correct.
- [ ] **ING-02 — Chunker + scheme cards** · M2 · Est 1.5h · Deps: ING-01
  - `ingestion/chunk.py`: section-aware ~450/80 tokens, never crosses pages, heading prefix; scheme_card chunks from `schemes.json`.
  - AC: tests prove no chunk spans two pages and every chunk has `scheme_id`, `source_file`, `page`, `chunk_id`.
- [ ] **ING-03 — Embedder + Chroma + ingestion CLI** · M2 · Est 1.5h · Deps: ING-02
  - `adapters/embeddings/` (e5 with prefixes), `adapters/vectorstore/` (Chroma), `ingestion/index.py`, `manifest.py`, `run.py` with `--rebuild`.
  - AC: `python -m ingestion.run` works as written; a second run is a no-op; ~900 chunks for 20 schemes.
- [ ] **RAG-01 — Query understanding** · M2 · Est 1.5h · Deps: LLM-01, LANG-01
  - `core/understanding/query_understanding.py`, `prompts/understand.md`; client language override; no-LLM fallback.
  - AC: unit tests with mock LLM for hi/ta/en/Hinglish and a follow-up; fallback works when the LLM raises.
- [ ] **RAG-02 — Retriever** · M2 · Est 1h · Deps: ING-03
  - `core/rag/retriever.py`: `query:` prefix, filtered 6 + unfiltered 4, merge, dedupe, score = 1 − distance.
  - AC: tests with a temp Chroma; wrong hint still returns the right scheme via unfiltered search.
- [ ] **RAG-03 — Context + generator + prompts** · M2 · Est 2h · Deps: RAG-02, LLM-01
  - `context_builder.py` (≤ 4 chunks / 2500 tokens, `[S1]` labels), `generator.py`, `prompts/answer.md`; threshold gate → not-found.
  - AC: below `RAG_MIN_SCORE` no LLM call is made (asserted in tests); prompt placeholders exactly `{{context}}`, `{{question}}`, `{{language_name}}`, `{{history}}`.
- [ ] **RAG-04 — Guardrails** · M3 · Est 2h · Deps: RAG-03
  - `core/rag/guardrails.py` + `prompts/repair.md`: JSON repair once → snippet fallback; citation filter; numeric grounding; confidence cap; URL mapping.
  - AC: pure unit tests for each rule, including a fake citation, an invented ₹ amount and Devanagari numerals.
- [ ] **RAG-05 — Answer cache** · M3 · Est 45m · Deps: RAG-03
  - `utils/cache.py` LRU with TTL; key = normalised `query_en` + language + hints; skipped for follow-ups; demo cache persisted when `DEMO_MODE=true`.
  - AC: a repeated question returns in < 100 ms; cache hit shown in debug.
- [ ] **RAG-06 — Golden set + eval script** · M3 · Est 1.5h · Deps: RAG-02
  - `tests/eval/golden_qa.json` format with QA-01's questions; `scripts/eval_rag.py` computes hit@4, not-found accuracy, citation validity; writes `report.json`.
  - AC: `python scripts/eval_rag.py` works as written and prints a summary table.
- [ ] **RAG-07 — Chat service** · M2 · Est 2h · Deps: RAG-01, RAG-04, BE-05, ELG-03, LANG-02
  - `services/chat_service.py` `handle_turn` with the full pipeline order from `backend/CLAUDE.md`; history; persist; debug timings.
  - AC: integration tests (mock LLM, temp Chroma) for RAG answer, not-found, eligibility routing, greeting, LLM failure fallback.
  - Note: to reach M2 on time, a first version can land before RAG-04 with guardrails stubbed to "citation filter only", then be completed in M3.
- [ ] **RAG-08 — Tuning** · M3 · Est 1.5h · Deps: RAG-06
  - Tune `RAG_MIN_SCORE`, chunk size/overlap, top_k, context size on the golden set.
  - AC: hit@4 ≥ 85%; final values and evidence recorded in `DECISIONS.md` tuned values table.

## AI-2 — Eligibility, language, voice

- [ ] **LANG-01 — Languages registry + script detector** · M1 · Est 1h · Deps: BE-01
  - `core/language/languages.py`: codes, names (native + English), BCP-47 voice tags, scripts; script detector incl. Hinglish heuristic.
  - AC: unit tests for Devanagari, Tamil, Latin English and Hinglish samples.
- [ ] **LANG-02 — Canned replies** · M1 · Est 45m · Deps: LANG-01
  - `core/language/canned.py`: greeting, help, not-found (with portal link slot), out-of-scope, unclear, service-error — in every supported language.
  - AC: test asserts every canned key exists for every language.
- [ ] **ELG-01 — Operators** · M2 · Est 1h · Deps: BE-03
  - `core/eligibility/operators.py`: `eq`, `neq`, `in`, `not_in`, `gt`, `gte`, `lt`, `lte`, `between`, `is_true`, `is_false`, three-valued.
  - AC: 100% branch coverage on operators; missing value → `unknown`.
- [ ] **ELG-02 — Engine + loader** · M2 · Est 2h · Deps: ELG-01, DATA-03
  - `engine.py` (all/any/not, reasons, minimal missing fields, sorting), `loader.py` (validate JSON at startup).
  - AC: for every scheme in `eligibility_rules.json` there are eligible, not-eligible and need-more-info tests; invalid rules fail at load.
- [ ] **ELG-03 — Eligibility service** · M2 · Est 1h · Deps: ELG-02, BE-06
  - `services/eligibility_service.py`: questions list (optionally per scheme), check, attach scheme names in the requested language.
  - AC: profile never passed to storage or logs (test with a spy store).
- [ ] **ELG-04 — Eligibility routes** · M2 · Est 45m · Deps: ELG-03
  - `api/v1/eligibility.py`: `GET /eligibility/questions`, `POST /eligibility/check`.
  - AC: match the contract; invalid profile values → 422 envelope.
- [ ] **VOICE-01 — Speech recognition + MicButton** · M3 · Est 1.5h · Deps: FE-03
  - `features/voice/useSpeechRecognition.ts`, `MicButton.tsx`; language from the switcher; feature detection.
  - AC: Hindi speech fills the composer in Chrome; unsupported browser hides the mic, shows a hint, never crashes.
- [ ] **VOICE-02 — Speech synthesis + Listen** · M3 · Est 1h · Deps: VOICE-01
  - `features/voice/useSpeechSynthesis.ts`; Listen button on each answer; stop/replay; best voice for language.
  - AC: Hindi and Tamil answers are spoken where voices exist; missing voice shows a hint.
- [ ] **TR-01 — (Stretch) Bhashini adapter** · M3+ · Est 2h · Deps: RAG-07
  - `adapters/translation/bhashini_translator.py` behind the translation Protocol; enabled by `TRANSLATION_PROVIDER=bhashini`.
  - AC: off by default; integration test with a fake HTTP server; no effect on the default path.

## CS — Data, bot, QA, demo

- [ ] **DATA-01 — `schemes.json` (10 → 20 schemes)** · M1 (10) / M3 (20) · Est 3h · Deps: —
  - Per `DATA_SCHEMAS.md`: id, names in en/hi/ta, aliases, category, level, summary, benefits, documents, apply steps, official URL, priority.
  - AC: validates against the schema; 10 schemes by M1, 20 by M3; every entry checked against its PDF.
- [ ] **DATA-02 — PDFs + SOURCES.md** · M1 · Est 2h · Deps: —
  - Download official PDFs into `data/raw/pdfs/` named `<scheme_id>__<short-title>.pdf`; `data/raw/SOURCES.md` with URL, ministry, date, sha256.
  - AC: text-based (not scanned) PDFs; every file listed in SOURCES.md.
- [ ] **DATA-03 — Rules + profile fields** · M2 · Est 3h · Deps: DATA-01
  - `eligibility_rules.json` and `profile_fields.json` per `DATA_SCHEMAS.md`; each rule notes the source page.
  - AC: loader validates both files; at least 3 test profiles per scheme written down for AI-2.
- [ ] **QA-01 — Golden questions** · M2 · Est 2h · Deps: DATA-02
  - ~60 questions in en/hi/ta with expected `scheme_id` + page, including ~10 that must be not-found.
  - AC: stored in `backend/tests/eval/golden_qa.json` in the format agreed with AI-1.
- [ ] **QA-02 — Integration tests** · M3 · Est 2h · Deps: BE-07, ELG-04
  - `backend/tests/integration/`: chat (answer, not-found, fallback), eligibility, schemes, feedback, error envelope.
  - AC: runs with `LLM_PROVIDER=mock` and temp dirs; never hits real APIs.
- [ ] **QA-03 — Smoke test** · M3 · Est 1h · Deps: BE-07
  - `backend/scripts/smoke_test.py` against a running server: health, one question per language, eligibility, PDF link.
  - AC: `python scripts/smoke_test.py` prints pass/fail per check and exits non-zero on failure.
- [ ] **BOT-01 — (Stretch) Telegram bot** · M3+ · Est 2h · Deps: BE-07
  - `bots/telegram/bot.py`: HTTP client of `/api/v1/chat`; citations as links; `/lang` command.
  - AC: `python bot.py` works as written; imports nothing from `backend/`.
- [ ] **DEMO-01 — Demo script + backup video + slides** · M4 · Est 3h · Deps: M3
  - Script covering the six demo-critical features; recorded backup video; short slide deck.
  - AC: demo rehearsed twice end to end; video plays offline.

---

## Parallel start

What each person picks up first, so nobody waits on anyone else.

| Person | First tasks | Why it doesn't block |
|---|---|---|
| **AI-1** | LLM-01 → ING-01 | LLM adapters need only the backend scaffold; ING-01 starts as soon as the first PDFs land |
| **AI-2** | LANG-01 → ELG-01 | Pure Python in `core/`; ELG-01 needs only the domain models |
| **FSD** | BE-01 → FE-01 → FE-02 | Scaffold unblocks everyone; the frontend then runs on mocks |
| **CS** | DATA-02 → DATA-01 | PDFs unblock ingestion; `schemes.json` unblocks schemes and rules |
