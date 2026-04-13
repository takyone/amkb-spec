# 99 — Rationale

This document records the design decisions behind AMKB and the
alternatives that were considered and rejected. It is non-normative.
Its purpose is to let future readers — and future versions of this
specification — understand *why* the protocol is shaped the way it is.

Each entry follows the same structure: the decision, the alternatives
considered, and the reason for the choice. Entries are ordered
roughly by influence on the protocol's shape.

## R1. Source is a Node kind, not a first-class type

**Decision.** Sources are represented as Nodes with `kind="source"`
and `layer="L_source"`, not as a separate first-class type.

**Alternatives considered.**

- **A: Source as an attribute of Node.** Each Node carries a
  `source_uri` attribute. Rejected: loses source metadata, prevents
  source-to-source relationships, makes lineage partial.
- **B: Source as a first-class type alongside Node and Edge.**
  The protocol would define a third top-level concept. Rejected:
  increases the type surface, duplicates CRUD operations, breaks
  the "everything is a graph element" principle.
- **C: Source as a Node kind (chosen).** A source is a Node like any
  other, with a specific `kind` and `layer`. Lineage is expressed
  as edges. CRUD is shared.

**Reasons for C.**

1. Keeps the protocol minimal: two first-class graph types, not three.
2. Lineage (`derived_from`, `attested_by`, `superseded_by`) becomes
   natural graph traversal, not a special case.
3. Agents learn one CRUD API and apply it uniformly.
4. Source-to-source relationships (versioning, translation, citation)
   fall out for free.
5. Implementers may still back sources with a separate physical table
   — the protocol concerns the logical model, not the storage layout.

## R2. Sources are not in the retrieval space

**Decision.** Semantic queries MUST NOT return Nodes with
`kind="source"`.

**Alternatives considered.**

- **A: Allow sources in retrieval if the implementation chooses.**
  Rejected: makes AMKB indistinguishable from traditional RAG and
  removes the protocol's defining property.
- **B: Forbid sources unconditionally (chosen).**

**Reasons for B.**

The central architectural claim of AMKB is that *attestation is
separated from retrieval*. Traditional RAG conflates the two: a
chunk of source text is both what you index and what you retrieve.
AMKB treats concepts as the retrieval target and sources as the
immutable truth anchors behind them. If source Nodes can appear in
retrieval results, this separation collapses and the protocol loses
its reason to exist.

This rule is the single MUST-level constraint that most directly
distinguishes an AMKB from alternative designs. Implementations that
want source retrieval are not implementing AMKB; they are implementing
a different, compatible protocol.

## R3. The actor type is named "Actor", not "Agent"

**Decision.** The type that identifies the cause of a mutation is
named `Actor`, not `Agent`, even though the protocol itself is called
"Agent-Managed Knowledge Base".

**Alternatives considered.**

- **A: Name the type `Agent` to match the protocol name.** Rejected:
  the word "Agent" is a current industry branding that may not
  survive the protocol. Twenty years from now, the dominant kind of
  autonomous software that maintains knowledge bases may be called
  something else entirely (swarm, process, co-pilot, BCI). The
  abstraction — "the thing that caused this mutation" — is timeless;
  the branding is not.
- **B: Name the type `Actor` (chosen).**

**Reasons for B.**

1. `Actor` is a well-established term in data governance, distributed
   systems, security, and audit literature. It is neutral with respect
   to the nature of the causing entity.
2. It keeps `kind="llm"`, `kind="human"`, `kind="automation"`, and
   future kinds all first-class, instead of privileging one.
3. The protocol name (AMKB) remains historically accurate: at the
   time of v1.0, the dominant actor kind managing these knowledge
   bases is in fact an LLM agent. But the type name does not need
   to die when that fact changes.

## R4. Ingestion is not a protocol operation

**Decision.** There is no `ingest` operation in the protocol. Ingesting
a document is a *transaction pattern*: open a tagged transaction,
create a source Node, create concept Nodes, link them with
`derived_from`, commit.

**Alternatives considered.**

- **A: Define `ingest(source, extractor, ...)` as a primitive.**
  Rejected: the protocol would have to know about extractors,
  chunking strategies, and retry policies — none of which are its
  business.
- **B: Ingest as a transaction pattern (chosen).**

**Reasons for B.**

1. Chunking and extraction are implementation and domain concerns.
   The protocol sees only the result.
2. The transaction + tag machinery already handles the "atomic batch
   that can be reverted" requirement. Adding `ingest` would duplicate
   this.
3. Different ingestion pipelines (PDF, URL, conversation, image)
   need different internal structure. Forcing them through a single
   operation would either limit them or require an elaborate options
   argument.
4. Documenting ingest as a pattern (in the examples directory) keeps
   the protocol small while still giving implementers a canonical
   shape to follow.

## R5. Embedding representations are out of scope

**Decision.** The protocol does not define embedding formats, vector
indexes, or similarity metrics. Implementations MAY store embeddings
internally, but not under canonical keys in `Node.attrs`.

**Alternatives considered.**

- **A: Standardize an embedding field.** Rejected: models and metrics
  evolve independently; portability across embedders is impossible
  anyway (different dimensions, different spaces).
- **B: Exclude embeddings entirely (chosen).**

**Reasons for B.**

1. Embedding is a moving target; freezing it in the spec guarantees
   the spec goes stale.
2. Vector stores (Chroma, pgvector, LEANN, sqlite-vec) have their
   own evolving APIs. Fitting a protocol over them is unnecessary
   coupling.
3. Graph-weighted retrieval (the Spikuit approach) combines vector
   similarity with graph structure; the graph is in the protocol,
   the vectors are in the implementation. This division preserves
   both.
4. Migration between implementations requires embedding recomputation
   regardless — no portability is lost by excluding embeddings.

## R6. Retire, not delete

**Decision.** Nodes and Edges are retired (tombstoned), not
physically deleted, in all protocol-level operations. Physical
deletion, if an implementation chooses to offer it, is outside
the protocol.

**Alternatives considered.**

- **A: Physical delete as the default.** Rejected: breaks lineage,
  breaks revert, breaks audit.
- **B: Retire-by-default, delete-by-opt-in at protocol level.**
  Rejected: too much surface for a corner case.
- **C: Retire-only at protocol level; delete is implementation
  concern (chosen).**

**Reasons for C.**

1. Lineage preservation is a cornerstone of AMKB's value proposition.
   Physical delete would require either breaking lineage edges or
   leaving dangling references — both are worse than tombstones.
2. Revert is simpler: undoing a retirement is just flipping state.
   Undoing a physical delete would require restoring all the content
   and re-emitting the original creation events.
3. Storage pressure is an implementation concern. Implementations
   that need to reclaim space can implement physical delete as an
   admin operation outside the protocol contract.

## R7. Events as source of truth for observers

**Decision.** External observers (UI, monitoring, replication,
audit) consume events, not store state.

**Alternatives considered.**

- **A: Observers poll the store for state.** Rejected: requires
  observers to implement diff logic; scales poorly; loses intent.
- **B: Observers subscribe to events (chosen).**

**Reasons for B.**

1. Events carry intent (`actor`, `reason`, `tag`), which state does not.
2. Event streams are append-only, which matches how observers
   naturally want to process changes.
3. Replication is event sourcing; observing and replicating become
   the same mechanism.
4. A store that cannot emit events cannot meet AMKB-L1, so observer
   implementations can depend on the event channel being present.

## R8. Retrieval is specified by shape, not by quality

**Decision.** The conformance test suite checks only the *shape* of
retrieval results (what kinds can and cannot appear, how many, in
what structure). It does not check ranking quality.

**Alternatives considered.**

- **A: Define a retrieval quality benchmark.** Rejected: would force
  all implementations toward a single retrieval strategy, killing
  the intended diversity of implementations.
- **B: Leave retrieval entirely unspecified.** Rejected: observers
  and agents need to know at least what kind of answers come back.
- **C: Specify shape only (chosen).**

**Reasons for C.**

1. Retrieval quality is where implementations should compete. The
   protocol should not pick a winner.
2. Quality benchmarks belong in a separate project (`amkb-bench`),
   which can evolve faster than the spec.
3. Shape constraints are sufficient for agents and observers to
   program against.

## R9. Embedding durability is implementation-defined

**Decision.** The protocol does not require events or state to carry
embedding vectors; only `content_hash` is required so external
indexes can detect staleness.

**Rationale.** External vector stores (Chroma, pgvector, ...) are
often durable on their own terms. Requiring the AMKB store to also
hold embeddings would force redundant storage and forbid
implementations that separate graph from vectors. Content hashes
are sufficient for invalidation.

## Open Questions (for v0.2 and beyond)

The following questions are not yet decided and will shape future
versions of the spec.

### Q1. Query DSL

How much of structural query should be standardized? Chapter 02
implies `get_by_id`, `find_by_attr`, and `neighbors` at L4a, but
the full argument shape is not yet fixed. Work in progress: draft
a minimal Query type in Chapter 03 (not yet written).

### Q2. Actor authentication and capability

Chapter 01 defers all authorization to an optional L5. What is the
right shape for L5 policy — capability tokens, role claims, ACLs,
or something else? Enterprise implementations need this; single-user
implementations do not. L5 should be completely optional and should
not affect L1-L4b implementations at all.

### Q3. Error model

The full error model — codes, categories, retry hints, partial
failure semantics — is mentioned in Chapter 02 but not yet written
up as its own chapter. Planned for v0.2.

### Q4. Conformance test suite

The test suite itself — runner shape, required test cases, level
claims — is planned for a `conformance/` directory. Planned for v0.2.

### Q5. Schema evolution

How backward-compatible must future spec versions be? The
working answer is "not at all during 0.x, strict after 1.0",
with migration tools expected to fill the gap. This is a
pragmatic choice for an early-stage spec and may be revisited.

### Q6. Category / Domain separation

Chapter 01 introduces categories (as Nodes) and domains (as
attributes). The exact relationship between them — can a category
span multiple domains? must it be confined to one? — is not yet
normatively specified. This will be settled when the conformance
tests for L_category are written.

### Q7. Summary nodes

The Spikuit reference implementation uses `kind="summary"` for
community-level summaries. Is this a distinct canonical kind, a
subtype of `category`, or an implementation-defined extension?
Not yet decided.

## History

- **v0.1.0 (pre-draft).** Initial release. Establishes Node/Edge/Actor/
  Transaction/Event as the five first-class concepts, defines
  the L_source / L_concept / L_category layer system, reserves
  core kinds and relations, forbids sources from the retrieval
  space. Error model, conformance suite, and operations chapter
  remain to be written.
