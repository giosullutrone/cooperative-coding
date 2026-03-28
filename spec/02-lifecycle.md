# CooperativeCoding Specification: Lifecycle

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures: nodes, edges, metadata. This document defines the motion: how the canvas and code stay in sync, how the agent processes requests, and how the human maintains visibility and control over the system's architecture.

CooperativeCoding uses a continuous bidirectional sync model. Every change on either side (canvas or code) produces a sync request for the other side, unless sync determines they already match. The agent sits in the middle, processing sync requests in both directions. The human has full visibility of every change through the canvas and can intervene at any time by editing the canvas or the code.

There are no phases, no proposal workflows, and no status transitions. The canvas and code are always in sync. The human's authority comes from being able to edit both the canvas and the code at any time. Version control provides review, audit, and rollback.

---

## 2. Entry Points

CooperativeCoding supports three entry points. Each starts from a different place but converges on the same continuous sync loop. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 2.1 From Blank Canvas

The human opens an empty canvas and begins drawing the architecture by hand: class nodes, method nodes, field nodes, edges, and node content (responsibilities, pseudo code, signatures, constraints). Each canvas change produces a sync request. The agent implements the code to match.

### 2.2 From Natural Language

The human describes a system in natural language. The agent creates nodes and edges on the canvas. These canvas changes produce sync requests, and the agent implements the code. The human sees the architecture appear on the canvas and can edit, restructure, or remove anything at any time.

### 2.3 From Existing Code

The human points the tool at source files. The code is parsed and canvas nodes are generated to represent the existing architecture. From this point, the canvas and code are linked and the sync loop begins.

---

## 3. The Sync Loop

Once a session is active, regardless of which entry point started it, the system operates as a continuous sync loop:

```
    Canvas change ──> code request ──> agent updates code
                                              |
                                        agent changed code?
                                              |
                                    no: done
                                    yes: design request ──> agent updates canvas
                                                                  |
                                                            agent changed canvas?
                                                                  |
                                                        no: done
                                                        yes: code request ──> ... (loop continues)
```

Every change on either side produces a sync request. The agent processes it and may produce changes on the other side, which in turn produce new sync requests. The loop continues until the agent determines no further changes are needed. Changes made by sync itself that don't alter the target side MUST NOT produce new sync requests (see [Sync, Section 4](03-sync.md)).

### 3.1 What Produces Sync Requests

**From the canvas (code requests):** Any canvas change by the human or the agent produces a request to update the code. Examples:
- Creating a new class, method, or field node
- Editing node content (responsibility, pseudo code, signatures, constraints)
- Adding or removing edges (inheritance, composition, dependencies)
- Deleting a node or edge

**From the code (design requests):** Any code change by the human, the agent, or external sources produces a request to update the canvas. Examples:
- Implementing a new class, method, or field
- Changing a method signature or type annotation
- Adding or removing inheritance, imports, or typed fields
- Refactoring, renaming, or deleting code elements
- External changes (pulls, merges, teammate commits)

**From the user (user requests):** The human asks the agent to do something explicitly. Examples:
- "Fix this crash"
- "Add error handling to this method"
- "Write tests for UserService"
- "Refactor this class into two"
- "Help me write the pseudo code for this method"

The agent makes changes on whichever side makes sense, and those changes flow through the normal sync loop.

### 3.2 The Agent's Role

The agent processes sync requests and user requests. It can:

- **Make design changes.** The agent can create, edit, or delete nodes and edges on the canvas. These changes produce code requests, which the agent processes by updating the code to match.
- **Make code changes.** The agent can implement, refactor, fix, or test code. These changes produce design requests, which the agent processes by updating the canvas to match.
- **Loop autonomously.** Design changes trigger code updates, which may trigger canvas updates, which may trigger further code updates. The agent continues processing sync requests until design and code stabilize (no further changes needed). Each iteration is visible on the canvas, and the human can intervene at any point.
- **Respond to user requests.** The human can ask the agent to do anything. The agent decides whether to change code, canvas, or both, and the sync loop handles the rest.

### 3.3 The Human's Role

The human maintains design authority through direct action:

- **Edit the canvas at any time.** Canvas changes produce code requests. The agent updates the code to match. If the agent made a canvas change the human disagrees with, the human edits it, and the sync loop propagates the correction.
- **Edit code at any time.** Code changes produce design requests. The agent updates the canvas to match. The human sees the architectural impact of every code change.
- **Review via version control.** The change history provides a full audit trail of all canvas and code changes, who made them, and when. The human can review diffs, revert changes, and use branches for experimental work.
- **Request agent assistance.** The human can ask the agent for help with anything: writing pseudo code, analyzing the design, fixing bugs, writing tests, refactoring.

### 3.4 Agent Communication Protocol

The spec does not mandate a specific communication mechanism between agents and the canvas. Valid approaches include:
- **Direct file I/O:** The agent reads and writes the `.canvas` JSON file directly.
- **Canvas tool API:** The canvas tool exposes an API (HTTP, IPC, or plugin SDK) that the agent calls.
- **CLI mediation:** A CLI tool accepts structured input and modifies the canvas file.
- **Prompt-based:** The agent outputs structured JSON describing changes, and a harness applies them to the canvas.

All approaches MUST result in valid JSON Canvas files with correct `ccoding` metadata.

---

## 4. Multi-User Environments

The core spec uses "the human" for simplicity. In teams, multiple humans share the canvas. The sync model does not change: every change on either side produces a sync request for the other side, regardless of who made it.

Implementations targeting multi-user environments MAY extend the `ccoding` metadata to carry specific user identity for audit and attribution purposes. The spec does not prescribe a user identity format.

Implementations MAY restrict which users can edit which nodes or which parts of the codebase. The spec does not define a permission model. Role-based access, approval workflows, and team boundaries are organizational concerns that belong to implementations.

For multi-team projects, multiple canvas files (see [Sync, Section 7](03-sync.md)) provide a natural boundary: each team owns its canvas, cross-team dependencies are visible as edges or context nodes.
