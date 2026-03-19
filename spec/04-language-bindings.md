# CooperativeCoding Specification — Language Bindings

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures. The [Lifecycle](02-lifecycle.md) defines the motion. The [Sync](03-sync.md) defines the bridge. This document defines the adapter: how the abstract CooperativeCoding specification maps to a specific programming language's idioms, conventions, and tooling.

A language binding is the translation layer between the canvas world and the code world. On the canvas, a node has a `kind`, a `stereotype`, a documentation block in markdown, and edges carrying relation types. In code, the same element is a class with a specific declaration syntax, a docstring in a specific format, imports following a specific convention, and type annotations in a specific notation. The binding defines the exact mapping between these two representations — how `stereotype: "protocol"` becomes `class Foo(Protocol):` in Python, how a responsibility statement becomes a triple-quoted docstring, how a `composes` edge becomes a typed field declaration with the right import.

Without a binding, the sync engine cannot operate. It knows that an `inherits` edge means "generate an inheritance declaration" but it cannot know the syntax for that declaration without a language-specific mapping. It knows that a class node's documentation should appear in a doc comment but it cannot know whether that means `"""..."""`, `/** ... */`, or `/// ...` without a binding. The binding is what makes the abstract spec concrete.

This document defines the contract that any language binding MUST fulfill to be CooperativeCoding-compliant. It separates requirements into three tiers — MUST (required for compliance), SHOULD (recommended for quality), and MAY (optional enhancements) — following [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keyword semantics as established in [Introduction](00-introduction.md). A binding that satisfies all MUST requirements is compliant. A binding that also satisfies the SHOULD requirements is high-quality. A binding that goes further with MAY capabilities is comprehensive.

---

## 2. Required Capabilities (MUST)

This section defines the non-negotiable capabilities that every CooperativeCoding language binding MUST provide. A binding that omits any of these is not compliant and cannot be used by a spec-conformant sync engine.

### 2.1 Stereotype Mapping

The `stereotype` field on a node refines what a `kind` means in the target language. A `class` node with `stereotype: "protocol"` means something very different in Python (a `typing.Protocol` subclass) than in TypeScript (where the concept maps to an `interface`). The binding is where these language-specific meanings are defined.

A binding MUST define a stereotype mapping table. This table MUST include, for each recognized stereotype:

| Column | Description |
|---|---|
| **Stereotype** | The string value of `ccoding.stereotype` (e.g., `"protocol"`, `"dataclass"`, `"enum"`). |
| **Language construct** | The concrete syntax or declaration pattern in the target language (e.g., `class Foo(Protocol):` in Python, `interface Foo {}` in TypeScript). |
| **Required imports** | Any import statements that the stereotype necessitates (e.g., `from typing import Protocol` for Python protocols). |

The stereotype set is open. A binding defines its recognized stereotypes, but:

- Implementations MUST NOT reject unknown stereotypes. A canvas may contain stereotypes defined by a different binding, a future version of the current binding, or a user's custom extension. Unknown stereotypes SHOULD be treated as plain classes — generate a standard class declaration with no special syntax or imports.
- A binding MAY define additional stereotypes beyond the core set at any time. Adding a stereotype is a backwards-compatible change.
- A binding SHOULD document the behavioral differences between stereotypes. For example, a `dataclass` stereotype in Python implies `@dataclass` decoration, automatic `__init__`, and field ordering semantics that a plain `class` does not carry.

### 2.2 AST Parsing

The sync engine needs to read existing code to perform code-to-canvas sync, conflict detection, and reconciliation. The binding is responsible for defining how source files in the target language are parsed and what information is extracted.

A binding MUST be able to parse source files in the target language and extract the following elements:

**Class and type definitions.** For each class, struct, interface, trait, or equivalent named type in a source file:
- The class name.
- The fully qualified name (module path + class name).
- The inheritance hierarchy — which classes or interfaces this type extends or implements.
- The stereotype, inferred from language-specific markers (e.g., a Python class decorated with `@dataclass` maps to `stereotype: "dataclass"`; a class inheriting from `Protocol` maps to `stereotype: "protocol"`).

**Method and function signatures.** For each method or function within a class:
- The method name.
- The parameter list — name, type annotation (if present), and default value (if present) for each parameter.
- The return type annotation (if present).
- Whether the method is static, a class method, abstract, or carries other modifiers relevant to the language.

**Field and attribute declarations.** For each field, property, or attribute within a class:
- The field name.
- The type annotation (if present).
- Whether the field is required, optional, has a default value, or carries other modifiers.

**Type annotations.** The binding MUST extract type annotations wherever the language supports them — on fields, parameters, return types, and variables. Type annotations are critical for canvas accuracy: they determine the type names displayed in the node's fields and methods sections and the type references used by `composes` edges.

**Documentation blocks.** For each class, method, and field:
- The documentation content (docstring body, JSDoc description, doc comment text).
- Structured documentation sections where the language convention supports them (e.g., `:param name:` in Python reStructuredText, `@param name` in JSDoc, `# Arguments` in Rust doc comments).

**Inheritance and implementation relationships.** For each class:
- The list of parent classes (for `inherits` edges).
- The list of implemented interfaces, protocols, or traits (for `implements` edges).
- Composition relationships inferred from typed field declarations that reference other project classes (for `composes` edges).

### 2.3 Code Generation

The sync engine needs to write code from canvas nodes during canvas-to-code sync. The binding is responsible for defining how canvas data translates to source code in the target language.

A binding MUST be able to generate the following from canvas node data:

**Class skeletons.** Given a class node's metadata and text content, generate a complete class declaration including:
- The class keyword and name.
- Inheritance declarations derived from `inherits` edges.
- Implementation declarations derived from `implements` edges.
- A documentation block populated from the node's responsibility statement and description, formatted according to the language's documentation conventions (Section 2.4).
- Stereotype-specific decorators, annotations, or syntax (e.g., `@dataclass` for Python dataclasses, `abstract` keyword for abstract classes).

**Method stubs.** Given a method's name, parameters, return type, and documentation:
- Generate a method declaration with the correct signature.
- Include a documentation block with the method's responsibility, parameter descriptions, return type description, and exception documentation.
- Generate a stub body that satisfies the language's syntactic requirements (e.g., `pass` in Python, `throw new Error("Not implemented")` in TypeScript, `todo!()` in Rust).

**Field declarations.** Given a field's name, type annotation, and documentation:
- Generate a field declaration with the type annotation in the language's syntax.
- Include any documentation or comment conventions the language supports for fields.

**Import statements.** Given the dependencies of a class (from `depends` edges, `inherits` edges, `implements` edges, `composes` edges, and stereotype-required imports):
- Generate the correct import statements for the target language.
- Resolve module paths from `ccoding.qualifiedName` and `ccoding.source` values.

**Inheritance and implementation declarations.** Given `inherits` and `implements` edges:
- Generate the correct syntax for the target language (e.g., parenthesized base classes in Python, `extends`/`implements` clauses in TypeScript, trait bounds in Rust).

### 2.4 Documentation Mapping

The canvas node's text content carries the design intent: responsibility statements, pseudo code, field descriptions, method signatures, and constraints. The binding defines how this content maps to the language's documentation format.

A binding MUST define the mapping for each of the following:

**Responsibility statements.** How the "responsibility" description of a class or method appears in the language's documentation format. The responsibility is the one-sentence-to-one-paragraph summary of what the element does and why it exists. Examples:
- Python: the first paragraph of a triple-quoted docstring.
- TypeScript: the description section of a JSDoc comment.
- Rust: the first paragraph of a `///` doc comment.

**Pseudo code.** How pseudo code from a method's canvas documentation is represented in the generated code. Pseudo code describes the algorithm or logic flow in plain language. A binding MUST define where pseudo code appears — common approaches include:
- A dedicated section in the documentation block (e.g., `Algorithm:` or `Steps:` heading in a docstring).
- Inline comments within the method stub body.
- A separate documentation section that the language's doc tooling recognizes.

**Method signature documentation.** How method parameters, return types, and exceptions are documented. A binding MUST define the format for:
- Parameter descriptions (name, type, purpose).
- Return type descriptions (type, semantics).
- Exception or error documentation (which exceptions may be raised and when).

**Field descriptions.** How field-level documentation from the canvas appears in code. A binding MUST define whether fields are documented via:
- Inline comments adjacent to the field declaration.
- A section in the class-level documentation block.
- Per-field documentation blocks (where the language supports them).

---

## 3. Recommended Capabilities (SHOULD)

These capabilities are not required for compliance but significantly improve the binding's usefulness. A binding that omits them will work but will produce less polished results.

### 3.1 Import Resolution

When the sync engine generates code, it needs to produce correct import statements. The binding SHOULD define a complete import resolution strategy that maps qualified names to import statements.

Import resolution involves:

- Mapping `ccoding.qualifiedName` values (dot-notation identifiers like `parsers.document.DocumentParser`) to the target language's module system and import syntax.
- Handling the distinction between relative and absolute imports where the language supports both.
- Resolving standard library types (e.g., `List`, `Dict`, `Optional` in Python) to the correct import source and syntax for the target language version.
- Handling re-exports, namespace packages, and other module system features specific to the target language.

### 3.2 Type Mapping

Canvas nodes use type names in field declarations and method signatures. These names are written in a language-neutral notation on the canvas (e.g., `List<T>`, `Map<K, V>`, `Optional<T>`). The binding SHOULD define how these canvas-level type names map to the target language's concrete type syntax.

A type mapping table SHOULD include:

| Canvas Type | Language Type | Notes |
|---|---|---|
| `List<T>` | `list[T]` (Python 3.9+) or `List[T]` (older) | Version-dependent |
| `Map<K, V>` | `dict[K, V]` (Python 3.9+) | |
| `Optional<T>` | `T \| None` (Python 3.10+) or `Optional[T]` | |
| ... | ... | Language-specific |

The table above is illustrative. Each binding defines its own mappings for the full set of generic types commonly used on canvases.

### 3.3 Concrete Sync Rules

The [Sync](03-sync.md) spec defines abstract rules: "generate an inheritance declaration", "add an import statement", "create a class skeleton." These rules are language-agnostic by design. The binding SHOULD extend them with language-specific details that the sync engine needs to apply the abstract rules correctly.

Concrete sync rule extensions SHOULD address:

- **File path conventions.** How `ccoding.qualifiedName` maps to file system paths. For example, in Python, `parsers.document.DocumentParser` maps to `parsers/document.py` (a module containing the class). In Java, it might map to `parsers/document/DocumentParser.java` (one class per file). In Rust, the mapping follows the module tree defined in `mod` declarations.
- **Package initialization.** What files the sync engine must create when generating a new package. Python requires `__init__.py`; Go requires no initialization file; Rust requires a `mod.rs` or a declaration in the parent module.
- **Naming conventions.** How canvas element names (which follow a language-neutral convention on the canvas) are transformed to the target language's naming conventions. For example, a class named `DocumentParser` stays PascalCase in most languages, but a field named `plugin_manager` should be `pluginManager` in TypeScript and `plugin_manager` in Python.
- **Method body preservation.** How the sync engine identifies the boundary between a method's signature (which sync owns) and its body (which code owns). Python uses indentation; TypeScript and Java use braces; Rust uses braces. The binding SHOULD define the parsing strategy for locating this boundary.

---

## 4. Optional Capabilities (MAY)

These capabilities go beyond what the spec requires or recommends. They are enhancements that specific bindings may offer to provide a richer experience for their target language.

### 4.1 Structured Node Content Format

The core spec treats node text as opaque — a markdown string with no prescribed internal structure ([Data Model, Section 9](01-data-model.md)). A binding MAY define a structured content format that enables richer round-trip fidelity between canvas and code.

A structured content format typically defines:

- **Heading conventions.** Which markdown headings correspond to which sections (e.g., `## ClassName` for the class title, `### Fields` for the fields section, `### Methods` for the methods section).
- **Field notation.** How fields are listed within a node (e.g., `- name: Type — description` as a markdown list item).
- **Method notation.** How methods are listed within a node (e.g., `- method_name(param: Type) -> ReturnType — description`).
- **Responsibility section.** Where the responsibility statement appears (e.g., as a blockquote immediately after the class heading: `> Responsible for ...`).

Defining a structured content format is OPTIONAL because the spec deliberately does not mandate one. Different canvas tools may render markdown differently, and different users have different preferences. However, a binding that defines a structured format enables the sync engine to perform precise, section-level round-tripping rather than treating the entire node text as a monolithic blob.

### 4.2 Code Style Configuration

A binding MAY support language-specific formatting preferences that control the appearance of generated code. Common configuration options include:

- Indentation style (tabs vs. spaces, indentation width).
- Line length limits.
- Import sorting and grouping conventions.
- Trailing commas, semicolons, and other syntactic preferences.
- Documentation format variants (e.g., reStructuredText vs. Google style vs. NumPy style docstrings in Python).

When code style configuration is supported, the binding SHOULD integrate with the language's existing formatting tools (e.g., Black/Ruff for Python, Prettier for TypeScript, rustfmt for Rust) rather than implementing its own formatter. Generated code SHOULD pass through the project's configured formatter before being written to disk.

### 4.3 Multi-File Strategies

A binding MAY define strategies for how canvas nodes map to file organization. The default assumption in the core spec is one class per source mapping (`ccoding.source` on each node), but real projects use diverse file organization patterns:

- **One class per file.** Standard in Java and common in TypeScript. Each class node maps to its own file.
- **Module-based grouping.** Common in Python. Multiple related classes share a module file. The binding defines how to determine which classes belong in the same file.
- **Feature-based grouping.** Classes grouped by feature or domain rather than by type. The binding defines the grouping heuristic.

When a binding defines multi-file strategies, it SHOULD specify how the sync engine determines file boundaries when generating code from canvas nodes that lack an explicit `ccoding.source` value.

---

## 5. Binding Declaration

Every language binding document exists as a companion to this contract. To ensure consistency and discoverability, a binding document SHOULD include a declaration header with the following metadata:

| Field | Description | Example |
|---|---|---|
| **Target language** | The language identifier string. This value MUST match the `ccoding.language` values used in canvas node metadata. | `python`, `typescript`, `rust`, `go`, `java` |
| **Spec version** | The CooperativeCoding specification version that this binding targets. Uses semantic versioning as defined in [Introduction](00-introduction.md). | `1.0.0` |
| **Status** | The maturity and provenance of the binding. | `reference`, `community`, `draft` |

### Status Definitions

- **`reference`** — An official, first-party binding maintained alongside the core spec. Reference bindings are the authoritative examples for how to implement the binding contract. They are held to the highest standard of completeness and correctness.
- **`community`** — A third-party binding contributed by the community. Community bindings follow the same contract but are maintained independently. They are reviewed for compliance but may make different choices within the spec's degrees of freedom.
- **`draft`** — A work-in-progress binding that is not yet complete. Draft bindings MAY omit some MUST-level capabilities while they are under development, but MUST document which capabilities are missing. A draft binding MUST NOT be relied upon for production use.

---

## 6. Creating a New Binding

To create a CooperativeCoding binding for a new language, follow these steps:

1. **Read this contract document.** Understand the MUST, SHOULD, and MAY tiers. Every binding starts from the same contract.

2. **Use the Python binding as a template.** The Python binding ([bindings/python.md](../bindings/python.md)) is the reference implementation of this contract. It demonstrates the expected structure, level of detail, and style for a binding document. Start by copying its structure and replacing the Python-specific content with your language's equivalents.

3. **Define the stereotype mapping table.** List every meaningful subtype of `class` in your target language. For each stereotype, document the concrete syntax, required imports, and behavioral implications. Cover at least the equivalents of `protocol`/`interface`, `dataclass`/`record`, `enum`, `abstract`, and `singleton` — or document why a concept does not apply to your language.

4. **Define the documentation format mapping.** Specify how responsibility statements, pseudo code, method parameters, return types, exceptions, and field descriptions map to your language's documentation conventions. If your language has multiple documentation styles (e.g., Python's reStructuredText vs. Google vs. NumPy docstrings), pick a default and document how to switch.

5. **Implement the required AST parsing and code generation capabilities.** This is the implementation work — writing the parser and generator that the sync engine will use. The binding document describes the mapping; the implementation executes it.

6. **Submit a PR to the spec repository.** New bindings are contributed through pull requests. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the contribution guidelines, review process, and licensing requirements.

### Binding Document Structure

A new binding document SHOULD follow this structure:

```
# CooperativeCoding <Language> Binding

**Target language:** <identifier>
**Spec version:** <semver>
**Status:** <reference|community|draft>

## Stereotype Mapping
(table of stereotypes → language constructs)

## Documentation Format
(mapping of canvas documentation → language doc format)

## Type Mapping
(table of canvas types → language types)

## Import Resolution
(rules for generating import statements)

## File Organization
(how qualified names map to file paths)

## Concrete Sync Rules
(language-specific extensions to the abstract sync rules)
```

---

## 7. Existing Bindings

The following table lists all known CooperativeCoding language bindings. Reference bindings are maintained in this repository. Community bindings are maintained externally and linked here for discoverability.

| Language | Document | Status |
|---|---|---|
| Python | [bindings/python.md](../bindings/python.md) | Reference binding |
