---
description: Implement one backlog task with plan-first workflow
argument-hint: <TASK-ID e.g. RAG-03>
---
Implement backlog task **$ARGUMENTS**.
1. Re-read AGENTS.md sections 2–5.
2. Find $ARGUMENTS in docs/BACKLOG.md. If any dependency is not ticked [x], stop and tell me which.
3. Read only the docs this task needs.
4. Post a plan and WAIT for my "ok": files to create/modify (full paths, each matching a row in AGENTS.md §4), the owner of each file, approach in ≤8 bullets, tests and the exact command to run them, any new dependency or config key.
5. After approval, implement in small steps; run tests/lint from AGENTS.md §7 after each step.
6. Finish: run /check-structure, tick the task in docs/BACKLOG.md, and report with the AGENTS.md §9 checklist plus the commit message "$ARGUMENTS: <summary>".
