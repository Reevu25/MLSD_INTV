# G7 · Shopping Search Ranking — **INTERVIEW SHEET**

> **Tier 2 · #20** — multi-objective ranking with a genuine three-party conflict.
> Full study file: [`../G7_shopping_ranking.md`](../G7_shopping_ranking.md) · [diagrams](../G7_shopping_ranking_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the ranking stack · (d) datasets · (e) building the training set · (f) the models and their objectives · (g) serving the constrained ranker · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Sellers bidding? Returns data? Catalogue quality? First-party listings? (§1) |
| **Explainability** is easy points | "Why this result?" · **per-seller attribution by stage** · each floor's **shadow price** published (§8) |
| **Security** beyond the gateway | **Adversarial sellers**: attribute gaming, fake discounts, counterfeits that want to be merged (§8) |
| **Troubleshoot before being asked** | Raise **the catalogue**, **the return lag** and **price freshness** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Three parties, conflicting objectives** — and *constraints, not weights* | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · the floors as NFRs · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The results page · ⚠ **purchase *without return*** — a delayed label | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | ⚠ **The catalogue is the hard data problem** · the latency budget | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Structured intent → filter + retrieve → dedup → score → **rank under constraints** | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Catalogue normalisation · product matching · label lag · the scoring models | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING + THE CONFLICT** | **(g)** | Why not a blend · **λ is an output of the floor** · shadow prices · freshness | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Returns as the slow truth · seller-side panels · the proxy↔truth check | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Aggregated merchants · photo and conversational shopping · **relevance-only fallback** | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"What makes this different from web search is that there are three parties with genuinely conflicting objectives, and pretending otherwise is how these systems go wrong. The buyer wants the right product at a fair price, the seller wants their listing shown, the platform wants revenue. In web search, relevance and business value mostly point the same way. In shopping they demonstrably do not — the highest-margin product is frequently not the best product for this buyer, and any ranker with a revenue term will find that out and exploit it.**
>
> **So the decision I'd defend hardest is how that conflict is expressed. I would not blend relevance and revenue into a single weighted score. I'd formulate it as a constrained optimisation: maximise expected revenue subject to a relevance floor, a price-competitiveness floor, and a seller-quality floor.**
>
> **The reason is auditability. With a blended score, a high enough bid can buy any position, and when quality regresses six months later nobody can tell whether it came from the model or from auction pressure — the weight moved, quietly, in a config. A constraint is a thing you can point at in a design review; a weight is not."**

**Then two more commitments:**
1. **⚠ The label is purchase *without return*.** *"A system optimised for purchase will cheerfully sell people things they send back — negative margin, bad experience, and it looks like a success for about a quarter."* Returns arrive on a **30-day lag** → fast proxy for iteration, slow truth for gating.
2. **⚠ The catalogue is the hard data problem.** *"A query like 'red running shoes size 10 under $100' is about ninety percent **structured filtering**, not semantic matching. A team that starts with a better ranking model and leaves the catalogue messy will underperform a team that does the reverse."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the results page and the label, the catalogue and latency budgets, the ranking stack, the data and models, how the constrained ranker runs per query, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Are sellers bidding for placement?** *(If yes, the conflict is explicit and the constraint formulation is non-negotiable.)*
2. **Do we own fulfilment and returns data?** *(Determines whether "purchase without return" is observable at all.)*
3. **How clean is the catalogue?** *(This is usually the real answer to "why is quality bad".)*
4. **Does the platform also sell its own products?** *(Then self-preferencing is a regulatory question — the EU's Digital Markets Act — not just a ranking one.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | Shopping search on web and app: a query box plus filters |
| Catalogue | ~1B offers from millions of merchants → on the order of **100M distinct products** after matching |
| Traffic | Tens of thousands of QPS at peak |
| Placement | Sellers bid for **sponsored** slots, which share the page with organic results |
| Returns | We own fulfilment, so returns are observable on a **~30-day lag** |
| Freshness | Prices and stock change many times a day |

**FR:** structured + semantic retrieval · multi-objective ranking under constraints · **product matching/deduplication** across sellers · attribute normalisation · price and availability freshness.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Query latency p95 | ≤ 300 ms |
| **Relevance floor** | A hard constraint — **not tradeable against revenue**; violations = 0 |
| Price/availability freshness | **Minutes** — showing an out-of-stock or wrong-price item is a trust failure |
| **Return-rate ceiling** per slice | A monitored guardrail |
| Sponsored labelling | 100% — every paid placement is marked |

**Scope / NON-GOALS:**
- ❌ **A blended relevance+revenue score.** Position, with the auditability reason.
- ❌ **Optimising purchase alone.** Position: purchase *without return*.
- ❌ The auction mechanism (G21/G24) · logistics · pricing strategy · review-fraud detection (adjacent, real).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
[ red running shoes size 10 under $100 ]
  Filters applied: Running shoes · Red · Size 10 · Under $100    ← the parsed intent, editable

  Sponsored · Brand A Trail Runner — red · $89                   ← labelled, and above every floor
  Brand B Road Racer — red · from $95 at 6 sellers · 4.6★        ← matched: one product, a seller choice
  Brand C Cushion 9 — "crimson", shown as red · $99              ← a normalised attribute
  …
  "Why this result?" on every card                                ← relevance · price · seller rating
```

**The label:**

```
THE LABEL LADDER — each rung better and more delayed:

  CLICK        → WRONG. Optimises thumbnails and headline prices that
                 turn out to be misleading.
  PURCHASE     → better, but SPARSE and DELAYED
  ⭐ PURCHASE WITHOUT RETURN  → the objective that actually matters
                 · negative margin avoided
                 · and it's the buyer's real satisfaction

⚠ RETURNS ARRIVE ON A ~30-DAY LAG → a delayed-feedback problem (G22):
   · FAST PROXY for iteration (purchase, plus early return signals)
   · SLOW TRUTH for gating (purchase-without-return, matured cohort)
   · ⚠ AND A STANDING CHECK THAT THE PROXY STILL PREDICTS THE TRUTH
     — because when it stops, you'll optimise the proxy for a quarter
     before anyone notices.

GUARDRAILS (not tradeable): relevance floor · price competitiveness ·
  seller quality · return rate per category
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Purchase-without-return per search session**, on a matured cohort | The buyer got the right thing and kept it |
| Fast proxy | Purchase rate + early return signals (cancellations, "not as described" contacts) | Iteration speed, checked against the truth |
| Offline | NDCG on human relevance judgements, stratified by query type | The gate before any traffic |
| **Guardrails** | **Floor violations = 0** · return rate per category · price competitiveness · seller concentration · stale-price rate | Hard limits, one per party |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **the unglamorous truth** `A1b` `I1a` `I1d`

```
⚠ THE CATALOGUE IS WHERE THE QUALITY ACTUALLY LIVES.

  millions of merchants supply INCONSISTENT, INCOMPLETE and sometimes
  DELIBERATELY MISLEADING attributes:
    colour: "crimson" / "red" / "RED-2847" / ""
    size:   "10" / "US10" / "44 EU" / "M"
    brand:  misspelled, or claimed falsely

  TWO HARD PROBLEMS:
   1. ATTRIBUTE NORMALISATION — map free text to a controlled vocabulary
   2. ⭐ PRODUCT MATCHING — recognise that 15 listings from 15 sellers
      are THE SAME PHYSICAL PRODUCT (so you can compare price, merge
      reviews, and not show the same shoe 15 times)

⚠ AND THE QUERY ARITHMETIC:
  "red running shoes size 10 under $100"
   = colour filter + category + size filter + price filter
   ≈ 90% STRUCTURED FILTERING, ~10% semantic matching
   → if the attributes are wrong, NO ranking model can recover it.

LATENCY BUDGET (p95 ≤ 300 ms):
   query understanding ~20 ms · filter + retrieve ~50 ms · dedup ~10 ms
   · heavy model on the top ~500 ~80 ms · constrained re-rank ~10 ms
   · the rest: fan-out, network, rendering

INDEX: ~100M products × (structured fields + an embedding) ≈ hundreds of GB
   → SHARD by product, REPLICATE for QPS
   → prices and stock on a SEPARATE FAST PATH, updated in minutes
     without rebuilding embeddings
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Expressing the conflict | A blended score (a·relevance + b·revenue) vs **constrained optimisation** | A floor can be pointed at in review; a weight drifts |
| The label | Click vs purchase vs **purchase without return** | Optimising purchase sells returns |
| Where to invest first | A better ranker vs **the catalogue** | ~90% structured filtering; no ranker recovers wrong attributes |
| Retrieval | Pure embeddings vs **structured filter + semantic** | Size and price must be exact; semantics handle the rest |
| Duplicate offers | Show every listing vs **match and collapse with a seller choice** | Price comparison, merged reviews, no shoe shown 15 times |
| Sponsored placement | A bid buys any position vs **a bid competes only among results that clear the floors, and is labelled** | The floors hold for paid results too |

### The system — *draw this in 90 seconds* `A2a`

```
 QUERY: "red running shoes size 10 under $100"
       │
┌─ 1. QUERY UNDERSTANDING → STRUCTURED INTENT ───────────────────
│ TASK: parse the query into a category + attribute constraints
│ OUTPUT: {category: running_shoes, colour: red, size: 10,
│          price_max: 100} + the free-text remainder
│ ⚠ this is most of the work on most queries
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. RETRIEVE = STRUCTURED FILTER + SEMANTIC ───────────────────
│ TASK: exact filters over the NORMALISED catalogue (size, price,
│       in stock), then embedding similarity for the remainder
│ OUTPUT: a few thousand candidate offers
│ ⚠ filter FIRST — a wrong-size or out-of-stock result is worse
│   than a missing one
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. PRODUCT DEDUPLICATION ─────────────────────────────────────
│ TASK: collapse N sellers' offers of the SAME product, using the
│       offline product-matching clusters
│ OUTPUT: ~500 products, each with a seller choice
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. SCORE — the learned models (§5.3) ─────────────────────────
│ TASK: per product, predict relevance, P(purchase ∧ no return),
│       price competitiveness, seller quality, expected revenue
│ OUTPUT: a score VECTOR per product — not one blended number
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⭐ RANK AS CONSTRAINED OPTIMISATION ────────────────────────
│   maximise   E[revenue]
│   subject to RELEVANCE ≥ floor · PRICE-COMPETITIVENESS ≥ floor
│              · SELLER QUALITY ≥ floor
│ ⚠ NOT a blended score. A bid cannot buy a position that
│   violates a floor, and the floors are REVIEWABLE ARTIFACTS.
│ OUTPUT: the results page, sponsored slots labelled
└──────┬─────────────────────────────────────────────────────────
       ▼
 RESULTS → click · purchase · ⚠ RETURN (30-day lag) → back into training
```

**What "E[revenue]" means here:** it's built from the purchase-without-return model. A sale that comes back is negative margin, not revenue, so the objective and the label agree.

### ⚠ Raise these yourself, before the interviewer does `F2b` `F1b` `I1b`

- **The catalogue:** *"Before we talk about the ranker: this query is ~90% structured filtering. If the attributes are wrong, no model recovers it."*
- **The cascade:** *"The heavy model can't score a few thousand candidates inside 300 ms, so a cheap first pass keeps ~500 and the heavy model ranks those."*
- **The return lag:** *"The label I want arrives 30 days late, so: a fast proxy, a slow truth, and a standing check between them."*
- **Price freshness:** *"A stale price is a trust failure, so prices and stock update on their own fast path, and checkout re-validates them."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Constraints, not a blended weight** | "A bid shouldn't be able to buy any position. **A constraint is a thing you can point at in a design review; a weight is not**" | Leaves revenue on the table, visibly |
| **2** | **Purchase-without-return as the objective** | "Optimising purchase sells people things they send back — negative margin, and it looks successful for a quarter" | A 30-day label lag on the gating metric |
| **3** | **Invest in the catalogue before the model** | "'Red shoes size 10 under $100' is 90% structured filtering — wrong attributes can't be recovered by any ranker" | Unglamorous work that's hard to get funded |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Merchant feeds** | Titles, attributes, prices, stock | Inconsistent, incomplete, **adversarial** |
| **Product images** | Image-derived attributes; a matching signal | The same stock photo reused across products |
| **GTIN / UPC / MPN identifiers** | Exact product matching | Missing or wrong on many offers |
| **Search logs**: impressions → clicks → carts → purchases | Ranking labels | Position bias; a click isn't satisfaction |
| **Returns and return reasons** | The slow truth, and a catalogue-error signal ("not as described") | ~30-day lag; only where we own fulfilment |
| **Human relevance judgements** | The offline gate | Expensive; stratify by query type |
| **Seller history**: ratings, disputes, delivery | The seller-quality score | New sellers have none |
| **Reviews** | A quality signal | Fake reviews |

### 5.2 Building the training set (e) `F1c`

- **The label:** join impressions → purchases → returns, and label **purchase ∧ not returned within the window** only once the cohort has **matured**. The recent, unmatured tail gets G22's delayed-feedback treatment.
- **Position bias:** log the position and train with a shallow **position tower** that is dropped at serving; keep a small **exploration** slice for unbiased data.
- **⚠ Price-band confound:** cheap items are clicked and returned more, so normalise per price band, or the model learns "cheap = good".
- **Per-category windows:** return behaviour differs hugely between apparel and electronics, so label windows and calibration are set per category.
- **Clean out:** bot traffic, fraudulent orders, and **supplied attributes treated as claims, not facts** (FU 4).

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective |
|---|---|---|
| **Query understanding** | Query → category + attribute slots | Classification + sequence labelling, trained on query → purchased-product attributes |
| **Attribute normalisation** | Free text → a controlled vocabulary, per category | Classification + entity linking |
| **Product matching** | Are two offers the same product? | A pairwise model over title, image and attributes, thresholded for **precision** (FU 8) |
| **Relevance** | Graded relevance of a product to the query | Pairwise ranking loss (LambdaRank) on human judgements |
| **Purchase and return** | P(purchase) and P(return \| purchase) | Multi-task cross-entropy; their product gives **P(purchase ∧ no return)**, **calibrated**, because it feeds E[revenue] |
| **The ranker** | Order the page | Maximise E[revenue] subject to the floors (§6) |

### 5.4 Fine-tune an existing model (f) `F2c`

- **Attributes and matching:** LoRA-tune a pretrained image-text model on labelled offers, so attributes come from the **image and title**, not just the seller's claim. Distil it for the catalogue's volume.
- **Relevance labels:** an LLM judge, **calibrated against human judgements** on a stratified sample, makes the offline gate affordable at catalogue scale. Humans keep the final say on the gate set.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · INGEST      merchant feeds → validation → images
STAGE 2 · NORMALISE   attributes → controlled vocabulary per category
STAGE 3 · MATCH       offers → product clusters (precision-tuned), human review for the head
STAGE 4 · INDEX       products: structured fields + embeddings; prices and stock on a fast path
STAGE 5 · LABEL       impressions ⋈ purchases ⋈ returns, on matured cohorts
STAGE 6 · TRAIN       relevance + purchase/return models → calibrate → re-solve the floors' multipliers
STAGE 7 · GATE        human judgements + replay + zero floor violations → canary
```

---

## 6 · SERVING + THE CONFLICT *(min 32–38)* — probe **(g)** · **the argument to make well** `F3a` `F3b` `A3b` `I1b`

```
WHY NOT A BLENDED SCORE?  score = a·relevance + b·revenue

 1. A HIGH ENOUGH BID BUYS ANY POSITION. There is no floor; the
    trade is continuous and unbounded.
 2. THE WEIGHT IS A CONFIG VALUE. It moves quietly, incrementally,
    in the direction of the metric with a dollar sign — and each
    move is individually defensible.
 3. ⚠ SIX MONTHS LATER, WHEN QUALITY HAS REGRESSED, NOBODY CAN TELL
    WHETHER IT CAME FROM THE MODEL OR FROM AUCTION PRESSURE.

CONSTRAINED FORM:
   maximise E[revenue]  s.t.  relevance ≥ R, price-comp ≥ P, quality ≥ Q
 ✓ the floors are EXPLICIT, OWNED, VERSIONED, REVIEWABLE
 ✓ a violation is a BINARY EVENT you can alert on, not a drift
 ✓ and the shadow price of each constraint tells you EXACTLY what
   the relevance floor is costing in revenue — which makes the
   trade a CONVERSATION rather than a silent slide.

⭐ HOW IT RUNS PER QUERY — and the probe it pre-empts:
   PER-ITEM floors → ELIGIBILITY FILTERS (binary: in or out)
   PAGE-LEVEL floors → Lagrange multipliers: the ranker sorts by
     E[revenue] + λ·relevance, with λ SOLVED on recent traffic
     (dual ascent) so that the floor holds, and re-solved on a schedule
   "Isn't that a weighted sum again?" Its FORM is. But λ is an OUTPUT
     of the floor, not a dial anyone sets: when bids rise, λ rises with
     them and the floor holds. A hand-set weight would let relevance
     slide. And λ IS the shadow price, reported on every re-solve.
```

- **The objective** `A3b`: maximise E[revenue], built from P(purchase ∧ no return), subject to floors that are owned, versioned artifacts; λ is derived, never configured.
- **Efficiency** `F3a`: a **cascade**: a cheap scorer over a few thousand candidates, the heavy model on the top ~500, a distilled relevance model. The trade is some recall at the cascade boundary, which is measured, not assumed.
- **Scale** `F3b`: the index is sharded by product and replicated for QPS; head queries' candidate sets are cached, while **prices and stock are always read from the fast path**.
- **Reliability** `I1b`: if the heavy model times out, serve the cheap scorer's order **with the floors still enforced**; an item whose price or stock is unknown is **hidden or marked**, never shown as current, and checkout re-validates it; the last-good model and multipliers are kept for rollback.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE      human judgements on a stratified query sample (the gate) · replay of
             logged traffic for purchase-without-return on MATURED data ·
             floor violations (must be zero) · catalogue health
ONLINE       A/B on the fast proxy (purchase + early return signals)
             ⚠ two-sided: the experiment moves seller traffic too → seller panel
LONG-HORIZON a holdback until returns mature (30+ days) → the ship decision
STANDING     proxy ↔ truth correlation, re-measured per category (FU 9)
```

**What's on the dashboard:**

| | |
|---|---|
| **Headline** | **Purchase-without-return rate**, on a matured cohort |
| **Fast proxy** | Purchase rate + early return signals, with a standing **proxy↔truth correlation check** |
| **Hard guardrails** | Relevance floor violations (should be **zero**) · return rate per category · price competitiveness |
| **Seller-side** | Seller concentration · new-seller reach · **the three-party balance is a panel, not a footnote** |
| **Catalogue health** | Attribute coverage and normalisation accuracy · product-match precision · **stale price/availability rate** |
| **Human relevance judgements** | On a stratified query sample — the offline gate |

**Success** = purchase-without-return per session up on matured cohorts, with zero floor violations, return rates flat or falling, seller concentration stable and buyer retention not down. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | Top results get clicked regardless | Position tower; exploration slice |
| **Rich-get-richer** | Established sellers accumulate signal | **New-seller reach** guardrail; exploration |
| **Price band** | Cheap items are clicked and returned more | Normalise per price band |
| **Self-preferencing** | First-party listings favoured by the platform's own ranker | The same rules for first-party listings, audited (DMA) |
| **Willingness to pay** | Personalisation shows pricier items to those who'll pay | Personalise on stated preferences (size, brand), not inferred willingness to pay |
| **Return-label bias** | Return propensity differs by category and region | Per-category windows and calibration |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 300 ms through the cascade; the heavy model on ~500 products only; cost per 1,000 queries | p95 · $ per 1,000 queries |
| **Reliability** `I1b` | Ranker timeout → the cheap scorer, floors still enforced; unknown price or stock hidden and re-checked at checkout; last-good model and multipliers | Fallback rate · stale-price rate |
| **Usability** `I1c` | The parsed intent shown as **editable filters**; one result per product with a seller choice; prices and stock that are right | Reformulation rate · filter edits |
| **Scalability** `I1d` | ~100M products sharded and replicated; a feed-to-index pipeline that scales with merchants; a separate price fast path | Feed-to-index latency · QPS headroom |
| **Privacy** `I1e` | Purchase history drives personalisation only with controls; **sensitive purchases** (health, intimate) excluded from personalisation and "recently viewed"; **sellers see aggregates, never buyer identities** | Sensitive-category exclusions · access audits |
| **Safety** `I1f` | **Recalled, prohibited and age-restricted products** filtered at retrieval; counterfeits kept out of matched clusters | Recalled-item impressions = 0 |
| **Security** `I1h` | ⚠ **Adversarial sellers**: attribute gaming, fake "original price" discounts, counterfeits that want a merge, fake reviews, click fraud on sponsored slots → claims not facts, cross-seller consistency, image-derived attributes, the seller-quality floor | Gaming caught · bad merges found |
| **Ethics / bias** `I1i` | Three-party fairness: **no self-preferencing** (the EU's DMA), new-seller reach, no personalising on inferred willingness to pay | Seller concentration · new-seller reach |
| **Explainability** `I1j` | **"Why this result?"** and sponsored labels for buyers; **per-seller impression attribution by stage** (retrieval, dedup, ranking, policy) for sellers; **each floor's shadow price** for leadership | Seller disputes resolved · % results explained |

> ⭐ *"In a marketplace, production readiness means every party can see why: the buyer sees why this result, the seller sees which stage moved them, and leadership sees what each floor costs."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **A marketplace you don't own** (aggregated third-party merchants): you lose the returns signal and catalogue control (FU 10).
- **Photo queries** ("these shoes, but in red"): the structured intent comes from an image plus text, which connects to G1's visual search.
- **Conversational shopping**: an agent that asks "what's your budget?" turns the structured intent into dialogue state, and **the same floors** apply to whatever it recommends.

**When the revenue term can't be trusted — the relevance-only pipeline** `A5b` `F2a`:

```
the proxy stops predicting the truth, a floor-violation alert fires,
or the revenue model is suspected of gaming the floors
  → DROP the revenue term: rank by P(purchase ∧ no return), subject to
    the relevance floor
  → sponsored results move to FIXED, labelled slots
  → catalogue, matching and floors unchanged
  → gate the next model on MATURED purchase-without-return only
  "Revenue optimisation is something the floors earn, not a default."
```

**Top 3 failure modes:**
1. **Silent relevance erosion under revenue pressure** — exactly what the constraint formulation exists to prevent.
2. **Selling things people return** — looks like success for a quarter, then margin and CSAT both turn.
3. **Catalogue rot** — attributes drift, product matching degrades, and the ranker is blamed.

**Q1 ships:** **catalogue normalisation + product matching**, and the **constraint formulation with explicit floors**.
> *"Catalogue first, because it caps everything else and no ranking model recovers wrong attributes — and the constraints before any revenue term exists, because retrofitting a floor onto a system that already has a blended weight is a political fight rather than an engineering change."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why constraints instead of just tuning the weights?"** `A3b` `A5a`
> **Auditability**, and it's a structural argument rather than a stylistic preference. With a blended score, a high enough bid can buy any position — the trade is continuous and unbounded, and there's no floor anywhere. Worse, the weight is a config value: it moves quietly and incrementally in the direction of the metric with a dollar sign attached, and **each individual move is defensible**. Six months later quality has regressed and **nobody can tell whether it came from the model or from auction pressure**. With a constrained formulation — maximise expected revenue subject to relevance, price-competitiveness and seller-quality floors — the floors are explicit, owned, versioned artifacts, a violation is a **binary event you can alert on** rather than a drift, and the **shadow price** of each constraint tells you exactly what the relevance floor costs in revenue. That turns the trade into a conversation someone has to have, instead of a silent slide. *A constraint is a thing you can point at in a design review; a weight is not.*

**2. "What's your training label?"** `F1b` `A2c`
> **Purchase without return**, and the ladder to get there matters. Click is wrong — it optimises thumbnails and headline prices that turn out to be misleading, and the model gets very good at that. Purchase is better but sparse and delayed. But a system optimised for purchase will **cheerfully sell people things they send back**: negative margin, a bad experience, and it looks like a success for about a quarter until returns land. So the objective is purchase-without-return, which makes this a **delayed-feedback problem** with a ~30-day lag — a fast proxy for iteration, the slow truth for gating, and critically a **standing check that the proxy still predicts the truth**, because when that relationship breaks you'll optimise the proxy for a quarter before anyone notices. I'd also normalise by price band, since cheap items are both clicked and returned more, and without that the model learns "cheap equals good."

**3. "Where does most of the quality actually come from?"** `A1a` `A2d`
> The **catalogue**, and it's the least glamorous answer. Millions of merchants supply inconsistent, incomplete and sometimes deliberately misleading attributes — colour as "crimson" or "RED-2847" or blank, size as "10" or "US10" or "44 EU". Two hard problems sit there: **attribute normalisation** into a controlled vocabulary per category, and **product matching** — recognising that fifteen listings from fifteen sellers are the same physical product, so you can compare prices, merge reviews, and not show the same shoe fifteen times. And the arithmetic makes it decisive: a query like "red running shoes size 10 under $100" is roughly **ninety percent structured filtering**, not semantic matching — so if the attributes are wrong, **no ranking model can recover it**. A team that starts with a better ranker and leaves the catalogue messy will underperform a team that does the reverse, and I'd fund it in that order.

**4. "A seller is gaming their attributes. What do you do?"** `I1h` `F1c`
> Treat supplied attributes as **claims, not facts** — that's the design stance, because the party supplying them has an incentive to mis-state them. The common games: wrong category to dodge competition, inflated "original price" to manufacture a discount, keyword-stuffed titles, and mis-stated condition. Defences: **verify the head** of the catalogue with image-and-title-derived attributes and flag disagreement with the seller-supplied value; **cross-seller consistency** on matched products — if fourteen sellers say this is a size 10 and one says 11, that's evidence; **behavioural signals**, since mis-stated attributes produce elevated return rates and "item not as described" complaints, which is the strongest downstream signal; and **seller-quality as a ranking constraint** so systematic mis-statement costs visibility. I'd note this is the same adversarial structure as spam — it's an **economic contest**, so the goal is to make gaming unprofitable rather than to detect every instance.

**5. "Revenue is up 5% after a ranking change. Ship?"** `I2b` `I2a`
> Not until I've checked what it came out of, because in a three-party marketplace a revenue gain can be **borrowed rather than earned**. First: were any **relevance-floor violations** recorded? Under the constrained formulation that's a binary check and it's the whole point of having built it that way. Second: what happened to the **return rate**, because a 5% revenue gain accompanied by rising returns is negative margin arriving on a 30-day lag. Third: **buyer-side outcomes** — repeat purchase and session success — since a gain that comes from showing higher-margin worse matches shows up later as reduced retention, and the lag is long enough that nobody attributes it correctly. And fourth: **seller concentration**, because a change that routes traffic to fewer, larger sellers is an ecosystem risk that no relevance metric captures. If the floors held, returns are flat and buyer outcomes are stable, I'd ship — the constraints exist precisely so that revenue optimisation *within* them is safe.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you set the relevance floor, and who owns it?"** `A3b` `I2b`
> Empirically for the level, and **outside the revenue team** for the ownership — and the second half is the more important answer. For the level: measure buyer outcomes (purchase-without-return, repeat visit) across relevance bands on held-out traffic, find where they degrade materially, and set the floor there with margin — so it's grounded in buyer harm rather than being an arbitrary threshold. Then publish the **shadow price**: what this floor costs in revenue per quarter, which makes the trade explicit rather than hidden and, usefully, defends the floor better than an appeal to principle does. On ownership: the floor makes the revenue dashboard look worse and makes the buyer-experience dashboard look better, so if the team measured on revenue also owns the floor, it will erode one defensible increment at a time — same structural dynamic as an exploration budget or a fraud allow-through budget. So I'd want it owned by whoever is accountable for **buyer retention**, versioned, with changes requiring a review that shows the buyer-side impact.

**7. "Sellers say the algorithm is killing their business. Are they right?"** `I1j` `I1i`
> Sometimes, and the system should be able to answer rather than deflect. At this scale we're a **market maker** — a ranking change is an economic event for the people on the other side of it — so seller-side impact belongs in the launch review alongside buyer metrics. What I'd build to answer it: **per-seller impression attribution**, so I can say whether their impressions actually fell and which stage caused it — retrieval (an attribute change made them un-filterable), deduplication (they were merged into another seller's listing), or ranking; and a **distinction between a policy action and a ranking change**, because conflating those destroys trust and they're completely different conversations. What I'd resist is optimising for seller traffic directly, since that produces a marketplace serving sellers rather than buyers, and buyers are who decide whether it survives. What I *would* fund is **new-seller reach as a monitored guardrail**, because a purely engagement-optimal ranker is rich-get-richer and the catalogue stops renewing — the same argument as new-creator reach in recommendation.

**8. "Product matching says two listings are the same product. It's wrong. What breaks?"** `F1b` `I1h`
> Several things at once, and asymmetrically — which is why I'd tune matching for **precision over recall**. A **false merge** is the expensive error: two different products collapse into one result, so the price comparison is nonsense (you're comparing a genuine item to a counterfeit or a different model), reviews from one product attach to another, and a buyer can order expecting one thing and receive another — which lands as "item not as described," a return, and a trust failure. A **false split** is merely inefficient: the same shoe appears fifteen times, the results page is redundant, and price comparison doesn't work — annoying, not harmful. So the threshold sits well toward precision, with **human review for the head** of the catalogue where the volume justifies it, and **downstream detection** via return reasons and complaint clustering, since "not as described" concentrated on a matched cluster is strong evidence of a bad merge. I'd also treat **counterfeits** as a distinct problem that abuses matching deliberately — a counterfeit *wants* to be merged with the genuine product, which makes matching an adversarial surface rather than a neutral one.

**9. "Your fast proxy stops predicting the slow truth. How would you know, and what then?"** `F2a` `I2a`
> I'd know because I'd have built the **standing correlation check** — periodically re-measuring, on matured cohorts, whether the proxy (purchase plus early signals) still predicts purchase-without-return, and treating a decline in that correlation as an incident in its own right rather than as noise. That check is the thing that makes a fast-proxy-plus-slow-truth design safe; without it the proxy silently becomes decorative and you optimise it for a quarter. When it breaks, the diagnosis is usually one of three: the **model found a way to satisfy the proxy without the truth** — the classic Goodhart pattern, and the signature is proxy up, truth flat; the **category mix shifted**, so a proxy calibrated on electronics is now being applied to apparel, which has a completely different return profile; or a **product change** — free returns, a new policy — altered the relationship structurally. The responses differ: re-fit the proxy per category, tighten the gate back onto the slow truth temporarily, and in the Goodhart case treat it as a **reward-hacking finding** and add the exploited behaviour to the guardrails.

**10. "Would this design change if it were a marketplace you don't own — say, aggregating third-party merchants?"** `F3c` `A2c`
> Materially, and in a direction that makes most of it harder. Three changes. **You lose the returns signal**, which is the objective I built the whole label ladder toward — if fulfilment happens on the merchant's side you may never see returns, so the gating metric has to fall back to weaker proxies (post-click dwell, repeat visits to that merchant, complaint rate) and I'd be explicit that the system is now flying with a less reliable instrument. **Catalogue quality gets worse and less fixable** — you're consuming feeds rather than owning the data, so normalisation becomes an inference problem over adversarially-supplied text with no ability to enforce a schema, which raises the value of image-derived and cross-seller-consistency attributes. And **the three-party conflict sharpens**, because merchants are paying for placement and you have less visibility into whether the transaction went well — so the constraint formulation matters *more*, not less, since the relevance floor is now the main thing standing between the ranker and pure auction pressure. The piece that transfers unchanged is product matching, and it becomes even more central because price comparison across merchants is the core value proposition.

---

## ⭐ THE L6 LINE

> **"Three parties with conflicting objectives — so express the conflict as constraints with explicit floors, not as a blended weight, because a constraint is something you can point at in a design review and a weight quietly moves. And the label is purchase *without return*, on a 30-day lag."**

## 🧭 HOOKS — *steer by what you mention*

- *"A constraint is a thing you can point at in a design review"* → invites the org/governance branch (**good at L6**)
- *"Purchase without return"* → invites the delayed-labels branch (G22 crossover)
- *"90% structured filtering"* → invites the catalogue/data branch
- *"Attributes are claims, not facts"* → invites the adversarial branch
- *"We're a market maker"* → invites the marketplace-fairness branch (**good at L6**)
- *"The shadow price of the relevance floor"* → invites the optimisation branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three parties"* — buyer/seller/platform regenerates the whole design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the catalogue problem and the constraint formulation than finish the ranker. Sound right?"*

**Name and defer:**
- "**Review authenticity** is an adversarial problem layered on seller quality."
- "**Personalisation** interacts awkwardly with price sensitivity — showing expensive items to people who'll pay is a fairness question."
- "**Counterfeit detection** abuses product matching deliberately; it's a separate adversarial system."
- "**Inventory and price freshness** is its own pipeline with a minutes-level SLO."
- "**Sponsored vs organic blending** on the same page is where the constraint formulation gets tested hardest."
- "**Cross-border** — currency, duties, shipping time — changes both relevance and the price-competitiveness floor."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the catalogue and the filter before the ranker | §0, §3 | *"'Red running shoes size 10 under $100' is ~90% structured filtering."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"p95 ≤ 300 ms: filter, dedup, the heavy model on the top ~500."* |
| **A1c** | Degrade without lowering quality | §6 | *"Ranker times out → the cheap scorer, floors still enforced; an unknown price is hidden, never shown as current."* |
| **A2a** | The full stack | §4 diagram | Intent → filter + retrieve → dedup → score → constrained rank → results → returns |
| **A2b** | Model components | §5.3 | Query understanding, normalisation, matching, relevance, purchase and return |
| **A2c** | Data sources | §5.1 | Merchant feeds, images, identifiers, logs, returns, judgements, seller history |
| **A2d** | Staged pipeline | §5.5 | Ingest → normalise → match → index → label → train → gate |
| **A3a** | Top-down, unprompted | §0 | *"Three parties with conflicting objectives"* + the plan |
| **A3b** | A principled objective for the trade-off | §4, §6, FU 1 | *"Maximise E[revenue] subject to floors; λ is an output of the floor, not a dial."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Bidding? Returns data? Catalogue quality? First-party listings? |
| **A5a** | Alternatives compared | §4, FU 1 | Blend vs constraints; click vs purchase vs without-return; ranker vs catalogue first |
| **A5b** | A full alternative pipeline on demand | §9 | The relevance-only pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Multi-task P(purchase) × P(no return \| purchase), calibrated; a precision-tuned matcher."* |
| **F1b** | Metrics and their trade-offs | §2, FU 2 | The label ladder: click → purchase → purchase without return |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Position, rich-get-richer, price band, self-preferencing, return-label bias |
| **F2a** | Something fails → a fallback | §9, FU 9 | The relevance-only pipeline; gating back on the slow truth |
| **F2b** | Latency too tight → design for it | §4 | The cascade, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A LoRA image-text attribute model; an LLM judge calibrated to humans |
| **F3a** | Efficiency with a quality trade | §6 | Cheap first pass, heavy model on ~500, a distilled relevance model |
| **F3b** | Scale levers | §3, §6 | A sharded, replicated index; a price-and-stock fast path |
| **F3c** | Extension, with cost and maturity risk | §9, FU 10 | Aggregated merchants; photo queries; conversational shopping |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 ≤ 300 ms; $ per 1,000 queries |
| **I1b** | Reliability | §6, §8 | The fallback ranker; stale prices hidden and re-checked at checkout |
| **I1c** | Usability | §2, §8 | Editable parsed filters; one result per product |
| **I1d** | Scalability | §3, §8 | A sharded index; the feed-to-index pipeline |
| **I1e** | Privacy | §8 | Sensitive purchases out of personalisation; sellers see aggregates only |
| **I1f** | Safety | §8 | Recalled, prohibited and age-restricted products filtered at retrieval |
| **I1h** | Security | §8, FU 4 | **Adversarial sellers**: attributes are claims, not facts |
| **I1i** | Ethics / bias | §7, §8, FU 7 | No self-preferencing; new-seller reach; no pricing on willingness to pay |
| **I1j** | Explainability | §8, FU 7 | "Why this result?"; per-seller stage attribution; published shadow prices |
| **I2a** | Offline → online | §7, FU 5 | Judgements → replay → A/B on the proxy → matured holdback |
| **I2b** | The success definition | §7, FU 5 | *"Purchase-without-return up on matured cohorts, with zero floor violations."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training set → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
