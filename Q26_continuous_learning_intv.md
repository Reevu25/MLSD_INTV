# Q26 · Continuous Learning from Production — **INTERVIEW SHEET**

> **Tier 3 · #29** — the retraining/monitoring prompt. Google weights MLOps heavily.
> Full study file: [`../Q26_continuous_learning.md`](../Q26_continuous_learning.md) · [diagrams](../Q26_continuous_learning_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the teams see · (c) the loop · (d) the feedback signals · (e) curating the training set · (f) the update objectives and estimators · (g) the update ladder and rollback · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Verifiable outcomes? Rollback per layer? Uninfluenced traffic? Consent to train on interactions? (§1) |
| **Explainability** is easy points | Every promotion leaves an **evidence record**; **lineage** from each model version to the data that trained it (§2, §8) |
| **Security** beyond the gateway | **Feedback poisoning** — users or bots steering the model through its own signals (the Tay lesson) (§8) |
| **Troubleshoot before being asked** | Raise **silent join loss** and **self-consumption** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A closed-loop control system, not a training pipeline** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · rollback as the SLO · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + THE FROZEN HOLDOUT** | **(b)**, (h) | ⭐ **Build it before the pipeline that feeds it** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Logging volume · join windows · label budget · rollback time per layer | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | The loop, and **where it's cut** | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Signals ranked by gameability · curation · OPE | A2b–d, F1a, F1c, F2c |
| **32–38** | **THE UPDATE LADDER + ROLLBACK** | **(g)** | ⚠ "Continuous" ≠ continuous weight updates · drift | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | The automated gate · holdback · segment erosion | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Bandits, active learning · **freeze-the-loop fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The thing to be clear about before drawing anything: this is a closed-loop control system, not a training pipeline. The model we deploy shapes the traffic that trains the next model.**
>
> **That single property causes every interesting failure here, and it means the system can improve on every metric we own while the product gets worse — because the metrics are computed on a distribution the model itself created.**
>
> **So my first design decision isn't the labeller or the trainer. It's the frozen holdout that lives outside the flywheel: a fixed evaluation set drawn from a traffic source the model does not influence, never updated by the loop, refreshed only by a deliberate human act. Without it I have no fixed point and no way to detect a slow slide — and I'd build that before I built the pipeline that feeds it."**

**Then two more commitments:**
1. **Signals ranked by gameability** — *"**Verifiable outcomes** — did the code pass CI, did the SQL execute, did the refund post — are gold, and the most valuable engineering work here is changing the **product** so it emits more of them. **Implicit signals** — retry, edit distance, escalation — are dense but confounded. **Explicit thumbs** are sparse, around 0.1–1%, and biased toward anger. I use all three and never let the cheap ones drive weight updates alone."*
2. **⚠ "Continuous" does not mean continuous weight updates** — *"I tier the update velocity by blast radius and reversibility. Each layer moves as fast as its rollback path allows, and no faster."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the teams see and the frozen holdout, the budgets, the loop and where it's cut, the signals and curation, the update ladder, the promotion gate, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What verifiable outcomes does the product already emit?** *(If few, the highest-value work is **changing the product**, not the pipeline.)*
2. **What's the rollback path for each layer?** *(It sets the update velocity — §6.)*
3. **Is there traffic the model doesn't influence?** *(If not, the frozen holdout has to be *created*, and that's the first thing to build.)*
4. **What does user consent cover?** *(Training on interactions needs consent, and deletion requests must reach training sets.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Product | An LLM assistant with several surfaces: coding help, SQL, customer support |
| Volume | ~10⁸ model decisions a day |
| Outcomes | Some verifiable (CI, query execution, refund posted); most implicit |
| Layers | Retrieval index, prompts/configs, routers, LoRA adapters, base weights |
| Consent | Interactions usable for training where users opted in; enterprise data excluded |

**FR:** collect feedback · join delayed labels · **propensity logging** · evaluate candidates against a frozen holdout · promote by tier · **roll back fast** · detect drift.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Frozen holdout** | Outside the loop, **refreshed only by deliberate human act** |
| **Rollback time** | Per layer, and it **bounds the update cadence** |
| Label-join completeness | Tracked — a silent drop in joins looks like a metric change |
| **Propensity logging** | On every decision, or off-policy estimation is impossible later |
| Promotion gate | Blocking, automated, on the frozen holdout |

**Scope / NON-GOALS:**
- ❌ **Continuous weight updates.** Position: velocity is tiered by rollback path.
- ❌ **Training on the loop's own output unfiltered.** That's the self-consumption failure.
- ❌ The base model's pretraining (Q1) · the eval harness itself (Q5, consumed here).

---

## 2 · PRODUCT + THE FROZEN HOLDOUT *(min 6–9)* — probes **(b)**, (h) · **⭐ the L6 move** `I1c` `F1b` `I2b`

**What the teams see (b)** — the product of this system is a **promotion decision with its evidence**:

```
PROMOTION REQUEST · support-router v42 (replaces v41) · layer: ROUTER (daily)
  frozen holdout         +1.8 pts  (noise floor ±0.6)          ✓
  off-policy estimate    +2.1 pts  (DR, 95% CI +0.9 … +3.3)     ✓
  guardrails             safety ✓ · latency ✓ · cost ✓
  segments               no slice below −1.0 pt                 ✓
  label-join completeness 98.7%  (7-day mean 98.9%)             ✓
  training data lineage  2026-10-01 … 10-12 · 412k curated · consent-filtered
  rollback               config flip, rehearsed 3 days ago (41 s)
  DECISION: PROMOTE → canary 5% → full
```

**And what end users see:** thumbs and "edit before sending" on outputs, with a clear note that feedback may be used to improve the product (where they consented).

```
⚠ WHY EVERY IN-LOOP METRIC IS COMPROMISED:

   model → shapes what users see → shapes what they do →
     → becomes training data → next model → …

   ⚠ SO THE METRICS ARE COMPUTED ON A DISTRIBUTION THE MODEL
     ITSELF CREATED. The system can improve on EVERY metric we own
     while the product gets worse, and nothing in the loop can see it.

⭐ THE FIXED POINT: A FROZEN HOLDOUT OUTSIDE THE FLYWHEEL
   · drawn from a traffic source the model does NOT influence
     (a held-out population, a non-personalised surface, or
      deliberately-collected fresh data)
   · NEVER updated by the loop
   · refreshed ONLY by a deliberate human act, and that refresh is
     itself an event — it invalidates comparisons across it
   · ⚠ BUILD IT BEFORE THE PIPELINE THAT FEEDS IT

AND ITS COMPANION: a LONG-HORIZON HOLDBACK population, so slow
  degradation has somewhere to show up. Both take months to become
  useful, which is exactly why they're a Q1 deliverable.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Quality on the frozen holdout, and the holdback gap over months** | Improvement the loop can't fake |
| Signals | Verifiable-outcome rate · implicit signals · thumbs (severity only) | In-loop, compromised by design |
| Screening | Off-policy estimates with intervals | Cheap candidate filtering |
| **Guardrails** | Join completeness · rollback time per layer · segment regressions · training-corpus diversity | The loop's health |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
LOGGING: ~10⁸ decisions/day × ~1–2 KB (version, features, PROPENSITY,
   join key) ≈ 100–200 GB/day → partitioned by day and surface,
   with retention set by consent and label windows

JOINS: outcomes arrive minutes (CI, SQL) to weeks (refund disputes)
   → join windows per signal; completeness tracked per source

LABEL BUDGET: human review is the scarce resource
   → spend it on disagreement cases and uncertainty, plus a
     RANDOM SLICE for honest measurement (never only active learning)

UPDATE COMPUTE, PER LAYER:
   index rebuild minutes · router retrain an hour · LoRA adapter hours
   · base weights days of a large job
   → and ROLLBACK TIME PER LAYER is the budget that matters most:
     it caps how often each layer may change (§6)

EXPERIMENT SLOTS: A/B traffic is finite → off-policy screening decides
   which candidates earn a slot
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Update velocity | Continuous weight updates vs **a ladder tiered by rollback** | Most adaptive value lives in the fast, reversible layers |
| Training data | Everything users accepted vs **curated, verifiable-first** | Bulk accepted output is self-consumption |
| Evaluation | In-loop metrics vs **a frozen holdout outside the loop** | In-loop metrics measure the model's own distribution |
| Candidate screening | A/B everything vs **OPE screening → holdout gate → A/B** | Experiment slots are finite |
| Promotion | A human per release vs **an automated, versioned gate** | "We'll notice" isn't a plan at daily cadence |

### The system — *draw this in 90 seconds* `A2a`

```
 PRODUCTION TRAFFIC
       │
┌─ 1. SERVING — every decision logged ───────────────────────────
│ model version · features · ⭐ PROPENSITY P(action taken) · join key
│ ⚠ propensity can't be reconstructed later
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. FEEDBACK COLLECTION — three tiers (§5.1) ──────────────────
│ ⚠ DELAYED-LABEL JOIN: outcomes arrive hours to weeks later;
│   join completeness is a MONITORED metric
│ OUTPUT: decisions joined to outcomes, as of a date
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. CURATION — NOT everything becomes training data ───────────
│ filter · weight · deliberately sample · consent-filter
│ OUTPUT: a versioned, lineage-tracked training snapshot
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. TRAIN A CANDIDATE at the right rung of the ladder (§6) ────
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⭐ GATE — the FROZEN HOLDOUT, outside the loop ─────────────
│ + off-policy estimate (IPS / doubly-robust) + guardrails + canaries
│ OUTPUT: promote · reject — with an evidence record (§2)
└──────┬─────────────────────────────────────────────────────────
       ▼
 PROMOTE BY TIER  →  ⚠ ROLLBACK PATH READY  →  back to serving (the loop closes)
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Silent join loss:** *"An upstream schema change drops 30% of joins and the metric moves. Teams debug the model for a month. Join completeness is monitored per source."*
- **Self-consumption:** *"Training on accepted outputs narrows the distribution. Curation sits between feedback and training, and corpus diversity is monitored."*
- **Labels too slow for the cadence:** *"Weeks-late outcomes can't gate a daily router — so OPE and the frozen holdout gate it, and late labels correct it."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Frozen holdout before the pipeline** | "Without a fixed point outside the loop, every metric is computed on a distribution the model created" | It's a Q1 deliverable that produces no user value for months |
| **2** | **Velocity tiered by rollback** | "Each layer moves as fast as its rollback path allows, and no faster" | Slower adaptation on the layers that matter most |
| **3** | **Propensity logged on every decision** | "You cannot reconstruct it later — and without it, off-policy estimation is impossible forever" | Logging cost and schema discipline on the hot path |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The feedback signals, ranked by gameability (d) `A2c`

```
 ⭐ TIER 1 — VERIFIABLE OUTCOMES                     [gold]
    did the code pass CI · did the SQL execute · did the refund post
    · did the booking confirm
    → objective, ungameable, and directly trainable
    ⚠ THE HIGHEST-VALUE ENGINEERING WORK HERE IS CHANGING THE
      PRODUCT SO IT EMITS MORE OF THESE. That's a product
      conversation, not an ML one, and it's the biggest lever.

 TIER 2 — IMPLICIT SIGNALS                          [dense, confounded]
    retry · edit distance on what the user kept · escalation to a
    human · follow-up question · abandonment
    → plentiful, and every one has a confound (a retry might mean
      the answer was wrong, or the user changed their mind)

 TIER 3 — EXPLICIT FEEDBACK                         [sparse, biased]
    thumbs up/down: ~0.1–1% response rate, BIASED TOWARD ANGER
    → useful as a signal of severe failure, useless as a quality
      distribution

⚠ USE ALL THREE. NEVER LET THE CHEAP ONES DRIVE WEIGHT UPDATES ALONE.
```

Plus two sources outside the loop: **human-reviewed samples** (disagreements, corrections, a random slice) and the **frozen holdout** itself, which is never trained on.

### 5.2 Curating the training set (e) `F1c`

- **Verifiable first:** where the label comes from the world, the self-reference is broken.
- **Weight implicit signals** by how reliable each has proven against verifiable outcomes, per surface.
- **Deliberately sample:** corrections, user edits and disagreement cases carry more signal than bulk accepted output.
- **Delayed labels:** snapshot by *label-as-of* date; weight recent, incomplete data by elapsed time (G22) rather than trusting or dropping it.
- **Consent and deletion:** only consented interactions; deletion requests remove records from future snapshots; enterprise data excluded.
- **Watch diversity** of each snapshot over time — collapse shows up as narrowing before it shows up as degradation.

### 5.3 The update objectives and estimators (f) `A2b` `F1a`

| Rung | What changes | Objective |
|---|---|---|
| **Index / knowledge** | Documents and facts | No training — correctness and freshness of content |
| **Prompts, configs, thresholds** | Behaviour knobs | Thresholds re-calibrated on verifiable outcomes |
| **Routers, small classifiers** | Which path or model handles a request | Cross-entropy on curated, outcome-labelled routes |
| **LoRA adapters** | Surface-specific behaviour | SFT on curated outputs; **DPO on (model output, user-edited version) pairs** |
| **Base weights** | Everything | A periodic, fully gated fine-tune |

**Off-policy estimation** — how a candidate would have done on logged traffic:

```
IPS:  V̂(π_new) = (1/n) Σ r_i · π_new(a_i | x_i) / π_log(a_i | x_i)
DR:   a reward model's prediction + an IPS correction on its residual
      → lower variance, still unbiased if either part is right
⚠ high variance when π_new is far from π_log → a SCREENING tool,
  not the gate
```

### 5.4 Fine-tune an existing model (f) `F2c`

The weekly rung is **LoRA**, precisely because an adapter **detaches** in seconds — a rollback path the base weights don't have. Edits make good preference data: the user's corrected version is preferred over the model's original, which is a DPO pair the product emits for free.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · LOG        decisions with propensity and join keys
STAGE 2 · JOIN       outcomes as they arrive · completeness checked per source
STAGE 3 · CURATE     verifiable-first · weighted · sampled · consent-filtered
STAGE 4 · SNAPSHOT   versioned, with lineage to every source
STAGE 5 · TRAIN      at the rung whose rollback allows it
STAGE 6 · GATE       OPE screen → frozen holdout → guardrails → canary → promote
```

---

## 6 · THE UPDATE LADDER + ROLLBACK *(min 32–38)* — probe **(g)** · **what "continuous" actually means** `F3a` `F3b` `A3b` `I1b`

```
⚠ "CONTINUOUS LEARNING" DOES NOT MEAN CONTINUOUS WEIGHT UPDATES.
   VELOCITY IS TIERED BY BLAST RADIUS AND REVERSIBILITY:

   LAYER                          CADENCE      ROLLBACK
   ─────────────────────────────────────────────────────────
   retrieval index / knowledge    MINUTES      instant (re-index)
   prompts · configs · thresholds HOURS        instant (config flip)
   routers · small classifiers    DAILY        fast (swap model)
   adapters / LoRA                WEEKLY       fast (detach)
   ⚠ BASE WEIGHTS                 MONTHLY      SLOW + EXPENSIVE

⭐ THE RULE: EACH LAYER MOVES AS FAST AS ITS ROLLBACK PATH ALLOWS,
   AND NO FASTER.

WHY THIS IS THE RIGHT FRAME: most of the adaptive value people want
  from "continuous learning" lives in the TOP TWO ROWS — new
  knowledge, changed policy, adjusted thresholds — which are
  minutes-to-hours and instantly reversible.
  ⚠ Reaching for base-weight updates to get that is choosing the
    slowest, least reversible layer to solve a fast, reversible problem.

DRIFT — three different things, three different responses:
  · FEATURE drift   (inputs moved)        → monitor distributions
  · PREDICTION drift(outputs moved)       → cheap, no labels needed
  · ⚠ LABEL drift   (the relationship moved) → the one that matters,
      and the hardest, because labels arrive late
    → proxy it with prediction drift + the frozen holdout

ROLLBACK: the cadence is set by it, so it's a first-class artifact —
  automated, tested, and ⚠ REHEARSED. An untested rollback path is
  not a rollback path.
```

- **The objective** `A3b`: promote only if the frozen-holdout gain clears the noise floor, **no guardrail or segment regresses**, and the OPE lower bound isn't below the incumbent — and a layer may change no faster than its rollback.
- **Efficiency** `F3a`: OPE screens candidates before they spend experiment slots; the fast rungs deliver most of the value at a fraction of the cost. The trade: OPE is high-variance for policies far from the logged one.
- **Scale** `F3b`: logs partitioned by day and surface; joins run incrementally; per-surface adapters train independently.
- **Reliability** `I1b`: last-good kept **per layer**; automated, rehearsed rollback; promotions pause automatically when join completeness drops.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
THE PATH OF A CANDIDATE:
  OPE screen on logged traffic (cheap, with intervals)
  → FROZEN HOLDOUT gate (blocking, noise floor stated)
  → guardrails + canary suites + per-segment checks
  → canary traffic → A/B where the rung justifies it
  → LONG-HORIZON HOLDBACK keeps watching for months
WHO OWNS THE GATE: thresholds are a versioned artifact owned by product
  quality, not by the team measured on model velocity (FU 9);
  refreshing the frozen holdout needs explicit sign-off
```

**Success** = the frozen holdout and the long-horizon holdback both improve over months, no segment quietly erodes, rollbacks are fast and rare, and join completeness stays stable. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Feedback-loop bias** | The model shapes the traffic it learns from | Frozen holdout; propensity logging; exploration |
| **Survivorship** | Users the model fails leave, and their absence improves the aggregate | Segment metrics; holdback; churn analysis |
| **Explicit-feedback bias** | Thumbs come mostly from angry users | Use for severity, never as a quality distribution |
| **Engagement bias** | Heavy users dominate implicit signals | Per-user weighting; stratified review |
| **Language** | Less traffic → fewer signals → slower improvement for some locales | Per-locale holdouts and review budgets |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Logging overhead off the hot path; update compute sized per rung | Logging p99 overhead · $ per promotion |
| **Reliability** `I1b` | **Rehearsed rollback per layer**; last-good kept; promotions pause on join loss | Rollback time · failed rollbacks = 0 |
| **Usability** `I1c` | For teams: a promotion request with its evidence; for users: lightweight feedback and edit affordances | Time to promote · feedback rate |
| **Scalability** `I1d` | Partitioned logs; incremental joins; per-surface adapters | Join lag · pipeline throughput |
| **Privacy** `I1e` | **Consent** gates training use; PII scrubbed from logs; **deletion propagates** to future snapshots; enterprise data excluded | Unconsented records in snapshots = 0 |
| **Safety** `I1f` | Safety suites are **blocking guardrails** at every promotion; feedback can't teach the model a harmful behaviour unreviewed | Safety regressions promoted = 0 |
| **Security** `I1h` | ⚠ **Feedback poisoning**: coordinated thumbs, scripted edits or bots steering the model (Tay, 2016) → rate limits and trust weighting on feedback, anomaly detection, verifiable-first curation; log integrity | Anomalous feedback caught |
| **Ethics / bias** `I1i` | Segments the loop pushes out are watched; locales with little traffic get their own holdouts | Worst-segment delta per promotion |
| **Explainability** `I1j` | An **evidence record** for every promotion; **lineage** from model version to training data; every decision logged with its version | % promotions with evidence · lineage coverage |

> ⭐ *"A loop that only ever improves on its own metrics is evidence of a measurement problem. Production readiness is the fixed point, the rollback, and the record of why each change shipped."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Online / bandit learning:** the extreme end of the ladder; needs exploration accounting and its own rollback story.
- **Active learning** on the label budget: spend it on uncertainty, keep a random slice for honest measurement.
- **Multi-model fleets:** rollback becomes a coordination problem across models that depend on each other.

**When the loop can't be trusted — the freeze-the-loop pipeline** `A5b` `F2a`:

```
the frozen holdout diverges from in-loop metrics, join completeness drops,
or feedback poisoning is suspected
  → FREEZE promotions of every learned rung (routers, adapters, weights)
  → PIN each layer to its last-good version
  → keep only the top rung (index, configs) moving, with human review
  → quarantine suspect feedback windows from all future snapshots
  → diagnose → fix → resume through the gate, with the holdout confirming
  "When the loop is suspect, stop learning from it — serving can continue."
```

**Top 3 failure modes:**
1. **Self-confirming drift** — the loop improves every in-loop metric while the product degrades, and without a frozen holdout nothing can detect it.
2. **Silent join loss** — an upstream change drops 30% of label joins; the metric moves; the team debugs the model for a month.
3. **Self-consumption** — the model trains on its own outputs, the distribution narrows, and diversity collapses (Q8's model-collapse argument, applied to a live loop).

**Q1 ships:** the **frozen holdout**, **propensity logging**, and the **rollback path**.
> *"All three produce no user-visible value and all three are impossible to retrofit — propensity can't be reconstructed, the holdout needs months to be useful, and an untested rollback isn't one."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "What's the first thing you build?"** `A1a` `I2a`
> The **frozen holdout**, before the pipeline that feeds it — and that's a deliberate ordering rather than a quirk. This is a closed-loop control system: the model we deploy shapes the traffic that trains the next model, so **every metric computed inside the loop is computed on a distribution the model itself created**. The system can improve on every number we own while the product gets worse, and nothing inside the loop can see that. So I need a fixed point: a fixed evaluation set drawn from traffic the model doesn't influence, never updated by the loop, refreshed only by a deliberate human act — and that refresh is itself an event, because it invalidates comparisons across it. Alongside it, a **long-horizon holdback population**, so slow degradation has somewhere to show up. Both take months to become useful, which is precisely why they're a quarter-one deliverable rather than something you add when you need them.

**2. "What feedback signals do you use?"** `A2c` `F1b`
> Three tiers, ranked by how gameable they are. **Verifiable outcomes** are gold — did the code pass CI, did the SQL execute, did the refund actually post, did the booking confirm. They're objective, ungameable and directly trainable, and the most valuable engineering work in this whole system is **changing the product so it emits more of them** — which is a product conversation rather than an ML one and is the biggest available lever. **Implicit signals** — retry, edit distance on what the user kept, escalation to a human, abandonment — are dense but every one has a confound: a retry might mean the answer was wrong or that the user changed their mind. **Explicit thumbs** are sparse, around 0.1 to 1% response rate, and **biased toward anger** — useful as a detector of severe failure, useless as a quality distribution. I use all three and never let the cheap ones drive weight updates on their own.

**3. "How often do you retrain?"** `A5a` `I1b`
> That's the wrong unit — **"continuous learning" doesn't mean continuous weight updates**, and conflating them is how these systems get dangerous. I tier the velocity by blast radius and reversibility: the **retrieval index and knowledge** update in minutes and roll back instantly by re-indexing; **prompts, configs and thresholds** in hours with a config flip to undo; **routers and small classifiers** daily; **adapters** weekly; **base weights** monthly, because rollback there is slow and expensive. The rule is that **each layer moves as fast as its rollback path allows, and no faster**. And the reframe worth stating: most of the adaptive value people actually want from "continuous learning" lives in the top two rows — new knowledge, changed policy, adjusted thresholds — which are minutes-to-hours and instantly reversible. Reaching for base-weight updates to get that is choosing the slowest, least reversible layer to solve a fast, reversible problem.

**4. "Labels arrive three weeks late. How does the loop work?"** `F2b` `A2d`
> Every decision gets a **join key and a timestamp**, and outcomes are joined back as they arrive — so the training set is **mutable**, which breaks the usual immutable-dataset assumptions and needs snapshotting by label-as-of date. Recent data is systematically incomplete, so it's **importance-weighted by elapsed time** rather than either excluded or trusted naively, which is the same structure as delayed conversions. And the operational point I'd emphasise: **join completeness is a monitored metric**, because a silent drop in joins — an upstream schema change, a dropped event, a renamed field — looks *exactly* like a change in the metric itself, and teams will debug the model for a month before anyone checks the pipeline. That's one of the most common and most expensive failures in this kind of system, and it costs almost nothing to instrument.

**5. "How do you evaluate a candidate model without an A/B test?"** `F1a` `I2a`
> **Off-policy estimation**, which requires a decision made on day one: **log the propensity** — the probability of the action actually taken — at decision time. With that, I can estimate how a different policy would have performed on logged traffic using inverse-propensity scoring or a doubly-robust estimator, which lets me screen candidates cheaply before spending experiment slots. The critical constraint is that **propensity cannot be reconstructed later** — if it wasn't logged, the capability is gone permanently, which is why it's a Q1 item despite producing no visible value. The honest limits: IPS has high variance when the candidate policy differs a lot from the logged one, and it can't see effects that only appear when the new policy is at full traffic — so it's a **screening tool**, and the frozen holdout plus a real experiment remain the gate.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Every metric is up and users are complaining. What's going on?"** `F1c` `I2b`
> That's the signature failure of a closed loop, and I'd treat the complaints as the more reliable signal. The mechanism: the model shapes what users see, which shapes what they do, which becomes the training data — so the metrics measure **agreement with a distribution the model created**, not quality. Concretely this shows up as the system getting very good at the users and queries it already serves well, while degrading on everything it has pushed out of the distribution — and those users complain or leave, and their absence *improves* the aggregate. Where I'd look: the **frozen holdout**, which is the whole reason it exists and which should show the divergence; **segment-level metrics**, because an aggregate gain frequently hides a large regression in a minority slice; and the **long-horizon holdback**, since slow degradation is invisible in a two-week window. The structural lesson I'd draw: **in a closed loop, a metric that only ever goes up is evidence of a measurement problem**, not of sustained improvement — and the fix is architectural (a fixed point outside the loop) rather than a better metric inside it.

**7. "Isn't training on your own model's output just model collapse?"** `F1c` `A2d`
> It's the same mechanism and it's a real risk, which is why **curation sits between feedback and training** rather than the loop being closed automatically. The failure: accepted outputs are model outputs, so training on them narrows the distribution — each round clips the tails, and diversity degrades faster than quality does, so the metrics look fine for a while. Three mitigations. **Prefer verifiable outcomes**, where the label comes from the world rather than from the model — a test passing is ground truth regardless of who wrote the code, which breaks the self-reference entirely. **Filter and weight deliberately**: human-reviewed samples, disagreement cases, and corrections carry far more signal than bulk accepted output, and bulk accepted output is mostly what you already know. And **monitor diversity statistics of the training corpus over time**, not just quality — because collapse shows up as narrowing before it shows up as degradation. The general position: the loop provides the **candidates**, and something outside the loop — verification, humans, or a frozen reference — provides the **judgement**.

**8. "Your model updated and quality dropped. Walk me through the response."** `I1b` `A1c`
> **Roll back first, diagnose second** — and the rollback should be automated, tested and rehearsed, because an untested rollback path is not a rollback path and you find that out at the worst moment. Then the diagnosis, in order of likelihood: **was it the model at all?** Check the frozen holdout for the previous version too — if that also dropped, something upstream changed and the model is the wrong suspect. **Did the training data change?** Join completeness, feature coverage, and a sudden shift in the labelled distribution are the usual culprits, and a silent upstream schema change is far more common than a genuinely worse model. **Did the distribution shift?** A seasonal or product change means the new model may be correct and the *evaluation* stale. And **which segment dropped?** — an aggregate decline is often one slice collapsing. The reason the tiered ladder matters here is that it bounds the damage: if the change was a config or an index update, rollback is instant and the blast radius is minutes; if it was base weights, it's slow and expensive, which is exactly why that layer moves monthly.

**9. "Who owns the decision to promote a model?"** `A3b` `I1j`
> The **gate owns it, and the gate is automated** — because at any meaningful cadence there's no human in the loop for each promotion, and "we'll notice" is not a plan. So promotion is blocking on: the **frozen holdout** with a noise floor, **guardrail metrics** that capability cannot outvote, **canary suites**, and an off-policy estimate where applicable. But the interesting part of the question is organisational rather than mechanical: **who owns the gate's thresholds**, and the answer shouldn't be the team measured on model improvement — because those thresholds make their primary metric look worse and will erode one defensible increment at a time. That's the same structural argument as an exploration budget or a relevance floor. So: the gate's criteria are a **versioned, reviewable artifact** with an owner accountable for product quality rather than model velocity, and changing a threshold is a reviewed change rather than a config edit. And I'd want the **frozen-holdout refresh** to require explicit sign-off too, since refreshing it is the easiest way to make a failing gate pass.

**10. "One quarter, and the team wants to ship the flywheel. What do you actually build?"** `I2b` `A3a`
> The three things that produce **no user-visible value and cannot be retrofitted** — and I'd make that argument explicitly, because it's an unpopular quarter-one plan. **Propensity logging**, because it cannot be reconstructed later and losing it forecloses off-policy evaluation permanently. **The frozen holdout and the long-horizon holdback**, because they take months to become useful and the moment you need them is the moment it's too late to start. And the **rollback path, tested**, because it's what sets the safe update cadence for everything afterwards. Then, if there's room, the **top two rungs of the ladder** — retrieval-index and config updates — which deliver most of the adaptive value people actually want, in minutes, with instant rollback. What I'd resist is starting with the trainer, because a training pipeline feeding a loop with no fixed point outside it is a machine for confidently making the product worse — and it will look like it's working for a quarter or two, which is the dangerous part.

---

## ⭐ THE L6 LINE

> **"This is a closed-loop control system, not a training pipeline — the model shapes the traffic that trains the next model, so every in-loop metric is computed on a distribution the model created. Build the frozen holdout before the pipeline that feeds it, and tier update velocity by rollback path."**

## 🧭 HOOKS — *steer by what you mention*

- *"A closed-loop control system"* → invites the feedback-dynamics branch (**good at L6**)
- *"The frozen holdout lives outside the flywheel"* → invites the evaluation branch
- *"Change the product so it emits verifiable outcomes"* → invites the product/ML boundary branch (**good at L6**)
- *"Each layer moves as fast as its rollback allows"* → invites the deployment/ops branch
- *"Propensity can't be reconstructed later"* → invites the off-policy/causal branch
- *"Join completeness looks like a metric change"* → invites the data-engineering branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the update ladder"* — index/config/router/adapter/weights regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the frozen holdout and drift than finish the label joins. Sound right?"*

**Name and defer:**
- "**The eval harness** that the gate runs is its own design (Q5) — contamination, noise floor, judge calibration."
- "**Online/bandit learning** is the extreme end of this ladder and needs exploration accounting."
- "**Feature stores and point-in-time correctness** are the substrate this depends on."
- "**Drift detection as a platform** across many models is a different scope."
- "**Active learning** on the label budget — spend it on uncertainty, keep a random slice for honest measurement."
- "**Multi-model fleets** make rollback a coordination problem rather than a single switch."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the fixed point first | §0, FU 1 | *"Build the frozen holdout before the pipeline that feeds it."* |
| **A1b** | Logging, label and rollback budgets | §1, §3 | *"~10⁸ decisions × 1–2 KB a day; rollback time per layer caps cadence."* |
| **A1c** | Degrade without lowering quality | §6, §9, FU 8 | *"Roll back first, diagnose second; freeze learned rungs when the loop is suspect."* |
| **A2a** | The full stack | §4 diagram | Serving with propensity → joins → curation → train → gate → promote |
| **A2b** | Model components | §5.3 | The five rungs and their objectives; IPS and DR estimators |
| **A2c** | Data sources | §5.1, FU 2 | Verifiable, implicit, explicit signals; reviewed samples; the holdout |
| **A2d** | Staged pipeline | §5.5, FU 4, 7 | Log → join → curate → snapshot → train → gate |
| **A3a** | Top-down, unprompted | §0, FU 10 | *"A closed-loop control system"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 9 | *"Promote on holdout gain over the noise floor, no segment regressing, OPE bound above incumbent."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Verifiable outcomes · rollback · uninfluenced traffic · consent |
| **A5a** | Alternatives compared | §4, FU 3 | Continuous weights vs ladder; all accepted vs curated; in-loop vs frozen holdout |
| **A5b** | A full alternative pipeline on demand | §9 | Freeze-the-loop |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right estimator and loss per rung | §5.3, FU 5 | *"DPO on (output, user-edited) pairs; doubly-robust OPE for screening."* |
| **F1b** | Metrics and their trade-offs | §2, FU 2 | *"In-loop metrics are computed on a distribution the model created."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 6, 7 | Loop bias, survivorship, anger-biased thumbs, engagement, language |
| **F2a** | Something fails → a fallback | §9 | Freeze-the-loop |
| **F2b** | Labels too slow → design for it | §4, FU 4 | OPE + holdout gate fast rungs; late labels correct, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Weekly LoRA adapters that detach in seconds |
| **F3a** | Efficiency with a quality trade | §6 | OPE screening; fast rungs deliver most value |
| **F3b** | Scale levers | §6 | Partitioned logs; incremental joins; per-surface adapters |
| **F3c** | Extension, with cost and maturity risk | §9 | Bandits, active learning, multi-model fleets |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Logging overhead; $ per promotion |
| **I1b** | Reliability | §6, §8, FU 3, 8 | Rehearsed rollback per layer |
| **I1c** | Usability | §2, §8 | Promotion requests with evidence; feedback affordances |
| **I1d** | Scalability | §3, §8 | Partitioned logs; incremental joins |
| **I1e** | Privacy | §5.2, §8 | Consent; deletion propagates to snapshots |
| **I1f** | Safety | §8 | Blocking safety suites at every promotion |
| **I1h** | Security | §8 | **Feedback poisoning** (Tay); trust-weighted feedback |
| **I1i** | Ethics / bias | §7, §8 | Segments the loop pushes out; low-traffic locales |
| **I1j** | Explainability | §2, §8, FU 9 | Evidence record per promotion; lineage |
| **I2a** | Offline → online | §7, FU 1, 5 | OPE → holdout → guardrails → canary → A/B → holdback |
| **I2b** | The success definition | §7, FU 6, 10 | *"Holdout and holdback improve over months; no segment erodes."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) loop → §4 · (d) signals → §5.1 · (e) curation → §5.2 · (f) objectives → §5.3–5.4 · (g) ladder → §6 · (h) success → §2, §7 · (i) extension → §9.
