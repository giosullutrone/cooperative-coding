# CooperativeCoding Specification: Sync

**Version 2.0.0**

---

## 1. Overview

The [Design Contract](01-design-contract.md) defines what every code element must carry. This document defines the sync rules: what happens when the canvas changes, what happens when code changes, and how the two sides stay aligned.

Sync is bidirectional. Canvas changes update code. Code changes update the canvas. Every change on either side always produces a sync request for the other side. Each request is evaluated and the necessary changes are applied, which may be none if the other side already matches. When a sync request produces no changes, no further sync requests are generated, and the loop stabilizes.

The rules in this document are intentionally abstract. They define *what* must happen, not *how* an implementation achieves it.

---

## 2. The Sync Loop

Once a session is active, regardless of which entry point started it, the system operates as a continuous sync loop:

```
    Canvas change ──> code request ──> code updated
                                            |
                                      code changed?
                                            |
                                  no: done
                                  yes: design request ──> canvas updated
                                                                |
                                                          canvas changed?
                                                                |
                                                      no: done
                                                      yes: code request ──> ... (loop continues)
```

Every change on either side produces a sync request. Processing a request may produce changes on the other side, which in turn produce new sync requests. The loop continues until no further changes are needed. Changes made by sync itself that don't alter the target side MUST NOT produce new sync requests.

### 2.1 What Produces Sync Requests

**From the canvas (code requests):** Any canvas change by the human or the agent produces a request to update the code. Examples:
- Creating a new code element
- Editing design content (responsibility, pseudo code, signatures, constraints)
- Adding or removing relationships (inheritance, composition, dependencies)
- Deleting an element or relationship

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

**From the agent (agent proposals):** The agent may proactively propose or make changes based on its own analysis. Examples:
- Suggesting a design improvement after detecting unclear responsibilities
- Proposing to split a class that has grown too large
- Flagging inconsistencies between canvas and code

Agent-initiated changes flow through the same sync loop. The human retains design authority and can accept, modify, or reject any agent proposal by editing the canvas or the code.

---

## 3. Canvas to Code

When the canvas changes, a code request is produced. The code request is evaluated and the corresponding source code is updated to match.

The general rule: every canvas change produces a code request. Each element maps to a code construct, and each relationship maps to language-specific code, as defined by the active language binding. The human, agent, or sync engine decides what code changes are needed.

Sync MUST NOT overwrite method bodies. Design content maps to documentation and signatures in code; the implementation inside method bodies is the code's domain.

### 3.1 Change Propagation

When code is updated from a code request, the full impact of the change MUST be considered. For example, if a method signature changes, code that calls that method MUST also be updated. If the ripple effects require architectural changes, the canvas is updated, which produces further code requests through the normal sync loop.

---

## 4. Code to Canvas

When source code changes, a design request is produced. The design request is evaluated and the corresponding canvas elements are updated to match.

The general rule: every code change produces a design request. The human, agent, or sync engine decides what canvas changes are needed, which may be none for changes that are purely implementation details (local variables, control flow, performance optimizations within method bodies).

Sync MUST NOT overwrite canvas content that has no corresponding code representation (e.g., manually added design notes within an element).

### 4.1 Test Results

Test results are runtime data, not architectural changes. Implementations MAY choose to surface test results on the canvas. When they do, the format and presentation are defined by the active language binding and the canvas implementation.

---

## 5. Convergence

The sync loop stabilizes because each sync request brings the two sides closer to alignment. When a code request updates the code to match the canvas, the resulting design request finds the canvas already matches the code, and no further changes are needed. The loop terminates.

Implementations MUST ensure that the sync loop converges. If each sync response always brings the two sides closer to agreement (rather than introducing new divergence), the loop will terminate. An implementation where sync oscillates between two states is a bug.

---

## 6. State Tracking

Implementations SHOULD maintain a mapping between canvas elements and code elements so that sync requests can be evaluated. The element's identity (see [Design Contract, Section 2](01-design-contract.md)) is the recommended key for this mapping. Implementations MAY use version control, content hashing, file watchers, stateless diffing, or any other mechanism for detecting changes and tracking state.

When no prior mapping exists (first sync or new project), sync SHOULD perform a reconciliation pass: match canvas elements to code elements by identity and flag any unmatched elements for human attention.

---

## 7. Sync Triggers

Sync is continuous: every change on either side produces a sync request. Implementations define the mechanism for detecting changes (file watchers, save hooks, version control hooks, polling, etc.). Implementations SHOULD also support a manual sync command for cases where automatic detection misses a change or the user wants to force a full reconciliation.

---

## 8. Multiple Canvases

A project MAY have multiple canvases representing different architectural views. If the same code element appears on multiple canvases, sync MUST keep all canvas representations consistent: a code change MUST update all canvas elements that map to the same identity. Implementations SHOULD warn users when the same element appears on multiple canvases.

---

## 9. Entry Points

CooperativeCoding supports three entry points. Each starts from a different place but converges on the same continuous sync loop. Implementations SHOULD support all three entry points, though MAY prioritize based on their primary use case.

### 9.1 From Blank Canvas

The human opens an empty canvas and begins creating the architecture by hand: code elements, relationships, and design content (responsibilities, pseudo code, signatures, constraints). Each canvas change produces a sync request, and the code is updated to match.

### 9.2 From Natural Language

The human describes a system in natural language. The agent creates elements and relationships on the canvas. These canvas changes produce sync requests, and the code is updated to match. The human sees the architecture appear on the canvas and can edit, restructure, or remove anything at any time.

### 9.3 From Existing Code

The human points the tool at source files. The code is parsed and canvas elements are generated to represent the existing architecture. From this point, the canvas and code are linked and the sync loop begins.

---

## 10. Roles

### 10.1 The Agent's Role

The agent participates in the sync loop alongside the human. It can:

- **Make design changes.** The agent can create, edit, or delete elements and relationships on the canvas. These changes produce code requests.
- **Make code changes.** The agent can implement, refactor, fix, or test code. These changes produce design requests.
- **Loop autonomously.** Design changes trigger code updates, which may trigger canvas updates, which may trigger further code updates. The agent can continue processing until design and code stabilize (no further changes needed). Each iteration is visible on the canvas, and the human can intervene at any point.
- **Respond to user requests.** The human can ask the agent to do anything. The agent decides whether to change code, canvas, or both, and the sync loop handles the rest.

### 10.2 The Human's Role

The human maintains design authority through direct action:

- **Edit the canvas at any time.** Canvas changes produce code requests. The code is updated to match. If a canvas change was made that the human disagrees with, the human edits it, and the sync loop propagates the correction.
- **Edit code at any time.** Code changes produce design requests. The canvas is updated to match. The human sees the architectural impact of every code change.
- **Review via version control.** The change history provides a full audit trail of all canvas and code changes, who made them, and when. The human can review diffs, revert changes, and use branches for experimental work.
- **Request agent assistance.** The human can ask the agent for help with anything: writing pseudo code, analyzing the design, fixing bugs, writing tests, refactoring.

---

## 11. Agent Communication Protocol

The spec does not mandate a specific communication mechanism between agents and the canvas. Valid approaches include:
- **Direct file I/O:** The agent reads and writes canvas files directly.
- **Canvas tool API:** The canvas tool exposes an API (HTTP, IPC, or plugin SDK) that the agent calls.
- **CLI mediation:** A CLI tool accepts structured input and modifies the canvas.
- **Prompt-based:** The agent outputs structured descriptions of changes, and a harness applies them to the canvas.

All approaches MUST result in valid canvas state that fulfills the design contract.

---

## 12. Multi-User Environments

The core spec uses "the human" for simplicity. In teams, multiple humans share the canvas. The sync model does not change: every change on either side produces a sync request for the other side, regardless of who made it.

Implementations targeting multi-user environments MAY extend the system to carry user identity for audit and attribution purposes. The spec does not prescribe a user identity format.

Implementations MAY restrict which users can edit which elements or which parts of the codebase. The spec does not define a permission model. Role-based access, approval workflows, and team boundaries are organizational concerns that belong to implementations.

