# TwinMapper — Grouped Modules for Development

---

## Build-Time Pipeline

### Phase 1 — Foundation

- `twinmapper-core`
- `twinmapper-definition-model`

### Phase 2 — Definition Readers

- `twinmapper-format-yaml`
- `twinmapper-format-json`
- `twinmapper-format-bpmn`

### Phase 3 — Code Generation

- `twinmapper-codegen`

---

## Runtime Platform

### Phase 4 — Runtime Base

- `twinmapper-runtime`

### Phase 5 — Runtime Engines

- `twinmapper-runtime-binding`
- `twinmapper-runtime-objectmap`

### Phase 6 — Validation

- `twinmapper-validation`

---

## Consumption Layer A — Build Consumption

### Phase 7 — Build Plugins

- `twinmapper-gradle-plugin`
- `twinmapper-maven-plugin`

### Phase 9 — CLI

- `twinmapper-cli`

---

## Consumption Layer B — Application Consumption

### Phase 8 — Spring Boot Starter

- `twinmapper-spring-boot-starter`

---

## Optional Extensions

### Phase 10 — Optional Modules

- `twinmapper-annotations`
- `twinmapper-annotation-processor`
- `twinmapper-runtime-compat`

---

## Dependency Spine

**Primary:**

`core → definition-model → format-* → codegen → runtime → runtime-binding → runtime-objectmap → validation → spring-boot-starter`

**Orthogonal:**

`codegen → gradle-plugin / maven-plugin / cli`
