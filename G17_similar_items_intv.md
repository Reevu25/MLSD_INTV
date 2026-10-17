# G17 · Similar Items / "You May Also Like" — **INTERVIEW SHEET**

> **Tier 3 · #22** — common, and cheap to learn. Two definitional questions carry the whole answer.
> Full study file: [`../G17_similar_items.md`](../G17_similar_items.md) · [diagrams](../G17_similar_items_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the module looks like · (c) the pipeline · (d) datasets · (e) building the pairs · (f) the models and their objectives · (g) serving and cold start · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Where does the module sit? How big is the tail? Personalisation? How fast does the catalogue churn? (§1) |
| **Explainability** is easy points | A relation label on the module; PMI is **explainable to a merchandiser** ("co-bought 4× more than chance"); overrides logged (§8) |
| **Security** beyond the gateway | **Shilling**: fake co-views and co-purchases to attach a product to a best-seller's page (§8) |
| **Troubleshoot before being asked** | Raise **the CTR trap** and **hot-key skew** in the offline job yourself (§3, §4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Similar in what sense?** · **popularity dominates co-occurrence** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · concentration as an SLO · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Two placements, two relations · ⚠ **evaluate on the TAIL** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | ⭐ **Precomputable** — 8 GB of KV, no model in the hot path · hot keys | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Logs → normalised association + content → top-K per relation → KV → re-rank | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | **PMI / lift, not raw counts** · support floors · content embeddings for the tail | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING + COLD START** | **(g)** | Placement picks the relation · exploration for new items · availability at serve time | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Sliced by popularity decile · concentration guardrail · placement outcomes | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Sessions, bundles, seasonality · **content-only fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Two questions decide this design, and both are about definitions rather than models — which is why it's a better question than it first looks.**
>
> **First: similar in what sense? There are at least four and they're different products — substitutes, complements, stylistic neighbours, same-category. And the right answer depends on where the module sits: on a product page before purchase the user is comparing, so they want substitutes; in a post-purchase or cart context substitutes are useless — 'here's another sofa you could have bought instead' — and complements are what convert. Placement determines the semantics, and a module that gets this wrong looks perfectly reasonable and converts terribly.**
>
> **Second, the central technical problem: co-occurrence data is dominated by popularity. 'Users who viewed X also viewed Y' from raw counts returns the current best-seller for almost every X, because the best-seller co-occurs with everything."**

**Then the two consequences:**
1. **Normalise by popularity — PMI or lift, not raw co-occurrence** — *"and then evaluate specifically on **tail items**, because that's where the failure lives and an aggregate CTR metric will hide it."*
2. **⭐ This system is conditioned on an item, not a user — so it's precomputable.** Top-K offline per item, a KV lookup at serve time, **no online model in the hot path**, and personalisation is a light re-rank rather than the main event. *"A genuinely different cost profile from every other recommender in this group."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the module and its metrics, why it precomputes, the pipeline, the normalisation and the models, serving and cold start, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Where does the module sit — product page, cart, post-purchase, email?** *(**This determines substitutes vs complements**, which is the whole product.)*
2. **How large is the tail, and do we care about it?** *(If the catalogue is mostly tail, popularity normalisation is the entire job.)*
3. **Is personalisation available and wanted?** *(It's a re-rank here, not the retrieval.)*
4. **How fast does the catalogue churn?** *(Daily refresh is fine for furniture, not for fashion drops — FU 8.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | An e-commerce site: product page, cart, post-purchase email |
| Catalogue | ~10⁷ items; **most of them tail** (few sessions each) |
| Logs | Sessions with views, carts and purchases; ~10⁸ sessions a day |
| Refresh | Daily for most of the catalogue; faster for fast-moving categories |
| Personalisation | Wanted, as a light re-rank over shared lists |

**FR:** top-K similar items per item, per **relation type** · offline computation with a refresh cadence · KV serving · cold start for new items · optional personalised re-rank.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Serving latency | **≤ 10 ms** — it's a KV lookup |
| Coverage | **Every item has neighbours**, including the tail |
| **Popularity concentration** | A monitored guardrail — *what share of all recommendations go to the top 1%?* |
| Refresh | Daily is usually fine; faster for fast-moving catalogues |
| Availability | **Nothing out of stock or delisted** is ever shown |

**Scope / NON-GOALS:**
- ❌ **Raw co-occurrence counts.** Position, with the reason.
- ❌ **An online model in the hot path.** It's precomputable; don't pay for it.
- ❌ Personalised homepage recommendation (G13/G14 — conditioned on a *user*) · search (G7).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
PRODUCT PAGE — Trail running shoe, red          ← user is COMPARING
  Similar shoes                                  ← SUBSTITUTES, labelled
  [ Road shoe A ] [ Trail shoe B ] [ Trail shoe C ] [ … ]

CART — Trail running shoe, red                   ← user has DECIDED
  Goes well with                                 ← COMPLEMENTS, labelled
  [ Running socks ] [ Gaiters ] [ Hydration vest ] [ … ]

  ⚠ never "here's another shoe you could have bought" in the cart
```

```
⚠ THE AGGREGATE METRIC HIDES THE FAILURE.

  a naive item2item system returns the same handful of POPULAR items
  for the ENTIRE CATALOGUE — and it looks FINE on aggregate CTR,
  because those popular items genuinely do get clicked.

  → so the metric must be SLICED BY ITEM POPULARITY DECILE, and the
    number that matters is performance on the TAIL.

AND A GUARDRAIL METRIC, not just a quality one:
  ⭐ POPULARITY CONCENTRATION — what fraction of all recommendations
     across the catalogue point at the top 1% of items?
     Rising ⇒ the normalisation is failing, regardless of CTR.

PLACEMENT-APPROPRIATE OUTCOME:
  product page (substitutes) → did they buy ANY of the compared set?
  cart / post-purchase (complements) → attach rate, basket size
  ⚠ using one metric across both placements grades the wrong thing.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **The placement-appropriate outcome**: purchase of any compared item (product page) · attach rate (cart) | Incremental value, not clicks |
| Quality | Outcome **sliced by popularity decile** · human "is this similar?" judgements | Where the failure lives |
| Coverage | Share of items with ≥ K credible neighbours | The tail gets served |
| **Guardrails** | **Popularity concentration** · out-of-stock impressions = 0 · p99 ≤ 10 ms | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **⭐ the structural simplification** `A1b` `I1a` `I1d`

```
⭐ CONDITIONED ON AN ITEM, NOT A USER → PRECOMPUTABLE.

  catalogue ~10⁷ items × top-K=50 neighbours × ~16 B
    ≈ 8 GB   ← fits in a KV store trivially

  SERVING: item_id → [neighbour list]   ← a LOOKUP. ~1 ms.
  ⚠ NO MODEL IN THE HOT PATH. The whole cost is OFFLINE.

OFFLINE COST — the real work:
  co-occurrence over sessions: a large but ordinary MapReduce/Spark job
  ⚠ the skew problem: the best-seller co-occurs with EVERYTHING, so
    its reducer key is enormous → cap or salt hot keys (same shape as
    the LSH bucket skew in Q8's dedup)
  ⚠ and the pair count is quadratic in session length → cap items per
    session before emitting pairs

⚠ CONTRAST WITH G13: a personalised recommender needs a model per
  request over 10⁹ items. This needs a hash lookup. Say that out loud —
  recognising when a problem is NOT an online ML problem is the point.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Association score | Raw co-occurrence vs **PMI / lift with a support floor** | Raw counts return the best-seller for every item |
| Where the work runs | An online model per request vs **precompute + KV lookup** | Conditioned on an item, so it doesn't depend on who asks |
| Relation | One "similar" list vs **separate lists per relation, picked by placement** | Substitutes and complements are different products |
| Tail and cold start | Behaviour only vs **blend with content embeddings** | Most of the catalogue has too little co-occurrence |
| Representation | Replace with a two-tower embedding vs **blend** | Embeddings help the tail; co-occurrence is explainable and keeps relations apart (FU 9) |
| Complements | Symmetric co-occurrence vs **ordered co-purchase** | A case follows a phone, not the reverse (FU 7) |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE (where everything happens, daily) ═══════════════════════════════
┌─ 1. SESSION / PURCHASE LOGS → PAIRS ───────────────────────────
│ TASK: emit co-view, co-cart and ORDERED co-purchase pairs —
│       ⚠ SEPARATE relations, because they mean different things
│ OUTPUT: pair counts per relation (bots removed, hot keys salted)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. ⭐ NORMALISED ASSOCIATION, NOT RAW COUNTS ──────────────────
│   PMI(x,y) = log[ P(x,y) / (P(x)·P(y)) ]   or lift = P(y|x) / P(y)
│   → divides OUT the popularity of y
│ ⚠ + a MINIMUM SUPPORT threshold, or rare pairs get absurd PMI
│   from 2 co-occurrences
│ OUTPUT: a score per (x, y, relation)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. CONTENT / EMBEDDING SIMILARITY — for the TAIL ─────────────
│ TASK: neighbours from attributes, image and text, for items with
│       too little support, and for cold start
│ BLEND: behavioural where support is high, content where it isn't
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. TOP-K PER ITEM, PER RELATION TYPE ─────────────────────────
│ dedup near-identical listings · diversity within K · blocks
│ OUTPUT: KV STORE  item_id × relation → [neighbours]
└────────────────────────────────────────────────────────────────

══ ONLINE (trivial, ≤ 10 ms) ════════════════════════════════════════════════
 request(item_id, PLACEMENT)
   → pick the RELATION by placement
       product page → SUBSTITUTES · cart / post-purchase → COMPLEMENTS
   → KV lookup → availability filter → light personalised re-rank → serve
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `F1b` `I1b`

- **The CTR trap:** *"The fastest way to raise CTR here is to show more popular items — so I'd never ship on aggregate CTR (FU 6)."*
- **Hot keys:** *"The best-seller co-occurs with everything, so its reducer key is huge — salt or cap it, and cap items per session."*
- **Availability at serve time:** *"Stock changes faster than the offline job, so availability is filtered at lookup, not baked into the list."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **PMI/lift over raw co-occurrence** | "The best-seller co-occurs with everything — raw counts return it for every item, and aggregate CTR won't tell you" | Rare pairs get unstable estimates → needs a support floor |
| **2** | **Precompute + KV, no online model** | "It's conditioned on an item, not a user — recognising it isn't an online ML problem is the design" | Staleness between refreshes; personalisation is only a re-rank |
| **3** | **Separate relations by placement** | "Substitutes on a product page, complements in the cart — one list for both looks reasonable and converts terribly" | Multiple lists to compute, store and evaluate |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Session logs** (views) | Co-view → substitutes | Bots and crawlers view everything |
| **Cart and purchase logs**, with order | Co-cart, **ordered co-purchase** → complements | Sparse; the strongest signal |
| **Catalogue**: attributes, title, images | Content similarity for the tail and cold start | Seller-supplied, inconsistent (G7) |
| **Stock and listing status** | Serve-time availability filter | Changes by the minute |
| **Human judgements** | "Is this a good substitute / complement?" | Cheap to collect; stratify by decile |
| **Merchandiser rules** | Blocks, pins, category rules | Must be versioned (FU 10) |

### 5.2 Building the pairs (e) `F1c`

- **Remove bots and crawlers** before counting — a crawler session co-views thousands of items and floods every pair.
- **Define the window per relation:** a session for co-view, a short ordered window for complements.
- **Cap items per session** before emitting pairs; one 500-item session shouldn't outweigh hundreds of real ones.
- **Deduplicate near-identical listings** first, or "similar" fills with the same product from fifteen sellers.
- **For the embedding model:** positives are **high-PMI pairs**; in-batch negatives over-sample popular items, so apply a **popularity (logQ) correction** or the model learns popularity again.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Component | Task | Objective |
|---|---|---|
| **Normalised association** | Score (x, y) per relation | PMI / lift with **shrinkage toward the prior** below the support floor — counting, not training |
| **Complement direction** | Is y a complement of x? | **Ordered** lift: P(y after x) vs P(y), plus a cross-category check |
| **Content embedding** | Neighbours for the tail and cold start | Contrastive loss on high-PMI pairs, so it learns *behavioural* similarity from content |
| **Light re-rank** | Order the K for this placement and user | GBDT on the **placement outcome** (purchase-of-any / attach), never raw CTR |

### 5.4 Fine-tune an existing model (f) `F2c`

LoRA-tune a pretrained image-text encoder contrastively on the catalogue's high-PMI pairs. It brings general visual and textual knowledge to items with no behaviour; the fine-tune teaches it what *this* catalogue's shoppers treat as similar. Embeddings are computed offline and land in the same KV.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · CLEAN       sessions → bot removal → near-duplicate collapse
STAGE 2 · PAIRS       co-view / co-cart / ordered co-purchase, hot keys salted
STAGE 3 · SCORE       PMI / lift with support floor and shrinkage
STAGE 4 · EMBED       content embeddings for low-support and new items
STAGE 5 · BLEND+TOPK  per relation, dedup, diversity, merchandiser blocks
STAGE 6 · PUBLISH     validate (coverage, concentration) → atomic KV swap
```

---

## 6 · SERVING + COLD START *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
PLACEMENT DETERMINES SEMANTICS — the product decision:
  PRODUCT PAGE (pre-purchase)  → SUBSTITUTES (user is comparing)
  CART / POST-PURCHASE         → COMPLEMENTS
     ⚠ "here's another sofa you could have bought instead" is the
       classic failure — reasonable-looking, converts terribly,
       and mildly insulting after purchase
  EMAIL / RE-ENGAGEMENT        → usually complements + fresh arrivals

COLD START: a new item has NO co-occurrence.
  → CONTENT similarity from day one (attributes, image, text)
  → plus an exploration slot so it can ACQUIRE co-occurrence data,
    otherwise it never gets shown and never earns behavioural signal
    — rich-get-richer again.

PERSONALISATION: a LIGHT re-rank over the precomputed K
  (availability, price band, user's prior brand affinity) —
  ⚠ not a personalised retrieval, which would destroy the
    precomputation that makes this cheap.
```

- **The objective** `A3b`: rank by the placement outcome, **subject to** a popularity-concentration ceiling and a coverage floor — the same constraint-not-weight argument as G7.
- **Efficiency** `F3a`: precompute everything; tiered refresh (head and fast categories often, tail daily or weekly). The trade is staleness, bounded by serve-time availability filtering.
- **Scale** `F3b`: 8 GB replicated KV; the offline job scales with sessions, with hot keys salted.
- **Reliability** `I1b`: a missing or stale list falls back to **content neighbours, then category best-sellers**, never to an empty module; the last-good KV snapshot is kept for rollback.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Sliced by popularity decile** | **The tail is the test.** Aggregate CTR hides the failure entirely |
| **⭐ Popularity concentration** | Share of all recommendations pointing at the top 1% — the guardrail |
| **Placement-appropriate outcome** | Substitutes → purchase of *any* compared item; complements → attach rate |
| **Coverage** | Fraction of catalogue items with ≥K credible neighbours |
| **Human judgement** | On a stratified sample — "is this actually similar?" is cheap for humans to answer |
| **Staleness** | Refresh lag, and whether fast-moving categories need a faster cadence |

```
ROLLOUT:  offline (decile-sliced judgements + replay) → A/B PER PLACEMENT on
          the placement outcome → guardrails: concentration, coverage,
          out-of-stock impressions → atomic KV swap, last-good kept
```

**Success** = the placement outcome up, with tail deciles up rather than only the head, popularity concentration flat or falling, and coverage near complete. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Popularity** | Raw co-occurrence favours best-sellers | PMI / lift; concentration guardrail |
| **Rich-get-richer** | New items never gain co-occurrence | Content cold start; exploration slots |
| **Position** | The first neighbour gets clicked anyway | Position-aware evaluation; interleaving |
| **Seller size** | Large sellers' items accumulate behaviour | Tail-decile evaluation; small-seller coverage |
| **Stereotyped associations** | Co-purchase can encode gendered or cultural stereotypes | Human review of top associations in sensitive categories |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | A ~1 ms KV lookup; the cost is the offline job | p99 lookup · offline $ per refresh |
| **Reliability** `I1b` | Content → category fallback, never an empty module; last-good snapshot; availability at serve time | Fallback rate · out-of-stock impressions |
| **Usability** `I1c` | **The right relation per placement, labelled** ("Similar" vs "Goes well with"); no near-duplicates; no post-purchase substitutes | Placement outcome · dismissals |
| **Scalability** `I1d` | 8 GB replicated KV; salted hot keys; tiered refresh | Job runtime · KV size |
| **Privacy** `I1e` | Co-occurrence is aggregate; the **support floor doubles as k-anonymity** — a rare pair could reveal one person's purchases; **sensitive categories** (health, intimate) excluded from co-purchase relations | Pairs below floor shown = 0 |
| **Safety** `I1f` | No harmful complements (weapon parts, chemical precursors, age-restricted bundles) — a blocklist over the relation | Blocked-pair impressions = 0 |
| **Security** `I1h` | ⚠ **Shilling**: sellers or bots fake co-views and co-purchases to attach their item to a best-seller's page → bot filtering, trusted-actor counting, velocity anomalies on new pairs | Suspicious pair velocity · caught attacks |
| **Ethics / bias** `I1i` | Tail and small sellers get neighbours; stereotyped associations reviewed; merchandiser pins **budgeted** | Small-seller coverage · pinned share |
| **Explainability** `I1j` | A relation label for users; **PMI evidence for merchandisers** ("co-bought 4× more than chance"); every override logged and attributed | % lists with evidence · override audit |

> ⭐ *"The design is cheap, so production readiness here is mostly about the two things that silently erode it: popularity creeping back, and manual overrides piling up."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Session-based sequential models** (item2vec, GRU4Rec): stronger behavioural signal where sessions are rich, at the cost of an offline training pipeline.
- **Bundles and sets:** choosing a *set* that works together rather than top-K independently is a different optimisation.
- **Seasonality:** the same anchor has different complements in December — time-conditioned lists for seasonal categories.

**When the behavioural signal can't be trusted — the content-only pipeline** `A5b` `F2a`:

```
a shilling attack, a logging outage, or a new market with no history
  → switch the affected items or market to CONTENT-ONLY neighbours
    (fine-tuned embeddings, same relation rules by category)
  → fill gaps with category best-sellers, diversity-capped
  → keep the concentration and availability guardrails
  → rebuild behavioural lists from clean logs; blend back in as support returns
  "Content is worse at nuance and immune to fake behaviour."
```

**Top 3 failure modes:**
1. **Best-seller collapse** — every item recommends the same five products, and aggregate CTR says it's working.
2. **Wrong relation for the placement** — substitutes shown after purchase; looks fine, converts terribly.
3. **Tail starvation** — no co-occurrence, no content fallback, so most of the catalogue has no neighbours and is never surfaced.

**Q1 ships:** **PMI-normalised co-occurrence with a support floor + content fallback for the tail + placement-specific relations**, served from a KV store.
> *"All of it is offline work and the serving is a lookup — which is why this is one of the few systems where the whole thing can genuinely ship in a quarter."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "What does 'similar' mean here?"** `A4a` `I1c`
> There are at least four meanings and they're different products: **substitutes** (another item that does the same job), **complements** (bought together), **stylistic** neighbours, and same-category. And the right one **depends on where the module sits**. On a product page before purchase the user is comparing, so they want substitutes. In a cart or post-purchase context substitutes are useless and slightly insulting — "here's another sofa you could have bought instead" — and complements are what convert. So **placement determines the semantics**, which means I'd compute and store separate relation types and select by surface rather than shipping one "similar items" list everywhere. The reason this matters more than it sounds: a module with the wrong relation **looks perfectly reasonable in a demo** and converts terribly, so it survives review and fails quietly in production.

**2. "Why not just use 'users who viewed X also viewed Y'?"** `F1a` `F1b`
> Because **raw co-occurrence is dominated by popularity**. `count(X,Y)` is large whenever Y is popular, regardless of any actual relationship to X — so for almost every X, the top co-occurring item is the current best-seller, because the best-seller co-occurs with everything. A naive item2item system therefore returns the same handful of items for the entire catalogue. And the trap is that **it looks fine on aggregate CTR**, because those popular items genuinely do get clicked, so the metric confirms a broken system. The fix is to normalise: **PMI** — `log[P(x,y) / (P(x)P(y))]` — or lift, which asks whether they co-occur more than chance and divides the popularity of Y straight out. Then evaluate **sliced by popularity decile**, because the failure lives in the tail and the aggregate can't see it.

**3. "PMI over-rewards rare pairs. How do you handle that?"** `F1c` `A3b`
> That's the genuine failure mode of the fix, and it's symmetric to the problem it solves: two obscure items that happened to co-occur twice get an enormous PMI, because the denominator is tiny. So a **minimum support threshold** — a pair needs enough co-occurrences before it's eligible at all — or **shrinkage toward the prior**, which degrades gracefully rather than cliff-edging. It's the same discipline as k-anonymity in autocomplete: rare things shouldn't be promoted on thin evidence, and the threshold is doing both a quality and a stability job. For items that can't clear the support floor — which is most of the tail — I fall back to **content and embedding similarity**, and blend: trust the behavioural signal where support is high, content where it isn't. That blend is what gives the tail neighbours at all, and the tail is most of the catalogue.

**4. "What's the serving architecture?"** `A1a` `F3b`
> A **key-value lookup**, and recognising that is the design. This system is conditioned on an **item, not a user**, so the neighbour list doesn't depend on who's asking — which means it's fully **precomputable**: compute top-K per item offline, store `item_id → [neighbours]`, and serve with a hash lookup in about a millisecond. Ten million items at K=50 is roughly eight gigabytes, which is trivial. **There is no model in the hot path**, and personalisation is a light re-rank over the precomputed K — availability, price band, brand affinity — rather than a personalised retrieval, which would destroy the precomputation that makes the whole thing cheap. The contrast worth stating: a personalised recommender needs a model per request over a billion items; this needs a hash lookup, and **noticing when a problem isn't an online ML problem** is worth more than a better model would be.

**5. "A new item is added. What do you show?"** `F2a` `F1c`
> Content-based neighbours from day one — attributes, title, image embedding — because a new item has **no co-occurrence data** by definition, and waiting for it means the item is never shown, never accumulates behaviour, and never earns behavioural neighbours. That's rich-get-richer again. So: content similarity immediately, then **blend in behavioural signal as support accumulates**, with the blend weight driven by the support count rather than by time. And I'd add an **exploration slot** in other items' neighbour lists so new items actually get impressions — otherwise the content-based neighbours exist but nobody ever sees the item, and it stays cold indefinitely. The reverse direction is easier and worth mentioning: a new item's *own* page needs neighbours, which content similarity handles; getting the new item *into other items' lists* is the harder half and the one that needs the exploration budget.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your CTR went up 8% after a change. Is the module better?"** `I2b` `F1b`
> Not established, and this is the metric trap the whole design exists around. The fastest way to raise CTR on this surface is to **show more popular items**, because popular items get clicked regardless of their relationship to the anchor item — so an 8% CTR gain is entirely consistent with the normalisation having **weakened** and the module having collapsed back toward best-seller recommendations. So before believing it: check **popularity concentration** — what share of recommendations now point at the top 1% — and check **CTR sliced by popularity decile**, because a gain concentrated in the head with tail performance flat or down is the signature. Then check the **placement-appropriate outcome** rather than the click: on a product page, did they purchase any item in the compared set; in a cart, did attach rate and basket size move. A click on a best-seller that was going to be bought anyway is **cannibalisation, not incremental value** — which is the deeper reason CTR is a weak metric here.

**7. "How do you know a complement is actually a complement and not just a popular add-on?"** `F1a` `A2c`
> The same normalisation argument, one level deeper — and it's worth separating **conditional lift from mere frequency**. Batteries co-occur with everything; phone cases co-occur with phones specifically. PMI handles the first order of this, but complements have a structural asymmetry worth exploiting: a genuine complement shows **directional and temporal** structure — the case is bought *after* or *with* the phone, not before, and `P(case | phone) ≫ P(phone | case)`. So I'd compute the relation on **ordered co-purchase within a session or a short window** rather than symmetric co-occurrence, which distinguishes "bought together because related" from "both popular." Category structure helps too: a complement usually crosses categories while a substitute stays within one, so a same-category high-lift pair is probably a substitute mislabelled. And the honest check is the **placement-appropriate outcome** — attach rate on genuinely complementary pairs is much higher than on popular-add-on pairs, so the metric can adjudicate this if you're measuring the right thing.

**8. "Your offline job takes 14 hours and the catalogue changes hourly. Reconcile that."** `A1b` `I1b`
> First I'd ask whether it needs to be reconciled, because for most catalogues **daily is genuinely fine** — item-to-item relationships are stable over days, and refreshing hourly would mostly recompute the same numbers at high cost. Where it isn't fine is fast-moving inventory: fashion drops, ticketing, marketplaces with high churn. Then the answer is **tiered refresh rather than uniform**: recompute the head and the fast-moving categories frequently, the long tail daily or weekly, since the tail's relationships are both more stable and supported by less data anyway. Plus an **incremental path** for the specific cases that matter — a new item gets content-based neighbours immediately at write time, and items whose co-occurrence changed materially are re-scored without a full rebuild. The failure I'd guard against is **staleness in the wrong direction**: recommending out-of-stock or delisted items is worse than recommending slightly stale-but-available ones, so availability filtering happens **at serve time**, not in the offline job — which decouples the freshness requirement from the recomputation cadence entirely.

**9. "Would you replace all of this with item embeddings from a two-tower model?"** `A5a` `I1j`
> It's a reasonable alternative and I'd want to be specific about what changes rather than treating it as strictly better. Embeddings **generalise to the tail** — an item with no co-occurrence still gets a position — and they handle multimodal signal naturally, which is real value. But three things get worse. **Interpretability**: "these co-occurred 4× more than chance" is explainable to a merchandiser and debuggable when wrong; "cosine 0.87" is neither. **Relation types collapse**: a single embedding space encodes one notion of similarity, and I've argued the whole design turns on substitutes-versus-complements being *different* relations — you'd need separate spaces or a conditioned model, which is most of the complexity back. And **popularity bias returns through the back door** if the embeddings are trained on the same interaction data, since popular items get more training signal and end up centrally placed. So my answer is a **blend rather than a replacement**: behavioural association where support is high, embeddings for the tail and cold start — which is what I proposed anyway, with embeddings earning their place exactly where co-occurrence is thin.

**10. "One quarter, and the merchandising team wants manual control over what's shown. How do you handle that?"** `I1j` `A3b`
> I'd build for it deliberately rather than resist it, because the alternative is they override the system out-of-band and you lose both the control and the visibility. Three mechanisms. **Hard blocks** — pairs that must never appear together (competitor products, an item and its own accessory-of-a-different-generation, anything legally sensitive) — implemented as a filter over the precomputed list, versioned and owned, which is cheap and uncontroversial. **Pins** for a small number of business-driven placements, with an **explicit budget**: a pinned slot is a slot the algorithm didn't choose, so I'd cap the share and **measure the pinned slots' performance separately**, which usually settles the argument empirically within a quarter. And **category-level rules** rather than item-level where possible, because item-level overrides accumulate into an unmaintainable list that nobody can reason about and nobody dares delete. What I'd insist on is that overrides are **logged, versioned and attributed**, so six months later you can tell whether a quality regression came from the model or from accumulated manual intervention — same argument as constraints-versus-weights in shopping ranking: make the human decisions **visible artifacts** rather than invisible drift.

---

## ⭐ THE L6 LINE

> **"Two definitional questions carry this: similar *in what sense* — which the placement decides — and the fact that raw co-occurrence returns the best-seller for every item. Normalise with PMI, evaluate on the tail, and notice that it's conditioned on an item rather than a user, so it precomputes to a KV lookup."**

## 🧭 HOOKS — *steer by what you mention*

- *"Placement determines the semantics"* → invites the product-design branch
- *"The best-seller co-occurs with everything"* → invites the statistics/normalisation branch
- *"It's conditioned on an item, not a user"* → invites the systems/cost branch (**good at L6** — recognising a non-ML problem)
- *"Aggregate CTR hides the failure"* → invites the metrics branch (**good at L6**)
- *"PMI over-rewards rare pairs"* → invites the estimation branch
- *"Make manual overrides visible artifacts"* → invites the governance branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the four senses of similar"* — substitutes/complements/style/category regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the popularity problem and tail evaluation than finish the serving. Sound right?"*

**Name and defer:**
- "**Session-based sequential models** (item2vec, GRU4Rec) are a stronger behavioural signal where session data is rich."
- "**Bundling and set recommendation** — choosing a *set* rather than top-K independently — is a different optimisation."
- "**Cross-sell vs upsell** is a business distinction that maps onto complements vs premium substitutes."
- "**Near-duplicate detection** overlaps here — you don't want fifteen listings of the same product as 'similar'."
- "**Multi-objective re-ranking** over the precomputed K (margin, availability, diversity) is where business logic lands."
- "**Seasonality** shifts complements materially — the same anchor item has different neighbours in December."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: precompute, look up | §0, §3 | *"Conditioned on an item, not a user — so it precomputes to a KV lookup."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"10⁷ items × 50 neighbours × 16 B ≈ 8 GB; a 1 ms lookup."* |
| **A1c** | Degrade without lowering quality | §6 | *"A missing list falls back to content neighbours, then category best-sellers."* |
| **A2a** | The full stack | §4 diagram | Pairs → normalised association → content blend → top-K per relation → KV → re-rank |
| **A2b** | Model components | §5.3 | PMI/lift, ordered complement lift, content embeddings, light re-rank |
| **A2c** | Data sources | §5.1 | Sessions, ordered purchases, catalogue, stock, judgements, merchandiser rules |
| **A2d** | Staged pipeline | §5.5 | Clean → pairs → score → embed → blend → publish |
| **A3a** | Top-down, unprompted | §0 | *"Similar in what sense?"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 10 | *"Rank by the placement outcome, subject to a concentration ceiling."* |
| **A4a** | Clarify first, assumptions explicit | §1, FU 1 | Placement · tail size · personalisation · churn |
| **A5a** | Alternatives compared | §4, FU 9 | Raw vs PMI; online vs precompute; one list vs per relation; two-tower vs blend |
| **A5b** | A full alternative pipeline on demand | §9 | The content-only pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 7 | *"PMI with shrinkage; ordered lift for complements; contrastive on high-PMI pairs."* |
| **F1b** | Metrics and their trade-offs | §2, FU 6 | *"Aggregate CTR hides the failure — slice by popularity decile."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Popularity, rich-get-richer, logQ correction, seller size, stereotypes |
| **F2a** | Something fails → a fallback | §9, FU 5 | Content-only pipeline; content cold start |
| **F2b** | Latency or job time too long → design for it | §3, §4 | Hot-key salting and session caps, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A LoRA image-text encoder on high-PMI pairs |
| **F3a** | Efficiency with a quality trade | §6 | Tiered refresh, bounded by serve-time availability |
| **F3b** | Scale levers | §3, §6 | Replicated KV; salted keys |
| **F3c** | Extension, with cost and maturity risk | §9 | Sequential models, bundles, seasonality |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p99 lookup; offline $ per refresh |
| **I1b** | Reliability | §6, §8, FU 8 | Fallback chain; availability at serve time |
| **I1c** | Usability | §2, §8 | The right relation per placement, labelled |
| **I1d** | Scalability | §3, §8 | 8 GB KV; tiered refresh |
| **I1e** | Privacy | §8 | The support floor as k-anonymity; sensitive categories excluded |
| **I1f** | Safety | §8 | No harmful complements |
| **I1h** | Security | §8 | **Shilling** with fake co-views and co-purchases |
| **I1i** | Ethics / bias | §7, §8 | Small sellers; stereotyped associations; budgeted pins |
| **I1j** | Explainability | §8, FU 9, 10 | PMI evidence for merchandisers; logged overrides |
| **I2a** | Offline → online | §7 | Decile-sliced offline → A/B per placement → guarded KV swap |
| **I2b** | The success definition | §7, FU 6 | *"Placement outcome up, tail deciles up, concentration flat."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) pipeline → §4 · (d) data → §5.1 · (e) pairs → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
