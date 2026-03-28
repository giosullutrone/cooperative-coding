# CooperativeCoding Specification: Language Bindings

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures. The [Lifecycle](02-lifecycle.md) defines the sync loop. The [Sync](03-sync.md) defines the rules. This document defines the adapter: how the abstract CooperativeCoding specification maps to a specific programming language's idioms, conventions, and tooling.

A language binding is the translation layer between the canvas world and the code world. On the canvas, a node has a `kind`, a `stereotype`, documentation in markdown, and edges carrying relation types. In code, the same element is a class with a specific declaration syntax, a docstring in a specific format, imports following a specific convention, and type annotations in a specific notation. The binding defines the exact mapping between these two representations.

Without a binding, sync cannot operate. It knows that an `inherits` edge means "generate an inheritance declaration" but it cannot know the syntax for that declaration without a language-specific mapping. The binding is what makes the abstract spec concrete.

A _language binding_ has two parts: (1) a **binding specification**, a markdown document in this repository describing the mapping rules, and (2) a **binding implementation**, executable code (parser, generator, type mapper) that sync uses at runtime. The spec document defines the contract; the implementation fulfills it.

This document defines the contract that any language binding MUST fulfill to be CooperativeCoding-compliant. It separates requirements into three tiers: MUST (required for compliance), SHOULD (recommended for quality), and MAY (optional enhancements), following [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keyword semantics as established in [Introduction](00-introduction.md).

---

## 2. Required Capabilities (MUST)

This section defines the non-negotiable capabilities that every CooperativeCoding language binding MUST provide.

### 2.1 Stereotype Mapping

The `stereotype` field on a node refines what a `kind` means in the target language. A `class` node with `stereotype: "protocol"` means something very different in Python (a `typing.Protocol` subclass) than in TypeScript (where the concept maps to an `interface`). The binding is where these language-specific meanings are defined.

A binding MUST define a stereotype mapping table. This table MUST include, for each recognized stereotype:

| Column | Description |
|---|---|
| **Stereotype** | The string value of `ccoding.stereotype` (e.g., `"protocol"`, `"dataclass"`, `"enum"`). |
| **Language construct** | The concrete syntax or declaration pattern in the target language. |
| **Required imports** | Any import statements that the stereotype necessitates. |

The stereotype set is open. A binding defines its recognized stereotypes, but:

- Implementations MUST NOT reject unknown stereotypes. Unknown stereotypes SHOULD be treated as plain classes.
- A binding MAY define additional stereotypes at any time. Adding a stereotype is a backwards-compatible change.
- A binding SHOULD document the behavioral differences between stereotypes.

### 2.2 AST Parsing

Sync needs to read existing code to process design requests (code changes that update the canvas). The binding is responsible for defining how source files in the target language are parsed and what information is extracted.

A binding MUST be able to parse source files and extract:

- **Class and type definitions:** name, fully qualified name, inheritance hierarchy, stereotype (inferred from language-specific markers).
- **Method and function signatures:** name, parameter list (name, type, default value), return type, modifiers (static, abstract, etc.).
- **Field and attribute declarations:** name, type annotation, required/optional/default.
- **Type annotations:** wherever the language supports them.
- **Documentation:** content and structured sections (e.g., `:param name:` in Python, `@param name` in JSDoc).
- **Relationships:** parent classes, implemented interfaces, composition via typed fields.

### 2.3 Code Generation

Sync needs to write code from canvas nodes when processing code requests (canvas changes that update the code). The binding is responsible for defining how canvas data translates to source code.

A binding MUST be able to generate:

- **Class skeletons:** class declaration, inheritance, implementation declarations, documentation, stereotype-specific syntax.
- **Method stubs:** method declaration with correct signature, documentation, and a stub body.
- **Field declarations:** field with type annotation and documentation.
- **Import statements:** correct imports from `depends`, `inherits`, `implements`, `composes` edges and stereotype requirements.
- **Inheritance and implementation declarations:** correct syntax for the target language.
- **Edge label interpretation:** how edge labels are parsed and used during code generation (e.g., extracting field names from `composes` labels).

### 2.4 Test Node Support

Test nodes are core code-element nodes (`ccoding.kind: "test"`), and a compliant binding MUST define how they map to the target language's testing conventions.

A binding MUST define:

- **Test framework mapping:** the default test framework and how test node content maps to that framework's conventions.
- **Test class generation:** how to generate test source files from test nodes and their connected `tests` edges.
- **Test result format:** how test execution results are represented in the test node's content, and how sync reads results from the framework's output.
- **Test file conventions:** where test files live, how they are named, and how `ccoding.source` is derived.

### 2.5 Documentation Mapping

The canvas node's text content carries the design intent. The binding defines how this content maps to the language's documentation format.

A binding MUST define the mapping for:

- **Responsibility statements:** how the responsibility description appears in the language's documentation format.
- **Pseudo code:** how pseudo code from a method's canvas content is represented in the generated code.
- **Method signature documentation:** how parameters, return types, and exceptions are documented.
- **Field descriptions:** how field-level documentation appears in code.

---

## 3. Recommended Capabilities (SHOULD)

### 3.1 Import Resolution

The binding SHOULD define a complete import resolution strategy that maps qualified names to import statements: handling relative vs. absolute imports, standard library types, re-exports, and namespace packages.

### 3.2 Type Mapping

Canvas nodes use language-neutral type names (e.g., `List<T>`, `Map<K, V>`, `Optional<T>`). The binding SHOULD define how these map to the target language's concrete type syntax.

### 3.3 Concrete Sync Rules

The [Sync](03-sync.md) spec defines abstract rules. The binding SHOULD extend them with language-specific details:

- **File path conventions:** how `ccoding.qualifiedName` maps to file system paths.
- **Package initialization:** what files sync must create for a new package.
- **Naming conventions:** how canvas element names transform to the target language's conventions.
- **Method body preservation:** how sync identifies the boundary between a method's signature and its body, so that sync can update signatures without overwriting method implementations.

---

## 4. Optional Capabilities (MAY)

### 4.1 Structured Node Content Format

The core spec treats node text as opaque ([Data Model, Section 8](01-data-model.md)). A binding MAY define a structured content format (heading conventions, field notation, method notation, responsibility section placement) that enables richer round-trip fidelity between canvas and code.

### 4.2 Code Style Configuration

A binding MAY support language-specific formatting preferences. When supported, the binding SHOULD integrate with the language's existing formatting tools (e.g., Black/Ruff for Python, Prettier for TypeScript, rustfmt for Rust) rather than implementing its own formatter.

### 4.3 Multi-File Strategies

A binding MAY define strategies for how canvas nodes map to file organization (one class per file, module-based grouping, feature-based grouping).

---

### Sync Architecture

The spec is deliberately silent on deployment topology. Valid architectures include:

- A **standalone sync daemon** that watches file changes and runs sync automatically.
- A **canvas plugin component** where the plugin embeds sync logic and triggers it on canvas save.
- An **agent-integrated sync** where the AI agent invokes sync as part of the sync loop.
- A **CLI tool** that the user or agent invokes manually.

All architectures MUST implement the same sync loop defined in §03-sync.

### Minimum Viable Binding

Binding authors MAY adopt a phased implementation approach:

**Phase 1: Code requests only.** Implement code generation from canvas nodes (processing code requests). Requires: stereotype mapping, code generation, documentation formatting, edge label interpretation. Does not require: AST parsing or design request processing.

**Phase 2: Full sync loop.** Add AST parsing and design request processing (code changes updating the canvas). Requires: all Phase 1 capabilities plus AST parsing, type translation, and import resolution.

**Phase 3: Full compliance.** Add test node support, all SHOULD-level features, and edge-case handling.

A Phase 1 binding is useful but does not support the full sync loop. The binding document MUST declare which phase it covers.

### Conformance Testing

Binding authors SHOULD publish a conformance test suite including:

- A set of `.canvas` files representing common patterns (single class, inheritance hierarchy, package structure, test nodes).
- The expected generated source files for each canvas (canvas-to-code golden tests).
- A set of source files and the expected canvas JSON after import (code-to-canvas golden tests).
- Round-trip tests demonstrating that canvas to code to canvas produces stable output.

---

## 5. Binding Declaration

Every language binding document SHOULD include a declaration header with the following metadata:

| Field | Description | Example |
|---|---|---|
| **Target language** | The language identifier string. MUST match `ccoding.language` values. | `python`, `typescript`, `rust`, `go`, `java` |
| **Spec version** | The CooperativeCoding spec version this binding targets. | `1.0.0` |
| **Status** | The maturity of the binding. | `reference`, `community`, `draft` |

### Status Definitions

- **`reference`**: An official, first-party binding maintained alongside the core spec.
- **`community`**: A third-party binding contributed by the community.
- **`draft`**: A work-in-progress binding. MUST document which capabilities are missing.

---

## 6. Creating a New Binding

1. **Read this contract document.** Understand the MUST, SHOULD, and MAY tiers.
2. **Use the Python binding as a template.** The Python binding ([bindings/python.md](../bindings/python.md)) is the reference implementation of this contract.
3. **Define the stereotype mapping table.**
4. **Define the documentation format mapping.**
5. **Implement the required AST parsing and code generation capabilities.**
6. **Submit a PR to the spec repository.** See [CONTRIBUTING.md](../CONTRIBUTING.md).

### Binding Document Structure

A new binding document SHOULD follow this structure:

```
# CooperativeCoding <Language> Binding

**Target language:** <identifier>
**Spec version:** <semver>
**Status:** <reference|community|draft>

## Stereotype Mapping
## Documentation Format
## Type Mapping
## Import Resolution
## File Organization
## Concrete Sync Rules
```

---

## 7. Existing Bindings

| Language | Document | Status |
|---|---|---|
| Python | [bindings/python.md](../bindings/python.md) | Reference binding |
