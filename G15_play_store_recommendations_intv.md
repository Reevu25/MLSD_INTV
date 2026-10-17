# G15 · Play Store Recommendations — **INTERVIEW SHEET**

> **Tier 3 · #27** — cold start + delayed labels. The label argument is the whole question.
> Full study file: [`../G15_play_store_recommendations.md`](../G15_play_store_recommendations.md) · [diagrams](../G15_play_store_recommendations_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the user sees · (c) the ranking stack · (d) datasets · (e) the delayed-label training set · (f) the models and their objectives · (g) serving and cold start · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Business objective? Developer reach a goal? Which surfaces? Are minors among the users? (§1) |
| **Explainability** is easy points | A reason on every card ("Because you installed …"); **per-app attribution** for developers (§2, §8) |
| **Security** beyond the gateway | **Install fraud**, fake reviews, bot-driven "retention" inflating the signals we train on (§8) |
| **Troubleshoot before being asked** | Raise **stale importance weights** and **category baselines** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Install is the available objective and it's the wrong one** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · retention as the gate · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Install → D1 open → **D7/D30 retention** · uninstall as a negative | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Small corpus, **huge popularity skew**, permanent cold start | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Quota'd sources → multi-task ranker → re-rank · the label bridge | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | ⭐ **Importance weighting by historical retention** · category baselines | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING + COLD START** | **(g)** | ⚠ **New apps are the ecosystem, not an edge case** · exploration quota | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | The proxy↔truth check · matured-cohort gate · holdback | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Re-engagement, subscriptions · **retention-floored fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The label is the whole question here, so I'd start there rather than at the architecture.**
>
> **Install is the available objective and it is the wrong one. It's immediate, complete, and it measures the store listing rather than the app — a polished icon and five good screenshots get an install, and the app is uninstalled the next morning. Worse, an install-optimised ranker converges on a specific and recognisable failure: the store fills with hyper-casual games and utilities that have excellent install rates and terrible retention, because that's exactly what the objective asked for.**
>
> **The objective that matters is retained usage — D7 or D30 retention. But that label arrives one to four weeks after the ranking decision, which breaks the training loop: I can't wait a month to learn from today's impressions."**

**Then the resolution, and the second problem:**
> ⭐ **"So: the delayed-label pattern. A fast proxy for iteration — install, then day-one open — importance-weighted by the historical retention rate of that app and category, plus the slow truth at D7/D30 for correction and gating. Same shape as purchase-without-return in [G7](G7_shopping_ranking_intv.md)."**
>
> **"And the second structural problem: this is a permanent cold-start domain. New apps arrive constantly with zero engagement history, and they're the ecosystem rather than an edge case — so content and metadata features carry far more weight here than collaborative signal."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and the label ladder, the shape of the corpus, the ranking stack, the delayed-label training set and the models, serving and cold start, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What's the business objective — installs, retention, or revenue?** *(If it's installs, I want to renegotiate before designing.)*
2. **Is developer-side reach a goal?** *(Determines whether new apps get a funded exploration budget.)*
3. **What surfaces — home, category, search, post-install?** *(Each has a different intent and a different right answer.)*
4. **Are minors among the users?** *(Family-safe filtering and age-restricted categories change the candidate set.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | The store's **home feed** (rows of app cards), plus category pages |
| Users | ~10⁹ devices; ~10⁸ home loads a day |
| Catalogue | A few million apps; thousands of new ones a day |
| Signals | Impressions, installs, opens and uninstalls, with the user's consent for usage data |
| Business | Organic ranking optimises user outcomes; paid placement is separate (G21) |

**FR:** rank apps per user and surface · multiple candidate sources with quotas · cold start for new apps · **delayed-label training** · category and locale awareness.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Latency p95 | ≤ 200 ms |
| **Gating metric** | **D7/D30 retention**, on a matured cohort |
| Freshness | New app rankable within **hours** |
| **New-app reach** | A monitored guardrail, not an aspiration |
| Proxy↔truth correlation | Tracked, and **an incident when it breaks** |

**Scope / NON-GOALS:**
- ❌ **Optimising installs.** Position, with the named failure it produces.
- ❌ **Waiting 30 days for clean labels.** Iteration would be impossible.
- ❌ Search ranking within the store (adjacent) · ads/paid placement (G21) · app review and policy (G25/G27).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
HOME
  Recommended for you
  [ Puzzle app A ]  [ Budget app B ]  [ Running app C ]   "Because you installed X"
  New & notable                                             ← the EXPLORATION row
  [ new app D ]  [ new app E ]  [ new app F ]               labelled as new
  Top in Productivity                                       ← category row, diversity
  …
  Long-press any card → "Not interested"                    ← a direct negative
```

```
THE LADDER — each rung better and more delayed:

  IMPRESSION → CLICK → INSTALL → D1 OPEN → D7 RETAINED → D30 RETAINED
   immediate ←──────────────────────────────────────────→ 30 days
   measures the LISTING          measures the APP

⚠ INSTALL MEASURES THE STORE LISTING, NOT THE APP.
  A polished icon and five screenshots get an install. The app is
  uninstalled the next morning and the metric already counted it.

⚠ THE NAMED FAILURE OF AN INSTALL OBJECTIVE:
  the store fills with hyper-casual games and utilities — excellent
  install rate, terrible retention — because THAT IS WHAT THE
  OBJECTIVE ASKED FOR. It isn't a bug; it's correct optimisation of
  a wrong proxy.

  ⚠ AND UNINSTALL-WITHIN-24-HOURS IS AN EXPLICIT NEGATIVE, not a
    weak positive. Train a head on it.

GATE ON: D7/D30 retention (matured cohort)
ITERATE ON: install + D1 open, IMPORTANCE-WEIGHTED (§5)
AND TRACK: ⭐ does the proxy still predict the truth?
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **D7-retained installs per home visit**, matured cohort, relative to category baseline | The app was worth installing |
| Fast proxy | Install + **D1 open**, importance-weighted | Iteration speed |
| Monitor | **Proxy↔truth correlation**, per category | Licenses the proxy |
| **Guardrails** | Uninstall-within-24h · new-app reach · category concentration · p95 latency | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **the shape, not the size** `A1b` `I1a` `I1d`

```
CORPUS: ~millions of apps — TINY next to G13's 10⁹ videos.
  → retrieval is NOT the hard part. Ranking and labels are.

BUT TWO STRUCTURAL PROBLEMS THE SIZE HIDES:

 1. ⚠ EXTREME POPULARITY SKEW — a handful of apps are most of the
    installs. A collaborative model trained on raw engagement
    recommends the top-50 chart to everyone, and the CTR metric
    APPROVES, because those apps do get installed.
    → same normalisation argument as G17 (PMI/lift, not raw counts)

 2. ⚠ PERMANENT COLD START — new apps arrive continuously, with
    ZERO engagement history, and they ARE the ecosystem.
    → so CONTENT/METADATA features carry more weight here than
      collaborative signal, and a GBDT over features often beats a
      two-tower embedding that needs interaction history
      (the G16 argument)

LABEL LATENCY: D30 truth means today's model is gated on decisions
  made a month ago → the training set is MUTABLE (G22's problem).

SERVING: ~10⁸ home loads/day ≈ ~1.2k QPS average, several × at peak
  → a few hundred candidates per request, ranked by a GBDT/DNN in
    tens of ms; app features cached (the corpus is small enough)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Objective | Install vs revenue vs **retention, through a delayed-label bridge** | Install measures the listing; revenue invites dark patterns (FU 8) |
| Model family | Two-tower on interactions vs **GBDT/DNN over rich content features** | New apps have no interactions |
| Collaborative signal | Raw co-install vs **PMI-normalised** | Raw counts return the top-50 chart |
| Candidates | One generator vs **several sources with quotas**, incl. new-app exploration | No single source dominates; the catalogue renews |
| Mixed surfaces | One score across categories vs **category baselines and quotas** | No exchange rate between "retained a game" and "retained a bank" (FU 6) |

### The system — *draw this in 90 seconds* `A2a`

```
 USER + SURFACE + CONTEXT
       │
┌─ 1. CANDIDATE SOURCES, EACH WITH A QUOTA ──────────────────────
│ collaborative (co-installed, ⚠ PMI-normalised) · content-similar ·
│ category / trending · ⭐ NEW-APP EXPLORATION · contextual
│ OUTPUT: a few hundred apps, already-installed and incompatible removed
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. RANKER — multi-task ───────────────────────────────────────
│ heads: p(click) · p(install) · ⭐ p(D1 open) · p(D7 retained)
│        ⚠ NEGATIVE: p(uninstall < 24h)
│ features: user history · app CONTENT + METADATA (category, size,
│   permissions, reviews, developer track record) · context
│ OUTPUT: expected D7-retained installs per app
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RE-RANK ───────────────────────────────────────────────────
│ category diversity · quotas · policy and age filters · device fit
│ OUTPUT: the rows the user sees
└──────┬─────────────────────────────────────────────────────────
       ▼
 IMPRESSIONS → install (minutes) → D1 (1 day) → D7/D30 (weeks)
      │                                            │
      └──── fast proxy, IMPORTANCE-WEIGHTED ───────┘
                        ▼
 ⭐ STANDING CHECK: does the proxy still predict the truth?
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `F1c` `I1b`

- **Stale weights:** *"The retention weights are last quarter's estimates. An app update changes its retention, so the weights refresh on a cadence and the correlation check catches drift."*
- **Category baselines:** *"15% D7 can be great for a game and poor for a bank. Weights are relative to the category, or the store tilts toward high-retention categories by accident."*
- **Popularity skew:** *"Raw co-install recommends the chart to everyone — PMI, and evaluation by popularity decile."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Retention as the gate, install as the proxy** | "Install measures the listing, not the app — and an install-optimised store has a known, recognisable failure mode" | A 7–30 day lag on the gating metric |
| **2** | **Importance-weight the proxy** | "Weight today's install by the historical retention rate of that app and category — it's an *estimate* of the label I can't see yet" | The weights are themselves estimated and can go stale |
| **3** | **Content features over collaborative** | "New apps have zero history and they're the ecosystem, not an edge case" | Weaker personalisation for the head, where collaborative would win |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Impressions, clicks, installs** | The fast signal | Position bias; install fraud |
| **Opens and uninstalls** (consented usage) | D1 open, D7/D30 retention, uninstall-within-24h | Weeks of lag; sensitive (FU on privacy) |
| **App metadata**: category, description, screenshots, permissions, size | Cold-start features | Developer-written, optimised for installs |
| **Developer track record** | The strongest cold-start prior | New developers have none |
| **Reviews and ratings** | Quality signal | Fake reviews |
| **Policy and age-rating status** | Hard filters | Upstream (review pipeline) |

### 5.2 The delayed-label training set (e) `F1c`

```
⭐ THE MECHANISM — say it precisely:

  today's impression → install observed in minutes
                     → D1 open observed tomorrow
                     → D7/D30 retention observed in weeks

  TRAIN ON THE FAST SIGNAL, WEIGHTED BY AN ESTIMATE OF THE SLOW ONE:

     weight(install of app A by user U)
        ≈ historical P(D7 retained | installed A, segment(U))
          RELATIVE TO A's CATEGORY BASELINE

  → an install of a high-retention app counts for MORE than an
    install of a known churn-heavy one, TODAY, without waiting.
  → and D1 OPEN is the cheapest large improvement: one day of lag,
    and it already filters most of the "installed and never opened"
    population.

⚠ THE FAILURE: the weights go stale. An app's retention changes
  (an update, a monetisation change, a seasonal shift) and the model
  keeps using last quarter's estimate.
  → refresh weights on a cadence, and ⭐ RUN A STANDING CHECK THAT
    THE PROXY STILL PREDICTS THE TRUTH — when that correlation
    breaks, you optimise the proxy for a quarter before noticing.
```

- **Matured cohorts only** for the D7/D30 heads; the unmatured tail is treated as **censored**, not negative (G22).
- **Uninstall within 24h** is a labelled negative; "not interested" taps too.
- **Remove fraudulent installs** (install farms, incentivised installs) before they become labels.
- **New apps** get weights from their category and developer until they have their own retention history.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective |
|---|---|---|
| **Multi-task ranker** | Score each candidate | Cross-entropy per head (click, install, D1, D7, uninstall-24h), installs **importance-weighted** |
| **Ranking score** | Order the slate | **P(install) × P(D7 retained \| install)** minus an uninstall penalty, within category quotas |
| **Collaborative source** | "Users who installed X" | PMI / lift on co-installs, with a support floor |
| **Content encoder** | Neighbours for new apps | Embeds description and screenshots; trained so retention-weighted co-installed apps sit close |

### 5.4 Fine-tune an existing model (f) `F2c`

LoRA-tune a pretrained image–text encoder on app descriptions and screenshots, with **co-installs weighted by retention** as the positive pairs. New apps get a meaningful position on day one, learned from what users *kept*, not what they clicked. Its embeddings feed both the content-similar source and the ranker.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · LOG         impressions → installs → opens → uninstalls (consented)
STAGE 2 · CLEAN       fraud removal · policy status join
STAGE 3 · WEIGHT      retention estimates per app × segment, relative to category
STAGE 4 · TRAIN       multi-task ranker on weighted fast labels + matured slow labels
STAGE 5 · CHECK       proxy↔truth correlation per category → refresh or alert
STAGE 6 · GATE        matured-cohort D7 on the last model · guardrails → ship
```

---

## 6 · SERVING + COLD START *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

**New apps are the ecosystem:**
- **Content and metadata features from day one** — category, description, screenshots, permissions requested, developer track record, APK size, review text.
- **⭐ A funded exploration quota**, because without impressions an app never earns engagement data and never escapes — rich-get-richer, and the catalogue stops renewing.
- **Developer-track-record features** are the strongest cold-start prior available: a studio with three retained hits is evidence about the fourth.

- **The objective** `A3b`: maximise expected **D7-retained installs** per visit, **subject to** an exploration quota, category quotas on mixed surfaces, and policy floors — constraints, not weights.
- **Efficiency** `F3a`: a GBDT or small DNN over cached app features is cheap; heavy content embeddings are computed offline per app. The trade: offline embeddings lag a new screenshot or description by a refresh cycle.
- **Scale** `F3b`: the app corpus fits in memory per serving node; user features come from a sharded store; candidate lists for common contexts are cached.
- **Reliability** `I1b`: if the ranker is down, serve **category charts filtered by a retention floor**, never the raw install chart; if the weights fail validation, keep the last validated set.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Gate** | D7/D30 retention on a **matured cohort** |
| **Iterate** | Install + D1 open, importance-weighted |
| **⭐ Standing check** | **Proxy↔truth correlation** — a decline is an incident, not noise |
| **Sliced** | By category (retention baselines differ enormously — a game vs a banking app), locale, device tier, **new vs established app** |
| **Guardrails** | New-app reach · category concentration · **uninstall-24h rate** |

```
ROLLOUT: offline replay on matured data → A/B on the weighted proxy (1–2 weeks)
  → the gate decision on that cohort's MATURED D7 → ramp
  → a LONG-HORIZON HOLDBACK for months, because slow drift toward
    high-install, low-value apps is invisible inside a 30-day window
```

**Success** = D7-retained installs per visit up relative to category baselines, uninstall-within-24h down, new-app reach held, and the proxy still predicting the truth. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Popularity** | Co-installs dominated by the chart | PMI; popularity-decile evaluation |
| **Rich-get-richer** | New apps never get impressions | Funded exploration quota; new-app reach guardrail |
| **Category baseline** | Absolute retention favours some categories | Category-relative weights; per-category reporting |
| **Position** | Top cards installed regardless | Position-aware training; exploration slices |
| **Device tier** | Low-end devices get fewer compatible apps and worse data | Per-tier evaluation |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 200 ms; cached app features; offline embeddings | p95 · $ per 1,000 home loads |
| **Reliability** `I1b` | Retention-floored category charts as fallback; last validated weights kept | Fallback rate · weight validation failures |
| **Usability** `I1c` | Reasons on cards; "not interested"; no already-installed or incompatible apps | Uninstall-24h · dismissals |
| **Scalability** `I1d` | Small corpus in memory; sharded user features; cached candidates | QPS headroom at peak |
| **Privacy** `I1e` | **Which apps you use reveals health, finance, dating** → usage data only with consent, aggregated where possible; sensitive categories not used to infer interests; user controls | Sensitive-category inference = 0 |
| **Safety** `I1f` | Policy-violating and malware-flagged apps never recommended; **age-appropriate** filtering for minors; gambling and loan apps restricted by region and age | Unsafe impressions = 0 |
| **Security** `I1h` | ⚠ **Install farms, incentivised installs, fake reviews, bots faking opens** → fraud removal before labels; velocity anomalies; developer-level penalties | Fraudulent installs caught |
| **Ethics / bias** `I1i` | **New-app reach**; developer fairness; **no self-preferencing** of first-party apps (the EU's DMA applies to app stores) | New-app reach · first-party share audit |
| **Explainability** `I1j` | "Because you installed …" on cards; **per-app impression attribution** for developers; policy actions distinguished from ranking changes | % cards with a reason · developer disputes resolved |

> ⭐ *"The store is a market maker. Production readiness means users get apps they keep, developers can see why they rank where they do, and the proxy is checked against the truth every week."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Re-engagement of installed apps:** a separate surface with an **uplift** problem — recommending an app the user would open anyway is worthless.
- **Subscriptions vs one-off apps:** completely different retention semantics; D30 can mean "forgot to cancel".
- **In-store search:** has a stated intent, so it's a ranking problem with a query (G2-shaped).

**When the bridge breaks — the retention-floored pipeline** `A5b` `F2a`:

```
the proxy↔truth correlation drops past threshold, or weights fail validation
  → FREEZE the importance weights at the last validated set
  → gate releases on MATURED D7 only (slower, but trustworthy)
  → home feed: rank within category by smoothed D7 retention
    (Bayesian-shrunk toward the category mean), exploration quota kept
  → diagnose: stale weights · category-mix shift · Goodhart (proxy up, truth flat)
  "Slower iteration on the truth beats fast iteration on a broken proxy."
```

**Top 3 failure modes:**
1. **Hyper-casual collapse** — install-optimised ranking fills the store with high-install, low-retention apps, and the install dashboard says it's working.
2. **Stale importance weights** — the proxy quietly stops predicting retention, and nobody notices for a quarter.
3. **New-app starvation** — no exploration budget, so only incumbents are reachable and the catalogue stops renewing.

**Q1 ships:** the **D1-open signal + importance weighting by historical retention**, and the **proxy↔truth correlation check**.
> *"D1 open is the cheapest large win — one day of lag, and it already removes most of the installed-and-never-opened population. And the correlation check ships with it, because otherwise the whole bridge is unmonitored."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not optimise installs? It's the obvious conversion event."** `F1b` `A5a`
> Because **install measures the store listing, not the app**. A polished icon and five good screenshots get an install, and the app is uninstalled the next morning — the metric already counted it as a success. And the failure this produces is specific and recognisable: an install-optimised ranker fills the store with **hyper-casual games and utilities** that have excellent install rates and terrible retention, because that's precisely what the objective asked for. It isn't a bug; it's correct optimisation of a wrong proxy. The objective that matters is **retained usage** — D7 or D30 — with **uninstall within 24 hours as an explicit negative** rather than a weak positive. The problem is that retention arrives one to four weeks after the ranking decision, which is what makes this a delayed-label design rather than a straightforward ranking one.

**2. "How do you train on a label that arrives four weeks late?"** `F1a` `A2d`
> With a **bridge**: a fast proxy for iteration plus the slow truth for gating, which is the same shape as purchase-without-return in shopping. Concretely, I train on today's install signal but **importance-weight it by the historical retention rate of that app and that user segment** — so an install of a high-retention app counts for more than an install of a known churn-heavy one, *today*, without waiting. Then **D1 open** is the cheapest large improvement available: one day of lag, and it already filters out most of the installed-and-never-opened population, which is where the listing-versus-app gap concentrates. And the slow truth at D7/D30 gates releases and corrects the weights. The critical discipline is a **standing check that the proxy still predicts the truth** — because when that correlation breaks, you optimise the proxy for a quarter before anyone notices.

**3. "A brand-new app with zero installs. What happens?"** `F2a` `I1i`
> It gets ranked on **content and metadata**, because this is a permanent cold-start domain rather than an occasional edge case — new apps arrive continuously and they *are* the ecosystem. Features available on day one: category, description and screenshots, permissions requested, APK size, and — the strongest single prior — **developer track record**, since a studio with three retained hits is real evidence about the fourth. This is also why a GBDT over rich features often beats a two-tower embedding here: the embedding needs interaction history that doesn't exist. And the modelling isn't sufficient on its own — without impressions the app never earns engagement data and never escapes, so it needs a **funded exploration quota**. That's a business decision with a measurable cost, and I'd want it owned explicitly, because it's the mechanism that keeps the catalogue renewing rather than converging on incumbents.

**4. "Your ranker recommends the top-50 chart to everyone. Why?"** `F1c` `A5a`
> **Popularity skew in the collaborative signal.** A handful of apps account for most installs, so raw "installed-also-installed" counts are dominated by them — the top apps co-occur with everything, exactly as the best-seller does in similar-items. And the trap is that the **CTR and install metrics approve**, because those apps genuinely do get installed, so the system looks healthy while doing nothing personalised. The fixes are the same as G17's: **normalise by popularity** — PMI or lift rather than raw co-occurrence — **evaluate sliced by app-popularity decile** so the tail failure is visible, and track **category concentration** as a guardrail. I'd also lean harder on content features, which don't have this bias, and keep the candidate sources quota'd so no single source can dominate the slate.

**5. "How do you know the proxy is still working?"** `I2a` `F1b`
> A **standing correlation check**, and I'd treat it as a first-class piece of the system rather than an analysis someone does occasionally. Periodically, on matured cohorts, I re-measure whether the importance-weighted install signal still predicts D7/D30 retention — and a decline in that correlation is an **incident**, not noise, because everything downstream depends on it. When it breaks, the usual causes are: the **weights have gone stale** (an app changed — an update, a monetisation change, a seasonal shift — and I'm using last quarter's retention estimate), the **category mix shifted** so weights calibrated on one distribution are applied to another, or the **model found a way to satisfy the proxy without the truth**, which is the Goodhart case and shows up as proxy-up-truth-flat. Different responses: refresh weights, re-fit per category, or treat it as a reward-hacking finding and add a guardrail.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Retention baselines differ hugely by category. Does that break your weighting?"** `F1c` `A3b`
> It does if the weighting is global, and this is the slice that matters most in practice. A hyper-casual game with 15% D7 retention may be performing *well* for its category, while a banking app at 40% may be underperforming — so a single global retention weight systematically under-rewards entire categories and pushes the store toward whichever categories happen to have high absolute retention, which is a ranking decision nobody made deliberately. So the weight has to be **relative to a category baseline**: how does this app's retention compare to its category's distribution, rather than to the store's. That also changes evaluation — retention must be reported **per category**, and a headline number that mixes them is close to meaningless. The harder version of the question is whether categories should be comparable at all: a user's home feed puts a game next to a banking app, and there's no principled exchange rate between "retained a game" and "retained a utility." My answer is that the **surface decides** — category-specific surfaces rank within a category, and the mixed home surface needs explicit **category quotas** rather than pretending one score orders them.

**7. "A developer says your algorithm buried their app. What do you tell them?"** `I1j` `I1i`
> I'd want the system able to answer rather than deflect, because at this scale the store is a **market maker** and a ranking change is an economic event for a business. Three things need to exist: **per-app impression attribution**, so I can say whether impressions actually fell and which stage caused it — did they drop out of a candidate source, did their ranking score fall, or did the *surface* change; a clear **distinction between a policy action and a ranking change**, because those are completely different conversations and conflating them destroys trust irreparably; and an **explainability path** that distinguishes "your retention declined" from "a model update changed the ranking" from "your competitors improved." The honest framing I'd give the developer is that **ranking reflects user outcomes** — if D7 retention fell, that's the signal, and it's actionable. What I'd resist internally is optimising for developer satisfaction directly, since that produces a store serving developers rather than users; what I *would* fund is **new-app reach as a monitored guardrail**, because an ecosystem that only surfaces incumbents stops producing the new apps the store depends on.

**8. "Would you use revenue as the objective instead?"** `A3b` `I1i`
> Not as *the* objective, and this is where the three-party structure from shopping ranking applies. Revenue is real and it's measurable, but optimising it directly has a predictable failure: the store fills with **aggressively monetised apps** — heavy IAP, dark patterns, subscription traps — which have excellent revenue per install and produce user harm that shows up much later as reduced trust in the store itself. And the lag is long enough that nobody attributes it correctly. So I'd express it the way G7 does: **maximise revenue subject to floors** on retention, user rating, and policy compliance — constraints rather than a blended weight, because a constraint is a reviewable artifact and a weight quietly moves in the direction of the metric with a dollar sign. I'd also separate **organic ranking from paid placement** cleanly, since blending them is where this goes wrong fastest, and be explicit that the organic ranker's job is user outcomes — the store monetises through a different mechanism, and conflating the two costs the thing that makes the store worth using.

**9. "You have D30 retention as ground truth. Isn't that enough to gate on?"** `I2b` `I2a`
> It's necessary and it's **not sufficient**, for two reasons. First, **D30 is itself a proxy** for the thing that matters — an app retained at 30 days may still be a subscription the user forgot to cancel, and long-run value is closer to sustained engagement plus the user's continued trust in the store. Second, and more practically, **a metric that lags 30 days cannot gate a weekly release cadence** — you'd be gating this week's model on decisions from last month's model, which is a different system. So D30 is the **slow truth used for correction and for periodic deep evaluation**, while the gate for routine releases is the importance-weighted proxy *plus* the correlation check that licenses it. The thing I'd add beyond both is a **long-horizon holdback** — a population whose recommendations have been generated by a deliberately different policy for months — because the failure this system is most prone to (a slow drift toward high-install low-value apps) is precisely the kind that a 30-day window still can't see.

**10. "One quarter, and you inherit an install-optimised system. What do you do first?"** `I2b` `A3a`
> **Measure the gap before changing the objective**, because the change will look like a regression on every dashboard the org currently watches and I need the evidence ready. So: first, instrument **D1 open and D7 retention** and compute, for the current ranker, how much of its recommended volume goes to apps with poor retention — that number is the argument, and in an install-optimised store it's usually startling. Second, ship **D1 open as a training signal with importance weighting**, which is the cheapest large win: one day of lag, no architectural change, and it removes most of the installed-and-never-opened population. Third, ship the **correlation check**, so the bridge is monitored from the start rather than retrofitted. What I would *not* do in quarter one is switch the headline metric, because installs will fall, and without the retention numbers already published and agreed as the new gate, that reads as a failed launch rather than a deliberate trade. The sequencing principle: **build the evidence and the replacement metric before you ask anyone to accept a worse number on the old one.**

---

## ⭐ THE L6 LINE

> **"Install measures the store listing, not the app — and an install-optimised store has a named failure mode. So: retention as the gate, install-plus-D1-open as the proxy, importance-weighted by historical retention, with a standing check that the proxy still predicts the truth."**

## 🧭 HOOKS — *steer by what you mention*

- *"Install measures the listing, not the app"* → invites the labels branch
- *"The delayed-label bridge"* → invites the G22/G7 crossover
- *"New apps are the ecosystem, not an edge case"* → invites the cold-start branch
- *"The proxy↔truth correlation is itself a metric"* → invites the measurement branch (**good at L6**)
- *"The store is a market maker"* → invites the ecosystem/org branch (**good at L6**)
- *"Retention baselines differ by category"* → invites the slicing branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the label ladder"* — impression → install → D1 → D7 regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover cold start and the correlation check than finish the ranker. Sound right?"*

**Name and defer:**
- "**In-store search ranking** is adjacent and has a stated intent, so it's a different problem."
- "**Paid placement** should be separated from organic cleanly; blending them is where this goes wrong."
- "**App review and policy** is upstream and gates what's rankable at all."
- "**Locale and device compatibility** are hard filters that precede ranking, not features within it."
- "**Subscription vs one-off** apps have completely different retention semantics."
- "**Re-engagement of installed apps** is a separate surface with an uplift problem."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the right label first | §0 | *"Install is the available objective and it's the wrong one."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"~10⁸ home loads a day; a few hundred candidates ranked in tens of ms."* |
| **A1c** | Degrade without lowering quality | §6 | *"Ranker down → retention-floored category charts, never the raw install chart."* |
| **A2a** | The full stack | §4 diagram | Quota'd sources → multi-task ranker → re-rank → the label bridge |
| **A2b** | Model components | §5.3 | Multi-task ranker, PMI co-install source, content encoder |
| **A2c** | Data sources | §5.1 | Installs, consented usage, metadata, developer history, reviews, policy status |
| **A2d** | Staged pipeline | §5.5, FU 2 | Log → clean → weight → train → check → gate |
| **A3a** | Top-down, unprompted | §0, FU 10 | *"The label is the whole question"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 6, 8 | *"Maximise D7-retained installs subject to quotas and policy floors."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Objective · developer reach · surfaces · minors |
| **A5a** | Alternatives compared | §4, FU 1, 4 | Install vs revenue vs retention; two-tower vs GBDT; raw vs PMI |
| **A5b** | A full alternative pipeline on demand | §9 | The retention-floored pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 2 | *"Per-head cross-entropy, installs importance-weighted; P(install) × P(D7 \| install)."* |
| **F1b** | Metrics and their trade-offs | §2, FU 1, 5 | The ladder: install measures the listing, retention measures the app |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 4, 6 | Popularity, rich-get-richer, category baselines, position, device tier |
| **F2a** | Something fails → a fallback | §9, FU 3 | Retention-floored pipeline; content cold start |
| **F2b** | Label too slow → design for it | §4, §5.2 | The bridge and stale weights, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA image–text encoder on retention-weighted co-installs |
| **F3a** | Efficiency with a quality trade | §6 | GBDT over cached features; offline embeddings |
| **F3b** | Scale levers | §3, §6 | Corpus in memory; sharded user features |
| **F3c** | Extension, with cost and maturity risk | §9 | Re-engagement uplift, subscriptions, store search |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 ≤ 200 ms |
| **I1b** | Reliability | §6, §8 | Fallback charts; last validated weights |
| **I1c** | Usability | §2, §8 | Reasons; "not interested"; no installed or incompatible apps |
| **I1d** | Scalability | §3, §8 | Small corpus, cached candidates |
| **I1e** | Privacy | §8 | App usage reveals sensitive facts → consent, no sensitive inference |
| **I1f** | Safety | §8, FU 10 | Policy and malware filters; age-appropriate recommendations |
| **I1h** | Security | §8 | **Install farms, fake reviews, faked opens** |
| **I1i** | Ethics / bias | §8, FU 3, 7, 8 | New-app reach; no self-preferencing (DMA) |
| **I1j** | Explainability | §2, §8, FU 7 | Reasons on cards; per-app attribution for developers |
| **I2a** | Offline → online | §7, FU 5, 9 | Replay → A/B on the proxy → matured gate → long holdback |
| **I2b** | The success definition | §7, FU 9, 10 | *"D7-retained installs up, uninstall-24h down, new-app reach held."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) delayed labels → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
