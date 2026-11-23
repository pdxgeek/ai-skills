# Architecture: layout, wiring, config, database, HTTP, logging, tooling, operations

How a Python service is put together so each piece has one home and can be tested with a
constructor call. Sources: the Python Packaging User Guide (src layout, pyproject), the
uv and ruff documentation, PEP 621/735, the FastAPI docs (lifespan, dependencies,
exception handlers), the SQLAlchemy 2.0 ORM guide, the Alembic tutorial, the pydantic
settings docs, the structlog docs, RFC 9457, the Twelve-Factor App.

Contents: Project layout · Layers · Dependency wiring · Config and env · Toolchain ·
Database · HTTP with FastAPI · Logging · Security basics · Process lifecycle · Packaging

## Project layout

```
pyproject.toml          # metadata, dependencies, tool config (ruff, mypy/pyright, pytest)
uv.lock                 # committed
.python-version         # written by `uv python pin`
src/<package>/
  __init__.py
  main.py               # composition root: build settings, deps, app; nothing else
  config.py             # the one Settings class
  db.py                 # engine/session factory builders (functions, not module globals)
  errors.py             # the per-context base exceptions
  logging.py            # structlog configuration
  orders/
    __init__.py
    routes.py           # FastAPI router: parse, call service, map to response
    service.py          # rules, transaction boundary
    repository.py       # SQL / ORM queries, row -> dataclass
    models.py           # ORM mapped classes for this feature
    schemas.py          # pydantic request/response models
    domain.py           # frozen dataclasses, result unions, NewType ids
    test_service.py     # or tests/orders/test_service.py mirroring the package
tests/
  conftest.py           # shared fixtures: settings, database container, client
alembic/                # migrations, when the repo owns its schema
```

- `src/` layout so the installed package is what tests import, never the working
  directory by accident.
- Package by feature. A feature folder is complete: anyone reading `orders/` sees every
  layer of orders. Shared kernel modules (`db.py`, `errors.py`, `clock.py`) stay few and
  stable.
- A feature never imports another feature's `repository` or `models`; it calls the other
  feature's service through a `Protocol` it declares, or reacts to an event.
- Tests live beside the module (`test_service.py`) or in `tests/` mirroring the package.
  One convention per repo.

## Layers

| Layer | Owns | Never |
|---|---|---|
| route (`routes.py`) | HTTP parsing, pydantic schemas, status codes, `Depends`, mapping domain errors via the exception handler | SQL, business rules, transactions |
| service (`service.py`) | rules, orchestration, the transaction (`async with session.begin()`), domain events | `Request`, `Response`, pydantic request schemas, raw SQL |
| repository (`repository.py`) | SQL / ORM statements, row to dataclass mapping, `NULL` to `None` | rules, HTTP types, committing |
| db (`db.py`) | engine and session factory construction | application logic |

A service receives and returns domain dataclasses. The route converts between pydantic
schemas and domain types; the repository converts between rows and domain types. Each
conversion is written once.

## Dependency wiring

- Dependencies are explicit constructor or function parameters, typed as the narrowest
  `Protocol` the consumer needs:

  ```python
  class OrderRepository(Protocol):
      async def find(self, order_id: OrderId) -> Order | None: ...
      async def save(self, order: Order) -> None: ...

  class OrderService:
      def __init__(self, repo: OrderRepository, clock: Clock, log: BoundLogger) -> None:
          self._repo = repo
          self._clock = clock
          self._log = log
  ```

- One composition root (`main.py` or `app.py`) builds settings, engine, repositories,
  services and the FastAPI app, in that order. Nothing else constructs a service.
- FastAPI `Depends` is the route layer's way to reach the graph: a `get_order_service`
  dependency reads the service off `app.state` (set in the lifespan) or builds one per
  request with a per-request session. Services never import `fastapi`.
- No module-level singletons: `engine = create_async_engine(...)` at import time runs on
  `import` in every test and cannot be swapped. A `build_engine(settings)` function is
  called by the root.
- No global mutable state, no `import` for side effects, no service locator, no DI
  framework. The constructor is the contract.
- Clock and randomness are dependencies (`Clock` protocol with `now() -> datetime`,
  `random.Random` instance) so tests control them without patching.

## Config and env

- One `Settings(BaseSettings)` in `config.py` with typed fields, `SecretStr` for
  secrets, `model_config = SettingsConfigDict(env_prefix="APP_", frozen=True)`. Built
  once in the root; a missing or malformed variable fails the boot with a readable
  error.
- `os.environ` appears in `config.py` only. Features receive the fields they need, not
  the whole `Settings`.
- `.env` files for local development through `env_file`, never committed. Production
  gets real environment variables from the orchestrator or secret store.
- No feature flags read as raw strings in services; the settings field is a `bool` or a
  `StrEnum`.

## Toolchain

- **uv** manages Python versions (`uv python pin 3.13`), the virtualenv (`.venv/`,
  ignored), dependencies (`uv add httpx`, `uv add --dev pytest`) and the lockfile
  (`uv.lock`, committed). Every command runs as `uv run <cmd>`; CI uses `uv sync
  --frozen`. Never `pip install` globally, never `requirements.txt` as the source of
  truth, never edit `uv.lock` by hand.
- **ruff** is the formatter and the linter. In `pyproject.toml`:

  ```toml
  [tool.ruff]
  target-version = "py313"
  line-length = 100

  [tool.ruff.lint]
  select = ["E", "F", "W", "I", "N", "UP", "B", "A", "C4", "SIM", "RET", "ARG", "PTH",
            "ERA", "PL", "RUF", "ASYNC", "S", "T20", "ANN", "TCH", "DTZ", "LOG", "G"]
  ignore = ["ANN401"]  # keep `Any` banned via the type checker instead
  ```

  `T20` bans `print`, `PTH` bans `os.path`, `DTZ` bans naive datetimes, `G` bans
  f-strings in log calls, `ASYNC` catches blocking calls in coroutines, `S` is bandit.
  `ruff format` replaces black and isort.
- **Type checker**: `mypy --strict` or `pyright` strict, configured in `pyproject.toml`,
  run in CI as a gate. Third-party packages without stubs get `types-*` packages or a
  per-module `ignore_missing_imports`, never a global one.
- **pre-commit** runs ruff format, ruff check and the type checker on staged files.
- CI gates: `uv run ruff format --check`, `uv run ruff check`, the type checker,
  `uv run pytest`. A gate that is allowed to fail is not a gate.

## Database

- **Postgres** by default. Its `JSONB`, arrays, `RETURNING`, partial indexes, advisory
  locks and `ON CONFLICT` cover most of what tempts people toward a second store.
  Another engine only when the repo already runs it.
- **SQLAlchemy 2.0** with the ORM for mapped entities and `select()` for every query;
  **psycopg 3** (sync or async) or **asyncpg** as the driver. Raw SQL through
  `text()` with bound parameters when the ORM fights the query; never f-strings into
  SQL.
- Mapped classes:

  ```python
  class Base(DeclarativeBase): ...

  class OrderRow(Base):
      __tablename__ = "orders"
      id: Mapped[int] = mapped_column(primary_key=True)
      status: Mapped[str] = mapped_column(String(20))
      closed_at: Mapped[datetime | None]
  ```

  Rows map to a frozen `Order` dataclass in the repository. ORM objects never leave the
  repository; a route that returns one serialises lazy attributes by accident.
- Sessions: one per request or job, opened by the root or a `Depends`, passed into the
  service, closed in `finally` or by the context manager. The service owns the
  transaction: `async with session.begin():` around the use case. Repositories never
  `commit()`.
- `expire_on_commit=False` on the session factory for async; set relationship loading
  explicitly (`lazy="raise"` by default, `selectinload` per use case) so an N+1 fails
  loudly instead of silently.
- Ids: `BIGINT GENERATED ALWAYS AS IDENTITY` or a sequence; UUIDv7 when ids are minted
  by the application. Timestamps `TIMESTAMPTZ`. Money `NUMERIC`.
- **Alembic** owns the schema. Every change is a migration file reviewed with the code;
  `autogenerate` is a draft, read before committing. No `Base.metadata.create_all` in
  production code. Migrations run as a deploy step, not at import time.
- Lightweight services with no ORM need: `psycopg` with `row_factory=dict_row`, then an
  explicit dataclass mapper. Still no `Any` beyond the mapper.

## HTTP with FastAPI

- App built by a `create_app(settings, deps) -> FastAPI` function so tests build one with
  fakes. Startup and shutdown in a `lifespan` async context manager, never
  `@app.on_event`.
- Routers per feature (`APIRouter(prefix="/orders", tags=["orders"])`), included by the
  root. Route functions are thin: parse via the pydantic schema in the signature, call
  one service method, return a response schema. No rules, no SQL.
- Response models declared (`response_model=OrderResponse` or a return annotation) so
  the OpenAPI schema is true and extra fields never leak.
- Domain errors reach the client through exception handlers registered once:

  ```python
  @app.exception_handler(DomainError)
  async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
      problem = to_problem_details(exc)  # type, title, status, detail, instance
      return JSONResponse(problem.model_dump(), status_code=problem.status,
                          media_type="application/problem+json")
  ```

  Routes never raise `HTTPException` for a domain condition. Unknown exceptions are
  logged once by a catch-all handler and become a 500 with no detail.
- Validation errors from pydantic are also mapped to problem details, with the field
  errors in an `errors` extension.
- `httpx.AsyncClient` for outbound calls, built once in the lifespan with explicit
  `timeout`, passed in as a dependency, closed on shutdown. Retries with a bounded
  backoff on idempotent calls only.
- Request id from the incoming header or a fresh UUID, bound to the logger in a
  middleware, returned in the response.

## Logging

- **structlog** configured once in `logging.py`: ISO timestamps, log level, the bound
  context, JSON renderer in production, console renderer locally. The stdlib `logging`
  is routed through structlog so library logs land in the same stream.
- Every log line is an event name in constant `snake_case` words plus keyword fields.
  `log.info("order_closed", order_id=oid, reason=reason)`. Never
  `log.info(f"closed {oid}")`; the platform cannot group what it cannot match.
- Per-request context with `structlog.contextvars.bind_contextvars(request_id=...)` in
  the middleware, cleared at the end. Services get a logger through the constructor
  and bind feature context (`log.bind(order_id=oid)`).
- Levels: `error` means someone acts now; `warning` is handled but abnormal; `info` is a
  business lifecycle event, one or two per request; `debug` is off in production.
- Log once per event at the layer that owns it. Failures are logged by the exception
  handler or job runner, once, with `exc_info=True`. No catch-log-reraise ladders.
- Secrets never reach a log line: a processor redacts known keys (`password`, `token`,
  `authorization`); `SecretStr` renders as `**********`; request and response bodies
  are not logged.
- `print()` only in a CLI's output path, through `typer.echo` or `rich`, never for
  diagnostics.

## Security basics

- Parameters, never string formatting, in SQL. `LiteralString` on raw query helpers.
- Validate size and shape at the edge: body size limits in the server, pydantic
  constraints on lengths and ranges, `max_length` on free text.
- Secrets from the environment or secret store through `Settings`, typed `SecretStr`,
  never in code or logs. Rotate by restart.
- `secrets` module for tokens, never `random`. `hashlib`/`hmac` with a modern
  algorithm; `argon2-cffi` or `bcrypt` for passwords.
- Dependencies pinned by `uv.lock`; `uv lock --upgrade` on a schedule with the tests as
  the gate. `pip-audit` or the ruff `S` rules in CI.
- No `pickle`, `yaml.load` without `SafeLoader`, `eval`, `exec`, or `subprocess` with
  `shell=True` on untrusted input.
- Timeouts on every outbound call and every database statement
  (`statement_timeout`). A missing timeout is an outage waiting for a slow peer.

## Process lifecycle

- `uvicorn` (or `granian`) runs the app; the process manager or orchestrator runs
  several workers. The app itself never forks.
- `lifespan` opens the engine and clients on startup and closes them on shutdown in
  reverse order. On `SIGTERM` uvicorn stops accepting, waits for in-flight requests up
  to the grace period, then runs the shutdown half of the lifespan.
- Health endpoints: `/health/live` returns 200 when the process is up; `/health/ready`
  checks the database with a cheap `SELECT 1` and a short timeout.
- Background jobs are a separate entry point (`python -m package.jobs run_x`) sharing
  the same composition root, locked with a database advisory lock or a scheduler that
  guarantees one runner. Never a thread started at web-app import time.
- Unhandled exception in a job: log once with the job name and inputs, mark the run
  failed, exit non-zero so the scheduler sees it.

## Packaging

- `pyproject.toml` with PEP 621 metadata, `[project.scripts]` for CLI entry points,
  `[dependency-groups]` (PEP 735) for `dev` and `test`.
- Docker: multi-stage, `uv sync --frozen --no-dev` in the build stage, copy `.venv` and
  `src/` into a slim runtime image, run as a non-root user, `PYTHONUNBUFFERED=1`,
  `PYTHONDONTWRITEBYTECODE=1`. The image never contains the test suite or the lockfile
  resolver.
- Version from `pyproject.toml` via `importlib.metadata.version("package")`, never a
  hand-maintained `__version__` string.
