# 02 — Types

This document gives field-level normative rules for the five
first-class concepts defined in [01 — Concepts](01-concepts.md).
Statements in this document use RFC 2119 keywords (**MUST**,
**MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, **OPTIONAL**) in
their normative sense.

Notation below uses a Python-like syntax for clarity. The normative
content is in the prose, not in the syntax; implementations in other
languages MUST preserve the semantics, not the syntactic form.

## 2.1 References

A **reference** is an opaque, stable identifier for a Node or an Edge.

- `NodeRef` — identifier for a Node.
- `EdgeRef` — identifier for an Edge.
- `ActorId` — identifier for an Actor.
- `TransactionRef` — identifier for a Transaction.
- `ChangeSetRef` — identifier for a committed ChangeSet.
- `Timestamp` — a point in time, monotonic within a single store.

### Stability

- A `NodeRef` MUST remain valid for the lifetime of its Node,
  including across content rewrites and attribute updates.
- A `NodeRef` MUST NOT be reused after the Node is retired.
- An `EdgeRef` MUST remain valid for the lifetime of its Edge.
- A retired Node or Edge MUST remain resolvable by its reference;
  lookup returns the tombstoned entity.

### Opacity

- `NodeRef` and `EdgeRef` MUST be treated as opaque by callers.
  Callers MUST NOT parse them, derive structure from them, or assume
  a particular format.
- Implementations MAY use any internal format for references
  (integers, UUIDs, URIs, content-addressed hashes) provided the
  stability and opacity rules above are upheld.

### Equality

- Two references are equal if and only if they resolve to the same
  underlying entity. Implementations MUST provide a deterministic
  equality check on references.

## 2.2 Node

```
Node:
    ref:       NodeRef          # stable identifier (assigned on create)
    kind:      str              # e.g., "concept", "source", "category"
    layer:     str              # e.g., "L_concept", "L_source", "L_category"
    content:   str              # human-readable payload
    attrs:     dict[str, Any]   # typed attributes
    state:     NodeState        # "live" or "retired"
    created_at: Timestamp
    updated_at: Timestamp
    retired_at: Timestamp | None
```

### 2.2.1 Required fields

- `ref`, `kind`, `layer`, `content`, `attrs`, `state`, `created_at`,
  `updated_at` MUST be present on every Node.
- `retired_at` MUST be present if and only if `state == "retired"`.

### 2.2.2 `kind`

- `kind` MUST be a non-empty string.
- `kind` is drawn from an **open enumeration**. The following values
  are **reserved** with the semantics given in Chapter 01:
  - `source` — attestation artifact
  - `concept` — atomic knowledge unit
  - `category` — aggregation or generalization
- Implementations MAY define additional kinds. Non-reserved kinds
  SHOULD use the prefix `ext:` to avoid collision (e.g.,
  `ext:snippet`, `ext:claim`).
- A Node's `kind` MUST NOT change after creation. Changing the role
  of a Node requires retiring it and creating a new Node with the
  desired kind.

### 2.2.3 `layer`

- `layer` MUST be a non-empty string.
- The following layer values are **reserved**:
  - `L_source` — attestation layer
  - `L_concept` — retrieval layer
  - `L_category` — aggregate layer
- Implementations MUST provide at least `L_concept`.
- Implementations MAY define additional layers. Non-reserved layers
  SHOULD use the prefix `L_ext_` (e.g., `L_ext_claim`).
- A Node's `layer` MUST NOT change after creation.

### 2.2.4 `kind`/`layer` consistency

The following pairings are **required** when the corresponding `kind`
is used:

| `kind`     | required `layer` |
|------------|------------------|
| `source`   | `L_source`       |
| `concept`  | `L_concept`      |
| `category` | `L_category`     |

Implementations MUST reject creation of a Node whose `kind` is
reserved but whose `layer` does not match the pairing above.

### 2.2.5 `content`

- `content` MUST be a string.
- `content` MUST NOT be empty for live Nodes with `kind="concept"`.
- For `kind="source"`, `content` MAY carry either inline text or a
  reference to external content (URL, blob pointer). The distinction
  is carried in `attrs` (see 2.2.7).
- Changing `content` is a **rewrite** operation and MUST update
  `updated_at` and produce an Event (see Chapter 04).

### 2.2.6 `attrs`

- `attrs` is a map from string keys to typed values.
- Keys are case-sensitive strings.
- Values MUST be JSON-serializable (strings, numbers, booleans,
  null, arrays of values, objects of string-keyed values).
- Keys prefixed with `ext:` are **implementation-defined**.
  Implementations MUST ignore `ext:`-prefixed keys they do not
  recognize during import and MUST NOT fail on them.
- Keys prefixed with `spk:`, `amkb:`, or other registered prefixes
  are **reserved**. Unregistered reserved prefixes are currently:
  - `amkb:` — reserved for future protocol use
  - `spk:` — reserved for Spikuit profile extensions

### 2.2.7 Reserved attributes

The following keys, when present, MUST have the specified meaning.

| Key              | Applies to          | Type           | Meaning |
|------------------|---------------------|----------------|---------|
| `domain`         | any Node            | string         | Administrative partition (see Chapter 01). |
| `content_ref`    | `kind="source"`     | string (URI)   | Pointer to external original content. |
| `content_hash`   | `kind="source"`     | string         | Hash of the original content, algorithm-prefixed (e.g., `sha256:...`). |
| `fetched_at`     | `kind="source"`     | Timestamp      | When the source was acquired. |
| `extractor`      | `kind="source"`     | string         | Name/version of the extractor used at ingest. |

Implementations MUST NOT assign different meanings to these keys.
Implementations MAY ignore reserved keys they do not use.

### 2.2.8 `state`

- `state` MUST be one of `"live"` or `"retired"`.
- Transitions:
  - A Node is created `live`.
  - A live Node MAY be retired. Retirement is terminal.
  - A retired Node MUST NOT be revived by any operation other than
    `revert` of its retirement.
- Retired Nodes MUST remain resolvable by `NodeRef` and MUST
  continue to participate in lineage traversal.

### 2.2.9 Retrieval-space membership

- Nodes with `kind="source"` MUST NOT appear in the retrieval space
  for any intent query, regardless of query parameters.
- Nodes with `kind="concept"` and `state="live"` MUST be eligible for
  the retrieval space.
- Retired Nodes MUST NOT appear in the retrieval space.
- Implementations MAY include other `kind` values (e.g., `category`)
  in the retrieval space at their discretion. This choice SHOULD be
  documented.

## 2.3 Edge

```
Edge:
    ref:       EdgeRef
    rel:       str              # relation type
    src:       NodeRef
    dst:       NodeRef
    attrs:     dict[str, Any]
    state:     EdgeState        # "live" or "retired"
    created_at: Timestamp
    retired_at: Timestamp | None
```

### 2.3.1 Required fields

- `ref`, `rel`, `src`, `dst`, `attrs`, `state`, `created_at`
  MUST be present on every Edge.
- `retired_at` MUST be present if and only if `state == "retired"`.

### 2.3.2 `rel`

- `rel` MUST be a non-empty string.
- The following relation values are **reserved** with fixed semantics:

**Attestation (concept → source):**

| `rel`             | Direction              | Meaning |
|-------------------|------------------------|---------|
| `derived_from`    | `L_concept` → `L_source` | Concept was extracted from the source. |
| `attested_by`     | `L_concept` → `L_source` | Source independently supports the concept. |
| `contradicted_by` | `L_concept` → `L_source` | Source contradicts the concept (disputed). |

**Source versioning (source → source):**

| `rel`           | Direction               | Meaning |
|-----------------|-------------------------|---------|
| `superseded_by` | `L_source` → `L_source` | Source has been replaced by a newer source. |

**Aggregation (category ↔ concept, category ↔ category):**

| `rel`         | Direction                  | Meaning |
|---------------|----------------------------|---------|
| `contains`    | `L_category` → `L_concept` | Category groups the concept. |
| `belongs_to`  | `L_concept` → `L_category` | Inverse of contains. |
| `generalizes` | `L_category` → `L_category` | Category subsumes another category. |

**Intra-concept (concept ↔ concept):**

| `rel`        | Direction / Symmetry | Meaning |
|--------------|----------------------|---------|
| `requires`   | Directed             | A requires understanding of B. |
| `extends`    | Directed             | A extends B. |
| `contrasts`  | Symmetric in meaning | A contrasts with B. |
| `relates_to` | Symmetric in meaning | General association. |

- Implementations MAY define additional `rel` values. Non-reserved
  relations SHOULD use the prefix `ext:` (e.g., `ext:cites`).
- Implementations MUST NOT use a reserved `rel` with semantics that
  differ from those given above.

### 2.3.3 Layer constraints on edges

- An Edge with a reserved `rel` MUST connect Nodes whose layers match
  the pairing given in 2.3.2.
- Implementations MUST reject Edges that use a reserved `rel` with
  mismatched layers.
- Concept Nodes MUST NOT have outgoing edges to Source Nodes except
  via the attestation relations (`derived_from`, `attested_by`,
  `contradicted_by`). Any other concept→source edge is forbidden.
- Source Nodes MAY have edges among themselves (source-layer edges).
  Such edges MUST NOT affect retrieval of concepts.

### 2.3.4 Self-loops and multiplicity

- An Edge MUST NOT have `src == dst` for reserved relations.
- Implementations MAY permit self-loops for non-reserved relations.
- Multiple live Edges between the same pair of Nodes with the same
  `rel` MAY exist. Implementations SHOULD treat them as distinct
  until explicitly merged.

### 2.3.5 Retirement

- Retiring an Edge MUST NOT retire its endpoint Nodes.
- Retiring a Node MUST retire all Edges where the Node is `src` or
  `dst`. Retired Edges remain resolvable for lineage and history.

## 2.4 Actor

```
Actor:
    id:       ActorId
    kind:     str              # "llm" | "human" | "automation" | "composite" | ext:...
    profile:  dict[str, Any]   # implementation-defined
```

### 2.4.1 Required fields

- `id`, `kind` MUST be present on every Actor.
- `profile` MAY be empty but MUST be present as an object.

### 2.4.2 `id`

- `ActorId` is an opaque string.
- `ActorId` MUST be stable for the lifetime of the Actor within a
  store.
- Two Actors with different `id` MUST be treated as distinct even
  if their `profile` is identical.

### 2.4.3 `kind`

- `kind` MUST be a non-empty string.
- The following values are **reserved**:
  - `llm` — an autonomous agent driven by a language model
  - `human` — a human operator
  - `automation` — a non-interactive script or pipeline
  - `composite` — an actor composed of multiple sub-actors
- Implementations MAY define additional kinds with the `ext:` prefix.
- The reserved values above MUST NOT be reused with different meaning.

### 2.4.4 Attribution requirement

- Every mutation operation (create, rewrite, retire, merge, link,
  unlink) MUST carry an Actor identifier.
- Operations that do not carry an Actor MUST be rejected with
  `E_MISSING_REQUIRED_ATTR`.

## 2.5 Transaction

```
Transaction:
    ref:       TransactionRef
    tag:       str              # human-readable batch name
    actor:     ActorId
    state:     TxState          # "open" | "committed" | "aborted"
    started_at: Timestamp
    closed_at:  Timestamp | None
```

### 2.5.1 Required fields

- `ref`, `tag`, `actor`, `state`, `started_at` MUST be present.
- `closed_at` MUST be present if and only if `state` is `committed`
  or `aborted`.

### 2.5.2 `tag`

- `tag` MUST be a non-empty string.
- `tag` SHOULD be human-readable and SHOULD identify the purpose of
  the batch (e.g., `ingest/papers-2026-04`, `consolidate/merge-duplicates`).
- Implementations MAY enforce uniqueness of `tag` across open
  transactions. They MUST NOT enforce uniqueness across historical
  transactions; the same `tag` MAY be used repeatedly over time.

### 2.5.3 Atomicity

- A committed Transaction's mutations MUST take effect together or
  not at all. An observer MUST NOT see a partial result of a
  committed Transaction.
- An aborted Transaction MUST leave the store unchanged, as if the
  Transaction had never begun.

### 2.5.4 Operation errors within an open Transaction

- An operation error raised within an open Transaction MUST NOT
  automatically abort the Transaction. The Actor MAY handle the
  error and continue with further operations.
- If the Actor calls `commit` on a Transaction that contains
  unresolved operation errors, the `commit` MUST fail with the
  accumulated errors unless every error was an idempotent no-op
  (e.g., retiring an already-retired Node).

### 2.5.5 Isolation

- Concurrent Transactions SHOULD be isolated from each other such
  that mutations in one Transaction are not visible to another
  until commit.
- Implementations MAY provide weaker or stronger isolation as long
  as committed state is consistent with some serialization of the
  concurrent Transactions.

## 2.6 Event and ChangeSet

```
ChangeSet:
    ref:       ChangeSetRef
    tx_ref:    TransactionRef
    tag:       str
    actor:     ActorId
    committed_at: Timestamp
    events:    list[Event]
```

```
Event:
    kind:    EventKind          # "node.created" | "node.rewritten" | ...
    target:  NodeRef | EdgeRef  # the entity affected
    before:  dict | None        # state before the change (if applicable)
    after:   dict | None        # state after the change (if applicable)
    meta:    dict[str, Any]     # implementation-defined extension data
```

### 2.6.1 Event kinds

The following event kinds are **reserved**:

- `node.created`
- `node.rewritten`
- `node.retired`
- `node.merged`
- `edge.created`
- `edge.retired`

Implementations MAY define additional event kinds with the `ext:`
prefix.

### 2.6.2 Emission

- Every committed Transaction MUST produce exactly one ChangeSet.
- A ChangeSet MUST contain one Event for each mutation that took
  effect.
- Aborted Transactions MUST NOT produce a ChangeSet.
- Events within a ChangeSet MUST be ordered consistently with the
  order of the operations within the Transaction.

### 2.6.3 Durability

- **MUST (Level B, minimum).** Committed ChangeSets and their Events
  MUST be recoverable after store restart.
- **SHOULD (Level C).** Events SHOULD be persisted to an append-only
  log that is independent of the primary store state, so that
  corruption of the primary store does not erase event history.
- **MAY (Level C+).** Implementations MAY stream events to external
  sinks (message queues, object stores) in near-real-time.

### 2.6.4 Ordering

- Events within a single ChangeSet MUST be totally ordered.
- ChangeSets MUST be ordered in at least a causal order consistent
  with commit time.
- Implementations MAY provide a total order across all ChangeSets.
  If they do, the total order MUST be consistent with commit time.

### 2.6.5 Content references

- When an Event's `target` changes `content`, the Event's `after`
  (and, if possible, `before`) MUST include a `content_hash` field
  so that external indexes can detect staleness without re-fetching.

## 2.7 Relevance Estimation

Intent queries (see 3.4.4) return concept Nodes ordered by the
implementation's estimate of their **relevance** to the given intent.
This specification is deliberately silent about how relevance is
estimated.

- This specification does NOT define what "relevance" means, how it
  is computed, or what evidence an implementation uses to produce
  its ordering.
- This specification does NOT define an embedding representation, a
  vector index format, a similarity metric, a lexical scoring
  function, or any other particular estimation strategy. All such
  strategies are equivalent from the protocol's perspective.
- Implementations MAY use any combination of techniques — embedding
  similarity, lexical matching, graph structure, learned rankers,
  language-model judgment, user profile signals, or anything else —
  to produce their ordering.
- Implementations MUST NOT place relevance-estimation state
  (embedding vectors, inverted indexes, learned weights) in Node
  or Edge `attrs` under canonical keys defined by this specification.
- Implementations MAY expose such state via `ext:`-prefixed
  attributes. Other implementations MUST ignore such attributes
  during import and export.
- When a mutation changes `Node.content`, the resulting Event MUST
  include a `content_hash` (see 2.6.5) so that external retrieval
  indexes can detect staleness without re-reading content.

> **Non-normative note.** "Relevance" is a property of the pair
> (intent, Node), not of the Node alone. The same Node MAY be highly
> relevant to one intent and irrelevant to another. An implementation's
> relevance estimator is effectively a function
> `estimate(intent, Node) -> ordering_signal`. The protocol specifies
> neither the signal type (scalar score, rank, class) nor the shape
> of the function.

## 2.8 Validation Summary

An AMKB implementation MUST reject operations that would violate
any of the following invariants:

1. A Node's `kind` is reserved but its `layer` does not match
   the reserved pairing (2.2.4).
2. A Node with `kind="concept"` has empty `content` (2.2.5).
3. An Edge uses a reserved `rel` with endpoint layers that do not
   match the reserved pairing (2.3.3).
4. A Concept Node has an outgoing edge to a Source Node with a
   non-attestation `rel` (2.3.3).
5. A reserved `rel` is used with semantics different from those
   specified in 2.3.2.
6. A mutation operation is invoked without an Actor (2.4.4).
7. An attempt is made to mutate a retired Node or Edge other than
   by reverting its retirement.
8. A semantic query returns a Node with `kind="source"` (2.2.9).

Violations of these invariants MUST raise errors from the
appropriate category (`INVALID` or `CONSTRAINT`); the full error
model is defined in a forthcoming chapter.
