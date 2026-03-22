# CooperativeCoding Specification — Sync

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures. The [Lifecycle](02-lifecycle.md) defines the motion of proposals and statuses. This document defines the bridge: how the visual canvas and the source code stay aligned as both evolve.

Sync is the bidirectional process that keeps canvas design and source code in agreement. When a human renames a method on the canvas, the corresponding code updates. When an agent adds a field in code, the corresponding canvas node updates. When both sides change the same element at the same time, sync detects the conflict and refuses to silently overwrite either side.

Bidirectionality is what makes the canvas trustworthy. Without it, the canvas degrades into stale documentation — a pretty picture that no one believes. With it, the canvas becomes a live architectural view that always reflects the real system. This document defines the abstract sync rules that all implementations MUST follow, regardless of canvas tool, programming language, or agent integration.

The rules in this document are intentionally abstract. They define *what* must happen, not *how* an implementation achieves it. A CLI tool, an IDE plugin, and a standalone sync daemon may all use different mechanisms — file watchers, save hooks, content hashing, AST diffing — but they MUST all produce behavior consistent with these rules.

---

## 2. Truth Model

Canvas and code are two complementary views of the same system. Neither is "primary." Each is the source of truth for a different dimension of the software:

- **Canvas is source of truth for architecture** — what classes exist, how they relate to each other, what their responsibilities are, what design intent governs them. The canvas captures the shape of the system: inheritance hierarchies, composition relationships, interface contracts, and the documentation blocks that define each element's purpose and constraints.

- **Code is source of truth for implementation details** — function bodies, local variables, control flow, performance optimizations, error handling strategies, runtime behavior, and everything that lives inside a method body. The code captures how the system actually works at the line-by-line level.

- **Neither side can unilaterally overwrite the other.** When canvas and code disagree about the same element, the sync engine MUST detect the conflict and present it for human resolution. There is no automatic winner.

This truth model produces a clean separation of concerns. The human controls architecture on the canvas. The agent (or human) controls implementation in code. The sync engine is the mediator that keeps the two views consistent. When the boundary is ambiguous — a method signature change that affects both architecture and implementation — the conflict detection mechanism ensures a human makes the call.

---

## 3. Canvas → Code Sync Rules

When accepted elements on the canvas change, the sync engine MUST propagate those changes to the corresponding source code. This section defines the normative rules for canvas-to-code sync. Every rule in this table is MUST-level unless explicitly stated otherwise.

| Canvas Change | Code Effect |
|---|---|
| New accepted class node | Generate a source file containing a class skeleton — class declaration, constructor or initialization structure, field declarations, method stubs, and a documentation block populated from the node's responsibility statement and content. The file path MUST follow the `ccoding.source` field if set, or be derived from the `ccoding.qualifiedName` using the active language binding's conventions. |
| Edit class fields or methods (inline) | Update the class definition in the source file. Added fields MUST be added to the class body. Removed fields MUST be removed from the class body (or marked deprecated if the implementation follows a conservative strategy). Changed type annotations MUST be reflected in code. |
| Edit method documentation or pseudo code | Update the documentation block (docstring, JSDoc, doc comment) in the corresponding code. The sync engine MUST NOT overwrite the method body — documentation and implementation are separate concerns. If the pseudo code changes significantly, implementations SHOULD flag the method body for agent re-implementation rather than modifying it directly. |
| New `inherits` edge (accepted) | Add an inheritance declaration to the source class. For example, `class Foo(Bar)` in Python, `class Foo extends Bar` in TypeScript. If the parent class is in a different module, an import statement MUST also be generated. |
| New `implements` edge (accepted) | Add an interface or protocol implementation declaration to the source class. For example, protocol conformance annotation in Python, `implements` clause in TypeScript. If the interface is in a different module, an import statement MUST also be generated. |
| New `composes` edge (accepted) | Add a field declaration on the source class with the composed type. The field name MUST be derived from the edge label following the parsing rules defined in [Data Model, Section 7](01-data-model.md). If the composed type is in a different module, an import statement MUST also be generated. |
| New `depends` edge (accepted) | Add an import statement in the source file of the dependent class. The import MUST reference the target class's module and name as determined by the `ccoding.qualifiedName` and `ccoding.source` fields and the active language binding's import conventions. |
| Delete accepted node from canvas | Mark the corresponding code as deprecated using the language's deprecation mechanism (e.g., `@deprecated` decorator, `/** @deprecated */` JSDoc tag, `#[deprecated]` attribute). Implementations MUST NOT auto-delete source files or class definitions. Deletion of code is a destructive action that requires explicit human confirmation outside of sync. |
| Delete accepted edge from canvas | Remove the corresponding code construct (inheritance declaration, import statement, field declaration). Edge deletion is less destructive than node deletion — it removes a relationship, not an entire definition — so sync MAY apply it directly. Implementations SHOULD still provide a preview or confirmation for `inherits` and `implements` edge deletions, which can have cascading effects. |
| Node with `status: "proposed"` | MUST NOT generate code. Ghost nodes are canvas-only. They exist as visual proposals and have zero effect on the codebase. The sync engine MUST skip all elements with `status: "proposed"` or `status: "rejected"`. |
| Node with `status: "stale"` | MUST NOT sync to code. Stale nodes are awaiting human resolution. The sync engine MUST NOT generate, update, or remove code based on stale elements. |

### 3.1 Detail Node Sync

Method and field detail nodes (connected to their parent class via `detail` edges) are synced as part of the parent class's code, not as independent files. When a detail node changes:

- The sync engine MUST locate the corresponding method or field within the parent class's source file using the detail node's `ccoding.qualifiedName`.
- Changes to the detail node's documentation block MUST update the method or field's documentation in code.
- Changes to the detail node's signature (method parameters, return type, field type) MUST update the corresponding signature in code.
- The `detail` edge itself produces no independent code construct — it is a structural relationship on the canvas that the sync engine uses to locate the parent class.

### 3.2 Package Node Sync

Package nodes (`ccoding.kind: "package"`) represent directories or module groupings. When a new package node is accepted:

- The sync engine SHOULD create the corresponding directory structure if it does not exist.
- The sync engine SHOULD create any required package initialization files (e.g., `__init__.py` in Python) as defined by the active language binding.
- Package nodes are organizational — their primary role is grouping, not code generation.

### 3.3 Test Node Sync (Canvas → Code)

Test nodes (`ccoding.kind: "test"`) are synced to code as test class skeletons. When a new test node is accepted:

- The sync engine MUST generate a test source file containing a test class skeleton — the test class declaration, test method stubs for each test method listed in the node's pseudo code, and import statements for the class(es) under test as determined by connected `tests` edges.
- The file path MUST follow the `ccoding.source` field if set, or be derived from the `ccoding.qualifiedName` using the active language binding's test file conventions (e.g., `tests/test_document.py` in Python, `__tests__/document.test.ts` in TypeScript).
- Each test method stub MUST include a documentation block populated from the corresponding pseudo code entry in the test node's content.
- The sync engine SHOULD generate appropriate test framework boilerplate as defined by the active language binding (e.g., `pytest` fixtures in Python, `describe`/`it` blocks in TypeScript).

When a test node's content changes on the canvas (test methods added, removed, or their pseudo code modified):

- The sync engine MUST add new test method stubs for added test methods.
- The sync engine MUST update documentation blocks for modified test methods.
- The sync engine MUST NOT overwrite existing test method bodies — the implementation logic within test methods is code's domain, just as it is for regular method bodies.

When a `tests` edge is added between a test node and a class node:

- The sync engine MUST add an import statement for the class under test in the test source file, if one does not already exist.

---

## 4. Code → Canvas Sync Rules

When source code changes, the sync engine MUST propagate those changes to the corresponding canvas nodes. Code-to-canvas sync ensures that the canvas remains an accurate architectural view even as the codebase evolves through direct editing, agent implementation, or refactoring.

| Code Change | Canvas Effect |
|---|---|
| New class in a tracked source file | Create a class node on the canvas with `status: "accepted"`. The node's `ccoding.qualifiedName` MUST be set from the class's fully qualified name. The node's `ccoding.source` MUST be set to the file path (relative to project root). The node's text content MUST be populated with the class name, responsibility (from the documentation block), fields, methods, and type annotations as defined by the active language binding's content conventions. |
| New method in an existing class | Update the parent class node's method list in its text content. If the method includes documentation with significant design content (responsibility statement, pseudo code, constraints), the sync engine SHOULD create a method detail node and connect it to the parent class via a `detail` edge. If the method is a simple accessor, utility, or has no significant documentation, adding it to the class node's inline method list is sufficient. |
| Changed documentation (docstring, JSDoc, doc comment) | Update the corresponding canvas node's text content. Documentation changes flow from code to canvas because the agent or human may have refined the documentation during implementation. The sync engine MUST update only the documentation sections of the node text, preserving any canvas-only content (such as manually added design notes within the node). |
| Changed type annotations or method signatures | Update the corresponding canvas node's fields section and method signatures. Changed parameter types, return types, and field types MUST be reflected on the canvas. If a method has a detail node, the detail node's content MUST also be updated. |
| New inheritance declaration | Create an `inherits` edge from the subclass node to the parent class node. If the parent class does not have a canvas node, the sync engine SHOULD create one (with `status: "accepted"` and content populated from the code). |
| New interface implementation | Create an `implements` edge from the implementing class node to the interface node. If the interface does not have a canvas node, the sync engine SHOULD create one. |
| New composition (typed field referencing another class) | Create a `composes` edge from the containing class to the composed type's class node. The edge label MUST include the field name following the conventions in [Data Model, Section 7](01-data-model.md). |
| New import of a tracked class | Create a `depends` edge from the importing class to the imported class. Implementations SHOULD only create `depends` edges for imports of classes that have canvas nodes — standard library imports and utility imports SHOULD NOT clutter the canvas. |
| Deleted class | Mark the corresponding canvas node as `stale` by setting `ccoding.status` to `"stale"`. Implementations MUST NOT auto-delete canvas nodes. The human decides whether to remove the node from the canvas, restore the code, or re-link the node to a replacement. |
| Deleted method | If the method has a detail node, mark it as `stale`. If the method was only listed inline in the class node, remove it from the class node's method list. Implementations SHOULD prefer marking as stale over silent removal when the method was architecturally significant (had a detail node or was an edge endpoint). |
| Renamed class or method | If the sync engine can detect the rename (e.g., through heuristics, git history, or language server analysis), it SHOULD update the canvas node's `ccoding.qualifiedName` and text content to reflect the new name. If the rename cannot be confidently detected, the sync engine MUST mark the old element as `stale` and create a new `accepted` element for the renamed construct. |

**Rename detection heuristics:** Implementations SHOULD use the following signals to detect renames:
- Same file, different class name (high confidence).
- Same qualified name prefix, different final segment (medium confidence).
- Matching field/method structure in a new class (low confidence).

When confidence is below a threshold defined by the implementation, the sync engine SHOULD mark the old node as `stale` and create a new node rather than assuming a rename. Implementations SHOULD allow the human to confirm or reject detected renames.

### 4.1 Scope of Code-to-Canvas Sync

Not every code change is architecturally significant. The sync engine MUST update the canvas for structural changes (new classes, changed signatures, changed documentation, changed relationships). The sync engine MUST NOT attempt to reflect implementation details on the canvas — local variable changes, control flow modifications, performance optimizations within method bodies, and other line-level code changes are the code's domain and have no canvas representation.

### 4.2 New Files and Untracked Code

When the sync engine encounters a source file that contains classes not present on the canvas:

- If the file is within the project's tracked directories, the sync engine SHOULD create canvas nodes for the new classes.
- Implementations MAY require explicit opt-in for tracking new directories (e.g., a configuration that specifies which source paths are synced).
- Newly created canvas nodes from code import MUST have `status: "accepted"` — the code exists and is authoritative.

### 4.3 Test Node Sync (Code → Canvas)

Test execution produces results — pass, fail, or error for each test method — that MUST be reflected on the canvas. This is a unique aspect of test nodes: they carry execution state in addition to design intent.

**Test result import.** When the sync engine detects updated test results (from test output files, CI artifacts, or framework-specific result formats as defined by the active language binding), it MUST update the test node's structured content to reflect the current results. Each test method in the node's content MUST show its most recent execution status: pass, fail (with failure message), or error (with error details). The format of the results section is defined by the active language binding.

**New test class in code.** When the sync engine encounters a test class in a tracked source file that has no corresponding canvas node, it SHOULD create a test node on the canvas with `status: "accepted"`. The sync engine SHOULD also create `tests` edges to any class nodes whose classes are imported and exercised by the test class. Detecting which classes a test exercises MAY rely on import analysis, naming conventions (e.g., `TestDocumentParser` tests `DocumentParser`), or explicit markers as defined by the active language binding.

**Changed test methods.** When test methods are added, removed, or modified in code, the sync engine MUST update the test node's content following the same rules as regular method sync (Section 4): new methods are added to the node's method list, removed methods are flagged or removed, and documentation changes propagate to the canvas.

**Staleness propagation.** When a class node connected via a `tests` edge changes its structural content (fields, methods, or signature changes — not just documentation), the sync engine SHOULD mark the connected test nodes as `stale` per the lifecycle rules defined in [Lifecycle, Section 4.5](02-lifecycle.md). This signals to the human that the test suite may need updating to reflect the class changes.

---

## 5. Abstract Sync Mapping

This section provides the complete mapping between canvas elements and code elements. It is the reference table that implementations use to determine what canvas construct corresponds to what code construct. The mapping is bidirectional — each row defines a correspondence that applies in both sync directions.

| Canvas Element | Code Element | Notes |
|---|---|---|
| Class node | Class or type definition + documentation block | The primary mapping. The class node's text content maps to the class declaration, its docstring, and its structural signature (fields and methods). |
| Method detail node | Method signature + documentation block | A method promoted to its own node. The detail node's content maps to the method's signature, parameters, return type, and documentation. The method body is code's domain and is NOT represented on the canvas. |
| Class fields section (inline) | Attribute declarations + type annotations | Fields listed within a class node's text content map to field declarations in the class body. Type annotations on the canvas map to type annotations in code. |
| Field detail node | Field declaration + documentation block | A field promoted to its own node. The detail node's content maps to the field's declaration, type annotation, default value documentation, and any associated documentation. |
| Edge `inherits` | Inheritance declaration | Maps to the language's inheritance syntax. One edge = one parent class in the inheritance list. |
| Edge `implements` | Interface or protocol implementation | Maps to the language's interface implementation syntax. One edge = one implemented interface. |
| Edge `composes` | Typed field declaration | The edge label provides the field name. The target node provides the field type. Maps to a field declaration on the source class. |
| Edge `depends` | Import statement | Maps to an import of the target class in the source class's file. |
| Test node | Test class definition + test method stubs + documentation blocks | A test node maps to a test class in a test source file. The node's test method pseudo code maps to test method stubs and documentation. The node's results section maps to the most recent test execution output. |
| Edge `tests` | Import statement + test class reference | Maps to an import of the class under test in the test source file. The sync engine also uses this edge to propagate staleness when the target class changes. |
| Edge `calls` | Not synced (informational) | Documents runtime method call flow on the canvas. The actual calls exist in method bodies, which are code's domain. The sync engine MUST NOT generate or remove code based on `calls` edges. |
| Edge `detail` | Not synced (structural) | A canvas-only structural relationship linking a class node to its promoted method or field node. The sync engine uses this edge to locate parent-child relationships but does not generate an independent code construct from it. |
| `status: "proposed"` | Not in code (ghost) | Ghost elements exist only on the canvas. They have zero code representation. |
| `status: "rejected"` | Not in code (decision record) | Rejected elements exist only on the canvas as decision history. |
| `status: "stale"` | Not in code (broken link) | Stale elements exist on the canvas but their code counterpart is missing or moved. |
| Context nodes | Not synced (canvas-only) | Context nodes are collaboration artifacts — design rationale, references, decision logs. They have no code representation. |
| Edge `context` | Not synced (canvas-only) | Context edges link context nodes to code-element nodes. They are canvas-only associations with no code effect. |

---

## 6. Conflict Detection and Resolution

Conflicts arise when both the canvas and the code change the same element between sync cycles. Without conflict detection, sync becomes a last-write-wins race that silently destroys work. This section defines the normative rules for detecting and resolving conflicts.

### 6.1 What Constitutes a Conflict

A conflict exists when all three of the following conditions are true:

1. An element has a canvas representation (a node or edge) AND a code representation (a class, method, field, or relationship).
2. The canvas representation has changed since the last successful sync.
3. The code representation has changed since the last successful sync.

If only one side changed, there is no conflict — the change propagates normally from the changed side to the unchanged side. A conflict requires both sides to have diverged from the last known common state.

### 6.2 Conflict Granularity

Conflicts MUST be detected at the element level, not the file level. A single source file may contain multiple classes, and a single canvas may contain dozens of nodes. If the human edits the documentation of `ClassA` on the canvas while the agent modifies the documentation of `ClassB` in the same source file, there is no conflict — the changes affect different elements.

Element-level granularity means:

- **Node-level:** Each canvas node / code element pair is an independent unit for conflict detection. A change to one node's documentation does not conflict with a change to a different node in the same file.
- **Edge-level:** Each relationship (edge / code construct pair) is an independent unit. Adding a new `composes` edge does not conflict with modifying an existing `inherits` edge on the same class, even though both affect the same source file.
- **Section-level (RECOMMENDED):** Within a single node, implementations SHOULD distinguish between documentation changes and signature changes. If the human edits a method's documentation on the canvas while the agent changes the method's parameter types in code, implementations SHOULD detect this as a conflict but MAY resolve it automatically if the changes affect different sections (documentation vs. signature).

### 6.3 Conflict Rules

The following rules are normative:

1. Implementations MUST detect conflicts as defined in Section 6.1.
2. Implementations MUST NOT silently overwrite either side when a conflict is detected. A conflict always requires human awareness.
3. Implementations SHOULD present both versions (canvas version and code version) to the human for resolution.
4. Implementations SHOULD identify the specific element and section in conflict, not just flag the entire file or node.
5. Implementations MAY offer automatic resolution strategies (e.g., "keep canvas version", "keep code version", "merge both") but MUST NOT apply any strategy without explicit human selection.

**Convergent edge creation:** If both the canvas and code independently create the same relationship (e.g., human adds a `composes` edge on canvas while the agent adds a typed field in code), and both resolve to the same semantic relationship, this is NOT a conflict. The sync engine SHOULD recognize convergent edits and merge them without user intervention.

### 6.4 Resolution Strategies

The spec does not mandate a specific resolution mechanism, but RECOMMENDS the following strategies that implementations SHOULD support:

**Three-way merge.** Using the sync state (Section 7) as the common ancestor, compute a three-way diff between the ancestor, the canvas version, and the code version. If the changes are in non-overlapping regions (e.g., canvas changed the responsibility statement while code changed a type annotation), merge automatically. If the changes overlap, present the conflict to the human.

**Side selection.** Present both versions and let the human choose: "keep canvas" (overwrite code with canvas state), "keep code" (overwrite canvas with code state), or "keep both" (attempt a manual merge).

**Defer.** Mark the element as conflicted and skip it during the current sync cycle. The conflict persists until the human resolves it. Deferred conflicts MUST be visible — implementations SHOULD display a conflict indicator on the canvas node and SHOULD prevent further sync of the conflicted element until resolution.

### 6.5 Post-Resolution

After a conflict is resolved:

- The sync state (Section 7) MUST be updated to reflect the resolved state as the new common ancestor.
- The resolved element MUST be consistent on both sides — canvas and code MUST agree on the element's current state.
- If the resolution involved choosing one side over the other, the losing side MUST be updated to match the winner.

---

## 7. State Tracking

To detect what changed between syncs, implementations need a record of what the canvas and code looked like at the last successful sync. This record is the sync state.

### 7.1 Requirements

1. Implementations MUST track sync state to support change detection and conflict identification. Without sync state, the sync engine cannot distinguish between "this element is new" and "this element existed before but I have no record of it."
2. Sync state MUST capture enough information about each synced element to detect changes on both sides. At minimum, this means recording the content (or a content-derived fingerprint) of each canvas node's text and metadata, and the corresponding code construct's content.
3. Sync state SHOULD persist across sessions. If the user closes the canvas tool and reopens it later, the sync engine SHOULD be able to resume from the last known state without re-syncing everything from scratch.

### 7.2 Recommended Approach

The spec RECOMMENDS content hashing as the state tracking mechanism:

- For each synced element, compute a hash of the canvas representation (node text + relevant `ccoding` metadata) and a hash of the code representation (the corresponding code block — class definition, method signature + documentation, field declaration).
- Store both hashes in the sync state, keyed by the element's `ccoding.qualifiedName`.
- On the next sync cycle, recompute both hashes and compare:
  - Canvas hash changed, code hash unchanged → canvas changed, propagate to code.
  - Code hash changed, canvas hash unchanged → code changed, propagate to canvas.
  - Both hashes changed → conflict. Apply the rules from Section 6.
  - Neither hash changed → no change. Skip this element.

This approach is simple, efficient, and deterministic. Content hashing avoids the complexity of tracking individual edit operations and works regardless of how the changes were made (manual editing, agent generation, external tooling, git operations).

**Incremental sync (RECOMMENDED):** Full-canvas scans on every sync cycle can be expensive for large projects. Implementations SHOULD use file-system modification timestamps to pre-filter which source files need re-hashing. Canvas tools SHOULD track which nodes were modified since the last save to avoid re-hashing unchanged nodes.

### 7.3 Storage

The spec does NOT prescribe a specific state format or storage location. Implementations choose their own. RECOMMENDED conventions:

- **Location:** A project-relative path such as `.ccoding/sync-state.json` or `.ccoding/sync/` directory. Placing it in a `.ccoding` directory keeps CooperativeCoding artifacts together and makes them easy to gitignore if desired.
- **Format:** JSON, YAML, SQLite, or any format the implementation prefers. The state is an implementation detail, not an interoperability surface — different tools do not need to share sync state.
- **Version control:** Implementations SHOULD recommend that users add the sync state to `.gitignore`. The sync state is a local cache, not a shared artifact. Each developer's sync state reflects their local canvas and code state.

### 7.4 Initial Sync

When no prior sync state exists (first sync, new project, or state file deleted):

- All canvas elements with `status: "accepted"` and all code elements in tracked directories are treated as "new."
- The sync engine MUST NOT treat the absence of prior state as a conflict. Instead, it SHOULD perform a reconciliation pass: match canvas nodes to code elements by `ccoding.qualifiedName`, populate the sync state with the current hashes, and flag any unmatched elements for human attention.
- After the initial reconciliation, the sync state is established and subsequent sync cycles operate normally.

---

## 8. Sync Triggers

Sync does not happen continuously — it runs at specific moments determined by the implementation. This section defines the trigger categories and their requirements.

### 8.1 Manual Sync

Implementations MUST support manual sync — an explicit command that the user invokes to trigger a sync cycle. Examples:

- A CLI command: `ccoding sync`
- A canvas tool command: a button, menu item, or keyboard shortcut
- An agent command: "sync canvas and code"

Manual sync is the baseline. Every implementation MUST provide it. It gives the user full control over when sync runs and ensures that sync never happens unexpectedly.

### 8.2 Automatic Sync

Implementations MAY support automatic sync — sync triggered by events without explicit user invocation. Examples:

- **File watcher:** Detect when a source file is saved and trigger code-to-canvas sync.
- **Canvas save hook:** Detect when the canvas file is saved and trigger canvas-to-code sync.
- **IDE event:** Hook into the IDE's file change notifications.
- **Periodic polling:** Check for changes on a timer.

Automatic sync is OPTIONAL. Implementations that support it MUST provide a way to disable it — some users prefer manual control. Implementations that support automatic sync SHOULD also support manual sync, so users can force a sync cycle at any time.

### Sync Serialization

Implementations MUST serialize sync operations within a project. Concurrent sync cycles MUST NOT run simultaneously. Implementations SHOULD use a lock file (e.g., `.ccoding/sync.lock`) to enforce this.

If a lock file is stale (e.g., the process that created it crashed), implementations SHOULD check whether the owning process is still alive before breaking the lock. A lock older than 5 minutes with no active owning process MAY be considered stale.

### 8.3 Trigger Rules

Regardless of trigger type, the following rules apply to all sync cycles:

1. Implementations MUST NOT sync ghost (`proposed`) or `rejected` elements to code. This rule applies regardless of how sync is triggered.
2. Implementations MUST NOT sync stale elements to code until a human resolves the staleness.
3. Implementations SHOULD provide a dry-run or diff mode that previews what sync would change without applying the changes. This is especially valuable for manual sync, where the user wants to review before committing to the changes.
4. Implementations SHOULD log or display what sync changed after each cycle, so the user can verify that the result matches expectations.

---

## 9. Ordering and Atomicity

A sync cycle may involve many elements — new nodes, changed documentation, new edges, deleted classes. The order in which these changes are applied and the guarantees around partial failure matter for correctness and recoverability.

### 9.1 Atomicity

Sync operations SHOULD be atomic: either all changes for a sync cycle apply successfully, or none of them do. Atomicity prevents the system from ending up in an inconsistent state where some elements are synced and others are not.

Achieving full atomicity may not be practical in all implementations (especially those that write to multiple independent files). In such cases:

- Implementations SHOULD use a staging approach: compute all changes first, validate them, then apply them as a batch.
- If a sync cycle fails partway through, implementations SHOULD roll back to the previous state. At minimum, the sync state (Section 7) MUST NOT be updated for elements that were not successfully synced — this ensures that the next sync cycle will retry the failed elements.
- Implementations SHOULD report which elements failed and why, so the user can resolve the issue.

### 9.2 Dependency Ordering

Within a sync cycle, elements SHOULD be processed in dependency order to ensure that prerequisites exist before dependents are created. The RECOMMENDED ordering:

1. **Package nodes** — Create directory structures and package initialization files first.
2. **Class nodes without dependencies** — Base classes, interfaces, and standalone classes.
3. **Inheritance and implementation edges** — Establish parent-child relationships. Process parents before children.
4. **Class nodes with dependencies** — Classes that extend or implement other classes. These come after their parents and interfaces are in place.
5. **Composition and dependency edges** — Field declarations and import statements.
6. **Method and field detail nodes** — Promoted elements that belong to classes already synced.
7. **Deletions and deprecations** — Processed last, after all additions and updates, to avoid removing something that a new element depends on during the transition.

Implementations MAY use a different ordering strategy (e.g., topological sort on the dependency graph) as long as the result is the same: parent types exist before child types reference them, imported types exist before import statements reference them, and classes exist before their fields and methods are synced.

### 9.3 Idempotency

Sync operations SHOULD be idempotent: running the same sync cycle twice with no intervening changes SHOULD produce no additional modifications. This property ensures that automatic sync (Section 8.2) does not create spurious changes when triggered repeatedly, and that manual sync can be safely invoked as a "just make sure everything is aligned" operation.

---

## 10. Edge Cases

This section addresses specific scenarios that fall outside the main sync rules but that implementations will encounter in practice.

### 10.1 Circular Dependencies

If the canvas contains a circular dependency (A depends on B, B depends on C, C depends on A), the sync engine MUST handle it without entering an infinite loop. The dependency ordering in Section 9.2 may not produce a valid total order for circular graphs. Implementations SHOULD detect cycles, process the cycle's members in an arbitrary but stable order, and emit a warning to the user that a circular dependency exists.

### 10.2 Multiple Canvas Files

A project MAY have multiple `.canvas` files representing different architectural views (e.g., one for the core module, one for the plugin system, one for the API layer). If the same code element appears on multiple canvases:

- The sync engine MUST keep all canvas representations consistent. A change in code MUST update all canvas nodes that map to the same `ccoding.qualifiedName`.
- If different canvases show conflicting states for the same element, the sync engine MUST treat this as a conflict and present it for human resolution.
- Implementations SHOULD warn users when the same element appears on multiple canvases, as this increases the risk of conflicting edits.

### Version Control Integration

JSON Canvas files are JSON, which does not merge well with line-based version control tools. When two developers edit the same `.canvas` file and their changes conflict at the git merge level:

1. Implementations SHOULD resolve the git conflict manually or with a JSON-aware merge tool.
2. After resolving the git conflict, implementations SHOULD run a full sync cycle to reconcile any inconsistencies.
3. Implementations MAY provide a post-merge hook that automatically validates and re-syncs the canvas.
4. Teams SHOULD consider organizing canvases to minimize overlap — e.g., one canvas per subsystem or per developer area.

### 10.3 External Code Changes

Code may change outside of the canvas-agent workflow — a teammate pushes a commit, a merge brings in changes, or the user edits code directly in an editor without touching the canvas. The sync engine MUST handle these changes the same way it handles any code change: compare the current code against the sync state, detect changes, and propagate them to the canvas.

### 10.4 Canvas Without Code (Design-Only Elements)

An accepted class node on the canvas that has never been synced to code (e.g., the user just created it and hasn't run sync yet) is a valid state. The sync engine SHOULD treat it as a new element on the next sync cycle and generate the corresponding code. The absence of code for an accepted node is not a conflict or an error — it simply means the canvas is ahead of the code.

### 10.5 Code Without Canvas (Untracked Code)

Source files that exist in the project but have no corresponding canvas nodes are untracked code. The sync engine SHOULD NOT automatically create canvas nodes for all untracked code — this would flood the canvas with every utility function and configuration file in the project. Instead:

- Implementations SHOULD respect a tracking configuration that defines which source paths are synced.
- New classes within tracked paths SHOULD get canvas nodes (as defined in Section 4.2).
- Classes outside tracked paths SHOULD be ignored by sync unless the user explicitly imports them.

### Error Taxonomy

Sync operations MAY produce the following error categories. Implementations SHOULD use these categories for structured logging and user-facing messages:

| Category | Description | Example |
|---|---|---|
| `parse_error` | Source file has syntax errors preventing AST extraction | `SyntaxError in parsers/document.py line 42` |
| `conflict` | Both canvas and code changed the same element | `Conflict on DocumentParser: canvas and code both modified` |
| `missing_source` | `ccoding.source` references a file that does not exist | `File parsers/document.py not found` |
| `orphan` | A detail node has no parent class via `detail` edge | `Method parse() has no parent class node` |
| `schema_error` | Invalid `ccoding` metadata on a node or edge | `Node abc123: unknown status value 'draft'` |
| `path_violation` | `ccoding.source` resolves outside the project root | `Path ../../../etc/passwd escapes project root` |
| `lock_error` | Another sync process holds the lock | `Sync lock held by process 12345` |

### Structured Observability

Implementations SHOULD emit structured sync events for debugging and monitoring. Each sync cycle SHOULD produce a summary including:
- Cycle ID (unique identifier)
- Timestamp (start and end)
- Direction (canvas-to-code, code-to-canvas, or bidirectional)
- Elements processed (count by direction)
- Conflicts detected (count and element identifiers)
- Errors encountered (count by category)
- Duration (total and per-element breakdown for cycles exceeding a threshold)

Implementations MAY expose a read-only **diagnostic command** (e.g., `ccoding check`) that reports drift between canvas and code without modifying either side. This command SHOULD output the list of elements that would be synced, any detected conflicts, and any errors — without performing the sync.
