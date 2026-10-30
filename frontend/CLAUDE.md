# Frontend rules (in addition to /AGENTS.md and /CLAUDE.md)

## Structure
- `pages/` only compose features; no business logic or fetching there.
- Feature code lives in `features/<name>/components/` and `features/<name>/hooks/`.
- Move a component to `components/ui/` only when 2+ features use it.
- `lib/api-client.ts` is the only place that calls `fetch`. It switches to `lib/mocks/` when `VITE_USE_MOCKS=true`.
- `types/api.ts` mirrors `docs/API_CONTRACT.md` exactly (snake_case).
- Every UI string exists in every i18n file (`en.json`, `hi.json`, `ta.json`).

## UX requirements (scoring features — never drop them)
1. **Mic button** always visible on chat, using the Web Speech API with the current language. If unsupported, hide it and show a hint — never crash.
2. **Citation chips** open `/static/pdfs/<file>#page=<n>` in a new tab.
3. A **Listen** button on each answer.
4. A **confidence badge** (`high` / `medium` / `low` / `none`); `none` gets the "not found in official documents" style.
5. A **language switcher** persisted in `localStorage`, wrapped in try/catch.
6. The **eligibility form** is generated from `GET /api/v1/eligibility/questions`.
7. A toggleable **pipeline debug panel** showing intent, sources with scores, and timings.
8. Works at **360px** width, tap targets **≥ 44px**, **PWA installable**.

## Style
- Tailwind only.
- Base font size 16–18px.
- Loading, error, and retry states everywhere — never a blank screen.
