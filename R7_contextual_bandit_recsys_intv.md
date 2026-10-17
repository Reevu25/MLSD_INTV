# R7 · A Feedback-Driven Recommender: Contextual Bandit with Online Learning — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md)–[R6](R6_realtime_recsys_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **Expect these probes:** (a) clarifying questions: users, use cases, read/write patterns, scale, latency, availability, retention · (b) model choice, justified · (c) features and cold start · (d) feedback and the reward definition · (e) logging for learning · (f) exploration and the selection algorithm · (g) offline evaluation · (h) online experiments and monitoring · (i) non-stationarity, scale, and the three given follow-ups.
>
> **How this differs from its neighbours:** the study file [G20 feedback-driven recommender](../G20_feedback_driven_recommender.md) is the **catalogue-scale** version: millions of arms, bandits over a learned representation, the case for exploration budgets. This sheet is the **concrete selection layer**: **4 candidates in, 1 out, an update per event**, with the posterior maths, the propensity computation, the logging schema and the launch plan written down. [R6](R6_realtime_recsys_intv.md) covers the surrounding real-time stack.
>
> **Related sheets:** [Q26 continuous learning](Q26_continuous_learning_intv.md) (propensities, off-policy estimation, gates) · [G22 delayed conversion](G22_delayed_conversion_intv.md) (rewards that arrive late) · [G13 YouTube recommendation](G13_youtube_recommendation_intv.md) (exploration as ecosystem funding).

---

## 📋 THE QUESTION (verbatim)

> **Design feedback-driven recommender**
> **Design: Contextual Bandit Recommendation with Online Learning**
>
> You are designing an online learning recommendation system. At each user interaction:
> - You receive exactly 4 candidate items from an upstream candidate generator.
> - You must choose exactly 1 item to show the user.
> - You receive immediate feedback (e.g., click or dwell time).
> - The model must update online so that future selections improve over time.
>
> **Provide a design that covers:**
> - Model choice (with justification) for a contextual bandit setup.
> - Feature engineering for users, items, and context, including handling cold start.
> - Feedback handling and reward definition, including delayed/implicit signals and logging for learning.
> - Exploration–exploitation strategy and the selection algorithm.
> - Offline evaluation methodology and online experimentation/monitoring.
>
> State any minimal assumptions you need (e.g., feedback semantics, latency constraints), and make your design robust to non-stationarity and scale.
>
> **Clarifying Questions to Ask — Guidance**
> - Clarify users, core use cases, read/write patterns, scale, latency, availability, and data retention.
> - State explicit assumptions before making sizing or architecture decisions.
> - Prioritize the functional path first, then address reliability, security, observability, and rollout.
>
> **What a Strong Answer Covers — Guidance**
> - A scoped requirements summary with concrete non-goals and success metrics.
> - ML-specific data, model, evaluation, serving, and monitoring choices.
> - Reasoned trade-offs among simple and scalable designs, including bottlenecks and failure modes.
> - A validation, monitoring, migration, and launch plan appropriate for the risk level.
>
> **Follow-up Questions — Guidance**
> - What breaks first at 10x traffic or data volume?
> - How would you degrade gracefully during dependency failures?
> - What metrics and alerts would prove the design is healthy after launch?

### The question's checklist, and where each item is earned

| The question asks for | Where |
|---|---|
| Model choice, justified | §4 (the algorithm comparison), §5.4 (neural-linear Thompson sampling, with the update equations) |
| Features for users, items, context + cold start | §5.2, §5.3 |
| Reward definition, delayed / implicit signals | §2 (the reward), §5.5 (the join window, the delayed bridge) |
| Logging for learning | §5.6 (the decision record, field by field) |
| Exploration–exploitation and the selection algorithm | §5.4, §6 (sampling the 4 scores, the propensity floor) |
| Offline evaluation · online experiments · monitoring | §7 |
| Non-stationarity and scale | §5.4 (time-based decay), §3, §9, FU 1 |
| Requirements summary, non-goals, success metrics | §1, §2 |
| Simple vs scalable trade-offs, bottlenecks, failure modes | §4, §9 (the simple design), FU 1 |
| Validation, monitoring, migration, launch plan | §7 (the phased launch) |
| The three given follow-ups | FU 1, 2, 3 |

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | The guidance lists them: users, use cases, read/write patterns, scale, latency, availability, retention. Plus the two that matter most for a bandit: *"what counts as success?"* and *"what happens if the shown item is a bad pick?"* (§1) |
| **Explainability** is easy points | **Every decision is replayable**: logged features, posterior version and **the random seed** reproduce the exact choice, and each pick is labelled exploit or explore (§8) |
| **Security** beyond the gateway | **An online learner is steerable**: anyone who can generate clicks can move the posterior within minutes. Per-user contribution caps, bot filtering before the update (§8, FU 9) |
| **Troubleshoot before being asked** | The prompt names the three follow-ups, so raise them early: **10× breaks the join state first**, **the learner going down is survivable**, and **propensity-logging failure means stop exploring** (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

> The guidance says *"functional path first, then reliability, security, observability, rollout"*. The spine follows that order.

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Arms change every round → model uncertainty, not arm counts** · K = 4 makes evaluation cheap · the reward is a definition | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | The guidance's seven questions **unprompted** · assumptions · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + REWARD + METRICS** | **(d)**, (h) | One slot, one shot · **the reward, written down** · success metrics | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | (i) | 4×10⁴ decisions/s · a learner on one core · a 16 KB posterior · the join state | A1b, I1a, I1d |
| **12–20** | **ARCHITECTURE** | **(b)** | Algorithm comparison · serving / join / learner / trainer · problems raised yourself | A2a, A5a, A1c, F2b |
| **20–32** | **DATA, FEATURES, MODEL, LOGGING** | **(b) (c) (d) (e) (f)** | Features + cold start · **neural-linear Thompson sampling** · decay · the decision record | A2b, A2c, A2d, F1a, F1c, F2c |
| **32–37** | **SERVING THE DECISION** | **(f)** | Sample 4 scores, not 64 weights · **the propensity floor** · reproducible randomness | F3a, F3b, A3b, I1b |
| **37–44** | **EVALUATION, EXPERIMENTS, LAUNCH** | **(g) (h)** | Replay / IPS / DR · per-arm learners · holdouts · **the phased launch** · alerts | I2a, I2b, F1b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, as a checklist | I1a–I1j |
| **48–52** | **SCALE, FAILURES, THE SIMPLE DESIGN** | **(i)** | The degradation ladder · the ε-greedy v0 · slates and long horizons · top 3 failures | F3c, F2a, A5b |
| **52–60** | **THE THREE GIVEN FOLLOW-UPS + PROBES** | — | 10× · dependency failures · health metrics | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Three things shape this design. First, the arms change every round: the four candidates are different items each time, so a per-arm bandit that counts outcomes per item can't work. The model has to score (context, item) features and know how uncertain it is, and that uncertainty is what drives exploration. I'd use neural-linear Thompson sampling: a deep representation trained offline, with a Bayesian linear head on top that updates online, in closed form, in microseconds per event.**
>
> **Second, four candidates is a gift for evaluation. I can keep every candidate's selection probability above a few percent, so importance weights stay bounded and off-policy evaluation is low-variance. That makes logged propensities and a small uniform-random slice day-one requirements: they're cheap, and they can't be added retroactively.**
>
> **Third, the reward is a definition, not a given. A click that bounces in three seconds isn't a success, and 'no click yet' isn't a failure. So I'd define the reward from click, length-normalised dwell and hides, finalise it after a short window, and log the components separately so the definition can change. Non-stationarity is handled with time-based decay on the posterior, which also keeps exploration from dying out."**

**Then say the plan (`A3a`):** *"Clarifying questions, the reward and success metrics, capacity, the functional path (model, features, feedback, logging, selection), then evaluation and the launch plan, and finally reliability, security and scale, which is where your three follow-ups live."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked: the guidance's list, made concrete:**
1. **Users and core use case:** who sees this one item, and where? A hero slot on the home screen, a "next item" card, a notification? *(Assume a single hero slot in a consumer app. Decides the cost of a bad pick.)*
2. **What counts as success?** A click, dwell, a purchase, coming back tomorrow? *(Assume click plus dwell now, conversion and retention later.)*
3. **Read/write patterns:** per decision, one feature read and one decision write; ~1–2 feedback events per decision. *(Confirms the write volume is bigger than the read.)*
4. **Scale:** DAU and decisions per user per day? *(Below.)*
5. **Latency:** what's this layer's share of the end-to-end budget? *(Assume P99 ≤ 10 ms for the bandit layer.)*
6. **Availability:** can the slot ever be empty? *(Assume no: worst case, upstream's top candidate.)*
7. **Data retention:** how long may decision logs and feedback be kept? *(Assume 90 days raw; the uniform slice longer, pseudonymised.)*
8. **Are the 4 candidates already policy-safe and ordered by upstream?** *(Assume yes, with upstream scores provided. That makes exploring among them low-risk.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Users | ~2×10⁷ DAU · ~50 decisions per user per day |
| Decisions | ~10⁹/day ≈ 1.2×10⁴/s average, **~4×10⁴/s at peak** |
| Candidates | Exactly 4 per decision from upstream, policy-filtered, with upstream scores · catalogue ~10⁶ items, ~10⁴ new per day |
| Feedback | **Click: seconds. Dwell: resolved within minutes.** Hide: explicit. Conversion and next-day return: hours to days |
| Latency | **P99 ≤ 10 ms** for selection, inside an app budget of ~100 ms |
| Availability | 99.95%, with **upstream's top-1** as the floor, so the slot is never empty |
| Retention | Decision logs 90 days raw; uniform-random slice up to a year, pseudonymised; no per-user model parameters |

**FR:** choose 1 of 4 per request · update online from feedback · **log every decision with propensities** · keep a uniform-random slice · handle new users and new items · per-surface exploration settings · a kill switch back to upstream's top-1.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Selection latency** | **P99 ≤ 10 ms** |
| **Feedback → posterior** | **≤ 1 min** from window close to serving |
| **Propensity logging completeness** | **100%**, or exploration stops (§9) |
| **Minimum propensity** | **≥ 2%** for every candidate (bounded importance weights) |
| Availability | 99.95%; fallback never leaves the slot empty |

**Scope / NON-GOALS:**
- ❌ **Per-item (tabular) bandits.** Position: the arms change every round, and most items appear too rarely to learn from their own counts.
- ❌ **Full reinforcement learning** over sessions. Contextual bandits are the right level; long horizons enter through the reward, not through credit assignment (§9 extension).
- ❌ **Re-doing candidate generation.** But I'll negotiate one thing with upstream: a **fresh-item slot** among the 4, because a bandit can only explore what it's given (§5.3).
- ❌ Choosing more than one item (a slate): an extension (§9).

---

## 2 · PRODUCT + REWARD + METRICS *(min 6–9)* — probes **(d)**, (h) `I1c` `F1b` `I2b`

**One slot, one shot:**

```
upstream → [ A ] [ B ] [ C ] [ D ]   4 plausible, policy-safe items, with upstream scores
                    │
          BANDIT: choose 1, mostly the best bet, sometimes the most informative
                    ▼
            ┌───────────────┐
            │   [ C ]       │   ← the only item the user sees
            └───────────────┘
   feedback: click? dwell? hide?  → reward → posterior update → the next decision
⚠ a bad pick wastes the WHOLE impression: exploration is bounded and targeted
```

**The reward, written down:**

```
r = −1                  the user HIDES or reports the shown item
  =  0                  no click by the time the window closes, OR a click that
                        BOUNCES (dwell < 5 s: the item didn't deliver)
  = 0.5 + 0.5 · q       a click with dwell ≥ 5 s, where
                        q = the dwell's QUANTILE among items of similar length ∈ [0, 1]
                        (length-normalised: a long item doesn't win just by being long)

r ∈ [−1, 1], FINALISED when the window closes (§5.5), and only then learned from
⚠ the weights (−1, 0.5, 0.5) are a product decision: LOG THE COMPONENTS
  (click, dwell, hide), not just r, so the definition can change and be
  re-evaluated offline on old logs
LATER (batch, not online): conversion and next-day return join through the
  delayed bridge (§5.5)
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Mean reward per decision** vs **upstream-top-1** (the bandit's added value) + next-day return | Is the selection layer worth having? |
| Components | CTR · bounce rate · length-normalised dwell · hide rate | Read together; the reward is their blend |
| Learning health | **Regret proxy** on the uniform slice · calibration of predicted reward · **calibration of uncertainty** | Is the model learning and honest? |
| Exploration | Share of exploratory picks · **min propensity** · catalogue coverage · fresh-item time-to-first-impression | Is exploration bounded and doing its job? |
| **Guardrails** | Hide rate · P99 latency · fallback rate · logging completeness | Ship blockers |

> **Say:** *"The reward is click plus length-normalised dwell, minus hides. I'd compare it against simply showing upstream's top candidate, because that's what this layer must beat to justify its existence."*

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — probe (i) `A1b` `I1a` `I1d`

```
DECISIONS      2×10⁷ DAU × 50 = 10⁹/day ≈ 1.2×10⁴/s avg, ~4×10⁴/s peak
FEEDBACK       ~1.5 events per decision → ~6×10⁴ events/s at peak

SERVING, per decision (P99 ≤ 10 ms):
  user + context features: one KV read                       ~3 ms
  item embeddings for 4: IN-PROCESS cache                     ~0 ms
  φ(x, a) for 4: context tower once + a small interaction
    MLP × 4 (item towers precomputed)                        ~1 ms
  sample 4 scores + propensities (§6)                       < 0.1 ms
  log the decision (async, durable buffer)                   ~0.5 ms

THE LEARNER, online head with d = 64:
  rank-1 update ≈ 2·d² = 8×10³ FLOPs → 4×10⁴/s × 8×10³ ≈ 3.3×10⁸ FLOP/s
  → ONE CPU CORE; mini-batched as B += ZᵀZ it's a BLAS call
THE POSTERIOR:  μ (64 floats) + a 64×64 Cholesky factor ≈ 16.6 KB
  → broadcast to every serving pod every ~10 s; a trivial payload

THE JOIN (decision ↔ feedback within a 5-min window):
  open decisions = 4×10⁴/s × 300 s = 1.2×10⁷
  × 2 KB (full features)   ≈ 24 GB of stream state   ← the first thing to break at 10× (FU 1)
  × 200 B (features by REFERENCE: a snapshot ID)      ≈ 2.4 GB

LOGS   10⁹ decisions × ~2 KB (context + 4 candidates' features + propensities)
       ≈ 2 TB/day raw → ~0.3 TB/day compressed columnar → ~27 TB at 90 days
```

---

## 4 · ARCHITECTURE *(min 12–20)* — probe **(b)** `A2a` `A5a` `A1c` `F2b`

### The decision first: which bandit `A5a`

| Option | How it chooses | Online update | Verdict for **4 changing candidates** |
|---|---|---|---|
| **Per-item (tabular) Beta-Bernoulli TS** | A posterior per item | Count | ❌ Arms change every round; most items are seen too rarely |
| **ε-greedy over a logistic regression** | Best predicted, or uniform with prob ε | SGD | ✅ **The simple v0** (§9): easy propensities. ❌ Explores blindly: as often among confidently-bad as among uncertain |
| **LinUCB** (shared linear model) | Mean + α·uncertainty bonus | Closed form | Deterministic, so propensities are 0/1 unless randomised; α is hard to calibrate |
| **Linear Thompson sampling / Bayesian logistic** | Sample a posterior, act greedily on the sample | Closed form (or Laplace) | Explores **in proportion to the chance each candidate is best**. Linear in raw features is too weak |
| ⭐ **Neural-linear Thompson sampling** | TS on a Bayesian linear head over a **deep representation** φ(x, a) | Head: closed form per event; φ: retrained daily | **The choice**: deep features, calibrated uncertainty, microsecond updates. A robust performer in published deep-bandit comparisons |
| Bootstrapped NN ensemble TS | Sample one of N heads | SGD on each head | Good uncertainty, but N× serving and training cost; per-event updates of N networks |
| Full NeuralUCB / NeuralTS | Uncertainty from the network's gradients | Per-event gradient + covariance in parameter space | Too expensive per event at 4×10⁴/s |

> **Say:** *"Neural-linear splits the problem by timescale. The representation learns slowly and offline, where it's safe and gated; the linear head learns per event, where closed-form updates are cheap and its posterior gives Thompson sampling for free."*

### The system — *draw this in 90 seconds* `A2a`

```
══ ONLINE · per decision (P99 ≤ 10 ms) ══════════════════════════════════════════
 upstream: 4 candidates + scores ──►
┌─ 1. FEATURES ──────────────────────────────────────────────────
│ TASK: one KV read (user, session) · item embeddings in-process ·
│       context from the request
│ OUTPUT: z_a = φ(x, a) for a = 1..4   (d = 64)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. SELECT ────────────────────────────────────────────────────
│ TASK: sample the 4 scores from the cached posterior (§6) → argmax;
│       mix in the uniform FLOOR; compute all 4 PROPENSITIES
│ OUTPUT: the chosen item · p(a|x) for all 4 · explore/exploit flag · seed
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. LOG ───────────────────────────────────────────────────────
│ TASK: write the DECISION RECORD (§5.6) to a durable local buffer → event bus
│ OUTPUT: decision_id returned with the item (the client echoes it in feedback)
└────────────────────────────────────────────────────────────────

══ NEARLINE (seconds–minutes) ═══════════════════════════════════════════════════
 EVENT BUS: decisions + feedback, partitioned by decision_id
   ─► JOIN (window 5 min, or session end) → reward finalised from COMPONENTS
   ─► bot / fraud filter · per-user contribution cap
   ─► LEARNER: decayed sufficient statistics (B, f) → μ, Cholesky factor
   ─► VALIDATE (finite, well-conditioned, sane mean) → PUBLISH every ~10 s
   ─► every pod swaps in the new posterior (version-tagged)

══ OFFLINE (daily) ══════════════════════════════════════════════════════════════
 decision logs + delayed outcomes (conversion, next-day return)
   → retrain the REPRESENTATION φ (warm-start; IPS-weighted on the uniform slice)
   → OFF-POLICY EVALUATION (replay / DR) vs the current policy → GATE
   → re-fit the head in the new φ space by replaying recent hours (FU 10)
   → shadow → swap φ and head TOGETHER (one version)
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **10× breaks the join first** *(given FU 1)*: *"The decision–feedback join holds every open decision for five minutes. That's 24 GB of state today and 240 GB at 10×, so features are stored by reference, not by value."*
- **The learner going down is survivable** *(given FU 2)*: *"Pods keep sampling from the last valid posterior; it's stale, not wrong. The decisions are durably logged, so the learner replays when it returns."*
- **No logging means no exploring** *(given FU 2)*: *"If the decision log can't record propensities, exploration is pure cost with no learning. The selector drops to greedy until logging recovers."*
- **'No click yet' isn't a negative** `F1c`: *"A decision is learned from only when its window closes; updating at once would teach the model that everything fails."*
- **The latency squeeze** `F2b` `A1c`: *"If φ for four candidates doesn't fit, item towers are precomputed and only the context tower runs per request. The bandit maths itself is microseconds."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Neural-linear Thompson sampling** | "Slow, safe representation learning; fast, closed-form head learning" | Uncertainty only in the last layer; φ's own uncertainty is ignored |
| **2** | **A 2% propensity floor** | "Bounded importance weights and exploration that never fully stops" | ~8% of picks are uniform among 4: a small, bounded cost |
| **3** | **Reward finalised after a window** | "No click yet isn't a failure" | Updates lag by up to five minutes |
| **4** | **Time-based decay** | "Old evidence fades on a clock, so uncertainty never collapses" | Some stationary signal is forgotten; the half-life is tuned on drift |

---

## 5 · DATA, FEATURES, MODEL, LOGGING *(min 20–32)* — probes **(b) (c) (d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 Data sources `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Decision records** (§5.6) | Context, all 4 candidates, propensities, the choice | Must be complete: an unlogged decision is unlearnable |
| **Feedback events** | Click, dwell, hide, with decision_id | Late, duplicated, out of order; bots |
| **User and session features** (feature store) | History, affinities, session state | Freshness; must match what serving read |
| **Item features + content embeddings** | The only signal a new item has | Creator-written metadata can be gamed |
| **Upstream scores** | The generator's opinion of each candidate | A strong feature, and a dependency (FU 7) |
| **Delayed outcomes** | Conversion, next-day return | Hours to days late |
| **Uniform-random slice** (1% of decisions) | Unbiased rewards for all 4 positions → OPE gold standard | Small and precious; keep it longer |

### 5.2 Features for users, items and context (c)

| Group | Features |
|---|---|
| **User** | Long-term embedding (daily) · category affinities (decayed counts) · tenure · activity level · **recent clicks and hides** (streaming) |
| **Item** | Content embedding (text, image) · category · creator quality · age · length · **smoothed recent CTR**: (clicks + α)/(impressions + α + β), prior from the category |
| **Context** | Time of day, day of week · device · surface · session depth · network · locale |
| **User × item** | Affinity of the user to the item's category · similarity to the last-clicked items · **times this user was shown this item recently** (fatigue) |
| **Candidate-set** | **Upstream score and upstream rank** · the item's score relative to the other three |

```
φ(x, a) = MLP( [ u(x) ⊙ v(a) ,  u(x) ,  v(a) ,  cross features ,  upstream score/rank ] ) ∈ ℝ⁶⁴
  u(x) context tower (once per request) · v(a) item tower (PRECOMPUTED per item)
  → only a small MLP runs four times per request
  · initialised from the UPSTREAM ranker's towers, fine-tuned on bandit logs (F2c)
```

### 5.3 Cold start (c) `F2a` `F1c`

| Case | What carries it | How exploration treats it |
|---|---|---|
| **New item** | Content embedding + category + creator priors → v(a) is meaningful on day 0; recent-CTR features default with a "missing" flag | **Automatically explored more**: its z_a points in a less-observed direction, so `z_aᵀ B⁻¹ z_a` (the posterior variance) is larger and Thompson sampling picks it more often. No hand-made novelty bonus |
| **New item that never reaches the 4** | Nothing: the bandit can only explore what upstream gives it | ⚠ **Negotiate a fresh-item slot**: in some fraction of requests, one of the 4 comes from a fresh-item pool |
| **New user** | Context (device, locale, time, **referrer**) + session events so far; the global model generalises from similar contexts | ⚠ **Cap exploration in the first session**: uncertainty is highest exactly where a bad pick costs the most (they haven't decided to stay). Scale v down for new users and lean on upstream's order |
| **New surface or context** | Surface is a feature; the shared head transfers | Separate exploration scale per surface |

> *"There's one bandit for everyone, not one per user. Per-user bandits never get enough events to learn; a shared model learns from everyone and personalises through features."*

### 5.4 The model and the selection algorithm (b) (f) `A2b` `F1a`

```
NOTATION
  z_a = φ(x, a) ∈ ℝ^d     features of candidate a in context x (d = 64)
  r ∈ [−1, 1]             the finalised reward (§2)
  w ∈ ℝ^d                 the head's weights: expected reward ≈ wᵀ z
  λ                       prior precision (ridge)       v   exploration scale
  γ = 2^(−Δt/H)           decay over elapsed time Δt, with half-life H

THE POSTERIOR (Bayesian linear regression, decayed):
  B = λ·I + Σ_s γ_s · z_s z_sᵀ          precision
  f =       Σ_s γ_s · r_s · z_s
  μ = B⁻¹ f                              posterior mean
  w ~ N( μ, v² · B⁻¹ )                   what Thompson sampling draws from

ONLINE UPDATE, per mini-batch (Z = the batch's features, r = their rewards):
  B ← γ · (B − λI) + λI + ZᵀZ             decay the evidence, never the prior
  f ← γ · f + Zᵀ r
  re-factorise B (Cholesky of a 64×64: microseconds) → μ, and S with S Sᵀ = B⁻¹
  (no decay, single event: Sherman–Morrison,
     B⁻¹ ← B⁻¹ − (B⁻¹ z zᵀ B⁻¹) / (1 + zᵀ B⁻¹ z))

SELECTION, THOMPSON SAMPLING over 4 candidates:
  sample w̃ = μ + v · S · ε,  ε ~ N(0, I)          one draw of the world
  a* = argmax_{a ∈ 1..4}  w̃ᵀ z_a                  act greedily on the draw
  → a candidate is chosen with the probability that it is the BEST, given what
    we know: confident losers almost never, plausible winners often

FOR A PURE CLICK REWARD: Bayesian logistic regression with a diagonal Laplace
  approximation (as in Chapelle & Li's 2011 display-ads study), updated per
  mini-batch. The bounded blended reward above is well served by the Gaussian head.
```

**Why this fits:** the head's update is exact and cheap; uncertainty is calibrated by construction for a linear model; Thompson sampling needs **no exploration rate to tune** beyond the scale v; and the decayed statistics are sums, so they **shard** (FU 1).

**Non-stationarity: three layers** `F1c`:
1. **Decay by time, not by count:** half-life H (e.g. hours to a day), tuned by measuring how fast a posterior fit on older data loses predictive likelihood on recent data. Decay keeps `B` from growing without bound, so **uncertainty never collapses** and exploration never dies.
2. **Fast-moving features:** recent CTR, trending and time features let the model track drift **without** waiting for the weights.
3. **Change detection:** if predictive log-likelihood on the last hour drops sharply (a launch, an outage, a holiday), **inflate the covariance** (scale B down) so the bandit re-explores, instead of trusting stale certainty.

### 5.5 Feedback handling: windows, delays, implicit signals (d) `F1c`

```
THE JOIN: decisions and feedback keyed by decision_id
  window W = 5 min, or earlier on session end / app backgrounded
  · click: usually in seconds · dwell: ends at the next event or backgrounding
  · DEDUPE by event ID; drop feedback for unknown decision_ids
  · reward computed from COMPONENTS at window close → learner
  ⚠ an impression with no click is labelled 0 only AT CLOSE: "not yet" ≠ "no"

IMPLICIT SIGNALS: bounce (< 5 s) → 0, not success · hide → −1 · scroll-past
  without an impression event → NOT a decision (dropped: the item wasn't seen)

DELAYED SIGNALS (conversion, next-day return): NOT in the online update
  · joined in the daily batch, used to retrain φ and to re-weight the reward
  · the bridge: estimate how often each fast reward level led to conversion;
    if that relationship drifts, the online reward is re-weighted (G22, G15)
```

### 5.6 Logging for learning: the decision record (e) `A2d`

| Field | Why it's there |
|---|---|
| `decision_id`, timestamp, surface | The join key; event time |
| User and context **features as served** (or a snapshot reference) | Train and evaluate on exactly what the selector saw |
| **All 4 candidates**: IDs, features, upstream scores and ranks | Off-policy evaluation needs the **unchosen** candidates too |
| **Propensities for all 4**, not just the chosen one | Evaluating any other policy needs p for every action |
| Chosen item · explore/exploit flag (did the draw's argmax differ from the mean's?) | Attribution; exploration accounting |
| **Posterior version, φ version, RNG seed** | Exact replay of the decision |
| Posterior **mean score and variance** per candidate | Calibration of reward and of uncertainty |
| Reward **components** (click, dwell, hide; later conversion) + the reward-definition version | Change the reward later and re-evaluate old logs |
| Fallback tier, if any | So degraded decisions aren't mistaken for the policy's |

> ⭐ *"Propensities can't be backfilled. A year of logs without them is a year from which no counterfactual question can ever be answered. So it's a day-one field, and the system refuses to explore without it."*

### 5.7 The staged pipeline `A2d`

```
STAGE 1 · LOG       decision records (durable buffer → event bus) · feedback events
STAGE 2 · JOIN      window close → reward from components · bot filter · contribution caps
STAGE 3 · LEARN     decayed (B, f) → μ, S → validate → publish every ~10 s
STAGE 4 · RETRAIN   daily φ (warm-start, IPS-weighted) + delayed outcomes
STAGE 5 · GATE      off-policy evaluation vs current → shadow → swap φ + head together
STAGE 6 · MONITOR   the health panel (§7) · auto-fallback on failures (§9)
```

---

## 6 · SERVING THE DECISION *(min 32–37)* — probe **(f)** `F3a` `F3b` `A3b` `I1b`

```
⭐ SAMPLE THE 4 SCORES, NOT THE 64 WEIGHTS:
  the scores s = Z w̃ are jointly Gaussian:  s ~ N( Zμ ,  v² · (Z S)(Z S)ᵀ )
  Z S is 4×64 → a 4×4 covariance → each draw is 16 multiply-adds

PROPENSITIES (Thompson sampling has no closed form for P(argmax = a)):
  draw M = 256 score vectors → P̂_TS(a) = share of draws in which a wins
  → 256 × 16 multiply-adds: microseconds

THE FLOOR (mix in uniform with ε_f = 0.08):
  with probability ε_f pick uniformly among the 4, else the TS argmax
  p(a | x) = ε_f / 4 + (1 − ε_f) · P̂_TS(a)    ≥ 0.02 for every candidate
  → importance weights ≤ 1/0.02 = 50, and exploration never fully stops
  · the floor also makes the Monte-Carlo error on tiny P̂_TS harmless

REPRODUCIBLE RANDOMNESS: the RNG seed derives from decision_id and is logged
  → any decision can be replayed exactly; a refresh within seconds reuses the
    same decision_id, so the slot doesn't flicker or double-log
```

- **The principled objective for the trade-off** `A3b`: Thompson sampling allocates picks **by the posterior probability that each candidate is best**. That's the exploit/explore trade made per decision, not by a hand-tuned rate. The two dials that remain are explicit and owned: the scale **v** (per surface, lower for new users) and the floor **ε_f**.
- **Efficiency** `F3a`: item towers precomputed; the context tower runs once per request (int8 on CPU, gated on offline loss); the sampling maths is microseconds. The quality trade is in φ's size, measured by replay.
- **Scale** `F3b`: the posterior is 16.6 KB, broadcast to every pod; serving is stateless; the learner's statistics are additive.
- **Reliability** `I1b`: pods keep the last **valid** posterior if publishing stops; a posterior that fails validation (non-finite, ill-conditioned, mean jump) is never swapped in.

---

## 7 · EVALUATION, EXPERIMENTS, LAUNCH *(min 37–44)* — probes **(g) (h)** `I2a` `I2b` `F1b` `F1c`

### Offline evaluation (g)

```
THE UNIFORM SLICE makes it easy: 1% of decisions pick uniformly among the 4,
  with p = 0.25 each.

REPLAY (Li et al. 2011): stream the uniform slice in time order; keep a decision
  only when the candidate policy picks the logged item; average those rewards.
  → unbiased, and with K = 4 a quarter of events survive
  → it can evaluate a LEARNING policy: let it update on the events it keeps,
    which is the right test for an online learner, not a frozen snapshot

IPS / SELF-NORMALISED IPS on ALL logs (thanks to the floor, weights ≤ 50):
  V̂_IPS = (1/n) Σ_i [ π(a_i|x_i) / p_i ] · r_i
DOUBLY ROBUST (what I'd actually report): a reward model r̂ plus a correction
  V̂_DR = (1/n) Σ_i [ Σ_a π(a|x_i) · r̂(x_i, a)  +  (π(a_i|x_i)/p_i) · (r_i − r̂(x_i, a_i)) ]
  → low variance from r̂, unbiased if either r̂ or the propensities are right

ALSO CHECK: effective sample size of the weights · calibration of predicted
  reward · CALIBRATION OF UNCERTAINTY (do 90% posterior intervals contain the
  outcome ~90% of the time, on the uniform slice?) · regret vs the best of 4
  (estimated with r̂ on the uniform slice) · tuning v, H, ε_f by replay
```

### Online experiments (h)

- **User-level randomisation.** Arms are *policies*, and **each arm runs its own learner on its own arm's feedback**; a shared learner would let control learn from treatment's traffic (R6 FU 8).
- **Three standing holdouts:** the **uniform slice** (1%: the evaluation data), **upstream-top-1** (1%: the bandit's added value), and a **greedy, no-exploration holdout** (1–2%, for months: exploration's long-horizon payoff, because a short test always says "explore less", as G20 explains).
- Primary: reward per decision, next-day return. Guardrails: hide rate, bounce, P99 latency, fallback rate, logging completeness. CUPED on per-user reward; ≥ 1–2 weeks for novelty.

### The validation, migration and launch plan — *risk-appropriate* `I2a`

> **The risk level:** moderate. It's one slot, but the 4 candidates are already policy-filtered, so the worst a bad pick can do is show a weaker *acceptable* item. That allows a fast ramp with automatic rollback, not a months-long programme.

```
PHASE 0 · INSTRUMENT (weeks 1–2)   today's system shows upstream's top-1:
  a deterministic policy with propensity 1, which is USELESS for off-policy
  evaluation. So FIRST: decision IDs, full decision records, and the 1% UNIFORM
  SLICE in the CURRENT system → start collecting evaluable data before the
  bandit exists
PHASE 1 · OFFLINE   train φ + head on the logs; replay and DR vs upstream-top-1
PHASE 2 · SHADOW    the bandit picks and logs propensities live but doesn't show:
                    latency, propensity spread, agreement with upstream
PHASE 3 · RAMP      1% → 5% → 25% → 50%, auto-rollback on any guardrail;
                    the kill switch is a flag back to upstream-top-1
PHASE 4 · STEADY    100% minus the three holdouts; daily φ swaps go through
                    the same OPE gate → shadow → canary path
```

### Health: metrics and alerts that prove it's working *(given FU 3)*

| Signal | Healthy | Alert → response |
|---|---|---|
| **Reward per decision vs upstream-top-1 holdout** | Positive, stable | Below for 1 h → page; below for 1 day → fall back to upstream-top-1 |
| **Choice distribution**: entropy, share picking upstream's #1 | Between "always #1" and uniform | Collapse to one rank, or drift to uniform → investigate (FU 7) |
| **Min propensity; importance-weight ESS** | ≥ 2%; ESS stable | Floor violated → **block exploration**, page |
| **Logging completeness** (decisions with full records) | 100% | < 99.9% → exploration off (§9) |
| **Join rate; window-close lag** | > 99% joined; < 6 min | Drop → the feedback path is broken: freeze the learner |
| **Posterior age at the pods** | < 1 min | > 10 min → alert (stale but serving) |
| **Posterior health**: trace of B⁻¹, ‖μ‖, condition number | Smooth | Jumps → reject publish; investigate data |
| **Reward-model calibration; uncertainty calibration** | Predicted ≈ observed; 90% intervals cover ~90% | Off → recalibrate v, or retrain φ |
| **Hide rate on explore vs exploit picks** | Explore slightly higher, bounded | Gap widening → shrink v |
| **P99 latency; fallback rate per tier** | ≤ 10 ms; < 1% | Breach → shift to the cheaper tier |

**Bias sources and mitigations** `F1c`:

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Selection** | Only the chosen item gets a reward | Propensities for all 4; the uniform slice; DR estimation |
| **Delayed-feedback** | "No click yet" read as failure | Reward finalised at window close |
| **Upstream** | The bandit can only choose among what upstream offers | A negotiated fresh-item slot; monitor coverage |
| **Length** | Long items accumulate dwell | Dwell quantile by length |
| **Exploration burden** | Exploration's cost falls on whoever is explored on | Cap per-user exploration; lower v for new users (§8) |
| **Stale certainty** | A drifted world, a confident posterior | Time-based decay; covariance inflation on change |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Selection P99 ≤ 10 ms; posterior updates within a minute of window close; cost per 1,000 decisions | P99 · update lag · $/1k decisions |
| **Reliability** `I1b` | Last-valid posterior on pods; validated publishes; durable decision log with replay; the degradation ladder (§9) | Fallback rate per tier · posterior age |
| **Usability** `I1c` | One slot that's rarely a bad pick: bounded exploration, lower for new users; no flicker on refresh (same decision_id); hides act on the next decision | Hide rate · bounce rate |
| **Scalability** `I1d` | Additive sufficient statistics shard the learner; 16 KB posterior; join state by reference | Learner lag · join state size |
| **Privacy** `I1e` | **No per-user parameters**: the head is aggregate statistics over features; raw decision logs 90 days; the uniform slice pseudonymised; **a deletion request can be honoured exactly**: subtract the user's (decayed) contributions from (B, f), and retrain φ without them | Deletion latency · retention compliance |
| **Safety** `I1f` | Exploration happens **only among upstream's policy-filtered candidates**; no exploration on sensitive surfaces; reduced exploration for minors and new users | Policy-violating picks = 0 |
| **Security** `I1h` | Signed decision IDs, auth, rate limits, and **beyond the gateway**: ⚠ **an online learner is steerable**: bots clicking one item push its posterior in minutes → bot filtering **before** the update, **per-user contribution caps** per hour, anomaly alerts on per-item reward jumps, untrusted accounts' feedback delayed (FU 9) | Filtered events · per-item anomaly alerts |
| **Ethics / bias** `I1i` | Exploration gives new items and creators a chance, through the fresh-item slot; **the exploration burden is spread**, capped per user so no segment pays for everyone's learning; reward calibration audited per segment | Exposure for new items · per-segment calibration |
| **Explainability** `I1j` | **Every decision is replayable** from its logged features, posterior version and seed; each pick is labelled **explore or exploit**; posterior mean and variance per candidate are logged, so "why this one?" has an answer | % decisions replayable · explore/exploit attribution |

> ⭐ *"A bandit makes a small experiment out of every decision, so it owes users the same things any experiment does: bounded risk, a record of what was done, and the ability to stop."*

---

## 9 · SCALE, FAILURES, THE SIMPLE DESIGN *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

### Graceful degradation: the ladder *(given FU 2)* `F2a`

```
TIER 0 · FULL          neural-linear TS, fresh posterior, floor on, logging on
TIER 1 · STALE         learner or feedback stream down → keep sampling from the
                       last valid posterior (stale, not wrong); decisions still
                       logged; the learner replays the log when it returns
TIER 2 · THIN FEATURES feature store slow → defaults + "missing" flags (φ is
                       trained with feature dropout) and exploration scale v halved
TIER 3 · UPSTREAM      selector down or over budget → upstream's top-1,
                       logged with propensity 1 and a fallback flag
TIER 4 · NO LOGGING    the decision log can't record propensities → GREEDY:
                       exploration without a record is pure cost
EVERY TIER: the slot is never empty, and every decision says which tier made it
```

### The simple design, in full: what I'd ship first if asked for "simplest that works" `A5b`

```
v0 · ε-GREEDY OVER AN HOURLY LOGISTIC REGRESSION
  model     logistic regression on hashed crossed features (user × item category,
            context × item, upstream score), predicting P(click without bounce)
  select    with probability ε = 0.1 uniform among the 4, else the argmax
            → propensities are trivial: ε/4 + (1 − ε)·1[a = argmax]
  learn     hourly SGD warm-start from the joined logs (no per-event update)
  log       the same decision record (§5.6): it's what makes v1 possible
  evaluate  replay on the uniform part of its own ε traffic
  WHY IT'S A FAIR START: trivial propensities, easy to debug, and it produces
  exactly the data v1 needs
  WHY v1 REPLACES IT: it explores as often among confidently-bad candidates as
  among uncertain ones, updates hourly not per event, and can't model the
  interactions φ captures
```

### Extensions `F3c`

- **Slates:** if the surface grows to show 3 of the 4, the action space becomes ordered subsets (24 orderings of 3). Use a factorised slate bandit (score items, sample positions); propensities get harder and off-policy evaluation needs slate-aware estimators. The maturity risk is higher variance and subtler position bias.
- **Long-horizon reward:** optimising next-day return rather than the next click is reinforcement learning territory. The cost is credit assignment over sessions and much harder evaluation, so the first step is a **better reward** (the delayed bridge in §5.5), not an RL agent.

### Top 3 failure modes

1. **Ossification:** no decay and no floor; the posterior becomes certain, exploration stops, and the bandit becomes "always pick upstream's #1" while metrics look stable.
2. **Fake negatives:** updating before the window closes teaches the model that every pick fails, and the bandit chases noise.
3. **Steering:** an online learner without contribution caps lets a click ring pick the winner within an hour.

**Q1 ships:** **the decision record with propensities and the uniform slice in the current system** (Phase 0), then the v0 ε-greedy, then neural-linear Thompson sampling.
> *"Logging first, because it's the only part that can't be added later, and it's what lets every later version be evaluated before it reaches a user."*

---

## 🔥 TOP 5 FOLLOW-UPS — *the first three are the prompt's own*

**1. "What breaks first at 10× traffic or data volume?"** `A1b` `I1d` `F3b`
> Not the bandit maths: selection is a few microseconds of linear algebra per decision, and the learner's update is about 8,000 FLOPs per event, so 10× is about 3 GFLOP/s, still a mini-batched BLAS call on one machine. In order, what does break: **first, the decision–feedback join.** It holds every open decision for the five-minute window: about 1.2×10⁷ open decisions today and 1.2×10⁸ at 10×. At 2 KB each that's 24 GB of stream state growing to 240 GB, which overruns the stream processor's memory and checkpoint times. Fix: store features **by reference** (a snapshot ID, about 200 B), partition by decision_id, and close windows on session end. **Second, log volume**: 2 TB a day raw becomes 20 TB; log candidate features by reference too, keep full rows only for the uniform slice, and compress columnar. **Third, the learner's ingest**, if it's one process: its statistics B and f are **sums**, so shards compute partial (ΔB, Δf) and a reducer adds them. That's exact, not approximate, which is a property worth saying out loud. **Fourth, feature-store reads**, which scale horizontally. And one subtle effect: with 10× data the posterior tightens 10× faster, so exploration shrinks; that's correct if the world is stationary, which is why decay is set on a **clock**, tied to measured drift, not on an event count.

**2. "How would you degrade gracefully during dependency failures?"** `I1b` `F2a` `A1c`
> Walk the dependencies and name what each failure costs. **Learner or feedback stream down**: pods keep sampling from the last valid posterior. Recommendations go stale, not wrong, and since every decision is durably logged, the learner replays the backlog when it returns. I alert on posterior age. **A bad publish** (NaNs, an ill-conditioned matrix, a jump in the mean) is rejected by validation before any pod swaps it in. **Feature store slow**: features fall back to defaults with missing flags, which the representation saw in training through feature dropout, and I halve the exploration scale, because uncertainty estimates on thin features are less trustworthy. **The selector itself down or over budget**: upstream's top-1, logged with propensity 1 and a fallback flag, so the slot is never empty and the data isn't mistaken for the policy's. And the one people miss: **logging down**. If I can't record propensities, every exploratory pick is pure cost with nothing learned, so the selector goes **greedy** until logging recovers. I'd make the fallback rate per tier an SLO, because a system serving tier 3 to a fifth of traffic has an outage nobody declared.

**3. "What metrics and alerts would prove the design is healthy after launch?"** `I2b` `F1b` `I2a`
> Four groups, each answering a different question. **Is it adding value?** Reward per decision against the upstream-top-1 holdout, and next-day return; this is the bandit's reason to exist. **Is it learning honestly?** Calibration of predicted reward, and **calibration of uncertainty**: on the uniform slice, do the 90% posterior intervals contain the outcome about 90% of the time? If they're too narrow, the bandit under-explores; too wide, it wastes picks. **Is exploration bounded and alive?** Minimum propensity (the floor must hold), effective sample size of the importance weights, the choice distribution (entropy, the share of picks that are upstream's #1, which is neither stuck at 100% nor drifting to uniform), the hide rate on exploratory versus exploitative picks, and catalogue coverage. **Is the machinery working?** Logging completeness at 100%, join rate, window-close lag, posterior age at the pods, posterior condition number, P99 latency, and fallback rate per tier. Each has a threshold and an action, from paging to an automatic fallback; the table in §7 is the runbook.

**4. "Why Thompson sampling and not ε-greedy or UCB?"** `A5a` `F1a`
> With four plausible candidates, ε-greedy isn't absurd, and it's my v0, because its propensities are trivial and it's easy to debug. But it explores **blindly**: with probability ε it picks uniformly, so it spends as many picks on a candidate it's confident is the worst as on one it genuinely can't call. Thompson sampling picks each candidate **with the probability that it's the best**, given the posterior, so confident losers almost never get shown and close calls get resolved quickly. That's the right allocation, especially on a single slot where every wasted pick is a wasted impression. It also has no exploration rate to tune beyond a scale, and exploration shrinks naturally as evidence accumulates and grows again when decay or covariance inflation says the world moved. UCB makes a similar trade deterministically, which is a drawback here: deterministic picks have propensities of 0 or 1, which breaks off-policy evaluation unless I randomise anyway, and the bonus coefficient is hard to calibrate on a learned representation. What I give up with Thompson sampling is a closed-form propensity, which I get back by Monte Carlo over the four scores.

**5. "How do you get propensities out of Thompson sampling?"** `F1a` `A3b`
> There's no closed form for the probability that each of four correlated Gaussian scores is the largest, so I estimate it by Monte Carlo, cheaply. The four scores are jointly Gaussian, mean Zμ and covariance v²·(ZS)(ZS)ᵀ, where S is the posterior's Cholesky factor, so I sample 4-dimensional score vectors directly rather than 64-dimensional weights. That's 16 multiply-adds per draw, and 256 draws take microseconds. The share of draws each candidate wins is its Thompson-sampling probability. Then I mix in a uniform floor: with probability 0.08 pick uniformly, so every candidate's propensity is at least 2% and importance weights are capped at 50. That makes the Monte-Carlo error on tiny probabilities harmless and keeps off-policy estimates stable. The seed derives from the decision_id and is logged, so the decision and its propensities can be recomputed exactly later.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "The business cares about next-day retention, not clicks. Your reward is a click."** `F1b` `F1c`
> Then the click is a proxy, and the design should admit it. Online, the bandit needs a reward that resolves within minutes, so it keeps the fast reward: click, length-normalised dwell, hides. Offline, I join next-day return and conversions to the decision logs and measure the **bridge**: for each level of the fast reward, how often did it lead to the outcome we care about? If dwell-heavy picks predict return and quick clicks don't, the fast reward's weights change to match. That's a change I can evaluate on old logs, because I logged the components. I monitor the proxy-to-truth correlation, because it drifts. And the experiment's verdict comes from next-day return in the A/B, not from reward per decision. What I wouldn't do is put a day-delayed reward into the online update directly: a learner that waits a day isn't online, and one that guesses is learning noise.

**7. "Your bandit picks upstream's first candidate 95% of the time. Is it broken?"** `F2a` `F1c`
> Maybe not. Upstream may simply be good, and the bandit has learned that its rank-1 usually wins. Three checks. **Is the floor working?** Every candidate should still get at least 2%, so 95% on rank 1 can't hide a missing floor. **On the uniform slice, is rank 1 actually best about 95% of the time?** If rank 2 wins 30% of those, the bandit is over-trusting upstream. The usual cause is that the upstream score feature dominates φ, so the head barely uses anything else. Fixes: regularise or bucket that feature, and check **uncertainty calibration**, because an over-confident posterior stops exploring. **Has decay stopped?** A posterior that's been accumulating without forgetting becomes certain about everything; the trace of B⁻¹ trending to zero is the symptom. If the checks pass, 95% is a finding, not a failure, and the value of the layer is the 5% it gets right that upstream doesn't, measured against the upstream-top-1 holdout.

**8. "How do you evaluate, offline, a policy that learns while it runs?"** `I2a` `F1a`
> A frozen-policy estimator like IPS answers the wrong question: it scores a snapshot, but what ships is a learner whose behaviour changes with every update. The replay method handles this. On the uniform slice, stream the logged decisions in time order; whenever the candidate policy, given its **current** state, picks the same item that was logged, keep that event, give it the reward, and **let the policy update on it**; otherwise skip it. Because logging was uniform, the kept events are an unbiased sample of what the learning policy would have experienced. With four candidates, a quarter of events survive, which is efficient enough to compare learning rates, decay half-lives, exploration scales and floors. Two caveats: replay runs on the slice's volume, which is slower than reality, so learning-speed conclusions transfer imperfectly; and anything that depends on the bandit's influence on upstream's future candidates isn't captured. So replay chooses the finalists, and the A/B with per-arm learners decides.

**9. "Bots are clicking to manipulate the bandit."** `I1h` `F1c`
> An online learner is directly steerable, so this is a design problem, not an afterthought. Anyone who can generate feedback on decisions where an item was shown can push that item's direction in feature space within minutes. Defences in the update path. **Filter before learning**: bot and emulator detection, signed decision IDs so feedback can't be fabricated for decisions that never happened, and feedback only accepted for decisions actually served to that session. **Cap each user's contribution** to the sufficient statistics per hour, so no single account, or a ring of new ones, can dominate an update; untrusted or brand-new accounts' feedback is **delayed and down-weighted** until it looks organic. **Detect**: alerts on sudden per-item or per-creator reward jumps and on the posterior mean moving faster than it historically does. **Contain**: because the statistics are sums, a confirmed attack's contributions can be **subtracted** exactly, which is a real advantage of the linear head. And the daily φ retrain uses the same filtered data, so the attack doesn't leak into the representation.

**10. "φ is retrained daily. What happens to the online posterior when it changes?"** `A2d` `F2c` `I1b`
> It becomes meaningless: the head's weights and covariance live in φ's coordinate system, and a new φ is a new coordinate system. Naively starting from the prior would make the bandit re-explore from scratch, a burst of bad picks every day. So φ and the head are **one versioned unit**. Before a swap, I **re-fit the head in the new space by replaying the last several hours** of logged decisions: the records hold the raw features, so I compute new z's and rebuild (B, f) with the same decay. The new pair then goes through the gate: off-policy evaluation against the current pair, a shadow period checking that the choice distribution and propensities are sane, then a canary. Serving swaps φ and posterior **atomically**, by version, so no decision ever uses a new φ with an old head. And I watch the hours after a swap for an exploration spike, which would mean the replayed posterior was too uncertain, usually because the replay window was too short.

---

## ⭐ THE L6 LINE

> **"Four changing candidates means a shared model with honest uncertainty, not per-item counts: neural-linear Thompson sampling, with a representation that learns daily behind a gate and a head that learns per event in closed form. Every decision is logged with propensities above a floor, rewards are finalised after a window, decay runs on a clock, and every dependency failure has a defined, logged fallback."**

## 🧭 HOOKS — *steer by what you mention*

- *"The arms change every round"* → invites the model-choice branch (**good at L6**)
- *"Four candidates is a gift for evaluation"* → invites the replay / IPS / DR branch
- *"Sample the four scores, not the 64 weights"* → invites the propensity branch
- *"'No click yet' isn't a negative"* → invites the feedback-window branch
- *"Decay on a clock"* → invites the non-stationarity branch
- *"Sufficient statistics are sums"* → invites the 10× and the deletion branches (**good at L6**)
- *"No logging means no exploring"* → invites the degradation branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to one decision: four candidates in, features, a posterior draw, a pick, a log record, a reward at window close, an update."* The loop regenerates the design.
- **Running out of time?** → *"I have five minutes. I'd rather cover the three follow-ups you listed, 10×, degradation and health metrics, than the slate extension. Sound right?"*

**Name and defer:**
- "**Upstream candidate generation** is a separate system; I only ask it for a fresh-item slot."
- "**Slate bandits** if the surface shows more than one item."
- "**Long-horizon RL** over sessions, once the delayed-reward bridge has shown what the fast reward misses."
- "**Uncertainty in φ itself** (ensembles of representations) if last-layer uncertainty proves too narrow."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"Neural-linear Thompson sampling: a daily representation, a per-event closed-form head."* |
| **A1b** | Latency and capacity budgets | §1, §3, FU 1 | *"4×10⁴ decisions/s; P99 ≤ 10 ms; the learner on one core; a 16 KB posterior; 24 GB of join state."* |
| **A1c** | Degrade without lowering quality | §4, §9, FU 2 | *"Stale, not wrong: pods keep the last valid posterior."* |
| **A2a** | The full stack | §4 diagram | Features → select → log · join → learner → publish · daily φ → gate → swap |
| **A2b** | Model components | §5.2, §5.4 | φ (context and item towers), the Bayesian linear head, the reward model for DR |
| **A2c** | Data sources | §5.1 | Decision records, feedback, features, upstream scores, delayed outcomes, the uniform slice |
| **A2d** | Staged pipeline | §5.6, §5.7, FU 10 | Log → join → learn → retrain → gate → monitor; the decision record |
| **A3a** | Top-down, unprompted | §0 | *"Arms change every round; four is a gift for evaluation; the reward is a definition"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 5 | *"Thompson sampling picks each candidate with the probability it's best; v and the floor are the only dials."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Use case, success, read/write, scale, latency, availability, retention, upstream safety |
| **A5a** | Alternatives compared | §4, FU 4 | Tabular, ε-greedy, LinUCB, linear TS, neural-linear, ensembles, NeuralTS |
| **A5b** | A full alternative pipeline on demand | §9 | The ε-greedy v0, end to end |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and maths per subproblem | §5.4, §7, FU 4, 5, 8 | *"Decayed Bayesian linear posterior; sample four scores; replay, IPS and DR."* |
| **F1b** | Metrics and their trade-offs | §2, §7, FU 3, 6 | *"Reward vs upstream-top-1; calibration of uncertainty; the fast-to-slow reward bridge."* |
| **F1c** | Bias sources and mitigations | §5.4, §5.5, §7, FU 7, 9 | Selection, delayed feedback, upstream, length, exploration burden, stale certainty |
| **F2a** | Something fails → a fallback | §5.3, §9, FU 2, 7 | The degradation ladder; greedy when logging fails |
| **F2b** | Latency too tight → design for it | §4 | Precomputed item towers, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.2, FU 10 | φ initialised from the upstream ranker's towers; the head re-fit on replay |
| **F3a** | Efficiency with a quality trade | §6 | int8 context tower gated on offline loss; microsecond sampling |
| **F3b** | Scale levers | §3, §6, FU 1 | Additive statistics shard exactly; features by reference |
| **F3c** | Extension, with cost and maturity risk | §9 | Slate bandits; long-horizon RL |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | P99 ≤ 10 ms; update within a minute; cost per 1,000 decisions |
| **I1b** | Reliability | §6, §8, §9, FU 2, 10 | Validated publishes; durable logs with replay; atomic φ + head swaps |
| **I1c** | Usability | §2, §8 | Bounded exploration on a single slot; no flicker on refresh |
| **I1d** | Scalability | §3, §8, FU 1 | Sharded learner; join state by reference |
| **I1e** | Privacy | §8 | No per-user parameters; deletion by subtracting contributions |
| **I1f** | Safety | §8 | Exploration only among policy-filtered candidates |
| **I1h** | Security | §8, FU 9 | **An online learner is steerable**: caps, filters, subtraction |
| **I1i** | Ethics / bias | §8 | The exploration burden spread and capped per user |
| **I1j** | Explainability | §8 | Every decision replayable from features, versions and seed; explore/exploit labels |
| **I2a** | Offline → online | §7, FU 8 | Replay / DR → shadow → per-arm A/B → three holdouts → the phased launch |
| **I2b** | The success definition | §2, §7, FU 3 | *"Reward per decision above upstream-top-1, next-day return up, guardrails held."* |

**🧭 Probe map:** (a) clarifying → §1 · (b) model choice → §4, §5.4 · (c) features, cold start → §5.2, §5.3 · (d) feedback, reward → §2, §5.5 · (e) logging → §5.6 · (f) exploration, selection → §5.4, §6 · (g) offline evaluation → §7 · (h) experiments, monitoring → §7 · (i) non-stationarity, scale, the given follow-ups → §5.4, §9, FU 1–3.
