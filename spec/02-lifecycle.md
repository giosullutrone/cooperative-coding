# CooperativeCoding Specification — Lifecycle

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures — nodes, edges, metadata, statuses. This document defines the motion: how elements move between statuses, how humans and agents cooperate on the canvas, and how the ghost proposal workflow turns tentative ideas into accepted architecture.

A CooperativeCoding session is a continuous loop of design and implementation. The human draws architecture, the agent proposes improvements, the human reviews and decides, the agent implements what's accepted, and the sync engine keeps canvas and code aligned. Every element on the canvas has a lifecycle governed by a small, well-defined state machine. Every interaction between human and agent follows a cooperation protocol that preserves human design authority while giving the agent meaningful creative latitude.

This document is the authoritative reference for the lifecycle semantics. It defines entry points (how a session begins), the cooperation loop (how human and agent interact), the status state machine (how elements transition between statuses), ghost semantics (the properties and rules for proposals), and bulk operations (how multiple transitions happen at once).

---

## 2. Entry Points

CooperativeCoding supports three entry points into a design session. Each starts from a different place but converges on the same cooperation loop. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 2.1 From Blank Canvas — Human Sketches First

The most direct entry point. A human opens an empty canvas and begins drawing the architecture by hand.

1. **Human creates nodes.** The human adds class nodes, method detail nodes, field detail nodes, and edges directly on the canvas. Each node gets a documentation block — responsibility statement, fields, methods, pseudo code, constraints. Each edge gets a relation type and a descriptive label.

2. **Human fills in design intent.** The documentation blocks are the bridge to implementation. The human writes what each class is responsible for, what each method does in plain language or pseudo code, what constraints each field carries. This is the contract the agent will implement against.

3. **Agent implements.** When the human invokes the agent (explicitly or through an automation trigger), the agent reads the canvas state, interprets the documentation blocks, and generates source files. The sync engine establishes the mapping between canvas nodes and code constructs. From this point forward, the canvas and code are linked — changes in either propagate through sync.

All nodes created by the human in this workflow start with `status: "accepted"`. They are deliberate design decisions, not proposals. The agent treats them as authoritative from the moment they appear.

### 2.2 From Natural Language — Agent Proposes

The fastest way to bootstrap a design. A human describes a system in natural language, and the agent translates that description into a canvas layout.

1. **Human describes the system.** The human provides a text description — anything from a single sentence ("Build a plugin system with a registry, loader, and execution pipeline") to a detailed requirements document. The description lives outside the canvas: in a chat message, a context node, or an external document.

2. **Agent generates initial design.** The agent interprets the description and creates nodes and edges on the canvas. Every element the agent creates arrives as a ghost — `status: "proposed"`, `proposedBy: "agent"`, with a `proposalRationale` explaining why the agent chose this particular design. The agent MUST NOT create `accepted` nodes from natural language input. The initial design is always a proposal.

3. **Human reviews.** The human sees the proposed layout and makes decisions: accept nodes that match the intent, reject nodes that don't belong, modify ghosts before accepting them (editing documentation blocks, renaming classes, adjusting relationships). The human MAY also create new nodes by hand during review, which start as `accepted`.

4. **Agent implements accepted design.** Once the human has reviewed the proposals, the agent generates code for all `accepted` elements. The sync engine establishes the canvas-to-code mapping. The cooperation loop begins.

### 2.3 From Existing Code — Import

The entry point for legacy systems, ongoing projects, or any codebase that predates CooperativeCoding.

1. **Human points the tool at source files.** The human selects a directory, a set of files, or a module path. The tool may be a CLI command, a canvas tool action, or an agent instruction.

2. **Sync engine parses code.** The sync engine (or agent acting as the sync engine) reads the source files and extracts the architectural structure: classes, methods, fields, inheritance hierarchies, composition relationships, dependencies, and module organization. The extraction follows the active language binding's conventions.

3. **Canvas nodes are generated.** Each extracted code element becomes a node on the canvas with `status: "accepted"` — the code exists and is authoritative. Edges are created for detected relationships. Documentation blocks are populated from existing docstrings, comments, and type annotations. The sync mapping is established immediately.

4. **Human and agent iterate.** The human now has a visual representation of the existing architecture. They can rearrange nodes for clarity, add documentation where it's missing, flag elements for redesign, and ask the agent for analysis ("What's wrong with this design?", "Where are the circular dependencies?"). The agent can propose improvements as ghost nodes. The cooperation loop begins.

---

## 3. The Cooperation Loop

Once a session is active — regardless of which entry point started it — the human and agent interact through a continuous cooperation loop. The loop has no fixed sequence; either party can act at any time, and the sync engine keeps everything aligned.

```
Human edits canvas  ──→  Sync updates code
       ↑                        ↓
  Human accepts/          Agent reads code
  rejects proposals       + canvas state
       ↑                        ↓
Agent proposes      ←──  Agent identifies
ghost nodes               improvements
```

The loop is asynchronous and interleaved. The human might edit three nodes, accept two ghosts, and request agent input — all before the agent finishes analyzing the last change. Implementations MUST handle concurrent activity gracefully. The sync engine is the serialization point: all changes — human or agent — flow through sync to reach the other side.

### Concurrency

Implementations MUST serialize sync cycles — concurrent sync operations on the same project MUST NOT overlap. If a sync cycle is already in progress when a new one is triggered, the new cycle MUST be queued or skipped.

When a human is actively editing a canvas node, the sync engine SHOULD defer propagation of that node until the edit is complete (e.g., on save or after an idle timeout).

When an agent is writing code while a sync cycle is running, the sync engine SHOULD detect the in-progress write and either wait for it to complete or flag the affected elements for re-sync in the next cycle.

Implementations MUST use file-level or project-level locking (e.g., a `.ccoding/sync.lock` file) to prevent concurrent sync processes from corrupting state.

### 3.1 Human Actions

The human is the design authority. Every action the human takes on the canvas is authoritative and immediate.

**Create, edit, delete nodes and edges.** The human can add new nodes (which start as `accepted`), modify existing nodes (updating documentation blocks, renaming, changing stereotypes), and delete nodes (which removes them from the canvas and triggers sync to handle the code side). Edge operations follow the same pattern: create, relabel, retype, or delete.

**Edit documentation blocks, pseudo code, responsibilities.** The documentation block is the contract. When the human updates a method's pseudo code or a class's responsibility statement, the sync engine propagates the change to code. The agent reads the updated contract and adjusts its implementation accordingly.

**Accept a ghost.** The human transitions a `proposed` element to `accepted`. This is the fundamental approval action — it tells the agent and the sync engine: "This is now part of the design. Generate the code." Accepting a ghost marks it for code generation in the next sync cycle. Implementations MAY trigger sync immediately upon acceptance, or MAY batch accepted nodes and sync on the next manual or automatic trigger. The lifecycle requirement is that acceptance eventually results in code generation — the timing is implementation-defined.

**Reject a ghost.** The human transitions a `proposed` element to `rejected`. The element is excluded from the active design. Implementations SHOULD hide or visually suppress rejected elements. The sync engine MUST NOT generate code for rejected elements.

**Request agent input.** The human asks the agent to analyze, propose, or implement. Examples: "What's missing from this design?", "Implement this class", "Split this into two classes", "Add error handling". The request may be explicit (a command or chat message) or implicit (a canvas annotation or trigger).

### Agent Communication Protocol

The spec does not mandate a specific communication mechanism between agents and the canvas. Valid approaches include:
- **Direct file I/O:** The agent reads and writes the `.canvas` JSON file directly, adding ghost nodes and edges as JSON objects in the `nodes` and `edges` arrays.
- **Canvas tool API:** The canvas tool exposes an API (HTTP, IPC, or plugin SDK) that the agent calls to propose changes.
- **CLI mediation:** A CLI tool (e.g., `ccoding propose`) accepts structured input and modifies the canvas file.
- **Prompt-based:** The agent outputs structured JSON describing proposals, and a harness applies them to the canvas.

All approaches MUST result in valid JSON Canvas files with correct `ccoding` metadata. The agent integration layer MUST ensure that proposals are created as `proposed` status — agents MUST NOT create nodes with `accepted` status directly.

### 3.2 Agent Actions

The agent operates within the bounds set by the cooperation protocol. It has creative latitude to propose and analyze, but it cannot unilaterally change the accepted design.

**Propose.** The agent creates ghost nodes and edges on the canvas with `status: "proposed"`, `proposedBy: "agent"`, and a `proposalRationale` explaining the reasoning. Proposals are the agent's primary design contribution. Agents MUST NOT modify accepted nodes without human approval. If the agent believes an accepted element should change, it MUST express this as a proposal — either a new ghost that would replace the existing element, or a context node explaining the suggested change.

**Implement.** The agent generates or updates source code from accepted canvas nodes. Implementation reads the documentation blocks, interprets the design intent, and produces code that fulfills the contract. The agent SHOULD generate code that matches the structure expressed on the canvas — the class hierarchy, the method signatures, the composition relationships. Implementation is triggered by acceptance of ghosts, by explicit human request, or by sync detecting that canvas changes need code updates.

**Sync.** The agent (or a dedicated sync engine) detects changes on either side and propagates them. When code changes, the corresponding canvas nodes update. When canvas nodes change, the corresponding code updates. Sync is covered in detail in [Sync](03-sync.md) — this document defines only the lifecycle transitions that sync triggers (specifically, the `accepted → stale` and `stale → accepted` transitions).

**Analyze.** The agent examines the canvas and code for design issues and reports findings. Analysis MAY be triggered by human request or run automatically. Common analysis targets include: circular dependencies, missing interfaces, Single Responsibility Principle violations, unused abstractions, incomplete documentation blocks, and inconsistencies between canvas and code. Analysis results SHOULD be expressed as context nodes (connected via `context` edges) or as ghost proposals that address the identified issues.

---

## 4. Status State Machine

Every CooperativeCoding element — node or edge — carries a `status` field that governs its lifecycle. The status state machine defines exactly four states and five transitions. This section is the authoritative, normative definition.

### 4.1 State Diagram

```
proposed  ──accept──→  accepted  ──(code deleted)──→  stale
    │                                                    │
    └──reject──→  rejected                    (code restored)
                     │                                   │
                     └──reconsider──→  proposed    accepted ←─┘
```

### 4.2 States

| State | Meaning |
|---|---|
| `proposed` | A ghost — a tentative design element awaiting human review. Not synced to code. |
| `accepted` | Part of the canonical design. Synced to code. |
| `rejected` | A declined proposal. Not synced to code. Preserved for decision history. |
| `stale` | An accepted element whose corresponding code was deleted or moved. Awaiting human resolution. |

These four values — `proposed`, `accepted`, `rejected`, `stale` — are the ONLY valid values for `ccoding.status`. Implementations MUST reject or ignore any other value. Implementations MUST NOT invent additional statuses.

### 4.3 Transitions

| Transition | Trigger | Who | Effect |
|---|---|---|---|
| `proposed → accepted` | Human accepts the proposal | Human | The element becomes part of the canonical design. The sync engine MUST generate or update the corresponding code. The `proposalRationale` SHOULD be cleared (set to `null`). Visual styling MUST change from ghost to full-weight. |
| `proposed → rejected` | Human rejects the proposal | Human | The element is excluded from the active design. Implementations SHOULD hide or grey out the element. The sync engine MUST NOT generate code for it. The `proposalRationale` SHOULD be preserved so the decision context remains. |
| `rejected → proposed` | Human reconsiders | Human | The element returns to the proposal state for re-review. The `proposedBy` field MUST remain unchanged (it records who originally proposed the element). The human MAY modify the element before or after reconsidering. |
| `accepted → stale` | Sync detects code deletion or unresolvable move/rename | Sync engine | The canvas node remains but its link to the codebase is broken. A visual indicator MUST appear on the canvas (e.g., warning icon, muted color, strikethrough). The implementation MUST NOT auto-delete the stale node — the human decides what to do. |
| `stale → accepted` | Sync detects code restoration, or human manually re-links | Human or Sync | The element becomes active again. Sync resumes normally. The visual indicator is removed. |

When a node transitions to `stale`, implementations MUST set `ccoding.staleReason` to one of:
- `"code-deleted"` — the corresponding source code was deleted, moved, or renamed beyond recognition.
- `"code-renamed"` — the sync engine detected a likely rename but could not auto-resolve it.
- `"dependency-changed"` — a node connected via a `tests` edge had structural changes (staleness propagation).

This field allows canvas tools to display context-specific guidance (e.g., 'Source file missing' vs. 'Test may be outdated').

### Deletion Cascade

When a human deletes an accepted class node from the canvas:
1. All `detail` edges from that node are also removed.
2. Detail nodes (methods, fields) that were connected ONLY via a `detail` edge to the deleted class become `stale` with `staleReason: "code-deleted"`. They are NOT automatically deleted — the human decides.
3. Incoming edges from other nodes (e.g., `inherits`, `composes`) are removed. The source nodes of those edges are NOT affected — they retain their status.
4. The corresponding source code is marked as deprecated (see §03-sync, Section 3). Implementations MUST NOT delete source files without explicit human confirmation.
5. These operations SHOULD be treated as a single atomic transaction by the sync engine.

### 4.4 Transition Rules

All implementations MUST support all five transitions defined above. The following rules are normative:

1. **No direct `accepted → rejected` transition.** If a human wants to remove an accepted element from the design, they delete the node from the canvas. Deletion is a canvas operation, not a status transition. The sync engine handles the code side (removing or marking the corresponding code). The rationale: `rejected` means "this was proposed and I said no" — it is a decision about a proposal, not a retroactive rejection of something that was already part of the design.

2. **No direct `rejected → accepted` transition.** A rejected element must pass through `proposed` first via the `reconsider` transition. This ensures that every acceptance is a conscious review of the current state of the element, not an accidental undo. The human reconsiders, reviews the element (possibly modifying it), and then explicitly accepts.

3. **Only humans trigger `accept`, `reject`, and `reconsider`.** Agents MUST NOT transition elements to `accepted`, `rejected`, or back to `proposed` from `rejected`. These transitions require human judgment and represent design authority. The agent's role is to propose and implement, not to approve or dismiss.

4. **Only the sync engine triggers `accepted → stale`.** This transition is automated — it happens when the sync engine detects that the corresponding code no longer exists or has moved in a way sync cannot resolve. Humans and agents MUST NOT manually set `status` to `stale`.

5. **`stale → accepted` may be triggered by sync or human.** If the sync engine detects that the code has been restored (the file and qualified name match again), it SHOULD automatically transition the element back to `accepted`. Alternatively, the human may manually re-link the node to a new code location, which also transitions it to `accepted`.

### 4.5 Test Node Lifecycle

Test nodes (`ccoding.kind: "test"`) follow the same four-state state machine as all other code-element nodes, but they carry additional lifecycle semantics tied to the classes they verify and the results of test execution.

**Staleness propagation from class changes.** When an accepted class node changes — its fields, methods, documentation, or relationships are modified on the canvas or in code — all test nodes connected to that class via `tests` edges SHOULD transition to `stale`. The rationale: a change to the class under test means the test suite may no longer accurately verify the class's behavior. The test's pseudo code, assertions, and expected outcomes may reference methods, signatures, or behaviors that have changed. Marking the test node as stale signals to the human that the test suite needs review and possible update.

Implementations SHOULD trigger this staleness propagation automatically during sync. The sync engine detects that a class node's content has changed, follows the `tests` edges to find connected test nodes, and sets their `ccoding.status` to `"stale"`. This is an extension of the `accepted → stale` transition from Section 4.3 — the trigger is a change in a related element rather than deletion of the element's own code.

Staleness propagation via `tests` edges is **single-hop only**: when a class changes, only test nodes directly connected to that class via a `tests` edge become stale. Staleness does NOT propagate transitively — if TestA tests ClassA and ClassB, and ClassA changes, only TestA becomes stale. ClassB is not affected.

**Test result updates.** Test nodes carry execution results in their structured content — pass, fail, or error status for each test method, along with failure details. When test results change (because the test suite was re-executed), the test node's content MUST be updated to reflect the new results. This update follows the standard code-to-canvas sync path: the sync engine reads the test results from the code side (test output files, CI artifacts, or framework-specific result formats as defined by the active language binding) and updates the test node's text content on the canvas.

A test node whose results change from passing to failing does NOT transition to `stale` — the node's link to its code is intact, and the results accurately reflect the current state. Staleness is reserved for structural disconnection (class changes that invalidate the test's design), not for test failures. Test failures are a content update, not a status transition.

**Restoration from staleness.** A stale test node transitions back to `accepted` when the human updates the test node's content to align with the changed class (updating pseudo code, adjusting assertions, adding or removing test methods) and the sync engine confirms that the test code matches the canvas representation. Alternatively, if the class change is reverted and the original class structure is restored, the test node MAY be automatically transitioned back to `accepted` by the sync engine.

---

## 5. Ghost Semantics

A ghost is any node or edge with `status: "proposed"`. Ghosts are the mechanism through which agents (and humans) express tentative design ideas without committing them to the codebase. This section defines the complete semantics of ghost elements.

### 5.1 Identity

A ghost is identified solely by its `status` field. Any node with `ccoding.status` set to `"proposed"` is a ghost, regardless of its `kind`, `stereotype`, or other metadata. Any edge with `ccoding.status` set to `"proposed"` is a ghost edge.

### 5.2 Required Metadata

When `status` is `"proposed"`, the following metadata fields carry specific requirements:

- **`proposedBy`** — MUST be set to `"agent"` or `"human"`. This field records the origin of the proposal. It MUST NOT be `null` or omitted when `status` is `"proposed"`.
- **`proposalRationale`** — SHOULD be present when `proposedBy` is `"agent"`. Agents SHOULD always explain why they are proposing an element. When `proposedBy` is `"human"`, the rationale is OPTIONAL — humans may or may not want to record their reasoning.

### 5.3 Lifecycle Behavior

- A ghost MUST NOT cause any code generation or modification. The sync engine MUST ignore `proposed` elements entirely.
- A ghost MUST be visually distinguished from accepted elements on the canvas. Implementations SHOULD use reduced opacity, dashed borders, distinct coloring, or other clearly differentiated styling.
- A human MAY modify a ghost before accepting it. Edits to a ghost's documentation block, name, stereotype, or relationships do not change its status — it remains `proposed` until explicitly accepted.
- A human MAY modify a ghost after rejecting it (transitioning back to `proposed` via `reconsider`, editing, then accepting or rejecting again).

### 5.4 Multi-Element Proposals

Multiple ghosts can be proposed simultaneously as part of a single design suggestion. For example, the agent might propose "split this class into two" by creating:

- Two new class ghost nodes (each with `status: "proposed"`)
- Edges between the new ghosts (also `status: "proposed"`)
- Edges from existing accepted nodes to the new ghosts (the edges are `proposed` even though one endpoint is `accepted`)

There is no formal "proposal group" concept in the data model — each ghost is an independent element with its own status. However, agents SHOULD use the `proposalRationale` field to link related ghosts conceptually (e.g., "Part of the split of OrderProcessor into OrderValidator and OrderExecutor"). Implementations MAY provide UI affordances for grouping related proposals, but this is not required by the spec.

### 5.5 Acceptance and Rejection Effects on Metadata

When a ghost is **accepted** (`proposed → accepted`):
- `status` changes to `"accepted"`.
- `proposalRationale` SHOULD be cleared (set to `null`). The rationale served its purpose during review and is no longer needed as active metadata. Implementations MAY preserve it in a separate audit log.
- `proposedBy` MAY be preserved after acceptance for audit trail purposes, or MAY be cleared (set to `null`). This is an implementation choice. The spec does not mandate either behavior.

When a ghost is **rejected** (`proposed → rejected`):
- `status` changes to `"rejected"`.
- `proposalRationale` SHOULD be preserved. The rationale explains why the element was proposed, which provides context for why it was rejected.
- `proposedBy` MUST be preserved. The origin of the proposal remains relevant for decision history.

---

## 6. Bulk Operations

Design reviews often involve many ghosts at once — an agent might propose an entire subsystem as a set of ghost nodes and edges. Reviewing each element individually is tedious when the human wants to approve or dismiss the whole batch.

### 6.1 Operations

**`accept-all`** — Accept all elements with `status: "proposed"` at once, transitioning each to `accepted`. Implementations SHOULD support this operation.

**`reject-all`** — Reject all elements with `status: "proposed"` at once, transitioning each to `rejected`. Implementations SHOULD support this operation.

### 6.2 Rules

Bulk operations MUST apply the same transition rules as individual operations. Specifically:

- Each element in the bulk operation undergoes the same state transition it would undergo individually. There are no shortcuts or special-case semantics for bulk transitions.
- The `proposalRationale` SHOULD be cleared on each accepted element, just as it would be for individual acceptance.
- The sync engine MUST generate code for all newly accepted elements, just as it would for individual acceptance. Implementations MAY batch the code generation for efficiency, but the result MUST be identical to accepting each element sequentially.
- Bulk operations apply only to `proposed` elements. They do not affect `accepted`, `rejected`, or `stale` elements.

### 6.3 Selective Bulk Operations

Implementations MAY support selective bulk operations — accepting or rejecting a subset of proposed elements (e.g., "accept all proposed classes but not the proposed edges", or "accept all proposals from this agent session"). These are implementation conveniences that follow the same transition rules. The spec does not define a specific selection mechanism.
