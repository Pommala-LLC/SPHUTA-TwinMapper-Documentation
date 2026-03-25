# TwinMapper — Testing Strategy

## Overview

This document defines how TwinMapper itself is tested (internal testing strategy) and how consuming applications should test their TwinMapper-based code (consumer testing guidance). It covers all modules across the entire TwinMapper platform: foundation, format readers, code generation, runtime engines, validation, build tooling, Spring Boot integration, and optional extensions.

---

## Internal Testing Principles

| Principle | Rationale |
|-----------|-----------|
| Every generated artifact must be covered by a round-trip test | Confirms actual field values, defaults, and alias resolution rather than only asserting internal structure |
| No test should rely on reflection to verify generated behavior | Generated code is reflection-free by design; tests must mirror that contract |
| Security-sensitive paths must have explicit security tests | BPMN XXE rejection and SnakeYAML safe construction are non-negotiable security controls |
| SPI ordering must be verified by tests that register multiple implementations | Deterministic ordering via `OrderComparator` and `@Order` is a core platform guarantee |
| All Spring integration points must have Spring Boot integration tests | `@SpringBootTest` with `@AutoConfigureTwinMapper` is the required verification mechanism |
| Optional features must be tested in both states | Disabled by default and enabled by property — both paths require explicit coverage |
| Codegen determinism must be enforced in CI | Identical inputs must always produce identical outputs; byte-for-byte comparison on repeated runs |
| Immutability contracts are explicit, not implicit | Records and collection-bearing model objects must verify defensive copying and unmodifiable collection behavior |

---

## Test Categories

TwinMapper defines sixteen test categories organized into seven implementation buckets. Each category maps to specific modules, behaviors, and verification requirements.

### Category 1 — Unit Tests

Pure no-I/O, no-Spring tests for isolated logic across all modules. This is the first and broadest test layer.

| Module | What to Unit Test |
|--------|-------------------|
| `twinmapper-core` | Diagnostic codes, diagnostic collector, exception hierarchy, naming types (`Namespace`, `QualifiedName`, `TypeReference`), path types (`FieldPath`, `PathSegment`), source location, SPI enums, utility classes (`FieldNameUtils`, `Preconditions`, `TwinMapperConstants`) |
| `twinmapper-definition-model` | Meta-model construction, equality, field validation, `DefinitionSet` and `DefinitionSetId` identity, `GenerationOptionsDefinition`, `ObjectTypeDefinition`, `EnumTypeDefinition`, `EnumValueDefinition`, sealed hierarchy enforcement, `FieldDefinition`, `CollectionDefinition`, `DeprecationDefinition`, `ForbiddenFieldDefinition`, constraint definitions, mapping definitions, `MapperMode`, `NullPolicy`, `UnmappedTargetPolicy`, `ProfileDefinition`, `ConverterDefinition` |
| `twinmapper-codegen` | DTO shape from a given `DefinitionSet`, enum values, binder structure, mapper structure, registry structure, metadata descriptor correctness |
| `twinmapper-validation` | Each constraint type independently, conditional rules, one-of constraints, mutually exclusive constraints, aggregate `ValidationReport` behavior |
| `twinmapper-format-yaml` | YAML DSL parsing, safe construction enforcement, alias resolution, DSL construct acceptance and rejection |
| `twinmapper-format-json` | JSON definition reading, field mapping, error reporting |
| `twinmapper-format-bpmn` | BPMN intermediate model correctness, XXE rejection, supported element parsing, unsupported element rejection |
| `twinmapper-runtime-binding` | `NodeCursor` behavior, binder dispatch, error accumulation, strictness mode behavior |
| `twinmapper-runtime-objectmap` | Mapper dispatch, null policies, patch semantics, converter invocation |

### Category 2 — Integration Tests

Tests that verify module interactions and Spring context wiring using `@SpringBootTest` and `@AutoConfigureTwinMapper`.

```java
@SpringBootTest(classes = TwinMapperTestApplication.class)
@AutoConfigureTwinMapper
class CustomerBindingIntegrationTest {

    @Autowired
    TwinMapperRuntime twinMapperRuntime;

    @Test
    void shouldBindValidYamlDocumentToCustomerDto() throws Exception {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-valid.yaml");
        CustomerDto result = twinMapperRuntime.readYaml(yaml, CustomerDto.class);

        assertThat(result.id()).isEqualTo("CUST-001");
        assertThat(result.fullName()).isEqualTo("Jane Smith");
        assertThat(result.status()).isEqualTo(CustomerStatus.ACTIVE);
    }

    @Test
    void shouldRejectUnknownFieldsInStrictMode() {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-unknown-field.yaml");

        assertThatThrownBy(() -> twinMapperRuntime.readYaml(yaml, CustomerDto.class))
            .isInstanceOf(BindingException.class)
            .hasMessageContaining("UNKNOWN_FIELD");
    }
}
```

### Category 3 — Round-Trip Tests

Every generated binder must have a round-trip test verifying that a known valid document produces the expected DTO with correct field values, defaults, and alias resolution. These confirm actual runtime behavior rather than only asserting internal structure.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class CustomerRoundTripTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldRoundTripCustomerYaml() throws Exception {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-valid.yaml");
        CustomerDto result = runtime.readYaml(yaml, CustomerDto.class);

        assertThat(result.id()).isEqualTo("CUST-001");
        assertThat(result.fullName()).isEqualTo("Jane Smith");
        assertThat(result.status()).isEqualTo(CustomerStatus.ACTIVE);
        assertThat(result.address().city()).isEqualTo("Portland");
    }
}
```

### Category 4 — Security Tests

Mandatory for all parser modules. These tests are non-negotiable and must be present in every CI run.

**BPMN XXE Rejection:**

```java
@Test
void shouldRejectXxePayload() {
    String xxePayload = """
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
        <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
          <process id="&xxe;" />
        </definitions>
        """;

    assertThatThrownBy(() ->
        bpmnParser.parse(new ByteArrayInputStream(xxePayload.getBytes()), ParsingContext.defaults()))
        .isInstanceOf(DocumentParseException.class);
}
```

**SnakeYAML SafeConstructor Enforcement:**

```java
@Test
void shouldRejectArbitraryClassInstantiation() {
    String maliciousYaml = "!!com.example.SensitiveClass\n  secret: value";

    assertThatThrownBy(() -> yamlParser.parse(maliciousYaml))
        .isInstanceOf(YAMLException.class);
}
```

### Category 5 — SPI and Extension-Ordering Tests

Since TwinMapper has SPI-driven readers, parsers, registries, and runtime selection, ordering tests are required whenever multiple implementations can be registered.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class SpiOrderingTest {

    @Autowired
    List<RuntimeDocumentParser> parsers;

    @Test
    void shouldResolveHigherPriorityParserFirst() {
        assertThat(parsers.get(0)).isInstanceOf(CustomHighPriorityParser.class);
        assertThat(parsers.get(1)).isInstanceOf(DefaultJsonParser.class);
    }
}
```

SPI ordering is enforced through Spring's `OrderComparator`, `PriorityOrdered`, `@Order`, and `Ordered` interfaces. Tests must register at least two implementations and assert deterministic resolution order.

### Category 6 — Auto-Configuration and Starter Tests

For the Spring Boot starter, test slice activation, bean registration, and `AutoConfiguration.imports` correctness.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTwinMapper
class StarterAutoConfigurationTest {

    @Autowired
    ApplicationContext context;

    @Test
    void shouldRegisterBinderRegistry() {
        assertThat(context.getBean(GeneratedBinderRegistry.class)).isNotNull();
    }

    @Test
    void shouldRegisterMapperRegistry() {
        assertThat(context.getBean(TwinMapperObjectMapperRegistry.class)).isNotNull();
    }
}
```

A separate test must verify that `AutoConfiguration.imports` is correctly formed and all listed classes exist on the classpath.

### Category 7 — Feature-Flag and Optional-Feature Tests

For all optional features (`twinmapper.features.*`), always test both states: disabled by default and enabled by property.

```java
@SpringBootTest(properties = "twinmapper.features.convention-mapping=false")
@AutoConfigureTwinMapper
class ConventionMappingDisabledTest {

    @Test
    void shouldNotActivateConventionMapping() {
        // Assert convention mapping beans are absent
    }
}

@SpringBootTest(properties = "twinmapper.features.convention-mapping=true")
@AutoConfigureTwinMapper
class ConventionMappingEnabledTest {

    @Test
    void shouldActivateConventionMapping() {
        // Assert convention mapping beans are present and functional
    }
}
```

### Category 8 — Determinism Tests

Critical for code generation. The codegen module must produce byte-for-byte identical output across repeated runs with the same input.

```java
@Test
void shouldProduceDeterministicOutput() throws Exception {
    DefinitionSet input = loadFixtureDefinitionSet();

    String firstRun = codegen.generate(input);
    String secondRun = codegen.generate(input);

    assertThat(firstRun).isEqualTo(secondRun);
}
```

CI pipelines should run codegen twice per build and assert identical output.

### Category 9 — Fixture-Based Format Tests

Organize test fixtures by format and scenario:

```
src/test/resources/fixtures/
├── yaml/
│   ├── customer-valid.yaml
│   ├── customer-missing-required.yaml
│   ├── customer-unknown-field.yaml
│   ├── customer-deprecated-alias.yaml
│   └── customer-forbidden-field.yaml
├── json/
│   ├── order-valid.json
│   └── order-invalid-enum.json
└── bpmn/
    ├── simple-workflow.bpmn
    ├── xxe-payload.xml
    └── unsupported-element.bpmn
```

Fixture categories: valid documents, missing required fields, unknown fields, deprecated aliases, invalid enums, unsupported BPMN elements, and XXE payloads.

### Category 10 — Definition DSL Parsing Tests

For YAML, JSON, and BPMN definition readers: syntax acceptance, syntax rejection, import and reference handling, and semantic validation. Every new DSL construct requires both a positive parsing test and a negative parsing test.

### Category 11 — Definition-Model Validation Tests

Tests for `DefinitionModelValidator` and structural/meta-model rules: required fields, invalid combinations, cross-reference checks, typed constraint integrity, and mapping-definition consistency within the model itself.

| Scenario | Assertion |
|----------|-----------|
| Valid `DefinitionSet` passes with no errors | No ERROR diagnostics emitted |
| Duplicate object type names | ERROR diagnostic |
| Duplicate enum type names | ERROR diagnostic |
| Object type name collides with enum type name | ERROR diagnostic |
| Duplicate field names within one object type | ERROR diagnostic |
| Duplicate aliases within one field | ERROR diagnostic |
| Duplicate mapping names | ERROR diagnostic |
| Missing `DefinitionSetId` | ERROR diagnostic |
| Invalid namespace on id | ERROR diagnostic |
| Blank type name | ERROR diagnostic |
| Blank field name | ERROR diagnostic |
| Missing type reference on field | ERROR diagnostic |
| Blank enum value name | ERROR diagnostic |
| Duplicate enum value names | ERROR diagnostic |
| Missing source type on mapping | ERROR diagnostic |
| Missing target type on mapping | ERROR diagnostic |
| Duplicate field mapping target paths | ERROR diagnostic |
| Valid complex model passes cleanly | Multiple types, enums, mappings — all valid, no diagnostics |

### Category 12 — Code Generation Contract Tests

For DTO shapes, enum values, binder classes, validator classes, mapper classes, registry classes, and package/base-package behavior. These verify the codegen contract defined in `CODEGEN_CONTRACT.md`.

| Artifact | Verification |
|----------|-------------|
| DTO class | Record or mutable class shape, field names, types, `@Nullable` annotations, Lombok-free |
| Enum class | Values match definition, naming convention applied |
| Metadata descriptor | `REQUIRED_FIELDS`, `DEFAULTS`, `FORBIDDEN_FIELDS`, `ALIASES_*`, source field names |
| Document binder | `NodeCursor` traversal, alias resolution, required and optional field handling, nested binding |
| Schema validator | Each constraint type, error code correctness, `ValidationReport` structure |
| Create mapper | Source-to-target field mapping, converter invocation |
| Update mapper | Non-null overwrite behavior, null preservation |
| Patch mapper | Non-null-only application semantics |
| Inverse mapper | Reverse direction correctness |
| Binder registry | Lookup by target type, `NoBinderFoundException` on missing |
| Mapper registry | Lookup by source and target type pair |

### Category 13 — Binding Tests

For the runtime binding engine: required fields, defaults, alias handling, unknown-field behavior, forbidden fields, enum conversion, nested binding, path-aware diagnostics, and strictness mode differences (STRICT, COMPATIBLE, LENIENT).

```java
@SpringBootTest
@AutoConfigureTwinMapper
class BindingBehaviorTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldApplyDefaultValues() {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-defaults.yaml");
        CustomerDto result = runtime.readYaml(yaml, CustomerDto.class);

        assertThat(result.status()).isEqualTo(CustomerStatus.ACTIVE); // default value
    }

    @Test
    void shouldResolveAliases() {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-alias.yaml");
        CustomerDto result = runtime.readYaml(yaml, CustomerDto.class);

        assertThat(result.fullName()).isEqualTo("Jane Smith"); // resolved from "name" alias
    }

    @Test
    void shouldProducePathAwareDiagnostics() {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-missing-required.yaml");

        BindingResult<CustomerDto> result = runtime.readYamlWithResult(yaml, CustomerDto.class);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.errors()).anyMatch(e ->
            e.fieldPath().toString().equals("id")
            && e.errorCode().equals("MISSING_REQUIRED_FIELD"));
    }
}
```

### Category 14 — Validation-Rule Tests

For min/max, minLength/maxLength, regex, conditional rules, one-of, mutually exclusive constraints, and aggregate `ValidationReport` behavior.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class ValidationRuleTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldRejectValueBelowMinimum() {
        OrderDto dto = new OrderDto(/* quantity: -1, below min of 0 */);

        ValidationReport report = runtime.validate(dto);

        assertThat(report.hasErrors()).isTrue();
        assertThat(report.errors()).anyMatch(e ->
            e.fieldPath().toString().equals("quantity")
            && e.errorCode().equals("MIN_VIOLATION"));
    }

    @Test
    void shouldEnforceOneOfConstraint() {
        PaymentDto dto = new PaymentDto(/* neither card nor bank set */);

        ValidationReport report = runtime.validate(dto);

        assertThat(report.hasErrors()).isTrue();
        assertThat(report.errors()).anyMatch(e ->
            e.errorCode().equals("ONE_OF_VIOLATION"));
    }
}
```

### Category 15 — Object-Mapping Tests

For create, update, patch, inverse flows, null-policy handling, converter use, mapper registry lookup, and collection and nested mapping behavior.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class ObjectMappingTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldNotOverwriteExistingValueWithNullInUpdateMode() {
        CustomerEntity existing = new CustomerEntity();
        existing.setId(1L);
        existing.setCustomerCode("CUST-001");
        existing.setStatusCode("active");

        Customer domain = new Customer(
            new CustomerId(1L),
            null,              // code is null in source
            CustomerStatus.SUSPENDED,
            null
        );

        CustomerEntity updated = runtime.update(domain, existing);

        assertThat(updated.getCustomerCode()).isEqualTo("CUST-001");    // preserved
        assertThat(updated.getStatusCode()).isEqualTo("suspended");     // updated
    }

    @Test
    void shouldApplyOnlyNonNullFieldsInPatchMode() {
        CustomerEntity existing = new CustomerEntity();
        existing.setId(1L);
        existing.setCustomerCode("CUST-001");
        existing.setStatusCode("active");

        CustomerPatchDto patch = new CustomerPatchDto(null, "suspended");

        CustomerEntity patched = runtime.patch(patch, existing);

        assertThat(patched.getCustomerCode()).isEqualTo("CUST-001");   // untouched
        assertThat(patched.getStatusCode()).isEqualTo("suspended");    // applied
    }
}
```

### Category 16 — Diagnostic and Error-Contract Tests

For emitted code, category, severity, path, and message-template correctness. TwinMapper has a frozen diagnostic vocabulary, so these contracts must be explicitly verified.

| Component | What to Test |
|-----------|-------------|
| `DiagnosticCode` | Every code has non-null `defaultSeverity()` and `category()`, no duplicate code names, all expected categories are covered, specific expected codes exist |
| `TwinMapperDiagnostic` | Builder produces complete record, builder rejects null required fields, `isError()` and `isWarning()` convenience methods, `category()` delegation to code, nullable fields accept null |
| `DiagnosticCollector` | Add and retrieve diagnostics, `hasErrors()` and `hasWarnings()` filtering, immutable snapshot, clear behavior, collector merging |
| Exception hierarchy | Each exception carries diagnostics, exception-to-diagnostic extraction, message formatting |

---

## Immutability Enforcement Tests

This is an explicit cross-cutting category, not a scattered assertion style. Records and collection-bearing model objects must verify defensive copying and unmodifiable collection behavior.

```java
@Test
void shouldReturnUnmodifiableFieldList() {
    ObjectTypeDefinition type = buildObjectTypeWithFields("field1", "field2");

    assertThatThrownBy(() -> type.fields().add(someField()))
        .isInstanceOf(UnsupportedOperationException.class);
}

@Test
void shouldDefensivelyCopyInputCollections() {
    List<FieldDefinition> mutableFields = new ArrayList<>(List.of(field1, field2));
    ObjectTypeDefinition type = ObjectTypeDefinition.builder()
        .name("Test")
        .fields(mutableFields)
        .build();

    mutableFields.add(field3);  // mutate original

    assertThat(type.fields()).hasSize(2);  // model unaffected
}
```

Since the canonical meta-model is record-heavy and immutable, these contracts must be verified at the foundation layer and maintained across all model types.

---

## Value-Object and Primitive Contract Tests

For small shared foundation types: `Namespace`, `QualifiedName`, `TypeReference`, `FieldPath`, `PathSegment`, `SourceLocation`, `SourcePosition`, and `DefinitionSetId`.

| Concern | Examples |
|---------|----------|
| Normalization | Leading/trailing whitespace stripped, case rules applied |
| Equality | Two instances with same content are equal; different content are not |
| Sentinel/default handling | Default or empty values behave correctly |
| Invalid-input rejection | Null, blank, and malformed inputs throw appropriate exceptions |
| Convenience factory methods | `FieldPath.of("a.b.c")` produces correct segment list |
| String representation | `toString()` produces human-readable, path-aware output |

---

## Implementation Buckets

For implementation planning, all sixteen categories group into seven practical buckets:

| Bucket | Categories Included | Primary Modules |
|--------|---------------------|-----------------|
| **Foundation tests** | Unit tests (core), value-object/primitive contracts, immutability enforcement | `twinmapper-core` |
| **Model tests** | Definition-model validation, diagnostic contract tests | `twinmapper-definition-model`, `twinmapper-core` |
| **Parser tests** | DSL parsing tests, fixture-based format tests, security tests | `twinmapper-format-yaml`, `twinmapper-format-json`, `twinmapper-format-bpmn` |
| **Codegen tests** | Code generation contract tests, determinism tests | `twinmapper-codegen` |
| **Binding tests** | Binding behavior tests, round-trip tests | `twinmapper-runtime-binding` |
| **Mapping and validation tests** | Object-mapping tests, validation-rule tests | `twinmapper-runtime-objectmap`, `twinmapper-validation` |
| **Platform tests** | Integration tests, SPI ordering, auto-configuration/starter tests, feature-flag tests | `twinmapper-spring-boot-starter`, all modules with Spring integration points |

---

## Test Slice Support

TwinMapper provides a dedicated `@AutoConfigureTwinMapper` annotation for use in Spring Boot test slices. This activates only the TwinMapper auto-configuration without triggering unrelated auto-configurations.

```java
// Minimum context for binding tests — no web layer, no database
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTwinMapper
class BindingOnlyTest {
    // Only TwinMapper beans loaded
}
```

---

## Test Requirements by Change Type

Every pull request must include tests for the changed behavior. The required tests vary by change type:

| Change Type | Required Tests |
|-------------|----------------|
| New definition DSL construct | DSL parsing test, codegen output test, binder behavior test |
| New validation constraint | Constraint positive test, constraint negative test, error message test |
| New object mapping feature | Create, update, and patch mode tests, null policy tests |
| Format module change | Format-specific parsing test, round-trip test |
| Security-sensitive change | XXE rejection test or SafeConstructor enforcement test |
| SPI change | SPI ordering test with multiple registrations |
| Starter/auto-config change | `@SpringBootTest` with `@AutoConfigureTwinMapper` integration test |
| New optional feature | Feature disabled by default test, feature enabled by property test |

### Test Coverage Requirements

- Unit tests for all new logic paths.
- Integration tests for all Spring context interactions.
- At least one round-trip test for any new binding or mapping capability.
- Security tests for any parser changes.

---

## CI Recommendations

| CI Gate | Trigger | Description |
|---------|---------|-------------|
| Unit tests | Every commit | Run all unit tests across all modules |
| Integration tests | Every pull request | Run all Spring Boot integration tests |
| Security tests | Every pull request | Run XXE rejection and SnakeYAML safe construction tests as part of the integration suite |
| Round-trip tests | Every pull request | Run round-trip tests for every supported format |
| Determinism check | Every pull request | Run codegen twice and assert byte-for-byte identical output |
| AutoConfiguration.imports verification | Every pull request | Verify the imports file is correctly formed and all listed classes exist on the classpath |
| Feature-flag tests | Every pull request | Run optional-feature tests in both enabled and disabled states |

---

## Consumer Testing Guidance

This section covers how consuming applications should test their TwinMapper-based code.

### Testing Binding in Consumer Applications

```java
@SpringBootTest
@AutoConfigureTwinMapper
class MyApplicationBindingTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldBindMyDomainDocument() throws Exception {
        InputStream yaml = getClass().getResourceAsStream("/my-domain-valid.yaml");
        MyDomainDto result = runtime.readYaml(yaml, MyDomainDto.class);

        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("expected-value");
    }
}
```

### Testing Validation in Consumer Applications

```java
@SpringBootTest
@AutoConfigureTwinMapper
class MyApplicationValidationTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldValidateMyDomainDto() {
        MyDomainDto dto = buildInvalidDto();

        ValidationReport report = runtime.validate(dto);

        assertThat(report.hasErrors()).isTrue();
    }
}
```

### Testing Object Mapping in Consumer Applications

```java
@SpringBootTest
@AutoConfigureTwinMapper
class MyApplicationMappingTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldMapDomainToEntity() {
        MyDomain domain = buildDomainObject();

        MyEntity entity = runtime.create(domain, MyEntity.class);

        assertThat(entity.getName()).isEqualTo(domain.name());
    }
}
```

### Testing Custom Validators

```java
@SpringBootTest
@AutoConfigureTwinMapper
class CustomValidationExtensionTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldApplyCustomValidationRule() {
        WorkflowDefinitionDto dto = buildWorkflowWithNoStates();

        ValidationReport report = runtime.validate(dto);

        assertThat(report.hasErrors()).isTrue();
        assertThat(report.errors()).anyMatch(e ->
            e.message().contains("at least one state"));
    }
}
```

### Testing Custom Converters

Register a custom `Converter<S, T>` in the Spring context and verify it is invoked during mapping:

```java
@SpringBootTest
@AutoConfigureTwinMapper
class CustomConverterTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldUseRegisteredConverter() {
        OrderDto dto = buildOrderDtoWithCustomStatus();

        OrderEntity entity = runtime.create(dto, OrderEntity.class);

        assertThat(entity.getStatusCode()).isEqualTo("expected-converted-value");
    }
}
```

---

## Module-to-Category Mapping

This table shows which test categories apply to each TwinMapper module.

| Module | Applicable Test Categories |
|--------|---------------------------|
| `twinmapper-core` | Unit, diagnostic contract, value-object/primitive, immutability |
| `twinmapper-definition-model` | Unit, definition-model validation, immutability |
| `twinmapper-format-yaml` | Unit, DSL parsing, fixture-based format, security (SafeConstructor) |
| `twinmapper-format-json` | Unit, DSL parsing, fixture-based format |
| `twinmapper-format-bpmn` | Unit, DSL parsing, fixture-based format, security (XXE) |
| `twinmapper-codegen` | Unit, codegen contract, determinism |
| `twinmapper-runtime` | Unit |
| `twinmapper-runtime-binding` | Unit, binding, round-trip, integration |
| `twinmapper-runtime-objectmap` | Unit, object-mapping, integration |
| `twinmapper-validation` | Unit, validation-rule, integration |
| `twinmapper-spring-boot-starter` | Auto-configuration/starter, feature-flag, integration, SPI ordering |
| `twinmapper-gradle-plugin` | Integration (build-tool-specific) |
| `twinmapper-maven-plugin` | Integration (build-tool-specific) |
| `twinmapper-cli` | Unit, integration |
| `twinmapper-annotations` | Unit, integration (annotation-processor) |
| `twinmapper-annotation-processor` | Unit, integration (annotation-processor) |
| `twinmapper-runtime-compat` | Unit, integration (opt-in only) |
