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

This document is an informative (non-normative) appendix to the [CooperativeCoding specification](../spec/00-introduction.md). It describes how the abstract concepts defined in the core spec — nodes, edges, stereotypes, documentation blocks, sync rules — map to Python's type system, documentation conventions, and module structure.

The mapping aims for zero surprise: a Python developer reading the generated code should see idiomatic Python, and a canvas user reading a node should see clean, language-neutral design documentation. The binding sits in the middle, translating between these two views.

Everything described here reflects what the reference implementation does. Other implementations targeting Python are free to make different choices within the spec's degrees of freedom.

---

## 2. Stereotype Mapping

The `ccoding.stereotype` field on a class node determines which Python construct the sync engine generates. When no stereotype is set, the node maps to a plain class.

| `ccoding.stereotype` | Python Construct | Import Required |
|---|---|---|
| *(none)* | `class Foo:` | — |
| `protocol` | `class Foo(Protocol):` | `from typing import Protocol` |
| `abstract` | `class Foo(ABC):` | `from abc import ABC, abstractmethod` |
| `dataclass` | `@dataclass class Foo:` | `from dataclasses import dataclass` |
| `enum` | `class Foo(Enum):` | `from enum import Enum` |

**Behavioral notes:**

- **`protocol`** — Methods on a protocol class are not decorated with `@abstractmethod`. Protocol uses structural subtyping: any class with matching method signatures satisfies the protocol without explicit inheritance. The canvas `implements` edge translates to explicit Protocol inheritance in code, which opts into nominal checking.
- **`abstract`** — Methods listed on the canvas node that lack a pseudo code section are generated with `@abstractmethod` and an ellipsis body (`...`). Methods with pseudo code are generated as concrete methods.
- **`dataclass`** — Fields listed on the canvas node become dataclass fields with type annotations. The field order on the canvas determines the field order in the generated dataclass (which affects `__init__` parameter order). Fields with defaults come after fields without.
- **`enum`** — Fields listed on the canvas node become enum members. The type column in the canvas field list determines the enum value type (e.g., `str` produces a `class Foo(str, Enum):` mixin).

Unknown stereotypes are treated as plain classes — the sync engine generates a standard `class` declaration with no special syntax or imports.

---

## 3. Documentation Format

Python uses [Google-style docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings) extended with CooperativeCoding sections. This format is compatible with standard Python documentation tools (Sphinx with `napoleon`, pdoc, mkdocstrings) while carrying the additional design information that the canvas displays.

Four sections are added beyond the standard Google docstring style:

- **`Responsibility:`** — appears on class and method docstrings. Describes what this element owns in the system. This is the most important section for design: it defines the boundaries.
- **`Pseudo Code:`** — appears on method docstrings only. A numbered step-by-step description of the algorithm. Deliberately not real code — it describes what happens, not how in language-specific terms.
- **`Collaborators:`** — appears on class docstrings only. Lists other classes this one works with and why. On the canvas, this information is primarily conveyed through edges, but the docstring captures it in prose for readability in code.
- **`Constraints:`** — appears on field documentation only (as comment blocks). Describes invariants, validation rules, and immutability requirements.

These sections are designed to be safely ignored by standard Python documentation tools that do not recognize them — they appear as additional text in the rendered docstring.

### 3.1 Class Docstring

A class docstring maps to the class node's structured markdown on canvas. The first line is the one-line summary. The `Responsibility:` block defines what the class owns. `Collaborators:` captures edge-derived relationships in prose. `Attributes:` maps to the canvas fields section.

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

**Section mapping:**

| Docstring Section | Canvas Node Section | Purpose |
|---|---|---|
| First line | Heading + summary | One-line summary |
| `Responsibility:` | Blockquote (`>`) | What this class owns |
| `Collaborators:` | Inferred from edges | Related classes |
| `Attributes:` | Fields section | Class fields |

### 3.2 Method Docstring

A method docstring maps to the method detail node's structured markdown on canvas. The `Pseudo Code:` section is the algorithm description — it round-trips between the canvas `### Pseudo Code` section and the docstring. Standard Google-style sections (`Args:`, `Returns:`, `Raises:`) map to the `### Signature` section on canvas.

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

**Section mapping:**

| Docstring Section | Canvas Node Section | Purpose |
|---|---|---|
| First line | `## ClassName.method` heading | One-line summary |
| `Responsibility:` | `### Responsibility` | What this method is responsible for |
| `Pseudo Code:` | `### Pseudo Code` | Step-by-step algorithm description |
| `Args:` | `### Signature` IN fields | Input parameters |
| `Returns:` | `### Signature` OUT fields | Return value |
| `Raises:` | `### Signature` RAISES fields | Exceptions |

### 3.3 Field Documentation

When a field is promoted to its own detail node on the canvas, its design information maps to a comment block above the field annotation in Python code:

```python
class DocumentParser(Protocol):
    # Responsibility: Holds parser configuration including tokenizer
    # settings and output format preferences.
    # Constraints: Immutable after initialization. Validated on
    # construction via ParserConfig.validate().
    # Default: ParserConfig.default()
    config: ParserConfig
```

**Section mapping:**

| Field Detail Section | Python Code | Purpose |
|---|---|---|
| `### Responsibility` | Comment block above field | What this field represents |
| `### Type` | Type annotation | The field's type |
| `### Constraints` | Comment block above field | Invariants and validation rules |
| `### Default` | Default value assignment or comment | Default value |

---

## 4. Node Content Format

The reference implementation uses structured markdown in canvas node text. This is not part of the core spec (node text is opaque per [Data Model, Section 9](../spec/01-data-model.md)), but is documented here for interoperability with the reference implementation.

Tools that consume canvas files produced by the reference implementation can rely on these formats. Tools that produce canvas files for the reference implementation to consume should follow them. Other implementations may use different internal formats as long as the sync mapping is consistent.

### 4.1 Class Node

A class node contains the class name as a heading, a responsibility blockquote, and sections for fields and methods. The `●` marker after a field or method indicates that a detail node exists for that element.

```markdown
## DocumentParser

> Responsible for parsing raw documents into structured AST nodes

### Fields
- config: `ParserConfig`
- _cache: `Map<String, AST>`
- plugins: `List<ParserPlugin>`

### Methods
- parse(source: `String`) -> `AST`
- validate(ast: `AST`) -> `Boolean`
```

### 4.2 Method Detail Node

A method detail node contains the fully qualified method name, a responsibility section, the signature broken into IN/OUT/RAISES, and numbered pseudo code.

```markdown
## DocumentParser.parse

### Responsibility
Transform raw source into a validated AST, applying all registered plugins in order.

### Signature
- **IN:** source: `String` — raw document text
- **OUT:** `AST` — parsed syntax tree
- **RAISES:** `ParseError` — on malformed input

### Pseudo Code
1. Check _cache for source hash
2. If cached, return cached AST
3. Tokenize source using config.tokenizer
4. Build raw AST from token stream
5. For each plugin: ast = plugin.transform(ast)
6. Validate final AST structure
7. Cache and return
```

### 4.3 Field Detail Node

A field detail node contains the fully qualified field name, a responsibility section, the type, and any constraints on the field's usage.

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

---

## 5. Test Node Format

Test nodes (`ccoding.kind: "test"`) map to Python test classes following [pytest](https://docs.pytest.org/) conventions. The reference implementation uses pytest as the default test framework. Other Python test frameworks (unittest, nose2) MAY be supported through configuration, but this binding documents the pytest mapping.

### 5.1 Test Framework Mapping

| Aspect | pytest Convention |
|---|---|
| Test file location | `tests/` directory mirroring the source structure (e.g., `src/parsers/document.py` → `tests/parsers/test_document.py`) |
| Test file naming | `test_<module>.py` — prefixed with `test_` |
| Test class naming | `Test<ClassName>` — prefixed with `Test`, no parentheses (pytest does not require inheritance from a base class) |
| Test method naming | `test_<description>` — prefixed with `test_`, using snake_case |
| Fixtures | `@pytest.fixture` decorators for setup and shared state |
| Parametrization | `@pytest.mark.parametrize` for data-driven tests |

### 5.2 Test Class Structured Content

A test node on the canvas uses the following structured markdown format. This is the test-specific counterpart of the class node format defined in Section 4.1.

```markdown
## TestDocumentParser

> Tests for DocumentParser — verifies the parsing pipeline from raw text to validated AST

### Class Under Test
- DocumentParser (`parsers.document.DocumentParser`)

### Test Methods
- test_parse_simple_document — parses a minimal valid document and checks AST structure
- test_parse_cached_result — verifies that repeated parsing returns cached AST
- test_parse_with_plugins — applies plugins in order and checks transformed AST
- test_parse_malformed_input — expects ParseError on invalid input
- test_validate_rejects_broken_ast — validates that malformed ASTs are rejected

### Results
- test_parse_simple_document: **PASS**
- test_parse_cached_result: **PASS**
- test_parse_with_plugins: **FAIL** — `AssertionError: expected 3 plugins applied, got 2`
- test_parse_malformed_input: **PASS**
- test_validate_rejects_broken_ast: **ERROR** — `TypeError: validate() missing 1 required positional argument: 'strict'`
```

**Section mapping:**

| Canvas Section | Purpose |
|---|---|
| `## TestClassName` heading | Test class name |
| Blockquote (`>`) | One-line summary of what is being tested |
| `### Class Under Test` | Lists class(es) being verified — corresponds to `tests` edges |
| `### Test Methods` | Each method as `test_name — pseudo code description of the scenario` |
| `### Results` | Each method as `test_name: **STATUS**` with optional failure/error details |

### 5.3 Generated Python Code

Given the test node above with a `tests` edge to a `DocumentParser` class node, the sync engine generates:

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
- Each test method's docstring comes from the pseudo code description in the `### Test Methods` section.
- Method bodies are `raise NotImplementedError` — the human or agent implements the actual test logic. For pytest, the reference implementation does not use `pass` or `...` because a passing test with no assertions is misleading.
- Return type annotations are `-> None` for all test methods, following the convention that tests do not return values.

### 5.4 Test Result Format

The sync engine reads test results from pytest's output. The reference implementation supports two result sources:

- **pytest JSON report** — produced by `pytest --json-report` (via the `pytest-json-report` plugin). The sync engine reads the JSON file and maps each test's `outcome` field (`passed`, `failed`, `error`) to the canvas result status.
- **JUnit XML** — produced by `pytest --junitxml=results.xml`. The sync engine parses the XML and extracts test case outcomes.

Result status mapping:

| pytest Outcome | Canvas Status | Details |
|---|---|---|
| `passed` | **PASS** | No additional details. |
| `failed` | **FAIL** | The assertion message from the `longrepr` or `message` field, truncated to a single line for canvas readability. |
| `error` | **ERROR** | The exception type and message from the `longrepr` or `message` field, truncated to a single line. |
| `skipped` | **SKIP** | The skip reason, if available. |

When test results are imported, the sync engine updates the `### Results` section of the test node's canvas content. Existing result entries are overwritten with the new status. Test methods that appear in the results but not in the `### Test Methods` section SHOULD be added to the test methods list (this handles tests added directly in code). Test methods that appear in the `### Test Methods` section but not in the results retain their previous result status — the absence of a result does not imply removal.

---

## 6. Type Hint Mapping

Canvas uses language-neutral type names in the `### Fields` and `### Signature` sections. The binding translates these to Python's native type syntax. The reference implementation performs this translation during both code generation (canvas to code) and code parsing (code to canvas).

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

**Additional conventions used by the reference implementation:**

- **`Tuple<T, U>`** maps to `tuple[T, U]`
- **`Callable<[Args], Return>`** maps to `Callable[[Args], Return]` (from `collections.abc`)
- **`Union<T, U>`** maps to `T | U`
- **Custom types** (e.g., `ParserConfig`, `AST`) pass through unchanged — they are class names defined elsewhere on the canvas
- **Generics** — `T` as a type variable maps to `TypeVar("T")` from `typing`

The Python 3.10+ union syntax (`X | Y`) is preferred over `Optional[X]` and `Union[X, Y]`. The reference implementation targets Python 3.11+ and uses the modern syntax throughout.

---

## 7. Example Round-Trip

This section walks through a complete cycle: a class node on the canvas, the Python code generated from it, an edit made in code, and the resulting canvas update. This demonstrates how the binding keeps both representations in sync.

### 7.1 Canvas Data (Starting Point)

A canvas file contains a `DocumentParser` class node and a detail node for its `parse` method, connected by a `detail` edge:

```json
{
  "nodes": [
    {
      "id": "node-parser",
      "type": "text",
      "x": 100,
      "y": 100,
      "width": 340,
      "height": 280,
      "text": "## DocumentParser\n\n> Responsible for parsing raw documents into structured AST nodes\n\n### Fields\n- config: `ParserConfig`\n- _cache: `Map<String, AST>`\n\n### Methods\n- parse(source: `String`) -> `AST` ●\n- validate(ast: `AST`) -> `Boolean`",
      "ccoding": {
        "kind": "class",
        "stereotype": "protocol",
        "language": "python",
        "source": "src/parsers/document.py",
        "qualifiedName": "parsers.document.DocumentParser",
        "status": "accepted",
        "proposedBy": null,
        "proposalRationale": null
      }
    },
    {
      "id": "node-parse-method",
      "type": "text",
      "x": 520,
      "y": 100,
      "width": 340,
      "height": 300,
      "text": "## DocumentParser.parse\n\n### Responsibility\nTransform raw source into a validated AST, applying all registered plugins.\n\n### Signature\n- **IN:** source: `String` — raw document text\n- **OUT:** `AST` — parsed syntax tree\n- **RAISES:** `ParseError` — on malformed input\n\n### Pseudo Code\n1. Check _cache for source hash\n2. If cached, return cached AST\n3. Tokenize source using config.tokenizer\n4. Build raw AST from token stream\n5. Validate final AST structure\n6. Cache and return",
      "ccoding": {
        "kind": "method",
        "language": "python",
        "source": "src/parsers/document.py",
        "qualifiedName": "parsers.document.DocumentParser.parse",
        "status": "accepted",
        "proposedBy": null,
        "proposalRationale": null
      }
    }
  ],
  "edges": [
    {
      "id": "edge-detail-parse",
      "fromNode": "node-parser",
      "toNode": "node-parse-method",
      "fromSide": "right",
      "toSide": "left",
      "label": "parse()",
      "ccoding": {
        "relation": "detail",
        "status": "accepted",
        "proposedBy": null,
        "proposalRationale": null
      }
    }
  ]
}
```

### 7.2 Generated Python Code

The sync engine reads the canvas and generates `src/parsers/document.py`:

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

    def validate(self, ast: AST) -> bool:
        """Validate an AST structure.

        Args:
            ast: The abstract syntax tree to validate.

        Returns:
            True if the AST is structurally valid.
        """
        ...
```

Key observations:

- The `protocol` stereotype produced `class DocumentParser(Protocol):` with the `typing.Protocol` import.
- Canvas type `Map<String, AST>` became `dict[str, AST]` via the type hint mapping.
- Canvas type `Boolean` became `bool`.
- The `parse` method has the full docstring with `Responsibility:` and `Pseudo Code:` sections from the detail node.
- The `validate` method has a minimal docstring since it had no detail node.
- Method bodies are `...` (ellipsis) — the protocol stub. For non-protocol classes, the reference implementation generates `raise NotImplementedError`.

### 7.3 Code Edit

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

Two changes were made: a new keyword-only parameter `strict: bool = False`, and an updated pseudo code (step 5 added, step 6 updated).

### 7.4 Canvas Update

The sync engine detects the code change and updates the canvas. The class node's method list and the method detail node both change.

The class node's method line updates:

```markdown
- parse(source: `String`, *, strict: `Boolean` = False) -> `AST` ●
```

The method detail node's content updates:

```markdown
## DocumentParser.parse

### Responsibility
Transform raw source into a validated AST, applying all registered plugins.

### Signature
- **IN:** source: `String` — raw document text
- **IN:** strict: `Boolean` = False — if True, treat warnings as errors during validation
- **OUT:** `AST` — parsed syntax tree
- **RAISES:** `ParseError` — on malformed input

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

## 8. Reference Implementation

The full working implementation is available at [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python). The repository contains the sync engine, code parser, code generator, and docstring mapping logic that this appendix describes.

Key modules:

| Module | Path | Responsibility |
|---|---|---|
| AST parser | `ccoding/code/parser.py` | Parses Python source files into an intermediate representation. Extracts classes, methods, fields, type annotations, docstrings, and relationships. Infers stereotypes from base classes and decorators. |
| Code generator | `ccoding/code/generator.py` | Generates Python source files from canvas node data. Produces class skeletons, method stubs, field declarations, import statements, and docstrings. |
| Docstring mapping | `ccoding/code/docstring.py` | Bidirectional translation between Google-style docstrings (with CooperativeCoding extensions) and canvas node structured markdown. Handles the custom `Responsibility:`, `Pseudo Code:`, `Collaborators:`, and `Constraints:` sections. |
| Sync engine | `ccoding/sync/engine.py` | Orchestrates bidirectional sync between canvas files and Python source. Detects changes on both sides, applies the binding's mapping rules, and flags conflicts for human resolution. |

The reference implementation also includes:

- **Type mapper** (`ccoding/code/types.py`) — translates between canvas language-neutral types and Python type annotations, including generic types and union syntax. *(Planned — not yet implemented)*
- **Import resolver** (`ccoding/code/imports.py`) — generates correct import statements from `ccoding.qualifiedName` values, stereotype requirements, and type annotation dependencies. *(Planned — not yet implemented)*
- **Canvas reader** (`ccoding/canvas/reader.py`) — reads JSON Canvas files into the internal model, preserving all standard fields and `ccoding` metadata.
- **Canvas writer** (`ccoding/canvas/writer.py`) — writes the internal model back to JSON Canvas files, preserving all standard fields and `ccoding` metadata through round-trips.
