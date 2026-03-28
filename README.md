# CooperativeCoding

> Current agentic coding treats software as text to be generated.
> CooperativeCoding treats it as architecture to be negotiated.

AI agents are remarkably good at implementing code that satisfies a given specification, but they are not yet capable of producing clean software designs. CooperativeCoding gives the human a visual canvas where architecture is defined in natural language, while the code stays in continuous bidirectional sync. Both the human and the agent can change either side, and every change is immediately visible.

## What is CooperativeCoding?

CooperativeCoding is an open specification for human-AI collaborative software design. It defines a shared data model built on [JSON Canvas v1.0](https://jsoncanvas.org/spec/1.0/) where architectural decisions live as first-class objects: classes, interfaces, methods, fields, and their relationships. The canvas and the code stay in sync through a continuous loop, and the human maintains authority by editing the canvas at any time.

The spec is language-agnostic and tool-agnostic. It can be implemented for any programming language (Python, TypeScript, Rust, ...) and any canvas tool (Obsidian, VS Code, Excalidraw, ...).

## How It Works

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

1. **Design** on a visual canvas: create classes, methods, fields as nodes with responsibilities, pseudo code, and relationships as edges.
2. **Sync** keeps canvas and code aligned. Canvas changes produce code requests (update the code). Code changes produce design requests (update the canvas).
3. **The agent loops autonomously**: implement, test, fix, repeat. Every change is visible on the canvas.
4. **The human intervenes** at any time by editing the canvas or the code. Version control provides audit, review, and rollback.

## Visual Example

### Canvas: A UserService with its methods

```
┌─────────────────────────────┐
│ UserService                 │
│                             │
│ > Manages user accounts:    │
│   creation, lookup,         │
│   updates, and deletion.    │
│                             │
│ ### Constraints             │
│ - All operations require    │
│   authenticated context     │
└──────────┬──────────────────┘
           │ member
     ┌─────┼──────────┐
     │     │          │
     v     v          v
┌─────────┐ ┌────────┐ ┌──────────────────────────┐
│ get_user│ │ delete_ │ │ register                 │
│         │ │ user   │ │                          │
│ > Look  │ │ > Soft │ │ > Create a new user      │
│   up a  │ │   dele │ │   account with validated │
│   user  │ │   te a │ │   input, hashed creds,   │
│   by ID │ │   user │ │   and a welcome email.   │
│         │ │        │ │                          │
└─────────┘ └────────┘ │ ### Pseudo Code          │
                       │ 1. Validate name/email   │
                       │ 2. Check for duplicates  │
                       │ 3. Hash password         │
                       │ 4. Create user record    │
                       │ 5. Send welcome email    │
                       │ 6. Return new user       │
                       └───────┬──────────────────┘
                               │ calls
                               v
                       ┌──────────────────┐
                       │ PasswordHasher   │
                       │                  │
                       │ > Securely hash  │
                       │   passwords      │
                       └──────────────────┘
```

### Generated Python code (from canvas)

```python
class UserService:
    """Manages user accounts: creation, lookup, updates, and deletion.

    Responsibility:
        All user lifecycle operations including registration,
        lookup, updates, and soft deletion.
    """

    def get_user(self, user_id: str) -> User:
        """Look up a user by ID."""
        raise NotImplementedError

    def delete_user(self, user_id: str) -> None:
        """Soft delete a user."""
        raise NotImplementedError

    def register(self, name: str, email: str, password: str) -> User:
        """Create a new user account with validated input, hashed creds, and a welcome email.

        Pseudo Code:
            1. Validate name and email format
            2. Check for duplicates in db
            3. Hash password using hasher
            4. Create user record in db
            5. Send welcome email
            6. Return new user
        """
        raise NotImplementedError
```

### The sync loop in action

1. Human creates the `UserService` class and `register` method on the canvas with pseudo code
2. Canvas change produces a **code request**. Agent generates the Python code above.
3. Agent implements `register()`, discovers it needs a `validate_email()` helper
4. Code change produces a **design request**. Agent adds a `validate_email` method node to the canvas.
5. Human sees the new method on the canvas. The loop stabilizes.
6. Human edits the pseudo code on the canvas to add step "7. Provision default settings"
7. Canvas change produces a **code request**. Agent updates the implementation.

## The Specification

| Document | Description |
|---|---|
| [Introduction](spec/00-introduction.md) | Vision, principles, and terminology |
| [Data Model](spec/01-data-model.md) | Canvas nodes, edges, and metadata schemas |
| [Lifecycle](spec/02-lifecycle.md) | Sync loop, entry points, and roles |
| [Sync](spec/03-sync.md) | Sync semantics |
| [Language Bindings](spec/04-language-bindings.md) | Contract for language-specific mappings |

## Language Bindings

| Language | Binding | Status |
|---|---|---|
| Python | [bindings/python.md](bindings/python.md) | Reference binding |

Want to add a binding for your language? See [CONTRIBUTING.md](CONTRIBUTING.md).

## Reference Implementation

The Python reference implementation is available at [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python).

## Contributing

CooperativeCoding is an open initiative. We welcome contributions to the spec, new language bindings, and implementations for new canvas tools. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

This specification is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/).
