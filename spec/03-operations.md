# 03 — Operations

This document specifies the operations an AMKB implementation
provides. It uses RFC 2119 keywords in their normative sense and
builds on the types defined in [02 — Types](02-types.md).

Operations are grouped by lifecycle phase:

- **3.2** Node lifecycle: create, rewrite, retire, merge
- **3.3** Edge lifecycle: link, unlink
- **3.4** Query: get, find_by_attr, neighbors, retrieve
- **3.5** Transactions: begin, commit, abort
- **3.6** History and revert: history, diff, revert
- **3.7** Events: events subscription

Each operation specification follows a fixed template:

- **Signature** — the parameters and return type
- **Parameters** — semantic meaning of each parameter
- **Preconditions** — what MUST hold before the call
- **Postconditions** — what MUST hold after a successful call
- **Events** — event kinds emitted on success
- **Errors** — canonical errors the operation MAY raise
- **Conformance level** — the minimum level that MUST offer this operation

Error codes referenced here (`E_INVALID_KIND`, etc.) are defined in
[05 — Errors](05-errors.md).

## 3.1 Conventions

### 3.1.1 Notation

Operation signatures use a Python-flavored syntax:

```
operation_name(required_arg, ...) -> return_type
operation_name(required_arg, *, keyword_arg=default) -> return_type
```

The normative content is the prose, not the syntax. Implementations
in other languages MUST preserve semantics, not syntactic form.

### 3.1.2 Transaction-owned operations

All mutation and query operations live on the `Transaction` object.
An implementation MUST expose operations via an object returned
from `Store.begin()`:

```python
async with store.begin(tag="example", actor=actor) as tx:
    ref = await tx.create(kind="concept", layer="L_concept",
                          content="...", attrs={})
    await tx.link(ref, source_ref, rel="derived_from")
    await tx.commit()
```

For convenience, implementations MAY provide operation methods on
`Store` directly. When provided, these MUST behave as if a
single-operation transaction were opened, the operation executed,
and the transaction committed. The `tag` for such implicit
transactions MUST be `"auto"` or an implementation-defined
auto-generated value. The Actor MUST still be provided.

Implementations MUST NOT provide mutation operations that run
outside any transaction. Read-only query operations (3.4.1, 3.4.2,
3.4.3, 3.4.4) MAY be offered directly on `Store` without opening
a transaction.

### 3.1.3 Async vs sync

The signatures below are presented in async form for clarity.
Implementations MAY offer sync or async APIs. Cross-language SDKs
SHOULD follow host-language conventions.

### 3.1.4 Idempotency

Some operations are **idempotent**: calling them a second time with
the same arguments has no additional effect and MUST NOT raise an
error. Idempotent operations are marked as such in their spec.

Notable idempotent cases:

- `retire(ref)` on an already-retired entity MUST succeed as a no-op.
- `unlink(ref)` on an already-retired edge MUST succeed as a no-op.

### 3.1.5 Error raising contract

- Operations raise errors by category (`NOT_FOUND`, `CONFLICT`,
  `INVALID`, `CONSTRAINT`, `INTERNAL`, `PERMISSION`).
- An error raised within an open transaction MUST NOT automatically
  abort the transaction (see 2.5.4).
- An implementation MAY raise additional impl-defined errors with
  `ext:` prefix. Clients unfamiliar with an `ext:` code MUST treat
  it as the containing category's generic error.

## 3.2 Node Lifecycle

### 3.2.1 `create`

**Signature**

```
create(
    kind: str,
    layer: str,
    content: str,
    attrs: dict[str, Any] = {},
) -> NodeRef
```

**Parameters**

- `kind`: the Node's kind (e.g., `"concept"`, `"source"`).
- `layer`: the Node's layer (e.g., `"L_concept"`).
- `content`: the Node's content payload.
- `attrs`: typed attributes; MAY be empty.

**Preconditions**

- The transaction MUST be in `open` state.
- `kind` MUST be a non-empty string.
- `layer` MUST be a non-empty string.
- If `kind` is a reserved value, `layer` MUST match the reserved
  pairing (see 2.2.4).
- If `kind == "concept"`, `content` MUST NOT be empty.
- `attrs` MUST contain only JSON-serializable values.

**Postconditions**

- A new Node is created with `state="live"`.
- The returned `NodeRef` MUST resolve to the new Node via `get`.
- `created_at` and `updated_at` are set to the current time.
- `retired_at` is absent.

**Events**

- `node.created`

**Errors**

- `E_INVALID_KIND` — `kind` empty or malformed
- `E_INVALID_LAYER` — `layer` empty or malformed
- `E_EMPTY_CONTENT` — `content` empty for `kind="concept"`
- `E_CROSS_LAYER_INVALID` — reserved kind with wrong layer
- `E_MISSING_REQUIRED_ATTR` — reserved attributes missing
- `E_TRANSACTION_CLOSED` — transaction not open
- `E_MISSING_REQUIRED_ATTR` — actor not set on transaction
- `E_INTERNAL` — backend failure

**Conformance**: L1 (Core)

---

### 3.2.2 `rewrite`

**Signature**

```
rewrite(
    ref: NodeRef,
    content: str,
    *,
    reason: str,
) -> NodeRef
```

**Parameters**

- `ref`: identifier of the Node to rewrite.
- `content`: new content. MUST NOT be empty for `kind="concept"`.
- `reason`: human-readable motivation for the rewrite. MUST NOT be
  empty.

**Preconditions**

- The transaction MUST be in `open` state.
- `ref` MUST resolve to a live Node.
- For `kind="concept"`, `content` MUST NOT be empty.

**Postconditions**

- The Node's `content` is replaced.
- `updated_at` is updated to the current time.
- The Node's `ref` is unchanged (see 2.1).
- The Node's prior content is retained in lineage.

**Events**

- `node.rewritten`
- The event's `before` and `after` MUST include `content_hash` for
  downstream index invalidation (see 2.6.5).

**Errors**

- `E_NODE_NOT_FOUND`
- `E_NODE_ALREADY_RETIRED`
- `E_EMPTY_CONTENT`
- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L1 (Core)

---

### 3.2.3 `retire`

**Signature**

```
retire(
    ref: NodeRef,
    *,
    reason: str,
) -> None
```

**Parameters**

- `ref`: identifier of the Node to retire.
- `reason`: human-readable motivation.

**Preconditions**

- The transaction MUST be in `open` state.
- `ref` MUST resolve to a Node (live or retired).

**Postconditions**

- The Node's `state` is `"retired"`.
- `retired_at` is set to the current time.
- All Edges where the Node is `src` or `dst` MUST also be retired.
- The Node remains resolvable by `ref` (tombstone).
- Lineage from or to this Node continues to function.

**Idempotency**

- If the Node is already retired, the call MUST succeed as a no-op
  and MUST NOT emit a new `node.retired` event.

**Events**

- `node.retired` (only on state change)
- `edge.retired` for each affected edge

**Errors**

- `E_NODE_NOT_FOUND`
- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L1 (Core)

---

### 3.2.4 `merge`

**Signature**

```
merge(
    refs: list[NodeRef],
    *,
    content: str,
    attrs: dict[str, Any] = {},
    reason: str,
) -> NodeRef
```

**Parameters**

- `refs`: Nodes to merge. MUST contain at least two distinct Node
  references.
- `content`: the merged Node's content.
- `attrs`: merged Node's attributes.
- `reason`: human-readable motivation.

**Preconditions**

- The transaction MUST be in `open` state.
- All `refs` MUST resolve to live Nodes.
- All Nodes in `refs` MUST share the same `kind` and `layer`.
- Merging MUST NOT produce a lineage cycle (i.e., none of the
  Nodes in `refs` MUST be an ancestor of any other via prior merges).

**Postconditions**

- A new Node is created with the merged `content` and `attrs`.
- All Nodes in `refs` are retired.
- Incoming Edges to retired Nodes are redirected to the new Node
  via lineage — implementations MAY represent this as either
  re-pointing live edges or creating new edges and retiring the
  originals.
- The new Node's lineage records all Nodes in `refs` as ancestors.
- A subsequent `revert` of the merge MUST be able to restore the
  original Nodes and edges.

**Events**

- `node.merged` with `meta.ancestors` listing the retired refs
- `node.created` for the new Node
- `node.retired` for each Node in `refs`
- `edge.retired` / `edge.created` as needed for edge redirection

**Errors**

- `E_NODE_NOT_FOUND`
- `E_NODE_ALREADY_RETIRED`
- `E_MERGE_CONFLICT` — kind/layer mismatch among refs
- `E_LINEAGE_CYCLE`
- `E_INVALID` — fewer than two distinct refs
- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L2 (Lineage)

## 3.3 Edge Lifecycle

### 3.3.1 `link`

**Signature**

```
link(
    src: NodeRef,
    dst: NodeRef,
    rel: str,
    attrs: dict[str, Any] = {},
) -> EdgeRef
```

**Parameters**

- `src`: source Node of the edge.
- `dst`: destination Node of the edge.
- `rel`: relation type.
- `attrs`: edge attributes. MAY be empty.

**Preconditions**

- The transaction MUST be in `open` state.
- Both `src` and `dst` MUST resolve to live Nodes.
- `rel` MUST be a non-empty string.
- If `rel` is a reserved relation, the endpoint layers MUST match
  the reserved pairing (see 2.3.2, 2.3.3).
- A concept→source edge MUST use one of the attestation relations
  (`derived_from`, `attested_by`, `contradicted_by`).
- `src != dst` if `rel` is reserved.

**Postconditions**

- A new Edge is created with `state="live"`.
- The returned `EdgeRef` MUST resolve to the new Edge.

**Events**

- `edge.created`

**Errors**

- `E_NODE_NOT_FOUND`
- `E_NODE_ALREADY_RETIRED`
- `E_INVALID_REL`
- `E_CROSS_LAYER_INVALID`
- `E_CONCEPT_TO_NONSOURCE_ATTEST`
- `E_SELF_LOOP`
- `E_RESERVED_REL_MISUSE`
- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L1 (Core)

---

### 3.3.2 `unlink`

**Signature**

```
unlink(
    ref: EdgeRef,
    *,
    reason: str,
) -> None
```

**Parameters**

- `ref`: identifier of the Edge to retire.
- `reason`: human-readable motivation.

**Preconditions**

- The transaction MUST be in `open` state.
- `ref` MUST resolve to an Edge (live or retired).

**Postconditions**

- The Edge's `state` is `"retired"`.
- `retired_at` is set.

**Idempotency**

- If the Edge is already retired, the call MUST succeed as a no-op
  and MUST NOT emit a new `edge.retired` event.

**Events**

- `edge.retired` (only on state change)

**Errors**

- `E_EDGE_NOT_FOUND`
- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L1 (Core)

## 3.4 Query

Query operations are read-only. They MAY be invoked on `Store`
directly (outside any transaction) or on `Transaction` (within an
open transaction, observing the transaction's own uncommitted
mutations).

### 3.4.1 `get`

**Signature**

```
get(ref: NodeRef) -> Node
get(ref: EdgeRef) -> Edge
```

**Parameters**

- `ref`: a `NodeRef` or `EdgeRef`.

**Preconditions**

- `ref` MUST resolve to a known entity (live or retired).

**Postconditions**

- Returns the full entity, including its state and attributes.
- Retired entities MUST be returned successfully.

**Errors**

- `E_NODE_NOT_FOUND` / `E_EDGE_NOT_FOUND`

**Conformance**: L1 (Core)

---

### 3.4.2 `find_by_attr`

**Signature**

```
find_by_attr(
    attributes: dict[str, Any],
    *,
    kind: str | None = None,
    layer: str | None = None,
    include_retired: bool = False,
    limit: int = 100,
) -> list[NodeRef]
```

**Parameters**

- `attributes`: a dict of attribute key/value pairs. A Node matches
  if, for every key in `attributes`, the Node's `attrs` contains
  that key with an **equal** value. This is pure equality matching;
  richer predicates live in `filters` (see 3.4.4).
- `kind`: if set, only Nodes with matching `kind` are returned.
- `layer`: if set, only Nodes with matching `layer` are returned.
- `include_retired`: if `false` (default), retired Nodes MUST be
  excluded.
- `limit`: maximum number of results. MUST be positive.

**Semantics**

- Matching is deterministic: two compliant implementations MUST
  return the same set of `NodeRef` (order MAY differ).
- `attributes` conditions are combined with logical AND.
- Equality uses JSON equality semantics (strings are case-sensitive,
  numbers are compared as equal if they represent the same numeric
  value, arrays and objects are compared deeply).

**Postconditions**

- Returns a list of `NodeRef` satisfying all conditions.
- List length does not exceed `limit`.

**Errors**

- `E_INVALID` — `limit <= 0` or malformed `attributes`
- `E_INTERNAL`

**Conformance**: L4a (Structural)

---

### 3.4.3 `neighbors`

**Signature**

```
neighbors(
    ref: NodeRef,
    *,
    rel: str | list[str] | None = None,
    direction: Direction = "out",
    depth: int = 1,
    include_retired: bool = False,
    limit: int = 100,
) -> list[NodeRef]
```

where `Direction` is one of `"out"`, `"in"`, `"both"`.

**Parameters**

- `ref`: the starting Node.
- `rel`: if set, restricts traversal to the given relation(s).
- `direction`: `"out"` follows outgoing edges, `"in"` incoming,
  `"both"` either.
- `depth`: traversal depth. MUST be ≥ 1.
- `include_retired`: whether retired Nodes and Edges are traversed.
- `limit`: maximum results.

**Semantics**

- `depth = 1` returns direct neighbors reachable via one edge.
- `depth > 1` returns Nodes reachable within `depth` edges.
- Cycles MUST NOT cause infinite loops; each Node MUST appear at
  most once in the returned list.

**Conformance**

- Implementations claiming L4a MUST support `depth = 1`.
- Implementations claiming L4a SHOULD support `depth > 1`.
  Implementations that do not support `depth > 1` MUST raise
  `E_INVALID` with a message clearly indicating the limitation.

**Postconditions**

- Returns a list of `NodeRef` reachable from `ref` under the given
  constraints.

**Errors**

- `E_NODE_NOT_FOUND`
- `E_INVALID` — `depth < 1`, `limit <= 0`, or unsupported depth
- `E_INTERNAL`

**Conformance**: L4a (Structural, depth=1); SHOULD L4a (depth>1)

---

### 3.4.4 `retrieve`

An intent query. Given a free-text `intent`, the implementation
returns concept Nodes it judges to be relevant to that intent, in
its own relevance order. The protocol does not define relevance;
see §2.7 for the intentional silence on how it is estimated.

**Signature**

```
retrieve(
    intent: str,
    *,
    k: int = 10,
    layer: str | list[str] | None = None,
    filters: Filter | None = None,
) -> list[RetrievalHit]
```

where `RetrievalHit` is:

```
RetrievalHit:
    ref:   NodeRef
    score: float | None    # implementation-defined; OPTIONAL
```

and `Filter` is defined in 3.4.5.

**Parameters**

- `intent`: a free-text statement of what the caller wants. MUST
  NOT be empty. Implementations interpret `intent` using their own
  relevance estimator; the protocol places no requirement on the
  interpretation.
- `k`: maximum number of hits. MUST be positive.
- `layer`: restrict hits to the given layer(s). If `None`, hits
  MUST come from the implementation's default retrieval-space
  layers (at minimum, `L_concept`).
- `filters`: a `Filter` expression (see 3.4.5). If set, hits MUST
  satisfy the filter before relevance estimation is applied.

**Semantics**

- The returned list is **ordered** by the implementation's
  relevance estimate, most relevant first. The *criterion* for
  relevance is implementation-defined; see §2.7.
- Hits MUST NOT include any Node with `kind="source"`, regardless
  of `layer` or `filters`.
- Hits MUST NOT include retired Nodes.
- Two compliant implementations MAY return different hits in
  different orders for the same `intent`. Both are correct.
- Conformance testing checks the **shape** of results (cardinality,
  retrieval-space membership, ordering property, filter
  compliance) — not the ranking quality, nor the criterion used
  to produce the order. See 06 — Conformance.

**Score field (OPTIONAL)**

- `RetrievalHit.score` MAY be `None`. Implementations that do not
  expose a numeric relevance estimate (e.g., rank-only, learned-
  ranker, LLM-judge, categorical relevance) MUST set `score = None`
  and rely on list order to carry the ordering.
- If `score` is set, it MUST be a real number and MUST be consistent
  with list order: for any two hits `a, b` at positions `i < j` in
  the returned list, `a.score >= b.score` MUST hold.
- The absolute magnitude and scale of `score` are
  implementation-defined. Callers MUST NOT assume cross-implementation
  comparability of scores, and SHOULD treat `score` as a
  monotonically-ordered signal at most.

**Postconditions**

- Returns a list of at most `k` hits.
- Each hit's `ref` resolves to a live Node whose `kind` is not
  `"source"`.
- The list is ordered by the implementation's relevance estimate.

**Errors**

- `E_INVALID` — empty `intent`, non-positive `k`, or malformed
  `filters`
- `E_SOURCE_IN_RETRIEVAL` — implementation bug; MUST NOT occur
  in a conformant implementation
- `E_INTERNAL`

**Conformance**: L4b (Intent)

---

### 3.4.5 `Filter` type

`Filter` is a composable predicate over Node attributes, used by
intent retrieval (3.4.4). It is distinct from the `attributes`
parameter of `find_by_attr`: `attributes` is equality-only, while
`Filter` supports richer predicates.

A `Filter` narrows the candidate set *before* the implementation's
relevance estimator ranks the survivors. Filters are deterministic;
the subsequent ranking is not.

**Minimal filter algebra**

```
Filter := Eq(key, value)
        | In(key, values)
        | Range(key, min=None, max=None, *, inclusive=True)
        | And(filters)
        | Or(filters)
        | Not(filter)
```

**Semantics**

- `Eq(key, value)`: attribute `key` exists and equals `value`.
  Equality is JSON equality.
- `In(key, values)`: attribute `key` exists and its value is an
  element of `values`.
- `Range(key, min, max, inclusive)`: attribute `key` exists, is
  numeric or a timestamp, and lies within `[min, max]`. `min` or
  `max` MAY be `None` for open-ended ranges. `inclusive` controls
  endpoint inclusion.
- `And(filters)`: all sub-filters match.
- `Or(filters)`: at least one sub-filter matches.
- `Not(filter)`: sub-filter does not match.

**Requirements**

- An implementation claiming L4b (Intent) MUST support `Eq`,
  `In`, `And`, `Or`.
- An implementation claiming L4b SHOULD support `Range` and `Not`.
- Implementations MAY add extension filter kinds with `ext:` prefix.
  Clients unfamiliar with an extension kind MUST reject filters
  containing it with `E_INVALID`.

**Rationale**

The filter algebra is kept intentionally small. Sophisticated query
languages belong to implementations; the protocol only guarantees
a portable subset sufficient for typical retrieval refinement
("concepts in domain X with confidence ≥ 0.7", "concepts tagged
`rust` or `go`", and so on).

## 3.5 Transactions

### 3.5.1 `begin`

**Signature**

```
begin(*, tag: str, actor: Actor) -> Transaction
```

**Parameters**

- `tag`: human-readable batch identifier. MUST NOT be empty.
- `actor`: the Actor responsible for all mutations in this
  transaction.

**Preconditions**

- `actor` MUST be a valid `Actor`.
- `tag` MUST be a non-empty string.

**Postconditions**

- A new `Transaction` object is returned in `state="open"`.
- `started_at` is set to the current time.
- The Transaction's `actor` is fixed and MUST NOT change.

**Errors**

- `E_INVALID` — empty `tag` or invalid `actor`
- `E_INTERNAL`

**Conformance**: L3 (Transactional)

---

### 3.5.2 `commit`

**Signature**

```
commit() -> ChangeSet
```

**Preconditions**

- The Transaction MUST be in `open` state.
- All operations in the Transaction MUST be in a consistent final
  state. If any operation raised an error and was not resolved,
  commit MUST fail unless every unresolved error is an idempotent
  no-op (2.5.4).

**Postconditions**

- The Transaction's `state` becomes `committed`.
- All mutations within the Transaction take effect atomically.
- A single `ChangeSet` is emitted containing one Event per
  mutation.
- `closed_at` is set.

**Events**

- One ChangeSet event per operation, emitted in order.

**Errors**

- `E_TRANSACTION_CLOSED` — already committed or aborted
- `E_CONCURRENT_MODIFICATION` — optimistic concurrency failure
- `E_CONSTRAINT` — invariant violation detected at commit time
- `E_INTERNAL`

**Conformance**: L3 (Transactional)

---

### 3.5.3 `abort`

**Signature**

```
abort() -> None
```

**Preconditions**

- The Transaction MUST be in `open` state.

**Postconditions**

- The Transaction's `state` becomes `aborted`.
- No mutations take effect.
- No `ChangeSet` is emitted.
- `closed_at` is set.

**Errors**

- `E_TRANSACTION_CLOSED`
- `E_INTERNAL`

**Conformance**: L3 (Transactional)

## 3.6 History and Revert

### 3.6.1 `history`

**Signature**

```
history(
    *,
    since: Timestamp | None = None,
    until: Timestamp | None = None,
    actor: ActorId | None = None,
    tag: str | None = None,
    limit: int = 100,
) -> list[ChangeSetRef]
```

**Parameters**

- `since`, `until`: time window. MAY be omitted.
- `actor`: if set, only ChangeSets from this Actor.
- `tag`: if set, only ChangeSets whose transaction `tag` equals this.
- `limit`: max results. MUST be positive.

**Postconditions**

- Returns a list of `ChangeSetRef` matching the filters, ordered
  by commit time descending.

**Errors**

- `E_INVALID` — non-positive `limit` or inverted window
- `E_INTERNAL`

**Conformance**: L1 (Core)

---

### 3.6.2 `diff`

**Signature**

```
diff(from_ts: Timestamp, to_ts: Timestamp) -> list[Event]
```

**Parameters**

- `from_ts`, `to_ts`: time window, inclusive of both endpoints.

**Postconditions**

- Returns the list of Events committed strictly between `from_ts`
  and `to_ts`, in commit order.
- Applying the returned Events to a store that was in the state
  at `from_ts` MUST produce the state at `to_ts`.

**Errors**

- `E_INVALID` — `from_ts > to_ts`
- `E_INTERNAL`

**Conformance**: L2 (Lineage)

---

### 3.6.3 `revert`

**Signature**

```
revert(
    target: ChangeSetRef | str,
    *,
    reason: str,
    actor: Actor,
) -> ChangeSet
```

**Parameters**

- `target`: either a specific `ChangeSetRef` or a transaction `tag`
  string. If a tag is given and multiple ChangeSets share the tag,
  all of them MUST be reverted together in reverse commit order.
- `reason`: human-readable motivation.
- `actor`: the Actor performing the revert.

**Semantics**

- Revert creates a **new** ChangeSet whose mutations are the
  inverse of the target's mutations. It does not remove the
  original ChangeSet from history.
- Reverting a `revert` MUST restore the original state.
- Revert respects lineage: if the target contains merges,
  reverting restores the merged-away Nodes and edges.
- If the store's current state has diverged from the target
  (later mutations affect the same entities), implementations
  MAY raise `E_CONFLICT`; they SHOULD attempt best-effort
  revert before raising.

**Postconditions**

- A new ChangeSet is emitted whose events invert the target.
- `history()` now contains both the original and the inverse
  ChangeSets.

**Errors**

- `E_CHANGESET_NOT_FOUND`
- `E_CONFLICT` — state diverged, revert not possible
- `E_CONSTRAINT` — invariant would be violated
- `E_INTERNAL`

**Conformance**: L1 (Core)

## 3.7 Events

### 3.7.1 `events`

**Signature**

```
events(
    *,
    since: Timestamp | None = None,
    follow: bool = False,
) -> AsyncIterator[Event]
```

**Parameters**

- `since`: if set, start from this timestamp. If omitted, start
  from the earliest retained event.
- `follow`: if `true`, the iterator MUST continue yielding new
  events as they are committed, and MUST NOT terminate until the
  consumer closes it. If `false`, the iterator yields events up
  to the current time and terminates.

**Semantics**

- Events are yielded in the order defined in 2.6.4 (total within
  a ChangeSet, causal across ChangeSets).
- Implementations MUST NOT skip events within the requested range.
- Implementations MUST NOT re-order events within a ChangeSet.
- `follow = true` mode implies the caller takes responsibility
  for closing the iterator; implementations MAY impose a max
  subscription count.

**Errors**

- `E_INVALID` — `since` in the future
- `E_INTERNAL`

**Conformance**: L1 (Core)

## 3.8 Summary Table

| Operation         | Owner       | Conformance |
|-------------------|-------------|-------------|
| `create`          | Transaction | L1          |
| `rewrite`         | Transaction | L1          |
| `retire`          | Transaction | L1          |
| `merge`           | Transaction | L2          |
| `link`            | Transaction | L1          |
| `unlink`          | Transaction | L1          |
| `get`             | Store / Tx  | L1          |
| `find_by_attr`    | Store / Tx  | L4a         |
| `neighbors`       | Store / Tx  | L4a (d=1 MUST; d>1 SHOULD) |
| `retrieve`        | Store / Tx  | L4b (Intent) |
| `begin`           | Store       | L3          |
| `commit`          | Transaction | L3          |
| `abort`           | Transaction | L3          |
| `history`         | Store       | L1          |
| `diff`            | Store       | L2          |
| `revert`          | Store       | L1          |
| `events`          | Store       | L1          |

Implementations claim a conformance level by supporting all
operations marked at or below that level, and by passing the
corresponding conformance tests (see 06 — Conformance, forthcoming).
