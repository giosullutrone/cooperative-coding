# CooperativeCoding Specification — Introduction

**Version 1.0.0**

---

> Current agentic coding treats software as text to be generated. CooperativeCoding treats it as architecture to be negotiated.

AI agents can write code, but they can't share architectural intent with humans. The result is drift, loss of design coherence, and humans reduced to code reviewers. CooperativeCoding is an open standard that fixes this by giving architecture a machine-readable, tool-agnostic home — where humans and agents co-design, with a shared source of truth between them.

---

## The Core Insight

Code design and code implementation are different skills that benefit from different tools. Humans excel at architectural thinking — defining responsibilities, drawing boundaries, seeing the big picture. Agents excel at translating precise specifications into correct, complete code. Today, both activities happen in the same text editor, forcing the human to think at the wrong level of abstraction and the agent to guess at the right one.

CooperativeCoding separates these concerns:

- **Both work on a visual canvas** — a simplified UML where classes, interfaces, methods, fields, and their relationships are first-class objects. Each element carries a documentation block with its responsibility and pseudo code. The canvas shows what truly matters: the architecture, the contracts, the design intent. Both humans and agents create, modify, and connect elements on this canvas.
- **The agent also works on the code** — it reads the canvas design, implements the underlying source code, and keeps canvas and code in bidirectional sync.
- **They cooperate** — either party can propose architectural changes. Agent-created elements appear as ghost proposals (awaiting human review). The human remains the final design authority — accepting, rejecting, or modifying proposals before anything gets implemented.

---

## Scope

This specification defines the open CooperativeCoding standard — a community contribution that any canvas tool, any programming language, and any agentic system can adopt. It is intentionally language-agnostic and tool-agnostic. The standard defines the data model, the cooperation lifecycle, the sync semantics, and the contract that language bindings MUST fulfill. It does not prescribe how any particular tool renders nodes, how any particular agent reasons about architecture, or how any particular language maps its idioms.

Concrete implementations require three things:

1. **A canvas tool implementation** — a plugin or extension for a visual canvas application (e.g., Obsidian, VS Code, Excalidraw, tldraw) that renders CooperativeCoding nodes and supports the cooperation UX.
2. **A language binding** — a mapping from the abstract CooperativeCoding spec to a specific programming language's idioms and conventions (e.g., Python docstrings, TypeScript JSDoc, Rust doc comments).
3. **An agent integration** — a skill, plugin, or configuration that teaches an agentic coding tool (e.g., Claude Code, Cursor, Copilot) to work within the CooperativeCoding paradigm.

---

## Core Principles

### 1. Design Authority is Human

Software architecture is a discipline of tradeoffs — performance versus readability, flexibility versus simplicity, speed-to-market versus long-term maintainability. These tradeoffs require judgment that emerges from business context, team culture, and lived experience. Agents can contribute valuable design ideas, but lack the full picture of these constraints. In CooperativeCoding, the human always has final say on architecture. The agent proposes, the human disposes. No design change — no new class, no new relationship, no restructured inheritance hierarchy — is applied without explicit human approval. Agents MUST express design ideas as ghost proposals and MUST NOT modify accepted nodes without going through the proposal workflow.

### 2. Progressive Detail

Not everything in a system deserves the same level of attention. A utility method that formats a timestamp is not architecturally significant; a method that orchestrates a complex pipeline is. CooperativeCoding embraces this asymmetry through progressive detail: class nodes give the overview, showing fields and methods as inline lists. Only when a method or field needs design attention — complex logic, important constraints, or a role as an edge target — does it get promoted to its own detail node. The canvas highlights architecture, not every line of code. This keeps the visual workspace readable and forces both human and agent to focus on what matters.

### 3. Documentation as Contract

The documentation block is the bridge between the canvas and the code. It contains the responsibility statement, pseudo code, type signatures, and constraints that both the human and agent rely on. When a human writes "Transform raw source into a validated AST, applying all registered plugins in order" as a method's responsibility, that sentence becomes a contract. The agent reads it to generate the implementation. The sync engine reads it to detect drift. Clear documentation leads to correct implementation; vague documentation leads to guesswork. Implementations SHOULD treat the documentation block as the authoritative specification for what code should do.

### 4. Bidirectional Truth

Canvas and code are two views of the same system. A class exists on the canvas as a node with fields, methods, and relationships. The same class exists in code as a file with a definition, imports, and documentation. Neither view is "primary." Changes in either MUST propagate to the other through the sync engine. When the human renames a method on the canvas, the code updates. When the agent adds a field in code, the canvas updates. This bidirectionality is what keeps the shared source of truth honest — without it, the canvas becomes stale documentation that no one trusts.

### 5. Visual Simplicity

The canvas shows a simplified UML focused on what matters for architectural thinking: classes, interfaces, packages, dependencies, and method call flow. It deliberately omits what doesn't serve design — implementation details, control flow within methods, line-by-line logic. The visual vocabulary is intentionally small: a handful of node kinds, a handful of edge relations, and structured markdown for content. This constraint is a feature. A canvas cluttered with every implementation detail is no better than reading the source code directly. The canvas SHOULD remain a place where you can see the shape of the system at a glance.

### 6. Any Entry Point

Different projects start from different places. A greenfield system starts from a blank canvas or a natural-language description. A legacy system starts from existing code that needs to be understood and evolved. CooperativeCoding supports all three entry points: sketch nodes by hand on a blank canvas, describe a system in natural language and let the agent propose an initial design as ghost nodes, or import existing source files and let the sync engine generate the canvas representation. The paradigm meets you where you are. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 7. Language-Agnostic, Tool-Agnostic

The concepts at the heart of CooperativeCoding — nodes, edges, documentation blocks, ghost proposals, the cooperation lifecycle — are universal. They apply whether you're writing Python or Rust, using Obsidian or VS Code, collaborating with Claude or Copilot. Each programming language gets a concrete binding that maps the abstract spec to its idioms (stereotypes, documentation format, type notation). Each canvas tool gets an implementation that maps the abstract rendering to its capabilities. Each agent gets an integration that teaches it the cooperation workflow. The core spec defines the contract; the bindings fulfill it.

---

## Terminology

This section defines the key terms used throughout the CooperativeCoding specification. These definitions are normative — when these terms appear in the spec documents, they carry exactly the meaning given here.

**Canvas**
A visual workspace containing nodes and edges, stored as a [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/) file enriched with CooperativeCoding metadata in the `ccoding` namespace. A single canvas file represents one architectural view of a system or subsystem.

**Node**
A visual element on the canvas representing either a code element (class, method, field, package) or a context item (note, file, link). Code element nodes carry `ccoding` metadata that identifies their kind, language, source file mapping, and lifecycle status. Context item nodes are standard JSON Canvas nodes without `ccoding.kind` metadata.

**Edge**
A connection between two nodes representing a relationship. Each edge carries a `ccoding.relation` type that defines its semantics: `inherits`, `implements`, `composes`, `depends`, `calls`, `detail`, or `context`. Edges MAY carry a `label` that describes the nature, purpose, and constraints of the relationship in human-readable text.

**Ghost**
A node or edge with `status: "proposed"` in its `ccoding` metadata that awaits human review. Ghosts are created by agents proposing design changes or by humans sketching tentative ideas. Each ghost carries a `proposedBy` field (`"agent"` or `"human"`) and an optional `proposalRationale` explaining the reasoning behind the proposal. A ghost becomes real when the human accepts it; it becomes invisible when the human rejects it.

**Binding** (language binding)
A mapping from the abstract CooperativeCoding specification to a specific programming language's idioms and conventions. A binding defines the valid stereotypes for that language, how structured markdown sections map to the language's documentation format, how canvas elements map to code constructs, and how the language's type system is represented in node content.

**Sync**
The bidirectional process of keeping the canvas design and the source code aligned. When canvas elements change, sync updates the corresponding code. When code changes, sync updates the corresponding canvas nodes. Sync MUST detect conflicts when both sides change the same element and MUST NOT silently overwrite either side.

**Detail node**
A method or field that has been promoted from an inline entry within a class node to its own standalone node on the canvas. A detail node is connected to its parent class node by a `detail` edge. Promotion happens when the element needs design attention — complex logic, important constraints, or a role as the source or target of another edge.

**Context node**
A plain JSON Canvas node (text, file, or link type) without `ccoding.kind` metadata, used for design rationale, references, decision logs, and other collaboration artifacts. Context nodes are not synced to code. They MAY be connected to code element nodes via `context` edges, and they MAY carry minimal `ccoding` metadata (`status` and `proposedBy`) when proposed as ghosts by an agent.

**Stereotype**
A language-specific subtype of a node kind. Stereotypes refine what a "class" means in a given language — for example, `protocol`, `dataclass`, and `enum` are stereotypes of the `class` kind in a Python binding, while `interface`, `type`, and `enum` serve the same role in a TypeScript binding. The set of valid stereotypes is an open set defined by each language binding.

**Qualified name**
The fully qualified identifier for a code element in the target language, used to establish a stable identity link between a canvas node and its corresponding code construct. For example, `parsers.document.DocumentParser` identifies a class within a Python module hierarchy, and `parsers.document.DocumentParser.parse` identifies one of its methods.

---

## RFC 2119 Keywords

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Versioning

The CooperativeCoding specification uses [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH):

- **MAJOR** — breaking changes to the data model, lifecycle semantics, or sync contract. Implementations targeting a previous major version are not guaranteed to work.
- **MINOR** — backwards-compatible additions. New node kinds, new edge relations, new optional metadata fields. Existing implementations continue to work but MAY not support the new features.
- **PATCH** — clarifications, editorial fixes, and examples. No behavioral changes.

Implementations SHOULD declare which spec version they target. A canvas file SHOULD include the spec version in its metadata so that tools can detect compatibility.

---

## Non-Goals

The following topics are explicitly out of scope for this specification. They are important concerns, but they belong to specific implementations rather than the universal standard.

- **Node content format** — The structured markdown format for node text is defined at the spec level as a recommended convention. Language bindings define the concrete section mappings. Implementations MAY extend the format for their needs.
- **Canvas tool UX conventions** — How nodes are rendered, how ghost accept/reject controls appear, how context nodes are highlighted on selection — these are left to each canvas tool implementation. The spec provides recommended visual styles but does not mandate them.
- **Agent behavior and skill design** — How an agent reasons about architecture, when it proposes changes, what heuristics it uses to detect design issues — these are left to each agent integration. The spec defines the cooperation protocol, not the intelligence behind it.
- **Layout algorithms** — How nodes are arranged on the canvas (force-directed, hierarchical, manual) is an implementation detail. The spec stores absolute positions per JSON Canvas but does not prescribe how they are determined.

---

## Document Index

This introduction is the first of five documents that make up the CooperativeCoding specification:

1. **Introduction** (this document) — vision, principles, terminology, and scope
2. [Data Model](01-data-model.md) — canvas nodes, edges, and metadata schemas
3. [Lifecycle](02-lifecycle.md) — ghost proposals and status transitions
4. [Sync](03-sync.md) — bidirectional sync semantics
5. [Language Bindings](04-language-bindings.md) — contract for language-specific mappings
