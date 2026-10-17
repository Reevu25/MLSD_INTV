# G21 · Ad Click Prediction — **INTERVIEW SHEET**

> **Tier 1 · #3** — calibration > AUC; the Google-specific label-leakage angle.
> Full study file: [`../G21_ad_click_prediction.md`](../G21_ad_click_prediction.md) · [diagrams](../G21_ad_click_prediction_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the scoring stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Auction rule, label definition, new-ad SLA, surface — and the **scoping line** (§0, §1) |
| **Explainability** is easy points | Advertiser-facing **Quality Score**, per-slice calibration plots, reason codes for disputes (§8) |
| **Security** beyond the gateway | **Invalid traffic / click fraud** corrupting labels *and* charges (§8) |
| **Troubleshoot before being asked** | Raise the **10 ms squeeze** and **counter-feature leakage** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **The output is a price, not a ranking** · scope to the pCTR model | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The auction as consumer · **calibration by slice** > log loss > AUC | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | 10⁵ req/s × 1,000 = **10⁸ predictions/s** · ~10 µs each · the fetch dominates | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Model-family alternatives · fetch → hash → hybrid → correction → calibration | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Position bias · **downsampling correction** · point-in-time features · objective | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | The fetch is the bottleneck · quantized tables · exploration budget | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Calibration gates · both sides of the marketplace · auto-rollback | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multimodal creatives · cold-start path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The model I'm building is the pCTR estimator. What makes it different from every other binary classifier is *who consumes it* — an auction, which multiplies the probability by money. So the output is a price, not a ranking.**
>
> **Allocation and charge come from `expected value = pCTR × bid`, so that single fact reorders every priority: calibration beats AUC, and it's not close. A model with AUC 0.80 and perfect calibration prices every impression correctly; a model with AUC 0.82 that runs 20% high overcharges every advertiser — and looks better on the metric most teams report.**
>
> **Because AUC is invariant to any monotone transformation of the scores, it literally cannot see calibration error."**

> ⭐ **And scope it in one line, because "design ad click prediction" is ambiguous:** *"I'll take this as **the pCTR model that feeds the auction** — bidding and pacing are a separate system, conversion prediction is another. Shout if you wanted the broader stack."*

**Then three commitments:**
1. **Headline metrics are log loss and calibration BY SLICE** — AUC demoted to a secondary discriminative check.
2. **The regime drives the architecture** — ~10⁸ predictions/sec (10⁵ requests × ~1000 candidates each), single-digit ms, billions of sparse categorical values → hashed embeddings + memorisation/generalisation hybrid.
3. **Two mechanical traps I'll handle explicitly** — position bias, and **negative-downsampling correction**. Forgetting the second inflates every price.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, how the auction consumes the score and what I measure, capacity, the architecture, data and training, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What auction and pricing rule?** *(Second-price puts pCTR into the price formula; first-price with **autobidding** puts it back in through the bid — `bid = pCTR × pCVR × target` — so calibration matters either way.)*
2. **Is the label a click, or a click plus a dwell/quality condition?** *(Changes the label definition entirely.)*
3. **How fast must a new ad get traffic?** *(Cold start is an advertiser-facing SLA, not just an ML problem.)*
4. **Which surface: Search ads, or display / YouTube?** *(Different features, different position effects, different calibration slices.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | **Search ads**; display as a variant |
| Auction | GSP-like, with advertisers using automated bidding |
| Load | ~10⁵ requests/s × ~1,000 candidate ads each |
| Label | A click, **after invalid-traffic filtering** |
| Freshness | Hourly retraining; new ads need traffic within hours |
| Personalisation | Consent-gated; **no targeting on sensitive categories** |

**FR:** pCTR per (user, ad, context) · a calibrated **absolute** probability · cold start for new ads · per-slice calibration monitoring · fast retraining with automated gates.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Scoring latency** | **p99 ≤ 10 ms** for ~1000 candidate ads |
| Request rate | ~10⁵ requests/sec (search queries / ad-slot opportunities) |
| **Prediction rate** | **~10⁸ pCTR predictions/sec** — every request scores ~1000 candidates |
| **Calibration error** | **< 2% per major slice** ← the gate |
| Model freshness | Hours, not days (ads and users churn fast) |
| Availability | 99.99%; on a fetch timeout, **calibrated priors**, never zero |

**Scope / NON-GOALS:**
- ❌ **Gating on AUC.** Position, with the reason.
- ❌ Bidding strategy (G24) · conversion prediction (G22) · the auction mechanism itself · ad creative quality.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the product is (b):** the pCTR model has three audiences.

```
1. THE AUCTION (the consumer)
   for each query: ~1,000 eligible ads → pCTR × bid (+ quality) → ranked
   → the top few are SHOWN and each click is CHARGED from the same scores

2. THE USER: a results page with a few relevant ads, marked "Sponsored"

3. THE ADVERTISER: a dashboard with clicks, cost per click, and a
   QUALITY SCORE (expected CTR · ad relevance · landing-page experience)
   → the pCTR system feeds an explanation advertisers act on
```

**The metrics:**

```
⚠ WHY AUC CANNOT SEE THE FAILURE THAT MATTERS

  AUC measures RANKING ONLY. It is invariant to any monotone
  transformation of the scores.
      pCTR × 1.2  →  SAME AUC, every advertiser overcharged 20%.

  → so AUC is structurally blind to mis-pricing.

HEADLINE:  LOG LOSS  +  CALIBRATION BY SLICE
SECONDARY: AUC (discriminative sanity check only)

⚠ "BY SLICE" IS DOING REAL WORK:
  a model calibrated ON AVERAGE while 2× high on mobile and
  2× low on desktop is mis-pricing SYSTEMATICALLY —
  and the global number hides it completely.
```

**Slices that must each be calibrated:** device · placement · vertical · new vs established advertiser · geo · time-of-day.

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | Revenue that **holds**: advertiser ROI steady or rising, user ad experience intact | A revenue win taken out of advertiser ROI is borrowed |
| Offline | **Log loss + calibration by slice** · AUC as a sanity check | The gate |
| Online | Revenue · CTR · **advertiser conversions / ROI** · user ad-relevance signals | Both sides of the marketplace |
| **Guardrails** | Calibration error < 2% per slice · p99 ≤ 10 ms · invalid-traffic rate · long-term ad-blindness holdout | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
⭐ STATE THE TWO RATES SEPARATELY — the multiplication is the point:

  REQUESTS     ~10⁵ /sec       one per search query / ad-slot opportunity
  × CANDIDATES ~1000 /request  fresh per request — NOT per user
  = PREDICTIONS ~10⁸ /sec      ← what the scoring fleet actually serves

  ≤10 ms p99 per request, for ALL ~1000 candidates
  → budget per prediction ≈ 10 ms / 1000 = ~10 µs, and the fetch eats most of it
  ⚠ quoting "10⁶ QPS" alone hides which rate you mean — an interviewer
    will multiply, so do it for them

FEATURE SPACE: billions of categorical values
  (user × ad × context crossed)
  → one-hot is impossible → HASHING TRICK into a fixed table
  → embedding table is the memory story: 10⁸–10⁹ rows × ~16–64 dims

⚠ THE BOTTLENECK IS NOT THE MODEL:
  model forward pass on a small MLP  ≈  ~1 ms
  FETCHING ~1000 ads × ~100 features  ≈  the rest of the budget
  → this is a DISTRIBUTED LOOKUP problem with a model attached
    (same shape as G29's graph aggregates and Q7's KV cache)

COST: track $ per MILLION predictions; at 10⁸/s the fleet is large, so the
  first levers are ad-side feature caching and quantized tables (§6)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| Logistic regression + hand-made crosses | Fast, naturally calibrated, easy to debug | No generalisation to unseen combinations |
| GBDT | Strong on dense numeric features | Weak on **billions of sparse IDs** |
| **Wide & deep / DCN / DLRM-style** ✅ | The wide side memorises exact pairs; the deep side generalises over hashed embeddings | Big embedding tables to shard and serve |
| Calibration inside the model vs a **separate calibration layer** ✅ | A separate layer refits hourly per slice | One more component to monitor |

> *"The feature space decides it: billions of sparse IDs need embeddings, and exact-pair memorisation needs the wide side."*

### The system — *draw this in 90 seconds* `A2a`

```
 REQUEST (user, query, context) + ~1,000 candidate ads from ad retrieval
       │
┌─ 1. FEATURE FETCH  ⚠ THE REAL BOTTLENECK ──────────────────────
│ user · ad · query · cross features · COUNTER FEATURES (ad CTR so
│ far, user CTR) — all POINT-IN-TIME · batched KV lookups, ad side cached
│ OUTPUT: ~100 features × 1,000 ads
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. HASHING → EMBEDDING LOOKUP ────────────────────────────────
│ billions of sparse values → a fixed hashed table (no vocabulary)
│ OUTPUT: dense vectors
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MODEL — wide (memorise) + deep (generalise) ───────────────
│ + SHALLOW POSITION TOWER, dropped at serving
│ one batched forward pass for all 1,000 ads
│ OUTPUT: p′ (on the downsampled training distribution)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. ⚠ DOWNSAMPLING CORRECTION, THEN CALIBRATION ───────────────
│ p = p′ / (p′ + (1 − p′)/w)   →   isotonic / Platt layer per slice
│ OUTPUT: calibrated pCTR — an absolute probability
└──────┬─────────────────────────────────────────────────────────
       ▼
 pCTR × bid → AUCTION → allocation + charge
       ▼
 LOGGING → training   ⚠ logs carry position bias and only shown ads have labels
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The 10 ms squeeze:** *"If 1,000 candidates don't fit in 10 ms, I add a light first-pass model (1,000 → 100) and run the full model on the top 100. And if a feature fetch times out, I fall back to **calibrated ad-level priors**, never to zero, because a zero is a mis-price."*
- **Counter-feature leakage:** *"'This ad's CTR so far' is the strongest feature and it leaks the label if its window includes the current impression. Offline looks spectacular, online collapses. Every counter is computed **strictly before** the event."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Calibration layer on top of the model** | "The model optimises log loss; isotonic/Platt fixes residual bias per slice cheaply and can be refit hourly" | Another component to monitor; can mask a real model regression |
| **2** | **Hashing over vocabulary** | "Billions of values, and new ones appear constantly — a fixed vocab is stale the day you ship it" | Hash collisions; mitigated by table size and multiple hashes |
| **3** | **Downsample negatives, then correct** | "Base CTR <1% — training on all negatives is mostly wasted compute" | You *must* apply the correction; forgetting it inflates every price |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Impression logs, with position** | Shown ads, positions, clicks | **Position bias**; only shown ads have labels |
| **Invalid-traffic verdicts** | Which clicks were bots / fraud | Must be applied **before** training |
| **Ad, campaign, advertiser metadata** | Vertical, history, bids | Hierarchy for cold-start back-off |
| **Creatives** | Ad text, images, landing pages | The only signal a new ad has |
| **User and query context** | Query, device, geo, time, consented history | Consent; sensitive categories excluded |
| **Counter features** | Rolling CTRs per ad / user / query | ⚠ **Label leakage** unless point-in-time |
| **Conversions** | What happened after the click | Delayed (G22); not this model's label |

### 5.2 Choosing the training set (e) `F1c`

```
1. ELIGIBILITY   drop INVALID TRAFFIC first: bot clicks are wrong labels
                 AND wrong charges
2. POINT-IN-TIME every counter feature computed from data strictly BEFORE
                 the impression (a feature-store guarantee, not a modelling one)
3. DOWNSAMPLE    all positives + a fraction w of negatives (base CTR < 1%)
                 → REMEMBER the correction p = p′/(p′ + (1−p′)/w)
4. DEBIAS        position as an input to the shallow tower only · keep an
                 EXPLORATION slice (new-ad traffic) as an unbiased eval set
5. SPLIT BY TIME train on the past window, validate on the next hour/day;
                 check every calibration slice has enough rows
```

### 5.3 The model and the objective (f) `A2b` `F1a`

- **Model:** hybrid **wide** (memorisation: exact feature crosses) + **deep** (generalisation: hashed embeddings) — the shape exists because ads need *both* "this exact user-ad pair clicked before" and "users like this like ads like that".
- **Loss:** **log loss** on the downsampled data → the **downsampling correction** → an **isotonic calibration layer per slice**.
- **Position bias:** a click at position 1 ≠ a click at position 5. Train on raw clicks and the model learns **placement, not relevance** → a shallow position tower, dropped at serving.
- **Presentation bias:** you only observe clicks on ads you showed → exploration traffic, usually funded as new-ad cold-start.

> **⚠ Negative downsampling correction** — the single most-forgotten mechanical detail. Skip it → systematically inflated pCTR → **overcharging advertisers**, which is commercial and legal, not just inaccurate.

### 5.4 Fine-tune, don't retrain (f) `F2c`

- **Warm start:** each hourly run continues from the previous model on the newest window (continual training), so it keeps up with new ads and users without a full retrain.
- **Pretrained encoders:** query and creative text embeddings from a pretrained language model, adapted with **LoRA** on click data, give a new ad and a rare query a sensible representation from day one.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · LOGS        impressions + positions + clicks → IVT filtering
STAGE 2 · FEATURES    point-in-time counters from the feature store
STAGE 3 · TRAIN       downsample → warm-started hybrid model → correction
STAGE 4 · CALIBRATE   isotonic per slice on a held-out recent window
STAGE 5 · GATE        automated (FU 8): calibration per slice, log loss vs incumbent,
                      golden-set prediction diff, feature coverage → shadow → serve
CADENCE: hourly, with automatic rollback keyed on LIVE calibration
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
LATENCY BUDGET (≤10 ms p99):
  feature fetch (batched, cached)   ~5–6 ms  ← DOMINATES
  embedding lookup                  ~1 ms
  model forward                     ~1–2 ms
  correction + calibration          <1 ms

OPTIMISATIONS THAT ACTUALLY MATTER:
  · aggressive caching of ad-side features (they change slowly)
  · batch all 1000 candidates into one forward pass
  · quantized embedding tables to fit in memory
  · ⚠ optimise the FETCH before optimising the MODEL
```

- **Quantization** `F3a`: int8 / fp16 embedding tables: lower precision means less memory per row and faster lookups, with the calibration slices re-checked after quantizing.
- **Scale** `F3b`: hashed tables sharded across the fleet; one batched pass per request; ad-side features cached.
- **The allocation objective** `A3b`: rank by `pCTR × bid` (plus quality), with an **explicit exploration budget** for new ads, owned as a marketplace decision rather than buried in the model.
- **Reliability** `I1b`: calibrated priors on fetch timeouts; automatic rollback on live calibration drift.

**What breaks:** feature-store tail latency (p99 of 1000 parallel fetches is brutal — one slow shard sets your p99) · embedding table version skew · a stale counter feature silently degrading a whole vertical.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE GATES   log loss + CALIBRATION PLOT PER SLICE (blocking) · AUC as a check
                golden-set prediction diff · feature-coverage checks
SHADOW          score live traffic without pricing on it
A/B             revenue AND advertiser ROI AND user metrics — not revenue alone
LONG-TERM       holdouts for ad load / ad blindness, read over months
AUTO-ROLLBACK   keyed on LIVE calibration, because revenue moves too slowly and
                too noisily to catch a pricing bug
⚠ SHOWN-ONLY    only shown ads have clicks → exploration traffic + IPS for offline
  LABELS        evaluation of a new model
```

**Success** = log loss down, every calibration slice within 2%, revenue up **with** advertiser ROI holding or rising (*"a revenue win taken from advertiser ROI is borrowed, not earned"*). `I2b`

**Panels:** **calibration drift per slice** (the headline) · predicted vs actual CTR by decile · counter-feature freshness · feature-fetch p99 · new-ad coverage. **The alarm:** calibration drifting on *one slice* while the global number looks fine.

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | Clicks reflect placement | Shallow position tower, dropped at serving |
| **Selection** | Only shown ads get labels | Exploration (new-ad traffic); IPS |
| **Rich-get-richer** | Shown ads build counters and get shown more | Content features, hierarchical back-off, funded exploration |
| **Slice under-representation** | New surfaces and small advertisers are poorly calibrated | Per-slice calibration with minimum data; conservative priors |
| **⚠ Delivery skew** | Housing, jobs and credit ads can reach groups unevenly even with neutral targeting | Audit delivery parity for these categories (§8) |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p99 ≤ 10 ms for ~1,000 candidates; ~10⁸ predictions/s; the fetch dominates; cost per million predictions | Fetch p99 · $/M predictions |
| **Reliability** `I1b` | Hourly retrains behind **automated gates + automatic rollback keyed on live calibration** (FU 8); calibrated priors on fetch timeout | Rollbacks · calibration alarms |
| **Usability** `I1c` | Advertisers: predictable prices and a **cold-start SLA** for new ads. Users: relevant ads, frequency caps | New-ad time-to-traffic · ad-blindness trend |
| **Scalability** `I1d` | Hashing trick; sharded embedding tables; one batched forward pass per request; ad-side feature caching | Table size · cache hit rate |
| **Privacy** `I1e` | Consent-gated personalisation; **no personalised targeting on sensitive categories** (health, religion, sexuality, financial hardship); cohort-level signals where user-level aren't allowed; retention limits | Policy audit · consent coverage |
| **Safety** `I1f` | Ads pass policy review upstream (G27); **brand safety**: no ads next to harmful content; scams don't get to buy reach | Policy-violating impressions |
| **Security** `I1h` | ⚠ **Invalid traffic / click fraud** corrupts labels *and* charges advertisers → filtered **before training and before billing**; advertisers gaming features | IVT rate filtered · refunds |
| **Ethics / bias** `I1i` | ⚠ **Delivery skew**: housing, employment and credit ads can reach groups unevenly even with neutral targeting (the 2022 US DOJ–Meta settlement) → audit delivery parity; funded exploration so new advertisers aren't locked out | Delivery parity · new-advertiser share |
| **Explainability** `I1j` | Advertiser-facing **Quality Score** components; per-slice calibration plots for disputes (FU 7); feature attributions for debugging | Dispute resolution time |

> ⭐ *"The marketplace has two sides, and production-readiness has to cover both: users and advertisers each need the system to be explainable, private and fair."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multimodal creatives:** embed image and video ads so a **brand-new ad has a representation** before it has clicks. Cost: an encoder per creative at upload. Risk: creative similarity predicts CTR weakly compared with history, so it's a prior that backs off as impressions accumulate.
- **pCVR and autobidding:** with automated bidding, `bid = pCTR × pCVR × target`, so pCTR and pCVR errors **multiply** inside the bid. Calibration has to hold for the product, not just for each model.

**When the model has nothing to go on (a new ad, a new surface), the fallback pipeline** `A5b` `F2a`:

```
no history for this ad / slice
  → HIERARCHICAL BACK-OFF: campaign → advertiser → vertical CTR, shrunk to a prior
  → + CREATIVE EMBEDDING from the pretrained encoder (§5.4)
  → + an EXPLORATION allocation so it earns impressions regardless of pCTR
  → CONSERVATIVE calibration for the new slice; fold it into the main
    calibration only once it has enough data (FU 10)
```

**Top 3 failure modes:**
1. **Silent mis-pricing** — calibration drifts, AUC unchanged, revenue looks fine short-term, advertisers systematically overcharged.
2. **Counter-feature leakage** — offline metrics look spectacular, online performance collapses. The classic.
3. **Feedback loop** — ads that get shown accumulate counter features and get shown more; new ads never escape. *Rich-get-richer on the ad side.*

**Q1 ships:** calibration measurement **per slice** + the downsampling correction + the position tower.
> *"Calibration infrastructure first, because it's the thing that tells you whether anything else is working — and because mis-pricing is the failure that doesn't announce itself."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Your AUC went from 0.80 to 0.82. Ship it?"** `F1b` `I2a`
> Not on that alone — I'd want the calibration plot first. AUC measures ranking and is invariant to monotone transformations, so a model can improve AUC while becoming systematically mis-calibrated, and in an auction that means mis-pricing every impression. The concrete failure: pCTR running 20% high overcharges every advertiser, and AUC is *structurally incapable* of showing it. So my gate is log loss plus **calibration by slice**, with AUC as a discriminative sanity check. And "by slice" matters — a model calibrated on average while running 2× high on mobile and 2× low on desktop is broken in a way the global number hides.

**2. "Why downsample negatives, and what's the catch?"** `F1a` `F1c`
> Base CTR is well under 1%, so most training compute goes into negatives that carry little signal — downsampling to a fraction `w` makes training tractable and doesn't hurt ranking. The catch is that the model now outputs an **inflated probability** `p'` on a re-weighted distribution, and you must correct back: `p = p' / (p' + (1−p')/w)`. Skip that and every pCTR is systematically high, which in an auction means over-charging advertisers — a commercial and legal problem, not an accuracy one. It's the single most-forgotten mechanical detail in this system, and it's the kind of thing that distinguishes people who've shipped this from people who've read about it.

**3. "How do you handle position bias?"** `F1c`
> A click at position 1 and a click at position 5 mean different things, so training on raw clicks teaches the model about **placement rather than relevance**. I'd add a **shallow position tower** trained jointly and dropped at serving, so predictions are made as if every ad were at position 1. Shallow deliberately — you want it to absorb position effects without letting position leak into the content representation. The residual issue is **presentation bias**: you only observe clicks on ads you actually showed, which position correction doesn't fix. That needs exploration budget, and in ads it's usually funded as new-ad cold-start traffic rather than called exploration.

**4. "A brand-new ad has no history. What's your prediction?"** `F2a` `I1i`
> This is where counter features hurt you — "this ad's CTR so far" is the most predictive feature available and a new ad has none, so a naive model predicts near-zero, the ad never gets shown, and it never accumulates history. **Rich-get-richer.** Three things: **content-based features** (advertiser, vertical, creative attributes, landing-page signals) so a new ad has a representation before it has engagement; **hierarchical fallbacks** — back off to campaign-level, then advertiser-level, then vertical-level CTR with shrinkage toward the prior; and an **explicit exploration allocation** so new ads get impressions regardless of predicted CTR. The third is a business decision with a measurable cost, and I'd want it owned explicitly, since it's the mechanism that keeps the marketplace open to new advertisers.

**5. "Offline metrics were great, online tanked. What happened?"** `F2a` `F1c`
> My first suspicion is **counter-feature leakage**, and it's the most common version of this failure. Features like "this ad's CTR in the last hour" are computed over a window that, in training, may include the impression being predicted — so the model effectively sees the label. Offline it looks spectacular; online that information doesn't exist at prediction time and performance collapses. The fix is strict **point-in-time correctness**: every feature computed only from data available strictly before the event, which is a feature-store guarantee, not a modelling one. Other candidates in order: **train/serve skew** in feature computation, **distribution shift** between the training window and live traffic, and a **missing downsampling correction** that offline metrics on the re-weighted distribution wouldn't reveal.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Revenue is up 3%. Is that good?"** `I2b` `A3a`
> Not necessarily, and I'd want to know *how* before celebrating. Revenue can rise because the model got better at matching, or because pCTR inflated and everyone is being charged more per click — those look identical on a revenue dashboard and are opposite outcomes. So I'd check **calibration** first, then **advertiser-side ROI**, because a revenue gain that comes out of advertiser return is borrowed rather than earned: advertisers measure their own ROI, and they reduce budgets when it degrades, so the gain reverses on a lag long enough that nobody attributes it correctly. That's the structural argument I'd make — **the marketplace has two sides and only one of them is on our dashboard.** The healthy version of a revenue win shows up as advertisers *increasing* budgets, and that's the confirmation I'd wait for.

**7. "Advertisers complain your pCTR is too high. How do you investigate?"** `I1j` `F2a`
> They may well be right, and it's a claim with a definitive answer, which is unusual and worth using. I'd take their traffic slice and plot **predicted vs actual CTR by decile** — if predictions sit systematically above the diagonal, the model is inflated for them and that's the end of the investigation. Then the diagnosis in order: **is the downsampling correction applied**; **is their slice under-represented** in training so the calibration layer never fit it; **are counter features stale** for their campaigns; and **is there a slice interaction** — calibrated globally, inflated on the exact combination of device and vertical they buy. The organisational point I'd add: advertisers have their own measurement and will detect systematic over-charging eventually, so **per-slice calibration is a trust asset**, not just an accuracy metric, and it's cheaper to find it ourselves than to have it found for us.

**8. "You retrain hourly. How do you know a retrain didn't break something?"** `I1b` `I2a`
> Automated gates, because at hourly cadence no human is in the loop and "we'll notice" is not a plan. Every candidate model faces: **calibration per slice against a held-out window** — blocking, not advisory; **log loss versus the incumbent** on the same window; a **golden-set prediction diff**, where a fixed set of impressions should produce nearly identical scores and a large shift means something structural changed rather than improved; and **feature-coverage checks**, since the most common silent retrain failure is an upstream feature going null and the model quietly learning to ignore it. Then **shadow scoring** before any traffic, and a **fast automatic rollback** keyed on live calibration rather than on revenue, because revenue moves too slowly and too noisily to catch a pricing bug. The principle: at this cadence the *pipeline* is the thing under test, not the model.

**9. "Should the ranker and the pricer be the same model?"** `A5a` `A3b`
> They're the same score today — `pCTR × bid` drives both allocation and charge — and that coupling is exactly why calibration is non-negotiable here rather than merely nice. If they were decoupled you could tolerate a mis-calibrated ranker, because ranking only needs monotonicity. Since they're coupled, **every calibration error is a pricing error**. I wouldn't split them, because a separate pricing model introduces a consistency problem that's worse than the one it solves: two models disagreeing means you allocate on one belief and charge on another, which is very hard to explain to an advertiser. What I *would* separate is the **calibration layer** — keep the main model focused on discrimination and let a cheap, frequently-refit isotonic layer own absolute correctness per slice. That gives you most of the benefit of decoupling with none of the inconsistency.

**10. "Traffic doubled overnight from a new surface. What do you expect to break?"** `F2a` `I1b`
> Calibration, first and most importantly — the new surface is a slice the model has never seen, so its predictions there are extrapolations, and my global calibration number will stay healthy while that slice is badly wrong. That's the failure mode the per-slice panel exists for. Second, **feature coverage**: a new surface usually means missing context features, and models degrade *silently* on missing features rather than erroring. Third, **the feature-fetch p99** — doubled traffic against a cache tuned for the old distribution means a cold cache and tail latency, and since the fetch dominates the budget, that's where the timeouts appear. My response would be to **hold the new surface out of the auction at full weight initially** — serve it with a conservative prior and an explicit exploration allocation until there's enough data to calibrate it, then fold it in. Shipping a new slice straight into live pricing is how you discover mis-calibration through advertiser complaints.

---

## ⭐ THE L6 LINE

> **"The output is a price, not a ranking — so AUC is structurally blind to the failure that matters. Calibration by slice is the gate, the downsampling correction is mandatory, and counter features leak the label unless they're point-in-time correct."**

## 🧭 HOOKS — *steer by what you mention*

- ⭐ *"What makes it different is **who consumes it**"* → invites the requirements-from-the-consumer branch (**good at L6**)
- *"AUC is invariant to monotone transformations"* → invites the metrics/statistics branch
- *"The feature fetch is the bottleneck, not the model"* → invites the systems/serving branch
- *"Counter features must be point-in-time correct"* → invites the **feature store** branch (G32 territory)
- *"The marketplace has two sides and one is on our dashboard"* → invites the org/leadership branch (**good at L6**)
- *"New ads never escape without funded exploration"* → invites the cold-start / marketplace-health branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the latency budget"* — the 10 ms table regenerates the architecture.
- **Running out of time?** → *"I have five minutes — I'd rather cover the failure modes than finish the feature engineering. Sound right?"*

**Name and defer:**
- "**Conversion prediction** is a separate model with delayed labels — that's its own design."
- "**Budget pacing** sits downstream and is a controller, not a predictor."
- "**Multi-slot / whole-page optimisation** has externality effects between ads on the same page."
- "**Privacy-preserving signals** — cohort-based rather than user-level — change the feature space materially."
- "**Auction mechanics** (reserve prices, second-price) interact with calibration; I'd keep them separate here."
- "**Embedding table sharding** across the serving fleet is a real infra problem at this size."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a lookup-dominated scorer feeding the auction | §0, §3 | *"This is a distributed lookup problem with a model attached."* |
| **A1b** | Latency and capacity budgets | §1, §3, §6 | *"10⁵ requests × 1,000 candidates = 10⁸ predictions/s, ~10 µs each."* |
| **A1c** | Degrade without lowering quality for everyone | §4 | *"A light first pass 1,000 → 100; calibrated priors on a timeout, never zero."* |
| **A2a** | The full stack | §4 diagram | Fetch → hash → hybrid model → correction → calibration → auction |
| **A2b** | Model components | §5.3 | *"Wide memorises exact pairs, deep generalises over hashed embeddings."* |
| **A2c** | Data sources | §5.1 | Impressions with position, IVT verdicts, metadata, creatives, counters |
| **A2d** | Staged pipeline | §5.5 | Logs → point-in-time features → train → calibrate → automated gate |
| **A3a** | Top-down, unprompted | §0 | *"The output is a price, not a ranking"* + the scoping line + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"pCTR × bid, with an explicit, owned exploration budget for new ads."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Auction rule · label · new-ad SLA · surface |
| **A5a** | Alternatives compared | §4, FU 9 | LR vs GBDT vs wide & deep; in-model vs separate calibration |
| **A5b** | A full alternative pipeline on demand | §9 | Hierarchical back-off + creative embedding + exploration |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Log loss, the downsampling correction, then isotonic per slice."* |
| **F1b** | Metrics and their trade-offs | §2 | *"AUC is invariant to monotone transforms, so it can't see mis-pricing."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Position, selection, rich-get-richer, slice coverage, delivery skew |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 5 | Counter-feature leakage; the cold-start path |
| **F2b** | Latency too tight → tiering | §4 | The first-pass model, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Hourly warm start; LoRA-adapted query and creative encoders |
| **F3a** | Quantization with a quality trade | §6 | *"Quantized tables, with the calibration slices re-checked after."* |
| **F3b** | Scale via precomputation and light models | §3, §6 | Hashing, sharded tables, cached ad features, one batched pass |
| **F3c** | Extension, with cost and maturity risk | §9 | Multimodal creatives; pCTR × pCVR inside autobidding |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Fetch p99; $ per million predictions |
| **I1b** | Reliability | §6, §8, FU 8 | Automated gates + rollback on live calibration |
| **I1c** | Usability | §2, §8 | Predictable prices; a cold-start SLA; relevant ads |
| **I1d** | Scalability | §3, §8 | Hashing, sharding, caching |
| **I1e** | Privacy | §8 | Consent; no sensitive-category targeting |
| **I1f** | Safety | §8 | Brand safety; policy review upstream |
| **I1h** | Security | §8 | **Invalid traffic** filtered before training and billing |
| **I1i** | Ethics / bias | §7, §8 | Delivery-parity audits; exploration for new advertisers |
| **I1j** | Explainability | §2, §8, FU 7 | Quality Score; calibration plots for disputes |
| **I2a** | Offline → online, correcting for shown-only labels | §7 | Calibration gates → shadow → A/B → long-term holdouts; exploration + IPS |
| **I2b** | The success definition | §7, FU 6 | *"Revenue up with advertiser ROI holding, every slice within 2%."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
