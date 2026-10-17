# G23 · Ads Evaluation Framework — **INTERVIEW SHEET**

> **Tier 3 · #30** — the Ads deep dive on experimentation. The interference argument is the whole question.
> Full study file: [`../G23_ads_evaluation_framework.md`](../G23_ads_evaluation_framework.md) · [diagrams](../G23_ads_evaluation_framework_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the teams see · (c) the framework · (d) datasets · (e) building the estimators' data · (f) the estimators and their objectives · (g) running experiments: power and duration · (h) how you know the framework works · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Does it affect allocation or bidding? Advertiser adaptation time? Throughput? Is budget isolation possible? (§1) |
| **Explainability** is easy points | Every decision recorded with **rung, MDE, interval, offline prediction and realised result** (§2, §8) |
| **Security** beyond the gateway | **Invalid traffic** contaminating results · assignment integrity · metric definitions under access control (§8) |
| **Troubleshoot before being asked** | Raise **underpowered geo tests** and **experiments interfering with each other** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **"Run an A/B test" does not work in an auction** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · rung selection as a gate · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + THE THREE CHANNELS** | **(b)**, (h) | Competition · budget · learning — and the bias's direction | I1c, F1b, I2b |
| **9–12** | **THE LADDER + POWER** | — | ⭐ Which rung · why geo tests cost so much · CUPED arithmetic | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Rung gate · offline screen · assignment · analysis · the correlation record | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Propensity-logged auctions · IPS/DR · auction replay · calibration | A2b–d, F1a, F1c, F2c |
| **32–38** | **RUNNING EXPERIMENTS** | **(g)** | Variance reduction · settling time · SRM checks · conflicts | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | ⭐ **Offline↔online correlation** · advertiser adaptation · holdbacks | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Incrementality, advertiser-side tests · **switchback fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"I want to challenge the premise most answers start from, because it's the whole question: 'run an A/B test' does not work in an auction.**
>
> **User-level randomisation assumes the arms are independent. In a marketplace with a shared auction and shared advertiser budgets, they are not. If treatment bids more aggressively it wins impressions control would have won, it consumes budgets control was also drawing on, and it changes the competitive density control experiences.**
>
> **So the measured difference contains an interference term that will not exist once treatment is at 100% — and the direction is predictable: naive A/B systematically over-states any change that affects allocation or bidding, because treatment is partly winning by taking from control rather than by being better."**

**Then the two structural moves:**
1. **⭐ Not one method — a ladder** — *"offline counterfactual estimation for screening; user-level A/B only for changes that genuinely don't touch the auction; **geo or cluster randomisation** for anything that changes allocation, because each market is then a self-contained auction; switchbacks where geography is unavailable; and a long-horizon holdback as the slow ground truth. The engineering judgement is knowing which rung a given change belongs on."*
2. **⭐ The framework's own metric** — *"The framework's job is not to run experiments. It's to **make the offline metric predict the online outcome** — and that's itself measurable: track, across many past experiments, the correlation between predicted offline lift and realised online lift. **Improving offline–online correlation multiplies the team's iteration rate**, and I've rarely seen it treated as a first-class metric even though it determines how fast everything else moves."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what teams see and the three interference channels, the method ladder and its power arithmetic, the framework's architecture, the data and estimators, running experiments well, how the framework measures itself, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Does the change affect allocation or bidding?** *(⭐ **This decides which rung of the ladder** — and it's the only question that matters first.)*
2. **How long do advertisers take to adapt?** *(Their bid response is a second, slower interference channel.)*
3. **What's the experiment throughput we need?** *(Geo tests are expensive and slow — throughput bounds the whole team's velocity.)*
4. **Can the platform isolate budgets per arm?** *(If yes, one interference channel disappears — FU 6.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Marketplace | A search-ads auction: millions of advertisers, shared daily budgets, automated bidding |
| Changes | Ranking models (G21/G22), auction and reserve-price changes, pacing, UI |
| Markets | ~200 geo units usable for randomisation |
| Teams | Dozens of teams, hundreds of candidate changes a quarter |
| Logging | Auctions logged with **propensities** from a small randomised exploration slice |

**FR:** offline counterfactual estimation · multiple randomisation units (user, geo, cluster, time) · **budget isolation** where possible · variance reduction · long-horizon holdbacks · **offline↔online correlation tracking**.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Correct rung selection** | A gate in the experiment-design review, not a suggestion |
| **Offline↔online correlation** | **Tracked as the framework's own KPI** |
| Experiment throughput | Bounded by geo-test capacity — and that's a team-velocity constraint |
| Minimum detectable effect | Stated **per method** — geo tests have far less power |
| Holdback | Long-horizon, continuously maintained |

**Scope / NON-GOALS:**
- ❌ **User-level A/B as the default.** Position, with the interference mechanism.
- ❌ **Reporting a naive lift for an allocation-affecting change.** It's biased upward, predictably.
- ❌ The ranking models themselves (G21/G22) · the auction mechanism · budget pacing (G24).

---

## 2 · PRODUCT + THE THREE CHANNELS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the teams see (b):**

```
EXPERIMENT DESIGN REVIEW · "pCVR model v8 in the auction"
  Affects allocation or bidding?  YES  → user-level A/B NOT ALLOWED
  Recommended rung: GEO, 100 + 100 markets, matched pairs, CUPED
  Offline screen (DR):  +1.9% revenue (CI +0.8 … +3.0)
  MDE at 80% power: 1.2% (with CUPED) · duration: 4 weeks (settling time)
  Conflicts: none in the bidding layer this month

RESULT · week 4
  revenue +1.4% (CI +0.3 … +2.5) · advertiser ROI flat · ad load flat
  offline predicted +1.9% → recorded for the correlation tracker
  Ship, with a 6-month holdback: "+1.4% at 4 weeks; long-run TBD"
```

```
⚠ WHY USER-LEVEL A/B BREAKS IN A MARKETPLACE — three channels:

 1. COMPETITION — treatment bids more aggressively, WINS IMPRESSIONS
    CONTROL WOULD HAVE WON. Control's measured performance drops
    because treatment took from it, not because treatment is better.

 2. BUDGET — the arms SHARE advertiser budgets. Treatment spends a
    campaign's daily budget earlier; control then sees that advertiser
    absent from auctions it would otherwise have competed in.

 3. LEARNING — shared models and pacing controllers see BOTH arms'
    traffic and adapt to the mixture, so neither arm is running
    against the system it would face at 100%.

⭐ THE DIRECTION IS PREDICTABLE, WHICH IS THE USEFUL PART:
   naive A/B SYSTEMATICALLY OVER-STATES any change that affects
   allocation or bidding — treatment is partly winning BY TAKING
   FROM CONTROL.
   → so the bias isn't random noise you can average away; it's a
     consistent upward distortion, and shipping on it means shipping
     changes whose measured lift WILL NOT MATERIALISE at 100%.
```

**The metric stack — for the framework itself:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Offline↔online correlation** across past experiments | Sets everyone's iteration rate |
| Decision quality | Share of shipped lifts that **held in the long-horizon holdback** | Did we ship things that work? |
| Velocity | Time to decision · experiments per quarter | Throughput |
| **Guardrails** | Rung violations = 0 · SRM failures caught · advertiser ROI and ad load in every readout | Integrity |

---

## 3 · THE LADDER + POWER *(min 9–12)* — **⭐ know which rung** `A1b` `I1a` `I1d`

```
 RUNG              WHEN                        BIAS      VARIANCE  COST
 ─────────────────────────────────────────────────────────────────────
 1. OFFLINE        screening, ranking-only     high      low       ~free
    counterfactual  changes; replay logged
    (IPS / DR)      traffic
 2. USER A/B       ONLY if the change genuinely low*     low       cheap
                    does NOT touch allocation
                    or bidding   (*low IF that holds)
 3. ⭐ GEO /        any change to allocation    low       HIGH      expensive
    CLUSTER         or bidding — each market is           (few units)
                    a SELF-CONTAINED AUCTION
 4. SWITCHBACK     where geography is          medium    medium    medium
                    unavailable; alternate the
                    policy over time windows
 5. LONG-HORIZON   the slow ground truth;      lowest    highest   ongoing
    HOLDBACK        catches advertiser
                    adaptation

⚠ THE ENGINEERING JUDGEMENT IS RUNG SELECTION, and the deciding
  question is exactly one: DOES THIS CHANGE AFFECT ALLOCATION OR
  BIDDING? If yes, rungs 1–2 are invalid no matter how convenient.

⭐ THE POWER ARITHMETIC — why CUPED is load-bearing:
   MDE ≈ (z_α/2 + z_β) · σ · √(2/n) ≈ 2.8 · σ · √(2/n)
   100 markets per arm, between-market σ ≈ 10%
     → MDE ≈ 2.8 × 10% × 0.14 ≈ 4%      ← too coarse to be useful
   CUPED with pre-period correlation ρ = 0.95 shrinks σ by √(1−ρ²) ≈ 0.31
     → MDE ≈ 1.2%                       ← feasible
   (user-level A/B with millions of users detects ~0.1%; that MDE
    does NOT transfer to the geo test)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Method | User A/B for everything vs **a ladder with rung selection as a gate** | Interference makes user A/B biased for allocation changes |
| Budgets | Shared across arms vs **isolated per arm** where possible | Removes the budget channel structurally |
| Analysis | Bespoke per experiment vs **one analysis layer** (CUPED, cluster-robust SEs, sequential tests) | Few geo units make variance reduction mandatory |
| Concurrent tests | Free-for-all vs **orthogonal layers + a conflict policy** | Two bidding changes are never independent |
| Offline metric | Trusted by habit vs **calibrated by tracked correlation** | Decorative offline metrics cap iteration speed |

### The system — *draw this in 90 seconds* `A2a`

```
 PROPOSED CHANGE
       │
┌─ 1. ⭐ RUNG SELECTION — a GATE in experiment design ────────────
│ "does this affect allocation or bidding?" — answered by the
│ framework from the change's declared layer, not by the author
│ OUTPUT: the allowed rungs
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. OFFLINE COUNTERFACTUAL (screening) ────────────────────────
│ replay logged auctions · IPS / doubly-robust
│ ⚠ needs PROPENSITY logged at decision time
│ OUTPUT: predicted lift with an interval — survivors only go on
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. ASSIGNMENT SERVICE ────────────────────────────────────────
│ unit: user · GEO · cluster · time-slice · orthogonal layers
│ ⚠ BUDGET ISOLATION where the platform allows (removes channel 2)
│ OUTPUT: logged assignments + a sample-ratio check
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. ANALYSIS LAYER ────────────────────────────────────────────
│ CUPED / pre-period covariates · matched pairs · synthetic control
│ cluster-robust SEs · sequential testing · MDE PER METHOD
│ OUTPUT: effect + interval on revenue, advertiser ROI, user metrics
└──────┬─────────────────────────────────────────────────────────
       ▼
 DECISION  →  ⭐ RECORD (offline prediction, online result)
       ▼
┌─ 5. ⭐ THE FRAMEWORK'S OWN KPI ─────────────────────────────────
│ OFFLINE↔ONLINE CORRELATION across experiments, per change class
│ weak ⇒ the offline metric is DECORATIVE and everyone iterates
│ at the speed of geo tests
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Underpowered geo tests:** *"With tens of units, an MDE borrowed from user-level tests turns a real effect into 'no effect'. The MDE is stated per method."*
- **Experiments interfering with each other:** *"Two bidding changes on the same traffic bias each other. Allocation changes are serialised or tested as a combination."*
- **Advertiser adaptation:** *"A two-week result measures a transient; the holdback measures where bidders settle."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Geo randomisation for allocation changes** | "Each market is a self-contained auction, so interference is *inside* the unit rather than between arms" | Far fewer units ⇒ much less power, much slower, much more expensive |
| **2** | **Budget isolation per arm** | "It removes the budget interference channel structurally — the highest-value infrastructure investment in this framework" | Real platform work; not always possible |
| **3** | **Offline↔online correlation as the KPI** | "The framework's job is to make the offline metric predict the online outcome — improving that multiplies the team's iteration rate" | It's a meta-metric; hard to get funded |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Auction logs with propensities** | Bids, allocations, prices — the replay substrate | Without a randomised exploration slice, IPS has no support |
| **Clicks and conversions** | Outcomes | Conversions are delayed (G22) |
| **Budgets and pacing logs** | Who was budget-constrained, and when | Needed to model channel 2 |
| **Market metadata and pre-period metrics** | CUPED covariates, matched pairs | Markets are heterogeneous |
| **The experiment registry** | Past (offline prediction, online result) pairs | Only survivors of screening got online tests |
| **Invalid-traffic flags** | Clean outcomes | Fraud inflates or deflates arms |

### 5.2 Building the estimators' data (e) `F1c`

- **Keep a small randomised exploration slice** in serving, so logged propensities cover the actions a candidate might take — IPS can't estimate what was never tried.
- **Remove invalid traffic** before any outcome reaches an estimator or a readout.
- **Matured conversions only**, or censoring-corrected ones, for conversion metrics.
- **⚠ The correlation tracker has a selection problem:** only changes that **passed** offline screening get online tests, so the (offline, online) pairs cover a restricted range — and correlation on a restricted range is biased. Send a **random sample of offline-rejected changes** online too, occasionally, to see the whole range.
- **Stratify the registry** by change class (ranking, bidding, pricing, UI), since the offline metric can be good for one and useless for another.

### 5.3 The estimators and their objectives (f) `A2b` `F1a`

| Estimator | Task | Form |
|---|---|---|
| **IPS** | How would a new policy have done on logged traffic? | V̂ = (1/n) Σ rᵢ · π_new(aᵢ\|xᵢ) / π_log(aᵢ\|xᵢ) |
| **Doubly robust** | The same, with lower variance | A reward model's prediction + an IPS correction on its residual |
| **Auction replay** | Re-run logged auctions under the candidate | Bidders held fixed — so its **fidelity** must itself be measured |
| **CUPED** | Remove baseline variance | Y′ = Y − θ(X − E[X]), θ = cov(X, Y) / var(X), X = pre-period metric |
| **Synthetic control** | A counterfactual for a treated market | Weights on control markets fit to the pre-period |
| **Calibration model** | Offline lift → expected online lift, per change class | Shrinkage regression on registry pairs |

### 5.4 Fine-tune an existing model (f) `F2c`

The doubly-robust estimator needs a reward model, and the platform already has one: the production **pCTR / pCVR models** (G21/G22). Re-use them, **recalibrated on the randomised exploration slice**, rather than training a new outcome model from scratch. A bidder-response model for replay is likewise adapted from historical advertiser behaviour, not built new.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · GATE       declared layer → allowed rungs
STAGE 2 · SCREEN     DR / replay on logged auctions → predicted lift + interval
STAGE 3 · DESIGN     unit, pairs, CUPED covariates, MDE, duration ≥ settling time
STAGE 4 · RUN        assignment + SRM checks + guardrail auto-stops
STAGE 5 · ANALYSE    effect with intervals on revenue, advertiser ROI, user metrics
STAGE 6 · RECORD     (offline prediction, online result) → correlation per class
STAGE 7 · HOLD BACK  long-horizon holdback for shipped allocation changes
```

---

## 6 · RUNNING EXPERIMENTS *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
⚠ GEO TESTS ARE STATISTICALLY EXPENSIVE — say why, precisely:
   the UNIT OF RANDOMISATION IS THE MARKET, not the user.
   So N is the number of markets (tens), NOT users (millions).
   → power collapses, and markets are HETEROGENEOUS (size,
     competitive density, vertical mix), which adds variance on top.

SO VARIANCE REDUCTION ISN'T OPTIONAL HERE, IT'S LOAD-BEARING:
 · ⭐ CUPED / pre-period covariates — use each market's PRE-EXPERIMENT
   behaviour to explain away its baseline variance. Often the
   difference between a feasible test and an infeasible one.
 · MATCHED PAIRS — pair similar markets, randomise within pairs
 · SYNTHETIC CONTROL — construct a weighted composite of control
   markets that tracks the treated market pre-period
 · SWITCHBACKS — each market is its own control across time,
   ⚠ but carryover effects (pacing controllers, advertiser learning)
     bias short windows

⚠ AND STATE THE MDE PER METHOD. "We can detect a 0.5% lift" is a
  property of user-level A/B, not of the geo test you're actually
  running — and quietly reusing the wrong MDE is how teams conclude
  "no effect" from an underpowered test.

DURATION: at least the SETTLING TIME of the mechanism (pacing, bidders);
  early looks only through a pre-registered SEQUENTIAL test (FU 7)
```

- **The decision rule** `A3b`: ship when the interval's lower bound on the primary metric is above zero **at the right rung**, after the settling time, with advertiser ROI and user-experience guardrails holding — and report "+X at N weeks" until the holdback speaks.
- **Efficiency** `F3a`: offline screening sends only survivors to experiments; CUPED and matched pairs cut geo-test duration. The trade: screening can wrongly reject good changes, which is why some rejected ones are tested anyway (§5.2).
- **Scale** `F3b`: orthogonal layers let independent changes share traffic; allocation changes share a layer and are serialised.
- **Reliability** `I1b`: **sample-ratio-mismatch checks** on every assignment; automated guardrail stops; a kill switch per experiment; versioned metric definitions so readouts are reproducible.

---

## 7 · EVAL + ROLLOUT + BIAS *(min 38–44)* — probe **(h)** · **the framework's own metric** `I2a` `I2b` `F1b` `F1c`

```
⭐ THE FRAMEWORK'S JOB IS NOT TO RUN EXPERIMENTS.
   IT IS TO MAKE THE OFFLINE METRIC PREDICT THE ONLINE OUTCOME.

   MEASURE IT: across many past experiments, record
     (predicted offline lift, realised online lift)
   → the CORRELATION is the framework's KPI.

   WEAK CORRELATION ⇒ the offline metric is DECORATIVE, and everyone
     iterates at the speed of experiments — weeks for a geo test.
   ⭐ IMPROVING IT MULTIPLIES THE TEAM'S ITERATION RATE, and it's
     rarely treated as a first-class metric even though it determines
     how fast everything else moves.

LONG HORIZON — the slower interference channel:
  ⚠ ADVERTISERS ADAPT. A change that looks good in two weeks can be
    partly or wholly competed away once bidders respond to the new
    equilibrium — and that response takes weeks to months.
  → LONG-HORIZON HOLDBACKS are the only instrument that sees it,
    and like every holdback they must exist BEFORE you need them.
  → so the honest statement of a result is "+X% at two weeks,
    +Y% at six months", and those are often materially different.
```

**Rollout of a change:** offline screen → the right rung → ramp → long-horizon holdback. **Rollout of the framework:** start by back-filling the registry from past experiments, so the correlation has a history on day one.

**Success** = the correlation rising per change class, shipped lifts holding in the holdback, zero rung violations, and time to decision falling. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Interference** | Treatment takes from control — always upward | Rung selection; budget isolation |
| **Range restriction** | Correlation measured only on screened survivors | Test a random sample of rejected changes |
| **Novelty and adaptation** | Short tests measure transients | Duration ≥ settling time; holdbacks |
| **Peeking** | Repeated looks inflate false positives | Pre-registered sequential testing |
| **Market heterogeneity** | A few big markets dominate | Matched pairs; stratified randomisation |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Offline screens in hours; geo tests in weeks, cut by CUPED | Time to decision · screens per day |
| **Reliability** `I1b` | **SRM checks**, automated guardrail stops, kill switches, versioned metric definitions | SRM failures caught · reproducible readouts |
| **Usability** `I1c` | A design-review tool that **picks the rung and computes the MDE**; readouts with intervals, not point estimates | Design-review time · rung violations |
| **Scalability** `I1d` | Orthogonal layers; a conflict policy for allocation changes; registry across hundreds of tests | Concurrent experiments · conflicts caught |
| **Privacy** `I1e` | Analysis on aggregated user data; conversion data only where consent covers it; **one advertiser's results never visible to another** | Cross-advertiser access = 0 |
| **Safety** `I1f` | Guardrails on ad load and ad quality stop an experiment automatically; policy filters unaffected by any arm | Guardrail stops · harmful-ad exposure |
| **Security** `I1h` | ⚠ **Invalid traffic** filtered before analysis; **assignment integrity** (arms can't be detected or self-selected); metric definitions and readouts access-controlled and audited against p-hacking | Invalid traffic in readouts · audit trail |
| **Ethics / bias** `I1i` | Small advertisers don't bear experiment costs disproportionately (budget isolation); users not exposed to extreme ad load; advertisers told experiments run | Advertiser ROI by size in every readout |
| **Explainability** `I1j` | Every decision record: **rung, MDE, interval, offline prediction, realised result**, and "+X at N weeks" | % decisions with a full record |

> ⭐ *"An experimentation framework is production-ready when its decisions are reproducible, its estimates are honest about their power, and it can show how often it was right."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Incrementality testing** (ghost ads, PSA controls): would the conversion have happened anyway — a related but distinct measurement problem.
- **Advertiser-side experimentation:** the same interference problem from the other side of the market.
- **Auction simulation as the primary instrument** at high interference — then its fidelity is the thing to measure.

**When geo randomisation isn't possible — the switchback pipeline** `A5b` `F2a`:

```
a global change (one auction for everyone), or too few comparable markets
  → SWITCHBACK: alternate old and new policy over time windows per market
  → WASHOUT periods between windows so pacing and bidder carryover fade
  → window length ≥ the settling time; analyse with time-series methods
  → SYNTHETIC CONTROL for any market that can't switch
  → and still a long-horizon holdback where any population can be held out
  "When you can't split space, split time — and pay for the carryover."
```

**Top 3 failure modes:**
1. **Shipping an over-stated lift** — a user-level A/B on an allocation-affecting change, whose measured gain never materialises at 100%.
2. **Underpowered geo test read as "no effect"** — the MDE from a user-level test quietly reused for a test with tens of units.
3. **Decorative offline metric** — nobody tracks whether it predicts anything, so every change waits on a geo test and iteration slows to a crawl.

**Q1 ships:** **rung selection as a gate** + **offline↔online correlation tracking**.
> *"Rung selection because it's free and it stops the most expensive error — shipping on a biased estimate. And the correlation tracking because it's the metric that determines how fast everything else can move, and it only becomes useful after it has accumulated a history of experiments."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why can't you just A/B test it?"** `A5a` `F1c`
> Because **user-level randomisation assumes the arms are independent, and in a marketplace they aren't** — there are three interference channels. **Competition**: if treatment bids more aggressively, it wins impressions control would have won, so control's measured performance drops because treatment took from it rather than because treatment is better. **Budget**: the arms share advertiser budgets, so treatment spending a campaign's daily budget earlier means control sees that advertiser absent from auctions it would have competed in. **Learning**: shared models and pacing controllers see both arms and adapt to the mixture, so neither arm faces the system it would face at 100%. And the crucial property is that **the bias has a predictable direction** — naive A/B **systematically over-states** any change affecting allocation or bidding. It isn't noise you can average away; it's a consistent upward distortion, so you ship changes whose measured lift will not materialise.

**2. "So what do you do instead?"** `A2a` `A3b`
> Use a **ladder of methods** and get the rung selection right, because no single method works for everything. **Offline counterfactual estimation** — replaying logged auctions with IPS or doubly-robust estimators — for screening, which is cheap and fast and is where the iteration actually happens. **User-level A/B** only for changes that genuinely don't touch allocation or bidding, like a UI change or a latency optimisation. **Geo or cluster randomisation** for anything that changes allocation, because each market is then a **self-contained auction** and the interference happens *inside* the unit rather than between arms. **Switchbacks** where geography isn't available. And a **long-horizon holdback** as the slow ground truth. The engineering judgement is knowing which rung a change belongs on, and the deciding question is exactly one: **does this affect allocation or bidding?** If yes, the cheap rungs are invalid no matter how convenient they are.

**3. "Geo tests sound expensive. How expensive?"** `A1b` `F3a`
> Very, and the reason is worth stating precisely: **the unit of randomisation is the market, not the user**, so N is tens rather than millions — statistical power collapses. And markets are **heterogeneous** in size, competitive density and vertical mix, which adds variance on top of the small N. So variance reduction isn't an optimisation here, it's **load-bearing**: **CUPED** using each market's pre-experiment behaviour to explain away baseline variance is often the difference between a feasible test and an infeasible one; **matched pairs** of similar markets randomised within pairs; and **synthetic control**, constructing a weighted composite of control markets that tracks the treated market's pre-period. And I'd insist on **stating the MDE per method** — "we can detect a 0.5% lift" is a property of user-level A/B, not of the geo test you're actually running, and quietly reusing the wrong MDE is exactly how a team concludes "no effect" from an underpowered test.

**4. "What's the framework's own success metric?"** `I2b` `F1b`
> **Offline↔online correlation**, and this is the thing I'd organise the team around. The framework's job isn't to run experiments — it's to **make the offline metric predict the online outcome**, and that's directly measurable: across many past experiments, record the predicted offline lift and the realised online lift, and track the correlation. If it's weak, **the offline metric is decorative** and everyone iterates at the speed of experiments — which for a marketplace test is weeks. If it's strong, most changes are screened offline in hours and only the survivors consume experiment slots. So **improving that correlation multiplies the team's iteration rate**, which makes it a higher-leverage investment than almost any individual experiment. I've rarely seen it treated as a first-class metric even though it determines how fast everything else moves, and getting it onto a dashboard is usually the single most valuable thing this framework can do.

**5. "Your test showed +4% and it didn't hold at 100%. Why?"** `I2a` `F1c`
> Two candidates, and they're distinguishable. **Interference**: if the change touched allocation or bidding and you ran a user-level A/B, the +4% included a term that came from taking impressions and budget from control — that term disappears at 100% by construction, and the direction is exactly what you observed. That's the most likely explanation and it's a design error rather than bad luck. The second is **advertiser adaptation**: even a correctly-run geo test measures a two-week equilibrium, and advertisers respond to the new environment over weeks to months, competing away part of the gain. That's why the honest statement of a result is **"+X% at two weeks, +Y% at six months"** and why the long-horizon holdback exists. The diagnostic: check whether the change affected allocation (which implicates interference) and whether the decay profile matches bidder response times (which implicates adaptation) — and in either case the fix is upstream, in rung selection, rather than in the analysis.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How would you remove the budget interference channel entirely?"** `F3b` `A1c`
> **Budget isolation per arm** — give each experiment arm its own budget pool for participating advertisers, so treatment spending a campaign's budget can't starve control's auctions. That removes channel two structurally rather than correcting for it statistically, and it's the **highest-value infrastructure investment** in this framework because it's the one interference channel that's genuinely eliminable. The cost is real platform work: budgets are an advertiser-facing contract, so you're splitting a customer's stated daily budget across arms, which needs to be either proportional and invisible or explicitly agreed — and it interacts with pacing controllers that assume a single pool. It also doesn't help with **competition** (channel one), since arms still bid into shared auctions, or **learning** (channel three), since shared models still see both. So it's a partial fix that moves some allocation-affecting changes down the ladder from geo to user-level, which is enormously valuable given the cost difference — but I'd be careful not to oversell it as making user-level A/B universally valid.

**7. "An experiment is significant at day 3. Ship it?"** `F1b` `I1b`
> No, and for two reasons beyond the usual peeking problem. The statistical one: **repeated significance testing inflates false positives**, so if you're going to look early you need a **sequential testing** procedure with an appropriate spending function, decided in advance — otherwise "significant at day 3" means much less than the p-value suggests. The domain-specific one matters more: **three days is not an equilibrium**. Pacing controllers are still adapting, advertisers haven't responded, and in a geo test you're looking at a handful of markets where a single large advertiser's campaign schedule can dominate the signal. Marketplace effects have a characteristic settling time, and reading a result inside it measures the transient rather than the steady state. So my rule: **pre-register the duration based on the settling time of the mechanism**, use sequential testing if early stopping is needed for a genuine reason (a harm signal, say), and treat early significance as a reason to keep watching rather than to ship.

**8. "Two teams' experiments are running on the same traffic. Is that a problem?"** `I1d` `A3b`
> Usually yes, and it's under-appreciated. If both changes touch the auction, they **interfere with each other** as well as with their own controls — team A's more aggressive bidding changes the competitive density team B is measuring against, so both read biased results and neither knows. The standard defence is **orthogonal layering**: assign experiments to layers such that within a layer assignments are mutually exclusive, and across layers they're independent by construction. That works well for genuinely independent changes and it does **not** work when both changes affect the same mechanism — two bidding changes are not independent no matter how you randomise them. So the framework needs a **conflict policy**: changes affecting allocation are serialised or explicitly tested as a combination, and that's an experiment-design gate rather than an analysis fix. I'd also note the organisational cost, since this is the real reason it gets skipped: serialisation reduces experiment throughput, which is the team's velocity, so there's constant pressure to declare things independent that aren't — and that's another argument for the offline screening rung, which has no conflict problem at all.

**9. "Your offline↔online correlation is 0.3. What now?"** `F2a` `F1a`
> That's weak enough that the offline metric is close to decorative, and I'd treat improving it as the team's highest-leverage project rather than a side quest — because at 0.3 every change needs an expensive online test, which caps the whole team's iteration rate. Diagnosis in order: **is the offline metric measuring the right quantity?** A metric that optimises ranking quality won't predict revenue outcomes that depend on calibration and pricing. **Is the counterfactual estimation sound?** IPS has high variance when the candidate policy diverges from the logged one, and unlogged or mis-specified propensities bias it — and propensity logging is the thing that can't be retrofitted. **Is the online measurement itself noisy or biased?** If the online results are geo tests with wide intervals, a low correlation may reflect noise in the *outcome* rather than a bad predictor, which is a completely different fix. And **is it uniformly low, or low for a class of changes?** Often the offline metric predicts ranking changes well and bidding changes badly, which suggests keeping it as the screening rung for the former and not trusting it for the latter — a more useful conclusion than one aggregate number.

**10. "The business wants to ship faster. What do you actually change?"** `A3b` `I2b`
> I'd attack the **throughput constraint** rather than the statistical standards, because loosening standards on a marketplace test means shipping changes whose lift won't materialise, and that's slower in the end. Three moves, in order of leverage. **Improve offline↔online correlation**, because if most changes can be screened offline in hours, experiment slots go to the ones that genuinely need them — this is the biggest available win and it compounds. **Build budget isolation**, which moves a class of changes from the geo rung down to the user-level rung, and the cost difference there is weeks versus days. **Invest in variance reduction** — CUPED, matched pairs, synthetic control — because that's what makes a geo test feasible in three weeks instead of eight, and it's cheap analytical work relative to its effect on throughput. What I'd refuse is the tempting version: running allocation-affecting changes as user-level A/Bs because they're faster. That doesn't increase velocity, it **increases the rate of shipping things that don't work** — and since the bias is systematically upward, you'd be preferentially shipping the changes that are most over-stated, which is worse than random.

---

## ⭐ THE L6 LINE

> **"'Run an A/B test' doesn't work in an auction — the arms aren't independent, and naive A/B systematically over-states any allocation-affecting change because treatment wins by taking from control. So: a ladder of methods with rung selection as a gate, and the framework's own KPI is offline↔online correlation, because that's what sets everyone's iteration rate."**

## 🧭 HOOKS — *steer by what you mention*

- *"The arms aren't independent"* → invites the causal-inference branch
- *"The bias has a predictable direction"* → invites the statistics branch (**good at L6**)
- *"Each market is a self-contained auction"* → invites the experiment-design branch
- *"The framework's job is to make offline predict online"* → invites the team-velocity/org branch (**good at L6**)
- *"Advertisers adapt over weeks to months"* → invites the long-horizon branch
- *"Budget isolation removes a channel structurally"* → invites the infrastructure branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the three interference channels"* — competition, budget, learning regenerates the ladder.
- **Running out of time?** → *"I have five minutes — I'd rather cover offline↔online correlation than finish the variance reduction. Sound right?"*

**Name and defer:**
- "**Incrementality testing** — would the conversion have happened anyway — is a related but distinct measurement problem."
- "**Ghost ads / PSA controls** are a cleaner incrementality design where the platform supports them."
- "**Advertiser-side experimentation** has the same interference problem from the other side of the market."
- "**Sequential testing and always-valid inference** are what make early stopping legitimate."
- "**Simulation / auction replay** becomes the primary instrument at high interference, and then its own fidelity is the thing to measure."
- "**Metric sensitivity analysis** — which metrics can this framework actually move — bounds what's testable at all."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the right rung, not more tests | §0 | *"'Run an A/B test' does not work in an auction."* |
| **A1b** | Power and throughput budgets | §1, §3, FU 3 | *"100 markets per arm: MDE ≈ 4%, ≈ 1.2% with CUPED."* |
| **A1c** | Degrade without lowering quality | §4, FU 6 | *"Budget isolation moves a class of changes down the ladder."* |
| **A2a** | The full stack | §4 diagram, FU 2 | Rung gate → offline screen → assignment → analysis → correlation record |
| **A2b** | Model components | §5.3 | IPS, DR, auction replay, CUPED, synthetic control, calibration |
| **A2c** | Data sources | §5.1 | Propensity-logged auctions, conversions, pacing logs, pre-period metrics, registry |
| **A2d** | Staged pipeline | §5.5 | Gate → screen → design → run → analyse → record → hold back |
| **A3a** | Top-down, unprompted | §0 | *"Challenge the premise"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 2, 8, 10 | *"Ship when the lower bound clears zero at the right rung, after the settling time."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Allocation? Adaptation time? Throughput? Budget isolation? |
| **A5a** | Alternatives compared | §4, FU 1 | User A/B vs ladder; shared vs isolated budgets; free-for-all vs layers |
| **A5b** | A full alternative pipeline on demand | §9 | The switchback pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right estimator per question | §5.3, FU 9 | *"DR for screening; CUPED for variance; synthetic control for single markets."* |
| **F1b** | Metrics and their trade-offs | §2, FU 4, 7 | *"The bias has a predictable direction — upward."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 1, 5 | Interference, range restriction, adaptation, peeking, heterogeneity |
| **F2a** | Something fails → a fallback | §9, FU 9 | Switchbacks; per-class trust in the offline metric |
| **F2b** | Power too low → design for it | §3, §4 | MDE per method, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Production pCTR/pCVR recalibrated on the exploration slice |
| **F3a** | Efficiency with a quality trade | §6, FU 3 | Offline screening; CUPED and matched pairs |
| **F3b** | Scale levers | §6, FU 6 | Orthogonal layers; budget isolation |
| **F3c** | Extension, with cost and maturity risk | §9 | Incrementality, advertiser-side tests, simulation |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Time to decision |
| **I1b** | Reliability | §6, §8, FU 7 | SRM checks; guardrail auto-stops |
| **I1c** | Usability | §2, §8 | A design tool that picks the rung and the MDE |
| **I1d** | Scalability | §6, §8, FU 8 | Orthogonal layers + a conflict policy |
| **I1e** | Privacy | §8 | Aggregated analysis; advertiser data isolation |
| **I1f** | Safety | §8 | Ad-load and ad-quality guardrails stop experiments |
| **I1h** | Security | §8 | **Invalid traffic**; assignment integrity; audited readouts |
| **I1i** | Ethics / bias | §8 | Small advertisers; user ad load; transparency |
| **I1j** | Explainability | §2, §8 | The full decision record, "+X at N weeks" |
| **I2a** | Offline → online | §7, FU 5 | Screen → rung → ramp → holdback |
| **I2b** | The success definition | §7, FU 4, 10 | *"Correlation rising; shipped lifts holding in the holdback."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) framework → §4 · (d) data → §5.1 · (e) estimator data → §5.2 · (f) estimators → §5.3–5.4 · (g) running experiments → §6 · (h) success → §2, §7 · (i) extension → §9.
