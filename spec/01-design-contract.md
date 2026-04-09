# CooperativeCoding Specification: Design Contract

**Version 2.0.0**

---

## 1. Overview

The [Introduction](00-introduction.md) defines the vision and principles. This document defines the design contract: what every code element must carry, how implementations identify and track elements, and what relationships must be maintained for sync to operate.

The design contract is deliberately abstract. It defines the requirements that any implementation must fulfill, regardless of what canvas format, visualization approach, or storage mechanism it uses. The contract is the bridge between the spec's principles and a working implementation.

---

## 2. Code Element Identity

Every code element that participates in the architecture MUST have a stable identity. This identity is what sync uses to link a canvas representation to its corresponding code construct. Without stable identity, sync cannot track elements across changes.

### Requirements

- Each code element MUST have a unique identity within the project scope.
- The identity MUST be stable across changes to the element's content, position, or visualization on the canvas.
- The identity MUST be resolvable to a specific construct in the source code.
- When a code element is renamed in code, sync MUST be able to detect the rename and update the canvas identity accordingly (or flag it for human attention).

### Qualified Names

The RECOMMENDED approach for identity is a **qualified name**: a dot-notation identifier that locates the element within the language's namespace hierarchy. For example, `parsers.document.DocumentParser` identifies a class, and `parsers.document.DocumentParser.parse` identifies one of its methods.

Language bindings define the format of qualified names for their target language. Implementations MAY use alternative identity mechanisms (UUIDs, content hashing, etc.) as long as they fulfill the stability and resolvability requirements above.

---

## 3. Design Content

Every code element on the canvas carries natural language content that defines its design intent. This content is the authoritative specification for what the corresponding code should do.

### 3.1 Responsibility Statement (REQUIRED)

Every code element MUST carry a **responsibility statement**: a natural language description of what this element owns in the system. The responsibility defines boundaries: what this element does, and implicitly, what it does not do.

A good responsibility statement answers: "If I need to change X, is this the element I modify?" It is the single most important piece of design content because it defines the element's reason for existing.

### 3.2 Pseudo Code (REQUIRED for methods)

Every method or function MUST carry **pseudo code**: a numbered, step-by-step description of the algorithm in natural language.

Pseudo code is deliberately not real code. It describes what happens, not how in language-specific terms. This keeps the canvas readable by humans who may not know the target language and ensures the design intent survives language changes.

Code is implemented against the pseudo code steps. When a human writes:

```
1. Check cache for source hash
2. If cached, return cached result
3. Tokenize source using config
4. Build AST from token stream
5. Validate structure
6. Cache and return
```

...the implementation follows those steps. The pseudo code is the contract between the design intent and the implementation.

### 3.3 Additional Content (OPTIONAL)

Language bindings MAY define additional content that elements can carry. Common examples include:

- **Constraints**: invariants, thread-safety guarantees, performance requirements
- **Signatures**: parameter types, return types, exceptions
- **Collaborators**: other elements this one works with and why

The core spec does not prescribe these. Each binding decides what additional content is useful for its target language and how it maps to the language's documentation format.

---

## 4. Relationships

Code elements exist in relationship to each other: a class inherits from another, a method calls another method, a module depends on another module. These relationships affect code generation (inheritance declarations, import statements, typed fields) and architectural visibility (seeing how components connect).

### Requirements

- Implementations MUST track relationships between code elements that affect code generation. If a relationship produces a code construct (an inheritance declaration, an import statement, a typed field), the implementation must know about it so sync can generate and update the corresponding code.
- Implementations MUST make relationships visible on the canvas in some form, so that humans can see how components connect. The visualization mechanism (edges, links, containment, annotations, proximity, or any other approach) is an implementation choice.
- When a relationship changes on either side (canvas or code), sync MUST propagate the change to the other side.

### No Prescribed Relationship Types

This spec does not enumerate a fixed set of relationship types. Different languages have different relationship semantics (Python's protocols vs. Java's interfaces, Go's implicit interfaces vs. TypeScript's explicit ones), and different canvas tools have different ways of representing relationships.

Each language binding defines what relationships matter for its target language and how they map to code constructs. Each canvas implementation defines how those relationships are visualized.

---

## 5. Source Mapping

Code elements on the canvas MUST be mappable to their location in the source code. This is how sync knows where to read and write code.

### Requirements

- Each code element SHOULD carry a reference to its source file.
- When no source mapping exists (new element on canvas), the language binding defines where the source file should be created based on the element's identity and the project's file conventions.

---

## 6. Stereotypes

A stereotype is a language-specific subtype that refines what a code element means in a given language. For example, a "class" might be a plain class, a protocol, a dataclass, an enum, or an abstract base class in Python. Each of these produces different code.

### Requirements

- The set of valid stereotypes is an open set defined by each language binding.
- A binding MAY define additional stereotypes at any time. Adding a stereotype is a backwards-compatible change.

---

## 7. Context Elements

Canvas content that carries no code identity (design rationale, reference material, decision logs, external links) is a context element. Context elements are not synced to code. They exist to help humans and agents reason about the architecture.

- Context elements have no identity in the sync system.
- Changes to context elements do not produce sync requests.
- Context elements MAY be visually connected to code elements for reference purposes.
- Implementations define how context elements are stored and visualized.

---

## 8. Multi-Language Support

A single canvas MAY contain code elements targeting different programming languages. Each element is synced using the binding for its own language.

- Implementations SHOULD make the target language of each element visible on the canvas.
