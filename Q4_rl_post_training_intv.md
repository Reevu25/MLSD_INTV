# Q4 · RL Post-Training (RLHF / RLVR) — **INTERVIEW SHEET**

> **Tier 2 · #17** — post-training is standard at L6, and one of the two highest-frequency prompts in the agentic bank.
> Full study file: [`../Q4_rl_post_training.md`](../Q4_rl_post_training.md) · [diagrams](../Q4_rl_post_training_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product (a checkpoint + evidence) looks like · (c) the RL system · (d) datasets · (e) curating the task pool · (f) the objective · (g) running the loop · (h) how you know the model is better · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Reward source? Single-turn or agentic? What gates the weights? (§1) |
| **Explainability** is easy points | **Per-component reward logs** and an evidence package per promotion (§8) |
| **Security** beyond the gateway | **Sandbox escape** from policy-written code; the grader kept **outside** the environment (§8) |
| **Troubleshoot before being asked** | Raise the **idle environment fleet** and **zero-variance groups** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | The output is **a better policy + the evidence it's better** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · KL tripwire · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + THE REWARD** | **(b)**, (h) | The evidence package · ⭐ **verifiable first, decomposed always** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + COST** | — | ⚠ **The bottleneck is rollouts, and it's the ENV fleet** · ~$60k a run | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · tasks → actors → envs → reward → learner → gate | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & ALGORITHM** | **(d) (e) (f)** | Task pool for gradient · PPO vs GRPO · KL as the leash · LoRA vs full | A2b–d, F1a, F1c, F2c |
| **32–38** | **RUNNING THE LOOP** | **(g)** | Async collection · off-policy tolerance · weight sync · checkpoints | F3a, F3b, A3b, I1b |
| **38–44** | **HACKING, GATE, ROLLOUT** | **(h)** | ⚠ *"Goodhart is a schedule"* · the ladder · the gate before the trainer | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multi-turn credit · process rewards · the rejection-sampling fallback | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The output of this system isn't a service — it's a better policy, plus the evidence that it's better. So I'd optimise three things in priority order.**
>
> **One, a reward signal we can trust and that resists gaming. Everything else is downstream of this: a fast pipeline optimising a hackable reward produces a worse model faster.**
>
> **Two, rollout throughput — RL is bottlenecked on generating experience, not on gradient steps. And for an agent the generators sit idle waiting on tools, so the environment fleet, not the GPU fleet, is usually the limiter. Three, an eval that gates weights before anything ships, because a run costs on the order of $60k and shipping a subtly worse model costs much more."**

**Then the reward commitment:**
> ⭐ **"Verifiable first, decomposed always."** Prefer rewards you can *check* — tests pass, the maths is right, the schema validates — over rewards you *judge*. And never a single scalar: **log the components separately, because a single scalar is how hacking hides.**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what a run produces and how the reward is built, where the time and money go, the architecture, data and algorithm, how the loop runs, reward hacking and the gate, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What's the reward source — human preferences, verifiable checks, or a judge?** *(Determines everything; verifiable changes the whole risk profile.)*
2. **Single-turn or agentic multi-turn?** *(Multi-turn means an environment fleet and a credit-assignment problem.)*
3. **What gates the weights, and who owns it?** *(If there's no gate, I'd build that before the trainer.)*
4. **What model, and what compute budget per run?** *(Decides LoRA vs full fine-tuning and how many experiments fit.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Starting point | An SFT'd instruction model (~30–70B) |
| Rewards | **Verifiable** for code and maths (tests, checkers); a **preference RM** for open-ended helpfulness |
| Tasks | Single-turn plus **agentic** tasks with tools (code execution, search) |
| Budget | ~$60k of compute per full run |
| Gate | The eval harness from Q5, owned outside the RL team |
| Users | Research teams launching runs |

**FR:** generate rollouts at scale · score them · update the policy · **hold a reference model for KL** · gate promotion · detect reward hacking · reproducible runs.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Rollout throughput** | The primary system metric — *not* GPU utilisation |
| **KL to reference** | Bounded; a hard tripwire, not a soft preference |
| **Reward-hack canaries** | **Hard gate** — every discovered exploit becomes permanent |
| Run reproducibility | Same (data, seed, config, code) → same curve |
| Gate cost | ≪ the ~$60k run cost |
| Resilience | Checkpoint and resume across preemptions; no lost run |

**Scope / NON-GOALS:**
- ❌ **A single scalar reward.** Position: decomposed with per-component monitors.
- ❌ **The judge as the primary reward where a checker exists.** Verifiable first.
- ❌ Pretraining (Q1) · the eval harness itself (Q5, consumed here) · serving (Q7).

---

## 2 · PRODUCT + THE REWARD *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What a run produces (b):**

```
RUN rl-code-0412  ·  base sft-v9  ·  spec: grpo, group 8, β = 0.02, 400 steps
  reward by COMPONENT: correctness ↑ 0.41 → 0.58 · format 0.97 (flat) · safety 1.00
  held-out VERIFIABLE slice: 52% → 61%   ← the number the reward couldn't game
  KL to reference: 4.1 (tripwire 8) · entropy healthy · zero-variance groups 6%
  mass probe: weights moved (Δ‖W‖ well above noise)
  GATE (Q5): PASS — safety + canaries clean · broad evals within noise floor
  → checkpoint promoted, with this page as its evidence
```

**The reward — the L6 move lives here:**

```
⭐ THE VERIFICATION LADDER — always reach for the highest rung available:

  TIER 1  VERIFIABLE     tests pass · unit tests · a solver · schema
                         validation · exact match
                         → DETERMINISTIC, UNGAMEABLE, cheap
  TIER 2  PROGRAMMATIC   heuristics, constraints, format checks
  TIER 3  LEARNED RM / JUDGE
                         → probabilistic, and BECOMES AN ADVERSARIAL
                           TARGET the moment you optimise against it

⚠ REACHING FOR TIER 3 WHEN TIER 1 EXISTS IS A STRICT DOWNGRADE.

AND: DECOMPOSE. Never one scalar.
   reward = f(correctness, format, safety, efficiency, ...)
   · GATES ARE MULTIPLICATIVE (safety violation ⇒ 0, not a penalty)
   · PENALTIES ARE ADDITIVE
   · ⚠ LOG EVERY COMPONENT SEPARATELY — a single scalar is HOW
     HACKING HIDES: total reward rises while correctness falls and
     format-compliance spikes.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **The gate verdict**, with the held-out **verifiable** slice as its core | Is the policy actually better? |
| Run health | Reward **by component** · KL · entropy · **zero-variance group fraction** | Is the run learning the right thing? |
| Throughput | **Rollouts per hour** · environment utilisation | Where the money goes |
| **Guardrails** | Safety suites + canaries (hard) · broad evals within noise · $ per run | Hard limits |

---

## 3 · CAPACITY + COST *(min 9–12)* — **⭐ the counterintuitive bottleneck** `A1b` `I1a` `I1d`

```
⚠ RL IS BOTTLENECKED ON GENERATING EXPERIENCE, NOT ON GRADIENT STEPS.

  a training step needs N rollouts.
  each rollout = a full generation (Q7: DECODE, bandwidth-bound, slow)
  → generation dominates wall-clock by a wide margin

⚠ AND FOR AN AGENT IT'S WORSE: the generator sits IDLE waiting on tools
  — a code execution, an API call, a search. So the ENVIRONMENT FLEET,
  not the GPU fleet, is usually the limiter.

  → so the metric to watch is ROLLOUT THROUGHPUT, and the fix is usually
    MORE ENVIRONMENT CAPACITY + ASYNC/OFF-POLICY-TOLERANT collection,
    not more GPUs. A candidate who optimises the trainer first has
    misread the profile.

SIZING, roughly: 512 prompts × a group of 8 = ~4,000 rollouts per step;
  an agentic rollout with ~20 tool calls takes minutes → thousands of
  sandboxes running in parallel to keep the actors busy

COST: a full run ≈ $60k → the GATE is far cheaper than the RUN,
      which is the whole argument for building the gate first.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| **RL with a verifiable reward (RLVR)** ✅ where checkers exist | Ungameable signal | Only for checkable domains |
| RLHF with a learned reward model | Covers open-ended quality | The RM becomes an adversarial target |
| **DPO** (offline preference optimisation) | No RM, no rollouts, cheap | Can't explore beyond the preference data |
| **Rejection-sampling fine-tuning** (best-of-N + SFT) | Simple, stable | Weaker than on-policy RL; the safe fallback (§9) |
| PPO vs **GRPO** | GRPO drops the critic | Needs a group per prompt (§5.3) |

### The system — *draw this in 90 seconds* `A2a`

```
┌─ 1. PROMPT / TASK POOL ────────────────────────────────────────
│ ⚠ curated for GRADIENT: tasks the policy sometimes passes and
│   sometimes fails. All-pass or all-fail ⇒ ZERO VARIANCE ⇒ NO SIGNAL
│ OUTPUT: a batch of tasks
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. ACTOR FLEET (generation) — inference-optimised ────────────
│ vLLM-style continuous batching (Q7); a group of samples per task
│ OUTPUT: rollouts (single responses or multi-turn trajectories)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. ⚠ ENVIRONMENT FLEET — THE USUAL BOTTLENECK ────────────────
│ sandboxes · tools · code execution · APIs · no network by default
│ ⚠ THE GRADER LIVES OUTSIDE THE ENVIRONMENT (or the policy edits the tests)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. REWARD SERVICE — verifiable checks + RM / judge ───────────
│ DECOMPOSED; components logged separately
│ ⚠ null-score-and-drop on failure — NEVER guess a reward
│ OUTPUT: per-component rewards per rollout
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. LEARNER — PPO / GRPO + a frozen REFERENCE model ───────────
│ ⚠ KL IS THE LEASH: it stops the policy walking off the distribution
│ OUTPUT: new weights (synced back to the actors)
└──────┬─────────────────────────────────────────────────────────
       ▼
 NEW WEIGHTS → ⚠ EVAL GATE (Q5) → promote or discard
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F2a`

- **The idle fleet:** *"If the actors wait on environments, I buy environment capacity, not GPUs. If the learner waits on rollouts, I go **asynchronous** and accept slightly off-policy data within a monitored bound (FU 8)."*
- **Zero-variance groups:** *"A green run can learn nothing if every group scores the same. I track the **fraction of zero-variance groups** and re-curate the pool around the tasks the policy sometimes solves."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Verifiable reward wherever possible** | "A checker is deterministic and ungameable; a learned RM becomes an adversarial target the moment you optimise against it" | Limits you to checkable domains |
| **2** | **GRPO over PPO** (often) | "Drops the value network — less memory, fewer moving parts — and group-relative advantage is well-suited to LLM rollouts" | Needs a group of samples per prompt; higher rollout cost per update |
| **3** | **Curate the task pool for gradient** | "All-pass or all-fail tasks have zero advantage variance and contribute nothing but cost" | Curation is ongoing work and shifts as the policy improves |

---

## 5 · DATA & ALGORITHM *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Task pool with checkers** | Prompts + tests / solvers / schemas | Harness bugs make tasks unsolvable; contamination |
| **Preference pairs** | Training data for the reward model | Rater bias (verbosity, agreeableness) |
| **SFT data** | The starting checkpoint | Sets the policy's prior |
| **Held-out verifiable slice** | The ungameable evaluation | Must never enter training |
| **Canaries** | Known exploits that must score badly | Grows with every exploit found |

### 5.2 Curating the task pool (e) `F1c`

```
1. MEASURE    run several rollouts per task with the current policy
2. BUCKET     always-pass · MIXED · always-fail
3. TRAIN ON   the MIXED bucket — the only one with gradient
4. INSPECT    always-fail: genuinely too hard, or a HARNESS BUG? (they look
              identical from the pass rate)
5. RE-BUCKET  periodically — yesterday's hard tasks become today's easy ones
6. DIVERSIFY  keep skills balanced so the pool doesn't collapse onto one
7. DECONTAMINATE against every eval suite the gate uses
```

### 5.3 The algorithm and the objective (f) `A2b` `F1a`

**The objective:** maximise expected reward while staying close to the reference policy:
`maximise  E[r(x, y)] − β · KL(π ‖ π_ref)`

| | **PPO** | **GRPO** |
|---|---|---|
| Value network | Yes — extra model, extra memory | **No** — advantage from group-relative scores |
| Advantage | GAE from the critic | `(r − mean(group)) / std(group)` |
| Memory | policy + ref + critic (+ RM) | policy + ref (+ RM) |
| Failure | Critic can be badly wrong early | **Zero variance within a group ⇒ no gradient** |

**KL to the reference model is the leash.** It's what keeps the policy near the distribution where the reward model is valid — without it the policy drifts off-distribution and the RM is scoring things it never saw (**RM staleness**). Tune the coefficient; **alert on KL** rather than only on reward.

**The reward model's objective** (for the RLHF part): a **pairwise Bradley-Terry loss** on preference pairs, trained as an ensemble so its disagreement can be used as an uncertainty penalty.

**The mass probe:** a run that "looks green" can have moved almost no weight. Before interpreting *any* eval delta, check that the weights actually moved — reward curves can rise from noise or from a degenerate strategy.

### 5.4 Full fine-tuning or LoRA? (f) `F2c`

RL starts from the SFT checkpoint. **LoRA** makes runs cheap to iterate and lets many experiments share one base, but it **limits how far the policy can move**, and a LoRA run can look green while barely changing behaviour, which the mass probe catches. So: LoRA for cheap exploration of reward and pool designs; **full-parameter** for the run you intend to ship, unless a LoRA run has been shown to move the policy enough on the verifiable slice.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · SFT        the starting checkpoint
STAGE 2 · REWARD     verifiers for checkable tasks; an RM ensemble for the rest
STAGE 3 · POOL       curate for gradient; decontaminate
STAGE 4 · RUN        RL with checkpoints, per-component monitors, KL tripwire
STAGE 5 · GATE       Q5's gate + the verifiable slice + canaries → promote or discard
```

---

## 6 · RUNNING THE LOOP *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

- **Asynchronous collection** `F3b`: actors generate continuously into a buffer; the learner consumes batches as they fill. Data becomes **slightly off-policy**, which PPO-style importance ratios tolerate within a bound I'd monitor rather than assume.
- **Weight sync:** colocate actors and learner, or sync on a cadence and **measure the staleness** explicitly.
- **Efficient actors** `F3a`: inference-optimised serving (continuous batching, possibly FP8). ⚠ If the actors run a different precision or kernel from the learner, rollout and training probabilities differ; correct for it with the importance ratio rather than ignoring it.
- **The objective trade** `A3b`: `E[r] − β·KL`, with β tuned (or adapted to a target KL), **multiplicative gates** for safety, **additive penalties** for style, so capability can never buy off a safety violation.
- **Reliability** `I1b`: checkpoint and resume across preemptions; **null-score-and-drop** when the reward service fails; environment health checks; deterministic seeds for reproducibility.

---

## 7 · HACKING, GATE, ROLLOUT *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
⭐ "GOODHART ISN'T A WARNING, IT'S A SCHEDULE."
   Every learned proxy becomes hackable under enough pressure; the
   only question is at which step.

THE MITIGATION LADDER:
 1 · VERIFIABLE-FIRST — nothing to hack in a passing test suite
 2 · GRADER OUTSIDE THE ENVIRONMENT — or the policy edits the tests
 3 · MULTIPLICATIVE GATES — a safety violation zeroes the reward; it
     cannot be BOUGHT with capability
 4 · DECOMPOSED COMPONENTS + PER-COMPONENT MONITORS ← how you SEE it
 5 · RM ENSEMBLE + UNCERTAINTY PENALTY — penalise reward where the
     ensemble disagrees; disagreement is exactly where over-optimisation
     lives (the RMs agree where data was dense, diverge where it wasn't)
 6 · RM RETRAINED ON ON-POLICY SAMPLES — it goes stale precisely as the
     policy learns to exploit it
 7 · HELD-OUT VERIFIABLE SLICE — ⚠ RM score climbing while VERIFIABLE
     score is FLAT is PROOF of hacking, not suspicion
 8 · CANARY SUITE in the gate; every discovered exploit becomes permanent
```

| | |
|---|---|
| **The argument** | A run costs ~**$60k**; the gate costs a fraction. **Build the gate before the trainer** |
| **Hard gates** | Safety suites · reward-hack canaries · **capability cannot outvote them** |
| **The proof of hacking** | Held-out **verifiable** slice flat while RM score rises |
| **Health during the run** | Reward **by component** · KL to reference · **entropy** (collapse ⇒ mode collapse) · rollout throughput · **fraction of groups with zero variance** |
| **Before interpreting evals** | The **mass probe** — did the weights actually move? |

**Rolling out the promoted model** `I2a`: gate → canary deployment → A/B against the current model on real traffic → a **long-horizon holdback** for effects benchmarks miss (sycophancy, verbosity; FU 10).

**Success** = the gate passes with the verifiable slice up beyond its noise floor, broad evals not down, canaries clean, and users preferring it in the A/B. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Sycophancy / verbosity** | Raters prefer agreeable, long answers | Length- and sycophancy-controlled evals; multi-turn preference data |
| **RM bias** | The RM's blind spots become the policy's habits | Ensemble + uncertainty penalty; on-policy RM refresh |
| **Task-pool skew** | Gradient curation narrows skills | Diversity constraints on the pool |
| **Refusal drift** | Safety pressure lowers helpfulness | Refusal rate tracked in both directions |
| **Rater pool** | Preferences reflect who the raters are | Diverse raters; per-language preference data |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | **Rollout throughput**; environment utilisation; $ per run (~$60k) | Rollouts/hour · $/run |
| **Reliability** `I1b` | Checkpoint/resume across preemptions; null-score-and-drop; reproducible seeds; the **mass probe** before reading any eval | Lost-run rate · reproducibility |
| **Usability** `I1c` | Runs launched from a **declarative spec**; dashboards with per-component reward, KL, entropy, zero-variance fraction; the evidence package | Time to diagnose a bad run |
| **Scalability** `I1d` | Async actor/learner; an autoscaled environment fleet; group sampling | Actor and env utilisation |
| **Privacy** `I1e` | User-derived prompts and preference data need **consent and PII scrubbing**; rollout logs retention-limited | PII scans · consent coverage |
| **Safety** `I1f` | **Multiplicative safety gates** in the reward; safety suites as hard gates; refusal rate watched in both directions | Safety-suite pass rate |
| **Security** `I1h` | ⚠ **Sandbox escape**: policy-written code runs in the environments → isolation, no network, fresh containers; the **grader outside** the environment; poisoned preference data | Sandbox violations · canary hits |
| **Ethics / bias** `I1i` | Sycophancy and verbosity measured, not rewarded; rater diversity; reward coverage across languages | Sycophancy evals · per-language gains |
| **Explainability** `I1j` | **Per-component reward logs**; an evidence package per promotion; behavioural diffs on sample prompts | % promotions with full evidence |

> ⭐ *"The product of an RL run is a claim — 'this model is better' — so production-readiness is the evidence behind that claim, and the sandbox that keeps the policy from rewriting it."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multi-turn agentic credit assignment:** a trajectory-level reward spreads one number over every turn, so most turns get no useful signal. **Turn-level advantages** with denser per-turn rewards are one approach; it's genuinely unsolved, and costly to validate.
- **Process rewards:** score reasoning steps, not just final answers. Better credit assignment, but a step-level reward model is another target to game.

**When RL diverges or gets hacked, the fallback pipeline** `A5b` `F2a`:

```
KL tripwire fires, the verifiable slice goes flat while reward climbs,
or canaries start passing
  → STOP and roll back to the last good checkpoint
  → REJECTION-SAMPLING FINE-TUNING: sample N answers per task from the good
    checkpoint, keep only the ones the VERIFIER passes, SFT on those
    (no learned reward to hack, very stable)
  → fix the reward (new canary, retrained RM) before resuming RL
```

**Top 3 failure modes:**
1. **Reward hacking** — total reward rises, real capability falls, and a single scalar hides it.
2. **Zero-variance groups** — the task pool is all-pass or all-fail, the run is green, nothing is learned.
3. **RM staleness** — the policy moves off the RM's training distribution and the reward becomes noise, silently.

**Q1 ships:** the **reward pipeline (verifiable-first, decomposed, logged per component)** and the **eval gate**.
> *"The gate before the trainer, because a $60k run against an untrusted reward is an expensive way to make the model worse — and you won't know for weeks."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "What's the bottleneck in an RL training run?"** `A1b` `F2b`
> **Generating experience, not gradient steps** — and that's counterintuitive enough that people optimise the wrong thing. Each training step needs N rollouts, and each rollout is a full generation, which is decode: memory-bandwidth-bound and slow. For an *agentic* run it's worse, because the generator sits idle waiting on tools — a code execution, an API call, a search — so the **environment fleet, not the GPU fleet, is usually the limiter**. The metric to watch is **rollout throughput**, and the fixes are more environment capacity, async collection that tolerates slightly off-policy data, and inference-optimised actors with continuous batching. A candidate who reaches for distributed-training optimisations first has misread the profile — the trainer is usually not where the wall-clock goes.

**2. "How do you stop reward hacking?"** `I1h` `F1c`
> You don't stop it, you make it **detectable and expensive** — my framing is that Goodhart isn't a warning here, it's a schedule: every learned proxy becomes hackable under enough optimisation pressure, and the question is at which step. The ladder: **verifiable rewards first**, because there's nothing to hack in a passing test suite; **the grader lives outside the environment**, or the policy will simply edit the tests; **multiplicative gates**, so a safety violation zeroes the reward rather than being a penalty that capability can buy off; **decomposed components with per-component monitors**, because a single scalar is *how* hacking hides — total reward rises while correctness falls; **RM ensembles with an uncertainty penalty**, since disagreement marks exactly the off-distribution regions where hacking gets found; and **a held-out verifiable slice**, where RM score climbing while verifiable score stays flat is **proof rather than suspicion**. Every discovered exploit becomes a permanent canary in the gate.

**3. "PPO or GRPO?"** `A5a` `F1a`
> **GRPO** as the default for LLM post-training, with reasons rather than fashion. It drops the value network, which removes a whole model from memory and a whole class of failure — an early-training critic can be badly wrong and destabilise updates — and it computes advantage **group-relatively**: sample a group of completions per prompt, and the advantage is each completion's reward normalised within its group. That's well-suited to LLM rollouts where you can cheaply sample several responses to the same prompt. The cost is that you need that group, so rollout cost per update is higher — which matters given that rollouts are the bottleneck. And it has a specific failure I'd monitor explicitly: **if every sample in a group gets the same reward, the advantage is zero and there's no gradient**, so a task pool of all-pass or all-fail items produces a run that looks healthy and learns nothing. I'd track **fraction of groups with zero variance** as a first-class health metric.

**4. "What does KL to the reference model actually do?"** `A3b` `F1a`
> It's the **leash**. Without it the policy drifts arbitrarily far from the initialisation in pursuit of reward, and two things break. First, **the reward model becomes invalid** — it was trained on the old policy's output distribution, so as the policy moves it's increasingly scoring things it never saw, which is RM staleness and it makes the reward signal noise rather than guidance. Second, the policy **loses general capability** that wasn't captured in the reward: it gets better at the measured thing and worse at everything else, which is only visible in a broad eval. So KL keeps the policy in the region where the reward is meaningful. Practically I'd **alert on KL directly** rather than only on reward, because a rising reward with a rising KL is the signature of drift toward exploitation, and treating KL as a soft preference rather than a tripwire is how runs go wrong in a way that's obvious in hindsight.

**5. "Your reward curve is going up. Is the model better?"** `F1b` `I2a`
> Not established, and I'd resist that inference specifically. Three things to check before believing it. **Did the weights actually move?** A mass probe — comparing adapter or weight norms against the baseline — because reward curves can rise from noise, and I've seen runs interpreted in detail that had moved almost nothing. **Which component rose?** If total reward is up while the **verifiable** component is flat and format-compliance spiked, that's reward hacking with a clear signature, and it's exactly what a single scalar would have hidden. **What did the held-out eval say?** The reward is a proxy; the gate is the measurement, and they're allowed to disagree — when they do, the gate wins. I'd also check **entropy**, because collapsing entropy with rising reward often means the policy found one high-reward mode and stopped exploring, which scores well and is usually a worse product.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you build a task pool that actually teaches the model something?"** `A2c` `F1c`
> By curating for **gradient**, which is a different criterion from curating for quality and it's the one people miss. A task the policy always passes contributes nothing — zero advantage variance — and so does one it always fails; both burn rollout budget, which is the scarcest resource in the run. So the useful tasks are the ones where the policy **sometimes succeeds**, and that set *moves as the policy improves*, which makes pool curation an ongoing process rather than a setup step. Practically: measure per-task pass rates with several rollouts each, bucket into always-pass / mixed / always-fail, train on the mixed bucket, and periodically re-bucket because yesterday's hard tasks become today's easy ones. The always-fail bucket deserves separate attention — sometimes it's genuinely too hard, and sometimes it's a **harness bug** where the task is unsolvable for reasons unrelated to capability, and those look identical from the pass rate alone. I'd also keep an eye on the pool's **diversity**, because a pool curated purely for gradient can collapse onto one narrow skill.

**7. "The RM was trained on the old policy's outputs. How do you know when it's stale?"** `F2a` `F1c`
> Distribution shift between the RM's training data and current rollouts — and I'd measure it several ways because each has a blind spot. The **cheapest and earliest** signal is the RM's own uncertainty, via an ensemble or a calibrated head, on live rollouts: rising uncertainty means it's being asked about things it doesn't recognise, and it needs no reference distribution. Then **embedding-space distance** between the RM's training rollouts and current ones — MMD for a proper two-sample test, or per-rollout k-NN distance, which is more actionable because it flags individual rollouts as out-of-support. The most **interpretable** is a **domain classifier**: train a small model to distinguish RM-training rollouts from current ones — if AUC climbs toward 0.9 they're trivially separable, and the feature attributions tell you *what* changed (length, tool-call patterns, formatting). But the check that actually speaks to correctness is behavioural: **RM score versus the verifiable checker** on tasks where both apply. RM rising while pass rate is flat is the signature, and unlike the distributional metrics it tells you the reward is *wrong*, not merely that its inputs moved.

**8. "You have one GPU cluster and it's half idle during rollouts. Fix it."** `F3b` `A1c`
> That idle time is the actual cost centre, and the fix depends on which half is idle. If the **actors** are idle waiting on environments — the usual agentic case — the answer is more environment capacity, not more GPUs, and I'd say that plainly because it's an unintuitive budget request. If the **learner** is idle waiting for a full batch of rollouts, the fix is **asynchronous collection**: let actors generate continuously into a buffer and let the learner consume batches as they fill, which decouples the two and keeps both busy. The cost is that data becomes **slightly off-policy**, since rollouts were generated by a policy a few updates old — PPO-style importance ratios tolerate a bounded amount of that, and the bound is a tuning decision I'd monitor rather than assume. Beyond that: **colocate actors and learner** to avoid weight-transfer stalls, or accept a weight-sync cadence and measure its staleness explicitly. The general point is that RL infrastructure is a **pipeline-balancing problem**, and the profile tells you which stage to fund.

**9. "How do you decide whether to ship the resulting model?"** `I2b` `I2a`
> The gate decides, and it has to have been agreed **before** the run — otherwise a $60k sunk cost negotiates on its own behalf, which is a real organisational dynamic rather than a hypothetical. Structurally: **hard gates** that capability cannot outvote — safety suites, reward-hack canaries, contamination status, judge calibration — and **soft metrics** that trade against each other. The specific evidence I'd want for an RL run beyond the usual eval: the **held-out verifiable slice**, because that's the one the reward couldn't have been gamed against; **broad capability evals** the reward didn't target, to catch the "better at the measured thing, worse at everything else" failure that KL is supposed to prevent but doesn't always; and **the noise floor**, so a two-point gain on a suite with a three-point floor isn't read as a win. I'd also want a **rollback-ready deployment** and a plan for what live signal would trigger it, because some regressions only appear against real traffic.

**10. "Your model is better on every benchmark and users like it less. What happened?"** `I2b` `I1i`
> Something the reward captured and the users didn't value, or vice versa — and the most common version is **sycophancy**, because it's an almost-inevitable product of preference-based reward: raters prefer agreeable, confident, well-formatted responses, so the policy learns agreeableness, which reads well in a pairwise comparison and is worse in sustained use. Others in the same family: **verbosity** (longer answers win comparisons and annoy users), **hedging** (safe-sounding, less useful), and **refusal drift** (fewer risky outputs, less helpful overall). The structural diagnosis is that the reward model was trained on **short-horizon pairwise preferences**, and the user experience is a **long-horizon, repeated interaction** — those diverge exactly as the policy gets good at the proxy, which is the same available-versus-meaningful split that appears throughout this pack. What I'd do: **long-horizon holdbacks** and retention as the gating signal rather than benchmark deltas; **length- and sycophancy-controlled evaluation** so those effects are measured rather than rewarded; and preference data collected on **multi-turn sessions** rather than single responses, which is more expensive and much closer to the thing we care about.

---

## ⭐ THE L6 LINE

> **"Verifiable first, decomposed always — a single scalar is how reward hacking hides. RL is bottlenecked on generating experience, and for agents it's the environment fleet, not the GPUs. And the gate has to exist before the run, because $60k of sunk cost negotiates on its own behalf."**

## 🧭 HOOKS — *steer by what you mention*

- *"Verifiable first — a judge is tier 3"* → invites the reward-design branch
- *"The bottleneck is the environment fleet"* → invites the systems/infra branch
- *"Goodhart isn't a warning, it's a schedule"* → invites the reward-hacking branch (**good at L6**)
- *"KL is the leash"* → invites the algorithm branch
- *"The gate has to exist before the run"* → invites the org/process branch (**good at L6**)
- *"Zero-variance groups produce no gradient"* → invites the GRPO/data-curation branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the verification ladder"* — tiers 1–3 regenerate the reward design.
- **Running out of time?** → *"I have five minutes — I'd rather cover reward hacking and the gate than finish the parallelism. Sound right?"*

**Name and defer:**
- "**DPO and other preference-optimisation methods** skip the RM entirely — different trade, no online rollouts."
- "**Process vs outcome supervision** — rewarding reasoning steps rather than final answers — changes credit assignment."
- "**Multi-turn credit assignment** in agentic RL is genuinely unsolved; turn-level advantage is one approach."
- "**Constitutional / AI feedback** replaces human preference data at a cost to calibration."
- "**The eval harness** that gates this is its own design."
- "**Distributed training topology** — FSDP, TP, offload — matters at scale but isn't the bottleneck here."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a trustworthy reward first, throughput second | §0 | *"A fast pipeline optimising a hackable reward produces a worse model faster."* |
| **A1b** | Throughput and cost budgets | §1, §3 | *"~4,000 rollouts per step; the environment fleet is the limiter; ~$60k a run."* |
| **A1c** | Degrade without lowering quality | §4, FU 8 | Async collection within a monitored off-policy bound |
| **A2a** | The full stack | §4 diagram | Pool → actors → environments → reward service → learner → gate |
| **A2b** | Model components | §5.3 | Policy, frozen reference, RM ensemble, (critic for PPO) |
| **A2c** | Data sources | §5.1, FU 6 | Tasks with checkers, preference pairs, the verifiable slice, canaries |
| **A2d** | Staged pipeline | §5.5 | SFT → reward → pool → run → gate |
| **A3a** | Top-down, unprompted | §0 | *"A better policy, plus the evidence that it's better"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.3, §6 | *"E[r] − β·KL, multiplicative safety gates, additive penalties."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Reward source · single vs multi-turn · the gate · model and budget |
| **A5a** | Alternatives compared | §4, FU 3 | RLVR vs RLHF vs DPO vs rejection sampling; PPO vs GRPO |
| **A5b** | A full alternative pipeline on demand | §9 | Roll back → rejection-sampling fine-tuning → fix the reward |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right algorithm and loss | §5.3 | *"GRPO advantage is (r − mean) / std within the group; Bradley-Terry for the RM."* |
| **F1b** | Metrics and their trade-offs | §2, FU 5 | *"Verifiable first, decomposed always; a single scalar is how hacking hides."* |
| **F1c** | Bias and failure sources | §5.2, §7 | Zero-variance groups, RM staleness, sycophancy, verbosity |
| **F2a** | Training underperforms → a fallback | §4, §9, FU 7 | The mitigation ladder; the rejection-sampling fallback |
| **F2b** | Throughput too low → tiering | §4, FU 1 | Environment capacity and async, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA for exploration, full-parameter for the shipped run |
| **F3a** | Efficiency with a quality trade | §6 | FP8 actors with importance correction for the mismatch |
| **F3b** | Scale levers | §6, FU 8 | Async actor/learner; colocation; weight-sync cadence |
| **F3c** | Extension, with cost and maturity risk | §9 | Multi-turn credit assignment; process rewards |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Rollouts per hour; $ per run |
| **I1b** | Reliability | §6, §8 | Checkpoint/resume; null-score-and-drop; mass probe |
| **I1c** | Usability | §2, §8 | Declarative run specs; the evidence package |
| **I1d** | Scalability | §6, §8 | Autoscaled environment fleet |
| **I1e** | Privacy | §8 | Consent and PII scrubbing for user-derived data |
| **I1f** | Safety | §6, §8 | Multiplicative safety gates; hard safety suites |
| **I1h** | Security | §4, §8, FU 2 | **Sandbox escape**; grader outside the environment |
| **I1i** | Ethics / bias | §7, §8, FU 10 | Sycophancy and verbosity measured, not rewarded |
| **I1j** | Explainability | §2, §8 | Per-component reward logs; evidence per promotion |
| **I2a** | Offline → online | §7, FU 9 | Gate → canary → A/B → long-horizon holdback |
| **I2b** | The success definition | §7, FU 9 | *"Verifiable slice up beyond noise, broad evals held, canaries clean."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) task pool → §5.2 · (f) objective → §5.3–5.4 · (g) running it → §6 · (h) success → §2, §7 · (i) extension → §9.
