# L2 — Lineage Conformance Tests

Level 2 adds lineage tracking on top of L1. An L2 implementation MUST
preserve the chain of rewrites, merges, and retirements such that the
origin of any live Node can be traced back through its predecessors.

This matrix is a **draft** and enumerates intended coverage only. L2
inherits all L1 requirements; an implementation claiming L2 MUST pass
every L1 test in addition to every test below.

## rewrite

### L2.rewrite.01 — Rewrite preserves predecessor link

**What.** Rewriting a live Node produces a new live Node whose
lineage chain includes the rewritten predecessor.

**Spec.** 03-operations §3.3 (rewrite), 01-concepts (lineage).

**Setup.** A live Node `a`.

**Action.** `node_rewrite(a, new_content="...")`, `commit`.

**Expected.** A new Node `a'` is live; `a` is retired; `lineage(a')`
contains `a`.

### L2.rewrite.02 — Rewrite of retired Node rejected

**What.** Rewriting a retired Node raises `E_NODE_ALREADY_RETIRED`.

**Spec.** 05-errors §5.4.

**Setup.** A retired Node `a`.

**Action.** `node_rewrite(a, ...)`.

**Expected.** `E_NODE_ALREADY_RETIRED` is raised.

## merge

### L2.merge.01 — Merge of `k` Nodes into one

**What.** Merging `k` live Nodes of the same kind/layer produces one
live Node and `k` retired Nodes, all connected by lineage.

**Spec.** 03-operations §3.3 (merge), 04-events §4.7.1.

**Setup.** `k` live Nodes `n1..nk`, same `kind` and `layer`.

**Action.** `node_merge([n1..nk], new_content="...")`, `commit`.

**Expected.** A new live Node `m`; each `ni` is retired; `lineage(m)`
contains all of `n1..nk`.

### L2.merge.02 — Merge with mismatched kind rejected

**What.** Merging Nodes of differing `kind` or `layer` raises
`E_MERGE_CONFLICT`.

**Spec.** 05-errors §5.4.

**Setup.** Two live Nodes with different `kind`.

**Action.** `node_merge([n1, n2], ...)`.

**Expected.** `E_MERGE_CONFLICT` is raised.

### L2.merge.03 — Merge produces correct event shape

**What.** A merge of `k` Nodes emits `k` retirement events plus one
creation event in a single ChangeSet, with `meta` referencing all
inputs and the merged Node.

**Spec.** 04-events §4.2.4, §4.7.1.

**Setup.** Subscribe via `events(follow=true)`.

**Action.** Merge 3 Nodes.

**Expected.** The ChangeSet contains exactly 4 events (3 retire, 1
create). The creation event's `meta` lists all 3 source refs and the
merged ref.

## lineage query

### L2.lineage.01 — Transitive predecessor retrieval

**What.** `lineage(n)` returns all transitive predecessors of `n`,
across rewrites and merges.

**Spec.** 03-operations §3.5.

**Setup.** A chain `a → a' → a''` via two rewrites.

**Action.** `lineage(a'')`.

**Expected.** Both `a` and `a'` appear in the result.

### L2.lineage.02 — Cycle prevention

**What.** An operation that would introduce a lineage cycle raises
`E_LINEAGE_CYCLE`.

**Spec.** 05-errors §5.4.

**Setup.** A live Node `a` with predecessor `a0`.

**Action.** Attempt an operation that would record `a0` as a
descendant of `a`.

**Expected.** `E_LINEAGE_CYCLE` is raised.

## retention

### L2.retain.01 — Event retention while successor live

**What.** Events from a merge transaction MUST be retained as long as
any Node the merge produced remains live.

**Spec.** 04-events §4.5.1.

**Setup.** Commit a merge; do not prune.

**Action.** Read the merge's events via `events(since=None)`.

**Expected.** All merge events are still present.
