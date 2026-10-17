# G13 · YouTube Recommendation — **INTERVIEW SHEET**

> **Tier 1 · #1** — the single most-asked ML system design question anywhere.
> Full study file: [`../G13_youtube_recommendation.md`](../G13_youtube_recommendation.md) · [diagrams](../G13_youtube_recommendation_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the serving stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Surface and *"what are we judged on?"* come before any box (§1) |
| **Explainability** is easy points | A reason on every card, per-creator reach attribution, owned head weights (§8) |
| **Security** beyond the gateway | **Fake engagement** poisoning the labels; creators gaming thumbnails (§8) |
| **Troubleshoot before being asked** | Raise the **latency squeeze** and **silent source failure** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | The architecture isn't the answer — the **objective** is | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The home feed · available vs **meaningful** metrics · guardrails | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | 10⁹ → 20 in 100 ms: the funnel · per-stage budget · cost | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · sources with quotas · multi-task ranker · degrade gracefully | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Biased labels · training-set selection · the objective · pipeline | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Budget · sharding · freshness · cheaper serving · the re-rank objective | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Offline screens · A/B + guardrails · **long-horizon holdback** · bias | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multimodal cold start · fallback path · top 3 failures · Q1 ships | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Everyone draws the same three boxes — candidate generation, ranking, re-ranking — and the architecture is genuinely not where the difficulty is. The hard part is what you optimise, and what you refuse to regress.**
>
> **Watch-time optimisation has documented pathologies, and they aren't bugs — they're the system correctly maximising the thing it was told to maximise: longer videos accrue more watch time, emotionally activating content retains attention, and a narrowing sequence keeps people on the platform. Every dashboard improves while the product gets worse.**
>
> **So I'll start at the objective function, then come back to the retrieval stack."**

**Then three commitments, one line each:**
1. **Multi-task with negative heads** — watch time normalised for length, click, like, share, subscribe, **plus dismiss / don't-recommend-channel / survey regret**. *A model with no negative head cannot learn that a watch can be unwanted.*
2. **Named guardrails I refuse to trade** — satisfaction, diversity, new-creator reach, regret rate. **As ship blockers**, because otherwise they lose to a metric with a dollar sign.
3. **Diversity comes from candidate generation, not re-ranking.** You cannot re-rank variety into a homogeneous set.

**Then say the plan (`A3a`):** *"I'll go: clarifying questions and assumptions, the product and its metrics, capacity, the architecture, data and training, serving, evaluation, a production-readiness sweep, then extensions and failure modes. Stop me wherever you'd like depth."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Home feed, up-next, or Shorts?** *(Up-next drives the rabbit-hole risk; home is a cold-start problem; Shorts has a swipe-level signal.)*
2. **What business metric are we judged on?** *(If it's watch time, I want to negotiate the guardrails now, not after the launch review.)*
3. **Is new-creator reach a goal or a nice-to-have?** *(Decides whether exploration gets a budget.)*
4. **What personalisation is allowed?** *(Signed-out users, made-for-kids content, paused watch history.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | **Home feed**, signed-in; up-next treated as a variant |
| Scale | ~10⁹ videos · ~2×10⁹ monthly users · ~10⁵ QPS at peak |
| Output | ~20 videos per page load, each with a reason |
| Objective | Satisfaction-weighted engagement **inside named guardrails** |
| Personalisation | Watch and search history, respecting activity settings; **none on made-for-kids content** |
| Freshness | A new upload is reachable within minutes |

**FR:** a personalised slate · several candidate sources, each with a quota · freshness · controls (*Not interested*, *Don't recommend channel*) that act immediately · a reason per card · creator-side reach.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| End-to-end p95 | **≤ 100 ms** |
| Candidate generation | ≤ 30 ms over ~10⁹ videos |
| Ranking | ≤ 40 ms over ~1,000 candidates |
| Freshness | New video reachable in **minutes** |
| Availability | 99.9%; on failure, a **degraded slate**, never an error page |
| **Guardrails (ship blockers)** | Survey satisfaction · regret rate · new-creator reach · trajectory diversity · borderline-content impressions |

**Scope / NON-GOALS — say these out loud:**
- ❌ **A single-objective watch-time model.** Position, with the reason below.
- ❌ Search (G2) · content moderation (G25) · ads (G21).
- ❌ Retraining on every interaction: near-line, not online.

> **⭐ The reason, spelled out — six mechanisms, each the model doing its job *correctly*:**
> **Long-video bias** — a 3-hour video accrues more watch time than a perfect 4-minute answer. Unnormalised, the model learns *longer is better*.
> **Clickbait** — a misleading thumbnail wins the click, and a partial watch still accumulates time.
> **Engagement ≠ satisfaction** — outrage and anxiety hold attention very effectively. You optimise toward *activating*, not *valuable*.
> **Rabbit holes** — a progressively narrowing sequence maximises session time. Each step is locally optimal; the **trajectory** is the harm.
> **Homogenisation** — the safest watch-time bet is more of what you just watched, so the feed collapses.
> **Self-reinforcement** — the model shapes the logs, the logs train the model; beliefs about what it never shows are unfalsifiable.
>
> ⚠ **None of these are bugs — they are correct maximisation of a wrong proxy.** Which is why the fix is the **objective**, not the model.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
HOME FEED — one page load, ~20 cards
  [thumb] How to fix a bike chain      "Because you watched: bike repair"   ⋮
  [thumb] New upload from @YourChannel "From your subscriptions"            ⋮
  [thumb] A creator you've never seen  "New for you"   ← exploration slot   ⋮
  [thumb] ...  mixed sources · never 3 in a row from one channel
  ⋮ menu: Not interested · Don't recommend channel   ← act on the NEXT load
  occasionally, in-feed: "Was this video worth your time?"   ← the survey card
```

**Available vs meaningful (the L6 move):**

| # | AVAILABLE *(cheap, immediate, gameable)* | MEANINGFUL *(what we actually want)* | Why they diverge |
|---|---|---|---|
| 1 | **Watch time** | **Survey satisfaction** — *"was it worth your time?"* | Long and activating content wins time without being valuable |
| 2 | **CTR** | **Return rate next week** | Clickbait wins the click and costs the return |
| 3 | **Session length** | **Survey-measured REGRET rate** | A rabbit hole is a long session and a bad evening |
| 4 | *(no cheap proxy exists)* | **New-creator reach** | Pure engagement optimisation is rich-get-richer; the corpus stops renewing |

> ⚠ **Row 4 has no left-hand entry, and that is the point** — new-creator reach has no cheap proxy at all, which is exactly why it must be an **owned guardrail** rather than something you hope falls out.

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **7-day return rate** + **survey satisfaction** | Did the feed create value, not just time? |
| Engagement | Length-normalised watch time · CTR · likes, shares | Proximate signals, read **with** the guardrails |
| Offline | Recall@k (retrieval) · per-head AUC + **calibration** (ranking) | **Screens, never decides** |
| **Guardrails** | Regret rate · new-creator reach · trajectory diversity · borderline impressions · p95 latency | **Ship blockers** |

> **"Every one of these has an available metric and a meaningful one, and they diverge exactly when the system gets good at optimising. I gate on the second."**

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
CORPUS      ~10⁹ videos
USERS       ~2×10⁹ MAU · ~10⁵ QPS peak
BUDGET      100 ms end-to-end

THE FUNNEL — each stage ~10× cheaper per item than the next:
  10⁹  videos
   ↓   candidate gen: N sources × ANN, ~30 ms      ← cheap per item, wide
  ~10³ candidates
   ↓   ranking: multi-task DNN, ~40 ms             ← expensive per item, narrow
  ~10²
   ↓   re-rank: diversity, dedup, policy, ~10 ms
  ~20  shown

⚠ THE POINT: you cannot score 10⁹ items in 100 ms. The funnel isn't an
  optimisation — it's the only feasible shape. Cheap-wide → expensive-narrow.

RANKER LOAD: 10⁵ requests/s × ~10³ candidates = ~10⁸ candidate scorings/s
  → the ranker fleet is the cost line: track $ per 1,000 requests, and a
    light pre-ranker (§4) is the first lever when it grows
INDEX: 10⁹ item vectors × 128-d int8 ≈ 128 GB → hash-sharded, replicated 2–3×
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Stage | Option | Strength | Why not / when |
|---|---|---|---|
| Candidate gen | Item-to-item co-watch | Cheap, interpretable | Weak for new users; amplifies what's already popular |
| Candidate gen | **Two-tower ANN** ✅ | Personalised; item vectors **precompute**, so one user-tower pass per request | Needs history, so new uploads need the fresh source |
| Candidate gen | Graph walks (user–video graph) | Good long-tail reach | Expensive to keep fresh at 10⁹ items |
| Ranking | Single-objective watch time | Simple, one label | **Can't learn that a watch was unwanted** (the six mechanisms, §1) |
| Ranking | **Multi-task, shared-bottom or MMoE** ✅ | Positive and negative heads; MMoE handles conflicting tasks | More labels; the head weights need an owner |

> *"Two-tower for retrieval because item vectors precompute; multi-task for ranking because a model with no negative head can't learn that a watch was unwanted."*

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE / NEAR-LINE ══════════════════════════════════════════════════════
 LOGS: impressions WITH POSITION, watches, likes, dismissals, survey answers
   → daily training, warm-started from yesterday → user tower · item tower · ranker
 ITEM TOWER → a vector per video → ANN INDEX (hash-sharded), refreshed near-line
 NEW UPLOADS → content features → the FRESH source within minutes

══ ONLINE · per request ═════════════════════════════════════════════════════
 request (user, context)
       │
┌─ 1. FEATURE FETCH ─────────────────────────────────────────────
│ user history, subscriptions, context · ~20 ms, often the real bottleneck
│ OUTPUT: the user's features
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CANDIDATE GENERATION — SOURCES, EACH WITH A QUOTA ─────────
│ two-tower ANN · subscriptions · fresh · topical · EXPLORATION
│ ⚠ diversity is created HERE; a re-ranker can't invent it later
│ OUTPUT: ~1,000 candidates, each tagged with its source
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RANKER — multi-task DNN (shared-bottom or MMoE) ───────────
│ heads: watch time (length-normalised) · click · like · share · subscribe
│ ⚠ NEGATIVE HEADS: dismiss · don't-recommend-channel · REGRET
│ + shallow POSITION TOWER, dropped at serving
│ score = OWNED weighted blend of calibrated heads
│ OUTPUT: ~100 scored videos
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. RE-RANK ───────────────────────────────────────────────────
│ dedup · policy (borderline content demoted) · TRAJECTORY diversity
│ · per-channel caps · freshness
│ OUTPUT: ~20 videos, each with a reason
└──────┬─────────────────────────────────────────────────────────
       ▼
 CLIENT → LOGS   ⚠ logs are the output of the CURRENT policy (a feedback loop)
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The latency squeeze:** *"If the multi-task DNN can't score 1,000 candidates inside 40 ms, I won't shrink the model for everyone. I'd add a **light pre-ranker** (1,000 → 200: a small DNN or GBDT on cheap features) and keep the heavy model for the 200."*
- **Silent source failure:** *"One source can return nothing and the slate just gets worse, with no error anywhere. So each source has a timeout, its slots are refilled from the other quotas, and **per-source contribution to the final slate** is a dashboard panel."*

### The defensible choices — know the trade for each

| # | Decision | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Multi-task, not single** | "No negative head ⇒ can't learn a watch was unwanted" | More heads = more label sparsity; weights need an owner |
| **2** | **Diversity in candidate gen, not re-rank** | "You can't re-rank variety into a homogeneous set" | Quotas cost some short-term engagement |
| **3** | **Exploration has a funded budget** | "It's ecosystem funding — without it only incumbents are reachable" | A measurable, accepted engagement cost |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Impression logs, with position** | Clicks, watch time, skips | **Position bias**; only shown videos have labels |
| **Explicit negatives** | *Not interested*, *Don't recommend channel* | Sparse, but trustworthy |
| **Survey cards** ("Was this worth your time?") | **Satisfaction and regret** labels | Sparse and expensive, but the only label that measures the failure mode directly. ⚠ An **intermittent in-feed card**, not a per-video button; **Report** is a different system (policy, G25) |
| **Content features** | Title, thumbnail, transcript, frames | The only signal a new video has |
| **User context** | Subscriptions, device, time, locale | Activity settings; signed-out users |
| **Creator data** | Channel age, upload cadence | Feeds the exploration source |

### 5.2 Choosing the training set (e) `F1c`

```
1. ELIGIBILITY   drop invalid engagement (view bots, sub farms) FIRST: it's
                 poisoned labels, not users · no personalised training rows
                 for made-for-kids content · respect deleted history
2. LABELS        watch time NORMALISED by video length · positives net of
                 dismissals · regret / satisfaction from survey cards
                 (joined late) · non-impressions are not negatives
3. DEBIAS        position as an input to the shallow tower only · keep a
                 small EXPLORATION slice whose labels aren't shaped by the
                 current ranker
4. BALANCE       cap rows per heavy user · downsample the very head of the
                 popularity curve
5. SPLIT BY TIME train on days 1…N, validate on day N+1; the exploration
                 slice is the unbiased evaluation set
```

### 5.3 The models and the objective (f) `A2b` `F1a`

- **Retrieval, two-tower:** a user tower (history, context) and an item tower (content, engagement stats), scored by dot product. Loss: **sampled softmax with in-batch negatives**, plus a **logQ correction** so popular videos aren't over-penalised for appearing as negatives often.
- **Ranking, multi-task:** a shared bottom (or MMoE) with one head per objective. **Log loss** for click / like / share / subscribe / **dismiss** / **regret**; a **weighted regression** for length-normalised watch time. The final score is a weighted blend of **calibrated** heads, and **the weights are a product decision with an owner**, not a config nudge.
- **Satisfaction model:** trained on survey answers, then used to predict satisfaction for every impression, because surveys cover a tiny sample. *Satisfaction has to be bought before it can be optimised.*

### 5.4 Fine-tune, don't retrain (f) `F2c`

- **Warm start:** each daily run continues from yesterday's checkpoint on fresh logs, which is how a model at this scale keeps up with drift.
- **Content encoder:** a pretrained video/text model adapted with **LoRA** on co-watch pairs (contrastive loss), so a brand-new video lands near similar videos before anyone has watched it.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · LOGS         impressions + positions + watches + explicit feedback
STAGE 2 · LABELS       join survey answers (they arrive later) · normalise watch time
STAGE 3 · TRAIN        two-tower + multi-task ranker, warm-started; calibrate heads
STAGE 4 · INDEX        recompute item vectors near-line → rebuild / refresh the ANN
STAGE 5 · GATE         offline screens → shadow → A/B with the guardrail panel
CADENCE: fresh source in minutes · item vectors near-line · models daily
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
LATENCY BUDGET (sums to 100 ms):
  feature fetch        ~20 ms   ← often the real bottleneck, not the model
  candidate gen (∥)    ~30 ms
  ranking              ~40 ms
  re-rank              ~10 ms

SHARDING: shard the ANN index by item-id HASH, scatter-gather, replicate 2–3×.
  ⚠ NOT semantic/topic sharding — hotspots + recall cliffs at boundaries.

FRESHNESS: item towers recomputed near-line; new videos enter via the
  FRESH source, not by waiting for the embedding refresh.
```

- **Scale via precomputation** `F3b`: item vectors are precomputed, so serving runs only the user tower; the index is hash-sharded.
- **Cheaper serving** `F3a`: **int8** embedding tables and a **distilled ranker** (a smaller model trained on the big model's scores), with the quality trade measured before launch.
- **The re-rank objective** `A3b`: `λ · predicted value − (1 − λ) · similarity to what this session already watched`, plus per-channel caps. That's **trajectory** diversity written as an objective, with λ owned like the head weights.
- **Reliability** `I1b`: every source has a timeout; if the ranker is down, serve a **cached popular + subscriptions slate** rather than an error.

**What breaks:** feature-store latency spikes · a candidate source failing silently (the slate degrades but nothing errors) · embedding version skew between towers.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE SCREENS  recall@k (retrieval) · per-head AUC + calibration (ranking)
                 ⚠ scored on the EXPLORATION slice (or IPS-weighted), because the
                   logs only label what the current ranker chose to show
                 track the OFFLINE↔ONLINE correlation as its own metric
SHADOW           score live traffic without serving: latency, source mix, policy flags
A/B              engagement + the GUARDRAIL PANEL, which can block a ship
                 ⚠ engagement wins frequently REVERSE over months
LONG-HORIZON     a small population held out for months — the only instrument
HOLDBACK         that sees slow degradation; build it BEFORE you need it
INTERLEAVING     faster, lower-variance ranker comparisons
```

**Success** = return rate and survey satisfaction up, regret not up, new-creator reach held, and the long-horizon holdback agreeing. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | Top slots get clicks regardless of quality | Shallow position tower, dropped at serving |
| **Presentation / selection** | You only observe what you showed | Exploration slice; IPS for offline eval |
| **Popularity** | Popular videos dominate both labels and negatives | logQ correction; head downsampling |
| **Objective bias** | Long videos and clickbait win the proxy | Length normalisation; negative heads; satisfaction model |
| **Feedback loop** | The model shapes the logs that train it | Exploration + long-horizon holdbacks, the only exits |
| **Creator-side** | Incumbents get reach, newcomers don't | Funded exploration; new-creator reach as a guardrail |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | 100 ms p95 end to end; the per-stage budget (§6); cost per 1,000 requests for the ranker fleet | p95 per stage · $/1k requests |
| **Reliability** `I1b` | Each source has a timeout and a quota; if the ranker is down, a **cached popular + subscriptions slate**; the per-source panel catches silent failure | % slates in degraded mode · per-source contribution |
| **Usability** `I1c` | Controls (*Not interested*, *Don't recommend channel*, pause history) that take effect **on the next load**; trajectory diversity so the feed doesn't narrow | Dismiss rate · trajectory diversity |
| **Scalability** `I1d` | The funnel; hash-sharded ANN; item towers near-line; the fresh source for new uploads | Index freshness · QPS headroom |
| **Privacy** `I1e` | Paused or deleted watch history respected end to end; **no personalisation on made-for-kids content**; no sensitive-category inference in features; log retention limits | Deletion propagation time · policy audits |
| **Safety** `I1f` | **Borderline content demoted in recommendations** even when it stays up (*hosting ≠ amplifying*, G25); for teens, limits on repeated exposure to body-image content | Borderline share of impressions |
| **Security** `I1h` | ⚠ **Fake engagement** (view bots, sub farms) poisons labels *and* ranking → filtered **before training**; abuse rate limits; creators gaming thumbnails are caught by the negative heads | Invalid engagement filtered |
| **Ethics / bias** `I1i` | Rich-get-richer → **funded exploration** and **new-creator reach** as an owned guardrail; language and region coverage | New-creator reach · creator exposure concentration |
| **Explainability** `I1j` | **A reason on every card** ("Because you watched X"); **per-creator reach attribution** to answer "did the algorithm kill my channel?" (FU 9); **owned, reviewed head weights**; feature attributions for debugging | % cards with a reason · attribution coverage |

> ⭐ *"A recommendation is distribution, so safety here is mostly about what we amplify, not what we host."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multimodal content embeddings:** encode frames, audio and transcript so a **new video has a representation before it has views**. Cost: a video encoder on every upload (~500 hours/minute). Risk: content similarity predicts satisfaction worse than behaviour does, so it's a cold-start bridge that hands over to behavioural signals as views arrive.
- **Shorts:** swipe-away is a fast, strong negative, and the loop is minutes long, so the near-line pipeline matters more than model size.

**When retrieval underperforms (a new user, a new video), the fallback pipeline** `A5b` `F2a`:

```
no usable history
  → CONTENT SOURCE: the LoRA-tuned content encoder (§5.4) → ANN over content vectors
  → + onboarding topics + popular-in-locale + exploration quota
  → the SAME ranker (scores stay comparable with the main path)
  → policy filter + per-channel caps
  → quality gate: if confidence is low, more popular-in-locale, fewer guesses
```

**Top 3 failure modes:**
1. **Rabbit holes** — measure **trajectory diversity**, not slate diversity. Slate diversity looks fine while the *sequence* narrows.
2. **Self-reinforcing feedback** — model shapes logs, logs train model. Exploration + holdbacks are the only exits.
3. **Silent source failure** — one candidate source returns nothing, slate quality drops, no alarm fires.

**Q1 ships:** the multi-task ranker **with negative heads** + the guardrail panel + the long-horizon holdback infrastructure.
> *"Holdbacks first, because they take months to become useful and you can't retrofit the months."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just optimise watch time?"** `A5a` `F1b`
> Because it has documented pathologies that are *correct* optimisation of a wrong proxy: long-video bias, emotionally activating content, and progressive narrowing. Every dashboard improves while the product degrades. So: multi-task with **negative heads**, and **survey satisfaction as a named guardrail** that can block a ship — because otherwise it loses every argument to a revenue metric.

**2. "How do you handle position bias?"** `F1c`
> A **shallow position tower** trained jointly, then **dropped at serving** — so the model predicts as if everything were at position 1. It's shallow deliberately: you want it to absorb position effects without letting position leak into the content representation. The deeper issue is **presentation bias** — I only observe what I showed — which position correction doesn't fix and exploration does.

**3. "Your A/B shows +3% watch time. Ship it?"** `I2a` `I2b`
> Not on that alone. Short-term engagement wins **frequently reverse over months** in recommendation, so a two-week test is structurally blind to the failure this system is most prone to. I'd check the **guardrail panel** first, then want a **long-horizon holdback** — a population held out for months — before calling it a win. And I'd note the organisational point: the holdback has to exist *before* the decision, which is why I build it in Q1.

**4. "How do you get diversity?"** `A2a` `A3b`
> **In candidate generation, not re-ranking** — that's the systems point most answers miss. If every source returns the same homogeneous set, no re-ranker can create variety that was never retrieved. So: multiple sources with **quotas** (collaborative, subscriptions, fresh, topical, exploration), and the quota allocation is a **product decision**, not a tuning parameter. Then measure **trajectory** diversity across a session, not slate diversity within one.

**5. "New creator, no watch history. What happens?"** `I1i` `F2a`
> Nothing, unless I fund it — which is the honest answer. A pure engagement-optimal system is a **rich-get-richer** machine: only incumbents are reachable and the corpus stops renewing. So exploration is a **funded budget line** with **new-creator reach as an owned guardrail**, and I'd accept a measurable short-term engagement cost for it. Mechanically: an exploration source with a guaranteed quota, plus content-based features so a new video has a representation before it has engagement.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your guardrails and watch time conflict. The VP wants the watch-time win. What do you do?"** `I2b` `A3a`
> I'd have made this argument *before* the conflict, which is the real answer — a guardrail negotiated under pressure loses. Concretely I'd bring three things: the **long-horizon holdback** showing whether the win persists past a quarter, the **regret-rate delta**, and a cost estimate in retention rather than in satisfaction points, because that's the currency the argument is actually conducted in. If the holdback shows the win reverses, that settles it empirically rather than as a values debate. If it genuinely holds and regret is flat, I'd ship it — guardrails are ship *blockers*, not vetoes on everything, and treating them as absolute is how they get removed entirely. The structural point I'd make to the VP: these metrics have **no natural owner** because they make the primary dashboard look worse, so they need to sit outside the team measured on engagement or they erode one defensible increment at a time.

**7. "How do you know a recommendation was bad if the user watched it?"** `F1b` `F1c`
> You largely don't, from behaviour — that's the core measurement problem and why the system needs labels that behaviour can't produce. A watch is compatible with satisfaction, with compulsion, and with regret, and they're indistinguishable in the log. So: **survey regret** as a sparse, expensive, directly-asked label; **dismissals and channel blocks** as volunteered negatives; **return rate at a week**, which catches the case where the session went well and the user didn't come back; and the **long-horizon holdback**, which is the only instrument that sees a slow degradation. The deeper point is that this system's characteristic failure is invisible to its characteristic metric, so the labels have to be *bought* — either with survey infrastructure or with held-out population — rather than harvested.

**8. "Two videos have identical predicted watch time. What breaks the tie, and who decides?"** `A3b` `I1j`
> Mechanically: the other heads — like, share, subscribe, and the negative heads — combined under the weight vector, then the re-ranker's diversity and freshness terms. But the interesting half is the second question. **Those weights are a product decision, not a tuning parameter**, and I'd insist they have a named owner and a review cadence, because a silent weight change is a silent change to what the platform values, applied to billions of impressions, with no code review and no artifact. I've seen this go wrong as "someone nudged a coefficient in a config" — which is a policy change disguised as an ops change. Making the weight vector a reviewed artifact is cheap and it's the kind of thing that separates an L6 answer.

**9. "A creator says your algorithm killed their channel. Are they right?"** `I1j` `I1i`
> Possibly, and I'd want the system to be able to answer rather than to deflect. Three things need to exist: **per-creator reach attribution** so I can tell whether their impressions actually fell and which source stopped supplying them; a **distinction between a policy action and a ranking change**, because those are completely different conversations and conflating them destroys trust; and an **explainability path** that says whether the drop came from a model update, a change in their own content, or a shift in what their audience watches. The honest framing: at this scale we are a **market maker**, and a ranking change is an economic event for the people on the other side of it. That doesn't mean we can't make changes — it means creator-side impact belongs in the launch review alongside user-side metrics, and "we didn't intend it" isn't an answer when the effect is predictable.

**10. "Engagement is up 4% and 7-day retention is flat. Ship or not?"** `I2b` `F1b`
> Not yet, and the flatness is the interesting signal rather than a neutral one. If engagement rose and retention didn't move, the additional watching isn't producing additional *value* to the user — which is exactly the signature of the pathology I named at the start, where the system found a way to extract more time without delivering more satisfaction. So I'd look at **regret rate** and **survey satisfaction** first; if either moved the wrong way, that's a decline regardless of the engagement number. I'd also check **who** the gain came from — if it's concentrated in a small heavy-use cohort while the median user is flat, that's a different product than the aggregate suggests. And I'd want the **long-horizon holdback** before committing, because the failure mode here is specifically one that a two-week window cannot resolve. If everything else is clean and the holdback is healthy, ship — but the burden of proof sits on the engagement win, not on the guardrails.

---
## ⭐ THE L6 LINE

> **"The architecture is table stakes. What separates the answer is naming the objective's pathologies before being asked, putting guardrails up as ship blockers, and knowing that diversity is a retrieval problem, not a re-ranking one."**

## 🧭 HOOKS — *steer by what you mention*

The conversation goes wherever they take it, but **you choose what is visible to pull on.** Drop these deliberately and let them pick:

- *"Diversity comes from candidate generation"* → invites the retrieval-architecture branch
- *"Offline metrics screen, they don't decide"* → invites the evaluation branch
- *"The head weights are a product decision with an owner"* → invites the org/leadership branch (**good at L6**)
- *"Logs are the output of the current policy"* → invites the bandit/exploration branch
- *"Exploration is ecosystem funding"* → invites the marketplace/two-sided branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the guardrails for a second"* — re-read your own NFR list off the board.
- **Running out of time?** → *"I have five minutes — I'd rather cover failure modes than finish the sharding detail. Sound right?"*

**Name and defer** — 8 seconds each, nearly full credit, use them freely:
- "**User cold start** is its own problem — content features plus a fast onboarding signal; happy to go there."
- "**Calibration across heads** matters for the blend; the scores have to be comparable before you can weight them."
- "**Sequence models** for the user tower are a real upgrade over pooled history — I'd A/B it."
- "**Multi-objective weight tuning** can be done with constrained optimisation rather than hand-tuning."
- "**Creator-side fairness** has its own metrics; it's not the same as user-side diversity."
- "**Live/near-line feature freshness** for trending content is a separate pipeline."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a cheap-wide → expensive-narrow funnel | §3, §4 | *"You cannot score 10⁹ items in 100 ms. The funnel is the only feasible shape."* |
| **A1b** | Latency and capacity budgets | §1, §3, §6 | *"100 ms: ~20 fetch, ~30 candidates, ~40 rank, ~10 re-rank; 10⁸ candidate scorings a second."* |
| **A1c** | Degrade without lowering quality for everyone | §4 | *"A light pre-ranker 1,000 → 200, not a smaller model for everyone."* |
| **A2a** | The full stack | §4 diagram | Walk fetch → sources with quotas → multi-task ranker → re-rank → logs |
| **A2b** | Model components | §5.3 | *"Two-tower for retrieval; shared-bottom or MMoE for ranking, with negative heads and a position tower."* |
| **A2c** | Data sources | §5.1 | Impressions with position, explicit negatives, **survey cards**, content features |
| **A2d** | Staged pipeline | §5.5 | Logs → labels (surveys join late) → train → index → gate |
| **A3a** | Top-down, unprompted | §0 | *"I'll start at the objective function, then come back to the retrieval stack."* + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"λ · value − (1 − λ) · similarity to this session, with λ owned like the head weights."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Surface · business metric · new-creator reach · personalisation limits |
| **A5a** | Alternatives compared | §4 | Co-watch vs two-tower vs graph; single objective vs multi-task |
| **A5b** | A full alternative pipeline on demand | §9 | The content-based cold-start path |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Sampled softmax with in-batch negatives and logQ for retrieval; per-head losses for ranking."* |
| **F1b** | Metrics and their trade-offs | §2, §7 | *"Watch time is available, satisfaction is meaningful; offline screens, never decides."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Position, presentation, popularity, objective bias, feedback loop, creator-side |
| **F2a** | Retrieval underperforms → a fallback | §4, §9 | Silent source failure; the content-based cold-start path |
| **F2b** | Latency too tight → tiering | §4 | The pre-ranker, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Warm start daily; LoRA on a pretrained content encoder with co-watch pairs |
| **F3a** | Quantization / efficiency with a quality trade | §6 | *"Int8 tables and a distilled ranker, measured before launch."* |
| **F3b** | Scale via precomputation and light models | §3, §6 | Precomputed item vectors, hash-sharded ANN, near-line towers |
| **F3c** | Extension, with cost and maturity risk | §9 | Multimodal content embeddings; Shorts |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 per stage; $ per 1,000 requests |
| **I1b** | Reliability | §6, §8 | Source timeouts; a cached slate when the ranker is down |
| **I1c** | Usability | §2, §8 | Controls that act on the next load; a reason per card |
| **I1d** | Scalability | §3, §8 | The funnel; sharded index; near-line towers |
| **I1e** | Privacy | §8 | Paused history respected; no personalisation on kids' content |
| **I1f** | Safety | §8 | *"Safety here is what we amplify, not what we host."* |
| **I1h** | Security | §8 | **Fake engagement** filtered before training |
| **I1i** | Ethics / bias | §7, §8 | New-creator reach as an owned guardrail |
| **I1j** | Explainability | §8, FU 9 | A reason per card; per-creator reach attribution; owned weights |
| **I2a** | Offline → online, correcting for shown-only labels | §7 | Exploration slice / IPS → shadow → A/B → long-horizon holdback |
| **I2b** | The success definition | §2, §7 | *"Return rate and satisfaction up, regret not up, guardrails held."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
