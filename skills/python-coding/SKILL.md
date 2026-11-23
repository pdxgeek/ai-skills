---
name: python-coding
description: How to write Python in this codebase - the house standard for any Python work on 3.12, 3.13 or 3.14, from a one-line fix to a new service. Use it every time you write, change, review or test Python - FastAPI routes, services, repositories, pydantic models, dataclasses, scripts, CLIs, config, tests - even when the user only says "add a function", "fix this bug", "write the endpoint for X" or "clean this up" and never mentions Python, style or quality. Also use it when reviewing a diff that shows Any, "# type: ignore", bare except, mutable default arguments, dicts passed between layers, isinstance ladders, print(), f-strings inside log calls, requests or os.path in new code, module-level singletons, or mock.patch of the project's own modules. The refactoring sub-skill inside covers converting existing code safely.
---

# Writing Python

This is how Python gets written here: types the checker proves, frozen dataclasses for
data, pydantic at the boundary and plain objects inside, small named functions, one home
for every rule, errors that travel to one handler, structured logs, and tests that run
the real thing with fakes passed in. The standard is the same for a new service and a
two-line fix.

The body of this file is the standard in brief. The references carry the depth and the
reasons; open them when the task touches their area.

| When the task involves | Read |
|---|---|
| Any non-trivial type, function, async flow, error path, match statement | `references/style.md` |
| New modules, project layout, dependency wiring, config, database, FastAPI, logging, uv/ruff/mypy, shutdown | `references/architecture.md` |
| Writing or changing tests, deciding what to fake, database or HTTP tests | `references/testing.md` |
| Changing code that already works: cleanups, modernising, "make this a comprehension" | `references/refactoring.md` (the refactoring sub-skill) |

## How to work

**Before writing.** Read the surrounding module and one neighbour to learn the repo's
conventions: layout, how dependencies arrive, which validator, ORM and logger are in
use, how errors reach the client. The repo's convention wins over this document
wherever they differ. Check `requires-python` and the installed majors of pydantic,
SQLAlchemy, FastAPI and pytest before using an API that changed between versions (see
the version notes in each reference).

**While writing.** Start from the types: the input shape, the output shape, the error
cases. Then write the function so the checker proves it. One level of abstraction per
function, names that say what things mean, each piece of knowledge in one place. Parse
at the boundary, trust inside. Let errors propagate to the one layer that acts on them.

**The toolchain.** `uv` owns the interpreter, the virtualenv and the lockfile:
`uv run <cmd>` for every command, `uv add` for dependencies, `uv sync` after a pull.
Never `pip install` into a global interpreter, never edit `uv.lock` by hand, never
change `requires-python` except to deliberately move the project. Lint and format with
`ruff`, type check with `mypy --strict` or `pyright` in strict mode, whichever the repo
configures. Details in `references/architecture.md`.

**Before calling it done.** Run `uv run ruff format`, `uv run ruff check`, the type
checker and the tests scoped to the touched code. Python runs untyped code happily, so
"it runs" is never evidence that it type checks. State what was verified and how; when
something could not be verified, say so first.

## The standard in brief

### Types prove the code

- Every function signature is fully annotated, parameters and return. Locals are
  inferred. The type checker runs in strict mode and the build fails on errors.
- No `Any`. `object` for a value not yet examined, narrowed by `isinstance`, a
  `TypeGuard`/`TypeIs` function or a pydantic parse. `Any` switches the checker off for
  everything it touches.
- No `cast()` and no `# type: ignore` except at a verified boundary with a comment
  saying why, and `# type: ignore[code]` always names the error code. A guard or an
  honest `T | None` is the fix.
- `X | None`, never `Optional[X]`; `list[int]`, never `List[int]`; `collections.abc`
  (`Sequence`, `Mapping`, `Iterable`, `Callable`) for parameters, concrete types for
  returns. PEP 695 syntax for generics (`def first[T](items: Sequence[T]) -> T`) and
  `type Alias = ...` for aliases.
- Model states as a union of frozen dataclasses with a `Literal` discriminant, not one
  class with optional fields. `match` on the discriminant and finish with
  `case _: assert_never(value)` so adding a variant lists every place that must learn
  about it.
- `Protocol` for the shape a function needs from a collaborator; the implementation
  never imports the protocol. `TypedDict` only for JSON-shaped dicts at a boundary that
  pydantic does not own. `NewType` for ids (`UserId = NewType("UserId", int)`) minted at
  the boundary, so an `OrderId` cannot be passed where a `UserId` goes.

### Data

- `@dataclass(frozen=True, slots=True)` for every internal value: results, commands,
  events, projections, compound keys. `kw_only=True` when it has more than two fields.
  Copy with `dataclasses.replace`. Never a mutable default; `field(default_factory=list)`.
- `enum.StrEnum` for closed sets that cross a wire; `Literal` for a small internal set.
- Immutable collections by type: accept `Sequence`/`Mapping`, store `tuple` and
  `frozenset`, return what the caller needs. Never mutate an argument.
- pydantic `BaseModel` only at trust boundaries (HTTP bodies, env, queue messages,
  third-party JSON). Internal code passes dataclasses, never raw dicts. `model_config =
  ConfigDict(frozen=True, extra="forbid")`.
- `datetime` is always aware (`tzinfo=UTC`) and comes from an injected clock, never
  `datetime.now()` inside a service. `pathlib.Path`, never `os.path`. `decimal.Decimal`
  for money.

### None

- `None` is the one "absent". `if value is None`, never `if not value`, when `0`, `""`
  and `[]` are legitimate. Return `T | None` for "no result"; raise for "should exist".
- Narrow once at the top of the function with a guard, then the rest of the body sees
  `T`. No `assert x is not None` outside tests; raise a typed error or return.
- Database `NULL` is mapped to `None` once, in the row mapper or the ORM model, with a
  type that says so.

### Functions and names

- One level of abstraction per function: it orchestrates named steps or it is a step.
  The orchestrator calls each step in turn and every step returns to it; a step never
  ends by calling the next step. A body that needs section comments is several
  functions. Around twenty lines is the signal to look for the seam.
- Guards first, early return, happy path at the shallowest indent. At most three
  positional parameters, then keyword-only (`*,`). A boolean parameter that switches
  behaviour is two functions.
- Pure functions in the middle, side effects at the edges. Comprehensions and generator
  expressions for a transform; a `for` loop when the body mutates locals, breaks early
  or needs several statements. `any()`/`all()` over a counted filter. `sorted(key=...)`
  with `operator.attrgetter`, never `cmp_to_key` for a plain field.
- PEP 8 names: `snake_case` functions and variables, `PascalCase` classes, `UPPER_SNAKE`
  only for true constants, `_leading_underscore` for module-private. Functions are
  verbs that state the effect (`load_active_prompt`); booleans are predicates
  (`is_multi_unit`, `has_policy`); collections are plural; a dict is named by what it
  indexes (`policies_by_property_id`). No abbreviations beyond `id`, `url`, `db`,
  `ctx`. Comments say why, never what. Docstrings on public modules, classes and
  functions; one line unless the behaviour needs more.

### Modules and layers

- `src/<package>/` layout with `pyproject.toml` at the root. Package by feature:
  `orders/routes.py`, `orders/service.py`, `orders/repository.py`, `orders/schemas.py`,
  `orders/test_service.py` side by side. Buckets named `services/`, `utils/` at the top
  level suit only a small app.
- Dependency direction is one way: route, service, repository, database. Routes parse,
  validate and map errors to HTTP. Services hold the rules and own the transaction.
  Repositories hold the SQL or ORM queries and return mapped dataclasses. No request
  objects below the route, no SQL above the repository.
- Dependencies arrive as parameters. A service is a class whose `__init__` takes its
  collaborators typed as `Protocol`s, or a closure built by a factory function; one
  composition root builds the graph. FastAPI `Depends` lives in the route layer only.
  No module-level singletons (`engine = create_engine(...)` at import time), no global
  mutable state, no `import` for side effects.
- Absolute imports from the package root; no `from x import *`; no re-export barrels in
  `__init__.py` beyond the package's public API. Imports at the top, sorted by ruff.
- Settings are one `pydantic_settings.BaseSettings` class, built once at startup and
  passed in. A missing variable fails the boot, not a request. `os.environ` appears in
  that module only.
- Standard library first: `pathlib`, `datetime`, `itertools`, `functools`, `statistics`,
  `tomllib`, `zoneinfo`, `asyncio.TaskGroup`. `httpx` over `requests`; nothing pulled
  in for what the platform already does.

### Boundaries, errors and async

- Parse, do not validate: every trust boundary goes through a pydantic model, and the
  model is the only source of the type. `model_validate` at the edge; the internals
  receive the typed value and never re-check it.
- Exceptions are the error channel. One `DomainError(Exception)` per bounded context,
  leaf types only when a caller branches on them, standard exceptions
  (`ValueError`, `KeyError`) for programmer mistakes. Raise with `from` when wrapping so
  the cause is kept. `except Exception` only at the one boundary that logs and converts;
  never bare `except:`, never a swallowed exception, never `except ... : pass` without a
  comment saying why.
- Domain errors are mapped to HTTP in one place, a FastAPI exception handler, as
  RFC 9457 problem details. Routes never write `HTTPException(404)` by hand. Never
  leak stack traces or SQL to the client.
- Return `T | None` or raise; a function does one or the other. A result union of
  dataclasses when the caller must branch on several expected outcomes.
- `async def` end to end when the framework is async; never call blocking I/O
  (`time.sleep`, `requests`, a sync driver) inside a coroutine. `asyncio.TaskGroup` for
  independent work, bounded by an `asyncio.Semaphore` over user-sized input;
  `asyncio.timeout()` for deadlines. `asyncio.to_thread` for the one blocking call that
  has no async client. Every `Task` is awaited or stored; no fire-and-forget.
- Context managers (`with`, `async with`) for every resource: files, sessions,
  connections, clients, locks. Write your own with `contextlib.contextmanager`.

### Logging

One structured `structlog` logger, configured once at startup and bound per request with
`request_id`. Every line has a constant event name and the values as keyword fields:
`log.info("screening_extraction_completed", property_id=pid, duration_ms=ms)`. Never
f-strings or `%` inside a log call. Log once per event at the layer that owns it;
failures are logged by the exception handler, once, with `exc_info`. Secrets redacted at
the processor. `print()` never appears outside a CLI's output path.

### Tests

pytest. A `test_*.py` beside the module or under `tests/` mirroring the package; one
`def test_<behaviour>` per behaviour in plain present tense. Arrange, act, assert.
Fixtures for setup, `@pytest.mark.parametrize` over copy-pasted cases. Fakes over mocks:
pass a two-line fake through the constructor; never `mock.patch` a module under `src/`.
Repositories run against a real Postgres in a container. HTTP edges are faked with
`respx`. A test that cannot fail is deleted. Details in `references/testing.md`.

## Refactoring existing code

Changing code that already works is its own discipline: equivalence first, delete before
converting, one move per smell, verified with the project's tests. The smell-to-move table
and worked examples live in `references/refactoring.md`. Read it whenever the task is a
cleanup, a modernisation or a review of someone else's diff.

## Reporting the work

State what was built or changed and what was verified, in plain sentences. Name the tests
that cover it. When a decision went against this standard because the repo does it
differently, say so in one line. When behaviour changed in any observable way, that is the
headline, not a footnote.
