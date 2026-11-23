# Preferred stack

The default pick for every kind of dependency a Node service needs. Each pick is proven:
widely deployed, actively maintained by an organisation rather than one person, with a
stable API and a long release history. The repo's existing choice wins where it differs;
a library not on this list needs a one-line reason in the pull request.

How to choose, in order: the Node platform (`node:` built-ins, `fetch`, `AbortSignal`),
then what the framework already provides, then this list. Nothing is added for what the
platform already does.

Contents: Platform (shared across languages) · Runtime and build · Framework · Data ·
Messaging and caching · Clients and integration · Observability · Security · Testing ·
Lint and format · Not this

## Platform (shared across languages)

| Need | Pick | Notes |
|---|---|---|
| Relational database | **PostgreSQL** (RDS) | One engine for everything relational. `JSONB`, arrays, partial indexes, `RETURNING`, advisory locks and `ON CONFLICT` remove most reasons to add a second store. |
| Schema migrations | **Liquibase** | Changesets reviewed with the code, run as a deploy step, never at application start in production. |
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
| Runtime | Current **Node LTS**, pinned with **Volta** in `package.json` | Never nvm or brew. `engines` set so a wrong Node fails `npm ci`. |
| Package manager | **npm** with a committed `package-lock.json`; `npm ci` in CI | pnpm where the repo already uses it. |
| Language | **TypeScript**, strict, ESM (`"type": "module"`) | `erasableSyntaxOnly` so Node's type stripping can run it. |
| Running TypeScript | Node's native type stripping (`node --experimental-strip-types` or 22.18+/24 default), **tsx** where a repo still needs transforms | `tsc --noEmit` is the type check; `ts-node` is retired. |
| Bundling (Lambda, CLI) | **esbuild** | One file per entry point, external AWS SDK. |
| Monorepo | **npm workspaces**; **Turborepo** only when build caching is measured to matter | |

## Framework

| Need | Pick | Notes |
|---|---|---|
| HTTP server | **Fastify** for a new service; **Express 5** where the repo already runs it | Fastify's schema-based routing and built-in pino fit the standard. Express 5 handles async rejections natively. |
| Validation and parsing | **zod** | `z.infer` is the only source of the type. `fastify-type-provider-zod` on Fastify. |
| Dependency wiring | Factory functions taking a `deps` object, one composition root | No DI container, no decorators. |
| GraphQL (when required) | **GraphQL Yoga** with **Pothos** for a code-first schema | REST by default; GraphQL only when clients need the shape flexibility. |
| OpenAPI | Generated from the zod schemas (`@asteasolutions/zod-to-openapi` or Fastify Swagger) | Never hand-maintained. |

## Data

| Need | Pick | Notes |
|---|---|---|
| Driver | **`pg`** (node-postgres) with a pool built in the composition root | **`postgres`** (porsager) acceptable where already in use. |
| Query layer | **Kysely** with **kysely-codegen** for types generated from the live schema | Typed SQL that stays SQL. Generated types are never edited by hand. |
| Migrations | **Liquibase** | See Platform. The Node service does not run migrations at start. |
| Ids | `BIGINT` identity for internal keys; **UUIDv7** (`uuid` v10+ `v7()`) for public identifiers | `crypto.randomUUID()` only where a v4 is fine. |
| Money | Integers in minor units (`amountCents: number`) or **big.js** for fractional math | Never `number` arithmetic on dollar amounts. |
| Time | **Temporal** where the runtime has it; **date-fns** and `Intl` otherwise; ISO strings on the wire; `TIMESTAMPTZ` in the database | Never moment, never a mutable `Date` passed around. An injected `clock` in services. |

## Messaging and caching

| Need | Pick | Notes |
|---|---|---|
| AWS SDK | **AWS SDK for JavaScript v3** (`@aws-sdk/client-*`), one client per service built once | Never v2 (`aws-sdk`). |
| SQS consumer | **sqs-consumer** (BBC) over `@aws-sdk/client-sqs` | Visibility timeout longer than the handler's worst case; idempotent handlers keyed by message id. |
| SNS publishing | `@aws-sdk/client-sns` behind a `Publisher` interface in `deps` | |
| In-process cache | **lru-cache** with `max` and `ttl` on every instance | The only in-process cache. |
| Shared cache | **Redis** through **ioredis** | Only when several instances must see the same entry. Never as a queue, never as the primary store. |
| Distributed lock | Postgres advisory locks (`pg_try_advisory_lock`) | For the rare one-runner section. |
| Background work | A separate worker entry point consuming SQS, sharing the composition root | Never a thread or `setInterval` loop inside the web process. |

## Clients and integration

| Need | Pick | Notes |
|---|---|---|
| HTTP client | Built-in **`fetch`** (undici) with `AbortSignal.timeout` on every call | **undici** `Agent` directly when pooling or keep-alive settings matter. No axios, no node-fetch. |
| Retries | **p-retry** on idempotent calls, with backoff and jitter | |
| Concurrency limit | **p-limit** | Over user-sized input. |
| Email | The provider's REST API through `fetch`, each message a typed function with a template | |
| PDF | **pdf-lib** for assembly; **Playwright** rendering HTML when layout matters | Never a headless Chrome inside the request path. |
| CSV | **csv-parse** / **csv-stringify** (the `csv` package) | Streaming on large files. |
| Feature flags | Typed booleans or unions in the config module | A flag service only when flags change at runtime. |

## Observability

| Need | Pick | Notes |
|---|---|---|
| Logging | **pino**, one instance injected, child logger per request | Constant `message` naming the event, values as fields. `pino-pretty` locally only. |
| Traces and metrics | **OpenTelemetry** (`@opentelemetry/sdk-node` with auto-instrumentations), exported to **Datadog**; **dd-trace** where the repo already runs it | Trace id in every log line. |
| Health | `/health/live` and `/health/ready` routes; readiness checks the pool with a timeout | |

## Security

| Need | Pick | Notes |
|---|---|---|
| JWT and JOSE | **jose** | Never `jsonwebtoken` in new code. |
| Passwords | **argon2** (or `bcrypt`) | Never a hand-rolled hash. |
| HTTP hardening | **@fastify/helmet** / **helmet**, **@fastify/rate-limit** / **express-rate-limit**, **@fastify/cors** / **cors** | |
| Dependency vulnerabilities | **Dependabot** plus `npm audit --omit=dev` in CI | |

## Testing

| Need | Pick | Notes |
|---|---|---|
| Runner | **Vitest** | Never Jest, Mocha or Chai in new code. |
| HTTP edges | **MSW** with `onUnhandledRequest: 'error'`; `app.inject()` on Fastify / **supertest** on Express for inbound | |
| Database | **Testcontainers** (`@testcontainers/postgresql`) | Never SQLite or an in-memory fake standing in for Postgres. |
| Browser end to end | **Playwright** | |
| Fakes | Two-line objects passed through `deps` | Never `vi.mock` of `src/`. |
| Property tests | **fast-check** when an invariant holds for all inputs | |

## Lint and format

| Need | Pick | Notes |
|---|---|---|
| Lint | **ESLint** flat config with **typescript-eslint** `strictTypeChecked` and `stylisticTypeChecked` | `no-floating-promises`, `no-explicit-any`, `consistent-type-imports` on. |
| Format | **Prettier** | **Biome** acceptable where a repo adopts it for both jobs. Formatting is a CI gate, never a review comment. |
| Dead code | **knip** | Unused exports and dependencies fail CI. |

## Not this

| Avoid | Because | Use instead |
|---|---|---|
| axios, node-fetch, got, request | The platform has `fetch` and `AbortSignal` | built-in `fetch` |
| lodash, underscore, ramda | `structuredClone`, `Object.groupBy`, `toSorted`, `at`, spread cover it | the platform |
| moment, dayjs | Mutable or superseded | Temporal, date-fns, `Intl` |
| TypeORM, Sequelize, MikroORM | Decorator ORMs hide the SQL and fight strict types | Kysely |
| Prisma | A separate query engine and its own migration model next to Liquibase | Kysely |
| Jest, Mocha, Chai, Sinon | Slower, weaker ESM and TypeScript support | Vitest |
| `vi.mock('../src/...')` | Hides the dependency that was never injected | fakes through `deps` |
| BullMQ, Bee-Queue, Agenda | A Redis queue to operate when SQS exists | SQS |
| `node-cron`, `setInterval` jobs in the web process | Runs once per instance and dies with the pod | EventBridge Scheduler |
| Winston, Bunyan | Slower, no per-request child logger contract | pino |
| `jsonwebtoken` | Maintenance mode, weak algorithm defaults historically | jose |
| `dotenv` in production, `process.env` read in services | Config is parsed once at boot | the config module |
| `express-async-errors`, `.catch(next)` ladders | Express 5 handles async rejections | nothing |
| ts-node, nodemon | Node strips types and watches natively | `node --watch`, tsx where needed |
| AWS SDK v2 (`aws-sdk`) | End of support | `@aws-sdk/client-*` v3 |
| MySQL, MariaDB, MongoDB, DynamoDB for a new service | One engine the team operates well; `JSONB` covers document-shaped columns | Postgres |
| `enum`, `namespace`, class parameter properties | Not erasable; Node cannot strip them | `as const` unions, plain modules |
