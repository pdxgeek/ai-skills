---
name: typescript-coding
description: Write, review and refactor TypeScript on Node to a textbook standard - strict types with no any or non-null assertions, small single-purpose functions, intention-revealing names, layered separation of concerns (route, service, repository), DRY and SOLID applied to modules, async/await done right, and Vitest tests that exercise real behaviour instead of mocking the codebase. Use whenever writing, reviewing, or cleaning up TypeScript, JavaScript or Node code - new modules, Express or GraphQL handlers, repository functions, scripts, tests, or any diff showing any, as casts, the ! operator, forEach pushing into arrays, await inside loops, SQL or business rules in a controller, boolean flag parameters, copy-pasted validation blocks, or jest.mock of the project's own modules. Trigger on requests like "clean this up", "make this idiomatic", "modernize this", "is this good TypeScript", "looks sus", "review this", or "write a service/repository/handler for X", even when the user does not say TypeScript or Node.
---

# Pristine Node TypeScript

Produce TypeScript you would put in a textbook: correct, strict, small, named well,
layered, and tested. The two reference files carry the depth; this file is the method.

- `references/best-practices.md`: the distilled canon (Effective TypeScript 2nd ed.,
  the TypeScript handbook's Do's and Don'ts, the Google TypeScript Style Guide,
  typescript-eslint strict-type-checked, the Node.js docs, Clean Code and SOLID
  restated for modules and functions, Vitest guidance, Datadog log conventions). Read it before any substantial
  piece of new code or review so recommendations match the sources, not folklore.
- `references/worked-examples.md`: thirteen real before/after rewrites with the reasoning
  that decided each. Read it when converting actual code; the shapes to hunt for are
  named there.

## Ground rules

1. **Equivalence first.** Before changing anything, name what the old code does that
   the new code must still do: evaluation order, which awaits ran in sequence, side
   effects, distinct log lines, which errors escape and with what status. A rewrite that
   turns sequential writes into `Promise.all`, or merges two different "not found"
   branches, is a bug wearing a cleanup's clothes. When behaviour does change, verify it
   is harmless and say so out loud, or keep the original shape.
2. **The type system is the first test.** Strict mode on, no `any`, no `!`, no `as`
   except `as const` and a commented cast at a verified boundary. `unknown` in, narrow
   with a type guard or a zod schema, typed out. If the compiler cannot prove it, the
   code has not shown it. A red squiggle you silence is a bug you scheduled.
3. **One level of abstraction per function.** A function either orchestrates named steps
   or does one step; never both. If a body needs a comment to separate its phases, each
   phase is a function. Keep bodies around twenty lines, at most three parameters before
   an options object, no boolean flags that switch behaviour.
4. **Names carry the meaning the type cannot.** Functions are verbs
   (`loadActivePrompt`, `markCriteriaRejected`), booleans read as predicates
   (`isMultiUnit`, `hasPolicy`), collections are plural, and no abbreviation beyond
   `id`, `url`, `db`. A name that needs a comment is wrong. Follow the repo's file
   convention exactly; where there is none, `camelCase.kind.ts`.
5. **Layers do not leak.** Routes and controllers parse, validate and map errors to HTTP;
   services hold rules and transactions; repositories hold SQL and return rows mapped
   once at the edge. No `req`/`res` below the controller, no SQL above the repository,
   no business rules in either edge. Cross-cutting needs (clock, logger, db, fetch) are
   passed in, not imported as singletons, so tests can supply their own.
6. **DRY means one home for each piece of knowledge, not zero repetition.** Extract when
   the same rule appears twice and would have to change together. Two blocks that look
   alike but encode different decisions stay apart. Prefer a named helper over a flag.
7. **Push work to the source.** An existence test over fetched rows is an `exists`
   query. Filtering, counting and sorting belong in SQL when the data comes from SQL.
8. **Tests prove behaviour, not wiring.** Test the public function, not its internals.
   Never mock the project's own modules; inject the dependency and pass a fake. Mock only
   the network edge. Repository functions get integration tests against the real
   database. One behaviour per test, named as the sentence it proves.
9. **Delete before you convert.** Dead branches, redundant checks the type already
   proves, and arms subsumed by another arm come out first. The best refactor of
   redundant code is removal.
10. **Verify with the project's tools.** Lint, build and the tests scoped to the touched
    code run green before the change is called safe.

## The moves, by smell

| Smell in the code | Move |
|---|---|
| `any`, `Record<string, any>`, `as SomeType` on a row or JSON | `unknown` plus a zod schema or type guard at the boundary, typed from there (example 1) |
| `value!` or `as Foo` to silence a maybe-undefined | Narrow with a guard, or throw a typed `NotFoundError`; the `!` is `get()` on an Optional (example 2) |
| Ladder of `if (!a) return; if (!a.b) return;` | `a?.b ?? fallback`; keep a guard only where the branch does something distinct, like logging (example 3) |
| `forEach` pushing into an array; `reduce` building an object | `map`/`filter`/`flatMap`; `Object.groupBy`/`Map.groupBy`; `Object.fromEntries` (example 4) |
| `await` inside a `for` over independent items | `Promise.all`; keep the loop when the items depend on each other or the pool is small (example 5) |
| SQL, validation and rules in one handler | Controller, service, repository, with the transaction in the service (example 6) |
| `doThing(x, true)`; string unions typed as `string` | Two functions or an options object; `as const` object plus derived union, exhaustive `switch` with a `never` check (example 7) |
| `vi.mock('../src/...')` of the project's own module | Inject the dependency; test the real function with a fake collaborator (example 8) |
| Same validate-and-400 block in every handler | One schema, one `parseBody` helper that throws a typed error the error middleware maps (example 9) |
| A function with phases separated by blank lines or comments | Named steps at one level of abstraction (example 10) |
| `.then().catch(err => logger.error(err))` | `async`/`await`; let it propagate to the layer that can act; wrap with `cause` (example 11) |
| `console.log`, values interpolated into a log message, a log line with no message | Injected pino logger, constant message naming the event, values as fields (example 12) |
| `(await repo.findAll(...)).length > 0` | An `exists` query (example 13) |
| `enum` | `as const` object plus `typeof` union; type stripping and tree shaking both prefer it |
| `null` flowing through internal code | `undefined` inside; map database `null` once at the repository edge |
| `let` that is assigned once | `const` |
| `require`, missing `node:` prefix, `process.env` outside config | ESM `import`, `node:fs/promises`, one config module that parses env with zod at startup |

Types in one breath: infer locals, annotate every export and every function signature;
`interface` for object shapes, `type` for unions and aliases; discriminated unions over
bags of optional fields; `readonly` on arrays and parameters you do not mutate;
`satisfies` to check a literal without widening it; narrow parameter types to what the
function reads (`Pick<Property, 'id' | 'companyId'>`), widen return types never.

Functions in one breath: pure where possible, side effects at the edges; early-return
guards at the top, happy path unindented; no argument mutation; one return type, not a
union of success shapes and `null`; a result union (`{ ok: true, value } | { ok: false,
reason }`) when the caller must branch, a thrown typed error when it cannot recover.

Async in one breath: `async`/`await` everywhere, never `.then` chains; no floating
promises, write `void` when you mean it; `Promise.all` for independent work,
`Promise.allSettled` when partial failure is expected; `AbortSignal` for anything that
can be cancelled; `await using` for resources that must close.

SOLID in one breath, for modules: a module exports one concern; add behaviour by adding a
variant or a module and let the compiler find the switches; a function that accepts a
type honours all of it; depend on the narrowest interface at the point of use; depend on
what is passed in, not on what is imported.

Logging in one breath: structured JSON through one injected pino logger; every line
has `message`, a constant sentence naming the event, with values as fields so Datadog can
group patterns and facet; `level` as a string, `timestamp` in ISO 8601, `service`, `env`,
`version` on every line, `requestId` inside a request through a child logger; one line
per event at the layer that owns it; secrets redacted at the logger.

Tests in one breath: Vitest, `describe` per function, `it` per behaviour named as a
sentence; arrange, act, assert, with blank lines between; `it.each` for tables; fakes
over mocks; the real database for repositories; coverage is a smoke alarm, not a target.

## Reporting the change

State what changed and what was proven equal, in plain sentences. When a conversion was
declined, give the specific blocker (sequential writes, distinct logging, a connection
pool of five), not a vague preference. When behaviour differs in any observable way, that
is the headline of the report, not a footnote. Name the tests that cover the change.
