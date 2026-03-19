# CooperativeCoding Specification — Data Model

**Version 1.0.0**

---

## 1. Overview

CooperativeCoding is a semantic layer on top of [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/). Every `.canvas` file produced by CooperativeCoding is two things simultaneously: a valid JSON Canvas v1.0 document that any compliant tool can open, render, and edit — and an enriched architectural description that CooperativeCoding-aware tools use to provide design semantics, lifecycle tracking, and bidirectional code sync.

The enrichment lives in a single namespace: `ccoding`. This is an object attached to nodes and edges alongside the standard JSON Canvas fields. Tools that don't understand `ccoding` ignore it safely — the file remains a readable canvas of text nodes and labeled edges. Tools that do understand `ccoding` gain access to the full CooperativeCoding paradigm: typed code elements, semantic relationships, ghost proposals, and sync mappings.

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

---

## 3. Node Metadata Schema

The `ccoding` object on a node carries the semantic identity, language mapping, and lifecycle status of a code element. This is the primary extension point where CooperativeCoding adds meaning to what JSON Canvas sees as a plain text box.

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
    "qualifiedName": "parsers.document.DocumentParser",
    "status": "accepted",
    "proposedBy": null,
    "proposalRationale": null
  }
}
```

### Field Definitions

| Field | Required | Values | Description |
|---|---|---|---|
| `kind` | REQUIRED | `class`, `method`, `field`, `package` (core); `interface`, `module` (extended) | The type of code element this node represents. Core kinds MUST be supported by all implementations. Extended kinds SHOULD be supported. A minimal implementation MAY represent `interface` as `class` with a stereotype (e.g., `"stereotype": "protocol"`), and MAY represent `module` as `package`. |
| `stereotype` | OPTIONAL | Open set (e.g., `protocol`, `dataclass`, `abstract`, `enum`, `singleton`, `mixin`) | Language-specific subtype that refines the `kind`. Implementations MUST NOT reject unknown stereotype values — the set is open and extensible. Each language binding defines its recognized stereotypes; unrecognized stereotypes SHOULD be preserved and displayed as-is. |
| `language` | OPTIONAL | Language identifier string (e.g., `python`, `typescript`, `rust`, `go`) | The programming language for this element. When omitted, implementations SHOULD infer the language from the canvas-level default or the language binding in use. |
| `source` | OPTIONAL | Relative file path (e.g., `src/parsers/document.py`) | Path to the source file this node maps to, relative to the project root. Used by the sync engine to locate the corresponding code. Implementations MUST treat this as a project-relative path, never absolute. |
| `qualifiedName` | RECOMMENDED | Dot-notation identifier (e.g., `parsers.document.DocumentParser`) | Fully qualified name in the target language's namespace. This is the stable identity link between a canvas node and its code counterpart. The sync engine uses this value — not the node `id` — to match canvas elements to code elements. Implementations SHOULD include `qualifiedName` on all code-element nodes. |
| `status` | REQUIRED for tracked nodes | `accepted`, `proposed`, `rejected`, `stale` | Current lifecycle status. Governs whether the element is synced to code, awaiting review, or flagged for attention. See Section 4 and [Lifecycle](02-lifecycle.md) for full semantics. |
| `proposedBy` | OPTIONAL | `"agent"`, `"human"`, or `null` | Who created this element as a proposal. MUST be set when `status` is `"proposed"`. MAY be preserved after acceptance for audit trail purposes, or MAY be cleared (set to `null`). |
| `proposalRationale` | OPTIONAL | String or `null` | Free-text explanation for why this element was proposed. Intended for human review — the agent explains its reasoning, or the human explains their tentative idea. SHOULD be present when `proposedBy` is `"agent"`. |

### Kind Semantics

The `kind` field determines what type of code construct the node represents and how it participates in the canvas:

- **`class`** — A class, struct, trait, protocol, or equivalent named type in the target language. This is the primary building block of the canvas. A class node's text content contains its responsibility, fields, methods, and constraints.
- **`method`** — A method or function that has been promoted to its own detail node. Always connected to its parent class via a `detail` edge. A method node's text content contains the method signature, responsibility, pseudo code, and constraints.
- **`field`** — A field or property that has been promoted to its own detail node. Always connected to its parent class via a `detail` edge. Used when a field carries significant design semantics (e.g., a configuration object, a dependency injection point).
- **`package`** — A package, module, namespace, or directory that groups related classes. Used for high-level architectural views. Package nodes MAY contain a listing of their contents in the text field.
- **`interface`** (extended) — Explicitly marks a node as an interface, protocol, trait, or abstract contract rather than a concrete class. Implementations that don't support this kind SHOULD represent it as `"kind": "class"` with an appropriate stereotype.
- **`module`** (extended) — A single-file module or compilation unit. Distinguished from `package` in languages where the distinction matters (e.g., Python's module vs. package). Implementations that don't support this kind SHOULD represent it as `package`.

---

## 4. Status Semantics

The `status` field governs how implementations treat a node or edge. Each value carries precise, normative semantics.

### `accepted`

The element is part of the canonical design. It represents a deliberate architectural decision that has been confirmed by a human.

- Implementations MUST sync `accepted` elements to code during canvas-to-code sync.
- Implementations MUST update `accepted` elements from code during code-to-canvas sync.
- Canvas tools SHOULD render `accepted` elements with full visual weight (normal opacity, standard colors).

### `proposed`

The element is a ghost — a tentative proposal awaiting human review. It exists on the canvas as a visual suggestion but has no authority over the codebase.

- Implementations MUST NOT sync `proposed` elements to code. A ghost MUST NOT create, modify, or delete any source file.
- Canvas tools SHOULD render `proposed` elements with reduced visual weight (lower opacity, dashed borders, or other distinct styling) to clearly distinguish them from accepted elements.
- The `proposedBy` field MUST be set to `"agent"` or `"human"` when `status` is `"proposed"`.

### `rejected`

The element was a proposal that a human explicitly declined. It remains in the canvas file as a record of the decision but is not part of the active design.

- Implementations MUST NOT sync `rejected` elements to code.
- Canvas tools SHOULD hide rejected elements by default, or render them with a visually suppressed style (greyed out, collapsed, or filtered to a separate view).
- Implementations SHOULD preserve `rejected` elements in the canvas file rather than deleting them, so the decision history is retained.

### `stale`

The element's corresponding code was deleted, moved, or renamed in a way the sync engine could not automatically resolve. The canvas node remains, but its link to the codebase is broken.

- Implementations MUST NOT auto-delete stale nodes. The human decides whether to remove, update, or restore the element.
- Implementations MUST NOT sync stale elements to code until a human resolves the staleness.
- Canvas tools SHOULD render stale elements with a distinct visual indicator (e.g., a warning icon, a strikethrough, or a muted color).
- A stale node MAY transition back to `accepted` if the corresponding code is restored or the human manually re-links it.

### Status Transition Summary

```
proposed  ──accept──▸  accepted
proposed  ──reject──▸  rejected
accepted  ──stale───▸  stale      (detected by sync)
stale     ──restore─▸  accepted   (human or sync action)
rejected  ──reconsider──▸  proposed   (human reconsideration)
```

---

## 5. Edge Metadata Schema

The `ccoding` object on an edge carries the semantic relationship type and lifecycle status. While JSON Canvas edges are unlabeled connections by default, CooperativeCoding edges carry typed relationships that the sync engine uses to generate code constructs (inheritance declarations, imports, field types) and that canvas tools use to render relationship-specific styling.

### Complete JSON Example

```json
{
  "id": "edge-1",
  "fromNode": "node-1",
  "toNode": "node-2",
  "fromSide": "bottom",
  "toSide": "top",
  "label": "plugins — Applied sequentially during parse(). Order matters.",
  "ccoding": {
    "relation": "composes",
    "status": "accepted",
    "proposedBy": null,
    "proposalRationale": null
  }
}
```

### Field Definitions

| Field | Required | Values | Description |
|---|---|---|---|
| `relation` | REQUIRED | See Section 6 (Relation Types) | The type of relationship this edge represents. Determines how the sync engine maps the edge to code and how canvas tools render it. |
| `status` | OPTIONAL | `accepted`, `proposed`, `rejected`, `stale` | Lifecycle status. Follows the same semantics as node status (Section 4). Defaults to `accepted` if omitted. When an edge connects two `accepted` nodes but the edge itself is `proposed`, the relationship is a ghost — the nodes exist in code, but the connection between them is tentative. |
| `proposedBy` | OPTIONAL | `"agent"`, `"human"`, or `null` | Who created this edge as a proposal. Same semantics as the node-level field. |
| `proposalRationale` | OPTIONAL | String or `null` | Explanation for why this relationship was proposed. Same semantics as the node-level field. |

### Edge Status Defaults

When a `ccoding` object is present on an edge but `status` is omitted, implementations MUST treat the edge as `accepted`. When the entire `ccoding` object is absent, the edge is a plain JSON Canvas edge with no CooperativeCoding semantics — it is neither synced nor tracked.

---

## 6. Relation Types

Each relation type defines a specific semantic relationship between two nodes. The `relation` field MUST contain one of the following values. Implementations MUST support all relation types listed here.

| Relation | Semantics | Directionality | Sync Mapping |
|---|---|---|---|
| `inherits` | The source class extends or subclasses the target class. | `fromNode` inherits from `toNode`. | Generates an inheritance declaration in code (e.g., `class Foo(Bar)` in Python, `class Foo extends Bar` in TypeScript). |
| `implements` | The source class implements, conforms to, or adopts the target interface/protocol/trait. | `fromNode` implements `toNode`. | Generates an implementation declaration (e.g., protocol conformance in Python, `implements` clause in TypeScript). |
| `composes` | The source class contains the target as a field (has-a relationship). The edge label carries the field name and composition semantics. | `fromNode` has a field of type `toNode`. | Generates a typed field declaration on the source class. The field name comes from the edge label. |
| `depends` | The source class uses the target class (import-level dependency). | `fromNode` depends on `toNode`. | Generates an import statement in the source file. |
| `calls` | A method on the source calls a method on the target. Informational — documents runtime flow. | `fromNode` calls `toNode`. | Not directly synced. Used for documentation, sequence reasoning, and architectural analysis. Implementations MAY generate call-site comments. |
| `detail` | Links a class node to one of its promoted method or field nodes. The target is a detail of the source. | `fromNode` is the parent class; `toNode` is the detail node. | The detail node's content is synced as part of the parent class's code. The edge itself produces no independent code construct. |
| `context` | Links a context node to a CooperativeCoding code-element node. Provides design rationale, references, or other non-code information. | Direction is flexible. Either node may be `fromNode`. | Not synced — canvas-only. Context edges and their target context nodes are collaboration artifacts that exist solely on the canvas. |

### Relation Validity

Implementations SHOULD validate that edges connect appropriate node kinds:

- `inherits` — SHOULD connect `class` to `class` (or `interface` to `interface`).
- `implements` — SHOULD connect `class` to `class` or `interface` where the target has an interface/protocol stereotype.
- `composes` — SHOULD connect `class` to `class`.
- `depends` — MAY connect any code-element nodes.
- `calls` — SHOULD connect `method` to `method`, or `class` to `class` (shorthand for "some method in A calls some method in B").
- `detail` — MUST connect `class` to `method` or `class` to `field`.
- `context` — MUST have at least one endpoint that is a context node (a node without `ccoding.kind`).

Implementations SHOULD warn on invalid combinations but MUST NOT reject the canvas file. Permissive reading ensures forward compatibility and cross-tool interoperability.

---

## 7. Edge Labels

The JSON Canvas `label` field is OPTIONAL but RECOMMENDED for CooperativeCoding edges. Labels serve two purposes: they provide human-readable context on the canvas, and they carry structured information that the sync engine uses (particularly for `composes` edges, where the label contains the field name).

### Label Conventions

| Relation | Label Carries | Examples |
|---|---|---|
| `composes` | Field name, optionally followed by a dash and composition semantics. The field name portion (before the first ` — `) is used by the sync engine to name the generated field. | `"config"`, `"plugins — Applied sequentially during parse(). Order matters."` |
| `depends` | What is used from the dependency and/or why the dependency exists. | `"TokenStream — used for lexical analysis"`, `"re — regex pattern matching"` |
| `calls` | When or why the call happens. Describes the runtime flow. | `"After tokenization"`, `"On validation failure"` |
| `inherits` | Nature of the inheritance relationship. | `"Base parsing interface"`, `"Abstract document handler"` |
| `implements` | The contract or capability being fulfilled. | `"Serialization support"`, `"Iterator protocol"` |
| `detail` | The method or field name that was promoted. | `"parse()"`, `"config"`, `"validate()"` |
| `context` | Type of context being provided. | `"rationale"`, `"reference"`, `"decision log"`, `"RFC link"` |

### Label Parsing

For `composes` edges, the sync engine MUST parse the label to extract the field name:

1. If the label contains ` — ` (space, em dash, space), the substring before the first ` — ` is the field name. The remainder is a descriptive comment.
2. If the label contains no ` — `, the entire label is the field name.
3. If the label is absent, the sync engine SHOULD derive a field name from the target node's name (lowercased, snake_cased per the language binding conventions).

For all other relation types, labels are informational. The sync engine MAY include label text in generated code comments but MUST NOT depend on label content for correctness.

---

## 8. Context Nodes

Context nodes are plain JSON Canvas nodes — text notes, file embeds, links, or groups — that carry no `ccoding.kind` metadata. They exist on the canvas as collaboration artifacts: design rationale, reference material, decision logs, external links, and any other non-code information that helps humans and agents reason about the architecture.

### Properties

- Context nodes have **no `ccoding` metadata by default**. They are standard JSON Canvas nodes in every respect.
- Context nodes are **NOT synced to code**. They exist only on the canvas. The sync engine MUST ignore them.
- Context nodes **can be connected** to CooperativeCoding code-element nodes via `context` edges, establishing a non-synced association between a design note and the code element it relates to.
- Context nodes **can be proposed as ghosts**. When an agent creates a context node as a suggestion, it attaches a minimal `ccoding` object containing only `status` and `proposedBy`. This makes the context node visible as a ghost without giving it code-element semantics.

### Ghost Context Node Example

```json
{
  "id": "note-1",
  "type": "text",
  "x": 400,
  "y": 200,
  "width": 250,
  "height": 150,
  "text": "Protocol chosen over ABC because ABCs cannot enforce method signatures at type-check time in Python 3.12+. Protocol also enables structural subtyping, which is more Pythonic for plugin systems.",
  "ccoding": {
    "status": "proposed",
    "proposedBy": "agent"
  }
}
```

In this example, the node has no `kind` — it is not a code element. The `ccoding` object serves only to mark it as a ghost proposal from the agent. When the human accepts this ghost, the `ccoding` object MAY be removed entirely (returning the node to a plain context node) or MAY be updated to `"status": "accepted"` if the implementation tracks context node lifecycle.

### Context Node with `context` Edge

A context node is typically connected to the code element it describes:

```json
{
  "id": "context-edge-1",
  "fromNode": "note-1",
  "toNode": "node-1",
  "label": "rationale",
  "ccoding": {
    "relation": "context"
  }
}
```

---

## 9. Node Text Content

The `text` field on a JSON Canvas node is a markdown string. CooperativeCoding treats this field as an opaque string at the spec level — the data model defines its type (string) and its preservation requirements, but does not prescribe its internal structure.

### Formal Requirements

- Implementations MUST preserve node text content exactly through read/write cycles. No normalization, no reformatting, no stripping of whitespace or markdown constructs. The text a human wrote is the text that persists.
- The spec does NOT prescribe a content format. There is no required heading structure, no mandatory sections, no enforced markdown dialect.
- Language bindings MAY define structured content conventions that map specific markdown sections to language constructs. For example, a Python binding might define that an `## Methods` section lists the class's methods with their signatures and descriptions. These conventions are defined by the binding, not by this spec.
- Canvas tool implementations MAY parse node text for rendering purposes (e.g., extracting the first heading as the node title) but MUST NOT modify the text as a side effect of rendering.

See [bindings/python.md](../bindings/python.md) for an example of structured node content conventions.

---

## 10. Edge Targeting Rule

Edges in CooperativeCoding always connect node to node. There is no sub-node anchoring — an edge cannot target a specific method or field within a class node's text content.

This constraint is deliberate. JSON Canvas v1.0 defines edges between nodes, not between regions within nodes. CooperativeCoding inherits this model exactly. The consequence is simple and consistent:

> **If a method or field needs to be the source or target of an edge, it MUST be promoted to its own detail node.**

This means a method that calls another method, a field that composes another class, or any element that participates in a relationship must exist as a standalone node on the canvas. The `detail` edge connects it back to its parent class, maintaining the structural relationship.

This rule keeps the data model aligned with JSON Canvas's native capabilities, ensures edges are unambiguous (every endpoint is a node with a unique `id`), and reinforces the progressive detail principle — only architecturally significant elements earn their own nodes.

### Example: Method-to-Method Call

If `DocumentParser.parse()` calls `PluginManager.apply()`, both methods must be promoted:

```json
{
  "nodes": [
    {
      "id": "class-parser",
      "type": "text",
      "x": 0, "y": 0, "width": 300, "height": 200,
      "text": "## DocumentParser\n\n> Parses raw documents into structured AST nodes",
      "ccoding": {
        "kind": "class",
        "qualifiedName": "parsers.document.DocumentParser",
        "status": "accepted"
      }
    },
    {
      "id": "method-parse",
      "type": "text",
      "x": 0, "y": 250, "width": 300, "height": 150,
      "text": "## parse()\n\n> Tokenize input, apply plugins, return AST",
      "ccoding": {
        "kind": "method",
        "qualifiedName": "parsers.document.DocumentParser.parse",
        "status": "accepted"
      }
    },
    {
      "id": "class-plugins",
      "type": "text",
      "x": 400, "y": 0, "width": 300, "height": 200,
      "text": "## PluginManager\n\n> Manages and applies parsing plugins",
      "ccoding": {
        "kind": "class",
        "qualifiedName": "parsers.plugins.PluginManager",
        "status": "accepted"
      }
    },
    {
      "id": "method-apply",
      "type": "text",
      "x": 400, "y": 250, "width": 300, "height": 150,
      "text": "## apply()\n\n> Run all registered plugins on the AST",
      "ccoding": {
        "kind": "method",
        "qualifiedName": "parsers.plugins.PluginManager.apply",
        "status": "accepted"
      }
    }
  ],
  "edges": [
    {
      "id": "detail-1",
      "fromNode": "class-parser",
      "toNode": "method-parse",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "parse()",
      "ccoding": { "relation": "detail" }
    },
    {
      "id": "detail-2",
      "fromNode": "class-plugins",
      "toNode": "method-apply",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "apply()",
      "ccoding": { "relation": "detail" }
    },
    {
      "id": "call-1",
      "fromNode": "method-parse",
      "toNode": "method-apply",
      "label": "After tokenization — applies all plugins to raw AST",
      "ccoding": { "relation": "calls", "status": "accepted" }
    }
  ]
}
```

This example shows the full chain: class nodes own method detail nodes via `detail` edges, and the `calls` edge connects the two method nodes directly. The canvas clearly shows both the structural hierarchy and the runtime flow.
