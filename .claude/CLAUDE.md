# amkb-spec — Development Guide

## Overview

This repository holds the **normative specification** for AMKB
(Agent-Managed Knowledge Base), a protocol for knowledge bases
maintained by agents rather than humans. The spec is the source
of truth; implementations (including the Python SDK and Spikuit)
conform to it.

This repository contains **documents, not code**. There is no build
step, no runtime, no tests against an implementation. Quality is
measured by clarity, internal consistency, and the ability of
implementers to produce conformant systems from the text alone.

## Repository Structure

```
amkb-spec/
├── LICENSE                 # Apache-2.0
├── README.md               # Entry point for readers
├── .claude/CLAUDE.md       # This file
├── spec/                   # Normative specification documents
│   ├── 00-overview.md      # Scope, audience, non-goals
│   ├── 01-concepts.md      # Conceptual model
│   ├── 02-types.md         # Field-level normative rules
│   └── 99-rationale.md     # Design decision log
├── schema/                 # (future) JSON Schema, reference types
├── conformance/            # (future) Test suite for implementations
└── examples/               # (future) Worked examples
```

## Writing Conventions

### Normative language

Specification documents use RFC 2119 / RFC 8174 keywords:
**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, **OPTIONAL**.

These terms are normative **only when written in all capitals**. Lowercase
uses ("you should consider") are non-normative narrative.

Every normative statement MUST be testable in principle. If a statement
cannot be checked by a conformance test (even a manual one), rewrite it
as non-normative prose or drop it.

### Document structure

- Each document starts with a one-paragraph summary of what it covers.
- Use level-2 headings (`##`) for top-level sections.
- Put non-normative commentary in blockquotes or explicitly marked
  "Non-normative" sections.
- Prefer examples inline rather than in separate files, until the
  examples become large enough to warrant their own file.

### Language

- Normative text: **English**.
- Non-normative commentary, rationale, and examples: **English**.
- Issue / discussion threads: English or Japanese both acceptable.

### Tone

- Prefer precise over casual. "A Node has an `id`" not "Nodes come with ids".
- Define terms before using them. First use of a concept in a document
  should include a definition or a link to the definition.
- Avoid unexplained acronyms. AMKB is fine; everything else should be
  spelled out on first use.

## Commit Message Conventions

Use prefixes so history is scannable:

```
spec:     Change to spec/ documents (most common)
schema:   Change to schema/ files
conformance:  Change to conformance/ tests
example:  Change to examples/
docs:     README, CLAUDE.md, meta-documentation
chore:    Tooling, gitignore, license, etc.
```

Examples:

```
spec: initial 00-overview.md draft
spec: add Actor kind reservation in 02-types.md
docs: clarify reading order in README
chore: add Apache-2.0 LICENSE
```

Breaking changes should be marked with `!` after the prefix:

```
spec!: rename Actor.kind values, drop "agent" in favor of "llm"
```

## Versioning

Follow Semantic Versioning. During the `0.x` series, breaking changes
are allowed between minor versions and SHOULD be called out in commit
messages with the `!` suffix.

The `spec/` directory as a whole represents a single version. Version
is tracked in README and in git tags (`v0.1.0`, `v0.2.0`, ...).

## Design Principles (internal)

When in doubt, fall back to these principles:

1. **Keep the protocol minimal.** Every added concept, field, or
   operation raises the bar for implementers. Prefer subtraction.
2. **Distinguish MUST from SHOULD carefully.** MUSTs block conformance;
   SHOULDs guide quality. Conservative spec authors err toward SHOULD.
3. **Design for the agent that maintains the KB, not the human who
   reads it.** Human-friendly debugging belongs in implementations;
   agent-friendly contracts belong in the spec.
4. **Non-goals are first-class.** If you catch yourself explaining
   "AMKB does not specify X because Y", put Y in 00-overview under
   Non-Goals. It saves every future reader the same question.
5. **Rationale belongs in 99-rationale.** Normative text should state
   the rule. The reason lives in rationale, cross-referenced.

## Out of Scope for this Repository

- Python SDK implementation → [amkb-sdk](https://github.com/takyone/amkb-sdk)
- Reference implementation → [Spikuit](https://github.com/takyone/spikuit)
- Benchmarks and evaluation → future `amkb-bench` (not yet planned)
- Ecosystem tooling (CLI, visualizers) → not part of spec

If a proposed change touches code, it belongs in one of those repos,
not here.
