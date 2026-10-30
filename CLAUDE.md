@AGENTS.md

# Claude Code — session rules (in addition to AGENTS.md)
AGENTS.md section 2 (R1–R7) are hard requirements.

## Every session
1. Before editing, find the target folder in AGENTS.md section 3 and the matching row in section 4. Inside backend/ or frontend/, that folder's CLAUDE.md also applies.
2. Prefer editing existing files. A new file must match a row in section 4, or stop and ask.
3. API change: docs/API_CONTRACT.md → backend/app/schemas/ → frontend/src/types/api.ts → frontend/src/lib/mocks/ together.
4. Never move, rename, or add top-level folders. If the structure seems wrong, say so — don't change it.
5. Finish every task by running the checks in AGENTS.md section 7 for the side you touched, then /check-structure, then report.

## After context compaction
Re-read AGENTS.md sections 2–5 and the current task in docs/BACKLOG.md before continuing. Don't rely on memory of the folder layout.
