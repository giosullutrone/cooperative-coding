# CooperativeCoding

> Current agentic coding treats software as text to be generated.
> CooperativeCoding treats it as architecture to be negotiated.

AI agents can write code, but they can't share architectural intent with humans. The result is drift, loss of design coherence, and humans reduced to code reviewers. CooperativeCoding is an open standard that fixes this by giving architecture a machine-readable, tool-agnostic home — where humans design and agents implement, with a shared source of truth between them.

## What is CooperativeCoding?

CooperativeCoding is an open specification for human-AI collaborative software design. It defines a shared data model — built on [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/) — where architectural decisions live as first-class objects: classes, interfaces, methods, fields, and their relationships. Humans design on a visual canvas, AI agents propose improvements and implement code, and a bidirectional sync engine keeps everything aligned.

The spec is language-agnostic and tool-agnostic. It can be implemented for any programming language (Python, TypeScript, Rust, ...) and any canvas tool (Obsidian, VS Code, Excalidraw, ...).

## How It Works

1. **Human designs** — creates architectural nodes on a visual canvas (classes, interfaces, packages) with responsibilities, fields, methods, and relationships
2. **Agent proposes** — analyzes the design and proposes improvements as *ghost nodes* (dashed borders, awaiting review)
3. **Human reviews** — accepts, rejects, or modifies proposals. The human is always the design authority.
4. **Agent implements** — generates source code from accepted canvas nodes, following the documentation contracts
5. **Sync keeps truth** — bidirectional sync ensures canvas and code stay aligned as both evolve

## The Specification

| Document | Description |
|---|---|
| [Introduction](spec/00-introduction.md) | Vision, principles, and terminology |
| [Data Model](spec/01-data-model.md) | Canvas nodes, edges, and metadata schemas |
| [Lifecycle](spec/02-lifecycle.md) | Ghost proposals and status transitions |
| [Sync](spec/03-sync.md) | Bidirectional sync semantics |
| [Language Bindings](spec/04-language-bindings.md) | Contract for language-specific mappings |

## Language Bindings

| Language | Binding | Status |
|---|---|---|
| Python | [bindings/python.md](bindings/python.md) | Reference binding |

Want to add a binding for your language? See [CONTRIBUTING.md](CONTRIBUTING.md).

## Reference Implementation

The Python reference implementation is available at [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python) — includes a CLI (`ccoding`), an Obsidian canvas plugin, and a Claude Code skill.

## Contributing

CooperativeCoding is an open initiative. We welcome contributions to the spec, new language bindings, and implementations for new canvas tools. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

This specification is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/).
