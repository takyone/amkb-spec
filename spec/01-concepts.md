# 01 — Concepts

This document defines the conceptual model of AMKB. It introduces the
five first-class concepts of the protocol, the layer structure that
organizes them, and the relationships between them. Field-level
normative rules appear in [02 — Types](02-types.md).

This document is conceptual. Wherever it uses "must" or "cannot" in
lowercase, treat the statement as a narrative summary of a normative
rule stated formally in Chapter 02.

## The Five First-Class Concepts

AMKB defines exactly five first-class concepts. Everything else —
sources, concepts, categories, ingestion batches — is expressed in
terms of these five.

| Concept      | One-line definition                                     |
|--------------|---------------------------------------------------------|
| Node         | A unit of knowledge or content in the store.            |
| Edge         | A typed relationship between two Nodes.                 |
| Actor        | An entity that causes mutations to the store.           |
| Transaction  | A named, atomic group of mutations.                     |
| Event        | The externalized, append-only record of a Transaction.  |

These five are sufficient to describe an agent-managed knowledge base.
If a proposed feature cannot be expressed in terms of Nodes, Edges,
Actors, Transactions, and Events, it belongs to an implementation, not
to the protocol.

### Node

A **Node** is a unit of knowledge. Every Node has:

- A stable **reference** (`NodeRef`) that does not change for the
  lifetime of the Node, even across content rewrites.
- A **kind**, an identifier drawn from an open enumeration, which
  describes what role the Node plays. The protocol reserves a small
  set of canonical kinds (`source`, `concept`, `category`);
  implementations MAY define more.
- A **layer** that locates the Node in the retrieval architecture.
  Layers are described later in this document.
- A **content** string — the human-readable payload.
- A set of typed **attributes** — key/value pairs that carry metadata
  the agent or the implementation finds useful.
- A **lifecycle state**: live or retired. Retired Nodes are not
  deleted; they are tombstoned so that lineage is preserved.

The combination of `kind` and `layer` determines how a Node
participates in retrieval, lineage, and cross-layer traversal. A Node
with `kind="source"` and `layer="L_source"` is an attestation
artifact; a Node with `kind="concept"` and `layer="L_concept"` is a
retrieval target.

### Edge

An **Edge** is a typed, directed relationship between two Nodes.
Every Edge has:

- A stable reference (`EdgeRef`).
- A **relation** type (`rel`), drawn from an open enumeration.
- A **source** Node and a **destination** Node.
- Attributes (like Nodes).
- A lifecycle state (live or retired).

Edge semantics depend on the relation type. The protocol reserves a
small set of relation types with fixed meaning — `derived_from`,
`attested_by`, `contradicted_by`, `superseded_by`, `contains`,
`belongs_to`, `generalizes`, `requires`, `extends`, `contrasts`,
`relates_to` — and leaves others to implementations.

Some relations are directional by nature (`derived_from` always points
from a concept to its source). Others are symmetric in meaning
(`contrasts`, `relates_to`) even though they are stored as directed
edges. The directionality rules appear in Chapter 02.

### Actor

An **Actor** is an entity that causes mutations to the store. Every
operation that changes the store carries the identifier of an Actor.

An Actor has:

- A stable identifier (`ActorId`), opaque to the protocol.
- A **kind**, drawn from an open enumeration. Canonical values include
  `llm`, `human`, `automation`, and `composite`.
- A **profile**: implementation-defined metadata (e.g., model name,
  user identifier, script name).

The protocol does not prescribe how Actors are authenticated or
authorized. It only requires that mutations be attributed. Whether
a particular Actor is allowed to perform a particular operation is
left to an optional future conformance level (Level 5).

> **Why "Actor" and not "Agent"?** The word "Agent" is the current
> branding, but the underlying abstraction is timeless: every mutation
> has a cause, and that cause has an identity. Twenty years from now
> the dominant kind of agent may be something unrecognizable today;
> the `Actor` type remains because it names the abstraction, not the
> fashion. The protocol name (AMKB = Agent-Managed Knowledge Base) is
> historical; the internal type name is neutral. See Rationale.

### Transaction

A **Transaction** is a named, atomic group of mutations.

Every mutation to an AMKB store happens within some transaction.
Single-operation convenience APIs implicitly begin and commit a
transaction around the operation.

Transactions have three key properties:

1. **Atomicity.** When a transaction commits, all of its mutations
   take effect together. When a transaction aborts (or fails to
   commit), none of its mutations take effect.
2. **Naming.** Transactions carry a `tag` — a human-readable
   identifier chosen by the Actor — which makes a batch of related
   changes findable and revertable as a unit.
3. **Actor attribution.** A transaction carries the Actor that opened
   it. All mutations within the transaction inherit this attribution.

Transactions are the unit of revert. Given a transaction tag,
the store can produce an inverse transaction that undoes the batch
(see Chapter 03).

> Ingestion — the process of reading a source and extracting concepts
> — is not a protocol operation. It is a transaction pattern: open a
> tagged transaction, create a source Node, create concept Nodes,
> link them with `derived_from`, commit. This pattern is documented
> in the examples directory.

### Event

An **Event** is the externalized record of a committed transaction.

When a transaction commits, the store produces one or more Events
that describe what changed. Events are append-only, ordered (at
least causally), and available through a subscription mechanism.

Events enable:

- **Audit.** What changed, when, and by whom — answerable from the
  event log without inspecting store state.
- **Replication.** A second store can be brought up to date by
  replaying events.
- **Observability.** External tools (dashboards, visualizers,
  compliance systems) can subscribe without depending on the
  store's internal structure.
- **Revert.** A tagged transaction can be reverted by applying the
  inverse of its event set.

The protocol specifies the shape of events and minimum durability
guarantees (Chapter 04). It does not specify a wire format, a
transport, or a storage medium.

## Layers: Where Nodes Live

The Nodes in an AMKB are organized into **layers**. A layer is a
logical grouping of Nodes with shared retrieval behavior. The
protocol defines three canonical layers:

| Layer       | Purpose                                                |
|-------------|--------------------------------------------------------|
| `L_source`  | Attestation layer. Sources of truth.                   |
| `L_concept` | Retrieval layer. The primary target of semantic query. |
| `L_category`| Aggregate layer. Coarse-grained, navigable.            |

An implementation MUST provide `L_concept`. The other layers are
OPTIONAL but, if provided, MUST follow the semantics described here.

### `L_source` — Attestation Layer

The source layer holds Nodes that represent attested material:
documents, papers, web pages, datasets, conversations. Source Nodes
carry pointers to their original content (URL, path, blob reference)
and metadata about how they were acquired (timestamp, extractor
version, content hash).

**Source Nodes are not in the retrieval space.** Intent queries
never return source Nodes. Source Nodes are reached only through
structural traversal — typically from a concept Node along a
`derived_from` or `attested_by` edge.

This is the central architectural choice of AMKB. Traditional RAG
treats source chunks as both storage and retrieval targets; AMKB
separates the two. Source Nodes are the stable truth anchors for
concepts; concept Nodes are what the agent and user actually query.

### `L_concept` — Retrieval Layer

The concept layer holds Nodes that represent atomic units of
knowledge. A concept is a self-contained statement, definition, or
explanation — the granularity at which the agent finds it useful to
retrieve and reason.

**Concept Nodes are the primary target of intent retrieval.**
Implementations MAY add additional retrieval-space kinds, but
concept is always retrievable.

Concept Nodes are typically created by an agent during ingestion:
the agent reads a source, extracts atomic claims, and emits a concept
Node for each. The concept is linked to its source via `derived_from`.

### `L_category` — Aggregate Layer

The category layer is optional. When present, it holds Nodes that
aggregate or generalize concepts. A category is a coarse-grained
pointer into a region of the concept layer: "linear algebra",
"observability", "Rust ownership".

Category Nodes enable two things:

1. **Two-stage retrieval.** An agent can retrieve categories
   first and then descend into their concepts, reducing the
   search space from thousands of concepts to a handful of
   categories.
2. **Meta-structure.** Relations between categories
   (`generalizes`, `relates_to`) represent high-level knowledge
   architecture without polluting the concept layer.

Categories contain concepts via `contains` edges. A concept belongs
to at most one category via `belongs_to`; implementations MAY relax
this to allow overlapping category membership.

### Cross-Layer Edges

Edges may cross layers. Cross-layer edges have a fixed meaning:

| Relation        | Direction                     | Purpose                       |
|-----------------|-------------------------------|-------------------------------|
| `derived_from`  | L_concept → L_source          | Concept was extracted from S. |
| `attested_by`   | L_concept → L_source          | Source supports the concept.  |
| `contradicted_by` | L_concept → L_source        | Source contradicts the concept. |
| `superseded_by` | L_source → L_source (intra)   | Newer source replaces older.  |
| `contains`      | L_category → L_concept        | Category groups concept.      |
| `belongs_to`    | L_concept → L_category        | Inverse of contains.          |
| `generalizes`   | L_category → L_category       | Category subsumes category.   |

## Lineage

**Lineage** is the record of how a Node came to be. Every mutation
that produces a Node (creation, rewrite, merge) adds to that Node's
lineage.

Lineage serves two purposes:

1. **Provenance.** Given a Node, the agent or a human auditor can
   trace back to the Nodes and sources that produced it.
2. **Revert safety.** When a mutation is reverted, lineage lets the
   store restore any Nodes that were merged or rewritten as part
   of the mutation.

Lineage is never broken by retirement. A retired Node's lineage is
still readable, and Nodes that reference the retired Node through
lineage continue to work.

Merges are the most complex lineage operation. When Nodes A and B
are merged into C, A and B are retired and C records both as
ancestors. Later operations on C can trace the merge; a revert of
the merge can restore A and B.

## The Retrieval Space

The **retrieval space** is the set of Nodes that can be returned by
intent query operations.

The protocol defines the retrieval space by negative constraint:

- Nodes with `kind="source"` MUST NOT appear in the retrieval space.
- Nodes with `kind="concept"` MUST appear in the retrieval space.
- Other kinds MAY or MAY NOT appear, at the implementation's
  discretion, provided the above two rules hold.

This negative specification is deliberate. The protocol does not
prescribe what the retrieval space *is*; it prescribes what cannot
be in it. The forbidden inclusion of sources is what distinguishes
an AMKB from a traditional source-indexed RAG.

Retrieval results are always ordered, but the ordering criterion is
implementation-defined. Two AMKB stores MAY return different results
for the same intent query and both be compliant. Conformance tests
check the *shape* of retrieval (what can and cannot appear), not the
*quality* of ordering.

## Structural vs Intent Query

AMKB offers two query modalities that differ in their determinism
guarantees.

**Structural query** is deterministic. `get`, `find_by_attr`, and
`neighbors` return exactly the Nodes or Edges that match their
criteria. Two compliant implementations MUST return the same answers
to the same structural query (modulo result ordering, which MAY
differ).

**Intent query** is non-deterministic. `retrieve(intent, k)` returns
up to `k` concept Nodes ordered by the implementation's estimate of
their **relevance** to the given `intent`. The protocol does not
define what "relevance" means, how it is computed, or what evidence
the implementation uses to estimate it. An implementation MAY use
embedding similarity, lexical similarity, graph structure, learned
rankers, language-model judgment, or any combination — the protocol
treats all of these as equivalent "relevance estimators".

Two compliant implementations MAY return different Nodes in different
orders for the same intent query. Both are correct: the protocol
specifies only the *shape* of the answer, not the *criterion* used
to produce it.

> The informal term "semantic query" is sometimes used for intent
> query. This specification prefers "intent" because it does not
> presuppose any particular estimation strategy (embedding-based,
> lexical, or otherwise).

Agent developers should use structural queries when they need exact
answers (lineage traversal, audit, specific-ID lookup) and intent
queries when they want the implementation to surface concepts that
are plausibly relevant to an open question.

## The Actor Model in Practice

Every Node and every Edge carries a trail of Actor attribution in
its lineage. This trail answers two questions simultaneously:

- **Authorship.** Which Actor created this, and which Actors have
  modified it?
- **Responsibility.** Which Actor's decisions led to the current state
  of the graph?

A typical AMKB has at least two Actor kinds active:

- A **`llm`** Actor for the agent performing ingestion, merging,
  and curation. Multiple `llm` Actors may coexist (a strong model
  for curation, a cheap model for routine tagging).
- A **`human`** Actor for the user, used when the user directly
  edits or confirms a curator proposal.

Automation pipelines, ingest jobs, and test fixtures typically use
an `automation` Actor. Composite Actors (e.g., "llm-with-human-review")
use `composite` or an implementation-defined kind.

## Summary

An AMKB is a graph of Nodes and Edges, organized into layers,
mutated by Actors within tagged Transactions, observable through
Events. Source Nodes attest, concept Nodes retrieve, category Nodes
aggregate. Lineage is preserved through retirement. Retrieval is
separated from attestation.

This much is the model. The next chapter gives it teeth.
