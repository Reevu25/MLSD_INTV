# G22 · Delayed Conversion Prediction — **INTERVIEW SHEET**

> **Tier 2 · #16** — the Ads-team deep dive, and the cleanest "censored labels" question in the pack.
> Full study file: [`../G22_delayed_conversion.md`](../G22_delayed_conversion.md) · [diagrams](../G22_delayed_conversion_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the system · (d) datasets · (e) building the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Attribution model, delay by vertical, bidding or reporting? (§1) |
| **Explainability** is easy points | Delay-adjusted advertiser reports, the **attribution version** on every label (§8) |
| **Security** beyond the gateway | **Fake conversion pings** manipulating bidding (§8) |
| **Troubleshoot before being asked** | Raise the **mis-specified delay model** and **seasonal shifts** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A negative label is never final** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · calibration gates · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | pCVR inside the bid · **calibration** — the bias is multiplicative, AUC can't see it | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | The delay distribution · freshness vs completeness · the label join | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Wait vs window vs **model the delay** · the joint model + label pipeline | A2a, A5a, A1c, F2b |
| **22–32** | **DATA + THE LIKELIHOOD** | **(d) (e) (f)** | Attribution defines the label · ⭐ **the censored objective — say the formula** | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Calibrated pCVR × target · per-vertical λ · rollback on the recency plot | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Matured cohorts · calibration by elapsed time · coverage | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Subscriptions, offline conversions · the new-vertical path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The structural fact that drives this design: a negative label is never final.**
>
> **In ordinary supervised learning a zero is a zero. Here, 'has not converted yet' is a mixture of two completely different things — 'will never convert' and 'will convert tomorrow' — and at training time I cannot tell them apart. So the recent portion of my training set is contaminated with false negatives that will later resolve into positives, and a model trained naively on it systematically underestimates conversion rate.**
>
> **And it's a calibration bias specifically — which matters because this prediction is multiplied by a bid. A uniformly low model doesn't lose ranking quality, so AUC won't move; it under-bids across the platform and under-delivers advertisers' campaigns."**

**Then kill both naive answers explicitly:**
- ❌ *"Wait thirty days for clean labels"* → a model trained on data from a month ago, in an ecosystem where creatives change hourly. **An L4 answer.**
- ❌ *"Use a one-day window"* → a fresh model with badly biased labels, and **the bias is worst precisely in the high-value verticals** where conversion takes weeks.
> ✅ **"So: model the delay rather than choosing a window."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, how the prediction is used and measured, the delay distribution, the architecture, the data and the likelihood, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What's the attribution window and model?** *(⚠ **Attribution *defines* the label** — last-click vs data-driven changes what "converted" means.)*
2. **What does the delay distribution look like by vertical?** *(Ecommerce: hours. B2B/auto/travel: weeks. One window can't serve both.)*
3. **Is the prediction used for bidding, or for reporting?** *(Bidding ⇒ calibration is everything; reporting tolerates lag.)*
4. **Which conversions can we observe at all?** *(Cross-device and privacy-restricted conversions are invisible, not late.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Use | **pCVR inside automated bidding**: bid = pCTR × pCVR × target CPA |
| Conversions | Reported by advertisers' tags and conversion APIs, delays from minutes to ~60 days |
| Attribution | Data-driven by default; its **version** stamped on every label |
| Verticals | Retail (hours) through travel, auto and B2B (weeks) |
| Freshness | Daily retraining (or better) |
| Privacy | Some segments only measurable as modelled or aggregated conversions |

**FR:** predict `P(convert)` per (user, ad, context) · **calibrated in absolute terms** · fresh model despite incomplete labels · per-vertical delay handling · recalibration as late conversions land.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Calibration error** | **< 5% per vertical** ← the gate |
| Model freshness | Retrain **daily or better** |
| Scoring latency | Inherits G21's budget (~10 ms) |
| **Bias vs elapsed time** | Flat — no systematic under-prediction on recent data |
| Availability | A last-good model and per-vertical priors if training fails |

**Scope / NON-GOALS:**
- ❌ **Waiting for clean labels.** Position, with the reason.
- ❌ **A fixed conversion window** as the primary mechanism.
- ❌ The attribution model itself (I consume it, and I'd flag that it's upstream and decisive) · click prediction (G21) · pacing (G24).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the product is (b):**

```
ADVERTISER sets a target: "$40 per conversion"
  → for every auction: bid = pCTR × pCVR × $40
     ↑ pCVR running 20% low ⇒ every bid 20% low ⇒ the campaign under-delivers

ADVERTISER REPORT (what they see)
  conversions by day … the last few days marked "still reporting"
  ↑ recent days fill in as late conversions arrive; an honest report says so,
    or advertisers misread the lag as a performance drop
```

**The metrics:**

```
⚠ THE BIAS IS MULTIPLICATIVE, SO AUC IS BLIND TO IT — AGAIN.

  every prediction scaled by the same factor k < 1
    → ranking UNCHANGED → AUC UNCHANGED
    → but pCVR × bid is systematically LOW
    → UNDER-BIDDING platform-wide, advertisers UNDER-DELIVERED

HEADLINE:  CALIBRATION, per vertical AND per elapsed-time bucket
SECONDARY: log loss · AUC as a discriminative sanity check

⚠ THE DIAGNOSTIC PLOT THAT FINDS THIS:
   predicted vs actual conversion rate, BUCKETED BY HOW LONG AGO
   THE CLICK HAPPENED.
   A healthy model is flat. A naive model slopes DOWN for recent
   clicks — which is the delayed-feedback bias, made visible.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Advertiser delivery against target** (conversions at the target CPA) and ROI | Is the bid right? |
| Offline | **Calibration per vertical and per elapsed-time bucket** · log loss · AUC as a check | The gate |
| Ground truth | Performance on a **fully matured cohort** | Unbiased, but weeks old |
| **Guardrails** | Coverage (share of conversions observable) · delay-model fit · scoring p99 | Silent failures |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **the freshness/completeness trade** `A1b` `I1a` `I1d`

```
THE DELAY DISTRIBUTION IS THE WHOLE PROBLEM:

  ecommerce impulse buy   → minutes to hours    (labels ~complete in 1 day)
  considered purchase     → days
  B2B / auto / travel     → WEEKS               (labels ~complete in 30–60 d)

  ⚠ ONE GLOBAL WINDOW IS WRONG FOR BOTH ENDS:
     30-day window → model is 30 days stale
     1-day window  → ~complete for ecommerce, CATASTROPHIC for B2B,
                     and the bias is LARGEST in the HIGHEST-VALUE verticals

TRAINING DATA AT ANY MOMENT:
  old clicks   → labels essentially complete, distribution stale
  recent clicks→ labels badly incomplete, distribution current
  ⚠ you need BOTH, weighted by how much of the delay has elapsed.

THE LABEL JOIN: every conversion must be matched to its click across a
  ~60-day window → a streaming join with 60 days of click state, partitioned
  by click date. Scoring itself inherits G21's ~10 ms / ~10⁸ predictions/s.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| Wait 30 days for clean labels | Unbiased labels | **Stale**: the world they describe is gone |
| A 1-day window | Fresh | **Biased**, worst in the high-value long-delay verticals |
| Importance weighting by elapsed time | Simple, no delay model | Needs the delay distribution anyway to set the weights |
| **Joint model of p and the delay (censored likelihood)** ✅ | Fresh **and** correctly weighted | A delay assumption that can be wrong (FU 6) |

### The system — *draw this in 90 seconds* `A2a`

```
 CLICK → features
       │
┌─ 1. JOINT MODEL ───────────────────────────────────────────────
│ head 1: p = P(this click will EVER convert)
│ head 2: λ = delay-distribution parameters, conditioned on features
│         (B2B gets a different λ from retail)
│ trained with the CENSORED LIKELIHOOD (§5.3)
│ OUTPUT: p
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CALIBRATION LAYER (per vertical) ──────────────────────────
│ fitted on matured cohorts; checked per elapsed-time bucket
│ OUTPUT: calibrated pCVR → × pCTR × target → AUCTION
└────────────────────────────────────────────────────────────────

══ THE LABEL PIPELINE (continuous) ══════════════════════════════════════════
┌─ 3. CONVERSION JOIN ───────────────────────────────────────────
│ conversions stream in for 30–60 days → matched to their clicks under the
│ current ATTRIBUTION VERSION (stamped on every label)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. TRAINING ROWS — ⚠ MUTABLE ─────────────────────────────────
│ each row carries its ELAPSED TIME; rows are RE-LABELLED as conversions
│ arrive → snapshots "labels as of time T" for reproducibility
│ OUTPUT: daily training sets with elapsed time + importance weights
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `F2a` `F2b` `A1c`

- **A mis-specified delay model:** *"Real delays are multi-modal (an impulse spike, a consideration bump, a long B2B tail), so an exponential is wrong. I'd watch calibration **by delay bucket**, and fall back to a discrete-time hazard model that assumes no shape (FU 6)."*
- **Seasonal shifts:** *"Black Friday compresses delays. λ is refit more often than p, conditioned on season and campaign type, with the delay fit as its own panel (FU 7)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Model the delay, don't pick a window** | "A window forces a choice between stale and biased; modelling `f(d;λ)` lets recent data contribute *correctly weighted*" | A parametric delay assumption that can be wrong |
| **2** | **Per-vertical λ** | "B2B and impulse retail have delay distributions orders of magnitude apart — one λ is wrong for both" | More parameters, sparser per-vertical data |
| **3** | **Fresh model + correction over clean-but-stale** | "Creatives and campaigns change hourly; a 30-day-old model is wrong about the current ecosystem" | You're always correcting an estimate rather than observing truth |

---

## 5 · DATA + THE LIKELIHOOD *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The data (d): attribution defines the label `A2c`

```
"Did this click convert?" IS NOT A FACT ABOUT THE WORLD.
It's the output of an ATTRIBUTION MODEL:
   last-click · first-click · position-based · data-driven

→ change the attribution model and EVERY LABEL CHANGES,
  so every historical metric becomes incomparable — the same
  structural issue as a rubric change or a policy redefinition.
→ therefore: ATTRIBUTION VERSION is stamped on every label,
  and a change to it is a METRIC CHANGE, announced as such.
```

| Source | What it gives | Watch out for |
|---|---|---|
| **Click logs** | Features, timestamp, elapsed time | Invalid traffic |
| **Conversion events** | Tags, conversion APIs, offline uploads | Delay; fakes; dedup |
| **Attribution output** | Which click gets the credit | Its version changes every label |
| **Modelled / aggregated conversions** | Privacy-restricted segments | Noisy aggregates, not per-click labels |

### 5.2 Building the training set (e) `F1c`

```
1. ELAPSED TIME     every row carries time since click — a feature AND a weight
2. LABEL AS-OF      datasets versioned by (click window, labels as of date T),
                    so a training run is reproducible
3. COVERAGE         cross-device and privacy-blocked conversions are INVISIBLE,
                    not late: correct by estimated coverage per segment, don't
                    treat them as censored (FU 5)
4. CLEAN            invalid clicks removed; duplicate conversion pings removed
5. EVAL SEPARATELY  hold out fully MATURED cohorts for unbiased evaluation
```

### 5.3 The model and the objective (f): the censored likelihood `A2b` `F1a`

```
MODEL TWO THINGS JOINTLY:
   p        = P(this click will EVER convert)
   f(d;λ)   = the DELAY distribution (e.g. exponential / Weibull),
              conditioned on features — B2B has a different λ than retail

THEN THE CENSORED LIKELIHOOD — write this down:

   OBSERVED CONVERSION at delay d :   p · f(d; λ)
   NOT YET CONVERTED, elapsed e   :   (1 − p)  +  p · S(e; λ)
                                       ↑           ↑
                            never converts    WILL convert, just
                                              hasn't yet  (S = survival)

⚠ THAT SECOND TERM IS THE WHOLE IDEA. The naive model implicitly sets
  it to (1 − p), asserting that everything not-yet-converted never will.
```

**Choosing the delay family:** exponential is the textbook default and is wrong for real data, which is **multi-modal**. A Weibull or a mixture fits better; a **discrete-time hazard** (`P(convert on day k | not yet)` per bucket) assumes no shape and is the robust production default.

### 5.4 Fine-tune rather than retrain from scratch (f) `F2c`

Each daily run **warm-starts** from yesterday's model. The p-head can share the embedding tables and feature backbone with the pCTR model (G21), adapted with a small head per vertical, so sparse verticals borrow strength instead of being trained alone.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · JOIN       conversions → clicks, under the stamped attribution version
STAGE 2 · SNAPSHOT   training rows with elapsed time, labels as of today
STAGE 3 · TRAIN      joint model (p + λ), warm-started, censored likelihood
STAGE 4 · CALIBRATE  per vertical, on matured cohorts
STAGE 5 · GATE       flat calibration across elapsed-time buckets → shadow → serve
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

- **The scoring path** inherits G21: ~10 ms, a batched forward pass, calibrated pCVR multiplied into the bid.
- **Efficiency and scale** `F3a` `F3b`: the shared backbone with pCTR, quantized embedding tables, and **λ refit separately** on a seasonal cadence, since the delay shape moves faster than the conversion relationships.
- **The bidding objective** `A3b`: `bid = pCTR × pCVR × target`, so **calibration of the product** matters, not just of each model; per-vertical calibration is the lever.
- **Reliability** `I1b`: **automatic rollback** when the elapsed-time calibration plot slopes on live traffic; a last-good model; per-vertical priors when a vertical's data is thin.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **The gate** | **Calibration per vertical**, and **per elapsed-time bucket** |
| **The diagnostic plot** | Predicted vs actual, bucketed by click recency — a downward slope on recent clicks *is* the bias |
| **Cohort evaluation** | Evaluate a **fully-matured cohort** (clicks old enough that labels are complete) to get unbiased truth |
| **Delay-distribution fit** | Is `f(d;λ)` still describing reality? Seasonal shifts (holidays) move it materially |
| **Coverage** | Share of conversions observed at all — falling coverage looks like falling conversion rate |
| **Online** | Advertiser delivery and ROI, not platform revenue alone |

**Rollout** `I2a`: offline gates (calibration by vertical and by elapsed time) → shadow scoring → A/B on **advertiser delivery and ROI**, read only after enough time for that cohort's conversions to arrive → ramp. ⚠ Evaluate with **labels as of the prediction time**, or a model looks better than it was.

**Success** = calibration flat across elapsed-time buckets and within 5% per vertical, advertisers delivering against their targets, and budgets holding or growing. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Delayed feedback** | Recent negatives are future positives | The censored likelihood; elapsed-time weights |
| **Vertical** | Long-delay verticals are hit hardest | Per-vertical λ and calibration |
| **Coverage** | Invisible conversions look like low conversion | Coverage correction per segment |
| **Attribution drift** | A model change silently relabels history | The version stamp; announced metric changes |
| **Evaluation leakage** | Today's labels are more complete than training-time labels | Replay labels as of prediction time |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Scoring ≤ 10 ms (shared with pCTR); daily retrains; $ per million predictions | p99 · training time |
| **Reliability** `I1b` | Auto-rollback on the recency-calibration slope; last-good model; idempotent label backfills; reproducible snapshots | Rollbacks · snapshot reproducibility |
| **Usability** `I1c` | Advertiser reports mark recent days **"still reporting"** and show conversion lag, so the lag isn't misread as a drop; predictable delivery against target | Support tickets about "missing" conversions |
| **Scalability** `I1d` | A streaming 60-day click↔conversion join, partitioned by click date; mutable labels stored as versioned snapshots | Join lag · storage growth |
| **Privacy** `I1e` | Consent respected; privacy-restricted segments measured as **modelled or aggregated** conversions, never re-identified; retention limits | Consent coverage · audits |
| **Safety** `I1f` | Sensitive conversion types (health) not used for personalised targeting; conversion actions must be policy-compliant | Policy audits |
| **Security** `I1h` | ⚠ **Fake conversion pings** (bots or bad actors firing tags to steer bidding) → dedup, validation, anomaly detection on conversion streams; invalid clicks filtered | Anomalous conversion spikes caught |
| **Ethics / bias** `I1i` | Small advertisers with sparse conversions shrunk toward vertical priors; coverage gaps by browser or device corrected, not mistaken for low quality | Calibration by advertiser size · coverage by segment |
| **Explainability** `I1j` | The **attribution version** on every label; per-vertical λ reports; the recency plot as the shared diagnostic; advertisers see why bids moved | % labels with a version stamp |

> ⭐ *"This model is a price input for other people's businesses, so production-readiness includes making the delay visible to them: an honest 'still reporting' beats a dashboard that looks like a crash."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Subscriptions:** the event recurs (convert, renew, churn), so the target becomes **lifetime value with a survival model on retention**; a free trial creates a hard spike at day 14 that belongs in the model explicitly (FU 10).
- **Offline conversions** (in-store, phone) arrive by upload, with their own latency and coverage profile.
- **Incrementality:** attribution says who gets credit; incrementality asks whether the conversion would have happened anyway. A different question, answered with experiments.

**When a vertical has no reliable delay history, the fallback pipeline** `A5b` `F2a`:

```
a new vertical, a new conversion type, or a delay fit that just broke
  → a DISCRETE-TIME HAZARD per day-bucket (no parametric shape to be wrong)
  → borrow λ from SIMILAR verticals as a prior, shrink toward it
  → CONSERVATIVE calibration until a matured cohort exists
  → advertiser reports flag the estimate as provisional
```

**Top 3 failure modes:**
1. **Systematic under-bidding** — the naive bias, invisible to AUC, showing up as advertisers under-delivering and reducing budgets on a lag.
2. **Attribution change treated as a model regression** — labels moved, nobody stamped the version, and a quarter is spent debugging the model.
3. **Delay-distribution shift** — a holiday period changes purchase latency; `λ` is stale and calibration silently drifts.

**Q1 ships:** the **elapsed-time-bucketed calibration plot** + importance weighting by elapsed delay.
> *"The diagnostic first — it makes an invisible bias visible, and it's what tells you whether the delay model is earning its complexity. The joint censored-likelihood model is Q2."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just wait 30 days for clean labels?"** `A5a` `A3a`
> Because you'd have a model trained on data from a month ago in an ecosystem where campaigns, creatives, bids and competitors change hourly — the labels are clean and the *world* they describe is gone. That's the L4 answer, and the symmetric mistake is a one-day window, which gives a fresh model with badly biased labels, and the bias is **worst precisely in the high-value verticals** where conversion takes weeks. Both are forced choices between staleness and bias, and the resolution is to stop choosing: **model the delay**. If I jointly model `p` — will this ever convert — and `f(d;λ)` — the delay distribution — then recent, incompletely-labelled data can contribute *correctly weighted* rather than being either excluded or trusted naively. The window stops being a modelling decision and becomes a reporting convention.

**2. "Write down the likelihood."** `F1a`
> For an **observed conversion** at delay `d`, the contribution is `p · f(d; λ)` — it converted, and it took that long. For a click that has **not yet converted** with elapsed time `e`, the contribution is `(1 − p) + p · S(e; λ)` — either it never converts, *or* it will convert and simply hasn't yet, where `S` is the survival function. **That second term is the whole idea.** The naive model implicitly sets it to `(1 − p)`, asserting that everything not-yet-converted never will — which is exactly the false-negative contamination, and it's why the naive model systematically under-predicts. Fitting `λ` conditioned on features matters too, because a B2B lead and an impulse retail purchase have delay distributions orders of magnitude apart, and a single global `λ` is wrong for both ends of the distribution.

**3. "How would you detect this bias in production?"** `F1b` `I2a`
> With one plot: **predicted versus actual conversion rate, bucketed by how long ago the click happened.** A healthy model is flat across those buckets. A model with delayed-feedback bias **slopes downward for recent clicks** — it under-predicts exactly where labels are least complete — and that shape is unmistakable once you look for it. The reason this diagnostic matters so much is that the bias is **multiplicative and roughly uniform in rank**, so AUC doesn't move and every standard dashboard looks healthy while the platform under-bids. I'd also evaluate on a **fully-matured cohort** — clicks old enough that labels are essentially complete — to get an unbiased estimate of true performance, accepting that it tells you about a model version from weeks ago. Those two together (the recency plot for the live model, the matured cohort for ground truth) are the minimum instrumentation.

**4. "What does a change in attribution do to you?"** `A2c` `I1j`
> It changes **every label**, which makes it a metric change rather than a configuration change — and this is the thing I'd raise unprompted. "Did this click convert" isn't a fact about the world; it's the output of an attribution model, and switching from last-click to data-driven attribution reassigns conversions across touchpoints, so the same historical click now has a different label. Every historical metric becomes incomparable, and if nobody flagged it, the team spends a quarter debugging a "model regression" that is actually a label redefinition. So: **attribution version is stamped on every label**, a change to it invalidates comparisons explicitly, and it's announced as a metric change the way a rubric version bump would be. It also has a modelling consequence — different attribution models have different **delay characteristics**, so `λ` has to be refit rather than carried over.

**5. "Some conversions you'll never see at all. Does the model handle that?"** `F1c` `I1e`
> Not with the censoring machinery, and conflating the two is a real error. Cross-device conversions, conversions on a platform that blocks the signal, and privacy-restricted measurement are **structurally invisible** — they are not delayed, they will never arrive — so modelling them as censored means the survival term waits forever for something that isn't coming, and the model over-predicts to compensate. Those need a **coverage** treatment instead: estimate what fraction of true conversions are observable in each segment, and correct the calibration by that factor, ideally using a segment where you *can* observe ground truth (a panel, a conversion-API integration, an advertiser-supplied reconciliation). The practical warning sign is that **falling coverage looks exactly like falling conversion rate** on a dashboard, so a measurement change gets misread as a performance change — which is why coverage is a tracked panel rather than a constant.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your delay model assumes exponential. What if it's wrong?"** `F2a` `F1a`
> It is wrong, in a specific and known way: real conversion delays are **multi-modal**, not exponential. There's a spike within the first hour (impulse), a second mass over a few days (consideration), and a long tail over weeks (B2B, high-consideration goods) — an exponential fits none of that well and will systematically mis-weight the middle. So in practice I'd use something with more shape — a **Weibull**, or a **mixture** of two or three components, or a discrete-time hazard model that makes no parametric assumption at all and just learns `P(convert at day k | not yet converted)` per bucket. The discrete-hazard version is my default in production because it's robust and inspectable, at the cost of more parameters and needing enough data per bucket. The general principle I'd state: **the parametric assumption is a modelling convenience, and its failure shows up as calibration error concentrated in specific delay ranges** — so I'd monitor calibration *by delay bucket*, which is exactly the diagnostic that reveals a mis-specified `f(d;λ)` rather than a bad `p`.

**7. "Black Friday. What breaks?"** `F2a` `I1b`
> `λ` — the delay distribution shifts, and everything downstream of it goes wrong quietly. During a shopping peak, purchase latency **compresses** (people who would have deliberated for a week buy the same day) and the mix shifts toward impulse verticals. A model carrying last month's `λ` now believes conversions are still pending that have already happened and been counted, which distorts the correction in both directions depending on the vertical. Three responses: make **`λ` conditional on time-of-year and campaign type** rather than fitting it globally; **refit the delay model more frequently than the conversion model**, since `λ` moves on a seasonal cadence while `p`'s feature relationships are more stable; and monitor the **delay-distribution fit** as its own panel so a shift is detected rather than inferred from calibration drift weeks later. I'd also note the reverse case, which is worse: after the peak, delays **lengthen** again and a `λ` fitted on peak data under-corrects, so the damage persists past the event.

**8. "Advertisers say you're under-delivering their campaigns. Walk me through the investigation."** `I1j` `F2a`
> They're describing the exact symptom of this bias, so I'd take it as a strong prior rather than a complaint to triage. First the **recency-bucketed calibration plot** on their traffic — a downward slope on recent clicks confirms delayed-feedback bias immediately. If it's flat, the next candidates in order: **their vertical's delay distribution** may be much longer than the global `λ` we fitted, which is the single most common slice failure and is worst for exactly the high-value advertisers who complain; **coverage loss** in their conversion tracking, which looks like under-performance but is a measurement problem; and an **attribution change** on their account that reassigned credit. What makes this worth doing carefully is the feedback dynamic: under-bidding causes under-delivery, which causes the advertiser to **reduce budgets**, which reduces our revenue on a lag long enough that nobody attributes it to a calibration bug. So I'd frame the fix internally as revenue protection, not accuracy — that's the version that gets prioritised.

**9. "You're retraining daily on mutable labels. What breaks in the pipeline?"** `A2d` `I1b`
> The assumption that a training set is immutable, which most ML infrastructure quietly depends on. A row logged on Monday as "no conversion" becomes a positive on Thursday, so **the same input row has different labels in different builds** — which breaks reproducibility (you can't reconstruct last week's training set unless you snapshotted it), breaks naive caching (a cached feature-label pair is silently wrong), and breaks A/B comparisons between models trained on "the same" data. The fixes: **snapshot the label state with a timestamp** so any training run is reproducible as "labels as known at time T"; carry **elapsed time as a feature and an importance weight** rather than baking the correction into the label; and version datasets by (click window, label-as-of date) rather than by click window alone. I'd also flag the evaluation consequence, which bites people: a model evaluated today on last month's clicks sees *more complete* labels than it had at training time, which makes it look better than it was — so the eval has to replay **label state as of the prediction**, not as of now.

**10. "Would you use the same design if this were a subscription business rather than ecommerce?"** `F3c` `A5a`
> The censoring machinery transfers, but two things change materially. First, the **event is recurring rather than one-shot** — a subscription converts, renews, churns, and reactivates — so `p` = "will ever convert" is the wrong quantity; you want expected **lifetime value with a survival model on retention**, which is the same mathematics pointed at a different variable and is genuinely closer to a hazard-rate problem than a delayed-conversion one. Second, the **delay is much longer and the trial structure creates a hard discontinuity**: a 14-day free trial means almost no conversions before day 14 and a spike at day 14–15, which no smooth `f(d;λ)` fits — that's a structural feature of the product that belongs in the model explicitly rather than being learned as an odd bump. Third, the business consequence inverts: in ecommerce, under-predicting costs you delivery; in subscriptions, over-predicting LTV means **acquiring customers you lose money on**, and that error compounds because it's paid up front and recovered over months. So I'd keep the censored likelihood, replace the single conversion event with a survival model over the subscription lifecycle, and make the calibration gate asymmetric — over-prediction is the expensive direction.

---

## ⭐ THE L6 LINE

> **"A negative label is never final — 'not converted yet' mixes 'never will' with 'will tomorrow'. So model the delay rather than choosing a window, and the pending term `(1−p) + p·S(e;λ)` is the whole idea. The bias is multiplicative, so AUC is blind to it."**

## 🧭 HOOKS — *steer by what you mention*

- *"A negative label is never final"* → invites the survival-analysis / statistics branch
- *"The bias is multiplicative so AUC can't see it"* → invites the calibration branch (G21 crossover)
- *"Attribution defines the label"* → invites the measurement/versioning branch (**good at L6**)
- *"Some conversions are invisible, not delayed"* → invites the privacy/coverage branch
- *"The training set is mutable"* → invites the ML-infrastructure branch
- *"Under-bidding reduces advertiser budgets on a lag"* → invites the marketplace/org branch (**good at L6**)

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me write the likelihood again"* — the two terms regenerate the whole design.
- **Running out of time?** → *"I have five minutes — I'd rather cover attribution and the detection plot than finish the feature engineering. Sound right?"*

**Name and defer:**
- "**Click prediction** is the upstream sibling; same calibration argument, no censoring."
- "**Budget pacing** consumes this estimate and is a controller, not a predictor."
- "**Privacy-preserving measurement** — aggregated and noised conversion reporting — changes the label pipeline fundamentally."
- "**Incrementality** is a different question from attribution: would the conversion have happened anyway?"
- "**Multi-touch attribution modelling** is its own system and it's upstream of everything here."
- "**Offline conversions** (in-store, phone) arrive by upload with their own latency and coverage profile."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a fresh model with a delay correction | §0, §4 | *"Model the delay rather than choosing a window."* |
| **A1b** | Latency, freshness and capacity budgets | §1, §3 | *"Scoring inherits ~10 ms; the label join spans ~60 days of clicks."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §9 | A discrete hazard and borrowed λ when the fit breaks |
| **A2a** | The full stack | §4 diagram | Joint model → calibration → auction; join → mutable rows |
| **A2b** | Model components | §5.3 | p-head + λ-head under the censored likelihood |
| **A2c** | Data sources | §5.1 | Clicks, conversions, **attribution output**, modelled conversions |
| **A2d** | Staged pipeline | §5.5 | Join → snapshot → train → calibrate → gate |
| **A3a** | Top-down, unprompted | §0 | *"A negative label is never final"* + killing both naive answers + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"bid = pCTR × pCVR × target, so calibrate the product."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Attribution · delay by vertical · bidding or reporting · observability |
| **A5a** | Alternatives compared | §4, FU 1 | Wait vs window vs weighting vs joint model |
| **A5b** | A full alternative pipeline on demand | §9 | The no-history path |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 2 | *"(1 − p) + p·S(e;λ) — that second term is the whole idea."* |
| **F1b** | Metrics and their trade-offs | §2 | *"The bias is multiplicative, so AUC is blind to it."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Delayed feedback, coverage, attribution drift, eval leakage |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 6, 7 | Mis-specified delay; seasonal shifts |
| **F2b** | Too slow / stale → tiering | §4, §6 | λ refit on its own cadence |
| **F2c** | Fine-tune an existing model | §5.4 | Warm start; a shared backbone with pCTR plus vertical heads |
| **F3a** | Efficiency with a quality trade | §6 | Quantized shared tables |
| **F3b** | Scale levers | §3, §6 | Partitioned streaming join; shared scoring path |
| **F3c** | Extension, with cost and maturity risk | §9, FU 10 | Subscriptions, offline conversions, incrementality |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §8 | Scoring p99; daily training |
| **I1b** | Reliability | §6, §8 | Rollback on the recency slope; reproducible snapshots |
| **I1c** | Usability | §2, §8 | "Still reporting" in advertiser reports |
| **I1d** | Scalability | §3, §8 | The 60-day join; versioned mutable labels |
| **I1e** | Privacy | §8, FU 5 | Modelled / aggregated conversions; consent |
| **I1f** | Safety | §8 | Sensitive conversion types excluded from targeting |
| **I1h** | Security | §8 | **Fake conversion pings** |
| **I1i** | Ethics / bias | §8 | Small advertisers; coverage gaps by segment |
| **I1j** | Explainability | §8, FU 4 | Attribution version on every label |
| **I2a** | Offline → online | §7 | Matured cohorts → shadow → A/B read after maturation |
| **I2b** | The success definition | §7 | *"Flat across elapsed-time buckets; advertisers delivering against target."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training set → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
