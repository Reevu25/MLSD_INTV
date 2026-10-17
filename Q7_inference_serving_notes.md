# Q7 · LLM Inference Serving — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`Q7_inference_serving_intv.md`](Q7_inference_serving_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **⚠ Three of these exposed real errors, now fixed in the sheet, the study file, the diagrams and TIER1:**
> §3 the 42 ms "hard floor" was a **1-GPU-equivalent** for a model that can't fit on one GPU · §4 GQA is **8×**, not ~10× · §7 PagedAttention's waste figure is **60–80% → <4%**, not 30–60% · §9 *"FP8 makes you more memory-bound"* was **wrong**.

**Jump:** [1 goodput + SLO](#1--what-is-goodput-and-what-is-the-slo) · [2 ⭐ FLOP per byte of what?](#2--roofline-flop-per-byte-of-what) · [3 ⚠ 70B doesn't fit on one H100](#3--70b-doesnt-fit-on-one-h100) · [4 GQA: why 8 KV heads](#4--gqa-why-8-kv-heads-and-q-heads-per-kv-head) · [5 tenant + sticky routing](#5--tenant-and-sticky-routing) · [6 continuous batching + chunked prefill](#6--continuous-batching-and-chunked-prefill) · [7 PagedAttention + prefix sharing](#7--pagedattention-and-prefix-sharing) · [8 TP and latency](#8--tp-what-it-is-and-does-it-lower-latency) · [9 ⚠ FP8 and the ratio](#9--fp8-and-the-compute-to-bandwidth-ratio)

---

## 1 · What is goodput, and what is the SLO?

**SLO** = the sheet's NFR pair: **TTFT p95 ≤ 500 ms** and **ITL p95 ≤ 50 ms**. A request is "good" only if it met **both**.

**Goodput** = throughput counted **only over requests that met their SLO**. Canonical form (DistServe, 2024): *the max request rate at which ≥90% of requests meet both TTFT and TPOT.*

Bigger batch → more raw tokens/s, but everyone's ITL rises:

| Config | Raw throughput | Met SLO | Goodput |
|---|---|---|---|
| A: moderate batch | 2,000 tok/s | 98% | **~1,960** |
| B: admit everything | 3,500 tok/s | 40% (ITL p95 = 80 ms) | **~1,400** |

> ⭐ Throughput says ship B; goodput says A. *"The graph goes up while the product gets worse."* Goodput refuses that trade by construction.

---

## 2 · Roofline: FLOP per byte of *what*?

**Bytes of WEIGHTS read from HBM**, not token bytes.

```
each weight (bf16) = 2 bytes, read ONCE per decode step
that weight is used by all B tokens in the batch → 2·B FLOPs (multiply + add)

intensity = 2B / 2 = B FLOP/byte
H100      = 989 TFLOP/s ÷ 3.35 TB/s ≈ 295 FLOP/byte
→ compute-bound only when B ≳ 295  →  "batch ≈ 300"
```

**A token isn't measured in bytes here.** Inside the model it's one activation row (h = 8,192 numbers). Activations are B×h; weights are h², so **weights dominate the bytes** and batch is the only thing that reuses them.

⚠ **The KV exception:** each sequence also reads **its own** KV cache every step (2.6 GB at 8k), and that is **not shared** across the batch.
- At 61 seqs: ~160 GB of KV reads vs 140 GB of weights per step.
- **Attention stays bandwidth-bound at any batch.** "Batch ≈ 300" applies to the weight matmuls only.
- This is why ITL rises with **batch × context**, the trade goodput polices.

---

## 3 · 70B doesn't fit on one H100

**Correct.** 140 GB of weights > 80 GB (even the 94/96 GB variants). The sheet's 42 ms was a **single-GPU-equivalent** it never labelled.

```
42 ms    = 140 GB ÷ 3.35 TB/s           ← "if one GPU could hold it" (illustration)

4×H100, TP=4:
  each GPU holds + reads 35 GB → 35 ÷ 3.35 ≈ 10.5 ms
  + 160 all-reduces/token (NVLink)  → ~13–15 ms floor
  + KV reads at full batch (§2)     → ~20–25 ms at 61 × 8k
```

The **batch ≈ 300 break-even is unchanged** under TP: every GPU has the same ratio and the same intensity B.

---

## 4 · GQA: why 8 KV heads, and Q heads per KV head

**It's heads, not tokens.** Llama-70B: **64 query heads, 8 KV heads → 8 query heads share each KV head.**

**Tokens:** *every* token in the context stores its own K and V (per KV head, per layer). 4,096 B is per token per layer, so total = context length × 320 KB. 8,192 tokens → 2.6 GB.

```
MHA (64 KV heads): 2 × 64 × 128 × 2B = 32 KB/tok/layer → 2.6 MB/token → 8k = 21 GB  → ~7 seqs
GQA ( 8 KV heads): 2 ×  8 × 128 × 2B =  4 KB/tok/layer → 320 KB/token → 8k = 2.6 GB → ~61 seqs
```

⚠ **The saving is 64/8 = 8×, not "~10×".**

**Why 8, not 16 or 24:**
- **Quality:** the GQA paper (Ainslie et al., 2023) found ~8 groups ≈ MHA quality. MQA (1 head) loses quality; 16 doubles the KV cost for marginal gain.
- **TP fit:** serving is TP=8 inside a node → **exactly one KV head per GPU**. With TP greater than the KV-head count, heads are **duplicated** and memory is wasted. 24 doesn't divide over 8 GPUs.
- **Fixed at training time.** At serving you inherit it.

---

## 5 · Tenant and sticky routing

**Tenant** = a customer of the shared fleet with its own quota: internal products (Gemini app, Workspace, Search) or Vertex AI customers. Shared GPUs → one heavy tenant can starve the rest (noisy neighbour).

**Token-weighted quotas:** a 100k-token prompt ≈ 1,000× a 100-token one, so requests/sec is the wrong unit. The scarcest thing to cap is **KV footprint**.

**Sticky routing** = always send a **session** to the **same replica**: `consistent_hash(session_id) → replica`.

```
the prefix cache (KV of the conversation so far) lives in ONE replica's HBM
turn 5 resends turns 1–4:
  same replica  → prefill only the new suffix
  other replica → recompute everything
round-robin over N replicas → ~1/N hit rate
```

⚠ **It fights load balancing:** a hot session can't move to an idle replica, so the router scores **cache value against queue depth**.

---

## 6 · Continuous batching and chunked prefill

(It's **prefill** chunking. "Prefix" is the caching in §5/§7.)

| | How it works | Problem / gain |
|---|---|---|
| **Static batching** | Take 8 requests, run until **all** finish | Short answers' slots idle until the longest ends |
| **Continuous batching** (Orca, 2022) | Schedule **every decode step**: finished seq leaves, queued one joins | Batch stays full → **5–10×** |

**Chunked prefill** (Sarathi-Serve): a 100k-token prefill run in one go stalls every decode in flight (**head-of-line blocking**). Instead:

```
each scheduler step has a token budget (e.g. 2,048):
  [ 1 decode token for each of 60 running seqs ] + [ next 512-token chunk of the long prompt ]
```

⭐ **Why it's nearly free:** decode is bandwidth-bound, so the compute units sit idle. Prefill chunks are compute-heavy and **piggyback on that spare compute**. Decoders see a small ITL bump, not a stall; the long request pays a slightly worse TTFT.

### 📊 Figure: how a prefill chunk fills the idle compute

*Setup: 70B, 4×H100, TP=4. 60 sequences decoding, then a 100k-token prompt arrives. Weight matmuls only, at peak FLOPs, so the numbers are illustrative; real MFU is lower, so the real free room is smaller.*

**A. Inside ONE decode step, per GPU: where the idle compute comes from**

```
Every step must stream this GPU's 35 GB of weights from HBM, no matter what:
   35 GB ÷ 3.35 TB/s = 10.5 ms                    ← the step can't be shorter

In those 10.5 ms the tensor cores COULD do:
   989 TFLOP/s × 10.5 ms ≈ 1.0e13 FLOPs
   ÷ 3.5e10 FLOPs per token per GPU (2 × 70B ÷ 4)  ≈ 295 TOKEN-SLOTS per step
   (this is the roofline's "batch ≈ 300" again, seen from inside one step)

                       0 ms ─────────────────────────────────────── 10.5 ms
HBM (weight reads)     ████████████████████████████████████████████  100% busy

TENSOR CORES
 decode only (60)      █████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   60 / 295 slots ≈ 20%
                                ↑ ~235 slots IDLE: paid for, unused

 decode (60)           █████████▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓   295 / 295 ≈ 100%
 + prefill chunk (235)          ↑ the chunk rides on the SAME weight read
                                  → step still ≈ 10.5 ms → decoders see ~no ITL change

█ decode tokens   ▓ prefill-chunk tokens   ░ idle compute
```

⭐ **The mechanism in one line:** the weights are read **once per step** and then multiplied against **every token in the step**. Adding prefill tokens adds FLOPs but **no extra weight bytes**, so until the step reaches ~295 tokens, those FLOPs fill compute that was already idle.

**B. The timeline: without vs with chunking**

```
WITHOUT CHUNKING: the 100k prefill runs as one block

  time ─►  10ms 10ms 10ms ┌──────────── 100k prefill ≈ 3.5 s ────────────┐ 10ms 10ms
  decode     ■    ■    ■  │ · · · · · · · ·  ALL 60 STALLED  · · · · · · │  ■    ■
                          └───────────────────────────────────────────────┘
  → 60 users see a 3.5 s freeze mid-stream → ITL p99 explodes (head-of-line blocking)


WITH CHUNKING: each step = all decodes + one prefill chunk

  time ─►  ┌────┐┌────┐┌────┐┌────┐┌────┐        ┌────┐
           │■ ▓ ││■ ▓ ││■ ▓ ││■ ▓ ││■ ▓ │  ···   │■ ▓ │ → long prompt's FIRST TOKEN
           └────┘└────┘└────┘└────┘└────┘        └────┘
  → every step still emits a token for all 60 decoders: no freeze
  → the long prompt advances one chunk per step

  ■ = 60 decode tokens   ▓ = one prefill chunk
```

**C. The chunk size is a dial: ITL for everyone vs TTFT for the long prompt**

| Prefill tokens per step | Step time (≈ decoders' ITL) | Long prompt's TTFT (100k) |
|---|---|---|
| 0 (no long prompt) | 10.5 ms | — |
| **~235** (fills idle compute exactly) | **~10.5 ms**, i.e. free | 100k ÷ 235 ≈ 425 steps × 10.5 ms ≈ **4.5 s** |
| **512** | ~20 ms (now compute-bound, still < 50 ms SLO) | 196 steps × 20 ms ≈ **3.9 s** |
| 100k in one go | **3.5 s stall** ❌ | **3.5 s** |

> ⭐ **Say:** *"A decode step pays for ~300 tokens of compute and uses 60. Chunked prefill spends the other ~240 on someone's prompt for free. Push the chunk past that and you're trading everyone's ITL for one user's TTFT, so the token budget is set from the ITL SLO."*

⚠ **Two caveats:**
- The long request's TTFT gets **slightly worse** than an unchunked prefill (3.9–4.5 s vs 3.5 s). That's the price, and it's the right trade: a stutter mid-stream is more noticeable than a slower first token.
- Each chunk also reads the KV of the chunks before it, so late chunks of a 100k prompt cost more attention work. Hence "slightly", not "zero".

---

## 7 · PagedAttention and prefix sharing

> **One-line version:** PagedAttention stores the KV cache the way an OS stores a process's memory: in **small fixed-size pages scattered anywhere**, with a **per-sequence table** that maps "token position" → "where it physically lives". Memory is handed out **one page at a time, only when needed**.

### Step 1 · The problem: contiguous allocation

A sequence's KV cache **grows by 320 KB with every token**, and you don't know at admission how long the answer will be. So older servers **reserved a contiguous region for the max length** up front:

```
CONTIGUOUS (before): reserve max_len = 2,048 token-slots per request at admission

HBM ┌──────────── A: 2,048 reserved ────────────┐┌─ gap ─┐┌──────── B: 2,048 reserved ────────┐
    │█████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░││ free  ││██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
    └───────────────────────────────────────────┘└───────┘└───────────────────────────────────┘
     █ real tokens   ░ reserved but empty
```

| Waste | What it is |
|---|---|
| **Reservation** | Slots held **now** for tokens that don't exist yet |
| **Internal fragmentation** | Slots **never** used: A stops at 300 tokens, so 1,748 slots were wasted the whole time |
| **External fragmentation** | Free gaps **between** regions, each too small for a new 2,048-slot request |

⚠ **Measured (vLLM paper): 60–80% of KV memory wasted**; only ~20–38% held real tokens. Wasted KV = fewer sequences in the batch = lower throughput.

### Step 2 · The fix: fixed-size blocks + a block table

```
PHYSICAL KV POOL: carved into ~30k BLOCKS at startup
   one block = 16 tokens' K and V for ALL 80 layers ≈ 5 MB (≈1.3 MB per GPU at TP=4)

 block id:   0     1     2     3     4     5     6     7     8   ...
           ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
           │ B·0 │ A·1 │free │ A·2 │ B·1 │free │free │ A·0 │free │ ...
           └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

BLOCK TABLE per sequence (logical block → physical block):
   A (40 tokens):  [ 7, 1, 3 ]    A·0 = tokens 0–15 · A·1 = 16–31 · A·2 = 32–39 (8 of 16 used)
   B (20 tokens):  [ 0, 4 ]       B·1 = tokens 16–19 (4 of 16 used)
FREE LIST: [ 2, 5, 6, 8, … ]
```

⭐ **Two ideas do all the work:**
- **Blocks don't need to be adjacent.** A's tokens live in blocks 7, 1, 3, scattered. The block table hides that, so **external fragmentation disappears**: any free block fits any sequence.
- **Allocate on demand.** A holds exactly 3 blocks for 40 tokens, so **there's no reservation**.

### Step 3 · The lifecycle: grow, finish, run out

```
GROW     A generates tokens 40…47 → written into block 3 (slots 8–15)
         token 48: block 3 is full → pop a block off the free list (2)
         → A's table becomes [ 7, 1, 3, 2 ]     (one O(1) allocation every 16 tokens)

FINISH   A emits EOS → blocks 7, 1, 3, 2 go back to the free list IMMEDIATELY
         → the scheduler admits a waiting request next step (this is what makes
           continuous batching work: memory frees at token granularity)

RUN OUT  free list empty → PREEMPT a sequence: swap its blocks to CPU, or drop them
         and recompute later. Too much of this = "preemption thrash" (failure mode 2)
```

### Step 4 · How attention reads scattered blocks

Standard attention kernels expect each sequence's K and V as **one contiguous tensor**. The **PagedAttention kernel** takes the block table and gathers instead:

```
A's new token must attend to K,V of tokens 0…47:
   for token i:   physical block = table[ i // 16 ],   slot = i % 16
   e.g. i = 37 →  table[2] = 3,  slot 5   → read block 3, slot 5
```

- **The same block table serves all 80 layers.** Each layer has its own KV pool, indexed by the same block IDs. With TP, each GPU holds its KV heads' slice of every block.
- **Cost:** a little indirection per read. **Benefit:** 2–4× more sequences in the batch (vLLM paper). Easily worth it.

### Step 5 · Why the waste drops to <4%

```
a 1,000-token request:
   contiguous, max_len 8,192  → 7,192 empty slots          = 88% wasted
   paged, 16-token blocks     → 63 blocks = 1,008 slots     → 8 empty = 0.8% wasted

worst case per sequence = 15 empty slots (an almost-empty last block)
```

**Why 16 tokens per block:** smaller blocks waste less but mean more table entries and less efficient gathers. Larger blocks read efficiently but waste more in the last block. 16 is vLLM's default compromise.

### Step 6 · Prefix sharing falls out for free

Because sequences only hold **pointers** (block IDs), two sequences can point at the **same** physical block. Each block keeps a **reference count**.

```
A and C share the same 32-token system prompt, then diverge:

   A table: [ 7, 1, 3 ]        blocks 7 and 1: refcount = 2
   C table: [ 7, 1, 9 ]        ← C reuses 7 and 1: the prompt's KV is stored ONCE
                     └ C's own tokens start in a fresh block
```

- **Copy-on-write:** with parallel sampling (n = 4 answers to one prompt), all four share the prompt's blocks, *including* a partly-filled last block. When sample 2 writes into a shared block (refcount > 1), that block is **copied first**, then written. Exactly like `fork()` in an OS.
- **Finding a match across requests:** vLLM hashes each **full** block by *(previous block's hash, its 16 token IDs)*. A new request walks its prompt block by block, looking up each hash. It reuses blocks until the first miss, then allocates fresh. SGLang does the same with a **radix tree**. Reuse happens in **whole-block units**.
- **Freed ≠ forgotten:** a finished sequence's blocks with refcount 0 go on an **LRU evictable list**, not straight to the free list. The next request with the same prefix can still hit them. They're overwritten only when free blocks run out.

### The OS analogy, line by line

| OS virtual memory | PagedAttention |
|---|---|
| Process | Sequence |
| Page (4 KB) | Block (16 tokens) |
| Page table | Block table |
| Virtual address | Token position |
| Physical frame | Physical block in HBM |
| Demand paging | Allocate a block when the last one fills |
| Shared pages + copy-on-write (`fork`) | Prefix sharing + CoW (parallel sampling) |
| Swap to disk | Preemption: swap KV to CPU |

> ⭐ **Say:** *"PagedAttention is virtual memory for the KV cache: 16-token blocks anywhere in HBM, a block table per sequence, allocated on demand. That kills reservation and external fragmentation, leaving at most one partly-filled block per sequence, so waste goes from 60–80% to under 4%. And because sequences hold pointers, prefix sharing and copy-on-write come for free."*

---

## 8 · TP: what it is, and does it lower latency?

**Tensor parallelism** (Megatron-style) **splits every weight matrix across GPUs**:

```
attention: heads split across GPUs            → ALL-REDUCE
MLP:       W1 split by columns, W2 by rows     → ALL-REDUCE
→ 2 per layer × 80 layers = 160 all-reduces per token
```

**To fit:** 140 GB ÷ 4 = 35 GB/GPU, with room left for KV.

**Does higher TP lower latency? Yes, with diminishing returns:**

| | Effect of higher TP |
|---|---|
| **TTFT** (prefill, compute-bound) | FLOPs split N ways, comms amortised over many tokens → **drops ~N×** |
| **ITL** (decode, bandwidth-bound) | Each GPU reads 1/N of the weights → **drops**, but 160 all-reduces add a roughly **fixed** cost per token → diminishing returns |
| **Throughput per GPU** | More comms per token → **often lower** than more replicas at lower TP |
| **KV room** | Weights spread thinner → **more KV** per replica (why it's a real trade) |

⭐ **NVLink matters because those 160 all-reduces are on the critical path of every token.** NVLink ~900 GB/s per H100 is fine; InfiniBand across nodes ~50 GB/s per GPU is not → **TP stays inside one 8-GPU node**.

> **TP is a latency knob.** "TP to fit, replicas to scale" is the default.

---

## 9 · FP8 and the compute-to-bandwidth ratio

⚠ **The sheet's sentence was wrong.** The *hardware* ratio does double in FP8 (1,979 ÷ 3.35 ≈ 590), but FP8 weights are **1 byte**, so the *workload's* intensity doubles too:

```
bf16: intensity = 2B / 2 bytes = B    vs hardware 295 → break-even ≈ 300
FP8:  intensity = 2B / 1 byte  = 2B   vs hardware 590 → break-even ≈ 300  (UNCHANGED)
```

**What FP8 actually buys in decode:**
1. **Half the bytes per step** → the ITL floor roughly **halves**. Faster, but because of **fewer bytes**, not faster math.
2. **Half the weight memory** (140 → 70 GB) → **+70 GB for KV** → a bigger batch.
3. The **2× faster FP8 tensor cores barely matter** in decode, because you're not compute-bound. They help **prefill**.

*Weight-only INT4/INT8* (dequantise to bf16 for the math) cuts bytes **without** raising peak FLOPs → break-even drops to ~150 → **less** memory-bound. That's why INT4 can be **slower at high batch**: dequantisation becomes real work.

> ⭐ **Say:** *"Quantization helps decode by moving fewer bytes, which gives a lower latency floor and room for a bigger batch, not by faster math."*

---

## 🔁 THE NINE THINGS I KEPT GETTING WRONG

1. **Goodput counts only requests that met BOTH TTFT and ITL.** Raw throughput can rise while goodput falls.
2. ⭐ **"FLOP/byte" is per byte of WEIGHTS.** Each weight is read once and reused by B tokens → intensity = B → batch ≈ 300.
3. ⚠ **KV reads aren't shared across the batch.** Attention stays bandwidth-bound at any batch; ITL grows with batch × context.
4. ⚠ **42 ms is a 1-GPU-equivalent.** 70B bf16 doesn't fit on one H100; at TP=4 it's ~10.5 ms + all-reduces ≈ 13–15 ms.
5. **GQA is heads, not tokens:** 64 Q heads share 8 KV heads → **8×** saving. 8 = near-MHA quality + one KV head per GPU at TP=8.
6. **Sticky routing exists because the prefix cache lives in one replica's HBM.** Round-robin → 1/N hit rate.
7. **Chunked *prefill*** (not prefix) piggybacks compute-heavy chunks on decode's idle compute. Continuous batching = schedule every step.
8. **PagedAttention = virtual memory for KV:** 16-token blocks + block table → waste **60–80% → <4%**. Prefix sharing = two block tables pointing at the same blocks.
9. ⚠ **FP8 does NOT make you more memory-bound.** Hardware ratio and intensity both double → break-even stays ~300. The win is fewer bytes + bigger batch. **TP lowers TTFT ~N× and ITL with diminishing returns**, and stays inside NVLink.
