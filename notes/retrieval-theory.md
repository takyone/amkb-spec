# Retrieval Theory — 作業メモ

非規範。spec化前のたたき台。AMKBの "Intent query" を支える理論的土台を整理し、穴を洗い出すためのメモ。

---

## 1. 基礎オブジェクト

| 記号 | 名前 | 説明 |
|---|---|---|
| `S` | Store | AMKB instance。Node/Edge/layer を持つグラフ |
| `t` | Snapshot time | `S` は時間に依存して変わる。`S_t` と書くべき場面あり |
| `I` | Intent space | 検索要求の集合。中身は protocol 非規定 |
| `i ∈ I` | Intent | 1つの検索要求 |
| `D ⊆ S` | Retrieval domain | intent query が返し得る Node の集合。AMKB制約: `kind=source ∉ D`, `kind=concept ⊆ D` |
| `n ∈ D` | Candidate Node | スコア対象 |
| `O` | Score codomain | intent score の値空間。ℝ / rank / categorical / ⊥ |
| `f` | Intent Score Function (ISF) | `f: I × D × S_t → O` |

ISFが取る引数はこの4つ。`S_t` を陽に書くのが大事で、BM25のIDF、graph centrality、user historyなどは全部 "`S_t` から読む情報" として統一できる。

---

## 2. Intent Score Function の分類 (store-state 依存度)

ISFは `S_t` のどこまでを読むかで4段階に分類できる。これが一番キレイな軸:

| Class | 依存 | 例 |
|---|---|---|
| **C0 Pointwise** | `(i, n)` のみ | 埋め込み cos類似度 (`embed(i)·embed(n)`) |
| **C1 Corpus-aware** | + コーパス統計 | BM25 (IDF), TF-IDF |
| **C2 Graph-aware** | + グラフ構造 | PageRank加重, spreading activation, neighbor propagation |
| **C3 History-aware** | + 利用者履歴 | 個人化rerank, FSRS状態の利用 (※) |

※ FSRS *state* を ISF の入力に使うのはAMKB内で可能 (例: "ユーザーが最近復習したNodeを優先"). ただしFSRSの**スケジューリング決定** (次いつ見せるか) はAMKB外。**"retrievalに使う" のはOK、"whenを決める" のは別レイヤー**。ここ混同しやすい。

この分類の利点:
- 実装を見るとき「このシステムは C2 か C0 か」で本質が分かる
- 古典RAGはほぼ C0 一択に張り付いていた歴史、という見立てができる
- AMKBの新規性は **C2/C3 を本気で使うことを agent-management で現実的にした** 点、と言語化できる

---

## 3. Codomain の一般化 — スコアではなく preorder

ISF の出力を ℝ に固定するのは狭い。より一般には:

> **ISFは intent 条件下の Node preorder を誘導する関数である**

- Scalar score: 全順序の誘導。top-k が well-defined
- Rank: 順序のみ。`score = None` 運用はここ
- Categorical ("relevant / somewhat / not"): partial order
- Binary (LLM-judge が yes/no): 2階層の partition
- Distribution: `p(n | i)` → 確率で順序誘導

spec で `RetrievalHit.score: float | None` にしたのはこの preorder 抽象の帰結。protocolは「返り値リストが preorder を表現する順序を持つ」ことだけ要求すればよい。

**これ、R13 追補で書けるクリーンな一般化だと思う。**

---

## 4. ISFの代数 — 合成オペレータ

実装者は複数のISFを組み合わせる。名前を付けておくと見通しがよい:

| 演算 | 定義 | 用途 |
|---|---|---|
| **Weighted sum** | `f = Σ wᵢ·fᵢ` | Spikuit流 (BM25 + sim + centrality) |
| **Max / Min** | `f = max(f₁, f₂)` | OR/AND 的なmerge |
| **Lexicographic** | `f_A 優先、tie は f_B で割る` | 階層的優先 |
| **Filter-then-rank** | `{n : f_A(n) > θ}` を f_B で並べる | `retrieve(filter=..., intent=...)` の意味論 |
| **Cascade** | `f_A` で top-N → `f_B` で rerank | 2-stage retrieval |
| **Ensemble** | 複数ISFの投票/rank fusion | Reciprocal Rank Fusion など |

AMKBの `filter` 引数は実は **Filter-then-rank の filter部分を protocol が規定している** という見方ができる。「filter は intent scoring の *前* に走る」という現在の記述は、この代数の特殊ケースとして位置づけ直せる。

---

## 5. 古典RAGの退化ケース化

古典的な embedding-based RAG をこのフレームで書くと:

```
I          = ℝ^d (query embedding space)
D          = {source chunks} ← ここが問題
f(i, n)    = cos(i, embed(n))    (C0, pointwise)
output     = top-k by f
```

退化点が2つ明確になる:
1. **D = sources** であること (AMKB: `D ⊇ concepts, D ∩ sources = ∅`)
2. **ISFクラスが C0** に張り付いていること

AMKBが分離したのは (1) だけで、(2) は実装自由。つまりAMKBは **"D の形を変えた" 拡張** が本質で、ISFクラスの拡張は副次的。この切り分けは重要。

---

## 6. Spikuit = 具体化された ISF + retrieval domain

Spikuit の retrieval は以下の具体化:

```
I          = 自然言語文字列
D          = L_concept ∪ L_category
f(i, n, S) = w1·BM25(i, n)
           + w2·semantic_sim(embed(i), embed(n))
           + w3·graph_centrality(n, S)
           + w4·activation_bonus(i, n, S)    ← 近傍からの伝播
           + ...
output     = top-k
```

これは C2 (graph-aware) ISFの例。AMKBはこの具体的重み構成を要求しないが、Spikuitは1つの選択をしている。

**Spikuit のうち AMKB外のもの**:
- FSRS次回復習時刻決定 (scheduling、intentを入力に取らない)
- 復習セッションUI (Tutor layer)
- Quiz生成 (これは ISF ではなく content generation)

→ **Spikuit = AMKB-compliant Brain + Tutor layer + Content generator** の3層。AMKBはBrain部分の契約にすぎない。

---

## 7. 穴・未解決・要議論

### H1. Intent space I は未規定でよいか

現状spec: "intent" の構造を規定しない。pros/cons:
- ✅ 実装自由 (string / struct / embedding どれでも)
- ❌ 異なる実装間で intent互換性がない → benchmark比較ができない
- ❌ agentが `store_A.retrieve(i)` → 結果を `store_B.retrieve(i')` に繋ぐ migration シナリオで詰む

**未解決**: 最小限の intent interchange format を後から追加する余地を残すべきか? (optional conformance)

### H2. Set-level objective が pointwise ISFで表現できない

MMR (Maximum Marginal Relevance), diversity, coverage は **(intent, 選択集合) の関数** であって (intent, Node) の関数ではない。現状の `ISF: I × D × S → O` 枠組みではこれらを第一級で扱えない。

**選択肢**:
- (a) 無視する: ユーザー側で iterative に呼ぶ想定
- (b) **Set Utility Function (SUF)**: `U: I × 2^D × S → ℝ`, retrieval = argmax_{T ⊆ D, |T|=k} U(i, T, S) と一般化する。ISF は `U(i, T) = Σ f(i, n)` の特殊ケース
- (c) iterative ISF: `f_k(i, n, S, 既選択集合)` と、前ステップ依存の形で許す

**個人的意見**: (b) が理論的には一番キレイ。ISF は SUF のセパラブル特殊ケース。ただ実装者が書く分には (a) + 将来 (b) が現実的。

### H3. 時間依存の明示

ISFが `S_t` に依存するのは自明だが、これを spec に書くか。書くと「retrieve の結果は同じ intent でも異なるsnapshotで異なり得る」と明文化できる。event subscriptionとの関係 (snapshot reasoning) と地続きになる。

### H4. 学習型ISFと component decomposition の非共存

Weighted-sum 流の説明は「pointwise特徴量の線形和」を暗黙前提にしている。しかし LLM-judge ISF や learned cross-encoder は **decomposable でない** (black-box で、重み分解できない)。

Retrieval Theory としては:
- ISFは **任意の写像** として定義 (decomposability を要求しない)
- **ある種のISFは decomposable という追加性質を持つ** (Spikuitの weighted sum はこの族)
- **観測可能性** は別問題: agentがISFの内部を見られるかは protocol の外

このレベルで整理すれば両立する。R13補遺で「ISF = black box 関数。内部構造は protocol 非規定」と書けばよい。

### H5. "Relevance" という語の位置

このメモでは "intent score" を物的な用語として使い、"relevance" は **intent score が近似しようとしている直観概念** として扱っている。
- relevance: 人間が直感する「関連性」
- intent score: それを近似するために ISF が計算する具体値
- ISF: "query に関連している蓋然性" の proxy estimator

これはspec本文でも保てる切り分け: "relevance" は直感、"intent score" は計算可能物。

### H6. Retrieval domain の時間変化

D は `kind=source ∉, kind=concept ⊆` という静的制約だが、concept Nodeの retire は D を縮める。retirementはtombstoneされるだけで Dには含まれない想定だが、lineage traversalでは到達可能。このあたりの "D vs reachable set" の区別を整理する余地あり。

### H7. 神経科学アナロジーの理論化

Nodes=neurons, Edges=synapses, intent=input pattern, spreading activation=dynamics, ISF=readout。
- Pointwise ISF = zero-step readout (入力パターンと各Nodeの内積)
- Graph-aware ISF = k-step readout (k回伝播した後の activation)
- Spikuit = 有限step spreading + weighted readout

この「ISFは グラフ上の dynamical system の readout関数」という言い換えは R13 の Spikuit 例に美しく接続する。古典RAGの ISF は zero-step 読み出し、という対比が決まる。

---

## 8. 理論の骨格 (整理後の章立て候補)

spec/01-concepts.md に短い "Retrieval Theory Primer" sectionとして入れる案、あるいは別chapter (例: spec/04-retrieval-theory.md) にする案、どちらもあり得る。

**案A: 01-concepts に1節追加 (軽量)**
- "Intent query" subsection の末尾に vocabulary定義だけ置く
- I, D, f の形式的シグネチャ
- 古典RAGが退化ケースであることの1段落

**案B: 独立 chapter (重量)**
- §1 Basic objects
- §2 ISF signature, preorder codomain
- §3 Store-state dependence hierarchy (C0–C3)
- §4 ISF algebra
- §5 Set utility generalization (H2への返答)
- §6 Classical RAG degenerate case
- §7 Non-normative implementation examples (Spikuit)
- §8 Open questions (H1–H7)

**個人の推し**: AMKBが L4b を "shape only" にしている以上、Retrieval Theory を厚めに書いて **"shape only" の正当化を理論的に与える** のは筋が通る。案B寄り、ただし00-overviewの "非目標: relevance estimation" と密に cross-ref する。

---

## 9. 次の議論ポイント (旧)

> **Status 2026-04-13**: H2 は §10 の階層モデルで resolved。H1 は §11 で distribution lift により部分的に resolved。残る open は §14 に再掲。

1. ~~H2 (set-level objectives) をどう扱うか~~ → §10 で Lv-II/III階層で解消
2. ~~H1 (intent space未規定) を protocol化するか~~ → §11 で Δ(I) lift により neutral に解決
3. 理論章は 01 embed vs 独立 chapter どちらにするか → 案B 独立 chapter 推し
4. C0–C3 classification を spec本体に書くか rationale 止まりか → rationale + 独立chapterで扱う
5. H7 (dynamical readout view) を spec に書くか、それとも Spikuit 側 README に書くか → §13 で深掘り

---

## 10. Lv-II / Lv-III の境界を形式化する

### 10.1 階層の inclusion 構造

3つのレベルを厳密に書き下す:

**Lv-I (Scoring Function)**:
```
f : I × D × S → O
retrieve_I(i, k, S) := top_k_by(f, i, S)
```
出力は `(i, S)` の deterministic function (tie-breakingを除く)。

**Lv-II (Set Utility Function)**:
```
U : I × 2^D × S → ℝ
retrieve_II(i, k, S) := argmax_{T ⊆ D, |T|=k} U(i, T, S)
```
出力は `(i, S)` の deterministic function。**argmax は implementation が近似してよい** (greedy, beam, etc.)。

**Lv-III (Retrieval Policy)**:
```
state    = (i, S, history, internal)
action   ∈ { score_topk(pred, f), filter(pred), follow(n, rel), 
             reformulate(i'), stop(T), ... }
π        : state → action
obs      : action × S → observation
history' = history ++ (action, obs)
```
出力は state trajectory の終端で emit される `stop(T)`。**出力が `(i, S)` のみの関数とは限らない** — policy の内部乱数、観測順序、中断条件に依存する場合がある。

### 10.2 Inclusion の証明スケッチ

**Lemma 1 (I ⊊ II)**. 任意の Lv-I `f` に対し separable SUF `U_f(i,T,S) := Σ_{n∈T} f(i,n,S)` を取れば `argmax U_f = top_k(f)`。よって Lv-I ⊆ Lv-II。

**Strict**: MMR の SUF
```
U_MMR(i, T) = λ·Σ_{n∈T} Sim(i,n) − (1−λ)·Σ_{n,m∈T, n≠m} Sim(n,m)
```
は non-separable (interaction項を含む)。separable な `f` が存在すれば Σ f で `U_MMR` を書けるが、interaction項があるので不可能。よって `U_MMR ∉` Lv-I、Lv-I ⊊ Lv-II。

**Lemma 2 (II ⊊ III)**. 任意の Lv-II `(U, k)` に対し policy
```
π_U,k(state) = stop(argmax_{|T|=k} U(i, T, S))
```
を取れば 1 step で停止し Lv-II と同じ出力。state 空間は自明。よって Lv-II ⊆ Lv-III。

**Strict**: FLARE (Jiang et al. 2023) は LLM 生成中に token 信頼度を観測し、低信頼度時に追加 retrieve を実行する。追加 retrieve の query は 前ステップの生成観測に依存する。この依存は `(i, S)` のみから決まらない (生成器の内部状態に依存) ので、**ある固定 U を事前に定義して argmax を取る** という書き方ができない。よって Lv-II ⊊ Lv-III。

### 10.3 核心の demarcation

境界の本質は **"中間観測に対する適応性"**:

> Lv-II iff **出力が `(i, S)` のみで決まる** (static objective で書ける)
> Lv-III iff **出力が retrieval 中の観測に依存する** (adaptive, feedback-driven)

この区別は機構 (mechanism) ではなく **振る舞い (behavior)** で決まる。iteration, beam search, greedy approximation を内部で使っていても、input-output が `(i, S)` で決まっていれば Lv-II。

### 10.4 ややこしいケース — "behavioral Lv-II, mechanistic Lv-III"

現実のシステムは多くが **実装上は multi-step (mechanistic Lv-III) だが、input-output 振る舞いとしては static objective で書ける (behavioral Lv-II)**:

- **GraphRAG (fixed plan)**: 常に top-3 カテゴリ → 各 top-5 concept、という固定プラン。出力は `(i, S)` のみで決まる → **behavioral Lv-II**。
- **GraphRAG (adaptive plan)**: カテゴリ score が閾値を超えたものだけ descent。出力が閾値通過パターンに依存 → **behavioral Lv-III**。
- **Self-RAG**: LLM が "retrieve すべきか" を critic する。critic 出力は LLM 内部状態に依存 → **behavioral Lv-III**。

この区別は重要で、**benchmark 的には behavioral level を見るべき**。mechanistic 階層は実装選択の自由度を表すにすぎない。

### 10.5 含意

1. **Protocol が観測する level は behavioral のみ**。AMKB の `retrieve` は 1 コールで結果を返すので、呼び出し側から見れば全てが behavioral Lv-II (以下)。Lv-III の真価は **複数の AMKB op を組み合わせる policy** を書く段階で現れる。
2. **Benchmark は behavioral level で比較する**。同じ `(i, S)` に対して出力が決まるなら、mechanism 違いは隠蔽してよい。
3. **Spikuit のような単実装 retrieve は behavioral Lv-II**。内部で spreading activation を回しても、外から見れば `(i, S) → T` の関数。Lv-III になるのは agent が **複数回 retrieve を呼ぶ use pattern** の時だけ。

---

## 11. Intent distribution lift — H1 との合流

### 11.1 xQuAD と intent mixture

Santos et al. (2010) の xQuAD は intent に hidden sub-intent 分布を仮定する:
```
p(s | i) : sub-intent distribution
U_xQuAD(i, T) = (1−λ)·rel(i,T) + λ·Σ_s p(s|i)·(1 − ∏_{n∈T}(1 − rel(n,s)))
```
"apple" が fruit と company のどちらを指すか不確実な時、両方の sub-intent をカバーするよう T を選ぶ。

これは Lv-II の自然な拡張として書ける:
```
U_mixed(μ, T, S) := E_{i ~ μ}[ U(i, T, S) ]
```
ただし `μ ∈ Δ(I)` は intent space 上の分布。

### 11.2 Δ(I) lift

より一般に: **intent space `I` を distribution lift する**:
```
I* := Δ(I)    (I 上の probability measures)
f*(μ, n, S) := E_{i~μ}[ f(i, n, S) ]
U*(μ, T, S) := E_{i~μ}[ U(i, T, S) ]
π*(μ, S, h) := ... (policy over beliefs)
```

これは Bayesian decision theory の標準的 lift で、特別なことは何もしていない。ただ protocol 的に重要なのは **`I` が何であっても `Δ(I)` は自動的に存在する** こと — AMKB は `I` を未規定のままにできる。

### 11.3 H1 への回答

H1: "intent space I は未規定でよいか" に対する理論的回答:

> **Yes, 未規定でよい。ただし "intent は分布的である可能性がある" ことは明記する。**

これにより:
- 実装は point intent (`i ∈ I`) を受け取る simple API を提供してよい
- 実装は uncertain intent (`μ ∈ Δ(I)`) を受け取る rich API を提供してもよい
- 両者は同じ protocol の下で共存できる (point = Dirac delta としての Δ(I) 要素)

**agent の実装指針**: agent が user の意図について不確実な時、point intent を複数回 retrieve するより、**belief distribution を 1 回の expected-utility retrieve に圧縮する** 方がクリーン。ただし現状 protocol は point intent のみ。将来の optional extension として distribution intent を検討する余地あり。

### 11.4 Active inference との接続

Friston の active inference では、agent は世界モデル上の belief `q(s)` を持ち、行動選択は expected free energy 最小化で行う:
```
G(π) = E_q[ log q(s|π) − log p(s, o | π) ]
     = epistemic value + pragmatic value
```
retrieval を action と見れば、各 retrieve 呼び出しは `q(goal | i, history)` を更新する観測行為。**Lv-III policy = active inference agent の retrieval module**、という対応が自然。

これは philosophical framing にすぎないが、**"agentic retrieval は belief-driven policy である"** という立場を正当化する。Self-RAG や FLARE が経験的に成功している理由を、active inference は「epistemic value を考慮する policy が好かれる」と説明できる。

---

## 12. AMKB action space の完全性

### 12.1 問い

AMKB の現 operation set `{ get, find_by_attr, neighbors, retrieve }` は、Lv-III retrieval policy の action space として完全か?

### 12.2 分類

policy が取りたい action を store-side と agent-side に分ける:

**Store-side (AMKBが提供すべき)**:

| Action | AMKB op | 状態 |
|---|---|---|
| Exact-ID fetch | `get(ref)` | ✅ |
| Deterministic filter | `find_by_attr(pred)` | ✅ |
| Graph traversal | `neighbors(n, rel, depth)` | ✅ |
| Fuzzy ranking | `retrieve(i, k, filter)` | ✅ |
| Count without fetch | — | ❌ Not in AMKB |
| Shortest path | — | ❌ Not in AMKB (composable) |
| Random sample | — | ❌ Not in AMKB |
| Explain ranking evidence | — | ❌ Not in AMKB |

**Agent-side (AMKBの責務ではない)**:

| Action | 説明 |
|---|---|
| `reformulate(i) → i'` | query rewriting。agent の内部プロンプトで処理 |
| `decompose(i) → [s]` | sub-query 生成。agent 内 |
| `critique(n, i) → score` | LLM-judge。agent 内で LLM 呼び出し |
| `synthesize(T) → answer` | 最終回答生成。retrieval の外 |

agent-side は store に触らないので protocol の操作ではない。**AMKB は store 面だけ規定する**。

### 12.3 完全性テーゼ (非形式的)

> **Thesis**: 任意の Lv-III retrieval policy π は、store に触る部分を AMKB の 4 op で書ける。agent-side の action は実装者の自由。

**反証となる候補**:

1. **`count(filter)`**: "マッチ数が 1000 を超えたら絞り込みモードに入る" のような policy。現状は `find_by_attr(filter)` して長さを数えるしかない。→ materialize コストが嵩む。
2. **`sample(kind, n)`**: exploration。現状は `find_by_attr(kind=X)` して agent 側で shuffle。→ store が大きい時コスト大。
3. **`explain(retrieve_call) → evidence`**: ranking 根拠の取得。現状は hidden。→ protocol 的に leak 禁止にしたが、policy が次の retrieve を決めるのに欲しい場合がある。
4. **`path(n1, n2, constraints)`**: graph path。現状は `neighbors` 反復で実装。→ 効率問題。

評価:
- 1, 2, 4 は **効率問題であって表現力問題ではない**。agent が頑張れば composition で書ける。→ protocol の核には入れない。L4a のサブレベル or optional extension で後付け可能。
- 3 は微妙。**ranking evidence を protocol-level で観測可能にするか** は哲学的決定。現 R13 の立場 ("relevance は impl 定義") と整合させるには、**evidence も impl 定義** として optional にするのが無難。

### 12.4 セッション状態

もう一つの候補: **session 抽象** — 既に返した Node を覚えておき、"まだ見ていないもの" を返すモード。選択肢:

- **(a) Stateless**: agent が seen-set を自前で持ち `filter=NOT IN seen` として渡す。protocol は変更不要。
- **(b) Server session**: `session = begin_session(); session.retrieve(i)` がサーバー側で seen を追跡。
- **(c) Cursor**: `retrieve` が cursor を返し、advance する。

現状 (a)。agent-side で十分扱える。(b)/(c) は複数クライアント共有や長時間 session に欲しくなるが、v0.x では不要。

### 12.5 結論

**AMKB の現 4-op core は Lv-III policy の action space として "表現力完全、効率は非最適"**。

Rationale への追加案 (R14 候補):

> **R14 (candidate)**: The 4-operation retrieval-side core (`get`, `find_by_attr`, `neighbors`, `retrieve`) is chosen to be minimally complete as an action space for Lv-III retrieval policies. Quality-of-life primitives (`count`, `sample`, `path`, `explain`) are expressible via composition and are deferred to optional extensions. Policy-level actions (reformulation, critique, synthesis) are agent-internal and outside protocol scope.

---

## 13. 神経科学アナロジーの厳密化

アナロジーを3本の柱で整理する。厳しい方から順に: Hopfield (数学的), CLS (生物学的), Active Inference (哲学的)。

### 13.1 Modern Hopfield Networks — ISF の数学的基盤

Ramsauer et al. (2020) "Hopfield Networks Is All You Need" (arXiv:2008.02217):

Classical Hopfield: `x ← sign(Σ ξ_i ξ_i^T x)` が最近接格納パターンに収束。容量 O(N)。

Modern Hopfield: continuous state, update rule
```
x_new = X softmax(β X^T x)
```
ここで `X = [ξ_1, ..., ξ_N]` は格納パターン行列。これは **transformer attention 1 step と同一**。容量は指数的。

**AMKB との対応**:
- Nodes = 格納パターン ξ_i
- Node embedding = ξ_i (content の数値表現)
- Intent embedding = query x
- **Pointwise ISF = 1-step modern Hopfield readout** = `softmax(β ⟨x, embed(n)⟩)`
- **Graph-aware ISF = k-step Hopfield with graph-structured propagation** = neighbors で pattern を混ぜながら収束させる
- **Spikuit の spreading activation = k-step Hopfield on concept graph**

**定理的含意**: Modern Hopfield の指数容量は、AMKB の concept layer が **埋め込み次元より指数的に多くの concept を catastrophic interference なしに保持できる** ことを意味する。ただし **パターン分離 (inter-concept distance)** が崩れると容量が急落する。

**設計への予測**:
1. **merge は Hopfield capacity を守る操作**。重複 concept を merge するのは冗長性除去ではなく、pattern 干渉回避による容量保持。
2. **concept embedding は互いに直交に近いほうがよい**。つまり ingestion agent は「既存 concept と区別される atomic claim」を抽出すべき、という設計指針に数学的根拠が与えられる。
3. **L_category は第 2 層 Hopfield**。category レベルで粗い attention を掛け、選ばれた category 内で細かい attention を掛ける 2-stage Hopfield readout として解釈可能。GraphRAG の hierarchical retrieval と一致。

これは非常に強い: **merge と ingest granularity という AMKB 側の疑似的設計判断に、Modern Hopfield が数学的正当化を提供する**。R13 / R14 rationale で真面目に引ける。

### 13.2 Complementary Learning Systems — layer 構造の生物学的基盤

McClelland, McNaughton, O'Reilly (1995) "Why there are complementary learning systems in the hippocampus and neocortex":

Thesis: 哺乳類の脳は 2 つの相補的記憶系を持つ:

| 系 | 特性 | 役割 |
|---|---|---|
| **Hippocampus** | Fast learning, sparse, pattern-separated, episodic | 個別事象の即時記憶 |
| **Neocortex** | Slow learning, dense, pattern-completing, semantic | 統計的規則性の抽出 |

なぜ 2 系必要か: 密な表現上の即時学習は catastrophic interference を起こす。fast-sparse 系と slow-dense 系を分離し、consolidation (hippocampus → neocortex の漸進的転送) で両立させる。これが **stability-plasticity dilemma** の生物学的解決。

**AMKB との対応**:
- **L_source ≈ Hippocampus**: 個別 source を pattern-separated に保持。各 source は distinct な tombstone 付き identity を持つ。retrieval targetではない (sparse access)。
- **L_concept ≈ Neocortex**: source 群から抽出された semantic generalization。retrieval dense target。
- **Ingestion = Consolidation**: agent が source を読み concept を抽出する過程は、hippocampus → neocortex の systems-level consolidation そのもの。
- **L_category ≈ Schematic memory**: 高次スキーマ。neocortex の抽象化機能の上層。

**設計予測**:
1. **source と concept を分離する理由は catastrophic interference 回避**。1 つの store に source chunks と concepts を混在させると、新規 source ingest のたびに concept 表現が干渉を受ける。**AMKB の layer 分離は単なる整理ではなく、計算論的安定性のため**。
2. **agent-managed ingestion = 人工 consolidation**。睡眠中の hippocampal replay が agent の ingestion 工程に対応する。継続的に新しい source を読み、既存 concept と統合 (merge) / 対比 (contradict) / 拡張 (extend) する工程は、consolidation dynamics の工学版。
3. **source の retirement ≠ deletion**。hippocampus は consolidation 後も episodic memory を一定期間保持する。AMKB が retirement を tombstone (deletion でない) としたのは、lineage 保持だけでなく **consolidation 遅延への備え** としても解釈できる。

### 13.3 Active Inference — policy 層の哲学的基盤

Friston (2010) "The free-energy principle: a unified brain theory?" (Nature Reviews Neuroscience):

Thesis: agent は variational free energy を最小化する。perception = posterior inference、action = expected free energy を最小化する policy 選択。

```
F = E_q[log q(s) − log p(s, o)]    (free energy)
G(π) = E_q[log q(s|π) − log p(s, o|π)]
     = -epistemic value - pragmatic value  (expected free energy)
```

**Retrieval への対応**:
- Agent は user の goal について belief `q(goal | i, history)` を持つ
- 各 `retrieve(i)` 呼び出しは **belief を更新する観測行為**
- policy π は epistemic value (belief の不確実性減らし) と pragmatic value (goal 達成) のトレードオフで action を選ぶ
- Self-RAG, FLARE は暗黙に expected free energy を近似している

**含意**:
1. **agentic retrieval の根拠**。なぜ multi-step retrieval が単発 retrieval より良いのか? → epistemic value を段階的に積むから。active inference はこれを formal に説明する。
2. **retrieval budget の理論化**。何回 retrieve するかは expected free energy reduction とコストのバランスで決まる。これは empirical tuning でなく原理的に導ける。
3. **critic の位置**。Self-RAG の critic = posterior entropy estimator。LLM confidence は free energy の proxy。

Active inference はこの水準では metaphor に近く、actionable ではない。ただし **"agentic retrieval は Bayesian-optimal policy の工学的近似"** というスタンスを protocol design に持たせるには十分。

### 13.4 3 本の使い分け

| Analogy | 水準 | spec での扱い |
|---|---|---|
| Modern Hopfield | 数学的・厳密 | R13/R14 rationale で引用。merge 操作の定量的正当化 |
| CLS (hippocampus/neocortex) | 生物学的・堅実 | 01-concepts layers section の non-normative sidebar として引用 |
| Active Inference | 哲学的・補助的 | 独立 retrieval-theory chapter の末尾に "further reading" として |

**個人的推し**: Hopfield と CLS は「AMKB の設計選択 (layer 分離 + merge) に後付けでない数学・生物学的根拠がある」ことを示す強い武器。**軽率に引かないが、真面目に引く価値がある**。Active Inference は引き過ぎると protocol を Friston 信者向けに見せてしまうので控えめに。

---

## 14. 残る open questions (再掲)

§7 の H1–H7 のうち解消・再分類後:

| ID | 状態 | 備考 |
|---|---|---|
| H1 (intent space 未規定) | partial (§11) | Δ(I) lift で理論的には decomposable。interchange format は将来課題 |
| H2 (set-level) | **resolved** (§10) | Lv-II/III 階層で吸収 |
| H3 (時間依存の明示) | open | spec に書くか決まっていない |
| H4 (learned ISF decomposability) | **resolved** (§10) | "ISFは black box、decomposable は追加性質" で解決 |
| H5 (relevance vs intent score 語彙) | **resolved** (R13) | spec で分離済み |
| H6 (retirement と D) | open | tombstone / lineage traversal の境界を詰める必要 |
| H7 (dynamical readout view) | deepened (§13) | Modern Hopfield で mathematized |

**新たな open**:

- **H8 (Lv-II/III の境界が behavioral なのか mechanistic なのか)**: §10.4 参照。benchmark は behavioral で比較すべき、という主張だが、実装者説明には mechanistic 階層も有用。どう書き分けるか。
- **H9 (action space 効率拡張)**: §12.3 の `count`, `sample`, `path`, `explain`。L4a 内のサブレベルにするか、別 optional level にするか。
- **H10 (Hopfield 根拠の扱い)**: §13.1 の設計予測 (merge は capacity 保持) を spec に書くと数学的コミットメントを伴う。強すぎる気もする。rationale 止まりが無難か。→ **§17 で decide**。

---

## 15. Hopfield による merge 操作の形式的正当化

### 15.1 Modern Hopfield の容量定理 (Ramsauer 2020 要約)

Ramsauer et al. (2020) Theorem 3 & 4 (arXiv:2008.02217) を AMKB 記法で書き直す。

記法:
- 埋め込み次元 `d`
- 格納パターン行列 `X = [ξ_1, ..., ξ_N] ∈ ℝ^{d×N}` (concept Node の embedding)
- 温度 `β > 0`
- Update rule: `ξ_new = X · softmax(β · X^T · ξ)`

**分離量**:
```
Δ(X) := min_i [ ||ξ_i||² − max_{j≠i} ⟨ξ_i, ξ_j⟩ ]
```
これは「各パターン `ξ_i` が、自分に最も近い他パターンからどれだけ離れているか」の最小値。

**定理 (Ramsauer Thm 3, 簡略形)**. Query `ξ` が `ξ_i* + ε` (真のパターン + noise) である時、1-step update `X softmax(βX^Tξ)` の retrieval 誤差は
```
||retrieval_error|| ≤ 2 exp(−β · (Δ(X) − 2·M·||ε||))
```
ここで `M = max_i ||ξ_i||`。

**定理 (Ramsauer Thm 4, 簡略形)**. 分離 `Δ > 0` を満たすランダムパターン群の期待容量は `O(d^{c/4})` で指数的。

要点:
1. **Δ(X) が退化 (→0) すると retrieval 誤差は exponential に 1 に近づく** — 任意の `β` でも救済不能。
2. **容量は Δ を保つ限り dimension `d` に対して指数的** — L_concept は理論上非常に多くの concept を保持可能。
3. **容量の急落は分離の退化で起こる** — 容量は「何個入るか」ではなく「どれだけ分離されているか」で決まる。

### 15.2 Near-duplicate 挿入と Δ 退化

AMKB の ingestion は agent が source から concept を抽出する。source が既存 concept に近い内容を含むと、新規 concept `ξ_new` が既存 `ξ_k` と小さな angle を持つ可能性がある:
```
⟨ξ_new, ξ_k⟩ ≈ ||ξ_new|| · ||ξ_k||    (cos ≈ 1)
```
この時
```
||ξ_new||² − ⟨ξ_new, ξ_k⟩ ≈ 0
```
となり `Δ(X) ≈ 0`。定理により、`ξ_new` と `ξ_k` 周辺を照会する intent は retrieval 不能 (どちらが返るか 50-50 に収束)。

**Quantitative 予測**: Ingestion が無制限に進むと、有限時間で `Δ(X) < ε` となる source 分布が存在する。具体的には:

**補題 (非形式)**. Source 群が埋め込み空間上で dense (任意の `ε` に対し `ε`-密) な領域を含むなら、ingestion が N → ∞ で `Δ(X) → 0`。

これは自明: dense 領域ではランダム2点が near-duplicate。merge なしなら Δ 退化は不可避。

### 15.3 Merge が果たす役割

Merge operation `merge(n_i, n_j) → n_merged`:
- `n_i, n_j` を tombstone
- `n_merged` を新規作成 (embedding は `ξ_merged`, 典型的に `(ξ_i + ξ_j)/2` の正規化 or agent による再合成)
- `derived_from` を union: `n_merged.derived_from = n_i.derived_from ∪ n_j.derived_from`
- lineage は保持

Merge 後の X は `{ξ_1, ..., ξ_N} \ {ξ_i, ξ_j} ∪ {ξ_merged}` となる。merge が near-duplicate pair `(i,j)` を解消する条件は:
```
⟨ξ_merged, ξ_k⟩ < ⟨ξ_i, ξ_k⟩ ∨ ⟨ξ_j, ξ_k⟩  for all k ∉ {i,j}
```
(merged が既存の他パターンに更に近づいていない)。典型的には成立する (合成は near-duplicate 2点の中間なので他との距離は増える)。

**命題 (Merge Necessity for C0 ISF)**:

> **Proposition 15.1**. Suppose an AMKB implementation uses a C0 (pointwise embedding) ISF. Let `Δ(X_t)` be the pattern separation after `t` ingestions without merges. Then:
> 1. **Degradation**: For source distributions supported on a dense region of embedding space, `Δ(X_t) → 0` as `t → ∞`.
> 2. **Retrieval collapse**: Once `Δ(X_t) < β^{-1}`, per Ramsauer Thm 3 the retrieval error bound is O(1) — concept retrieval becomes unreliable.
> 3. **Merge restores separation**: A greedy merge policy that merges any pair with similarity above threshold `1 − δ` maintains `Δ(X_t) ≥ δ·min_i ||ξ_i||²` indefinitely.
>
> Hence **merge is necessary** (not merely convenient) for embedding-based AMKB implementations that aim to retain unbounded ingestion.

証明スケッチ: (1) dense 領域に N 点を一様サンプリングすると最近接点間距離の期待値は `O(N^{-1/d})` → `Δ → 0`。(2) は Ramsauer Thm 3 直接適用。(3) は merge 後の Δ が閾値で下から抑えられることから。

### 15.4 非 C0 ISF への一般化

Hopfield 議論は C0 (pointwise embedding similarity) を前提にしている。C1–C3 ISF でも merge は意味を持つか?

| ISF Class | Near-duplicate の害 | Merge が解消する理由 |
|---|---|---|
| **C0 Pointwise** | Pattern interference (Ramsauer Thm 3) | 分離 Δ を回復 |
| **C1 Corpus-aware (BM25)** | IDF washout: 重複概念の token が頻出 = IDF 低下 = 識別性低下 | Token 分布の冗長性除去 |
| **C2 Graph-aware** | Graph centrality splitting: 同一意味概念の centrality が複数 Node に分散 | Centrality 集約 |
| **C3 LLM-judge** | Context length bloat: 重複候補が prompt を占有、注意希釈 | 候補集合の圧縮 |

いずれの ISF class でも "near-duplicate は retrieval quality を下げ、merge がそれを restore する" という構造は共通。Hopfield は C0 に対する **定量的 instance** であり、他 class は類似の定性的 argument を持つ。

**一般化命題 (非形式)**:

> **Thesis 15.2 (Near-Duplicate Interference)**. Any retrieval mechanism that treats concept Nodes as individual competing units — pointwise embeddings, lexical terms, graph nodes, or LLM-ranked candidates — degrades in quality as near-duplicate concepts accumulate. The merge operation, combined with `derived_from` lineage preservation, is the minimal protocol-level primitive that reverses this degradation without information loss.

これは **どの ISF class を選んでも merge は必要**、という protocol-level 主張。Hopfield はその最も厳密な instance。

### 15.5 設計への具体的含意

1. **Merge threshold の理論**: C0 実装は `1 − ⟨ξ_i,ξ_j⟩/(||ξ_i||·||ξ_j||) < δ` で merge を起動する threshold policy を採用できる。`δ` は capacity target と trade-off。
2. **Merge の agent policy**: curator agent は定期的に top-K 類似 pair を走査し、意味的に等価なら merge、等価でないが近いなら `contrasts` で接続、という分岐を持つと Hopfield 予測に沿う。
3. **Embedding 選択**: `d` が大きく、埋め込みが等方 (isotropic) であるほど容量が高い。AMKB は embedding を規定しないが、実装ガイドとして "anisotropic embedding は merge 頻度を上げる" と記載可能。
4. **L_category の二段 Hopfield**: category 層は coarse-grained Hopfield (少数パターン、高分離)。先に category を読み出し、選ばれた category 内で concept Hopfield を走らせる 2 段構成は、Ramsauer 的には **"hierarchical Hopfield with improved capacity per level"**。GraphRAG の hierarchical retrieval と一致。

---

## 16. Behavioral vs Mechanistic の形式化

### 16.1 問題

§10.4 で見たように、ある retrieval system は:
- **Mechanistically Lv-III** (実装は multi-step policy)
- **Behaviorally Lv-II** (入出力関数は静的 objective で書ける)

両方ありうる。どちらを protocol が観測すべきか。

### 16.2 観測フレームの概念

Retrieval system `R` を確率変数 `R: Input → Distribution(Output)` と見る。

**定義 (Observation Frame)**. 観測フレーム `O` は、観測者が `R` の評価時にアクセスできる変数の集合。

例:
- `O_protocol = { intent, k, filter, store_snapshot }`
- `O_agent = O_protocol ∪ { conversation_history, agent_scratchpad, LLM_internal_state }`
- `O_session = O_protocol ∪ { previously_returned_nodes }`

### 16.3 Observational Level

**定義 (Behavioral Level at Frame)**. Retrieval system `R` の frame `O` における behavioral level は、`O` 上の変数のみを用いて `R` の出力分布を特徴付けられる最小の level `ℓ ∈ {Lv-I, Lv-II, Lv-III}`。

形式化:
```
Lv-I(O):   ∃ f: O_input × D × S → ℝ. R(o, S) =_d top_k_by(f, o, S)
Lv-II(O):  ∃ U: O_input × 2^D × S → ℝ. R(o, S) =_d argmax_{|T|=k} U(o, T, S)
Lv-III(O): above two とも不成立
```
ここで `=_d` は分布一致、`o ∈ O_input` は観測可能入力。

**鍵となる observation**:

> 同じ system `R` でも、frame を広げると level が下がりうる。狭い frame で Lv-III に見えるものが、広い frame で Lv-II に見える。

### 16.4 Protocol Level

AMKB の protocol observation frame を固定する:

**定義 (AMKB Protocol Frame)**.
```
O_AMKB := { intent, k, filter, store_snapshot_at_call_time }
```
(つまり retrieve call argument と store の snapshot)

**定義 (Protocol Level)**. Retrieval system の AMKB protocol level は、`O_AMKB` における behavioral level。

### 16.5 分類結果

代表的システムを `O_AMKB` で分類:

| System | Mechanistic | Protocol Level | 説明 |
|---|---|---|---|
| BM25 RAG | Lv-I | Lv-I | 出力 = `top_k(BM25(i, n))` |
| Embedding RAG | Lv-I | Lv-I | 出力 = `top_k(cos)` |
| MMR post-ranker | Lv-II | Lv-II | 出力 = `argmax U_MMR` |
| Portfolio IR | Lv-II | Lv-II | 出力 = `argmax U_portfolio` |
| GraphRAG (fixed plan) | Lv-III | Lv-II | 固定 2-stage → 出力は `(i, S)` のみで決まる |
| GraphRAG (adaptive) | Lv-III | Lv-III | Threshold 通過に依存 |
| Self-RAG | Lv-III | Lv-III | LLM critic が intermediate observation 消費 |
| FLARE | Lv-III | Lv-III | 生成中の token entropy が retrieval を誘発 |
| ReAct-retrieval | Lv-III | Lv-III | tool-use loop |

**観察**: 多くの "agentic" システムは **mechanistic Lv-III, protocol Lv-III** だが、GraphRAG の固定プラン版のように **mechanism が multi-step でも protocol から見れば Lv-II** という ケース が実在する。

### 16.6 Protocol design への含意

1. **AMKB は protocol level で実装を比較する**。conformance test は `O_AMKB` で決まる入出力を見る。mechanism は implementation-private。
2. **Benchmark は frame を宣言すべき**。"この benchmark は `O_AMKB` frame で比較" と明記することで、"Self-RAG は Lv-III だから評価できない" のような混乱を避ける。同じ benchmark で Lv-II・Lv-III 両方評価可能 (出力を見るだけ)。
3. **Level は property of (system, frame) ペア**、system 単独の property ではない。これは圏論の "functor into the appropriate category" 的な関係。

### 16.7 注意: Non-determinism の扱い

Lv-II の `argmax` に tie-breaking randomness が入ると、分布的には Lv-II でも決定的関数ではなくなる。これは protocol level では:
- **Lv-II (stochastic)** を許容 (`R(o, S) =_d argmax ... with random tie-break`)
- Lv-III への昇格は不要 (観測できるのは分布、静的 `U` があれば十分)

つまり: **randomness は level を上げない**。level を上げるのは **feedback observation**。

### 16.8 R15 候補 (rationale 追加案)

> **R15 (candidate)**: The retrieval level hierarchy (Lv-I / Lv-II / Lv-III) is defined *at a specified observation frame*. AMKB adopts the protocol observation frame `O_AMKB = (intent, k, filter, store_snapshot)`. An implementation's protocol-level behavior is determined by what function this frame computes, independent of mechanistic implementation. Two implementations with identical `O_AMKB`-behavior are interchangeable under the protocol regardless of whether one is internally multi-step agentic and the other is single-pass.

---

## 17. H10 Decision — Hopfield をどこに置くか

### 17.1 トレードオフ

Hopfield 根拠を spec 本文 (normative) に置く案:
- **Pro**: AMKB に強い理論基盤を与え、"単なるグラフ DB spec" 以上の差別化になる。実装者に merge 操作の意義を明確に伝えられる
- **Con**: Normative text は testable でなければならない。"merge は capacity を保つ" は測定可能だが conformance test にしづらい。Ramsauer の定理に AMKB spec をバインドすると理論更新で spec が古びる。C0 以外の ISF 実装者は "これは私に関係ない" と感じる

Rationale のみに置く案:
- **Pro**: Rationale は元々 non-normative、理論議論の自然な居場所
- **Con**: Rationale は「なぜそう決めたか」の記録であって、積極的に読まれるとは限らない。長大な理論説明を突っ込むと rationale の性質が変わる

独立した理論文書に切り出す案:
- **Pro**: Spec の normative surface を痩せたまま保てる。理論は spec より速く更新できる (new citation, refinement)。仕様読者と理論読者を分けられる
- **Con**: 2 つの文書を管理する手間。cross-reference の保守

### 17.2 決定

**採用**: **独立した理論文書** + **rationale からの cross-reference**。

構成:

```
amkb-spec/
├── spec/               # Normative (変更慎重)
│   ├── 00-overview.md
│   ├── 01-concepts.md
│   ├── 02-types.md
│   ├── 03-operations.md
│   ├── 04-events.md  (未作成)
│   ├── 05-errors.md  (未作成)
│   └── 99-rationale.md  (R13 と R14/R15 候補から theory/ へ cross-ref)
├── theory/             # Non-normative (理論 companion、更新柔軟)  ← 新設
│   └── retrieval.md    # Retrieval Theory primer
└── notes/              # Scratch (git tracked, WIP)
    └── retrieval-theory.md   # 現在のこのメモ
```

**`theory/retrieval.md`** には以下を収録:

1. **Vocabulary**: intent, intent score, ISF, SUF, policy (§1–§2 相当)
2. **3-level hierarchy**: Lv-I/II/III (§10)
3. **Observation frames and protocol level** (§16)
4. **ISF classification** C0–C3 (§2 of notes)
5. **ISF algebra** (§4 of notes)
6. **Classical RAG as degenerate case** (§5 of notes)
7. **Near-duplicate interference and merge justification** (§15)
   - General thesis (Thesis 15.2)
   - Hopfield instance (Prop 15.1, Ramsauer formal)
8. **CLS layer analogy** (§13.2)
9. **Further reading**: Hopfield, CLS, active inference, MMR, xQuAD, Self-RAG, FLARE, GraphRAG
10. **Open questions**

`theory/` は `spec/` と同格のディレクトリで、README から明示的にリンク。ただし **normative ではない** ことを冒頭で宣言。

**Rationale 側**:
- R13 末尾: "See `theory/retrieval.md` §2 for the three-level hierarchy that formalizes the 'shape only' stance."
- R14 (新規): "The 4-op core is chosen to be a minimally complete action space for Lv-III policies (see `theory/retrieval.md` §3)."
- R15 (新規): "Protocol level is defined at the AMKB observation frame (see `theory/retrieval.md` §4)."
- **New R16 (Merge Necessity)**: "Merge is not merely a cleanup operation. It is necessary for retention of retrieval quality under continuous ingestion, formalized for C0 ISFs by the Hopfield capacity bound (see `theory/retrieval.md` §7)."

**README 側**:
- Reading order: 00 → 01 → 02 → 03 → ... → 99 (normative)
- "For theoretical background, see `theory/retrieval.md` (non-normative)."
- Positioning: "AMKB is a protocol; its theoretical companion lives in `theory/`."

### 17.3 この決定の利点

1. **Normative minimalism を保つ**: spec は誰でも実装できる薄い protocol 文書のまま
2. **理論の深さを可視化**: `theory/` の存在自体が "AMKB は serious に設計された" というシグナル
3. **更新速度の分離**: 理論は citation 追加・refinement を柔軟に、spec は breaking change に慎重に
4. **読者の分離**: 実装者は `spec/` を、研究者や theory 関心者は `theory/` を読む。両方読んでも整合

### 17.4 実施プラン

1. `theory/retrieval.md` を新規作成 (このメモの §1–§16 を清書して移行)
2. `99-rationale.md` に R14, R15, R16 追加 (theory への cross-ref 付き)
3. `README.md` 更新: theory ディレクトリへの言及
4. `notes/retrieval-theory.md` はそのまま作業メモとして残す (あるいは不要になれば削除)

合わせて **spec 本体の変更は最小** に留まる: 既に R13 で "relevance は impl 定義" は書いてあり、`theory/` はそれを補強するだけ。breaking change ではない。
