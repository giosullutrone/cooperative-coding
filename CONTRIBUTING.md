# Contributing to CooperativeCoding

CooperativeCoding is an open initiative. Whether you want to refine the spec, add a language binding, or report an ambiguity, your contribution is welcome.

## Proposing Spec Changes

Spec changes should be discussed before implementation:

1. **Open an issue** describing the change, its motivation, and any impact on existing sections.
2. Once there is consensus, submit a pull request against the relevant `spec/*.md` file.
3. Prefer backwards-compatible changes. If a change would break existing conforming implementations, call that out explicitly in the issue so it can be evaluated.

## Adding a Language Binding

To add a binding for a new language:

1. Read [spec/04-language-bindings.md](spec/04-language-bindings.md) — it defines the contract every binding must satisfy.
2. Use [bindings/python.md](bindings/python.md) as a structural template.
3. Create `bindings/<language>.md` with the mapping rules for your language.
4. Submit a pull request. The PR description should note which parts of the contract are covered and any language-specific decisions you made.

## Reporting Issues

Use GitHub Issues to report:

- Errors or contradictions in the spec
- Ambiguous definitions that could lead to divergent implementations
- Missing definitions that a conforming implementation would need

Include a reference to the specific section (document and heading) so the issue is easy to locate.

## Style Guide

- **Spec documents** (`spec/*.md`) use a hybrid style: narrative prose combined with normative keywords per [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) (MUST, SHOULD, MAY). Every normative requirement should be testable.
- **Binding documents** (`bindings/*.md`) are informative. They describe how the spec maps to a specific language. Normative keywords are not required.

## Code of Conduct

Be respectful. Focus on the ideas, not the people.
