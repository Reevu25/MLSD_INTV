# G1 · Visual Search (Lens) — **INTERVIEW SHEET**

> **Tier 2 · #14** — core Google product; ANN at scale.
> Full study file: [`../G1_visual_search.md`](../G1_visual_search.md) · [diagrams](../G1_visual_search_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the retrieval stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | **Which of the three products?** Camera or upload? Is Shopping the goal? (§1) |
| **Explainability** is easy points | "**This is**" vs "**Similar items**" labels, the matched region, source attribution (§8) |
| **Security** beyond the gateway | **Embedding inversion** of the shipped model; counterfeit listings gaming same-product matching (§8) |
| **Troubleshoot before being asked** | Raise the **verification budget** and the **wrong crop** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Three different products hide inside "visual search"** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals stated as positions | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | "This is" vs "Similar" · ⚠ **co-clicks grade the homework with your own answer key** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | 10⁹ images × 512 d → **quantize or it doesn't fit** · 300 ms budget | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · crop → route → embed → ANN → **geometric verification** | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Contrastive training · **hard negatives** · de-biasing · the domain gap | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | On-device vs cloud · hash sharding · index refresh | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Human-labelled set **not click-derived** · head vs tail · popularity concentration | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Image + text queries · live camera · the abstention path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Before I draw anything I need to resolve an ambiguity that I think is the whole question, because there are three different products hiding inside 'visual search' — and they need different labels, different losses, different indexes and different evaluation.**
>
> **Visually similar: same colours, texture, composition — a striped sofa returns other striped sofas. Semantically similar: same category — a golden retriever returns other dogs. Same product / same instance: this exact sofa, photographed in a different room under different lighting.**
>
> **A user pointing a camera at a sofa almost always wants the third, and the third is the one that monetises. But a user pointing at a plant wants the second. So I'd build same-product as the primary objective with semantic as the fallback, with an intent classifier in front — and I'd say out loud that a single embedding space cannot serve all three well, because the notion of distance is genuinely different."**

**Then the labels commitment:**
> **⚠ "Where do the labels come from? In practice, co-clicks — free and abundant, and **position-biased and popularity-biased**. A model trained naively on them learns to retrieve *popular* things rather than *matching* things, and the metric looks great because it's built from the same biased clicks. So: position de-biasing on the training data, and a small **human-labelled eval set that is not click-derived** — otherwise I'm grading the homework with the answer key I wrote."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and how I'd measure it honestly, capacity, the architecture, data and training, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Which of the three products?** *(Same-instance / semantic / visual-similarity need different everything — **the whole question**.)*
2. **Is this camera-first or image-first?** *(A live camera stream is a latency and on-device problem; an uploaded image isn't.)*
3. **Is Shopping the monetisation path?** *(If so, same-product recall is the business metric, not a modelling preference.)*
4. **What must it never do?** *(Face recognition is the obvious one: deliberately out of scope.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Product | Lens-style: point the camera at an object → same product (shoppable) or similar items |
| Corpus | ~10⁹ indexed images (catalogues + the web), catalogues changing hourly |
| Primary objective | **Same product**, with semantic as the fallback |
| Queries | Mostly live camera on phones; some uploads |
| Faces | **Excluded by design** (not embedded, not matched) |
| Freshness | New products searchable in hours |

**FR:** query by image (camera or upload) · same-product retrieval with **semantic fallback** · intent classification · crop/region selection · result attribution back to a source.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| End-to-end p95 | ≤ **300 ms** from capture |
| Corpus | ~10⁹ indexed images |
| **Same-product precision@1** | The headline — a wrong product is worse than no result |
| Index freshness | New products searchable in **hours** |
| Availability | 99.9%; if verification can't run, show "similar items", never assert "this is" |
| Cost | $ per 1,000 queries (verification is the heavy stage) |

**Scope / NON-GOALS:**
- ❌ **One embedding space serving all three products.** Position, with the reason.
- ❌ **Evaluating on click-derived labels alone.** Position, with the reason.
- ❌ OCR/text-in-image (G11) · face recognition (deliberately out) · video.

> **How to say these two — they are *positions*, not scope cuts, so deliver the reason in the same breath:**
>
> ❌ *"One embedding space for all three products is a non-goal."* — sounds like you're ducking work.
>
> ✅ **"I'm deliberately not building one embedding space for all three. The notion of distance is genuinely different — same-product must be invariant to lighting and angle but sensitive to small design differences; semantic similarity needs the exact opposite. One space is mediocre at all three, and that's not a capacity problem a bigger model fixes."**
>
> ✅ **"I won't evaluate on click-derived labels alone — co-clicks are position- and popularity-biased, so the metric is built from the same bias the model learned. That's grading the homework with your own answer key."**

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
CAMERA pointed at a sofa in a living room
  → a crop box snaps to the sofa (draggable — the user can fix it)
  CONFIDENT MATCH:   "This is the Harlow 3-seater (Oak & Linen)"
                     $1,299 at Store A · $1,249 at Store B      [source photo]
  NO GEOMETRIC MATCH: "Similar items"  → a grid of similar sofas
                     ↑ honest abstention: no "this is" without evidence
  pointed at a plant: "Monstera deliciosa" + care info  (semantic, not shopping)
```

**The metrics:**

```
⚠ THE LABEL PROBLEM IS THE MEASUREMENT PROBLEM.

CO-CLICKS are the free label: user searched an image, clicked a result.
  · POSITION-BIASED — top results get clicked regardless of match
  · POPULARITY-BIASED — popular products get clicked more
  → a model trained naively learns to retrieve POPULAR, not MATCHING
  → and the METRIC looks great, because the metric is built from the
    SAME biased clicks.  ⚠ GRADING THE HOMEWORK WITH YOUR OWN ANSWER KEY.

SO:
  TRAIN on de-biased clicks (position tower; inverse-propensity weighting)
  EVALUATE on a SMALL HUMAN-LABELLED SET that is NOT click-derived
    · same-product precision@1  ← the headline
    · recall@k for the retrieval stage
    · ⚠ SLICED: head vs TAIL products, and by category
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Same-product precision@1** on a human-labelled, non-click set | A wrong product is worse than none |
| Retrieval | Recall@100 before verification | Can verification even see the right item? |
| Honesty | **Abstention rate** ("similar items" instead of "this is") | Zero abstention means it's guessing |
| Online | Post-click outcome (bought, stayed) | CTR alone reproduces the bias |
| **Guardrails** | **Popularity concentration** · tail precision · p95 · $/1k queries | Marketplace health and cost |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
~10⁹ images × 512-d embeddings × 4 B (fp32) = 2 TB   ← doesn't fit
  → int8:  512 GB   → shard across nodes, resident
  → PQ:    ~65 GB   → fewer nodes, with a recall cost you buy back
                       by retrieving wider and re-ranking

⚠ WHY RESIDENT: HNSW is a dependent graph traversal (~500 hops, each
  hop determined by the last) — disk-resident is latency-infeasible
  at a 300 ms end-to-end budget.

LATENCY BUDGET (~300 ms):
  capture + on-device preprocess   ~50 ms
  embed (on-device or cloud)       ~60 ms
  ANN over 10⁹                     ~40 ms
  GEOMETRIC VERIFICATION on top-k  ~80 ms   ← the same-product step
  metadata fetch + assemble        ~40 ms

COST: verification (local-feature matching + RANSAC on ~100 candidates)
  is the heavy per-query compute → track $ per 1,000 queries; it's why
  verification runs on the top-k only
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Spaces | One embedding space vs **a space per product** | Contradictory invariances: one space is mediocre at all three |
| Same-product test | Embedding similarity alone vs **+ geometric verification** | Similarity says "looks like"; RANSAC says "is" |
| Routing | A trained intent classifier vs **rules over context** vs **run both, let verification decide** | Intent isn't in the pixels; verification gives evidence instead of a guess |
| Where it runs | All on-device vs all cloud vs **split** | Crop and a small embedding on-device; heavy verification in the cloud |

### The system — *draw this in 90 seconds* `A2a`

```
 CAMERA / UPLOAD
       │
┌─ 1. REGION PROPOSAL / CROP (on-device) ────────────────────────
│ what is the user actually pointing at? (often the whole difference
│ between a good and a useless result) · draggable by the user
│ ⚠ face regions detected and EXCLUDED here
│ OUTPUT: one cropped object
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. ROUTE (rules over context, §4 note) ───────────────────────
│ same-product? semantic? → which index / space to query (or both)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. EMBEDDING MODEL (small on-device, or cloud) ───────────────
│ contrastive-trained with HARD NEGATIVES
│ OUTPUT: a 512-d vector per space
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. ANN over 10⁹ (HNSW, int8 / PQ) ────────────────────────────
│ hash-sharded, scatter-gather, replicated
│ OUTPUT: top ~100 candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⭐ GEOMETRIC VERIFICATION (RANSAC on local features) ──────
│ the step that separates SAME-PRODUCT from merely similar;
│ expensive, so only on the top ~100
│ ⚠ ALSO THE ABSTENTION MECHANISM: no geometric consensus ⇒ "SIMILAR
│   ITEMS", never "THIS IS"
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. RE-RANK — availability, price, quality, merchant trust ────
└──────┬─────────────────────────────────────────────────────────
       ▼
 RESULTS (or an honest "no match")
```

> **⚠ Push back on your own box: the intent classifier may not need to exist.**
> **Intent is not a property of the image** — the same sofa photo is same-product if they're shopping and semantic if they're decorating — so it isn't cleanly labelable and two annotators will disagree, which becomes the model's noise floor. **Most of the signal is in the context, not the pixels:** entry surface (Shopping tab vs a design app), accompanying text ("where can I buy this"), coarse object category (sofa → shoppable · plant → semantic), session history. **A rules-first router over those needs no intent labels at all** and is debuggable.
>
> ⭐ **And the strongest version avoids the decision entirely: run both retrievals and let GEOMETRIC VERIFICATION arbitrate.** A confident homography means a same-product answer exists → *"This is the X."* No consensus → *"Similar items."* That replaces a **prediction about user intent** with **evidence about what the corpus can deliver** — and verification is already in the path. Cost: two retrievals, which is affordable because retrieval is cheap and verification is the expensive stage.
>
> *A trained classifier is the Q3 version, once behavioural labels (tapped a merchant link → same-product) have accumulated — and those are circular until a seed system exists.*

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The verification budget:** *"RANSAC on 100 candidates is ~80 ms. Under load I verify fewer candidates, and if verification can't run at all I show **'similar items'** rather than asserting a product. Never a confident guess."*
- **The wrong crop:** *"If region proposal picks the rug, everything downstream correctly answers the wrong question. So the crop is a first-class component with its own metric, and the user can drag it (FU 7)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Separate spaces + an intent router** | "The notion of *distance* is genuinely different across the three products — one space is mediocre at all of them" | Multiple indexes; a router that can be wrong |
| **2** | **Geometric verification on top-k** | "Embedding similarity says 'looks like'; RANSAC on local features says '**is**'. It's what makes same-product work — and it's the abstention mechanism" | ~80 ms and real compute on every query |
| **3** | **Abstain rather than assert** | "A confidently wrong product is worse than no result — it destroys trust and, in Shopping, sends money to the wrong merchant" | Lower coverage; a visible "no match" rate |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Catalogue photos** (many per SKU) | Positives for same-product; the cold-start representation | Studio lighting, white backgrounds |
| **User-photo ↔ catalogue pairs** | The real query domain | Scarce and expensive; the domain gap |
| **Co-clicks** | Abundant weak labels | **Position and popularity bias** |
| **Catalogue structure** | Free hard negatives (same line, different model) | Mislabelled listings |
| **Confirmed errors** (rejected results, returns) | The highest-signal negatives | Scarce |
| **Human-labelled eval set** | Ground truth that isn't click-derived | Expensive; must be sliced head vs tail |

### 5.2 Choosing the training set (e) `F1c`

```
1. POSITIVES      multiple photos of the same SKU · catalogue ↔ user-photo pairs ·
                  identity-preserving augmentations (lighting, angle, crop, background)
2. HARD NEGATIVES different SKUs of the SAME product line, same colour different
                  model · mined from the current index · confirmed production errors
                  ⚠ require catalogue-level evidence: a mined "negative" that is the
                  same product under another listing poisons training
3. DE-BIAS CLICKS position tower; inverse-propensity weights
4. BALANCE        upweight TAIL products and user-photo queries (the catalogue is
                  mostly tail; clicks are mostly head)
5. EXCLUDE        face regions; the human eval set is never trained on
```

### 5.3 The model and the objective (f) `A2b` `F1a`

**Training:** contrastive (InfoNCE-style) over (query-image, positive) pairs.
- **Positives** for same-product: multiple photos of the same SKU, catalogue↔user-photo pairs, augmentations that preserve identity (lighting, angle, crop, background).
- **⚠ Hard negatives are the whole game** — different SKUs of the *same* product line, same colour different model. Random negatives are trivially separable and teach nothing.
- **Augmentation defines the invariances**: what you augment over is what the model learns to ignore. Same-product must be invariant to lighting/background/angle but **sensitive to small design differences** — the opposite of a semantic-similarity model.

**Labels and their bias:** co-click position and popularity bias (§2) · **long-tail products have almost no clicks**, so a click-trained model is worst exactly where the catalogue is largest · user photos and catalogue photos are **different domains** (lighting, clutter, angle) — the domain gap is a real training problem.

### 5.4 Fine-tune an existing model (f) `F2c`

Start from a pretrained vision backbone and fine-tune it (fully, or with **LoRA** adapters per product space) with the contrastive objective above. One shared backbone with a small head or adapter per space (same-product, semantic) is cheaper than separate models; the **on-device** version is distilled from the cloud model.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · INGEST     catalogue feeds (hourly) → crop + face exclusion → embed
STAGE 2 · INDEX      add new vectors to the HNSW shards; a full rebuild on a cadence
STAGE 3 · TRAIN      contrastive with mined hard negatives (refreshed as confusions move)
STAGE 4 · GATE       human-labelled set: same-product P@1, head vs tail, abstention
STAGE 5 · ROLL OUT   ⚠ a new embedding model = a NEW INDEX: dual-index, then cut over
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
ON-DEVICE vs CLOUD EMBEDDING — a real trade to name:
  ON-DEVICE  · privacy: the raw image never leaves
             · latency: no upload of a large image
             · works offline / poor connectivity
             ⚠ model must be SMALL, and updating it means an app release
             ⚠ ⚠ AND: the embedding you ship is now a PUBLIC artifact —
               embedding-inversion attacks can partially reconstruct
               images, so the shipped model is an attack surface
  CLOUD      · bigger model, updated continuously, cheap to iterate
             ⚠ uploads the user's photo — a privacy and cost question

→ REALISTIC ANSWER: on-device for the CROP/INTENT and a small embedding
  for the common path; cloud for the heavy same-product verification.

SHARDING: hash by image id, scatter-gather, replicate 2–3×.
  ⚠ not semantic sharding — recall cliffs at category boundaries.
```

- **Quantization** `F3a`: int8 or PQ vectors (retrieve wider to buy back recall), and a quantized, distilled on-device model, with same-product P@1 re-checked after each step.
- **Scale** `F3b`: the cheap-wide → expensive-narrow cascade; verification only on the top ~100.
- **The result objective** `A3b`: show "**This is**" only when geometric consensus clears a threshold tuned on the human set for precision; otherwise "**Similar items**". The threshold encodes *"a wrong product is worse than none"*.
- **Reliability** `I1b`: replicas; verification timeouts degrade to "similar items"; embedding changes roll out as a dual index.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **The gate** | **Human-labelled, non-click-derived** same-product precision@1 |
| **Sliced** | **Head vs tail products** · category · user-photo vs catalogue-photo query |
| **Abstention rate** | Tracked as a feature — a system that never says "no match" is lying |
| **Popularity concentration** | What share of results go to the top 1% of products? Rising = the bias is winning |
| **Online** | Click-through **and post-click** outcome (did they buy / stay?) — CTR alone reproduces the bias |

**Rollout** `I2a`: offline gate on the human set → **dual-index shadow** (the new embedding serves no one yet) → A/B on post-click outcomes, never CTR alone → cut over. Offline comparisons against logs use IPS, because clicks only exist for what was shown.

**Success** = same-product P@1 up on the human set **including the tail**, abstention honest, post-click outcomes up, popularity concentration not rising. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | Top results get clicked regardless of match | Position tower; IPS |
| **Popularity** | Popular products get clicked and retrieved more | Non-click eval; popularity-concentration panel; tail upweighting |
| **Domain gap** | Catalogue ≠ user photos | User-photo pairs; slice the eval by query type |
| **Tail** | No clicks → never trained → never retrieved | Catalogue-based training; content-based retrieval |
| **Annotator disagreement on intent** | Intent isn't in the pixels | Rules over context; let verification arbitrate |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 300 ms from capture; the per-stage budget; cost per 1,000 queries | p95 per stage · $/1k queries |
| **Reliability** `I1b` | Verification timeouts degrade to "similar items"; replicated shards; dual-index migrations | Degraded-answer rate · shard availability |
| **Usability** `I1c` | **Draggable crop**; honest "this is" vs "similar items"; source attribution | Crop-adjust rate · abstention rate |
| **Scalability** `I1d` | int8 / PQ ANN over 10⁹, hash-sharded; verification only on top-k; hourly catalogue ingestion | Index size · QPS headroom |
| **Privacy** `I1e` | Crop and a small embedding **on-device**; photos not retained by default; **faces excluded architecturally** (never embedded) | Retention audits · face-exclusion tests |
| **Safety** `I1f` | **No face recognition / person identification**; SafeSearch on results; sensitive images (e.g. medical) handled carefully | Face-match attempts blocked |
| **Security** `I1h` | ⚠ **Embedding inversion** of the shipped on-device model; ⚠ **counterfeit listings** reusing real catalogue photos to win same-product matches → merchant trust signals | Counterfeit reports · inversion red-team |
| **Ethics / bias** `I1i` | We're a **market maker**: popularity concentration across merchants; tail products and small merchants must stay findable | Popularity concentration · tail P@1 |
| **Explainability** `I1j` | "**This is**" vs "**Similar**" makes confidence explicit; the matched region shown; the source photo and merchant attributed | % results with attribution |

> ⭐ *"Visual search points a camera at the world, so production-readiness is mostly about what it must refuse to do: identify people, or assert a match it can't verify."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Image + text queries** ("this, but in blue"): combine the image embedding with a text modifier (composed retrieval), or filter by catalogue attributes. Cost: a joint image-text model. Risk: composed retrieval is far less mature than either alone.
- **Live camera / AR:** the budget becomes **per frame**, not per query, so most of the stack has to move on-device and run continuously.

**When verification finds no match, the fallback pipeline** `A5b` `F2a`:

```
no geometric consensus for any same-product candidate
  → don't assert: label the answer "Similar items"
  → query the SEMANTIC space (category-level similarity) instead
  → offer to ADJUST THE CROP (the most common real cause)
  → log the miss: repeated misses on a category = a catalogue or crop gap
```

**Top 3 failure modes:**
1. **Popularity collapse** — the model retrieves popular products regardless of match, and the click metric confirms it's working.
2. **Confident wrong product** — visually near-identical SKU; in Shopping this sends money to the wrong merchant.
3. **Tail invisibility** — long-tail products have no clicks, so they're never trained on and never retrieved; the catalogue is mostly tail.

**Q1 ships:** the **intent classifier + same-product index + geometric verification**, and the **human-labelled eval set**.
> *"The eval set first, honestly — without a non-click-derived ground truth I can't tell whether any of the rest is working, and every improvement will be measured against the bias I'm trying to remove."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why can't one embedding space serve all of this?"** `A5a` `F1a`
> Because the notion of **distance** is genuinely different across the three products. Same-product retrieval must be **invariant** to lighting, background, angle and crop, while being **highly sensitive** to small design differences — a different stitching pattern means a different SKU. Semantic similarity is the opposite: it must be invariant to exactly those small design differences and sensitive to category. Visual similarity cares about colour and composition, which the other two deliberately discard. Those are contradictory invariances, so a single space trained on all three is mediocre at each — it's not a capacity problem you can solve with a bigger model. So: **separate spaces, separate indexes, and an intent classifier in front**, which introduces its own failure mode (a misrouted query) that I'd measure directly.

**2. "Your labels are co-clicks. What's wrong with that?"** `F1c` `I2a`
> They're free and abundant, and they're **position-biased and popularity-biased** — so a model trained naively on them learns to retrieve *popular* things rather than *matching* things. The trap is what happens next: the offline metric is built from the same biased clicks, so it **confirms that the biased model is working**. You're grading the homework with the answer key you wrote. Two fixes, and both are needed. On training: **position de-biasing** — a shallow position tower dropped at serving, and inverse-propensity weighting where I can estimate examination probability. On evaluation: a small **human-labelled set that is not click-derived**, which is the only thing that can tell me whether the de-biasing worked. And I'd slice that set **head versus tail**, because the popularity bias is invisible in an aggregate dominated by head queries.

**3. "How do you actually tell 'the same sofa' from 'a similar sofa'?"** `A2b` `A1c`
> Embedding similarity gets you to "looks like"; **geometric verification** gets you to "is." After ANN returns the top ~100 candidates, I run local-feature matching with **RANSAC** to find a consistent geometric transform between the query image and each candidate — if the keypoints agree under a single homography, it's the same object photographed differently; if they don't, it's a different object that happens to embed nearby. It's expensive, which is exactly why it runs on the top 100 rather than the corpus — the classic cheap-wide-then-expensive-narrow cascade. And it does a second job that I'd highlight: it's the **abstention mechanism**. If nothing achieves geometric consensus, the honest output is **no match**, and I'd rather return nothing than a confidently wrong product — especially in Shopping, where a wrong product sends money to the wrong merchant.

**4. "On-device or cloud embedding?"** `A5a` `I1e`
> Both, split by role, and the trade is more than latency. **On-device** keeps the raw image on the phone, which is a genuine privacy win for a camera product, avoids uploading a large image, and works with poor connectivity — but the model must be small, and updating it means an app release, so iteration is slow. There's also a security consequence people miss: **a shipped embedding model is a public artifact**, and embedding-inversion attacks can partially reconstruct images from embeddings, so the model itself becomes an attack surface. **Cloud** allows a much bigger model, continuous updates, and cheap iteration, at the cost of uploading the user's photo. The realistic split: on-device for **crop/region proposal and intent**, plus a small embedding for the common path; cloud for the heavy **same-product verification** where the accuracy actually matters.

**5. "A brand-new product is added to the catalogue. Is it findable?"** `F2a` `I1i`
> Yes, and this is where visual search is genuinely better off than a click-trained recommender — because **retrieval is content-based**. A new SKU has catalogue photos, so it gets an embedding and enters the index immediately; it doesn't need engagement history to be retrievable. The cold-start problem shows up in the **re-ranking** stage instead, where availability, merchant trust and popularity features are missing, so a new product can be retrieved but ranked below established ones. Mitigations: **shrinkage toward category-level priors** for missing merchant features, and an **exploration allocation** so new products get impressions. The bigger structural risk is the tail: long-tail products have almost no clicks *ever*, so if the embedding model is trained primarily on click data it's weakest exactly where the catalogue is largest — which is an argument for training on **catalogue-photo pairs and augmentations** rather than relying on engagement.

---

## 🔥🔥 SIX HARDER ONES — *the L6 separators*

**6. "How do you build hard negatives at this scale?"** `F1a` `A2c`
> This is most of the modelling work, because random negatives are trivially separable and teach the model nothing — it learns "sofa ≠ bicycle," which it already knew. The negatives that matter are **different SKUs from the same product line**: same colour, same silhouette, one design detail apart. Three sources, in increasing order of value. **Catalogue structure** is free and surprisingly strong — same brand, same category, different model number gives you near-misses by construction. **Mining from the current index**: retrieve top-k for a training query and take the high-scoring non-matches, which are by definition the model's current confusions; this needs periodic refreshing because the confusions move as the model improves. And **confirmed errors from production** — cases where a user rejected a result or returned a purchased item — which are the highest-signal negatives and the scarcest. The practical caution is **false negatives in mining**: a mined "negative" that's actually the same product under a different listing poisons training, so I'd require catalogue-level evidence rather than trusting the retrieval rank alone.

**7. "The user photographs a sofa in a cluttered living room. What breaks first?"** `F2a` `I1c`
> The **crop**, before any of the ML I've described. If region proposal picks the rug or the coffee table, everything downstream is correctly answering the wrong question, and the user experiences it as "visual search is bad" rather than "it looked at the wrong object." So I'd treat region selection as a first-class component with its own metric, not a preprocessing detail — and I'd make it **interactive**, letting the user adjust the crop, because a two-second correction beats a perfect model applied to the wrong region. Second failure is the **domain gap**: catalogue photos are shot on white backgrounds under studio lighting, user photos are cluttered and badly lit, so a model trained mainly on catalogue imagery degrades on exactly the query distribution it will see. That's a training-data problem — you need user-photo↔catalogue pairs, which are expensive to obtain and which I'd prioritise over more catalogue data. Third is **occlusion and viewpoint**: geometric verification is robust to a lot, but a sofa photographed from behind has few matchable keypoints and the right answer there is abstention.

**8. "Precision@1 is 92% but merchants are complaining. What's happening?"** `I1i` `F1b`
> Almost certainly an **aggregate hiding a slice**, and merchants are a good detector because they see their own slice. The likeliest story: 92% is dominated by head products, and precision on **tail products** — which is most of the catalogue and most of the merchants — is far lower, so the average looks healthy while the long tail of merchants gets systematically wrong or absent results. Second possibility: **popularity concentration**, where results disproportionately route to a handful of large merchants regardless of match quality, which is a marketplace-fairness problem rather than an accuracy one and won't show up in precision at all — I'd measure the **share of results going to the top 1% of products** and track it over time. Third: the metric is measured on a **click-derived set**, which reproduces the bias and can't see this by construction. The organisational framing I'd give: at this scale we're a **market maker**, so merchant-side impact belongs in the launch review alongside precision — and a merchant complaint is often the earliest available signal of a slice failure we aren't instrumented for.

**9. "Would you ship this if it can't recognise faces?"** `I1e` `I1f`
> Yes, and I'd treat excluding faces as a **deliberate scope decision rather than a capability gap** — which is the position I'd defend rather than apologise for. Face recognition on a general visual-search product is a categorically different risk: it enables identifying strangers from a photo, it's regulated very differently across jurisdictions, and the failure mode is a harm to a person rather than a bad shopping result. So faces are out of scope by design, and I'd want that enforced **architecturally** — face regions detected and excluded from embedding rather than merely filtered from results, because a filter at the end still means the index contains face embeddings and the capability exists. The analogous decision elsewhere in this pack is Street View blurring, where the asymmetry runs ~10,000:1 toward over-blurring. What I'd add for completeness: the product still needs **person detection** to *avoid* matching on people, so there's real engineering in not doing the thing, and that work is worth naming rather than assuming absence is free.

**10. "Your intent classifier needs labelled data. Where does it come from?"** `A5a` `A3a`
> This is the right challenge, and my honest answer is that I'd try not to need it. **Intent isn't a property of the image** — the same sofa photo is same-product if the user is shopping and semantic if they're decorating — so it can't be labelled from the pixels, two annotators will disagree, and that disagreement becomes the model's noise floor. The labels that *do* exist are **behavioural** — tapped through to a merchant means same-product, browsed a similar-category grid means semantic — which are free and from the real distribution, but **circular**: you only observe behaviour on results you already showed, so you need a seed system first. So the sequencing is: **rules first** over context that requires no labels (entry surface, accompanying text, coarse object category, session history), which carries most of the signal because intent correlates far more with *where the user is* than with what they photographed. Then, where it's genuinely ambiguous, **run both retrievals and let geometric verification arbitrate** — a confident homography means a same-product answer exists, which is **evidence about what the corpus can deliver rather than a prediction about intent**. A trained classifier is the Q3 version, once behavioural labels have accumulated and I can measure misrouting directly.

**11. "One quarter, small team. What do you cut?"** `A1c` `A3a`
> I'd cut the number of **products** rather than the depth of any one — pick **same-product only**, drop semantic and visual-similarity entirely, and drop the intent classifier along with them, because a router with only one destination is pure cost. That gives a coherent, evaluable product: camera → crop → embedding → ANN → geometric verification → result or honest abstention. What I would **not** cut is the **human-labelled evaluation set**, even though it produces no user-facing capability, because without a non-click-derived ground truth every subsequent improvement is measured against the bias I'm trying to remove, and the team will spend the quarter optimising popularity while the metric applauds. I'd also keep **geometric verification**, since it's what makes same-product actually work and it doubles as the abstention mechanism — shipping embedding-similarity alone would produce confident wrong products, which is the failure that destroys trust fastest and is hardest to recover from. Semantic fallback comes in Q2, once there's a trustworthy measurement to justify it.

---

## ⭐ THE L6 LINE

> **"Three different products hide inside 'visual search' and one embedding space cannot serve them — the notion of distance is genuinely different. And co-clicks are position- and popularity-biased, so evaluating on them is grading the homework with your own answer key."**

## 🧭 HOOKS — *steer by what you mention*

- *"Three products, three notions of distance"* → invites the modelling/representation branch
- *"Grading the homework with your own answer key"* → invites the labels/evaluation branch (**good at L6**)
- *"Geometric verification is also the abstention mechanism"* → invites the classical-CV branch
- *"A shipped embedding model is a public artifact"* → invites the on-device/security branch
- *"We're a market maker"* → invites the marketplace-fairness branch (**good at L6**)
- *"Hard negatives are the whole game"* → invites the contrastive-training branch
- *"The intent classifier may not need to exist"* → invites the do-we-need-ML-here branch (**good at L6**)

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three products"* — the intent split regenerates the architecture.
- **Running out of time?** → *"I have five minutes — I'd rather cover the label bias and abstention than finish the sharding. Sound right?"*

**Name and defer:**
- "**Text-in-image / OCR** is a separate pipeline that often runs in parallel on the same capture."
- "**Multimodal query** — image plus text refinement ('this, but in blue') — is a real and growing product."
- "**Index refresh** for a catalogue that changes hourly is its own freshness pipeline."
- "**Embedding-model migration** requires a dual-index rollout; you can't mix versions."
- "**Counterfeit detection** is an adversarial problem layered on top of same-product matching."
- "**AR / live camera** changes the latency budget completely — it's per-frame, not per-query."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: cheap ANN, expensive verification on the top-k | §0, §4 | *"Embedding similarity says 'looks like'; RANSAC says 'is'."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"2 TB in fp32 doesn't fit; int8 512 GB, PQ ~65 GB; 300 ms end to end."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §6 | *"If verification can't run, 'similar items' — never a confident guess."* |
| **A2a** | The full stack | §4 diagram | Crop → route → embed → ANN → verify → re-rank |
| **A2b** | Model components | §5.3 | Contrastive embeddings per space; geometric verification |
| **A2c** | Data sources | §5.1 | Catalogue photos, user-photo pairs, co-clicks, confirmed errors, human set |
| **A2d** | Staged pipeline | §5.5 | Ingest → index → train → gate → dual-index rollout |
| **A3a** | Top-down, unprompted | §0 | *"Three different products hide inside 'visual search'"* + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"'This is' only above a consensus threshold tuned for precision."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Which product · camera or upload · Shopping · what it must never do |
| **A5a** | Alternatives compared | §4, FU 1, 4, 10 | One space vs several; similarity vs verification; classifier vs rules vs both |
| **A5b** | A full alternative pipeline on demand | §9 | The no-match path: semantic space + crop adjustment |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 6 | *"Hard negatives are the whole game; augmentation defines the invariances."* |
| **F1b** | Metrics and their trade-offs | §2 | *"Co-clicks grade the homework with your own answer key."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Position, popularity, domain gap, tail |
| **F2a** | Retrieval underperforms → a fallback | §4, §9, FU 7 | The wrong crop; the no-match path |
| **F2b** | Latency too tight → tiering | §4 | Fewer verified candidates, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A pretrained backbone with per-space adapters; distilled on-device |
| **F3a** | Quantization with a quality trade | §3, §6 | int8 / PQ, retrieve wider to buy back recall |
| **F3b** | Scale via the cascade | §3, §6 | Verification only on the top ~100 |
| **F3c** | Extension, with cost and maturity risk | §9 | Image + text queries; live camera |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 per stage; $ per 1,000 queries |
| **I1b** | Reliability | §6, §8 | Degrade to "similar items"; dual-index migration |
| **I1c** | Usability | §2, §8 | Draggable crop; honest abstention |
| **I1d** | Scalability | §3, §8 | Sharded int8 / PQ ANN |
| **I1e** | Privacy | §6, §8, FU 9 | On-device crop; **faces never embedded** |
| **I1f** | Safety | §8 | No person identification |
| **I1h** | Security | §6, §8 | **Embedding inversion**; counterfeit listings |
| **I1i** | Ethics / bias | §7, §8, FU 8 | Market maker: merchant concentration, tail |
| **I1j** | Explainability | §2, §8 | "This is" vs "Similar"; attribution |
| **I2a** | Offline → online, correcting for shown-only labels | §7 | Human set → dual-index shadow → A/B on post-click outcomes; IPS |
| **I2b** | The success definition | §7 | *"P@1 up including the tail, honest abstention, concentration not rising."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
