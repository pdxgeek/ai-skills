# Preferred stack

The default pick for every kind of dependency a Python service needs. Each pick is proven:
widely deployed, actively maintained by an organisation rather than one person, with a
stable API and a long release history. The repo's existing choice wins where it differs;
a library not on this list needs a one-line reason in the pull request.

How to choose, in order: the standard library, then what the framework already provides,
then this list. Nothing is added for what the platform already does.

Contents: Platform (shared across languages) · Runtime and build · Framework · Data ·
Messaging and caching · Clients and integration · Observability · Security · Testing ·
Lint, format and types · Not this

## Platform (shared across languages)

| Need | Pick | Notes |
|---|---|---|
| Relational database | **PostgreSQL** (RDS) | One engine for everything relational. `JSONB`, arrays, partial indexes, `RETURNING`, advisory locks and `ON CONFLICT` remove most reasons to add a second store. |
| Schema migrations | **Liquibase** | Changesets reviewed with the code, run as a deploy step, never at application start in production. **Alembic** when the Python service is the only owner of its schema and nothing else runs Liquibase. |
| Work queue | **Amazon SQS** | Standard queues by default; FIFO only when ordering or exactly-once matters, and then one message group per aggregate. Dead-letter queue on every queue. |
| Publish/subscribe, fan-out | **Amazon SNS** | Topic per event type, SQS subscriptions per consumer. Producers never know their consumers. |
| Object storage | **Amazon S3** | Presigned URLs for uploads and downloads; the service never proxies bytes. |
| Secrets and config | **AWS Secrets Manager** for secrets, **SSM Parameter Store** for non-secret config | Read at boot into typed config; never at request time, never from a committed file. |
| Scheduled work | **EventBridge Scheduler** firing an SQS message or Lambda | No in-process cron in a web service. |
| Shared cache (cross-instance) | **Redis** (ElastiCache) only when a cache must be shared | In-process first; see Messaging and caching. |
| Telemetry | **OpenTelemetry** traces and metrics, shipped to **Datadog** | Structured JSON logs; one trace id across services. |
| Serverless glue | **AWS Lambda** for event-driven glue, never for the core API | Same logging and telemetry contract as the service. |
| Infrastructure | **Terraform** | Nothing created by hand in the console. |
| CI | **GitHub Actions** | Format, lint, type check, tests as required gates. |
| Containers | **Docker**, multi-stage, non-root | One image per service. |

## Runtime and build

| Need | Pick | Notes |
|---|---|---|
| Interpreter | Current stable **CPython 3.x** (3.13 or 3.14), pinned with `uv python pin` | `.python-version` committed. |
| Project, env, lock | **uv** | `pyproject.toml` plus `uv.lock`; `uv run` for every command; `uv sync --frozen` in CI. Never pip, pipenv, poetry, pip-tools or conda. |
| Packaging metadata | PEP 621 `[project]`, PEP 735 `[dependency-groups]` | `[project.scripts]` for entry points. |
| CLI | **Typer** (on Click) | **rich** for output a person reads. |

## Framework

| Need | Pick | Notes |
|---|---|---|
| HTTP API | **FastAPI** on **uvicorn** | `lifespan` for startup and shutdown; routers per feature. |
| Validation and settings | **pydantic** v2 and **pydantic-settings** | The model is the type. |
| Dependency wiring | Constructor parameters typed as `Protocol`s, one composition root | FastAPI `Depends` only in the route layer. No DI framework. |
| Templating | **Jinja2** | Emails and documents; autoescape on. |
| Admin-heavy web app (when required) | **Django** with the Django ORM and its migrations | Only for an app whose value is the admin and auth plumbing; otherwise FastAPI. |

## Data

| Need | Pick | Notes |
|---|---|---|
| Driver | **psycopg 3** (sync and async, one library) | **asyncpg** where a repo already uses it with SQLAlchemy async. |
| ORM and SQL | **SQLAlchemy 2.0**: `Mapped` models for aggregates, `select()` for queries, `text()` with bound parameters when the ORM fights the query | Never the legacy `Query` API. |
| Migrations | **Liquibase** or **Alembic** | See Platform. |
| Ids | `BIGINT` identity for internal keys; **UUIDv7** (`uuid.uuid7()` on 3.14, **uuid-utils** below) for public identifiers | |
| Money | **`decimal.Decimal`** with an explicit quantize, stored as `NUMERIC` | Never `float`. |
| Time | **`datetime`** aware, **`zoneinfo`**, an injected clock, `TIMESTAMPTZ` | Never `pytz`, never naive datetimes, never `utcnow()`. |
| Data analysis (when required) | **polars** for new work; **pandas** where the ecosystem demands it | Not in a request path. |

## Messaging and caching

| Need | Pick | Notes |
|---|---|---|
| AWS SDK | **boto3** in sync code; **aioboto3** (on aiobotocore) in async code | Clients built once in the composition root, typed with **boto3-stubs** / **types-aiobotocore**. |
| SQS consumer | A long-polling loop in a worker entry point over the SDK, with visibility timeout longer than the handler's worst case and idempotent handlers keyed by message id | Never Celery, never RQ. |
| SNS publishing | The SDK behind a `Publisher` `Protocol` | |
| In-process cache | **cachetools** (`TTLCache`, `LRUCache`) with a bound and a TTL; `functools.cache` for pure, bounded functions | The only in-process caches. |
| Shared cache | **Redis** through **redis-py** (`redis.asyncio` in async code) | Only when several instances must see the same entry. Never as a queue, never as the primary store. |
| Distributed lock | Postgres advisory locks | For the rare one-runner section. |
| Background work | A separate worker entry point consuming SQS, sharing the composition root | Never a thread started at web-app import time. |

## Clients and integration

| Need | Pick | Notes |
|---|---|---|
| HTTP client | **httpx** (`AsyncClient` in async code), one client per upstream built in the lifespan, explicit `timeout` | Never `requests` in new code, never `urllib3` directly, never `aiohttp` next to httpx. |
| Retries | **tenacity** on idempotent calls, with backoff and jitter | |
| Email | The provider's REST API through httpx, each message a typed function with a Jinja2 template | |
| PDF | **WeasyPrint** from HTML; **pypdf** for assembly | Never in the request path for large documents. |
| CSV, Excel | stdlib **csv**; **openpyxl** for Excel | Streaming on large files. |
| Feature flags | Typed `bool` or `StrEnum` fields on `Settings` | A flag service only when flags change at runtime. |

## Observability

| Need | Pick | Notes |
|---|---|---|
| Logging | **structlog**, configured once, stdlib `logging` routed through it | Constant event name, values as keyword fields. |
| Traces and metrics | **OpenTelemetry** (`opentelemetry-sdk`, `opentelemetry-instrumentation-fastapi`, `-sqlalchemy`, `-httpx`), exported to **Datadog**; **ddtrace** where the repo already runs it | Trace id bound into every log line. |
| Health | `/health/live` and `/health/ready` routes; readiness runs `SELECT 1` with a timeout | |

## Security

| Need | Pick | Notes |
|---|---|---|
| JWT and JOSE | **joserfc** (or **PyJWT** where already in use) | Never `python-jose`. |
| Passwords | **argon2-cffi** | Never a hand-rolled hash, never `hashlib` for passwords. |
| Secrets in code | `SecretStr` on `Settings` | Never logged, never in a repr. |
| Dependency vulnerabilities | **Dependabot** plus **pip-audit** in CI; ruff's `S` rules for code | |

## Testing

| Need | Pick | Notes |
|---|---|---|
| Runner | **pytest** with **pytest-asyncio** (`asyncio_mode = "auto"`) | Never `unittest` classes in new tests. |
| HTTP edges | **respx** for outbound (`assert_all_called`, `assert_all_mocked`); `httpx.AsyncClient` over `ASGITransport` for inbound | |
| Database | **testcontainers** (`PostgresContainer`) | Never SQLite standing in for Postgres. |
| Fakes | Small classes satisfying the `Protocol`, passed through the constructor | Never `mock.patch` of `src/`. |
| Factories | **factory_boy** or a `make_<thing>(**overrides)` function | |
| Property tests | **Hypothesis** | |
| Parallel | **pytest-xdist** | |
| Time (third-party code only) | **time-machine** | The project's own code takes an injected clock. |

## Lint, format and types

| Need | Pick | Notes |
|---|---|---|
| Lint and format | **ruff** (`ruff format`, `ruff check`) | Replaces black, isort, flake8, pylint, pyupgrade, bandit. |
| Type checker | **mypy --strict** or **pyright** strict; one per repo | A type error fails CI. Stubs from `types-*` packages, never a global `ignore_missing_imports`. |
| Hooks | **pre-commit** running ruff and the type checker | |

## Not this

| Avoid | Because | Use instead |
|---|---|---|
| requests, urllib3 direct, aiohttp | Blocks the event loop, or a second HTTP stack | httpx |
| Celery, RQ, Dramatiq, Huey | A broker to operate when SQS exists; Celery's failure modes are famous | an SQS worker |
| Redis as a queue or a primary store | Visibility, retries and durability have to be rebuilt | SQS, Postgres |
| pip, pipenv, poetry, pip-tools, conda | Slower, or a second lockfile format | uv |
| black, isort, flake8, pylint, autopep8 | Superseded by one faster tool | ruff |
| Flask for a new API | No typing, no validation, no async model | FastAPI |
| SQLAlchemy 1.x `Query`, Peewee, Tortoise, SQLModel | Legacy API, small communities, or a thin layer that fights pydantic v2 and SQLAlchemy 2 | SQLAlchemy 2.0 |
| SQLite in production or in tests standing in for Postgres | The dialect differs where it hurts | Postgres, testcontainers |
| marshmallow, cerberus, attrs-based validation | Superseded for boundary parsing | pydantic v2 |
| pytz, arrow, pendulum, `datetime.utcnow()` | `zoneinfo` and aware `datetime` cover it | stdlib |
| `python-jose` | Unmaintained | joserfc, PyJWT |
| `unittest.mock.patch` of `src/` | Hides the dependency that was never injected | constructor fakes |
| `print()` for diagnostics, `logging.info(f"...")` | Unstructured, ungroupable | structlog events |
| `pickle` for storage or transport, `yaml.load` without `SafeLoader` | Arbitrary code execution | JSON through pydantic; `yaml.safe_load` |
| APScheduler, schedule, a `while True: sleep` loop in the web process | Runs once per instance and dies with the pod | EventBridge Scheduler |
| MySQL, MariaDB, MongoDB, DynamoDB for a new service | One engine the team operates well; `JSONB` covers document-shaped columns | Postgres |
| `os.path`, `open().read()` without `with` | Superseded, leaks handles | `pathlib.Path` |
