# CooperativeCoding Python Language Binding

*Informative appendix showing how the CooperativeCoding specification maps to Python.*

---

**Target language:** `python`
**Spec version:** v1.0.0
**Status:** Reference binding
**Python version:** 3.11+

This binding is used by the [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python) reference implementation.

---

## 1. Overview

This document is an informative (non-normative) appendix to the [CooperativeCoding specification](../spec/00-introduction.md). It describes how the abstract concepts defined in the core spec (nodes, edges, stereotypes, node content, sync rules) map to Python's type system, documentation conventions, and module structure.

The mapping aims for zero surprise: a Python developer reading the generated code should see idiomatic Python, and a canvas user reading a node should see clean, language-neutral design documentation. The binding sits in the middle, translating between these two views.

Everything described here reflects what the reference implementation does. Other implementations targeting Python are free to make different choices within the spec's degrees of freedom.

---

## 2. Stereotype Mapping

The `ccoding.stereotype` field on a class node determines which Python construct sync generates. When no stereotype is set, the node maps to a plain class.

| `ccoding.stereotype` | Python Construct | Import Required |
|---|---|---|
| *(none)* | `class Foo:` | none |
| `protocol` | `class Foo(Protocol):` | `from typing import Protocol` |
| `abstract` | `class Foo(ABC):` | `from abc import ABC, abstractmethod` |
| `dataclass` | `@dataclass class Foo:` | `from dataclasses import dataclass` |
| `enum` | `class Foo(Enum):` | `from enum import Enum` |

**Behavioral notes:**

- **`protocol`**: Methods on a protocol class are not decorated with `@abstractmethod`. Protocol uses structural subtyping: any class with matching method signatures satisfies the protocol without explicit inheritance. The canvas `implements` edge translates to explicit Protocol inheritance in code, which opts into nominal checking.
- **`abstract`**: Methods listed on the canvas that lack a pseudo code section are generated with `@abstractmethod` and an ellipsis body (`...`). Methods with pseudo code are generated as concrete methods.
- **`dataclass`**: Fields are generated as dataclass fields with type annotations. The field order on the canvas determines the field order in the generated dataclass (which affects `__init__` parameter order). Fields with defaults come after fields without.
- **`enum`**: Enum members are extracted as `constant` nodes with `member` edges to the enum class. Since enum members rarely have external connections, they are typically inlined into the enum class body under a `### Constants` subsection. The type column in the canvas field node determines the enum value type (e.g., `str` produces a `class Foo(str, Enum):` mixin). Private members (names starting with `_`) are excluded.

Unknown stereotypes are treated as plain classes.

---

## 3. Documentation Format

Python uses [Google-style docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings) extended with CooperativeCoding sections. This format is compatible with standard Python documentation tools (Sphinx with `napoleon`, pdoc, mkdocstrings) while carrying the additional design information that the canvas displays.

Three sections are added beyond the standard Google docstring style:

- **`Responsibility:`**: appears on class and method docstrings. Describes what this element owns in the system. This is the most important section for design: it defines the boundaries.
- **`Pseudo Code:`**: appears on method docstrings only. A numbered step-by-step description of the algorithm. Deliberately not real code: it describes what happens, not how in language-specific terms.
- **`Collaborators:`**: appears on class docstrings only. Lists other classes this one works with and why. Derived from edges connected to the class node during sync.

These sections are designed to be safely ignored by standard Python documentation tools that do not recognize them.

### 3.1 Class Docstring

A class docstring maps to the class node's content on the canvas. The first line is the one-line summary. The `Responsibility:` block defines what the class owns. `Collaborators:` captures edge-derived relationships in prose. `Attributes:` maps to the class's field nodes.

```python
class DocumentParser(Protocol):
    """Parse raw documents into structured AST nodes.

    Responsibility:
        Owns the full parsing pipeline from raw text to validated AST,
        including plugin application and caching.

    Collaborators:
        ParserConfig: Provides tokenizer and parsing settings.
        ParserPlugin: Transforms AST during parsing pipeline.

    Attributes:
        config: Parser configuration and settings.
        _cache: Memoization cache keyed by source hash.
        plugins: Ordered list of transform plugins.
    """
```

### 3.2 Method Docstring

A method docstring maps to the method node's content on the canvas. The `Pseudo Code:` section is the algorithm description. Standard Google-style sections (`Args:`, `Returns:`, `Raises:`) map to the signature information in the method node.

```python
def parse(self, source: str) -> AST:
    """Transform raw source into a validated AST.

    Responsibility:
        Parse raw document text into structured AST nodes,
        applying all registered plugins in order.

    Pseudo Code:
        1. Check _cache for source hash
        2. If cached, return cached AST
        3. Tokenize source using config.tokenizer
        4. Build raw AST from token stream
        5. For each plugin: ast = plugin.transform(ast)
        6. Validate final AST structure
        7. Cache and return

    Args:
        source: Raw document text to parse.

    Returns:
        Parsed and validated abstract syntax tree.

    Raises:
        ParseError: If the source is malformed.
    """
```

### 3.3 Field Documentation

Field-level design information maps to a comment block above the field annotation in Python code:

```python
class DocumentParser(Protocol):
    # Responsibility: Holds parser configuration including tokenizer
    # settings and output format preferences.
    # Constraints: Immutable after initialization. Validated on
    # construction via ParserConfig.validate().
    # Default: ParserConfig.default()
    config: ParserConfig
```

---

## 4. Node Content Format

The reference implementation uses structured markdown in canvas node text. This is not part of the core spec (node text is opaque per [Data Model, Section 8](../spec/01-data-model.md)), but is documented here for interoperability with the reference implementation.

### Underscore Escaping

Python identifiers frequently contain underscores (`_`) which can trigger unintended markdown formatting (e.g., `__init__` renders as bold text). The reference implementation escapes all underscores in identifier names within node content: `__init__` becomes `\_\_init\_\_`, `my_method` becomes `my\_method`. This escaping applies to method names, field names, module names, and constant names in heading text. The `node.name` field retains the unescaped Python identifier for code generation.

### 4.1 Class Node

A class node contains the class-level information: name, responsibility, and constraints. Members (methods, fields, constants) that have no connections outside their parent class are inlined into the class body rather than existing as separate nodes. Only members with external connections (e.g., `calls`, `overrides`, `tests` edges to/from other nodes) get their own canvas nodes.

An inlined class node uses `### Fields`, `### Methods`, and `### Constants` subsections separated by horizontal rules:

```markdown
## DocumentParser

> Responsible for parsing raw documents into structured AST nodes

---

### Fields
- *config*: ParserConfig
- *\_cache*: dict[str, AST]

### Methods
- parse(source: str) -> AST
  > Transform raw source into a validated AST
- validate(ast: AST, strict: bool) -> bool
  > Check AST structure for errors

### Constraints
- Thread-safe for concurrent parsing
- Plugin order must be deterministic
```

Members that DO have external connections (e.g., a method targeted by a `calls` or `overrides` edge from another class) remain as separate nodes connected via `member` edges, following the standard method/field node format.

### 4.2 Method Node

A method node contains the fully qualified method name, a responsibility section, the signature broken into IN/OUT/RAISES, and numbered pseudo code.

```markdown
## DocumentParser.parse

### Responsibility
Transform raw source into a validated AST, applying all registered plugins in order.

### Signature
- **IN:** source: `String` -- raw document text
- **OUT:** `AST` -- parsed syntax tree
- **RAISES:** `ParseError` -- on malformed input

### Pseudo Code
1. Check _cache for source hash
2. If cached, return cached AST
3. Tokenize source using config.tokenizer
4. Build raw AST from token stream
5. For each plugin: ast = plugin.transform(ast)
6. Validate final AST structure
7. Cache and return
```

**Variadic and special parameters:** Python-specific parameter forms are mapped as follows:
- `*args` is represented in canvas as `*args: type` (or `*args` if untyped)
- `**kwargs` is represented in canvas as `**kwargs: type` (or `**kwargs` if untyped)
- Keyword-only arguments (after `*`) are represented with a `*` separator in the parameter list
- Positional-only arguments (before `/`) are represented with a `/` separator in the parameter list

Round-trip fidelity: these special forms MUST survive canvas to code to canvas without loss.

### 4.3 Field Node

A field node contains the fully qualified field name, a responsibility section, the type, and any constraints.

```markdown
## DocumentParser.config

### Responsibility
Holds parser configuration including tokenizer settings.

### Type
`ParserConfig`

### Constraints
- Immutable after parser initialization
- Validated on construction
```

### 4.4 Test Node

A test node contains the test class name, what is being tested, the test methods, and results.

```markdown
## TestDocumentParser

> Tests for DocumentParser: verifies the parsing pipeline from raw text to validated AST

### Class Under Test
- DocumentParser (`parsers.document.DocumentParser`)

### Test Methods
- test_parse_simple_document: parses a minimal valid document and checks AST structure
- test_parse_cached_result: verifies that repeated parsing returns cached AST
- test_parse_with_plugins: applies plugins in order and checks transformed AST
- test_parse_malformed_input: expects ParseError on invalid input
- test_validate_rejects_broken_ast: validates that malformed ASTs are rejected

### Results
- test_parse_simple_document: **PASS**
- test_parse_cached_result: **PASS**
- test_parse_with_plugins: **FAIL** -- `AssertionError: expected 3 plugins applied, got 2`
- test_parse_malformed_input: **PASS**
- test_validate_rejects_broken_ast: **ERROR** -- `TypeError: validate() missing 1 required positional argument: 'strict'`
```

---

## 5. Test Framework Mapping

The reference implementation uses [pytest](https://docs.pytest.org/) as the default test framework.

| Aspect | pytest Convention |
|---|---|
| Test file location | `tests/` directory mirroring the source structure (e.g., `src/parsers/document.py` to `tests/parsers/test_document.py`) |
| Test file naming | `test_<module>.py` |
| Test class naming | `Test<ClassName>` (no base class required) |
| Test method naming | `test_<description>` using snake_case |
| Fixtures | `@pytest.fixture` decorators |
| Parametrization | `@pytest.mark.parametrize` |

### Generated Test Code

Given the test node above with a `tests` edge to a `DocumentParser` class node, sync generates:

```python
import pytest

from parsers.document import DocumentParser


class TestDocumentParser:
    """Tests for DocumentParser.

    Verifies the parsing pipeline from raw text to validated AST.
    """

    def test_parse_simple_document(self) -> None:
        """Parse a minimal valid document and check AST structure."""
        raise NotImplementedError

    def test_parse_cached_result(self) -> None:
        """Verify that repeated parsing returns cached AST."""
        raise NotImplementedError

    def test_parse_with_plugins(self) -> None:
        """Apply plugins in order and check transformed AST."""
        raise NotImplementedError

    def test_parse_malformed_input(self) -> None:
        """Expect ParseError on invalid input."""
        raise NotImplementedError

    def test_validate_rejects_broken_ast(self) -> None:
        """Validate that malformed ASTs are rejected."""
        raise NotImplementedError
```

Key observations:

- The `tests` edge to `DocumentParser` produced the `from parsers.document import DocumentParser` import, derived from the target node's `ccoding.qualifiedName`.
- Test method names carry the `test_` prefix required by pytest discovery.
- Each test method's docstring comes from the pseudo code description in the test node.
- Method bodies are `raise NotImplementedError`. For pytest, the reference implementation does not use `pass` or `...` because a passing test with no assertions is misleading.
- Return type annotations are `-> None` for all test methods.

**Stub detection during sync:** A method body is considered a stub if it consists solely of:
- `raise NotImplementedError` (or `raise NotImplementedError(...)`)
- `pass`
- `...` (Ellipsis)
- A single docstring with no other statements

Any other body content is considered implemented and MUST NOT be overwritten during sync. When the canvas changes a method's signature, sync MUST update the signature while preserving the existing body.

### Test Result Format

The sync reads test results from pytest's output. The reference implementation supports two result sources:

- **pytest JSON report** (via the `pytest-json-report` plugin). Maps each test's `outcome` field to canvas result status.
- **JUnit XML** (via `pytest --junitxml=results.xml`).

Result status mapping:

| pytest Outcome | Canvas Status | Details |
|---|---|---|
| `passed` | **PASS** | No additional details. |
| `failed` | **FAIL** | Assertion message, truncated to a single line. |
| `error` | **ERROR** | Exception type and message, truncated to a single line. |
| `skipped` | **SKIP** | Skip reason, if available. |

---

## 6. Type Hint Mapping

Canvas uses language-neutral type names. The binding translates these to Python's native type syntax during sync in both directions.

| Canvas Type | Python Type |
|---|---|
| `String` | `str` |
| `Integer` | `int` |
| `Boolean` | `bool` |
| `Float` | `float` |
| `List<T>` | `list[T]` |
| `Map<K, V>` | `dict[K, V]` |
| `Optional<T>` | `T \| None` |
| `Set<T>` | `set[T]` |
| `Void` | `None` |

Additional conventions:

- **`Tuple<T, U>`** maps to `tuple[T, U]`
- **`Callable<[Args], Return>`** maps to `Callable[[Args], Return]` (from `collections.abc`)
- **`Union<T, U>`** maps to `T | U`
- **Custom types** (e.g., `ParserConfig`, `AST`) pass through unchanged
- **Generics**: `T` as a type variable maps to `TypeVar("T")` from `typing`

The Python 3.10+ union syntax (`X | Y`) is preferred over `Optional[X]` and `Union[X, Y]`. The reference implementation targets Python 3.11+ and uses the modern syntax throughout.

---

## 7. Edge Label Interpretation

For `composes` edges, the reference implementation parses the label to extract the field name:

1. If the label contains `:` or ` - `, the substring before the first separator is the field name. The remainder is a descriptive comment.
2. If the label contains no separator, the entire label is the field name.
3. If the label is absent, the field name is derived from the target node's name (lowercased, snake_cased).

For all other relation types, labels are informational and not used during code generation.

---

## 8. Example Round-Trip

This section walks through a complete cycle: a class node on the canvas, the Python code generated from it, an edit made in code, and the resulting canvas update.

### 8.1 Canvas Data (Starting Point)

A canvas file contains a `DocumentParser` class node and a separate method node for `parse`, connected by a `member` edge:

```json
{
  "nodes": [
    {
      "id": "node-parser",
      "type": "text",
      "x": 100,
      "y": 100,
      "width": 340,
      "height": 200,
      "text": "## DocumentParser\n\n> Responsible for parsing raw documents into structured AST nodes\n\n### Constraints\n- Thread-safe for concurrent parsing",
      "ccoding": {
        "kind": "class",
        "stereotype": "protocol",
        "language": "python",
        "source": "src/parsers/document.py",
        "qualifiedName": "parsers.document.DocumentParser"
      }
    },
    {
      "id": "node-parse-method",
      "type": "text",
      "x": 520,
      "y": 100,
      "width": 340,
      "height": 300,
      "text": "## DocumentParser.parse\n\n### Responsibility\nTransform raw source into a validated AST, applying all registered plugins.\n\n### Signature\n- **IN:** source: `String` -- raw document text\n- **OUT:** `AST` -- parsed syntax tree\n- **RAISES:** `ParseError` -- on malformed input\n\n### Pseudo Code\n1. Check _cache for source hash\n2. If cached, return cached AST\n3. Tokenize source using config.tokenizer\n4. Build raw AST from token stream\n5. Validate final AST structure\n6. Cache and return",
      "ccoding": {
        "kind": "method",
        "language": "python",
        "source": "src/parsers/document.py",
        "qualifiedName": "parsers.document.DocumentParser.parse"
      }
    }
  ],
  "edges": [
    {
      "id": "edge-member-parse",
      "fromNode": "node-parser",
      "toNode": "node-parse-method",
      "fromSide": "right",
      "toSide": "left",
      "ccoding": {
        "relation": "member"
      }
    }
  ]
}
```

### 8.2 Generated Python Code

Sync reads the canvas and generates `src/parsers/document.py`:

```python
from __future__ import annotations

from typing import Protocol


class DocumentParser(Protocol):
    """Parse raw documents into structured AST nodes.

    Responsibility:
        Owns the full parsing pipeline from raw text to validated AST.

    Attributes:
        config: Parser configuration and settings.
        _cache: Memoization cache keyed by source hash.
    """

    config: ParserConfig
    _cache: dict[str, AST]

    def parse(self, source: str) -> AST:
        """Transform raw source into a validated AST.

        Responsibility:
            Transform raw source into a validated AST, applying all
            registered plugins.

        Pseudo Code:
            1. Check _cache for source hash
            2. If cached, return cached AST
            3. Tokenize source using config.tokenizer
            4. Build raw AST from token stream
            5. Validate final AST structure
            6. Cache and return

        Args:
            source: Raw document text to parse.

        Returns:
            Parsed and validated abstract syntax tree.

        Raises:
            ParseError: If the source is malformed.
        """
        ...
```

Key observations:

- The `protocol` stereotype produced `class DocumentParser(Protocol):` with the `typing.Protocol` import.
- Canvas type `Map<String, AST>` became `dict[str, AST]` via the type hint mapping.
- The `parse` method has the full docstring with `Responsibility:` and `Pseudo Code:` sections from the method node.
- Method bodies are `...` (ellipsis), the protocol stub. For non-protocol classes, the reference implementation generates `raise NotImplementedError`.

### 8.3 Code Edit

A developer adds a plugin step to the `parse` method's pseudo code and adds a new parameter:

```python
    def parse(self, source: str, *, strict: bool = False) -> AST:
        """Transform raw source into a validated AST.

        Responsibility:
            Transform raw source into a validated AST, applying all
            registered plugins.

        Pseudo Code:
            1. Check _cache for source hash
            2. If cached, return cached AST
            3. Tokenize source using config.tokenizer
            4. Build raw AST from token stream
            5. For each plugin: ast = plugin.transform(ast)
            6. Validate final AST structure (strict mode raises on warnings)
            7. Cache and return

        Args:
            source: Raw document text to parse.
            strict: If True, treat warnings as errors during validation.

        Returns:
            Parsed and validated abstract syntax tree.

        Raises:
            ParseError: If the source is malformed.
        """
        ...
```

### 8.4 Canvas Update

The code change produces a design request. Sync updates the method node on the canvas:

```markdown
## DocumentParser.parse

### Responsibility
Transform raw source into a validated AST, applying all registered plugins.

### Signature
- **IN:** source: `String` -- raw document text
- **IN:** strict: `Boolean` = False -- if True, treat warnings as errors during validation
- **OUT:** `AST` -- parsed syntax tree
- **RAISES:** `ParseError` -- on malformed input

### Pseudo Code
1. Check _cache for source hash
2. If cached, return cached AST
3. Tokenize source using config.tokenizer
4. Build raw AST from token stream
5. For each plugin: ast = plugin.transform(ast)
6. Validate final AST structure (strict mode raises on warnings)
7. Cache and return
```

The Python type `bool` was translated back to the canvas type `Boolean`, and `False` was preserved as the default value. The new pseudo code step and the updated step were carried over verbatim. The round-trip is lossless for the information the binding tracks.

---

## 9. Reference Implementation

The full working implementation is available at [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python). The repository contains the sync logic, code parser, code generator, and docstring mapping logic that this appendix describes.
