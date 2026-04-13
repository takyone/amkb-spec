# 00 — Overview

This document introduces AMKB (Agent-Managed Knowledge Base), describes
its scope, its audience, and its non-goals. It is non-normative; later
chapters carry the normative rules.

## What AMKB Is

AMKB is a protocol for knowledge bases whose maintenance is delegated
to an agent. It defines:

- The **conceptual model** of the knowledge base: what entities exist,
  how they relate, and how changes are expressed.
- The **operations** an agent uses to mutate and query the store.
- The **constraints and invariants** a compliant store must uphold.
- The **events** a store emits so that external observers can audit or
  replicate its state.

AMKB is deliberately thin. It does not prescribe how to embed text,
how to schedule reviews, how to detect communities, or how to rank
retrieval results. Those decisions belong to implementations, not to
the protocol.

## What "Agent-Managed" Means

In a traditional retrieval-augmented generation (RAG) system, a human
operator performs the data-engineering work: chunking source material,
tuning extractors, filtering duplicates, deciding when to re-index,
pruning stale entries. The quality of the knowledge base is bounded by
how much of this work the operator is willing to do.

In an AMKB, those responsibilities shift to an agent. The human user
decides **what matters** and **why** — the data management role.
The agent decides **how** — ingestion strategy, merge decisions,
lineage tracking, restructuring, retirement of obsolete material.

This inversion produces three structural properties that distinguish
AMKB from traditional RAG:

1. **Maintenance cost scales with model capability, not engineering
   effort.** As the agent's underlying model improves, the knowledge
   base is better maintained — without additional human labor.

2. **Attestation is separated from retrieval.** Concept Nodes are
   the target of intent queries; source Nodes attest to them but are
   never returned as hits. The protocol treats "relevance" as
   implementation-defined — embedding similarity, lexical scoring,
   learned rankers, LLM judges, and graph walks are all equivalent
   from the protocol's perspective. Traditional RAG collapses source
   and concept into one; AMKB makes the distinction structural.

3. **Every mutation is attributed and auditable.** All operations
   carry an `Actor` identifier and emit an event record. The
   agent's work is itself a first-class artifact.

## Who This Specification Is For

- **Implementers** building an AMKB-compliant store. The specification
  tells you what your store must do to claim conformance.
- **Agent developers** writing code that manages an AMKB. The
  specification gives you a stable surface against which to program.
- **Tool authors** building observers, visualizers, migration utilities,
  or conformance checkers. The specification gives you a target your
  tool can depend on across implementations.
- **Protocol readers** who want to understand the design choices behind
  AMKB. The rationale chapter explains the "why"; this chapter and the
  concepts chapter explain the "what".

This specification is **not** a user manual for any particular AMKB
implementation. See the relevant implementation's documentation for
installation, configuration, and day-to-day use.

## Relationship to Reference Implementation

[Spikuit](https://github.com/takyone/spikuit) is the reference
implementation of this specification. Spikuit adds features not
covered by the protocol — notably spaced-repetition scheduling,
activation spreading, and community detection — as implementation
extensions.

The existence of these extensions is informative for protocol design
but not part of the specification. A Spikuit-compatible store need
not implement them. Conversely, any AMKB-compliant store can be used
to back an agent that performs AMKB-defined operations, regardless
of whether it also exposes Spikuit-specific features.

## Non-Goals

AMKB does **not** specify any of the following. Implementations are
free to choose, extend, or omit them as they see fit.

### Relevance estimation and ranking

AMKB specifies the **shape** of intent query results (what goes in,
what comes out, what cannot appear). It does **not** specify what
"relevance" means, how it is estimated, what evidence is used, or
how hits are ordered. Two compliant implementations MAY return
different hits in different orders for the same intent query and
both be correct.

> Rationale: the protocol treats all relevance estimators — embedding
> similarity, lexical scoring, graph walks, learned rankers, language-
> model judgment, user-profile-aware retrieval — as equivalent. Freezing
> any one of them in the spec would kill the diversity of implementations
> the protocol is designed to enable.

### Embedding representations and vector indexes

A particular case of the above: AMKB does not define an embedding
format, a vector index format, or a similarity metric. Implementations
are free to use any embedding model (or none), any index structure,
and any metric — or to use no vectors at all. Node content is
specified at the text level; its numerical representation, if any,
is an internal concern.

### Scheduling and activation dynamics

Spaced repetition, forgetting curves, activation spreading,
pressure/decay dynamics, and similar mechanisms are out of scope.
An AMKB is a graph store with lineage and events; it is not a
learning system. Implementations that wish to add scheduling do so
as extensions.

> Rationale: scheduling is the defining feature of Spikuit but is
> not common to all agent-managed knowledge bases. An AMKB backing
> an enterprise compliance system has no use for FSRS.

### Chunking policy

How a source document is broken into concept nodes is the agent's
decision, not the protocol's. AMKB observes only the result:
concept nodes and their `derived_from` edges to source nodes.

> Rationale: chunking strategies vary by content type, domain, and
> downstream use. The agent is better positioned to choose than
> the protocol.

### Authorization and access control

AMKB specifies that every mutation carries an `Actor`, but it does
not specify whether particular actors are permitted to perform
particular operations. Capability and policy layers are reserved
for a future optional conformance level (see Level 5, forthcoming).

> Rationale: single-user local implementations have no need for
> authorization. Enterprise implementations have elaborate needs
> that cannot be captured in a minimal protocol. Leaving this
> layer optional lets both cases be addressed cleanly.

### User interface

AMKB does not specify a CLI, a GUI, an API binding, or a wire
format. The protocol is language-agnostic and transport-agnostic.
Python, TypeScript, and Rust SDKs may all coexist; each binds the
conceptual model to its host language as it sees fit.

### Storage backend

AMKB does not specify how data is persisted. SQLite, PostgreSQL,
graph databases, document stores, and cloud object stores are all
valid backends, provided the implementation upholds the protocol's
invariants.

## Conformance Levels (Preview)

AMKB defines progressive levels of conformance, patterned after LSP.
Full definitions appear in later chapters; the sketch below is for
orientation.

| Level | Name          | Includes                                          |
|-------|---------------|---------------------------------------------------|
| L1    | Core          | Node/Edge CRUD, events, history, revert           |
| L2    | Lineage       | L1 + merge, lineage traversal, diff               |
| L3    | Transactional | L2 + named transactions, atomic commit            |
| L4a   | Structural    | L3 + `get`, `find_by_attr`, `neighbors`           |
| L4b   | Intent        | L4a + `retrieve(intent)` (shape only, not ranking)|
| L5    | Policy        | L4b + actor capabilities, authorization (optional)|

An implementation claims a level by stating it and passing the
corresponding conformance tests. Claiming a level requires passing
all lower levels; cherry-picking is not allowed.

## Versioning and Stability

This specification follows [Semantic Versioning](https://semver.org/).
During the `0.x` series, breaking changes may occur between minor
versions. Implementations should pin to an exact version.

The normative terms of RFC 2119 / RFC 8174 apply throughout this
specification. Keywords **MUST**, **MUST NOT**, **SHOULD**,
**SHOULD NOT**, **MAY**, and **OPTIONAL** are normative only when
written in all capitals.

## Where to Read Next

- [01 — Concepts](01-concepts.md): the conceptual model.
- [02 — Types](02-types.md): field-level normative rules.
- [99 — Rationale](99-rationale.md): design decisions and alternatives.
