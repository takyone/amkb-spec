# L4a — Structural Conformance Tests

Level 4a adds structural retrieval over the graph: neighbor traversal,
depth-bounded walks, and structural filters. L4a implementations MUST
support the `neighbors` operation with depth, direction, and relation
filters, and MUST respect layer boundaries during traversal.

This matrix is a **draft**. L4a inherits all requirements of L1–L3.

## neighbors

### L4a.neighbors.01 — Depth-1 traversal

**What.** `neighbors(n, depth=1)` returns exactly the Nodes directly
connected to `n` by a live edge.

**Spec.** 03-operations §3.4 (neighbors).

**Setup.** A Node `n` with edges to `a`, `b`, `c`.

**Action.** `neighbors(n, depth=1)`.

**Expected.** `{a, b, c}` are returned; no Node more than one hop
away is returned.

### L4a.neighbors.02 — Depth bound respected

**What.** `neighbors(n, depth=k)` does not return Nodes farther than
`k` hops away.

**Spec.** 03-operations §3.4.

**Setup.** A path `n → a → b → c` with `k = 2`.

**Action.** `neighbors(n, depth=2)`.

**Expected.** `{a, b}` are returned; `c` is not.

### L4a.neighbors.03 — Invalid depth rejected

**What.** `neighbors(n, depth=0)` or negative depth raises
`E_INVALID`.

**Spec.** 05-errors §5.4.

**Setup.** A live Node `n`.

**Action.** `neighbors(n, depth=0)`.

**Expected.** `E_INVALID` is raised.

### L4a.neighbors.04 — Relation filter

**What.** `neighbors` with a `rel` filter returns only edges matching
the filter.

**Spec.** 03-operations §3.4.

**Setup.** A Node `n` with `relates_to` edges to `a` and
`derived_from` edges to `b`.

**Action.** `neighbors(n, rel="relates_to")`.

**Expected.** `{a}` is returned; `b` is not.

### L4a.neighbors.05 — Retired edges excluded

**What.** Retired edges MUST NOT contribute to neighbor results.

**Spec.** 03-operations §3.4.

**Setup.** A Node `n` with one live and one retired edge.

**Action.** `neighbors(n)`.

**Expected.** Only the endpoint of the live edge is returned.

## traversal correctness

### L4a.walk.01 — No duplicate nodes in result

**What.** A multi-path walk that reaches the same Node more than
once returns that Node only once.

**Spec.** 03-operations §3.4.

**Setup.** A diamond `n → a → c` and `n → b → c`.

**Action.** `neighbors(n, depth=2)`.

**Expected.** `c` appears exactly once in the result.

### L4a.walk.02 — Source Nodes excluded from walks

**What.** Traversal MUST NOT return Nodes with `kind="source"`.

**Spec.** 01-concepts (retrieval invariant).

**Setup.** A concept Node `n` connected by `attested_by` to a source
Node `s`.

**Action.** `neighbors(n, depth=1)`.

**Expected.** `s` is not in the result.
