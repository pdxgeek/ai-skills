---
name: java-coding
description: Write and refactor modern Java (17+) using stream pipelines, disciplined Optional chains, records, and var. Use whenever writing, reviewing, or cleaning up Java code - converting null-check-and-early-return code to chains, replacing loops or CollectionUtils.isEmpty checks, introducing records for multi-value results, or deciding where var belongs. Trigger on requests like "make this a stream", "clean this up", "modernize this", "looks sus", or any Java diff showing orElse(null), isPresent()/get(), guard clauses before chains, or hand-rolled tuples, even when the user does not say "refactor".
---

# Modern Java refactoring

Convert legacy-shaped Java into Java 17 idioms without changing behavior. The two
reference files carry the depth; this file is the method.

- `references/best-practices.md`: the distilled canon (Effective Java, Stuart Marks'
  Optional rules, Goetz on records, the OpenJDK var style guide). Read it before any
  substantial refactor or review so recommendations match the sources, not folklore.
- `references/worked-examples.md`: nine real before/after refactors with the reasoning
  that decided each. Read it when converting actual code; the shapes to hunt for are
  named there.

## Ground rules

1. **Equivalence first.** Before converting anything, name what the old code does that
   the new code must still do: evaluation order, side effects, distinct log lines,
   which exceptions escape, eager vs lazy defaults. A conversion that quietly merges two
   different empty-branches or broadens a catch is a bug wearing a cleanup's clothes.
   When a conversion does change something (say, a lookup now runs before a role check),
   either verify the change is harmless and say so out loud, or keep the guard.
2. **Chains are the default, not a religion.** Prefer the Optional/stream chain whenever
   it is behaviorally identical and at least as readable. Keep the imperative form when
   the body mutates locals, needs break/early exit across iterations, or a builder API
   is inherently imperative. Never force a chain that needs contortions; fix the reason
   for the contortion instead (see rule 3).
3. **Named helpers make chains readable.** A pipeline reads as a sentence only when each
   stage is a short expression or a well-named call. Extract fat lambdas into private
   helper methods whose names state intent (`requireStripeCustomer`, `resolvePayeeEmail`).
   This is also the standard unblocking move: a callee that throws a checked exception
   cannot sit in a lambda, so wrap it in a helper that translates the failure into the
   domain's runtime exception, then chain.
4. **Push work to the source.** An existence test over repository results is an
   `existsBy...` query, not `!CollectionUtils.isEmpty(findAll...)`. Repository list
   results are never null, so null-tolerant wrappers on them are dead code.
5. **Delete before you convert.** Check compound conditions for arms subsumed by another
   arm and lookups duplicated by a central checker. The best refactor of redundant code
   is removal.
6. **Verify with the project's tests.** Compile and run the tests scoped to the touched
   code before claiming the refactor is safe.

## The moves, by smell

| Smell in the code | Move |
|---|---|
| `findOne(...).orElse(null)` then null check | `map`/`filter`/`flatMap` chain (example 1) |
| Null guard on a value the next line wraps anyway | Start from `Optional.ofNullable(x)` (examples 2, 5) |
| Two early returns, one of them logs | `.or(() -> { log...; return Optional.empty(); })` or keep the guard; never merge distinct empties silently (example 3) |
| try/catch blocking a chain | Extract a translating helper (example 4) |
| `isPresent()` + `get()` | The chaining API; `get()` is banned, `orElseThrow()` when presence is proven |
| `!isEmpty(repo.findAll...)` | Derived `existsBy...` query (example 6) |
| Hand-rolled pair, parallel locals, Map.Entry abuse | Private or local record (example 7) |
| sort then `get(0)` | `min`/`max` with a composed Comparator, unique-key tiebreak (example 8) |
| `filter(...).count() > 0` | `anyMatch` |
| forEach accumulating into a collection | A collector (`groupingBy`, `toMap`, `toList()`) |
| Unreadable one-line mega-chain | `var` intermediates with role-revealing names, or a local record |

Optional discipline in one breath: Optionals are for return types; never fields,
parameters, or collection elements; never `get()`; `orElseGet` when the default costs
anything; `orElse(null)` only at a boundary handing off to a nullable-consuming API; a
single-step `ofNullable(x).orElse(y)` is a ternary, write the ternary.

Records in one breath: transparent immutable carriers; validate and defensively copy in
the compact constructor; local records to name pipeline intermediates; never JPA
entities; excellent as projections, compound keys, and multi-value returns.

var in one breath: use it when the initializer states the type or to name intermediate
steps of a broken-up chain; the variable name then carries what the type used to; avoid
it with diamond, bare generic factories, and bare int literals.

## Reporting the refactor

State what changed and what was proven equal, in plain sentences. When a conversion was
declined, give the specific blocker (checked exception, mutation ordering, distinct
logging), not a vague preference. When behavior differs in any observable way, that is
the headline of the report, not a footnote.
