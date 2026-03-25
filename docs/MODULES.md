# TwinMapper — Module Definitions

All modules are part of the default TwinMapper product shape unless marked optional. The platform is split into five families: Core, Runtime, Format, Tooling, and Integration.

---

## Core Group

### `twinmapper-core`

**Role:** Shared foundation for all other modules.

**Responsibilities:**
- Common error model and result model
- Shared constants and base exceptions
- Base SPI interfaces shared across all modules
- Shared diagnostics structures and error codes
- Common utility helpers and marker interfaces
- Path and reference abstractions used by validation and binding
- Shared policy enums and configuration types

**Spring utilities used:**
- `Assert`, `StringUtils`, `ObjectUtils`, `ClassUtils` from `spring-core`
- `AnnotationUtils`, `ReflectionUtils` for shared reflection access
- `ResourceUtils`, `PathMatchingResourcePatternResolver` for resource loading
- `OrderComparator`, `PriorityOrdered`, `@Order`, `Ordered` for deterministic SPI ordering

**Constraints:**
- No Spring container dependency
- No Spring application context required
- Pure Java contracts and utilities only

---

### `twinmapper-definition-model`

**Role:** Canonical internal meta-model. The single source of truth for all generated code and runtime behavior.

**Responsibilities:**
- Represent all definition inputs in one canonical internal form
- Hold all document-type definitions, mapping definitions, constraints, aliases, defaults, version metadata, and generation options

**Contains:**
- `DefinitionSet`
- `TypeDefinition`, `ObjectTypeDefinition`, `EnumTypeDefinition`
- `FieldDefinition`, `CollectionDefinition`, `MapDefinition`
- `AliasDefinition`, `ConstraintDefinition`, `ConditionalConstraintDefinition`
- `DefaultValueDefinition`, `DeprecationDefinition`, `ForbiddenFieldDefinition`
- `VersionDefinition`
- `ObjectMappingDefinition`, `ProfileDefinition`, `ConverterDefinition`
- `GenerationOptionsDefinition`

**Covers all constraint types:**
- required, optional, default
- aliases, deprecated aliases, forbidden fields
- one-of, mutually exclusive, min, max, minLength, maxLength, pattern/regex
- conditional requirements, mapping definitions, inverse mapping declarations
- reusable profiles, converter references, null policies, version metadata

**Constraints:**
- Format-neutral — independent of YAML, JSON, and BPMN syntax
- No Spring dependency
- Jackson-based definition reading lives in format modules, not here

---

### `twinmapper-codegen`

**Role:** Build-time code generation engine. Consumes the internal definition model and emits Java source code.

**Responsibilities:**
- Generate DTOs from definitions
- Generate enums from declared values
- Generate metadata descriptor classes
- Generate document binder classes for YAML, JSON, and BPMN
- Generate schema validator classes
- Generate registry classes
- Generate object mapper classes (create, update, patch, inverse)
- Generate collection and nested mappers

**Generated output rules:**
- Immutable DTOs by default; Java records where suitable
- Mutable classes generated only when generation options require it
- Lombok-free generated output
- Java-friendly field names with original source names preserved in metadata
- Optional `@Component` on generated mapper and validator beans when Spring bean mode is enabled
- `@Qualifier`, `@Primary`, `@Conditional` support for generated bean disambiguation
- `basePackage` configuration option to align with consuming application package structure

**Constraints:**
- Build-time only — no Spring runtime dependency
- Source writer tool is an open implementation decision (Spring-neutral, actively maintained tool required)

---

### `twinmapper-validation`

**Role:** Validation contracts, generated validator execution, and aggregated reporting.

**Responsibilities:**
- Run generated schema validators
- Aggregate path-aware validation results into `ValidationReport`
- Expose reusable validation report model
- Provide validation contracts usable by runtime binding and object mapping flows
- Support handwritten semantic validators outside generated schema validation

**Covers:**
- Required fields, optional fields, default values
- Enum validation, forbidden fields, deprecated field handling
- Conditional requirements, one-of rules, mutually exclusive rules
- Min, max, minLength, maxLength, regex/pattern
- Unknown field validation, collection and map type checks
- Path-aware aggregated error reporting

**Spring-native integration:**
- `Validator`, `SmartValidator`, `Errors`, `BindingResult`
- `WebDataBinder`, `LocalValidatorFactoryBean`, `MethodValidationPostProcessor`
- `ConstraintValidatorFactory` for Spring-injected custom constraint validators
- `HandlerMethodArgumentResolver` for Spring MVC method parameter binding
- `ResponseEntityExceptionHandler` alignment for `MethodArgumentNotValidException` and `ConstraintViolationException`
- `@Valid`, `@Validated`, `jakarta.validation` JSR-380

---

## Runtime Group

### `twinmapper-runtime`

**Role:** Shared-base runtime module.

**Locked definition:** `twinmapper-runtime` is a shared-base runtime module containing contracts, common utilities, and shared abstractions only. It contains no format-specific binding logic and no object-mapping implementations. Those live in `twinmapper-runtime-binding` and `twinmapper-runtime-objectmap`.

**Contains:**
- `TwinMapperRuntime` facade interface
- `BindingContext`, `MappingContext`
- Registry interfaces for binders, mappers, converters, and metadata
- `AliasResolutionPolicy` shared contract
- `CompatibilityMode` enum (STRICT, COMPATIBLE, LENIENT)
- `TwinMapperRuntimeConfigurer` interface for Spring-idiomatic programmatic customization
- Common runtime policies and result models
- Shared diagnostic transport models

**Constraints:**
- Pure Java contracts
- Optional Spring integration hooks only — must not require Spring container

---

### `twinmapper-runtime-binding`

**Role:** Document engine runtime. Executes generated binders against parsed runtime documents.

**Responsibilities:**
- Dispatch to generated binder classes
- Orchestrate `NodeCursor` traversal abstraction
- Accumulate binding errors into `BindingResult`
- Enforce required fields, defaults, and alias resolution
- Report path-aware diagnostics

**Core features:**
- YAML document binding
- JSON document binding
- BPMN XML document binding
- Nested object binding
- Collection and map binding
- Alias-aware field binding
- Default value population
- Required-field enforcement
- Strict unknown-field behavior
- Path-aware error collection

**Spring integration:**
- `PathMatchingResourcePatternResolver` with Ant-style patterns for definition directory scanning
- `ResourceLoader` mandatory for all classpath resource access
- `Environment` for property-backed definition resolution

**Optional features housed here (disabled by default):**
- Alias/fallback compatibility resolution
- Bounded runtime discovery helpers
- Legacy-name compatibility resolution

All optional features in this module are explicit opt-in, policy-controlled, and auditable.

---

### `twinmapper-runtime-objectmap`

**Role:** Object engine runtime. Executes generated object mappers across Java layer boundaries.

**Responsibilities:**
- Dispatch create, update, patch, and inverse mappers
- Registry-based mapper lookup
- Apply converters and mapping policies
- Support all named mapping flows

**Supported mapping flows:**
- DTO ↔ DTO
- Entity ↔ Domain
- Domain ↔ DTO
- Entity ↔ DTO
- Entity ↔ Projection
- Request/Command ↔ Domain
- Domain ↔ Response/View/Event
- Patch DTO ↔ Entity and Domain

**Spring integration:**
- `ConversionService`, `GenericConversionService`, `FormattingConversionService`, `DefaultConversionService` as converter backend in Spring applications
- `Converter`, `ConverterFactory`, `GenericConverter`, `ConditionalGenericConverter` as SPI contracts for custom converters
- `BeanWrapper` for UPDATE and PATCH mode on Spring-managed bean targets — handles nested property paths, type conversion, and JPA-loaded proxy targets
- `DirectFieldAccessor` for targets without setters (value objects with final fields)
- `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, `ProxyUtils.getUserClass()` for proxy-safe reflection — mandatory for correctness on Spring-managed proxied beans

**Optional features housed here (disabled by default):**
- Controlled convention-based mapping mode via `ConventionMappingResolver` and `ConventionMatchingPolicy`
- Reflection-based compatibility helpers for migration and onboarding use cases

---

## Format Group

### `twinmapper-format-json`

**Role:** JSON definition reader (build-time) and JSON document parser (runtime).

**Responsibilities:**
- Load fixed JSON definitions into internal meta-model at build time
- Parse JSON runtime documents into `NodeCursor` abstraction
- Provide JSON-specific diagnostics

**Tools and integration:**
- Jackson `ObjectMapper` and `JsonMapper.builder()` (modern Jackson 2.12+ API)
- `Jackson2ObjectMapperBuilderCustomizer` registered by starter for field-naming consistency between TwinMapper DTOs and Spring Boot's auto-configured `ObjectMapper`
- `@JsonNaming`, `@JsonAlias`, `PropertyNamingStrategies` on generated DTOs for kebab-case to camelCase alignment
- `MappingJackson2HttpMessageConverter` as the integration point when TwinMapper DTOs are used in Spring MVC REST flows
- Zero additional dependency cost in Spring Boot

---

### `twinmapper-format-yaml`

**Role:** YAML definition reader (build-time) and YAML document parser (runtime).

**Responsibilities:**
- Load TwinMapper-native YAML DSL definitions at build time
- Load YAML object-mapping DSL definitions at build time
- Parse YAML runtime documents into `NodeCursor` abstraction
- Provide YAML-specific diagnostics

**Tools and integration:**
- SnakeYAML with mandatory safe construction: `new Yaml(new SafeConstructor(new LoaderOptions()))` — default SnakeYAML constructor allows arbitrary Java object deserialization and must never be used
- `YamlPropertiesFactoryBean`, `YamlMapFactoryBean` as alignment references for Spring Boot YAML loading model
- `ConfigDataLoader` and `ConfigDataLocationResolver` for `spring.config.import` support in Spring Boot 2.4+ — allows `spring.config.import=twinmapper:classpath:/definitions/`

---

### `twinmapper-format-bpmn`

**Role:** BPMN support definition reader (build-time) and BPMN XML parser (runtime).

**Locked decision:** No Camunda dependency. No third-party BPMN library. JDK StAX only.

**Responsibilities:**
- Load BPMN support model definitions at build time
- Parse BPMN XML runtime documents into `BpmnIntermediateModel`
- Map `BpmnIntermediateModel` into TwinMapper's internal definition model (build time) or `NodeCursor` (runtime)
- Provide BPMN-specific diagnostics

**Internal two-layer design:**
- `BpmnDocumentParser` — StAX-based reader, produces `BpmnIntermediateModel`
- `BpmnIntermediateModelMapper` — maps intermediate model into internal definition model or NodeCursor

**Supported BPMN vocabulary:**
- Process, SubProcess, CallActivity
- Task variants (UserTask, ServiceTask, ScriptTask, etc.)
- Gateway variants (Exclusive, Inclusive, Parallel, Event-based)
- Start/End/Intermediate events and their event definitions
- SequenceFlow, BoundaryEvent
- Extension elements and attributes
- LaneSet (metadata only)

**Security requirements:**
- `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES` must be set to `false`
- `XMLInputFactory.SUPPORT_DTD` must be set to `false`
- Both must be set — omitting either is a security vulnerability

**Spring integration:**
- `ResourceLoader` mandatory (not optional) for all BPMN classpath file resolution — required for correctness in OSGi, GraalVM native image, and application server deployments

---

## Tooling Group

### `twinmapper-gradle-plugin`

**Role:** Gradle build integration for TwinMapper code generation.

**Responsibilities:**
- Scan configured definition directories
- Validate definitions before generation
- Invoke codegen and produce Java sources
- Register generated source directory via `sourceSets.main.java.srcDir(generatedDir)`
- Support deterministic and CI-friendly build output
- Support dry-run and diagnostics options

**References:**
- Gradle Plugin API and Worker API
- Pkl Gradle plugin as structural reference pattern for build wiring and drift detection — not a dependency

---

### `twinmapper-maven-plugin`

**Role:** Maven build integration for TwinMapper code generation.

**Responsibilities:**
- Bind to `generate-sources` lifecycle phase
- Validate definitions before generation
- Invoke codegen and produce Java sources
- Register generated source directory via `project.addCompileSourceRoot(generatedDir)`
- Support dry-run and validation-only goals

**References:**
- Maven Plugin API and `AbstractMojo`
- jsonschema2pojo Maven plugin as narrow build-lifecycle reference — not a dependency

---

### `twinmapper-cli`

**Role:** Developer tooling only. Not a runtime component. Not involved in document binding or object mapping at runtime.

**Responsibilities:**
- Validate definition files
- Dry-run code generation
- Inspect and report on generated output
- Definition linting and diagnostics
- Offline inspection helpers

**Optional dev tooling housed here:**
- Sample-to-definition assistant — inspects sample YAML/JSON/BPMN inputs and proposes draft TwinMapper definitions; requires human review; never automatic live schema learning; CLI flag activation only

**Spring Boot CLI integration:**
- `SpringApplication.setWebApplicationType(WebApplicationType.NONE)` mandatory — prevents embedded servlet container startup during CLI execution
- `ApplicationRunner`, `CommandLineRunner`, `ExitCodeGenerator`, `SpringApplication.exit()` as the concrete CLI application contracts
- Picocli Spring Boot starter (`info.picocli:picocli-spring-boot-starter`) if Picocli is chosen — integrates `@Command` methods as Spring beans

---

## Integration Group

### `twinmapper-spring-boot-starter`

**Role:** First-class Spring Boot integration layer. Auto-configures the full TwinMapper runtime in a Spring Boot application.

**Responsibilities:**
- Auto-configure `TwinMapperRuntime` bean
- Register binder registry and object mapper registry as beans
- Wire Spring `ConversionService` into TwinMapper converter registry
- Wrap generated validators as Spring `Validator` implementations
- Expose optional metadata endpoints via Actuator
- Provide test support utilities

**Auto-configuration:**
- `@AutoConfiguration`
- `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
- `@ConditionalOnMissingBean(TwinMapperRuntime.class)` references interface type — any custom implementation suppresses auto-configured one
- `@EnableConfigurationProperties(TwinMapperProperties.class)`
- `AutoConfiguration.imports` for Spring Boot 3.x/4.x — `spring.factories` is legacy
- `spring-configuration-metadata.json` under `META-INF/` for IDE auto-completion of all `twinmapper.*` properties

**TwinMapperProperties (`@ConfigurationProperties(prefix = "twinmapper")`):**
- Strictness mode
- Definition locations
- Generation options
- Optional feature flags
- Compatibility toggles

**Named application events:**
- `TwinMapperBindingSuccessEvent`
- `TwinMapperBindingFailureEvent`
- `TwinMapperMappingSuccessEvent`
- `TwinMapperMappingFailureEvent`

**Actuator integration:**
- `InfoContributor` — exposes registered mapper counts, supported formats, strictness mode
- `HealthIndicator` — reports whether all required definition files loaded successfully

**Test support:**
- `@ImportAutoConfiguration(TwinMapperAutoConfiguration.class)` for Spring Boot test slice inclusion
- `@AutoConfigureTwinMapper` test annotation following the `@AutoConfigureMockMvc` pattern
- `spring-boot-test-autoconfigure` declared as test-scoped dependency

**Explicit activation annotations (for non-Boot or controlled Spring contexts):**
- `@EnableTwinMapper`
- `@EnableTwinMapperBinding`
- `@EnableTwinMapperObjectEngine`

---

## Optional Modules

### `twinmapper-annotations` *(optional)*

Annotation definitions only. No processor logic, no runtime mapping logic.

Annotations: `@TwinMap`, `@TwinField`, `@TwinIgnore`, `@TwinConverter`, `@TwinNullPolicy`, `@TwinVersion`.

Pure Java annotations. No dependencies. Activated via `twinmapper.annotations.enabled=true`.

---

### `twinmapper-annotation-processor` *(optional)*

APT processor that bridges TwinMapper annotations into the internal definition model. Feeds the same codegen pipeline as YAML definitions. Definition files remain the primary source of truth — annotations are convenience only.

Tools: Java Annotation Processing API, Google AutoService for processor registration.

---

### `twinmapper-runtime-compat` *(optional)*

Separate module for reflection-based compatibility helpers and legacy bridging utilities.

Keeps optional compatibility behavior out of the main `twinmapper-runtime-objectmap` path.

Contains: migration helper mapper, dev-time compatibility copier, bounded reflection-assisted legacy object reader, legacy format bridge utilities.

Rules: never on the default classpath, explicit dependency declaration required, must not replace generated mappers as the primary path, all reflection use is bounded and auditable, proxy-safe via `AopUtils`/`AopProxyUtils`/`ProxyUtils`.

---

## Module Dependency Overview

```
twinmapper-core
  └── twinmapper-definition-model
        ├── twinmapper-codegen
        ├── twinmapper-format-json
        ├── twinmapper-format-yaml
        └── twinmapper-format-bpmn

twinmapper-runtime (shared-base)
  ├── twinmapper-runtime-binding
  │     └── twinmapper-format-json
  │     └── twinmapper-format-yaml
  │     └── twinmapper-format-bpmn
  └── twinmapper-runtime-objectmap

twinmapper-validation
  └── twinmapper-runtime-binding (error model)

twinmapper-spring-boot-starter
  ├── twinmapper-runtime-binding
  ├── twinmapper-runtime-objectmap
  └── twinmapper-validation

twinmapper-gradle-plugin
  └── twinmapper-codegen

twinmapper-maven-plugin
  └── twinmapper-codegen

twinmapper-cli
  └── twinmapper-codegen (dry-run)
  └── twinmapper-validation
```