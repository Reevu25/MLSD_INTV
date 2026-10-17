# Q5 · Model-Promotion Gating / Eval Harness — **INTERVIEW SHEET**

> **Tier 1 · #8** — Google weights MLOps heavily; *"how do you know it's better?"*
> Full study file: [`../Q5_eval_reward_harness.md`](../Q5_eval_reward_harness.md) · [diagrams](../Q5_eval_reward_harness_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product (scorecard + gate) looks like · (c) the harness · (d) the eval data · (e) choosing and curating suites · (f) training the judge · (g) running it · (h) how you know the eval works · (i) the extension.

> ⚠ **SCOPE THIS IN THE FIRST 60 SECONDS.** There are two deliverables — the **offline gating harness** (the spine) and the **online RL reward path** (the branch). Only one fits. Ask §1 Q1 and build offline; give the reward path two minutes at the end (§9).

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | **Offline gate or RL reward?**, what it blocks, the human-labelling loop (§1) |
| **Explainability** is easy points | Judge **justification before score**; an **evidence package** with every verdict (§8) |
| **Security** beyond the gateway | **Untrusted model code** in the sandbox; hidden test sets that must not leak into training (§8) |
| **Troubleshoot before being asked** | Raise the **cost squeeze** and **"passed, but worse in production"** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **When CI breaks, tests fail. When an eval breaks, it keeps producing numbers** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | ⚠ **Scoping Q first** · assumptions · the eval's **own** SLOs · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The scorecard + gate verdict · κ, noise floor, contamination · pass^k | I1c, F1b, I2b |
| **9–12** | **CAPACITY + COST** | — | Cost per pass → **tiering** → the **cache key** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Scorecard vs composite · registry → cache → runners → 4 scorers → gate | A2a, A5a, A1c, F2b |
| **22–32** | **DATA + THE JUDGE** | **(d) (e) (f)** | Suites · curation + contamination · the judge, its biases, its training | A2b–d, F1a, F1c, F2c |
| **32–38** | **RUNNING IT** | **(g)** | Noise floor · `pass@k` vs `pass^k` · hard vs soft gates · sandboxes | F3a, F3b, A3b, I1b |
| **38–44** | **TRUST + ROLLOUT** | **(h)** | Contamination controls · **frozen reference model** · did the eval work? | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multimodal judges · the RL reward branch · fallback · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Evaluation is CI for models, and it deserves the same discipline: hermetic, versioned, deterministic where possible, a cheap tier and an expensive tier, and a gate with teeth.**
>
> **But there's a difference that makes it harder, and it's the thing I'd design around: when CI is broken, your tests fail. When an eval is broken, it keeps producing numbers. A contaminated benchmark, a biased judge, a harness bug that silently drops half the cases — none of these raise an error. They produce a plausible score, and then every training decision and every promotion runs on it.**
>
> **So my design principle is: the eval is a system under test, with its own regression suite, its own SLOs, and its own monitoring."**

**Then the scoping move + three priorities:**
> *"Offline gating or online RL reward? I'll build the **offline gating path as the spine** — but the RL loop shares the task definitions, the scoring logic and the judge, and **keeping them the same code is a correctness requirement**, because a reward that differs from the eval means training toward one thing and measuring another."*

1. **Trustworthiness** — contamination, judge bias, noise.
2. **Throughput and cost** — a full agentic eval is expensive enough to shape how often you run it.
3. **Reproducibility** — *a number you can't reproduce isn't a measurement, it's an anecdote.*

**Then say the plan (`A3a`):** *"The scoping question and assumptions, what the gate produces and the eval's own metrics, cost and tiering, the architecture, the data and the judge, how runs are scored statistically, how we trust the eval, a production-readiness sweep, then the RL branch and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked — #1 is the scoping question:**
1. **⚠ Offline gating, online reward, or both?** *(Different SLOs; only one fits in the room.)*
2. **What decisions does this block?** *(If it gates weight promotion it's a **blocking gate**, not a dashboard.)*
3. **Is there a human labelling loop?** *(Determines whether the judge can be calibrated at all.)*
4. **Which capabilities and modalities?** *(Text, code, agentic, multimodal each need different scorers.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Deliverable | The **offline promotion gate** for an LLM family; RL reward as a branch |
| Suites | ~10,000 tasks: QA, code, agentic, safety, multilingual; private and versioned |
| Scorers | Exact/programmatic · execution · LLM judge · human preference |
| Cadence | Smoke on every commit · full nightly · gate at promotion |
| Humans | A rater pool with overlapping assignments, so agreement is measurable |
| Consumers | Research teams, the release owner, safety reviewers |

**FR:** run any model version against any suite → versioned, reproducible, comparable result · four scorer types (exact, execution, **judge**, human) · **gate promotions** with a declarative policy + evidence package · report contamination, judge disagreement and noise **alongside every score** · version everything.

**NFR / SLO — these are the SLOs of the *eval itself*:**

| Dimension | Target |
|---|---|
| **Judge–human agreement** | **κ ≥ 0.6**, per rubric, re-measured every release |
| **Reported noise floor** | Every headline number ships with a CI |
| **Contamination check** | Every suite, every model, **blocking** |
| **Reproducibility** | Same (model_sha, suite_ver, harness_ver, seed) → identical score |
| Smoke tier | < 20 min (every commit) · Full tier < 4 h |
| Cost | Tracked in **$ per run per tier** |

**Scope / NON-GOALS:**
- ❌ **A single "overall model quality" score.** *"I'd actively refuse — it destroys actionability and hands the org one thing to Goodhart."*
- ❌ Authoring the eval content (I build the pipeline; domain experts write tasks) · training the reward model · running the annotation workforce.

> **Say the refusal out loud.** Interviewers reward a candidate who declines to build the thing that would obviously be misused.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
RESEARCHER, after a commit:   "T0 smoke: PASS — no suite moved beyond its noise floor"
                              reproduce: eval run --model abc123 --suite T0@v14 --seed 7

RELEASE OWNER, at promotion:  GATE VERDICT: BLOCKED
                              reason: safety-refusal suite −2.1 pts (floor ±0.8) — HARD GATE
                              scorecard: reasoning +3.0 ±0.9 · code +1.2 ±1.1 · multilingual −0.4 ±0.7
                              contamination: clean · judge κ: 0.71 (ok) · harness self-test: pass
                              evidence: 37 regressed tasks, side-by-side outputs, judge rationales
```

**The eval's own metrics:**

```
THE EVAL IS ACCOUNTABLE, NOT ASSUMED. Its own metrics:

  JUDGE–HUMAN AGREEMENT  Cohen's κ, PER RUBRIC, per suite
      ⚠ not raw % (lies under class imbalance)
      ⚠ and HUMAN INTER-RATER AGREEMENT IS THE CEILING —
        a judge at 85% vs 70%-consistent labels is uninterpretable

  NOISE FLOOR  measured empirically per suite (same model, several seeds)
      → published beside every score
      → "delta must exceed the floor" ENCODED IN THE GATE POLICY

  CONTAMINATION STATUS  shipped with every score, blocking

  pass@k SAYS THE CAPABILITY EXISTS.  pass^k SAYS IT'S RELIABLE.
      Users get one attempt → gate agentic suites on pass^k.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Gate verdicts that predict production outcomes** (promoted models win their A/Bs) | The eval's real job |
| Trust | κ per rubric · noise floor · contamination status · frozen-reference drift | Whether any number can be believed |
| Throughput | Smoke < 20 min · full < 4 h · cache hit rate | Whether people actually run it |
| **Guardrails** | Hard gates (safety, canaries, contamination, κ) never outvoted · $ per run | Hard limits |

---

## 3 · CAPACITY + COST *(min 9–12)* — **⭐ the cache key is the detail to say** `A1b` `I1a` `I1d`

```
TIERING (the direct consequence of cost):
  T0 SMOKE  ~200 tasks, STRATIFIED   every commit   ~$15, 20 min
  T1 FULL   ~10,000 tasks            nightly        ~$500, 3 h
  T2 GATE   full + human slices      at promotion

⚠ T0 is a REPRESENTATIVE stratified sample — NOT the tasks the last
  checkpoint failed. Its job is catching CATASTROPHIC regressions, and a
  failure-curated set is blind exactly where the last model was strong.

THE RESULT CACHE — highest-ROI infrastructure here:
  key = SHA256( model_sha + task_id + task_ver + prompt_template_ver
              + decode_params + HARNESS_VER + judge_model + rubric_ver )

  ⚠ WHY harness_ver IS IN THE KEY — the part people get wrong:
     without it, a scorer BUGFIX leaves stale wrong results cached
     forever, silently contaminating every later comparison.
     With it, fixing a bug AUTOMATICALLY invalidates every result the
     bug touched. Very hard to retrofit → a day-one decision.

  Observed hit rates: 60–80% during active development.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| What the gate reports | One composite score vs **a scorecard with CIs + a binary verdict** | Scorecard: a composite destroys actionability and invites Goodhart (FU 6) |
| How free-form output is scored | Humans only vs **an LLM judge, calibrated against humans** vs programmatic only | Judge for scale, **calibrated** and checked by canaries; programmatic wherever a checker exists |
| Reward vs gate | Share the judge model vs **share the library and rubric, not the model** | A judge used as a reward becomes an adversarial target |
| Benchmarks | Public only vs **private, versioned, refreshed suites** | Public sets leak into training |

### The system — *draw this in 90 seconds* `A2a`

```
 MODEL VERSION + SUITE VERSION
       │
┌─ 1. SUITE REGISTRY — IMMUTABLE VERSIONS ───────────────────────
│ tasks · rubrics · checkers · ⚠ changing a task CREATES A NEW VERSION
│ OUTPUT: the exact task set for this run
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. SCHEDULER — ⚠ CACHE LOOKUP BEFORE DISPATCH ────────────────
│ tiering · cost-bucketed sharding (an 80k-token agentic task and a
│ 3k QA task don't share a shard)
│ OUTPUT: only the uncached work, sharded
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RUNNER FLEET + SANDBOXES (hermetic) ───────────────────────
│ fresh container per task · no network · resource limits
│ OUTPUT: model outputs + execution traces
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. FOUR SCORERS BEHIND ONE INTERFACE ─────────────────────────
│ exact/programmatic · execution · JUDGE · human preference
│ ⚠ THIS LIBRARY IS SHARED WITH THE RL PATH (the judge MODEL is not)
│ OUTPUT: per-task scores + judge rationales
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. IMMUTABLE RESULT STORE (+ CIs, noise floor) ───────────────
│ OUTPUT: comparable, reproducible results
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. DECLARATIVE GATE ──────────────────────────────────────────
│ HARD GATES: safety · canaries · contamination · judge κ · harness self-test
│ ⚠ CAPABILITY CANNOT OUTVOTE A HARD GATE
│ OUTPUT: PASS / BLOCKED + the evidence package
└────────────────────────────────────────────────────────────────
 + FROZEN REFERENCE MODEL re-scored periodically
   ⚠ its score must NEVER change. If it does, THE EVAL CHANGED.
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The cost squeeze:** *"If the full suite is too slow or expensive for every commit, I don't shrink the gate. I tier it (T0 smoke on every commit, T1 nightly, T2 at promotion) and let the cache do most of the saving."*
- **"Everything passed, but production is worse":** *"Then either the eval changed or it doesn't cover production. The frozen reference model tells me which within one run (FU 5)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **`harness_ver` in the cache key** | "A scorer bugfix must invalidate every result that bug touched — otherwise stale wrong numbers contaminate comparisons forever" | Lower hit rate after harness changes |
| **2** | **Share the scoring library with RL, not the judge model** | "Same code and rubric is a correctness requirement. But the **judge model** must differ, or the policy optimises against the thing that approves it" | Two judges to calibrate |
| **3** | **Refuse the composite score** | "It destroys actionability and gives the org one thing to Goodhart" | Leadership wants one number; you owe them a scorecard instead |

---

## 5 · DATA + THE JUDGE *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The eval data (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Private, versioned suites** | Tasks + checkers + rubrics per capability | Go stale; saturate |
| **Freshly authored sets** | Uncontaminated by construction (written after model cutoffs) | Expensive |
| **Human ratings** (overlapping assignments) | Ground truth for the judge; inter-rater agreement | The ceiling on what anything can measure |
| **Production samples** (consented, PII-scrubbed) | Coverage of what users actually do | Privacy |
| **Red-team and safety suites** | Hard-gate material | Must be refreshed against current outputs |

### 5.2 Choosing and curating suites (e) `F1c`

```
T0 SMOKE        ~200 tasks STRATIFIED across capabilities — representative,
                not "what the last model failed"
CONTAMINATION   canary strings in every private task (grep any training corpus) ·
                n-gram overlap / membership probes · FRESH-vs-OLD delta
DISCRIMINATIVE  for each suite: when did it last change a decision? suites that
POWER AUDIT     never separate candidates are pure cost → retire them
SATURATION      a suite everyone scores 97% on is retired from the gate and kept
                as a regression tripwire (FU 10)
COVERAGE        slices by language, capability, and production traffic share
```

### 5.3 The judge — what it does, and its biases `A2b` `F1a`

**What it actually does here — say this before designing it:**
- **Offline (spine):** one of four scorer types, for tasks with **no programmatic checker** — free-form quality. Feeds the promotion gate.
- **Online (branch):** supplies the **RL reward** at ~20/s.
- ⚠ **Share the library and the rubric; do NOT share the model.** A judge used as a reward becomes an **adversarial target**; gating with that same model deletes the gate's independence.

**Rubric design:** decompose into dimensions · binary/few-point scales · **justification before the score** · anchored examples · **separate correctness from helpfulness** (or it rewards fluent wrongness) · **version it — a rubric change is a metric change**.

**Bias catalogue — name these:** position (score **both orders**, and the flip rate is a free quality metric) · length/verbosity · **self-preference** (never let a family judge itself in a gating comparison) · style-over-substance · sycophancy · score compression.

**⚠ Judge-hacking canaries:** fixed responses that **must score badly** — verbose-but-empty, fluent-but-wrong, well-formatted-with-no-substance, flattering, padded citations. If any score well, **the judge is broken and the gate fails**, regardless of the model.

### 5.4 Training the judge: the PEFT objective (f) `F2c`

If we train our own judge: start from an existing model and fine-tune it with **PEFT (LoRA)** on human-labelled data, using a **pairwise preference loss** (which of two responses is better) or a **per-rubric-dimension classification loss**. Then **calibrate** against held-out human labels, and require κ ≥ 0.6 before it gates anything. Distil it into a smaller judge for the T1 tier.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · AUTHOR     domain experts write tasks + checkers + rubrics → new suite version
STAGE 2 · CHECK      contamination controls · canaries · a dry run on the frozen reference
STAGE 3 · CALIBRATE  judge vs human labels per rubric → κ must clear the bar
STAGE 4 · RUN        tiered, cached, sandboxed
STAGE 5 · GATE       declarative policy → verdict + evidence package → release decision
```

---

## 6 · RUNNING IT *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

- **The noise floor:** re-run the same model with several seeds per suite, publish the floor beside every score, and encode *"delta must exceed the floor"* in the gate.
- **Reliability, not just capability:** `pass@k` says the capability exists; **`pass^k`** (succeeds on all k attempts) says it's reliable. Users get one attempt, so agentic suites gate on `pass^k`.
- **The gate policy** `A3b`: **hard gates** (safety, judge-hack canaries, contamination, judge κ, harness self-test) are categorical — **capability cannot outvote them**; soft metrics must clear their noise floor.
- **Efficiency** `F3a` `F3b`: the result cache (60–80% hits); cost-bucketed sharding; a **distilled / quantized judge** for T1, re-checked for κ after compression.
- **Reliability** `I1b`: hermetic sandboxes; flaky-task quarantine; the **frozen reference model** catches harness, judge or sandbox drift.

---

## 7 · TRUST + ROLLOUT *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

**Contamination — three controls:**
```
1. CANARY STRINGS in every private task → grep any training corpus for a
   DEFINITIVE answer. Costs nothing; the only check that gives certainty.
2. n-gram overlap / membership probes where you can(not) scan the data.
3. ⚠ FRESH-vs-OLD DELTA — performance on an old public set vs a FRESHLY
   AUTHORED set of equal difficulty. That gap IS the contamination
   estimate; works without training-data access, and it's showable to a VP.
```

**The frozen reference model:** a fixed checkpoint re-scored periodically. **Its score must never change.** If it moves, the *eval* changed — harness bug, judge drift, a task edited in place, a sandbox image update. One extra run; catches a whole class of otherwise-invisible failures.

**Rolling out a change to the eval itself** `I2a`: a new gate policy, judge or suite version runs in **shadow** beside the current one, and we compare verdicts on recent candidates before it takes over.

**Did the eval work?** `I2b`: track whether **gate verdicts predicted production outcomes** — did promoted models actually win their A/Bs, and would the blocked ones have lost? That offline-to-online agreement rate is the eval's own success metric.

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Judge biases** | Position, length, self-preference, style, sycophancy | Both-orders scoring; length controls; cross-family judges; canaries |
| **Contamination** | Test items in training data inflate scores | Canaries; overlap probes; fresh-vs-old delta |
| **Coverage** | "Better on average" hides "worse for some" | Slices by language and capability |
| **Saturation** | A maxed-out suite stops discriminating | Discriminative-power audit; fresh suites |
| **Human ceiling** | Raters disagree, so labels are noisy | Measure inter-rater agreement first |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Smoke < 20 min, full < 4 h; **$ per run per tier**; cost-bucketed sharding | Wall-clock per tier · $/run |
| **Reliability** `I1b` | **Frozen reference model** whose score must never change; harness self-tests; flaky-task quarantine | Reference-score drift · flake rate |
| **Usability** `I1c` | Researchers get a **scorecard with CIs**, a diff against the incumbent, and a **one-line reproduce command** | Time-to-answer · reruns per question |
| **Scalability** `I1d` | Result cache (60–80% hit rate); tiering; runner fleet with sandboxes | Cache hit rate · queue time |
| **Privacy** `I1e` | Eval sets built from user data need **consent and PII scrubbing**; human raters see only what's needed; access logs | PII scans · rater access audits |
| **Safety** `I1f` | **Safety suites are hard gates** that capability can't outvote; red-team suites refreshed against current outputs | Hard-gate pass rate |
| **Security** `I1h` | ⚠ **Hidden test sets** access-controlled, so they can't leak into training (contamination is also a security problem); sandboxed code execution (below) | Canary-string hits · sandbox violations |
| **Ethics / bias** `I1i` | **Coverage by language and user group**, so "better on average" can't hide "worse for some"; judge biases (length, position, self-preference) measured | Score by slice · judge bias metrics |
| **Explainability** `I1j` | Judges **write their justification before the score**; every gate verdict ships an **evidence package** (which tasks moved, which gates blocked, why) | % verdicts with evidence |

> ⭐ *"The eval is itself a production system: it has users (researchers), a security boundary (hidden test sets and untrusted code), and a success metric (did its verdicts predict reality)."*

> ⚠ **Worth saying in full: the execution scorer runs untrusted code.** Model-written code executes in the sandbox, so: **no network, resource limits, a fresh container per task**. A sandbox escape from an eval machine is a real security incident.

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multimodal outputs:** images, audio and video need a **VLM judge**, which is far less validated than a text judge. It gets its own κ against humans and more human slices until it earns trust.
- **The RL reward branch (two minutes):** the same task definitions and scoring library serve RL, but with **opposite SLOs** — a single pass at ~20 judgements/s, null-score-and-drop on failure — and a **different judge model** from the gate, so the policy can't optimise against its own approver.

**When the judge can't be trusted, the fallback pipeline** `A5b` `F2a`:

```
judge κ drops below the bar, a canary scores well, or the judge is down
  → GATE on verifiable scorers only (programmatic + execution) for this release
  → add a HUMAN-RATED slice for the free-form capabilities, sized to the decision
  → BLOCK promotion on the free-form suites until the judge is re-calibrated
  → log the gap in the evidence package so the release owner sees what wasn't measured
```

**Top 3 failure modes:**
1. **Silent harness bug** — drops half the cases, produces a plausible score, nobody notices. → golden-result regression suite for the harness itself.
2. **Contamination discovered late** — every historical comparison is retroactively invalid.
3. **Judge drift** — a judge-model update shifts every judged number; the quality trend is measuring the instrument.

**Q1 ships:** result store + immutable versioning + the cache (**with harness_ver**) + programmatic and execution scorers + the **noise-floor measurement**.
> *"Ship the objective half first — it's trustworthy by construction and it unblocks releases. The judge comes in Q2, after calibration, because **a judge you haven't calibrated is worse than no judge**."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "How do you know your eval isn't lying to you?"** `I2a` `F1c`
> Three threats, three controls. **Contamination**: canary strings in every private task for a definitive grep, n-gram overlap where I can scan the data, membership probes where I can't, and — most usefully — the **fresh-vs-old delta**: performance on an old public set versus a freshly authored set of equal difficulty. That gap *is* the contamination estimate, it works without training-data access, and it's in a form you can show a VP. **Judge bias**: κ against human labels **per rubric**, not aggregate; both-orders scoring for position bias; length normalisation; no self-judging; judge-hack canaries as a **hard gate**. **Noise**: measure the empirical floor per suite with several seeds, publish it beside the score, and encode "delta must exceed the floor" in the gate policy. Underneath all three: **the eval is a system that fails silently, so it gets its own regression tests and monitoring** — including a frozen reference model whose score must never change.

**2. "Your judge agrees with humans 85% of the time. Is that good?"** `F1b`
> It depends entirely on a number you haven't given me: **how often humans agree with each other.** If inter-rater agreement is 95%, then 85% is a real gap and the judge is missing something systematic. If humans agree only 70% of the time, then 85% agreement with a 70%-consistent ground truth is close to meaningless — the judge might be *better* than any individual rater, or fitting the noise, and you can't tell. So first I'd measure inter-rater reliability with overlapping assignments, because **it's the ceiling on everything else**. Second, I'd distrust the aggregate: 85% overall can be 95% on formatting and 60% on factual accuracy, hiding that the judge is useless at the dimension you care most about — so κ **per dimension and per suite**. Third, raw percentage is the wrong statistic under class imbalance: if 80% of items are "acceptable," an always-acceptable judge scores 80% having learned nothing. **Cohen's κ, per dimension, against a measured human ceiling.**

**3. "Why is `harness_ver` in the cache key?"** `I1b` `A2d`
> Because the cache stores *what you ran → the score it got*, and the key must name **everything that could change the score** — including the code that computes it. That's the part people miss, since the scorer doesn't feel like an input. Concretely: if an answer-extraction bug is fixed in February, every result cached in January was computed by broken code and **never recomputed**, because nothing in its key changed. You then compare a March model against a January number and see a breakthrough that's actually a scorer difference — with nothing in the system telling you. With `harness_ver` in the key, fixing the bug **automatically invalidates every result that bug touched**. And it's very hard to retrofit: without it you have a cache full of results with no record of which scorer produced them, so you can't selectively invalidate — you throw everything away or keep known-wrong comparisons. Day-one decision.

**4. "Design the rubric for judging helpfulness."** `F1a` `I1j`
> I'd decompose it, because "helpfulness" as a single holistic score is exactly what judges are unreliable at. Separable dimensions: **instruction-following**, **factual accuracy** given available evidence, **completeness**, and **appropriate length**. Each gets a **binary or few-point scale with anchored examples**, because fine-grained scales have poor inter-rater reliability for humans and models alike. I'd require the judge to **write its justification before emitting the score**, so evidence precedes verdict rather than rationalising it. I'd **version the rubric**, because a rubric change is a metric change and results across versions aren't comparable. And critically I'd **separate helpfulness from correctness** — a confidently wrong answer can seem extremely helpful, and if the rubric conflates them the judge learns to reward fluent wrongness. Then calibrate against human labels and don't let it gate anything until κ clears the bar.

**5. "Everything passed, but the model is clearly worse in production. What broke?"** `F2a` `I2a`
> I'd have built the mechanism for this: the **frozen reference model**, a fixed checkpoint re-scored on every suite periodically. Its score should never change, so if it moved I have a timestamp bounding when the *eval* changed. From there it's a bisection over what can change: harness version, prompt template version, judge model, rubric version, sandbox image, task content. **Immutable task versioning** eliminates one whole class immediately, because a task *cannot* have been edited in place. The **golden-result regression suite** catches scorer bugs directly. And because `harness_ver` is in the cache key, I can tell exactly which historical results a code change invalidated rather than guessing. If the eval is clean, then it's a **coverage gap** — production traffic contains something the suite doesn't — and the fix is a fresh suite authored from production distribution, not more of the same benchmark.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Leadership wants one number for model quality. Give it to them."** `A5a` `A3a`
> I'd decline the composite and offer something better, and I'd make the argument in their terms rather than as a purity objection. Collapsing to one number **destroys the information that makes evals actionable** — a single score that drops tells you nothing about what to do — and more importantly it hands the entire org **one thing to Goodhart**, which reliably produces optimisation against the aggregate rather than against quality. What I'd give instead: a **scorecard with confidence intervals** — a small number of suite-level scores, each with its noise floor, plus contamination and judge-agreement status — and a **declarative gate policy** that says what blocks a release. That answers the real question underneath the request, which is "can I tell at a glance whether we're okay to ship," without creating a metric that will be gamed within two quarters. If pressed I'd offer a **binary gate verdict** — passed / blocked, with the blocking reason — which is one number and is honest.

**7. "The safety team says the RL model got more capable and also more willing to help with disallowed requests. Your gate passed. What broke?"** `I1f` `A3b`
> The gate treated safety as a **soft metric that capability could outvote**, and that's a structural bug in the policy rather than a measurement failure. Capability and safety regressions frequently co-occur precisely because the same post-training pressure that raises helpfulness lowers refusal, so a weighted-average gate will happily trade one for the other and report a net improvement. The fix is **hard-gate semantics**: safety suites, judge-hack canaries, contamination status and judge κ are categorically different from capability metrics — **capability cannot outvote a hard gate**, and no aggregate weighting exists between them. Then the second question is whether the safety suite had **coverage** of the specific behaviour, which is a different failure: if the RL objective found a region the suite doesn't sample, the gate was measuring the wrong thing. That's an argument for **adversarially-authored safety suites refreshed against current policy outputs**, since a static suite goes stale exactly as the policy learns to exploit it.

**8. "Your eval costs $500 a run and researchers want it on every commit. Resolve that."** `F2b` `A1c`
> Tiering plus caching, and then a conversation about what's actually earning its cost. **T0 smoke**, ~200 stratified tasks at ~$15 and 20 minutes, runs on every commit and catches catastrophic regressions — deliberately *representative* rather than curated from past failures, because a failure-curated set is blind exactly where the last model was strong. **T1 full** nightly, **T2 gate** at promotion. Then the **result cache**, which typically hits 60–80% during active development because researchers re-run the same suites while iterating on something unrelated — so adding five tasks costs five tasks, not ten thousand. Beyond that I'd run a **discriminative-power audit**: for each suite, when did it last change a decision? Suites that never differentiate between candidate models are pure cost, and they accumulate because everyone adds and nobody removes. What I would *not* cut is the fresh held-out set or the calibration sample — those are what make the remaining numbers mean anything.

**9. "You use the same judge for the RL reward and the promotion gate. Isn't that circular?"** `A5a` `F1c`
> **Structurally, yes — and that's precisely why I don't share the model.** The scoring library and the rubric *are* shared, because that part is a correctness requirement: if RL trains against one implementation and the gate measures with another, you optimise toward one thing and certify against another. But the **judge model** should be a different family, or at minimum a judge held out of the RL loop entirely, plus **verifiable slices in the gate that no judge touches** — programmatic and execution scorers are ungameable in a way no judge is. The reason is that the moment a judge becomes a reward it stops being a measurement and becomes an **adversarial target**: the policy finds its blind spots reliably, and a gate sharing those blind spots cannot see it. Even with independence I'd defend in depth, because two judges sharing a rubric can share a blind spot: **judge-hack canaries as a hard gate**, the **frozen reference model**, and safety hard-gates capability can't outvote.

**10. "Your best benchmark saturated — everyone scores 97%. Now what?"** `I2b` `F2a`
> Saturation means the benchmark has stopped **discriminating**, which makes it worthless for its actual job even though the number looks great — and I'd say that plainly, because teams keep reporting saturated benchmarks long after they've stopped informing decisions. Three responses. **Retire it from the gate** and keep it only as a regression tripwire — a drop still means something even when a rise doesn't. **Author a harder fresh suite** from the failure modes the current models still exhibit, which the discriminative-power audit will have identified, and author it *after* the models' cutoffs so it's provably uncontaminated. And **shift the measurement axis**: when correctness saturates, the remaining signal is usually in **reliability rather than capability** — which is where `pass^k` beats `pass@k`, since users get one attempt and a model that succeeds 97% of the time on one try but 60% across five consecutive attempts is not actually solved. I'd also treat saturation as a **prompt to check contamination**, because a sudden jump to 97% is at least as likely to be leakage as progress.

---

## ⭐ THE L6 LINE

> **"When CI breaks, tests fail. When an eval breaks, it keeps producing numbers — so the eval is a system under test, with its own SLOs, regression suite, and a frozen reference model whose score must never change."**

## 🧭 HOOKS — *steer by what you mention*

- *"The eval fails silently"* → invites the reliability/testing branch
- *"Human inter-rater agreement is the ceiling"* → invites the annotation branch (Q27 territory)
- *"harness_ver is in the cache key"* → invites the infra/reproducibility branch
- *"Capability cannot outvote a hard gate"* → invites the policy/org branch (**good at L6**)
- *"A judge used as a reward becomes an adversarial target"* → invites the Q12 deep dive
- *"I'd refuse to build a composite score"* → invites the metrics-design branch (**good at L6**)

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three threats"* — contamination / judge bias / noise regenerates the answer.
- **Running out of time?** → *"I have five minutes — I'd rather cover contamination and the noise floor than finish the sharding. Sound right?"*

**Name and defer:**
- "**The judge as a platform** — training, distillation economics, drift — is a whole separate design."
- "**Agentic multi-turn evaluation** needs trajectory-level scoring, not per-response."
- "**Human annotation operations** — rubrics, agreement, reviewer calibration — is its own system."
- "**Online reward serving** for RL has opposite SLOs: single-pass, null-score-and-drop on failure."
- "**Cost-bucketed sharding** matters — an 80k-token agentic task and a 3k QA task in one shard wrecks wall-clock."
- "**Evaluating third-party models** is useful later; not the v1 that unblocks our releases."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: cheap verifiable scorers first, a calibrated judge where needed | §0, §4 | *"Evaluation is CI for models: hermetic, versioned, tiered, with a gate with teeth."* |
| **A1b** | Cost and latency budgets | §1, §3 | *"T0 ~$15 in 20 min on every commit; T1 ~$500 in 3 h nightly."* |
| **A1c** | Degrade without lowering the gate's quality | §4 | *"I don't shrink the gate; I tier it and let the cache do the saving."* |
| **A2a** | The full stack | §4 diagram | Registry → scheduler + cache → sandboxes → four scorers → store → gate |
| **A2b** | Model components | §5.3 | The judge: rubric, bias controls, canaries |
| **A2c** | Data sources | §5.1 | Private suites, fresh sets, human ratings, production samples, red-team |
| **A2d** | Staged pipeline | §5.5 | Author → check → calibrate → run → gate |
| **A3a** | Top-down, unprompted | §0 | *"When an eval breaks, it keeps producing numbers"* + the scoping move + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Hard gates are categorical; capability can't outvote them."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Offline vs RL · what it blocks · human loop · modalities |
| **A5a** | Alternatives compared | §4, FU 6, 9 | Composite vs scorecard; judge vs human vs programmatic; shared judge vs shared library |
| **A5b** | A full alternative pipeline on demand | §9 | Verifiable-only gate + a human slice when the judge can't be trusted |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right tool per subproblem | §5.3 | *"Justification before the score; separate correctness from helpfulness."* |
| **F1b** | Metrics and their trade-offs | §2, FU 2 | *"κ, not raw agreement; pass^k, not pass@k, for agents."* |
| **F1c** | Bias sources and mitigations | §5.3, §7 | Position, length, self-preference, contamination, coverage |
| **F2a** | Something underperforms → a fallback | §4, §9, FU 5 | The frozen reference model; the verifiable-only fallback |
| **F2b** | Too slow / costly → tiering | §4, FU 8 | Tiers + cache, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A LoRA judge with a pairwise preference loss, then calibrated |
| **F3a** | Efficiency with a quality trade | §6 | *"A distilled, quantized judge for T1, re-checked for κ."* |
| **F3b** | Scale via caching and sharding | §3, §6 | The cache key with `harness_ver`; cost-bucketed shards |
| **F3c** | Extension, with cost and maturity risk | §9 | VLM judges; the RL reward branch |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §1, §8 | Wall-clock per tier; $ per run |
| **I1b** | Reliability | §6, §8 | The frozen reference; harness self-tests |
| **I1c** | Usability | §2, §8 | A scorecard with CIs and a one-line reproduce command |
| **I1d** | Scalability | §3, §8 | Cache hit rate; runner fleet |
| **I1e** | Privacy | §8 | Consent and PII scrubbing for user-derived sets |
| **I1f** | Safety | §6, §8, FU 7 | Safety suites as hard gates |
| **I1h** | Security | §8 | **Untrusted code in sandboxes**; hidden test sets protected |
| **I1i** | Ethics / bias | §7, §8 | Coverage by language and group; judge bias metrics |
| **I1j** | Explainability | §5.3, §8 | Judge rationales; the evidence package |
| **I2a** | Offline → online, and rolling out eval changes | §7 | Shadow a new gate beside the old; contamination controls |
| **I2b** | The success definition | §7 | *"Gate verdicts that predict A/B outcomes."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) curation → §5.2 · (f) judge training → §5.4 · (g) running it → §3, §6 · (h) does the eval work → §7 · (i) extension → §9.
