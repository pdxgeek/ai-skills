# Refactoring existing code (sub-skill)

Changing code that already works is a different job from writing new code. The standard
in SKILL.md still applies to the result; this file is about getting there without
changing what the code does. Read it for cleanups, modernisations, "make this a
comprehension", and reviews of someone else's diff.

Contents: Method · The moves, by smell · Worked examples · Reporting

## Method

1. **Equivalence first.** Before changing anything, write down what the old code does
   that the new code must still do: evaluation order, which awaits ran in sequence,
   side effects, distinct log lines, which exceptions escape and of what type. A rewrite
   that turns sequential writes into a `TaskGroup`, or merges two different "not found"
   branches, is a bug wearing a cleanup's clothes. When behaviour does change, verify it
   is harmless and say so in the report, or keep the original shape.
2. **Delete before you convert.** Dead branches, checks the type already proves,
   `isinstance` arms that can never match, `# type: ignore` comments the checker no
   longer needs (`warn_unused_ignores`) come out first.
3. **One move per smell.** The table below pairs each shape with the move that replaces
   it. Apply the move, run the tests, then look for the next smell.
4. **Keep the imperative form when it is honest.** A `for` whose body mutates locals,
   breaks early or awaits sequentially on purpose stays a loop. Never force a
   comprehension that needs a walrus and a nested conditional to fit.
5. **Named helpers make pipelines readable.** Extract the body of a fat lambda or a
   three-line comprehension condition into a named function with a type signature.
6. **Verify with the project's tools.** `ruff check`, the type checker and the tests
   scoped to the touched code run green before the change is called safe. Add a test
   first when the behaviour being preserved has none.

## The moves, by smell

| Smell in the code | Move |
|---|---|
| `Any`, `dict[str, Any]`, `cast(Foo, row)` on a row or JSON | pydantic model or dataclass mapper at the boundary, typed from there (example 1) |
| `assert x is not None`, `x  # type: ignore`, `Optional` unwrapped by hope | Guard that raises a typed error or returns (example 2) |
| `if a and a.b and a.b.c:` ladder; `try: x = d["k"] except KeyError: x = None` | Narrow once with a walrus guard; `d.get("k")`; keep a guard only where it does something distinct (example 3) |
| `result = []` then `for ... result.append(...)`; `reduce` building a dict | Comprehension, `dict` comprehension, `Counter`, `defaultdict`, `itertools.groupby` (example 4) |
| `for item in items: await client.send(item)` over independent items | `asyncio.TaskGroup` bounded by a `Semaphore`; keep the loop when items depend on each other or the pool is small (example 5) |
| SQL, validation and rules in one route function | Route, service, repository, with the transaction in the service (example 6) |
| `def do_thing(x, force=False)`; a status typed `str` compared to literals | Two functions or a keyword-only enum; `StrEnum` plus exhaustive `match` (example 7) |
| `@patch("app.orders.service.repo")` in a test | Inject the dependency through `__init__`; test with a fake (example 8) |
| `isinstance` chain over a closed set of classes | `match` with class patterns and `assert_never` (example 9) |
| A function with phases separated by blank lines or `# now ...` comments | Named steps at one level of abstraction (example 10) |
| `except Exception as e: logger.error(e); raise` at every layer; `except: pass` | Let it propagate; wrap with `from e` once; `contextlib.suppress` with a reason (example 11) |
| `print(f"...")`, `logger.info(f"user {uid} ...")`, `logger.info("done")` with no fields | Injected structlog logger, constant event name, values as fields (example 12) |
| `len(await repo.find_all(...)) > 0` | An `exists` query (example 13) |
| `class Config:` in a pydantic model, `.dict()`, `@validator` | `model_config = ConfigDict(...)`, `.model_dump()`, `@field_validator` |
| `session.query(X).filter(...)` | `select(X).where(...)` executed on the session |
| `os.path.join`, `open(path).read()` | `pathlib.Path`, `Path.read_text()`, or `with open(...)` |
| `datetime.now()`, `datetime.utcnow()` | `clock.now()` injected; `datetime.now(tz=UTC)` inside the clock only |
| `Optional[X]`, `List[X]`, `Union[A, B]`, `from __future__ import annotations` on 3.14 | `X | None`, `list[X]`, `A | B`; drop the future import when `requires-python >= 3.14` |
| Mutable default (`def f(items=[])`) | `items: Sequence[Item] = ()` or `None` with a guard; `field(default_factory=list)` in dataclasses |
| `requests.get(url)` in a coroutine; `time.sleep` in `async def` | `httpx.AsyncClient`; `await asyncio.sleep` |
| `asyncio.gather(*tasks)` with unchecked results; `asyncio.create_task` never awaited | `TaskGroup`; keep a reference and await it |
| `global counter`; module-level `engine = create_engine(...)` | A parameter; a `build_engine(settings)` called from the root |
| `class Foo:` holding only `@staticmethod`s | Module-level functions |
| Getter and setter methods (`get_x`, `set_x`) | Attribute or `@property`; frozen dataclass with `replace` |
| `type(x) == Foo` | `isinstance(x, Foo)` or a `match` |
| `"%s" % x`, `"{}".format(x)` | f-string (outside log calls) |
| Hand-rolled `__eq__`/`__repr__`/`__hash__` on a plain data holder | `@dataclass(frozen=True)` |

## Worked examples

Each example shows the shape as found, the move, and what to check for equivalence.

### 1. A typed boundary instead of `Any` rows

Found: `rows: list[dict[str, Any]] = cur.fetchall()` and callers doing `row["status"]`
and `cast(str, row["email"])`.

Move: a frozen `Policy` dataclass and one `_to_policy(row: Mapping[str, object]) ->
Policy` in the repository that reads each column with an explicit type check or a
`TypeAdapter`. The repository returns `list[Policy]`. Every `cast` downstream disappears.

Check: a column that was sometimes `NULL` becomes `str | None` in the dataclass, and the
checker now lists every caller that assumed it was present.

### 2. Narrow or raise instead of `assert`

Found: `order = await repo.find(oid); assert order is not None; order.close()`.

Move: `if order is None: raise OrderNotFoundError(oid)`. The exception handler maps it
to a 404 problem.

Check: under `python -O` the old `assert` vanished and the code raised `AttributeError`
later with a 500; the new code is a 404. That is a behaviour change, and the right one.
Say so.

### 3. Collapse a guard ladder, keep the guard that differs

Found:
```python
if payload is None:
    return None
if payload.get("tenant") is None:
    log.warning("no tenant")
    return None
if payload["tenant"].get("email") is None:
    return None
return payload["tenant"]["email"]
```

Move: after pydantic parses `payload` into `Payload` with `tenant: Tenant | None` and
`Tenant.email: str | None`, the body is a guard that logs, then `return
payload.tenant.email`. The warning log is a distinct behaviour and stays.

Check: the logged warning still fires in the same case and only that case.

### 4. Comprehensions, not accumulators

Found:
```python
by_property = {}
for p in policies:
    if p.active:
        by_property.setdefault(p.property_id, []).append(p)
```

Move: `by_property = defaultdict(list); for p in filter(is_active, policies):
by_property[p.property_id].append(p)`, or when the input is sorted by property id,
`itertools.groupby`. For a flat transform, a comprehension:
`active_ids = [p.id for p in policies if p.active]`.

Check: iteration order of the dict is insertion order in both; a `set` would not be.

### 5. Concurrent when independent, sequential when not

Found: `for prop in properties: result = await client.sync(prop); results.append(result)`.

Move: when the calls are independent and the downstream can take it:
```python
sem = asyncio.Semaphore(8)
async def one(prop: Property) -> SyncResult:
    async with sem:
        return await client.sync(prop)
async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(one(p)) for p in properties]
results = [t.result() for t in tasks]
```
Keep the loop when each call depends on the previous one, when the order of side effects
matters, or when the downstream is rate limited and the sequential form was the limiter.

Check: the old loop stopped at the first failure and left later items untouched; the
`TaskGroup` cancels siblings and raises an `ExceptionGroup`. Decide which the use case
needs and say which you kept.

### 6. Separate the concerns a route had gathered

Found: a route function that reads the body as a dict, checks fields by hand, runs
`session.execute(text(...))` twice, decides whether the order can close, commits, and
returns `{"ok": True}`.

Move: a pydantic `CloseOrderRequest` in the signature; `OrderService.close(order_id,
reason)` holding the rule and `async with session.begin()`; `OrderRepository.find` and
`.save` holding the statements; a `CloseOrderResponse` schema. The route becomes three
lines.

Check: the old code committed between the two statements; the new code commits once.
Confirm no reader depended on seeing the intermediate state.

### 7. Two functions instead of a flag; an enum instead of a string

Found: `def notify(user, urgent=False)` with `if urgent:` branching twice, and
`status: str` compared to `"active"` in six places.

Move: `notify(user)` and `notify_urgently(user)`, or `notify(user, *, priority:
Priority)` with a `StrEnum` and a `match`. `class Status(StrEnum)` replaces the string;
the six comparisons become one `match` or `status is Status.ACTIVE`.

Check: an unknown status used to fall through silently; `assert_never` now makes it a
type error, and a runtime `ValueError` at the `StrEnum` constructor. Say so.

### 8. Inject the dependency instead of patching the module

Found: `@patch("app.orders.service.OrderRepository")` in five tests, each stubbing
`.find.return_value`.

Move: `OrderService.__init__(self, repo: OrderRepository, ...)`; a `FakeOrderRepository`
with a dict in `tests/fakes.py`. Each test does `service = OrderService(repo=fake,
clock=FakeClock(...))`. The patches and the `return_value` ladders go away.

Check: a patch that stubbed a method the fake does not implement reveals a call the
`Protocol` never declared. That is a finding, not noise.

### 9. A match instead of an isinstance chain

Found:
```python
if isinstance(event, Created): ...
elif isinstance(event, Shipped): ...
elif isinstance(event, Cancelled): ...
```

Move: `type Event = Created | Shipped | Cancelled` and
```python
match event:
    case Created(order_id=oid): ...
    case Shipped(tracking=t): ...
    case Cancelled(reason=r): ...
    case _: assert_never(event)
```

Check: the old chain silently did nothing for an unknown subclass; the new one is a type
error at the `assert_never` when a variant is added to the union.

### 10. One level of abstraction

Found: a forty-line `process_upload` with phases marked `# parse`, `# validate`, `# save`,
`# notify`.

Move: `parsed = _parse(upload)`, `_validate(parsed)`, `saved = await
_save(parsed)`, `await _notify(saved)`; the orchestrator is four lines and each step has
its own signature and test.

Check: a local assigned in phase two and read in phase four is a parameter now, which
the extraction makes visible.

### 11. Let errors travel

Found: every layer wraps the call in `try/except Exception as e: logger.error(str(e));
raise`, and one place does `except Exception: pass`.

Move: delete the catch-log-reraise blocks; the exception handler logs once with
`exc_info`. Where a wrap adds meaning, `raise PaymentFailedError(invoice_id) from e`.
The `pass` becomes `with contextlib.suppress(FileNotFoundError):` with a comment, or a
real handler.

Check: the error used to be logged four times with four partial messages; now once with
the full traceback. The suppressed exception type is now explicit; anything else that
used to be swallowed will surface, which is the point.

### 12. A log line is an event with fields

Found: `logger.info(f"Synced {count} properties for {landlord.name} in {elapsed}s")`.

Move: `log.info("properties_synced", count=count, landlord_id=landlord.id,
duration_ms=int(elapsed * 1000))`. The landlord's name was personal data in a log and is
gone; the id is enough to join.

Check: the dashboard that searched the old message text needs its query updated. Say so.

### 13. Existence belongs in the query

Found: `if len(await repo.find_by_property(pid)) > 0:` loading every row to test for one.

Move: `if await repo.exists_for_property(pid):` with `select(exists().where(...))`.

Check: the old code also triggered any lazy loads on the rows; the new code loads
nothing else, which is the gain.

## Reporting

For each move, name the smell, the move, and what you checked for equivalence. When a
move changed observable behaviour (an exception type, a status code, a log line, the
order of side effects), that is the headline. Name the tests that ran and whether any
were added. When a smell was left in place because fixing it would widen the change, say
which and why.
