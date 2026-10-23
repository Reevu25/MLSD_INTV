# TIER 1 — QUICK REFERENCE
### Do not walk in without these ten

> One page per question: **the opening**, **the diagram**, **the defensible choices**, **the number**.
> Full sheets are `*_intv.md` in this folder. This file is the skim layer — use it the morning of.

**The timed spine is the same for all thirty sheets:**
`0–1 opening · 1–6 requirements · 6–9 product + metrics · 9–12 capacity · 12–22 architecture · 22–32 data & model · 32–38 serving · 38–44 eval, rollout, bias · 44–48 production-readiness sweep · 48–52 extension + failures · 52–60 probes`

> ⭐ **Graded against Google's L6 form (3 parameters, 10 sub-parameters):** all ten Tier-1 sheets are rewritten in the [R1](R1_prompt_suggestions_intv.md) / [R2](R2_video_query_search_intv.md) mould, with the same 60-minute structure: **§0** opening · **§1** requirements (questions asked unprompted + an assumptions table) · **§2** product + metrics · **§3** capacity · **§4** architecture (alternatives, a TASK/OUTPUT stack, problems raised before you're asked) · **§5** data, training-set selection and the training objective · **§6** serving · **§7** eval, rollout and bias · **§8 production-readiness sweep** (all nine pointers: performance · reliability · usability · scalability · privacy · safety · security · ethics/bias · explainability, at ≈ min 44–48) · **§9** extension + failures. Every heading and follow-up carries its rubric codes, and each sheet ends with a 32-pointer coverage map.
>
> **Practice prompts in the same format:** [R1](R1_prompt_suggestions_intv.md) prompt suggestions · [R2](R2_video_query_search_intv.md) video-as-query search · [R3](R3_discover_weekly_intv.md) weekly discovery playlist (Discover Weekly–style, refreshed after every 10 songs) · [R4](R4_watch_next_losses_optimizers_intv.md) watch-next recommender, with every loss and the optimizer updates derived (§5 losses, §6 optimizer replace data & serving) · [R5](R5_product_or_video_recsys_intv.md) product **or** video recommender: one platform, two domains, and where they diverge · [R6](R6_realtime_recsys_intv.md) real-time recommender: offline / nearline / online, the feature store, caching and fallbacks (full question included) · [R7](R7_contextual_bandit_recsys_intv.md) contextual bandit, 4 candidates → 1: neural-linear Thompson sampling, propensities, replay/DR, the launch plan (full question included) · [R8](R8_rag_system_design_eval_intv.md) RAG over a private corpus: hybrid retrieval, ACL and deletion guarantees, and a stage-by-stage eval matrix (failure mode · metric · labels) (full question included) · [R9](R9_internal_doc_search_agent_intv.md) agent for internal document search: tools run as the user, permissions re-checked at query time, a routed bounded agent, the useful failure, cache safety across users (full question included) · [R10](R10_fraud_detection_platform_intv.md) fraud detection across transactions, logins, signups and ad clicks: one risk platform, a model and cost matrix per event type, cold start, imbalance, the delayed-label loop, preventing model exploitation (full question included).

---

## 1 · G13 — YouTube Recommendation
*The single most-asked ML SD question anywhere.*

**OPENING.** *"Everyone draws the same three boxes — candidate generation, ranking, re-ranking — and the architecture is genuinely not where the difficulty is. **The hard part is what you optimise, and what you refuse to regress.** Watch-time pathologies aren't bugs; they're the system correctly maximising a wrong proxy. Every dashboard improves while the product gets worse."*

**THREE COMMITMENTS.** Multi-task **with negative heads** (dismiss, block, survey regret) · **named guardrails as ship blockers** · **diversity comes from candidate generation, not re-ranking**.

```
USER → CANDIDATE GEN (N sources, EACH WITH A QUOTA) → ~1000
     → MULTI-TASK RANKER (+ negative heads, + position tower) → ~100
     → RE-RANK (trajectory diversity) → ~20
     ⚠ logs are the output of the CURRENT policy
```

**DEFENSIBLE CHOICES**
- **Multi-task, not single** — no negative head ⇒ can't learn a watch was unwanted. *Cost: label sparsity; weights need an owner.*
- **Diversity in retrieval** — you can't re-rank variety into a homogeneous set. *Cost: quotas cost short-term engagement.*
- **Exploration is funded** — ecosystem funding; without it only incumbents are reachable. *Cost: a measured engagement hit.*

**NUMBER.** 10⁹ videos → ~1000 → ~100 → ~20, in **100 ms**. The funnel isn't an optimisation, it's the only feasible shape.
**⭐** *Naming the objective's pathologies before being asked, and knowing diversity is a retrieval problem.*

---

## 2 · Q6 — RAG / Retrieval for an Agent
*The GenAI default. Every 2026 loop has one.*

**OPENING.** *"Retrieval for an agent is not retrieval for a chatbot. For an agent it's a tool called repeatedly — so I optimise **cheap, high-recall, filterable retrieval I can call five times**, not one perfect ranked list. And before building any of it: **do we need RAG at all?** Long context is simpler for a small stable corpus."*

**THREE PRIORITIES.** **First-stage recall** (a miss is unrecoverable) · **permission correctness** (a security property, enforced in the query) · **failure attribution**.

```
QUERY → resolve principals → [BM25 ∥ ANN]  ⚠ ACL PREDICATE INSIDE BOTH
      → RRF fuse → 100 → authoritative ACL re-check
      → CROSS-ENCODER RERANK (~120 ms, dominates) → 10 → generator
```

**DEFENSIBLE CHOICES**
- **Hybrid, not dense-only** — they fail differently; for recall I want the union. *Cost: two indexes.*
- **ACL pre-filter** — post-filtering leaves the slots the excluded docs should have filled simply empty. *Cost: filtered ANN fragments.*
- **Quantize + retrieve wider** — PQ blurs the ranking, doesn't lose the chunk. *Cost: only while k is affordable to rerank.*

**NUMBER.** 200M chunks × 768 × 4B = **614 GB fp32 → 154 GB int8**. HNSW is a dependent graph walk, so it must be resident.
**⭐** *Recall@100 is the ceiling; the attribution split — was the gold chunk retrieved? — is what stops a quarter of prompt-tuning.*

---

## 3 · G21 — Ad Click Prediction
*Calibration > AUC, with the mechanism.*

**OPENING.** *"The output is **a price, not a ranking**. `expected value = pCTR × bid`, so the probability is multiplied by money. **Calibration beats AUC and it's not close** — AUC is invariant to monotone transforms, so it **literally cannot see** a model running 20% high."*

**THREE COMMITMENTS.** Log loss + **calibration by slice** as headline · hashed sparse features + memorisation/generalisation hybrid · **position bias and the downsampling correction**.

```
REQUEST + ~1000 ads → ⚠ FEATURE FETCH (the real bottleneck)
   → hashing → embeddings → wide+deep (+ position tower)
   → ⚠ DOWNSAMPLING CORRECTION  p = p'/(p' + (1−p')/w)
   → calibration layer → pCTR × bid → AUCTION
```

**DEFENSIBLE CHOICES**
- **Calibration layer on top** — refit hourly per slice, cheaply. *Cost: can mask a real model regression.*
- **Hashing over vocabulary** — billions of values, new ones constantly. *Cost: collisions.*
- **Downsample then correct** — base CTR <1%. *Cost: forget the correction and you over-charge every advertiser.*

**NUMBER.** ~10⁵ requests/s × ~1000 ads/request = **~10⁸ predictions/s**, **≤10 ms p99** per request (~10 µs/prediction). Model forward ≈1 ms; **the fetch is the rest**.
**⭐** *Counter features leak the label unless point-in-time correct — great offline, collapses online.*

---

## 4 · G25 — Harmful Content Detection
⭐ *Confirmed asked in a Google L6 loop.*

**OPENING.** *"It is **a policy problem with a classifier attached**, and the ordering matters — the highest-leverage artifact isn't the model, it's the **policy taxonomy**. Without it annotators disagree, the model learns their disagreement as noise, and every metric is uninterpretable."*

**THREE COMMITMENTS.** N policies, each with its own threshold and **action ladder** · **the borderline band is the product** and **review capacity sets the thresholds** · **prevalence (view-weighted) is the headline**.

```
UPLOAD → STAGE 0 HASH MATCH (cheapest, highest-value, non-ML)
       → cheap per-modality → expensive MULTIMODAL FUSION (combination harms)
       → per-policy thresholds → [auto-allow │ BORDERLINE │ auto-act]
       → QUEUE ordered by P(violating) × REACH × severity
       → action ladder → APPEALS ──► training data
```

**DEFENSIBLE CHOICES**
- **N policies, not one classifier** — prevalence spans six orders of magnitude and the cost ordering doesn't match. *Cost: N models.*
- **Capacity sets thresholds** — else the effective policy is "whatever review got to," chosen by nobody. *Cost: you can't freely pick a P-R point.*
- **Hash matching before any ML** — exact-match on known-bad is near-perfect and free. *Cost: known content only.*

**NUMBER.** 5M items/day vs 30k reviews/day ⇒ **≥99.4% must be automatic**. That's arithmetic, not preference.
**⭐** *Prevalence by views, because harm scales with reach — item-level P/R can look excellent while distributed harm goes unmeasured.*

---

## 5 · Q7 — LLM Inference Serving
*Underpins every GenAI answer you give.*

**OPENING.** *"Inference is **two workloads behind one API**. Prefill is compute-bound and sets TTFT. Decode reads **every weight from HBM to produce one token** — memory-bandwidth-bound, sequential, and nearly all the cost. So **batch size IS throughput**."*

**THE CHAIN.** `COST → THROUGHPUT → BATCH SIZE → KV CACHE MEMORY.`

```
REQUESTS → ROUTER (token-weighted quotas · SESSION-STICKY for prefix cache)
         → CONTINUOUS BATCHING (+ chunked prefill)
         → PAGED KV (+ prefix sharing via radix tree)
         → TP inside the node only  ⚠ TP to FIT, REPLICAS to SCALE
```

**DEFENSIBLE CHOICES**
- **Continuous, not static batching** — a finished sequence leaves every step. *Cost: scheduler complexity.*
- **TP only to fit** — 160 all-reduces/pass against one token's arithmetic. *Cost: higher TP does free KV; it's a real trade.*
- **Chunked prefill** — one 100k prompt stalls every decode in flight. *Cost: slightly worse TTFT for that request.*

**NUMBER.** 70B bf16: **140 GB per token-step ÷ 3.35 TB/s ≈ 42 ms** 1-GPU-equivalent (~13–15 ms on 4×H100, TP=4). Need batch ~**300** to be compute-bound; KV at **320 KB/token** caps you at **~61**. Running at ~20%.
**⭐** *Goodput, not throughput — raw throughput doubles by making everyone slow.*

---

## 6 · G31 — Maps ETA Prediction
⭐ *Confirmed asked in a Google L6 loop ("traffic data").*

**OPENING.** *"Fix the output type first: **an ETA is a distribution, not a number**. The question is 'will I be late' — a **tail** question. So: quantiles with pinball loss, and **different surfaces get different quantiles** because their loss functions genuinely differ."*

**THREE COMMITMENTS.** **A route's ETA is not the sum of its segments'** · **our routing changes the traffic we predict** (equilibrium target, counterfactual eval, real externality) · **stability fights accuracy**.

```
PROBES → MAP-MATCH (⚠ STOPPED ≠ STUCK) → TRAFFIC STATE (⚠ k-anon gate)
   ├─► EDGE-COST TABLE (~1 µs) → ROUTER  ⚠ NO NN INSIDE A*
   └─► GNN → supersegment embeddings → ROUTE MODEL
        + per-supersegment LIVE/HISTORICAL BLEND BY ARRIVAL HORIZON
        → QUANTILES → surface logic → ⚠ IN-TRIP REFRESH + HYSTERESIS
```

**DEFENSIBLE CHOICES**
- **Quantiles, not a point** — the question is about the tail. *Cost: more to calibrate.*
- **Route-level over supersegments** — quantiles don't add; turns belong to no segment. *Cost: harder error attribution.*
- **Precomputed aggregates** — you can't walk a billion-node graph in 100 ms. *Cost: slight staleness, which is fine.*

**NUMBER.** In-trip refresh is **2–5× the initial-request path** (~140k QPS vs ~70k). Design for cheap incremental re-scoring.
**⭐** *Display ≈ P50, "leave by" ≈ P90 — the asymmetry lives in the quantile choice, explicitly.*

---

## 7 · G29 — Fraud Detection
*Named in multiple "most common" lists.*

**OPENING.** *"Extreme imbalance means **ROC-AUC is actively misleading** — its FPR denominator is the enormous legitimate population. **PR-AUC is the headline.** But the bigger point: **both error types are denominated in currency**, which almost no other ML problem gives you."*

**THREE COMMITMENTS.** Threshold from a **cost matrix with customer LTV inside `C_FP`** · **the graph is where the fraud is** · **you never learn about your blocks**.

```
TXN → ENTITY RESOLUTION → FEATURE ASSEMBLY
      + PRECOMPUTED GRAPH AGGREGATES  ⚠ A LOOKUP, NOT A TRAVERSAL
   → GBDT (explainable for regulated declines) → calibrated P(fraud)
   → COST-DERIVED LADDER: allow → ⭐STEP-UP AUTH → hold → decline → ring action
   + ⚠ RANDOM ALLOW-THROUGH  → QUEUE by P × value × COMPONENT SIZE
```

**DEFENSIBLE CHOICES**
- **Cost matrix, not F1** — F1 has no idea what a chargeback costs. *Cost: needs real LTV from outside the team.*
- **GBDT, not a GNN, in the hot path** — explainability is often regulatory. *Cost: leaves graph signal on the table.*
- **Budgeted random allow-through** — the only unbiased signal about the blocked population. *Cost: deliberate fraud loss.*

**NUMBER.** 1M txns, 1k fraud: flag 10k catch 500 → **FPR 0.95% "excellent", precision 5%** — 19 of 20 flags is a real customer.
**⭐** *Step-up auth converts an uncertain block into a cheap test **that also generates a label**.*

---

## 8 · Q5 — Model-Promotion Gating / Eval Harness
⚠ *Scope this in the first 60 seconds — two deliverables, one room.*

**OPENING.** *"Evaluation is CI for models — with one difference that I'd design around: **when CI breaks, your tests fail. When an eval breaks, it keeps producing numbers.** So the eval is a system under test, with its own SLOs, regression suite and monitoring."*

**SCOPING MOVE.** *"Offline gating or online reward? I'll build **offline as the spine** — but the RL loop shares the task definitions, scoring logic and judge, and keeping them the same code is a correctness requirement."*

```
MODEL + SUITE → IMMUTABLE REGISTRY → ⚠ CACHE LOOKUP BEFORE DISPATCH
   → runners + sandboxes → 4 SCORERS (exact · execution · JUDGE · human)
   → immutable results (+ CIs, noise floor) → DECLARATIVE GATE
   ⚠ HARD GATES: safety · canaries · contamination · κ — capability can't outvote
   + FROZEN REFERENCE MODEL: its score must NEVER change
```

**DEFENSIBLE CHOICES**
- **`harness_ver` in the cache key** — a scorer bugfix must invalidate every result it touched. *Cost: hit rate after changes.*
- **Share the library with RL, not the judge model** — else the policy optimises against its own approver. *Cost: two judges to calibrate.*
- **Refuse the composite score** — it destroys actionability and gives the org one thing to Goodhart. *Cost: leadership wants one number.*

**NUMBER.** T0 smoke ~200 stratified tasks, **$15/20 min**, every commit · T1 full 10k, **$500/3 h**, nightly.
**⭐** *Contamination doesn't degrade the model — it **invalidates the measurement**. Canary strings give certainty.*

---

## 9 · G14 — Discover Feed
*The other half of the canonical Google reco prompt.*

**OPENING.** *"**There's no query and no action that started the session.** I'm ranking against an **inferred goal** with almost no signal about whether the inference was right — a scroll-past tells me nothing. So this is the hardest **measurement** problem in the group, and **metric design comes before model design**."*

**TWO COMMITMENTS.** **Freshness and enduring interest are different axes** (sources + quotas + per-topic timeliness) · **⚠ Discover *externalises* the profile** — so an accurate inference can be harmful.

```
OPEN → PROFILE ASSEMBLY  ⚠ SENSITIVE-CATEGORY SUPPRESSION *PRE-RETRIEVAL*
     → SOURCES WITH QUOTAS (enduring · breaking · local · exploration)
     → MULTI-TASK RANKER × TIMELINESS-SENSITIVITY PER TOPIC
       (+ negative head: not-interested)
     → re-rank (cross-day dedup) → FEED + "why am I seeing this" + CONTROLS
```

**DEFENSIBLE CHOICES**
- **Suppression pre-retrieval** — post-filtering leaves an empty slot and the pattern still leaks. *Cost: lose genuinely wanted content.*
- **Timeliness per topic** — a sports score decays in hours, a recipe not at all. *Cost: a per-topic model.*
- **Non-interaction is missing, not negative** — a scroll-past can't distinguish bad cards from a lift. *Cost: far fewer labels.*

**NUMBER.** Candidate pool is **time-bounded, not corpus-bounded** (~10⁶–10⁷ live). The cost is **freshness**, not index size.
**⭐** *Return rate, not session engagement. Engagement up + opens down = narrowing.*

---

## 10 · Q10 — Agent Safety
*Agents + injection are explicitly tested in 2026.*

**OPENING.** *"Two reframings. **Safety is a system property, not a model property** — so layers, each with what it catches and what it costs. And **the dominant risk for an agent is not saying a harmful thing, it's *doing* one** — which relocates the design from content classifiers to **capability restriction and blast-radius tiering**."*

**THE PRINCIPLE + THE COUNTERWEIGHT.** ⭐ *"The agent cannot do the dangerous thing because it **has no credential for it**, not because it was told not to."* · ⚠ **Over-refusal is a real failure, not a safe default.**

```
T1 misuse │ T2 INJECTION │ T3 UNAUTHORIZED ACTION │ T4 leakage │ T5 emergent
⚠ for an AGENT, T2+T3 dominate — and classifiers barely touch either

REQUEST → input classifier → AGENT LOOP (untrusted content FENCED as data)
  → ⭐ CAPABILITY RESTRICTION → BLAST-RADIUS TIERS T0–T4 + approval gates
  → SANDBOX, ⚠ NO DEFAULT EGRESS → output classifier
  → immutable audit · monitoring · KILL SWITCH (works MID-ACTION)
```

**DEFENSIBLE CHOICES**
- **Capability over classifiers** — prompt controls are probabilistic, capability controls deterministic. *Cost: design effort per tool.*
- **Gate only T2/T3; reads are free** — gating reads trains reflexive approval. *Cost: risk accepted on reversible actions.*
- **Prefer *undo* to *approve*** — frictionless and equally protective. *Cost: needs real undo infrastructure.*

**NUMBER.** Harmful action rate **0 for the top tier** — irreversible actions can't be a percentage. Kill switch **<5 s, mid-action**.
**⭐** *Tier-3 **approval latency**: if users approve in under two seconds, the gate is theater.*

---

### THE TIER-1 PATTERN MAP

| Pattern | Appears in |
|---|---|
| **Available vs meaningful metric** | G13 · G14 · G29 · G25 |
| **Cheap-wide → expensive-narrow cascade** | G13 · Q6 · G25 · Q7 |
| **The metric with no natural owner** | G13 (satisfaction) · G31 (stability) · G29 (allow-through) |
| **A lookup, not a computation** | G21 (features) · G29 (graph) · G31 (aggregates) · Q7 (prefix cache) |
| **You only observe what you did** | G13 (logs) · G29 (blocks) · G14 (shown cards) |
| **Capacity/asymmetry sets the threshold** | G25 (review capacity) · G29 (cost matrix) · Q10 (blast radius) |
