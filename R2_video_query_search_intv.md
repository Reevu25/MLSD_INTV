# R2 · Search YouTube with a Short Video as the Query — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **The prompt:** *"Design a system to search YouTube videos using a short video as the query. Not exact match: semantically similar videos. E.g. a user has a video of a cool breakdance trick and wants to find videos of the same trick in a different context."*
>
> **Expect the same probes as R1:** (a) assumptions · (b) the final product · (c) the inference stack · (d) datasets · (e) downselecting training data · (f) the training objective (PEFT on a video foundation model) · (g) serving · (h) how you'll know it worked · (i) the extension (text + video queries, other domains).
>
> **Related sheets:** [G1 visual search](G1_visual_search_intv.md) (image as query: HNSW + PQ, geometric verification) · [G2 YouTube video search](G2_youtube_video_search_intv.md) (text as query).

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | The first one is *"what does similar mean?"*, and it decides the model (§1) |
| **Explainability** is easy points | Show the matched moment, a move label, and a **pose-skeleton overlay** (§8) |
| **Security** beyond the gateway | **Sandboxed video decoding** (codec exploits), index scraping, and **misuse as a person-finder** (§8) |
| **Troubleshoot before being asked** | Raise *"generic embeddings return the same scene, not the same move"* and *"the model learns to match the dancer, not the trick"* yourself (§4, §5) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **Same action, different context** → appearance-invariant, motion-sensitive · **index moments, not videos** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · scope | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Trim → results as moments with timestamps · **context diversity** as a metric | I1c, F1b, I2b |
| **9–13** | **CAPACITY** | — | ⚠ You can't index every 2 s of YouTube → **tiered index + moment proposals** | A1b, I1a, I1d |
| **13–23** | **ARCHITECTURE** | **(c)** | Indexing pipeline + query path · **4 options compared** · temporal reranker · confidence gate | A2a, A5a, A1c, F2b |
| **23–33** | **DATA & MODEL** | **(d) (e) (f)** | Weak labels · **multi-camera positives** · **shortcut negatives** · contrastive + PEFT · pipeline | A2b, A2c, A2d, F1a, F2c |
| **33–38** | **SERVING** | **(g)** | Client trim + downscale · quantize · PQ · cache viral queries · **MMR on appearance** | F3a, F3b, A3b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Expert-labelled pairs · shadow → vertical A/B · bias incl. pose-model fairness | I2a, I2b, F1c |
| **44–48** | **PRODUCTION READINESS SWEEP** | — | Privacy (**not a person-finder**) · safety · security · ethics · explainability | I1b, I1e–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Video + text queries · other domains · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES / YOUR QUESTIONS** | — | Harder follow-ups | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The word that defines this problem is 'similar', so I want to pin it down first. The user wants the same trick in a different context: different dancer, venue, lighting, camera angle, music. So the representation has to be invariant to appearance and sensitive to motion, which is the opposite of what a generic image or video embedding does. Those mostly encode the scene, and would return the same venue or the same outfit.**
>
> **The second decision is the unit. A trick lasts three seconds inside a twenty-minute battle video, so I'll index moments, not videos, and return a timestamp, not just a link.**
>
> **So the critical path is: encode the query's moves with a motion- and pose-based embedding, retrieve candidate moments from an approximate-nearest-neighbour index, re-rank by temporally aligning the sequence of moves, and then deliberately diversify the context, because a page of re-uploads of the same clip is exactly what the user didn't ask for."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the product, metrics, capacity, architecture, data and training, serving, evaluation, a production-readiness sweep, then extensions."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What does "similar" mean?** The same specific move (fine-grained), the same activity (any breakdancing), or the same scene? *(Decides the model. Assume: the same move.)*
2. **What's the query?** A phone clip the user recorded (5–30 s)? A trimmed moment of a YouTube video? Does it contain several moves?
3. **Return videos or moments?** *(Assume: moments, with a timestamp deep link.)*
4. **Which corpus?** All public YouTube, or a vertical first (dance, sports, skate, fitness)? How fresh must new uploads be?
5. **Latency:** interactive, or "we'll notify you"? *(Assume interactive.)*
6. **Privacy:** the clip may show the user's friends or children. Can we keep it?

**State assumptions:**

| Assumption | Value |
|---|---|
| Similarity | **Same move / action**, invariant to performer, venue, lighting, clothing, camera angle, music, speed |
| Query | 5–30 s clip from the phone; the user can **trim** to the move |
| Output | **Top 20 moments**, grouped by video, each a deep link to the timestamp |
| Corpus | Public videos only (private and unlisted never indexed); **launch on human-activity verticals**, expand later |
| Freshness | New uploads searchable within ~24 h |
| Scale | ~10⁹ indexable videos, ~500 hours uploaded per minute; feature traffic ~10⁶–10⁷ queries/day |
| Query clip | Processed **ephemerally**: not stored or trained on without consent |

**FR:** upload and trim a clip · moment-level semantic retrieval · temporal localisation in results · context diversity · near-duplicate removal · policy and availability filtering · explanation of each match · feedback ("not this move") · a text fallback when confidence is low.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Query latency** | **P95 ≤ 2 s after upload completes** (upload kept small by client-side trimming) |
| Freshness | New uploads indexed within 24 h |
| Index cost | Bounded by a **tiered** index (§3), tracked as $ per indexed hour |
| **Policy-violating result rate** | ~0 on audited samples (hard gate) |
| **Identity-leakage rate** | Retrieving the *same person* no more often than chance (hard gate, §8) |
| Availability | 99.9%; on failure, fall back to text-based results |

**Scope / NON-GOALS:**
- ❌ **Exact-match / copy detection.** That's Content ID. Here near-duplicates are a **filter**, not the goal.
- ❌ **Finding a person.** We match **moves**, never identities (§8).
- ❌ Text-query video search ([G2](G2_youtube_video_search_intv.md)); recommendations.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**The product (b):**

```
1. USER records or picks a clip         2. TRIM to the move (optional but powerful)
   [▶ my_friend_airflare.mp4  0:18]        |◀━━━━[■■■■]━━━━━━▶|  0:06 – 0:09

3. RESULTS: moments, not videos, each starting AT the move
   ┌────────────────────────────────────────────────────────────┐
   │ ▶ [looping 3-s preview]  "Bboy X – Red Bull BC One"  @2:41 │
   │   Same move: AIRFLARE · outdoor · different dancer         │
   │   [show skeleton comparison]                               │
   ├────────────────────────────────────────────────────────────┤
   │ ▶ [looping 3-s preview]  "Airflare tutorial for beginners" │
   │   Same move: AIRFLARE · tutorial · slow motion  @0:55      │
   └────────────────────────────────────────────────────────────┘
   filters: [tutorials] [performances] [slow-mo]   ·   👎 "not this move"
```

**Why trimming matters:** a 30 s clip may contain five moves. Letting the user mark the one they mean removes the biggest ambiguity for almost no cost. When they don't trim, propose the most "move-like" segment and let them adjust.

**Metrics:**

| Layer | Metric | Why |
|---|---|---|
| **North star** | **Successful search rate**: a result watched past the matched moment, or rewatched, or saved | The user found the move |
| Failure signal | **Re-trim / re-query rate** · 👎 "not this move" rate | Users retry when it's wrong |
| Offline, retrieval | **Recall@100** of a same-move moment | Can the reranker even see it? |
| Offline, ranking | **NDCG@10** with **graded relevance** (same move > same move family > same dance style > unrelated) · mAP | Fine-grained ordering |
| Offline, localisation | **Temporal IoU** of the returned timestamp vs the true move | Deep links must land on the move |
| ⭐ **Requirement-specific** | **Context diversity@10**: distinct performers / venues among the relevant results · **near-duplicate rate** (should be ~0) | The user asked for *different context* |
| Model health | **Calibration** of the similarity score | The confidence gate thresholds on it |
| **Guardrails** | P95 latency · cost/query · policy-violating result rate · **identity-leakage rate** | Hard limits |

---

## 3 · CAPACITY *(min 9–13)* — **⭐ the insight that reshapes the design** `A1b` `I1a` `I1d`

```
NAIVE: embed every 2-second window of YouTube
  ~10⁹ videos × ~10 min × 1 window / 2 s  ≈ 3×10¹¹ windows
  × 128 B each (int8, 128-d)              ≈ 40 TB of vectors
  and most windows contain no human action at all
  ⚠ NOT FEASIBLE, and mostly useless

TIERED (order-of-magnitude):
  TIER A · VIDEO-LEVEL, every video (reuse YouTube's existing video embeddings)
           10⁹ × 128 B                                   ≈ 128 GB
  TIER B · MOMENT-LEVEL, only HUMAN-ACTIVITY videos (classifier: ~10–20%)
           ~10⁸–2×10⁸ videos × ~30 MOMENT PROPOSALS each
           (motion peaks + shot boundaries, not a fixed grid)
           ≈ 3–6×10⁹ moments × 64 B (product-quantised)  ≈ 200–400 GB
           → sharded by vertical, replicated; fits a few dozen index servers

INDEXING NEW UPLOADS
  500 h/min ≈ 720k hours/day → ~100k hours/day of human-activity video
  cheap motion features find proposals first; the EXPENSIVE encoder runs
  only on proposals → a dedicated pool of tens to low hundreds of GPUs.
  Backfilling the existing corpus is a one-off batch job.

QUERY TRAFFIC: ~10⁶–10⁷/day ≈ 10–100 QPS avg, a few hundred at peak
  → small; cost per query is dominated by GPU encode + rerank

QUERY LATENCY BUDGET (after upload), target ~0.9 s, P95 ≤ 2 s:
  gate + safety hash scan                    50 ms
  decode + downsample (224p, 8 fps)         100 ms
  pose estimation (~80 frames, batched)     100 ms
  query encode (video + pose, int8)         100 ms
  ANN fan-out over shards                    50 ms
  temporal rerank of top-200 (GPU)          200 ms
  filters, dedupe, diversity, grouping       50 ms
  network / overhead                        200 ms
  (text-caption source runs IN PARALLEL; dropped if it misses the deadline)

UPLOAD: trim + downscale ON THE PHONE to 224p / 8 fps → ~300 KB for 10 s
  → the model needs nothing more, and upload drops from seconds to ~1 s
```

---

## 4 · ARCHITECTURE *(min 13–23)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: four ways to do it `A5a`

| Option | How | Strength | Fatal weakness here |
|---|---|---|---|
| **1. Caption, then text search** | A vision-language model describes the clip ("breakdancer doing an airflare"); run normal YouTube search | Cheap, reuses search, interpretable | Fails on **unnamed / fine-grained moves**; vocabulary gap; titles often don't name the move |
| **2. Generic frame embeddings** (CLIP-style, averaged over frames) | Embed frames, ANN | Off-the-shelf | ⚠ **Appearance-dominated**: returns the same venue, outfit or lighting. **The opposite of "different context"** |
| **3. Motion + pose moment embeddings** | A video encoder trained for action similarity, plus a pose-sequence encoder; index moments | **The right invariance**: motion, not appearance | Expensive to index; needs training data for fine-grained moves |
| **4. Near-duplicate fingerprinting** (Content ID-style) | Perceptual video hashes | Precise, cheap | Finds **copies**, explicitly what the user doesn't want |

> **Decision:** *"Option 3 as the primary retrieval, option 1 as a cheap parallel source and as the fallback when option 3 is unsure, and option 4 as a **filter** that removes copies and re-uploads. Option 2 is the trap: it looks like it works in a demo because it returns breakdancing, but not the trick."*

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE / BATCH · indexing the corpus ═════════════════════════════════════
 NEW UPLOAD (already transcoded by YouTube)
       │
┌─ 1. ELIGIBILITY ───────────────────────────────────────────────
│ public only · not removed / restricted · human-activity classifier
│ OUTPUT: does this video get MOMENT-level indexing? (~10–20% yes)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. MOMENT PROPOSALS ──────────────────────────────────────────
│ shot boundaries + motion-energy peaks + "people moving" detector
│ OUTPUT: ~30 candidate moments per video, each 2–6 s, with times
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MOMENT ENCODERS (GPU) ─────────────────────────────────────
│ a. VIDEO ENCODER: a video transformer tuned for action similarity
│ b. POSE ENCODER: human keypoints over time → a sequence model
│    (appearance-invariant by construction: a skeleton has no outfit)
│ c. fuse → ONE action embedding per moment
│ d. a separate APPEARANCE embedding (scene, people) for diversity
│ e. a perceptual FINGERPRINT for near-duplicate detection
└──────┬─────────────────────────────────────────────────────────
       ▼
 ANN INDEX over action embeddings (product-quantised, sharded by vertical)
 + METADATA STORE: video id, start/end, channel, policy status, regions,
   appearance embedding, fingerprint cluster id

══ ONLINE · per query ════════════════════════════════════════════════════════
 phone: trim + downscale to 224p / 8 fps → upload
       │
┌─ A. GATE ──────────────────────────────────────────────────────
│ validate size, length, codec · decode in a SANDBOX · OAuth · rate
│ limit · safety hash scan (known illegal content → block + report)
│ OUTPUT: a safe, decoded clip
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ B. QUERY UNDERSTANDING ───────────────────────────────────────
│ pick the move (the user's trim, or the strongest proposal) ·
│ pose estimation · the same encoders as offline
│ OUTPUT: 1–3 query action embeddings + the pose sequence
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ C. CANDIDATES (in parallel) ──────────────────────────────────
│ 1. ANN over the moment index → top ~1,000 moments
│ 2. TEXT SOURCE: caption the clip ("airflare") → title/chapter search
│ 3. VIDEO-LEVEL ANN (tier A) for videos without moment indexing
│ OUTPUT: ~1,000 candidate moments, each tagged with its source
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ D. TEMPORAL RERANKER (GPU, top ~200) ─────────────────────────
│ align the query's pose/frame SEQUENCE against each candidate's:
│ dynamic time warping, or a small cross-attention model.
│ Checks the SAME SEQUENCE OF MOVES, at any speed, not just a
│ similar average (G1's geometric verification, in time)
│ OUTPUT: calibrated match score + refined start/end per candidate
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ E. FILTER, DEDUPE, DIVERSIFY ─────────────────────────────────
│ policy + region + age filters · collapse near-duplicates by
│ fingerprint cluster (keep the original upload) · MMR where
│ "similarity" uses the APPEARANCE embedding → different contexts
│ OUTPUT: top 20 moments + metadata (move label, timestamps, reason)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ F. CONFIDENCE GATE ───────────────────────────────────────────
│ high → show moments · medium → mix in text-source results,
│ labelled "related" · low → "No close matches. Try trimming to
│ the move." ⚠ never a page of confident-looking junk
└────────────────────────────────────────────────────────────────
```

> ⭐ **The neat trick in step E:** use **two embeddings for two jobs**. The **action** embedding decides relevance; the **appearance** embedding decides diversity. MMR then picks results that are close in action and far apart in appearance, which is the user's request written as an objective.

### Raise these two problems yourself (before the interviewer does) `F2b` `F2a`

- **"Latency is too tight for a GPU reranker on every query."** → Tier it with the confidence gate: if the ANN's top scores are already well separated and calibrated-high, rerank only the top 50; if they're flat, rerank 200, and let the text source fill gaps. Client-side trimming and downscaling buys back most of the budget.
- **"The embedding returns the same scene, not the same move."** → That's appearance leakage. It's fixed in training (§5: shortcut negatives, background augmentation, pose stream) and contained in serving by the appearance-based MMR.

### The defensible choices

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Moments, not videos** | "The trick is 3 s of a 20-min video" | A much bigger index → tiering and proposals |
| **2** | **Pose stream alongside the video encoder** | "A skeleton has no outfit and no background" | Pose fails in the dark, under occlusion, in baggy clothes → fuse, don't replace |
| **3** | **Temporal reranker** | "Same sequence of moves, at any speed" | GPU cost on the query path, bounded to top-200 |
| **4** | **Diversity on appearance** | "The user asked for different context, so make it the objective" | Slightly lower top-1 relevance |

---

## 5 · DATA & MODEL *(min 23–33)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F2c`

### 5.1 Datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Public action datasets**: Kinetics (broad actions), **Something-Something** (motion-centric), **FineGym** and **Diving48** (fine-grained, built to defeat appearance shortcuts) | Pretraining and fine-grained evaluation | Licences; few dance moves |
| **YouTube titles, descriptions, chapters, timestamped comments** ("airflare at 2:41") | **Weak moment-level labels** at huge scale | Noisy; English-heavy; often name the dancer, not the move |
| ⭐ **Multi-camera footage of the same event** (a battle filmed by several phones) | **Natural positives: same move, same instant, different viewpoint and context** | Needs time alignment (audio sync helps) |
| **Slow-motion replays inside a video** | Positives that differ in **speed** | Detection of replays |
| **Self-supervised video** (no labels) | Masked-video pretraining of the encoder | Learns appearance too, unless fine-tuned against it |
| **Expert-labelled pairs** (dancers, coaches label "same move / different") | The **gold evaluation set**, per vertical | Expensive: use active learning to pick pairs |
| **Post-launch logs**: query → watched results, 👎 "not this move" | Fine-tuning signal | Only shown results get labels (§7) |

### 5.2 Downselecting the training set (e)

```
  1. ELIGIBILITY   public videos only; human-activity verticals; drop
                   content made for kids (privacy); drop removed/restricted
  2. QUALITY       enough resolution, people detectable, pose confidence high,
                   not just a static talking head
  3. ⚠ REMOVE COPIES  collapse re-uploads by fingerprint BEFORE making pairs.
                   If copies become positives, the model learns appearance
                   matching: identical footage IS identical appearance
  4. BALANCE       cap per move (the long tail matters), per channel, per
                   performer; stratify by camera conditions, body types,
                   skin tones, clothing (pose models are weaker on some)
  5. SPLIT         hold out whole PERFORMERS and CHANNELS for validation,
                   so the score can't come from recognising the dancer
  6. HARD NEGATIVES  same performer + different move; same venue + different
                   move (§5.3)
```

### 5.3 The models and the objective (f) `A2b` `F1a` `F2c`

**Encoders** (one moment → one action vector):
- **Video encoder:** a large pretrained **video foundation model** (masked-video pretraining), adapted for action similarity.
- **Pose encoder:** 2D (or lifted 3D) keypoints per frame → a sequence model. Appearance-invariant by construction; weak when people are occluded or poorly lit.
- **Fusion:** concatenate and project, gated by pose confidence (no reliable pose → lean on video).

**The objective: contrastive, with shortcut-killing negatives.**

```
anchor: a moment of move M
POSITIVES (pull together):  the same move M, by a DIFFERENT performer, venue,
                            camera angle or speed (multi-camera, weak labels,
                            slow-mo replays, background-swap augmentation)
HARD NEGATIVES (push apart): the SAME performer doing a DIFFERENT move
                            the SAME venue/scene with a DIFFERENT move

L = InfoNCE / supervised contrastive, with in-batch negatives:
    L = − log [ exp(s(a, p)/τ) / ( exp(s(a, p)/τ) + Σ_n exp(s(a, n)/τ) ) ]
```

> ⭐ **Say:** *"The negatives are where the invariance comes from. If every negative is a different person in a different place, the model can win by recognising the person or the place. Same-dancer-different-move negatives force it to look at the move."*

**If PEFT (probe f):** keep the video foundation model **frozen**, train **LoRA adapters plus a projection head** with the contrastive objective above. Cheaper, iterates fast, keeps general visual knowledge, and lets each vertical (dance, skate, gymnastics) get its own adapter on one shared base. **Then distil** into a smaller encoder for the indexing pipeline, because the big model is too expensive to run over 100k hours a day.

**The reranker:** a small cross-attention model over the query and candidate pose/frame sequences (or dynamic time warping on pose for a cheaper baseline), trained with a **pairwise ranking loss** on expert-labelled pairs and click data. A second head predicts the **start/end** of the move in the candidate (for the deep link).

### 5.4 The staged pipeline `A2d`

```
STAGE 1 · PRETRAIN    self-supervised video model (or take an existing one)
STAGE 2 · PAIRS       weak labels + multi-camera + replays → dedupe copies →
                      positives, hard negatives, performer-held-out splits
STAGE 3 · ADAPT       LoRA + contrastive → distil to the indexing encoder →
                      train the reranker on expert pairs
STAGE 4 · INDEX       moment proposals → embed → PQ → shard → swap in atomically
STAGE 5 · EVALUATE    expert-labelled set per vertical, identity-leakage test,
                      fairness slices → shadow → A/B
CADENCE: new uploads continuously (≤ 24 h) · adapters monthly · re-embed the
         corpus only when the encoder changes (versioned indexes)
```

---

## 6 · SERVING *(min 33–38)* — probe **(g)** `F3a` `F3b` `A3b`

- **Client-side work first:** trimming and downscaling to 224p / 8 fps on the phone; the model needs no more, and upload time dominates otherwise.
- **Precomputed corpus embeddings:** only the **query** is encoded online. The index uses **product quantisation** (64 B per moment) and is sharded by vertical. `F3b`
- **Quantization:** int8 / FP8 for the query encoder and the reranker: lower-precision weights and activations, so less memory and bandwidth per query; cheaper and faster, **gated on the expert eval set**, since fine-grained distinctions are exactly what quantization erodes first. `F3a`
- **Cache viral queries:** the same viral clip gets uploaded by thousands of users. Key a cache on the **query's perceptual fingerprint** and skip the whole pipeline on a hit.
- **MMR on appearance:** `score = λ · action_relevance − (1 − λ) · max appearance-similarity to results already picked`, plus at most N results per channel. `A3b`
- **Reliability:** if the GPU pool is degraded, serve the **text source** with an honest label rather than failing. `I1b`

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE GATES   expert-labelled pairs per vertical: recall@100, NDCG@10 (graded),
                temporal IoU, context-diversity@10, near-duplicate rate,
                identity-leakage test, fairness slices, calibration
SHADOW          run on real queries, log, don't show: latency, cost, overlap
                with the text-source baseline, policy flags
DOGFOOD         a dance / skate community panel rates results side by side
A/B             launch on ONE vertical: successful-search rate, re-query rate,
                "not this move" rate, guardrails → expand vertical by vertical
⚠ SHOWN-ONLY LABELS   users can only click what we showed, in a grid where the
                top-left gets the most clicks → randomise positions on a small
                slice, IPS-weight clicks, and use INTERLEAVING to compare rankers
```

**Success** = successful-search rate up, re-query and 👎 rates down, guardrails held, and the feature retains users who tried it. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position** | The first results in the grid get clicked regardless | Randomised slice, IPS, interleaving |
| **Selection** | Only shown moments get feedback | Exploration; expert labels for evaluation |
| **Popularity** | Famous dancers and viral clips dominate | Per-channel caps; creator-diversity metric |
| **⚠ Representation** | **Pose estimators are weaker on some skin tones, body types, baggy clothing (common in breaking), low light, and on disabled athletes** → their moves are under-retrieved | Recall sliced by these groups on the expert set; augment; fuse with the video stream |
| **Language** | Weak labels come from titles, mostly English | Multilingual titles and comments; the visual stream doesn't need language |
| **Feedback loop** | Clicked results become training data | Exploration traffic; refresh from expert labels |

---

## 8 · PRODUCTION READINESS SWEEP *(min 44–48)* `I1b` `I1e`–`I1j`

| Dimension | What you do | Metric |
|---|---|---|
| **Performance** `I1a` | P95 ≤ 2 s after upload; per-stage budget; GPU cost per query and per indexed hour | P95 · $/query · $/indexed hour |
| **Reliability** `I1b` | Text source as the fallback; index replicas; versioned index swaps with rollback | Error rate · % served by fallback |
| **Usability** `I1c` | Trim UI; results start **at the move**; looping previews; filters (tutorial / performance / slow-mo); "not this move" | Re-trim rate · 👎 rate |
| **Scalability** `I1d` | Tiered index; moment proposals; PQ; distilled indexing encoder; cache for viral queries | Index size · indexing backlog |
| **Privacy** `I1e` | Query clips processed **ephemerally**, not stored or trained on without consent; public videos only; private / unlisted never indexed; **no face recognition anywhere** | Retention audits · **identity-leakage rate** |
| **Safety** `I1f` | Hash-scan query clips (known illegal content → block and report as legally required); filter results by policy status; dangerous-stunt content carries YouTube's existing warnings | Policy-violating result rate |
| **Security** `I1h` | Validation · OAuth · rate limits · ⚠ **sandboxed video decoding** (malformed files exploit codecs) · ⚠ **scraping the index** by query floods · ⚠ **adversarial clips** probing the embedding | Blocked uploads · anomaly alerts |
| **Ethics** `I1i` | ⚠ **Not a person-finder.** Someone could upload a clip of a stranger to find more videos *of them*. Train for identity invariance, **measure identity leakage as a red-team metric**, no face features, rate limits. Plus fairness slices from §7 | Identity-leakage rate · recall parity across groups |
| **Explainability** `I1j` | Each result shows **the matched moment**, a **move label** ("Same move: airflare"), and an optional **side-by-side skeleton overlay** of query vs result; provenance logged per result (source, score, index version) | % results with a label · audit coverage |

> ⭐ **The ethics line that separates L6:** *"The same embedding that finds the same move in a different context will find the same person in a different context if I let it. So identity invariance is a requirement, with a metric, not a hope."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Video + text queries** ("this move, but a beginner tutorial"): composed retrieval. Use the action embedding for the move and the text as a **filter or modifier** on metadata (tutorial, slow motion) or a text-conditioned reranker.
- **Audio:** music is a **confounder** for "same move", so it's excluded from the action embedding. But for **"same choreography to the same song"** it's the strongest signal, a different product mode.
- **Other domains** (skate, gymnastics, parkour, sports technique): the same pipeline with a **per-vertical adapter** and its own expert set. **Cost:** each vertical needs labelled pairs. **Embedding-maturity risk:** fine-grained action embeddings are far less proven than image embeddings, so launch vertical by vertical, gated on expert-set recall.

**When retrieval underperforms, the fallback pipeline** `F2a` `A5b`:

```
low calibrated match score
  → VLM CAPTION of the query clip ("breakdancer, power move, legs in V, spinning on hands")
  → TEXT SEARCH over titles, chapters, timestamped comments
  → RE-SCORE the text hits with the temporal reranker where moment embeddings exist
  → POLICY FILTER + DEDUPE
  → QUALITY GATE: show as "Related videos", or say "no close matches"
```

**Top 3 failure modes:**
1. **Appearance shortcut**: results share the venue or outfit, not the move. Detected by the context-diversity metric and the expert set → shortcut negatives, pose stream, appearance MMR.
2. **Identity leakage**: the system becomes a person-finder. Detected by the red-team metric → retrain, rate limits, a launch gate.
3. **Re-upload flood**: a viral clip's 500 copies fill the page. → fingerprint clustering, keep the original (credits the creator).

**Q1 ships:** one vertical (dance), moment index for that vertical, the pose + video encoder, DTW reranker, fingerprint dedupe, appearance MMR, the text fallback, and the identity-leakage gate. The cross-attention reranker and other verticals follow.

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just embed frames with CLIP and do nearest neighbours?"**
> Because frame embeddings mostly encode appearance: the scene, the clothes, the lighting, the objects. For "same trick, different context", that's backwards: it would rank a different move in the same venue above the same move somewhere else. It also loses time: averaging frames throws away the order of movements, and a trick *is* an order of movements. It's a fine baseline, and a fine source for the appearance embedding I use for diversity, but not the relevance signal.

**2. "The trick lasts 3 seconds in a 20-minute video. How do you find it?"**
> Index moments, not videos. Moment proposals from shot boundaries, motion-energy peaks and a "people moving" detector give ~30 candidates per video instead of 600 fixed windows, and each gets its own embedding with start and end times. The reranker refines the boundaries, so the deep link lands on the move. For videos outside the moment index, a video-level embedding finds them coarsely, and I can compute moment embeddings on demand for the few that make the shortlist.

**3. "Results keep showing the same dancer as the query."**
> The model learned a shortcut: recognising the person is easier than recognising the move. Fix it in the data: hard negatives of the same performer doing a different move, positives of the same move by different performers, background-swap augmentation, and validation splits that hold out whole performers so the shortcut can't score. Measure it with an identity-leakage test. And it isn't just a quality bug: it's the privacy failure mode in §8.

**4. "Where do labels for fine-grained moves come from?"**
> Mostly weak supervision at scale: titles, chapters and timestamped comments naming moves. The best positives come free: multi-camera recordings of the same event give the same move at the same instant from different viewpoints, and slow-motion replays give it at different speeds. Self-supervised pretraining needs no labels. The expensive part is the **expert-labelled evaluation set**, so I'd use active learning to spend expert time on the pairs the model is least sure about.

**5. "How do you know it worked?"**
> Offline: recall@100 and graded NDCG@10 on an expert-labelled set per vertical, temporal IoU for the deep links, and **context-diversity@10**, because "different context" is part of the requirement. Online: successful-search rate (watched past the matched moment, rewatched or saved) as the north star, re-query and "not this move" as failure signals, with guardrails on latency, policy violations and identity leakage. Launch one vertical at a time.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Could this be misused?"**
> Yes, as a person-finder: upload a clip of someone and find more videos of them. The same invariance that makes it useful (same move, different context) becomes harmful if the embedding encodes identity. So identity invariance is a **requirement with a metric**: a red-team test of how often results show the same person as the query, held below chance as a launch gate. No face features anywhere, query clips not retained, and rate limits against systematic probing.

**7. "The same trick filmed mirrored, at half speed, from above. Still a match?"**
> Each is an invariance to decide, not assume. **Speed:** yes, handled by temporal augmentation in training and dynamic time warping in the reranker. **Viewpoint:** yes, from multi-camera positives and optionally 3D pose lifting. **Mirroring:** ask the domain experts, because in some sports a left-handed version is a different skill. That's exactly why the expert-labelled set encodes the policy rather than an engineer's guess.

**8. "The index costs too much. Cut it 10×."**
> Tighten the funnel before the encoder: raise the human-activity threshold, index only videos with some minimum engagement, and cut proposals per video from ~30 to ~10 by ranking them on motion energy. Shrink each vector: 64-dimensional, product-quantised to 32 bytes. Reuse existing YouTube video embeddings for the coarse tier. And make the long tail **lazy**: compute moment embeddings on demand when a coarse match makes a shortlist, and cache them.

**9. "A brand-new trick goes viral and the model has never seen it."**
> The embedding is trained on motion, not on a fixed list of move names, so a new trick still lands near itself: its copies and imitations will be close to each other even before anyone names it. The text source catches up as titles start naming it. Weak labels refresh continuously, and the expert set gets the new move added. This is the advantage of retrieval over a classifier with fixed classes.

**10. "Every result is a re-upload of one viral clip."**
> Near-duplicate collapse: perceptual fingerprints cluster copies, and each cluster contributes one result, preferably the **original upload**, which also credits the creator. Then the appearance-based MMR pushes the rest of the page toward different contexts. And I'd track the near-duplicate rate in results as a metric, because a viral clip can flood the page overnight.

---

## ⭐ THE L6 LINE

> **"Similar means same action, different context. So the representation must be invariant to appearance and sensitive to motion, the unit is a moment rather than a video, and the same invariance that makes it useful must never extend to identity."**

## 🧭 HOOKS — *steer by what you mention*

- *"Generic embeddings return the same scene, not the same move"* → invites the representation / training branch (**good at L6**)
- *"Index moments, not videos"* → invites the capacity / indexing branch
- *"Two embeddings for two jobs: action for relevance, appearance for diversity"* → invites the ranking branch
- *"Same-dancer-different-move negatives"* → invites the data / shortcut-learning branch (**good at L6**)
- *"It must never become a person-finder"* → invites the ethics / privacy branch (**good at L6**)
- *"G1's geometric verification, in time"* → invites the reranker branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to what 'similar' means."* Same move, different context regenerates the whole design.
- **Running out of time?** → *"I have five minutes. I'd rather cover the person-finder risk and the readiness sweep than finish the indexing pipeline. Sound right?"*

**Name and defer:**
- "**Content ID** solves exact match; we use it only to collapse copies."
- "**Choreography matching to a song** is a different mode where audio is the main signal."
- "**3D pose lifting** helps viewpoint invariance; it's an upgrade, not the baseline."
- "**On-device pose extraction** would be even more private; worth a follow-up."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The form's pointers were written for R1; each row restates the pointer **for this question**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | Practical critical path: moment ANN + temporal rerank, with the text path as fallback | §0, §4 | *"Encode the moves, retrieve moments, align the sequence, diversify the context."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"P95 ≤ 2 s after upload; here's the per-stage budget; client-side trimming keeps upload small."* |
| **A1c** | Tiered, confidence-gated serving instead of lowering quality | §4 | *"Rerank 50 or 200 depending on confidence; mix in text results; or say no close matches."* |
| **A2a** | The full stack: gate, query understanding, candidates, reranker, filters, gate | §4 diagram | Walk the boxes |
| **A2b** | Model components: video encoder + pose encoder, fused; ANN; cross-attention reranker | §5.3 | *"A skeleton has no outfit and no background."* |
| **A2c** | Data sources | §5.1 | Including multi-camera positives |
| **A2d** | Staged pipeline | §5.4 | Pretrain → pairs → adapt → index → evaluate |
| **A3a** | Top-down, unprompted organisation | §0 | State the plan after the opening |
| **A3b** | Diversity objective with λ | §4, §6 | *"MMR where similarity uses the appearance embedding."* |
| **A4a** | Clarifying questions first, assumptions explicit | §1 | *"What does similar mean?"*, asked first |
| **A5a** | Alternatives compared | §4 | The four-option table |
| **A5b** | A full fallback pipeline on demand | §9 | Caption → text search → rerank → filter → gate |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | Right model per subproblem: contrastive loss, in-batch + hard negatives, pairwise reranker loss | §5.3 | *"The negatives are where the invariance comes from."* |
| **F1b** | Metrics and trade-offs | §2, §7 | Recall@100, graded NDCG, temporal IoU, **context diversity**, calibration |
| **F1c** | Bias sources and mitigations | §7 | Including **pose-model representation bias** |
| **F2a** | Retrieval underperforms → fallback | §4, §9 | Raise it yourself |
| **F2b** | Latency too tight → tiering | §4 | Raise it yourself |
| **F2c** | Fine-tune an existing model on positive/negative pairs | §5.3 | *"LoRA adapters on a frozen video foundation model, contrastive, then distil."* |
| **F3a** | Quantization with a quality trade-off | §6 | *"Gated on the expert set, because fine distinctions erode first."* |
| **F3b** | Scale via index, precomputation, light models | §3, §6 | *"Only the query is encoded online; PQ; tiered index."* |
| **F3c** | Extension with cost and maturity risk | §9 | *"Fine-grained action embeddings are far less proven than image ones, so launch vertical by vertical."* |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance: latency + cost tracking | §3, §8 | $/query and $/indexed hour |
| **I1b** | Reliability | §6, §8 | Text fallback; versioned index swaps |
| **I1c** | Usability | §2 | Trim UI; results start at the move |
| **I1d** | Scalability | §3, §8 | Tiered index + proposals + PQ |
| **I1e** | Privacy | §8 | Ephemeral query clips; no face recognition |
| **I1f** | Safety | §8 | Hash-scan queries; policy-filtered results |
| **I1h** | Security | §8 | **Sandboxed decoding**, scraping, adversarial clips |
| **I1i** | Ethics / bias | §7, §8 | **Not a person-finder**; recall parity across groups |
| **I1j** | Explainability | §8 | Matched moment + move label + skeleton overlay |
| **I2a** | Offline → online with shown-only label correction | §7 | Randomised slice, IPS, interleaving |
| **I2b** | Success definition | §2, §7 | Successful-search rate, re-query rate, guardrails |
