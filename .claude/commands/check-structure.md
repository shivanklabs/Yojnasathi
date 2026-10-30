---
description: Verify the repo still matches the locked structure and layering rules
---
Audit the repo against AGENTS.md. Report only, change nothing.
1. List directories (ignore .git, node_modules, .venv, __pycache__, dist, caches) and compare with AGENTS.md §3. Report any extra directory.
2. Check every changed/added file matches a row in AGENTS.md §4.
3. Layering: in backend/app/core and backend/app/services, find imports of app.adapters that are not .base; in backend/app/adapters, find imports of app.core or app.services; in backend/app/api, find business logic. Report file:line.
4. Config: find hardcoded keys, model names, URLs, or thresholds outside config.py and .env.example.
5. Contract: compare touched types in backend/app/schemas and frontend/src/types/api.ts with docs/API_CONTRACT.md.
6. Frontend: find fetch( outside lib/api-client.ts, and hardcoded JSX strings.
7. No .py/.ts files in data/ or docs/.
Output a table: Check | Status (OK/VIOLATION) | Details. Then a fix list ordered by severity.
