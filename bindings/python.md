# CooperativeCoding Python Language Binding

*Reference binding showing how the CooperativeCoding specification maps to Python.*

---

**Target language:** `python`
**Spec version:** 2.0.0
**Phase:** Phase 2
**Status:** Reference binding
**Python version:** 3.11+

This binding is used by the [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python) reference implementation.

---

## 1. Overview

This document describes how the abstract concepts defined in the core spec (code elements, design content, relationships, sync rules) map to Python's type system, documentation conventions, and module structure.

The mapping aims for zero surprise: a Python developer reading the generated code should see idiomatic Python, and a canvas user reading an element should see clean, language-neutral design documentation. The binding sits in the middle, translating between these two views.

Python treats a source file as a first-class module element. Module-level constants and free-standing functions belong to that module. Implementations MAY inline free-standing functions into a module overview when they have no external relationships, or materialize them as separate function elements when they do, but the qualified name and source mapping MUST still round-trip.

Everything described here reflects what the reference implementation does. Other implementations targeting Python are free to make different choices within the spec's degrees of freedom.

---

## 2. Stereotype Mapping

The stereotype on a class element determines which Python construct sync generates. When no stereotype is set, the element maps to a plain class.

| Stereotype | Python Construct | Import Required |
|---|---|---|
| *(none)* | `class Foo:` | none |
| `protocol` | `class Foo(Protocol):` | `from typing import Protocol` |
| `abstract` | `class Foo(ABC):` | `from abc import ABC, abstractmethod` |
| `dataclass` | `@dataclass class Foo:` | `from dataclasses import dataclass` |
| `enum` | `class Foo(Enum):` | `from enum import Enum` |

**Behavioral notes:**

- **`protocol`**: Methods on a protocol class are not decorated with `@abstractmethod`. Protocol uses structural subtyping: any class with matching method signatures satisfies the protocol without explicit inheritance. An `implements` relationship translates to explicit Protocol inheritance in code, which opts into nominal checking.
- **`abstract`**: Methods listed on the canvas that lack a pseudo code section are generated with `@abstractmethod` and an ellipsis body (`...`). Methods with pseudo code are generated as concrete methods.
- **`dataclass`**: Fields are generated as dataclass fields with type annotations. The field order on the canvas determines the field order in the generated dataclass (which affects `__init__` parameter order). Fields with defaults come after fields without.
- **`enum`**: Enum members are represented as constants belonging to the enum class. The type of the constant determines the enum value type (e.g., `str` produces a `class Foo(str, Enum):` mixin). Private members (names starting with `_`) are excluded.

Unknown stereotypes MUST be preserved in design data and UI. The reference Python implementation falls back to plain-class code generation until an explicit mapping is added; it does not reject the element.

---

## 3. Documentation Format

Python uses [Google-style docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings) extended with CooperativeCoding sections. This format is compatible with standard Python documentation tools (Sphinx with `napoleon`, pdoc, mkdocstrings) while carrying the additional design information that the canvas displays.

Three sections are added beyond the standard Google docstring style:

- **`Responsibility:`**: appears on class and method docstrings. Describes what this element owns in the system. This is the most important section for design: it defines the boundaries.
- **`Pseudo Code:`**: appears on method docstrings only. A numbered step-by-step description of the algorithm. Deliberately not real code: it describes what happens, not how in language-specific terms.
- **`Collaborators:`**: appears on class docstrings only. Lists other classes this one works with and why. Derived from relationships connected to the class during sync.

These sections are designed to be safely ignored by standard Python documentation tools that do not recognize them.

### 3.1 Module Docstring

A module docstring maps to the module element's design content on the canvas. It carries the module-level responsibility and any high-level notes about what the file owns. Free-standing functions inside the module use standard function docstrings with CooperativeCoding sections in the same style as methods.

```python
"""Utility functions for parser configuration.

Responsibility:
    Own the module-level helpers for loading, validating, and normalizing
    parser configuration before class-level parsing begins.
"""


def load_config(path: str) -> ParserConfig:
    """Load parser configuration from disk.

    Responsibility:
        Read, validate, and return parser configuration from the supplied path.

    Raises:
        ConfigError: If the file cannot be parsed.
    """
```

### 3.2 Class Docstring

A class docstring maps to the class element's design content on the canvas. The first line is the one-line summary. The `Responsibility:` block defines what the class owns. `Collaborators:` captures relationship-derived information in prose. `Attributes:` maps to the class's fields.

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

### 3.3 Method Docstring

A method docstring maps to the method element's design content. The `Pseudo Code:` section is the algorithm description. Standard Google-style sections (`Args:`, `Returns:`, `Raises:`) map to the signature information.

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

### 3.4 Field Documentation

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

## 4. Canvas Content Format

The reference implementation uses structured markdown for element content on the canvas. This is not part of the core spec (design content format is up to each implementation), but is documented here for interoperability with the reference implementation.

### Underscore Escaping

Python identifiers frequently contain underscores (`_`) which can trigger unintended markdown formatting (e.g., `__init__` renders as bold text). The reference implementation escapes all underscores in identifier names within canvas content: `__init__` becomes `\_\_init\_\_`, `my_method` becomes `my\_method`.

### 4.1 Module Element

A module element represents one Python source file. It carries the module's qualified name, module-level responsibility, imports, constants, and any free-standing functions that belong to that file.

For object-oriented modules, the reference implementation uses a module overview element plus one child class element per exported class. For functional modules, the module element is self-contained and includes the free-standing functions inline.

```markdown
## parsers.config

### Responsibility
Own configuration loading, validation, and normalization helpers.

### Imports
- pathlib.Path
- yaml.safe_load

### Constants
- DEFAULT_TIMEOUT: Integer = 30

### Functions
- load_config(path: String) -> ParserConfig
  > Read and validate parser configuration from disk
- merge_overrides(base: ParserConfig, overrides: Map<String, Any>) -> ParserConfig
  > Apply runtime overrides to a base configuration
```

### 4.2 Class Element

A class element contains the class-level information: name, responsibility, and constraints. Members (methods, fields, constants) that have no connections outside their parent class are inlined into the class body rather than existing as separate elements. Only members with external connections get their own canvas elements.

An inlined class element uses `### Fields`, `### Methods`, and `### Constants` subsections:

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

### 4.3 Method Element

A method element contains the fully qualified method name, a responsibility section, the signature broken into IN/OUT/RAISES, and numbered pseudo code.

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
- `*args` is represented as `*args: type` (or `*args` if untyped)
- `**kwargs` is represented as `**kwargs: type` (or `**kwargs` if untyped)
- Keyword-only arguments (after `*`) are represented with a `*` separator
- Positional-only arguments (before `/`) are represented with a `/` separator

Round-trip fidelity: these special forms MUST survive canvas to code to canvas without loss.

### 4.4 Field Element

A field element contains the fully qualified field name, a responsibility section, the type, and any constraints.

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

### 4.5 Test Element

A test element contains the test class name, what is being tested, the test methods, and results.

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

Given a test element connected to a `DocumentParser` class, sync generates:

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

- The relationship to `DocumentParser` produced the `from parsers.document import DocumentParser` import, derived from the target element's qualified name.
- Test method names carry the `test_` prefix required by pytest discovery.
- Each test method's docstring comes from the pseudo code description in the test element.
- Method bodies are `raise NotImplementedError`. The reference implementation does not use `pass` or `...` because a passing test with no assertions is misleading.
- Return type annotations are `-> None` for all test methods.

**Stub detection during sync:** A method body is considered a stub if it consists solely of:
- `raise NotImplementedError` (or `raise NotImplementedError(...)`)
- `pass`
- `...` (Ellipsis)
- A single docstring with no other statements

Any other body content is considered implemented and MUST NOT be overwritten during sync. When the canvas changes a method's signature, sync MUST update the signature while preserving the existing body.

### Test Result Format

The sync reads test results from pytest's output. The reference implementation supports two result sources:

- **pytest JSON report** (via the `pytest-json-report` plugin).
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

## 7. Relationship Interpretation

The reference implementation tracks the following relationships and maps them to Python code constructs:

| Relationship | Code Construct | Label Interpretation |
|---|---|---|
| Inheritance | `class Child(Parent):` declaration | N/A |
| Protocol implementation | `class Impl(Protocol):` declaration | N/A |
| Composition (has-a) | Typed field declaration | Label before `:` or ` - ` is the field name; remainder is descriptive. If no label, field name is derived from the target element name (lowercased, snake_cased). |
| Dependency (uses) | Import statement | N/A |
| Membership | Method/field belongs to class body | N/A |
| Testing | Import of class under test | N/A |

For all other relationships, labels are informational and not used during code generation.

---

## 8. Example Round-Trip

This section walks through a complete cycle: a class element on the canvas, the Python code generated from it, an edit made in code, and the resulting canvas update.

### 8.1 Canvas (Starting Point)

A canvas contains a `DocumentParser` class element with a separate method element for `parse`. The class has stereotype `protocol`.

The class element's design content:

```markdown
## DocumentParser

> Responsible for parsing raw documents into structured AST nodes

### Constraints
- Thread-safe for concurrent parsing
```

The method element's design content:

```markdown
## DocumentParser.parse

### Responsibility
Transform raw source into a validated AST, applying all registered plugins.

### Signature
- **IN:** source: `String` -- raw document text
- **OUT:** `AST` -- parsed syntax tree
- **RAISES:** `ParseError` -- on malformed input

### Pseudo Code
1. Check _cache for source hash
2. If cached, return cached AST
3. Tokenize source using config.tokenizer
4. Build raw AST from token stream
5. Validate final AST structure
6. Cache and return
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
- Canvas type `String` became `str` via the type hint mapping.
- The `parse` method has the full docstring with `Responsibility:` and `Pseudo Code:` sections.
- Method body is `...` (ellipsis), the protocol stub. For non-protocol classes, the reference implementation generates `raise NotImplementedError`.

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

The code change produces a design request. Sync updates the method element on the canvas:

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

The full working implementation is available at [cooperative-coding-python](https://github.com/giosullutrone/cooperative-coding-python). The repository contains the sync logic, code parser, code generator, and docstring mapping logic that this binding describes.
