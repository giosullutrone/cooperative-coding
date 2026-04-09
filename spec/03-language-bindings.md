# CooperativeCoding Specification: Language Bindings

**Version 2.0.0**

---

## 1. Overview

The [Design Contract](01-design-contract.md) defines what every code element must carry. The [Sync](02-sync.md) defines the rules for keeping canvas and code aligned. This document defines the adapter: how the abstract CooperativeCoding specification maps to a specific programming language's idioms, conventions, and tooling.

A language binding is the translation layer between the canvas world and the code world. On the canvas, an element has design content: a responsibility statement, pseudo code, and whatever additional information the implementation provides. In code, the same element is a construct with a specific declaration syntax, documentation in a specific format, imports following a specific convention, and type annotations in a specific notation. The binding defines the exact mapping between these two representations.

Without a binding, sync cannot operate. It knows that design content must become code, but it cannot know the syntax, conventions, or idioms of the target language without a language-specific mapping. The binding is what makes the abstract spec concrete.

A _language binding_ has two parts: (1) a **binding specification**, a document describing the mapping rules, and (2) a **binding implementation**, executable code (parser, generator, type mapper) that sync uses at runtime. The spec document defines the contract; the implementation fulfills it.

This document defines the contract that any language binding MUST fulfill to be CooperativeCoding-compliant. It separates requirements into three tiers: MUST (required for compliance), SHOULD (recommended for quality), and MAY (optional enhancements), following [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keyword semantics as established in [Introduction](00-introduction.md).

---

## 2. Required Capabilities (MUST)

This section defines the non-negotiable capabilities that every CooperativeCoding language binding MUST provide.

### 2.1 Stereotype Mapping

Stereotypes refine what a code element means in the target language. A class with stereotype `protocol` means something very different in Python (a `typing.Protocol` subclass) than in TypeScript (where the concept maps to an `interface`). The binding is where these language-specific meanings are defined.

A binding MUST define a stereotype mapping table. This table MUST include, for each recognized stereotype:

| Column | Description |
|---|---|
| **Stereotype** | The string value (e.g., `"protocol"`, `"dataclass"`, `"enum"`). |
| **Language construct** | The concrete syntax or declaration pattern in the target language. |
| **Required imports** | Any import statements that the stereotype necessitates. |

The stereotype set is open. A binding defines its recognized stereotypes, but:

- A binding MAY define additional stereotypes at any time. Adding a stereotype is a backwards-compatible change.
- A binding SHOULD document the behavioral differences between stereotypes.

### 2.2 Source Code Parsing

Sync needs to read existing code to process design requests (code changes that update the canvas). The binding is responsible for defining how source files in the target language are parsed and what information is extracted.

A binding MUST be able to parse source files and extract:

- **Identifiable code elements:** classes, functions, methods, fields, modules, and other constructs with their qualified names.
- **Signatures:** parameter lists (name, type, default value), return types, modifiers (static, abstract, etc.).
- **Documentation:** content from the language's documentation format, including design content (responsibility, pseudo code) if present.
- **Relationships:** parent classes, implemented interfaces, composition via typed fields, imports, and other relationships that affect code generation.
- **Stereotypes:** inferred from language-specific markers (decorators, base classes, annotations).

### 2.3 Code Generation

Sync needs to write code from canvas elements when processing code requests (canvas changes that update the code). The binding is responsible for defining how canvas data translates to source code.

A binding MUST be able to generate:

- **Element declarations:** class, function, or module declarations with correct syntax for the target language, including stereotype-specific constructs.
- **Signatures:** method and function declarations with correct parameter types, return types, and modifiers.
- **Documentation:** design content (responsibility, pseudo code, and any additional content) mapped to the language's documentation format.
- **Relationship constructs:** inheritance declarations, import statements, typed fields, and other code that reflects tracked relationships.
- **Stub bodies:** appropriate placeholder bodies for unimplemented methods (e.g., `raise NotImplementedError` in Python, `throw new Error("Not implemented")` in TypeScript).

### 2.4 Documentation Mapping

The canvas element's design content carries the design intent. The binding defines how this content maps to the language's documentation format.

A binding MUST define the mapping for:

- **Responsibility statements:** how the responsibility description appears in the language's documentation format.
- **Pseudo code:** how pseudo code from a method's canvas content is represented in the generated code documentation.
- **Signature documentation:** how parameters, return types, and exceptions are documented.

---

## 3. Recommended Capabilities (SHOULD)

### 3.1 Import Resolution

The binding SHOULD define a complete import resolution strategy that maps qualified names to import statements: handling relative vs. absolute imports, standard library types, re-exports, and namespace packages.

### 3.2 Type Mapping

Canvas elements may use language-neutral type names (e.g., `List<T>`, `Map<K, V>`, `Optional<T>`). The binding SHOULD define how these map to the target language's concrete type syntax.

### 3.3 File Conventions

The binding SHOULD define:

- **File path conventions:** how an element's identity maps to file system paths.
- **Package initialization:** what files must be created for a new package or module.
- **Naming conventions:** how canvas element names transform to the target language's conventions.

### 3.4 Method Body Preservation

The binding SHOULD define how sync identifies the boundary between a method's signature and its body, so that sync can update signatures without overwriting method implementations. The binding SHOULD also define what constitutes a "stub" body (unimplemented placeholder) vs. an implemented body.

---

## 4. Optional Capabilities (MAY)

### 4.1 Test Framework Mapping

A binding MAY define how test elements map to the target language's testing conventions:

- **Test framework:** the default test framework and how test content maps to that framework's conventions.
- **Test file conventions:** where test files live, how they are named.
- **Test result format:** how test execution results are represented in canvas content, and how sync reads results from the framework's output.

### 4.2 Code Style Configuration

A binding MAY support language-specific formatting preferences. When supported, the binding SHOULD integrate with the language's existing formatting tools (e.g., Black/Ruff for Python, Prettier for TypeScript, rustfmt for Rust) rather than implementing its own formatter.

### 4.3 Multi-File Strategies

A binding MAY define strategies for how canvas elements map to file organization (one class per file, module-based grouping, feature-based grouping).

---

## 5. Sync Architecture

The spec is deliberately silent on deployment topology. Valid architectures include:

- A **standalone sync daemon** that watches file changes and runs sync automatically.
- A **canvas plugin component** where the plugin embeds sync logic and triggers it on canvas save.
- An **agent-integrated sync** where the AI agent invokes sync as part of the sync loop.
- A **CLI tool** that the user or agent invokes manually.

All architectures MUST implement the same sync semantics defined in [Sync](02-sync.md).

---

## 6. Phased Adoption

Binding authors MAY adopt a phased implementation approach:

**Phase 1: Code requests only.** Implement code generation from canvas elements (processing code requests). Requires: stereotype mapping, code generation, documentation formatting. Does not require: source code parsing or design request processing.

**Phase 2: Full sync loop.** Add source code parsing and design request processing (code changes updating the canvas). Requires: all Phase 1 capabilities plus parsing, type translation, and import resolution.

**Phase 3: Full compliance.** Add test framework mapping, all SHOULD-level features, and edge-case handling.

A Phase 1 binding is useful but does not support the full sync loop. The binding document MUST declare which phase it covers.

---

## 7. Conformance Testing

Binding authors SHOULD publish a conformance test suite including:

- A set of canvas representations covering common patterns (single class, inheritance hierarchy, package structure, tests).
- The expected generated source files for each canvas representation (canvas-to-code golden tests).
- A set of source files and the expected canvas representation after import (code-to-canvas golden tests).
- Round-trip tests demonstrating that canvas to code to canvas produces stable output.

---

## 8. Binding Declaration

Every language binding document SHOULD include a declaration header with the following metadata:

| Field | Description | Example |
|---|---|---|
| **Target language** | The language identifier string. | `python`, `typescript`, `rust`, `go`, `java` |
| **Spec version** | The CooperativeCoding spec version this binding targets. | `2.0.0` |
| **Phase** | The implementation phase this binding covers. | `Phase 1`, `Phase 2`, `Phase 3` |
| **Status** | The maturity of the binding. | `reference`, `community`, `draft` |

### Status Definitions

- **`reference`**: An official, first-party binding maintained alongside the core spec.
- **`community`**: A third-party binding contributed by the community.
- **`draft`**: A work-in-progress binding. MUST document which capabilities are missing.

---

## 9. Creating a New Binding

1. **Read this contract document.** Understand the MUST, SHOULD, and MAY tiers.
2. **Use the Python binding as a template.** The Python binding ([bindings/python.md](../bindings/python.md)) is the reference implementation of this contract.
3. **Define the stereotype mapping table.**
4. **Define the documentation format mapping.**
5. **Implement the required parsing and code generation capabilities.**
6. **Submit a PR to the spec repository.** See [CONTRIBUTING.md](../CONTRIBUTING.md).

### Binding Document Structure

A new binding document SHOULD follow this structure:

```
# CooperativeCoding <Language> Binding

**Target language:** <identifier>
**Spec version:** <semver>
**Phase:** <Phase 1|Phase 2|Phase 3>
**Status:** <reference|community|draft>

## Stereotype Mapping
## Documentation Format
## Type Mapping
## Import Resolution
## File Organization
## Test Framework Mapping (if Phase 3)
```

---

## 10. Existing Bindings

| Language | Document | Status |
|---|---|---|
| Python | [bindings/python.md](../bindings/python.md) | Reference binding |
