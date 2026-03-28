# CooperativeCoding Specification: Sync

**Version 1.0.0**

---

## 1. Overview

The [Data Model](01-data-model.md) defines the structures. The [Lifecycle](02-lifecycle.md) defines the sync loop and the roles of human and agent. This document defines the sync rules: what happens when canvas changes, what happens when code changes, and how the two sides stay aligned.

Sync is bidirectional. Canvas changes update code. Code changes update the canvas. Every change on either side always produces a sync request for the other side. The agent evaluates each request and applies the necessary changes, which may be none if the other side already matches. When a sync request produces no changes, no further sync requests are generated, and the loop stabilizes.

The rules in this document are intentionally abstract. They define *what* must happen, not *how* an implementation achieves it.

---

## 2. Canvas to Code

When the canvas changes (nodes created, edited, or deleted; edges created, edited, or deleted), a code request is produced. The agent evaluates the request and updates the corresponding source code to match.

The general rule: every canvas change produces a code request. Each node maps to a code element, and each edge maps to a relationship, as defined by the [Data Model](01-data-model.md) and the active language binding. Some edges have defined code constructs (e.g., `inherits` produces an inheritance declaration). Others (e.g., `calls`, `context`) have no defined code construct, but still produce sync requests that the agent evaluates. The agent decides what, if any, code changes are needed.

Sync MUST NOT overwrite method bodies. Node content maps to documentation and signatures in code; the implementation inside method bodies is the code's domain.

### 2.1 Change Propagation

When the agent updates code from a code request, it MUST consider the full impact of the change. For example, if a method signature changes, the agent MUST also update code that calls that method. If the ripple effects require architectural changes, the agent updates the canvas, which produces further code requests through the normal sync loop.

### 2.2 Test Node Sync (Canvas to Code)

Test nodes (`ccoding.kind: "test"`) follow the same canvas-to-code sync rules as other nodes. The active language binding defines how test nodes map to the target language's test framework conventions. Sync MUST NOT overwrite existing test method bodies, same as regular method bodies.

---

## 3. Code to Canvas

When source code changes, a design request is produced. The agent evaluates the request and updates the corresponding canvas nodes to match.

The general rule: every code change produces a design request. The agent evaluates it and decides what canvas changes are needed, which may be none for changes that are purely implementation details (local variables, control flow, performance optimizations within method bodies).

Sync MUST NOT overwrite canvas content that has no corresponding code representation (e.g., manually added design notes within a node).

### 3.1 Test Results

Test results are runtime data, not architectural changes. When sync detects updated test results, it MUST update the test node's content. The format of the results section is defined by the active language binding.

---

## 4. The "Already Matching" Check

The sync loop stabilizes because each sync request brings the two sides closer to alignment. When a code request updates the code to match the canvas, the resulting design request finds the canvas already matches the code, and the agent determines no further changes are needed. The loop terminates.

Implementations MUST ensure that the sync loop converges. If the agent's response to a sync request always brings the two sides closer to agreement (rather than introducing new divergence), the loop will terminate. An implementation where the agent oscillates between two states is a bug.

---

## 5. State Tracking

Implementations MUST maintain a mapping between canvas nodes and code elements so that sync requests can be evaluated. The `ccoding.qualifiedName` field is the primary key for this mapping. Implementations MAY use version control, content hashing, file watchers, or any other mechanism for detecting changes and tracking state.

When no prior mapping exists (first sync or new project), sync SHOULD perform a reconciliation pass: match canvas nodes to code elements by `ccoding.qualifiedName` and flag any unmatched elements for human attention.

---

## 6. Sync Triggers

Sync is continuous: every change on either side produces a sync request. Implementations define the mechanism for detecting changes (file watchers, save hooks, version control hooks, polling, etc.). Implementations SHOULD also support a manual sync command for cases where automatic detection misses a change or the user wants to force a full reconciliation.

---

## 7. Multiple Canvas Files

A project MAY have multiple `.canvas` files representing different architectural views. If the same code element appears on multiple canvases, sync MUST keep all canvas representations consistent: a code change MUST update all canvas nodes that map to the same `ccoding.qualifiedName`. Implementations SHOULD warn users when the same element appears on multiple canvases.
