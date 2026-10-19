# 00 · L6 ML SYSTEM DESIGN PLAYBOOK — *the format, the strategy, the cheat sheet*

> Distilled from **R1–R7** and the Google L6 feedback form. The form has **3 parameters, 10 sub-parameters and 32 pointers**: A has 12, F has 9, I has 11, and there is **no I1g**. (R1's "~35" was a rough count.) This file is question-agnostic: use it to turn **any** ML system design prompt into an answer that earns every pointer.
>
> **Read in this order:** Part 1 (how you're graded) → Part 2 (the format) → Part 3 (strategy) → Part 12 (the day-of page). The rest is reference; **Part 13 is the glossary of every metric and evaluation method used across the INTV sheets.**

---

## PART 1 · HOW YOU'RE GRADED — *the form, decoded*

### The one rule behind everything

> ⭐ **The grader can only tick a pointer for something you SAID.** If you know it and didn't say it, it doesn't exist. So the whole strategy is: make sure each of the 32 pointers has at least one sentence in your answer, and say it **before** you're asked.

### What the "mostly positive" sample candidate still lost

| Where they lost credit | Evidence in the feedback | The general lesson |
|---|---|---|
| **Clarifying questions came late** | A4a: *"**upon triggering**, candidate did cover latency/surfaces"* | Ask in **minute 1, unprompted**, then state assumptions out loud |
| **Explainability was blank** | I1j has no comment | It's the easiest pointer on the form: **say three audiences** (§ Part 6) |
| **Security stopped at the gateway** | I1h: validation, OAuth, rate limits only | Always add **the domain's attack**: poisoning, injection, gaming, steering |
| **Troubleshooting was reactive** | F2a/F2b read as answers to "what if…" | **Raise the failure yourself** in the architecture section |

### The map: 3 → 10 → 32

```
A · SYSTEM ARCHITECTURE / DESIGN (12)
  A1 practical, scalable system ........ a critical-path decision · latency budget · tiered serving
  A2 components and data ............... full stack · models · data sources · staged pipeline
  A3 drive the conversation ............ top-down unprompted · a principled trade-off objective
  A4 clarify upfront ................... questions + assumptions, unprompted
  A5 alternatives ...................... a comparison table · a full alternative pipeline on demand
F · AI FLUENCY (9)
  F1 right tools, no bias .............. models + losses · metrics + trade-offs · bias sources
  F2 troubleshoots independently ....... primary path fails · latency too tight · fine-tune existing
  F3 scaled-up AI ...................... efficiency (quantise/distil) · scale levers · extension
I · AI PRODUCT INTEGRATION (11)
  I1 production readiness (9) .......... performance · reliability · usability · scalability ·
                                         privacy · safety · security · ethics/bias · explainability
  I2 evaluation, rollout, success ...... offline → shadow → A/B (debiased) · the success definition
```

---

## PART 2 · THE ANSWERING FORMAT

### 2.1 The spine — *ten sections, always in this order*

> **Mnemonic: O-R-P-C-A-D-S-E-P-X** — *"Only Real Products Can Actually Deliver Safe, Effective Production eXperiences."*

| Min (60) | Min (45) | § | Section | What it must contain | Pointers it earns |
|---|---|---|---|---|---|
| 0–1 | 0–1 | **0** | **O**pening | The three-sentence opening + the plan | A1a, A3a |
| 1–6 | 1–4 | **1** | **R**equirements | 4–6 questions (each with *why it changes the design*) · assumptions table · FR · NFR/SLO table · non-goals **with positions** | A4a, A1b |
| 6–9 | 4–6 | **2** | **P**roduct + metrics | What the user sees (ASCII mock-up) · **the label ladder** (available → meaningful) · the metric stack | I1c, F1b, I2b |
| 9–12 | 6–8 | **3** | **C**apacity + budgets | QPS · per-stage latency budget that **sums** · memory · the cost line · **the trick** | A1b, I1a, I1d |
| 12–22 | 8–16 | **4** | **A**rchitecture | Alternatives table → the stack (TASK / OUTPUT boxes) → **problems raised yourself** → defensible choices | A2a, A5a, A1c, F2b |
| 22–32 | 16–24 | **5** | **D**ata & model | Data sources (+ trap each) · labels / training set · models **and losses** · fine-tune existing · staged pipeline | A2b, A2c, A2d, F1a, F1c, F2c |
| 32–38 | 24–28 | **6** | **S**erving | Efficiency (quality trade) · scale levers · the trade-off objective · reliability | F3a, F3b, A3b, I1b |
| 38–44 | 28–33 | **7** | **E**val, rollout, bias | Offline (debiased) → shadow → A/B → holdout · success definition · bias table | I2a, I2b, F1b, F1c |
| 44–48 | 33–36 | **8** | **P**roduction sweep | **Nine one-liners**, each with a metric | I1a–I1j |
| 48–52 | 36–38 | **9** | e**X**tension + failures | Extension (cost + maturity risk) · a full alternative pipeline · top 3 failures · Q1 ships | F3c, F2a, A5b |
| 52–60 | 38–45 | — | Probes | Follow-ups (Part 5) | whatever's left |

> ⚠ **If you're short on time, never drop §8.** It's nine pointers in four minutes, the best rate on the form. Say: *"I have five minutes; I'd rather do the production-readiness sweep than finish the training pipeline. Sound right?"*

### 2.2 The opening: three sentences — *DECISION · OBJECTIVE · TWIST*

```
1. DECISION  "Before any boxes, one decision drives the rest: ___."
             (what sits in the critical path · what 'similar/relevant/good' means
              · what the unit is · what's fast vs slow)
2. OBJECTIVE "And the metric isn't ___ (the available one); it's ___ (the meaningful
              one), because ___ (how the available one is gamed)."
3. TWIST     "One thing about this problem changes the design: ___."
             (the insight a generic answer would miss)
+ PLAN       "I'll go: questions and assumptions, the product and metrics, capacity,
              the architecture, data and models, serving, evaluation, a production-
              readiness sweep, then extensions. Stop me wherever you want depth."
```

**The seven openings, as worked examples:**

| Sheet | DECISION | OBJECTIVE (available → meaningful) | TWIST |
|---|---|---|---|
| **R1** suggestions | Retrieval + ranking in the critical path; generation is a gated fallback | CTR → **the conversation it starts succeeds** | A suggestion is an endorsement: a higher safety bar than an answer |
| **R2** video-as-query | "Similar" = same action → motion-sensitive, appearance-invariant | Clicks → **same move, different context** | Index **moments**, not videos |
| **R3** Discover Weekly | Discovery, not prediction: novelty is a hard constraint | Plays → **discoveries that stick** | Two clocks: the latency budget is two songs long |
| **R4** watch-next | The video on screen anchors retrieval and ranking | Clicks / watch time → **satisfied watch time** | One loss per label; one optimizer per parameter type |
| **R5** product or video | One platform, two domains: what's shared vs what differs | CTR / GMV → **purchases not returned** | Feedback acts at three speeds |
| **R6** real-time | Three freshness problems: features, items, weights | — | 50k items is 13 MB: exact retrieval, no ANN |
| **R7** bandit 4→1 | Arms change every round → model uncertainty, not arm counts | Click → **click + length-normalised dwell − hides** | K = 4 makes off-policy evaluation cheap |

### 2.3 Per-probe micro-format — *DECIDE · COMPARE · BUILD · BREAK · MEASURE*

Every time the interviewer probes a topic, answer in five beats, **in this order**:

| Beat | You say | Pointer it feeds |
|---|---|---|
| **DECIDE** | The answer first, in one sentence: *"I'd use X."* | A3a (top-down) |
| **COMPARE** | *"Over Y, because of ___; Y would win if ___."* | A5a |
| **BUILD** | The mechanism: model, loss, data, numbers | A2*, F1a |
| **BREAK** | *"Where this fails: ___, and the fix is ___."* | F2a/F2b |
| **MEASURE** | *"And I'd know it works when ___ moves, with ___ as a guardrail."* | F1b, I2a |

### 2.4 The micro-templates for each section

```
§1 REQUIREMENTS
  ⚠ Ask BEFORE you're asked: 4–6 questions, each "(decides ___)"
  ASSUMPTIONS table: scale · surfaces · output size · personalisation · freshness · existing assets
  FR: one line, comma-separated
  NFR / SLO table: latency per surface · freshness · availability ("on failure: ___, never ___")
                   · the domain's hard SLO (unsafe rate, privacy leakage, propensity logging…)
  NON-GOALS: ❌ the tempting wrong objective, "Position: ___" · adjacent systems (name the sheet)

§2 PRODUCT + METRICS
  ASCII mock-up of what the user sees, including the CONTROLS (dismiss, hide) and the REASON
  THE LADDER:  impression → click → engage → deep engage → ⭐ the meaningful outcome → returns
               + NEGATIVES: dismiss, hide, abandon, return, regret
  METRIC STACK: North star · fast proxies · offline (screens, never decides) · GUARDRAILS (ship blockers)
  "Say:" one line: "___ is available; ___ is meaningful."

§3 CAPACITY + BUDGETS
  traffic (avg + peak) · the per-stage latency budget (sums, with tail headroom)
  · memory (index, tables) · the cost line ("the ranker / the LLM is the cost driver")
  · ⭐ THE TRICK (precompute, prefetch, stream-ahead, "the budget is longer than it looks")

§4 ARCHITECTURE
  "The decision first": alternatives table on NAMED dimensions → choice + the trade
  the stack: OFFLINE / NEARLINE / ONLINE, boxes with TASK + OUTPUT, logs feeding back
  ⚠ "Raise these yourself": 3–4 failure modes, each with its fix (F2a, F2b, A1c)
  defensible choices: # · choice · "say this" · the trade you accept

§5 DATA & MODEL
  5.1 data sources: source · what it gives · ⚠ the trap
  5.2 labels / training set: eligibility → clean → label → debias → balance → time split
  5.3 models + objectives: per subproblem, model + loss + WHY it fits the label
  5.4 fine-tune an existing model: LoRA / warm start (F2c)
  5.5 the staged pipeline with cadences

§6 SERVING
  efficiency lever with its quality trade (F3a) · scale levers (F3b)
  · the trade-off objective as a formula with a knob (A3b) · the fallback chain (I1b)

§7 EVAL, ROLLOUT, BIAS
  OFFLINE (time-split, debiased) → SHADOW → A/B (ramped) → LONG-TERM HOLDOUT
  "Success = ___ up AND ___ up AND ___ not down, every guardrail held." (I2b)
  bias table: bias · mechanism here · mitigation

§8 SWEEP: nine rows, each "for THIS system" + a metric

§9 EXTENSION + FAILURES
  the extension with cost + maturity risk (F3c) · a full alternative pipeline (A5b)
  · top 3 failure modes · "Q1 ships: ___, because ___ can't be added later"
```

---

## PART 3 · STRATEGY — *how to run the hour*

### 3.1 The ten commandments

1. **Ask before you're asked.** Minute 1: questions, then an assumptions table. (A4a)
2. **Announce the plan, then follow it.** The plan *is* A3a. Say *"Stop me wherever you'd like depth."*
3. **Make one decision explicit before drawing.** The critical path, the meaning of the key word, or the unit. (A1a)
4. **Separate available from meaningful.** Every metric conversation has both. (F1b, I2b)
5. **Raise failures yourself.** Latency squeeze, primary path underperforming, cold start, feedback loop. (F2a, F2b, A1c)
6. **Compare before you choose.** A table on named dimensions, then *"the trade I accept is ___."* (A5a)
7. **Write the objective as a formula with a knob,** and say who owns the knob. (A3b)
8. **Put numbers on everything.** QPS, ms per stage, GB, $ per 1,000 requests. (A1b, I1a, I1d)
9. **Never skip the sweep;** nine one-liners, each with a metric. (I1a–j)
10. **End with what ships first and why.** Usually the thing that can't be added later: logging, propensities, holdouts.

### 3.2 Adapting to how the prompt is written

| The prompt… | Do this | Seen in |
|---|---|---|
| **is one line** ("Design X") | The clarifying questions do the scoping; narrow to 1–2 surfaces out loud | R1, R2, R3 |
| **lists aspects to cover** | Build a mental checklist table in minute 1 and say *"I'll cover all of them, in this order"*; use **their words** as section names | R5, R6, R7 |
| **has Parts with math** | Re-budget the hour out loud; write **notation first**; derive in four lines on the board | R4 |
| **gives hints** ("negatives nobody logged") | Treat each hint as a probe; repeat the hint's phrase when you answer it | R4 |
| **lists its follow-ups** | Raise them in §4 **before** they're asked; that converts F2 into "independent" | R7 |
| **says "products or videos"** / leaves a choice | Ask; if left to you, choose **and** mark where the other diverges | R5 |
| **gives a small number** ("4 candidates", "tens of thousands of items") | Do the arithmetic: small numbers usually change the design | R6, R7 |

### 3.3 Steering, interruptions, recovery

- **Hooks:** plant phrases that invite the branch you're strongest on (*"only shown items have labels"* → bias branch; *"showing nothing is a legitimate output"* → reliability branch).
- **Deep-dive pulled:** go deep, then return: *"Back to the spine: next is serving."*
- **Interviewer silent:** checkpoint: *"I've covered the stack; do you want depth on the models or move to evaluation?"*
- **Blanked:** go back to the driving decision (the DECISION sentence). It regenerates the architecture.
- **Out of time:** prioritise §8 and the success definition over finishing §5.
- **Name and defer:** *"X is its own system (sheet Gn); out of scope, but here's the interface."* That shows scope control, not ignorance.

### 3.4 The whiteboard layout

```
┌────────────────────┬────────────────────────────────────┬─────────────────────┐
│ LEFT: KEEP VISIBLE │ CENTRE: THE STACK                  │ RIGHT: RUNNING LIST │
│ assumptions table  │ offline / nearline / online boxes  │ numbers (QPS, ms,   │
│ NFR / SLOs         │ each box: TASK → OUTPUT            │ GB, $)              │
│ the label ladder   │ arrows labelled with volumes       │ trade-offs accepted │
│ north star +       │ logs looping back to training      │ "raised myself" ✓   │
│ guardrails         │                                    │ sweep: 9 ticks      │
└────────────────────┴────────────────────────────────────┴─────────────────────┘
```

### 3.5 How deep to explain a technique — *three depths*

> **Default to depth 2. Have depth 3 ready, and go there only when asked.** A bare name sounds like a buzzword and invites a probe; a full derivation spends minutes the sweep needs.

| Depth | When | Example: inverse-propensity weighting (~time) |
|---|---|---|
| **1 · Name** | Never alone | *"I'd use IPW."* ❌ sounds memorised |
| **2 · Name + mechanism + caveat** | **The default**, in the flow of the answer | *"Position bias: slot 1 is tapped more because it's seen more. I'd shuffle chip order on 1–2% of traffic to measure how often each slot is seen, then weight each tap by one over that, clipped so no weight exceeds ~10."* (~15 s) |
| **3 · Derivation + trade-offs** | When they probe ("how does that work?", "why is it unbiased?") | The propensity table, E[tap/κ] = P(useful), variance and clipping, per-surface κ, the shallow-tower alternative (R1 §5.2) (~60–90 s) |

---

## PART 4 · THE 32 POINTERS, MADE GENERIC — *your self-scoring checklist*

> Use the ☐ column in mock interviews: tick only what you **said**. Under 28/32 means re-run the spine.

### A · System Architecture / Design

| ☐ | Code | The grader wrote (R1) | **The generic move (any question)** | Say it like this | R2–R7 examples |
|---|---|---|---|---|---|
| ☐ | **A1a** | Retrieval + ranking over generation in the critical path; generation as fallback | Make the **critical-path decision** explicit: cheap, controllable and evaluable in the path; expensive and flexible behind a gate | *"___ wins on latency, cost, control and evaluability; ___ wins on ___, so it's a gated fallback."* | R4 seed anchor · R6 freshness tiers · R7 slow φ, fast head |
| ☐ | **A1b** | Per-surface P95 budgets | A latency budget **per surface and per stage** that sums, plus capacity maths | *"P95 ___ ms; per stage: ___ + ___ + ___ = ___, the rest is tail headroom."* | R3 two-song budget · R4 autoplay = remaining runtime · R6 63 of 100 ms |
| ☐ | **A1c** | Tiered confidence-gated serving instead of compromising quality | When the budget is tight: **tier / cascade / precompute**, never shrink quality for everyone | *"I wouldn't lower quality across the board; I'd tier it."* | R4 pre-ranker + seed cache · R6 deadline cascade · R7 stale-not-wrong |
| ☐ | **A2a** | The full inference stack | Draw the **end-to-end stack**: gate → context/filters → candidates → rank → re-rank → safety → response → logs | Boxes with TASK and OUTPUT, in 90 seconds | Every sheet's §4 |
| ☐ | **A2b** | Two-tower + GBDT / light neural ranker | **Name each model** and what it scores | *"Retrieval: ___; ranking: ___; re-rank: ___."* | R4 MMoE + shallow tower · R7 neural-linear head |
| ☐ | **A2c** | Data sources | A table of sources, **each with its trap** | *"Five sources, and the trap in each."* | R5 orders + returns · R6 served-feature logs |
| ☐ | **A2d** | Staged training pipeline | Stages from raw data to deploy, **with cadences** | *"Clean → label → train → index → gate; models daily, fresh tier in minutes."* | R6 the feature store · R7 the decision record |
| ☐ | **A3a** | Top-down, unprompted | Announce the plan; follow it; summarise at each probe | *"I'll go: ___. Stop me where you want depth."* | Every opening |
| ☐ | **A3b** | Diversity reranker with a λ objective | Resolve a trade-off with **a formula, a knob, and an owner** | *"score = λ·relevance − (1−λ)·similarity, λ tuned online"* / *"hard constraints first, then value"* | R3 constraints + DPP · R4 additive value score · R6 metric per ms · R7 Thompson sampling |
| ☐ | **A4a** | Covered latency/surfaces *upon triggering* | 4–6 questions **in minute 1**, each with what it decides; then an assumptions table | *"Before I design: ___? It decides ___."* | Every §1 |
| ☐ | **A5a** | Retrieval vs generation on 5 dimensions | An **alternatives table on named dimensions**, a choice, the trade | *"On control, cost, latency, coverage and evaluability, ___ wins; I accept ___."* | R4 eight decisions · R7 seven algorithms |
| ☐ | **A5b** | A full generation pipeline on demand | Have **one complete alternative pipeline** ready end to end | *"If ___ fails or is unavailable, here's the whole path: ___ → ___ → ___."* | R3 taste-cluster · R4 seed-only · R5 first-session · R6 ladder · R7 ε-greedy v0 |

### F · AI Fluency

| ☐ | Code | The grader wrote (R1) | **The generic move** | Say it like this | Examples |
|---|---|---|---|---|---|
| ☐ | **F1a** | Two-tower, contrastive loss, in-batch negatives; GBDT with pairwise loss | Per subproblem: **model + loss + why it fits the label** | *"The label is ___ (binary / bounded / one-of-N), so the loss is ___."* | R4 one loss per label · R5 ESMM |
| ☐ | **F1b** | recall@k, NDCG, MRR, diversity, coverage, calibration | Each metric **with what it's for**; available vs meaningful | *"Recall@k for retrieval, NDCG for ranking, calibration because the gate/blend uses the number."* | R4 the offline–online ledger |
| ☐ | **F1c** | Position, selection, feedback, popularity + mitigations | A **bias table**: the four standard ones + the domain's own | *"Only shown items have labels…"* | R4 duration · R5 price, seller size · R6 leakage · R7 delayed feedback |
| ☐ | **F2a** | Retrieval underperforming → generation fallback | Raise **"the primary path fails on a slice"** yourself, with a fallback | *"If ___ underperforms on ___, I'd ___."* | R4 logQ dial · R6 stale features → defaults |
| ☐ | **F2b** | Latency too tight → tiered serving | Raise **the latency squeeze** yourself | *"If ___ doesn't fit in ___ ms, I won't shrink it; I'd ___."* | Every §4 |
| ☐ | **F2c** | Fine-tune an existing encoder on accepted/dismissed pairs | **Fine-tune a pretrained model** (LoRA/PEFT) on your own feedback pairs; warm-start | *"LoRA on the existing encoder, contrastive on accepted vs dismissed."* | R4 content encoder · R7 φ from the upstream towers |
| ☐ | **F3a** | Quantization: cheaper, faster, quality trade | An **efficiency lever with an explicit quality trade**, gated on eval | *"int8/fp16 or distillation; cheaper and faster; gated on ___."* | R6 the lever table |
| ☐ | **F3b** | Embedding index, light rankers, precomputed embeddings | **Scale levers**: precompute · index · shard · cache · cascade | *"Only the query side runs online; everything else is precomputed."* | R4 row-wise Adagrad memory · R7 additive statistics |
| ☐ | **F3c** | Multimodal extension: cost + embedding maturity risk | An **extension with cost and maturity risk**, staged | *"It adds ___ cost and ___ is less proven, so I'd start with ___."* | R4 semantic IDs · R6 on-device · R7 slates |

### I · AI Product Integration

| ☐ | Code | The grader wrote (R1) | **The generic move** | Say it like this |
|---|---|---|---|---|
| ☐ | **I1a** | Per-surface latency + cost/request + P95 tracking | P95/P99 per surface and stage · **$ per 1,000 requests** · name the cost driver | *"The ___ is the cost line; I track $/1k requests per tier."* |
| ☐ | **I1b** | Availability factored into the design; tiered serving | Availability **shapes a design choice**; a fallback ladder; never blank | *"On failure: ___, never ___."* |
| ☐ | **I1c** | Two surfaces; 3–5 diverse results with metadata | What the user sees · diversity · **controls that act fast** · reasons | *"A hide acts on the next response."* |
| ☐ | **I1d** | Controlled inventory, index, precomputed embeddings | The scale story **with numbers** | *"___ GB index, sharded; ___ QPS headroom."* |
| ☐ | **I1e** | Guardrails + privacy-leakage metric; moderate personalisation | Minimisation · retention · **deletion propagation** · no sensitive inference · a metric | *"No per-user parameters, so deletion is immediate."* |
| ☐ | **I1f** | In-path filters, policy tags, unsafe and reputation metrics | In-path filters · **policy as constraints, not weights** · stricter bar for automatic actions · a metric | *"A finite weight eventually loses to a big enough engagement gain."* |
| ☐ | **I1h** | Validation, OAuth, rate limits | The gateway **+ the domain's attack** (Part 6) | *"Beyond the gateway: ___."* |
| ☐ | **I1i** | Bias identification, mitigation, calibration | Identify → mitigate → **calibrate per segment**; disclosure for paid or own items | *"Exposure parity and ECE per segment."* |
| ☐ | **I1j** | *(blank in the sample)* | **Three audiences**: user · engineer · policy | *"A reason on the item, provenance for on-call, an audit log for policy."* |
| ☐ | **I2a** | Offline (NDCG, MRR) → shadow → A/B; corrects for shown-only labels | Offline **debiased** → shadow → ramped A/B → long-term holdout | *"Offline on the randomised slice or with IPS, because logs only label what we showed."* |
| ☐ | **I2b** | Acceptance/CTR + downstream completion + retention | **Proxy up AND meaningful up AND long-term not down, guardrails held** | *"Acceptance up, completion up, retention not down, guardrails held."* |

---

## PART 5 · THE FOLLOW-UP PLAYBOOK

### 5.1 The format for any follow-up — *VERDICT · MECHANISM · LAYERS · PROOF · TRADE*

```
VERDICT    the answer in one sentence ("Not on that number alone." / "The join breaks first.")
MECHANISM  why it happens, specifically for this system
LAYERS     2–4 fixes, CHEAPEST FIRST (no-cost → small cost → bounded cost)
PROOF      the metric or check that confirms the diagnosis or the fix
TRADE      what you accept, said out loud
```

### 5.2 The twelve follow-up archetypes — *every R-sheet follow-up is one of these*

| # | They ask | The skeleton answer |
|---|---|---|
| 1 | **"Metric X went up. Ship it?"** | Not on X alone → the meaningful metric · guardrails · slices · novelty (long-term holdout) · the mechanism (*"tapped more but completed less = clickbait"*) |
| 2 | **"Offline improved, online is flat."** | Labels are the old policy's (check the randomised slice / OPE) · position leakage · calibration shifts the blend · the metric never predicted A/Bs (the ledger) · the gain lives in a small slice |
| 3 | **"What breaks first at 10×?"** | Not the maths → **state** (joins, caches) → **log volume** → **the single writer** (shard it; are the statistics additive?) → feature reads. Give numbers |
| 4 | **"Dependency X fails."** | The **degradation ladder**, tier by tier · never blank · filters apply at every tier · **fallback rate as an SLO** · *"degraded is a product decision, not an outage"* |
| 5 | **"How do you know it's healthy / worked?"** | Four groups: **value vs baseline holdout** · **learning honesty** (calibration) · **loop health** (coverage, concentration) · **machinery** (lag, completeness, latency). Each has a threshold and an action |
| 6 | **"It keeps recommending the same popular items."** | Which **term** causes it (positives, logQ restoring the prior, reward weights) · the dial · mixed negatives · quotas · **recall by popularity decile** |
| 7 | **"A brand-new item / user."** | Item: policy gate → content tower (ID dropout) → priors from neighbours → **exploration with propensities** → graduation. User: context + referrer → segment popular → **session model** → bandit mix → graduation |
| 8 | **"The business wants to boost X."** | A **policy decision, not a ranking tweak**: disclosed, capped, a **constraint** the user's interest still governs, owned and reported separately |
| 9 | **"Why not just use an LLM / one model / something simpler?"** | Compare on control, cost, latency, coverage, evaluability → when you **would** → *"as a gated fallback / a quota'd source / v0"* |
| 10 | **"Latency is over budget."** | **Measure per stage first** → no-cost fixes (parallelise, batch, hedge, in-process) → small-cost (distil, quantise, fewer candidates) → bounded (cascade on the slow tail) |
| 11 | **"Bias / fairness?"** | Identify (exposure, per-segment error) → mitigate (exploration, caps, quotas) → **calibrate per segment** → audit metric |
| 12 | **"Bots / adversaries."** | The **domain's** attack → filter **before training** → trust-weight signals → velocity anomalies → contain (gates, rollback, subtract contributions) |

---

## PART 6 · THE SWEEP TEMPLATE — *nine rows, generic, fill per question*

| Dimension | What to say (generic) | Metric to name |
|---|---|---|
| **Performance** `I1a` | P95/P99 per surface and per stage · the cost driver · $/1k requests per tier | P95 per stage · $/1k |
| **Reliability** `I1b` | Availability **shaped a design choice** · timeouts → next tier · the ladder · never blank | Fallback rate per tier · availability |
| **Usability** `I1c` | What the user sees · diversity · controls that act on the **next** response · reasons · no repeats or flicker | Dismiss/hide rate · repeat rate |
| **Scalability** `I1d` | Precompute · index · shard · cache · cascade, **with the numbers** | QPS headroom · index/stream lag |
| **Privacy** `I1e` | Minimise · retention limits · **deletion propagates** (stores, logs, retrain) · no sensitive inference · consent settings · k-anonymity for anything mined from users | Leakage rate · deletion latency |
| **Safety** `I1f` | In-path filters · policy as **constraints** · a stricter bar for automatic actions (autoplay, auto-send) · suppression in sensitive contexts · kids | Unsafe rate · reputation incidents |
| **Security** `I1h` | Validation, OAuth, rate limits **+ beyond the gateway** (pick the domain's): | Blocked attacks · anomalies caught |
| | · **LLM:** prompt injection via context; no URLs in generated output | |
| | · **Recsys:** fake engagement poisoning labels; co-engagement farming; lookalike uploads | |
| | · **Marketplace:** fake reviews; keyword stuffing; scraping | |
| | · **Online learning:** steering the learner; gate + caps + rollback | |
| | · **Media:** codec exploits (sandboxed decoding); misuse as a person-finder | |
| **Ethics / bias** `I1i` | Identify → mitigate → **calibrate per segment** · exposure for new or small creators and sellers · disclosure for paid or own items | Exposure parity · ECE per segment |
| **Explainability** `I1j` | **Three audiences**: the **user** (a reason on each item, from provenance) · the **engineer** (logged features, versions and seed, so every output is replayable; per-head attributions) · **policy / regulators** (an audit log; disclosure; a non-personalised option where required) | % items with a reason · % replayable |

> ⭐ Close the sweep with one line that ties it together, e.g. *"Every fast path needs a fast brake."* or *"A suggestion is an endorsement."*

---

## PART 7 · THE BUILDING-BLOCKS KIT — *when you need X, say Y*

### 7.1 Retrieval and ranking

| Need | Say | The one-line why |
|---|---|---|
| Retrieve from 10⁶–10⁹ items | **Two-tower**, dot product, item side precomputed, ANN index | Only the query tower runs online |
| …its loss | **Sampled softmax, in-batch negatives, logQ correction**, + uniform negatives | logQ fixes **sampling** bias, not popularity (R4 FU 1) |
| Retrieve from ~10⁴ items | **Exact dot product over an in-memory catalogue** | 50k × 64 floats = 13 MB, ~2 ms; no recall loss (R6) |
| Item-to-item | **PMI-normalised** co-occurrence; co-view = substitutes, co-purchase = complements | Raw co-counts recommend hits (G17, R5) |
| Rank with conflicting goals | **Multi-task MMoE**, BCE per head, **negative heads** (dismiss, hide, regret) | No negative head ⇒ can't learn a watch was unwanted |
| Combine heads | **An additive value score** of calibrated heads; serving weights **owned** | Explainable per head; a product decision, not a config nudge |
| Purchase / conversion | **ESMM**: pCTCVR = pCTR × pCVR over all impressions | CVR on clicked rows has selection bias (R5) |
| Watch time | **Quantile within a duration bucket**, soft-label BCE | Seconds reward length; bounded gradient (R4) |
| Position bias | **Shallow position tower** (dropped at serving) · IPS on a randomised slice | Learn "slot 1 is good" otherwise |
| A slate, not a list | **MMR / DPP** under hard constraints | *"Promises go in constraints; taste goes in the objective."* |
| Cheap → expensive | **Pre-ranker distilled** from the ranker; cascade | Spend the heavy model where it matters |

### 7.2 Data, labels, freshness

| Need | Say |
|---|---|
| A training set from logs | Eligibility (examined only) → clean (bots first) → label (the ladder) → debias (IPS / randomised slice) → balance (cap heavy users) → **time split** |
| Labels that arrive late | Matured windows + a **fast-proxy bridge** weighted by how often it became the truth + delay modelling (G22, G15) |
| Train/serve consistency | **One feature definition, two stores; log features at serving; point-in-time joins (≤ t − lag) for new features** (R6) |
| Freshness | Separate **features (seconds) · items (minutes) · weights (hourly, gated) · full retrain (daily)** (R6) |
| Cold start: items | Policy gate → **content tower with ID dropout** → priors from neighbours → **exploration with propensities** |
| Cold start: users | Context + referrer → segment-popular across categories → **session model** → bandit mix → graduation |
| Fine-tune existing | **LoRA on a pretrained encoder**, contrastive on your accepted/rejected pairs; warm-start daily |

### 7.3 Exploration and online learning

| Need | Say |
|---|---|
| Explore | **Thompson sampling** over model uncertainty (neural-linear or ensembles); not ε-greedy, which explores blindly |
| Evaluate a new policy offline | **Logged propensities** + IPS / **doubly robust**; clip weights; **replay** on a uniform slice for learning policies (R7) |
| The non-negotiable | *"Propensities can't be backfilled: a day-one field."* |
| Drift | **Time-based decay** (uncertainty never collapses) + covariance inflation on detected change |
| Defend exploration | *"It costs now and pays later; short tests always say explore less"* → a **long-horizon holdout** + catalogue coverage |

### 7.4 Serving, efficiency, reliability

| Need | Say |
|---|---|
| Latency too tight | **Tiered / confidence-gated** serving · deadline-aware cascade · precompute · prefetch (stream-ahead, page 2, track 8) |
| Cheaper | **Quantise** (int8/fp16) · **distil** · prune features by ablation, each **gated on eval** |
| Caching | Item-level in-process · user-level by freshness · candidate-level event-invalidated · result-level 30–60 s · **filters (hide, seen, policy) run AFTER every cache** · versions in the key |
| Failure | Timeouts + circuit breakers + bulkheads + load shedding → **a degradation ladder** that is never blank |
| Training at scale | **Row-wise Adagrad for embeddings, AdamW for dense** (R4) · additive sufficient statistics shard exactly (R7) |

### 7.5 Evaluation and rollout

```
OFFLINE    time-split · debiased (randomised slice / IPS / DR) · SLICED (popularity, new items/users)
           · calibration · "an offline metric earns its job by predicting past A/Bs" (the ledger)
SHADOW     live traffic, nothing shown: latency, safety flags, overlap, calibration
A/B        user-level · power first · 1% → 5% → 50% · ≥ 1–2 weeks · CUPED · guardrails
           ⚠ marketplace interference → item/cluster randomisation (R5)
           ⚠ continuously trained arms → per-arm training data (R6, R7)
INTERLEAVE for ranker-vs-ranker (fast, low variance)
HOLDOUT    long-term (months): retention, exploration payoff, novelty vs habit (D7 vs D28)
```

---

## PART 8 · RECOGNISE THE ARCHETYPE — *any prompt maps to one of nine*

| Archetype | Examples in the pack | **The decision that drives everything** | **The available → meaningful trap** | **The signature risk** |
|---|---|---|---|---|
| **Feed / recommendation** | G13, G14, G15, G16, R3, R4, R5 | The objective + the label ladder; the funnel | CTR / watch time → satisfaction, retention | Feedback loops, popularity, rabbit holes |
| **Search / retrieval** | G2, G3, G6, G7, R2, Q6 | What "relevant / similar" means; the unit of the index | Clicks → task success, purchase without return | Tail queries; latency; freshness |
| **LLM product feature** | R1, Q2, Q13, Q22, Q31 | What sits in the critical path: retrieval vs generation, model size | Acceptance → downstream success | Safety, hallucination, injection, cost |
| **Classification / trust & safety / fraud** | G25, G27, G28, G29, G10 | The operating point and the **cost asymmetry** of errors | Precision/recall → harm prevented, appeals | Adversarial adaptation; label lag; fairness |
| **Prediction / regression** | G21, G22, G31, G19 | Calibration and how the number is used downstream | AUC → calibrated, business-correct decisions | Delayed labels; drift |
| **Real-time / infra-heavy** | R6, Q7, Q8, Q29 | The freshness tiers and the latency budget | — | Skew, staleness, silent lag |
| **Online learning / bandits** | R7, G20, Q26 | The exploration budget and **propensities** | Short-term reward → long-horizon value | Ossification; steering; short-test bias |
| **Modelling-deep** (losses / optimizers) | R4, Q1, Q4, Q25 | Label type → loss; parameter type → optimizer | — | Sampling bias; instability; leakage |
| **Vision / multimodal** | G1, G8, G9, G11, G12, R2 | The **invariances** and the unit (frame, moment, region) | — | Labelling cost; privacy (faces, people) |

> **In the first 60 seconds, silently classify the prompt.** Then the archetype tells you the DECISION sentence, the OBJECTIVE trap, and which risk to raise yourself.

---

## PART 9 · NUMBERS TO HAVE READY

```
TIME          1 day = 86,400 s ≈ 10⁵ s     → avg QPS ≈ daily requests / 10⁵ · peak ≈ 2–3× avg
TRAFFIC       10⁸ DAU × 10 requests = 10⁹/day ≈ 10⁴ QPS avg, ~3×10⁴ peak
RANKER LOAD   QPS × candidates   (10⁵ × 500 = 5×10⁷ scorings/s → the cost line)
EMBEDDINGS    N × d × bytes: 10⁹ × 64 × 1 B (int8) = 64 GB · 5×10⁴ × 64 × 4 B = 13 MB
EXACT SEARCH  N × d multiply-adds: 5×10⁴ × 64 = 3.2×10⁶ ≈ 1–2 ms on one core
OPTIMIZER     Adam = 2 extra floats per parameter · row-wise Adagrad = 1 float per ROW
LOGS          requests × items shown × bytes   (2×10⁸ × 20 × 1 KB = 4 TB/day raw)
STREAMS       DAU × events/user/day ÷ 10⁵ = events/s   (10⁷ × 200 → ~2×10⁴/s)
JOIN STATE    events/s × window × bytes   (4×10⁴ × 300 s × 2 KB = 24 GB)
A/B POWER     n ∝ 1 / MDE²: half the detectable effect needs 4× the traffic

LATENCY       typeahead ~50 ms · search / feed 100–200 ms P95 · async suggestions 300–500 ms
              · a bandit selection layer ≤ 10 ms · LLM generation: hundreds of ms+
              → if an LLM is in the path, say how its latency is HIDDEN (streaming, prefetch)
```

---

## PART 10 · THE PHRASE BANK — *L6 lines that earn pointers*

| Line | Earns |
|---|---|
| *"Before any boxes, one decision drives the rest: ___."* | A1a, A3a |
| *"___ is available; ___ is meaningful."* | F1b, I2b |
| *"A tap that leads nowhere is worse than no suggestion."* | I2b |
| *"I wouldn't lower quality across the board; I'd tier it."* | A1c, F2b |
| *"Showing nothing is a legitimate output."* | I1b, I1f |
| *"Only shown items have labels."* | F1c, I2a |
| *"A model with no negative head can't learn that an engagement was unwanted."* | F1a |
| *"Promises go in constraints; taste goes in the objective."* | A3b, I1f |
| *"The weights are a product decision with an owner."* | A3b, I1i |
| *"Diversity comes from candidate generation; you can't re-rank variety into a homogeneous set."* | A2a, F1c |
| *"logQ fixes sampling bias, not popularity."* | F1a, F1c |
| *"Propensities can't be backfilled."* | I2a, A2d |
| *"Exploration costs now and pays later; short tests always say explore less."* | F1b, I2a |
| *"Offline screens; online decides. An offline metric earns its job by predicting past A/Bs."* | I2a, F1b |
| *"Log what you serve."* | A2d, I1j |
| *"A degraded list is a product decision, not an outage."* | I1b |
| *"Every fast path needs a fast brake."* | I1h, I1f |
| *"Explainability has three audiences."* | I1j |
| *"Fix the objective, not the model."* | A1a, I2b |

---

## PART 11 · WHAT SEPARATES THE LEVELS

| Level | What the answer looks like |
|---|---|
| **L5** | Correct components (two-tower, ranker, A/B), standard metrics, answers each probe when asked |
| **L6** | **Drives** the hour: a plan, an explicit critical-path decision, **available vs meaningful**, failures raised before being asked, alternatives with named trades, numbers everywhere, the nine-dimension sweep, a success definition with guardrails, and what ships first and why |
| **L7** | Frames the organisational and ecosystem incentives: who owns the objective's weights, why exploration and logging need **institutional** defence (holdouts built before anyone asks), and which standards (propensity logging, served-feature logs) should be platform-wide rather than per team |

---

## PART 12 · THE DAY-OF PAGE — *read this in the last ten minutes*

```
MINUTE 0   classify the archetype (Part 8) → pick the DECISION, the OBJECTIVE trap, the TWIST
MINUTE 1   ASK: 4–6 questions, each "(decides ___)" → ASSUMPTIONS table out loud
           OPEN: DECISION · OBJECTIVE · TWIST · PLAN  →  "stop me wherever you want depth"
EVERY PROBE  DECIDE · COMPARE · BUILD · BREAK · MEASURE
§2   the ladder: available → meaningful, + negatives; north star + guardrails
§3   numbers: QPS, per-stage ms that SUM, GB, $/1k, THE TRICK
§4   alternatives table → stack (TASK/OUTPUT) → ⚠ RAISE 3 FAILURES YOURSELF → trades
§5   sources + trap · labels · model + LOSS + why · LoRA on existing · staged pipeline
§6   quantise/distil (quality trade) · precompute/shard/cache · objective with a knob · ladder
§7   offline (debiased!) → shadow → A/B → holdout · "success = proxy ∧ meaningful ∧ long-term ∧ guardrails"
§8   NINE one-liners, each with a metric; security BEYOND the gateway; explainability = 3 audiences
§9   extension (cost + maturity risk) · a full alternative pipeline · top 3 failures · Q1 ships
FOLLOW-UPS   VERDICT · MECHANISM · LAYERS (cheapest first) · PROOF · TRADE
SHORT ON TIME   protect §8 and the success definition
```

> ⭐ **The sentence to carry in:** *"I'll make one decision explicit, measure what's meaningful rather than what's available, raise the failures before you ask, and finish with production readiness, because a design isn't done until it can fail safely and prove it worked."*

---

## PART 13 · EVERY METRIC AND EVAL IN THE PACK — *a glossary*

> Collected from all the `*_intv.md` sheets (G, Q and R series). **Sheets** names where each is used, so you can see it in context. Grouped by what the metric is **for**: an interviewer cares less that you know the name than that you put it in the right job (screen, north star, guardrail, health check).
>
> **Sheet codes:** G1 visual search · G2 video search · G3 photos NL search · G4 autocomplete · G6 internal doc search · G7 shopping ranking · G8 street-view blurring · G13 YouTube recs · G14 Discover feed · G15 Play Store · G17 similar items · G21 ad clicks · G22 delayed conversion · G23 ads eval framework · G25 harmful content · G29 fraud · G31 ETA · Q1 100B training · Q2 coding agent · Q4 RL post-training · Q5 eval harness · Q6 agent RAG · Q7 inference serving · Q8 data pipeline · Q9 agent memory · Q10 agent safety · Q12 LLM judge · Q13 support agent · Q19 prompt injection · Q26 continuous learning · R1–R8.

### 13.1 Offline: retrieval and ranking quality

| Metric | What it measures | Sheets |
|---|---|---|
| **Recall@k** | Share of queries where the right item is anywhere in the top k candidates. **The retrieval metric**: a miss here can't be fixed downstream | R1, R2 (@100), R4, R5, R6, R8 (@50), Q6 (@100), G2, G3, G13 |
| **Recall@k, sliced** | Recall computed per slice: popularity decile, new items, new users, filter selectivity, device tier, group | R4, R5, R6, G3, G6, G17, R2 (fairness slices) |
| **Precision@k / precision@1** | Share of the top k (or the top 1) that are correct | G1 (same-product P@1), R8 (P@5) |
| **NDCG@k** | Ranking quality with **graded** relevance, discounting by position: rewards the best items being highest | R1, R2, R5, R6, R8, G2, G7, Q6 |
| **MRR** (mean reciprocal rank) | Average of 1 / (rank of the first correct result); good when there's one right answer | R1, G4 (rank of the query the user finally issued) |
| **mAP** (mean average precision) | Precision averaged over every position where a relevant item appears | R2 |
| **Temporal IoU** | Overlap between the returned time span and the true one (intersection ÷ union) | R2 |
| **ANN recall vs exact search** | How much the approximate index loses against brute force; no labels needed | R1 (recall@200 vs exact), R8 (≥ 0.95) |
| **Answer-span containment** | Share of gold answer spans that sit inside **one** chunk; tests chunking | R8 |
| **Context recall** | Is the gold chunk still in the final prompt after dedupe and compression? | R8 |
| **Human relevance judgements** | Raters grade results on a stratified query sample; the offline gate when clicks are biased | G2, G7, G17 ("is this similar?"), R2 (expert pairs) |
| **Recall per stratum, with confidence intervals** | Recall reported separately for each difficulty stratum, each with a CI | G8 |
| **Recall of future organic discoveries** | On held-out weeks: did the listener later find, on their own, tracks we'd have recommended? | R3 |

### 13.2 Offline: probability and prediction quality

| Metric | What it measures | Sheets |
|---|---|---|
| **AUC** (ROC-AUC) | Probability a random positive scores above a random negative: ranking ability only, blind to calibration | G13, G14, G21, G22, R4, R5, R6 |
| **PR-AUC** | Area under precision–recall: the right ranking metric when positives are rare | G29 |
| **Log loss** (binary cross-entropy) | Penalises confident wrong probabilities; sensitive to calibration | G21, G22 |
| **Normalised entropy** | Log loss ÷ the entropy of the base rate: comparable across datasets with different base rates | R4, R5, R6 |
| **Calibration** (ECE, predicted/observed) | Do predicted probabilities match observed frequencies? Needed whenever a number is thresholded, multiplied or priced | R1, R2, R4, R5, R6, R7, G21 (< 2% per slice), G22 (per elapsed-time bucket), G25, G29, Q13 |
| **Uncertainty calibration** | Do the model's X% intervals contain the outcome X% of the time? | R7, G31 |
| **Precision / recall per class or policy** | Per category, never aggregated, because an average hides the weak policy | G25 |
| **Precision at the operating point** | Precision at the threshold or capacity actually used (e.g. what reviewers can handle) | G29 |
| **MAE / MAPE** | Mean absolute (percentage) error of a numeric prediction; reported, rarely gating | G31 |
| **Pinball (quantile) loss · interval width** | Sharpness of a quantile forecast: calibrated **and** narrow | G31 |
| **Lateness rate** | How often the real arrival came after the surfaced ETA quantile | G31 |
| **κ (Cohen's kappa)** | Agreement between two raters beyond chance; how an LLM judge is validated against humans | Q12, R8, Q5 |

### 13.3 Slate, ecosystem and feedback-loop metrics

| Metric | What it measures | Sheets |
|---|---|---|
| **Diversity** (intra-list distance, category or topic entropy) | How different the items in one list are from each other | R1, R4, R6, G14 |
| **Coverage** | Share of the catalogue (or categories) that gets shown at all; falling = the loop is narrowing | R1, R5, R6, R7, G2 (tail coverage), G17 (items with ≥ K neighbours) |
| **Context diversity@10** | Distinct performers and venues among relevant results (the requirement in R2) | R2 |
| **Near-duplicate / seed-duplicate rate** | Share of results that repeat each other, or the item on screen | R2, R4 |
| **Novelty share · repeats = 0** | Share of tracks the listener has never heard; no track recommended twice | R3 |
| **Popularity / impression concentration** | Share of impressions going to the top 1% (or a Gini coefficient) | G1, G17, R3 (artist), R4, R5, R6, G7 (seller), G14 (publisher) |
| **New-creator / new-app / small-seller / emerging-artist share** | Exposure going to newcomers: whether the cold-start door stays open | G13, G15, R3, R4, R5 |
| **Exposure parity by group** | Exposure and acceptance compared across locales, creator groups or regions | R1, R3, R5 |
| **Time to first impression for new items** | How fast a new item gets shown at all | R6, R7 |
| **Topic narrowing · trajectory diversity · autoplay-chain drift** | Whether a user's feed, or a chain of autoplays, is collapsing into one topic or drifting toward borderline content | G14, G13, R4 |
| **Induced-flow share** | Traffic on a road caused by our own routing: the ETA feedback-loop monitor | G31 |
| **Accuracy on the exploration slice** | Whether the main model is only right about what it chose to show | R4 |

### 13.4 Online: north stars and proxies

| Metric | What it measures | Sheets |
|---|---|---|
| **CTR / acceptance rate** | Clicks or taps per impression. The **available** metric, gameable, never the north star alone | R1, R4, R5, R6, R7, G13, G2 |
| **Downstream completion** | Did the conversation or task the click started actually succeed? | R1 |
| **Dismiss / hide / "not interested" rate** | Explicit negative feedback per impression | R1, R4, R5, R6, R7, G14 |
| **Retention: D7 / D28, return rate** | Share of users active again 7 / 28 days later, or returning next week | R1, R6, G13, G14, R4 (7-day return) |
| **Survey satisfaction · regret rate** | "Was this worth your time?": the direct satisfaction label | G13, R4 |
| **Satisfied watch time** | Watch time weighted by predicted satisfaction | R4 |
| **Watch-through · successful-search rate** | The user found it: watched past the matched part, saved, or rewatched | G2, R2 |
| **Reformulation / re-query / rephrase rate** | The user tried again, a volunteered failure signal | G2, G3, R2, R8 |
| **Zero-result rate** | Queries returning nothing | G3, Q6, R8 |
| **Sticky discoveries · save rate · ≥ 30 s stream · skip-through** | New tracks that lasted; fast proxies; whole-playlist skipping as a negative | R3 |
| **Net purchases · purchase without return · add-to-cart** | Purchases kept, not returned; add-to-cart as a fast, bridged proxy | R5, G7 |
| **D7-retained installs · uninstall within 24 h** | Installs that stuck, and installs regretted | G15 |
| **Attach rate · placement-appropriate outcome** | Complements bought together; the outcome right for each placement | G17 |
| **Keystrokes saved · suggestions leading to a successful search** | Effort saved by autocomplete, counted only when it led somewhere | G4 |
| **Resolution rate · repeat-contact rate · CSAT · escalation quality** | Problem fixed (no repeat contact in 7 days), satisfaction, and whether the human had to re-ask | Q13 |
| **Resolved rate · answer correctness · groundedness · citation precision / recall** | RAG success: the answer was right, supported by its cited sources, and cited correctly | R8, Q6, G6 |
| **Abstention rate on answerable vs unanswerable** | Says "I don't know" when it should, and not when it shouldn't | R8, Q6, G1, G6 |
| **Mean reward per decision vs a baseline** | The bandit's added value over simply taking upstream's top pick | R7 |
| **Expected loss per transaction · approval rate · chargeback rate** | Fraud loss plus the cost of declining good customers, in currency | G29 |
| **Prevalence per policy · appeal overturn rate** | Harm that reached people (view-weighted); how often enforcement was wrong | G25 |
| **Revenue that holds · advertiser ROI · delivery against target CPA** | Platform revenue that isn't taken out of advertisers' returns | G21, G22, G23 |
| **Merged PRs not reverted · pass^k · cost per solved task** | Coding-agent work that stuck; reliability across k attempts; the bill | Q2 |
| **Task success with defences on vs off · injection success rate · over-refusal** | Security that doesn't break the product | Q19, Q10 |
| **Repeated-question rate · write precision · correction rate** | Agent memory working without storing wrong things | Q9 |
| **Goodput per $ · TTFT · ITL / TPOT · p99/p50 TPOT · prefix-cache hit rate** | Serving: useful tokens per dollar, first-token and per-token latency, fairness, cache efficiency | Q7, R8 (TTFT) |
| **Loss vs scaling-law prediction · MFU · goodput** | Is a large training run on trajectory and using the hardware? | Q1 |
| **Reward by component · KL · entropy · zero-variance group fraction** | RL run health: learning the right thing, not drifting or collapsing | Q4 |
| **Ablation scorecard · token yield · dedup rate** | Data-pipeline quality, judged by training a small model on the corpus | Q8 |

### 13.5 Safety, privacy and trust guardrails

| Metric | What it measures | Sheets |
|---|---|---|
| **Unsafe / harmful suggestion rate** | Policy-violating outputs, from audited samples | R1, G4 |
| **Privacy leakage rate · identity-leakage rate** | Another user's data surfacing; the model matching the person instead of the content | R1, R2 |
| **Reputation incidents** | Screenshot-worthy failures | R1 |
| **ACL / permission leaks = 0 · cross-user leakage = 0** | Forbidden content retrieved or revealed; tested in CI with canary documents and outsider users | R8, Q6, G6, Q9 |
| **Deleted content cited = 0 · deletion lag · revocation p99** | Deleted or revoked content truly gone, within the SLA | R8, Q6, G6 |
| **Borderline / policy-violating impressions** | Exposure to content near the policy line | G13, R2, R4 |
| **Hallucinated-claim rate** | Claims not supported by any cited source | R8 |
| **Wrongful-action rate · harmful high-tier actions = 0** | Agents taking actions they shouldn't | Q13, Q10 |
| **Refusal rate (both directions) · authorisation-denial rate · approval latency** | Over- or under-refusing; attack signals; whether a human approval gate is real or rubber-stamped | Q10, Q19 |
| **Injection persistence = 0** | Untrusted content managing to write into durable memory | Q9, Q19 |
| **Suppression recall** | Sensitive topics correctly kept out of a feed, tested in CI | G14 |
| **Substantiated recall ≥ 99.9% · over-blur rate** | Blurring every face and plate, provably; not blurring what doesn't need it | G8 |
| **Decline-rate parity · overturn rate by language** | Error rates compared across groups: fairness | G29, G25 |
| **Contamination = 0** | Eval data leaking into training data | Q1, Q5, Q8 |

### 13.6 System and pipeline health

| Metric | What it measures | Sheets |
|---|---|---|
| **p95 / p99 latency, per surface and per stage** | Tail latency, where users feel slowness | Every sheet |
| **Cost per 1,000 requests ($/1k)** | Unit cost, tracked per tier or route | R1, R2, R5, R6, R8, G1, G2, Q6, Q7 |
| **Availability · error rate · fallback rate per tier** | Up, failing, or quietly degraded | R2, R6, R7, R8 |
| **Freshness lag · event-to-feature lag · stream lag** | How stale the index or features are | R6, R8, Q6, G14 |
| **Feature distribution shift (PSI, null rates)** | Inputs drifting from what the model trained on | R6 |
| **Training–serving skew** | Served features disagreeing with training features | R5, R6 |
| **Join completeness · logging completeness** | Outcomes joined to decisions; decisions logged with propensities | R7, Q26 |
| **Minimum propensity · effective sample size of the weights** | Whether off-policy estimates are still trustworthy | R7 |
| **Posterior age · condition number** | Online-learner health: stale or numerically unstable | R7 |
| **Label maturity · conversion coverage** | Whether delayed labels have arrived; the share of conversions observable at all | G22, R5 |
| **Queue depth and age** | Review capacity keeping up | G25 |
| **DLQ rate · parse failures by doc type** | Documents silently failing ingestion | Q6, R8 |
| **Proxy ↔ truth correlation** | Whether the fast proxy still predicts the slow truth; a decline is an incident | G7, G15, R3, R5 |
| **Offline ↔ online correlation** | Whether offline metrics predict A/B outcomes; the eval system's own KPI | G23, R4 (the ledger), Q5 |

### 13.7 Evaluation **methods** — *how* you measure, not *what*

| Method | What it is | Sheets |
|---|---|---|
| **Time-split (temporal) evaluation** | Train on the past, test on the future; a random split leaks the future | R4, R6, G25, G29 |
| **Matured-cohort evaluation** | Wait until delayed labels (purchases, returns, retention) arrive before judging | G7, G15, G22, R5 |
| **Gold sets** (human, expert, exhaustive, stratified) | Independently labelled evaluation data, because clicks only label what was shown | R2, R8, Q6, G1, G8 |
| **Bootstrapped labels** | Synthetic QA from the corpus → a human-verified core → mined production data → an unanswerable set | R8 |
| **LLM-as-judge, calibrated** | A model grades outputs, trusted only once validated against humans (κ), with bias checks | R8, Q12, Q5 |
| **Randomised / exploration slice** | A small share of traffic with randomised order or choices: unbiased labels and propensities | R1, R3, R4, R7, G13, G14 |
| **Inverse propensity weighting (IPS / IPW)** | Re-weight logged outcomes by 1 / P(shown) to remove position or selection bias | R1, R3, R7, G21 |
| **SNIPS · doubly robust** | Lower-variance off-policy estimators: self-normalised, or a reward model plus correction | R4, R5, R6, R7 |
| **Off-policy evaluation (OPE)** | Estimate a new policy's value from old logs, before showing it to anyone | R3, R7, Q26 |
| **Replay** | Re-run logged traffic through a candidate (prefix logs, contacts, or a learning bandit on a uniform slice) | R7, G4, G15, G17, Q13 |
| **Shadow** | The new system runs on live traffic and logs, but nothing is shown | Nearly every sheet |
| **Canary** | The new system serves a tiny slice, with automatic rollback | R1, R6, Q7, Q19 |
| **A/B test + ramp** | Randomised controlled experiment, ramped 1% → 5% → 50% | Every sheet |
| **CUPED · power analysis · layered experiments · A/A · SRM checks** | Variance reduction; sizing; running experiments side by side; validating the setup | R4, R5, R6, R7, G23 |
| **Interleaving** | Merge two rankers' results into one list and credit each by clicks; fast, low-variance | R1, R2, R4, R6, R8, G2, G4, G13 |
| **Long-term holdout / holdback** | A small group kept on the old system (or without the feature) for months: novelty vs real value | R1, R3, R4, R5, R6, R7, G13, G15, G21, Q9, Q26 |
| **Item / cluster / region randomisation** | For marketplaces and traffic, where user-level arms interfere | R5, G31 |
| **Per-arm training data** | Continuously trained arms each learn only from their own traffic | R6, R7 |
| **Champion / challenger · allow-through** | Fraud: the challenger decides a few percent; a small allow-through gives unbiased labels for what's normally blocked | G29 |
| **Label-only rollout** | Enforcement shown as interstitials before removals, since leaving harm up can't be A/B tested | G25 |
| **Red-team suites** | Attack sets as blocking gates; every found exploit becomes permanent | Q10, Q19, Q9 |
| **Regression suite from production failures** | Every triaged failure becomes a test case, labelled with its stage | R8, Q19 |
| **Canary documents / probes** | Synthetic items created, edited, deleted or permission-restricted, then probed end to end | R8, G6 |
| **Contamination checks** | Canary strings, n-gram overlap, and the fresh-vs-old delta | Q5, Q8, Q1 |
| **Anchor set · frozen reference · frozen holdout** | Fixed items or a fixed model re-scored over time: if the score moves, the eval changed | Q12, Q5, Q26 |
| **Mass probe** | Check that weights actually moved before interpreting an eval | Q4 |
| **Pre-agreed kill criterion** | Decide in advance when to stop a costly run, before sunk cost argues back | Q1 |
| **Noise floor** | The smallest difference an eval can detect; a result inside it is not a result | Q4, Q5, Q8, Q26 |
| **Offline–online ledger** | Every launch's offline deltas next to its A/B outcome: which metrics earn the right to gate | R4, G23 |

### 13.8 Scoring building blocks that get confused with metrics

> These **build or select** results; they don't **evaluate** them. The trap is the acronyms: **MRR** measures a ranking, while **MMR** and **DPP** build a diverse set.

| Technique | What it is | Where it's used | Sheets |
|---|---|---|---|
| **PMI** (pointwise mutual information) | log [ P(a,b) / (P(a)·P(b)) ]: how much more often two items co-occur than chance. Turns raw co-counts, which hits dominate, into real associations | Item-to-item candidate sources; a ranker feature; choosing training pairs | R3, G17 |
| **NPMI** (normalised PMI) | PMI scaled to −1…1, so rare pairs with one lucky co-occurrence don't dominate | Same, when the tail is noisy | R3 |
| **MMR** (maximal marginal relevance) | Greedy re-ranking: λ·relevance − (1−λ)·similarity to the **single closest** item already picked | Diversity re-rankers | R1, R2, R4, R5, R6 |
| **DPP** (determinantal point process) | Scores a **whole set** by the determinant of its quality-and-similarity matrix: high only when items are good **and** unlike each other. Its **log-det** can also be reported as a set-diversity measure | Slate selection, where the set is the product | R3 |
| **logQ correction** | Subtract log Q(item) from logits in sampled softmax, to undo popularity-biased negative sampling. It fixes **sampling** bias, not popularity | Two-tower retrieval losses | R1, R3, R4, R5 |
| **RRF** (reciprocal rank fusion) | Σ 1/(k + rank) across result lists: merges BM25 and dense rankings without calibrating their scores | Hybrid retrieval | R8 |
| **Thompson sampling** | Sample from the posterior, act greedily on the sample: explores in proportion to the chance each option is best | Bandits, exploration slots | R3, R6, R7 |

> **Say:** *"Every metric I name has a job: offline metrics screen, the north star decides, guardrails block, and health metrics tell me the measurement itself is still trustworthy."*
