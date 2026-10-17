# G14 · Discover Feed — **INTERVIEW SHEET**

> **Tier 1 · #9** — the other half of the canonical Google reco prompt (*"a real-time recommendation system like YouTube or Google Discover"*).
> Full study file: [`../G14_discover_feed.md`](../G14_discover_feed.md) · [diagrams](../G14_discover_feed_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the feed stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Which signals from which products, existing controls, how success is defined (§1) |
| **Explainability** is easy points | A **truthful "why am I seeing this"** on every card, and controls that visibly work (§8) |
| **Security** beyond the gateway | **Clickbait farms, SEO spam** and fake engagement gaming the click head (§8) |
| **Troubleshoot before being asked** | Raise the **latency squeeze** (precompute) and **narrowing** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **No query, no action** — ranking against an *inferred* goal | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · **suppression** as an FR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Cards with reasons + controls · CTR (available) vs **return rate** (meaningful) | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Corpus churns hourly · no query to narrow with · 300 ms budget | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · suppression **before** retrieval → quota'd sources → ranker | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | ⚠ **A scroll-past tells you nothing** · timeliness per topic · objective | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Freshness pipeline · precompute · the quota split · real controls | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Return rate · suppression CI · long-horizon holdback · narrowing | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | AI summaries · the cold-start path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The defining property is that there is no query and no action that started the session. In Search the user tells me what they want; in YouTube recommendation they at least told me what they just watched. Here someone opened a surface and I have to guess what they'd want right now, from a profile assembled from their behaviour on entirely different products.**
>
> **So the honest framing is: I'm ranking against an inferred goal, and I have almost no signal about whether the inference was right. A user who scrolls past everything has told me nothing usable — were the cards bad, or were they in a lift with thirty seconds to kill?**
>
> **That makes this the hardest measurement problem in the recommendation group, and I'd put the metric design before the model design."**

**Then two commitments — the second is the one that separates:**
1. **Freshness and enduring interest are different axes and this surface serves both.** Breaking news is valuable *because* it's new; a woodworking article from 2019 is valuable because it matches a stable interest. Ranking by recency destroys the second, by interest match destroys the first → separate sources with quotas + a **timeliness-sensitivity model per topic**.
2. **⚠ Discover externalises the profile.** In search ranking, personalisation is invisible — the user sees results, not the model's beliefs about them. **Discover *displays* its inferred interests, on a phone other people can see.** So an *accurate* inference can be harmful: a fertility article surfaced from one search is a genuine harm even though the model did its job.
> → **Sensitive-category suppression is a functional requirement, not a policy footnote.**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and what I can actually measure, capacity, the architecture, data and models, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What signals am I allowed to use, and from which products?** *(Cross-product inference is the whole basis — and the whole risk.)*
2. **Is there any explicit user control today?** *(A system that shows you what it thinks about you owes you a way to correct it.)*
3. **How is success defined — session engagement or return rate?** *(Session engagement has an obvious degenerate solution.)*
4. **Signed-in only? Which locales?** *(Thin profiles are the normal case for many users; locale changes the corpus.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | A Discover-style card feed in a mobile app, opened with **no query** |
| Signals | Cross-product interests (search, YouTube, location) **only with activity settings on** |
| Corpus | ~10⁶–10⁷ "currently live" items (news, articles, videos); churns hourly |
| Output | ~20 cards per load, each with a reason and controls |
| Users | Many with **thin profiles** (cold start is normal) |
| Locales | Global, so per-locale corpora and quality |

**FR:** rank a personalised card feed with **no query** · multiple sources with quotas · **timeliness-aware ranking per topic** · **sensitive-category suppression** · *"why am I seeing this"* + working controls · cold start for new users.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Feed load p95 | ≤ 300 ms |
| Content freshness | Breaking news reachable in **minutes** |
| **Suppression correctness** | **Near-1 recall on sensitive categories** — a miss is a harm, not a relevance error |
| Control latency | A "not interested" takes effect **immediately**, visibly |
| Availability | 99.9%; a precomputed or popular-local feed rather than an empty one |

**Scope / NON-GOALS:**
- ❌ **Optimising session engagement alone.** It has a degenerate solution and this surface is especially prone to it.
- ❌ Treating suppression as a downstream filter (it's **pre-retrieval** — §4).
- ❌ Content production/publisher relationships · search (G2) · the interest-profile *inference* itself (I consume it; it's its own system).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
DISCOVER FEED — opened with no query
  [image] "Storm warning for your area tonight"      Local · 12 min ago      ⋮
  [image] "A beginner's guide to dovetail joints"    Because you follow: Woodworking
  [image] "Your team won 3–1"                        Sports you follow
  [image] something new, outside your usual topics   ← exploration slot
  ⋮ menu: Why am I seeing this? · Not interested in <topic> ·
          Don't show content from <publisher> · Follow / unfollow topic
  ⚠ never: health, pregnancy, sexuality, religion, politics, financial distress
     inferred from your behaviour, on a screen others can see
```

**The metrics:**

```
⚠ THE MEASUREMENT PROBLEM IS THE PROBLEM.

AVAILABLE                      MEANINGFUL
CTR on cards        ────────►  RETURN RATE (did they come back?)
session dwell                  post-click SATISFACTION
scroll depth                   "not interested" rate (a negative you CAN trust)

⚠ A SCROLL-PAST IS NOT A NEGATIVE LABEL.
  bad cards? bored? 30 seconds in a lift? no signal distinguishes them.
  → training on scroll-past as negative teaches SESSION LENGTH, not quality.

SO: weight the signals you can TRUST —
  explicit "not interested" · long dwell after click · RETURN next day
  and treat non-interaction as MISSING, not negative.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Return rate** (next-day / weekly opens) | Was the surface worth opening? |
| Trusted signals | "Not interested" rate per topic and source · long dwell after click | Labels you can believe |
| Offline | Per-head AUC + calibration on the exploration slice | Screens only |
| **Guardrails** | **Suppression recall (CI-tested)** · topic narrowing over weeks · publisher concentration · freshness · p95 load | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
⚠ THE STRUCTURAL DIFFERENCE FROM SEARCH/YOUTUBE:
  · no query to narrow the corpus with
  · the corpus CHURNS — today's breaking news is tomorrow's noise
  · every impression is UNSOLICITED, so the precision bar is higher
    (a bad search result is ignored; a bad card is an intrusion)

CANDIDATE POOL is time-bounded, not corpus-bounded:
  ~10⁶–10⁷ items "currently live" rather than 10⁹ all-time
  → retrieval is cheaper than YouTube's, RANKING is harder
  → the cost is in FRESHNESS (continuous re-indexing), not index size

FRESHNESS PIPELINE dominates: new items must be embedded, classified
  for topic + sensitivity, and enter the pool within MINUTES.

LATENCY BUDGET (300 ms p95): profile assembly ~30 · candidate sources (∥) ~60
  · ranking ~120 · re-rank ~30 · overhead ~60
COST: the ranker on ~1,000 candidates per open is the bill → track $ per
  1,000 feeds; precomputing the feed in the background is the first lever
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Composition | One ranking score vs **source quotas** | Quotas: no exchange rate exists between "new" and "matches a stable interest" (FU 7) |
| Freshness | Global recency decay vs **per-topic timeliness** | A sports score decays in hours; a recipe doesn't decay |
| Sensitive topics | Post-filter vs **suppress before retrieval** | A post-filter leaves the slot empty and the pattern still leaks |
| Labels | Scroll-past as a negative vs **as missing** | A scroll-past can't tell bad cards from a user in a lift |

### The system — *draw this in 90 seconds* `A2a`

```
══ CONTINUOUS · the freshness pipeline ══════════════════════════════════════
 new article / video → embed → TOPIC classify → SENSITIVITY classify → live pool
   (minutes, on the critical path: you can't serve what you haven't classified)

══ ONLINE · the user opens the feed (no query) ══════════════════════════════
┌─ 1. PROFILE ASSEMBLY — cross-product interests ────────────────
│ ⚠ SENSITIVE-CATEGORY SUPPRESSION HAPPENS HERE, BEFORE RETRIEVAL —
│   not as a filter afterwards (else the slot is just empty and the
│   pattern still leaks)
│ OUTPUT: an interest profile with sensitive inferences removed
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CANDIDATE SOURCES, EACH WITH A QUOTA ──────────────────────
│ enduring interests · BREAKING / fresh · locally relevant ·
│ continuation · EXPLORATION   ⚠ the quota split IS the product decision
│ OUTPUT: ~1,000 candidates tagged by source
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RANKER — multi-task ───────────────────────────────────────
│ p(click) · p(long dwell) · p(return) · ⚠ NEGATIVE HEAD: p(not interested)
│ × TIMELINESS-SENSITIVITY PER TOPIC (a sports score decays in hours;
│   a recipe does not decay at all)
│ OUTPUT: scored candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. RE-RANK ───────────────────────────────────────────────────
│ source diversity · dedup across days (don't re-show what they
│ skipped) · publisher diversity · policy
│ OUTPUT: ~20 cards
└──────┬─────────────────────────────────────────────────────────
       ▼
 FEED + ⚠ "WHY AM I SEEING THIS" + WORKING CONTROLS
       ▼
 FEEDBACK: not-interested (trustworthy) · click + dwell · ⚠ scroll-past = MISSING
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The latency squeeze:** *"There's no query, so the feed can be **precomputed**: build the candidates in the background when the user is likely to open the app, and do only a light re-rank at open time. If the ranker is slow, I serve the precomputed feed rather than a worse model."*
- **Narrowing:** *"The feed can converge on three topics while every session metric looks fine. Quotas and exploration fix it on the retrieval side, and return rate is the alarm (FU 5)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Suppression pre-retrieval, not post-filter** | "Post-filtering leaves the slot empty and the *pattern* still leaks — the inference must never drive retrieval in the first place" | You lose genuinely wanted content in those categories |
| **2** | **Timeliness sensitivity per topic, not global recency decay** | "A sports score decays in hours; a recipe doesn't decay at all. One decay curve gets both wrong" | A per-topic model to maintain |
| **3** | **Non-interaction treated as missing, not negative** | "A scroll-past doesn't distinguish bad cards from a user in a lift" | Far fewer training labels; slower learning |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Cross-product interest signals** | Topics from search, YouTube, location | Consent; **sensitive inferences must not drive retrieval** |
| **The content corpus** | Articles, videos, publisher metadata | Syndicated duplicates; clickbait publishers |
| **Feed logs, with position** | Impressions, clicks, dwell | Scroll-past is **missing**, not negative |
| **Explicit controls** | Not interested, don't show publisher, follow topic | Sparse but trustworthy |
| **Return visits** | The closest thing to satisfaction | Slow and noisy |
| **Sensitivity labels** | Which items and topics are sensitive | Needs near-1 recall |

### 5.2 Choosing the training set (e) `F1c`

```
1. TRUSTED LABELS  positives = click + LONG dwell; strong negatives = "not interested"
                   ⚠ non-interaction EXCLUDED (missing), not used as a negative
2. CONSENT         only users with the relevant activity settings on; sensitive
                   inferences never become features
3. DEBIAS          position as a training-only input; an EXPLORATION slice kept as
                   the unbiased evaluation set
4. DEDUPE          syndicated copies of one story collapsed
5. SPLIT BY TIME   train on past weeks, validate on the next; per-locale slices
```

**Labels and their bias — this is the section that matters here:**
- **⚠ The absent negative.** No query means no relevance judgement, and non-interaction is uninterpretable. Treat it as **missing**, and weight the signals you *can* trust: explicit "not interested," long post-click dwell, **return next day**.
- **Position bias** — same as everywhere; shallow tower, dropped at serving.
- **Cold start is the normal case**, not an edge case — many users arrive with a thin cross-product profile → content features + popularity priors + **an exploration quota**, not a two-tower embedding that needs history.
- **Feedback loop** — showing a topic generates engagement on that topic, which reinforces the inference. Narrowing happens faster here than on YouTube because there's no query to interrupt it.

### 5.3 The models and the objective (f) `A2b` `F1a`

- **Ranker:** multi-task heads (click, long dwell, return, **not-interested**), trained with log loss on the **trusted** signals only, calibrated so the heads can be combined.
- **Timeliness-sensitivity model:** a separate small model per topic that predicts how fast value decays for that kind of content.
- **Sensitivity classifier:** calibrated for **near-1 recall**, because a miss is a harm.

### 5.4 Fine-tune an existing model (f) `F2c`

The item tower is a pretrained language / vision encoder fine-tuned with **LoRA** on (profile, long-dwell item) pairs with a **contrastive loss**, so a brand-new article has a good embedding the minute it's ingested.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · INGEST     crawl / feeds → embed → topic + sensitivity classify (minutes)
STAGE 2 · POOL       live pool with per-topic freshness
STAGE 3 · TRAIN      multi-task ranker + timeliness model on trusted labels (daily)
STAGE 4 · GATE       suppression CI tests (blocking) → offline screens → shadow → A/B
STAGE 5 · WATCH      return rate, narrowing and the long-horizon holdback
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
FRESHNESS IS THE PIPELINE, NOT THE INDEX:
  ingest → embed → topic-classify → SENSITIVITY-CLASSIFY → pool
  must complete in MINUTES for breaking news to be reachable.
  ⚠ sensitivity classification is ON THE CRITICAL PATH — you cannot
    serve an item you haven't classified, so it can't be a batch job.

CONTROLS MUST BE REAL:
  "not interested" takes effect IMMEDIATELY and VISIBLY.
  A control that appears to do nothing is worse than no control —
  it teaches users the system doesn't listen.

"WHY AM I SEEING THIS": needs a truthful, human-readable reason,
  which constrains the ranker toward attributable features.
```

- **Precompute** `F3b`: the feed's candidates are built in the background when the user is likely to open the app; open time does a light re-rank.
- **Efficiency** `F3a`: the item encoder runs once per item at ingestion; the ranker and encoder are quantized for serving, with the quality trade measured.
- **The composition objective** `A3b`: the **quota split** across sources is the explicit, owned objective (not something tuned to engagement, which collapses it back to the degenerate solution), with exploration kept on.
- **Reliability** `I1b`: an alarm on time-to-pool; if the freshness pipeline stalls, a precomputed or popular-local feed rather than a stale one served silently.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline** | **Return rate**, not session engagement |
| **Trustworthy negative** | "Not interested" rate per topic and per source |
| **⚠ Suppression** | Audited as a **security-style test in CI**: seed sensitive-signal profiles, assert nothing surfaces |
| **Diversity** | Source-quota adherence · publisher concentration · **topic narrowing over weeks** |
| **Freshness** | Time-to-pool for breaking items · staleness of the served feed |
| **Fairness** | Profile-accuracy **≠** profile-appropriateness — measure both |

**Rollout** `I2a`: suppression CI as a blocking gate → offline screens on the exploration slice (IPS for position) → shadow → A/B on **return rate** → a **long-horizon holdback** for narrowing, which a two-week test can't see.

**Success** = return rate up, not-interested rate down, suppression recall at ~1 in CI, topic entropy per user not shrinking. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Absent negative** | Scroll-past teaches session length | Non-interaction as missing |
| **Narrowing** | Shown topics reinforce the inference | Quotas, exploration, long-horizon holdback |
| **Profile-appropriateness** | An accurate inference can still harm when displayed | Pre-retrieval suppression; opt-in for sensitive topics |
| **Publisher concentration** | A few big publishers take the feed | Publisher diversity; new-source exploration |
| **Thin profiles** | Cold-start users get generic feeds | Content priors; bigger exploration quota early |
| **Position** | Top cards get clicked regardless | Shallow position tower |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Feed load p95 ≤ 300 ms; precompute where possible; cost per 1,000 feeds | p95 · $/1k feeds |
| **Reliability** `I1b` | If the freshness pipeline stalls, the feed goes stale silently → alarm on time-to-pool; fallback to a precomputed or popular-local feed | Time-to-pool · stale-feed rate |
| **Usability** `I1c` | **Controls that work immediately and visibly**; "why am I seeing this"; freshness per user, not global | Control latency · not-interested rate |
| **Scalability** `I1d` | Time-bounded pool (10⁶–10⁷ live items); continuous indexing; precomputed item embeddings | Index lag · QPS headroom |
| **Privacy** `I1e` | **Sensitive-category suppression pre-retrieval**; cross-product signals only with consent (activity settings); the feed is **visible on a shared screen** | Suppression CI tests · consent coverage |
| **Safety** `I1f` | Harmful and misleading content (G25 / G28) never amplified; graphic news handled carefully; teen protections | Policy-violating impressions |
| **Security** `I1h` | ⚠ Clickbait / SEO spam and fake engagement (below) | Spam-publisher share |
| **Ethics / bias** `I1i` | **Profile-accuracy ≠ profile-appropriateness**; **narrowing**; **publisher concentration** (FU 9) | Topic entropy over weeks · publisher concentration |
| **Explainability** `I1j` | A **truthful "why am I seeing this"** for every card, which constrains the ranker toward attributable features; publisher-side attribution for traffic changes | % cards with a truthful reason |

> ⭐ *"This surface displays its inferences, so privacy and explainability are the same requirement: never show an inference the user wouldn't want displayed, and always be able to say why a card is there."*

> ⚠ **Worth saying in full: clickbait farms and SEO spam** game the click head. Publisher trust signals, the long-dwell and not-interested heads, and filtering fake engagement before training stop them buying reach.

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extension: AI-generated summaries on cards.** An LLM summary per article (cached **per article**, not per user) adds cost and two risks: **accuracy** for news, so a groundedness check, and **attribution** for publishers, so a visible source link and monitoring of the traffic sent to them.

**When the profile is too thin to personalise, the fallback pipeline** `A5b` `F2a`:

```
new or signed-out user, or a thin cross-product profile
  → CONTENT + POPULARITY PRIORS: what's broadly valuable right now
  → LOCALLY RELEVANT items from coarse location (if permitted)
  → any LOW-SENSITIVITY signal available (never the suppressed categories)
  → a LARGER EXPLORATION QUOTA early: exploring is cheap when there's no profile
  → hand over to the personalised sources as trusted signals accumulate
```

**Top 3 failure modes:**
1. **A correct-but-harmful inference surfaces** — health, pregnancy, sexuality, politics, financial distress. **The model was right and the product failed.**
2. **Narrowing** — the feed converges on three topics and the user stops returning. Session metrics look fine while return rate erodes.
3. **Stale feed for a light user** — someone who opens it weekly sees last week's news; freshness is per-user, not global.

**Q1 ships:** **sensitive-category suppression pre-retrieval** + the control surface + return rate as the headline metric.
> *"Suppression first because it's the failure that's a harm rather than a relevance miss, and because retrofitting it behind an existing ranker is much harder than building the pipeline with it in place."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "The user scrolled past everything. What did you learn?"** `F1c` `F1b`
> Almost nothing usable, and treating it as a negative is the classic mistake here. A scroll-past doesn't distinguish "these cards were bad" from "I had thirty seconds in a lift" from "I was checking the weather widget above the feed." If I train on it as a negative label, the model learns to predict **session length rather than content quality** — it'll favour whatever holds attention in short sessions, which is a different objective than the one I want. So I treat non-interaction as **missing**, and weight the signals I can actually trust: explicit "not interested," which is volunteered and unambiguous; **long dwell after a click**, which is a real quality signal; and **return rate the next day**, which is the closest thing to satisfaction this surface produces. The consequence is far fewer labels and slower learning, and I'd accept that rather than train on a signal I can't interpret.

**2. "Why is sensitive-category suppression a functional requirement rather than a policy filter?"** `I1e` `A3b`
> Because of a property this surface has that search ranking doesn't: **Discover externalises the profile.** In search, personalisation is invisible — the user sees results, not the model's beliefs about them. Here the feed *displays* what the system inferred, on a phone that other people can see over someone's shoulder. That means an **accurate inference can be harmful**: surfacing a fertility article because of one search is a genuine harm even though the model did exactly its job. It also means post-filtering doesn't work — if I retrieve on a sensitive inference and then filter, the slot is empty and the *pattern* still leaks through what's adjacent and what's missing. So the inference must **never drive retrieval**, which makes it an architectural decision at profile-assembly time rather than a filter at the end. I'd accept the cost: users who genuinely want content in those categories get less of it, and that's the right side of the trade.

**3. "How do you balance breaking news against enduring interests?"** `A5a` `A2b`
> They're **different axes** and one ranking function can't serve both — that's the structural insight. Ranking by recency destroys the woodworking article that matches a stable interest; ranking by interest match destroys the breaking story that's valuable precisely *because* it's new. So: **separate candidate sources with quotas**, so the feed composition is a product decision rather than an emergent property of one score; plus a **timeliness-sensitivity model per topic**, because the decay rates differ enormously — a sports score is worthless in six hours, a recipe doesn't decay at all, and a global recency penalty gets both wrong in opposite directions. The quota split then becomes the thing to tune and to own, and I'd want it reviewed like a product decision rather than nudged like a hyperparameter.

**4. "New user, no history. What do you show?"** `F2a` `A5b`
> Cold start is the **normal case** here rather than an edge case, which is a genuine difference from YouTube — many people arrive with a thin or absent cross-product profile, and a two-tower model that needs interaction history has nothing to work with. So: **content and popularity priors** (what's broadly valuable right now, locally relevant given coarse geography), **any low-sensitivity signal that is available** — explicitly excluding the categories in §4 — and a **deliberately larger exploration quota** early, because the cost of exploring on a user with no profile is low and the information gain is high. The important design consequence is that the early feed shapes the profile that shapes every later feed, so an unlucky first session can send someone down a narrow path — which is an argument for keeping exploration on rather than annealing it to zero once the profile firms up.

**5. "Engagement is up but people are opening it less often."** `I2b` `F2a`
> That's the signature failure of this surface and I'd treat the return-rate decline as the real signal, not the engagement gain. It usually means **narrowing**: the feed converged on a few reliably-engaging topics, sessions got stickier, and the surface became less worth opening because it stopped being surprising. Session-level metrics improve while the product's actual value — being worth checking — erodes, and you don't see it for weeks. What I'd check: **topic entropy per user over time**, source-quota adherence (did one source quietly start dominating?), and the **not-interested rate**, which often rises before return rate falls. The fix is on the retrieval side — quotas and exploration — not the ranker, because you can't re-rank variety into a homogeneous candidate set. And I'd want **return rate as the headline metric** precisely so this argument doesn't have to be re-litigated every quarter.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your model correctly infers a user is pregnant. What should happen?"** `I1e` `I1i`
> Nothing should surface, and I'd want that to be a **property of the architecture rather than a judgement call at serving time**. The model being right is exactly what makes it dangerous: the harm isn't inaccuracy, it's **disclosure** — the feed is visible to whoever can see the phone, and the user never asked for that inference to be made public. So pregnancy sits in the suppressed set alongside health, sexuality, religion, politics and financial distress, and suppression happens **before retrieval** so the inference never reaches the candidate sources. I'd also resist the argument that "the user would find it useful," because that reasoning generalises to every sensitive category and the user has no way to know the inference exists in order to object. The principled version: **a system that shows you what it thinks about you should only show inferences the user would be comfortable having displayed** — and where we can't know, the default is suppression plus an *opt-in* control, not opt-out.

**7. "Isn't quota-based candidate generation just hiding a ranking problem?"** `A5a` `A3b`
> No, and the distinction matters. A ranker optimises a scalar; quotas encode a **composition decision that no scalar expresses**. If breaking news and enduring interests were commensurable — if there were a true exchange rate between "this is new" and "this matches a stable interest" — a single score would be fine. There isn't one, because their value to the user depends on context the system can't observe, and any fixed weighting will be wrong for most sessions. Quotas also do something a ranker structurally cannot: they **guarantee the candidate set contains variety**, and you can't re-rank diversity into a homogeneous pool. The honest cost is that quotas are a blunt instrument and the split is hand-set rather than learned — which is why I'd argue the split should be **owned and reviewed as a product decision** rather than tuned to a metric, since tuning it to engagement collapses it back to the degenerate solution quotas exist to prevent.

**8. "How would you know if the feed is narrowing someone's worldview?"** `I2a` `I1i`
> With difficulty, and I'd be honest that the available instruments are weaker than the concern. Within-session diversity is nearly useless — a slate can look varied while the *trajectory* across weeks converges — so I'd measure **topic entropy per user over long windows**, and specifically the **derivative**: is a given user's distribution tightening? Then **long-horizon holdbacks**, a population whose feed has been generated by a deliberately more diverse policy for months, compared on return rate and self-reported satisfaction — because a two-week A/B cannot see this effect at all. And **survey instruments**, since the thing I actually care about ("do you find this surface broadening or narrowing?") isn't inferable from behaviour. The structural problem I'd name: narrowing **improves every short-term metric**, so it has no natural opponent in the org, and the measurement has to exist before anyone needs it — which is why I'd build the holdback in Q1 even though its value only arrives months later.

**9. "Publishers say your feed is killing their traffic. Is that your problem?"** `I1i` `I1j`
> Partly, and the honest answer distinguishes two things. At this scale the feed is a **market maker** — a ranking change is an economic event for the people on the other side of it — so publisher-side impact belongs in a launch review alongside user metrics, and "we didn't intend it" isn't an answer when the effect is predictable. But I'd resist the framing that publisher traffic is the objective: optimising for it directly produces a feed that serves publishers rather than users, and the user is who decides whether the surface survives. What I'd actually build: **publisher concentration monitoring**, because a feed that consolidates onto a handful of large publishers is both an ecosystem risk and usually a diversity failure on the user side too; **attribution** so a publisher can be told whether their decline came from a ranking change, their own content, or a shift in audience interest; and a **new-source exploration quota**, for the same reason as new-creator reach on YouTube — without it the corpus stops renewing and the feed slowly becomes a list of incumbents.

**10. "You have no query, no explicit feedback, and unreliable implicit feedback. How do you make progress at all?"** `F1b` `A3a`
> By being deliberate about which signals to trust and buying the ones I can't get for free — and I'd frame this as the central engineering problem rather than a caveat. Three moves. **Trust the volunteered negatives**: "not interested" is unambiguous, so I'd make that control prominent and fast, which is both a user benefit and a labelling strategy — the control surface *is* data collection. **Buy satisfaction directly** with surveys on a sampled population, which is expensive and low-volume, so it calibrates the automated signals rather than gating releases. And **use return rate as the north star**, because it's slow and noisy but it's the only behavioural signal that means the surface was worth opening. Then accept the consequence: this system learns **slower** than a query-driven one, so the iteration loop is longer and the temptation to substitute a fast proxy — session engagement — is constant. Resisting that substitution is most of the job, and it's why the metric design has to come before the model design.

---

## ⭐ THE L6 LINE

> **"There's no query, so I'm ranking against an inferred goal with almost no signal about whether the inference was right — and this surface *displays* the inference, which makes an accurate one potentially harmful. Metric design comes before model design."**

## 🧭 HOOKS — *steer by what you mention*

- *"A scroll-past is not a negative label"* → invites the labels/measurement branch
- *"Discover externalises the profile"* → invites the privacy/product-harm branch (**good at L6**)
- *"Freshness and enduring interest are different axes"* → invites the retrieval-architecture branch
- *"Suppression is pre-retrieval, not a filter"* → invites the systems/architecture branch
- *"Return rate, not session engagement"* → invites the metrics/org branch (**good at L6**)
- *"The control surface is also the labelling strategy"* → invites the feedback-loop branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to what signals I can actually trust"* — that regenerates the whole metric and model section.
- **Running out of time?** → *"I have five minutes — I'd rather cover suppression and narrowing than finish the ranker detail. Sound right?"*

**Name and defer:**
- "**The interest-inference system itself** is upstream and its own design — I'm consuming its output."
- "**Locale and language** ranking is a large separate problem for a global surface."
- "**On-device personalisation** would change the privacy calculus materially."
- "**Notification timing** for feed content is a separate uplift problem — `P(open|sent)` selects for people who were coming anyway."
- "**Publisher ecosystem health** needs its own metrics; it's not the same as user-side diversity."
- "**Multi-surface consistency** — the same profile drives several products, so a suppression decision has to hold across all of them."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: suppression → quota'd sources → ranker, over a time-bounded pool | §0, §4 | *"Retrieval is cheaper than YouTube's; ranking is harder; the cost is freshness."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"300 ms p95; minutes from publish to pool."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §6 | *"Precompute the feed; if the ranker is slow, serve it rather than a worse model."* |
| **A2a** | The full stack | §4 diagram | Freshness pipeline + profile → sources → ranker → re-rank → feed + controls |
| **A2b** | Model components | §5.3 | Multi-task heads with a not-interested head; per-topic timeliness; sensitivity classifier |
| **A2c** | Data sources | §5.1 | Cross-product signals, corpus, logs, explicit controls, return visits |
| **A2d** | Staged pipeline | §5.5 | Ingest → pool → train → suppression CI gate → watch |
| **A3a** | Top-down, unprompted | §0 | *"Metric design comes before model design."* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 7 | *"The quota split is the owned objective, not tuned to engagement."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Allowed signals · existing controls · success definition · locales |
| **A5a** | Alternatives compared | §4, FU 7 | One score vs quotas; global vs per-topic decay; post-filter vs pre-retrieval |
| **A5b** | A full alternative pipeline on demand | §9 | The thin-profile path: priors + local + exploration |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3–5.4 | *"Log loss on trusted signals only; a LoRA item encoder with a contrastive loss."* |
| **F1b** | Metrics and their trade-offs | §2 | *"CTR is available; return rate is meaningful."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | *"A scroll-past is missing, not negative"*; narrowing; position |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 4 | Narrowing → quotas; thin profiles → the cold-start path |
| **F2b** | Latency too tight → tiering | §4 | Precompute, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA on a pretrained item encoder |
| **F3a** | Quantization with a quality trade | §6 | Quantized ranker and encoder |
| **F3b** | Scale via precomputation | §3, §6 | Embeddings at ingestion; background feed builds |
| **F3c** | Extension, with cost and maturity risk | §9 | AI summaries: accuracy and attribution |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 load; $ per 1,000 feeds |
| **I1b** | Reliability | §6, §8 | Time-to-pool alarm; precomputed fallback |
| **I1c** | Usability | §2, §8 | Controls that act immediately and visibly |
| **I1d** | Scalability | §3, §8 | Time-bounded pool; continuous indexing |
| **I1e** | Privacy | §8, FU 2, 6 | **Suppression before retrieval**, on a shared screen |
| **I1f** | Safety | §8 | No amplification of harmful or misleading content |
| **I1h** | Security | §8 | Clickbait farms, SEO spam, fake engagement |
| **I1i** | Ethics / bias | §7, §8, FU 8, 9 | Appropriateness ≠ accuracy; narrowing; publisher concentration |
| **I1j** | Explainability | §6, §8 | A truthful "why am I seeing this" |
| **I2a** | Offline → online | §7 | Suppression CI → exploration-slice screens → A/B → long-horizon holdback |
| **I2b** | The success definition | §7 | Return rate up, not-interested down, suppression ~1, no narrowing |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
