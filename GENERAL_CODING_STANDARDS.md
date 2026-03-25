# General Coding Standards

> **Status:** Finalized and locked.
> Deviations require explicit review and update.
> Code and documentation must move together.

---

## Contents

1. [Governance](#1-governance)
2. [Platform baseline](#2-platform-baseline)
3. [Architecture](#3-architecture)
4. [Dependency direction](#4-dependency-direction)
5. [Pluggability](#5-pluggability)
6. [Package and module discipline](#6-package-and-module-discipline)
7. [Interface design](#7-interface-design)
8. [Null and Optional policy](#8-null-and-optional-policy)
9. [Validation layering](#9-validation-layering)
10. [Enums](#10-enums)
11. [Application codes](#11-application-codes)
12. [Message codes](#12-message-codes)
13. [Exception model](#13-exception-model)
14. [Exception handler and error response](#14-exception-handler-and-error-response)
15. [HTTP status code standards](#15-http-status-code-standards)
16. [Code-to-response mapping](#16-code-to-response-mapping)
17. [Transport and timeout handling](#17-transport-and-timeout-handling)
18. [Retry and resilience](#18-retry-and-resilience)
19. [Async and threading](#19-async-and-threading)
20. [Tracking IDs and correlation](#20-tracking-ids-and-correlation)
21. [Logging](#21-logging)
22. [Persistence](#22-persistence)
23. [Security, observability, audit, tenancy](#23-security-observability-audit-tenancy)
24. [Comments and documentation](#24-comments-and-documentation)
25. [Immutability and defensive copying](#25-immutability-and-defensive-copying)
26. [Test categories](#26-test-categories)
27. [Test tooling stack](#27-test-tooling-stack)
28. [Testing rules](#28-testing-rules)

---

## 1. Governance

- Standards are locked once agreed.
- Deviations require explicit review and update to this document before implementation.
- Code and documentation must stay aligned — update both in the same commit.
- Stale comments and stale docs are defects.

---

## 2. Platform baseline

- **Java:** 21+
- **Spring Framework:** 7.x
- **Spring Boot:** 4.x

Do not add a Java feature to satisfy a checklist. Add it only when it makes the code meaningfully clearer.

---

## 3. Architecture

- Core owns contracts only.
- Each module owns one clear capability.
- Capabilities must be replaceable.
- Business logic must not depend on infrastructure branches.
- Composition modules wire things together but do not own business rules.

---

## 4. Dependency direction

- Dependencies flow inward toward contracts and core.
- No circular dependencies.
- No sideways coupling between implementation modules unless explicitly justified and documented.
- Do not hide infrastructure failures behind silent fallback. Infrastructure failures are problems that must surface.

---

## 5. Pluggability

- Prefer framework-native extension points.
- Use conditional bean registration for defaults and overrides.
- Default implementations must back off cleanly when a consumer provides its own bean.
- There must be exactly one effective bean per primary contract at injection time. Wrapper modules decorate the selected effective bean — they must not register themselves as competing beans.
- Activation should be structural and explicit, not hidden in business logic.

---

## 6. Package and module discipline

### Package placement and separation

Public APIs must live in `api`. Extension-point interfaces must live in `spi`. Domain and transport models must live in `model` subpackages. Enums should live in dedicated enum packages or files. Services, validators, repositories, controllers, DTOs, entities, adapters, application codes, and exception hierarchies must each live in responsibility-specific packages. Unrelated concerns must not be clubbed together in one package. Avoid vague catch-all packages such as `common` or `utils`; use `core` only when it is a clearly bounded architectural layer.

### Import direction

Package placement implies import direction. Consumer code imports from `api`, not from `spi`. Exception packages may import from code packages within the same layer, but must never import from external-facing or web-layer packages. Contract modules must have no implementation-level imports.

### Hard rules

- One public type per file.
- No nested DTOs inside controllers.
- No mixing DTOs and entities in the same package.
- `api` contains only what application code imports and uses directly.
- `spi` contains only framework extension points — application code must never import from `spi`.
- Demo and sample applications belong in a separate module — never inside any starter or library source tree.
- Every class must have a precise, responsibility-aligned home.

---

## 7. Interface design

- Create interfaces only for real extension points or meaningful test seams.
- Keep internal concrete classes concrete when no extension is needed.
- Downstream code should depend on the effective contract, not specific implementations.
- Do not extract interfaces for the sake of symmetry.

---

## 8. Null and Optional policy

### Decision per case

| Pattern | Use case |
|---|---|
| `Objects.requireNonNull` | Required collaborators, constructors, mandatory arguments |
| Null-to-default normalization | Record compact constructors, DTO normalization, config normalization |
| Plain `if null` check | Internal branching logic in providers, mappers, validators, factories |
| `Optional<T>` return type | Repository lookups, parsers, find-or-miss operations, genuine absence |
| `Optional<T>` for optional bean | Spring auto-configuration, optional collaborators |

### What must never happen

| Anti-pattern | Why |
|---|---|
| `Optional` field in JPA entity | JPA does not map Optional fields cleanly |
| `Optional` field in DTO or config property | Hurts serialization and binding clarity |
| `Optional` as a method parameter | Burdens the caller — bad API design |
| `Optional` return from every internal helper | Over-engineers simple code |
| `Optional.get()` without presence check | Null danger in a different form |
| `Optional<Collection<T>>` | Prefer empty collection |

---

## 9. Validation layering

- Validation must be split by responsibility.
- Input shape validation, command invariants, configuration validation, runtime policy validation, and final business invariants should not be mixed.
- Startup and configuration validation should collect all issues before failing — the operator sees the complete picture in one failure.
- Runtime validation must enforce final safety before side effects happen.
- Each validation layer owns exactly one responsibility. No layer duplicates another.

---

## 10. Enums

### When to use enums

Use enums for bounded domain states, category values, severity levels, lifecycle states, code families, and policy modes.

### Design rules

- One enum per concept.
- One public enum per file.
- Enum names must be stable and intention-revealing.
- Do not overload a single enum with unrelated domains.
- Do not use string literals throughout the codebase where a bounded enum exists.
- Enums used in external contracts must be versioned carefully.
- Internal enums may carry metadata methods when that metadata is intrinsic to the enum value.
- Use exhaustive switch on enum values — never if-else chains, never string-prefix matching.
- Do not use enums where values are truly open-ended and unbounded.

### Value-label enums vs behavior-carrying enums

Some enums are pure value labels used in logging and diagnostics only — never for branching logic. Other enums carry behavior through methods such as classification accessors or policy decisions. Both are valid. The distinction should be clear from the enum's design: if it carries methods beyond `code()` and `defaultMessage()`, it is a behavior-carrying enum and must be documented accordingly.

---

## 11. Application codes

### Two-layer code model

All projects should use a two-layer code system:

- **Internal application code** — used in logs, exceptions, metrics, audit records, tracing, support, and diagnostics.
- **External/API code** — used only in API responses or externally visible error payloads.

Internal codes must never be replaced by HTTP status alone. External/API codes must never be used as internal diagnostic identifiers. Logs and exceptions should carry internal codes. API responses may expose external codes and status, but should not leak internal-only implementation detail unless intentionally designed.

### Application code interface

Define a shared application code interface with at minimum:

- A stable code string accessor (e.g., `code()`)
- A default human-readable message accessor (e.g., `defaultMessage()`)

All code enums implement this single interface. Exceptions, logs, metrics, and error responses all consume this interface — it is the single thread that connects them.

### Code families

Each meaningful failure or business condition should have a typed application code. Every major subsystem gets its own code family. One enum per code family. Never mix prefixes from multiple families into the same enum.

Generic family examples: validation, request/input, business rules, provider/integration lookup, transport/client, persistence, async/execution, security, audit, configuration, core/service lifecycle.

### Code naming rules

- Use stable, readable prefixes (e.g., `{DOMAIN}-{LAYER}-{number}`).
- Keep numeric parts structured and documented.
- Codes must be unique within a project.
- Codes should be grep-friendly and dashboard-friendly.
- Do not rename codes casually once released.

### Sentinel codes

When framework exceptions bypass the domain exception hierarchy (e.g., Spring-originated validation exceptions like `MethodArgumentNotValidException`), use a reserved sentinel code so that the internal code field in error responses is always populated — never pass `null` where a code is expected.

### Category classification

When a single exception type must map to multiple external codes based on the nature of the failure, add a classification method to the code enum (e.g., `configCategory()`). The handler switches on the classification — never string-prefix matching, never if-else chains. The switch must be exhaustive.

### Codes that never produce an API response

Some codes are observability-only — they appear in logs and metrics but are never surfaced to API consumers. Examples include lifecycle events, policy-driven drops, deduplication collapses, fallback triggers, and informational diagnostics. Explicitly document which codes are response-producing and which are observability-only.

### Observability integration

Application codes should appear consistently in exception objects, error responses where appropriate, structured logs, metrics tags where useful, audit events, and support diagnostics.

---

## 12. Message codes

Message codes are different from application codes. Use message codes when:

- Messages are localized.
- User-facing text may vary by language.
- The same code maps to multiple rendered texts.
- UI/client teams need stable message identifiers.

### Rules

- Message codes identify message templates, not failure ownership.
- Do not use free-text messages as the only stable identifier.
- Message text may change; message code should remain stable.
- Message codes should not replace application codes.
- One error may have both an application code and a message code.

### Recommended split

- **Application code** = operational/engineering identity.
- **Message code** = user-facing or localization identity.

Message codes are optional. They are needed only when the system has localization or multi-language requirements. Not every project needs both layers.

---

## 13. Exception model

### Hierarchy rules

- Define an abstract base exception that carries the application code interface as a non-optional field.
- Every concrete exception extends the base and carries a precise internal application code.
- Never throw a framework-facing business exception without a code.
- Do not throw raw generic runtime exceptions across application boundaries.
- Translation layers must convert low-level failures into typed exceptions with codes before they leave the service layer.
- The exception type and the code should reinforce each other, not contradict each other.

### Constructor discipline

The abstract base exception should define three constructors:

| Constructor | Behavior |
|---|---|
| 1-arg: `(ApplicationCode)` | Uses `code.defaultMessage()` as the exception message |
| 2-arg: `(ApplicationCode, String)` | Uses the explicit message |
| 3-arg: `(ApplicationCode, String, Throwable)` | Uses the explicit message and wraps the cause |

All concrete exceptions delegate to the same three constructors. This ensures consistency: every exception always has a code, and optionally a custom message and a cause chain.

### Subtype relationships

Where it makes semantic sense, use exception subtypes to allow catch blocks at the right granularity. For example, a timeout exception may extend a transport exception so that callers can catch at either level. The subtype relationship must reflect a genuine is-a relationship.

### `InterruptedException` rule — mandatory

```
CORRECT:
  catch InterruptedException →
    Thread.currentThread().interrupt()   // always restore
    throw typed exception with code and cause

WRONG:
  catch InterruptedException →
    log.warn("interrupted")              // interrupt status lost forever
```

Never swallow `InterruptedException`. Always restore the interrupt flag. Always wrap in a typed exception with a code.

---

## 14. Exception handler and error response

### Handler rules

- Handle both framework validation exceptions (e.g., `MethodArgumentNotValidException`, `ConstraintViolationException`) and domain exceptions.
- `4xx` handlers do not log — client errors are not server problems.
- `5xx` handlers log at `ERROR` with stack trace only at the layer that owns the failure.
- Never log the same exception twice — the service layer logs the root cause, the handler returns the response.
- Configuration or category-based exceptions mapped via exhaustive switch — never string matching, never if-else chains.
- A catch-all `Exception` handler must always be present as the final safety net.

### Handler logging rules by HTTP status range

| HTTP status range | Log level | Stack trace |
|---|---|---|
| `4xx` client errors | None | No |
| `404` not found | `DEBUG` | No |
| `422` business rule violation | `WARN` | No |
| `5xx` configuration errors | `ERROR` | Yes — if not already logged by the service |
| `5xx` transport/integration errors | `ERROR` | Yes — the transport module logs root cause |
| `503` temporarily unavailable | `WARN` | No |
| Catch-all `Exception` | `ERROR` | Yes |

### Error response shape

Define a standard error response object. Recommended fields:

| Field | Description |
|---|---|
| `success` | Boolean — always `false` for error responses |
| `code` | External/API-facing code string |
| `httpStatus` | Numeric HTTP status |
| `message` | Human-readable error description |
| `internalCode` | Internal application code for support diagnostics — included when allowed by security and support exposure policy |
| `details` | Typed list of detail objects (e.g., field errors) — never `List<?>` |
| `path` | Request path that triggered the error |
| `timestamp` | ISO 8601 timestamp |

The `internalCode` field connects API consumers to support tickets and logs without exposing the internal code as the primary identifier. The `details` field must be a typed list (e.g., `List<FieldErrorDetail>`), never an untyped or raw list.

---

## 15. HTTP status code standards

HTTP status codes are transport semantics, not business identity.

### Rules

- HTTP status must communicate the response class correctly.
- Internal codes must still exist even when HTTP status is present.
- Do not encode full business meaning in HTTP status alone.
- Use typed mapping from exception type or category to HTTP response.
- Avoid string-prefix matching or ad hoc conditional chains when mapping codes to HTTP responses.
- Configuration/business category to HTTP mapping should be explicit and centralized.

### Recommended HTTP status model

| Status | Use for |
|---|---|
| `200` | Success |
| `207` | Partial success (e.g., batch with mixed outcomes) |
| `400` | Malformed or invalid client input |
| `401` | Authentication failure |
| `403` | Authorization failure |
| `404` | Resource not found |
| `409` | State conflict |
| `422` | Business-rule violation when syntax is valid but request is not acceptable |
| `429` | Throttling or rate limit |
| `500` | Internal configuration or application failure |
| `502` | Downstream transport or integration failure |
| `503` | Service or subsystem temporarily unavailable |
| `504` | Downstream timeout |

### External code enum carries embedded HTTP status

The external/API code enum should carry its HTTP status directly so the handler never maintains a separate mapping table:

```
GATEWAY_TIMEOUT("APP-HTTP-5040", HttpStatus.GATEWAY_TIMEOUT, "Downstream gateway timed out")
```

---

## 16. Code-to-response mapping

### Required mappings

There should be explicit, centralized mappings between:

- Application code → exception type
- Exception type → external/API code
- Exception type or category → HTTP status
- Application code → log severity where needed
- Application code → metrics and audit tags where needed

### Rules

- Mappings must be centralized — one handler class, not scattered across controllers and services.
- Mappings must be typed where possible — use exhaustive switch on enums, not string matching.
- Do not duplicate the same mapping logic in multiple layers.
- Do not scatter response mapping logic across controllers and services.

### What to avoid

- Using only HTTP status with no internal code.
- Using only free-text messages with no stable code.
- Mixing multiple code families in one enum.
- Throwing uncoded generic exceptions across boundaries.
- Leaking raw infrastructure exception text directly to clients.
- Duplicating mapping logic in several places.
- Relying on string-prefix checks instead of typed mappings.

---

## 17. Transport and timeout handling

- External transport concerns must be behind an SPI or adapter boundary.
- Timeouts must be explicitly configured — a missing or default timeout is a production bug.
- Transport exceptions must be translated into typed domain exceptions with internal codes before they leave the service layer. Every exit from a translator must return a typed exception — never a raw `RuntimeException`.
- Do not hide infrastructure failures behind silent fallback.
- Both transport-level timeouts (e.g., socket timeouts) and application-level deadlines (e.g., future timeouts) are required. Missing either layer leaves gaps.

---

## 18. Retry and resilience

- Retry must be policy-driven and exposed through an SPI.
- Retry only transient failures.
- Non-transient failures must fail fast.
- Circuit breaking, retry, timeout, throttling, and fallback must each have clearly separated responsibilities.
- Concurrency limits must be explicit (e.g., semaphore-based throttling before task submission).
- Application-level deadlines must be enforced on async operations to prevent orphaned tasks.

### Retriable vs non-retriable

Document which failure codes are retriable and which are not. Authentication failures, TLS failures, configuration problems, and business-rule violations are never retriable. Timeouts, connection refusals, and stale connections are typically retriable. Generic or ambiguous failures should be non-retriable by default with an explicit opt-in toggle.

---

## 19. Async and threading

- Never swallow `InterruptedException`. Always restore interrupt status.
- Context propagation across async boundaries should use a shared abstraction (e.g., an `ExecutionContextPropagator` SPI).
- Each context propagator should handle only its own context and clean up after execution.
- Propagator order must not matter for correctness.
- Concurrency limits must be explicit.
- Rejected execution must be translated into a typed exception, not swallowed or re-wrapped generically.

### Correct propagator pattern

```
capture context on calling thread
return wrapped task:
    restore context on worker thread
    try: run task
    finally: clear context
```

---

## 20. Tracking IDs and correlation

### Rules

- Tracking IDs (`correlationId`, `batchId`, `requestId`, `traceId`) are created once at the request entry point (web layer, API filter, or caller) and propagated via MDC.
- Downstream modules read from MDC — they never generate their own.
- Missing context results in `"untraced"` plus a `WARN` log — never a fallback `UUID`.
- Cross-thread MDC propagation is owned by context propagator beans, not by service or dispatcher code.

### Ownership model

| Responsibility | Owner |
|---|---|
| Create tracking IDs | Web layer, API filter, or external caller |
| Read tracking IDs | Every downstream module via `MDC.get()` |
| Generate tracking IDs | Nobody downstream — never `UUID.randomUUID()` in service, dispatcher, or provider code |
| Handle missing context | `"untraced"` + `log.warn()` — surfaces the gap, does not hide it |
| Cross-thread propagation | `ExecutionContextPropagator` beans |

### Lambda capture

When tracking IDs read from MDC are captured in lambdas, use the effectively-final pattern:

```
var rawId = MDC.get("key");
if (rawId == null || rawId.isBlank()) { log.warn(...); }
final var id = (rawId != null && !rawId.isBlank()) ? rawId : "untraced";
// id is effectively final — safe for lambda capture
```

---

## 21. Logging

### Logger setup

Every concrete class has its own logger. Never share loggers across classes.

### Log level rules

| Level | When to use |
|---|---|
| `DEBUG` | Detailed diagnostics — resolution details, provider chosen, fallback path, retry attempt detail |
| `INFO` | One line at major operation start, one line at successful completion |
| `WARN` | Recoverable anomaly — policy-driven drops, deduplication, degraded behavior, missing context |
| `ERROR` | Actual failure with stack trace — at the layer that owns the failure only |

### Duplicate logging rule

Do not log the same exception stack trace twice. The layer that owns the failure logs at `ERROR` with the full stack trace. Upstream layers that observe the same failure log at `WARN` with message only. The exception handler returns the response — it does not re-log at `ERROR`.

### PII handling

PII must be masked in `WARN` and `ERROR` logs. Full values at `DEBUG` are allowed only in explicitly approved non-production or tightly controlled support scenarios.

### Structured log fields

Include structured fields consistently so log aggregation tools can filter and correlate. Application codes must appear in all failure log lines. Tracking IDs (`correlationId`, `batchId`, `traceId`, `tenantId`) must appear in all relevant log lines.

### Log format

Use a consistent domain prefix on all log lines for easy filtering (e.g., `[email]`, `[payment]`, `[auth]`).

---

## 22. Persistence

- Repositories should expose only the operations allowed by the module's responsibility.
- Read-only modules must remain read-only — no `save`, `delete`, `update`, or mutation methods.
- Persistence errors must not leak raw infrastructure details to callers. Translate them into domain-appropriate messages.

| Scenario | Client message example |
|---|---|
| Resource not found | "Resource not found" |
| DB unavailable | "Provider is temporarily unavailable" |
| DB read timeout | "Lookup timed out" |
| Mapper or data inconsistency | "Configuration could not be resolved" |

---

## 23. Security, observability, audit, tenancy

- Treat these as separate cross-cutting capabilities.
- Add them through wrappers, interceptors, or dedicated modules.
- Do not hardwire them into core business flow unless they are part of the contract.
- Each capability is independently optional. When present, it must be testable and enforceable.
- Tenancy is a wrapper around existing contracts — core services are never rewritten for tenant logic. Isolation is applied at lookup, resolution, validation, and dispatch boundaries.

---

## 24. Comments and documentation

- Comment business rules and non-obvious decisions near the enforcement point.
- Do not comment obvious syntax.
- Keep documentation close to the enforced rule.
- When code changes, update the docs in the same change.
- Wrong comments are worse than no comments.
- Every public type gets a Javadoc — classes, interfaces, enums, and records.
- Use section comments only in genuinely long classes. Remove decorative banners where package structure and method visibility already communicate the same information.

---

## 25. Immutability and defensive copying

- Use records for immutable DTOs, commands, results, and value objects.
- Record compact constructors should use `Objects.requireNonNull` for mandatory fields and null-coalescence for optional fields.
- All collection fields should be defensively copied with `List.copyOf()` or `Map.copyOf()` at construction.
- Callers must not be able to mutate any object after construction.
- Null collections should default to empty collections, not remain null.

---

## 26. Test categories

Testing is a first-class engineering standard. Every module must define the smallest useful test set for its responsibility, and cross-module behavior must be verified through integration tests.

### 26.1 Unit tests

Use unit tests for pure logic with no Spring container and no network or database dependency.

Cover: value objects, normalization rules, defaulting rules, utility classes, mapping logic, policy decisions, error-code selection, retry classification, masking and formatting helpers.

### 26.2 Contract tests

Every public API and SPI must have contract-level tests.

Cover: public API behavior, SPI expectations for implementations, replacement and back-off behavior of defaults, additive extension points where multiple beans are collected and applied.

### 26.3 Validation tests

Validation must be tested by layer, not as one mixed bucket.

Cover: request shape validation, command invariants, startup configuration validation, runtime policy validation, final business invariants before side effects.

### 26.4 Configuration binding tests

Every configuration properties class must have binding tests.

Cover: defaults, override behavior, enum binding, duration binding, invalid configuration rejection, absence of deprecated or unsupported flags.

### 26.5 Auto-configuration tests

Every auto-configured module must have slice tests that prove: the default bean is created when expected, the default backs off when a consumer bean exists, activation conditions work correctly, optional integrations activate only when their dependencies are present.

### 26.6 Integration tests

Integration tests must verify real component collaboration across module boundaries.

Cover: full operation path, provider resolution, message construction, transport invocation, exception translation, startup validation, end-to-end HTTP flow where a web layer exists.

### 26.7 Web and handler tests

Where an HTTP module exists, add focused tests for: controller request/response behavior, validation error responses, exception-to-HTTP mapping, error response shape, status-code correctness.

### 26.8 Persistence tests

Where persistence exists, test: repository lookup behavior, read-only boundaries, mapper correctness, schema assumptions, failure translation.

Read-only modules must prove they do not expose mutation behavior.

### 26.9 Transport tests

Every transport adapter must have tests for: success path, timeout translation, authentication failure, TLS failure, connection failure, generic send failure, mandatory timeout wiring.

The transport layer is required to translate low-level failures into typed internal exceptions.

### 26.10 Retry and resilience tests

Where retry, throttling, timeout, or circuit breaking exists, test each capability separately.

Cover: retriable vs non-retriable failures, max attempts, delay behavior, generic-failure toggle, semaphore/bulkhead enforcement, per-operation timeout, batch timeout, circuit open/half-open/closed behavior.

### 26.11 Async and concurrency tests

Async code must be tested for correctness, not just completion.

Cover: task execution path, interrupt restoration, cancellation behavior, rejected execution behavior, timeout behavior, context propagation, cleanup after execution.

### 26.12 Batch and privacy tests

Any fan-out or bulk-send behavior must have dedicated tests.

Cover: one-item-per-recipient behavior, no recipient leakage, per-recipient success/failure aggregation, deduplication precedence, empty-input rejection, partial-success reporting.

### 26.13 Logging tests

Logging behavior is testable and should be tested where it carries business or operational meaning.

Cover: required structured fields, masking of PII, correct log levels, no duplicate stack-trace logging, correlation identifiers, batch identifiers.

### 26.14 Cross-cutting capability tests

Where observability, health, security, audit, idempotency, tenancy, or durability modules exist, each must have dedicated tests.

**Observability and health:** metric emission, expected tags, tracing/span creation, MDC trace bridging, health indicator status, readiness gating.

**Security:** unauthenticated access rejection, authorized access success, wrong-role rejection, per-resource authorization rules, API key handling, JWT handling.

**Audit:** audit emission on success and failure, required audit fields, append-only behavior.

**Idempotency:** first request success, duplicate request reuse/block behavior, cache/store correctness, TTL expiry, key derivation.

**Tenancy:** tenant isolation, tenant-scoped lookup, tenant-scoped validation, async tenant-context propagation, composite-key persistence lookups.

**Durability:** write-before-send flow, recovery after crash/restart, retry of pending items, cleanup/retention behavior.

### 26.15 Non-functional and production-safety tests

For modules with operational impact, include targeted tests for: startup fail-fast behavior, degraded-mode behavior, readiness before/after initialization, production-safe defaults, boundary conditions under load.

---

## 27. Test tooling stack

This standard assumes a **Spring MVC / servlet-first application style**. The choice of MockMvc over WebTestClient and the inclusion of Selenium for browser testing follow from that assumption. Reactive-stack projects should adapt accordingly.

### Default stack

Use `spring-boot-starter-test` as the base, plus:

- **Mockito** — default mocking tool
- **Selenium** — browser end-to-end tests only
- **Spring Security Test** — where security exists
- **`ApplicationContextRunner`** — for auto-configuration tests

Do **not** include **WebTestClient** in the standard stack. MockMvc is the standard web test tool for servlet-based applications.

### Tool-per-category mapping

| Category | Tools |
|---|---|
| Pure unit tests | JUnit 5 + AssertJ + Mockito |
| Contract tests | JUnit 5 + AssertJ + Mockito, Spring Test when bean wiring matters |
| Validation tests | JUnit 5 + AssertJ + Spring Validation + Mockito |
| Configuration binding | `ApplicationContextRunner` + JUnit 5 + AssertJ |
| Auto-configuration | `ApplicationContextRunner` + JUnit 5 + AssertJ |
| Spring integration | `@SpringBootTest` + JUnit 5 + AssertJ |
| Web/controller | MockMvc + `@WebMvcTest` + JUnit 5 + AssertJ + Mockito |
| Persistence | `@DataJpaTest` + JUnit 5 + AssertJ |
| JSON serialization | `@JsonTest` + JUnit 5 + AssertJ |
| REST client | `MockRestServiceServer` + Spring Test + JUnit 5 + AssertJ |
| Security | Spring Security Test + MockMvc + JUnit 5 + AssertJ |
| Logging | JUnit 5 + AssertJ + Logback test appender or equivalent |
| Async/concurrency | JUnit 5 + AssertJ + Mockito + Spring integration where executor wiring matters |
| Batch/privacy | JUnit 5 + AssertJ + Mockito + `@SpringBootTest` where cross-module behavior matters |
| Cross-cutting capabilities | JUnit 5 + AssertJ + Mockito + `@SpringBootTest` + `@DataJpaTest` where persistence is involved |
| Browser end-to-end | Selenium + JUnit 5 + AssertJ |

### Tool usage rules

**Mockito:** Use for unit tests, slice tests, contract tests, collaborator isolation. Do not use for repository correctness, full Spring wiring validation, or browser tests.

**Selenium:** Use only for real browser workflows, UI journey validation, and end-user interaction paths. Do not use for controller, service, repository, or configuration tests.

**MockMvc:** Use for servlet controller tests, exception handler tests, request/response assertions, and security behavior at the MVC layer.

**`ApplicationContextRunner`:** Use for auto-configuration tests, bean-condition tests, and property-driven setup tests.

---

## 28. Testing rules

- Prefer many small, focused tests over a few oversized integration tests.
- Test behavior and invariants, not private implementation details.
- Every bug fix should add or strengthen a test.
- Every replaceable module should prove both default behavior and override/back-off behavior.
- Cross-cutting capabilities such as retry, async, observability, security, audit, idempotency, and tenancy must each have dedicated tests when present.
- Integration tests should cover the real seams where failures matter most.
- Tests that verify logging, correlation, tracing, or MDC-dependent service flows should set tracking IDs in MDC before invocation and clear MDC afterward.
- Tracking ID tests verify behavior (operation succeeds, no "untraced" path), not MDC thread-local state.
- Module audits should check for tracking ID generation violations — no UUID, MDC, or tracking ID creation in downstream modules.

---

*All decisions were deliberated and locked.*
*Changes to any locked decision require explicit agreement and an update to this document before implementation.*
