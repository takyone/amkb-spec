# Retrieval Theory — AMKB Companion Document

**Non-normative.** This document provides the theoretical background for
AMKB's retrieval-related design choices. It is a companion to the
normative specification in `spec/` and is itself not normative. An
AMKB-compliant implementation need not adopt any theory described here;
conversely, this document may be updated independently of the spec,
under a separate versioning cadence.

Audience: implementers who want to understand *why* the protocol is
shaped the way it is, researchers evaluating AMKB against prior art,
and readers who find the spec's silence on relevance surprising and
want to see the reasoning behind it.

## Contents

1. Vocabulary
2. The Three-Level Hierarchy
3. Observation Frames and Protocol Level
4. ISF Classification by Store-State Dependence
5. ISF Algebra
6. Classical RAG as a Degenerate Case
7. Classical vs Agentic Retrieval — A Theoretical Accounting
8. Near-Duplicate Interference and the Role of Merge
9. Further Reading
10. Open Questions

---

## 1. Vocabulary

AMKB specifies the *shape* of intent-query results. Reasoning about
*how* those results are produced requires vocabulary that this document
introduces. None of these terms appear as normative terms in `spec/`;
they are tools for discussion, not protocol commitments.

- **Intent.** A retrieval request. Its representation is
  implementation-defined — a string, an embedding vector, a structured
  query, or a belief distribution over user goals. The intent space is
  denoted `I`.

- **Retrieval domain** `D ⊆ S`. The subset of Nodes that can appear as
  an intent-query result, where `S` is the store. AMKB constrains `D`
  by the negative rule `kind=source ∉ D` and the positive rule
  `kind=concept ⊆ D`.

- **Intent score** `o ∈ O`. A value in an ordered codomain that an
  implementation assigns to an `(intent, Node)` pair. The codomain may
  be ℝ, a rank, a categorical label, or absent entirely.

- **Intent score function (ISF)** `f: I × D × S → O`. The function an
  implementation uses to compute intent scores. The protocol does not
  specify `f`; implementations choose freely.

- **Set utility function (SUF)** `U: I × 2^D × S → ℝ`. A function on
  *sets* of Nodes rather than individual Nodes. Captures diversity,
  coverage, and other set-level objectives that pointwise ISFs cannot
  express.

- **Retrieval policy** `π`. A state-dependent strategy for producing a
  retrieval result. Actions include scoring subsets, filtering,
  following edges, reformulating the intent, and stopping.

*Relevance* — the intuitive notion of how well a Node answers an
intent — is treated as the informal target that intent scores
approximate. The protocol never defines relevance directly; it only
observes the result shape. This document uses "relevance" only in the
informal sense; formal statements use "intent score".

---

## 2. The Three-Level Hierarchy

Retrieval mechanisms fall into three nested levels, distinguished by
what function of the input they compute.

### Level I — Scoring Function

A Level I mechanism is characterized by an ISF `f` alone:
```
retrieve_I(i, k, S) := top_k_by(f, i, S)
```
The output is the `k` Nodes with highest `f`-score. Examples: BM25,
embedding cosine similarity, learned pointwise rankers, keyword match
counts.

### Level II — Set Utility Function

A Level II mechanism is characterized by a set utility function:
```
retrieve_II(i, k, S) := argmax_{T ⊆ D, |T|=k} U(i, T, S)
```
The output is the `k`-subset maximizing `U`. Level II strictly
subsumes Level I: any ISF `f` induces a separable SUF
`U_f(i, T, S) = Σ_{n∈T} f(i, n, S)`, and `argmax U_f = top_k by f`.

Level II \ Level I contains all mechanisms with non-separable
objectives. MMR (Carbonell & Goldstein 1998) is canonical:
```
U_MMR(i, T) = λ·Σ_{n∈T} Sim(i,n) − (1−λ)·Σ_{n,m∈T, n≠m} Sim(n,m)
```
The pairwise interaction term `Sim(n,m)` cannot be reproduced by any
separable `Σ f(i, n)` decomposition, so MMR is genuinely Level II.
Portfolio IR (Wang & Zhu 2009), xQuAD (Santos et al. 2010), and
diversification methods (Agrawal et al. 2009) live here.

### Level III — Retrieval Policy

A Level III mechanism is characterized by a state-dependent policy:
```
state    = (i, S, history, internal)
action   ∈ { score_topk(subset, f), filter(pred),
             follow(n, rel), reformulate(i'),
             stop(T), ... }
π        : state → action
```
The output is emitted when `π` reaches a `stop(T)` action. Level III
strictly subsumes Level II: any SUF `U` induces a trivial one-step
policy `π_U(state) := stop(argmax_{|T|=k} U(i, T, S))` that computes
the argmax and stops.

Level III \ Level II contains mechanisms whose output depends on
observations made *during* retrieval. Canonical examples:

- **FLARE** (Jiang et al. 2023). Generation-time token confidence
  triggers additional retrieval. The trigger cannot be pre-computed
  from the initial intent.
- **Self-RAG** (Asai et al. 2023). An LLM critic scores retrieved
  items and decides whether to continue. The critic's output depends
  on its own internal state.
- **ReAct-style retrieval** (Yao et al. 2022). Thought–action–observation
  loops where each action is conditioned on prior observations.

### Inclusion Structure

```
Level I ⊊ Level II ⊊ Level III
```

Each inclusion is strict, witnessed by MMR (Level II \ Level I) and
FLARE (Level III \ Level II). AMKB's L4b conformance specifies only
the *shape* of the result set; any level is conformant.

---

## 3. Observation Frames and Protocol Level

The hierarchy above is sensitive to what information an observer has
access to. A system that appears Level III at one observation frame
may appear Level II at another. This section formalizes that
dependency because it matters for benchmarking and conformance.

### 3.1 Observation Frame

An **observation frame** `O` is the set of variables visible to an
observer. Examples:

- `O_protocol_narrow = { intent, k, filter }`
- `O_AMKB = { intent, k, filter, store_snapshot_at_call }`
- `O_agent_wide = O_AMKB ∪ { conversation_history, scratchpad, LLM_state }`

### 3.2 Behavioral Level at a Frame

A retrieval system `R` has **behavioral level** `ℓ` at frame `O` iff
`ℓ` is the minimum level such that `R`'s output distribution can be
characterized using only variables in `O`:
```
Lv-I(O):   ∃ f.  R(o, S) =_d top_k_by(f, o, S)
Lv-II(O):  ∃ U.  R(o, S) =_d argmax_{|T|=k} U(o, T, S)
Lv-III(O): neither of the above
```
where `=_d` denotes equality in distribution and `o` ranges over input
variables in `O`.

### 3.3 The AMKB Protocol Frame

AMKB fixes:
```
O_AMKB := { intent, k, filter, store_snapshot }
```
A retrieval system's **protocol level** is its behavioral level at
`O_AMKB`. Two implementations with identical protocol-level behavior
are interchangeable under the protocol, regardless of how internally
agentic or single-pass one is relative to the other.

### 3.4 Mechanistic vs Protocol Level

A system's *mechanistic level* describes how it computes its result.
A system can be mechanistically Level III (multi-step, tool-using) yet
protocol Level II — its input-output behavior may be characterized by
a static objective over `O_AMKB`. A GraphRAG implementation with a
fixed two-stage plan ("top-3 categories, then top-5 concepts each")
is mechanistically multi-step but behaviorally single-objective.

The distinction matters for:

- **Benchmarking.** Comparisons should be performed at a declared
  frame. AMKB benchmarks operate at `O_AMKB` and compare final output
  lists; internal mechanism is out of scope.
- **Implementation freedom.** Any internal mechanism is permitted as
  long as protocol-level behavior conforms.

Randomness in tie-breaking does not raise the level: the level is
raised only by **feedback observation**, i.e., dependence on variables
outside the frame (generation-time LLM state, unobserved intermediate
results).

---

## 4. ISF Classification by Store-State Dependence

ISFs can be classified by how much of the store state they read:

| Class | Reads | Examples |
|-------|-------|----------|
| **C0** Pointwise | `(i, n)` only | Embedding cosine, dot product |
| **C1** Corpus-aware | `+ corpus statistics` | BM25 (IDF), TF-IDF |
| **C2** Graph-aware | `+ graph topology` | PageRank, spreading activation, graph centrality |
| **C3** History-aware | `+ user/session history` | Personalized reranking, recency weighting |

Classical RAG systems typically operate at C0 or C1. AMKB's retention
of full store state (graph structure, lineage, event history) makes
C2 and C3 ISFs natural; agent-managed merging keeps the store in a
state where such ISFs behave well (§8).

---

## 5. ISF Algebra

ISFs compose. Production systems typically build an ISF from simpler
components:

| Operator | Form | Use |
|----------|------|-----|
| Weighted sum | `f = Σ wᵢ·fᵢ` | Hybrid ranking (BM25 + embed + graph) |
| Max / Min | `f = max(f₁, f₂)` | OR-like / AND-like merge |
| Lexicographic | rank by `f_A`, break ties by `f_B` | Hierarchical priority |
| Filter-then-rank | `{n : g(n) > θ}` then rank by `f` | AMKB's `filter` argument |
| Cascade | top-N by `f_A`, rerank by `f_B` | Two-stage retrieval |
| Ensemble | rank fusion over multiple ISFs | Reciprocal Rank Fusion |

AMKB's `filter` argument on `retrieve` is the filter-then-rank
operator lifted to the protocol surface: the filter narrows `D`, and
the ISF ranks what remains.

---

## 6. Classical RAG as a Degenerate Case

Classical embedding-based RAG in this vocabulary:
```
I          = ℝ^d                  (query embedding space)
D          = {source chunks}       ← first degeneracy
f(i, n)    = cos(i, embed(n))      ← C0 pointwise ISF (second degeneracy)
Output     = top_k by f             ← Level I
```

Two simultaneous degeneracies characterize it:

1. **`D` includes sources.** Classical RAG indexes source material
   and returns source chunks as results. AMKB forbids this:
   `D ∩ sources = ∅`.
2. **ISF is C0 pointwise Level I.** A single similarity computation,
   no corpus / graph / history signals, no set objective, no policy.

AMKB relaxes (1) **structurally** — source is never returned — and
leaves (2) entirely to the implementation. An AMKB store with a C0
ISF reproduces classical RAG behavior on the concept layer while
still preserving source attestation via structural traversal. An
AMKB store with a C2 graph-aware ISF or a Level III policy produces
something classical RAG cannot.

The relaxation of (1) is the *structural* contribution of AMKB; the
freedom over (2) is the *theoretical* contribution.

---

## 7. Classical vs Agentic Retrieval — A Theoretical Accounting

Empirically, agentic RAG systems (Self-RAG, FLARE, ReAct-retrieval,
GraphRAG) outperform classical embedding RAG on ambiguous, multi-hop,
and knowledge-intensive queries. This section accounts for the gap
in terms of the framework above.

### 7.1 Where the gap comes from

Three independent contributions compose the gap:

**(a) Expressive reach.** Classical RAG is Level I; agentic RAG is
Level III. Strict inclusion means agentic RAG can express retrieval
functions classical RAG cannot: set-level objectives (diversity,
coverage), observation-dependent branching, iterative
disambiguation. For any query whose optimal result is not the
pointwise top-k, classical RAG is architecturally unable to reach it.

**(b) Epistemic–pragmatic trade.** Classical RAG performs a single
pointwise readout and collects no evidence beyond its initial input.
Agentic RAG can spend a retrieval step to *reduce uncertainty* about
the user's intent (epistemic value) before spending later steps to
satisfy it (pragmatic value). In active-inference terms (Friston
2010), classical RAG has no mechanism to minimize expected free
energy; agentic RAG approximates that minimization directly.

**(c) Near-duplicate recovery.** When the concept layer contains
near-duplicate entries, classical RAG's pointwise readout degrades
predictably (§8). Agentic RAG can respond to an ambiguous result by
*reformulating* the query or *descending* into neighbors, bypassing
the ambiguity zone. The level-up enables recovery paths from the
failure modes of Level I.

### 7.2 When the gap narrows

The framework predicts *when* agentic RAG is worth its extra cost:

- **Gap is small:** unambiguous query AND well-separated concept
  layer (`Δ(X)` large, §8) AND single-call budget. Classical RAG is
  near-optimal; Level-III expressive gain is unused.
- **Gap is large:** ambiguous query OR dense/under-merged concept
  layer OR multi-call budget. Each additional step contributes
  information gain that classical RAG cannot access.

This is an empirical hypothesis: systems that plot retrieval accuracy
as a function of concept-layer density and query entropy should
observe a crossover. Single-number benchmarks average over the full
distribution and hide the structure.

### 7.3 Why agentic RAG is not universally better

Three forces push against agentic RAG:

- **Latency.** Each retrieval step adds LLM calls. For unambiguous
  queries these steps produce no marginal gain.
- **Error compounding.** Multi-step policies can hallucinate
  intermediate beliefs and retrieve against the hallucination.
  Level-III policies have lower variance only when their
  epistemic-value estimator (typically an LLM critic) is
  well-calibrated.
- **Training overhead.** Learned Level-III policies require training
  signals harder to specify than pointwise relevance judgments.

### 7.4 Implication for AMKB design

AMKB's operation set is chosen as a minimally complete action space
for Level III policies: `get`, `find_by_attr`, `neighbors`, `retrieve`.
A policy `π` runs on an AMKB store by issuing these primitives;
classical RAG is the degenerate policy that issues one `retrieve` and
stops. The same AMKB store can back both classical and agentic
consumers without architectural change. The protocol does not
privilege one over the other — it provides the substrate both need.

---

## 8. Near-Duplicate Interference and the Role of Merge

### 8.1 The general thesis

Any retrieval mechanism that treats concept Nodes as individual
competing units degrades in quality as near-duplicate concepts
accumulate. The degradation takes different forms for different ISF
classes:

| ISF class | Mode of degradation |
|-----------|---------------------|
| C0 pointwise | Pattern interference, ambiguous readout |
| C1 corpus-aware | Term/IDF washout, signal dilution |
| C2 graph-aware | Centrality splitting, diluted paths |
| C3 history-aware | Context-length bloat, attention dilution |

The AMKB `merge` operation, combined with `derived_from` lineage
union, is the minimal protocol-level primitive that reverses this
degradation without losing information.

### 8.2 Hopfield instance — a quantitative form for C0

For C0 ISFs, Modern Hopfield Networks (Ramsauer et al. 2020) provide
a quantitative version of the general thesis.

Let the concept layer hold `N` embeddings
`X = [ξ_1, ..., ξ_N] ∈ ℝ^{d×N}`. Define the minimum pattern
separation:
```
Δ(X) := min_i [ ||ξ_i||² − max_{j≠i} ⟨ξ_i, ξ_j⟩ ]
```
Ramsauer Theorem 3 (simplified): for a query `ξ` near pattern `ξ_i*`
with noise `ε`, the one-step Hopfield retrieval error is bounded by
`O(exp(−β·(Δ(X) − 2M·‖ε‖)))` where `M = max_i ‖ξ_i‖` and `β` is the
inverse-temperature parameter. When `Δ(X) → 0` the bound saturates
and retrieval becomes unreliable at any `β`.

Ingestion without merging drives `Δ(X)` toward 0 whenever source
content is dense in embedding space: new concepts fall arbitrarily
close to existing ones, reducing separation. A greedy merge policy
that consolidates any pair with cosine similarity above `1 − δ` keeps
`Δ(X)` bounded below by `δ · min_i ‖ξ_i‖²` indefinitely.

**Consequence.** For C0 AMKB implementations, merge is not a cleanup
convenience. It is a *necessary* operation for retaining retrieval
quality under continuous ingestion. The general thesis holds for all
ISF classes; the Hopfield instance provides a quantitative form for
embedding-based retrieval specifically. Analogous quantitative
bounds for C1–C3 are open.

### 8.3 Implications

- Curator agents should monitor pairwise similarity and merge when
  thresholds are exceeded.
- The merge threshold is a tunable parameter trading capacity
  retention against concept granularity.
- AMKB's lineage preservation (`derived_from` union on merge) makes
  aggressive merging safe: reverts can restore the pre-merge state
  when the agent's merge decision was wrong.

---

## 9. Further Reading

**Classical IR foundations**

- Robertson & Sparck Jones (1976). Relevance weighting of search terms.
- Salton (1968). Automatic Information Organization and Retrieval.
- Robertson (1977). The probability ranking principle in IR.

**Set-level objectives and diversification**

- Carbonell & Goldstein (1998). The use of MMR, diversity-based reranking for reordering documents and producing summaries.
- Wang & Zhu (2009). Portfolio theory of information retrieval. SIGIR.
- Agrawal, Gollapudi, Halverson & Ieong (2009). Diversifying search results. WSDM.
- Santos, Macdonald & Ounis (2010). Exploiting query reformulations for web search result diversification. WWW. (xQuAD)
- Liu (2009). Learning to rank for information retrieval. Foundations and Trends.

**Agentic and multi-step retrieval**

- Yao et al. (2022). ReAct: Synergizing reasoning and acting in language models.
- Asai et al. (2023). Self-RAG: Learning to retrieve, generate, and critique through self-reflection.
- Jiang et al. (2023). Active retrieval augmented generation. (FLARE)
- Edge et al. (2024). From local to global: A graph RAG approach to query-focused summarization.

**Memory and neuroscience analogies**

- Ramsauer et al. (2020). Hopfield networks is all you need. arXiv:2008.02217.
- McClelland, McNaughton & O'Reilly (1995). Why there are complementary learning systems in the hippocampus and neocortex. Psychological Review.
- Friston (2010). The free-energy principle: a unified brain theory? Nature Reviews Neuroscience.

---

## 10. Open Questions

- **Q1.** AMKB leaves the intent space `I` unspecified. Should the
  protocol eventually provide an interchange format for intents to
  enable cross-implementation benchmarks? (See §3.)
- **Q2.** Level III conformance is defined at `O_AMKB` only.
  Implementations may internally observe richer frames, but the
  protocol cannot observe those differences. Is this the right cut?
- **Q3.** The Hopfield merge argument (§8.2) applies cleanly to C0
  ISFs. Can analogous quantitative bounds be derived for C1–C3?
- **Q4.** This document avoids specifying how an implementation
  should compute ISFs or policies. Is that silence a design strength
  (protocol minimalism) or a testing liability (hard to benchmark)?
