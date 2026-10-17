# G13 · YouTube Recommendation — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G13_youtube_recommendation_intv.md`](G13_youtube_recommendation_intv.md). These are the lines that didn't land first time. Skim on a revision pass; go back to the sheet for the full answer.
>
> **⚠ Three of these were corrections to the sheet, not just explanations** — §1/§8 (regret isn't the report button), §3 (the metrics table was ambiguously formatted), §2 (the reason was missing). All three are now fixed in the sheet.

**Jump:** [1+8 regret rate](#18--what-is-regret-rate--its-not-the-report-button) · [2 why single-objective fails](#2--why-single-objective-watch-time-fails) · [3 the metrics table](#3--the-availablemeaningful-table-written-properly) · [4 long-horizon holdback](#4--what-is-a-long-horizon-holdback) · [5 offline metrics](#5--offline-metrics--is-it-ndcgmrr-on-click-data) · [6 trajectory vs slate diversity](#6--trajectory-vs-slate-diversity) · [7 the position tower](#7--how-does-the-position-tower-actually-work) · [9 two-tower vs DiskANN](#9--two-tower-vs-diskann--and-is-the-ranker-a-cross-encoder) · [10 feature fetch](#10--what-is-feature-fetch-and-of-what) · [11 hash sharding](#11--hash-sharding--is-it-for-load-balancing) · [12 near-line + fresh source](#12--what-is-near-line-what-is-the-fresh-source) · [13 what breaks](#13--the-three-what-breaks-items-explained) · [14 nDCG vs offline↔online](#14--isnt-ndcgmrr-a-better-offline-metric-and-is-offlineonline-correlation-good) · [15 measuring satisfaction](#15--engagement--satisfaction--so-how-do-you-measure-satisfaction) · [16 clickbait](#16--how-do-you-detect-and-offset-clickbait)

---

## 1+8 · What is regret rate — **it's not the report button**

⚠ **I was right to doubt this, and the sheet was loose.** There is **no "regret" button** under a video. Regret is a **survey** measure — an intermittent in-feed card asking *"Was this video worth your time?"* — and a negative response is a **regretted view**.

| | **Report** | **Survey regret** |
|---|---|---|
| Who initiates | **The user**, unprompted | **We** ask, on a sample |
| Measures | **Policy violation** | *"Was that worth my time?"* |
| Volume | Tiny, heavily biased | Sampled, representative |
| Catches | Harmful content | ⭐ **Content that's fine but wasn't worth watching** |

**Why reports can't substitute:** a video can be perfectly policy-compliant and still waste your evening — **that's the failure watch-time optimisation produces, and nobody reports it.** Reports also select hard for outrage, so they're a terrible quality distribution.

**The three real mechanisms:**
- **Survey cards** — the actual regret instrument.
- **Not interested / Don't recommend this channel** — behavioural negatives, user-initiated.
- **Report** — ⚠ **a different system entirely** (policy enforcement, G25), not quality.

> *Sheet now says "survey-measured regret" and flags that Report is G25's problem.*

---

## 2 · Why single-objective watch time fails

*(This is now spelled out under the non-goals in the sheet — it was marked "Position, with the reason" without the reason.)*

Six mechanisms, **each of them the model doing its job correctly**:

| # | Mechanism | What the model learns |
|---|---|---|
| ① | **Long-video bias** | A 3-hour video accrues more watch time than a perfect 4-minute answer. Unnormalised ⇒ *longer is better* |
| ② | **Clickbait** | A misleading thumbnail wins the click, and a partial watch still accumulates time |
| ③ | **Engagement ≠ satisfaction** | Outrage and anxiety hold attention very effectively ⇒ optimise toward *activating*, not *valuable* |
| ④ | **Rabbit holes** | A progressively narrowing sequence maximises session time. Each step is locally optimal; **the trajectory is the harm** |
| ⑤ | **Homogenisation** | The safest watch-time bet is more of what you just watched ⇒ the feed collapses |
| ⑥ | **Self-reinforcement** | The model shapes the logs, the logs train the model ⇒ beliefs about what it never shows are unfalsifiable |

> ⭐ **None of these are bugs. They are correct maximisation of a wrong proxy** — which is why the fix is the **objective**, not the model. That's the whole reason the opening starts at the objective function rather than the retrieval stack.

---

## 3 · The available/meaningful table, written properly

⚠ **The original two-column block wasn't aligned pairs** — that was poor formatting on my part. The actual pairs:

| # | AVAILABLE *(cheap, immediate, gameable)* | MEANINGFUL *(what we actually want)* | Why they diverge |
|---|---|---|---|
| 1 | **Watch time** | **Survey satisfaction** — *"was it worth your time?"* | Long and activating content wins time without being valuable |
| 2 | **CTR** | **Return rate next week** | Clickbait wins the click and costs the return |
| 3 | **Session length** | **Survey-measured regret rate** | A rabbit hole is a long session and a bad evening |
| 4 | *(no cheap proxy exists)* | **New-creator reach** | Pure engagement optimisation is rich-get-richer; the corpus stops renewing |

> ⭐ **Row 4 having no left-hand entry is the point.** New-creator reach has **no cheap proxy at all**, which is exactly why it must be an **owned guardrail** rather than something you hope falls out of engagement.

**The sentence to say:** *"Each of these has an available metric and a meaningful one, and they diverge exactly when the system gets good at optimising. **I gate on the second.**"*

---

## 4 · What is a long-horizon holdback?

A **population excluded from a change for months**, so you can measure its long-run effect.

```
NORMAL A/B:  2 weeks → ship → everyone gets it
             ⚠ the counterfactual is GONE FOREVER

HOLDBACK:    a small % NEVER gets it
             → still a valid control at 6 months
```

**Why it's necessary:** engagement wins in recommendation **frequently reverse over months**. A change can lift watch time 3% in two weeks and reduce retention over six — because narrowing and homogenisation accumulate *slowly*. ⚠ **A two-week test is structurally blind to that: the effect doesn't exist yet when the test ends.**

**Two consequences:**
- ⭐ **You must build it before you need it.** Its value comes from elapsed time, so there is no retrofitting — hence "ships in Q1" despite producing nothing visible for months.
- **It costs you.** A small population is permanently on an older, worse-performing system. That's a real, accepted cost.

---

## 5 · Offline metrics — is it nDCG/MRR on click data?

**Right on the ground truth:** it's logged interaction data — clicks, watch time, likes, dismissals. Which is exactly why it carries all the biases (position, presentation) and why it **screens but never decides**.

**One correction on the metrics.** nDCG and MRR are *ranking* metrics — they ask whether the ordering is good. But this model's output is **per-head probabilities feeding a blend**, so:

| Metric | Measures | Why here |
|---|---|---|
| **Per-head AUC** | Discrimination per objective | Did the like-head improve without the dismiss-head regressing? |
| ⭐ **Per-head calibration** | Are the probabilities *right* | **Required for the blend** — combining heads whose scales aren't comparable produces a meaningless weighted sum |
| nDCG / MRR | Ordering of the final slate | Useful, and **secondary** |

> ⭐ **Calibration is the load-bearing one** — same argument as **G21**: once you combine or price with a score, discrimination isn't enough. **nDCG on a multi-task blend tells you the ordering looks fine while hiding that one head is systematically mis-scaled.**

---

## 6 · Trajectory vs slate diversity

**Slate** = within one page. **Trajectory** = across a session or weeks.

```
SLATE (one page):      cooking · football · music · gaming · news
                       ← looks diverse ✓

TRAJECTORY (session):  politics → politics-angrier → politics-fringe
                       ← NARROWING ✗
```

⚠ **A rabbit hole has perfect slate diversity at every step.** Each page offers variety, the user keeps picking the same thread, and the *next* page is built from that pick and narrows. **Measure only slates and the system looks healthy while the sequence collapses.**

So trajectory diversity measures the **entropy of what a user actually consumed over time** — and its **derivative**: *is this person's distribution tightening?* That's the only instrument that sees the failure, which is why it's listed separately in the re-rank box.

---

## 7 · How does the position tower actually work?

**The problem:** position 1 gets clicked far more than position 15 regardless of quality. Train on raw clicks and the model learns **placement, not relevance**.

```
TRAINING                                SERVING
  main tower:     f(user, item) ─┐        main tower: f(user, item)
                                 ├► p(click)        ↓
  position tower: g(rank) ───────┘        ⚠ POSITION TOWER DROPPED
                                          (equivalently: rank := 1 for all)
```

Both towers feed the prediction **jointly**, so the position tower **absorbs the variance explained by placement** — leaving the main tower to explain what's left, which is quality. At serving you drop it, so every item is scored **as if it were at position 1**, making scores comparable for ranking.

**⚠ Why shallow is deliberate:** a deep position tower with access to content features would start explaining *content* through position, **re-contaminating exactly what you're cleaning**. Keeping it shallow — an embedding of rank plus device/context — limits it to the placement effect.

**⚠ What it does NOT fix: presentation bias** — you only observe items you *showed*. Position correction can't touch that; **only exploration can.**

---

## 9 · Two-tower vs DiskANN — and is the ranker a cross-encoder?

### ⚠ Two-tower and ANN are not alternatives — they're different layers

> **The two-tower model *produces* the embeddings. ANN (HNSW/DiskANN) *searches* them. You need both.**

```
TRAINING:   two-tower learns  user_emb = f(user),  item_emb = g(item)
OFFLINE:    precompute item_emb for 10⁹ items → build the ANN index
SERVING:    user_emb = f(user)   [ONE forward pass, ~ms]
            → ANN search over the index → ~1000 candidates   [~30 ms]
```

⭐ **The two-tower architecture exists precisely to make ANN possible**: because the towers are separate, **item vectors don't depend on the user, so they precompute.** A cross-encoder couldn't — it would need 10⁹ forward passes per query.

**On DiskANN specifically — no, not here.** From G1's ladder, DiskANN is for when the index **outgrows RAM**. At 10⁹ items × 128-d int8 ≈ **128 GB**, which fits comfortably across a few resident nodes ⇒ in-memory HNSW, and keep the latency.

**Multiple ANN services — right instinct.** The sources are separate systems queried **in parallel**, and **only the collaborative one is a two-tower ANN**: subscriptions is a database lookup, fresh is a recency-sorted query.

### The ranker is NOT a cross-encoder in the attention sense

| | **Cross-encoder** (Q6's reranker) | **This ranker** |
|---|---|---|
| Mechanism | Query and doc **tokens attend to each other** | Features **concatenated** into one vector |
| Architecture | Transformer | Shared-bottom DNN (or MMoE) + per-task heads |
| Cost | ~120 ms for **100** items | ~40 ms for **1000** items, one batched pass |

It *is* "joint" in that user and item are in the same input — but it's a **feed-forward net over features**, not token-level cross-attention. **That's what makes 1000 items in ~40 ms feasible.**

**MMoE** replaces the single shared bottom with multiple expert sub-networks plus per-task gating, so **conflicting tasks** (watch time vs dismissal) don't have to share all parameters.

---

## 10 · What is "feature fetch", and of what?

**Fetching the feature *values* the model needs, from the feature store, at request time.** The model is a function of features, and most aren't in the request:

```
USER   (1 lookup):    watch history · topic affinities · demographics
                      · session context · language · device
ITEM   (~1000!):      age · channel stats · historical CTR/watch-time
                      · topic embedding · quality signals · language
CROSS:                has this user watched this channel? subscribed?
```

**⚠ Why it's the bottleneck:** that's **~1000 item lookups + crosses per request** across a distributed store, inside a ~100 ms budget. The model forward pass is **one batched matmul on a GPU — a few milliseconds.** The **network round-trips dominate.**

And the p99 is brutal: **the slowest of 1000 parallel fetches sets your latency**, so one slow shard blows the budget even when the median is fine.

> ⭐ **Same shape in G21** (feature fetch, not the model) **and G29** (precomputed graph aggregates): *at high volume and low latency it's a **distributed-lookup problem with a model attached**.* **Optimising the model first is optimising the wrong thing.**

---

## 11 · Hash sharding — is it for load balancing?

Partly — and the **bigger reason is recall**.

**Your point is correct:** hashing gives **even load**, no hot shards, and trivial rebalancing by adding replicas.

**But the more important reason** is G1's: ⚠ **semantic sharding causes recall cliffs.** If you shard by topic you must **classify the query's topic to route** — and a user with cross-topic interests gets candidates only from the shard you guessed, while genuinely better candidates on unqueried shards are **never retrieved**. A first-stage miss is unrecoverable.

> ⭐ **Hashing means no classifier ⇒ no misrouting ⇒ every shard searched ⇒ recall equals an unsharded index.** You pay N× fan-out to buy recall that doesn't depend on a routing guess.

**One nuance on wording:** hashing doesn't "deliberately spread similar videos" — it's **arbitrary**, and *arbitrary* is precisely what guarantees evenness. There's no locality of any kind, and that's the feature.

---

## 12 · What is "near-line"? What is the "fresh source"?

### Near-line = the third update regime

| Regime | When | Used here for |
|---|---|---|
| **Offline / batch** | Hours–days | Model training |
| ⭐ **Near-line** | **Seconds–minutes**, async, **event-triggered** | **Item-tower embeddings** |
| **Online** | Inside the request | Ranking |

**Near-line** = **not in the request path** (so no latency cost), but **not a nightly batch** either. A video is uploaded → a job computes its item embedding → it's upserted into the ANN index, landing within minutes.

### ⚠ The problem that creates

Even minutes is too slow for breaking news — and worse, **a brand-new video has no engagement features**, so its embedding is weak even once computed and a collaborative retriever won't surface it.

### The fresh source is the bypass

A **separate candidate source** selecting on **recency + metadata**, not on learned embeddings:

```
collaborative ANN → needs an embedding → needs engagement → SLOW for new
FRESH source      → "recent uploads from channels this user watches,
                     in topics they follow"  → available IMMEDIATELY
```

It has its **own quota** in the candidate mix, so new content is guaranteed slots **regardless of what the embedding-based sources think.**

> ⭐ **That's the architectural answer to cold start: a separate path with a guaranteed quota, not a better embedding.**

---

## 13 · The three "what breaks" items, explained

**① Feature-store latency spikes.** ~1000 parallel fetches per request means **p99 is the max of 1000 draws** — so even a small per-shard tail becomes your common case. A GC pause or hot shard blows the budget. *Mitigations:* aggressive caching of slow-changing item features · **timeout-with-partial-results** (serve with some features missing rather than failing) · tight tail-latency SLOs on the store.

**② ⚠ A candidate source failing silently — the nastiest one.** The fresh source returns empty. **No exception, no 500, no alert.** The other sources fill the slate and you serve 20 perfectly reasonable videos. **Quality degraded, nothing errored.** You find out weeks later via a diversity metric drifting.
> ⭐ *Fix:* **per-source contribution monitoring** — what fraction of the final slate came from each source — and **alert on a source's share going to zero.** That's exactly why that panel is in §7.

**③ Embedding version skew between towers.** The user tower and item tower **share an embedding space and must be deployed together.** Deploy user-tower v2 against an index built with item-tower v1 and the dot products are **meaningless** — garbage distances, **no error raised**, recommendations become random. *Mitigations:* version-stamp the index · **refuse to serve on a version mismatch** · dual-index during migration.
> *Same failure as **G3's** mixed-space library and **Q6's** embedding migration.*

---

## 14 · Isn't nDCG/MRR a better offline metric? And is offline↔online correlation good?

**On nDCG:** it's a reasonable **slate-level** metric and worth having — but **secondary**, for §5's reason. This is a multi-task model whose heads get **blended**, so **per-head calibration** determines whether the blend means anything, and **nDCG cannot see mis-calibration.**

**On offline↔online correlation — yes, and it's the sharper point.** It's the **meta-metric**: across many past experiments, record *(offline delta predicted, online delta realised)* and track the correlation.

> ⚠ **If that correlation is weak, your offline metric is decorative** — and every change waits on an A/B, so the team iterates **at the speed of experiments.**

⭐ Which makes it a **velocity multiplier**, not a quality metric: improving it means most changes get screened offline in hours instead of consuming experiment slots for weeks. *Same argument **G23** makes as **the framework's own KPI** — and it's rarely treated as first-class even though it determines how fast everything else can move.*

---

## 15 · "Engagement ≠ satisfaction" — so how do you *measure* satisfaction?

⚠ **You largely can't infer it from behaviour — you have to *buy* it.** That's the core measurement problem here, and the reason follow-up Q7 exists on the sheet.

> **A watch is compatible with satisfaction, with compulsion, and with regret — and they are indistinguishable in the log.**

### The ladder

**① ASK — surveys.** The only *direct* measurement. An intermittent in-feed card: *"Was this video worth your time?"*

| Property | |
|---|---|
| **Sampled ⇒ representative** | Unlike reports, which select for outrage |
| ⚠ **Sparse** — ~0.1–1% response | Can't train a dense ranker on it directly |
| ⚠ **Response bias** | People who answer aren't a random sample of those who watched |
| ⚠ **Expensive** | Survey slots are finite — each card costs feed real estate and goodwill |

**② ⭐ BRIDGE THE SPARSITY — train a satisfaction *model* on the surveys.** The move that makes surveys usable at scale: collect the sparse labels, train a model to predict *"would this user have said this was worth their time?"* from **dense behavioural features**, then use the **predicted** score in the ranker.
> So the surveys become a **calibration set for a dense proxy**, not the training signal itself. ⚠ And the predicted score then needs its own agreement check against **fresh** surveys — or you're ranking on a proxy nobody has validated lately.

**③ BEHAVIOURAL PROXIES that correlate with satisfaction** — none *are* satisfaction:
- ⭐ **Return rate** — came back tomorrow / next week. The strongest single behavioural signal, because it's revealed preference about **the product**, not about one video.
- **Volunteered negatives** — *Not interested*, *Don't recommend this channel*. Unambiguous and trustworthy; rare.
- **Likes / shares / subscribes** — costly actions, so they mean something. Biased toward certain content types.
- **Completion normalised for length** — 90% of a 10-minute video ≫ 2% of a 3-hour one.
- **Session shape** — ended in satisfaction or exhaustion? Honestly, mostly unreadable.

**④ LONG-HORIZON HOLDBACKS** (§4) — the only instrument that catches *slow* dissatisfaction, where the product degrades over months and **no per-video signal moves**.

### Say this

> *"Satisfaction isn't observable in behaviour, so I'd **buy** it: sampled survey cards as ground truth, a **satisfaction model trained on those sparse labels** to make them dense enough to rank with, and **return rate** as the behavioural check. Then long-horizon holdbacks, because the failure this system is most prone to doesn't show up in any per-video signal."*

⚠ **What not to claim:** that dwell time or completion rate *is* satisfaction. They're the **available** metrics — §3's whole argument is that available and meaningful diverge exactly when the system gets good at optimising.

---

## 16 · How do you detect and offset clickbait?

More tractable than §15, because **clickbait has a signature: it wins the click and loses everything after.**

### Detection — the signature is a GAP

```
CLICKBAIT PROFILE:
   high CTR                                  ← the thumbnail/title worked
   + LOW watch-through (normalised for length)
   + fast abandon (<10–30 s)
   + elevated dismissal / "not interested"
   + no likes, shares, subscribes
   + ⭐ low or NEGATIVE return rate for users who clicked it
```

> ⭐ **The cleanest formulation: `predicted CTR ≫ predicted watch-through`.** A video whose click rate massively over-performs its retention is **by definition promising more than it delivers**. That ratio is a direct, computable clickbait score — and it needs **no new labels**, because both terms are heads you already have.

Other signals: **thumbnail ↔ content mismatch** (multimodal — catches "shocked face + arrow" depicting something not in the video) · ⭐ **channel-level priors**, because clickbait is a **strategy, not an accident**, so a persistent CTR-to-retention gap across many uploads is far stronger evidence than any single video · **title patterns** (all-caps, "YOU WON'T BELIEVE") — weak and gameable alone, fine as one feature.

### Offsetting — four mechanisms, by leverage

**① ⭐ FIX THE OBJECTIVE — the structural answer.** Clickbait only works because the objective rewards it. Train on **watch-through, not CTR**, with a **fast-abandon negative head**: *a click followed by a three-second watch is a **negative**, not a weak positive.* Get this right and most clickbait stops being **profitable to the ranker** — far more durable than any detector.

**② The negative head learns it directly.** `p(fast abandon)` in the blend means the **ranker itself** down-weights clickbait, continuously, with no separate classifier to maintain.

**③ Channel-level down-weighting.** Since it's a strategy, a channel prior affects **future** uploads *before* they accumulate their own evidence. Highest-leverage detection-based lever.

**④ Creator-side incentives.** Surface the gap in creator analytics (*"high CTR, low retention"*) and reflect it in recommendation. ⭐ **The economic fix beats the detection fix** — same argument as fraud: make the strategy **unprofitable** rather than catching every instance.

### "What if a user clicks one?" — within the session

1. **The abandon is logged as a negative** — trains against that video *and* that channel.
2. **Session-level suppression** — down-weight that channel and topic cluster for the rest of the session. Cheap, and it prevents serving three more from the same channel.
3. ⚠ **Don't over-correct from one signal.** A fast abandon has innocent explanations — wrong video, got interrupted, already seen it. **One event adjusts a prior; it doesn't blacklist.** *(Same conservatism as Q9's write gate: one observation is weak evidence.)*

### ⚠ Two things to volunteer

> **It's adversarial, so this is an economic contest rather than an accuracy contest.** Creators optimise against whatever we reward, and a detector invites one more iteration — which is why the weight goes on **the objective and the channel prior**, not a title classifier. And use **temporal evaluation** (train past, test future), because a random split includes tomorrow's patterns in training and reports performance against an adversary that has already moved on.

> **You cannot fully separate clickbait from a genuinely compelling thumbnail.** A great thumbnail on a great video *also* produces high CTR. The distinguishing signal is **what happens after the click** — so you tolerate some false positives on good content with excellent packaging, and **the threshold is a real trade, not a detection problem you can solve.**

---

## 🔁 THE NINE THINGS I KEPT GETTING WRONG

1. **Regret is a survey card, not the report button.** Report is policy (G25); regret is quality.
2. **Two-tower and ANN aren't alternatives** — two-tower *produces* the vectors, ANN *searches* them. The split exists so item vectors precompute.
3. **The ranker isn't a cross-encoder** — concatenated features through a shared-bottom DNN, which is why 1000 items fit in ~40 ms.
4. **Calibration, not nDCG, is the offline metric that matters** — because the heads get blended.
5. **Hash sharding is about recall first, load second.** It's arbitrary, not a clever spread.
6. **A rabbit hole has perfect slate diversity** — only *trajectory* diversity sees it.
7. **The position tower doesn't fix presentation bias** — only exploration does.
8. **Satisfaction has to be *bought*, not inferred** — sampled surveys, then a model trained on them to make the signal dense. Dwell time is not satisfaction.
9. **Clickbait = `predicted CTR ≫ predicted watch-through`** — and the fix is the objective, not a detector. It is an economic contest.
