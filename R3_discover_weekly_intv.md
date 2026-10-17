# R3 · A Weekly Discovery Playlist (Discover Weekly–style) — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md) and [R2](R2_video_query_search_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **The prompt:** *"Design a recommendation system similar to Spotify Discover Weekly. It recommends a playlist of 10 songs each week. After those 10 songs are played, a new recommendation appears."*
>
> **Expect these probes:** (a) assumptions · (b) what the listener sees · (c) the two clocks: weekly batch and per-playlist assembly · (d) datasets · (e) labels from listening · (f) the models and their objectives · (g) serving the next playlist instantly · (h) how you know it worked · (i) the extension.
>
> **Related sheets:** [G13 YouTube recommendation](G13_youtube_recommendation_intv.md) (multi-source candidates, ecosystem guardrails) · [G17 similar items](G17_similar_items_intv.md) (popularity normalisation) · [G15 Play Store](G15_play_store_recommendations_intv.md) (the delayed-label bridge) · [Q26 continuous learning](Q26_continuous_learning_intv.md) (propensities, off-policy evaluation) · study files [G18 music + LLM](../G18_music_llm_recommender.md) and [G20 bandits](../G20_feedback_driven_recommender.md).

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | What counts as "played" — do skips count? Does Monday still refresh an unfinished playlist? Shared accounts? Paid placement? (§1) |
| **Explainability** is easy points | A **reason on every track**, built from where the slot came from — "Because you saved X", "New from an artist like Y" (§2, §8) |
| **Security** beyond the gateway | **Stream farming**: bots and mass-uploaded AI-generated tracks inflating co-listening — which also routes royalties (§8, FU 9) |
| **Troubleshoot before being asked** | Raise **skip-farming**, **pool exhaustion** for heavy listeners, and **household taste pollution** yourself (§3, §4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Discovery, not prediction** · **two clocks** · a playlist is a set | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · novelty as a hard constraint · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The playlist · ⚠ **sticky discoveries, not streams** · skip-through isn't success | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Weekly pools ≈ 1.2 TB · ~500 assemblies/s · **the latency budget is two songs long** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Weekly batch → per-user pool → triggered assembly → slate of 10 → prefetch | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Reached, not heard · two-tower + multi-task ranker + DPP + exploration bandit | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Prefetch at track 8 · learn from the last ten · availability at play time | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Future-discovery replay · weekly-cohort A/B · holdback · artist fairness | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Conversational steering, context · **taste-cluster fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Two things make this different from an ordinary recommender, and I'd design around both.**
>
> **First, it's a discovery product, not a prediction product. The songs a listener is most likely to play are songs they already know — so a model that optimises plays converges on the familiar, and the product quietly stops being Discover. So novelty is a hard constraint — every track is one they haven't heard — and the objective is discoveries that stick: tracks they save, or come back to in the following weeks. Not streams.**
>
> **Second, it runs on two clocks. The heavy work — learning each listener's taste and scoring a pool of a few hundred unheard candidates — runs weekly in batch. But a new playlist appears once the ten songs are played, so assembling each ten-song playlist is event-driven, and it should learn from what just happened: what they skipped, what they saved. The trigger is predictable, so I build the next playlist while the current one is still playing — the latency budget is two songs long, not two hundred milliseconds."**

**Then the third commitment:**
> ⭐ **"And a playlist is a set, not a top ten.** Ten slots, one artist each, some deliberately exploratory, in an order that flows — and the first track matters most, because an early skip ends the session."

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the listener sees and how we measure discovery, the capacity and the two clocks, the architecture, the data and models, serving the next playlist, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What counts as "played" — ten tracks reached, skips included, or ten tracks actually listened to?** *(Decides whether skipping farms new playlists — FU 2.)*
2. **Does the weekly refresh still happen — and is there a cap on refreshes?** *(The original product's Monday scarcity was part of its pull; an endless refresh can turn it into one more feed.)*
3. **Are accounts shared — family, a toddler, a party speaker?** *(Taste pollution — FU 7.)*
4. **Is there any paid placement for labels or artists?** *(Decides whether the objective needs a disclosure constraint — FU 8.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Users | ~500M; ~10⁸ open the playlist in a given week |
| Catalogue | ~10⁸ tracks; ~10⁵ new tracks a day |
| Unit | A playlist of 10 tracks ≈ 35 minutes of listening |
| Refresh | A new playlist on Monday, **or** when the current one is finished — whichever comes first |
| "Played" | Every position **reached** (streamed or skipped) |
| Surfaces | Mobile, desktop, car, smart speakers; offline download on mobile |

**FR:** weekly taste profile and candidate pool · assemble each playlist of **10 unheard tracks** · learn from the previous playlist · **prefetch the next** · work offline · a reason on each track · controls (hide artist, "not for me", exclude from taste profile).

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Next playlist ready** | **Before track 10 ends**, at p99 |
| **Novelty** | **100% unheard tracks**; no track recommended twice in 26 weeks |
| Availability | Every track licensed in the listener's country **at play time** |
| Weekly batch | Done for every active user before their local Monday morning |
| Artist fairness | Emerging-artist exposure share monitored as a guardrail |

**Scope / NON-GOALS:**
- ❌ **Optimising stream count.** Position: it produces familiarity, not discovery.
- ❌ **An LLM choosing tracks per request.** Position: cost, and it can name tracks that don't exist or aren't licensed. An LLM may write descriptions offline.
- ❌ The home feed, radio, search and editorial playlists (adjacent products).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the listener sees (b):**

```
DISCOVER — your 10 for this week                       [ Download ]
"Late-night indie folk, with a detour into desert blues"   ← written from the slots
  1  Track A — Artist P    Because you saved "…"
  2  Track B — Artist Q    New from an artist like R
  3  Track C — Artist S    Exploring: desert blues           ← an exploration slot, labelled
  …
 10  Track J — Artist Z
  ♡ save   ⊘ hide artist   "not for me"                       ← feedback that teaches

After track 10:  "Your next 10 is ready"  → plays on, no wait   ← built while you listened
Settings:  "Exclude this playlist from my taste profile"
```

```
THE LADDER — each rung better and more delayed:

  START → STREAM ≥ 30 s → COMPLETE → SAVE / ADD TO PLAYLIST / FOLLOW
        → ⭐ RE-LISTEN in the following weeks  (the discovery stuck)

⚠ PLAYS REWARD THE FAMILIAR. Optimise plays and the product becomes
  "songs you already like" — technically successful, no longer Discover.

⚠ AND THE REFRESH MECHANIC ADDS A NEW TRAP:
  "playlists consumed per week" goes UP when playlists get WORSE —
  listeners skip through ten tracks to reach the next ten.
  → skip-through is a NEGATIVE, and playlist count is NEVER a success metric.

GATE ON:    sticky discoveries per user-week (matured, 4 weeks)
ITERATE ON: saves + ≥30 s streams on exploratory slots, weighted
TRACK:      does the fast proxy still predict the sticky discovery? (G15)
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Sticky discoveries per user-week** — new tracks or artists saved or re-listened within 4 weeks | Discovery that lasted |
| Fast proxy | Save rate · ≥ 30 s stream rate · completion, **per slot type** | Iteration speed |
| Product health | Weekly return to the playlist · **total listening grown, not cannibalised** | Is it worth having? |
| **Guardrails** | Skip-through rate · repeats = 0 · unavailable tracks = 0 · emerging-artist share · artist concentration | Hard limits |

> **A product judgement to raise:** *"The weekly scarcity was part of the original product's appeal — a Monday ritual. I'd keep the weekly ten as the anchor and treat the next ten as 'keep discovering', and measure whether it adds listening or just cannibalises the listener's own playlists."*

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
WEEKLY BATCH — the heavy part:
  ~2×10⁸ active users × ~500 candidates scored each ≈ 10¹¹ user-track scores
  at ~10 µs a score, batched on CPU ≈ 10⁶ CPU-seconds ≈ ~280 CPU-hours
  → an ordinary weekly job; retrieval over 10⁸ tracks is the bigger
    engineering piece, and still routine
  POOLS: 2×10⁸ × 500 × (8 B id + 4 B score) ≈ 1.2 TB in a KV store

THE REFRESH LOAD:
  a playlist ≈ 10 × 3.5 min ≈ 35 min of listening
  ~10⁸ weekly users × ~3 playlists ≈ 3×10⁸ assemblies a week ≈ ~500/s average,
  several × that on Monday mornings and evenings
  each assembly: one KV read + re-rank ~500 + pick 10 → tens of ms of CPU

⭐ THE LATENCY BUDGET IS TWO SONGS LONG:
  the trigger (reaching track 10) is PREDICTABLE → build the next playlist
  when track 8 starts → ~7 minutes, not 200 ms
  → latency stops being the constraint; AVAILABILITY and OFFLINE become it

⚠ POOL EXHAUSTION — the heavy-listener problem:
  3 h a day ≈ 5 playlists a day ≈ 350 tracks a week, and the diversity
  rules burn candidates faster than that
  → top up the pool mid-week once it drops below ~100 eligible tracks

NEW TRACKS: ~10⁵ a day → audio embeddings computed at ingestion
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Objective | P(play) vs **P(sticky discovery)**, with novelty as a hard constraint | Plays reward the familiar |
| When the work runs | Fully online per playlist vs **weekly batch pools + event-driven assembly** | Taste moves slowly; feedback and mood move fast |
| The ten tracks | Top-10 by score vs **a slate**: one per artist, diversity, exploration slots, an order | The listener judges the ten together |
| Learning from the last ten | Wait for the weekly retrain vs **a session re-rank + a per-listener exploration bandit** | Each playlist is a fast feedback cycle |
| The LLM | Picks tracks per request vs **writes descriptions and reasons offline, from slot provenance** | Cost; and it can name tracks that don't exist or aren't licensed |
| Candidates | One two-tower model vs **several sources with quotas** | Audio neighbours carry new tracks; co-listening carries the head |

### The system — *draw this in 90 seconds* `A2a`

```
══ WEEKLY BATCH (Sunday night, per region) ══════════════════════════════════
┌─ 1. TASTE PROFILE ─────────────────────────────────────────────
│ TASK: embed each listener from months of listening — long-term
│       taste + recent drift, SEVERAL taste clusters per account
│ OUTPUT: user vectors, one per taste cluster
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CANDIDATE SOURCES, EACH WITH A QUOTA ──────────────────────
│ two-tower ANN over 10⁸ tracks · playlist co-occurrence (PMI-
│ normalised) · audio neighbours (new tracks) · artist-graph hops ·
│ new releases
│ FILTER: already heard · recommended recently · hidden artists ·
│         explicit setting · NOT LICENSED in this country
│ OUTPUT: ~1–2k unheard candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RANKER — multi-task ───────────────────────────────────────
│ heads: P(stream ≥ 30 s) · P(save) · ⭐ P(re-listen within 4 weeks)
│        ⚠ NEGATIVE: P(skip < 30 s)
│ OUTPUT: a scored POOL of ~500 per listener → KV store
└────────────────────────────────────────────────────────────────

══ PER PLAYLIST (Monday, or when the current one is nearly done) ═════════════
┌─ 4. TRIGGER — track 8 of the current playlist starts ──────────
│ OUTPUT: an assembly request carrying the last playlist's outcomes
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. SESSION RE-RANK + EXPLORATION BANDIT ──────────────────────
│ skipped tracks push their neighbours down; saves pull them up
│ the bandit sets how many slots are EXPLORATORY for this listener (1–4)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. SLATE ASSEMBLY — 10 tracks ────────────────────────────────
│ one per artist · DPP diversity · exploration slots · a quality floor
│ · an ORDER that flows (a strong, familiar-adjacent opener)
│ OUTPUT: 10 tracks + a reason per track + provenance logged
└──────┬─────────────────────────────────────────────────────────
       ▼
 PREFETCHED to the device (works offline) → "Your next 10 is ready"
       ▼
 OUTCOMES (skips, ≥ 30 s, saves) → step 5 next time · RE-LISTENS → weekly training
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Skip-farming:** *"If a skip counts as played, fast skippers farm new playlists. Skip-through is a negative, the next playlist gets more conservative, and playlist count is never a success metric."*
- **Pool exhaustion:** *"A heavy listener burns through 500 candidates by Thursday, so the pool is topped up mid-week — never by repeating a track."*
- **Household pollution:** *"Lullabies for a toddler shouldn't define a parent's taste — several taste clusters per account, and an 'exclude from my taste profile' control."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Novelty as a hard constraint, sticky discovery as the objective** | "The songs most likely to be played are songs you know. Optimise plays and Discover stops discovering" | A visibly lower short-term stream rate |
| **2** | **Weekly pools, event-driven assembly** | "Taste moves slowly, feedback moves fast — the expensive model runs weekly, the cheap one per playlist" | A taste change inside the week arrives late |
| **3** | **Build the next playlist before it's needed** | "The trigger is predictable, so the budget is two songs long" | Some prefetched playlists go unplayed |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Listening logs** | Starts, skips (and *when* in the track), ≥ 30 s streams, completions | Skips are noisy: mood and context, not only taste |
| **Saves, playlist adds, follows** | Strong positives | Sparse |
| **User-made playlists** (billions) | Which tracks people put together | Dominated by hits — normalise (G17) |
| **Audio** | Embeddings from the waveform — the only signal a brand-new track has | Sounding alike ≠ liked by the same people |
| **Text and metadata** | Credits, tags, artist bios, lyrics | Sparse for the tail; licensing |
| **Licensing and content flags** | Availability per country, explicit flags, policy status | Change mid-week |
| **Exploration logs with propensities** | Unbiased signal on what listeners would like | Small, and precious |

### 5.2 Labels from listening (e) `F1c`

- **Only reached positions count.** A track the listener never got to wasn't heard, so it is **not** a negative — just unexposed.
- **Skip < 30 s** → negative · **≥ 30 s** → weak positive (also roughly where a play counts as a stream) · **complete** → positive · **save / add / follow** → strong · **re-listen in the next 4 weeks** → the truth, joined late. Fast positives are weighted by how often similar ones became re-listens (G15's bridge).
- **Context-confounded skips** (car, party, a shared speaker) are down-weighted; sessions the listener excluded from their taste profile are dropped.
- **Stream fraud is removed** before anything becomes a label (§8).
- **Exploration slots log their propensities**, so they can be used for unbiased evaluation (Q26).

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective |
|---|---|---|
| **Two-tower retrieval** | User and track embeddings | Sampled softmax over co-listening, with a **logQ correction** so hits don't dominate |
| **Multi-task ranker** | Discovery value per track | Cross-entropy per head; score = P(save or re-listen) minus an early-skip penalty |
| **Slate selector** | Choose 10 | Greedy selection under hard constraints, with a **DPP** term for diversity |
| **Exploration bandit** | How adventurous to be with this listener | **Thompson sampling** on a Beta posterior of exploratory-slot success → 1–4 exploration slots |
| **Sequencer** | Order the 10 | Transitions scored on tempo, energy and key; a strong opener first |

```
⭐ THE SLATE OBJECTIVE — constraints first, then value:
  HARD:  every track unheard · ≤ 1 per artist · licensed here · explicit setting
         · k exploratory slots (k from the bandit) · P(skip < 30 s) ≤ τ per slot
  THEN:  maximise  Σ P(sticky discovery)  +  diversity (DPP log-det)
  → promises go in constraints; diversity is the one soft term, because it's
    taste quality, not a promise to anyone
```

### 5.4 Fine-tune an existing model (f) `F2c`

LoRA-tune a **pretrained audio model** contrastively on co-listening pairs, so its embedding learns what listeners treat as similar, not just what sounds similar. That is what lets a day-old track by an unknown artist land near its likely audience (FU 5). An LLM writes playlist descriptions and track reasons **offline**, grounded only in each slot's provenance — it never picks tracks.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · CLEAN      fraud filtering · excluded sessions dropped · licensing join
STAGE 2 · LABEL      reached positions only · skip / stream / save · late re-listen join
STAGE 3 · TRAIN      two-tower + ranker weekly · audio encoder for new tracks daily
STAGE 4 · POOL       retrieve → filter → score → per-listener pools (KV), per region
STAGE 5 · ASSEMBLE   per trigger: session re-rank → bandit → slate → sequence → prefetch
STAGE 6 · LEARN      outcomes → the bandit now · re-listens → next week's training
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
THE NEXT PLAYLIST IS BUILT BEFORE IT'S NEEDED:
  track 8 starts → assemble → push to the device → ready at track 10
  offline: download the current AND the next playlist together
  if the listener stops at track 6, the prefetched playlist waits — it
  isn't counted as exposed until one of its tracks starts

WHAT THE LAST TEN TEACH THE NEXT TEN (within minutes, no retrain):
  skipped in < 30 s   → push that track's neighbours and artist down
  saved / completed   → pull neighbours up
  skipped everything  → fewer exploratory slots, a safer opener, and
                        "Not in the mood for something new?"
  ⚠ none of this touches the LONG-TERM taste profile until the weekly
    run — one bad evening shouldn't rewrite a year of taste

AVAILABILITY IS CHECKED TWICE: at assembly, and when the track starts —
  licences change mid-week; a removed track is swapped in from the pool
```

- **The objective** `A3b`: the slate objective above — hard constraints (novelty, one per artist, licensing, explicit setting, exploration quota, skip floor), then sticky-discovery value plus diversity.
- **Efficiency** `F3a`: weekly pools plus a cheap per-playlist re-rank, instead of full retrieval per playlist. The trade is that a taste change inside the week arrives late; the session re-rank recovers most of it at tens of milliseconds. int8 track embeddings halve ANN memory.
- **Scale** `F3b`: pools sharded by user in a KV store; the batch runs per region ahead of local Monday; mid-week top-ups only for the heavy listeners who need them.
- **Reliability** `I1b`: the next playlist is already on the device. If assembly fails, the pool's top ten under the same rules; if the pool is empty, the taste-cluster fallback (§9). **Never a repeat, never a blank.**

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE    held-out weeks: did the listener LATER discover, on their own,
           tracks we would have recommended? (recall of future organic
           discoveries) · slate properties: novelty share, artist diversity,
           concentration · OPE on exploration slots with logged propensities
ONLINE     A/B by listener, in weekly cohorts: saves and ≥ 30 s on exploratory
           slots move fast; sticky discoveries after 4 weeks decide
LONG RUN   a holdback on the old system for months: retention and TOTAL
           listening — does discovery grow listening, or cannibalise it?
```

**Success** = sticky discoveries per user-week up, novelty and artist diversity held, total listening grown rather than cannibalised, and retention up in the holdback. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Popularity** | Co-listening favours hits | logQ correction; PMI; evaluation by popularity decile |
| **Exposure loop** | Only recommended tracks get feedback | Exploration slots with logged propensities |
| **Position** | Early tracks are heard most | Reached-only labels; position-aware training |
| **Skip ambiguity** | Skips reflect context as well as taste | Context down-weighting |
| **Artist gender and region** | Recommenders have been shown to amplify gender imbalance among artists; local artists lose to global hits | Exposure audits by artist group and region; emerging-artist guardrail |
| **Household pollution** | Shared accounts blend tastes | Several taste clusters; exclusion controls |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Next playlist ready before track 10 ends; weekly batch done before local Monday; cost per 1,000 playlists | Readiness p99 · batch completion time |
| **Reliability** `I1b` | Next playlist on the device; fallback chain (pool → taste cluster); availability re-checked at play time | Fallback rate · unavailable tracks = 0 |
| **Usability** `I1c` | A reason per track; save / hide artist / "not for me"; an order that flows; works offline | Save rate · skip-through rate |
| **Scalability** `I1d` | ~1.2 TB of pools sharded by user; per-region batch; top-ups only for heavy listeners | Assembly QPS headroom at peak |
| **Privacy** `I1e` | Listening reveals religion, health (sleep, grief, meditation), sexuality → **no sensitive inference**, nothing sensitive in shareable playlist titles; **private sessions** and **"exclude from my taste profile"** honoured; kids' profiles separate | Sensitive-inference audits · control usage |
| **Safety** `I1f` | Explicit-content setting enforced; kids' profiles; policy-flagged tracks (e.g. hate content) excluded; sensitive themes never used as exploration | Policy-violating plays = 0 |
| **Security** `I1h` | ⚠ **Stream farming**: bots and mass-uploaded AI-generated tracks inflate co-listening and route royalties → fraud filtering **before** training, trust-weighted signals, velocity anomalies on track pairs, audio-duplicate detection at upload | Fraud streams caught · duplicate uploads blocked |
| **Ethics / bias** `I1i` | Emerging and independent artists get exposure; gender and regional balance audited; any paid promotion **disclosed and capped**, never a hidden weight | Exposure share by artist group |
| **Explainability** `I1j` | A reason per track from **slot provenance**; the playlist description grounded in it; per-slot logs (source, score, exploratory flag, model version); artists see where their streams came from | % tracks with a reason · provenance coverage |

> ⭐ *"A discovery product earns trust one track at a time: each one new, each one explained, and none of them bought."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Conversational steering** ("more like track 3, but calmer"): an LLM turns the request into constraints for the same slate assembler — it never picks tracks itself (G18).
- **Context:** time of day and activity (workout, focus, commute) as features of the per-playlist assembly.
- **Podcasts and audiobooks:** different units of consumption and different labels — a separate product.

**When there's no personal signal to trust — the taste-cluster pipeline** `A5b` `F2a`:

```
a new listener, a polluted or fraud-hit history, or a failed personal pool
  → map the listener to a few TASTE CLUSTERS (onboarding picks, region,
    recent plays)
  → precomputed weekly cluster playlists, filtered by THIS listener's
    heard-set
  → the same slate rules: unheard, one per artist, licensed, explicit setting
  → more conservative exploration; personal pools return as signal builds
  "Less personal — still new, still never repeated."
```

**Top 3 failure modes:**
1. **Familiarity collapse** — optimising plays turns Discover into songs you already like, and the stream dashboard approves.
2. **Skip-farming** — the refresh rewards skipping, and playlist count rises as quality falls.
3. **Taste pollution** — a household's, or a fraud ring's, listening ends up defining the recommendations.

**Q1 ships:** **weekly pools with the novelty filter and the multi-task ranker**, the **event-driven assembler with prefetch**, and **exploration slots with propensity logging**.
> *"Exploration and propensities in Q1, because without them the system only ever learns about tracks it already chose to show — and that can't be fixed retroactively."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just recommend the songs they're most likely to play?"** `F1b` `A3b`
> Because the songs a listener is most likely to play are songs they already know — or close cousins from artists they already follow — and a model trained on plays will find that out and converge on it. The result is technically successful and no longer Discover: stream rates look great and nothing new is found. So I'd split the problem. **Novelty is a hard constraint**: every track is unheard, and I'd also track how many come from artists they've never played. **The objective is sticky discovery**: the probability the listener saves the track or comes back to it in the following weeks. That label arrives weeks late, so I'd iterate on saves and 30-second streams from the exploratory slots, weighted by how often similar ones became re-listens, and gate on the four-week truth — the same bridge as the Play Store's install-versus-retention problem. And I'd say the trade out loud: the short-term stream rate will be lower than a familiarity-optimised playlist's, and that is the price of the product doing its job.

**2. "The listener finishes the ten in five minutes by skipping everything. What happens?"** `I1c` `F2a`
> First, a definition I'd want settled up front: does a skip count as "played"? If it does — the natural reading and the simpler product — then skipping becomes a way to farm new playlists, and the system acquires a perverse incentive: **playlists consumed per week rises as playlists get worse**. So two rules. **Skip-through is a strong negative**, never an engagement win, and playlist count is never a success metric. And **the next playlist responds**: fewer exploratory slots, a safer opener closer to their known taste, the skipped tracks' neighbours pushed down. If they skip through again, I'd stop pretending and ask — "Not in the mood for something new?" — and offer a familiar mix, because sometimes the right discovery playlist is no discovery today. What I would not do is let one bad evening rewrite a year of taste: session feedback shapes the next slate within minutes, while the long-term profile moves weekly.

**3. "How is the next playlist ready the instant the tenth song ends?"** `F2b` `A1b`
> Because the trigger is predictable, the latency budget is far larger than it looks. When track 8 starts, I assemble the next ten — read the listener's pool, apply what the last eight tracks taught us, pick and order the slate — and push it to the device. That leaves about two songs, roughly seven minutes, for work that takes tens of milliseconds, so latency stops being the constraint and **availability** becomes it: the device should already hold the next playlist, which also makes it work in a car or on a plane. The heavy parts — taste modelling, retrieval over a hundred million tracks, scoring a few hundred candidates — already ran in the weekly batch. Two edge cases worth naming: if the listener stops at track 6, the prefetched playlist simply waits and isn't counted as exposed until one of its tracks starts; and if assembly fails, the device falls back to the pool's top ten under the same rules, so a backend hiccup never produces a repeat or a blank.

**4. "How do you turn scores into a ten-song playlist?"** `A2a` `F1a`
> As a slate rather than a top ten, because the listener judges the ten together. **Hard constraints first**: all unheard, one track per artist, licensed in their country, their explicit setting, and a ceiling on predicted early-skip for every slot. **Exploration slots**: a per-listener bandit decides how many of the ten sit deliberately further from their taste — Thompson sampling on how their past exploratory slots went — so an adventurous listener gets three or four and a cautious one gets one. **Diversity** across the rest through a determinantal point process, which rewards a set whose tracks are good *and* different from each other, so you don't get ten near-identical indie-folk songs. Then **order**: open with the highest-confidence track closest to their known taste, because an early skip ends the session; smooth the transitions in tempo, energy and key; spread the exploratory slots out rather than bunching them. It's a small combinatorial problem and greedy selection is good enough — and every slot logs where it came from, which powers both the explanation and the evaluation.

**5. "A brand-new track from an unknown artist. Can it ever appear?"** `F2c` `I1i`
> It has to, or the product only ever surfaces artists who are already established — the same rich-get-richer loop as new apps in a store. A new track has no listening history, so collaborative signals can't place it; what it does have is **audio**. I'd LoRA-tune a pretrained audio model contrastively on co-listening pairs, so its embedding learns what listeners treat as similar rather than just what sounds similar — then a day-old track lands near the tracks its likely audience already loves, and enters through the audio-neighbour source. Then it needs exposure: a **quota for new releases** inside the exploration slots, with its early saves and skips feeding back within the week. And I'd monitor **emerging-artist share** as a guardrail, because a ranker trained on engagement drifts toward proven artists unless something holds the door open. The safeguard on the other side is fraud: that new-release quota is exactly what stream farmers target, so new uploaders' signals are trust-weighted until they look organic.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Skip rate fell 10% after a change. Is the product better?"** `F1b` `I2b`
> Not established — on this product a falling skip rate is the classic signature of the failure I'm guarding against. The fastest way to cut skips is to recommend more familiar music: tracks from artists the listener already follows, or near-copies of songs they've worn out. So I'd check, in order: **artist novelty** — what share of tracks come from artists they've never played — and whether the exploration slots were actually delivered; **saves and four-week re-listens** on the new tracks, because the goal is a discovery that sticks, not a track that didn't annoy; **concentration** — are we steering everyone toward fewer, more popular tracks; and the **proxy↔truth check**, whether the fast signals still predict sticky discoveries. If skips fell *and* sticky discoveries rose with novelty held, it's a real win. If skips fell while novelty dropped, the model learned to stop discovering and the metric applauded.

**7. "A parent plays lullabies for their toddler every night. Now their Discover is full of lullabies."** `I1e` `F1c`
> That's taste pollution from a shared account, and it's one of the most common complaints about music recommenders — a partner's gym playlist or a party on the living-room speaker does the same. Three layers. **Model several tastes per account**: cluster the listening into taste profiles rather than averaging it into one vector, and assemble from the clusters that match this listener's usual contexts — a nightly lullaby cluster on the bedroom speaker is a different taste from the commute. **Give people control**: "exclude this playlist from my taste profile", private sessions that train nothing, and separate kids' profiles in family plans — the strongest fix, because it's explicit. **Use context**: device, time and session shape separate a household's listening from the owner's. And it's a privacy point as well: the playlist reveals what happens in a household — that there's a toddler, what someone plays late at night — so nothing like that belongs in a playlist title that might be shared.

**8. "Labels want to pay to get their songs into the playlist."** `I1i` `A3b`
> I'd treat that as a decision about what the product promises, not an ML tweak. A discovery playlist works because listeners trust the tracks were chosen for them; hidden payment turns it into ad inventory, and once listeners suspect that, saves and trust both fall. Some streaming services do run opt-in programmes where artists accept a lower royalty in exchange for a recommendation boost, and they've drawn criticism. If the business wants one, my conditions: it's **disclosed** to listeners; it's **capped** — a fixed number of slots or a bounded boost; and it's a **constraint the listener's interest still governs** — a promoted track must clear the same novelty rules and skip ceiling — never a hidden weight that a big enough payment can override. And I'd report promoted slots separately: if they're saved less, that cost belongs on a dashboard, not buried in an average.

**9. "Bots are streaming tracks to get them recommended."** `I1h` `A2c`
> That's stream farming, and it has money behind it: recommendations route royalties, so inflating co-listening pays twice — in plays and in placement. There has been a US fraud prosecution over bots streaming AI-generated tracks to collect millions in royalties. Defences, all before anything becomes a training label: **stream-level fraud detection** — account age, device and network clusters, unnaturally regular listening, plays that stop just past the stream threshold; **trust-weighted signals** — co-listening from new or suspicious accounts counts for little until it looks organic, the same idea as counting trusted actors in autocomplete; **velocity anomalies** on track pairs, since organic co-listening grows gradually; and **upload-side checks** — audio-duplicate detection and mass-upload patterns that flag AI-generated spam. The new-release exploration slots are the obvious target, so they draw only from uploaders above a trust threshold, and every confirmed ring becomes training data for the detector.

**10. "How do you know offline whether a playlist would have been a good discovery?"** `I2a` `F1a`
> Offline evaluation is harder than usual, because discovery means the listener has no history with these tracks. Three instruments. **Future organic discovery**: on held-out weeks, take what the listener later found and kept on their own — through search, friends, radio — and ask whether we'd have recommended it; recall of those future discoveries is a fair proxy for "we'd have found it first". **Off-policy estimates on the exploration slots**: because exploratory tracks were chosen with logged propensities, I can estimate how a new policy would have done on them, with the caveat that slate-level estimates are noisier than per-track ones. **Slate properties**: novelty share, artist diversity and concentration, plus the hard constraints. None of these is the truth — that's a four-week re-listen in a live test — so offline decides which candidates earn an A/B slot, and a long-horizon holdback decides whether the product is growing listening at all.

---

## ⭐ THE L6 LINE

> **"A discovery playlist optimised for plays stops discovering — so novelty is a hard constraint and the objective is discoveries that stick. It runs on two clocks: taste weekly in batch, and each ten-song slate on demand, built while the last one plays, from what the listener just skipped and saved."**

## 🧭 HOOKS — *steer by what you mention*

- *"Discovery, not prediction"* → invites the objective and metrics branch (**good at L6**)
- *"Two clocks"* → invites the systems branch
- *"The latency budget is two songs long"* → invites the serving branch
- *"Playlists consumed goes up when playlists get worse"* → invites the metric-trap branch (**good at L6**)
- *"A playlist is a set, not a top ten"* → invites the slate and diversity branch
- *"Recommendations route royalties"* → invites the fraud and ecosystem branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the two clocks"* — weekly taste plus per-playlist assembly regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the metric traps and stream fraud than finish the sequencer. Sound right?"*

**Name and defer:**
- "**Sequencing and transitions** — DJ-style ordering — is a model of its own."
- "**Lyrics and language** matter for multilingual catalogues and for sensitive themes."
- "**Real-time context radio** — a continuous stream rather than a ten-song unit — is a different surface."
- "**Social discovery** from friends' listening has its own privacy model."
- "**Artist-side analytics** — showing artists where their streams came from — is a product in itself."
- "**Royalty accounting** sits downstream of every recommendation and is why fraud matters so much."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the discovery objective and two clocks | §0 | *"Novelty is a hard constraint; the objective is discoveries that stick."* |
| **A1b** | Latency and capacity budgets | §1, §3, FU 3 | *"10¹¹ scores ≈ 280 CPU-hours a week; ~500 assemblies/s; a two-song latency budget."* |
| **A1c** | Degrade without lowering quality | §6, §9 | *"Assembly fails → the pool's top ten under the same rules; never a repeat, never a blank."* |
| **A2a** | The full stack | §4 diagram, FU 4 | Taste → sources → ranker → pool → trigger → session re-rank → slate → prefetch |
| **A2b** | Model components | §5.3 | Two-tower, multi-task ranker, slate selector, exploration bandit, sequencer |
| **A2c** | Data sources | §5.1, FU 9 | Listening logs, saves, user playlists, audio, metadata, licensing, exploration logs |
| **A2d** | Staged pipeline | §5.5 | Clean → label → train → pool → assemble → learn |
| **A3a** | Top-down, unprompted | §0 | *"Discovery, not prediction"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.3, §6, FU 1, 8 | *"Hard constraints first, then sticky-discovery value plus diversity."* |
| **A4a** | Clarify first, assumptions explicit | §1 | What counts as played? Weekly refresh? Shared accounts? Paid placement? |
| **A5a** | Alternatives compared | §4 | Plays vs discovery; online vs batch + events; top-10 vs slate; LLM picks vs LLM descriptions |
| **A5b** | A full alternative pipeline on demand | §9 | The taste-cluster pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 4, 10 | *"Sampled softmax with logQ; Thompson sampling for adventurousness; a DPP for the set."* |
| **F1b** | Metrics and their trade-offs | §2, FU 1, 6 | *"Playlists consumed goes up when playlists get worse."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 7 | Popularity, exposure loop, skip ambiguity, artist gender and region, households |
| **F2a** | Something fails → a fallback | §9, FU 2 | Taste-cluster pipeline; a familiar mix after skip-through |
| **F2b** | Latency too tight → design for it | §3, §4, FU 3 | Prefetch at track 8, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4, FU 5 | A LoRA-tuned audio encoder on co-listening pairs |
| **F3a** | Efficiency with a quality trade | §6 | Weekly pools + a cheap session re-rank; int8 embeddings |
| **F3b** | Scale levers | §3, §6 | Sharded pools; per-region batch; mid-week top-ups |
| **F3c** | Extension, with cost and maturity risk | §9 | Conversational steering, context, podcasts |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Ready before track 10; batch done before local Monday |
| **I1b** | Reliability | §6, §8 | Next playlist on the device; fallback chain; availability re-checked |
| **I1c** | Usability | §2, §8, FU 2 | Reasons, controls, a flowing order, offline |
| **I1d** | Scalability | §3, §8 | ~1.2 TB of pools; per-region batch |
| **I1e** | Privacy | §8, FU 7 | No sensitive inference; exclusion controls; private sessions; kids' profiles |
| **I1f** | Safety | §8 | Explicit setting; policy-flagged tracks excluded |
| **I1h** | Security | §8, FU 9 | **Stream farming** and AI-generated spam |
| **I1i** | Ethics / bias | §7, §8, FU 5, 8 | Emerging artists; gender and region; paid promotion disclosed and capped |
| **I1j** | Explainability | §2, §8 | Reasons from slot provenance; grounded descriptions |
| **I2a** | Offline → online | §7, FU 10 | Future-discovery replay → OPE → weekly-cohort A/B → holdback |
| **I2b** | The success definition | §7, FU 6 | *"Sticky discoveries up, novelty held, listening grown — not cannibalised."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) two clocks → §4 · (d) data → §5.1 · (e) labels → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
