# Preferred stack

The default pick for every kind of dependency a Java service needs. Each pick is proven:
widely deployed, actively maintained by an organisation rather than one person, with a
stable API and a long release history. The repo's existing choice wins where it differs;
a library not on this list needs a one-line reason in the pull request.

How to choose, in order: the JDK, then what Spring Boot already manages, then this list.
Nothing is added for what the platform already does.

Contents: Platform (shared across languages) · Runtime and build · Framework · Data ·
Messaging and caching · Clients and integration · Observability · Security · Testing ·
Static analysis · Not this

## Platform (shared across languages)

| Need | Pick | Notes |
|---|---|---|
| Relational database | **PostgreSQL** (RDS) | One engine for everything relational. `JSONB`, arrays, partial indexes, `RETURNING`, advisory locks and `ON CONFLICT` remove most reasons to add a second store. |
| Schema migrations | **Liquibase** | Changesets reviewed with the code, run as a deploy step, never at application start in production. |
| Work queue | **Amazon SQS** | Standard queues by default; FIFO only when ordering or exactly-once matters, and then one message group per aggregate. Dead-letter queue on every queue. |
| Publish/subscribe, fan-out | **Amazon SNS** | Topic per event type, SQS subscriptions per consumer. Producers never know their consumers. |
| Object storage | **Amazon S3** | Presigned URLs for uploads and downloads; the service never proxies bytes. |
| Secrets and config | **AWS Secrets Manager** for secrets, **SSM Parameter Store** for non-secret config | Read at boot into typed config; never at request time, never from a committed file. |
| Scheduled work | **EventBridge Scheduler** firing an SQS message or Lambda | In-process `@Scheduled` only with a distributed lock (ShedLock) when the schedule must live in the app. |
| Shared cache (cross-instance) | **Redis** (ElastiCache) only when a cache must be shared | In-process first; see Messaging and caching. |
| Telemetry | **OpenTelemetry** traces and metrics, shipped to **Datadog** | Structured JSON logs; one trace id across services. |
| Serverless glue | **AWS Lambda** for event-driven glue, never for the core API | Same logging and telemetry contract as the service. |
| Infrastructure | **Terraform** | Nothing created by hand in the console. |
| CI | **GitHub Actions** | Format, lint, type/compile, tests as required gates. |
| Containers | **Docker**, multi-stage, non-root | One image per service. |

## Runtime and build

| Need | Pick | Notes |
|---|---|---|
| JDK | Current LTS (**21**), **Temurin** or **Corretto**, managed by **jolta** | `.java-version` pinned and committed. Never brew or sdkman. |
| Build | **Gradle** with the Kotlin DSL and a version catalog (`libs.versions.toml`) | Maven where the repo already uses it. The wrapper is committed; builds run through it. |
| Formatter | **google-java-format** via **Spotless** | Formatting is a CI gate, never a review comment. |
| Boilerplate | **Records** first; **Lombok** only in repos that already use it, and never `@Data` on an entity | |

## Framework

| Need | Pick | Notes |
|---|---|---|
| Application | **Spring Boot** (current 3.x; 4 when the repo moves) | Starters, auto-config, `@ConfigurationProperties` records. |
| Web | **Spring MVC** with virtual threads (`spring.threads.virtual.enabled=true`) | WebFlux only for streaming or a proven need for back-pressure. Blocking code on virtual threads is the simpler model. |
| Validation | **Jakarta Bean Validation** (Hibernate Validator) | On request records and `@ConfigurationProperties`. |
| JSON | **Jackson** | Boot's default. Records serialise without annotations. |
| Templating (emails, documents) | **Thymeleaf** | Server-side only; the web UI is a separate front end. |
| Object mapping | Hand-written `static Foo from(Entity e)` on the record | **MapStruct** when a module has dozens of mappers; never ModelMapper or reflection mappers. |
| Dependency injection | Spring constructor injection | No other container. |

## Data

| Need | Pick | Notes |
|---|---|---|
| Driver | **pgjdbc** | Boot manages the version. |
| Connection pool | **HikariCP** | Boot's default; size it for the database, not the thread count. |
| ORM | **Spring Data JPA / Hibernate** for aggregates with identity and lifecycle | Every association `LAZY`, fetched per use case. |
| Queries that are not an aggregate | **`JdbcClient`** (Boot 3.2+) with a `RowMapper` to a record | Reports, projections, bulk reads. **jOOQ** when a module is query-heavy enough to want typed SQL. |
| Migrations | **Liquibase** | See Platform. |
| Ids | `BIGINT` sequence for internal keys; **UUIDv7** (`com.fasterxml.uuid:java-uuid-generator`) for public identifiers | Never `IDENTITY` on Postgres, never random UUIDv4 as a clustered key. |
| Money | **`BigDecimal`** with an explicit scale and `RoundingMode`, stored as `NUMERIC` | Never `double`. |
| Time | **`java.time`** with an injected `Clock`, stored as `TIMESTAMPTZ` | Never `Date`, `Calendar`, Joda-Time. |

## Messaging and caching

| Need | Pick | Notes |
|---|---|---|
| AWS SDK | **AWS SDK for Java 2.x** | Never 1.x in new code. |
| SQS consumers and SNS publishing | **Spring Cloud AWS** (`io.awspring.cloud`) `@SqsListener`, `SqsTemplate`, `SnsTemplate` | Visibility timeout longer than the handler's worst case; idempotent handlers keyed by message id. |
| In-process cache | **Caffeine** through Spring's cache abstraction (`spring-boot-starter-cache`) | Bounded size and a TTL on every cache. Caffeine is the only in-process cache. |
| Shared cache | **Redis** through **Lettuce** (Boot's default client) | Only when several instances must see the same entry. Never as a queue, never as the primary store. |
| Event streaming | **Kafka** (MSK) only with a measured throughput or replay need | SNS to SQS covers ordinary events. |
| Distributed lock | **ShedLock** on a Postgres table | For in-process schedules; Postgres advisory locks for one-off critical sections. |
| Background jobs | One class per job, `@Scheduled` plus ShedLock, or an SQS-driven worker | No Quartz unless the repo already runs it. |

## Clients and integration

| Need | Pick | Notes |
|---|---|---|
| HTTP client | **`RestClient`** (Boot 3.2+) built from the auto-configured builder, with explicit connect and read timeouts | `RestTemplate` is maintenance-only; `WebClient` only in WebFlux code. Declarative interfaces via `@HttpExchange` when a client has many endpoints. |
| Retries, circuit breaking, rate limiting | **Resilience4j** | Retries on idempotent calls only, with backoff and jitter. |
| Email | **Spring Mail** (Jakarta Mail) over the provider's SMTP, or the provider's REST API through `RestClient` | Every message is a typed class with a Thymeleaf template. |
| PDF | **OpenPDF** or **Apache PDFBox** | Never iText without a licence review. |
| CSV, Excel | **Jackson CSV** for CSV; **Apache POI** for Excel | |
| Feature flags | Typed `@ConfigurationProperties` booleans or enums | A flag service only when flags change at runtime. |

## Observability

| Need | Pick | Notes |
|---|---|---|
| Logging API | **SLF4J** over **Logback** | `{}` placeholders, throwable last. |
| Structured logs | Boot's `logging.structured.format.console=ecs` (or `logstash`) | JSON in every deployed environment; plain text locally. |
| Metrics and traces | **Micrometer** with the **OpenTelemetry** bridge, exported to **Datadog** | Trace id in every log line through MDC. |
| Health | **Spring Boot Actuator** `health`, `info`, `prometheus` where scraped | Readiness checks the database with a timeout. |

## Security

| Need | Pick | Notes |
|---|---|---|
| Framework | **Spring Security** | Stateless resource server. |
| Tokens | **Nimbus JOSE + JWT** through `spring-security-oauth2-resource-server` | Short-lived access tokens, refresh through a rotated opaque token. |
| Passwords | **Argon2** (`Argon2PasswordEncoder`) or **BCrypt** through `DelegatingPasswordEncoder` | Never MD5, SHA-x, or a hand-rolled scheme. |
| Dependency vulnerabilities | **Dependabot** plus **OWASP Dependency-Check** in CI | |

## Testing

| Need | Pick | Notes |
|---|---|---|
| Framework | **JUnit 5** | Package-private classes, `@Nested`, `@ParameterizedTest`. |
| Assertions | **AssertJ** | Never Hamcrest, never bare `assertEquals`. |
| Mocks | **Mockito** 5, strict stubs | Never mock types you do not own. |
| Database | **Testcontainers** Postgres with `@ServiceConnection` | Never H2 standing in for Postgres in new tests. |
| HTTP edges | **WireMock** for outbound; `MockMvcTester` / `RestTestClient` for inbound | |
| Async assertions | **Awaitility** | Never `Thread.sleep` in a test. |
| Architecture rules | **ArchUnit** | Layer and package dependency rules as tests. |
| Property tests | **jqwik** when an invariant holds for all inputs | |

## Static analysis

| Need | Pick | Notes |
|---|---|---|
| Bug patterns | **Error Prone** with **NullAway** | `@NullMarked` packages; a NullAway error fails the build. |
| Style | **Checkstyle** (Google config) | Enforces what the formatter cannot. |
| Dead code and complexity | **PMD** | Run in CI; findings are faults, not advice. |

## Not this

| Avoid | Because | Use instead |
|---|---|---|
| Guava, Apache Commons Lang/Collections for what the JDK has | `List.of`, `Optional`, `Objects`, `String` methods and streams cover it | the JDK |
| `RestTemplate`, Feign, OkHttp direct | Maintenance mode, or a second HTTP stack | `RestClient` |
| WebFlux for an ordinary CRUD service | Reactive code is harder to read, debug and test; virtual threads give the throughput | Spring MVC on virtual threads |
| Redis, Ehcache, Hazelcast, Infinispan as an in-process cache | A network hop or a second cluster for a local cache | Caffeine |
| Redis or a database table as a queue | Visibility, retries and dead-lettering have to be rebuilt | SQS |
| Kafka without a throughput or replay need | Operational weight for fan-out that SNS does | SNS to SQS |
| H2 in tests standing in for Postgres | The dialect differs where it hurts: `RETURNING`, `JSONB`, constraints | Testcontainers |
| Lombok `@Data`, `@EqualsAndHashCode`, `@ToString` on entities | Lazy-loading and equality bugs | records for DTOs; explicit methods on entities |
| ModelMapper, Dozer, Orika | Reflection mapping hides broken mappings until runtime | hand-written `from` methods or MapStruct |
| Joda-Time, `java.util.Date`, `Calendar` | Superseded by `java.time` | `java.time` |
| JUnit 4, Hamcrest, PowerMock, EasyMock | Superseded; PowerMock encourages testing privates | JUnit 5, AssertJ, Mockito |
| AWS SDK 1.x | End of support | SDK 2.x |
| MySQL, MariaDB, Oracle, SQL Server for a new service | One engine the team operates well | Postgres |
| MongoDB or DynamoDB as a primary store | Relational data without a schema costs more later; `JSONB` covers document-shaped columns | Postgres |
| Quartz for new schedules | Heavy for what `@Scheduled` plus ShedLock or EventBridge Scheduler does | ShedLock or EventBridge Scheduler |
| `javax.*` packages | Renamed to `jakarta.*` since Boot 3 | `jakarta.*` |
