# L4b — Intent Conformance Tests

Level 4b adds intent-driven retrieval: `retrieve` accepts a free-text
`intent`, returns hits with scores, and honors `k`/`filters`
combinations per the `retrieve` contract of 03-operations §3.4.4.

This matrix is a **draft**. L4b inherits L1 requirements; it is a
sibling of L4a and MAY be claimed independently.

## retrieve

### L4b.retrieve.01 — Hit carries score field

**What.** Every hit returned by `retrieve` carries a `score` that is
either `null` or a finite float.

**Spec.** 03-operations §3.4, 02-types.

**Setup.** A store with matching concept Nodes.

**Action.** `retrieve(intent, k=5)`.

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

**What.** `retrieve(intent, k=k, filters=F)` returns at most `k`
hits, all of which satisfy `F`.

**Spec.** 03-operations §3.4.

**Setup.** A store with more than `k` Nodes satisfying `F`.

**Action.** `retrieve(intent, k=k, filters=F)`.

**Expected.** `len(results) <= k` and every hit satisfies `F`.

### L4b.retrieve.04 — Non-positive limit rejected

**What.** `retrieve(intent, k=0)` raises `E_INVALID`.

**Spec.** 05-errors §5.4.

**Setup.** None.

**Action.** `retrieve(intent, k=0)`.

**Expected.** `E_INVALID` is raised.

### L4b.retrieve.05 — Empty store returns empty

**What.** `retrieve` on an empty store returns an empty list (not an
error).

**Spec.** 03-operations §3.4.

**Setup.** An empty store.

**Action.** `retrieve(intent)`.

**Expected.** The result is an empty list.

### L4b.retrieve.06 — Unsupported filter operator rejected

**What.** Passing a filter operator outside the §3.4.5 algebra
(`Eq`/`In`/`Range`/`And`/`Or`/`Not`) raises `E_INVALID`.

**Spec.** 03-operations §3.4.5, 05-errors §5.4.

**Setup.** None.

**Action.** `retrieve(intent, filters=Regex(key="attr",
pattern="foo"))`, where `Regex` is not one of the six forms in the
§3.4.5 filter algebra (see also 99-rationale.md Q1, which lists
regex match as an open candidate extension, not yet supported).

**Expected.** `E_INVALID` is raised.

## determinism

### L4b.retrieve.07 — Repeated call stability [INFORMATIVE]

**What.** Two consecutive `retrieve` calls with identical arguments
against an unchanged store return results in the same order. This
test is informative, not normative: 03-operations §3.4.4 does not
require determinism across calls, only that each call's list is
internally ordered by the implementation's relevance estimate.

**Spec.** 03-operations §3.4.4 (informative — no normative
determinism requirement is being tested here).

**Setup.** An unchanged store; no commits between calls.

**Action.** Call `retrieve(intent, k=k)` twice.

**Expected.** Both calls return the same ordered list of refs.
