---
name: explain-feature
description: Explain how a specific FastAPI framework feature works in this codebase, citing exact files and line numbers. Use when the user asks "how does X work", "what does Y do", "where is Z implemented", or wants an architectural walkthrough of a feature such as routing, dependency injection, OpenAPI generation, security, or response serialization.
---

# Explain a FastAPI feature

## When to use this skill

The user wants a guided, read-only walkthrough of an existing feature in the FastAPI source. Triggers:

- "How does dependency injection work?"
- "What does `Annotated[T, Query(...)]` do at runtime?"
- "Where is the OpenAPI schema generated?"
- "Walk me through how `Depends(...)` is resolved."
- "What is `_compat` for?"

Do **not** use this skill when the user wants to change behavior, add a feature, or fix a bug — switch to a coding workflow instead.

## Output shape

Reply with this structure:

1. **Summary** — one paragraph, plain English, no jargon dump.
2. **Entry point** — single `path:line` link where execution begins.
3. **Walkthrough** — 3–7 numbered steps. Each step is one sentence + one `path:line` citation.
4. **Key symbols** — short list (class / function name → file path).
5. **Tests that exercise it** — one or two `tests/test_*.py` references the user can run.

Use markdown links: `[fastapi/routing.py:412](fastapi/routing.py#L412)`. Every concrete claim needs a citation.

## Files to read first

Start from [CLAUDE.md](CLAUDE.md) — the "Where to look first" table maps common questions to entry files. For DI questions also read [fastapi/dependencies/CLAUDE.md](fastapi/dependencies/CLAUDE.md).

Common entry points by topic:

- Request flow / routing → [fastapi/applications.py](fastapi/applications.py), [fastapi/routing.py](fastapi/routing.py)
- Dependency injection → [fastapi/dependencies/utils.py](fastapi/dependencies/utils.py), [fastapi/dependencies/models.py](fastapi/dependencies/models.py)
- Parameter parsing (`Query`, `Path`, `Body`, …) → [fastapi/params.py](fastapi/params.py), [fastapi/param_functions.py](fastapi/param_functions.py)
- OpenAPI generation → [fastapi/openapi/utils.py](fastapi/openapi/utils.py)
- Response serialization → [fastapi/routing.py](fastapi/routing.py) `serialize_response`, [fastapi/encoders.py](fastapi/encoders.py)
- Pydantic v1/v2 differences → [fastapi/_compat/](fastapi/_compat/)
- Security / auth → [fastapi/security/](fastapi/security/)

## Workflow

1. **Map the question to an entry file** using the "Where to look first" table. If no row fits, `grep -rn "<symbol>" fastapi/` for the user's exact symbol.
2. **Read the entry file end-to-end.** Don't trust class names — read the body. Note the public surface.
3. **Follow the call graph one hop at a time.** For each cross-module call, read just the callee. Stop at Starlette, Pydantic, or `_compat` boundaries — name the boundary, don't dive in.
4. **Verify with a test.** `grep -rln "<symbol>" tests/` for at least one test that exercises the feature. Cite it so the reader can run it.
5. **Compose the answer in the output shape above.** Every step gets one citation.

## Conventions

- **Cite, don't paraphrase.** A walkthrough step without `path:line` is not done.
- **Stay inside `fastapi/`.** If the trail leaves the package, name the boundary call and stop.
- **No ASCII flowcharts.** A numbered list with file links is more accurate.
- **Read-only.** Never edit or write files in this skill. If the user pivots to "now change it", switch skills.
- **Cap at 7 steps.** If the feature genuinely needs more, ask the user to split the question.

## See also

- [CLAUDE.md](CLAUDE.md) — root project guide
- [fastapi/dependencies/CLAUDE.md](fastapi/dependencies/CLAUDE.md) — nested guide for DI
- `skills/trace-data-flow/SKILL.md` — sibling skill for end-to-end request traces (use when the user names a concrete request rather than a feature)
