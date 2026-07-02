# L1 — Core Conformance Tests

Level 1 covers the minimum behavior every AMKB implementation MUST
exhibit: Node and Edge CRUD, transaction lifecycle, event emission with
causal ordering and Level B durability, and the canonical error model.

This matrix is a **draft**: it enumerates intended coverage but is not
yet exhaustive. Contributions from implementers are welcome.

Normative references point into `spec/` at the section level. When a
test exercises a behavior defined in multiple chapters, the primary
section is listed first.

## create

### L1.create.01 — Concept Node with non-empty content

**What.** A Node with `kind="concept"` and non-empty `content` is
successfully created inside an open transaction and is visible after
commit.

**Spec.** 02-types §2.2, 03-operations §3.2.1.

**Setup.** Open a transaction with a valid `actor`.

**Action.** Call `create(kind="concept", layer="L_concept",
content="hello")`, then `commit`.

**Expected.** The returned `NodeRef` resolves to a live Node whose
`content` equals `"hello"` and whose `created_at` is the time the
`create` operation was issued (03-operations §3.2.1 Postconditions),
not the transaction's commit time.

### L1.create.02 — Concept Node with empty content rejected

**What.** Creating a concept Node with empty `content` raises
`E_EMPTY_CONTENT`.

**Spec.** 02-types §2.2.5, 05-errors §5.4 (`E_EMPTY_CONTENT`).

**Setup.** Open a transaction.

**Action.** Call `create(kind="concept", layer="L_concept",
content="")`.

**Expected.** `E_EMPTY_CONTENT` is raised; the transaction remains open
and no Node is created.

### L1.create.03 — Cross-layer kind violation rejected

**What.** Creating a Node with a reserved `kind`/`layer` pairing that
the spec forbids raises `E_CROSS_LAYER_INVALID`.

**Spec.** 02-types §2.2.4, 05-errors §5.4 (`E_CROSS_LAYER_INVALID`).

**Setup.** Open a transaction.

**Action.** Call `create(kind="source", layer="L_concept", ...)`.

**Expected.** `E_CROSS_LAYER_INVALID` is raised.

### L1.create.04 — Edge between live Nodes

**What.** An Edge between two live Nodes of compatible kinds is
created and visible after commit.

**Spec.** 02-types §2.3, 03-operations §3.3.1.

**Setup.** Two live Nodes `a`, `b` of compatible kind/layer for the
chosen `rel`.

**Action.** `link(src=a, dst=b, rel="relates_to")`, then
`commit`.

**Expected.** The Edge is retrievable and both endpoints expose it via
`neighbors`.

### L1.create.05 — Self-loop rejected

**What.** Creating an Edge with `src == dst` raises `E_SELF_LOOP`.

**Spec.** 02-types §2.3, 05-errors §5.4 (`E_SELF_LOOP`).

**Setup.** A live Node `a`.

**Action.** `link(src=a, dst=a, rel="relates_to")`.

**Expected.** `E_SELF_LOOP` is raised.

## retire

### L1.retire.01 — Retire live Node

**What.** Retiring a live Node produces a tombstoned state visible to
`get` and excluded from default `retrieve`.

**Spec.** 03-operations §3.2.3.

**Setup.** A live Node `a`.

**Action.** `retire(a)`, `commit`.

**Expected.** `get(a)` returns the tombstoned Node; `retrieve` with
default filters does not return `a`.

### L1.retire.02 — Retiring already-retired Node is a no-op

**What.** Retiring an already-retired Node does not raise and does not
emit an event.

**Spec.** 03-operations §3.2.3, 04-events §4.2.3.

**Setup.** A retired Node `a`.

**Action.** `retire(a)`, `commit`.

**Expected.** Commit succeeds. The resulting ChangeSet contains no
event for `a`.

## retrieve

### L1.retrieve.01 — Source Nodes excluded

**What.** `retrieve` MUST NOT return any Node with `kind="source"`.

**Spec.** 01-concepts, 05-errors §5.4 (`E_SOURCE_IN_RETRIEVAL`).

**Setup.** A mixed store containing source and concept Nodes.

**Action.** Call `retrieve` with any intent.

**Expected.** No result has `kind="source"`.

### L1.retrieve.02 — Limit respected

**What.** `retrieve(k=k)` returns at most `k` hits.

**Spec.** 03-operations §3.4.

**Setup.** A store with more than `k` matching concept Nodes.

**Action.** `retrieve(intent, k=k)`.

**Expected.** `len(results) <= k`.

### L1.retrieve.03 — Retired Nodes excluded by default

**What.** Retired Nodes MUST NOT appear in default `retrieve` output.

**Spec.** 03-operations §3.4.

**Setup.** A store containing one retired and one live Node that both
match the intent.

**Action.** `retrieve(intent)`.

**Expected.** Only the live Node is returned.

## events

### L1.events.01 — ChangeSet per commit

**What.** Each committed transaction produces exactly one ChangeSet.

**Spec.** 04-events §4.2.1.

**Setup.** Subscribe via `events(since=None, follow=true)`.

**Action.** Commit a transaction containing `n` effective mutations.

**Expected.** Exactly one ChangeSet containing `n` events is
delivered.

### L1.events.02 — Aborted transaction emits nothing

**What.** An aborted transaction MUST NOT produce any event.

**Spec.** 04-events §4.2.2.

**Setup.** Subscribe via `events(follow=true)`.

**Action.** Open a transaction, perform one operation, `abort`.

**Expected.** No event is delivered for the aborted operation.

### L1.events.03 — Durable across restart

**What.** Committed events survive a store restart (Level B).

**Spec.** 04-events §4.4.2.

**Setup.** Commit a transaction, then restart the store.

**Action.** `events(since=None, follow=false)`.

**Expected.** The committed events are returned in their original
order.

### L1.events.04 — Causal order across ChangeSets

**What.** If `A` commits before `B`, `A`'s events MUST appear entirely
before any of `B`'s events.

**Spec.** 04-events §4.3.2.

**Setup.** Commit two sequential transactions `A` and `B`.

**Action.** `events(since=None, follow=false)`.

**Expected.** All events from `A` precede all events from `B`.

### L1.events.05 — Within-ChangeSet order preserved

**What.** Events within a ChangeSet are ordered as the operations
were issued.

**Spec.** 04-events §4.3.1.

**Setup.** A transaction that creates `n1`, retires `n2`, and creates
`n3` in that order.

**Action.** `events(follow=false)`.

**Expected.** The ChangeSet contains the three events in the original
issue order.

## transactions

### L1.tx.01 — Operation on closed transaction rejected

**What.** Issuing an operation on a committed transaction raises
`E_TRANSACTION_CLOSED`.

**Spec.** 05-errors §5.2, §5.4.

**Setup.** A committed transaction `t`.

**Action.** `create(...)` on `t`.

**Expected.** `E_TRANSACTION_CLOSED` is raised.

### L1.tx.02 — Begin requires actor

**What.** `begin()` without a valid `actor` raises `E_INVALID`.

**Spec.** 03-operations §3.5.1, 05-errors §5.3.

**Setup.** None.

**Action.** `begin()` with no `actor`.

**Expected.** `E_INVALID` is raised. (`E_MISSING_REQUIRED_ATTR` is the
missing-actor code for `create` per 02-types §2.4.4; `begin()`'s own
error contract in §3.5.1 names only `E_INVALID`.)

## history

### L1.history.01 — ChangeSet retrievable by tag

**What.** A committed ChangeSet is retrievable by its transaction
`tag`.

**Spec.** 03-operations §3.6, 04-events §4.1.

**Setup.** Commit a transaction with tag `T`.

**Action.** `history(tag=T)`.

**Expected.** The ChangeSet for `T` is returned with its events.
