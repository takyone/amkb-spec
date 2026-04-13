# AMKB — Agent-Managed Knowledge Base Specification

> **Status: Pre-draft (v0.1.0).** This specification is under active design.
> Breaking changes happen without notice. Not ready for implementation
> outside the reference implementation ([Spikuit](https://github.com/takyone/spikuit)).

AMKB is a protocol for knowledge bases that are **maintained by agents
rather than humans**. In an AMKB, the user acts as the *Data Manager* —
deciding what matters and why — while an agent acts as the *Data Engineer*,
handling ingestion, deduplication, restructuring, and lineage.

This repository holds the **normative specification**. The Python SDK
lives in a separate repository ([amkb-sdk](https://github.com/takyone/amkb-sdk)),
and the reference implementation is [Spikuit](https://github.com/takyone/spikuit).

## Why AMKB?

Traditional RAG makes humans the data engineer: you chunk, you filter,
you re-index, you prune stale entries. AMKB inverts that role. Three
consequences follow:

1. **KB quality scales with model progress, not engineering effort.**
   As frontier models improve, an AMKB-compliant store is better
   maintained — without human intervention.
2. **Retrieval is separated from attestation.** Concepts live in the
   retrieval space; sources live outside it, as immutable truth anchors.
   Traditional RAG collapses these two into one; AMKB separates them.
3. **Every change is auditable.** All mutations are attributed to an
   `Actor`, carry intent, and emit events. The agent's work is a
   human-readable change log.

## Repository Structure

```
amkb-spec/
├── LICENSE                 Apache-2.0
├── README.md               This file
├── spec/                   Normative specification
│   ├── 00-overview.md      What AMKB is, non-goals, audience
│   ├── 01-concepts.md      Node, Edge, Actor, Transaction, Event, Layer
│   ├── 02-types.md         Field-level type and validation rules
│   ├── 03-operations.md    Operation signatures and contracts
│   ├── 04-events.md        Event durability, ordering, retention, subscription
│   ├── 05-errors.md        Error model, categories, canonical codes
│   └── 99-rationale.md     Design decisions and alternatives considered
├── theory/                 Non-normative theoretical companion
│   └── retrieval.md        Retrieval Theory primer (ISF, levels, frames, merge)
├── schema/                 (future) JSON Schema / msgspec types
├── conformance/            (future) Test suite for implementations
└── examples/               (future) Worked examples
```

## Reading Order

1. [Overview](spec/00-overview.md) — context and scope
2. [Concepts](spec/01-concepts.md) — conceptual model
3. [Types](spec/02-types.md) — normative field-level rules
4. [Operations](spec/03-operations.md) — operation signatures and contracts
5. [Events](spec/04-events.md) — event durability, ordering, subscription
6. [Errors](spec/05-errors.md) — error model and canonical codes
7. [Rationale](spec/99-rationale.md) — why it's this way

For theoretical background on the retrieval-related design decisions
(the three-level hierarchy, observation frames, ISF classification,
classical-vs-agentic accounting, and the Hopfield-grounded merge
argument), see [theory/retrieval.md](theory/retrieval.md). This
document is non-normative and may evolve independently of the spec.

## Specification Conventions

This specification uses the terms **MUST**, **MUST NOT**, **SHOULD**,
**SHOULD NOT**, **MAY**, and **OPTIONAL** as defined in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174), when, and only when,
they appear in all capitals.

## Versioning

This specification follows [Semantic Versioning](https://semver.org/).
During the `0.x` series, breaking changes may occur between minor
versions. Implementations should pin to an exact version until `1.0.0`.

## Status

- **Current version**: 0.1.0 (pre-draft)
- **Reference implementation**: [Spikuit](https://github.com/takyone/spikuit)
- **SDK**: [amkb-sdk](https://github.com/takyone/amkb-sdk) (not yet published)

See [spec/99-rationale.md](spec/99-rationale.md) for the design decisions
made at v0.1.0 and the open questions still under discussion.

## License

This specification is licensed under the Apache License, Version 2.0.
See [LICENSE](LICENSE) for the full text.

Implementations of this specification may be licensed separately under
any license of the implementer's choosing.
