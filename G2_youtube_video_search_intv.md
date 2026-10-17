# G2 · YouTube Video Search — **INTERVIEW SHEET**

> **Tier 2 · #11** — search ranking, the third classic alongside reco and ads.
> Full study file: [`../G2_youtube_video_search.md`](../G2_youtube_video_search.md) · [diagrams](../G2_youtube_video_search_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the search stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Video or **moment**? Head vs tail? Freshness? (§1) |
| **Explainability** is easy points | The **matched transcript line + timestamp** on each result, and creator-side search analytics (§8) |
| **Security** beyond the gateway | **Keyword-stuffed metadata** and **fake views** inflating the engagement prior (§8) |
| **Troubleshoot before being asked** | Raise the **re-rank budget** (route by query type) and **tail invisibility** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **The pixels are the least useful modality** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Results with moments · ⚠ **the label is watch-through, not clicks** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Power law → **head/tail indexing asymmetry** · re-rank cost | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · hybrid retrieval → fuse → multi-objective re-rank · routing | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Unit of retrieval · ASR · watch-through labels · the objective | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Moment-level for the head only · freshness · fast path for navigational | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Human judgements per slice · interleaving · tail coverage · ASR equity | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Visual + cross-lingual · the poor-metadata path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"I want to lead with the fact that should shape this design, because it's counterintuitive and it decides where the money goes: for the large majority of video-search queries, the pixels are the least useful modality.**
>
> **Title, description and the ASR transcript carry most of the retrievable signal, and engagement carries most of the ranking signal. Video understanding is enormously expensive and largely redundant with the text that already accompanies the video.**
>
> **So I'd build text-first retrieval and spend the visual-encoding budget only where text demonstrably fails — genuinely visual queries, moment-level search, and videos with poor metadata. Maybe 15% of queries, and I'd rather serve that slice well than encode ten billion videos uniformly."**

**Then two commitments:**
1. **Settle the unit of retrieval** — video-level answers *"which video"*; **moment-level** answers *"which second"*, which is the differentiated product and is essentially RAG over transcripts. Build video-level for everything, **moment-level only for the head**, because views follow an extreme power law.
2. **⚠ The label is watch time, not clicks.** *A click followed by a three-second watch is a **negative**.* Train on CTR and you optimise for thumbnails that promise what the video doesn't deliver — very effectively.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the results page and the label, capacity, the architecture, data and models, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Does "search" mean finding a video, or finding a moment inside one?** *(Two orders of magnitude difference in index size.)*
2. **How important is the tail vs the head?** *(Views follow a power law — this is the central cost decision.)*
3. **Is freshness a requirement?** *(Breaking-news search is a different retrieval problem from evergreen.)*
4. **Personalised, or the same results for everyone?** *(Personalisation imports recommendation signals, and their risks.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Product | YouTube search: a typed query → a ranked list of videos, with moments for long videos |
| Corpus | ~10⁹ videos; ~500 hours uploaded per minute |
| Traffic | Assume billions of queries/day → tens of thousands of QPS, more at peak |
| Query mix | Navigational, informational, visual, moment-seeking — roughly in that order of volume |
| Personalisation | Light (language, history with activity settings on); none on made-for-kids content |
| Freshness | Titles searchable in seconds, transcripts in minutes |

**FR:** hybrid retrieval over text + transcript · multi-objective re-ranking · moment-level for head content · freshness · query understanding (entities, intent).

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Query latency p95 | ≤ **200 ms** |
| Corpus | ~10⁹ videos, growing continuously |
| Freshness | New upload searchable in **minutes** |
| **Ranking objective** | **Watch-through**, not CTR |
| Availability | 99.9%; a lexical + popularity fallback when the learned stack is down |
| Cost | $ per 1,000 queries, dominated by the re-ranker |

**Scope / NON-GOALS:**
- ❌ **Uniform visual encoding of the whole corpus.** Position, with the cost reason.
- ❌ **Optimising CTR.** It has a known degenerate solution (clickbait).
- ❌ Recommendation (G13 — no query) · content moderation (G25) · live-stream search.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
QUERY: "how to replace a dishwasher pump"
  [thumb] Dishwasher pump replacement — full guide       18:42   1.2M views
          "…first disconnect the power, then remove the lower spray arm…"
           ↑ the matched transcript line, so the result explains itself
  ▶ KEY MOMENT  "Removing the old pump"  → jumps to 6:15        ← moment-level (head videos)
  [thumb] Bosch dishwasher not draining? Check this       9:03
  …
  "did you mean…", spelling correction, filters (duration, upload date)
```

**The metrics:**

```
⚠ THE LABEL DEFINITION IS THE HIGHEST-LEVERAGE DECISION IN THE PIPELINE.

AVAILABLE                      MEANINGFUL
CTR on results      ────────►  WATCH-THROUGH (clicked AND watched past
                                a meaningful threshold)
                               reformulation rate (did they try again?)
                               long-click / satisfied session

⚠ A CLICK FOLLOWED BY A 3-SECOND WATCH IS A NEGATIVE.
  Train on CTR → optimise thumbnails and titles that PROMISE what the
  video does not deliver. The model will do this very effectively.

POSITIVE = clicked AND watched past threshold (normalised for length)
NEGATIVE = clicked and abandoned fast  ← the signal CTR throws away
```

**Slice the metric by query type:** navigational (*"nirvana smells like teen spirit"*) · informational (*"how to fix a dishwasher"*) · **visual** (*"dog doing a backflip"*) · **moment** (*"the part where he explains pricing"*). They have different achievable ceilings and different best retrievers.

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Watch-through rate** on clicked results | The user found what they searched for |
| Failure signal | **Reformulation rate** · fast-abandon rate | Volunteered negatives |
| Offline | Human relevance judgements (NDCG@10) per query-type slice · recall@k for retrieval | The gate |
| **Guardrails** | **Tail coverage** (share of the corpus reachable by any query) · ASR quality by language · p95 latency · $/1k queries | Silent failures made visible |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **⭐ head/tail asymmetry is the cost decision** `A1b` `I1a` `I1d`

```
CORPUS ~10⁹ videos.  VIEWS FOLLOW AN EXTREME POWER LAW:
  the top ~1% of videos is most of the watch time.

→ UNIFORM INDEXING DEPTH IS THE WRONG DEFAULT.

  ALL VIDEOS:  title + description + tags + ASR transcript
               → lexical + dense index         (cheap, ~10⁹)
  HEAD ~1%:    + MOMENT-LEVEL transcript chunks (~10⁷ videos × ~50 chunks
               = ~5×10⁸ chunks — comparable to the video-level index)
               + visual encoding where it pays
  TAIL:        text-only, and that's the right answer

⚠ Moment-level over the WHOLE corpus would be ~5×10¹⁰ chunks — two
  orders of magnitude more, for content almost nobody searches.

ASR is the big batch cost: transcribe 10⁹ videos, prioritised by
  expected views, not upload order.

QUERY-SIDE COST: tens of thousands of QPS × ~100 re-ranked candidates
  = millions of cross-encoder scorings/s → the re-ranker is the bill,
  which is why navigational queries get a cheaper path (§4)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Modality | Uniform visual encoding vs **text-first, visual only where it pays** | Text carries most of the signal; spend pixels on the ~15% visual / moment / poor-metadata slice |
| Retrieval | BM25 only vs dense only vs **hybrid** | Literal queries need BM25; paraphrase needs dense; recall wants the union |
| Unit | Video-level only vs moment-level everywhere vs **moment-level for the head** | Moment-level everywhere is 100× the index for content nobody searches |
| Label | CTR vs **watch-through** | CTR trains clickbait |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE / NEAR-LINE ══════════════════════════════════════════════════════
 UPLOAD → title / description indexed in SECONDS
        → ASR transcript in MINUTES (prioritised by predicted views)
        → text embeddings → lexical + dense indexes (hash-sharded)
        → HEAD videos (by view VELOCITY): transcript chunks → moment index
        → engagement prior: watch-through per impression, invalid views removed

══ ONLINE · per query (~200 ms) ═════════════════════════════════════════════
 QUERY
       │
┌─ 1. QUERY UNDERSTANDING ───────────────────────────────────────
│ entities · intent · TYPE: navigational / informational / visual / moment
│ OUTPUT: a query plan (which retrievers, whether to re-rank fully)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. HYBRID RETRIEVAL (parallel) ───────────────────────────────
│ BM25 over title + description + TRANSCRIPT (⚠ hard to beat on literal
│ queries) · DENSE over text / transcript embeddings · ENGAGEMENT PRIOR
│ OUTPUT: candidates from each, fused with RRF (ranks, not scores)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. (HEAD ONLY) MOMENT-LEVEL RETRIEVAL ────────────────────────
│ transcript chunks → timestamped moments
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. MULTI-OBJECTIVE RE-RANKER ─────────────────────────────────
│ p(click) · p(WATCH-THROUGH) · freshness · quality
│ ⚠ negative head: p(fast abandon)
│ OUTPUT: ~20 results (+ timestamp deep links for moments)
└──────┬─────────────────────────────────────────────────────────
       ▼
 LOGGING: click + WATCH DURATION + reformulation
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **The re-rank budget:** *"The cross-encoder is ~100 of the 200 ms and most of the cost. Navigational queries (a song title, a creator) get a **fast lexical path with the popularity prior** and skip it; the expensive path is spent where it changes outcomes."*
- **Tail invisibility:** *"Poor ASR plus a popularity prior can make whole categories unreachable, and nothing complains. So tail coverage and ASR quality by language are panels, and poor-metadata videos get the visual backstop (§9)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Text-first, visual only where it pays** | "Pixels are largely redundant with the text that already accompanies the video — I'd rather serve the 15% visual slice well than encode 10⁹ videos uniformly" | Genuinely visual queries on tail content are served poorly |
| **2** | **Head/tail indexing asymmetry** | "Views follow an extreme power law; moment-level over the whole corpus is 100× the index for content nobody searches" | A tail video's best moment isn't findable |
| **3** | **Keep BM25 in the mix** | "Video queries are frequently literal — *'nirvana smells like teen spirit live 1992'* — and BM25 over a transcript is extremely hard to beat there" | Two indexes to maintain |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Video metadata** | Title, description, tags, channel | Keyword stuffing; misleading titles |
| **ASR transcripts** | The main retrievable text, and the moment index | Quality varies wildly by language and audio |
| **Search logs, with position** | Query → shown → clicked → **watch duration** → reformulation | Position bias; only shown results get labels |
| **Engagement stats** | Watch-through per impression | Fake views; popularity feedback loop |
| **Human relevance judgements** | Graded labels on a stratified query sample | Expensive; the offline gate |

### 5.2 Choosing the training set (e) `F1c`

```
1. LABELS       positive = clicked AND watched past a threshold (normalised for
                length) · negative = clicked and abandoned fast · reformulation
                = a volunteered negative · not-clicked ≠ irrelevant
2. CLEAN        remove invalid views (bots) before they shape labels or the prior;
                collapse re-uploads of the same video
3. DEBIAS       position as a training-only input; an exploration slice where
                under-exposed videos are shown, so the tail can earn labels
4. STRATIFY     by query type and by language, so navigational volume doesn't
                drown the hard slices
5. SPLIT BY TIME train on past weeks, validate on the next
```

**Labels and their bias:**
- **⚠ Watch-through, not CTR** (§2) — the single highest-leverage decision.
- **Position bias** — shallow tower, dropped at serving.
- **Popularity bias** — the engagement prior makes popular videos more findable, which makes them more popular. Needs an **exploration allocation** or the tail becomes unreachable.
- **ASR quality varies wildly** by language, audio quality, and domain jargon — and transcript quality directly caps retrievability. **A bad transcript makes a video invisible**, which is a silent quality failure concentrated in exactly the languages with least investment.

### 5.3 The models and the objective (f) `A2b` `F1a`

- **Retrieval:** BM25 + a **dense two-tower** (query tower, video-text tower) trained with a **contrastive loss, in-batch negatives plus BM25-mined hard negatives** on (query, watched-through video) pairs.
- **Re-ranker:** a cross-encoder over (query, title, description, transcript span, engagement features) with a **pairwise ranking loss** on watch-through, plus heads for p(click) and **p(fast abandon)**, combined with owned weights.
- **Moments:** transcript chunks embedded with the same text encoder; the moment is located by the best-matching chunk.

### 5.4 Fine-tune an existing model (f) `F2c`

Start the text towers from a pretrained multilingual encoder and adapt them with **LoRA** on (query, watched video) pairs. Multilingual pretraining is what keeps low-resource languages from collapsing, and adapters let each major language get its own small tune on one shared base.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · INGEST    metadata (seconds) → ASR (minutes, by predicted views) → embed
STAGE 2 · TIER      head by view VELOCITY → moment chunks (minutes, with an SLO)
STAGE 3 · LABEL     search logs → watch-through labels, invalid views removed
STAGE 4 · TRAIN     dense retriever + re-ranker, warm-started
STAGE 5 · GATE      human judgements per query-type slice → interleaving → A/B
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
LATENCY (~200 ms):
  query understanding      ~20 ms
  BM25 ∥ dense ∥ prior     ~40 ms
  fuse                      ~5 ms
  RE-RANK                 ~100 ms   ← dominates
  assemble + deep-links    ~10 ms

FRESHNESS: new upload → title/description indexed in SECONDS;
  ASR transcript follows in MINUTES (a separate pipeline with its
  own SLO, prioritised by predicted views).

SHARDING: hash by video id, scatter-gather, replicate.
  ⚠ NOT by topic — recall cliffs at boundaries.
```

- **Route by query type** `F3b`: navigational → lexical + prior, no cross-encoder; the full path only for informational, visual and moment queries.
- **Efficiency** `F3a`: a **distilled, quantized** cross-encoder (int8), checked per slice so the hard queries don't pay for the savings.
- **The ranking objective** `A3b`: `w₁·p(watch-through) + w₂·p(click) − w₃·p(fast abandon) + freshness`, with owned weights and an **exploration allocation** for under-exposed videos on queries whose top results are already confidently good.
- **Reliability** `I1b`: if the dense index or the re-ranker is down, serve lexical + prior with a flag; ASR backlog has its own alarm, because a stalled transcript pipeline silently hides new videos.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline** | **Watch-through rate**, not CTR |
| **Reformulation rate** | The user tried again = we failed, and it's a *volunteered* negative |
| **Sliced by query type** | Navigational / informational / visual / moment — different ceilings |
| **Human relevance judgements** | On a stratified query sample — the offline gate |
| **Tail coverage** | Are tail videos reachable at all? A popularity-prior system degrades this silently |
| **ASR quality by language** | Directly caps retrievability; a gap here is an equity issue |

**Rollout** `I2a`: human judgements per slice (blocking) → **interleaving** (fast, low-variance ranker comparisons) → A/B on watch-through and reformulation, gated **per slice**, because aggregates are dominated by the easy navigational queries. Clicks only exist for what we showed, so offline comparisons use the exploration slice or IPS, not raw logs.

**Success** = watch-through up and reformulation down **on every query-type slice**, with tail coverage and per-language quality not falling. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Objective bias** | CTR rewards clickbait | Watch-through labels; a fast-abandon head |
| **Popularity** | The prior makes the popular more findable | Exposure-normalised prior; exploration; tail-coverage panel |
| **Language / ASR** | Bad transcripts hide videos in some languages | ASR quality by language; visual and metadata backstop |
| **Position** | Top results get clicked anyway | Shallow position tower; interleaving |
| **Easy-slice dominance** | Navigational volume hides regressions on hard queries | Per-slice gates |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 200 ms; the re-ranker dominates; cost per 1,000 queries | p95 per stage · $/1k queries |
| **Reliability** `I1b` | Lexical + prior fallback when the learned stack is down; ASR-backlog alarm; replicas | Fallback rate · transcript lag |
| **Usability** `I1c` | **Timestamp deep links**, the matched transcript line in the snippet, spelling correction, filters | Reformulation rate · deep-link use |
| **Scalability** `I1d` | Head/tail indexing tiers; hash sharding; ASR prioritised by predicted views; routing by query type | Index size · QPS headroom |
| **Privacy** `I1e` | Personalisation only with search/watch history on; none on made-for-kids content; query-log retention limits | Policy audits · retention compliance |
| **Safety** `I1f` | SafeSearch / Restricted Mode; for news and health queries, **authoritative sources boosted** and borderline content demoted | Policy-violating results on sensitive queries |
| **Security** `I1h` | ⚠ **Keyword-stuffed or misleading metadata**; ⚠ **fake views** inflating the engagement prior → invalid engagement removed before it shapes the prior | Spam rate · invalid-view share |
| **Ethics / bias** `I1i` | **ASR quality by language** is an equity issue; tail and new creators need reachability | ASR quality by language · tail coverage |
| **Explainability** `I1j` | The **matched transcript line + timestamp** explains each result; creators see which searches found their video; feature attributions for ranking debugging | % results with a highlighted match |

> ⭐ *"In search, a bad transcript makes a good video invisible, so production-readiness includes the creators whose content we can't hear."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Visual queries** ("dog doing a backflip"): a CLIP-style text-to-frame index for the **head**, where pixels pay. Cost: a vision encoder per indexed frame. Risk: frame embeddings encode scenes better than actions, so they help "what's in it" more than "what happens".
- **Cross-lingual search:** query in one language, content in another, via multilingual embeddings over transcripts; quality is uneven across language pairs, so slice it.

**When text fails for a video (poor transcript, thin metadata), the fallback pipeline** `A5b` `F2a`:

```
ASR confidence low or metadata thin
  → lean harder on TITLE, CHANNEL and ENGAGEMENT signals for that video
  → VISUAL ENCODING as the backstop for exactly this slice
  → re-transcribe with a better model if the video starts getting views
  → watch tail coverage and per-language retrievability, so the gap is visible
```

**Top 3 failure modes:**
1. **Clickbait optimisation** — the direct consequence of a CTR objective; the model gets very good at it.
2. **Tail invisibility** — poor ASR plus a popularity prior means whole categories of content are unreachable, and no metric complains.
3. **Moment-level index staleness** — the head shifts as videos go viral, so what's "head" changes and the moment index has to follow.

**Q1 ships:** hybrid text retrieval + **watch-through labels** + the multi-objective re-ranker.
> *"Label definition first, because everything downstream trains on it — and a CTR-trained model is very hard to undo once it's shaped the corpus's thumbnails."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not encode the video content? It's video search."** `A5a` `F3c`
> Because for most queries the pixels are largely **redundant** with text that already exists — title, description, tags, and the ASR transcript carry most of the retrievable signal, and engagement carries most of the ranking signal. Encoding ten billion videos visually is enormously expensive and buys little on the majority of traffic. What it *does* buy is a specific, identifiable slice: **genuinely visual queries** ("dog doing a backflip"), **moment-level search**, and videos with poor or missing metadata — maybe 15% of queries. So I'd spend the visual budget there rather than uniformly, which is a much better return. The framing I'd use: this is a **coverage decision, not a capability decision** — I'm not saying visual understanding doesn't work, I'm saying uniform application of it is the wrong allocation against a power-law corpus.

**2. "What's your label, and why does it matter so much?"** `F1b` `F1c`
> **Clicked *and* watched past a meaningful threshold**, normalised for video length — not clicks. This is the highest-leverage decision in the whole data pipeline. If I train on CTR, the model learns to surface results with compelling thumbnails and titles, and it will do that **very effectively** — which means optimising for content that promises something it doesn't deliver. A click followed by a three-second watch is not a weak positive, it's a **negative**: the user was misled and came back. So the positive is watch-through, the fast abandon is an explicit negative head, and **reformulation** — the user searched again — is a volunteered negative I'd also weight. The organisational point: CTR is the metric that's easiest to move and easiest to report, so this definition needs to be settled before anyone is measured on it.

**3. "Video-level or moment-level retrieval?"** `A1b` `A5a`
> Both, but not uniformly — and settling this early matters because it changes the index size by two orders of magnitude. **Video-level** answers "which video" and is the right default for the whole corpus. **Moment-level** answers "which second," which is the genuinely differentiated product and is essentially **RAG over transcripts** — chunk the transcript, embed, retrieve, return a timestamped deep link. But moment-level over ten billion videos is roughly 5×10¹⁰ chunks, a hundred times the video-level index, for content almost nobody searches. So: **moment-level for the head** — the top ~1% of videos, which is most of the watch time — and video-level for everything else. **Head/tail asymmetry in indexing depth is the central cost decision here**, not a micro-optimisation.

**4. "Why keep BM25 when you have embeddings?"** `A5a` `F1a`
> Because a large share of video queries are **literal**, and lexical retrieval is extremely hard to beat on those. "Nirvana smells like teen spirit live 1992" is a navigational query with exact tokens — a title, an artist, a year — and BM25 over title plus transcript nails it, while a dense retriever may return semantically similar but wrong performances. Dense earns its place on paraphrase and description-style queries where the user's words don't match the video's. They **fail differently**, which is exactly the argument for keeping both and fusing with RRF: since the first stage's job is recall, I want the union rather than the better of the two. RRF also handles the fact that a BM25 score and a cosine similarity aren't comparable — it uses ranks, not scores.

**5. "A great video has a terrible transcript. What happens?"** `F2a` `I1i`
> It becomes **hard to find**, and that's a real and under-discussed failure mode. ASR quality varies enormously by language, audio quality, accent, and domain jargon, and since the transcript is the primary retrieval signal for most queries, **a bad transcript makes a good video effectively invisible**. What makes it worse is that it's silent — no metric complains, the video just gets no traffic, and the effect concentrates in exactly the languages and communities with the least ASR investment, which makes it an equity issue rather than only a quality one. Mitigations: **monitor ASR confidence and retrievability by language** as a first-class panel; fall back harder on title, description and **engagement signals** where transcript confidence is low; and use **visual encoding as the backstop** for exactly this slice — poor metadata is one of the three cases where pixels genuinely earn their cost.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you stop the engagement prior from making popular videos permanently more findable?"** `F1c` `I1i`
> You can't eliminate it — popularity genuinely *is* evidence of quality, and throwing it away makes results worse — so the goal is to stop it becoming **self-fulfilling**. The mechanism to name: a video that ranks well accumulates views, which strengthens its engagement prior, which makes it rank better. Left alone, the ranking converges on incumbents and the tail becomes unreachable. Three counters: **normalise engagement by exposure** rather than using raw views, so the prior estimates quality-per-impression rather than accumulated luck; hold an **exploration allocation** that surfaces under-exposed content on queries where the top results are already confidently good, since that's where exploration is cheapest; and **monitor tail coverage directly** — what fraction of the corpus is reachable by any query — because that degrades silently and no relevance metric will flag it. I'd also note this is the same structural problem as new-creator reach in recommendation, and it deserves an owned guardrail for the same reason.

**7. "The head shifts — a tail video goes viral. What breaks?"** `F2a` `I1b`
> The **moment-level index**, which was built on a head definition that's now stale. A newly viral video is exactly the one people are searching inside, and it's the one without transcript chunking — so the differentiated product fails precisely at the moment of peak demand. So "head" can't be a static list recomputed monthly; it needs to be **demand-driven and fast**: trigger moment-level indexing on a **velocity signal** (view acceleration, search volume against the video), not on cumulative views, since cumulative views lag by definition. I'd treat it as a **promotion pipeline** with an SLO — from "this video is trending" to "its moments are searchable" in minutes — and I'd allow **query-triggered indexing** as a backstop, where repeated moment-style queries against an unindexed video enqueue it immediately. The general principle: when the distribution has a heavy tail *and* moves, the tiering policy has to be reactive rather than scheduled.

**8. "Search and recommendation both rank videos. Should they share a model?"** `A5a` `A2b`
> They should share **infrastructure and signals**, not the objective — and conflating them is a common and expensive mistake. Search has a **query**, which is a stated intent, so relevance to that intent dominates and satisfying it quickly is a *success* (a short session can be a win). Recommendation has no query, so it's inferring intent, and session length reads very differently. If I use the recommendation ranker for search, I import an engagement objective into a surface where the user told me what they wanted — and I'll start returning engaging-but-irrelevant results, which is the clickbait failure by another route. What I'd share: the **video representations**, the **transcript and ASR pipeline**, the **quality and engagement features**, and the **serving infrastructure**. What I'd keep separate: the **label definition** (watch-through against a stated query vs satisfaction without one), the **objective weights**, and the **evaluation sets**. Shared features, separate heads, separate gates.

**9. "Half your traffic is navigational. Does the ML matter at all?"** `A1c` `I2b`
> Less than people expect, and saying so is the honest answer — a large share of video search is someone typing a song title or a creator name, where **lexical retrieval plus an engagement prior is close to optimal** and a learned ranker adds little. That has a real design consequence: I'd **route by query type** rather than sending everything through the expensive path. Navigational queries get a fast lexical path with a popularity prior and can skip the cross-encoder entirely — which saves most of the latency budget and most of the cost. The ML budget then concentrates on the queries where it changes outcomes: informational, visual, and moment-level. The trap I'd flag is that **aggregate relevance metrics are dominated by the easy slice**, so a model change that improves the hard 20% and slightly regresses the easy 80% looks like a loss, and a model that games navigational queries looks like a win. That's why the metric is **sliced by query type**, with per-slice gates rather than one number.

**10. "What would you do differently if this were podcast or lecture search instead?"** `F3c` `A3a`
> The balance shifts decisively toward **moment-level**, and that changes the cost calculus I argued for earlier. For music and short-form video, "which video" is usually the whole question; for a two-hour lecture or podcast, "which video" is nearly useless and **"which minute" is the product** — so moment-level stops being a head-only luxury and becomes the primary index, which I'd fund accordingly. Second, **transcript quality becomes the entire ceiling** rather than one signal among several, since there's no thumbnail, no visual content, and often thin metadata — so ASR investment and speaker diarisation move from supporting infrastructure to the core dependency. Third, the **engagement prior gets weaker and noisier**: long-form consumption is partial by nature, so "watched past a threshold" needs a different definition and completion rate is a poor signal. And **visual encoding becomes nearly worthless**, which is the clean version of my opening argument — the modality allocation should follow where the retrievable signal actually lives, and for this corpus it lives almost entirely in the transcript.

---

## ⭐ THE L6 LINE

> **"For most video queries the pixels are the least useful modality — so text-first retrieval, and visual budget only on the 15% where text fails. And the label is watch-through, not clicks, because a click plus a three-second watch is a negative."**

## 🧭 HOOKS — *steer by what you mention*

- *"The pixels are the least useful modality"* → invites the multimodal / cost-allocation branch
- *"A click plus a three-second watch is a negative"* → invites the labels branch
- *"Head/tail asymmetry is the central cost decision"* → invites the capacity/indexing branch
- *"Moment-level search is essentially RAG over transcripts"* → invites the Q6 crossover
- *"A bad transcript makes a good video invisible"* → invites the equity/quality branch (**good at L6**)
- *"Aggregate metrics are dominated by the easy slice"* → invites the evaluation branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the query types"* — navigational / informational / visual / moment regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the label definition and tail coverage than finish the sharding. Sound right?"*

**Name and defer:**
- "**Multilingual and cross-lingual search** — query in one language, content in another — is a large separate problem."
- "**Live-stream search** has no transcript yet and a different freshness model."
- "**Query autocomplete** is its own system and it shapes the query distribution this ranker sees."
- "**Thumbnail selection** interacts with the clickbait problem from the other side."
- "**Personalised search** blends this with recommendation signals and needs care not to import the engagement objective."
- "**Copyright and rights-holder signals** affect what's retrievable, independently of relevance."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: text-first hybrid retrieval, pixels only where they pay | §0, §4 | *"For most video queries the pixels are the least useful modality."* |
| **A1b** | Latency and capacity budgets | §1, §3, §6 | *"200 ms, half of it the re-ranker; moment-level everywhere is 100× the index."* |
| **A1c** | Degrade without lowering quality for everyone | §4 | *"Navigational queries take a fast lexical path and skip the cross-encoder."* |
| **A2a** | The full stack | §4 diagram | Understanding → hybrid retrieval → moments (head) → re-rank → logging |
| **A2b** | Model components | §5.3 | BM25 + dense two-tower; a multi-objective cross-encoder |
| **A2c** | Data sources | §5.1 | Metadata, ASR transcripts, search logs with watch duration, judgements |
| **A2d** | Staged pipeline | §5.5 | Ingest → tier by velocity → label → train → gate |
| **A3a** | Top-down, unprompted | §0 | The pixels line + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Watch-through minus fast-abandon, owned weights, plus an exploration allocation."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Video or moment · head vs tail · freshness · personalisation |
| **A5a** | Alternatives compared | §4, FU 1, 4 | Visual vs text-first; BM25 vs dense vs hybrid; unit of retrieval; label |
| **A5b** | A full alternative pipeline on demand | §9 | The poor-metadata path: title + engagement + visual backstop |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Contrastive with BM25 hard negatives; a pairwise loss on watch-through."* |
| **F1b** | Metrics and their trade-offs | §2 | *"A click followed by a three-second watch is a negative."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Popularity, position, ASR language, easy-slice dominance |
| **F2a** | Retrieval underperforms → a fallback | §4, §9, FU 5 | Tail invisibility; the visual backstop |
| **F2b** | Latency too tight → tiering | §4 | Routing by query type, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A LoRA-adapted multilingual encoder |
| **F3a** | Quantization with a quality trade | §6 | A distilled int8 cross-encoder, checked per slice |
| **F3b** | Scale via tiers and routing | §3, §6 | Head/tail tiers; ASR by predicted views |
| **F3c** | Extension, with cost and maturity risk | §9, FU 10 | Visual and cross-lingual search; podcasts |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 per stage; $ per 1,000 queries |
| **I1b** | Reliability | §6, §8 | Lexical fallback; ASR-backlog alarm |
| **I1c** | Usability | §2, §8 | Deep links; the matched transcript line |
| **I1d** | Scalability | §3, §8 | Tiered indexing; sharding |
| **I1e** | Privacy | §8 | History-based personalisation only with consent |
| **I1f** | Safety | §8 | Authoritative sources on news and health queries |
| **I1h** | Security | §8 | Metadata spam and fake views |
| **I1i** | Ethics / bias | §7, §8 | ASR quality by language as equity |
| **I1j** | Explainability | §2, §8 | The highlighted transcript match; creator search analytics |
| **I2a** | Offline → online, correcting for shown-only labels | §7 | Judgements → interleaving → per-slice A/B; exploration / IPS |
| **I2b** | The success definition | §7 | *"Watch-through up, reformulation down, on every slice."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
