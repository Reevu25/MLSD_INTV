# R6 · A Real-Time Recommendation System — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md)–[R5](R5_product_or_video_recsys_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **Expect these probes:** (a) assumptions · (b) architecture and data flow: offline / nearline / online · (c) features and the feature store · (d) baseline → advanced model, and why candidate generation + ranking · (e) cold start: new users · (f) cold start: new items · (g) latency vs accuracy: serving path, caching, TTLs · (h) metrics, A/B tests, production monitoring · (i) scalability, reliability, fallbacks.
>
> **How this differs from its neighbours:** [R5](R5_product_or_video_recsys_intv.md) is the platform view (products vs videos) and [R4](R4_watch_next_losses_optimizers_intv.md) derives the losses. This sheet is about **time**: how fresh each part of the system is, what that freshness costs, and what happens inside a 100 ms budget when something is slow.
>
> **Related sheets:** [G13 YouTube recommendation](G13_youtube_recommendation_intv.md) · [G22 delayed conversion](G22_delayed_conversion_intv.md) (label delay in streaming training) · [Q7 inference serving](Q7_inference_serving_intv.md) (serving fleets, batching) · [Q8 data pipeline](Q8_data_pipeline_intv.md) (streams, backfills) · [Q26 continuous learning](Q26_continuous_learning_intv.md) (incremental updates, gates, rollback).

---

## 📋 THE QUESTION (verbatim)

> You are asked to design a real-time recommendation system for a large-scale consumer product (for example, recommending items or content to users in a mobile app).
>
> **The system should:**
> - Serve personalized recommendations with low latency (end-to-end P95 latency target: e.g., < 100 ms from request to response at the service layer).
> - Support millions of daily active users and tens of thousands of candidate items that change over time.
> - Continuously incorporate new user interactions (clicks, views, purchases, etc.) to keep recommendations fresh.
>
> **Address the following aspects in your design:**
>
> **1. High-level Architecture & Data Flow**
> - Describe the overall pipeline from data generation to model training to online serving.
> - Explicitly separate offline, nearline, and online components where applicable.
>
> **2. Feature Engineering & Feature Store**
> - What kinds of user, item, and context features would you use?
> - How would you design a feature store to support both offline training and online inference with consistent features?
>
> **3. Modeling Approach**
> - Propose a baseline model (e.g., simple heuristics or a shallow model) and then a more advanced model (e.g., deep learning–based ranking).
> - Explain how you would structure the system as candidate generation + ranking (or another decomposition) and why.
>
> **4. Cold Start Problem**
> - How would you handle new users with little or no history?
> - How would you handle new items with no interaction data?
> - Discuss multiple strategies (e.g., content-based features, popularity-based recommendations, exploration).
>
> **5. Latency vs. Accuracy Trade-offs**
> - Given a strict latency budget, how would you design the serving path (caching, pre-computation, approximate search, etc.)?
> - Discuss concrete strategies to trade off model complexity/accuracy against serving latency and system cost.
> - Explain where you would use caching (e.g., user-level, item-level, or result-level caches) and what consistency/TTL strategies you might choose.
>
> **6. Monitoring, Evaluation, and Iteration**
> - What online metrics and offline metrics would you track to evaluate the recommender system?
> - How would you set up A/B testing or other online experiments?
> - Describe what you would monitor in production (e.g., model performance drift, feature distribution shift, latency, error rates) and how you would respond.
>
> **7. Scalability, Reliability, and Other Practical Considerations**
> - Discuss storage and computation choices (e.g., streaming system, message queues, scalable storage for logs, feature store, and models).
> - How would you design for fault tolerance, graceful degradation, and fallback behavior (e.g., if the model server is down or too slow)?
>
> *Clearly explain your assumptions and walk through your design step by step.*

### The question's seven aspects, and where each is earned

| Aspect | Where |
|---|---|
| 1 · Architecture & data flow, offline / nearline / online | §4 (the three-tier diagram) |
| 2 · Features & feature store | §5.2 (features by freshness), §5.3 (the store: one definition, point-in-time joins, log-at-serve) |
| 3 · Baseline → advanced; why candidate generation + ranking | §5.4 (with the arithmetic for *why two stages*, even at 50k items) |
| 4 · Cold start, users and items, several strategies | §5.5, FU 4 |
| 5 · Latency vs accuracy; caching and TTLs | §3 (budget), §6 (serving path, the lever table, the cache table) |
| 6 · Metrics, A/B tests, monitoring and response | §2, §7 (with a runbook: signal → threshold → response) |
| 7 · Storage, compute, fault tolerance, fallbacks | §9 (the stack, the degradation ladder) |

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | *"What kind of items, and how fast do they churn?"* and *"Which interactions must show up in the very next response?"* decide the freshness tiers (§1) |
| **Explainability** is easy points | **Features are logged at serving time**, so any recommendation can be **replayed exactly** and its reason ("Because you just viewed X") is grounded (§8) |
| **Security** beyond the gateway | **Real-time is fast to poison**: a click farm can push the trending source in minutes, and an ungated continuous-training loop pushes poisoned weights automatically (§8, FU 10) |
| **Troubleshoot before being asked** | Raise **stream lag**, **cache vs hide consistency**, the **model server going down**, and **label delay in streaming training** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Three freshness problems, three costs** · 50k items fit in 13 MB · a deadline and a fallback everywhere | A1a, A3a |
| **1–5** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · **freshness SLAs** as NFRs | A4a, A1b |
| **5–8** | **PRODUCT + METRICS** | (h) | A feed that reacts within the session · online, offline and system metrics · **what is real-time worth?** | I1c, F1b, I2b |
| **8–11** | **CAPACITY + BUDGETS** | (g) | 10⁴ QPS · 10⁵ events/s · exact retrieval in ~2 ms · the 100 ms budget | A1b, I1a, I1d |
| **11–20** | **ARCHITECTURE + DATA FLOW** | **(b)** | Offline / nearline / online · alternatives · four problems raised yourself | A2a, A5a, A1c, F2b |
| **20–30** | **FEATURES, STORE, MODELS, COLD START** | **(c) (d) (e) (f)** | Features by freshness · **point-in-time correctness** · baseline → advanced · cold start | A2b, A2c, A2d, F1a, F1c, F2c |
| **30–37** | **SERVING: LATENCY vs ACCURACY** | **(g)** | Serving path · **the lever table** · **the cache table, with TTLs** | F3a, F3b, A3b, I1b |
| **37–43** | **METRICS, EXPERIMENTS, MONITORING** | **(h)** | Offline → A/B · **arms contaminate each other's training data** · the runbook | I2a, I2b, F1b, F1c |
| **43–47** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, as a checklist | I1a–I1j |
| **47–52** | **SCALABILITY, RELIABILITY, FAILURES** | **(i)** | Storage and compute · **the degradation ladder** · on-device re-ranking · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"'Real-time' is three different freshness problems with very different costs: fresh features, fresh items, and fresh model weights. Most of the value comes from the first. A user's last few clicks, streamed into features within seconds, change the next recommendation more than retraining every minute would. So I'd make features and the item catalogue real-time, update weights incrementally on a schedule behind a quality gate, and keep true per-event learning for the few things that need it: new items' exploration statistics and the hide list.**
>
> **Second, the numbers change the design. The prompt's tens of thousands of candidate items, even though they change over time, is small: all their embeddings fit in about 13 MB, and the changes are just a few thousand adds and removes a day. So every serving machine holds the whole catalogue in memory, and retrieval is an exact dot product over all of it in a couple of milliseconds, with no approximate search and no recall loss. The 100 ms budget goes on features and the ranker.**
>
> **Third, at a 100 ms P95, the design is as much about what happens when something is slow as when it's fast. Every dependency has a deadline and a fallback, so the user always gets a list, and the fallback is the baseline model I'd build first anyway."**

**Then say the plan (`A3a`):** *"Assumptions, what the user sees and how we measure it, capacity and the latency budget, the architecture in three tiers, features and the feature store, baseline and advanced models, cold start, the serving path with caching, evaluation and monitoring, then storage, reliability and fallbacks."*

---

## 1 · REQUIREMENTS *(min 1–5)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
0. **What's the product, and which surface?** A short-video feed, a shopping home screen, a news app, a single "for you" card? *(Decides the labels (watch vs purchase vs read), how many items each response shows, how much exploration is tolerable, and whether a bad item costs a scroll or a whole slot. Assume a mobile-app feed showing 20 items per request.)*
1. **What are the items, and how fast do they churn?** Short videos, products, news, deals? *(Decides how fast new items must become recommendable. Assume ~10% of the catalogue turns over per day.)*
1b. **Are the "tens of thousands of candidate items" the WHOLE catalogue, or what an upstream retriever hands us per request?** *(⭐ This one changes the architecture. If it's the whole catalogue, all embeddings fit in ~13 MB, so every server holds them in memory and retrieval is an exact dot product, with no ANN index. If it's a per-request slice of a far larger catalogue, there's an upstream retrieval system to design or assume. Assume the whole catalogue, changing by a few thousand items a day.)*
2. **Which interactions must be reflected in the very next response?** *(A hide or a purchase: always. A click: within seconds. A purchase-driven taste shift: within the hour. This sets the freshness tiers.)*
3. **What's the objective?** Engagement, conversion, retention? *(Assume a blend, with retention as the gate.)*
4. **Is the 100 ms at the service layer only?** *(Assume yes: from request arrival to response, excluding the mobile network.)*
5. **Signed in? Any regulated data or regions?** *(Decides retention limits on raw events and deletion propagation.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Users | **~10⁷ DAU**, ~20 feed requests and ~200 events per user per day |
| Items | **~5×10⁴ active**; ~5×10³ new and ~5×10³ expiring per day |
| Surface | A mobile-app feed: **20 items per request**, refreshed on scroll and pull |
| Latency | **P95 < 100 ms** at the service layer; P99 tracked |
| Objective | Engagement and conversion **inside** retention and hide-rate guardrails |
| Training | **Daily** full retrain (the source of truth) + **hourly** incremental + **seconds-level** features and bandit statistics |

**FR:** personalised feed · reacts to in-session behaviour · new items recommendable quickly · hides and purchases honoured on the next response · cold-start handling for users and items · experiments · a fallback for every dependency.

**NFR / freshness SLAs (the real spec of "real-time"):**

| What | SLA |
|---|---|
| **Service latency** | **P95 < 100 ms**, P99 < 200 ms |
| **Event → online feature** | **≤ 10 s** P95: *freshness*, from the user's tap to the updated feature being readable (client → event bus → stream processor → online store). Not the same as the store's **read** latency, ~10 ms, inside the request. The gap is covered by the client sending its last few events with each request |
| **Hide / purchase → effect** | **The next response** (read-your-writes) |
| **New item → recommendable** | **≤ 5 min** after it passes policy checks |
| **Item removal → gone everywhere** | **≤ 1 min**, including caches |
| Model weights | Incremental **hourly**, gated; full retrain **daily** |
| Availability | 99.95%; a degraded list, never an error |

**Scope / NON-GOALS:**
- ❌ **Per-event online learning of the whole model.** Position: cost and risk outweigh the gain over fresh features (FU 1).
- ❌ Search, notifications and ads (they reuse the feature store, not the feed).
- ❌ On-device models in v1 (an extension, §9).

---

## 2 · PRODUCT + METRICS *(min 5–8)* — probe (h) `I1c` `F1b` `I2b`

**What the user sees, and why real-time matters to them:**

```
   09:02 · OPENS THE APP                      09:03 · AFTER 2 TAPS + PULL TO REFRESH
 ┌─────────────────────────────┐            ┌─────────────────────────────┐
 │  ↓ pull to refresh          │            │  ↓ refreshed just now       │
 │ ┌─────────────────────────┐ │            │ ┌─────────────────────────┐ │
 │ │ ▶ Easy weeknight pasta  │⋮│            │ │ ▶ Trail running: hills  │⋮│
 │ │ Because you watched      │ │            │ │ Because you just watched │ │
 │ │ "One-pot meals"          │ │            │ │ "Trail running basics"   │ │ ← SESSION
 │ └─────────────────────────┘ │            │ └─────────────────────────┘ │   signal,
 │ ┌─────────────────────────┐ │            │ ┌─────────────────────────┐ │   seconds old
 │ │ ▶ Trail running basics  │⋮│ ← tap 1    │ │ ▶ Best trails near you  │⋮│
 │ │ Trending in sports       │ │            │ │ Trending in trail running│ │
 │ └─────────────────────────┘ │            │ └─────────────────────────┘ │
 │ ┌─────────────────────────┐ │            │ ┌─────────────────────────┐ │
 │ │ ▶ Downhill technique    │⋮│ ← tap 2    │ │ ▶ Easy weeknight pasta  │⋮│ ← long-term
 │ │ New for you              │ │            │ │ Because you watched      │ │   interest
 │ └─────────────────────────┘ │            │ │ "One-pot meals"          │ │   still there
 │ ┌─────────────────────────┐ │            │ └─────────────────────────┘ │
 │ │ ▶ Road marathon pacing  │⋮│            │ ┌─────────────────────────┐ │
 │ │ Popular this week        │ │            │ │ ▶ Trail shoe review     │⋮│
 │ └─────────────────────────┘ │            │ └─────────────────────────┘ │
 └─────────────────────────────┘            └─────────────────────────────┘
   ⋮ menu on every card: Not interested · Don't show this channel
   ⚠ the two tapped videos are NOT shown again: "seen" filter, impression memory
```

```
09:02  opens the app → feed: their long-term interests + what's trending now
09:03  taps two trail-running videos in the feed and watches them
       (each tap is a click EVENT, streamed into session features)
09:03  pulls to refresh (drags the feed down to reload it: a NEW request)
       → more trail-running videos move up                            ← SECONDS later
09:04  hides a "road marathon pacing" video → gone, and so are its
       near-duplicates                                                ← NEXT response
09:05  scrolls to page 3 → no repeats of anything shown on pages 1–2    ← impression memory
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **D7 retention** + engaged time (or conversions) per DAU | Value that lasted |
| Online, engagement | CTR · dwell or watch time · conversion · items consumed per session | Proximate, read **with** guardrails |
| Online, negative | Hide rate · fast bounce · "seen it already" repeats | Ship blockers |
| Online, ecosystem | Catalogue coverage · **new-item time-to-first-impression** · concentration | Is the loop narrowing? |
| Offline | Recall@k (retrieval) · NDCG@k · AUC, normalised entropy, **calibration** (ranker) | **Screens, never decides** |
| **System** | P95 / P99 per stage · error rate · **fallback rate** · **event-to-feature lag** · cost per 1,000 requests | Real-time is an SLO, not a feature |

> ⭐ **The L6 move:** *"Early on I'd run one experiment that switches the real-time features off. That tells us what the streaming infrastructure is actually worth, and whether a feature needs ten-second freshness or would be fine hourly. Every real-time feature should earn its pipeline."*

---

## 3 · CAPACITY + BUDGETS *(min 8–11)* — probe (g) `A1b` `I1a` `I1d`

```
REQUESTS   10⁷ DAU × 20 = 2×10⁸/day ≈ 2.3×10³ QPS avg, ~10⁴ at peak
EVENTS     10⁷ × 200    = 2×10⁹/day ≈ 2.3×10⁴/s avg,  ~10⁵/s at peak
           → partitioned by user ID, so each user's events stay in order

⭐ THE CATALOGUE IS SMALL:
  item embeddings: 5×10⁴ × 64 dims × 4 B ≈ 12.8 MB  (+ features ≈ 50 MB)
  → EVERY serving pod holds the whole catalogue IN MEMORY
  EXACT retrieval: 5×10⁴ × 64 = 3.2×10⁶ multiply-adds ≈ 1–2 ms on one core
  → no ANN index, no recall loss, no index rebuilds; refresh = swap a 13 MB array

  TYPICAL CPU NUMBERS (one modern server core, order of magnitude):
    compute     SIMD fused multiply-adds (FMAs): AVX2 ≈ 16 per cycle × 3 GHz
                ≈ 5×10¹⁰ FMA/s peak, ~10¹⁰ FMA/s realistic in a simple loop
                (1 FMA = 2 FLOPs, so peak ≈ 10¹¹ FLOP/s = 100 GFLOP/s per core)
    memory      one core streaming from DRAM ≈ 10–20 GB/s
                (a whole socket ≈ 100–300 GB/s) · from L3 cache: several × faster
    L3 cache    tens of MB per socket, so the 13 MB table can stay IN CACHE
  WORKING IT OUT:
    compute   3.2×10⁶ ÷ ~10¹⁰/s                 ≈ 0.3 ms
    memory    12.8 MB ÷ ~15 GB/s (from DRAM)    ≈ 0.9 ms   ← the bottleneck
              (in L3 cache: ~0.2 ms · as int8, 3.2 MB: ~4× less)
    → MEMORY-BOUND: reading the vectors costs more than the arithmetic,
      so "1–2 ms" is a safe planning number, and int8 is the first lever
  COMPARE R1: 10⁶ vectors × 128-d × 4 B = 512 MB per query → ~35 ms at
    15 GB/s, which is why R1 needs an approximate (ANN) index and R6 doesn't
    · LATENCY: R1's retrieval slice is ~15 ms of its post-turn budget, so a
      35 ms scan breaks the SLA on every request
    · THROUGHPUT: at R1's ~35k QPS peak, 512 MB per query is ~18 TB/s of reads,
      ~90 CPU sockets doing nothing but scanning. ANN reads a small fraction
      of the vectors (~1–3 ms) at ~95–99% of the exact top-200's recall

WHY TWO STAGES ANYWAY:
  a DNN ranker at ~1 MFLOP per candidate × 5×10⁴ = 5×10¹⁰ FLOP per request
  × 10⁴ QPS = 5×10¹⁴ FLOP/s: far too expensive
  on ~500 candidates: 5×10⁶ scorings/s ≈ 5×10¹² FLOP/s → a few GPUs or ~100 CPU cores

  WHERE "~1 MFLOP PER CANDIDATE" COMES FROM:
    a dense layer does one multiply-add (2 FLOPs) per WEIGHT per example, so
    forward FLOPs ≈ 2 × the DENSE weights the example passes through
    e.g. MLP 512 → 512 → 256 → 1:  512·512 + 512·256 + 256 ≈ 3.9×10⁵ weights
         → ≈ 0.8 MFLOP, plus feature crosses and attention over the history ≈ 1
    ⚠ NOT "10⁶ parameters": EMBEDDING lookups cost almost no FLOPs (they're
      memory reads), so a recsys model with 10⁹ parameters, mostly embedding
      tables, can still be ~1 MFLOP per candidate
    production rankers are often 10–100× bigger (10–100 MFLOP), which is why
    the cascade (a cheap pre-ranker first) matters even more there

ONLINE STORE   users: 10⁷ × ~2 KB profile + last-50 events ≈ 30 GB, replicated
LOGS           2×10⁸ requests × 20 impressions × ~1 KB (with served features)
               ≈ 4 TB/day raw → ~0.5 TB/day compressed columnar

LATENCY BUDGET (P95 < 100 ms; ~65 ms planned, the rest is tail headroom)
  gateway: auth, rate limit, parse                          5 ms
  user + session features (one batched KV read)            10 ms   ┐ in
  item features + embeddings (IN-PROCESS)                   1 ms   ┘ parallel
  candidate sources in parallel (exact dot product,
    co-occurrence from last items, trending, fresh pool)    5 ms
  filters: hidden, seen, expired, policy                    2 ms
  ranker on ~500 (batched; GPU or CPU)                     25 ms
  re-rank: diversity, exploration slots, dedupe             5 ms
  serialise + respond                                      10 ms
  ─────────────────────────────────────────────────────────────
                                                         ≈ 63 ms
```

---

## 4 · ARCHITECTURE + DATA FLOW *(min 11–20)* — probe **(b)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| What "real-time" updates | Features only · features + incremental weights · **per-event online learning** of everything | **Features and catalogue in seconds; weights hourly behind a gate; per-event only for bandit stats and the hide list.** Fresh features carry most of the value at a fraction of the risk |
| Retrieval | ANN index · **exact dot product over the in-memory catalogue** | At 5×10⁴ items, exact is ~2 ms: no recall loss, no index to rebuild |
| Pipeline shape | Lambda (separate batch and stream code) · **one feature definition, compiled to both** | Two codebases for one feature is how training–serving skew starts |
| Training data | Recompute features offline · **log features at serving + point-in-time joins for new ones** | Logged features are exactly what the model saw |
| Ranker | GBDT · **DNN with feature crosses + sequence attention**, GBDT kept as the fallback | §5.4 |
| Precompute vs on-request | Everything per request · **event-triggered precompute of candidates; rank per request** | Moves work off the critical path; staleness bounded by the event trigger |

### The system — *draw this in 90 seconds* `A2a`

```
══ ONLINE · per request (< 100 ms) ════════════════════════════════════════════
 app ── request (user, context, + the CLIENT's last few events) ──►
┌─ 1. GATEWAY ───────────────────────────────────────────────────
│ TASK: auth, rate limit, deadline set to 90 ms and passed down
│ OUTPUT: a request with a deadline
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. FEATURE ASSEMBLY ──────────────────────────────────────────
│ user + session features: ONE batched read from the online store
│ item features: IN-PROCESS (the whole catalogue lives in memory)
│ context: from the request · on-demand crosses computed here
│ OUTPUT: features, each with a freshness timestamp
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. CANDIDATES, IN PARALLEL ───────────────────────────────────
│ exact two-tower over all 5×10⁴ · co-occurrence from the last 5
│ items · precomputed candidates (event-triggered) · trending ·
│ FRESH-ITEM pool (bandit) · FILTER: hidden, seen, expired, policy
│   EXACT TWO-TOWER, online: ONLY the USER tower runs → q (64-d, normalised)
│   the ITEM tower does NOT run here: its vectors were computed ahead of
│   time (at upload, refreshed every minute) into E, a 5×10⁴ × 64 matrix
│   held in memory · scores = E · q (one matrix-vector product: 5×10⁴ dot
│   products; vectors normalised, so each dot product IS the cosine) →
│   keep the top 500 with a partial sort (~1–2 ms)
│ OUTPUT: ~500 candidates, source-tagged
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. RANKER ──────── on deadline miss → GBDT fallback (§9) ─────
│ multi-task DNN: click · dwell/conversion · hide
│ OUTPUT: calibrated scores → a value score
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. RE-RANK → RESPONSE ────────────────────────────────────────
│ MMR diversity · exploration slots · dedupe · LOG served features
│ OUTPUT: 20 items + reasons; served features → the impression log
└────────────────────────────────────────────────────────────────

══ NEARLINE (seconds–minutes) ═════════════════════════════════════════════════
 CLIENT EVENTS ─► EVENT BUS (Pub/Sub or Kafka, partitioned by user)
               ─► STREAM PROCESSOR (Dataflow or Flink)
   1. CLEAN: dedupe by event ID · drop bots · order by EVENT time
   2. COMPUTE the real-time features, over 5 min / 1 h / 24 h windows:
        per USER: last-50 item sequence · clicks per category (5 min, 1 h)
                  · session length · HIDE LIST · times each item was shown
        per ITEM: click-through rate over the last hour (smoothed) · trending
   3. WRITE every update TWICE:
        → ONLINE store:  OVERWRITE the latest value (what serving reads)
        → OFFLINE store: APPEND a timestamped row (history for training)
   4. SIDE OUTPUTS, not feature-store writes:
        → candidate cache: recompute this user's candidates on strong events
          (STRONG = changes what they want: a click or deep watch, a hide, a
           purchase or add-to-cart, a search, a follow · WEAK, no recompute:
           an impression, a scroll-past, a two-second glance)
        → bandit state: update fresh items' Beta posteriors
        → the trending list (trust-weighted counts)
 NEW ITEM ─► policy check ─► content tower ─►
        → its EMBEDDING: broadcast to every pod's IN-MEMORY catalogue (13 MB)
        → its starting FEATURES (priors from similar items): ONLINE + OFFLINE
 STREAMING TRAINER: impressions joined to labels after a wait window → hourly
   incremental model → GATE → canary → MODEL REGISTRY (§5.6)

══ OFFLINE (hours–daily) ══════════════════════════════════════════════════════
 INPUTS: impression logs WITH the features served (the training rows)
         + delayed labels (conversions, returns)
   1. BATCH FEATURES, daily, from the warehouse: long-term user embedding ·
      category affinities · tenure
        → ONLINE store:  overwrite, so serving has today's long-term features
        → OFFLINE store: append, dated
   2. TRAINING DATA: logged served features + labels; point-in-time joins
      against the OFFLINE store only for features that are NEW
   3. FULL RETRAIN daily (the source of truth) → offline eval → MODEL REGISTRY
   4. DAILY SKEW CHECK: recompute a sample of logged rows from the OFFLINE store
      and compare with what serving actually logged
```

**What goes into which store:**

| | **ONLINE store** | **OFFLINE store** |
|---|---|---|
| **Holds** | The **latest** value per user and per item, overwritten | **Every** value, timestamped, append-only |
| **Real-time features** (from the stream, seconds) | User: last-50 sequence, clicks per category, session length, hide list, times shown · Item: last-hour CTR, trending | The full history of each of those updates |
| **Batch features** (daily) | Long-term user embedding, category affinities, tenure | The same, one dated row per day |
| **New items' starting features** | Priors from similar items | The same, timestamped |
| **Read by** | The ranker, every request (~5–10 ms) | Training-data builds (point-in-time joins), the daily skew check |
| **On Google Cloud** | Vertex AI Feature Store online serving (Bigtable-based) | BigQuery |

**Not in the feature store at all:** item **embeddings** (an in-memory catalogue on every pod) · the **candidate cache** (its own KV store) · **bandit posteriors** (the bandit's state) · **model weights** (the model registry) · **impression logs** with served features (log tables in the warehouse, joined with the feature store when training data is built).

**The one trick worth naming:** the client sends its **last few events with the request**. That gives read-your-writes for a hide or a tap without waiting for the stream, and the stream catches up a few seconds later.

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Stream lag:** *"If the stream processor falls behind, real-time features go stale silently. Every feature carries a timestamp; past its TTL it's replaced by a default with a 'missing' flag, and the model was trained with feature dropout, so missing is in-distribution (FU 6)."*
- **Caches vs hides:** *"A result cache must never show an item the user just hid, so the hide filter runs **after** the cache, on every response (FU 5)."*
- **The model server goes down:** *"Every stage has a deadline; past it, the next tier down answers: GBDT, cached results, segment-popular (FU 3)."*
- **Label delay in streaming training** `F1c`: *"An impression with no click yet isn't a negative yet. The streaming trainer waits a window before labelling, and conversions, which arrive later, are trained from the daily job (G22)."*
- **The latency squeeze** `F2b` `A1c`: *"Say the ranker can't score 500 candidates inside its 25 ms. A smaller ranker for everyone would lower quality on every request, so I wouldn't do that. Instead:"*
  1. **Send the ranker only the best ~200.** A **pre-ranker**, a small model trained to imitate the ranker's scores, scores all 500 cheaply (~2 ms) and passes on its top 200, so the ranker does 60% less work. ⚠ Pick the 200 with the pre-ranker, **not** by retrieval score: retrieval is too crude to know which 200 the ranker would like. Check offline that the ranker's own top 20 almost always survives the cut.
  2. **Move candidate generation off the request path.** Recompute each user's candidates in the background whenever a strong event arrives (a click, hide, purchase or search), and keep them in the candidate cache. The request then just reads them, and the milliseconds saved go to the ranker.
  - *"The heavy ranker stays the same size; it just scores only the candidates worth its time."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Fresh features, gated weights** | "Most of real-time's value is in features; fresh weights are expensive and risky" | Taste shifts that need new weights arrive within the hour, not the second |
| **2** | **Exact retrieval, catalogue in memory** | "50k items is 13 MB: exact is cheaper than approximate" | Doesn't survive a 100× catalogue; at 5×10⁶ items, switch to ANN |
| **3** | **Log features at serving** | "Train on exactly what the model saw" | Storage (~0.5 TB/day), and new features still need backfills |
| **4** | **A deadline and fallback per stage** | "A slow answer is a wrong answer at 100 ms" | Some requests get the GBDT, not the DNN: measured as fallback rate |

---

## 5 · FEATURES, FEATURE STORE, MODELS, COLD START *(min 20–30)* — probes **(c) (d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 Data sources `A2c`

| Source | Tier | What it gives | Watch out for |
|---|---|---|---|
| **Client events** (impression, click, view, dwell, purchase, hide) | Online → stream | Labels and real-time features | Duplicates, out-of-order and late events, bots |
| **Impression log with served features** | Offline | Exact training rows | Size; only shown items have labels |
| **Item catalogue + content** (text, images, metadata, creator) | Change events | Content features; the only signal a new item has | Seller- or creator-written: gaming |
| **Delayed outcomes** (purchases, returns, retention) | Offline | The real labels | Hours to weeks late |
| **Request context** | Online | Time, device, network, coarse location, surface, referrer | The referrer is strong, short-lived intent |
| **Exploration slice with propensities** | Offline | Unbiased labels and evaluation | Small and precious |

### 5.2 Features, by entity and by freshness (c)

| | **Batch (daily)** | **Streaming (seconds)** | **On-demand (per request)** |
|---|---|---|---|
| **User** | Long-term embedding · category affinities (decayed counts) · tenure · locale | **Last-50 item sequence** · clicks per category in the last 5 min / 1 h · session length · **hide list** · time since last visit | — |

> **Where the long-term user embedding comes from** (three options, simplest first): **(1)** a decayed average of the embeddings of items the user engaged with, with no model at all; **(2)** the **retrieval model's own user tower**, run in a daily batch over months of history (the usual choice: no extra model to own); **(3)** a **dedicated user model**, e.g. a sequence transformer trained to predict the user's future engagement, shared across products (Pinterest's PinnerFormer is a published example). In every case it's **precomputed daily**, because encoding hundreds of past events per request is expensive and long-term taste moves slowly; the session features cover what changed today. ⚠ It's a dependency: retrain the user model and the ranker's input shifts, so version them together.
| **Item** | Content embedding (text, image) · category · creator quality · age | **CTR in the last hour, Bayesian-smoothed** · trending velocity · availability | — |
| **Context** | — | — | Time of day · day of week · device · network · surface · referrer |
| **User × item** | Long-term affinity to the item's category | **Times shown to this user in 24 h** (impression fatigue) | **Similarity of the candidate to the last-clicked items** · candidate × context crosses |

```
SMOOTHED REAL-TIME CTR (so a 1-impression item isn't "100% CTR"):
  CTR_smooth = (clicks + α) / (impressions + α + β)
  α/(α+β) = the category's prior CTR · α+β = the prior's strength (pseudo-impressions)
```

### 5.3 The feature store: consistent offline and online (c) `A2d`

```
ONE DEFINITION PER FEATURE, in a registry:
  name · entity (user / item / pair) · source · transformation · freshness SLA ·
  TTL · DEFAULT value · owner
  → the SAME transformation code is compiled to the batch job and the stream job
    (one definition, two runtimes, not two implementations)

TWO STORES:
  ONLINE   low-latency key-value, keyed by entity · latest value + timestamp
           P99 read < 10 ms · replicated across zones
  OFFLINE  append-only, TIME-STAMPED log of every feature update (columnar,
           partitioned by hour) → the history needed for training

⭐ POINT-IN-TIME CORRECTNESS: the rule that keeps training honest
  for a training row = impression at time t:
    feature value = the latest update with timestamp ≤ t − lag
    where lag = the pipeline's real event-to-online-store delay
  · without "≤ t": LEAKAGE. The item's CTR would include THIS click, and the
    model learns from the future → great offline, worse online
  · without "− lag": training sees fresher features than serving ever had

TWO WAYS TO GET TRAINING FEATURES:
  1. LOG-AT-SERVE (default): the exact feature vector each impression was
     scored with is logged → zero skew, and every recommendation is replayable
  2. POINT-IN-TIME BACKFILL (for NEW features): join the offline log as of t − lag
     → then log-at-serve takes over once the feature ships

DAILY SKEW CHECK: recompute a sample of logged rows offline; any feature whose
  served and recomputed values disagree beyond tolerance blocks the next push
MISSING / STALE: past its TTL a feature returns its DEFAULT + a "missing" flag;
  the model is trained with ~5% feature dropout, so this is in-distribution
```

**Example feature stores** (name one, then say what you need from it: one definition, two stores, point-in-time joins, streaming writes):

| Feature store | Kind | Online store (serving) | Offline store (training) |
|---|---|---|---|
| **Feast** | Open source | Pluggable: Redis, DynamoDB, Bigtable… | Pluggable: BigQuery, Snowflake, Parquet files… |
| **Vertex AI Feature Store** | Managed (Google Cloud) | Bigtable-based or optimised online serving | BigQuery |
| **SageMaker Feature Store** | Managed (AWS) | A low-latency online store | S3, queryable through Athena / Glue |
| **Databricks feature engineering** | Managed (Databricks) | Online tables | Delta tables |
| **Tecton** | Managed (commercial) | Redis or DynamoDB | A data warehouse or lake |
| **Hopsworks** | Open source + managed | RonDB | A data lake (Hudi tables) |
| **Chronon** (Airbnb) · **Feathr** (LinkedIn) | Open-sourced in-house platforms | KV stores | Data lake / warehouse |
| **Michelangelo Palette** (Uber) | In-house, published | Cassandra-based | Hive / data lake |

> ⚠ **BigQuery is the OFFLINE side, not the online one.** In Vertex AI Feature Store, features are registered over **BigQuery** tables (the offline store and source of truth) and **synced** into an online store (Bigtable-based or optimised serving) that answers in milliseconds. A BigQuery query takes seconds, far too slow inside a 100 ms request. Seconds-fresh **session features** come from the stream (Dataflow) written straight to a low-latency store, and are also appended to BigQuery for training history. Point-in-time lookups for training run in BigQuery SQL (e.g. its `ML.FEATURES_AT_TIME` functions, or window functions).

> **Say:** *"Something like Feast or Vertex AI Feature Store: one feature definition, a Bigtable- or Redis-backed online store for serving, a warehouse offline store for training with point-in-time joins, and streaming writes for the session features."*

### 5.4 Baseline first, then the advanced model (d) `A2b` `F1a`

| Stage | **Baseline** (weeks 1–4) | **Advanced** |
|---|---|---|
| **Candidates** | Time-decayed popularity per segment · item-to-item co-occurrence from the last 5 items · the fresh-item pool | + **two-tower**, exact over all items (sampled softmax, logQ-corrected; R4 §5.2) · + a **sequence model** (transformer over the last 50 events, next-item) |
| **Ranker** | **GBDT** (or logistic regression) on ~100 hand-built features: smoothed CTR, user–category affinity, recency, fatigue, context | **DNN with explicit feature crosses** (DCN-v2 style) + **target attention** over the last 50 events (which recent items matter for *this* candidate) · **multi-task**: click, dwell/conversion, hide · shallow position tower |
| **Re-rank** | Rules: dedupe, one per creator per 5, freshness boost | MMR diversity + bandit exploration slots |
| **Freshness** | Streaming counts | + hourly incremental weights, minute-level fresh-item embeddings |

**Inside the advanced ranker: target attention and explicit crosses.**

```
TARGET ATTENTION (DIN-style): the user's history is summarised DIFFERENTLY for
each candidate, by attending to the past items that are relevant to IT
  history h₁…h₅₀ (last 50 items) · candidate v
  a_i = softmax_i( score(h_i, v) )   score = a small MLP on [h_i, v, h_i ⊙ v]
                                     (or simply h_iᵀ·W·v)
  u_v = Σ_i a_i · h_i                → "this user, as seen from candidate v"

  e.g. history: hiking boots · tent · pasta recipe · running shoes
    candidate "trail running shoes"  → weights ≈ 0.45 · 0.10 · 0.02 · 0.43
    candidate "one-pot pasta"        → weights ≈ 0.03 · 0.02 · 0.90 · 0.05
  → one averaged user vector would blur hiking and cooking together;
    attention lets the relevant slice of history speak for each candidate
  cost: 50 history items × 500 candidates = 25k small scores per request: fine

  [h_i, v, h_i ⊙ v] is ONE vector: three 64-d vectors CONCATENATED → 192-d
    h_i ⊙ v = the ELEMENTWISE product (multiply position by position): it hands
    the MLP "where the two items agree", which it struggles to learn from h_i
    and v alone. DIN also adds h_i − v → [h_i, v, h_i − v, h_i ⊙ v], 256-d
  softmax_i = a softmax OVER i, the 50 history items, for ONE candidate v:
    a_i = exp(s_i) / Σ_j exp(s_j), s_i = score(h_i, v) → positive, sums to 1
    e.g. scores 2.0 · 0.5 · −1.5 · 2.0 → exp 7.39 · 1.65 · 0.22 · 7.39 (sum 16.65)
         → weights 0.44 · 0.10 · 0.01 · 0.44
    each candidate gets its OWN softmax over the same history

EXPLICIT FEATURE CROSSES (DCN-v2): layers that multiply features together
  x_{l+1} = x₀ ⊙ ( W_l · x_l + b_l ) + x_l
    x₀    the ORIGINAL input: all features concatenated (user embedding,
          candidate embedding, context, dense features); fed into layer 1
          AND reused in every layer
    x_l   the OUTPUT of cross layer l, which is the input to layer l + 1
          (x₁ = layer 1's output, x₂ = layer 2's, …)
    W_l, b_l  layer l's learned weights and bias · ⊙ elementwise product
    + x_l     a residual: keep what layer l already had
  each layer MULTIPLIES by x₀ again, so it raises the interaction order by one:
    layer 1 → pairs ("affinity to running × item is a running item"),
    layer 2 → three-way products, and so on
  e.g. x₀ = [0.8 affinity to running · 1.0 category = running · 0.3 evening]
       if row 1 of W₀ has learned to pick the category: (W₀·x₀)₁ = 1.0
       x₁[1] = 0.8 × 1.0 + 0.8 = 1.6   ← contains affinity × category
  → learns interactions like "this user's affinity × this item's category"
    directly, instead of hoping a plain MLP discovers them; cross layers
    usually run in parallel with (or before) an MLP, and both feed the heads

THEN: [u_v, candidate features, context, crosses] → MLP → the heads
  (click, dwell/conversion, hide)
```

**Exploration belongs in retrieval too, not only in the final slots.** A re-ranker can only explore among candidates retrieval returned; an item that never reaches the candidate set can never be explored. So exploration runs at **both** levels:
- **Retrieval:** the **fresh-item pool** is its own candidate source with a quota, and a share of each request's candidates can be sampled from uncertain or tail items, or retrieved with noise added to the query vector (Thompson-style).
- **Re-rank:** the bandit's exploration slots decide which of those candidates actually get shown.
- ⚠ **Propensities get harder:** an item's chance of being shown is now P(reached the candidate set) × P(picked for a slot), and both must be logged for off-policy evaluation to stay valid.

**Why build the baseline first:** it forces the logging, the feature store and the A/B pipeline into existence; it sets the bar the DNN must beat online; and it **stays in production as the fallback** when the DNN misses its deadline.

**Why candidate generation + ranking, even at 50k items** (say the arithmetic from §3): exact two-tower scoring of everything costs ~2 ms because it's one dot product per item. The DNN costs ~1 MFLOP per item, because it crosses user and item features, and that's too expensive for all 50k. So the cheap model sees everything and the expensive model sees ~500. A second reason: **several sources** (co-occurrence, sequence, trending, fresh) supply candidates a single embedding would miss, which is where diversity and exploration come from.

```
THE VALUE SCORE (calibrated heads, owned weights):
  V = p_click · (w_d · ŷ_dwell + w_c · p_convert) − w_h · p_hide
  ranker loss: Σ_k α_k · BCE(head_k), each head masked to rows with its label
```

**How many models? The full inventory.**

| # | Learned model | What it does | When it runs |
|---|---|---|---|
| **1** | **Two-tower retrieval** (one model, two towers) | **User tower:** long-term embedding + session sequence + context · **Item tower:** item content + ID, for the whole ~5×10⁴-item catalogue | User tower per request; item tower precomputed |
| **2** | **Sequence model** (transformer over the last ~50 events) | Predicts the next item **in this session**: a second candidate source | Per request |
| **3** | **Content encoder** (pretrained text–image, LoRA-tuned) | Embeds **new items** so they're retrievable on day 0 | At item upload |
| **4** | **Pre-ranker** (distilled from the ranker) | Cuts 500 candidates to ~200, only if latency is tight | Per request |
| **5** | **Main ranker:** DNN with DCN-v2 crosses + target attention, multi-task heads (click, dwell/conversion, hide) | Orders the candidates | Per request |
| | ↳ **Shallow position tower** | Absorbs position bias | **Training only**, dropped at serving |
| **6** | **GBDT baseline** | The v0 ranker, **kept loaded as the fallback** when the DNN misses its deadline | Per request, on fallback |

**Not models, but statistics or algorithms:** the co-occurrence table (item-to-item counts, a candidate source) · segment popularity and trending counts (candidate sources, cold start) · the **Thompson-sampling bandit** (Beta posteriors for fresh items, a category bandit for new users) · **isotonic calibration** per ranker head · the **MMR** diversity re-rank.

> **The long-term user embedding is not a separate model:** it's model 1's **user tower run in a daily batch** over long history. And the catalogue is ~5×10⁴ items ("tens of thousands"), not 10k.
>
> **The shape to draw:** retrieval (models 1, 2, 3 + co-occurrence + trending) → pre-ranker (4) → ranker (5, with 6 as fallback) → calibration → MMR + bandit slots.

### 5.5 Cold start, several strategies each (e) (f) `F2a` `F1c`

**New users (little or no history):**

| Strategy | How | When it carries the weight |
|---|---|---|
| **Context** | Locale, device, time, **referrer** (the ad or shared link they came from) | Request 1 |
| **Popularity, segmented** | Time-decayed popular items for the user's segment, **spread across categories** | Page 1, as a measurement |
| **Onboarding** | Optional three-tap interest picker | If they do it |
| **Session-based model** | The sequence model needs no user ID, only this session's events; the client sends its last events with the request | **After 2–3 interactions: seconds, not days** |
| **Exploration** | A category-level Thompson-sampling bandit sets the mix | The first sessions |
| **Graduation** | Blend weight shifts from segment and session toward the personal model as history grows | No cliff |

**New items (no interaction data):**

| Strategy | How | Why |
|---|---|---|
| **Policy gate** | Checks before the item enters the catalogue | Fresh items get a boost; unverified ones mustn't |
| **Content-based embedding** | A content tower (text + image + metadata) places it in the same space as warm items; trained with **ID dropout** so it works without an ID | Recommendable within minutes |
| **Priors** | Smoothed-CTR prior from the k nearest items in content space and from the **creator's** history | The ranker needs a starting point |
| **Exploration** | **Thompson sampling**: each fresh item keeps a Beta(α, β) posterior on CTR, **updated from the stream in seconds**; exploration slots go to the highest sampled CTR; propensities logged | ⭐ Real-time is the best cold-start tool: a new item's estimate tightens within minutes |

> **The Beta posterior, from the item's own click stream:** start from a **prior** borrowed from similar items, e.g. category CTR 4% → Beta(α = 2, β = 48). Then every **examined** impression of this item updates it: a click → α + 1; shown and not clicked → β + 1. After 100 impressions with 8 clicks: Beta(10, 140), mean 6.7%, and much narrower. Per request, each fresh candidate draws a sample θ ~ Beta(α, β) and the highest θ wins the exploration slot, so an uncertain item sometimes draws high and gets tried, while a clearly weak one rarely does. It **graduates** to the normal sources once the posterior is tight enough.
| **Graduation** | When the posterior is tight enough, the item leaves the fresh pool for the main sources | The boost decays as evidence arrives |

### 5.6 Keeping the model fresh: four cadences `F2c` `F1c`

```
SECONDS   features · hide list · bandit posteriors · catalogue changes
MINUTES   new-item embeddings (content tower inference) · trending
HOURLY    INCREMENTAL training: warm-start from the current model, train on the
          last hour's stream-joined examples
          · LABEL WAIT: an impression is labelled only after a click window
            (e.g. 10 min); "no click yet" is not a negative
          · GATE before publish: AUC / normalised entropy / calibration on a
            held-out slice of the most recent data vs the current model;
            prediction-mean and NaN sanity checks
          · CANARY on ~1% for 15 min with an online CTR guard → auto-rollback
DAILY     FULL RETRAIN from the offline store with COMPLETE delayed labels
          (conversions, retention): the source of truth that corrects any
          drift the incremental updates accumulated
```

**Fine-tune, don't retrain** `F2c`: the content tower is a **pretrained text–image encoder, LoRA-tuned** contrastively on co-engagement pairs, so a new item's embedding reflects what users treat as related. Every hourly and daily run **warm-starts** from the current checkpoint.

*(Fully online per-event training, Monolith-style with minute-level parameter sync, is the next step if the hourly cadence demonstrably leaves value on the table. FU 1 gives the bar for it.)*

### 5.7 The staged pipeline `A2d`

```
STAGE 1 · INGEST     client events → event bus (partitioned by user) → dedupe, bot filter
STAGE 2 · FEATURES   stream: windows → online store + offline log · batch: daily aggregates
STAGE 3 · LABEL      stream join with a wait window · delayed labels from the daily job
STAGE 4 · TRAIN      hourly incremental (gated) · daily full retrain · calibrate
STAGE 5 · PUBLISH    model registry → canary → all pods · catalogue snapshot + deltas
STAGE 6 · EVALUATE   offline screens → A/B → the monitoring runbook (§7)
```

---

## 6 · SERVING: LATENCY vs ACCURACY *(min 30–37)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

### The serving path

```
PRECOMPUTED (off the request path):
  item embeddings + features → in-process catalogue on every pod
  user long-term embedding → daily, online store
  user candidate set → recomputed NEARLINE on strong events (click, hide, purchase)
PER REQUEST (on the path):
  one batched KV read → candidates → ranker on ~500 → re-rank → filters → respond
DEADLINE PROPAGATION: the gateway sets 90 ms; each stage gets what's left, and
  a stage that can't finish in its share returns its best partial result
```

### The levers: complexity, latency, cost — *price each one* `F3a` `A3b`

> **The principled objective** `A3b`: *"Maximise the online metric subject to P95 < 100 ms and a cost-per-request budget. So every lever below is priced in **metric change per millisecond and per dollar**, measured by ablation, not intuition."*

| Lever | Latency effect | Accuracy cost | System cost |
|---|---|---|---|
| **Exact dot product over the in-memory catalogue** (vs ANN) | ~2 ms; no network hop | **None** | Trivial at 13 MB |
| **Candidates to the ranker** (1,000 → 500 → 300) | ~Linear in ranker time | Recall loss on the tail, measurable | Lower |
| **Distilled pre-ranker** (small DNN trained on the ranker's scores) | Cuts the heavy model's input 2–4× | Small | Lower |
| **Distil the ranker** (student mimics teacher) | 2–5× | A small AUC loss, gated | Lower |
| **Quantisation** (fp16 / int8 weights and activations) | 2–4× | Tiny, gated on the eval set | Lower |
| **Feature pruning** (drop features whose fetch cost exceeds their gain) | Removes remote reads | Measured by ablation | Lower |
| **Event-triggered precompute** of candidates | Moves ~10–20 ms off the path | Staleness, bounded by the trigger | Storage + nearline compute |
| **Cross-request GPU batching** | Higher throughput | — | Adds a few ms of queueing: cap the batch wait |
| **Hedged reads** on the KV store | Cuts P99 | — | ~5% extra reads |
| **Deadline-aware cascade** (DNN if time allows, else GBDT) | Bounds the tail | Only on the slow tail | — |

### Where to cache, and how stale each cache may be `F3b` `I1b`

| Cache | What | Where | TTL / consistency | Invalidation |
|---|---|---|---|---|
| **Item-level** | All item embeddings + features | **In-process, every pod** | Versioned snapshot every ~1 min + streamed deltas | **Removal events broadcast immediately**; a removed item never survives a minute |
| **User-level, long-term** | Long-term embedding, affinities | Online KV + a small per-pod LRU | Hours (it's a daily feature) | Overwritten by the daily job |
| **User-level, session** | Session features, hide list | Online KV, **not cached** | Seconds; staleness here is the whole point of real-time | Write-through from the stream; the client's last events cover the gap |
| **Candidate-level** | ~500 candidates per user | KV, keyed by user | 5–10 min | **Event-triggered** recompute on a click, hide or purchase |
| **Result-level** | The final ranked page | Per session | **30–60 s**: for back-navigation, pagination and as a fallback | Any interaction in the session; **model version in the key** |
| **Segment-popular** | Top-N per segment | In-process | 1–5 min | Regenerated nearline |

**Three consistency rules to state out loud:**
1. **The hide filter and the seen filter run after every cache, on every response.** A cache can be stale about relevance, never about something the user rejected.
2. **The model version and feature-schema version are part of every cache key**, so a rollout never mixes old cached scores with new ones.
3. **Stale-while-revalidate** for the result cache: serve the cached page instantly while recomputing it in the background, unless the user has acted since it was computed.

---

## 7 · METRICS, EXPERIMENTS, MONITORING *(min 37–43)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

### Offline → online

```
OFFLINE SCREENS (time-split; never a random split, which leaks the future)
  retrieval  recall@k · sliced: new users, new items, popularity decile
  ranker     AUC · normalised entropy · calibration (predicted / observed)
  slate      coverage · diversity · repeat rate
  counterfactual  SNIPS on the exploration slice with logged propensities
  ⭐ trust an offline metric only as far as it has predicted past A/B results
SHADOW     score live traffic, show nothing: latency, calibration, overlap
A/B        hash USER IDs into arms · layered experiments for independent changes
           · power analysis on the primary metric first · 1% → 5% → 50%
           · ≥ 1–2 weeks (novelty, weekly cycles) · CUPED for variance
           · guardrails: P95 latency, error rate, hide rate, crash rate, retention
INTERLEAVING for ranker-vs-ranker: a faster read on ranking quality
HOLDOUT     ~1% on the old system for months: retention, the long-term truth
```

**Interleaving, worked through** (team-draft, the common version):

```
ONE USER, ONE REQUEST. Both rankers rank the SAME candidates:
  rank   ranker A (current)          ranker B (new)
   1     "Easy weeknight pasta"      "Trail running: hills"
   2     "Trail running: hills"      "Best trails near you"
   3     "Road marathon pacing"      "Trail shoe review"
   4     "Best trails near you"      "Easy weeknight pasta"

BUILD ONE LIST by taking turns, like captains picking teams (a coin flip
decides who picks first; each takes its best item not yet picked):
  coin flip → B first:  B picks "Trail running: hills"   (tagged B)
                        A picks "Easy weeknight pasta"   (tagged A)
                        B picks "Best trails near you"   (tagged B)
                        A picks "Road marathon pacing"   (tagged A)
  SHOWN: hills (B) · pasta (A) · best trails (B) · marathon (A)

THE USER TAPS "hills" and "best trails" → both B's picks → B wins 2–0
  (a tap on "pasta" would have given A a point)

OVER MANY REQUESTS: B wins 5,400 · A wins 4,100 · 500 ties
  → B preferred on 57% of decided requests: clearly significant after
    ~10⁴ requests, where an A/B test might need 10–100× more traffic

WHY IT'S FAIR: the coin flip balances who gets slot 1; the SAME user judges
  both rankers on the SAME request, so a picky user or an easy request
  cancels out
WHAT IT CAN'T TELL YOU: how much a business metric (retention, conversion)
  would move → interleaving picks the winner fast, an A/B measures the effect
```

> ⚠ **The real-time-specific trap:** *"Continuously trained models contaminate each other in an A/B. If both arms' models are retrained hourly on the **pooled** logs, control learns from treatment's traffic and the measured difference shrinks. For model experiments, each arm trains on **its own arm's logs**, or I accept and report the bias."* (FU 8)

**Success** = retention and engaged time (or conversions) up, hide rate not up, every system SLO held. `I2b`

### What I'd monitor in production, and the response — *the runbook*

| Signal | Example threshold | Automatic response | Human response |
|---|---|---|---|
| **P95 / P99 latency** per stage | P95 > 90 ms for 5 min | Shift traffic to the cascade's cheaper tier; shed low-priority load | Profile the slow stage |
| **Error rate** per dependency | > 1% | **Circuit breaker** → fallback tier | Page on-call |
| **Event-to-feature lag** | > 60 s | Stale features fall back to defaults + flags | Scale or replay the stream |
| **Feature distribution shift** (PSI, null rate per feature) | PSI > 0.2, or nulls doubling | **Block model pushes** | Find the upstream change |
| **Training–serving skew** (logged vs recomputed) | Beyond tolerance | Block pushes | Fix the definition, backfill |
| **Prediction drift**: mean predicted vs observed CTR | Calibration off > 10% | Recalibrate; if it persists, roll back | Investigate drift vs bug |
| **Online CTR per model version** | Canary below control | **Auto-rollback** the incremental push | Inspect the training window |
| **Fallback rate** | > 2% of requests | — | Capacity or a dependency is degrading |
| **Coverage / concentration** | Top-1% share rising week over week | — | The feedback loop is narrowing |

**Bias sources and mitigations** `F1c`:

| Bias | Mechanism | Mitigation |
|---|---|---|
| **Position** | Top slots get clicked regardless | Shallow position tower; randomised slice |
| **Selection** | Only shown items have labels | Exploration slots with propensities |
| **Popularity, faster** | Real-time trending amplifies what's already winning, within minutes | Velocity caps; trust-weighted counts; coverage monitor |
| **Label delay** | Recent impressions look like negatives | Wait windows; delayed labels from the daily job |
| **Leakage** | Features computed after the impression | Point-in-time joins; log-at-serve |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 43–47)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | P95 < 100 ms with a per-stage budget; cost per 1,000 requests; GPU/CPU utilisation | P95 / P99 per stage · $/1k requests |
| **Reliability** `I1b` | Deadlines, circuit breakers, the degradation ladder (§9); multi-region; stream replay | Fallback rate · availability |
| **Usability** `I1c` | Reacts within the session; hides act on the next response; no repeats across pages; stable page on back-navigation | Hide rate · repeat rate |
| **Scalability** `I1d` | Stateless rankers autoscaled; KV sharded by user; stream partitioned by user; catalogue replicated per pod | QPS headroom · stream lag |
| **Privacy** `I1e` | Event streams are behavioural data: **retention limits on raw events** (e.g. 90 days; aggregates longer); **deletion propagates** to the online store, offline logs and the next retrain (user features, not per-user ID embeddings, so there's no user parameter to scrub); coarse location only; no sensitive inference | Deletion latency · retention compliance |
| **Safety** `I1f` | New items pass policy checks **before** entering the catalogue; the trending source has velocity caps and human review for the top trends, because real-time amplifies harm fastest | Policy-violating impressions · trend review time |
| **Security** `I1h` | Signed client events, auth, rate limits, and **beyond the gateway**: ⚠ **click farms pushing trending in minutes** → trust-weighted counts, new-account down-weighting, velocity anomalies; ⚠ **poisoning the continuous-training loop** → the hourly gate, canary and auto-rollback are a security control too | Fraud events filtered · rejected pushes |
| **Ethics / bias** `I1i` | Fresh items and new creators get funded exploration; real-time popularity loops capped; performance audited per user segment and region | New-item time-to-first-impression · per-segment calibration |
| **Explainability** `I1j` | Reasons from source ("Because you just viewed X"); **log-at-serve makes every recommendation replayable** with its exact features and model version; per-head contributions logged | % responses replayable · % items with a reason |

> ⭐ *"In a real-time system, the speed that makes it useful is the same speed that lets a mistake, an attack or a bad model reach everyone, so every fast path needs a fast brake."*

---

## 9 · SCALABILITY, RELIABILITY, FAILURES *(min 47–52)* — probe **(i)** `F3c` `F2a` `A5b`

### Storage and compute choices

| Need | Choice | Why |
|---|---|---|
| **Event ingestion** | A durable, partitioned **message log** (Kafka / Pub/Sub class), partitioned by user ID, ~7 days' retention | Ordering per user; **replay** after a stream failure |
| **Stream processing** | A stateful stream processor (Flink / Dataflow class) with **event time, watermarks**, dedupe by event ID, idempotent upserts | Late and duplicate events are normal on mobile |
| **Online feature store** | Replicated low-latency KV (Bigtable / Redis class), P99 < 10 ms | One batched read per request |
| **Offline store and logs** | Object storage, **columnar files partitioned by hour**, plus a warehouse for joins | Cheap, scans well, point-in-time joins |
| **Training** | Accelerators for the daily retrain; a smaller fleet for hourly increments | Different cadences, different budgets |
| **Model registry** | Versioned artefacts + metadata (data window, gate results) | Rollback in one step |
| **Serving** | Stateless candidate and ranking services, autoscaled, in separate fleets | Isolation: a slow ranker can't starve candidate generation |

### Fault tolerance: the degradation ladder `A5b` `F2a`

```
TIER 0 · FULL       DNN ranker on ~500 candidates, all features          (normal)
TIER 1 · LIGHT      ranker missed its deadline → GBDT baseline on the same
                    candidates (it's always loaded: the baseline never leaves)
TIER 2 · CACHED     ranking service down → the user's result cache (≤ 60 s)
                    or the precomputed candidate set ordered by two-tower score
TIER 3 · POPULAR    user features unavailable → segment-popular + context
                    + the client's last events (session co-occurrence)
TIER 4 · STATIC     everything down → a last-known-good list bundled in the
                    response cache at the edge, or held on the device
EVERY TIER: the hide, seen, expired and policy filters still apply
"A degraded list is a product decision, not an outage."
```

**The mechanics:** a deadline propagated from the gateway · per-dependency **circuit breakers** · **bulkheads** (separate pools per dependency) · **load shedding** by request priority (a pull-to-refresh beats a background prefetch) · multi-region active-active · stream recovery by **replaying from the log's offsets** into idempotent upserts · model rollback to the last version that passed its gate.

### Extension: on-device re-ranking `F3c`

A small model on the phone re-ranks the server's top ~50 using signals the server never sees in time: scroll speed, dwell on the current card, an item skipped two seconds ago (Alibaba's EdgeRec is a published example). It has **zero network latency** for in-session reaction. The costs: model updates ship through the app, device heterogeneity makes latency unpredictable, and logging and experimentation get harder. Start with a tiny, server-trained re-ranker behind a flag.

### Top 3 failure modes

1. **Silent staleness:** the stream lags, features stop updating, nothing errors, and "real-time" quietly becomes "this morning". Hence the lag monitor and per-feature timestamps.
2. **Leakage:** features joined without point-in-time correctness make offline metrics look brilliant and online results disappoint.
3. **A bad push at machine speed:** an ungated incremental update (a bug, poisoned data, a broken feature) reaches every user within the hour. Hence the gate, canary and auto-rollback.

**Q1 ships:** the event bus and stream features, the feature store with **log-at-serve**, the **GBDT baseline** with popularity and co-occurrence sources, the degradation ladder, and the exploration slice. The DNN and hourly incremental training come in Q2, once the baseline has set the bar.
> *"Logging served features and the exploration slice come first, because neither can be added retroactively, and every later model depends on both."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "How fresh does the model need to be? Why not train online on every event?"** `A5a` `F1c` `I1h`
> Separate three things: features, items and weights. Most of real-time's value is in **features**: the last few clicks, the hide list, session counts, updated in seconds. Those change the next recommendation without touching a weight. **Weights** need to move when the relationship between features and outcomes changes, which happens over hours, not seconds: a new trend, a seasonal shift. Per-event learning of the whole model costs a lot. It needs a training system on the serving critical path; the labels are immature, because an impression without a click yet isn't a negative; it's unstable; and it's a **poisoning path**, because anything that can generate events can move weights immediately. So my default is hourly incremental updates behind a quality gate and a canary, a daily full retrain with complete labels, and **per-event updates only where they clearly pay**: bandit posteriors for new items and the hide list. Per-event or minute-level training of sparse embeddings, Monolith-style, is the next step **if** the experiment that compares hourly against daily weights shows that freshness of weights, not features, is where the remaining value is.

**2. "Explain point-in-time correctness. What goes wrong without it?"** `F1c` `A2d`
> Every training example is an impression at some time t, and its features must be the values the system **could have known at t**. That means the latest update with a timestamp at or before t, minus the pipeline's real lag. Without it you get **leakage**. The classic case: an item's real-time CTR feature, computed over a window that includes the very click you're trying to predict. The model learns that this feature is predictive, which it is, of the past. Offline metrics look excellent, and online they disappoint, because serving never had that information. The subtler version is forgetting the lag: if the stream takes ten seconds to update the online store, training must use values as of t minus ten seconds, or it sees fresher features than serving ever did. My defence is layered: **log features at serving time** so existing features are exactly what the model saw, use point-in-time joins only to backfill new features, and run a daily job that recomputes a sample of logged rows and flags disagreement.

**3. "The model server is down. What does the user see?"** `I1b` `A1c` `A5b`
> A slightly less personal feed, not an error. The ladder: if the DNN ranker misses its deadline or its circuit breaker is open, the **GBDT baseline** scores the same candidates. It's always loaded, and it's the model we shipped first, so it's well understood. If the ranking service is entirely down, the user's **result cache** from the last minute, or their precomputed candidate set ordered by two-tower score. If user features are unavailable too, **segment-popular** plus the client's last few events. And if everything is down, a last-known-good list at the edge. Every tier still applies the hide, seen, expired and policy filters, because degrading relevance is acceptable and showing something the user rejected isn't. Mechanically: deadlines propagated from the gateway, circuit breakers per dependency, bulkheads so one slow service can't exhaust the others' threads, and load shedding that drops background prefetches before user-initiated refreshes. I'd track the **fallback rate per tier** as an SLO, because a system that's quietly serving Tier 1 to 20% of traffic has an outage nobody declared.

**4. "An item was published five minutes ago. Walk me through how it gets recommended."** `F2c` `I1f` `F2a`
> Minute 0: it passes the **policy check**; nothing enters the catalogue unchecked, because fresh items get a boost and that boost mustn't go to harmful content. Minute 1: the **content tower**, a pretrained text–image encoder LoRA-tuned on co-engagement, computes its embedding. The item tower was trained with ID dropout, so the vector is sensible without any ID history. Its features (category, creator quality, an **initial CTR prior** borrowed from its nearest neighbours and its creator) are written to the catalogue. Minute 2: the catalogue delta is **broadcast to every serving pod**; since each pod holds the whole catalogue in memory, it's immediately retrievable by the exact two-tower and eligible for the **fresh-item pool**. From then on, **Thompson sampling** gives it exploration slots: its Beta posterior on CTR updates from the stream in seconds, so a good item earns more impressions within minutes and a bad one fades. When the posterior is tight, it graduates out of the fresh pool into the normal sources. This is where real-time pays off most: a batch system would know nothing about this item until tomorrow.

**5. "Where would you cache, and how do you stop a cache from showing an item the user just hid?"** `F3b` `I1c`
> Five caches, each with a staleness I'm willing to accept. **Item-level**: the whole catalogue, in-process on every pod, refreshed by a one-minute snapshot plus streamed deltas, with removals broadcast immediately. **Long-term user features**: KV plus a small local LRU, hours of staleness, since they're daily. **Session features: not cached**, because their freshness is the point. **Candidate sets**: per user, 5–10 minutes, recomputed when a strong event arrives. **Result pages**: 30–60 seconds, for pagination and back-navigation and as a fallback, with the model version in the key. The hide problem is solved by **ordering**, not by invalidation alone: the hide, seen and policy filters run **after** every cache, on every response, reading the hide list written through from the stream. And for read-your-writes, the client sends its last few events with the request, so a hide counts even if the stream is a few seconds behind. A cache can be stale about relevance, never about a rejection.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "The stream processor falls twenty minutes behind. What happens, and what do you do?"** `I1b` `F2a` `F1c`
> What happens is the dangerous part: **nothing visible**. Requests still succeed; they're just using twenty-minute-old session features, so the feed stops reacting to what the user just did, and engagement erodes quietly. Detection: every streaming feature carries its last-update timestamp, and **event-to-feature lag** is an SLO with an alert. Mitigation while it's behind: features past their TTL are replaced by their defaults with a "missing" flag, which the model has seen in training through feature dropout, so it degrades to "long-term interests only" rather than acting on misleading stale values. The client-sent last events keep the most important in-session signal alive even without the stream. Recovery: scale out the consumers and catch up by replaying from the log's offsets. Upserts are idempotent and windows use event time with watermarks, so a replay produces the same state as a live run. Afterwards, check the incremental trainer: an hour trained on lagged features should be rolled back or excluded.

**7. "Your continuously trained model's CTR dropped 4% overnight. Walk me through it."** `I2a` `F2a`
> First, is it the model? Compare CTR **per model version**: if the canary system served a new incremental push overnight and CTR fell on that version only, **roll back** first and investigate second. If every version dropped, it's not the model: check **data and features**, with distribution monitors (PSI, null rates) on served features, the skew check, and event-to-feature lag, because a broken upstream feature looks exactly like model drift. Then **the world**: a holiday, a news event or an app release changes behaviour, and the control arm and holdout will show the same drop. Then **logging**: a client release that changed how impressions are counted changes CTR without changing anything users experience. Only after ruling those out would I call it genuine **concept drift**, and the response is a recalibration or a retrain on a window weighted toward recent data. The structural lesson: per-version metrics, feature monitors and a holdout exist so that this diagnosis takes minutes, not days.

**8. "How do you A/B test two continuously trained models fairly?"** `I2a` `F1b`
> The catch is **shared training data**. If both arms' models are retrained hourly on the pooled logs, each learns from the other's traffic. Control picks up whatever behaviour treatment's recommendations caused, the difference between arms shrinks, and a real gain looks like noise. Options: train each arm's model **only on its own arm's logs**, which is the clean version and costs one pipeline per arm; or make the arms large enough that each arm's own data dominates, and report the bias. Either way, compare **like with like**: both arms at the same update cadence, so I'm not comparing a fresh model with a stale one. Beyond that, the normal discipline applies: user-level hashing, a power analysis up front, guardrails on latency and hides, 1–2 weeks for novelty, CUPED for variance, and a long-term holdout for retention. For ranker-only changes, **interleaving** is a faster, lower-variance first read before committing traffic to a full A/B.

**9. "P95 is 140 ms. Get it under 100 without losing accuracy."** `F2b` `A1c` `F3a`
> Measure before cutting: per-stage P95 from tracing, because the slow stage is often not the model. In my experience the order of suspects is: **feature fetches** (serial reads that should be one batched call, or a remote feature that should be in-process), **the ranker** (too many candidates, no batching, fp32), then **tail effects** (garbage collection, cold caches, a slow KV replica). Fixes in order of accuracy cost, cheapest first. No accuracy cost: parallelise and batch reads, **hedge** KV reads to cut the tail, move item features in-process, **event-triggered precompute** of candidates. Small, measured cost: a **distilled pre-ranker** to send 300 candidates instead of 500, **int8 / fp16** inference, pruning features whose fetch cost exceeds their ablation gain. Bounded cost: a **deadline-aware cascade** that serves the GBDT on the slowest few percent of requests. Each step is gated on an offline check and a short A/B, so I'm trading measured metric per millisecond, not guessing.

**10. "A click farm targets your trending source."** `I1h` `F1c`
> Real-time systems are easier to poison than batch ones: a coordinated burst moves a trending score in minutes, and if an ungated trainer is consuming the same events, it moves weights too. Defences at three layers. **Signals**: trending counts are **trust-weighted** (new accounts, emulators, unusual device or network clusters count for little), they use **distinct users** rather than raw events, and **velocity anomalies** (organic trends ramp; farms spike) trigger a hold. **Exposure**: the trending source has a velocity cap, items it surfaces still pass the ranker's quality heads, and the very top trends get human review. **Learning**: fraud-filtered events feed training, and the incremental gate plus canary rejects a push whose behaviour shifts abnormally, so the training loop has a brake. Signed client events and per-device rate limits make large-scale fabrication more expensive in the first place. And every confirmed attack becomes training data for the fraud detector.

---

## ⭐ THE L6 LINE

> **"Real-time is three freshness problems: features in seconds, items in minutes, weights hourly behind a gate. Fifty thousand items fit in memory, so retrieval is exact; every stage has a deadline and a fallback; and point-in-time correctness and logged serving features are what keep a fast system honest."**

## 🧭 HOOKS — *steer by what you mention*

- *"Three freshness problems, three costs"* → invites the online-learning branch (**good at L6**)
- *"Fifty thousand items is 13 MB"* → invites the exact-retrieval and caching branch
- *"Point-in-time correctness"* → invites the feature-store branch (**good at L6**)
- *"The hide filter runs after every cache"* → invites the consistency branch
- *"A degraded list is a product decision"* → invites the reliability and fallback branch
- *"Continuously trained arms contaminate each other"* → invites the experimentation branch
- *"Every fast path needs a fast brake"* → invites the security and safety branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three tiers: what's online, nearline and offline."* The diagram regenerates the design.
- **Running out of time?** → *"I have five minutes. I'd rather cover the degradation ladder and monitoring than finish the advanced ranker. Sound right?"* (Both are named aspects of this prompt.)

**Name and defer:**
- "**Per-event online learning of the full model** (Monolith-style) is a step I'd take only after measuring what weight freshness is worth."
- "**ANN indexes** come back if the catalogue grows ~100×; at 5×10⁶ items, exact search stops being cheap."
- "**Notifications** reuse the feature store but add a send-time and fatigue model."
- "**Cross-region feature replication** has its own consistency trade-offs; I've assumed per-region stores."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"Features and catalogue in seconds, weights hourly behind a gate; exact retrieval over an in-memory catalogue."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"10⁴ QPS, 10⁵ events/s; ~63 ms planned of a 100 ms P95; exact retrieval in ~2 ms."* |
| **A1c** | Degrade without lowering quality | §4, §6, FU 3, 9 | *"A deadline-aware cascade: the DNN when time allows, the GBDT on the slow tail."* |
| **A2a** | The full stack | §4 diagram | Online / nearline / offline, with the data flow between them |
| **A2b** | Model components | §5.4 | Baseline (popularity, co-occurrence, GBDT) → two-tower, sequence model, multi-task DNN |
| **A2c** | Data sources | §5.1 | Client events, impression logs with served features, catalogue, delayed outcomes, context |
| **A2d** | Staged pipeline | §5.3, §5.7 | Ingest → features → label → train → publish → evaluate; the feature registry |
| **A3a** | Top-down, unprompted | §0 | *"Three freshness problems"* + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Maximise the online metric subject to P95 and cost; price every lever in metric per millisecond."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Item churn? Which interactions must show up next response? Objective? Where is the 100 ms measured? |
| **A5a** | Alternatives compared | §4, FU 1 | Features vs incremental vs per-event learning; ANN vs exact; lambda vs one definition |
| **A5b** | A full alternative pipeline on demand | §9 | The degradation ladder, tier by tier |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.4, §5.5 | *"GBDT baseline, then a DNN with crosses and target attention; Thompson sampling for fresh items."* |
| **F1b** | Metrics and their trade-offs | §2, §7, FU 8 | *"Switch real-time features off once to learn what they're worth."* |
| **F1c** | Bias sources and mitigations | §5.6, §7, FU 2 | Position, selection, faster popularity loops, label delay, **leakage** |
| **F2a** | Something fails → a fallback | §9, FU 3, 6, 7 | The degradation ladder; defaults for stale features; per-version rollback |
| **F2b** | Latency too tight → design for it | §4, FU 9 | The pre-ranker and event-triggered precompute, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.6, FU 4 | A LoRA-tuned content encoder; warm-started hourly and daily runs |
| **F3a** | Efficiency with a quality trade | §6, FU 9 | Distillation, int8 / fp16, feature pruning, each gated |
| **F3b** | Scale levers | §6, §9, FU 5 | Catalogue replicated per pod; KV sharded by user; the cache table |
| **F3c** | Extension, with cost and maturity risk | §9 | On-device re-ranking |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | P95 < 100 ms with a per-stage budget; cost per 1,000 requests |
| **I1b** | Reliability | §6, §8, §9, FU 3, 6 | Deadlines, circuit breakers, bulkheads, stream replay, the ladder |
| **I1c** | Usability | §2, §8, FU 5 | Reacts within the session; hides act on the next response; no repeats |
| **I1d** | Scalability | §3, §8 | 10⁵ events/s partitioned by user; stateless autoscaled rankers |
| **I1e** | Privacy | §8 | Retention limits on raw events; deletion propagated to stores, logs and retrains |
| **I1f** | Safety | §8, FU 4 | Policy check before the catalogue; trending velocity caps and review |
| **I1h** | Security | §8, FU 10 | **Click farms on trending** and **poisoning the training loop** |
| **I1i** | Ethics / bias | §8 | Funded exploration for new items and creators; per-segment audits |
| **I1j** | Explainability | §8 | Log-at-serve makes every recommendation **replayable**; reasons from source |
| **I2a** | Offline → online | §7, FU 7, 8 | Time-split screens → shadow → user-hashed A/B with per-arm training → holdout |
| **I2b** | The success definition | §7 | *"Retention and engaged time up, hides not up, every system SLO held."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) architecture and data flow → §4 · (c) features and store → §5.2, §5.3 · (d) baseline → advanced, two stages → §5.4 · (e) new users → §5.5 · (f) new items → §5.5, FU 4 · (g) latency vs accuracy, caching → §3, §6 · (h) metrics, A/B, monitoring → §2, §7 · (i) scalability, reliability → §9.
