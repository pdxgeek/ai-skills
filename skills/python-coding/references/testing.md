# Testing

Tests prove behaviour through the public surface, use fakes passed in rather than patched
modules, and run the database and nothing else for real in a container. Sources: the
pytest docs (fixtures, parametrize, markers), pytest-asyncio, Testcontainers for Python,
respx, Hypothesis, the FastAPI testing guide, "Architecture Patterns with Python"
(Percival and Gregory).

Contents: Version notes · Shape and naming · Fixtures and parametrize · What to fake and
what to run for real · Config · Databases · HTTP edges · Time and randomness · Async
tests · Property tests · Type tests · The five outcomes

## Version notes

- pytest 8: `pytest.approx`, `pytest.raises(..., match=)`, `tmp_path`, `monkeypatch`,
  `caplog` are the standard fixtures. `setup.cfg`/`pytest.ini` give way to
  `[tool.pytest.ini_options]` in `pyproject.toml`.
- pytest-asyncio 0.23+: set `asyncio_mode = "auto"` so `async def test_*` just works;
  set `asyncio_default_fixture_loop_scope` explicitly to silence the deprecation
  warning. Or use `anyio`'s pytest plugin when the code is anyio-based.
- FastAPI: `httpx.AsyncClient(transport=ASGITransport(app=app), base_url="http://test")`
  replaces the sync `TestClient` for async apps; `TestClient` still works for sync.
- Testcontainers for Python 4: `PostgresContainer("postgres:17")` as a session fixture;
  `get_connection_url()` gives the DSN.
- Hypothesis 6: `@given` with strategies; `st.builds(FrozenDataclass)` for domain
  values.

## Shape and naming

- `test_<module>.py` beside the module or under `tests/` mirroring `src/`; one per repo.
- One behaviour per test, named as the behaviour it proves:
  `test_closes_order_when_balance_is_zero`, not `test_close` or `test_1`. No `unittest`
  classes; a `class TestOrderService:` grouping is fine when it has no `setUp`.
- Arrange, act, assert, separated by blank lines. No logic in tests: a `for` with
  asserts stops at the first failure and hides which input broke.
- Plain `assert` with pytest's rewriting; `pytest.raises(OrderNotFoundError, match=...)`
  for the exact statement that raises, never try/`pytest.fail`.
- `pytest.approx` for floats; compare `Decimal` exactly.
- Compare whole dataclasses, not field by field, so a new field is covered for free.
- A test that cannot fail is deleted. Coverage is a floor that catches untested files,
  not a target.

## Fixtures and parametrize

- Fixtures build the arrange step and nothing else. Scope them honestly: `function`
  for anything mutable, `session` for the database container and settings.
- Factories over fixtures that return one fixed object: a `make_order(**overrides)`
  function (or `factory_boy`) builds a valid order with named deviations, so each test
  states only what matters to it.
- `@pytest.mark.parametrize("amount, expected", [...], ids=[...])` over copy-pasted
  tests; `ids` so the failing case is readable.
- `conftest.py` holds shared fixtures; it never holds assertions or business logic.
- `monkeypatch` for environment variables and `sys.path`, never for the project's own
  modules.

## What to fake and what to run for real

- **Fake, through the constructor**: repositories (an in-memory dict keyed by id),
  clocks, random, outbound clients, message publishers. A fake is a small class that
  satisfies the `Protocol`; it lives in `tests/fakes.py` or beside the test.
- **Never** `unittest.mock.patch("package.module.thing")` on code under `src/`. Patching
  a module couples the test to an import path, hides that the dependency was never
  injected, and keeps passing when the real wiring is broken. The fix is to inject.
- **Mock sparingly**: `MagicMock` only for a third-party object with a wide surface
  that a two-line fake cannot stand in for, with `spec=` or `autospec=True` so a
  renamed method fails. `AsyncMock` for async collaborators. Assert on outcomes, not on
  `assert_called_with` ladders that restate the implementation.
- Do not mock what you do not own: `datetime`, `httpx`, the ORM session, `pathlib`.
  Wrap them behind your own `Protocol` or run the real thing.
- Do not mock value objects; build them.
- Services are tested with plain constructor calls and fakes, no app, no database.
  Repositories are tested against the real database. Routes are tested through the ASGI
  app with the service faked, proving parsing, status codes and problem-details
  mapping. A few end-to-end tests run the real graph against the container.

## Config

- A `settings` fixture builds `Settings(...)` explicitly with test values; never read a
  developer's `.env` in tests. `Settings.model_construct` to skip validation only when
  the test is about something else and says so.
- `_env_file=None` in the test constructor so a stray `.env` cannot leak in.

## Databases

- Testcontainers `PostgresContainer` as a session fixture; Alembic migrations run once
  against it. No SQLite stand-in: the dialect differs in exactly the places that bite
  (`RETURNING`, `JSONB`, constraints, timestamps).
- Each test runs inside a transaction that is rolled back, or truncates the touched
  tables in a fixture; never relies on test order.
- Repository tests assert on what came back from the database (round trip), not on the
  SQL string.
- Unique data per test (`uuid4()` in the email) when truncation is too slow, so tests
  can run in parallel with `pytest-xdist`.

## HTTP edges

- Outbound: `respx.mock(assert_all_called=True, assert_all_mocked=True)` so an
  unexpected request fails the test and a declared route that was never hit fails too.
  Match on method, URL and body; return the real payload shape from a fixture file.
- Inbound: `httpx.AsyncClient` over `ASGITransport(app=create_app(settings, fake_deps))`.
  Assert status, `content-type: application/problem+json` on errors, and the parsed
  body against the response schema.
- Never hit the network in a test; CI has none.

## Time and randomness

- Inject a `Clock` and a `random.Random(seed)`; the fake clock has `advance(seconds)`.
  `freezegun` or `time_machine` only for third-party code that calls `datetime.now()`
  itself.
- Timeouts and retries are tested by faking the clock or the sleep function, never by
  sleeping.

## Async tests

- `async def test_*` with `asyncio_mode = "auto"`; async fixtures with
  `@pytest_asyncio.fixture` (or anyio's `@pytest.fixture` under its plugin).
- One event loop scope per repo, declared in `pyproject.toml`; a session-scoped engine
  fixture needs a session-scoped loop.
- `TaskGroup` behaviour is tested by injecting a fake that fails for one item and
  asserting the `ExceptionGroup` arm handles it.

## Property tests

- Hypothesis for parsers, serialisers, money arithmetic and anything with an invariant
  that holds for all inputs (`decode(encode(x)) == x`). `st.builds` on frozen
  dataclasses, `st.from_type` on pydantic models through `hypothesis-jsonschema` or
  pydantic's own strategy.
- Keep the example database (`.hypothesis/`) out of git; set `deadline=None` on tests
  that touch the container.

## Type tests

- The type checker is the test for types; `assert_type(x, Order)` in a test module
  pins a type the checker must infer. `# type: ignore` in a test needs the same
  justification as in `src/`.
- `typing.assert_never` in `match` fallthroughs makes a forgotten variant a type error,
  which is cheaper than any runtime test.

## The five outcomes

Every unit under test has five kinds of behaviour, and a good suite touches each: the
happy path; the empty or boundary input (`[]`, `None`, `0`, the max length); the
rejected input (the typed error raised, the 4xx returned); the collaborator failure
(the fake raises, the client times out); and the state after (what was saved, published
or logged, checked through the fake). A suite with only happy paths tests the demo, not
the product.
