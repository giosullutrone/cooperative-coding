# Spec Layer Separation: Remove Implementation Details from Core Spec

**Date:** 2026-04-09

---

## Problem

The CooperativeCoding spec mixes two layers:

1. **The cooperation contract** — bidirectional sync, design authority, identity, responsibility, pseudo code
2. **A specific canvas data model** — JSON Canvas format, `ccoding` namespace, 10 edge relation types, 9 node kinds, metadata schemas

Layer 2 is a reference implementation presented as normative spec. This prevents valid implementations (like the Obsidian-based CLI) from being spec-compliant despite fulfilling the cooperation model.

## Decision

Strip Layer 2 entirely. The core spec defines only the cooperation contract. JSON Canvas implementation details belong in their own repo when someone builds them.

## Design Decisions (from brainstorming)

| Question | Decision |
|----------|----------|
| Canvas file format | **Format-agnostic.** No JSON Canvas, no `ccoding` namespace. The spec defines abstract concepts only. |
| "Every element is a node" | **Identity, not visualization.** Implementations MUST identify individual code elements for sync. No requirement on how they're visualized. |
| Relation/edge types | **None prescribed.** Implementations must track relationships that affect code generation. No enumerated set. |
| Node content contract | **Responsibility + pseudo code required.** Every code element carries a responsibility statement. Every method carries pseudo code. Format is up to the binding. |
| Language binding contract | **Reframed around capabilities.** MUST: parse, generate, map docs, define stereotypes. SHOULD: type mapping, imports, file conventions. MAY: test framework, code style. Remove references to specific node kinds/edge types. |
| Existing spec documents | **Rewrite in place.** Drop JSON Canvas reference implementation entirely. |

## Resulting Document Structure

4 documents (down from 5):

1. **00-introduction.md** — Vision, principles, terminology, scope. "Every Element is a Node" becomes "Every Element Has Identity." Remove JSON Canvas refs, JSON schemas section.
2. **01-design-contract.md** — Replaces old data model. Defines: identity (qualified names), responsibility requirement, pseudo code requirement, relationship tracking requirement, stereotypes, source mapping.
3. **02-sync.md** — Preserved with format-specific references removed. Sync loop, convergence, body preservation, state tracking.
4. **03-language-bindings.md** — Reframed MUST/SHOULD/MAY capabilities. Phased adoption kept.

## Removed Artifacts

- `spec/01-data-model.md` (replaced by `01-design-contract.md`)
- `spec/schema/ccoding-canvas.schema.json`
- `spec/schema/ccoding-node.schema.json`
- `spec/schema/ccoding-edge.schema.json`
- `spec/schema/README.md`
- All prescribed node kinds (class, method, field, package, test, interface, module, function, constant)
- All prescribed edge relation types (inherits, implements, composes, depends, member, calls, tests, context, contains, overrides)
- JSON Canvas field definitions and constraints
- Relation validity rules
- Edge label interpretation rules

## Python Binding Impact

`bindings/python.md` keeps its concrete mappings but reframes them as binding-level choices:
- Stereotype table, docstring format, type mapping, test framework — all presented as "the reference binding's convention"
- Remove references to `ccoding` metadata fields, node kinds, edge relation types
