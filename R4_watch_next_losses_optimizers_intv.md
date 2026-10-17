# R4 · Watch-Next Video Recommender — Losses and Optimizer Updates — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md), [R2](R2_video_query_search_intv.md) and [R3](R3_discover_weekly_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **The prompt:** *"Design the 'watch next' recommendations for a large video-sharing platform. While a user is watching a video, the system chooses and orders a list of videos to offer as what to watch next. This interview goes deep on modeling details. Beyond the architecture, expect to write down the exact loss function of every model you propose and the update formulas of the optimizer you would train it with."*
>
> **Three parts:** Part 1, design the system · Part 2, write every loss · Part 3, write the optimizer updates, starting from SGD.
>
> **Expect these probes** (the interviewer's own hints are in quotes): (a) clarifying questions · (b) *"what counts as a good next video"*: objective and labels · (c) *"the video on screen"*: the seed drives retrieval and ranking · (d) features, serving path, latency · (e) *"one loss per kind of label"* · (f) *"negatives nobody logged"*: sampled negatives and position bias · (g) *"build it up"*: SGD → AdamW, and ∂loss/∂logit · (h) *"sparse and dense parameters"* and practical settings · (i) offline → online, and feedback loops.
>
> **Related sheets:** [G13 YouTube recommendation](G13_youtube_recommendation_intv.md) (home feed: the same funnel, the six watch-time pathologies) · [G17 similar items](G17_similar_items_intv.md) (item-to-item, popularity normalisation) · [G21 ad click prediction](G21_ad_click_prediction_intv.md) (calibration, negative downsampling) · [Q26 continuous learning](Q26_continuous_learning_intv.md) (warm starts, propensities).

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | List, autoplay or both? What objective, and which guardrails? Plus your own: *"How do you want the hour split between the system and the math?"* (§1) |
| **Explainability** is easy points | A reason on every card, and an **additive value score**, so every pick decomposes into the heads that drove it (§8) |
| **Security** beyond the gateway | **Co-watch poisoning** by view farms, and **lookalike uploads** that copy a viral video's title and thumbnail to ride the content-similarity source (§8, FU 9) |
| **Troubleshoot before being asked** | Two of the four follow-ups are predictable: **popular-video collapse** and **the hour-old upload**. Raise both in §4, before they're asked |

### This prompt's own "strong answer" checklist, and where each item is earned

| The prompt says a strong answer covers | Where |
|---|---|
| An objective beyond clicks, with labels that match it | §2 (the label ladder), §5.3 (one loss per label) |
| A multi-stage system in which the current video drives candidate generation **and** ranking | §4 (seed-keyed sources, seed × candidate features, the seed inside the MMR set) |
| Exact, correctly defined losses, including sampled negatives and position bias | §5.2 (sampled softmax + logQ, and what the bias does), §5.5 (shallow tower, IPS) |
| Correct optimizer updates, with justified sparse vs dense choices | §6.1 (SGD → AdamW with bias correction), §6.3 (row-wise Adagrad vs AdamW, with memory math) |
| Offline metrics that predict online, an A/B plan, feedback-loop monitoring | §7 |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

> ⚠ **This prompt spends ~24 minutes on math**, so the system design has to fit into ~20. Say so in the opening; the interviewer will help you hold to it.

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **The video on screen is the anchor** · **satisfied watch time, not clicks** · one loss per label, one optimizer per parameter type | A1a, A3a |
| **1–5** | **REQUIREMENTS** | **(a)** | Five questions **unprompted** · assumptions · NFR · non-goals | A4a, A1b |
| **5–8** | **PRODUCT + OBJECTIVE** | **(b)**, (i) | List + autoplay · **the label ladder** · the metric stack | I1c, F1b, I2b |
| **8–10** | **CAPACITY + BUDGETS** | (d) | ~1.5×10⁵ QPS · 7.5×10⁷ scorings/s · a 135 ms budget · embedding-table memory | A1b, I1a, I1d |
| **10–20** | **ARCHITECTURE + SERVING** | **(c) (d)** | Alternatives · seed-driven stack · features · **raise popularity collapse and the fresh upload yourself** | A2a, A5a, A1c, F2b, A3b, F3a |
| **20–34** | **PART 2 · LOSSES** | **(e) (f)** | Notation · sampled softmax + **logQ** · one loss per label · combining · **position bias** | A2b, A2c, A2d, F1a, F1c, F2c |
| **34–44** | **PART 3 · OPTIMIZER** | **(g) (h)** | SGD → momentum → Adagrad → Adam (**bias correction**) → AdamW · **∂ℓ/∂z = p − y** · sparse vs dense · settings | F1a, F3b |
| **44–48** | **EVAL + ROLLOUT + LOOPS** | **(i)** | The **offline–online ledger** · A/B plan · feedback-loop monitors · bias table | I2a, I2b, F1b, F1c |
| **48–51** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, one line each | I1a–I1j |
| **51–53** | **EXTENSION + FAILURES** | — | Seed-only pipeline · semantic IDs · top 3 failures | F3c, F2a, A5b |
| **53–60** | **THE FOUR GIVEN FOLLOW-UPS** | — | Popularity · hour-old upload · L2 vs AdamW · sensational videos | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Two framing points before any boxes. First, on this surface the strongest signal we will ever have is the video on screen. So the current video drives retrieval, through co-watch and a two-tower model conditioned on it, and it appears in every ranking decision as seed-by-candidate features. The user's history personalises around it rather than replacing it.**
>
> **Second, the objective isn't the click. A good next video is one they click, watch for a long time relative to its length, and don't regret: a dismissal, a ten-second abandon or a 'not worth my time' survey answer says the watch was unwanted. So the ranker is multi-task with negative heads, and every loss I write will be tied to one of those labels.**
>
> **Since you want the math, here's how I'll spend the time: about twenty minutes on the system, then the loss for every model, then the optimizer. I'll treat embedding tables and dense layers as two different optimisation problems, because they are."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the product and what counts as a good next video, capacity, the architecture and serving path, then Part 2, the losses, and Part 3, the optimizer; then evaluation and feedback loops, a production-readiness sweep, and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–5)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **A ranked list beside the video, a single autoplay pick when it ends, or both?** *(Both. Autoplay is a decision made for the user, so it gets a stricter bar and its own label.)*
2. **What's the objective: watch time, satisfaction, or a blend? Which guardrails must not regress?** *(Satisfaction-weighted watch time. Guardrails: regret, dismissals, borderline impressions, new-creator reach, latency.)*
3. **Is the user signed in, so watch history is available?** *(Mostly. Signed-out, incognito or paused history → the seed-only pipeline, §9.)*
4. **What latency budget applies to the list?** *(p95 ≤ 150 ms server-side. The autoplay pick can be refreshed near the end of the video.)*
5. **How large is the catalogue, and how fast must new videos become recommendable?** *(~10⁹ videos; recommendable within the hour, once policy checks pass.)*
6. **Made-for-kids content and supervised accounts?** *(No personalised history on them; separate eligibility rules.)*
7. **How do you want the hour split between the system and the math?** *(This is `A3a`: you're managing the conversation.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surfaces | **Up-next list (~20)** + **the autoplay pick** (the top slot, played when the video ends) |
| Scale | ~10⁹ videos · ~5×10⁹ watch-page loads a day · ~1.5×10⁵ QPS at peak |
| Objective | **Satisfied watch time**: click × duration-debiased watch × satisfaction, minus the negatives, **inside named guardrails** |
| Personalisation | Watch history and subscriptions, respecting activity settings; none on made-for-kids content |
| Freshness | A new upload is **eligible within ~1 hour** (after transcoding and policy classifiers) |
| Training | Daily warm-start from yesterday's checkpoint, plus near-line updates for fresh-video embeddings |

**FR:** seed-conditioned candidates from several sources · multi-task ranking · re-ranking for diversity relative to the seed and the session · an autoplay pick with a stricter bar · a reason per card · controls (*Not interested*, *Don't recommend channel*, autoplay off) that act on the next load.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **List latency** | **p95 ≤ 150 ms** server-side; the list loads alongside the player, never blocking playback |
| **Autoplay pick** | Final by **T − 10 s** of the current video; refreshed with in-session signals |
| Freshness | Upload → eligible **≤ 1 h**; the fresh index updated in minutes |
| Availability | 99.9%; on failure, a **degraded but relevant list** (the seed-only path), never an empty panel |
| **Guardrails (ship blockers)** | Survey satisfaction · regret rate · dismiss rate · borderline impressions · new-creator reach · autoplay-chain drift |

**Scope / NON-GOALS:**
- ❌ **Optimising clicks or raw watch time.** Position: clicks reward thumbnails; raw watch time rewards long and sensational videos (G13's six pathologies).
- ❌ The home feed (G13), search (G2), Shorts' swipe feed (a different label structure).
- ❌ Ads in the up-next panel.

---

## 2 · PRODUCT + OBJECTIVE *(min 5–8)* — probes **(b)**, (i) · *"what counts as a good next video"* `I1c` `F1b` `I2b`

**What the viewer sees:**

```
WATCH PAGE
┌──────────────────────────────┐  UP NEXT                                Autoplay [on]
│ ▶ THE VIDEO ON SCREEN (seed) │  [thumb] Part 3 of this series   "Next in this series"  ← AUTOPLAY PICK
│   12:40 / 18:05              │  [thumb] Same technique, deeper  "Related to this video"
└──────────────────────────────┘  [thumb] New from @YourChannel   "From your subscriptions"
                                  [thumb] A creator you've never seen  "New for you"  ← exploration
                                  … ~20 cards · never 3 in a row from one channel
                                  ⋮  Not interested · Don't recommend channel        ← labels
AT T − 10 s:  "Up next in 8 s"  [Cancel]                                   ← a cancel is a label
```

**What counts as a good next video: the label ladder.**

```
IMPRESSION → CLICK → WATCH ≥ 10 s (not abandoned) → WATCHED DEEP *for its length*
          → LIKE / SHARE / SUBSCRIBE → ⭐ SURVEY "worth your time?" = yes
          → RETURNS next week

NEGATIVE LABELS (each gets its own head, because a model with no negative head
cannot learn that a watch was unwanted):
  dismiss · don't recommend channel · ABANDON < 10 s after the click (clickbait)
  · AUTOPLAY CANCEL · survey "regret"

⚠ TWO BIASES INSIDE "WATCH TIME" ITSELF:
  DURATION: a 3-hour video accrues more seconds than a perfect 4-minute one
    → label = the watch time's QUANTILE among videos of similar length (§5.3)
  CLICK COUPLING: watch time per impression = P(click) × E[watch | click]
    → clickbait inflates the first factor; model the two separately
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Satisfied watch time** from watch-next (watch time weighted by the satisfaction model, validated against surveys) + **7-day return** | Value that lasted |
| Engagement | CTR · duration-debiased watch · likes, shares · autoplay continue rate | Proximate, read **with** the guardrails |
| Offline | Recall@k (retrieval) · per-head AUC, **normalised entropy** and **calibration** (ranker) | **Screens, never decides** (§7) |
| **Guardrails** | Regret · dismiss · abandon < 10 s · borderline impressions · new-creator reach · **autoplay-chain drift** · p95 latency | **Ship blockers** |

> **Say:** *"Clicks are the available signal; satisfied watch time is the meaningful one. The ladder gives me one label per rung, and Part 2 gives each label its own loss."*

---

## 3 · CAPACITY + BUDGETS *(min 8–10)* — probe (d) `A1b` `I1a` `I1d`

```
TRAFFIC
  ~5×10⁹ watch-page loads/day ≈ 6×10⁴ QPS average, ~1.5×10⁵ at peak
  RANKER LOAD: 1.5×10⁵ × ~500 candidates = 7.5×10⁷ scorings/s at peak
    → the ranker fleet is the cost line; the pre-ranker (2,000 → 500) is lever #1

LATENCY BUDGET FOR THE LIST (p95 ≤ 150 ms; sums to ~135 ms):
  request gate (OAuth, rate limit, validate)           5 ms
  feature fetch: seed, user history, context          20 ms  ← often the real bottleneck
  candidate sources, ALL IN PARALLEL                   25 ms  (ANN + co-watch lookup)
  pre-ranker (2,000 → 500)                             10 ms
  MMoE ranker (500)                                    45 ms
  re-rank: dedupe, caps, MMR, policy, autoplay pick    10 ms
  network / serialisation                              20 ms

⭐ THE AUTOPLAY PICK HAS A LONGER BUDGET: the video on screen is 18 minutes long.
  Pick at page load, then REFRESH at T − 30 s with in-session signals (did they
  watch through? like it? skip around?). The budget is the remaining runtime.

HEAD-HEAVY SEEDS: a small share of videos are the seed for a large share of page
  loads → cache the SEED-ONLY candidate list per popular seed (co-watch + content
  sources), recompute hourly; personalise on top per request

MEMORY (this drives Part 3):
  ID embeddings: ~3×10⁸ rows (videos above a traffic threshold + channels + hashed
    categorical features; no per-user ID table, see §8 privacy) × 64 dims × 4 B ≈ 77 GB of weights
  ANN index: 10⁹ item vectors × 64-d int8 ≈ 64 GB per replica, sharded
```

---

## 4 · ARCHITECTURE + SERVING *(min 10–20)* — probes **(c) (d)** · *"the video on screen"* `A2a` `A5a` `A1c` `F2b` `A3b` `F3a`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Objective | Clicks · raw watch time · **click × duration-debiased watch × satisfaction − negatives** | Clicks reward thumbnails; raw watch time rewards length and sensationalism |
| Retrieval anchor | User only · seed only · **seed + user** | The video on screen is the strongest single signal; history personalises around it |
| Candidates | One two-tower · **several sources, each with a quota** | Co-watch is precise for head seeds but blind to new videos; content similarity covers new uploads |
| Ranker | GBDT · single-task DNN · **multi-task MMoE + shallow position tower** | Heads conflict (click vs dismiss); position is absorbed in training and dropped at serving |
| Watch-time head | Weighted LR (YouTube 2016) · MSE on log-seconds · **duration-quantile soft-label BCE** | Weighted LR learns E[watch per impression] but rewards length and couples to the click (FU 5) |
| Position bias | Ignore · IPS · **shallow tower, plus IPS on the randomised slice** | The tower needs no randomisation and has low variance; IPS is unbiased only if propensities are right |
| Re-rank | Top-k by score · **MMR with the seed in the "already picked" set + caps + policy** | Twenty near-copies of the seed is one recommendation |
| Optimizer | Adam everywhere · **row-wise Adagrad on embeddings, AdamW on dense layers** | §6.3: memory, sparsity and rare IDs |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE / NEAR-LINE ══════════════════════════════════════════════════════════
 LOGS: impressions WITH POSITION + surface (list | autoplay), clicks, watch time,
       dismissals, autoplay cancels, survey answers
   → labels (§5.3) → daily warm-started training: two-tower · MMoE ranker
 ITEM TOWER → a vector per video → ANN INDEX (daily) + FRESH TIER (streaming, minutes)
 CO-WATCH TABLE: seed → top-200 next-watched, PMI-normalised (hourly)
 UPLOAD PIPELINE: transcode → policy classifiers → content embedding → FRESH TIER

══ ONLINE · per watch-page load ════════════════════════════════════════════════
┌─ 1. REQUEST ───────────────────────────────────────────────────
│ TASK: authenticate, rate-limit, validate
│ OUTPUT: (user, SEED c, surface, device, watch progress on c)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. FEATURE FETCH ─────────────────────────────────────────────
│ TASK: seed features (ID + content embedding, topic, channel, length)
│       · user history (last ~100 watches) · context
│ OUTPUT: the inputs to every model below
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. CANDIDATE SOURCES, IN PARALLEL, EACH WITH A QUOTA ─────────
│ co-watch(c) · two-tower ANN with q = f(user, c) · next in series /
│ channel · content-similar(c) incl. the FRESH tier · subscriptions
│ (small quota) · EXPLORATION
│ FILTER: recently watched · policy-ineligible · kids rules · region
│ OUTPUT: ~2,000 candidates, each tagged with its source
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. PRE-RANKER ────────────────────────────────────────────────
│ TASK: a small model distilled from the ranker, cheap features
│ OUTPUT: ~500 candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. RANKER: MMoE, multi-task ──────────────────────────────────
│ inputs include SEED × CANDIDATE crosses (co-watch PMI, embedding
│ similarity, same series/channel, length ratio, language match)
│ heads: click · watch quantile · like · share · ⚠ dismiss · abandon
│        · satisfaction · autoplay-continue  + SHALLOW POSITION TOWER
│ OUTPUT: calibrated heads + an additive value score per candidate
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. RE-RANK ───────────────────────────────────────────────────
│ dedupe re-uploads · channel caps · MMR against {seed} ∪ picked ·
│ borderline demotion · AUTOPLAY PICK under a stricter bar
│ OUTPUT: ~20 cards + the autoplay pick, each with a reason + provenance
└──────┬─────────────────────────────────────────────────────────
       ▼
 CLIENT → LOGS   (and at T − 30 s: refresh the autoplay pick)
 ⚠ the logs are the output of the CURRENT policy: a feedback loop (§7)
```

**Where the video on screen enters (say this; it's a checklist item):** it **keys** three of the six sources (co-watch, series/channel, content-similar), it's an **input** to the two-tower query, it drives **seed × candidate features** in both rankers, and it's **inside the MMR set** in re-ranking, so the list can't fill up with near-copies of it.

### Features — probe (d)

| Group | Examples |
|---|---|
| **Seed (c)** | ID embedding, content embedding (title, transcript, frames), topic, channel, length, age, language, **how far the user has watched it** |
| **Candidate (v)** | The same, plus engagement stats (CTR, mean watch quantile, dismiss rate) and freshness |
| **Seed × candidate** | Co-watch count and PMI(c, v), cosine(e_c, e_v), same series / next episode, same channel, length ratio, language match |
| **User** | History pooled with attention **to the seed** (which past watches matter for *this* video), subscriptions, device, time |
| **Context** | Surface (list vs autoplay), **position: training only**, through the shallow tower |

### Serving, efficiency and the re-rank objective `F3a` `A3b` `I1b`

- **Efficiency** `F3a`: item vectors are precomputed; only the query tower runs per request. **int8** ANN vectors halve or quarter index memory at a small recall cost, gated on recall@k. The **pre-ranker is distilled** from the ranker, which trades a little quality on the tail of the 2,000 for 4× fewer heavy scorings.
- **The re-rank objective** `A3b`: greedy MMR, `pick v maximising  λ·score(v) − (1−λ)·max_{u ∈ {c} ∪ S} sim(v, u)`, where S is the set already picked and **the seed c is in the set from the start**. Then hard constraints: ≤ 2 per channel, borderline demoted, kids rules.
- **The autoplay pick** is the top of the list **only if** it clears a stricter bar: P(satisfaction) ≥ τ_auto, not borderline, not a near-duplicate of the seed. Otherwise the next one that does.
- **Reliability** `I1b`: each source has a timeout and its quota refills from the others; if user features time out → the **seed-only pipeline** (§9); if the ranker is down → the cached per-seed list. **Never an empty panel.**

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F2a`

- **The latency squeeze** `F2b` `A1c`: *"If 500 candidates don't fit in 45 ms, I won't shrink the ranker for everyone. I'd tighten the distilled pre-ranker and cache seed-only candidates for head seeds, so the heavy model only scores what it must."*
- **Popularity collapse** `F2a`: *"My retrieval loss has a logQ correction, which **restores** the popularity prior rather than removing it. If the list converges on the same popular videos, that term is the dial (FU 1)."*
- **The hour-old upload:** *"A new video has no ID embedding and no co-watch, so it enters through content similarity in a streaming fresh tier, with a funded exploration quota (FU 2)."*
- **Autoplay chains:** *"Each autoplay hop is locally optimal and the chain can still drift somewhere bad, so trajectory drift is a monitored guardrail, not an afterthought (FU 4)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Seed + user, not user only** | "On this surface the video on screen is the strongest signal we have" | Less personal for users with strong habits; history attention recovers most of it |
| **2** | **Multi-task with negative heads** | "No negative head ⇒ the model can't learn that a watch was unwanted" | More heads, sparser labels; the serving weights need an owner |
| **3** | **Duration-quantile watch label** | "Seconds reward length; quantiles compare like with like" | Loses the absolute-time signal; recovered by an explicit length feature |
| **4** | **Row-wise Adagrad + AdamW** | "Embeddings and dense layers are different optimisation problems" | Two optimizers and two learning-rate schedules to tune |

---

## 5 · PART 2 · THE LOSSES *(min 20–34)* — probes **(e) (f)** · *"one loss per kind of label"*, *"negatives nobody logged"* `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.0 Notation — *write this on the board first; every formula below uses it*

```
u        the user                    c   the SEED: the video on screen
x        = (u, c, context)           v   a candidate video        V  the corpus
B        batch size                  τ   softmax temperature
q = f_θ(x) ∈ ℝ^d     query tower output, L2-normalised
e_v = g_φ(v) ∈ ℝ^d   item tower output, L2-normalised (the seed c uses the SAME tower)
s(x, v) = ⟨q, e_v⟩ / τ        retrieval score
Q(v)     probability that v appears as a sampled item in a batch
ρ_i      reward weight of positive pair i (satisfaction-weighted, §5.2)
k        a ranking head: click · watch · like · share · dismiss · abandon · sat · auto
z_ik     logit of head k on row i      p_ik = σ(z_ik) = 1 / (1 + e^(−z_ik))
y_ik     label of head k on row i      m_ik ∈ {0,1}: 1 if head k has a label on row i
pos_i    the position impression i was shown at
κ_pos    P(examined | position): the examination propensity
α_k      TRAINING loss weight of head k     w_k  SERVING value weight of head k
```

### 5.1 The datasets `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Impression logs, with position and surface** | Clicks, watch time, abandons, autoplay cancels | **Position bias**; only shown videos have labels |
| **Next-watch sequences** (any surface) | (seed → next video) pairs for retrieval | Next-watches from **search or subscriptions** aren't shaped by our ranker: keep them, they soften the loop |
| **Explicit feedback** | Dismiss, don't recommend channel, likes, shares | Sparse, trustworthy |
| **Survey cards** ("worth your time?", 1–5) | The only direct satisfaction / regret label | Very sparse; asked on a sample |
| **Content features** | Title, description, transcript, thumbnail, frames | The only signal a new video has |
| **Policy classifier outputs** | Borderline / violative scores | A constraint, never something engagement learns around |
| **Exploration slice with logged propensities** | Labels not shaped by the current ranker; position randomisation | Small and precious |

### 5.2 Retrieval: sampled softmax with a logQ correction — *"negatives nobody logged"* `F1a` `F1c`

**The label:** pair i is (x_i, v_i), where v_i is the next video the user actually watched after seed c. The logs contain only positives. Nobody logged the 10⁹ − 1 videos they didn't pick, so **the negatives have to be sampled.**

```
THE IDEAL (infeasible over 10⁹ videos):
  P(v | x) = exp(s(x,v)) / Σ_{v' ∈ V} exp(s(x,v'))        L = −log P(v⁺ | x)

WHAT I TRAIN: in-batch sampled softmax, with logQ correction and reward weights
                         exp( s(x_i, v_i) − log Q(v_i) )
  L_retr = −(1/B) Σ_i ρ_i · log ───────────────────────────────────────
                         Σ_{j=1..B} exp( s(x_i, v_j) − log Q(v_j) )

  · every OTHER row's positive serves as a negative for row i, for free
  · ACCIDENTAL HITS: if v_j = v_i for j ≠ i (a popular video twice in one batch),
    mask that logit; it is not a negative
  · Q(v) ≈ v's share of training rows, from a streaming frequency estimate
  · ρ_i = the duration-quantile watch label × 1[not dismissed, not abandoned]
    → a clickbait "positive" carries almost no weight
```

**The bias that sampling introduces, and what the correction does.** This is the part interviewers probe:

```
In-batch negatives are drawn ∝ Q(v), i.e. by POPULARITY, not uniformly.

UNCORRECTED, the optimum is:   s*(x,v) = log P(v|x) − log Q(v) + const(x)
  → a PMI-like score: popular videos are OVER-PENALISED, because they show up
    as negatives in nearly every batch
CORRECTED (subtract log Q):    s*(x,v) = log P(v|x) + const(x)
  → a consistent estimate of the full softmax as B grows

⚠ So logQ corrects SAMPLING bias. It does not remove POPULARITY: it restores
  the true P(v|x), and the true next-watch distribution IS popularity-heavy.
  (This is FU 1.)

SECOND BIAS: the tail almost never appears as a negative, so tail embeddings
  are barely trained to be told apart → noisy tail scores.
  FIX: MIXED NEGATIVES. Add B' videos drawn uniformly from the corpus, and correct
  every negative with the MIXTURE's probability, not the in-batch Q.

WHY NOT use shown-but-not-clicked impressions as retrieval negatives?
  They were good enough to be retrieved and ranked: often relevant, just not
  chosen. Published retrieval work (Facebook's embedding-based retrieval) found
  them worse than random negatives for this stage. They belong in the RANKER.
```

**Why softmax fits this label:** the label is "which one video came next" among 10⁹, an implicit, positive-only, multiclass event. A softmax models exactly that distribution, and it's ranking-consistent with what retrieval is judged on (recall@k).

### 5.3 Ranking: one loss per kind of label — *"one loss per kind of label"* `F1a`

| Label | Kind | Rows it's defined on (m_ik = 1) | Loss | Why it fits |
|---|---|---|---|---|
| **Click** | Binary | Impressions, **examined** (§5.5) | BCE (+ position handling) | The blend needs a **calibrated** probability, and BCE is a proper scoring rule |
| **Watch quantile** | Bounded continuous, y ∈ [0,1] | Clicked rows | **Soft-label BCE** | Its minimiser is E[y \| x]; gradient bounded by 1; removes duration bias |
| **Like / share / subscribe** | Binary, sparse | Clicked rows | BCE | Conditional on the click, so CTR doesn't leak in |
| **Dismiss / don't recommend channel** | Binary, explicit negative | Impressions | BCE | The model must learn "unwanted" directly |
| **Abandon < 10 s** | Binary | Clicked rows | BCE | The clickbait detector: a click the content didn't earn |
| **Autoplay continue** | Binary | Autoplay rows | BCE | Autoplay is a different act from a click; a cancel is a strong negative |
| **Survey satisfaction** | Ordinal, 1–5, very sparse | Surveyed rows | **Softmax CE over 5 classes** → E[score] | Sparse but the only direct label; a satisfaction model is trained on it and distilled into a head |
| **Next watch** (retrieval) | Implicit positive among 10⁹ | Next-watch pairs | Sampled softmax + logQ (§5.2) | Multiclass over the corpus |

**The binary loss, once, with every symbol:**

```
ℓ_BCE(z, y) = −[ y · log σ(z) + (1 − y) · log(1 − σ(z)) ]
  z ∈ ℝ the head's logit, σ the sigmoid, y ∈ {0,1}, or y ∈ [0,1] for soft labels
```

**The watch-time label, defined precisely:**

```
y_watch = F_{d(v)}( T )
  T        seconds watched after the click
  d(v)     v's duration bucket (e.g. 30 log-spaced buckets of video length)
  F_d      the empirical CDF of T among clicked impressions in bucket d
→ "this watch was in the 80th percentile FOR VIDEOS THIS LONG"
  removes duration bias (cf. D2Q, KDD 2022); bounded in [0,1], so BCE applies
  · completion ratio min(T/len, 1) is the tempting alternative: it is biased
    toward SHORT videos, which complete easily
```

**The classic alternative, and why I don't use it as the main head** (FU 5 derives it): YouTube's 2016 weighted logistic regression weights **positives by watch time T** and negatives by 1. Its learned odds are `e^z ≈ E[T per impression]`. That's elegant, but it multiplies P(click) into the target (so clickbait leaks in), it grows with video length, and the approximation breaks when CTR isn't small.

### 5.4 Combining several objectives into one training loss `F1a` `A2b`

```
                      Σ_i m_ik · ω_i · ℓ_k(z_ik, y_ik)
L_rank = Σ_k  α_k · ──────────────────────────────────
                            Σ_i m_ik

  m_ik   masks each head to the rows where its label exists (watch head: clicked rows)
  ω_i    per-row weight: heavy-user cap; IPS weight on the click head if used (§5.5)
  α_k    TRAINING weight: balances the gradient scale of the heads
  the per-head normaliser stops a dense head (click) from drowning a sparse one (sat)

TOTAL:  L = L_rank + L_retr   (trained as separate models; the ranker also gets
        the retrieval score as a feature)

TWO SETS OF WEIGHTS — DON'T CONFUSE THEM:
  α_k  (training)  an OPTIMISATION choice: hand-set, or learned by uncertainty
                   weighting:  Σ_k [ ½·e^(−s_k)·L_k + ½·s_k ],  s_k = log σ_k² learned
                   (Kendall et al. 2018); it balances the heads, it does not decide value
  w_k  (serving)   a PRODUCT decision, owned and reviewed:
        score(v) = p_click · ( w_watch·ŷ_watch + w_like·p_like + w_sat·p_sat
                               − w_abandon·p_abandon )  −  w_dismiss·p_dismiss
        additive in the heads → every pick decomposes into what drove it (§8, I1j)

ARCHITECTURE FOR CONFLICTING HEADS: MMoE
  f_k(x) = h_k( Σ_e g_ke(x) · E_e(x) ),   g_k(x) = softmax(W_k x)
  E_e experts shared by all heads; g_k each head's own gate → click and dismiss
  can lean on different experts instead of fighting over one shared bottom
  (the architecture of YouTube's "Recommending What Video to Watch Next", RecSys 2019)

CALIBRATION, because the blend multiplies probabilities:
  negatives downsampled at rate r → correct:  p = p' / ( p' + (1 − p')/r )
  then isotonic per head on a fresh window; monitor predicted/observed per head
```

### 5.5 Position bias in the training data `F1c`

**The mechanism:** a click needs the viewer to **examine** the card and find it **relevant**. Under the examination hypothesis, `P(click | v, pos) = κ_pos · P(relevant | v)`. Train on raw clicks and the model learns "being in slot 1 is good", entrenching the old ranker.

```
OPTION A · SHALLOW POSITION TOWER (what I'd ship)
  z_click = z_main(x, v) + z_bias(pos, surface, device)
  · trained jointly; position features get ~10% dropout so z_main doesn't lean on them
  · at SERVING: drop z_bias (position is unknown before ranking anyway)
  · z_bias absorbs ≈ log κ_pos (exact in the small-CTR limit, where σ(z) ≈ e^z)
  · gradient: ∂ℓ/∂z_main = ∂ℓ/∂z_bias = p − y   (additive logits share it)

OPTION B · INVERSE PROPENSITY SCORING (unbiased, if κ is right)
  ℓ_IPS = −[ (y/κ_pos) · log p + (1 − y/κ_pos) · log(1 − p) ]
  · its expectation over examination equals the loss on TRUE relevance
  · κ_pos estimated from a small RANDOMISED-SWAP slice (swap slot 1 with slot k)
  · ⚠ (1 − y/κ) < 0 when y = 1 → negative weights, high variance
    → clip κ ≥ κ_min, and use it on the click head only

WHICH HEADS NEED IT: the CLICK head. Post-click heads (watch, like, abandon) are
  conditioned on the click and far less position-dependent.
AUTOPLAY is its own "position": it plays by default, so the label is
  "continued ≥ 30 s without cancel", not "started".
UNEXAMINED ≠ NEGATIVE: cards below the fold the user never scrolled to are
  dropped, not labelled 0.
```

### 5.6 Fine-tune an existing model `F2c`

- **Content encoder:** a pretrained video-text model, **LoRA-tuned contrastively on co-watch pairs** (InfoNCE, same form as §5.2), so its embedding learns what viewers treat as related, not just what looks alike. This is what places an hour-old upload (FU 2).
- **Warm start:** each daily run continues from yesterday's checkpoint, **optimizer state included** (§6.4), on the new day's logs.

### 5.7 The staged pipeline `A2d`

```
STAGE 1 · CLEAN     invalid traffic (view bots, co-watch farms) removed FIRST
STAGE 2 · LABEL     duration quantiles · abandons · autoplay cancels · survey join (late)
                    · unexamined dropped · propensities attached on the randomised slice
STAGE 3 · TRAIN     two-tower (sampled softmax) + MMoE ranker, warm-started; calibrate
STAGE 4 · INDEX     item vectors → ANN daily · fresh tier streaming · co-watch hourly
STAGE 5 · GATE      offline screens → shadow → A/B with the guardrail panel (§7)
CADENCE: fresh tier minutes · co-watch hourly · models daily · content encoder weekly
```

---

## 6 · PART 3 · THE OPTIMIZER *(min 34–44)* — probes **(g) (h)** · *"build it up"*, *"sparse and dense parameters"* `F1a` `F3b`

### 6.1 Build it up, from SGD, one added term at a time

```
θ  parameters    g_t = ∇_θ L_batch(θ_t)  the minibatch gradient at step t
η  learning rate     ⊙, ⊘, √ and g² are elementwise

STEP 0 · SGD
  θ_{t+1} = θ_t − η · g_t
  one step size for every parameter: oscillates along steep directions,
  crawls along flat ones, and the minibatch noise goes straight into θ

STEP 1 · + MOMENTUM: average the direction
  m_t = β · m_{t−1} + g_t           θ_{t+1} = θ_t − η · m_t
  ≈ an average of the last ~1/(1−β) gradients: damps oscillation, accelerates
  directions that agree step after step (β = 0.9 → ~10 steps)

STEP 2 · + PER-COORDINATE SCALE: Adagrad
  G_t = G_{t−1} + g_t ⊙ g_t          θ_{t+1} = θ_t − η · g_t ⊘ (√G_t + ε)
  coordinates with large or frequent gradients take SMALL steps; RARE ones keep
  LARGE steps  ← exactly what a rarely-seen video-ID row needs
  ⚠ G only grows, so the step only shrinks: over a long run it stops learning

STEP 3 · + FORGET OLD GRADIENTS: RMSProp
  v_t = β₂ · v_{t−1} + (1 − β₂) · g_t²     θ_{t+1} = θ_t − η · g_t ⊘ (√v_t + ε)
  an exponential average instead of a sum: the scale tracks RECENT gradients

STEP 4 · ADAM = momentum (as an average) + RMSProp + BIAS CORRECTION
  m_t = β₁ · m_{t−1} + (1 − β₁) · g_t          first moment
  v_t = β₂ · v_{t−1} + (1 − β₂) · g_t²         second moment
  m̂_t = m_t / (1 − β₁^t)                      bias-corrected
  v̂_t = v_t / (1 − β₂^t)
  θ_t = θ_{t−1} − η · m̂_t ⊘ (√v̂_t + ε)
  defaults β₁ = 0.9, β₂ = 0.999, ε = 1e-8
  (Step 1's m is Step 4's m scaled by 1/(1−β₁); only η's meaning changes)

STEP 5 · ADAMW = decay decoupled from the adaptive scale
  θ_t = θ_{t−1} − η · ( m̂_t ⊘ (√v̂_t + ε)  +  λ · θ_{t−1} )
  vs Adam + L2-in-the-loss, where g_t ← g_t + λθ_{t−1} and the decay then gets
  divided by √v̂_t along with everything else (FU 3)
```

**Why bias correction, derived:**

```
m_0 = 0  ⇒  m_t = (1 − β₁) · Σ_{i=1..t} β₁^(t−i) · g_i
         ⇒  E[m_t] = (1 − β₁^t) · E[g]       (for a stationary gradient)
So m_t is biased toward 0 for small t; dividing by (1 − β₁^t) removes it.
The same algebra gives E[v_t] = (1 − β₂^t) · E[g²].

WHAT GOES WRONG WITHOUT IT (constant gradient g, default betas):
  step size relative to η = (1 − β₁^t) / √(1 − β₂^t)
    t = 1:     0.1 / √0.001 = 3.16×
    t ≈ 12:    ≈ 6.6×      ← the peak
    t = 1,000: ≈ 1.26×
  v warms up more slowly than m (β₂ is closer to 1), so the error is in the
  dangerous direction: early steps are TOO BIG. With correction, step 1 is
  exactly η · sign(g).
```

### 6.2 The gradient of the ranking loss with respect to the logit — *derive it on the board*

```
ℓ(z, y) = −[ y · log σ(z) + (1 − y) · log(1 − σ(z)) ],    σ(z) = 1/(1 + e^(−z))

1.  σ'(z) = σ(z) · (1 − σ(z))
2.  ∂/∂z [ −y · log σ(z) ]            = −y · σ'(z)/σ(z)          = −y · (1 − σ(z))
3.  ∂/∂z [ −(1−y) · log(1 − σ(z)) ]   = (1−y) · σ'(z)/(1 − σ(z)) = (1 − y) · σ(z)
4.  sum:  −y + y·σ + σ − y·σ   ⇒    ∂ℓ/∂z = σ(z) − y = p − y

THE SAME RESULT, FOR EVERY VARIANT IN §5:
  soft label y ∈ [0,1] (watch quantile):   ∂ℓ/∂z = p − y       (same algebra)
  weighted LR, weight w (T on positives):  ∂ℓ/∂z = w · (p − y)
  IPS click head, target y/κ_pos:          ∂ℓ/∂z = p − y/κ_pos
  shallow tower, z = z_main + z_bias:      ∂ℓ/∂z_main = ∂ℓ/∂z_bias = p − y
  5-class survey softmax, π = softmax(z):  ∂ℓ/∂z_j = π_j − 1[j = label]
  retrieval, P_ij = softmax_j(s_ij − log Q_j):
                                           ∂L_i/∂s_ij = (ρ_i/B) · (P_ij − 1[i = j])
      → ∂L_i/∂q_i = (ρ_i/(Bτ)) · ( Σ_j P_ij · e_vj − e_vi ):
        pulled toward the positive, pushed from each negative in proportion
        to how confusable it is (small τ → hard negatives dominate)

WHY IT MATTERS:
  · |p − y| ≤ 1 per example: bounded and well-scaled. Compare MSE on raw
    seconds, ∂/∂ŷ ½(ŷ − T)² = ŷ − T, which is ~10⁴ for a three-hour video,
    another reason the watch head is a quantile with BCE
  · INTO THE SHARED LAYERS h:  ∂L/∂h = Σ_k α_k Σ_i m_ik · (p_ik − y_ik) · ∂z_ik/∂h
    the heads push h in different directions (click up, dismiss up), which is
    why MMoE gives each head its own gate
```

### 6.3 Sparse vs dense parameters — *"sparse and dense parameters"* `F3b`

| | **Embedding tables (sparse)** | **Dense layers** (towers, MMoE experts, gates, heads) |
|---|---|---|
| Size | ~3×10⁸ rows × 64 dims ≈ **77 GB** fp32 | ~10⁷–10⁸ parameters |
| Touched per step | Only the rows in the batch (~10⁵ of 3×10⁸) | Every parameter |
| **Optimizer** | **Row-wise Adagrad** | **AdamW** |
| Optimizer memory | **1 float per row: +1.2 GB** | 2 floats per parameter (m, v) |
| Adam's cost here | m and v per coordinate: **+154 GB** | Affordable |
| Learning rate | Higher (~0.01–0.05) | ~1e-3 peak, warm-up, then schedule |
| Weight decay | None, or small L2 on **touched rows only** | Decoupled, not on biases or norm gains |

```
ROW-WISE ADAGRAD, for row r of a table, gradient g_r ∈ ℝ^d (only if r is in the batch):
  G_r ← G_r + (1/d) · ‖g_r‖²                 ONE accumulator per row
  e_r ← e_r − η · g_r / (√G_r + ε)
  (the default in DLRM-style trainers)
```

**Why not Adam on the embeddings (say all three reasons):**
1. **Memory:** two moments per coordinate triples the table: +154 GB vs +1.2 GB.
2. **Sparsity semantics:** dense Adam keeps moving a row that **wasn't in the batch**, because m̂ is non-zero for tens of steps after the row's last appearance. That turns a sparse update into a dense one, so the cost is O(table) per step. "Lazy" Adam updates m and v only when the row appears, so its moment decay runs on occurrences rather than steps, and a rare row resumes with stale momentum.
3. **Rare IDs:** Adagrad's per-row accumulator gives a tail video **large steps** on the few examples it gets, and a head video small, stable ones. That is the behaviour we want for a power-law catalogue.

**Why AdamW on dense layers:** every parameter gets a gradient every step, so the moments are well-defined. Its per-coordinate scale handles layers whose gradients differ by orders of magnitude. Bias correction makes the start well-behaved, and decoupled decay regularises every weight at the same rate.

*(A wide, very sparse linear part, if there is one: **FTRL-Proximal**, whose L1 term produces exact zeros. That's the classic ads choice, G21.)*

### 6.4 Practical settings `F3b`

| Setting | Choice | Why |
|---|---|---|
| **Batch size** | Large (4k–16k) | More in-batch negatives for retrieval; scale the learning rate with care (≈ √B for Adam) and warm up |
| **Warm-up** | Linear over the first ~1–2k steps, dense AdamW | v̂ is noisy early even with bias correction |
| **Schedule, training from scratch** | Warm-up → cosine: `η_t = η_min + ½(η_max − η_min)(1 + cos(π·(t − W)/(T − W)))` | Anneals into a minimum |
| **Schedule, continuous daily training** | Warm-up → **constant** | A decaying schedule reaches ~0 and the model stops tracking drift |
| **Gradient clipping** | Global norm on dense gradients: `g ← g · min(1, c/‖g‖₂)`, c set from logged grad-norm percentiles | Survives the rare huge batch (a viral video, an IPS weight); κ_min clipping on IPS |
| **Weight decay** | AdamW λ ~1e-4 to 1e-2 on dense weights; **not** on biases, norm gains or embeddings | Decay on a rarely-seen row erases what little it learned |
| **Adagrad accumulator** | Initialise G₀ ≈ 0.1; in continuous training, **cap or decay** it | Otherwise a year of accumulation freezes head rows (FU 10) |
| **Precision** | bf16 forward and backward, **fp32 master weights and optimizer state**; int8 for the serving index | Moments in bf16 lose small updates |
| **Monitors** | Loss per head, grad norm, update-to-weight ratio (~1e-3), NaN guard, calibration per head | Catches divergence before the A/B does |

---

## 7 · EVAL, ROLLOUT, FEEDBACK LOOPS *(min 44–48)* — probe **(i)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE SCREENS (time-split: train to day N, test on day N+1)
  retrieval   recall@k of the actual next watch · SLICED by popularity decile
              and for fresh videos separately (an average hides the collapse)
  ranker      per-head AUC · normalised entropy (BCE ÷ entropy of the base rate)
              · calibration (predicted/observed per head, per surface)
  slate       seed-duplicate rate · intra-list diversity · channel concentration
  counterfactual  SNIPS on the exploration slice with logged propensities

⭐ OFFLINE METRICS THAT PREDICT ONLINE: keep an OFFLINE–ONLINE LEDGER.
  For every past launch, record the offline deltas and the A/B outcome.
  A metric earns its job by having predicted the sign of past A/Bs; one that
  didn't is a debugging aid, not a gate.

SHADOW      new model scores live traffic, nothing shown: latency, calibration,
            overlap with production, borderline share
A/B         randomise by USER (watch-next shapes the whole session) · 1% → 5% → 50%
            · ≥ 2 weeks (novelty, weekly seasonality) · CUPED (pre-period
            covariates) for heavy-tailed watch time
            PRIMARY: satisfied watch time, survey satisfaction, 7-day return
            GUARDRAILS: regret, dismiss, abandon, borderline, new-creator reach,
            autoplay-chain drift, p95 latency
INTERLEAVING for ranker-vs-ranker: faster and lower variance than a full A/B
LONG RUN    a ~1% holdback on the old system for months: return rate, satisfaction
```

**Success** = satisfied watch time and survey satisfaction up, 7-day return not down, every guardrail held. Raw watch time alone is **not** a success criterion. `I2b`

**Monitoring for feedback loops** `F1c`:

| Monitor | What it catches |
|---|---|
| Impression concentration (top-1% share, Gini) | Popularity collapse |
| Fresh and new-creator impression share | The cold-start door closing |
| Main model's accuracy **on the exploration slice** | The model only being right about what it chose to show |
| **Autoplay-chain drift**: topic distance and borderline score after k hops from the seed | Rabbit holes: each hop locally optimal, the chain harmful |
| Per-source contribution to the final list | A source silently returning nothing |
| Embedding drift of head videos | A video whose meaning changed (a re-upload, a news event) |

**Bias sources and mitigations** `F1c`:

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | Slot 1 is clicked regardless of quality | Shallow tower; IPS on the randomised slice; unexamined dropped |
| **Selection / exposure** | Only shown videos have labels | Exploration slice with propensities; next-watches from search kept |
| **Sampling** | In-batch negatives ∝ popularity | logQ correction (sampling); mixed negatives (tail) |
| **Popularity** | Positives are popularity-heavy by construction | Tune the correction strength (FU 1); popularity-sliced recall; quotas |
| **Duration** | Long videos accrue seconds | Duration-quantile label |
| **Autoplay default** | Autoplay "plays" without a choice | Its own label (continued, not cancelled) and its own head |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 48–51)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** In a math-heavy interview this gets three minutes, so say it as nine one-liners. Every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 150 ms for the list; autoplay final by T − 10 s; cost per 1,000 page loads, with the ranker as the cost line | p95 per surface · $/1k requests |
| **Reliability** `I1b` | Per-source timeouts with quota refill; seed-only fallback; cached per-seed list; never an empty panel | Fallback rate · empty-panel rate = 0 |
| **Usability** `I1c` | ~20 diverse cards + one autoplay pick; a reason per card; *Not interested*, *Don't recommend channel* and autoplay off act on the next load | Dismiss rate · autoplay cancel rate |
| **Scalability** `I1d` | Precomputed item vectors; sharded ANN + streaming fresh tier; per-seed caching for head seeds; row-wise Adagrad keeps optimizer state at +1.2 GB, not +154 GB | QPS headroom · index freshness |
| **Privacy** `I1e` | Watch history reveals health, politics, sexuality → no sensitive inference in reasons; paused history and incognito → seed-only; **the user tower reads history at request time** instead of a learned user-ID embedding, so deleting history takes effect immediately and there's no per-user parameter to scrub; no personalised data on made-for-kids content | Deletion latency · sensitive-reason audits |
| **Safety** `I1f` | Borderline content demoted by a **policy model**, not learned around; autoplay held to a stricter bar; self-harm and kids rules; autoplay-chain drift monitored | Borderline impressions · chain-drift alerts |
| **Security** `I1h` | OAuth, rate limits, request validation, **and beyond the gateway**: ⚠ **co-watch poisoning** (view farms watching A then B to plant B in A's list) → invalid-traffic filtering **before** training, trust-weighted co-watch, velocity anomalies on video pairs; ⚠ **lookalike uploads** copying a viral video's title and thumbnail to ride content similarity → near-duplicate detection, channel-trust priors | Invalid traffic caught · lookalikes blocked |
| **Ethics / bias** `I1i` | New-creator reach funded through exploration; exposure audited by creator region and language; per-segment calibration; no sensitive attributes as features | Exposure share by creator group · ECE per segment |
| **Explainability** `I1j` | A reason per card from **source provenance** ("Next in this series", "Related to this video"); the **additive value score** logs each head's contribution per impression, so "why was this picked?" has an answer; creators see which seed videos sent them traffic | % cards with a reason · per-head attribution coverage |

> ⭐ *"Watch-next is where one recommendation leads to the next, so the safety question isn't 'is this video OK?' but 'where does this chain go?'"*

---

## 9 · EXTENSION + FAILURES *(min 51–53)* `F3c` `F2a` `A5b`

**Extensions** `F3c`:
- **Generative retrieval with semantic IDs:** quantise each video's content embedding into a short code (RQ-VAE), and let a sequence model decode the next video's code from the watch history. It shares statistics across similar videos (good for the tail and new uploads), but decoding costs more than an ANN lookup, and it's **less mature at 10⁹-item, 10⁵-QPS scale**. I'd run it as **one more candidate source with a quota**, not a replacement.
- **Shorts:** a swipe is a different label (watch fraction of a 30-second video, swipe-away time); same losses, different heads and a separate surface.
- **Live streams:** no completion label and a short shelf life; the fresh tier plus a "live now" source.

**When there's no personal signal to trust: the seed-only pipeline** `A5b` `F2a`:

```
signed out · incognito · paused history · OR the user-feature fetch timed out
  → candidate sources keyed ONLY on the seed: co-watch(c), series/channel next,
    content-similar(c), trending-in-topic
  → the SAME ranker with user features masked (trained with user-feature
    dropout, so masked inputs are in-distribution)
  → the same re-rank: MMR against the seed, caps, policy, stricter autoplay
  "Less personal, still relevant: on this surface the seed carries most of the signal."
```

**Top 3 failure modes:**
1. **Popularity collapse:** the corrected loss faithfully learns P(v | x), the list converges on the same hits, and CTR approves (FU 1).
2. **Rabbit-hole drift through autoplay:** every hop locally optimal, the chain harmful; watch time rises the whole way (FU 4).
3. **Optimizer-state rot in continuous training:** Adagrad accumulators grow for a year, head rows freeze, and the model stops tracking what videos mean now (FU 10).

**Q1 ships:** the seed-keyed sources + two-tower with logQ and mixed negatives, the **MMoE ranker with negative heads and the shallow position tower**, the duration-quantile watch label, MMR with the seed in the set, and **the exploration slice with logged propensities**.
> *"Exploration and propensities in Q1, because without them every offline number I report is about videos the old system already chose to show."*

---

## 🔥 TOP 5 FOLLOW-UPS — *the first four are the prompt's own*

**1. "Your retrieval model keeps recommending the same popular videos. Which part of your loss is responsible, and how do you fix it?"** `F1c` `F2a` `F1a`
> Two terms, and it's worth being precise about which. Popularity enters through the **positives**: every training row is a logged next-watch, so popular videos are the positive in a huge share of rows. And it's **restored** by the **logQ correction**. In-batch negatives are sampled in proportion to popularity, so without the correction the optimum is log P(v|x) − log Q(v), a PMI-like score that over-penalises popular videos. With the correction, the optimum is log P(v|x), and the true next-watch distribution *is* popularity-heavy. So the correction fixes **sampling** bias; it doesn't remove **popularity**, and a model that collapses onto hits is doing exactly what the corrected loss asks. Fixes, in order: make the correction a dial, subtracting α·log Q(v) with α tuned between 0 (PMI) and 1 (true conditional), chosen on popularity-sliced recall and online new-creator reach; **mixed negatives**, uniform samples from the corpus so the tail is actually trained as negatives; **satisfaction-weighted positives** (ρ_i), so popular-but-regretted videos stop being strong positives; and outside the loss, **quotas** for the content-similar and fresh sources and an impression-concentration monitor. I'd first confirm the diagnosis with recall@k by popularity decile: if head recall is high and tail recall is near zero, it's the loss; if tail recall is fine, the collapse is happening in the ranker or the re-rank.

**2. "How would you make a video uploaded an hour ago eligible for watch-next?"** `F2c` `I1d` `A1b`
> It has no ID embedding worth anything and no co-watch, so the ID-keyed sources can't see it. Five pieces, all inside the hour. **Policy first**: transcode, then the policy classifiers, because a video that hasn't been checked shouldn't be recommendable at all. **Content embedding at upload**: the item tower takes content features (title, transcript, thumbnail, frames, channel embedding) alongside the ID embedding, and is trained with **ID dropout**, so it produces a sensible vector for an ID it has never seen. The LoRA-tuned content encoder places it near videos that viewers treat as related. **A streaming fresh tier** in the ANN index, upserted in minutes and merged at query time with the daily index, so the content-similar source keyed on the seed can retrieve it. **Ranker features that don't need history**: channel priors, the creator's typical watch quantile, upload age. And **a funded exploration quota** for fresh videos, with logged propensities: that's how it earns real labels, and near-line updates start training its ID row. Row-wise Adagrad helps here, because a brand-new row has an empty accumulator, so its first gradients take large steps. The metric is time-to-first-impression and its early satisfaction, sliced by creator size.

**3. "Why does Adam with L2 regularisation in the loss behave differently from Adam with decoupled weight decay, and when does it matter?"** `F1a` `F3b`
> With L2 in the loss, the decay term λθ is added to the gradient, so it flows into m and v and then gets **divided by √v̂** along with everything else. The effective decay on coordinate j becomes roughly ηλθ_j / √v̂_j, so parameters with a large gradient history are barely regularised, and parameters with small gradients are regularised hard. In the extreme, a parameter whose data gradient is about zero has m̂ ≈ λθ and √v̂ ≈ λ|θ|, so its update is about η·sign(θ): **λ cancels out**, and the weight is shrunk by a fixed η per step whatever λ you chose. AdamW applies the decay **outside** the adaptive step, θ ← θ − ηλθ, so every weight shrinks at the same relative rate, independent of its gradient history; that's what weight decay was supposed to mean. It matters when gradient scales are **heterogeneous**, which describes this model exactly: frequent versus rare embedding rows, and layers whose gradients differ by orders of magnitude. It also matters with large λ and long training, and with learning-rate schedules, because AdamW's decay scales with η, so the schedule anneals it too. In practice it's also why tuning λ under Adam+L2 is so fragile: λ and η interact through v̂. For the embeddings I avoid both, because decaying a rarely-seen row erases what little it learned; if anything, I use small L2 on the rows touched in the batch.

**4. "How do you keep the system from maximising watch time with ever more sensational videos?"** `I1f` `A3b` `I2b`
> By not letting watch time be the objective, at four levels. **Labels:** watch time is a duration quantile, and positives are weighted by satisfaction, so a long, regretted watch isn't a strong positive. **Heads:** negative heads (dismiss, abandon, survey regret) are in the value score with owned weights, and survey satisfaction is the only head trained on what we actually want. **Constraints, not weights:** borderline content is scored by a **policy model** and demoted or capped in re-ranking. That's a constraint engagement can't buy its way past, because any finite weight eventually loses to a large enough watch-time gain. YouTube has said publicly that demoting borderline content cut the watch time it got from recommendations by about 70% in the US. **Trajectory:** sensational drift happens over a chain, not in one pick, so autoplay holds a stricter bar and **chain drift** (topic distance and borderline score k hops from the seed) is a ship-blocking guardrail. Then the experiment: success is satisfaction and 7-day return, not watch time, and a long-term holdback tells me whether a watch-time gain was real value or a slow erosion. I'd expect, and accept, launches that lower raw watch time.

**5. "Show me why weighted logistic regression learns expected watch time."** `F1a` `A5a`
> Positives (clicked impressions) carry weight T_i, their watch time; negatives carry weight 1. For a group of identical inputs with k clicks out of N impressions, set the gradient of the weighted BCE to zero: Σ_pos T_i·(p − 1) + Σ_neg 1·(p − 0) = 0, which gives p·(ΣT + (N − k)) = ΣT, so the **odds** p/(1 − p) = ΣT / (N − k). Divide top and bottom by N: odds = E[T] / (1 − P), where E[T] is watch time per impression and P is the click rate. For small P that's ≈ E[T]·(1 + P) ≈ E[T], so serving uses e^z as expected watch time. It's elegant, and it's the YouTube 2016 design, but the derivation shows the three problems: the target is watch time **per impression**, so it multiplies in P(click) and rewards clickbait; it scales with video length; and when P isn't small, as on the top watch-next slot, the 1/(1 − P) factor inflates high-CTR items. That's why I model P(click) and a duration-quantile watch head separately and multiply at serving, where I can see both factors.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Why not just use Adam for everything?"** `F3b` `F1a`
> Three reasons, specific to embedding tables. **Memory**: Adam stores two moments per coordinate, so a 77 GB table needs another 154 GB of optimizer state; row-wise Adagrad needs one float per row, about 1.2 GB. **Sparsity**: in a batch, only ~10⁵ of 3×10⁸ rows have gradients. Dense Adam keeps moving the others for tens of steps, because their momentum is non-zero, which makes every step O(table). Lazy Adam avoids that but runs its moment decay per occurrence rather than per step, so a rare row comes back with stale momentum. **The behaviour we want**: a power-law catalogue needs big steps for rare rows and small ones for head rows, and that's Adagrad's accumulator by construction. Dense layers are the opposite case: every parameter gets a gradient every step, memory is small, and AdamW's per-coordinate scale and bias correction are what you want. So the split isn't a preference; it follows from the structure of the parameters.

**7. "Offline AUC improved, but the A/B is flat. Why?"** `I2a` `F1b`
> Four usual suspects. **The labels are the old policy's**: offline AUC is measured on impressions the current ranker chose to show, so a new model can be better at ranking those and no better at what it would show itself. I'd check SNIPS on the exploration slice. **Position leakage**: if the shallow tower isn't fully absorbing position, AUC rewards learning slot effects that vanish at serving. **Calibration**: the blend multiplies heads, so a better-ranked but mis-calibrated click head shifts the weights' effective balance and can leave the final order unchanged or worse. Predicted/observed per head tells you. **The metric doesn't predict the outcome**: AUC on clicks may simply not move satisfied watch time. That's what the offline–online ledger is for, so you can see whether this metric's sign has ever predicted an A/B. And one non-modelling reason: the gain may live in a slice too small to move the total, which a per-slice readout shows.

**8. "How do you set the serving weights that combine the heads?"** `A3b` `I2b`
> As a product decision with an owner, informed by experiments, not as a hyperparameter search on offline AUC. Mechanically: start from a principled form, P(click) × value-given-click − dismiss cost, with every head calibrated so the weights mean what they say. Then search the weights **online**: a small number of arms, or Bayesian optimisation over A/B arms, maximising the north star (satisfied watch time, 7-day return) **subject to** guardrails as constraints. Anything that must never be traded, like borderline content or kids rules, stays a constraint rather than a weight. Two things I'd insist on: the weights are versioned and reviewed like a policy, because changing w_dismiss is a decision about what users tolerate; and the additive form is kept, so per-head contributions are logged for every impression and any shift in what the system favours is visible.

**9. "Should autoplay use the same model as the list?"** `I1c` `I1f` `F1c`
> The same backbone, a different decision. Autoplay acts without a choice: the user may not even be looking. So its label is different: "continued at least 30 seconds without cancelling", not "started", and a cancel is a strong negative. It gets its **own head** on the shared MMoE, trained on autoplay rows. Its bias is different too: autoplay is effectively a position with near-certain "exposure", so it's a separate value of the surface feature in the shallow tower. And its bar is higher: a stricter satisfaction threshold, no borderline content, no near-duplicate of the seed, and stricter rules for kids. It's also the **chain**: autoplay is the step that repeats without user input, so chain drift is measured on autoplay sequences specifically, and I'd refresh the pick at T − 30 s using what happened during the current video. Sharing the backbone gives autoplay the list's data volume; separating the head, label and bar gives it the caution an unrequested action needs.

**10. "Your model trains continuously for a year with row-wise Adagrad. What goes wrong?"** `F3b` `F2a` `I1b`
> The accumulator G_r only grows, so the step for every frequently seen row shrinks toward zero. After a year, head videos' embeddings are effectively frozen. That's fine while a video's meaning is stable, and wrong when it changes: a re-uploaded clip with a new context, a news event that changes what a topic means, a channel that pivots. Dense layers have the mirror problem under a decaying schedule. Symptoms: per-row update norms collapsing for the head, head-video embedding drift going flat, and calibration on recent data slipping. Fixes: **cap** the accumulator, or make it an exponential average (row-wise RMSProp), so the step reflects recent gradients; keep the dense learning rate **constant after warm-up** in continuous training; and periodically re-initialise optimizer state for rows that have stopped moving, validated in shadow. The general lesson: **in continuous training, optimizer state is part of the model**. It ages, it should be monitored, and warm-starting it is a decision, not a default.

---

## ⭐ THE L6 LINE

> **"The video on screen anchors retrieval and ranking, and the objective is satisfied watch time, not the click: one label per rung of that ladder, one loss per label, and logQ understood as a fix for sampling bias, not popularity. Then two optimizers for two kinds of parameters: row-wise Adagrad for sparse tables, AdamW for dense layers."**

## 🧭 HOOKS — *steer by what you mention*

- *"The video on screen is the strongest signal we have"* → invites the seed-driven architecture branch
- *"logQ fixes sampling bias, not popularity"* → invites the retrieval-loss branch (**good at L6**)
- *"Negatives nobody logged"* → invites sampled softmax, mixed negatives, and why impressions aren't retrieval negatives
- *"∂ℓ/∂z = p − y, for every variant"* → invites the gradient derivation (have the four lines ready)
- *"Embeddings and dense layers are different optimisation problems"* → invites the Adagrad vs AdamW branch (**good at L6**)
- *"Optimizer state is part of the model"* → invites the continuous-training branch
- *"The safety question is where the chain goes"* → invites the autoplay and sensationalism branch

## 🆘 RECOVERY & DEFER

- **Blanked on the design?** → *"Let me go back to the seed: what does the video on screen key, and what does it feed?"* Sources, features and the MMR set regenerate the stack.
- **Blanked on a loss?** → *"What kind of label is it? Binary, bounded, ordinal or one-of-10⁹?"* The type picks the loss: BCE, soft BCE, softmax CE, sampled softmax.
- **Blanked on the optimizer?** → Build it up from SGD; each step fixes the previous one's problem.
- **Running out of time?** → *"I have five minutes. I'd rather do the bias correction and the sparse-vs-dense choice than finish the eval plan. Sound right?"* (Part 3 is a named section of this interview; the eval plan can be one sentence.)

**Name and defer:**
- "**Slate-level optimisation** (the value of the list as a whole, e.g. SlateQ) is its own model."
- "**Long-term value / RL** for the autoplay chain, optimising return rather than the next click, is a research direction I'd scope separately."
- "**Thumbnail and title personalisation** changes the click label underneath every head."
- "**The policy classifiers** for borderline content are their own system (G25)."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"The video on screen anchors retrieval and ranking; the objective is satisfied watch time."* |
| **A1b** | Latency and capacity budgets | §1, §3, FU 2 | *"7.5×10⁷ scorings/s at peak; 135 ms of a 150 ms p95; the autoplay budget is the remaining runtime."* |
| **A1c** | Degrade without lowering quality | §4 | *"A distilled pre-ranker and per-seed caching, so the heavy model scores only what it must."* |
| **A2a** | The full stack | §4 diagram | Request → features → seed-keyed sources → pre-ranker → MMoE → re-rank → logs |
| **A2b** | Model components | §5.4, §5.5 | Two-tower, MMoE with negative heads, shallow position tower, satisfaction model |
| **A2c** | Data sources | §5.1 | Impressions with position, next-watch sequences, explicit feedback, surveys, content, policy scores, exploration |
| **A2d** | Staged pipeline | §5.7 | Clean → label → train → index → gate |
| **A3a** | Top-down, unprompted | §0, §1 Q7 | *"Twenty minutes on the system, then the losses, then the optimizer."* |
| **A3b** | A principled objective for the trade-off | §4, §5.4, FU 4, 8 | *"MMR with the seed in the set; an additive value score with owned weights; borderline as a constraint."* |
| **A4a** | Clarify first, assumptions explicit | §1 | List or autoplay? Objective and guardrails? Signed in? Latency? Catalogue and freshness? |
| **A5a** | Alternatives compared | §4, FU 5 | Seed vs user anchor; weighted LR vs quantile BCE; shallow tower vs IPS; Adam vs Adagrad |
| **A5b** | A full alternative pipeline on demand | §9 | The seed-only pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.2–5.4, §6.1–6.2, FU 3, 5 | *"Sampled softmax with logQ; BCE per binary label; soft BCE for the watch quantile; ∂ℓ/∂z = p − y."* |
| **F1b** | Metrics and their trade-offs | §2, §7, FU 7 | *"Clicks are available, satisfied watch time is meaningful; an offline metric earns its job on the ledger."* |
| **F1c** | Bias sources and mitigations | §5.2, §5.5, §7, FU 1 | Position, selection, sampling, popularity, duration, autoplay default |
| **F2a** | Something fails → a fallback | §4, §9, FU 1, 10 | Seed-only pipeline; the logQ dial; accumulator caps |
| **F2b** | Latency too tight → design for it | §4 | The pre-ranker and per-seed cache, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.6, FU 2 | A LoRA-tuned content encoder on co-watch pairs; warm starts |
| **F3a** | Efficiency with a quality trade | §4 | int8 ANN vectors; a distilled pre-ranker, gated on recall |
| **F3b** | Scale levers | §6.3, §6.4, FU 6, 10 | *"Row-wise Adagrad: +1.2 GB of state instead of +154 GB; sparse updates stay sparse."* |
| **F3c** | Extension, with cost and maturity risk | §9 | Semantic-ID generative retrieval as a quota'd source; Shorts; live |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 ≤ 150 ms; autoplay by T − 10 s; cost per 1,000 page loads |
| **I1b** | Reliability | §4, §8 | Timeouts with quota refill; seed-only fallback; never an empty panel |
| **I1c** | Usability | §2, §8, FU 9 | Diverse cards, reasons, controls that act on the next load, a careful autoplay |
| **I1d** | Scalability | §3, §8, FU 2 | Sharded ANN + fresh tier; per-seed caching; small optimizer state |
| **I1e** | Privacy | §8 | History read at request time, so deletion is immediate; incognito → seed-only |
| **I1f** | Safety | §8, FU 4, 9 | Borderline as a policy constraint; a stricter autoplay bar; chain drift |
| **I1h** | Security | §8 | **Co-watch poisoning** and **lookalike uploads** |
| **I1i** | Ethics / bias | §7, §8 | New-creator reach; creator-region exposure; per-segment calibration |
| **I1j** | Explainability | §5.4, §8 | Reasons from provenance; an additive value score with per-head attribution |
| **I2a** | Offline → online | §7, FU 7 | Sliced offline screens → the ledger → shadow → user-level A/B with CUPED → holdback |
| **I2b** | The success definition | §7, FU 4, 8 | *"Satisfied watch time and satisfaction up, return not down, guardrails held; raw watch time is not success."* |

**🧭 Probe map:** (a) clarifying → §1 · (b) good next video → §2 · (c) the video on screen → §4 · (d) features, serving, latency → §3, §4 · (e) one loss per label → §5.3 · (f) negatives nobody logged → §5.2, §5.5 · (g) build it up → §6.1, §6.2 · (h) sparse and dense → §6.3, §6.4 · (i) offline → online, loops → §7.
