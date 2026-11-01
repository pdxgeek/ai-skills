# Modern Java best-practices canon

Distilled from primary sources: Effective Java 3rd ed. Items 45-48 and 55 (Bloch),
Stuart Marks' "Optional - The Mother of All Bikesheds" (Devoxx BE 2016), Brian Goetz's
records and data-oriented programming articles (InfoQ), the Oracle Java 17 records
chapter, and the OpenJDK "Local Variable Type Inference Style Guidelines" (Marks).

## Streams

**When streams, when loops (EJ 45)**
- Use streams for uniform work over sequences: map, filter, accumulate, group, search.
- Keep a loop when the body must mutate locals, break/continue/return the enclosing
  flow, or throw checked exceptions; lambdas can do none of these.
- Once you `map`, the original element is gone; when a later stage needs both the
  original and the derived value, use a local record to carry the pair, or keep the loop.
- Convert a loop only when the stream version is actually clearer. When in doubt, write
  both shapes mentally and pick the more readable one.
- Extract multi-line lambda bodies into named helper methods. Names matter more in a
  pipeline because there are no intermediate variable names to carry meaning.

**Purity and forEach (EJ 46)**
- Every function passed to a stream operation should be pure. Stateful lambdas defeat
  correctness and readability.
- `forEach` that does more than present already-computed results is a smell; the classic
  misuse is accumulating into a collection inside `forEach` instead of using a collector
  (`collect(groupingBy(identity(), counting()))`).
- Prefer `anyMatch`/`allMatch`/`noneMatch` over `filter(...).count() > 0` or
  `findFirst().isPresent()`; they short-circuit and state intent.
- When the source is a database, push filtering and existence down: a derived
  `existsBy...` repository method beats fetching rows to test emptiness.

**Collectors**
- Know `toMap` (with a merge function), `groupingBy` with downstream collectors,
  `joining`, and `teeing` (two aggregates in one pass, Java 12).
- Prefer `stream.toList()` (Java 16) over `collect(Collectors.toList())`; note it
  returns an unmodifiable list, so it is a behavior change if callers mutate the result.
  Use `Collectors.toCollection(ArrayList::new)` when mutability is required.
- `mapMulti` (Java 16) over `flatMap` when each element expands to a few (possibly zero)
  elements and building a stream per element is waste.

**Parallel (EJ 48)**
- Never `.parallel()` by default. Only with a measured speedup, a splittable source
  (ArrayList, arrays, ranges), a cheap reduction terminal, and pure functions. Never
  with `Stream.iterate` or `limit`. All parallel streams share one ForkJoinPool.

**Comparators**
- Compose declaratively: `comparing(...).thenComparing(...).reversed()`, with
  `comparingInt/Long` to avoid boxing. Add a final unique key (`thenComparing(User::getId)`)
  when a pick must be deterministic. Wrap with `Comparator.nullsFirst/nullsLast` instead
  of null-checking inside the lambda.
- `min`/`max` with a comparator replaces sort-then-take-first.

## Optional

Design intent (Marks, Goetz): a return-type mechanism for "no result" where null causes
errors. Eliminating every null is explicitly not the goal.

**The Marks rules**
1. Never use null for an Optional variable or return value.
2. Never call `get()`; since Java 10 write `orElseThrow()` when presence is proven, so
   the failure mode is explicit.
3. Prefer the alternative APIs over `isPresent()` + `get()`; that pair is just a null
   check wearing a costume.
4. Don't create an Optional just to chain one step to a value:
   `Optional.ofNullable(s).orElseGet(this::getDefault)` is a ternary with an allocation.
   Multi-step chains (map/filter/flatMap doing real work) are the intended style; the
   rule bans the single-step wrap-then-unwrap, not chains.
5. A nested chain or an `Optional<Optional<T>>` intermediate means it is too complex;
   unnest or use an if.
6. No Optional fields, parameters, or collection elements. Fields cost an allocation and
   a dependent load each; parameters force every call site to wrap; return the empty
   collection instead of `Optional<List<T>>`.
7. No identity-sensitive operations (`==`, locking) on Optionals; it is a value-based
   class.

**API patterns**
- `map`/`filter`/`flatMap` off an Optional you were given; `or(() -> fallback)` for a
  lazy alternative Optional (and it can carry a branch-specific side effect like a warn
  log); `ifPresent`/`ifPresentOrElse` for actions.
- `orElse(constant)` only for cheap constants; `orElseGet(supplier)` when the default is
  computed, because `orElse`'s argument is evaluated even when the value is present.
- `orElseThrow(MyException::new)` with a factory, so nothing is built on the happy path.
- A stream of Optionals flattens with `.flatMap(Optional::stream)`.
- Boundaries: nullable-in with `Optional.ofNullable(x)`; nullable-out with
  `orElse(null)`, which is the only sanctioned use of `orElse(null)`.
- `map` turns a null mapper result into empty; do not pre-guard for it.

**Return types (EJ 55)**
- Return `Optional<T>` when absence is a normal case the caller must handle. Never
  return null from an Optional-returning method. Never wrap containers. Use
  `OptionalInt/Long/Double` for primitives. Not as map keys or values.

## Records

Intent (Goetz): transparent carriers for immutable data; the API is the state. Derived
`equals`/`hashCode`/`toString` cover exactly the declared components.

- Reach for a record for compound keys, multi-value returns, DTOs, messages, and
  intermediate results. Keep helper records private and nested (or local to a method)
  when only one class cares; they are cheap enough to be single-use.
- Validate in the compact constructor; normalize by reassigning the constructor
  parameters there. All non-canonical constructors funnel through the canonical one.
- Records are shallowly immutable: defensively copy mutable components
  (`this.items = List.copyOf(items)`) or the equals/map-key contract breaks.
- Local records inside a method name the intermediate result of a pipeline (compute a
  derived value, sort on it, project it away); this is the Oracle-documented idiom.
- Do not use records for JPA entities (they need no-arg constructors, mutability, proxy
  identity); records are excellent as JPA projections. Jackson handles records from
  2.12. Java serialization deserializes through the canonical constructor, so the
  validation runs.
- Data-oriented shape: sealed interface + records as the alternatives, behavior in
  pattern-matching switches; illegal states become unrepresentable.

## var

The OpenJDK style guide's principles: readers over writers; the declaration must be
understandable from local context, without an IDE; the explicit type is one information
channel and may be muted only when the name and initializer carry the load.

- Use `var` when the initializer states the type: constructor calls, well-named
  factories (`var reader = Files.newBufferedReader(path)`), literals.
- Use `var` to break a mega-chain into named intermediate steps; unpronounceable generic
  types (`Optional<Map.Entry<String, Long>>`) are what push people into unreadable
  one-liners in the first place.
- Pair `var` with a better variable name; the name does the work the type used to.
- Keep scopes small first; `var` in a long scope hides a changed initializer type.
- Diamond and generic methods: `var q = new PriorityQueue<>()` infers `Object`; supply
  the type argument or skip `var`.
- Numeric literals: `var flags = 0` is int, and float fields infer float; the silent
  widening an explicit type performed disappears.
- Locals only; fields, parameters, and returns keep explicit types, where programming
  to the interface still fully applies.
