# G31 · Maps ETA Prediction — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G31_eta_prediction_intv.md`](G31_eta_prediction_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **Fixed in the sheet from these:** §6 `E[t]` was never defined (now: expected travel time + λ·σ, ranks the candidate routes) · §4 the calibration row now says *90% of trips* · §5 the stability row now says what is bounded and how.

**Jump:** [1 route comparison + pinball alpha](#1--route-comparison-variance-penalty-and-pinball-alpha) · [2 ⭐ why eval is counterfactual](#2--why-evaluation-is-counterfactual) · [3 stability vs accuracy](#3--why-stability-fights-accuracy) · [4 calibration: 90% of what?](#4--quantile-calibration-90-of-what) · [5 ETA vs displayed, bounded updates](#5--eta-vs-displayed-arrival-and-bounded-updates) · [6 what is E[t]](#6--what-is-et) · [7 ⭐ map-matching + Viterbi](#7--map-matching-with-an-hmm-and-viterbi) · [8 ⭐ the architecture step by step](#8--the-architecture-step-by-step)

---

## 1 · Route comparison, variance penalty, and pinball alpha

**"Routing" here = choosing which of the ~3 candidate routes to recommend.** The router (A*/CH) finds the candidates; the route-level model predicts each one's travel-time distribution; then something must **rank** them.

**Variance penalty = risk aversion.** Rank by mean time *plus* a charge for unpredictability:

```
score(route) = E[t] + λ·σ[t]        E[t] = mean travel time, σ = spread, λ = how risk-averse

Route A: mean 25 min, σ 2   (reliable highway)
Route B: mean 23 min, σ 10  (side streets: usually 20, sometimes 50)

λ = 0   → A 25, B 23   → pick B  (pure mean, ignores the 50-minute days)
λ = 0.5 → A 26, B 28   → pick A  ✓
```

> ⭐ A route that's usually 2 minutes faster but occasionally 40 minutes slower is **not** better. The penalty encodes that.

**Is it pinball loss with α < 0.5? No.**

| | What it is | Where it's used |
|---|---|---|
| **α < 0.5** (e.g. P10) | A **low, optimistic** quantile | The bottom of a shown range ("24–31 min") |
| **Variance penalty** | Uses the **width** of the distribution, e.g. σ ≈ (P90 − P10) ÷ 2.56 | Ranking routes |
| **E[t]** | The **mean**, which is not a quantile | Ranking routes |

Pinball loss trains quantiles, and the mean isn't one. Get it by **averaging evenly spaced quantiles** (P5, P15 … P95 ≈ E[t]), or add a separate head trained with MSE.

*(Ranking by one high quantile, e.g. P70, does almost the same job: for a roughly bell-shaped distribution a quantile ≈ mean + z·σ.)*

**"Leave by" = α ≈ 0.85–0.90: yes.** Pinball loss at α = 0.9:

```
L(y, q) = 0.9 · (y − q)    if actual y > prediction q   (you were LATE: costly)
          0.1 · (q − y)    if actual y ≤ q              (you were early: cheap)
→ being late costs 9× being early → q is pushed up until 90% of outcomes fall below it = P90
```

⚠ **Nuance:** travel time depends on **when you leave**, so "leave by" is solved backwards: find the **latest departure d** such that `d + P90(travel time | depart at d) ≤ deadline`. That's a small search over departure times, not one prediction.

---

## 2 · Why evaluation is counterfactual

**Evaluating an ETA model = compare predictions to actual travel times.** The problem: **the actual travel times were produced partly by our own routing**, which came from the deployed model. A different model would have routed people differently, so the traffic, and the "actuals", would have been different.

```
WORLD UNDER MODEL A (deployed): A doesn't send people down Elm St → Elm St stays fast (8 min)

Offline replay of MODEL B on A's logs:
   B predicts Elm St = 8 min  →  log says 8 min  →  B looks PERFECT ✓

Actually ship B: B routes 5,000 cars down Elm St → Elm St jams → 25 min
   → B's own prediction is now WRONG ✗
```

> ⭐ **The question we care about is "how accurate is B in the world B creates?" That world doesn't exist in our logs. It's a counterfactual.**

It biases both ways: a model that spreads traffic well can **look worse** offline (it predicts congestion the deployed model caused, on roads it would have relieved).

**Same pattern as elsewhere:** *you only observe what you did*. Travel times exist only for routes people actually drove.

**What to do instead:**
- **Region-level A/B** (whole metros per arm) so the interference stays inside an arm. User-level A/B is contaminated: treatment's cars clog control's roads.
- **Switchbacks:** alternate policies by hour or day in the same region.
- **A traffic simulator** (traffic assignment), whose offline↔online correlation you validate yourself.

---

## 3 · Why stability fights accuracy

Every in-trip refresh (every 10–30 s) brings new traffic data, so **each new ETA is individually more accurate**. Show every one and the user sees:

```
6:24 → 6:31 → 6:26 → 6:29 → 6:25 → 6:30      ← each "more accurate", the product WORSE
```

**Why that's worse than a slightly stale number:**
- **People act on the number.** They text "be there at 6:25" or decide whether to call ahead. A number that keeps moving can't be acted on.
- **Jitter reads as "the app doesn't know".** Users lose trust in *all* ETAs, including the accurate ones.
- **Most of the changes are noise.** ±2 min on a 3-hour drive carries no information worth interrupting for.

**So stability is designed in (hysteresis):**
- **Dead-band:** don't change the display unless |Δ| > a threshold that **scales with time remaining** (e.g. max(1 min, ~5% of remaining)). 2 min matters on a 15-min trip and is noise on a 3-hour one.
- **Asymmetric:** show **worse** news promptly (it's actionable: call ahead, reroute); absorb **better** news slowly (nothing to act on).
- **Avoid "remaining time going up" repeatedly**, the single most damaging pattern.

⚠ **Why it "fights":** the displayed number is now sometimes a few minutes stale, so **measured accuracy drops** while the product improves. Without a stability metric someone owns, the accuracy dashboard wins that argument every time.

---

## 4 · Quantile calibration: 90% of what?

**90% of trips.** Each trip gets its own predicted P90 (31 min for one, 12 for another). Calibration asks whether the actual time lands at or below **that trip's own P90** about 90% of the time:

```
10,000 completed trips, each with its own predicted P90:
   count trips where  actual travel time ≤ that trip's predicted P90

   ≈ 9,000 (90%)  → P90 is CALIBRATED ✓
     7,500 (75%)  → P90 too LOW: overconfident → "leave by" makes people late
     9,800 (98%)  → P90 too HIGH: too pessimistic → people leave needlessly early
```

**"Quantile calibration" = calibration of the predicted travel-time distribution.** Repeat for **every** quantile the model emits (P10 → 10%, P50 → 50%, P90 → 90%) and plot predicted vs observed: the **reliability curve**. On the diagonal = calibrated.

⚠ **Two checks on top:**
- **Per slice** (urban/rural, trip length, horizon, peak/off-peak). Global calibration can hide rural at 70% offset by urban at 95%.
- **Sharpness.** Calibration alone is gameable: a uselessly wide P10–P90 of 5 to 120 min can still be calibrated. Report interval width or the average pinball loss alongside (same lesson as G21's constant model).

---

## 5 · ETA vs displayed arrival, and bounded updates

**Not quite the same thing:**

| Term | What it is |
|---|---|
| **Predicted travel time** | The model's output: a **distribution** of remaining time (P10…P95) |
| **ETA** (estimated time of arrival) | Clock time = now + remaining travel time (usually P50) |
| **Displayed arrival time** | The ETA **after surface logic and hysteresis**: what's on screen |

The stability SLO is about the **displayed** number. The model's internal ETA can move every 10 s; the screen shouldn't.

**"Bounded update magnitude and frequency"** means two limits:

```
MAGNITUDE (dead-band): only change the display if |new − shown| > threshold
     threshold = max(1 min, ~5% of time remaining)
     → 15-min trip: changes < 1 min are ignored
     → 3-hour trip: changes < ~9 min are ignored

FREQUENCY (rate limit): at most one display change per N minutes,
     UNLESS the change is big (e.g. an incident adds 15 min → show now)

MEASURED AS: display changes per trip · total |Δ| per trip
     · "times the remaining ETA went UP" per trip
```

---

## 6 · What is E[t]?

**E[t] = the expected (mean) travel time** of a route: the average over its predicted distribution. Used with a spread penalty to **rank candidate routes** (§1):

```
route score = E[t] + λ·σ[t]
```

*(Fixed in the sheet: the metrics block and the opening now define it.)*

---

## 7 · Map-matching with an HMM and Viterbi

**The job:** turn a noisy GPS trace into **"which road segments did this car actually drive, and when?"** Only then can you compute per-segment speeds.

**Why it's hard:** GPS error is ±5 m in the open and ±20–50 m in urban canyons (multipath). A highway and its frontage road are ~20 m apart with very different speeds. **Snapping each point to the nearest road** gets these wrong, and a misattributed trace poisons **both** roads' speed estimates.

### The HMM framing

| HMM piece | In map-matching |
|---|---|
| **Hidden state** | Which road segment (and position on it) the car is really on |
| **Observation** | The GPS fix |
| **Emission P(GPS \| segment)** | How close the fix is to that segment: ∝ exp(−d² / 2σ²), d = perpendicular distance, σ ≈ 5–20 m (larger in canyons) |
| **Transition P(seg_i → seg_j)** | Is that move physically plausible? Compare the **driving distance** between the two candidates with the **straight-line distance** between the two GPS fixes: close → plausible; a big detour, no connection, or an impossible speed → ≈ 0 *(Newson & Krumm, 2009)* |

### Why Viterbi

**Viterbi finds the single most likely *sequence* of hidden states given *all* the points**, by dynamic programming:

```
best[t][s] = max over s' of  best[t−1][s'] × P(s' → s) × P(gps_t | s)    (keep a back-pointer)
at the end: take the best final state, follow the back-pointers → the whole path

cost: O(T · K²) with K ≈ 5–10 candidate segments within ~50 m of each fix
      vs K^T for brute force over every possible path
```

**The worked example: the parallel-road trap.**

```
HIGHWAY   ══════════════════════════════════════╗════════════►
                                                 ╚══ exit ramp ══►
FRONTAGE  ────────────────────────────────────────────────────────►

GPS:             •1          •2          •3              •4   (on the ramp)

emission:   hwy .45 / fr .55 at each of 1–3 (frontage slightly closer)   ramp .9 at 4
transition: hwy → ramp HIGH      frontage → ramp ≈ 0 (no connection)

GREEDY (nearest road):  frontage, frontage, frontage, ramp  ← an impossible path
VITERBI:                highway,  highway,  highway,  ramp  ← point 4 RE-DECIDES 1–3
```

⭐ **Why Viterbi and not something simpler:**
1. **Evidence from later points fixes earlier ones.** One point is ambiguous; the sequence isn't. Greedy can't look ahead.
2. **It combines both evidence types** (closeness and plausibility) in one score.
3. **It's exact and cheap:** a global optimum in O(T·K²), not K^T.
4. **It's robust to outliers.** One wild canyon fix with a bad emission gets outvoted by consistent transitions.
5. **It returns one coherent path.** Forward-backward gives per-point probabilities, which can produce a path that hops between roads. We need one drivable path.

**Live traffic variant:** run a **sliding-window (fixed-lag) Viterbi**, which commits points a few fixes behind real time, so speeds are available within seconds.

**After matching:** timestamps along the matched path → **traversal time per segment** → speed observations. Then clean them:

- ⚠ **Stopped ≠ stuck**, the biggest single cleaning rule. A car parked or at a coffee stop looks exactly like a jam. Signals that separate them: **other probes on the same segment are moving** (→ just this car stopped); the stop is **off the carriageway** (shoulder, lot); it's **at the end of a trip**; or it's **very long** with no stop-and-go pattern. Treat it as congestion by mistake and you route traffic **away** from a free-flowing road.
- **Mode confusion:** a pedestrian or cyclist on a road looks like crawling traffic. Filter by speed profile and activity recognition. Buses stop at stops.

---

## 8 · The architecture, step by step

```
 GPS PROBES ──► ① MAP-MATCHING ──► ② TRAFFIC STATE ◄── INCIDENTS · WEATHER · EVENTS
                                      │        │
                          ③ EDGE-COST TABLE   ④ GNN → SUPERSEGMENT EMBEDDINGS
                                      │        │
              request ──► ⑤ ROUTER ──(3 routes)──► ⑥ ROUTE-LEVEL MODEL → quantiles
                                                        │
                                               ⑦ SURFACE LOGIC
                                                        │
                                   ⑧ FIRST RESPONSE  ·  ⑨ IN-TRIP REFRESH
                                                        │
                    drivers follow our routes → new GPS probes → back to ① (THE FEEDBACK LOOP)
```

### ① Map-matching → speed observations
Phones with location on (opted in) send GPS traces. HMM/Viterbi (§7) maps them onto road segments → traversal time per segment → cleaned (stopped ≠ stuck, mode filtering). **Output: a stream of "segment X took 42 s at 17:03" observations.** It's the largest CPU line in the pipeline.

### ② Traffic state: per edge, speed + confidence
For every road segment ("edge" of the road graph), aggregate the recent observations over a rolling window (e.g. last 5–10 min):
- **speed:** a robust average (median, so one weird car doesn't swing it)
- **confidence:** how many distinct devices, how recent, how much they disagree

Incidents (closures, crashes), weather and events are folded in here: a closed road gets ∞ cost. Size: ~10⁹ edges × ~32 B ≈ 32 GB, held in RAM and geo-sharded by S2 cell.

⚠ **k-anonymity gate:** if fewer than **k distinct devices** contributed in the window, **publish no live estimate** for that edge, because 2–3 traces on a rural road can identify a person. The edge falls down the **fallback ladder**, and each rung **widens the interval**:

```
live observed (≥ k devices) → GNN-propagated from neighbours → historical profile
   (time-of-day × day-of-week) → road-class prior (speed limit, geometry)
```

Traffic state feeds **two consumers** with very different needs: ③ and ④.

### ③ Edge-cost table: one scalar per edge, ~1 µs
For the router: **expected traversal time per edge** (live where confident, else historical), plus turn costs. It's **a plain array lookup**, not a model call.

**Why so cheap:** a route search touches a huge number of edges. Plain Dijkstra/A* on a continental graph relaxes **millions**. Even with Contraction Hierarchies it's thousands. A neural net at ~1 ms per call × thousands of calls blows the 100 ms budget a hundred times over. **So no NN inside the search.**

*Detail worth knowing:* CH needs heavy preprocessing, but traffic changes weights every minute. Production systems use a **customizable** variant (CCH / CRP): preprocess the road **topology** once (hours), then **re-customize the weights** in seconds when traffic updates.

### ④ GNN over the road graph → supersegment embeddings
- **Supersegment** = a chain of consecutive segments (hundreds of metres to a few km) that share traffic behaviour, **including the junctions and turns inside it**. That's how "time that belongs to no segment" gets a home.
- **The GNN:** nodes = supersegments; edges = road connectivity; node features = current speeds + confidence, historical profiles, road attributes (lanes, class, limit), incidents. **Message passing** over k hops lets each node "see" its neighbourhood: a jam 2 km downstream shows up in this node's embedding before it arrives. It also fills sparse roads from well-observed neighbours (rung 2 of the ladder).
- **Output:** an embedding per supersegment, **precomputed** every minute or so and cached. At serving time it's a **lookup**, never a graph traversal. *(Google Maps + DeepMind published this design in 2021.)*

### ⑤ Router (A*/CH): find ~3 candidate routes
Given origin, destination and departure time, search the graph over **③'s scalar costs** and return the best route plus ~2 good, genuinely different alternatives (the **"3 rts"** arrow). For long trips the costs are **time-dependent**: an edge 2 hours out uses the cost expected *when you'll reach it*.

**The router only finds candidates. It doesn't produce the final ETA.** Summing scalar edge costs gives a rough total, which is wrong for the reasons in sheet §5 (correlation, unattributed time, quantiles don't add).

#### 📊 Where A*/CH sits: three clocks, one box

```
════════ OFFLINE · once per map release (hours) ═════════════════════════════════════
  ROAD MAP (topology: ~10⁹ edges, no traffic)
        │
        ▼
  CH PREPROCESSING: rank nodes (residential → motorway), add SHORTCUT edges
        │  output: the hierarchy STRUCTURE (which shortcuts exist)
        ▼
════════ EVERY ~1 MIN · as traffic changes (seconds) ═════════════════════════════════
  ② TRAFFIC STATE ──► ③ EDGE-COST TABLE  (one scalar per edge: "edge 8812 = 42 s")
                              │
                              ▼
                     CUSTOMIZE: recompute every shortcut's weight from the new
                     edge costs (CCH/CRP), so the hierarchy reflects live traffic
                              │
════════ PER REQUEST · online (~ms) ══════════════════════════════════════════════════
                              ▼
  "SF → San Jose,      ┌─────────────────────────────────────────────┐
   leave now"   ─────► │ ⑤ ROUTER = the A*/CH QUERY                  │
                       │   bidirectional search, climbs the hierarchy │
                       │   ~thousands of edge-cost LOOKUPS (~1 µs)    │
                       │   ⚠ no neural net in this loop               │
                       └──────────────────────┬──────────────────────┘
                                              │ 3 candidate routes
                                              │ (each = an ordered list of edges)
                                              ▼
                       ┌─────────────────────────────────────────────┐
                       │ ⑥ ROUTE-LEVEL MODEL                          │
                       │   edges → supersegments → look up ④'s        │
                       │   embeddings → P10…P95 per route             │
                       └──────────────────────┬──────────────────────┘
                                              ▼
                                ⑦ SURFACE LOGIC → recommend 1, show ETA
```

**What happens inside the ⑤ box for one request:**

```
 origin ●─residential─●─arterial─●═══ MOTORWAY 101 ═══●─arterial─●─residential─● dest
        └── forward search climbs ──┘                 └── backward search climbs ──┘
                                  meet on the motorway level → best path
        repeat with penalties on the best path's edges → 2 genuinely different alternatives
```

**One analogy:** the router is a **fast shortlister** (cheap scalar costs, thousands of lookups, picks 3 routes out of billions of possible paths). The route-level model is a **careful judge** (rich embeddings and a real model, but only 3 routes to score). That split is why the NN can be expensive: it never runs inside the search.

### ⑥ Route-level model: the actual ETA distribution, per route
For each of the 3 routes:
1. **Look up** the embeddings of its supersegments, in order (from ④).
2. **Blend live vs historical per supersegment, by arrival horizon.** A supersegment you'll reach in 5 min trusts live traffic. One you'll reach in 2 hours trusts the historical forecast (live info decays after ~15–45 min). It's **two passes**: estimate arrival times → set blend weights → re-estimate.
3. **Predict the whole route's distribution jointly:** a sequence model over the supersegments, plus route features (turn count, departure time), outputs **P10 … P95 together** (multi-quantile head, pinball loss, constrained so quantiles don't cross). Predicting jointly is what captures correlation.

This is the "heavy" model, and it runs on only 3 routes per request.

### ⑦ Surface logic: which number to show where
The model emits a distribution. The **product** picks from it:

| Surface | Uses |
|---|---|
| Which route to recommend | **E[t] + λ·σ** across the 3 routes |
| Displayed arrival | **≈ P50** |
| "Leave by" | **P85–P90** (solved backwards over departure time) |
| Flights / hard deadlines | **P95+** |
| Shown range | e.g. **P10–P90** → "24–31 min" |

It lives outside the model, so it's explicit, auditable, and changeable per surface.

### ⑧ First response
Return routes + displayed ETAs + range to the phone. Budget **p95 ≤ 100 ms** (~70k QPS at peak).

### ⑨ In-trip refresh: the volume driver (~140k QPS)
While you drive, the phone reports position every 10–30 s. Each refresh:
1. Map-match your current position on the known route.
2. Keep only the **remaining** supersegments; look up their **fresh** embeddings.
3. Re-run the route model on the **remainder**. This is an **update**, not a recompute, because the route is already known. It's the cheapest path in the system.
4. **Hysteresis** (§3, §5) decides whether the **displayed** ETA actually changes.
5. Check whether an alternative is now clearly better → offer a reroute if the gain clears a threshold.

**Why it's 2–5× the initial requests:** ~2×10⁸ navigations/day × ~20 min × an update every 10–30 s ≈ 10¹⁰/day.

### The loop closes
Drivers follow our routes → their GPS becomes tomorrow's probes → our recommendations **shape the traffic we predict**. So the target is an **equilibrium**, evaluation is **counterfactual** (§2), and we monitor **induced-flow share** (how much of an edge's traffic we put there).

> ⭐ **Say:** *"Two consumers of one traffic state: the router needs a scalar per edge in microseconds, so it gets a lookup table. The ETA needs a calibrated distribution for 3 routes, so it gets precomputed GNN embeddings and a route-level quantile model. Everything on the online path is a lookup, not a traversal."*

---

## 🔁 THE EIGHT THINGS I KEPT GETTING WRONG

1. **"Routing" = ranking the ~3 candidates** by **E[t] + λ·σ**. That isn't a low-α quantile: P10 is the bottom of the shown range. "Leave by" is α ≈ 0.85–0.9, solved backwards over departure time.
2. ⭐ **Offline eval replays the world the *deployed* model created.** A new model's accuracy in the world *it* would create is unobservable: counterfactual. Use region-level A/B and switchbacks.
3. **Each refresh is more accurate, but showing every one makes the product worse.** Dead-band scaled to remaining time, asymmetric (bad news fast, good news slow).
4. **Calibration = 90% of trips land at or below their own P90**, checked for every quantile and per slice, with sharpness alongside.
5. **ETA ≠ displayed arrival.** The model's ETA moves every 10 s; the displayed one is bounded in **magnitude** (dead-band) and **frequency** (rate limit, unless the change is big).
6. **E[t] = mean travel time.** Not a quantile: average the quantiles or add an MSE head.
7. ⭐ **Viterbi = the most likely *sequence* of road segments.** Emission = closeness, transition = drivable plausibility. Later points re-decide earlier ones, which is what beats parallel roads. O(T·K²).
8. ⭐ **Two consumers of one traffic state:** router → scalar lookup table (no NN in the search); ETA → GNN embeddings + route-level quantile model on 3 routes. Refresh is incremental and is the volume driver. k-anonymity sends sparse edges down a ladder that widens the interval.
