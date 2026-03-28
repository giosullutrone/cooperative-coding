# CooperativeCoding Specification: Introduction

**Version 1.0.0**

---

> Current agentic coding treats software as text to be generated. CooperativeCoding treats it as architecture to be negotiated.

AI agents are remarkably good at implementing code that satisfies a given specification, but they are not yet capable of producing clean software designs, the kind that make a codebase expandable, reusable, and navigable by humans.

CooperativeCoding gives the human a visual canvas where architecture is defined in natural language: responsibilities, boundaries, pseudo code, and relationships. The canvas and the code stay in continuous bidirectional sync. Both the human and the agent can change either side, and every change is immediately visible on the canvas. The human maintains authority by editing the canvas at any time.

---

## The Core Insight

Code design and code implementation are different skills that benefit from different tools. Humans excel at architectural thinking: defining responsibilities, drawing boundaries, seeing the big picture. Agents excel at translating precise specifications into correct, complete code. 

Today, both activities happen in the same text editor, forcing the human to think at the wrong level of abstraction and the agent to guess at the right one.

CooperativeCoding separates these concerns:

- **Both work on a visual canvas**, a simplified UML where classes, interfaces, methods, fields, and their relationships are first-class objects. Each element carries its own node content: responsibility statement and pseudo code. The canvas shows what truly matters: the architecture, the contracts, the design intent. Both humans and agents create, modify, and connect elements on this canvas.
- **Both also work on the code.** The agent implements source code from the canvas design, and the human can edit code directly. Sync keeps canvas and code aligned.
- **They cooperate.** The agent can loop autonomously (implement, test, fix, repeat) while every change stays visible on the canvas. The human can intervene at any point by editing the canvas or the code. Version control provides the audit trail, review, and rollback.

---

## Scope

This specification defines the open CooperativeCoding standard, a community contribution that any canvas tool, any programming language, and any agentic system can adopt. It is intentionally language-agnostic and tool-agnostic. The standard defines the data model, the sync loop, and the contract that language bindings MUST fulfill. It does not prescribe how any particular tool renders nodes, how any particular agent reasons about architecture, or how any particular language maps its idioms.

Concrete implementations require three things:

1. **A canvas tool implementation**: a plugin or extension for a visual canvas application (e.g., Obsidian, VS Code, Excalidraw, tldraw) that renders CooperativeCoding nodes.
2. **A language binding**: a mapping from the abstract CooperativeCoding spec to a specific programming language's idioms and conventions (e.g., Python docstrings, TypeScript JSDoc, Rust doc comments).
3. **A sync engine**: detects changes on both sides (canvas and code), produces sync requests, and tracks state. The agent queries the sync engine to know what needs to be updated.
4. **An agent integration**: a skill, plugin, or configuration that teaches an agentic coding tool (e.g., Claude Code, Cursor, Copilot) to work within the CooperativeCoding sync loop and principles.

---

## Core Principles

### 1. Design Authority is Human

Architectural decisions require judgment that agents do not have: business context, team dynamics, long-term maintainability tradeoffs. Both the human and the agent can change the canvas and the code, but the human always has final say. Every change is visible on the canvas, and the human can edit anything at any time. The canvas is where the human exercises architectural authority.

### 2. Every Element is a Node

Every class, method, and field is its own node on the canvas, carrying full documentation: responsibility, pseudo code, signatures, and constraints. Each member exists independently, with its own identity and detail.

The relationship between a class and its members (methods, fields) MUST be expressed on the canvas. Implementations may use edges, group containment, nesting, or any other mechanism native to their canvas tool, as long as the semantic parent-child relationship is preserved and queryable by the sync engine.

Canvas tools MAY provide ways to collapse, hide, or visually simplify nodes for readability. The underlying data always carries the full detail.

### 3. Node Content is the Contract

Each node on the canvas carries natural language content: a responsibility statement, pseudo code, type signatures, and constraints. This content is the authoritative specification for what the corresponding code should do. 

When a human writes "Validate input, check for duplicates, hash the password, create the user record" as a method's pseudo code, the agent implements against those steps. Clear node content leads to correct implementation; vague content leads to guesswork.

### 4. Bidirectional Sync

Canvas and code are always in sync. Every change on either side produces a sync request for the other side. The agent processes each sync request and decides what changes are needed, which may be nothing if the other side already matches. The human sees every change reflected on the canvas in real time.

Version control provides the audit trail and rollback capability.

### 5. Visual Simplicity

The canvas shows architecture, not code. Each node carries a responsibility statement and pseudo code that let you understand what a component does without reading the implementation. This makes it possible to see at a glance whether responsibilities are well distributed, whether a class is doing too much, or whether the system's structure is clean and navigable.

The visual vocabulary is intentionally small: a handful of node kinds, a handful of edge relations, and structured markdown for content. The canvas MUST remain a place where you can see the shape of the system at a glance.

### 6. Any Entry Point

Different projects start from different places. A greenfield system starts from a blank canvas or a natural-language description. A legacy system starts from existing code that needs to be understood and evolved. CooperativeCoding supports all three: sketch nodes by hand on a blank canvas, describe a system in natural language and let the agent create the initial design, or import existing source files and generate the canvas representation from code. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 7. Language-Agnostic, Tool-Agnostic

The core concepts of CooperativeCoding (nodes, edges, relationships, the sync loop) are universal. They apply whether you're writing Python or Rust, using Obsidian or VS Code, collaborating with Claude or Copilot. Each programming language gets a concrete binding that maps the spec to its idioms. Each canvas tool gets an implementation that maps the rendering to its capabilities. Each agent gets an integration that teaches it the sync loop. The core spec defines the contract; the bindings fulfill it.

---

## Terminology

This section defines the key terms used throughout the CooperativeCoding specification. These definitions are normative. When these terms appear in the spec documents, they carry exactly the meaning given here.

**Canvas**
A visual workspace containing nodes and edges, stored as a [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/) file enriched with CooperativeCoding metadata in the `ccoding` namespace. A single canvas file represents one architectural view of a system or subsystem.

**Node**
A visual element on the canvas representing either a code element (class, method, field, package) or a context item (note, file, link). Code element nodes carry `ccoding` metadata that identifies their kind, language, source file mapping, and other metadata. Context item nodes are standard JSON Canvas nodes without `ccoding.kind` metadata.

**Edge**
A connection between two nodes representing a relationship. Each edge carries a `ccoding.relation` type that defines its semantics: `inherits`, `implements`, `composes`, `depends`, `calls`, `tests`, or `context`. Edges MAY carry a `label` that describes the nature, purpose, and constraints of the relationship in human-readable text.

**Binding** (language binding)
A mapping from the abstract CooperativeCoding specification to a specific programming language's idioms and conventions. A binding defines the valid stereotypes for that language, how structured markdown sections map to the language's documentation format, how canvas elements map to code constructs, and how the language's type system is represented in node content.

**Sync**
The bidirectional process of keeping the canvas design and the source code aligned. Every change on either side produces a sync request for the other side. Canvas changes produce code requests (requests to update the code). Code changes produce design requests (requests to update the canvas).

**Sync request**
A change that the agent needs to process. Code requests update the code to match the canvas. Design requests update the canvas to match the code. The agent evaluates each request and applies the necessary changes, which may be nothing if the target side already matches.

**Test node**
A code-element node with `ccoding.kind` set to `"test"` representing a test class or test suite that verifies the behavior of one or more classes in the system. A test node's structured content includes the test class name, pseudo code of the test methods describing the scenarios being verified, and test execution results (pass, fail, or error with details). Test nodes are connected to the classes or methods they verify via `tests` edges.

**Context node**
A plain JSON Canvas node (text, file, or link type) without `ccoding.kind` metadata, used for design rationale, references, decision logs, and other collaboration artifacts. Context nodes are not synced to code. They MAY be connected to code element nodes via `context` edges.

**Stereotype**
A language-specific subtype of a node kind. Stereotypes refine what a "class" means in a given language. For example, `protocol`, `dataclass`, and `enum` are stereotypes of the `class` kind in a Python binding, while `interface`, `type`, and `enum` serve the same role in a TypeScript binding. The set of valid stereotypes is an open set defined by each language binding.

**Qualified name**
The fully qualified identifier for a code element in the target language, used to establish a stable identity link between a canvas node and its corresponding code construct. For example, `parsers.document.DocumentParser` identifies a class within a Python module hierarchy, and `parsers.document.DocumentParser.parse` identifies one of its methods.

---

## RFC 2119 Keywords

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Versioning

The CooperativeCoding specification uses [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH):

- **MAJOR**: breaking changes to the data model, lifecycle semantics, or sync contract. Implementations targeting a previous major version are not guaranteed to work.
- **MINOR**: backwards-compatible additions. New node kinds, new edge relations, new optional metadata fields. Existing implementations continue to work but MAY not support the new features.
- **PATCH**: clarifications, editorial fixes, and examples. No behavioral changes.

Implementations SHOULD declare which spec version they target. Every canvas file MUST include `ccoding.specVersion`. Tools encountering a canvas without `specVersion` SHOULD treat it as v1.0.0 for backward compatibility.

### Migration

When a new MAJOR version introduces breaking changes to the `ccoding` namespace, the spec repository MUST publish a migration guide.

Implementations SHOULD provide an automated migration tool or command that upgrades canvas files from version N to N+1.

During migration, implementations MUST preserve all node and edge content. Metadata transformations MUST be lossless: no user data may be discarded.

If an implementation encounters a canvas with a `specVersion` newer than it supports, it MUST warn the user and MUST NOT silently modify the file.

### Forward Compatibility

Stereotypes are open-set: implementations MUST NOT reject unknown stereotype values (see §01-data-model).

Node kinds and edge relations are closed-set for a given spec version. Implementations encountering an unknown `kind` value SHOULD preserve it and render the node as a generic card. Implementations encountering an unknown `relation` value SHOULD preserve it and render the edge as a plain arrow.

Implementations MUST preserve any unrecognized fields in the `ccoding` metadata object. This allows higher spec versions to add new fields without breaking older tools.

---

## Non-Goals

The following topics are explicitly out of scope for this specification. They are important concerns, but they belong to specific implementations rather than the universal standard.

- **Node content format**: The structured markdown format for node text is defined at the spec level as a recommended convention. Language bindings define the concrete section mappings. Implementations MAY extend the format for their needs.
- **Canvas tool UX conventions**: How nodes are rendered, how context nodes are highlighted on selection. These are left to each canvas tool implementation.
- **Agent behavior and skill design**: How an agent reasons about architecture, when it makes changes, what heuristics it uses to detect design issues. These are left to each agent integration. The spec defines the sync loop, not the intelligence behind it.
- **Sync engine implementation**: How the sync engine detects changes, tracks state, and produces sync requests. The spec defines what sync must do, not how. Implementations may use version control, content hashing, file watchers, or any other mechanism.
- **Layout algorithms**: How nodes are arranged on the canvas (force-directed, hierarchical, manual) is an implementation detail. The spec stores absolute positions per JSON Canvas but does not prescribe how they are determined.

---

## Security Considerations

- **Path traversal:** `ccoding.source` paths MUST be validated to not escape the project root directory.
- **Content sanitization:** Node text is rendered as markdown. Implementations SHOULD guard against script injection in rendered content.
- **Agent trust:** Agents can change both canvas and code. Implementations SHOULD provide audit trails for all changes.

---

## JSON Schemas

Machine-readable JSON Schemas for validating `ccoding` metadata are provided in the [`spec/schema/`](schema/) directory:

| Schema | Validates |
|--------|-----------|
| [`ccoding-canvas.schema.json`](schema/ccoding-canvas.schema.json) | Top-level `ccoding` object on the canvas |
| [`ccoding-node.schema.json`](schema/ccoding-node.schema.json) | `ccoding` object on individual nodes |
| [`ccoding-edge.schema.json`](schema/ccoding-edge.schema.json) | `ccoding` object on individual edges |

These schemas encode the field types, allowed values, and requirements. Implementations SHOULD use these schemas or equivalent validation logic. All schemas allow additional properties for forward compatibility.

---

## Document Index

This introduction is the first of five documents that make up the CooperativeCoding specification:

1. **Introduction** (this document): vision, principles, terminology, and scope
2. [Data Model](01-data-model.md): canvas nodes, edges, and metadata schemas
3. [Lifecycle](02-lifecycle.md): sync loop, entry points, and roles
4. [Sync](03-sync.md): sync semantics
5. [Language Bindings](04-language-bindings.md): contract for language-specific mappings
