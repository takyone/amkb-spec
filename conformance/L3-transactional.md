# L3 — Transactional Conformance Tests

Level 3 adds full transactional guarantees: concurrency control,
revertability, and constraint checking at commit. L3 implementations
MUST detect concurrent modification, support `revert`, and enforce
protocol invariants transactionally.

This matrix is a **draft**. L3 inherits all L1 and L2 requirements.

## concurrency

### L3.concurrent.01 — Concurrent modification detected

**What.** Two transactions that begin from the same snapshot and
commit conflicting mutations MUST NOT both succeed. The second commit
raises `E_CONCURRENT_MODIFICATION`.

**Spec.** 05-errors §5.4, 03-operations §3.5.2.

**Setup.** Two open transactions `T1`, `T2` both targeting the same
Node `a`.

**Action.** Commit `T1`, then `T2`.

**Expected.** `T1` commits; `T2` raises `E_CONCURRENT_MODIFICATION`
and is aborted.

### L3.concurrent.02 — Disjoint transactions both commit

**What.** Two concurrent transactions that modify disjoint state MAY
both commit successfully.

**Spec.** 03-operations §3.5.2.

**Setup.** Two transactions `T1`, `T2`, each targeting a different
Node.

**Action.** Commit both.

**Expected.** Both commits succeed; both ChangeSets are ordered
causally in the event log.

## revert

### L3.revert.01 — Revert of simple creation

**What.** Reverting a transaction that only created Nodes retires
those Nodes.

**Spec.** 03-operations §3.6, 04-events §4.1.

**Setup.** Commit a transaction `T` creating Node `a`.

**Action.** `revert(T)`, `commit`.

**Expected.** `a` is retired; a new revert ChangeSet is emitted
whose inverse reflects `T`.

### L3.revert.02 — Revert of merge

**What.** Reverting a merge retires the merged Node and restores the
source Nodes to a live state.

**Spec.** 03-operations §3.6.

**Setup.** Commit a merge of `n1, n2` into `m`.

**Action.** `revert(<merge-tag>)`, `commit`.

**Expected.** `m` is retired; `n1` and `n2` are live again.

### L3.revert.03 — Revert conflict MAY raise E_CONFLICT

**What.** Reverting a transaction whose effects have been diverged
by a subsequent transaction MAY raise `E_CONFLICT`; 03-operations
§3.6.3 makes this a MAY, not a MUST ("implementations MAY raise
`E_CONFLICT`; they SHOULD attempt best-effort revert before
raising"), so an implementation that instead performs a best-effort
revert is also conformant.

**Spec.** 03-operations §3.6.3, 05-errors §5.4.

**Setup.** Commit `T1` creating `a`; commit `T2` rewriting `a`.

**Action.** `revert(T1)`.

**Expected.** Either `E_CONFLICT` is raised (transaction unaffected),
or the implementation performs and documents a best-effort revert.
An implementation MUST NOT silently produce an incomplete revert
without raising `E_CONFLICT` (05-errors §5.4, `E_CONFLICT`).

### L3.revert.04 — Revert of unknown tag

**What.** Reverting a non-existent transaction raises
`E_CHANGESET_NOT_FOUND`.

**Spec.** 05-errors §5.4.

**Setup.** None.

**Action.** `revert("non-existent-tag")`.

**Expected.** `E_CHANGESET_NOT_FOUND` is raised.

## constraints

### L3.constraint.01 — Commit-time invariant violation

<!-- pending S-table -->
<!-- The "Node a requires Node b via a reserved attribute" mechanism
     referenced below does not exist anywhere in the spec (no
     operation or type establishes an inter-Node "required by"
     relationship via an attribute). Marked pending the Phase B
     normative work rather than resolved here. -->

**What.** A transaction that would leave the store in a state
violating a protocol invariant (e.g., a required attribute missing
after retirement) raises `E_CONSTRAINT` at commit and aborts.

**Spec.** 05-errors §5.2, §5.4.

**Setup.** A store where Node `a` requires Node `b` via a reserved
attribute.

**Action.** Open a transaction retiring `b`, then `commit`.

**Expected.** `E_CONSTRAINT` is raised; the transaction is aborted;
`b` remains live.
