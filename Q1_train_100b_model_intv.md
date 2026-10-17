# Q1 · Train a 100B-Parameter Model — **INTERVIEW SHEET**

> **Tier 3 · #26** — only if the team is infra-leaning; high variance. But it's one of the two highest-frequency prompts in the agentic bank.
> Full study file: [`../Q1_train_100b_model.md`](../Q1_train_100b_model.md) · [diagrams](../Q1_train_100b_model_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what "done" looks like · (c) the training system · (d) datasets · (e) the data mixture · (f) the objective and architecture · (g) running the job through failures · (h) how you know it's working · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Budget and deadline? Serving volume? Who owns the data pipeline? What hardware and network? (§1) |
| **Explainability** is easy points | A **data datasheet** (what went in), a **run log** of every intervention, a model card at hand-off (§8) |
| **Security** beyond the gateway | **The weights are the crown jewels** (exfiltration) · data poisoning · the code and container supply chain (§8) |
| **Troubleshoot before being asked** | Raise **goodput vs MTBF**, **stragglers** and **the serving constraint** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **One shot** · **goodput, not FLOPs** · data quality dominates | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · goodput as the SLO · non-goals | A4a, A1b |
| **6–9** | **WHAT "DONE" LOOKS LIKE** | **(b)**, (h) | The checkpoint, the dashboard, the go/no-go evidence | I1c, F1b, I2b |
| **9–12** | **THE ARITHMETIC** | — | ⭐ **Do `6ND` out loud** · 1.6 TB of state · $ per run | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | The 3D split, and **why in that order** · the run system around it | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Decontamination as **correctness** · mixture at 1B · muP · GQA decided here | A2b–d, F1a, F1c, F2c |
| **32–38** | **RUNNING THE JOB** | **(g)** | ⚠ **MTBF ~half a day** · async checkpoints · stragglers · spike policy | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + KILL + HAND-OFF** | **(h)** | Scaling-law band · checkpoint evals · **a kill criterion agreed in advance** | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | MoE, long context · **the pivot pipeline** if the run can't make the bar | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The deliverable here isn't a cluster — it's one model checkpoint that meets a quality bar, produced once, inside a fixed budget and calendar. Three consequences follow.**
>
> **First, this is a single-shot job. I get roughly one full-scale run, so the system has to be de-risked before the run by small-scale proxies — scaling-law ablations, muP hyperparameter transfer, a data bake-off at 1B. Anything I can only learn at 100B is a design failure.**
>
> **Second, the bottleneck is goodput, not FLOPs. At 4,096 GPUs the cluster mean-time-between-failures is on the order of half a day — checkpointing, restart speed and straggler tolerance will move the delivery date more than any kernel optimisation.**
>
> **Third, data quality dominates. Compute is roughly fixed by budget; the lever I actually control is what tokens go in — and I'd treat decontamination as a correctness requirement, not hygiene."**

**Then say the plan (`A3a`):** *"Requirements, what 'done' looks like, the arithmetic, the parallelism and the run system, data and model choices, running the job through failures, evaluation and how I'd know to kill it, a production-readiness sweep, then extensions and failure modes — including the serving story, because the inference cost of this model should be constraining my training decisions today."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What's the budget and the deadline?** *(This is a fixed-compute problem — the budget *is* the constraint.)*
2. **What will it be used for, and at what serving volume?** *(⚠ **Inference cost should constrain training decisions now.**)*
3. **Do we have the data pipeline, or am I building that too?** *(If the latter, that's Q8 and it's most of the work.)*
4. **What hardware, and is the cluster dedicated?** *(Network topology sets the parallelism; a shared cluster is a capacity risk — FU 9.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Model | A 100B **dense** decoder-only transformer |
| Tokens | ~2T (about 20 per parameter) |
| Cluster | 4,096 H100s, 8 per node on NVLink, InfiniBand between nodes, dedicated |
| Budget | Low single-digit $M of compute; a ~2-week window for the main run |
| After | Post-training (Q4) and serving at high volume (Q7) |

**FR:** train to a quality bar · reproducible · checkpointed and resumable · evaluated continuously · **decontaminated** against the eval suite.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Goodput** | **> 90%** of wall-clock doing useful work ← *the metric* |
| **MFU** | ~40–50% — respectable; 100% is not the goal |
| Checkpoint | Frequent enough that **lost work < MTBF-driven cost**; async, non-blocking |
| Restart | **Minutes, not hours** — this is what goodput actually buys |
| Reproducibility | Same (data order, seed, config, code) → same loss curve |

**Scope / NON-GOALS:**
- ❌ **Chasing peak MFU.** Position: goodput is the metric; MFU is an input to it.
- ❌ **Learning anything new at full scale.** De-risk small.
- ❌ Post-training/RLHF (Q4) · serving (Q7) · the data pipeline in depth (Q8, but I'd flag it dominates).

---

## 2 · WHAT "DONE" LOOKS LIKE *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the research team and leadership see (b):**

```
RUN 100B-v1 · day 6 of ~9 · $1.9M of $3.2M
  loss          ▁▂▃ tracking INSIDE the scaling-law band       ← the key chart
  goodput       92%   (MFU 46%)   interruptions: 11, median restart 6 min
  checkpoints   every ~25 min, async · last good: step 61,200
  evals @ ckpt  reasoning ↑ · code ↑ · multilingual ≈ prediction
  contamination check: PASS (blocking)
  kill criterion: loss > band upper edge for 2 consecutive evals → NOT MET
  interventions: 2 spikes — skip-batch (step 40,112), rollback 200 (step 52,900)
```

**The deliverable:** a checkpoint that meets the bar, **plus the evidence** — the loss against prediction, checkpoint evals, the data datasheet and the run log. *"The checkpoint without the evidence isn't shippable, because no one downstream can trust it."*

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **A checkpoint meeting the quality bar, inside budget and calendar** | The deliverable |
| Health | **Loss vs the scaling-law prediction** · gradient norms · spikes and recovery | Is it on trajectory? |
| Efficiency | **Goodput** (> 90%) · MFU · $ burned vs plan | What moves the date |
| **Guardrails** | **Contamination = 0** (blocking) · the pre-agreed kill criterion · downstream evals at checkpoints | Hard limits |

---

## 3 · THE ARITHMETIC *(min 9–12)* — **⭐ do this on the board** `A1b` `I1a` `I1d`

```
TRAINING FLOPs ≈ 6 · N · D
   N = 100e9 params · D = 2e12 tokens (Chinchilla-ish ~20 tok/param)
   → 6 × 1e11 × 2e12 = 1.2e24 FLOPs

CLUSTER: 4,096 H100s · ~1e15 BF16 FLOP/s each (realistic dense)
   at MFU 0.45 → 4096 × 1e15 × 0.45 = 1.8e18 FLOP/s
   → 1.2e24 / 1.8e18 ≈ 6.7e5 s ≈ 7.7 DAYS of PERFECT uptime
   → ⚠ at 90% goodput ≈ 8.5 days.  At 60% goodput ≈ 13 days.
      GOODPUT MOVES THE DATE MORE THAN MFU DOES.

MEMORY — why it can't fit on one GPU:
   params  100e9 × 2 B (bf16)          = 200 GB
   grads   100e9 × 2 B                 = 200 GB
   ⚠ ADAM STATES 100e9 × 8 B (fp32 m,v)= 800 GB
   + fp32 master weights 400 GB
   ≈ 1.6 TB of STATE before a single activation
   → an H100 has 80 GB. → SHARDING IS NOT OPTIONAL.

COST: 4,096 GPUs × ~9 days × 24 h ≈ 880k GPU-hours
   → ⚠ a rough order of $2–4M for one run. THAT is why it's single-shot.

STORAGE: a full checkpoint ≈ 1.6 TB → written every ~25 min, it needs
   sustained parallel-filesystem bandwidth, which is why it's ASYNC (§6)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Train at all? | From scratch vs **continued pretraining / distillation** of an existing base | From scratch only with a written reason (FU 10) |
| Model | Dense vs MoE | **Dense** here: simpler to de-risk; MoE changes memory and comms completely |
| Attention | MHA vs **GQA** | Decided here, sets the KV cache **forever** (FU 6) |
| Parallelism | Pure FSDP vs **TP 8 × PP 8 × DP 64 + ZeRO** | Ordered by communication cost (below) |
| Checkpointing | Synchronous vs **asynchronous** | A blocking 1.6 TB write idles the cluster ~20 times a run |
| What to optimise | MFU vs **goodput** | MTBF ~12 h makes restarts the bigger lever (FU 8) |

### The 3D split, and the order `A2a`

```
 ┌──────────────────────────────────────────────────────────┐
 │ TENSOR PARALLEL (TP)  — inside a node, over NVLink       │
 │  splits each weight matrix; 2 all-reduces per layer      │
 │  ⚠ NEVER cross the node boundary — collectives dominate  │
 │  → TP = 8 (one node)                                     │
 ├──────────────────────────────────────────────────────────┤
 │ PIPELINE PARALLEL (PP) — across nodes                    │
 │  splits LAYERS into stages; cheap point-to-point comms   │
 │  ⚠ THE BUBBLE: (PP−1)/(microbatches + PP−1) idle         │
 │  → interleaved/1F1B schedules + many microbatches        │
 ├──────────────────────────────────────────────────────────┤
 │ DATA PARALLEL (DP) — across the rest                     │
 │  + ZeRO/FSDP sharding of optimiser state (the 800 GB)    │
 └──────────────────────────────────────────────────────────┘
   4096 = TP 8 × PP 8 × DP 64

⚠ THE ORDER IS THE ANSWER, AND IT FOLLOWS FROM COMMUNICATION COST:
   TP is the most communication-intensive  → keep it INSIDE NVLink
   PP is the cheapest per boundary          → use it ACROSS nodes
   DP is last                               → scales with everything left

PLUS: activation checkpointing (recompute vs store — trades ~30%
  extra compute for a large memory win), and sequence/context
  parallelism if the context is long.
```

### The run system around it `A2a`

```
┌─ 1. DATA PIPELINE (Q8) ────────────────────────────────────────
│ TASK: filtered, deduped, decontaminated, mixed, tokenised shards
│ OUTPUT: a deterministic token stream (seeded order)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. TRAINER — 4,096 GPUs, TP × PP × DP ────────────────────────
│ OUTPUT: steps, loss, gradient norms, per-GPU step times
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. ASYNC CHECKPOINTER ────────────────────────────────────────
│ GPU → host memory in seconds; host → storage in the background
│ OUTPUT: a restorable checkpoint every ~25 min
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. HEALTH MONITOR + ORCHESTRATOR ─────────────────────────────
│ TASK: detect dead nodes, STRAGGLERS, silent data corruption
│ OUTPUT: evict → swap in a hot spare → restart from checkpoint
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. EVAL HARNESS at checkpoints ───────────────────────────────
│ OUTPUT: downstream evals + loss vs the scaling-law band
│         → the dashboard (§2) and the kill criterion (§7)
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `I1b`

- **Goodput vs MTBF:** *"Twenty interruptions in nine days. Restart speed moves the date more than any kernel."*
- **Stragglers:** *"Every collective is a barrier, so one throttled GPU paces 4,095 others — and it's invisible in average utilisation."*
- **The serving constraint:** *"GQA, depth vs width and vocabulary are decided here and paid for at inference forever."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **TP inside the node, PP across** | "TP does two all-reduces per layer — crossing the node boundary makes collectives dominate. PP's boundary comms are cheap point-to-point" | PP introduces the bubble; needs many microbatches to amortise |
| **2** | **Optimise goodput, not MFU** | "MTBF is ~half a day at this scale. A 5% MFU win is worth less than cutting restart time from an hour to five minutes" | Checkpointing costs bandwidth and some throughput |
| **3** | **De-risk everything at small scale** | "muP lets me transfer hyperparameters from a 1B proxy. Anything I can only learn at 100B is a design failure" | Small-scale conclusions don't always transfer — validate at an intermediate scale |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

> *"Compute is fixed by budget. The lever I control is what tokens go in — so I'd spend real time here, and this is [Q8](Q8_data_pipeline_intv.md)'s question in miniature."*

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Filtered web crawl** | Breadth; most of the tokens | Quality, spam, PII, opt-outs |
| **Code** | Reasoning and tool use | Licences; secrets in repos |
| **Books, papers** (licensed) | Long-form, high quality | Provenance and licensing |
| **Multilingual** | Coverage beyond English | Over-filtering strips it |
| **Math and synthetic data** | Targeted skills | Synthetic data narrows diversity |
| **The eval suite** | The go/no-go numbers | ⚠ Must never leak into training |

### 5.2 The data mixture (e) `F1c`

- **Quality filtering, dedup (exact + fuzzy), mixture** — and the mixture is **measured at 1B, not argued** (Q8).
- **⚠ Decontamination is a correctness requirement, not hygiene.** *"Contamination doesn't degrade the model, it **invalidates the measurement** — and at this cost I'm making a go/no-go decision on those numbers. Canary strings and n-gram overlap, run as a blocking check."*
- **⚠ Over-filtering is as damaging as under-filtering** — aggressive filters strip dialects, technical prose and low-resource languages, producing a cleaner corpus and a worse model.
- **Data order is part of reproducibility** — the same shuffle seed, or the loss curve isn't comparable.
- **Anneal at the end:** up-weight the highest-quality data during the final learning-rate decay, decided by the same 1B bake-off.

### 5.3 The objective and the architecture (f) `A2b` `F1a`

| Choice | Decision | Why |
|---|---|---|
| **Objective** | Next-token **cross-entropy** | The pretraining loss; everything else is post-training (Q4) |
| **Hyperparameters** | **muP**, tuned on a 1B proxy | LR transfers across width; tune where it's cheap |
| **Schedule** | Warmup, then cosine or warmup-stable-decay | WSD lets the decay phase double as the anneal |
| **Stability** | QK-norm, a small z-loss, gradient clipping | Fewer spikes in a run you can't repeat |
| **Attention** | **GQA** (8 KV heads) | ~10× smaller KV cache at serving (FU 6) |
| **Shape** | Depth vs width chosen with serving latency in mind | Equal loss can serve very differently |

### 5.4 Fine-tune an existing model (f) `F2c`

The serious alternative to this whole run: **continued pretraining** of an existing open-weights base, or **distillation** from a larger model, at a fraction of the cost (FU 10). From scratch is justified by a written reason — data provenance, a different mixture or language, or an architecture choice that can't be retrofitted. Inside the plan, the small proxies are the "existing models" that carry the learning: every decision is made on them first.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · PROXIES      scaling ladder (e.g. 150M → 1B → 8B) · muP transfer
STAGE 2 · DATA         mixture bake-off at 1B · decontamination check (blocking)
STAGE 3 · SMOKE        a few hours at FULL scale: NCCL, checkpoints, storage, stragglers
STAGE 4 · MAIN RUN     with checkpoint evals and the kill criterion live
STAGE 5 · ANNEAL       high-quality data during the final LR decay
STAGE 6 · HAND-OFF     checkpoint + datasheet + run log + evals → post-training (Q4)
```

---

## 6 · RUNNING THE JOB *(min 32–38)* — probe **(g)** · **⭐ where the date actually moves** `F3a` `F3b` `A3b` `I1b`

```
⚠ AT 4,096 GPUs, CLUSTER MTBF ≈ HALF A DAY.
  Over a 9-day run that's ~20 interruptions. Each one costs:
     (time since last checkpoint) + (restart time)

  → GOODPUT = useful compute / wall-clock, and it's THE metric.

WHAT ACTUALLY MOVES IT:
 · ASYNCHRONOUS CHECKPOINTING — copy to host memory fast, flush to
   storage in the background. A blocking checkpoint at 1.6 TB of
   state is minutes of the whole cluster idling.
 · FAST RESTART — keep the job warm, hot spares pre-allocated, and
   ⚠ don't re-initialise the whole NCCL world if you can avoid it
 · ⚠ STRAGGLER DETECTION — one slow GPU paces 4,095 others, because
   every collective is a barrier. A thermally-throttled card is a
   CLUSTER-WIDE slowdown that looks like nothing in aggregate.
 · SILENT DATA CORRUPTION — a card computing WRONG results without
   erroring. Detect with periodic deterministic checks; it's rare
   per-card and not rare across 4,096 of them.
 · LOSS-SPIKE POLICY DECIDED IN ADVANCE: skip the batch, roll back N
   steps, lower LR? ⚠ Decide it BEFORE the run, at 3am you won't.

⭐ HOW OFTEN TO CHECKPOINT — a formula, not a guess (Young/Daly):
   interval ≈ √(2 × blocking checkpoint cost × MTBF)
   e.g. √(2 × 30 s × 43,200 s) ≈ 1,600 s ≈ 27 min
   → making the blocking part smaller (async) is what lets you
     checkpoint often without paying for it
```

- **The objective** `A3b`: minimise **time to the quality bar** = compute ÷ (MFU × goodput) — so spend effort on whichever factor is further from its ceiling, which at this scale is almost always goodput.
- **Efficiency** `F3a`: activation checkpointing (~30% recompute for memory), communication overlapped with compute, FP8 matmuls only if the 1B and 8B proxies show stable loss — the trade is a numerics risk in a run you can't repeat.
- **Scale** `F3b`: the 3D layout scales with nodes; the data loader and the checkpoint filesystem must scale with it, or they become the straggler.
- **Reliability** `I1b`: async checkpoints, hot spares, straggler eviction, SDC checks, and a **pre-agreed spike policy** — every intervention written to the run log.

---

## 7 · EVAL + KILL + HAND-OFF *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
⭐ SINGLE-SHOT ⇒ EVERYTHING LEARNABLE SMALL MUST BE LEARNED SMALL:
 · muP (maximal update parametrisation) → HYPERPARAMETER TRANSFER
   from a 1B proxy; tune LR there, use it at 100B
 · SCALING-LAW ABLATIONS → predict the loss you should be getting,
   so you can tell at day 2 whether the run is on trajectory
 · DATA BAKE-OFF at 1B → the mixture decision
 · A FULL-SCALE SMOKE RUN of a few hours → shakes out the infra
   without burning the budget

DURING THE RUN — how you'd know to KILL it:
 · loss vs the SCALING-LAW PREDICTION (the most useful single chart)
 · gradient norms · ⚠ loss spikes and their recovery
 · downstream eval at checkpoints, not just loss
 · ⚠ AND A PRE-AGREED KILL CRITERION, because at $3M nobody wants to
   be the one to stop it — so the criterion must exist before the
   sunk cost starts negotiating on its own behalf.

HAND-OFF ("rollout" for this system): the final checkpoint goes to
  post-training (Q4) WITH its evidence → post-trained evals → the
  serving team (Q7), whose cost model was an input from day one.
```

**Success** = the bar met on decontaminated evals, inside the budget and the calendar, with goodput above 90% and every intervention explained in the run log. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Mixture bias** | Web data over-represents English and some viewpoints | Measured multilingual mixture; per-language evals |
| **Filter bias** | Quality and toxicity filters remove dialects disproportionately | Audit what filters remove, by dialect and language |
| **Contamination** | Benchmarks in the training data make evals optimistic | Blocking decontamination; canaries |
| **Eval bias** | English-centric benchmarks define "good" | Multilingual and task-specific eval suites |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | MFU ~45%; tokens per second; $ per run against plan | MFU · tokens/s · $ burned |
| **Reliability** `I1b` | **Goodput > 90%**: async checkpoints, hot spares, straggler eviction, SDC checks | Goodput · median restart time |
| **Usability** `I1c` | For researchers: a declarative run config, the dashboard, reproducible runs; for downstream: a documented checkpoint | Time to diagnose an incident |
| **Scalability** `I1d` | The 3D layout; data loader and checkpoint storage sized to the cluster | Loader stalls · checkpoint write time |
| **Privacy** `I1e` | PII scrubbed before training; crawler opt-outs honoured; **memorisation** of personal data tested at checkpoints | PII memorisation rate |
| **Safety** `I1f` | Harmful content filtered from the data; **dangerous-capability evals** at checkpoints before any release | Capability-eval results |
| **Security** `I1h` | ⚠ **Weights are the crown jewels**: encrypted checkpoints, least-privilege access, exfiltration monitoring; **data poisoning** defences in the pipeline; pinned, signed code and containers | Access audits · anomalous egress |
| **Ethics / bias** `I1i` | Filter audits for dialects; language coverage; licensing and provenance; **energy and carbon reported** | Per-language evals · tCO₂e |
| **Explainability** `I1j` | A **datasheet** of what went in; a **run log** of every intervention and why; a model card at hand-off | % interventions logged with a reason |

> ⭐ *"For a single-shot run, readiness is being able to explain every hour of the calendar and every token in the data."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **MoE:** sparse compute but all experts resident; expert parallelism and load balancing change the memory and communication story completely.
- **Long context:** sequence/context parallelism and a different data construction.
- **Multimodal:** an encoder, an alignment stage, and a separate data problem.

**When the run can't make the bar — the pivot pipeline** `A5b` `F2a`:

```
the kill criterion fires, or a data bug is found mid-run
  → STOP spending: checkpoint and freeze the run
  → diagnose on the proxies, not on the big run (data? optimisation? infra?)
  → choose the cheapest path to the bar:
      · fix the data and restart SMALLER with more tokens (cheaper to serve)
      · or continued pretraining of an existing base with our mixture
      · or distil from a larger model
  → the partial checkpoint still feeds research and the proxies' calibration
  "Restarting costs less than finishing a run that's spending compute wrong."
```

**Top 3 failure modes:**
1. **Goodput collapse** — a flapping node, slow storage, or a straggler turns 8 days into 15 and nobody notices because MFU looks fine.
2. **Contamination discovered after the run** — every quality number is retroactively meaningless, at $3M.
3. **Loss divergence with no pre-agreed policy** — a 3am judgement call with a sunk cost arguing for "let it run."

**Q1 ships (the first month, before the run):** the **de-risking suite** — muP transfer, scaling-law ablations, the data bake-off, and a **full-scale smoke run**.
> *"Because the run itself is a single shot, and the entire job is to make sure nothing is learned for the first time at 100B."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Do the compute arithmetic."** `A1b` `I1a`
> Training FLOPs are roughly **6ND** — six times parameters times tokens. At 100B parameters and 2 trillion tokens that's about **1.2 × 10²⁴ FLOPs**. On 4,096 H100s at a realistic **45% MFU**, the cluster delivers around 1.8 × 10¹⁸ FLOP/s, so the run is about **7.7 days of perfect uptime** — call it 8.5 days at 90% goodput, or **13 days at 60%**. That comparison is the point: **goodput moves the delivery date more than MFU does**, and a 5% MFU improvement is worth less than cutting restart time from an hour to five minutes. On memory: parameters at bf16 are 200 GB, gradients another 200, and **Adam states 800 GB** in fp32 plus master weights — about **1.6 TB of state before a single activation**, against 80 GB per H100. So sharding isn't an optimisation, it's the only way the model exists at all.

**2. "How do you parallelise it, and why in that order?"** `A2a` `F3b`
> Three dimensions, ordered by **communication cost**. **Tensor parallelism** splits each weight matrix and does two all-reduces per transformer layer — that's the most communication-intensive, so it must stay **inside a node on NVLink**; crossing the node boundary makes collectives dominate everything. TP=8. **Pipeline parallelism** splits layers into stages with cheap point-to-point communication at each boundary, so it's what I use **across nodes** — PP=8, with an interleaved 1F1B schedule and enough microbatches to amortise the bubble, which costs `(PP−1)/(microbatches+PP−1)` of idle time. **Data parallelism** takes whatever's left — DP=64 — with **ZeRO/FSDP sharding the optimiser state**, which is the 800 GB that otherwise doesn't fit. So 4,096 = 8 × 8 × 64. Plus **activation checkpointing**, trading roughly 30% extra compute for a large memory saving, which is usually worth it because memory is the binding constraint.

**3. "Cluster MTBF is half a day. What do you do about it?"** `I1b` `A1c`
> Treat **goodput as the primary metric** and engineer the restart path, because over a nine-day run that's roughly twenty interruptions and each costs time-since-last-checkpoint plus restart time. Concretely: **asynchronous checkpointing** — copy state to host memory quickly and flush to storage in the background, because a blocking checkpoint of 1.6 TB means minutes of the entire cluster idling, repeatedly. **Fast restart** — hot spares pre-allocated, the job kept warm, avoiding a full NCCL world re-initialisation where possible. **Straggler detection**, which is the one people miss: every collective is a barrier, so **one thermally-throttled GPU paces 4,095 others**, and it looks like nothing in an aggregate utilisation chart. And **silent data corruption** — a card computing wrong results without erroring — which is rare per-card and not rare across four thousand of them, so periodic deterministic checks are worth their cost.

**4. "The loss spikes at step 40,000. What do you do?"** `F2a` `I1b`
> Whatever the **pre-agreed policy** says, and having that policy is the actual answer — at three in the morning with $3M sunk, you don't want to be improvising. The policy should specify: skip the offending batch and continue, roll back N steps and resume with a different data order, or lower the learning rate. In practice the first response is usually **skip-and-continue** while checking whether it recovers, because many spikes are a single bad batch. What I'd look at to decide: **gradient norms** around the spike, whether it's recovering on its own, and the loss against the **scaling-law prediction** from the small-scale ablations — which is the single most useful chart during a run, because it tells you whether you're on trajectory rather than just whether the number went down. If it diverges and doesn't recover, roll back to the last good checkpoint with a changed seed and a reduced LR.

**5. "How do you de-risk a run you only get to do once?"** `A2d` `I2a`
> By making sure **nothing is learned for the first time at 100B** — anything I can only discover at full scale is a design failure. Four things before the run. **muP** — maximal update parametrisation — which lets hyperparameters, crucially the learning rate, **transfer from a 1B proxy**, so I tune where it's cheap and use the result where it isn't. **Scaling-law ablations** across several small scales, which give me a predicted loss curve so I can tell by day two whether the run is on trajectory rather than waiting to find out. **A data bake-off at 1B** to settle the mixture, because mixture moves benchmarks by points and can't be predicted from first principles. And a **full-scale smoke run** of a few hours, which shakes out the infrastructure — NCCL config, checkpoint path, storage throughput, straggler behaviour — without burning meaningful budget. The failures that kill these runs are almost always infrastructural, and the smoke run is what finds them.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How should the serving cost change your training decisions?"** `A3b` `F3a`
> Substantially, and this is the thing I'd raise unprompted because the decision is **irreversible after training**. The clearest case is **attention architecture**: choosing GQA over multi-head attention is decided here and it sets the KV cache size **forever** — from Q7's arithmetic, GQA with 8 KV heads gives roughly 320 KB per token against ~3.2 MB for full MHA, which is a 10× difference in how many sequences you can batch, and batch size *is* throughput in decode. Get that wrong and you've built a model that is permanently ten times more expensive to serve. Beyond that: **depth versus width** at equal parameter count and equal loss — a wider, shallower model often serves better because pipeline depth costs latency; **vocabulary size**, which trades embedding memory against sequence length; and whether to train at a **slightly smaller N with more tokens**, which is Chinchilla-suboptimal for training loss but produces a cheaper model to serve for its quality. The framing: **if this model will serve a trillion tokens, inference dominates lifetime cost**, and a training decision that saves 20% of serving cost is worth more than the entire training budget.

**7. "Two weeks in, the loss is above your scaling-law prediction. Kill it or continue?"** `I2b` `F1b`
> The decision should already be largely made, because a **kill criterion agreed before the run** is the only defence against a sunk cost negotiating on its own behalf — and at $3M, nobody wants to be the person who stopped it. So first: is the gap **within the band** the ablations predicted, given noise? Scaling-law extrapolation has real error bars and a small gap may be nothing. Then the diagnostic order: is it **data** (a mixture or pipeline bug — check token throughput and per-source yield), **optimisation** (gradient norms, LR schedule, a spike that didn't fully recover), or **infrastructure** (silent corruption on a subset of nodes, which shows up as a loss curve that's subtly worse than it should be). What tips me toward killing: the gap **widening** rather than holding, or a diagnosis that implicates data — because a data bug means the remaining compute is being spent wrong and restarting costs less than finishing. What tips me toward continuing: a stable offset with healthy gradients, since some of that is recoverable in post-training. And I'd want **downstream evals at checkpoints**, not just loss, because loss can be slightly high while the model is fine on what it's for.

**8. "Your MFU is 55% and someone wants to push it to 65%. Worth it?"** `F3a` `A3b`
> Probably not, and I'd redirect the effort — which is the judgement the goodput framing is for. A 10-point MFU gain is about a **15% reduction in compute time**, roughly a day on an eight-day run, and it typically costs weeks of kernel engineering. Meanwhile, if goodput is 75% rather than 90%, that's costing **more than two days** and the fixes — async checkpointing, faster restart, straggler eviction — are usually days of work, not weeks. So the first question is always *where is the wall-clock actually going*, and the honest answer at this scale is usually restarts and stragglers rather than kernel efficiency. There's a second-order argument too: MFU work is **fragile** — it's tuned to a specific shape, parallelism config and library version, and it breaks when any of those change — whereas goodput work is **durable** and benefits every future run. I'd fund the MFU work only if profiling shows a specific large inefficiency (a badly-fused attention kernel, a communication pattern that isn't overlapping) rather than as a general optimisation push.

**9. "Half your cluster becomes unavailable for a week. What now?"** `A1c` `F2a`
> Decide deliberately between three options rather than reflexively continuing, and the right answer depends on where in the run I am. **Continue at half scale**: the parallelism config has to change — halving DP doubles the steps needed at the same batch size, or I keep the global batch by doubling gradient accumulation, which is the cleaner option since it preserves the optimisation trajectory and just runs slower. That's usually right if I'm late in the run. **Pause and checkpoint**, resuming when capacity returns — right if the outage is genuinely bounded and short, and it costs nothing but calendar. **Restart smaller** — right only if I'm very early and the schedule matters more than the target size. What I'd avoid is silently changing the effective batch size, because that changes the optimisation trajectory mid-run and makes the loss curve non-comparable to the scaling-law prediction I'm using to judge health — I'd rather run slower with the same global batch than faster with a different one. And I'd note the organisational point: **capacity risk should have been in the plan**, since a week's outage at this scale is not a surprising event.

**10. "Would you train this from scratch at all?"** `A5a` `F2c`
> That's the right question to ask before spending $3M, and I'd want it asked explicitly rather than assumed away. The alternatives: **continued pretraining** of an existing open-weights model of similar scale, which costs a fraction and gets most of the way for most purposes; **post-training only** — SFT and RL on an existing base, which is where a large share of task-specific quality actually comes from; or **distillation** from a larger model if one is available. Training from scratch is justified when there's a genuine reason the existing bases don't serve: a **licence or provenance requirement** (you need to know what's in the data, which matters for some enterprise and regulated deployments), a **materially different data mixture** — a language, a domain, or a modality the public models under-serve — or a **serving-cost-driven architecture** choice like the GQA and depth/width decisions above that you can't retrofit. What I'd resist is training from scratch for prestige or because it's the default framing of the question, since the honest comparison usually favours starting from something. If the answer is "we need it from scratch," that reason should be written down in the design doc, because it's also the thing that justifies the budget.

---

## ⭐ THE L6 LINE

> **"One shot, so nothing is learned for the first time at 100B — muP and scaling-law ablations de-risk it small. Goodput, not MFU, is the metric, because MTBF is half a day. And inference cost should be constraining training decisions today: GQA is decided here and sets the KV cache forever."**

## 🧭 HOOKS — *steer by what you mention*

- *"6ND, and here's the memory"* → invites the arithmetic branch
- *"Goodput, not MFU"* → invites the fault-tolerance/systems branch (**good at L6**)
- *"TP inside the node, PP across"* → invites the parallelism branch
- *"Decontamination is correctness, not hygiene"* → invites the data branch (Q8)
- *"Inference cost should constrain training today"* → invites the serving crossover (Q7) (**good at L6**)
- *"The kill criterion must exist before the sunk cost argues"* → invites the org/process branch (**good at L6**)

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me redo the arithmetic"* — 6ND and the 1.6 TB of state regenerate the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover fault tolerance and the serving constraint than finish the parallelism. Sound right?"*

**Name and defer:**
- "**The data pipeline** is genuinely its own design and it dominates quality — that's Q8."
- "**Post-training** — SFT and RL — is where a lot of task quality actually comes from."
- "**MoE** changes the memory and communication story completely: sparse compute, all experts resident."
- "**Long-context training** needs sequence/context parallelism and a different data construction."
- "**Multimodal** adds an encoder, an alignment stage and a whole separate data problem."
- "**Kernel-level optimisation** (FlashAttention, fused ops) matters but it's MFU work, not goodput work."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: one de-risked run | §0 | *"Anything I can only learn at 100B is a design failure."* |
| **A1b** | Compute, memory and cost budgets | §1, §3, FU 1 | *"6ND = 1.2e24 FLOPs ≈ 7.7 days; 1.6 TB of state; ~$3M."* |
| **A1c** | Degrade without lowering quality | §4, FU 3, 9 | *"Lose half the cluster → keep the global batch with gradient accumulation."* |
| **A2a** | The full stack | §4, FU 2 | TP 8 × PP 8 × DP 64 + ZeRO; data → trainer → async checkpoints → health → evals |
| **A2b** | Model components | §5.3 | Objective, muP, schedule, stability, GQA, shape |
| **A2c** | Data sources | §5.1 | Web, code, books, multilingual, math/synthetic, the eval suite |
| **A2d** | Staged pipeline | §5.5, FU 5 | Proxies → data → smoke → main → anneal → hand-off |
| **A3a** | Top-down, unprompted | §0 | *"One shot; goodput not FLOPs; data dominates"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 6, 8 | *"Time to bar = compute ÷ (MFU × goodput); checkpoint every √(2δM)."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Budget? Serving volume? Data pipeline? Hardware? |
| **A5a** | Alternatives compared | §4, FU 10 | From scratch vs continued pretraining; dense vs MoE; MHA vs GQA; sync vs async |
| **A5b** | A full alternative pipeline on demand | §9 | The pivot pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right objective and architecture | §5.3 | *"Cross-entropy; muP-tuned LR; QK-norm and z-loss for stability."* |
| **F1b** | Metrics and their trade-offs | §2, FU 7 | *"Loss vs the scaling-law band is the most useful single chart."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Mixture, filter bias against dialects, contamination, English-centric evals |
| **F2a** | Something fails → a fallback | §9, FU 4, 9 | The pivot pipeline; the spike policy |
| **F2b** | Too slow → design for it | §4 | Goodput, stragglers, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4, FU 10 | Continued pretraining or distillation as the real alternative |
| **F3a** | Efficiency with a quality trade | §6, FU 6, 8 | Activation checkpointing; FP8 only if proxies are stable |
| **F3b** | Scale levers | §6, FU 2 | The 3D layout; storage and loader sized to the cluster |
| **F3c** | Extension, with cost and maturity risk | §9 | MoE, long context, multimodal |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8, FU 1 | MFU, tokens/s, $ against plan |
| **I1b** | Reliability | §6, §8, FU 3, 4 | **Goodput > 90%**; hot spares; SDC checks |
| **I1c** | Usability | §2, §8 | Declarative configs; the dashboard; a documented checkpoint |
| **I1d** | Scalability | §3, §8 | Checkpoint storage and loader sized to 4,096 GPUs |
| **I1e** | Privacy | §8 | PII scrubbed; opt-outs; memorisation tests |
| **I1f** | Safety | §8 | Dangerous-capability evals at checkpoints |
| **I1h** | Security | §8 | **Weights as crown jewels**; data poisoning; signed code |
| **I1i** | Ethics / bias | §7, §8 | Filter audits; language coverage; carbon reported |
| **I1j** | Explainability | §2, §8 | Datasheet, run log, model card |
| **I2a** | Offline → online | §7, FU 5 | Proxies → smoke → main run → hand-off to Q4 and Q7 |
| **I2b** | The success definition | §2, §7, FU 7 | *"The bar met on clean evals, inside budget and calendar."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) "done" → §2 · (c) system → §4 · (d) data → §5.1 · (e) mixture → §5.2 · (f) objective → §5.3–5.4 · (g) running the job → §6 · (h) success → §2, §7 · (i) extension → §9.
