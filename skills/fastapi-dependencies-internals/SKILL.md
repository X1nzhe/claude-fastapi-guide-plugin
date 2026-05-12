---
name: fastapi-dependencies-internals
description: Internal conventions, abstractions, and anti-patterns for FastAPI's dependency-injection core under fastapi/dependencies/. Use when reading or modifying files in fastapi/dependencies/, when working with Dependant / solve_dependencies / get_dependant / analyze_param, when extending the DI system with a new injectable kind, or when debugging dependency caching, scoping, or callable dispatch.
---

# fastapi/dependencies — internals

The dependency-injection core. [fastapi/dependencies/models.py](fastapi/dependencies/models.py) defines `Dependant` — a dataclass representing one callable's injection points. [fastapi/dependencies/utils.py](fastapi/dependencies/utils.py) holds `get_dependant` (build), `solve_dependencies` (evaluate), and the request-parameter parsers.

## When to use this skill

Use this skill whenever the work touches the DI subsystem:

- Reading, modifying, or extending files under [fastapi/dependencies/](fastapi/dependencies/)
- Working with `Dependant`, `solve_dependencies`, `get_dependant`, `analyze_param`, `request_params_to_args`, `request_body_to_args`
- Adding a new injectable parameter kind, security scope, or scope type
- Debugging dependency caching (`use_cache`, `cache_key`), generator scopes (`function` / `request`), or sync/async dispatch

Do **not** use this skill for general FastAPI request-flow questions — those go to the `trace-data-flow` or `explain-feature` skills. This one is specific to DI internals.

## Two-phase pattern (don't blur the phases)

1. **Build (once, at startup)** — `get_dependant` introspects a callable via `get_typed_signature` + `analyze_param` and produces a `Dependant` tree. Side-effect-free.
2. **Solve (per request)** — `solve_dependencies` walks the tree, recursively resolves sub-dependants, extracts path/query/header/cookie/body values, returns a `SolvedDependency`.

Building must be side-effect-free; solving is async. Don't move work between phases.

## Key abstractions

- **`Dependant`** ([fastapi/dependencies/models.py:32](fastapi/dependencies/models.py#L32)) is effectively **append-only after construction**. Its `@cached_property` slots (`is_coroutine_callable`, `is_gen_callable`, `is_async_gen_callable`, `cache_key`, `oauth_scopes`) memoize on first access — mutating fields afterwards corrupts the cache. Build a new `Dependant` instead of editing one.
- **Callable classification** uses `_unwrapped_call` + `_impartial` to see through `functools.partial` and `functools.wraps`. Always use these helpers; raw `inspect` misses wrapped callables.
- **`DependencyCacheKey = (call, oauth_scopes, computed_scope)`** — per-request cache; `Depends(..., use_cache=False)` opts out.
- **Scopes**: `"function"` vs `"request"` (generators default to `"request"`). Cleanup runs via `AsyncExitStack`s on `request.scope` (`fastapi_function_astack` / `fastapi_inner_astack`).

## Execution dispatch (in `solve_dependencies`)

Per sub-dependant, the branch is chosen from `Dependant`'s `cached_property` flags:

| Call shape | Branch |
|------------|--------|
| async generator | `asynccontextmanager(call)` → `stack.enter_async_context` |
| sync generator | `contextmanager_in_threadpool(contextmanager(call))` → `enter_async_context` |
| coroutine function | `await call(**values)` |
| plain sync callable | `await run_in_threadpool(call, **values)` |

Every sync path goes through anyio — never call user code directly on the event loop.

## Conventions

- **Pydantic access goes through `fastapi._compat`** (`ModelField`, `Undefined`, …). No direct `pydantic` imports here — breaks v1 support.
- **Errors are accumulated, not raised.** Parsers return `(values, errors)`; `solve_dependencies` aggregates them into one 422. Don't `raise` mid-parse for validation errors.
- **Dependency overrides** rebuild the sub-dependant with `get_dependant`; preserve that path when adding fields to `Dependant`.

## Don't

- **Don't mutate a `Dependant` after construction** — `cached_property` will hand back stale data.
- **Don't bypass `analyze_param`** when adding a new injectable kind; classification lives in one place on purpose.
- **Don't remove the `async_exit_stack` parameter of `solve_dependencies`** despite the TODO — it's kept for external monkey-patching (the comment says so).
- **Don't add a sync variant of `solve_dependencies`** — async-first; bridge sync user code via `run_in_threadpool` / `contextmanager_in_threadpool`.
- **Don't recurse into `solve_dependencies` without forwarding `dependency_cache`** — losing it silently double-executes deps and breaks `use_cache=True`.

## See also

- [fastapi/dependencies/utils.py](fastapi/dependencies/utils.py) — `solve_dependencies`, `get_dependant`, `analyze_param`
- [fastapi/dependencies/models.py](fastapi/dependencies/models.py) — `Dependant` dataclass
- [fastapi/_compat/](fastapi/_compat/) — Pydantic v1/v2 shims (don't bypass)
- `skills/trace-data-flow/SKILL.md` — sibling skill for end-to-end request traces
- `skills/explain-feature/SKILL.md` — sibling skill for static feature explanations
