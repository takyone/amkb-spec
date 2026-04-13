# 05 — Errors

This chapter defines the normative error model for AMKB operations.
Chapter 03 specifies which errors each operation MAY raise; this
chapter specifies what those errors mean, how they are categorized,
and what a caller MAY assume about retry semantics.

## 5.1 Error Shape

Every error raised by an AMKB operation MUST carry at least:

- **`code`** — a canonical identifier drawn from the catalog in §5.4.
  Opaque to downstream code except for equality comparison.
- **`message`** — a human-readable, non-localized string giving
  additional context. Callers MUST NOT parse this field
  programmatically.
- **`category`** — the category assigned to `code` in §5.3. MAY be
  derived from `code` by the caller; implementations MAY include it
  explicitly.

Implementations MAY include additional fields (stack traces,
correlation IDs, offending refs, hints) as long as the required
fields are present. The wire representation (exception class, error
struct, JSON object) is implementation-defined.

An error SHOULD NOT be raised for conditions that the operation's
contract in Chapter 03 does not list. If an implementation detects a
condition not covered by any canonical code, it MUST raise
`E_INTERNAL` with a descriptive message.

## 5.2 Transaction Behavior on Error

When an operation raises an error inside an open transaction, the
transaction MUST remain in the `open` state unless the error code
explicitly terminates it. The caller is free to retry the operation,
issue a different operation, or `abort` the transaction.

The following codes terminate the containing transaction (subsequent
operations in the same transaction MUST raise `E_TRANSACTION_CLOSED`):

- `E_CONCURRENT_MODIFICATION` — raised by `commit`; the transaction
  is aborted.
- `E_CONSTRAINT` — raised by `commit` or `revert`; the transaction is
  aborted.
- `E_INTERNAL` during `commit` — the transaction's final state is
  implementation-defined but MUST NOT be `committed`.

All other codes leave the transaction open and retryable.

## 5.3 Error Categories

Errors are grouped into five categories. The category determines
whether the caller can retry immediately, must change input, or must
treat the condition as unrecoverable.

| Category | Meaning | Caller retry |
|----------|---------|--------------|
| **validation** | Input violates a precondition | Fix input before retry |
| **not_found** | Referenced entity does not exist | Fix ref before retry |
| **state** | Store state incompatible with operation | MAY retry after state change |
| **invariant** | Protocol invariant violated | MUST NOT retry; implementation bug |
| **internal** | Implementation-side failure | MAY retry; MAY succeed later |

Category assignment is stable: a given code belongs to exactly one
category across all versions of the spec within a major version.

## 5.4 Canonical Error Codes

The following 22 codes are canonical. Implementations MUST use these
codes for the conditions described. Implementations MAY define
additional codes for implementation-specific conditions; such codes
MUST NOT collide with the canonical set and SHOULD be prefixed (e.g.,
`E_SPIKUIT_FSRS_STATE_MISMATCH`).

### Validation category

**`E_INVALID`**
*Category:* validation.
A catch-all for malformed arguments that do not match a more specific
code. Examples: non-positive `limit`, inverted time windows, `depth < 1`
on `neighbors`, empty transaction `tag`, unsupported filter operator
in an `attributes` map. Operations SHOULD prefer a more specific code
when one exists.

**`E_INVALID_KIND`**
*Category:* validation.
The `kind` field is empty, malformed, or otherwise syntactically
invalid. Does not apply when the `kind` is well-formed but
semantically incompatible with the layer — that is
`E_CROSS_LAYER_INVALID`.

**`E_INVALID_LAYER`**
*Category:* validation.
The `layer` field is empty, malformed, or names a layer the
implementation does not support.

**`E_INVALID_REL`**
*Category:* validation.
The `rel` field on an edge creation is empty, malformed, or names a
relation the implementation does not support.

**`E_EMPTY_CONTENT`**
*Category:* validation.
A Node with `kind="concept"` was created or rewritten with empty
`content`. Per 02-types §2.2.4, concept Nodes MUST have non-empty
content.

**`E_CROSS_LAYER_INVALID`**
*Category:* validation.
A reserved `kind`/`layer` pairing was violated. For example, a Node
with `kind="source"` placed in `L_concept`, or an edge with
`rel="derived_from"` pointing from a source Node to a concept Node.
See 02-types §2.4 for the full table of reserved pairings.

**`E_MISSING_REQUIRED_ATTR`**
*Category:* validation.
A reserved attribute required for the operation is missing. Examples:
a `begin()` call without `actor` set; a source Node without
`source_uri` (when the implementation requires it); a transaction
committed while an invariant requires a particular attribute.

**`E_RESERVED_REL_MISUSE`**
*Category:* validation.
A reserved relation was used in a way that violates its reserved
semantics. Example: adding a `contrasts` edge between Nodes in
different layers, or `generalizes` outside the category layer.

**`E_CONCEPT_TO_NONSOURCE_ATTEST`**
*Category:* validation.
An `attested_by` or `derived_from` edge was created with a destination
that is not a source Node. These relations are defined as
concept → source only.

**`E_SELF_LOOP`**
*Category:* validation.
An edge was created with identical `src` and `dst`. Self-loops are
forbidden on all relations.

### Not-found category

**`E_NODE_NOT_FOUND`**
*Category:* not_found.
A `NodeRef` passed to an operation does not correspond to any Node in
the store, retired or live.

**`E_EDGE_NOT_FOUND`**
*Category:* not_found.
An `EdgeRef` passed to an operation does not correspond to any Edge in
the store.

**`E_CHANGESET_NOT_FOUND`**
*Category:* not_found.
A transaction tag or changeset identifier passed to `revert` or
`history` does not correspond to any committed transaction.

### State category

**`E_NODE_ALREADY_RETIRED`**
*Category:* state.
An operation that requires a live Node (rewrite, merge, link as src
or dst) targeted a Node that is currently retired. Callers MAY
inspect lineage to find the replacement Node if one exists.

**`E_MERGE_CONFLICT`**
*Category:* state.
A `merge` call was issued with Nodes whose `kind` or `layer` are not
all equal. Merges are only valid among Nodes of the same kind and
layer.

**`E_LINEAGE_CYCLE`**
*Category:* state.
An operation would introduce a cycle in lineage — for example,
merging a Node into one of its own ancestors, or linking
`derived_from` in a way that closes a loop.

**`E_TRANSACTION_CLOSED`**
*Category:* state.
An operation was issued on a transaction that has already been
committed, aborted, or terminated by a prior error (see §5.2).

**`E_CONCURRENT_MODIFICATION`**
*Category:* state.
`commit` was called on a transaction whose optimistic-concurrency
check failed — the store state observed at `begin` has since diverged
in a way that would invalidate the transaction's effect. Callers MAY
retry the transaction from `begin`.

**`E_CONSTRAINT`**
*Category:* state.
`commit` or `revert` detected a protocol invariant that would be
violated by applying the transaction. Examples: retirement of a Node
still referenced by a required attribute on another Node; revert of a
transaction whose inverse would resurrect a retired Node in a layer
that now forbids its kind. The transaction MUST be aborted; the
caller MAY issue a different sequence of operations.

**`E_CONFLICT`**
*Category:* state.
`revert` was called on a transaction whose effects cannot be cleanly
undone because subsequent transactions have diverged the store state.
Implementations MAY attempt best-effort partial reverts; if they do,
they MUST raise `E_CONFLICT` rather than silently producing an
incomplete revert.

### Invariant category

**`E_SOURCE_IN_RETRIEVAL`**
*Category:* invariant.
A hit with `kind="source"` was about to be returned by `retrieve`.
This condition MUST NOT occur in a conformant implementation; per
01-concepts, source Nodes are excluded from the retrieval space by
negative constraint. The code exists to let implementations fail
loudly if an implementation bug would otherwise violate the
invariant. Callers MUST NOT retry.

### Internal category

**`E_INTERNAL`**
*Category:* internal.
An implementation-side failure that does not map to any other code.
Examples: storage backend unavailable, out-of-memory, serialization
failure, unexpected exception in an extension module. Callers MAY
retry; the operation MAY succeed later. Implementations SHOULD
include enough detail in `message` or additional fields to diagnose
the cause.

## 5.5 Extensibility

Implementations MAY define additional error codes for
implementation-specific conditions. Such codes:

1. MUST NOT collide with the canonical set in §5.4.
2. SHOULD be prefixed with an identifier of the implementation or
   extension (e.g., `E_SPIKUIT_*`).
3. SHOULD be assigned to a category from §5.3. If none fits,
   `internal` is the default.
4. MUST be documented by the implementation that defines them.

Future spec versions MAY promote widely-adopted implementation
extensions to the canonical set. Such promotions are additive and do
not require a major version bump unless they change the category or
semantics of an existing code.

## 5.6 Error Propagation Across Operations

A single operation raises at most one error. When an operation
internally invokes other operations (for example, `commit` validating
edges created earlier in the transaction), the implementation MUST
map internal failures to a single outermost error code, prioritizing
specificity. The internal chain MAY be exposed in optional fields
for diagnostics but MUST NOT alter the outermost `code`.

## 5.7 Summary Table

| Code | Category | Retry |
|------|----------|-------|
| `E_INVALID` | validation | After input fix |
| `E_INVALID_KIND` | validation | After input fix |
| `E_INVALID_LAYER` | validation | After input fix |
| `E_INVALID_REL` | validation | After input fix |
| `E_EMPTY_CONTENT` | validation | After input fix |
| `E_CROSS_LAYER_INVALID` | validation | After input fix |
| `E_MISSING_REQUIRED_ATTR` | validation | After input fix |
| `E_RESERVED_REL_MISUSE` | validation | After input fix |
| `E_CONCEPT_TO_NONSOURCE_ATTEST` | validation | After input fix |
| `E_SELF_LOOP` | validation | After input fix |
| `E_NODE_NOT_FOUND` | not_found | After ref fix |
| `E_EDGE_NOT_FOUND` | not_found | After ref fix |
| `E_CHANGESET_NOT_FOUND` | not_found | After ref fix |
| `E_NODE_ALREADY_RETIRED` | state | After state change |
| `E_MERGE_CONFLICT` | state | After input fix |
| `E_LINEAGE_CYCLE` | state | After input fix |
| `E_TRANSACTION_CLOSED` | state | Open new transaction |
| `E_CONCURRENT_MODIFICATION` | state | Retry from `begin` |
| `E_CONSTRAINT` | state | After input change |
| `E_CONFLICT` | state | Unrecoverable for this changeset |
| `E_SOURCE_IN_RETRIEVAL` | invariant | MUST NOT retry |
| `E_INTERNAL` | internal | MAY retry |
