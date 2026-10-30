# Backend rules (in addition to /AGENTS.md and /CLAUDE.md)

## Layering self-test
- Inside `app/core/` and `app/services/`, any import from `app.adapters` must come from a `.base` module only (e.g. `from app.adapters.llm.base import LLM`). Concrete adapters are imported only in `app/container.py` and tests.

## Chat pipeline (ChatService.handle_turn)
Preserve this order:

```
understand → route by intent →
    (retrieve → threshold gate → generate → guardrails)
  | eligibility engine
  | canned reply
→ persist → respond
```

- Never skip the threshold gate (`RAG_MIN_SCORE`) or the guardrails.

## External calls
- Every external call (LLM, translation, HTTP) has a timeout read from config and raises a domain error on failure.
- `mock_llm` stays deterministic: same input, same output.

## Prompts and LLM output
- Prompts live only in `app/core/rag/prompts/*.md`, using the placeholders `{{context}}`, `{{question}}`, `{{language_name}}`, `{{history}}`.
- LLM JSON output is validated with Pydantic. On failure: one repair attempt (`repair.md`), then fall back.

## Ingestion
- Idempotent: a sha256 manifest skips unchanged PDFs; `--rebuild` forces a full re-index.
- Every chunk carries `scheme_id`, `source_file`, `page`, `chunk_id`. A chunk without a page number is a bug.

## Tests
- Eligibility and guardrails are pure unit tests (no I/O).
- Integration tests use `LLM_PROVIDER=mock` and a temporary Chroma directory.
- Never hit real APIs in pytest.
