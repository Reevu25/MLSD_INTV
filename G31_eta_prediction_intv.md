# G31 · Maps ETA Prediction — **INTERVIEW SHEET**

> **Tier 1 · #6** — ⭐ **directly confirmed asked in a Google L6 loop** ("a system dealing with traffic data").
> Full study file: [`../G31_eta_prediction.md`](../G31_eta_prediction.md) · [diagrams](../G31_eta_prediction_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the system · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Which consumer, trip lengths, what we may do with probe data (§1) |
| **Explainability** is easy points | "Why this route", delay reasons on the map, honest ranges (§8) |
| **Security** beyond the gateway | **Spoofed traffic** (the 99-phones handcart), GPS spoofing, fake incident reports (§8) |
| **Troubleshoot before being asked** | Raise the **model timeout** and **sparse rural coverage** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **An ETA is a distribution, not a number** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · calibration + **stability** SLOs | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Routes, ranges, leave-by · **quantile per surface** · calibration, not MAE | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | ⚠ **The refresh path is 2–5× the initial requests** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · background (all drivers) vs request path (one user) · degrade | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Probes · map-matching · **segments don't add** · pinball loss · per-region | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Lookup, not traversal · horizon blend · hysteresis · E[t] + λσ | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Calibration by slice · **counterfactual eval** · region-level A/B | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Other modes · incident text · fallback path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"I want to fix the output type before I draw anything, because almost every mistake downstream follows from getting it wrong. An ETA is not a number, it is a distribution.**
>
> **The user's question is never 'what is the expected travel time' — it's 'will I be late?', which is a statement about the upper tail. So the model emits quantiles, trained with pinball loss, and the product decides which quantile to surface for which purpose.**
>
> **And those differ: comparing candidate routes wants the expected time E[t] plus a variance penalty, the displayed arrival time wants roughly the median, and the 'leave by' recommendation wants P85–P90 — because arriving early is small and linear, while missing a flight is large and discontinuous."**

**Then three commitments:**
1. **A route's ETA is NOT the sum of its segments' ETAs** — three compounding reasons (§5). Route-level model over supersegment embeddings.
2. **⚠ Our routing changes the traffic we're predicting** — the target is an **equilibrium**, evaluation is **counterfactual**, and there's a real externality.
3. **ETA stability fights accuracy** — a jittering ETA is a worse product than a slightly stale one. Designed, not emergent.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and how I measure it, capacity, the architecture, data and models, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Which consumer — the router, the displayed ETA, or in-trip refresh?** *(Three very different cost/latency profiles — **the highest-value clarification**.)*
2. **What's the trip-length distribution?** *(5 min is nowcasting; 4 hours is **forecasting** — a different problem.)*
3. **What are we allowed to do with probe data?** *(Privacy binds hardest exactly where coverage is worst.)*
4. **Driving only, or transit and walking too?** *(Transit is schedule plus deviation, a different model shape.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Product | Google-Maps-style **driving** directions and navigation |
| Load | ~2×10⁹ direction requests/day; ~2×10⁸ navigations/day |
| Probes | GPS from phones with location sharing **opted in**, aggregated only |
| Road graph | ~10⁹ edges, geo-sharded by S2 cell |
| Trips | Mostly 5–60 min; a long tail of multi-hour drives |
| Privacy | k-anonymity floor per edge, DP noise, retention limits |

**FR:** predictive distribution per route · cheap per-edge costs for the graph search · **in-trip refresh, stably** · live + historical blend · future departure times · graceful degradation on sparse coverage.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Directions request | p95 ≤ **100 ms** |
| **In-trip refresh** | p95 ≤ **50 ms** — ⚠ **and it's the volume driver** |
| Per-edge cost lookup | ≤ **1 µs** — the router can't call a neural net |
| **Quantile calibration** | Across trips, actual travel time ≤ the trip's own predicted P90 for **~90% of trips** (and likewise for every quantile) — **the headline** |
| **ETA stability** | The **displayed** arrival time changes rarely: changes below a threshold scaled to time remaining are suppressed, and update frequency is capped. Tracked as updates per trip + total \|Δ\| per trip |
| Availability | Always an ETA: a widened interval when the model or the data is weak |

> **Quantile calibration, in one breath:** across 10,000 trips, each with its *own* predicted P90, ~9,000 should arrive at or below it. **75% → overconfident** (people are late) · **98% → too pessimistic** (people leave needlessly early). Repeat for every quantile (P10→10%, P50→50%) → a **reliability curve** that should sit on the diagonal.
> ⚠ **Two guards:** check it **per slice** (urban/rural, trip length, horizon, peak), because rural at 70% plus urban at 95% can average to a "calibrated" 90%. And report **sharpness** (interval width or pinball loss) alongside, because a 5–120 min range is calibrated *and* useless.

**Scope / NON-GOALS:**
- ❌ **A point-estimate ETA.** Position, with the reason.
- ❌ **Gating on MAE** — reported, never the gate.
- ❌ Transit/walking/cycling · map construction · parking-time prediction (though it's in the user's experience even if not in our ETA).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
DIRECTIONS: A → B, leaving now
  ● Route 1 (recommended)   27 min   (24–31)   via US-101   "fastest, usually reliable"
  ○ Route 2                 26 min   (22–44)   via side streets   "can be much slower"
  ○ Route 3                 33 min   (31–36)
  "Leave by 8:41 to arrive by 9:15"   ← solved from P90, not P50

IN NAVIGATION: "Arrive 9:08"  ← the DISPLAYED ETA, stable: it moves only when
  the change is worth interrupting for ("+6 min: crash ahead")
```

**The metrics:**

```
⚠ MAE IS THE WRONG GATE — it's blind to the tail and to calibration.

HEADLINE:  QUANTILE CALIBRATION — does P90 empirically cover ~90%?
           LATENESS RATE — fraction of trips arriving after the surfaced ETA

WHICH QUANTILE FOR WHICH SURFACE (a product question with a real answer):
   displayed arrival      ≈ P50   (persistent pessimism erodes trust)
   "LEAVE BY" for a meeting ≈ P85–P90  (loss is DISCONTINUOUS)
   flights / hard deadlines  P95+
   route comparison        E[t] + λ·σ   E[t] = EXPECTED (mean) travel time,
                                        σ = its spread, λ = risk aversion
                           (a route usually 2 min faster but sometimes 40 min
                            slower should LOSE; ranks the ~3 candidate routes)
   shown uncertainty       the WIDTH, as a range ("24–31 min")

⚠ THE ASYMMETRY BELONGS IN THE QUANTILE CHOICE, not smuggled into the
  model as a bias term — there it's explicit, auditable, per-surface.
```

**Also bucket by trip length** (3 min of error on a 10-min trip ≠ on a 3-hour drive) and **slice by horizon** to test the live/historical blend directly.

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Lateness rate** at the surfaced quantile + **calibration** per slice | "Will I be late?" answered honestly |
| Sharpness | Interval width / pinball loss | Calibrated *and* useful |
| Stability | Displayed-ETA changes per trip · total \|Δ\| · "remaining time went up" count | The product, not the model |
| **Guardrails** | **Induced-flow share** on residential streets · k-gate coverage · p95 latency | The people who never opened the app |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **⭐ the insight that reorders the design** `A1b` `I1a` `I1d`

```
INITIAL DIRECTIONS REQUESTS
  ~2×10⁹/day → ~23k QPS avg, ~70k peak
  × ~3 candidate routes      → ~70k route-scorings/s

IN-TRIP REFRESH
  ~2×10⁸ active navigations/day × ~20 min × refresh every 10–30 s
  → ~10¹⁰/day  →  ~120–140k QPS sustained

⚠ THE REFRESH PATH IS 2–5× THE INITIAL-REQUEST PATH.

CONSEQUENCE: optimise for CHEAP INCREMENTAL RE-SCORING of a route
  already known — a refresh knows the remaining supersegments and the
  previous prediction, so it's an UPDATE, not a recomputation.
  (And that's also where the stability logic lives.)

STATE SIZE: ~10⁹ edges × ~32 B ≈ 32 GB → fits in RAM, geo-sharded (S2)

COST: the refresh path dominates the bill → track $ per 1,000 refreshes;
  the levers are incremental scoring, cached embeddings, and a small model
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| Sum of per-segment predictions | Cheap; reuses edge costs | Wrong: correlation, unattributed turn time, quantiles don't add (§5) |
| Historical profile only | Robust; needs no live data | Blind to today's incident |
| GNN with no route-level model | Captures how congestion spreads | Still sums at the end, so still non-additive |
| **Route-level model over supersegment embeddings** ✅ | Joint distribution per route; turns inside the unit | Bigger modelling unit; harder to attribute error |

### The system — *draw this in 90 seconds* `A2a`

```
══ BACKGROUND · runs continuously · ALL drivers ════════════════

   GPS TRACES FROM EVERY PROBE PHONE — millions of anonymous trips,
   NOT the requesting user's history
                        │
                        ▼
 ┌──────────────────────────────────────────────┐
 │ MAP-MATCHING (HMM/Viterbi), per trace        │
 │  "which edges did THAT car drive,            │
 │   and how long did each one take?"           │
 │  ⚠ parallel roads · canyons                  │
 │  ⚠ STOPPED ≠ STUCK ← biggest cleaning rule   │
 └──────────────────────┬───────────────────────┘
                        │ millions of observations from ALL cars:
                        │ "edge 8812 took 42 s at 17:03"
                        ▼
 ┌──────────────────────────────────────────────┐
 │ TRAFFIC STATE — EVERY edge, right NOW        │
 │  per edge: last ~5–10 min of ALL cars →      │
 │  speed + confidence (how many, how recent)   │
 │  + incidents · weather · events folded in    │
 │  ⚠ k-ANONYMITY GATE (privacy): if fewer      │
 │    than k distinct cars drove this edge      │
 │    recently, publish NO live speed for it    │
 │    (2–3 traces could identify a person).     │
 │    Use the next-best source instead, and     │
 │    WIDEN the ETA range at each step down:    │
 │    1st: neighbours' live speeds, via the GNN │
 │    2nd: historical speed, same hour + weekday│
 │    3rd: road-type default (limit, lanes)     │
 └────────┬──────────────────────────┬──────────┘
          │                          │
 ┌────────▼─────────┐   ┌────────────▼─────────────────┐
 │ EDGE-COST TABLE  │   │ GNN — a BATCH JOB, every     │
 │ for EVERY edge:  │   │ ~1–5 min, over the WHOLE     │
 │ expected SECONDS │   │ road graph: each supersegment│
 │ to drive it now  │   │ mixes in its neighbours'     │
 │ (live speed, or  │   │ traffic (message passing)    │
 │ the fallback)    │   │ → one EMBEDDING per          │
 │ + turn penalty   │   │   supersegment, written to a │
 │ e.g. edge 8812   │   │   KV CACHE: id → vector      │
 │ = 42 s           │   │ ⚠ the request path only READS│
 │ array in RAM,    │   │   the cache; the GNN never   │
 │ ~1 µs per read,  │   │   runs inside a user request │
 │ refreshed ~1 min │   │ ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │
 └────────┬─────────┘   │ SUPERSEGMENT = a chain of    │
          │             │   consecutive road segments  │
          │             │   (~0.5–3 km) whose traffic  │
          │             │   moves together, INCLUDING  │
          │             │   the turns and junctions    │
          │             │   inside it. e.g. "US-101 N, │
          │             │   exit 3 → exit 5". The unit │
          │             │   the ETA model works in, so │
          │             │   turn time has somewhere to │
          │             │   live (§5)                  │
          │             └────────────┬─────────────────┘
══════════╪══════════════════════════╪ REQUEST PATH · per request, ONE user ══
          │ edge costs               │ embeddings (from the cache)
          │ ◄─ USER asks: A → B, now │
          ▼                          │
 ┌────────────────────────────────┐  │
 │ ROUTER — A* / CH graph search  │  │
 │ TASK: find the ~3 best         │  │
 │   candidate routes from A to B │  │
 │   for this user                │  │
 │ HOW: search the road graph,    │  │
 │   reading the edge-cost table  │  │
 │   (1000s of ~1 µs lookups)     │  │
 │   ⚠ no neural net in here      │  │
 │ OUTPUT: 3 routes, each an      │  │
 │   ordered list of segments     │  │
 │   (just paths, NOT the ETA)    │  │
 └───────────────┬────────────────┘  │
                 │ the 3 routes      │
                 ▼                   ▼
 ┌──────────────────────────────────────────────────────────┐
 │ ROUTE-LEVEL ETA MODEL — runs once per candidate route    │
 │ TASK: predict how long THIS route will take, as a        │
 │   distribution (quantiles), not one number               │
 │ STEPS:                                                   │
 │  a. split the route into supersegments; LOOK UP each     │
 │     one's embedding from the cache                       │
 │  b. estimate WHEN the user reaches each supersegment     │
 │     (in 5 min? in 2 hours?) = its arrival HORIZON        │
 │  c. per supersegment, blend its LIVE speed (now) with    │
 │     its HISTORICAL speed (usual for that hour+weekday):  │
 │     reached soon → trust live; reached in 2 h → trust    │
 │     historical (live info goes stale after ~15–45 min)   │
 │  d. a sequence model over the supersegments → output     │
 │ OUTPUT, per route: travel-time QUANTILES, e.g.           │
 │   Route 1: P10 24 · P50 27 · P90 31 min                  │
 │   Route 2: P10 22 · P50 26 · P90 44 min  (riskier)       │
 │   Route 3: …                                             │
 └────────────────────────────┬─────────────────────────────┘
                              ▼
 ┌──────────────────────────────────────────────────────────┐
 │ SURFACE LOGIC                                            │
 │ TASK: turn the 3 distributions into what the user sees   │
 │ OUTPUT: recommended route = lowest E[t] + λ·σ            │
 │   (→ Route 1: slightly slower on average, far safer)     │
 │   shown ETA = its P50 · range = P10–P90 · leave-by = P90 │
 └────────────────────────────┬─────────────────────────────┘
                              ▼
          FIRST RESPONSE  │  IN-TRIP REFRESH (~140k QPS)
          (p95 ≤ 100 ms)  │  ⚠ incremental + HYSTERESIS
                          │  uses THIS user's own GPS, map-matched:
                          │  "where am I on my route?" → re-run the ETA
                          │  model on the REMAINING supersegments only
                          │  (the same trace also joins BACKGROUND as
                          │  one more anonymous probe)
```

> ⭐ **Say:** *"Segments are for search, supersegments are for prediction. The router needs a fine-grained scalar cost to find paths; the ETA model needs units big enough to contain turns, gather enough data, and keep the route sequence short."*

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The route model times out:** *"I return the router's summed edge costs with a **deliberately widened interval**, labelled as less certain: never no ETA, and never a falsely precise one."*
- **Sparse rural coverage:** *"Below the k-anonymity floor there's no live speed, so the edge falls down the ladder (neighbours via the GNN → historical → road-type default) and the interval widens at each step."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Quantiles, not a point estimate** | "The question is 'will I be late' — a tail question. A point estimate throws away exactly what answers it" | More outputs to calibrate and monitor |
| **2** | **Route-level model over supersegments** | "Quantiles don't add, errors are correlated, and turns belong to no segment" | Bigger modelling unit; harder to attribute error |
| **3** | **Precomputed aggregates, not traversal** | "You can't walk a billion-node graph in 100 ms — the online path is a **lookup**" | Slightly stale features; fine — embeddings refresh every few minutes and edge costs every minute |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Probe traces** (opted-in phones) | Per-edge speeds; completed trips as **labels** | Noise (canyons), parked cars, walkers; privacy |
| **The road graph** | Edges, turns, speed limits, lanes, restrictions | Map errors; closures |
| **Historical profiles** | Speed by edge × hour × weekday | Stale after road changes |
| **Incidents and reports** | Crashes, closures, user reports | Unverified; spoofable |
| **Weather and events** | Rain, stadium schedules | Scheduled anomalies (FU 9) |

### 5.2 Choosing the training set (e) `F1c`

```
1. MAP-MATCH     HMM / Viterbi per trace: the most likely SEQUENCE of roads,
                 so a later point (an exit ramp) can correct earlier ones
2. CLEAN         ⚠ STOPPED ≠ STUCK (a parked car is not a jam) · drop walkers
                 and cyclists by speed profile · split trips at long stops
3. PRIVACY       aggregate only; k-anonymous edges; DP noise on what's published
4. BUCKET        by trip length and urban/rural, so short urban trips don't drown
                 long rural ones
5. SPLIT BY TIME train on past weeks, validate on the next; ⚠ our own routing shaped
                 these trips, so offline accuracy is optimistic (§7)
```

**Map-matching is the hard cleaning step** (largest CPU line): urban-canyon multipath · **parallel roads** (a highway and its frontage road are metres apart with different speeds — a misattributed trace poisons *both*) · tunnels · **⚠ stopped ≠ stuck** (a parked car or coffee stop looks exactly like congestion, and then you route traffic *away* from a free-flowing road) · mode confusion · trip segmentation.

### 5.3 Why you can't sum the segments — the model follows from this `A2b`

```
REASON 1 · CORRELATION — congestion is correlated across neighbouring
   roads and over time. Summing per-segment predictions treats their
   errors as independent → UNDERSTATES the route's VARIANCE, and the
   tail is where the whole product lives.
REASON 2 · UNATTRIBUTED TIME — turns, signals, merges, ramp queues, and
   the TRANSITIONS between segments belong to no segment.
REASON 3 · QUANTILES DO NOT ADD — the sum of the segments' medians is
   not the median of the route. This one alone kills the approach.

THE FIX, three parts:
  SUPERSEGMENTS — turns and transitions sit INSIDE the unit
     → solves the unattributed-time problem
  + a GNN over the road graph → one embedding per supersegment
  + a ROUTE-LEVEL MODEL that predicts the whole route's distribution
    in one go, not segment by segment
     → solves correlation AND quantiles-don't-add
```

### 5.4 The objective, and fine-tuning (f) `F1a` `F2c`

- **Loss:** **pinball (quantile) loss** for P10 … P95 together, with a **non-crossing** constraint so P90 is never below P50. Pinball at α = 0.9 makes being late 9× as costly as being early, which pushes the prediction to the 90th percentile.
- **Fine-tune per region:** the GNN is trained globally, then **adapted per region** (region heads or LoRA-style adapters), because road networks and driving behaviour differ by city.

**Privacy as a first-class constraint:** aggregate only, **k-anonymity floor** (below k → no live estimate), DP noise, retention limits.
> **⚠ Name the tension:** *"Privacy binds hardest exactly where prediction is already worst — rural roads, where a single trace is most identifying. The honest response is to accept degraded ETAs there and widen the interval, not quietly lower k."*

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · PROBES        traces → map-matching → per-edge observations (seconds)
STAGE 2 · STATE         traffic state per edge, k-gated (every ~1 min)
STAGE 3 · EMBED         GNN batch job → supersegment embeddings → KV cache (~1–5 min)
STAGE 4 · TRAIN         route model on completed trips, pinball loss (daily / weekly)
STAGE 5 · GATE          calibration per slice → shadow → region-level A/B (§7)
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

**The horizon problem — a genuine modelling fork:**
```
A 3-hour drive: traffic on the FINAL segments at t≈180 min is NOT observable now.
  → LIVE traffic is informative ~15–45 min ahead, then decays fast.
  → beyond that you must FORECAST (historical profile by time-of-day ×
    day-of-week + known events) — a different problem.

⚠ THE BLEND WEIGHT IS PER-SUPERSEGMENT, BY WHEN YOU'LL TRAVERSE IT.
  A single global weight gets both ends wrong, in opposite directions.
  (Mildly recursive: estimate arrivals → blend → re-estimate. Two passes.)
```

**Three consumers, three cost profiles:**
| Consumer | Volume | Needs | Therefore |
|---|---|---|---|
| **Router** (A*/CH) | 1000s of edge-cost lookups/query (millions without CH) | scalar, ~1 µs | **A table lookup.** No NN inside A* |
| **Displayed ETA** | ~70k/s | accurate distribution, 1 route | The route-level model runs here (small, a few ms per route) |
| **In-trip refresh** | ~140k/s | a **stable** incremental update | Cheapest path; reuses cached embeddings |

- **Stability by design:** the displayed ETA changes only when |Δ| exceeds a threshold scaled to time remaining (a dead-band), at most once per few minutes unless the change is big; bad news shown promptly, good news absorbed slowly.
- **The route-ranking objective** `A3b`: recommend the route with the lowest **E[t] + λ·σ**, with λ (risk aversion) owned as a product decision.
- **Scale** `F3b` `F3a`: precomputed GNN embeddings; incremental refresh on the remaining supersegments only; a small, quantized route model.
- **Reliability** `I1b`: the coverage fallback ladder (each rung **widens the interval**); per-S2-shard failover; the widened-interval fallback when the model times out.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline** | **Quantile calibration** + **lateness rate** |
| Reported, never gating | MAE / MAPE |
| **Bucketed** | By trip length (relative *and* absolute separately) · urban/rural · peak/off-peak · coverage tier |
| **Horizon-sliced** | Error at t+5 min vs t+120 min — tests the blend directly |
| **Stability** | Update count and total update magnitude per trip |
| **⚠ Induced-flow share** | How much traffic on an edge is attributable to *our own* recommendations — the feedback-loop monitor |
| **A/B** | ⚠ **Contaminated by construction** — treatment routing changes control's traffic. Use **region-level randomisation**, not user-level |

**⚠ Why offline evaluation is counterfactual** `I2a`: the travel times in our logs were produced partly by our own routing. A new model would route people differently, so the traffic, and the "actuals", would differ. A model that sends 5,000 cars down a quiet street looks perfect on yesterday's logs and wrong the day it ships. So: offline calibration screens, a **traffic simulator** validated against reality, then **region-level A/B** or **switchbacks** (alternating policies over time).

**Success** = calibration within a few points on every slice, lateness rate down, stability within budget, induced flow on residential streets not rising. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Rural / sparse** | Privacy floors and few probes → worse ETAs | Honest, wider intervals; calibration sliced by coverage tier |
| **Horizon** | Live data says nothing about hour three | Per-supersegment blend by arrival horizon |
| **Feedback loop** | Our routing changes the traffic we learn from | Induced-flow share; region-level experiments |
| **Cleaning errors** | Stopped-vs-stuck mistakes mark free roads as jammed | Neighbour-probe checks; anomaly review |
| **Trip length** | Short trips dominate the data | Bucketed training and evaluation |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Directions p95 ≤ 100 ms; refresh p95 ≤ 50 ms at ~140k QPS; cost per 1,000 refreshes | p95 per path · $/1k refreshes |
| **Reliability** `I1b` | The **fallback ladder** (live → GNN-propagated → historical → road-class prior); per-S2-shard failover; the widened-interval fallback when the model times out | Fallback share · shard availability |
| **Usability** `I1c` | **Stable** displayed ETAs (hysteresis); honest ranges; "leave by" times | Updates per trip · lateness rate |
| **Scalability** `I1d` | Geo-sharded traffic state (~32 GB); incremental refresh; precomputed embeddings; customizable routing weights | Refresh QPS headroom |
| **Privacy** `I1e` | Aggregates only; **k-anonymity floor**; DP noise; retention limits; opted-in location only | k-gate coverage · retention audits |
| **Safety** `I1f` | Don't route through closed, flooded or evacuation-zone roads; a stable display reduces **driver distraction**; heavy vehicles kept off restricted roads | Routes through closures (≈0) |
| **Security** `I1h` | ⚠ **Spoofed traffic** (below); GPS spoofing; fake incident reports → corroboration before acting | Anomalous-jam alerts |
| **Ethics / bias** `I1i` | **Externality on residential streets** (FU 4) → policy penalties, induced-flow monitoring; **rural users get worse ETAs** because privacy floors bite there → widen intervals honestly | Induced-flow share · calibration by urban/rural |
| **Explainability** `I1j` | **"Why this route"** ("avoids a crash on 101, 6 min faster"); delay reasons on the map; internally, per-supersegment contribution to ETA error | % routes with a reason |

> ⭐ *"Our routing changes the traffic, so production-readiness includes the people who never opened the app: residents of the side streets we route onto."*

> ⚠ **Worth saying in full: spoofed traffic.** In 2020 an artist pulled **99 phones in a handcart** through Berlin and Google Maps showed a traffic jam on an empty street. So the traffic state requires **distinct-device diversity**, checks plausibility against neighbouring edges, and flags sudden jams for anomaly review.

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Other modes:** transit (schedule plus deviation) and cycling (weather-sensitive) each need their own model shape.
- **Text incident reports as features:** an LLM can extract "lane closed, northbound" from user reports. Cost: a model call per report. Risk: unverified reports, so require corroboration from probe speeds before a report moves an ETA.

**When the model or the data can't be trusted, the fallback pipeline** `A5b` `F2a`:

```
route model timed out, or the region's live data is thin
  → SUM the router's edge costs (the scalar path, always available)
  → WIDEN the interval by the coverage tier (ladder rung) of each segment
  → for a scheduled anomaly (a stadium event): apply the event calendar or a
    manual override now, retrain later (FU 9)
  → label the ETA as less certain in the UI rather than hiding it
```

**Top 3 failure modes:**
1. **The feedback loop** — route everyone onto the fast side street and it stops being fast. Self-defeating prediction + a real externality onto residents who never opted in.
2. **Instability** — 24 min, then 31, then 26. Each update more accurate, the product worse.
3. **Stopped-vs-stuck misclassification** — makes free-flowing roads look congested, then routes traffic away from them.

**Q1 ships:** **quantile output** (pinball loss) + calibration/lateness evaluation + per-surface quantile choice.
> *"Highest ratio of product impact to engineering cost in the plan — same features, same pipeline, different loss and head — and it unlocks 'leave by' times and honest ranges."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not predict expected travel time and minimise MAE?"** `F1b` `A5a`
> Because the user's question is never "what's the expected travel time" — it's "will I be late," a statement about the upper tail, and a point estimate throws away exactly the information that answers it. Travel-time distributions are strongly right-skewed: nothing goes better than free-flow, plenty goes far worse, so the mean sits well below the tail that determines whether someone misses a meeting. Two routes with identical means and very different variances are not the same product, and a point model can't distinguish them. So: quantiles with **pinball loss** — no distributional assumption, which matters given the skew — and **calibration as the headline**: does P90 empirically cover 90%? MAE stays as a familiar reference, but it's blind to both the tail and the calibration, and a team gating on it will ship a model that's sharper on average and worse at the only question users ask.

**2. "Just sum the per-segment predictions. What's wrong?"** `A5a` `F1a`
> Three things, and they compound. **Congestion is spatially and temporally correlated**, so summing marginals treats errors as independent and substantially *understates variance* — destroying the tail the product depends on. **A meaningful share of travel time belongs to no segment**: turns, signals, merges, ramp queues, and the transitions between segments, and a per-segment decomposition has nowhere to put it. And fatally on its own, **quantiles do not add** — the sum of medians isn't the median of the sum, so even perfectly calibrated per-segment quantiles give you a miscalibrated route quantile. The fix is structural: group segments into **supersegments** so turns and transitions sit *inside* the modelling unit, learn embeddings over the road graph, and predict the **route's distribution jointly**. That gets correlation and non-additivity by construction rather than by correction.

**3. "It's a three-hour drive. Live traffic tells you nothing about the last hour."** `A2b` `F1c`
> Correct, and it's a modelling fork rather than a detail. Live traffic is informative over roughly 15–45 minutes and decays quickly beyond that, so the far end of a long trip is a **forecasting** problem while the near end is **nowcasting** — different sources, different uncertainty. The key decision is that the blend must be **per-supersegment, keyed on when we expect to traverse it**, not per-route: a single global weight gets the beginning and end wrong in opposite directions. That makes the prediction mildly recursive — you need estimated arrival times to choose each blend, and arrival times are what you're computing — which two passes resolves. It also means uncertainty should **grow along the route**, which is the honest behaviour: a model whose interval doesn't widen with horizon is lying about something it knows.

**4. "Your routing changes the traffic you're predicting."** `I1i` `A3b`
> Yes, and I'd rather raise it than have it raised — it's what makes this a systems problem rather than a regression problem. Route enough drivers onto the fast side street and it stops being fast; the prediction is **self-defeating**. Three consequences: the **target is an equilibrium** — we're predicting the world *after* our recommendation lands; **evaluation is counterfactual**, since accuracy is measured against traffic our own routing produced, so a better model can score worse simply by moving more people; and there's a genuine **externality** onto residential streets whose residents never opted in. What I'd do: **load-aware routing** that accounts for the flow we're about to induce (closer to traffic assignment than prediction); **route diversification** across comparable alternatives, which damps herding and usefully doubles as the exploration that keeps alternatives observable; **explicit policy penalties** on sensitive road classes rather than letting an optimiser discover them; and monitor **induced-flow share** as the leading indicator that the loop is closing.

**5. "Users complain the ETA keeps jumping around."** `I1c` `F2a`
> That's a real product failure and it's in direct tension with the accuracy metric, which is why it usually has no owner. 24 minutes, then 31, then 26 is worse than a slightly stale number, even though each update was individually more accurate, because users experience instability as unreliability and can't act on a number that keeps moving. So: **hysteresis** — suppress updates below a magnitude threshold that scales with remaining trip time, since two minutes matters on a fifteen-minute trip and is noise on a three-hour one. **Deliberately asymmetric** — surface deterioration promptly because it's actionable (call ahead, reroute), absorb improvement slowly because it isn't. Avoid the most damaging pattern, remaining time repeatedly going *up*. And **measure it**: update count and total update magnitude per trip become tracked product metrics, because otherwise stability makes the accuracy dashboard look worse while making the product better, and it loses that argument every time.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you A/B test a routing change?"** `I2a`
> Carefully, because a naive A/B here is **contaminated by construction**: the treatment arm's routing changes the traffic that the control arm drives through, so the control is not a clean counterfactual. That's interference in a physical medium — the same structural problem as auction interference in ads. So user-level randomisation is invalid, and the ladder is: **region-level randomisation**, where whole metros are assigned to arms so the interference is contained within an arm; **time-sliced switchbacks**, alternating the policy over hours or days; and honest acknowledgement that any experiment large enough to move traffic is measuring an **equilibrium**, not a treatment effect. I'd also validate the offline↔online correlation of whatever simulator we use, because at this level of interference the simulator eventually becomes the primary instrument and its own accuracy becomes the thing to measure.

**7. "A rural road with three probe traces today. What do you show?"** `I1e` `F2a`
> A wider interval, via an explicit fallback ladder rather than a confident number from thin data. Live observed speed only if sample count clears both a confidence bar *and* the k-anonymity floor; then a **GNN-propagated estimate** from well-observed neighbours — the strongest practical argument for a graph model, since message passing carries information into sparse regions; then the historical time-of-day profile; then a road-class prior from speed limit and geometry. The important behaviour is that **each rung down widens the distribution**, because falling down the ladder *is* a statement of increased uncertainty and it costs nothing to represent honestly. And I'd flag the uncomfortable interaction: the k-anonymity floor bites hardest exactly here, because with three traces a single one is potentially identifying. **Privacy constrains us most where our predictions are already weakest**, and the right response is the widened interval, not a quietly lowered k because rural metrics look bad.

**8. "You're aggregating people's location traces. Defend that."** `I1e` `I1i`
> It has to be a first-class design constraint rather than compliance afterwards, because it genuinely shapes the architecture. The traffic path stores and serves **aggregates only** — never per-user segment attribution — with a hard **k-anonymity floor** so a sparse edge gets no live estimate and falls down the ladder, plus differential-privacy noise on published aggregates, retention limits, and coarsening on historical profiles. The tension I'd name explicitly is the one above: coverage sparsity and re-identification risk are **the same condition**, so the mechanism that protects people bites hardest where the product is weakest, and there will be sustained pressure to lower the threshold in the name of rural quality. That threshold should be owned **outside** the team whose metrics it hurts — for the same structural reason as an exploration budget or a fraud allow-through budget: a locally-optimised team erodes it one defensible increment at a time.

**9. "A new stadium opens and your ETAs are wrong every Saturday."** `F2a` `A5b`
> That's a **scheduled anomaly**, and the interesting part is that it's predictable in principle and invisible to a model trained on smooth historical averages. Three layers. **Event calendar as a feature** — stadium capacity, event start and end times — so the model can learn the pattern rather than being surprised by it weekly; this is the durable fix. **A fast manual override path** for the immediate problem, because a rule can ship in hours where a model retrain takes weeks — the same lesson as novel-harm response in moderation. And **anomaly detection on probe speeds** so that even an *unscheduled* surge is detected within minutes rather than waiting for the historical profile to absorb it over months. I'd also note the incident-lag point: incidents and events are **step changes**, and a model trained to predict smooth traffic responds to them too slowly by construction, which is an argument for keeping a non-learned fast path permanently rather than treating it as a stopgap.

**10. "Which quantile do you actually display, and who decides?"** `A3b` `I1j`
> Different surfaces get different quantiles because their loss functions genuinely differ, and I'd push back on the instinct to "just show P90 to be safe." Systematic pessimism has its own cost: users notice they always arrive early, start mentally discounting the ETA, and you've destroyed the calibration you were buying. So: displayed arrival ≈ **P50** as a best estimate; **"leave by" ≈ P85–P90** because that loss is discontinuous — you miss the meeting or you don't; flights explicitly higher; route *comparison* on expected time plus a **variance penalty**, because a route that's usually 2 minutes faster but occasionally 40 minutes slower isn't better; and where possible show the **width as a range**, which is more honest than any single number. On who decides: this is a **product decision that should be an owned, reviewable artifact**, not a constant in the serving code — because changing it changes what the system promises to hundreds of millions of people, and that shouldn't be indistinguishable from a config tweak.

---

## ⭐ THE L6 LINE

> **"An ETA is a distribution, not a number — the question is 'will I be late', which is a tail question. And our own routing changes the traffic we're predicting, so the target is an equilibrium and the evaluation is counterfactual."**

## 🧭 HOOKS — *steer by what you mention*

- *"Quantiles don't add"* → invites the modelling/statistics branch
- *"The refresh path is 2–5× the initial requests"* → invites the capacity/serving branch
- *"The online path is a lookup, not a traversal"* → invites the systems branch
- *"Our routing changes the traffic"* → invites the feedback-loop / externality branch (**good at L6**)
- *"Stability fights accuracy and has no natural owner"* → invites the org/metrics branch (**good at L6**)
- *"Privacy binds hardest where coverage is worst"* → invites the privacy branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three consumers"* — router / display / refresh regenerates the architecture.
- **Running out of time?** → *"I have five minutes — I'd rather cover the feedback loop and stability than finish the map-matching detail. Sound right?"*

**Name and defer:**
- "**Multi-modal ETA** (transit) is schedule-plus-deviation — a different problem shape."
- "**Incident detection** from probe anomalies is its own pipeline; incidents are step changes."
- "**Traffic assignment / equilibrium modelling** is the formal version of the feedback problem."
- "**The last 200 metres** — parking and walk-in — is outside our ETA but inside the user's experience."
- "**Driver heterogeneity** is irreducible variance and belongs in the distribution, not a per-driver correction."
- "**Geo-sharding by S2 cell** with cross-boundary routes is a real distributed-systems problem."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a lookup-only request path over precomputed state | §0, §4 | *"The online path is a lookup, not a traversal."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"The refresh path is 2–5× the initial requests: ~140k QPS at p95 ≤ 50 ms."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §9 | *"Summed edge costs with a deliberately widened interval — never no ETA."* |
| **A2a** | The full stack | §4 diagram | Background (all drivers) vs request path (one user), each box with its task and output |
| **A2b** | Model components | §5.3 | Supersegments + GNN embeddings + a route-level quantile model |
| **A2c** | Data sources | §5.1 | Probes, road graph, history, incidents, weather and events |
| **A2d** | Staged pipeline | §5.5 | Probes → state → embeddings → training → gate |
| **A3a** | Top-down, unprompted | §0 | *"I want to fix the output type before I draw anything."* + the plan |
| **A3b** | A principled objective for the trade-off | §2, §6 | *"Recommend the lowest E[t] + λ·σ; the asymmetry lives in the quantile choice."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Consumer · trip lengths · probe-data rules · modes |
| **A5a** | Alternatives compared | §4, FU 2 | Sum of segments vs history vs GNN-only vs route-level model |
| **A5b** | A full alternative pipeline on demand | §9 | Summed costs + widened interval + event overrides |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.4 | *"Pinball loss for P10…P95 together, non-crossing."* |
| **F1b** | Metrics and their trade-offs | §1, §2 | *"MAE is blind to the tail; calibration per slice, with sharpness alongside."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Rural sparsity, horizon, feedback loop, cleaning errors |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 7 | Sparse coverage → the ladder; timeout → widened interval |
| **F2b** | Latency too tight → tiering | §4, §6 | Scalar router, small route model, incremental refresh |
| **F2c** | Fine-tune an existing model | §5.4 | A global GNN adapted per region |
| **F3a** | Quantization / efficiency with a quality trade | §6 | A small, quantized route model; cached embeddings |
| **F3b** | Scale via precomputation | §3, §6 | Embeddings refreshed in batch; geo-sharded state |
| **F3c** | Extension, with cost and maturity risk | §9 | Other modes; LLM-read incident reports |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 per path; $ per 1,000 refreshes |
| **I1b** | Reliability | §6, §8 | The fallback ladder; shard failover |
| **I1c** | Usability | §2, §6, §8 | Stable displayed ETA; ranges; leave-by |
| **I1d** | Scalability | §3, §8 | 32 GB geo-sharded state; incremental refresh |
| **I1e** | Privacy | §5.4, §8 | k-anonymity floor; DP; *"don't quietly lower k"* |
| **I1f** | Safety | §8 | No routing through closures; less driver distraction |
| **I1h** | Security | §8 | **Spoofed traffic** — distinct-device checks |
| **I1i** | Ethics / bias | §7, §8, FU 4 | Residential externality; rural fairness |
| **I1j** | Explainability | §8, FU 10 | "Why this route"; delay reasons; who owns the quantile choice |
| **I2a** | Offline → online, correcting for our own influence | §7, FU 6 | *"Offline evaluation is counterfactual"* → simulator → region-level A/B |
| **I2b** | The success definition | §7 | Calibrated per slice, lateness down, stable, induced flow held |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
