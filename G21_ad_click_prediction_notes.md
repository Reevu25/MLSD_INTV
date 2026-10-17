# G21 · Ad Click Prediction — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G21_ad_click_prediction_intv.md`](G21_ad_click_prediction_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **⚠ One of these was a delivery problem, not a content problem:** §1 — the content was already click prediction (calibration 43 mentions, auction 8, and the non-goals exclude bidding), but the **opening led with the auction** and could be *heard* as answering a different question. Now fixed in the sheet.

**Jump:** [1 click prediction or auction?](#1--is-this-sheet-about-click-prediction-or-auction-bidding) · [2 what is calibration](#2--what-is-calibration-in-this-context) · [3 how is AUC measured](#3--how-is-auc-measured-on-pctr) · [4 ⭐ why 20% high overcharges](#4--why-does-running-20-high-overcharge--the-worked-example) · [5 why AUC is invariant](#5--why-is-auc-invariant-to-monotone-transformations) · [6 calibrated but useless](#6--good-calibration-bad-auc--the-degenerate-model) · [7 crosses + hashing + embedding table](#7--the-feature-space--context-crossed-hashing-and-the-embedding-table) · [8 where the 1000 ads come from](#8--where-do-the-1000-candidate-ads-come-from) · [9 feature store + caching](#9--batched-kv-lookups--from-where-caching-what) · [10 ⭐ downsampling correction](#10--negative-downsampling-correction--the-derivation) · [11 why hashed + wide/deep](#11--why-the-regime-forces-hashed-embeddings--a-widedeep-hybrid) · [12 first vs second price](#12--first-price-vs-second-price--why-it-changes-the-cost-of-calibration-error) · [13 click vs qualified click](#13--click-or-click-plus-a-dwellquality-condition)

---

## 1 · Is this sheet about click prediction, or auction bidding?

**Click prediction.** The counts settle it:

| Term | Mentions |
|---|---|
| calibrat* | **43** |
| feature | **41** |
| downsampl* | 11 · pCTR 10 |
| **auction** | **8** · bid 4 · pacing 1 |

And the non-goals explicitly exclude *"bidding strategy (G24) · conversion prediction (G22) · **the auction mechanism itself**."*

### So why does the opening mention the auction at all?

⭐ **Because the auction is the CONSUMER, not the system — and the consumer's requirements are what determine the design.** The chain:

```
the auction consumes pCTR            ← a fact about the environment
  → it MULTIPLIES pCTR by money       ← so absolute value matters, not order
    → CALIBRATION > AUC               ← THE DESIGN DECISION
      → per-slice calibration SLO
      → the downsampling correction is MANDATORY, not optional
      → counter-feature leakage is COMMERCIAL, not just inaccurate
```

**Every one of those is a click-prediction decision.** Without the auction, *"calibration beats AUC"* is an assertion with no backing. The auction appears to **derive** the requirement.

⚠ **But the risk I was pointing at is real — it's a *delivery* risk.** Spend 60 seconds on auctions and an interviewer may think you misheard the question. The fix was one clause:

```
❌ "In an auction, allocation and charge come from pCTR × bid…"
   → sounds like you're designing the auction

✅ "The model I'm building is the pCTR estimator. What makes it
   different from every other binary classifier is WHO CONSUMES IT
   — an auction, which multiplies the probability by money."
```

> ⭐ **The general pattern worth carrying: "who consumes this model's output?" is one of the highest-value clarifying questions in the pack.**
> **G21**: auction → calibration over AUC · **Q12**: RL reward → the judge becomes an *adversarial target* · **G31**: router vs display vs refresh → three cost profiles · **G22**: bidding vs reporting → calibration matters vs lag is tolerable.
> **Same model shape, completely different requirements.**

---

## 2 · What is calibration in this context?

**When the model says 0.02, the true click rate for those impressions is 2%.** You check it by **bucketing**:

```
Take every impression the model scored ~0.02.
Count what fraction actually got clicked.

predicted    actual     verdict
  0.01   →   0.010      ✓
  0.02   →   0.020      ✓
  0.05   →   0.049      ✓     → CALIBRATED

  0.01   →   0.0083     ✗  (1.2× high)
  0.02   →   0.0167     ✗  (1.2× high)
  0.05   →   0.0417     ✗  (1.2× high)
                              → runs 20% HIGH, uniformly
```

⭐ **This is about the ABSOLUTE VALUE of the probability, not the ordering.** In most ML problems you don't care — you threshold or you rank, and only order matters. **Here the number is an input to a money calculation**, so being 20% high isn't a 20% worse model, it's a **20% pricing error**.

⚠ **And "by slice" does real work:** calibrated *on average* while 2× high on mobile and 2× low on desktop is mis-pricing **systematically**, and the global number hides it completely.

---

## 3 · How is AUC measured on pCTR?

**AUC = the probability that a random clicked impression scored higher than a random non-clicked one.**

```
Clicked impressions:     scores  0.05, 0.03, 0.08
Non-clicked impressions: scores  0.01, 0.02, 0.04

Compare every (clicked, non-clicked) pair — 3 × 3 = 9 pairs:
  0.05 > 0.01 ✓   0.05 > 0.02 ✓   0.05 > 0.04 ✓
  0.03 > 0.01 ✓   0.03 > 0.02 ✓   0.03 < 0.04 ✗
  0.08 > 0.01 ✓   0.08 > 0.02 ✓   0.08 > 0.04 ✓

8 of 9 correctly ordered  →  AUC = 0.89
```

> ⭐ **AUC only ever asks "is this one bigger than that one?"** It never asks whether 0.05 was the *right* number. It is a **pure ordering statistic** — which is exactly why it is blind to what matters here.

---

## 4 · ⭐ Why does running 20% high *overcharge*? — the worked example

*(This is the step the sheet asserts without showing. Worth knowing cold.)*

Standard setup — a **second-price-style auction scored by expected value**:

```
EV = pCTR × bid       ← the auction ranks ads by this

winner pays ≈ (EV of runner-up) / (winner's pCTR)
                                   ↑
                   ⚠ THE WINNER'S pCTR IS IN THE DENOMINATOR
```

### Correctly calibrated

| | pCTR | bid | EV |
|---|---|---|---|
| Ad A (winner) | 0.020 | $5.00 | 0.100 |
| Ad B (runner-up) | 0.015 | $4.00 | 0.060 |

```
A pays:  0.060 / 0.020 = $3.00 per click
```

### Now every pCTR runs 20% high — ⚠ note the RANKING IS UNCHANGED, so AUC is identical

| | pCTR (inflated) | bid | EV |
|---|---|---|---|
| Ad A (winner) | 0.024 | $5.00 | 0.120 |
| Ad B (runner-up) | 0.018 | $4.00 | 0.072 |

```
A pays:  0.072 / 0.024 = $3.00 per click     ← per CLICK, unchanged
```

### ⭐ So where is the overcharge? In what the advertiser GETS for the money

```
We told the advertiser to expect:  0.024 × 1000 impressions = 24 clicks
They actually received:            0.020 × 1000 impressions = 20 clicks

⚠ They paid for an impression volume promised to yield 24 clicks
   and got 20.  EFFECTIVE CPC IS 20% HIGHER THAN QUOTED.
```

**Three concrete harms, all from the same inflation:**

- ⭐ **Budget burns faster than it delivers.** Pacing spends against **predicted** clicks, so the daily budget is consumed 20% faster than actual click volume justifies — full price for 80% of the promised outcome.
- ⚠ **Mis-allocation once miscalibration is per-slice.** The uniform case above preserves ranking. Real miscalibration is **per-slice** — 2× high on mobile, 2× low on desktop — and *then* **the ranking genuinely breaks**: mobile ads win auctions they should have lost, so advertisers are charged for wrongly-allocated impressions.
- **Reserve prices and floors misfire.** `pCTR × bid` is compared against revenue floors; inflate it and ads clear a floor they shouldn't, systematically over-serving low-quality inventory.

> ⭐ **The one-liner:** *"Inflated pCTR means we promise more clicks than we deliver. The advertiser's budget is consumed against a **prediction**, not against reality — so they pay for performance that never arrives, and the effective CPC is higher than quoted."*

---

## 5 · Why is AUC invariant to monotone transformations?

**Because AUC is computed entirely from comparisons, and a monotone function preserves every comparison.**

A monotone-increasing `f` satisfies:

```
a > b   ⟺   f(a) > f(b)      ← for ALL a, b
```

So every pair AUC examines gets the same verdict:

```
BEFORE:  clicked = 0.05,  non-clicked = 0.04   →  0.05 > 0.04  ✓

Apply f(x) = 1.2x  (monotone):
AFTER:   clicked = 0.06,  non-clicked = 0.048  →  0.06 > 0.048 ✓
```

The count of correctly-ordered pairs is unchanged, so AUC is **bit-identical** — not approximately, *exactly*.

### Transforms this covers — every one a pricing catastrophe

```
f(x) = 1.2 · x                 scale up 20%        → AUC UNCHANGED
f(x) = x²                      squash the low end  → AUC UNCHANGED
f(x) = sigmoid(10 · logit(x))  sharpen             → AUC UNCHANGED
f(x) = 0.5 · x + 0.01          shift and scale     → AUC UNCHANGED
```

> ⭐ **This is why it's a *structural* blindness, not a weak signal.** AUC is **mathematically incapable** of detecting a monotone rescaling — no sample size, no slicing, no threshold choice helps, because **the information is not in the statistic.**
>
> **So "AUC can't see calibration error" is a proof, not an observation.** Saying it that way signals you understand the statistic rather than having memorised the advice.

---

## 6 · "Good calibration, bad AUC" — the degenerate model

Base CTR is 1.5%. Build a model that outputs **`0.015` for every impression** — ignoring user, ad, context. One constant.

**Calibrated?** ⭐ *Perfectly.*

```
Every impression it scored 0.015  → that's ALL of them
Actual click rate in that bucket  → 1.5%
→ ECE = 0.  You cannot do better.
```

**Useful?** *Not at all.* **AUC = 0.5:**

```
Clicked:      0.015, 0.015, 0.015
Non-clicked:  0.015, 0.015, 0.015
Every pair is a TIE → no pair correctly ordered → AUC = 0.5 (random)
```

### ⚠ Why this matters: it proves calibration ALONE is insufficient

> **If you gated only on calibration, this model would pass with a perfect score.**

Which is exactly why AUC stays on the sheet as a **secondary discriminative check**:

| Metric | Question | Role |
|---|---|---|
| **AUC** | *Did the model learn anything?* | A **floor** — prove it beats the constant |
| **Calibration** | *Are the numbers right?* | The **gate** — prove it prices correctly |

**In that order of logic:** AUC establishes there is signal; calibration establishes the signal is correctly scaled.

### ⭐ The auction consequence — the interesting part

```
Every ad's EV = 0.015 × bid
→ pCTR is a CONSTANT, so it FACTORS OUT of the comparison
→ the auction ranks purely by BID
→ ⚠ THE HIGHEST BIDDER ALWAYS WINS, regardless of relevance
```

**You have silently converted a quality-weighted auction into a pure price auction.** And the consequences are counterintuitive:

- **Short-term revenue may look fine, or even rise** — you always serve the highest bidder
- **Ad relevance collapses** — users see expensive ads, not relevant ones
- **CTR falls** → advertisers get worse performance per dollar → reduce budgets → ⚠ **revenue declines on a lag long enough that nobody attributes it correctly**

*(Same borrowed-revenue pattern as the sheet's follow-up "revenue is up 3% — is that good?")*

### The version that actually happens

Nobody ships a literal constant. The realistic failure is **weak discrimination that passes a calibration check**:

```
A model predicting 0.014–0.016 for everything:
   calibration  ✓  (aggregate is right)
   AUC          ≈ 0.52   ← almost no signal
→ ⚠ CALIBRATED, AND NEARLY WORTHLESS
```

A dashboard showing only ECE would call that healthy.

> ⭐ **The practical diagnostic: always compare AUC against the base-rate null model.** If AUC is barely above 0.5, the model has learned essentially nothing and **you are effectively running a bid-only auction** — whatever the calibration plot says.

---

## 7 · The feature space — "context crossed", hashing, and the embedding table

### 7a · What a "cross" is, and why the space is billions

```
USER    : user_id · age bucket · country · device · language
          · topic affinities · past-click history
AD      : ad_id · advertiser_id · campaign_id · creative_id
          · category · landing-page domain
CONTEXT : query (search) or page topic (display) · time of day
          · day of week · slot position · placement
```

A **cross** is the conjunction of two or more, treated as **ONE new categorical feature**:

```
user_id × ad_id          → "has THIS user clicked THIS ad before?"
device × ad_category     → "mobile × travel"
query_token × advertiser → "'cheap flights' × Expedia"
```

> ⭐ **Crosses are where the predictive power is**, because click behaviour is **interaction-driven**: *mobile users click travel ads* is a fact neither "mobile" nor "travel" expresses alone. **A linear model over raw features cannot represent it — you must hand it the conjunction.**

**And crosses are why the space is billions** — it multiplies: `10⁸ users × 10⁷ ads = 10¹⁵ possible pairs`, plus `10⁵ query tokens × 10⁶ advertisers = 10¹¹`, plus dozens more templates. Each distinct value is conceptually its own binary feature ⇒ ⚠ **one-hot would need a 10¹⁵-wide vector. Impossible.**

### 7b · The hashing trick

```
WITHOUT hashing (a vocabulary):
  "user_8841 × ad_229" → dict lookup → index 4,192,885
  ⚠ the dict must hold every value ever seen, grows forever,
    and a NEW value has NO INDEX AT ALL

WITH hashing:
  index = hash("user_8841 × ad_229") mod 2²⁴     (16.7M buckets)
  → O(1) · no dictionary · no memory growth
  → ⭐ a brand-new ad or user gets an index IMMEDIATELY
```

| Property | Why it matters here |
|---|---|
| **Fixed memory** | You choose the table size; the feature space doesn't dictate it |
| **No vocabulary** | ⭐ **New ads and users appear constantly** — a fixed vocab is stale the day you ship |
| **Stateless** | Any serving node computes the same index with no coordination |

⚠ **The cost is collisions** — two unrelated values share an embedding row:

```
hash("Germany × campaign_8841") = 12,043
hash("mobile × ad_77291")       = 12,043    ← same row
```

**Why tolerable:** collisions are **random**, so they add *noise* rather than *bias*, and the frequent values carrying most of the signal mostly land in distinct buckets. **Mitigations:** size the table so collision rate is low for *head* values · or **multiple independent hash functions** summed (a collision in one rarely coincides with a collision in another).

### 7c · The embedding table — one row per hash bucket

```
Embedding table:  2²⁴ rows × 32 dims × 4 B  ≈  2 GB

row 0        : [0.21, -0.03, ..., 0.44]   ← whatever hashes here
row 12,043   : [...]  ← "Germany × campaign_8841" AND its collision
row 16,777,215
```

**The table IS the learned meaning of each bucket.** A sparse value becomes dense by looking up its row:

```
input   : {user_8841, ad_229, mobile, travel, "user_8841×ad_229", ...}
  ↓ hash each
indices : [4192885, 881, 12043, 90122, 551, ...]      ~100 of them
  ↓ look up each row, then pool (concat or sum)
dense   : [0.21, -0.03, 0.44, ...]                    ~few hundred dims
  ↓
MLP → pCTR
```

> ⭐ **This is why the embedding table dominates memory** (10⁸–10⁹ rows = tens to hundreds of GB) **while the MLP on top is tiny.** That asymmetry is the whole serving story — **the table gets sharded, the MLP gets replicated.**

### 7d · Yes — the MLP takes a vector and outputs pCTR

```
in : dense vector (~hundreds of dims) = pooled embeddings + numerics
out: one scalar in (0,1) = pCTR

shape: ~512 → 256 → 128 → 1, sigmoid at the end
~1 ms for ALL 1000 candidates in ONE batched forward pass
```

⚠ **The model is genuinely small** — all the parameters are in the embedding table. Which is why *"optimise the model"* is the wrong instinct here, and the opposite of an LLM.

---

## 8 · Where do the ~1000 candidate ads come from?

> ⭐ **Per REQUEST, not per user.** Every search query or ad-slot opportunity gets its own fresh candidate set — a user who searches five times triggers five independent retrieval → scoring → auction cycles. Nothing carries over except the user features in the store. One candidate set fills **all** the ad slots on that page (one auction, several positions). The ~1000 is an order of magnitude: a niche query may match a handful (or zero ⇒ no ads shown), a competitive one like *"car insurance"* thousands.
>
> ⚠ **And this is what exposed the sheet's capacity error:** it said ~10⁶ predictions/s *and* ~1000 candidates/request ⇒ only **10³ requests/s**, far too low for Google-scale search (~10⁵ queries/s). The consistent version: **10⁵ requests/s × ~1000 candidates = ~10⁸ predictions/s**, at ~10 µs per prediction. **Always state the two rates separately — an interviewer will multiply them.** *(Fixed in the sheet, the study file, the diagram, and the Tier 1 quick-ref.)*

⚠ **An ad-retrieval stage upstream — and it is NOT in scope for this question.** Worth saying rather than hand-waving:

```
ALL ads (10⁷)
  → TARGETING MATCH: does this ad's targeting admit this user/context?
      keyword/query match (search) · audience segments · geo
      · language · device · demographic
  → HARD FILTERS: budget not exhausted · campaign active
      · frequency cap not hit · brand-safety and policy eligible
  → ~1000 ELIGIBLE  →  pCTR scoring (THIS sheet)
  →  pCTR × bid  →  auction  →  ~1–5 shown
```

> ⭐ **The key difference from G13:** candidate generation here is largely **deterministic matching against advertiser-declared targeting**, not learned retrieval. **The advertiser *told you* who they want** — so it's an inverted-index intersection, not an ANN search. (Some systems add learned retrieval for broad-match, but targeting and budget filters dominate.)

---

## 9 · Batched KV lookups — from where? Caching what?

**From a feature store** — a low-latency KV service holding precomputed feature values:

```
FEATURE STORE (Bigtable / Redis / Cassandra-class, sharded by key)
  key: user:8841      → {age_bucket, country, topic_affinities,
                         7d_click_count, 30d_ctr, ...}
  key: ad:229         → {category, advertiser, creative_age,
                         7d_impressions, 7d_ctr, ...}
  key: u8841:adv5512  → {has this user clicked this advertiser before}
```

**"Batched"** = ONE request carrying **all 1000 ad keys**, not 1000 round trips. The client fans out per shard ⇒ ~N requests for N shards. ⭐ **That's the difference between feasible and impossible in a 10 ms budget.**

| Cache | Contents | Why |
|---|---|---|
| **In-process, on the serving node** | ⭐ **Ad-side features** | Ads change slowly — category and 7d CTR are stable for minutes; a hot ad set covers most traffic |
| **In-process, per request** | Advertiser / campaign features | ~1000 ads often share only a few hundred advertisers |
| ⚠ **NOT cached** | **User features** | Different user every request — nothing to reuse |
| ⚠ **NOT cached** | **Counter features** | **Freshness IS the point** — a stale CTR counter defeats the feature |

> ⭐ **So the cacheable side is ads and the uncacheable side is the user** — which is exactly why the fetch is the bottleneck: **you cannot amortise the per-request half.**

⚠ **And the p99 problem:** the slowest of ~1000 parallel lookups sets your latency. **Mitigations:** timeout-with-partial-results (serve with features missing rather than failing) · **hedged requests** to a second replica · tight tail-latency SLOs on the store.

---

## 10 · ⭐ Negative-downsampling correction — the derivation

### Why downsample

```
Base CTR < 1%:  10⁹ impressions/day → ~10⁷ clicks, ~9.9×10⁸ non-clicks
⚠ 99% of training compute goes into negatives that are individually
  almost uninformative.
```

Keep **all positives** and a fraction **`w`** of negatives — say `w = 0.01`:

```
Original:    10⁷ pos : 9.9×10⁸ neg    (1 : 99)
Downsampled: 10⁷ pos : 9.9×10⁶ neg    (1 : ~1)
→ ~100× less data, ranking signal preserved
```

### ⚠ Why the output is now wrong

The model learned on a distribution where **positives are ~100× over-represented**, so it predicts the click rate **of that re-weighted world**:

```
True CTR        = 1%
Downsampled CTR ≈ 50%     ← the model learns ~0.5, not ~0.01
⚠ every prediction inflated by roughly 1/w
```

> ⭐ **It is purely a CALIBRATION error — the ordering is untouched, so AUC is IDENTICAL and nothing on a standard dashboard moves.** That's why it's the most-forgotten detail in the system, and why it's called a *mechanical trap*.

### The derivation

Downsampling multiplies the **negative-class** weight by `w`, so in **odds** terms:

```
odds' = p'/(1−p')  =  (1/w) × [ p/(1−p) ]  =  odds / w
```

Inverting to recover `p`:

```
                  p'
      p  =  ──────────────────
             p' + (1 − p')/w
```

### ⭐ Sanity-check it with numbers — `w = 0.01`, model outputs `p' = 0.5`

```
p = 0.5 / (0.5 + 0.5/0.01)
  = 0.5 / (0.5 + 50)
  = 0.5 / 50.5
  = 0.0099   ≈ 1%        ✓ back to the true base rate
```

### Why forgetting it "inflates every price"

```
forget the correction
  → every pCTR ≈ 50× too high
  → EV = pCTR × bid is ~50× too high for every ad
  → ⚠ budgets consumed against a prediction promising ~50× the clicks
  → the advertiser pays for performance that was never going to arrive
  → and reserve prices / revenue floors are cleared by EVERYTHING,
    so low-quality inventory gets served
```

Per §4: the *per-click* price may survive (numerator and denominator both inflate), but **the budget burns against predicted clicks**, so delivery collapses against what was quoted.

> ⚠ **And the insidious part: AUC unchanged, ranking unchanged, the model "looks fine."** You find out from advertiser complaints.

### Then the calibration layer — why BOTH

```
raw output p'
  → ① ANALYTIC downsampling correction   (known shift, closed form, free)
  → ② LEARNED calibration layer          (isotonic / Platt, per slice)
  → calibrated pCTR → × bid → auction
```

**①** is exact but only fixes the *sampling*. **②** catches everything else — distribution drift, per-slice miscalibration, model-specific bias — and is cheap to **refit hourly without retraining the model**.

---

## 11 · Why the regime forces hashed embeddings + a wide/deep hybrid

| Constraint | What it kills |
|---|---|
| **~10⁸ predictions/sec** *(10⁵ requests × ~1000 candidates)* | Anything expensive per prediction — the budget is **~10 µs per ad** |
| **Single-digit ms** | No transformer, no cross-attention, no multi-pass. ⚠ And **the fetch**, not the model, already eats most of it |
| **Billions of sparse values** | One-hot impossible; a vocabulary unmaintainable → **hashing** |

**What survives:** hashed embeddings (fixed memory, no vocab, new values work instantly) + a small MLP (one batched pass, ~1 ms).

### ⭐ And why *hybrid* specifically — two incompatible needs

```
MEMORISATION  (wide — linear over EXPLICIT crosses)
  "user_8841 × ad_229 has clicked 4 of 5 times"
  → an EXACT, high-value fact. An embedding would SMOOTH IT AWAY.

GENERALISATION  (deep — MLP over embeddings)
  "users with travel affinity on mobile click flight ads"
  → covers the TAIL and COLD START, where no exact history exists
```

> ⭐ **Neither alone is sufficient.** Pure wide **cannot handle a new ad** — no history, no prediction. Pure deep **smooths away the sharp exact-pair signals** that are the most reliable evidence available.
>
> **The hybrid is a direct consequence of a power-law feature distribution: a dense head that deserves memorising, and a vast tail that needs generalising.**

*(On "could it be a random forest?" — no. Trees handle neither billions of sparse categorical values nor the embedding lookup that makes them tractable. GBDTs win on **small, dense, tabular** feature sets — which is why G29's fraud model is a GBDT and this one isn't.)*

---

## 12 · First-price vs second-price — why it changes the cost of calibration error

**First-price** — you pay your own bid:
```
price = your bid
⚠ pCTR affects ALLOCATION ONLY — who wins, not what they pay
```

**Second-price (EV-based)** — you pay roughly what you needed to beat the runner-up:
```
price ≈ (runner-up EV) / (your pCTR)
⚠ pCTR appears in BOTH allocation AND the price formula
```

| | First-price | Second-price |
|---|---|---|
| pCTR error → **wrong winner** | ✓ | ✓ |
| pCTR error → **wrong price** | ✗ *(price = your bid)* | ⭐ **✓ — directly in the formula** |
| Calibration error costs you | Mis-allocation | **Mis-allocation *and* mis-pricing** |

> ⭐ **So under second-price, calibration error is twice as expensive** — it corrupts both who wins and what they pay. Under first-price **with human-set bids**, a uniform miscalibration is relatively benign for *pricing* (everyone shifts, ranking holds, you pay your bid regardless) — though ⚠ **per-slice** miscalibration still breaks allocation.

### ⚠ Correction — that "benign" case mostly no longer exists: **autobidding**

**What platforms actually run** *(Google doesn't publish full mechanics; this is the publicly reported picture)*:

| Surface | Auction | Bidding |
|---|---|---|
| **Display / programmatic** (Ad Manager, AdX) | ⭐ **Unified first-price since 2019** (publicly announced) | Autobidding widely used |
| **Search ads** | **Not pure first-price** — a GSP-like hybrid ranked by **Ad Rank** (bid × quality signals × thresholds), with pricing adjustments on top | ⭐ **Smart Bidding** — the dominant way advertisers bid |

**Why display went first-price: header bidding.** Publishers send each impression to many exchanges at once and take the highest response — so a second-price auction *inside* one exchange stops being second-price once its winner competes against other exchanges. **First-price is the format that stays coherent when auctions are nested.**

**The idea of autobidding** — the advertiser states a **goal**, not a bid:

```
Target CPA  : "conversions at ~$20 each"
Target ROAS : "$4 revenue per $1 spent"
Max conversions within a $5k/day budget
```

…and **the platform computes a bid for every single auction:**

```
bid = pCTR × pCVR × value-per-conversion
    = pCTR × pCVR × $20                      (target-CPA example)
```

Per-auction, using everything the platform knows about the user and context — which a static human keyword bid can't do. And under first-price, deciding how far below your value to bid (**bid shading**) is hard for a human and natural for an algorithm — which is why the two arrived together.

### ⭐ Why it changes the conclusion: **pCTR is now inside the BID**

```
pCTR runs 20% high
  → bid = pCTR × pCVR × $20 is 20% high      ← the bidder trusts pCTR
  → first-price: you PAY your bid
  → ⚠ the advertiser overpays by 20%, DIRECTLY
```

| Setup | Pricing sensitivity to pCTR error |
|---|---|
| Second-price, human bids | **Direct** — pCTR in the price formula |
| First-price, human bids | Allocation only — relatively benign for price |
| ⭐ **First-price + autobidding** | **Direct again** — pCTR is in the *bid*, and you pay the bid |

> ⭐ **So on modern platforms calibration is non-negotiable whatever the auction format.** The error comes back into the price through the bidder instead of the price formula — **a stronger version of the sheet's argument, not a weaker one.**

**Which is why it's still a minute-one clarifying question** — but the honest answer now is *"ask, and expect calibration to be non-negotiable either way."* The only setup where it's weaker (first-price, human-set bids) is the least common one at scale.

> ⚠ **One structural point worth naming:** with autobidding **the platform sits on both sides of the transaction** — it computes the advertiser's bid *and* runs the auction that charges it. That raises a trust question (is the bidder optimising for the advertiser or the platform?) that advertisers and regulators have scrutinised. For the design it means **calibration is also a trust property**: a miscalibrated pCTR inside the bidder systematically overcharges the customers whose money the platform is spending on their behalf.

---

## 13 · "Click, or click plus a dwell/quality condition?"

The label definition, and it's the same argument as G2's watch-through and G7's purchase-without-return:

```
NAIVE LABEL:   click = 1
⚠ a click followed by an instant bounce is a NEGATIVE outcome —
  the user was misled, the advertiser paid, nobody benefited

QUALIFIED LABEL:  click AND (dwell > N seconds  OR  a landing-page
                             event fired  OR  not an immediate back-click)
```

**Why it matters:** optimising raw clicks rewards **misleading creatives**, exactly as CTR optimisation rewards clickbait in G2. A qualified label makes the model predict *valuable* clicks.

**⚠ The costs, which is why it's a question rather than an obvious yes:**
- **The label gets sparser** — you've thrown away the fast-bounce clicks, which were a meaningful fraction
- **It needs landing-page instrumentation**, which is often the *advertiser's* site and therefore outside your control
- ⚠ **It changes what you're calibrated against** — "pCTR" now means P(qualified click), so **the bid semantics change too**, and the advertiser needs to understand which they're bidding on

> ⭐ **So it's a question because the answer changes both the objective and the pricing contract** — not just a modelling tweak.

---

## 🔁 THE NINE THINGS I KEPT GETTING WRONG

1. **The sheet is about the pCTR model** — the auction is the **consumer** whose requirements derive the metric choice, not the system being designed.
2. **Calibration is about the absolute value; AUC is about order.** They are **independent properties**.
3. ⭐ **The overcharge isn't in the per-click price** — it's that the budget is consumed against a *prediction*, so the advertiser gets fewer clicks than promised. **Effective CPC rises.**
4. **AUC's blindness is a proof, not a weakness** — monotone `f` preserves every pairwise comparison, so the information isn't in the statistic.
5. ⚠ **Calibration alone is not enough** — a constant base-rate model is perfectly calibrated, has AUC 0.5, and turns the auction into pure bid-ranking.
6. ⭐ **The embedding table is the memory story; the MLP is tiny.** All the parameters are in 10⁸–10⁹ hashed rows — so the table shards and the MLP replicates.
7. **Hashing has no vocabulary, which is the point** — a brand-new ad gets an index immediately. Collisions add noise, not bias.
8. ⚠ **Downsampling makes the model predict the re-weighted world's CTR** (~50%, not ~1%) — and AUC is identical, so nothing on a dashboard moves.
9. **Second-price puts pCTR in the price formula** (calibration error costs twice: wrong winner *and* wrong price). ⚠ **First-price + autobidding puts it back in the price via the BID** — `bid = pCTR × pCVR × target` and you pay your bid. Only first-price with *human* bids is benign, and that's the rare case at scale. **Calibration is non-negotiable either way.**
