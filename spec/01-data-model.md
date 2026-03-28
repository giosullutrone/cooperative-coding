# CooperativeCoding Specification: Data Model

**Version 1.0.0**

---

## 1. Overview

CooperativeCoding is a semantic layer on top of [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/). Every `.canvas` file produced by CooperativeCoding is two things simultaneously: a valid JSON Canvas v1.0 document that any compliant tool can open, render, and edit, and an enriched architectural description that CooperativeCoding-aware tools use to provide design semantics and code sync.

The enrichment lives in a single namespace: `ccoding`. This is an object attached to nodes and edges alongside the standard JSON Canvas fields. Tools that don't understand `ccoding` ignore it safely. The file remains a readable canvas of text nodes and labeled edges. Tools that do understand `ccoding` gain access to the full CooperativeCoding paradigm: typed code elements, semantic relationships, and sync mappings.

This document defines the complete data model: the fields, the types, the constraints, and the semantics. It is the authoritative reference for what a CooperativeCoding canvas file contains and what each piece means.

---

## 2. Base JSON Canvas Fields

CooperativeCoding builds on the standard JSON Canvas v1.0 structure. A canvas file is a JSON object with two top-level arrays: `nodes` and `edges`. The [JSON Canvas v1.0 specification](https://jsoncanvas.org/spec/1.0/) is the authoritative reference for these fields. This section summarizes only the fields that CooperativeCoding extends or relies on.

### Node Fields

All CooperativeCoding code-element nodes use the `"text"` node type. The standard fields are:

| Field | Type | Description |
|---|---|---|
| `id` | String | Unique identifier for the node within the canvas. |
| `type` | String | Always `"text"` for CooperativeCoding code-element nodes. Context nodes MAY use any JSON Canvas node type (`"text"`, `"file"`, `"link"`, `"group"`). |
| `x` | Number | Horizontal position in pixels. |
| `y` | Number | Vertical position in pixels. |
| `width` | Number | Node width in pixels. |
| `height` | Number | Node height in pixels. |
| `text` | String | The node's text content (markdown). |
| `color` | String | Optional preset color (`"1"` through `"6"`) or hex color (e.g. `"#FF0000"`). |

### Edge Fields

| Field | Type | Description |
|---|---|---|
| `id` | String | Unique identifier for the edge within the canvas. |
| `fromNode` | String | ID of the source node. |
| `toNode` | String | ID of the target node. |
| `fromSide` | String | Optional. Side of the source node (`"top"`, `"right"`, `"bottom"`, `"left"`). |
| `toSide` | String | Optional. Side of the target node (`"top"`, `"right"`, `"bottom"`, `"left"`). |
| `fromEnd` | String | Optional. Shape of the source end (`"none"`, `"arrow"`). |
| `toEnd` | String | Optional. Shape of the target end (`"none"`, `"arrow"`). |
| `label` | String | Optional. Human-readable label displayed on the edge. |

CooperativeCoding adds the `ccoding` object to both nodes and edges. Implementations MUST NOT remove or alter any standard JSON Canvas fields when reading and writing canvas files. Unknown fields outside the `ccoding` namespace MUST be preserved through read/write cycles.

### Canvas-Level Metadata

In addition to node-level and edge-level `ccoding` objects, a canvas file MAY include a top-level `ccoding` object alongside the `nodes` and `edges` arrays. This provides canvas-wide defaults and compatibility information.

| Field | Required | Description |
|---|---|---|
| `specVersion` | RECOMMENDED | The CooperativeCoding spec version this canvas conforms to (e.g., `"1.0.0"`). Tools SHOULD use this for compatibility detection. |
| `language` | OPTIONAL | Default programming language for all nodes in this canvas (e.g., `"python"`). Individual nodes MAY override this with their own `ccoding.language` field. |

```json
{
  "ccoding": {
    "specVersion": "1.0.0",
    "language": "python"
  },
  "nodes": [...],
  "edges": [...]
}
```

Tools that do not recognize the top-level `ccoding` object MUST preserve it through read/write cycles.

### Cross-Canvas References

A project MAY use multiple `.canvas` files to represent different architectural views (see §03-sync, Section 7). Edges MUST NOT reference nodes in a different canvas file. The JSON Canvas format does not support cross-file references.

To express cross-canvas relationships, implementations SHOULD use `context` nodes that summarize the external dependency with a note like "See core.canvas, BaseService". A future spec version MAY introduce a formal cross-canvas reference mechanism.

---

## 3. Node Metadata Schema

The `ccoding` object on a node carries the semantic identity and language mapping of a code element. This is the primary extension point where CooperativeCoding adds meaning to what JSON Canvas sees as a plain text box.

### Complete JSON Example

```json
{
  "id": "node-1",
  "type": "text",
  "x": 100,
  "y": 200,
  "width": 300,
  "height": 400,
  "text": "## DocumentParser\n\n> Responsible for parsing raw documents into structured AST nodes",
  "ccoding": {
    "kind": "class",
    "stereotype": "protocol",
    "language": "python",
    "source": "src/parsers/document.py",
    "qualifiedName": "parsers.document.DocumentParser"
  }
}
```

### Field Definitions

| Field | Required | Values | Description |
|---|---|---|---|
| `kind` | REQUIRED | `class`, `method`, `field`, `package`, `test` (core); `interface`, `module`, `function`, `constant` (extended) | The type of code element this node represents. Core kinds MUST be supported by all implementations. Extended kinds SHOULD be supported. A minimal implementation MAY represent `interface` as `class` with a stereotype (e.g., `"stereotype": "protocol"`), and MAY represent `module` as `package`. |
| `stereotype` | OPTIONAL | Open set (e.g., `protocol`, `dataclass`, `abstract`, `enum`, `singleton`, `mixin`) | Language-specific subtype that refines the `kind`. Implementations MUST NOT reject unknown stereotype values. Each language binding defines its recognized stereotypes; unrecognized stereotypes SHOULD be preserved and displayed as-is. |
| `language` | OPTIONAL | Language identifier string (e.g., `python`, `typescript`, `rust`, `go`) | The programming language for this element. When omitted, implementations SHOULD infer the language from the canvas-level default or the language binding in use. A single canvas MAY contain nodes with different `ccoding.language` values. Each node is synced using the binding for its own language. Cross-language edges are informational only and are NOT synced to code. |
| `source` | OPTIONAL | Relative file path (e.g., `src/parsers/document.py`) | Path to the source file this node maps to, relative to the project root. Used by sync to locate the corresponding code. Implementations MUST treat this as a project-relative path, never absolute. Implementations MUST validate that resolved paths do not escape the project root directory. |
| `qualifiedName` | RECOMMENDED | Dot-notation identifier (e.g., `parsers.document.DocumentParser`) | Fully qualified name in the target language's namespace. This is the stable identity link between a canvas node and its code counterpart. Sync uses this value, not the node `id`, to match canvas elements to code elements. Implementations SHOULD include `qualifiedName` on all code-element nodes. |

### Kind Semantics

The `kind` field determines what type of code construct the node represents and how it participates in the canvas:

- **`class`** A class, struct, trait, protocol, or equivalent named type in the target language. This is the primary building block of the canvas. A class node's text content contains its responsibility and constraints. Its methods and fields are separate nodes connected to it.
- **`method`** A method or function. Every method is its own node on the canvas, connected to its parent class. A method node's text content contains the method signature, responsibility, pseudo code, and constraints.
- **`field`** A field or property. Every field is its own node on the canvas, connected to its parent class. A field node's text content contains the type, responsibility, and constraints.
- **`package`** A package, module, namespace, or directory that groups related classes. Used for high-level architectural views.
- **`interface`** (extended) Explicitly marks a node as an interface, protocol, trait, or abstract contract. Implementations that don't support this kind SHOULD represent it as `"kind": "class"` with an appropriate stereotype. The active language binding defines when to use `interface` vs `class` with a stereotype.
- **`test`** A test class or test suite that verifies the behavior of one or more classes. Connected to the class nodes it verifies via `tests` edges. A test node's text content contains the test class name, pseudo code of the test methods, and test execution results (pass, fail, or error with details).
- **`module`** (extended) A single-file module or compilation unit. Distinguished from `package` in languages where the distinction matters (e.g., Python's module vs. package). Implementations that don't support this kind SHOULD represent it as `package`.
- **`function`** (extended) A free-standing function or procedure that does not belong to any class. Connected to its containing `module` or `package` node.
- **`constant`** (extended) A module-level constant, configuration value, or global binding with architectural significance. Connected to its containing `module` or `package` node.

---

## 4. Edge Metadata Schema

The `ccoding` object on an edge carries the semantic relationship type. While JSON Canvas edges are unlabeled connections by default, CooperativeCoding edges carry typed relationships that sync uses to generate code constructs (inheritance declarations, imports, field types) and that canvas tools use to render relationship-specific styling.

### Complete JSON Example

```json
{
  "id": "edge-1",
  "fromNode": "node-1",
  "toNode": "node-2",
  "fromSide": "bottom",
  "toSide": "top",
  "label": "plugins: Applied sequentially during parse(). Order matters.",
  "ccoding": {
    "relation": "composes"
  }
}
```

### Field Definitions

| Field | Required | Values | Description |
|---|---|---|---|
| `relation` | REQUIRED | See Section 5 (Relation Types) | The type of relationship this edge represents. Determines how sync maps the edge to code and how canvas tools render it. |

When the entire `ccoding` object is absent, the edge is a plain JSON Canvas edge with no CooperativeCoding semantics. It is neither synced nor tracked.

---

## 5. Relation Types

Each relation type defines a specific semantic relationship between two nodes. The `relation` field MUST contain one of the following values. Implementations MUST support all relation types listed here.

| Relation | Semantics | Directionality | Code Construct |
|---|---|---|---|
| `inherits` | The source class extends or subclasses the target class. | `fromNode` inherits from `toNode`. | Inheritance declaration. |
| `implements` | The source class implements, conforms to, or adopts the target interface/protocol/trait. | `fromNode` implements `toNode`. | Implementation declaration. |
| `composes` | The source class contains the target as a field (has-a relationship). The edge label carries the field name. | `fromNode` has a field of type `toNode`. | Typed field declaration. |
| `depends` | The source class uses the target class (import-level dependency). | `fromNode` depends on `toNode`. | Import statement. |
| `member` | The target is a method or field that belongs to the source class. | `fromNode` is the class; `toNode` is the member. | The member is part of the class's code. |
| `calls` | A method on the source calls a method on the target. Documents runtime flow. | `fromNode` calls `toNode`. | No defined code construct. |
| `tests` | The source test node verifies the behavior of the target class node. | `fromNode` is the test node; `toNode` is the class under test. | Defined by the active language binding. |
| `context` | Links a context node to a code-element node. Provides design rationale or references. | Direction is flexible. | No defined code construct. |
| `contains` | Links a `package` or `module` node to the elements it contains. | `fromNode` is the container; `toNode` is the contained element. | Affects directory/file organization. |
| `overrides` | Indicates that a method node overrides a method in a parent class. | `fromNode` overrides `toNode`. | Defined by the active language binding. |

### Relation Validity

Implementations SHOULD validate that edges connect appropriate node kinds:

- `inherits`: SHOULD connect `class` to `class` (or `interface` to `interface`).
- `implements`: SHOULD connect `class` to `class` or `interface` where the target has an interface/protocol stereotype.
- `composes`: SHOULD connect `class` to `class`.
- `depends`: MAY connect any code-element nodes.
- `member`: MUST connect `class` to `method` or `class` to `field`.
- `calls`: SHOULD connect `method` to `method`, or `class` to `class`.
- `tests`: SHOULD connect `test` to `class` (or `interface`).
- `context`: MUST have at least one endpoint that is a context node (a node without `ccoding.kind`).
- `contains`: SHOULD connect `package` or `module` to `class`, `function`, or `constant`.
- `overrides`: SHOULD connect `method` to `method`.

Implementations SHOULD warn on invalid combinations but MUST NOT reject the canvas file.

---

## 6. Edge Labels

The JSON Canvas `label` field is OPTIONAL for CooperativeCoding edges. Labels provide human-readable context on the canvas. The active language binding defines whether and how labels are used during code generation.

---

## 7. Context Nodes

Context nodes are plain JSON Canvas nodes (text notes, file embeds, links, or groups) that carry no `ccoding.kind` metadata. They exist on the canvas as collaboration artifacts: design rationale, reference material, decision logs, external links, and any other non-code information that helps humans and agents reason about the architecture.

- Context nodes have **no `ccoding` metadata by default**. They are standard JSON Canvas nodes.
- Context nodes have **no defined code construct**. Changes to context nodes produce sync requests, but the agent will typically determine no code changes are needed.
- Context nodes **can be connected** to code-element nodes via `context` edges.

---

## 8. Node Text Content

The `text` field on a JSON Canvas node is a markdown string. CooperativeCoding treats this field as an opaque string at the spec level. The data model defines its type (string) and its preservation requirements, but does not prescribe its internal structure.

### Formal Requirements

- Implementations MUST preserve node text content exactly through read/write cycles. No normalization, no reformatting, no stripping of whitespace or markdown constructs.
- The spec does NOT prescribe a content format. There is no required heading structure, no mandatory sections, no enforced markdown dialect.
- Language bindings MAY define structured content conventions that map specific markdown sections to language constructs. These conventions are defined by the binding, not by this spec.
- Canvas tool implementations MAY parse node text for rendering purposes but MUST NOT modify the text as a side effect of rendering.

See [bindings/python.md](../bindings/python.md) for an example of structured node content conventions.

### Structured Content Convention

While the spec does not mandate a single content format for node text, practical sync requires that binding implementations parse the text to extract fields, methods, signatures, and documentation. Each language binding defines its own structured content convention (see §04-language-bindings).

Canvases are portable within a single binding but MAY NOT be portable across bindings that use different content conventions. Implementations MUST document their content format in the binding specification.
