# Style: types, data, functions, errors and async

How a Python function is written so it reads once and type checks forever. Sources: PEP 8,
PEP 484/526/544/586/604/612/634/695/696/750, the typing documentation, the mypy and
pyright strict-mode docs, the pydantic 2 docs, the asyncio documentation, the ruff rule
set, "Fluent Python" (Ramalho), "Effective Python" (Slatkin).

Contents: Version notes · Types · Type-level idioms · None · Data · Functions · Naming ·
Collections and comprehensions · Pattern matching · Boundaries and validation · Errors ·
Async · Performance

## Version notes (check `requires-python` before using an API)

- **3.12**: PEP 695 `class Box[T]:` / `def f[T](x: T)` / `type Alias = ...`; `@override`
  from `typing`; f-strings may nest the same quote; `itertools.batched`; `pathlib.Path.walk`.
- **3.13**: `typing.TypeIs` (narrows both arms, prefer over `TypeGuard`);
  `typing.ReadOnly` for TypedDict; `warnings.deprecated`; PEP 696 type-parameter
  defaults; free-threaded and JIT builds are experimental; `locals()` semantics fixed.
- **3.14**: PEP 649/749 deferred annotations, so `from __future__ import annotations`
  is unnecessary for forward references and `annotationlib` reads them; PEP 750 template
  strings (`t"..."`); PEP 758 `except A, B:` without parentheses; free threading is
  supported (PEP 779); `asyncio` introspection tools.
- **pydantic 2**: `model_validate`, `model_dump`, `ConfigDict`, `field_validator`,
  `model_validator`, `TypeAdapter`. The v1 names (`parse_obj`, `.dict()`, `class
  Config`, `@validator`) are deprecated shims; never write them in new code.
- **SQLAlchemy 2.0**: `Mapped[int]` + `mapped_column`, `select()` everywhere,
  `Session.execute(stmt).scalars()`. Legacy `Query` (`session.query(X)`) is not written
  in new code.
- **mypy**: `--strict`; `disallow_any_explicit` where the repo turns it on. **pyright**:
  `typeCheckingMode = "strict"`. One of the two, configured in `pyproject.toml`, run in
  CI.

## Types

- Annotate every parameter and return of every `def`, including `-> None`. Locals only
  when inference fails or the annotation documents a non-obvious shape.
- Builtin generics (`list[str]`, `dict[str, int]`, `tuple[int, ...]`), `X | None`, and
  `collections.abc` for abstract parameters. `typing.List`, `Optional`, `Union`,
  `Dict` are legacy.
- Accept the widest type that works, return the narrowest: a parameter is `Sequence[Item]`
  or `Iterable[Item]`, the return is `list[Item]` or `tuple[Item, ...]`. A parameter typed
  `list` forbids passing a tuple for no reason.
- `object`, not `Any`, for "I do not know yet". A function that returns `Any` poisons
  every caller; wrap third-party `Any` at the boundary and return a real type.
- Narrowing: `isinstance`, `is None`, `in`, a `match`, or a `TypeIs` function.
  `TypeIs[Foo]` narrows both branches; `TypeGuard[Foo]` narrows only the true branch
  and is for the cases `TypeIs` cannot express.
- `Final` for module constants and attributes assigned once; `ClassVar` for class-level
  state. `@final` on a class not designed for subclassing.
- `@override` (3.12) on every overriding method so a renamed base method breaks the
  build.
- `Protocol` for structural typing: the consumer declares the methods it needs; any
  class with those methods satisfies it without inheriting. `@runtime_checkable` only
  when an `isinstance` check against it is actually written.
- `ParamSpec` and `Concatenate` for decorators that keep the wrapped signature;
  `functools.wraps` always. A decorator typed `Callable[..., Any]` erases the signature
  of everything it touches.
- `TypedDict` for a JSON-shaped dict that must stay a dict (a third-party SDK demands
  it). Everywhere else a frozen dataclass or a pydantic model.
- `NewType` for ids. `UserId = NewType("UserId", int)`; construct `UserId(row.id)` in
  the repository, and the checker refuses `OrderId` where `UserId` is expected.

## Type-level idioms

- `Literal["active", "closed"]` for a small closed set used in one module; `StrEnum`
  when it crosses a wire or is reused.
- Discriminated unions: each variant a frozen dataclass with `kind: Literal["x"]`, the
  union a `type` alias, consumed with `match` and `assert_never`.
- `Self` for fluent methods and alternate constructors (`def with_x(self, x) -> Self`).
- `LiteralString` for SQL fragment parameters so a formatted string cannot be passed
  where a fixed query is expected.
- `Unpack[Kwargs]` with a `TypedDict` for a function that forwards `**kwargs`.
- `Never`/`NoReturn` on functions that always raise; `assert_never` in the fallthrough
  arm of a `match`.
- Generics only when a type parameter relates two positions; one used once is `object`
  in disguise.

## None

- `T | None` is the honest "may be absent". Check with `is None` / `is not None`, never
  truthiness, when the type has falsy legitimate values (`0`, `""`, `[]`, `Decimal(0)`).
- Narrow once, at the top: `if order is None: raise OrderNotFound(order_id)`. The rest
  of the body is typed `Order`.
- `assert x is not None` is a test idiom; in `src/` it is stripped under `-O` and reads
  as a shrug. Raise a typed error or return.
- Walrus to narrow and bind: `if (user := find_user(uid)) is not None:`.
- Default arguments are never `None` standing in for a mutable default when a real
  sentinel is needed; `field(default_factory=...)` on dataclasses, a module-level
  `_MISSING = object()` sentinel when `None` is a legal value.

## Data

- `@dataclass(frozen=True, slots=True, kw_only=True)` is the default internal value
  type. `frozen` gives hashing and stops accidental mutation, `slots` halves the memory
  and blocks typo attributes, `kw_only` keeps call sites readable past two fields.
- `__post_init__` validates invariants and raises `ValueError`; it never mutates (use
  `object.__setattr__` only to normalise in a frozen class, with a comment).
- `dataclasses.replace(order, status="closed")` for a changed copy; never mutate.
- `NamedTuple` only when tuple unpacking at call sites is the point (a coordinate, a
  key/value pair).
- `enum.StrEnum` (3.11) for sets that serialise; `enum.Enum` with `auto()` otherwise;
  `enum.Flag` for bit sets. Never compare an enum to its `.value` by hand; let `StrEnum`
  do it.
- `Decimal` for money, constructed from `str` or `int`, never from `float`.
  `datetime` always aware; `datetime.now(tz=UTC)` only inside the injected clock.
  `zoneinfo.ZoneInfo`, never `pytz`.
- pydantic models at boundaries only. `ConfigDict(frozen=True, extra="forbid",
  strict=True)` where the data has a contract. Map to a dataclass before it crosses into
  a service, or keep the pydantic type as the DTO and never let a service import a
  route's schema module.

## Functions

- One level of abstraction: a function orchestrates named steps or is one step. Blank
  lines separating phases, or a comment saying `# now save`, are the signal to extract.
- Guards first (`if not items: return []`), early return, happy path at the shallowest
  indent. Nesting beyond two levels is a seam.
- Parameters: at most three positional; the rest keyword-only after `*`. Positional-only
  (`/`) for a parameter whose name is an implementation detail. A boolean that switches
  behaviour is two functions or an enum. Never a mutable default.
- No argument mutation. Return a new value; `sorted()` over `.sort()` on a parameter.
- One shape of return. `T | None` for "may be absent", `raise` for "cannot continue", a
  result union when the caller branches on several expected outcomes. Never
  `return None` on one path and `return []` on another for the same meaning.
- Generators for streams of unknown or large size; `yield from` to delegate. A generator
  that reads a file owns its `with`.
- `functools.cache` only on pure functions with hashable arguments and bounded argument
  space; `lru_cache(maxsize=...)` otherwise. Never on methods (it caches `self`).
- `functools.partial` over a lambda that only fixes arguments. A lambda longer than one
  expression is a named function.
- Type-driven dispatch with `match` or `functools.singledispatch`, never an `isinstance`
  ladder.

## Naming

- `snake_case` functions, methods, variables, modules, packages. `PascalCase` classes,
  exceptions, type aliases, enums. `UPPER_SNAKE` module constants. `_private` for
  module- and class-private; never `__mangled` outside a mixin that needs it.
- Verbs for functions that do (`send_invoice`, `load_policy`), nouns for properties and
  pure accessors, predicates for booleans (`is_`, `has_`, `can_`, `should_`).
- Plural for collections; `items_by_key` for a dict; `item_count` for an int, never
  `num_items` or `n`.
- Modules are nouns (`repository.py`, `schemas.py`), not `utils.py` or `helpers.py`.
- Exceptions end in `Error` (`PolicyNotFoundError`), except a per-context base that is
  a noun (`BillingError`).
- Single-character names only for `i`, `j` in short index loops and `e` in `except`.

## Collections and comprehensions

- A comprehension for one transform or filter; a loop when the body is several
  statements, mutates locals, or breaks early. A nested comprehension that needs a
  second read is a loop or a named helper.
- Generator expression when the result is consumed once (`sum(x.total for x in orders)`);
  list when it is indexed or reused.
- `any()`/`all()` over `len([...]) > 0`. `next(iter(items), None)` for first-or-none.
  `collections.Counter`, `defaultdict`, `itertools.groupby` (on sorted input),
  `itertools.batched` (3.12), `more_itertools` only when the standard library lacks it.
- Dict comprehension for an index: `{p.property_id: p for p in policies}`. `dict.get`
  with a default over `if key in d` then `d[key]`. `setdefault` or `defaultdict` for
  grouping.
- `enumerate`, `zip(strict=True)`, `reversed`, never index arithmetic.
- `tuple` for a fixed heterogeneous record (or a dataclass), `list` for a homogeneous
  sequence, `set`/`frozenset` for membership, `dict` for a mapping. Return `tuple[Item,
  ...]` or `Sequence[Item]` when the caller must not mutate.
- Push filtering, counting and existence to the database: `SELECT EXISTS` over fetching
  rows and `len()`.

## Pattern matching

- `match` for a discriminated union, a parsed command, or a shape with several fields to
  destructure. `if`/`elif` for one or two boolean conditions.
- Class patterns with keyword fields: `case Shipped(tracking=tracking, carrier="ups"):`.
  Mapping and sequence patterns at JSON boundaries only after pydantic has parsed.
- Every `match` over a closed set ends with `case _: assert_never(value)`; one over an
  open set ends with a `case _:` that raises or returns the explicit default.
- Guards (`case X(amount=a) if a > limit:`) for a condition the pattern cannot express;
  a guard that duplicates a pattern is two cases.
- A bare name in a `case` is a capture, not a constant. Constants are dotted
  (`case Status.ACTIVE:`) or the match is on a `Literal`-typed enum.

## Boundaries and validation

- Parse, do not validate. The pydantic model is the type; `model_validate(payload)`
  returns it or raises `ValidationError`. There is no separate hand-written
  `TypedDict`.
- `field_validator` and `model_validator` for invariants that cross fields;
  `Annotated[int, Field(gt=0)]` for constraints on one field. `TypeAdapter` for
  validating a bare `list[Model]` or a scalar.
- `strict=True` where the input has a contract (an internal queue); lax mode at a
  public HTTP edge where `"42"` for an int is tolerated.
- Env and config through `pydantic_settings.BaseSettings`, parsed once at startup.
- Raw SQL rows map to a dataclass in the repository with explicit column types; no
  dict of `Any` leaves the repository.
- Never trust `isinstance(x, dict)` as validation; it proves nothing about the keys.

## Errors

- Exceptions are the error path; a result union is for several expected outcomes the
  caller must branch on. Never return error codes, `None` meaning "failed", or
  `(ok, value)` tuples.
- One base per bounded context: `class BillingError(Exception)`. Leaf classes only when
  a caller catches them specifically: `class InvoiceNotFoundError(BillingError)`. Carry
  structured fields (`self.invoice_id = invoice_id`) and build the message in
  `__init__`; never parse messages.
- `raise NewError(...) from e` when wrapping; `from None` only to deliberately hide a
  cause, with a comment. Never lose the cause.
- Catch the narrowest type that the handler can act on. `except Exception` only at the
  boundary that logs and converts (the HTTP exception handler, the job runner, `main`).
  Bare `except:` never (it catches `KeyboardInterrupt` and `SystemExit`).
- Never catch-log-reraise at every layer; the boundary logs once with `exc_info=True`.
  Never `except: pass`; an intentional ignore uses `contextlib.suppress(SpecificError)`
  with a comment saying why it is safe.
- `try` wraps the one statement that can raise, not the whole function. The `else`
  clause holds the code that runs only when nothing raised. `finally` for cleanup only
  when a context manager cannot express it.
- Exception groups (`except*`) when a `TaskGroup` can raise several; handle each type
  in its own `except*` arm.
- Standard exceptions for programmer errors: `ValueError` for a bad argument value,
  `TypeError` for a bad type, `KeyError`/`IndexError` from containers,
  `NotImplementedError` in an abstract method. Never raise `Exception` or
  `RuntimeError` for a domain condition.

## Async

- Pick one world per process: a FastAPI service is `async def` from route to driver
  (`asyncpg`, `psycopg` async, `httpx.AsyncClient`); a CLI or batch job is sync. Mixing
  blocks the event loop.
- Never block inside a coroutine: no `time.sleep`, `requests`, sync ORM session,
  `open().read()` of a large file, CPU-heavy loop. `asyncio.to_thread` for the one
  unavoidable blocking call; `ProcessPoolExecutor` for CPU work.
- `asyncio.TaskGroup` (3.11) for concurrent independent work; it cancels the siblings on
  the first failure and raises an `ExceptionGroup`. `asyncio.gather` only when partial
  results on failure are the point, with `return_exceptions=True` and each result
  checked.
- Bound concurrency over user-sized input with `asyncio.Semaphore`; the database pool
  and the downstream API are the choke points.
- `async with asyncio.timeout(seconds):` for deadlines; never `wait_for` wrapping a
  shared resource (cancellation can leak the connection).
- Every created `Task` is awaited or kept in a set with a done callback; a dropped
  `Task` is garbage collected mid-flight with no error.
- `async for` and async generators for streams; `contextlib.asynccontextmanager` for
  async resources; `anyio` when the code must run on both asyncio and trio.
- Cancellation is cooperative: catch `CancelledError` only to clean up, then re-raise.

## Performance

- Measure before optimising: `cProfile`, `py-spy`, `timeit`. A guess about the hot
  spot is usually wrong.
- Database round trips dominate: fetch what the use case needs in one statement with
  explicit `selectinload`/`joinedload`, never a lazy load inside a loop. `EXISTS`,
  `COUNT`, and aggregate in SQL.
- Generators and `itertools` over materialised intermediate lists on large data.
  `str.join` over `+=` in a loop. Sets for membership tests. `__slots__` (via
  `slots=True`) on classes created in bulk.
- `orjson` or `msgspec` only when JSON encoding is measured as the bottleneck.
- `functools.cache` on pure, bounded functions; a TTL cache from a library when the
  value goes stale, never a hand-rolled dict with timestamps.
