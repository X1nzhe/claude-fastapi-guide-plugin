---
name: trace-data-flow
description: Trace a concrete request through the FastAPI framework from ASGI receipt to response, naming every transformation step with file/line citations. Use when the user asks "what happens when I call X", "follow the path of a POST to /items", "how does a query parameter become a function argument", or any end-to-end runtime walkthrough of a specific request.
---

# Trace a request's data flow

## When to use this skill

The user wants to follow **one specific request** through the framework end-to-end: parsing, validation, dependency resolution, handler invocation, response serialization. Triggers:

- "What happens when I POST to `/items` with a JSON body?"
- "Trace a request through the DI system."
- "Follow a WebSocket handshake through the code."
- "How does a `Query(...)` parameter become a function argument at runtime?"

Do **not** use this skill for static feature explanations (no concrete request) — use `explain-feature` instead. Do not use it for code changes.

## Output shape

Reply with a numbered timeline. One step per transformation, each in this form:

> **N. Stage name** — `path:line`
> **in:** shape of the data entering the step
> **out:** shape leaving the step
> **note:** one sentence on what changed or why

Then a closing one-liner: **"Validation gates"** — which step rejects which kind of bad input (path coercion error vs body schema error vs dependency error). This is the most-asked follow-up; pre-answer it.

The canonical pipeline (skip stages that don't apply to the user's request):

1. ASGI receive (Starlette boundary — name, don't dive)
2. App routing → `APIRoute` match in `fastapi/routing.py`
3. The per-request callable from `APIRoute.get_route_handler`
4. Body read + content-type dispatch (JSON vs form vs file)
5. `solve_dependencies` — sub-deps + path/query/header/cookie extraction
6. Body validation via `ModelField` (from `_compat`)
7. Endpoint invocation (direct `await` vs `run_in_threadpool` bridge)
8. `serialize_response` → `jsonable_encoder` → `response_class`
9. ASGI send

## Files to read first

- [fastapi/applications.py](fastapi/applications.py) — `FastAPI.__call__` and middleware stack
- [fastapi/routing.py](fastapi/routing.py) — `APIRoute.get_route_handler` is the heart of the per-request pipeline
- [fastapi/dependencies/utils.py](fastapi/dependencies/utils.py) — `solve_dependencies`, `request_params_to_args`, `request_body_to_args`
- [fastapi/dependencies/models.py](fastapi/dependencies/models.py) — what `get_dependant` produced at startup
- [fastapi/encoders.py](fastapi/encoders.py) — `jsonable_encoder`
- [fastapi/dependencies/CLAUDE.md](fastapi/dependencies/CLAUDE.md) — two-phase build/solve, dispatch table for callable shapes

## Workflow

1. **Pin the request shape.** Method, path template, content-type, body shape, headers of interest. If the user gave only "a request", ask for these before tracing — generic traces are useless.
2. **Locate the route definition** in their example app, or reason from a synthetic `@app.<method>("<path>")` if no app was given. Note the handler signature — it determines which `solve_dependencies` branches matter.
3. **Open `APIRoute.get_route_handler`** in [fastapi/routing.py](fastapi/routing.py). Every step from here is one entry in the timeline.
4. **For each inner call, open the callee.** Capture input/output shape, not implementation detail. Each timeline entry cites `function-name @ path:line`.
5. **At `solve_dependencies`**, only expand sub-dependant recursion if this request has sub-deps; otherwise write "no sub-deps" and move on.
6. **At serialization**, follow `serialize_response` → `jsonable_encoder`. Note where Pydantic `model_dump` is invoked through `_compat` (the boundary).
7. **Cross-check with a test.** `grep -rln` in [tests/](tests/) for a test issuing a similar request. Cite it so the user can re-run.
8. **Write the timeline + validation-gates one-liner.** Linear; pick one branch when there's ambiguity (e.g. JSON vs form) and note the alternative in a single trailing sentence.

## Conventions

- **One concrete request, one timeline.** Don't cover all method/content-type combinations.
- **Name boundaries, don't cross them.** `await request.body()` enters Starlette; `model.model_validate(...)` enters Pydantic. Cite the call site and stop.
- **Cite `path:line` per step.** A timeline without line numbers is prose, not a trace.
- **Make async vs sync visible.** When the endpoint or a dep is sync, the `run_in_threadpool` bridge is its own step — don't elide it.
- **Caching is part of the trace.** If `solve_dependencies` hits `dependency_cache` for a sub-dep, that's a step (and a perf-relevant detail).

## See also

- `skills/explain-feature/SKILL.md` — sibling skill for static feature explanations
- [fastapi/dependencies/CLAUDE.md](fastapi/dependencies/CLAUDE.md) — execution dispatch table (async-gen / sync-gen / coroutine / sync)
- [CLAUDE.md](CLAUDE.md) "Where to look first" table
