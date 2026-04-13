# L4b — Intent Conformance Tests

Level 4b adds intent-driven retrieval: `retrieve` accepts a typed
`intent`, returns hits with scores, and honors limit/filter
combinations in a way consistent with the Intent Score Function (ISF)
contract of 03-operations §3.4.

This matrix is a **draft**. L4b inherits L1 requirements; it is a
sibling of L4a and MAY be claimed independently.

## retrieve

### L4b.retrieve.01 — Hit carries score field

**What.** Every hit returned by `retrieve` carries a `score` that is
either `null` or a finite float.

**Spec.** 03-operations §3.4, 02-types.

**Setup.** A store with matching concept Nodes.

**Action.** `retrieve(intent, limit=5)`.

**Expected.** Every result has a `score` field; each value is either
`null` or a finite float (not NaN, not Inf).

### L4b.retrieve.02 — Score ordering monotone

**What.** When scores are non-null, the returned hit list is ordered
by score descending.

**Spec.** 03-operations §3.4.4.

**Setup.** A store with at least three matching Nodes that yield
distinct non-null scores.

**Action.** `retrieve(intent)`.

**Expected.** For any two adjacent hits `h[i], h[i+1]`,
`h[i].score >= h[i+1].score`.

### L4b.retrieve.03 — Limit + filter interaction

**What.** `retrieve(intent, limit=k, filter=F)` returns at most `k`
hits, all of which satisfy `F`.

**Spec.** 03-operations §3.4.

**Setup.** A store with more than `k` Nodes satisfying `F`.

**Action.** `retrieve(intent, limit=k, filter=F)`.

**Expected.** `len(results) <= k` and every hit satisfies `F`.

### L4b.retrieve.04 — Non-positive limit rejected

**What.** `retrieve(intent, limit=0)` raises `E_INVALID`.

**Spec.** 05-errors §5.4.

**Setup.** None.

**Action.** `retrieve(intent, limit=0)`.

**Expected.** `E_INVALID` is raised.

### L4b.retrieve.05 — Empty store returns empty

**What.** `retrieve` on an empty store returns an empty list (not an
error).

**Spec.** 03-operations §3.4.

**Setup.** An empty store.

**Action.** `retrieve(intent)`.

**Expected.** The result is an empty list.

### L4b.retrieve.06 — Unsupported filter operator rejected

**What.** Passing an unsupported filter operator raises `E_INVALID`.

**Spec.** 05-errors §5.4.

**Setup.** None.

**Action.** `retrieve(intent, filter={"attr": {"$xx": 1}})` where
`$xx` is not supported.

**Expected.** `E_INVALID` is raised.

## determinism

### L4b.retrieve.07 — Repeated call stability

**What.** Two consecutive `retrieve` calls with identical arguments
against an unchanged store return results in the same order.

**Spec.** 03-operations §3.4 (informative; ISF is a function of
state).

**Setup.** An unchanged store; no commits between calls.

**Action.** Call `retrieve(intent, limit=k)` twice.

**Expected.** Both calls return the same ordered list of refs.
