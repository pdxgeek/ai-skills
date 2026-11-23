# ai-skills

Coding standards for AI agents, packaged as Claude Code skills. Each skill teaches one
language the same way: modern features used for what they are for, immutable data, small
named functions, one home for every rule, errors that travel to one handler, structured
logs, and tests that run the real thing with fakes passed in.

| Skill | Covers |
|---|---|
| `skills/java-coding` | Java 17/21/25 with Spring Boot: records, sealed types, Optional, streams, JPA, transactions, SLF4J, JUnit 5, AssertJ, Testcontainers, jolta for JDKs |
| `skills/typescript-coding` | TypeScript on Node: strict types, discriminated unions, zod at the boundary, Express/Fastify layering, pino, Vitest, MSW |
| `skills/python-coding` | Python 3.12/3.13/3.14: frozen dataclasses, pydantic at the boundary, FastAPI, SQLAlchemy 2.0 on Postgres, structlog, pytest, uv and ruff |

## Layout of a skill

Every skill has the same shape so an agent loads only what the task needs:

```
skills/<name>/
  SKILL.md                    # the standard in brief; loaded whenever the skill triggers
  references/style.md         # types, data, functions, errors, async
  references/architecture.md  # layout, wiring, config, database, HTTP, logging, tooling
  references/testing.md       # what to fake, what to run for real, test shape
  references/refactoring.md   # the sub-skill for changing code that already works
```

`SKILL.md` is under 200 lines and carries a table saying which reference to open for which
kind of task. The references hold the depth, the version notes and the worked examples.
Keep it that way: new rules go in the reference that owns the topic, and `SKILL.md` gets
one line pointing at it only when the rule matters on every task.

## Installing

For every project on a machine, copy or symlink into the user skills folder:

```bash
ln -s "$PWD/skills/python-coding" ~/.claude/skills/python-coding
```

For one repository, copy the folder into `.claude/skills/` and commit it, so every agent
working on that repo gets the same standard:

```bash
cp -R skills/python-coding /path/to/repo/.claude/skills/
```

A repository copy wins over this repo when they drift; sync it back here when a rule
changes.

## Writing a new skill

- Mirror the four references. A language with no async story can leave the section out;
  do not invent one.
- The `description` in `SKILL.md`'s front matter is what makes the skill trigger. Lead
  with writing, list the everyday phrasings ("add a function", "fix this bug"), and name
  the concrete smells a reviewer would see in a diff.
- Every rule carries its reason in the reference, in one sentence. A rule without a reason
  gets argued with.
- Say which version introduced an API. Agents get the installed version wrong when the
  skill does not tell them to check.
- Plain prose. No em-dashes, no "not X but Y", no bullet that is a paragraph.
