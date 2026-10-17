# Q7 · LLM Inference Serving — **INTERVIEW SHEET**

> **Tier 1 · #5** — prefill/decode, batching, KV cache. **Underpins every GenAI answer you'll give.**
> Full study file: [`../Q7_inference_serving.md`](../Q7_inference_serving.md) · [diagrams](../Q7_inference_serving_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product (the API) looks like · (c) the serving stack · (d) the workload data · (e) choosing the eval and replay sets · (f) fine-tuned models and quantization · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Traffic shape, TTFT/ITL targets, fine-tunes, tenants (§1) |
| **Explainability** is easy points | Per-request traces, per-tenant cost attribution, version stamps (§8) |
| **Security** beyond the gateway | **Token-weighted** limits, KV isolation, the **prefix-cache timing side channel** (§8) |
| **Troubleshoot before being asked** | Raise **latency too tight** and **p99 spikes** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **Two workloads behind one API** — prefill vs decode · the chain | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · **TTFT/ITL SLOs** · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The streaming API per priority class · **goodput**, not throughput | I1c, F1b, I2b |
| **9–12** | **CAPACITY + COST** | — | **Do the roofline out loud** → batch ~300 → KV caps it at ~61 · $/M tokens | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · router → scheduler → paged KV → execution · degrade | A2a, A5a, A1c, F2b |
| **22–32** | **WORKLOAD + MODEL MEMORY** | **(d) (e) (f)** | Traffic data · replay/eval sets · the KV ladder · multi-LoRA · pipeline | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Prefix cache + sticky routing · chunked prefill · the ladder | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Exact-artifact quality · replay load tests · canaries · fairness | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multimodal · long context · overload path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Let me start with the fact that determines the entire design: LLM inference is two different workloads behind one API.**
>
> **Prefill processes the whole prompt at once — a big matmul over many tokens, so it's compute-bound, it parallelises beautifully, and it sets time-to-first-token. Decode generates one token at a time, and each step must read the entire weight matrix from HBM to produce that single token — so it's memory-bandwidth-bound, inherently sequential, and it sets inter-token latency and almost all of your cost.**
>
> **The consequence is the most important line in this answer: because decode is bandwidth-bound, batch size IS throughput. You read the weights once whether you serve one sequence or a hundred."**

**Then the chain, which is the whole design:**
> **COST → THROUGHPUT → BATCH SIZE → KV CACHE MEMORY.**
> *"On an H100 in bf16 you need a batch near 300 to become compute-bound — and you almost never get there, because the KV cache runs out of memory first. So KV cache is the binding constraint on the economics of the entire system, which is why GQA, PagedAttention and KV quantization matter more than anything else on the list."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the API promises and how I measure it, the roofline and KV arithmetic, the architecture, the workload and memory levers, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What's the traffic shape — chat, agentic, or batch?** *(Agentic is prefill-heavy and prefix-cacheable; batch has no latency SLO at all.)*
2. **What are the TTFT and ITL targets?** *(These bound batch size, which bounds cost.)*
3. **Single model or many fine-tunes?** *(Multi-LoRA changes the serving design, §5.4.)*
4. **How many tenants, and may they share caches?** *(Isolation vs efficiency, §8.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Model | A **70B dense** model; bf16, moving to FP8 after an eval gate |
| Replica | **4×H100 80 GB, TP=4** inside one NVLink node |
| Traffic | Interactive chat + **agentic** (long, prefix-heavy) + offline batch |
| Tenants | Internal products and API customers, each with quotas |
| Context | Typical 8k; long-context requests up to 128k as their own class |
| Fine-tunes | Hundreds of LoRA adapters on the shared base |

**FR:** streaming · continuous batching · multi-tenant with quotas · prefix caching · admission control · graceful degradation · multi-LoRA serving.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **TTFT** | **p95 ≤ 500 ms** (interactive) |
| **ITL / TPOT** | **p95 ≤ 50 ms** (~20 tok/s — faster than reading) |
| **Goodput** | Maximise **tokens/sec that met SLO** |
| Utilisation | > 70% — below that you're buying idle GPUs |
| Availability | 99.9%; overload → **reject or shed at the door**, never a thrash spiral |
| Cost | Tracked in **$ per million output tokens**, per priority class |

**Scope / NON-GOALS:**
- ❌ **Optimising raw throughput.** It's trivially gamed by making everyone slow. **Goodput** instead.
- ❌ Training/fine-tuning infra · model quality · the prompt · retrieval (Q6).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the product is (b):** an API with promises per priority class.

```
POST /generate  {model, adapter?, priority: interactive | batch, prompt, max_tokens}
  → tokens STREAM back: first token fast (TTFT), then a steady rhythm (ITL)
  → over quota: 429 + retry-after, at the door — never a slow death mid-stream
  → every response stamped with model + quantization + adapter version

TENANT DASHBOARD: tokens used vs quota · p95 TTFT / ITL · cost
```

**The metrics:**

```
THREE LATENCY NUMBERS — genuinely different quantities:

  TTFT   prompt → first token        set by PREFILL   (compute-bound)
  ITL    gap between tokens          set by ONE DECODE STEP (bandwidth)
  E2E    TTFT + (out_tokens−1)×ITL   ← DECODE DOMINATES

  500-token answer @ 200ms TTFT, ~15ms ITL (4×H100) → ~7.7 s. Decode is ~97% of it.

⚠ WHY GOODPUT, NOT THROUGHPUT:
  raw throughput doubles if you let everyone's latency degrade —
  the graph goes UP while the product gets WORSE.
  GOODPUT = tokens/sec THAT MET THEIR SLO. Refuses that trade by construction.

⚠ AND THE FAIRNESS METRIC: p99/p50 TPOT RATIO
  high ratio ⇒ someone is being STARVED while the averages look fine
  (head-of-line blocking, or one tenant flooding the batch)
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Goodput per $**, by priority class | Useful tokens delivered, per unit of cost |
| Latency | TTFT and ITL by percentile | The SLOs |
| Fairness | **p99/p50 TPOT ratio**, per tenant | Someone starved while averages look fine |
| Efficiency | **Prefix-cache hit rate** · KV utilisation · achieved batch size | Where the money is |
| **Guardrails** | Quality of the **exact served artifact** · error / rejection rate · $/M tokens | Hard limits |

---

## 3 · CAPACITY + COST *(min 9–12)* — **⭐ DO THIS ARITHMETIC OUT LOUD** `A1b` `I1a` `I1d`

```
WHY DECODE IS BANDWIDTH-BOUND — 70B in bf16, batch 1:
     bytes moved  ≈ 140 GB (ALL weights, HBM → SRAM)
     FLOPs        ≈ 140 GFLOP  (2 × 70B, one token)
     intensity    = 1 FLOP/byte
   H100: ~989 TFLOP/s ÷ ~3.35 TB/s ≈ 295 FLOP/byte break-even
     → you need BATCH ≈ 300 to saturate compute.
     → time/token ≈ 140 GB ÷ 3.35 TB/s ≈ 42 ms   ← 1-GPU EQUIVALENT (it can't fit on one!)
     → on 4×H100, TP=4: 35 GB/GPU ÷ 3.35 TB/s ≈ 10.5 ms + 160 all-reduces ≈ 13–15 ms floor

   "FLOP/byte" = per byte of WEIGHTS read. Each bf16 weight (2 B) is read once per
     step and used by all B tokens (2B FLOPs) → intensity = B. Tokens add ~no bytes
     (activations are B×h, weights are h²).
   ⚠ EXCEPT KV: each sequence reads its OWN KV every step (2.6 GB at 8k) — NOT shared
     across the batch → attention stays bandwidth-bound at ANY batch, and ITL rises
     with batch × context. "Batch ≈ 300" is for the weight matmuls only.

WHY YOU NEVER GET THERE — KV CACHE:
   70B · 80 layers · GQA 8 KV heads · head_dim 128 · bf16
     per token/layer = 2(K,V) × 8 × 128 × 2B = 4,096 B
     × 80 layers                            ≈ 320 KB/token
     8k context → 2.6 GB PER SEQUENCE
   On 4×H100 = 320 GB:  −140 weights −20 overhead = 160 GB for KV
     → 160 / 2.6  ≈  61 CONCURRENT SEQUENCES

⚠ WE WANT ~300, WE GET ~61 → running at ~20% of achievable throughput,
  and the bottleneck is MEMORY, not compute. Every priority follows.

COST (one replica = 4 × H100 at ~$2/GPU-hr = ~$8/hr):
  61 sequences × ~50 tok/s ≈ 3,000 tok/s ≈ 11M tok/hr  → ~$0.73 per M output tokens
  FP8 weights + FP8 KV → batch ~190                    → ~$0.23 per M  (~3× cheaper,
                                                          via the batch ceiling)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Batching | Static vs **continuous** | Continuous: a finished sequence frees its slot every step (5–10×) |
| Phases | Colocated + chunked prefill vs **disaggregated** | Colocated first; disaggregate when prefill and decode need different parallelism (FU 8) |
| Parallelism | Higher TP vs **more replicas** | TP to fit and to cut latency; replicas to scale throughput |
| Quantization | FP8 (W8A8) vs INT4 weight-only | FP8 for high-batch serving; INT4 for memory-tight, low-batch deployments |

### The system — *draw this in 90 seconds* `A2a`

```
 REQUESTS (multi-tenant, priority-tagged)
       │
┌─ 1. ROUTER + ADMISSION CONTROL ────────────────────────────────
│ per-tenant quotas — ⚠ TOKEN-weighted, not request-weighted (100 vs 100k!)
│ ⚠ SESSION-STICKY routing (consistent hash) → lands on the replica that
│   holds this conversation's prefix cache; scored against queue depth
│ admit only what KV can carry to completion; else 429 at the door
│ OUTPUT: a request on one replica, or a clean rejection
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. SCHEDULER — CONTINUOUS BATCHING + CHUNKED PREFILL ─────────
│ finished sequences leave and new ones join EVERY step
│ long prompts prefilled in chunks, riding on decode's idle compute
│ priority classes: interactive before batch
│ OUTPUT: the batch for this step
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. KV CACHE MANAGER — PAGEDATTENTION ─────────────────────────
│ 16-token blocks + a block table per sequence (virtual memory for KV)
│ → waste < 4% (was 60–80% with contiguous pre-allocation)
│ + PREFIX SHARING: matched prefixes reuse cached blocks
│ OUTPUT: KV blocks for every sequence in the batch
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. MODEL EXECUTION — TP=4 inside the node (NVLink) ───────────
│ FP8 weights (eval-gated) · multi-LoRA adapters batched together
│ ⚠ TP to FIT the model, REPLICAS to SCALE
│ OUTPUT: one token per sequence per step
└──────┬─────────────────────────────────────────────────────────
       ▼
 STREAM TOKENS OUT · meter tokens per tenant · stamp versions
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **Latency too tight:** *"I don't degrade everyone together. **Priority classes** (interactive above batch), **route easy requests to a smaller model**, speculative decoding on the low-batch interactive pool, and under load **shed batch traffic first** and reject at the door. Rejecting is a feature; a thrash spiral that slows everyone down is the failure."*
- **p99 spikes while p50 is fine:** *"That's almost always head-of-line blocking from a long prefill, which is why chunked prefill is in the scheduler from day one (FU 5)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Continuous batching, not static** | "A finished sequence leaves and a new one joins *every step* — static batching wastes the whole batch waiting on the longest sequence" | Scheduler complexity; per-step overhead |
| **2** | **TP only to fit, then replicas** | "TP is a *latency* knob that costs aggregate throughput — 160 all-reduces per forward pass against one token's arithmetic" | Higher TP does free KV memory; it's a real trade, not one-directional |
| **3** | **Chunked prefill** | "One 100k prompt stalls every decode in flight — head-of-line blocking. Chunk it and interleave" | Slightly worse TTFT for the long request |

---

## 5 · WORKLOAD + MODEL MEMORY *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The data (d): the workload `A2c`

For a serving system the "dataset" is the **traffic**: prompt-length and output-length distributions per class, the share of each prompt that's a **reusable prefix**, the tenant mix, arrival burstiness, and the adapter mix. It drives capacity planning, scheduler tuning, load tests, and the quantization calibration.

### 5.2 Choosing the eval and replay sets (e) `F1c`

```
REPLAY SET       a week of real traffic (prompts redacted where required),
                 stratified by class: chat · agentic (long, prefix-heavy) · batch
                 · long-context · each top tenant → drives load tests
QUALITY SET      the model's eval suite, re-run on the EXACT quantized artifact,
                 sliced by LANGUAGE and by CONTEXT LENGTH (where quantization
                 damage concentrates)
CALIBRATION SET  a few thousand representative prompts to set FP8 scales
                 ⚠ not English-only, or other languages degrade
```

### 5.3 The memory levers: the KV ladder `A2b` `F1a`

| Lever | Effect | Note |
|---|---|---|
| **GQA** (8 KV heads not 64) | **8×** vs MHA — 8 query heads share each KV head | Decided at **training** time — the biggest single win, and you don't control it at serving. Why 8: ~MHA quality (GQA paper), and **one KV head per GPU at TP=8** |
| **PagedAttention** | KV waste **60–80% → <4%** | Over-reservation + fragmentation, not compression (vLLM paper) |
| **FP8 KV** | **2×** (320 → 160 KB/tok) | ⚠ **Gate on a long-context eval slice** — error accumulates over the softmax and long-range attention is low-margin |
| **FP8 weights** | 140 → 70 GB → **+70 GB for KV** | **Half the bytes per step** (≈2× lower ITL floor) **+ a bigger batch** — not faster math |
| **Prefix sharing** | Huge for agentic | A shared system prompt is stored once |

> **Say the quantization framing:** *"Quantization helps decode by **moving fewer bytes** — a lower latency floor and room for a **bigger batch** — not by faster math. FP8 doubles the H100's FLOP/byte ratio, but it also halves the bytes per weight, so the break-even batch stays ~300 and the faster tensor cores mostly sit idle in decode."*

### 5.4 Fine-tuned models in serving: the PEFT question (f) `F2c`

Product teams fine-tune with **PEFT (LoRA)**, so the fleet may need to serve hundreds of fine-tunes. Serve them on **one shared base model** with multi-LoRA batching (S-LoRA / Punica-style): adapters are small, loaded per request, and requests for different adapters share one batch. *"A thousand fine-tunes shouldn't mean a thousand fleets."* The quantization **objective** is the mirror image: pick the FP8 scales (per channel or per group) that minimise the quality loss on the calibration set, then prove it on the quality set.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · ARTIFACT    a new model or adapter version arrives
STAGE 2 · QUANTIZE    FP8 weights (+ KV) using the calibration set
STAGE 3 · QUALITY     the EXACT artifact vs bf16 on the quality set, by language and
                      context length → blocking if any slice regresses
STAGE 4 · LOAD TEST   the replay set at 1×, 2×, 5× load: goodput, p99/p50, preemptions
STAGE 5 · CANARY      one replica, then a region, then fleet-wide; automatic rollback
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

**Prefix caching + sticky routing — the biggest cost lever for agentic traffic:**
```
Agentic/chat resends the whole conversation each turn.
  → match the new request's tokens against cached KV blocks (chained hashes
    per 16-token block, or a radix tree), reuse every matched block, prefill
    only the SUFFIX.

⚠ BUT THE CACHE LIVES IN ONE REPLICA'S MEMORY.
  round-robin routing ⇒ 1/N hit rate. "A cache you don't land on
  is worthless." → SESSION-STICKY (consistent hashing), and route on
  LONGEST MATCHED PREFIX where you can.
  ⚠ It fights load balancing — score cache value against queue depth.
  At scale, a CLUSTER KV TIER (CPU memory / RDMA) turns a miss into a fetch:
  with GQA, moving 320 KB/token is ~10× cheaper than recomputing it.
```

**Chunked prefill is nearly free:** a decode step must stream all the weights anyway (~10.5 ms per GPU at TP=4), and in that time the tensor cores could process ~295 tokens. Sixty decodes use 60, so a ~235-token prefill chunk rides along at almost no cost. Past that, the chunk size trades everyone's ITL against one user's TTFT.

**The optimisation ladder, in order:**
1. Continuous batching (5–10×, table stakes)
2. PagedAttention (kills fragmentation)
3. **Prefix caching + sticky routing** (transformative for agentic)
4. FP8 weights + KV (~3× **via the batch ceiling**), eval-gated `F3a`
5. Chunked prefill (fixes long prompts stalling everyone)
6. **Speculative decoding — only at LOW batch** (follow-up 4)
7. Disaggregation (separate prefill/decode pools) at scale

- **The scheduling objective** `A3b`: maximise **goodput** subject to **token-weighted fairness** across tenants, with interactive priority over batch, rather than maximising raw throughput.
- **Scale** `F3b`: replicas to scale, TP to fit; prefix reuse; FP8 to raise the batch ceiling.
- **Reliability** `I1b`: admission control reserves KV to completion (no preemption thrash); a **warm pool** because cold start is 2–5 minutes; straggler detection inside TP groups.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
QUALITY GATE    the EXACT served artifact (quantized + adapter) vs reference, sliced
                by language and context length — never the bf16 model's numbers
LOAD TEST       the replay set: goodput, TTFT/ITL percentiles, p99/p50, preemptions
CANARY          one replica → one region → fleet, with automatic rollback
A/B             serving configs (FP8, chunk size, routing policy) compared on goodput
                per $ AND on quality signals from the product above
⚠ THE TRAP      a config that wins on a synthetic benchmark loses on real traffic,
                because the prompt-length and prefix-share mix decide everything
```

**Success** = goodput per $ up, TTFT/ITL SLOs met per class, p99/p50 bounded per tenant, quality unchanged on every slice. `I2b`

| Panel | Why |
|---|---|
| **The one number on the wall** | **Goodput by priority class** |
| **Prefix cache hit rate** | Largest cost lever for agentic, and it **degrades silently** when routing changes |
| **p99/p50 TPOT ratio** | A fairness metric; high means someone's starved |
| Also | Batch size distribution · KV utilisation · preemption rate · queue depth per tenant · TTFT/ITL by percentile |

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Quantization by language** | Low-resource languages and long-context reasoning degrade first | Quality gate sliced by language and length; multilingual calibration set |
| **Tenant starvation** | A heavy tenant floods the batch | Token-weighted quotas; per-tenant p99/p50 |
| **Length bias in scheduling** | Short requests always win; long ones starve | Aging in the queue; long-context class with its own SLO |
| **Benchmark bias** | Synthetic load hides real prefix and length mixes | Replay real traffic |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | TTFT p95 ≤ 500 ms, ITL p95 ≤ 50 ms; **goodput** as the headline; cost per million output tokens | Goodput · TTFT/ITL percentiles · $/M tokens |
| **Reliability** `I1b` | Admission control that reserves KV to completion; warm pool for the 2–5 min cold start; straggler detection in TP groups; versioned rollouts with rollback | Preemption rate · error rate · time to recover |
| **Usability** `I1c` | Streaming with **steady** inter-token latency (the p99/p50 ratio); TTFT fast enough to feel responsive; clean 429s with retry-after | p99/p50 TPOT ratio |
| **Scalability** `I1d` | Replicas to scale, TP to fit; prefix caching + sticky routing; disaggregation at scale; autoscaling on predicted load | Prefix hit rate · KV utilisation · headroom |
| **Privacy** `I1e` | ⚠ **Prefix caches shared across tenants can leak prompts through timing** (a cache hit is measurably faster; shown in 2024 research) → **isolate or salt caches per tenant**; prompt logs redacted and retention-limited | Cross-tenant cache isolation tests |
| **Safety** `I1f` | Input / output safety classifiers sit **on the serving path**, so they get their own latency budget, or someone will propose removing them | Classifier latency share |
| **Security** `I1h` | Auth + **token-weighted** rate limits; DoS via giant prompts → per-request KV caps; tenant isolation of KV memory | Rejected-request rate · per-tenant KV share |
| **Ethics / bias** `I1i` | ⚠ **Quantization damage is uneven**: low-resource languages and long-context reasoning degrade first → eval the quantized artifact **per language**; no tenant class starved entirely | Quality delta by language · per-tenant p99 |
| **Explainability** `I1j` | Every request carries a **trace** of where its time went (queue, prefill, decode, preemption); costs **attributed per tenant**; model / quantization / adapter version **stamped on every response** | Trace coverage |

> ⭐ *"For an inference platform, privacy and fairness live in the scheduler and the cache: a shared prefix cache is a side channel, and a quantized model is a different model for some languages."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multimodal inputs:** each image adds hundreds to thousands of tokens of prefill plus a vision-encoder pass on the critical path, and the KV cache grows with them. Image-heavy and long-context requests become their own **queue class** with their own SLOs, so they can't consume the interactive pool's KV budget.
- **MoE models:** all experts must be resident, so memory gets worse while compute per token drops; expert load imbalance becomes the new failure mode.

**When the pool is overloaded, the fallback pipeline** `A5b` `F2a`:

```
load above what KV can carry
  → SHED batch traffic first (it has no latency SLO)
  → ROUTE overflow interactive traffic to a SMALLER model, labelled as such
  → QUEUE with an honest retry-after for what remains
  → REJECT at the door rather than admit and thrash
  → meanwhile: scale from the WARM pool (cold start is minutes, not seconds)
```

**Top 3 failure modes:**
1. **Head-of-line blocking** — one 100k-token prefill stalls every decode in flight. p50 fine, p99 terrible. → chunked prefill.
2. **Preemption thrash** — admitted more than KV can sustain, so sequences get evicted and recomputed. Periodic spikes. → admission control.
3. **Silent prefix-cache collapse** — a routing change halves effective capacity and *nothing else looks different*.

**Q1 ships:** continuous batching + PagedAttention + **prefix caching with sticky routing** + the goodput/hit-rate/p99-ratio panels.
> *"Batching and paged KV are table stakes. Prefix caching is where the agentic cost actually is, and the panels ship with it because the cache fails silently."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why is decode so much slower per token than prefill?"** `A1a` `F1a`
> Different resources bind them. Prefill does a large matmul over many tokens, so there's plenty of arithmetic per byte of weights read — **compute-bound**. Decode produces one token and must read **every weight in the model** from HBM to do it: 140 GB for a 70B in bf16, which at 3.35 TB/s is about **42 ms per token** if one GPU could hold it — on 4×H100 with TP each GPU reads a quarter, so ~10 ms plus all-reduce overhead — with trivial FLOPs by comparison. So decode is **memory-bandwidth-bound** and the compute units sit nearly idle. The consequence is the most important sentence in the answer: **batching is nearly free in decode** — you read the weights once whether you serve 1 sequence or 100 — so throughput scales close to linearly with batch size. Which is why the whole design problem becomes "how do I make the batch bigger," and why the KV cache, which is what stops you, is the real subject.

**2. "How large does the batch need to be before you're compute-bound?"** `A1b` `F3a`
> Around **300 on an H100 in bf16**, and I'd derive it. A decode GEMM of shape [B,h]×[h,h] does 2·B·h² FLOPs while reading 2·h² bytes of weights, so arithmetic intensity is roughly **B FLOPs per byte**. The H100's ratio is 989 TFLOP/s over 3.35 TB/s ≈ **295 FLOP/byte**. So you need batch ≈ 300 to saturate compute, and below that adding requests is essentially free. In FP8 the hardware ratio doubles to ~590, but each weight is now one byte, so intensity doubles too and the **break-even stays ~300** — **quantization's benefit is fewer bytes (lower latency floor) and freed memory (bigger batch), not faster math.** One caveat: that's the weight matmuls — each sequence's KV read isn't shared across the batch, so attention never becomes compute-bound. And practically, a 70B at 8k context hits a KV ceiling near 60 sequences, so you're running at a fraction of achievable throughput the whole time.

**3. "Walk me through the KV cache math."** `A1b` `I1d`
> 70B, 80 layers, GQA with 8 KV heads, head_dim 128, bf16: per token per layer you store K and V, so 2 × 8 × 128 × 2 bytes = **4,096 bytes**; across 80 layers that's about **320 KB per token**. An 8k sequence is therefore 2.6 GB, 32k is 10.5 GB, and 200k is about **65 GB — most of a GPU for one request**. On 4×H100 you have 320 GB; weights take 140, activations and workspace ~20, leaving ~160 GB for KV ≈ **61 concurrent 8k sequences** — against the ~300 we'd want. So **KV memory holds us at roughly a fifth of achievable throughput**, and that single fact explains the priority order: GQA gives 8× over MHA (64 query heads sharing 8 KV heads) and is decided at training time; FP8 KV halves the per-token cost; PagedAttention cuts KV waste from 60–80% under contiguous pre-allocation to under 4%; prefix sharing lets sequences share blocks. **Every one of those is a throughput optimisation disguised as a memory optimisation.**

**4. "Should you use speculative decoding?"** `A5a` `F2b`
> Selectively, and knowing *when it stops working* is the real answer. It works because decode is bandwidth-bound: a small draft model proposes k tokens, the target verifies all k in one forward pass, and since you read the weights once either way, verifying five costs almost the same as generating one — so 2–3× on ITL. **But the benefit is a function of how bandwidth-bound you are.** As batch grows toward compute-bound, the draft model's compute and the wasted FLOPs on rejected positions become real work competing for saturated cores, and at high batch speculation can actively *reduce* throughput. So: deploy it on a **low-batch, latency-sensitive pool** — interactive chat with tight TPOT, or single-user/edge — and keep it off the saturated high-throughput pool. Monitor **acceptance rate**, not raw speedup: a poorly-aligned draft model gives 30% acceptance and you've paid for nothing.

**5. "p99 inter-token latency is terrible but p50 is fine."** `F2a` `I1c`
> That gap is almost always **head-of-line blocking from long prefills**. A 100k-token prompt is a large compute block, and while it runs every decode in flight stalls — so unrelated users see a spike mid-stream. Fix: **chunked prefill** — split it into chunks and interleave them with decode steps in the same batch, trading slightly worse TTFT for that one request against much better TPOT for everyone. The right trade, because a stutter mid-stream is more perceptible than a marginally slower first token. Other suspects in order: **preemption thrash** (admitted more than KV can sustain → evict and recompute → periodic spikes → fix with admission control); a **straggler in the TP group**, since all GPUs sync every layer; and **tenant contention**. I'd also make the **p99/p50 TPOT ratio** an explicit SLO — it's a fairness metric, and a high value means someone is starved while the averages look healthy.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "GPU utilisation is 95%. Are we efficient?"** `F1b` `I1a`
> Almost certainly not, and this is the most misleading number in the stack. "Utilisation" as reported by nvidia-smi means *a kernel was resident*, not that the arithmetic units were doing useful work — and in bandwidth-bound decode the GPU is busy **waiting on HBM** while the tensor cores idle. At batch 1 on a 70B you're at roughly 1 FLOP/byte against a 295 break-even, so you're using well under 1% of the compute while reporting high utilisation. The number I'd actually put on the wall is **goodput per dollar**, with **achieved batch size** and **KV utilisation** as the diagnostics — those tell you whether you're near the memory ceiling or leaving batch on the table. And I'd say the general lesson out loud: at this layer, **utilisation measures occupancy, not efficiency**, and optimising it directly will lead you to the wrong work.

**7. "One tenant is degrading everyone else. What do you do?"** `I1h` `I1i`
> This is the noisy-neighbour problem and the naive fix makes it worse. **Request-weighted quotas are the trap** — a tenant respecting a requests-per-second cap can still consume most of the fleet with 100k-token prompts, because a 100-token completion and a 100k-token one are not the same unit of work. So quotas must be **token-weighted**, and the scarcest resource to cap is **KV footprint**, since that's what determines how many sequences anyone can batch at all. Then **token-weighted fair queuing** for scheduling, **admission control** that rejects rather than degrades everyone when KV is exhausted, and a **burst allowance over a sustained floor** so idle capacity isn't wasted. The detection signal is the **p99/p50 TPOT ratio per tenant** — it moves before anyone files a ticket. And I'd note the prefix-cache interaction: sticky routing concentrates a heavy tenant on specific replicas, so tenant isolation and cache locality genuinely fight each other and the router has to score both.

**8. "When would you split prefill and decode onto separate hardware?"** `A5a` `A3b`
> When the two workloads' requirements have diverged enough that no single configuration serves both — which happens at scale with mixed traffic. The argument is that they want **opposite settings on almost every axis**: prefill is compute-bound, wants high TP (real FLOPs to divide, huge activations, collectives amortised over many tokens) and tolerates latency; decode is bandwidth-bound, wants **low TP** (160 all-reduces per forward pass against one token's arithmetic) and more replicas, and its SLO is per-token. Chunked prefill improves scheduling *within* one topology but can't give the two phases different parallelism — that's the ceiling that motivates **disaggregation**. The cost is that you must transfer the KV cache from the prefill pool to the decode pool, which is a real network problem and the main reason not to do it early. So: chunked prefill first, disaggregate when you can show the two phases are contending for genuinely incompatible configurations.

**9. "Your prefix cache hit rate dropped from 80% to 40% and nothing was deployed to the model."** `F2a` `I1b`
> Then **routing** changed, not the model — and this is the silent-degradation case I'd have built the panel for. The cache lives in one replica's memory, so the hit rate is a property of whether requests land where their KV already is. Causes in order: an **autoscaling event** added replicas and reshuffled the hash ring (which is why consistent hashing matters — it reshuffles ~1/N rather than everything); a **deploy or rolling restart** wiped cache on replicas that then came back empty; a **load-balancer config change** overrode session affinity; or a **traffic-shape change** — a new client not sending stable session IDs, so there's nothing to be sticky on. The consequence is what makes it urgent: for prefill-heavy agentic traffic, halving the hit rate can roughly **halve effective capacity**, and every other dashboard looks normal while cost per request doubles. That's exactly why hit rate is one of the three panels on the wall rather than a debugging metric.

**10. "The model is 2× bigger next quarter. What happens to your design?"** `F3a` `A1b`
> Weights double to ~280 GB, which does two things — and the second is the one people miss. Obviously I need more GPUs just to hold it, so minimum TP goes up. But the more damaging effect is on **KV headroom**: on the same 4×H100, weights would leave almost nothing for KV, so **batch size collapses**, and since batch *is* throughput in decode, cost per token rises far more than 2×. The levers, in order: **FP8 weights** to claw back ~140 GB, which buys batch rather than speed; **higher TP** to spread weights across more devices, accepting more collective overhead per token; and **KV quantization** with a long-context eval gate. I'd also re-check whether the **arithmetic intensity break-even** still implies the same target batch, and revisit **speculative decoding**, which becomes *more* attractive because larger models are more bandwidth-bound per token. The framing I'd give: **model size hits you twice — once on weight bytes, once on the batch you can no longer afford** — and only the second one shows up as a cost surprise.

---

## ⭐ THE L6 LINE

> **"Cost → throughput → batch size → KV cache memory. Every priority in this system follows from that chain, and the arithmetic — 320 KB/token, ~61 sequences where we'd want ~300 — is what makes it a memory problem rather than a compute one."**

## 🧭 HOOKS — *steer by what you mention*

- *"Batch size **is** throughput"* → invites the roofline/arithmetic branch
- *"KV cache is the binding constraint on the economics"* → invites the memory-optimisation branch
- *"A cache you don't land on is worthless"* → invites the routing/distributed-systems branch
- *"Goodput, not throughput — raw throughput is trivially gamed"* → invites the metrics/SLO branch (**good at L6**)
- *"TP is a latency knob that costs aggregate throughput"* → invites the parallelism branch
- *"Quantization's benefit is fewer bytes — a lower floor and a bigger batch — not faster math"* → invites the numerics branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me redo the KV arithmetic"* — 320 KB/token regenerates the entire design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the optimisation ladder in priority order than finish the sharding. Sound right?"*

**Name and defer:**
- "**Multi-LoRA serving** lets thousands of fine-tunes share one base fleet — that's its own design."
- "**Disaggregated prefill/decode** is the endgame at scale; the KV transfer is the hard part."
- "**MoE models** change the memory story completely — sparse activation, but all experts resident."
- "**Structured output / constrained decoding** adds per-token masking cost on the critical path."
- "**Cold start** on model load is minutes; it shapes autoscaling policy more than anything else."
- "**Long-context attention** (ring/sequence parallel) is a prefill-side technique, not a decode one."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: maximise batch, because decode is bandwidth-bound | §0 | *"Because decode is bandwidth-bound, batch size IS throughput."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"320 KB/token → 61 sequences where we'd want ~300; $0.73 per M tokens."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §9 | Priority classes, shed batch first, reject at the door |
| **A2a** | The full stack | §4 diagram | Router + admission → scheduler → paged KV → execution → stream |
| **A2b** | Model / memory components | §5.3 | The KV ladder: GQA, PagedAttention, FP8 KV and weights, prefix sharing |
| **A2c** | Data sources | §5.1 | The traffic: lengths, prefix share, tenant and adapter mix |
| **A2d** | Staged pipeline | §5.5 | Artifact → quantize → quality → load test → canary |
| **A3a** | Top-down, unprompted | §0 | *"Cost → throughput → batch size → KV memory"* + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Maximise goodput subject to token-weighted fairness."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Traffic shape · TTFT/ITL · fine-tunes · tenants |
| **A5a** | Alternatives compared | §4, FU 4, 8 | Static vs continuous; colocated vs disaggregated; TP vs replicas; FP8 vs INT4 |
| **A5b** | A full alternative pipeline on demand | §9 | The overload path: shed → smaller model → queue → reject |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right mechanism per subproblem | §5.3, FU 1 | *"Prefill is compute-bound, decode is bandwidth-bound."* |
| **F1b** | Metrics and their trade-offs | §2 | *"Goodput, not throughput; the p99/p50 ratio is a fairness metric."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Quantization by language; tenant starvation; length bias |
| **F2a** | Something underperforms → a fallback | §4, §9, FU 5, 9 | p99 spikes → chunked prefill; hit-rate collapse → routing |
| **F2b** | Latency too tight → tiering | §4 | Priority classes and smaller-model routing, raised **before** being asked |
| **F2c** | Fine-tuned models | §5.4 | Multi-LoRA on one shared base |
| **F3a** | Quantization with a quality trade | §5.3, §6 | *"Quantization moves fewer bytes; it isn't faster math."* |
| **F3b** | Scale levers | §6 | Replicas, prefix reuse, the cluster KV tier, FP8 |
| **F3c** | Extension, with cost and maturity risk | §9 | Multimodal prefill; MoE |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §2, §8 | Goodput, TTFT/ITL percentiles, $/M tokens |
| **I1b** | Reliability | §6, §8 | Admission control, warm pool, canaries |
| **I1c** | Usability | §2, §8 | A steady stream; clean 429s |
| **I1d** | Scalability | §6, §8 | Replicas, prefix caching, disaggregation |
| **I1e** | Privacy | §8 | **Prefix-cache timing side channel** → per-tenant caches |
| **I1f** | Safety | §8 | Safety classifiers budgeted on the serving path |
| **I1h** | Security | §8, FU 7 | Token-weighted limits; KV caps per request |
| **I1i** | Ethics / bias | §7, §8 | Quantization quality per language; no starved class |
| **I1j** | Explainability | §8 | Request traces, cost attribution, version stamps |
| **I2a** | Offline → online | §7 | Exact-artifact quality → replay load test → canary → A/B |
| **I2b** | The success definition | §7 | Goodput per $ up, SLOs met per class, quality unchanged |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) eval/replay sets → §5.2 · (f) fine-tunes and quantization → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
