# Pristine Node TypeScript: the canon

Distilled from primary sources: Effective TypeScript 2nd ed. (Vanderkam, 2024), the
TypeScript handbook and its "Do's and Don'ts", the Google TypeScript Style Guide, the
typescript-eslint `strict-type-checked` and `stylistic-type-checked` rule sets, the
Node.js API docs and ESM guide, Clean Code and Agile Software Development (Martin) for
functions and SOLID, Kent Beck's test shapes, the Vitest guide, and Datadog's log
attribute conventions.

Contents: Types · Nullability · Functions and naming · Directory hierarchy · Layers and
separation of concerns · DRY and SOLID for modules · Async and errors · Logging ·
Collections · Node specifics · Formatting and tooling · Tests

## Types

**No escape hatches (Effective TypeScript items 5, 9, 38 to 44)**
- `any` turns the checker off for everything it touches and spreads through inference.
  Use `unknown` for values you have not looked at yet, then narrow with `typeof`,
  `instanceof`, `in`, a user-defined type guard, or a zod `parse`. Narrow at the
  boundary once; inside, everything is typed.
- The non-null assertion `!` is an unverified claim. Either narrow (`if (!x) throw new
  NotFoundError(...)`) or make the type honest (`T | undefined`). `!` on a `Map.get`
  result is the commonest offender; `??` or a guard fixes it.
- `as` casts are claims too. Allowed: `as const`, and a cast at a verified boundary with a
  one-line comment saying why the checker cannot see it. Never `as unknown as T`.
- `@ts-ignore` and `@ts-expect-error` are for library bugs, with a link to the issue.

**Shapes**
- `interface` for object shapes, `type` for unions, intersections and aliases. Be
  consistent within a module.
- Model states as discriminated unions, not bags of optional fields. `{ status: 'active',
  activatedAt: Date } | { status: 'inactive', inactivatedAt: Date }` makes the impossible
  state unrepresentable; `{ status: string; activatedAt?: Date; inactivatedAt?: Date }`
  invites it.
- Replace `enum` with an `as const` object plus a derived union:
  `const Status = { Active: 'active', Inactive: 'inactive' } as const;`
  `type Status = (typeof Status)[keyof typeof Status];`
  It erases cleanly, works with Node's type stripping, and the string values are what the
  database holds anyway.
- `readonly` on parameters and arrays you do not mutate. `ReadonlyArray<T>` in signatures
  says "I will not change this" and the compiler enforces it.
- `satisfies` checks a literal against a type without widening it: `const routes = {...}
  satisfies Record<string, Handler>` keeps the keys.
- Accept the narrowest type you read (`Pick<Property, 'id' | 'companyId'>`), return the
  most precise type you can. Never return `T | null | undefined` when `T | undefined`
  will do.

**Inference**
- Let locals infer. Annotate every exported value, every function parameter and return
  type. The return annotation is documentation and catches drift when a branch is added.
- `z.infer<typeof Schema>` is the one source of truth for boundary types; never write
  the interface and the schema separately.

## Nullability

- `undefined` is the internal "absent". `null` enters from the database and from JSON and
  is mapped to `undefined` (or a default) once, in the row mapper. Internal code never
  checks both.
- `x == null` is the one sanctioned loose equality, because it covers both. Prefer `??`
  and `?.` over it when the shape allows.
- `a?.b?.c ?? fallback` replaces a guard ladder when every missing link means the same
  thing. When one missing link means something different (it logs, it returns a
  different status), keep that one guard and chain the rest.
- `??` not `||` for defaults, unless `0`, `''` and `false` are genuinely meant to fall
  through. `||=` and `??=` for lazy initialisation.
- Optional parameters go last and get defaults in the signature. Optional fields in an
  object type mean "may be absent"; do not use them to mean "may be null".

## Functions and naming

**Size and shape (Clean Code chapter 3, restated)**
- One level of abstraction per function. An orchestrator calls named steps; a step does
  one thing. Mixing them is what makes a function need section comments.
- Around twenty lines. Not a rule to game, a signal: past it, look for the phase boundary.
- Guards first, early return, happy path at the shallowest indent. Nesting beyond two
  levels is a function waiting to be extracted.
- At most three positional parameters; past that, an options object with named fields.
  A boolean parameter that switches behaviour is two functions. The caller
  `createProperty(input, true)` tells the reader nothing; `createMultiUnitProperty(input)`
  does.
- Do not mutate arguments. Return new values; `toSorted`, `toReversed`, `with`, spread.
- Pure functions in the middle, side effects at the edges. A function that computes and
  also writes to the database is two functions.
- One shape of return. Not `Property | null | 'not-allowed'`. When the caller must
  branch on failure, return a result union with a discriminant; when it cannot recover,
  throw a typed error.

**Naming**
- Functions are verbs or verb phrases that state the effect: `loadActivePrompt`,
  `markOldCriteriaRejected`, `toPropertyRow`. `get` for cheap synchronous reads, `load`
  or `fetch` for I/O, `to`/`from` for mappers, `build` for pure construction, `ensure`
  for idempotent setup.
- Booleans are predicates: `isMultiUnit`, `hasActivePolicy`, `canEdit`, `shouldRetry`.
- Collections are plural nouns; a `Map` is named by what it indexes (`policiesByPropertyId`).
- Types and classes are PascalCase nouns. No `I` prefix, no `Impl` suffix, no `Data`,
  `Info`, `Manager` filler.
- `UPPER_SNAKE` only for true module-level constants. A `const` local is camelCase.
- No abbreviations beyond `id`, `url`, `db`, `req`, `res`, `ctx`. `prop`, `cnt`, `tmp`,
  `val`, `arr` are not names.
- A name that needs a comment to explain it is the wrong name. A comment says why, never
  what.

## Directory hierarchy

The tree should let a newcomer predict where a thing lives before opening a file.

```
src/
  app.ts                 # wires middleware and routers; no logic
  server.ts              # listens and shuts down; the only file that reads the port
  config/                # one module that parses process.env with zod at startup
  routes/                # one router per resource; maps paths to controller functions
  controllers/           # HTTP in, HTTP out: parse, validate, call a service, map errors
  services/              # business rules, orchestration, transactions
  repositories/          # SQL only; one module per aggregate; returns mapped rows
  db/                    # the Kysely instance, generated schema types, transaction helper
  domain/                # types and pure functions shared across services (no I/O)
  middleware/            # auth, error handler, request context
  graphql/               # schema, resolvers, loaders, mappers
  utils/                 # tiny, pure, dependency-free helpers; if it grows, it is a domain
tests/
  <mirror of src>        # tests/repositories/policies.repository.test.ts tests the module of that name
  db/                    # harness: migrate, truncate, fixtures
  fixtures/              # row builders; one file per table
db/
  liquibase/changelog/   # tables/, indexes/, data/
  seed/                  # local development data
docs/
  README.md              # index of every document and its status
  decisions/             # one file per design decision, numbered
scripts/                 # one-off operational scripts, each runnable with node
```

- Organise by layer at the top, by aggregate inside the layer:
  `repositories/policies.repository.ts`, not `policies/repository.ts`, until the codebase
  is large enough that feature folders pay for themselves; then move whole features,
  never half.
- One module exports one concern. A file is named for what it exports:
  `policies.repository.ts` exports policy repository functions and nothing else.
- Tests mirror source paths exactly, so the test for a file is one path substitution
  away. No `__tests__` folders beside source; they hide the shape of the test suite.
- No `index.ts` barrel files inside `src/`. They hide dependency direction, defeat tree
  shaking, and create import cycles. Import from the module directly.
- `utils/` is a smell when it has more than a handful of files. Each cluster is a domain
  module waiting to be named.
- Generated files carry `.generated.ts` in the name, live next to what generates them,
  and are never edited by hand.
- Depth of three below `src/` is the ceiling. Past that, the taxonomy is wrong.

## Layers and separation of concerns

Dependency direction is one way: route, controller, service, repository, db. Each layer
knows only the one below it.

| Layer | Knows about | Never contains |
|---|---|---|
| Route | Express, controller functions | logic of any kind |
| Controller | `req`/`res`, zod schemas, services, the error mapping | SQL, business rules, repository calls that encode a rule |
| Service | domain types, repositories, transactions, the clock, the logger | `req`/`res`, HTTP status codes, SQL |
| Repository | the Kysely instance, generated table types, row mappers | business rules, HTTP, logging beyond debug |
| Domain | nothing with I/O | imports from any other layer |

- Validation happens once, at the boundary, with zod. The service receives a typed value
  and trusts it. Re-validating inside is noise; checking a rule the schema cannot express
  (this property belongs to that company) is the service's job.
- The transaction boundary is the service. A repository function takes the executor
  (`db` or `trx`) as a parameter so it works inside or outside a transaction.
- Row mappers (`toPolicy(row): Policy`) live with the repository. The service never sees
  snake_case.
- Errors cross layers as typed classes (`NotFoundError`, `ConflictError`,
  `ValidationError`) and are mapped to HTTP in one place: the error middleware.
  Controllers do not write `res.status(404)` by hand.

## DRY and SOLID for modules

**DRY, precisely**: every piece of knowledge has one authoritative home. Two code blocks
that look alike but encode different decisions are not duplication; two that encode the
same rule are, however different they look. The signal is "would these have to change
together?" Extract on the second real copy of a rule. Do not extract on resemblance; a
premature abstraction with a flag parameter costs more than the copy.

**SOLID, in TypeScript's own terms** (modules and functions, not class hierarchies):

- *Single responsibility*: a module exports one concern and has one reason to change. A
  `properties.repository.ts` changes when the properties tables change, not when the
  HTTP API does.
- *Open/closed*: extend by adding a variant to a union, a case to a lookup table, or a
  module beside the others. The compiler's exhaustiveness check (`const _: never = x`)
  then lists every place that must learn about the new variant. That is the point.
- *Liskov*: structural typing means anything with the right shape is accepted; so a
  function that takes a `Property` must behave correctly for every valid `Property`,
  including the fields it did not expect. Narrow the parameter type to what it reads.
- *Interface segregation*: declare the dependency as the smallest type that works, at
  the point of use: `(deps: { loadPolicy: PolicyRepository['loadPolicy'] })`, not the
  whole repository. Tests then fake one function.
- *Dependency inversion*: services receive their collaborators (repository functions, the
  clock, the logger, `fetch`) as parameters or through a factory, rather than importing
  module singletons. This is what makes `vi.mock` of your own code unnecessary, and it is
  the single biggest difference between testable and untestable Node code.

## Async and errors

- `async`/`await` everywhere. `.then` chains and callbacks are for library edges only, and
  `util.promisify` or `events.once` turns those into promises.
- No floating promises. Every promise is awaited, returned, or explicitly discarded with
  `void` and a reason. `@typescript-eslint/no-floating-promises` enforces it.
- Independent work runs together with `Promise.all`. Dependent work, or work against a
  small connection pool, stays sequential; converting a sequential loop of writes to
  `Promise.all` changes ordering and can exhaust a pool of five. Say which it is.
- `Promise.allSettled` when some failures are expected and the rest must proceed.
- Errors are classes extending `Error` with a `name`, carrying `cause` when wrapping:
  `throw new RepositoryError('insert failed', { cause: err })`. Never `throw 'string'`,
  never `catch (e: any)`; `catch (e)` gives `unknown`, narrow it.
- Catch where you can act: retry, translate, or add context. Catch-log-rethrow at every
  layer produces four log lines for one failure. Let it propagate to the error middleware,
  which logs once with the request id.
- `AbortSignal` is the cancellation primitive; pass it through to `fetch` and to anything
  long-running. `AbortSignal.timeout(ms)` replaces hand-rolled timers.
- `await using` (TS 5.2, Node 24) for resources that must be released: connections,
  file handles, locks.

## Logging

Logs are structured JSON on stdout, one object per line, shipped to Datadog. A log line
is a record, not a sentence with values glued in.

- One logger, pino, created in `src/logging/logger.ts` and injected. `console.*` never
  appears in `src/`.
- Every line has a `message` field. Datadog's log explorer, patterns and monitors key on
  `message`; pino's default key is `msg`, so configure `messageKey: 'message'`. A line
  without a message is a line nobody can search for.
- The message is a short, constant, human sentence that names the event:
  `'screening extraction completed'`. Values go in fields, never into the message:
  `logger.info({ propertyId, pageCount, durationMs }, 'screening extraction completed')`.
  Interpolating ids into the message creates one unique message per request and defeats
  Datadog's pattern grouping.
- Field conventions Datadog understands without remapping: `level` as a string
  (`formatters.level: (label) => ({ level: label })`), `timestamp` in ISO 8601, `service`,
  `env` and `version` as top-level base fields, errors under `error` with `kind`,
  `message` and `stack` (map `err` to that shape with a serializer). With dd-trace's log
  injection on, `dd.trace_id` and `dd.span_id` land on every line and the trace and the
  logs link both ways.
- `requestId` on every line within a request, through a child logger created in the
  request-context middleware and passed down, not through a global.
- Log once per event, at the layer that owns it. A failure is logged by the error
  middleware, with the request id, once. Repositories log at `debug` at most. Services log
  business events at `info`. Nothing logs at `warn` unless someone should look.
- Never log secrets, tokens, full request bodies, or document text. Redact with pino's
  `redact` option at the logger, not by remembering at each call site.
- Local development uses `pino-pretty` through a transport switch on `NODE_ENV`; the
  shipped format is always the raw JSON.

## Collections

- `map`, `filter`, `flatMap`, `some`, `every`, `find` for uniform work. `for...of` when
  the body mutates locals, breaks early, or awaits sequentially.
- `forEach` that pushes into an array is a `map` or `filter`. `reduce` that builds an
  object is `Object.groupBy`, `Map.groupBy` (ES2024), or `Object.fromEntries`.
- `some` over `filter(...).length > 0`; `find` over `filter(...)[0]`; `at(-1)` over
  `[arr.length - 1]`.
- `toSorted`, `toReversed`, `with` (ES2023) over the mutating forms.
- `Map` and `Set` for keyed lookups and membership; plain objects for records with known
  keys. `Record<string, T>` is a smell when the keys are ids; use a `Map`.
- `structuredClone` for deep copies, never `JSON.parse(JSON.stringify(x))`.
- Push filtering, sorting, counting and existence into SQL when the data comes from SQL.

## Node specifics

- ESM only: `"type": "module"`, `import`/`export`, `import type` for types, no
  `require`, no `__dirname` (use `import.meta.dirname`).
- Built-ins with the `node:` prefix: `node:fs/promises`, `node:path`, `node:crypto`.
- `process.env` is read in exactly one module, parsed with zod at startup, and exported
  as a typed frozen object. A missing variable fails the boot, not the first request.
- `fetch`, `AbortController`, `structuredClone`, `Array.fromAsync` are built in. No
  `node-fetch`, `axios`, `lodash` for what the platform does.
- Health checks and graceful shutdown (`SIGTERM` closes the server, then the pool) live
  in `server.ts` and nowhere else.

## Formatting and tooling

- Prettier with the repo's config; where there is none: 2 spaces, single quotes,
  semicolons, trailing commas, 100 columns. Formatting is never discussed in review.
- ESLint flat config with `typescript-eslint` `strict-type-checked` and
  `stylistic-type-checked`, plus `no-floating-promises`, `no-misused-promises`,
  `consistent-type-imports`, `switch-exhaustiveness-check`, and `no-console`. Lint
  errors block merge.
- `tsconfig`: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `noImplicitReturns`, `noFallthroughCasesInSwitch`, `verbatimModuleSyntax`,
  `erasableSyntaxOnly`, `module` and `moduleResolution` `NodeNext`.
- Imports ordered: `node:` built-ins, external packages, internal modules, type-only
  imports last in each group; one blank line between groups. No unused imports.
- One export per concern; a default export only where a framework demands it.

## Tests

- Vitest. `describe(functionName)`, `it('returns the active prompt when one exists')`:
  the test name is the sentence the test proves.
- Arrange, act, assert, separated by blank lines. One behaviour per test; several
  `expect`s are fine when they describe one outcome.
- `it.each` for tables of inputs and expected outputs.
- Fakes over mocks. A fake is a small real implementation (`const fakeClock = () => new
  Date('2026-01-01')`); a mock asserts on calls. Assert on outcomes, not on which
  internal function was called with what.
- Never `vi.mock` a module from `src/`. If a test needs to, the code under test is
  importing a collaborator it should receive as a parameter. Fix the code.
- Mock only the network edge (OpenAI, S3) and only at the client boundary.
- Repository tests run against the real database in Docker: insert fixtures, call the
  function, assert on the result and on the table state. Truncate between tests.
- Controller tests go through the real router with `supertest` and a fake service.
- Fixture builders, one per table, return a full valid row with overridable fields:
  `buildProperty({ status: 'inactive' })`.
- Coverage thresholds are a smoke alarm for untested files. Chasing a number produces
  tests that assert nothing; a test that cannot fail is deleted.
