# CooperativeCoding Specification: Introduction

**Version 2.0.0**

---

> Current agentic coding treats software as text to be generated. CooperativeCoding treats it as architecture to be negotiated.

AI agents are remarkably good at implementing code that satisfies a given specification, but they are not yet capable of producing clean software designs, the kind that make a codebase expandable, reusable, and navigable by humans.

CooperativeCoding gives the human a visual canvas where architecture is defined in natural language: responsibilities, boundaries, pseudo code, and relationships. The canvas and the code stay in continuous bidirectional sync. Both the human and the agent can change either side, and every change is immediately visible on the canvas. The human maintains authority by editing the canvas at any time.

---

## The Core Insight

Code design and code implementation are different skills that benefit from different tools. Humans excel at architectural thinking: defining responsibilities, drawing boundaries, seeing the big picture. Agents excel at translating precise specifications into correct, complete code. 

Today, both activities happen in the same text editor, forcing the human to think at the wrong level of abstraction and the agent to guess at the right one.

CooperativeCoding separates these concerns:

- **Both work on a visual canvas**, a workspace where classes, interfaces, methods, fields, and their relationships are visible. Each element carries its own design content: responsibility statement and pseudo code. The canvas shows what truly matters: the architecture, the contracts, the design intent. Both humans and agents create, modify, and connect elements on this canvas.
- **Both also work on the code.** The agent implements source code from the canvas design, and the human can edit code directly. Sync keeps canvas and code aligned.
- **They cooperate.** The agent can loop autonomously (implement, test, fix, repeat) while every change stays visible on the canvas. The human can intervene at any point by editing the canvas or the code. Version control provides the audit trail, review, and rollback.

---

## Scope

This specification defines the open CooperativeCoding standard, a community contribution that any canvas tool, any programming language, and any agentic system can adopt. It is intentionally language-agnostic, tool-agnostic, and format-agnostic. The standard defines the design contract, the sync loop, and the contract that language bindings MUST fulfill. It does not prescribe how any particular tool renders elements, how any particular agent reasons about architecture, how any particular language maps its idioms, or what file format the canvas uses.

Concrete implementations require three things:

1. **A canvas tool implementation**: a plugin, extension, or application that lets humans and agents work with architectural elements visually. The tool may use any format or storage mechanism (JSON files, markdown vaults, databases, custom formats) as long as it fulfills the design contract and sync semantics defined in this spec.
2. **A language binding**: a mapping from the abstract CooperativeCoding spec to a specific programming language's idioms and conventions (e.g., Python docstrings, TypeScript JSDoc, Rust doc comments).
3. **A sync engine**: detects changes on both sides (canvas and code), produces sync requests, and tracks state. The agent queries the sync engine to know what needs to be updated.
4. **An agent integration**: a skill, plugin, or configuration that teaches an agentic coding tool (e.g., Claude Code, Cursor, Copilot) to work within the CooperativeCoding sync loop and principles.

---

## Core Principles

### 1. Design Authority is Human

Architectural decisions require judgment that agents do not have: business context, team dynamics, long-term maintainability tradeoffs. Both the human and the agent can change the canvas and the code, but the human always has final say. Every change is visible on the canvas, and the human can edit anything at any time. The canvas is where the human exercises architectural authority.

### 2. Every Element Has Identity

Every code element (class, method, field, module) that participates in the architecture has a stable identity that sync uses to link canvas representations to code constructs. This identity persists across changes on either side. The mechanism for establishing identity (qualified names, unique IDs, content hashing, or any other approach) is defined by each implementation, but the identity MUST be stable enough for sync to track elements across changes.

Implementations are free to choose how elements are visualized: as individual nodes, as sections within a document, as collapsible tree entries, or any other representation. The spec requires only that each code element is identifiable, not that it occupies a specific visual form.

### 3. Design Content is the Contract

Each code element on the canvas carries natural language content that defines its design intent. At minimum:

- **Every code element** MUST carry a **responsibility statement**: what this element owns in the system, its boundaries, its purpose.
- **Every method** MUST carry **pseudo code**: a step-by-step description of the algorithm in natural language. Deliberately not real code: it describes what happens, not how in language-specific terms.

This content is the authoritative specification for what the corresponding code should do. When a human writes "Validate input, check for duplicates, hash the password, create the user record" as a method's pseudo code, the agent implements against those steps. Clear design content leads to correct implementation; vague content leads to guesswork.

Language bindings define additional content that elements MAY carry (constraints, signatures, collaborators, etc.) and how all content maps to the language's documentation format.

### 4. Bidirectional Sync

Canvas and code are always in sync. Every change on either side produces a sync request for the other side. The agent processes each sync request and decides what changes are needed, which may be nothing if the other side already matches. The human sees every change reflected on the canvas in real time.

Version control provides the audit trail and rollback capability.

### 5. Visual Simplicity

The canvas shows architecture, not code. Each element carries a responsibility statement and pseudo code that let you understand what a component does without reading the implementation. This makes it possible to see at a glance whether responsibilities are well distributed, whether a class is doing too much, or whether the system's structure is clean and navigable.

### 6. Any Entry Point

Different projects start from different places. A greenfield system starts from a blank canvas or a natural-language description. A legacy system starts from existing code that needs to be understood and evolved. CooperativeCoding supports all three: sketch elements by hand on a blank canvas, describe a system in natural language and let the agent create the initial design, or import existing source files and generate the canvas representation from code. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 7. Language-Agnostic, Tool-Agnostic, Format-Agnostic

The core concepts of CooperativeCoding (identity, design content, relationships, the sync loop) are universal. They apply whether you're writing Python or Rust, using Obsidian or VS Code, storing canvases as JSON or markdown. Each programming language gets a concrete binding that maps the spec to its idioms. Each canvas tool gets an implementation that maps the rendering to its capabilities. Each agent gets an integration that teaches it the sync loop. The core spec defines the contract; the bindings and implementations fulfill it.

---

## Terminology

This section defines the key terms used throughout the CooperativeCoding specification. These definitions are normative. When these terms appear in the spec documents, they carry exactly the meaning given here.

**Canvas**
A visual workspace where architectural elements and their relationships are represented. A canvas may be stored in any format (JSON, markdown, database, etc.) and rendered by any tool, as long as the implementation fulfills the design contract and sync semantics defined in this spec.

**Code element**
A construct in the source code that participates in the architecture: a class, method, field, function, module, package, test, or any other identifiable unit. Each code element has a stable identity, a responsibility statement, and (for methods) pseudo code. The set of code element types is not prescribed by this spec; language bindings define what types exist for their target language.

**Binding** (language binding)
A mapping from the abstract CooperativeCoding specification to a specific programming language's idioms and conventions. A binding defines how design content maps to the language's documentation format, what stereotypes are valid, how the language's type system is represented, and how code is parsed and generated.

**Stereotype**
A language-specific subtype that refines what a code element means in a given language. For example, `protocol`, `dataclass`, and `enum` are stereotypes of a class in a Python binding, while `interface`, `type`, and `enum` serve the same role in a TypeScript binding. The set of valid stereotypes is an open set defined by each language binding. Implementations MUST NOT reject unknown stereotype values.

**Sync**
The bidirectional process of keeping the canvas design and the source code aligned. Every change on either side produces a sync request for the other side. Canvas changes produce code requests (requests to update the code). Code changes produce design requests (requests to update the canvas).

**Sync request**
A change that the agent needs to process. Code requests update the code to match the canvas. Design requests update the canvas to match the code. The agent evaluates each request and applies the necessary changes, which may be nothing if the target side already matches.

**Qualified name**
A stable identifier for a code element used to link canvas representations to code constructs. The format is defined by each language binding (e.g., dot-notation like `parsers.document.DocumentParser` for Python). This is the primary mechanism for identity matching during sync.

**Context element**
Any canvas content that carries no code identity: design rationale, reference material, decision logs, external links, and other collaboration artifacts. Context elements are not synced to code. They exist to help humans and agents reason about the architecture.

---

## RFC 2119 Keywords

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Versioning

The CooperativeCoding specification uses [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH):

- **MAJOR**: breaking changes to the design contract, sync semantics, or binding contract. Implementations targeting a previous major version are not guaranteed to work.
- **MINOR**: backwards-compatible additions. New optional requirements, new guidance. Existing implementations continue to work but MAY not support the new features.
- **PATCH**: clarifications, editorial fixes, and examples. No behavioral changes.

Implementations SHOULD declare which spec version they target.

---

## Non-Goals

The following topics are explicitly out of scope for this specification. They are important concerns, but they belong to specific implementations rather than the universal standard.

- **Canvas format and storage**: How the canvas is stored (JSON, markdown, database), what file format it uses, and how elements are serialized. These are implementation choices.
- **Canvas tool UX conventions**: How elements are rendered, how they are arranged, what interactions are available. These are left to each canvas tool implementation.
- **Relationship representation**: How relationships between code elements are visualized (edges, wiki links, containment, annotations) and what types of relationships are tracked. Implementations define their own relationship vocabulary based on what their language binding needs for code generation.
- **Agent behavior and skill design**: How an agent reasons about architecture, when it makes changes, what heuristics it uses to detect design issues. These are left to each agent integration. The spec defines the sync loop, not the intelligence behind it.
- **Sync engine implementation**: How the sync engine detects changes, tracks state, and produces sync requests. The spec defines what sync must do, not how. Implementations may use version control, content hashing, file watchers, or any other mechanism.
- **Layout and arrangement**: How elements are arranged on the canvas (force-directed, hierarchical, manual, grid) is an implementation detail.

---

## Security Considerations

- **Path traversal:** Source mappings that link canvas elements to code files MUST be validated to not escape the project root directory.
- **Content sanitization:** Design content is often rendered as markdown or rich text. Implementations SHOULD guard against script injection in rendered content.
- **Agent trust:** Agents can change both canvas and code. Implementations SHOULD provide audit trails for all changes.

---

## Document Index

This introduction is the first of four documents that make up the CooperativeCoding specification:

1. **Introduction** (this document): vision, principles, terminology, and scope
2. [Design Contract](01-design-contract.md): what every code element must carry and how implementations track them
3. [Sync](02-sync.md): sync loop, convergence, and body preservation rules
4. [Language Bindings](03-language-bindings.md): contract for language-specific mappings
