# G3 · Photos Natural-Language Search — **INTERVIEW SHEET**

> **Tier 3 · #25** — multimodal + on-device. **Read with [`G1`](G1_visual_search_intv.md)**: same imagery, *inverted* scale.
> Full study file: [`../G3_photos_nl_search.md`](../G3_photos_nl_search.md) · [diagrams](../G3_photos_nl_search_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the user sees · (c) the on-device stack · (d) datasets · (e) building the training set · (f) the model and its objective · (g) serving and the update problem · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Library already in the cloud? Oldest device? Any labelled data? Screenshots and languages? (§1) |
| **Explainability** is easy points | Each result shows **what matched**: filter chips for metadata, a label for the semantic match (§2, §8) |
| **Security** beyond the gateway | **Embedding inversion** · the on-device index in backups · probing the shipped encoder (§8) |
| **Troubleshoot before being asked** | Raise **the mixed-space migration** and **the battery budget** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **The scale is upside down** · **privacy IS the architecture** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · on-device as the SLO · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | ⚠ **You can't look at the data** — evaluating blind | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | 50k photos ≈ **19 MB** on device · 10¹³ encodes in cloud · battery | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | On-device embed + on-device index · metadata filter first | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Personal-photo-like data · dual encoder · distil + quantise | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING + THE UPDATE PROBLEM** | **(g)** | ⚠ **Re-embedding a billion libraries** · versioned spaces | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Proxy corpora · DP aggregates · rollout by device tier | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | OCR, video, federated · **labels-only fallback** for old devices | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Two things frame this, and both invert what you'd expect coming from web-scale visual search.**
>
> **First: the scale is upside down. Lens is ten billion images in one shared index. Photos is ten thousand images per user, times a billion users. The per-user corpus is tiny — fifty thousand photos at 384 dimensions in int8 is about nineteen megabytes, which fits on the phone comfortably. So the hard problem is *not* ANN scale, which is the entire difficulty in G1. The hard problems are running a good enough encoder on a phone, and updating that encoder across a billion devices.**
>
> **Second: privacy is the architecture, not a requirement layered on top. This corpus is photographs of the user's children, their medical documents, screenshots of their bank statements. So 'where is the embedding computed and where does the index live' isn't a latency trade-off — it's a product-positioning and legal decision that determines every other choice."**

**Then the claim to challenge, and the cost argument:**
> ⚠ **"'The photo never leaves the device, we only upload vectors' is NOT the privacy guarantee it sounds like. Embedding inversion is real — from a 512-dimensional vector you can reconstruct a recognisable approximation of the original image. A cloud vector index is a cloud image index with extra steps, and I wouldn't let that claim into a design review unchallenged."**
>
> ⭐ **"And the cost argument points the same way, which is unusual: cloud-embedding a billion libraries is 10¹³ image encodes — prohibitive at any price. On-device inference is paid for by the user's battery, not by our fleet. On-device is simultaneously the private option and the only affordable one — and when the ethics and the economics agree, take the win."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and how we measure without seeing data, the two numbers, the on-device architecture, the data and the model, serving and the update problem, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Is the photo library already in the cloud?** *(If it is, the privacy argument changes — though embedding inversion still applies to the *index*.)*
2. **What's the oldest device we must support?** *(This sets the encoder size, which sets quality — the binding constraint.)*
3. **Can we collect *any* labelled data?** *(Determines whether evaluation is possible at all — §2.)*
4. **Which languages, and do screenshots and documents count?** *(Multilingual text encoders and OCR are different pipelines.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Users | ~10⁹ libraries; typical 10⁴ photos, heavy users 5×10⁴+ |
| Devices | A wide spread: recent phones with NPUs, and a long tail of old ones |
| Data access | **None** — we never see users' photos, queries' results or embeddings |
| Queries | Short natural language: "beach with the dog", "receipt from the plumber" |
| Network | Must work **offline** |

**FR:** natural-language search over a personal library · **on-device embedding + index** · incremental indexing of new photos · works offline · face/pet grouping (separate, consent-gated).

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Query latency | **< 100 ms on-device** — trivial at 50k vectors |
| **Index size** | **≤ ~20 MB** per 50k photos (int8, 384-d) |
| Battery / thermal | Backfill must not cook the phone — **overnight, on charge** |
| **Privacy** | **The raw photo and its embedding never leave the device** by default |
| Model update | ⚠ **Re-embedding must be feasible** (§6) |

**Scope / NON-GOALS:**
- ❌ **Cloud embedding as the default path.** Position: inversion + cost.
- ❌ **"We only upload vectors" as a privacy claim.** Position, with the reason.
- ❌ Shared/public photo search (G1) · face *recognition* against a global identity set (consent-gated grouping only) · editing.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **⭐ the L6 move: evaluating blind** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
[ beach with the dog in Rome last summer ]         works in airplane mode
  Filters applied: 📍 Rome · 📅 Jun–Aug 2025        ← METADATA, editable
  Matched: beach · dog                              ← the SEMANTIC part
  ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │    │ │    │ │    │ │    │   42 photos
  └────┘ └────┘ └────┘ └────┘
  Settings › Search: "Search runs on this device. Your photos and their
  search index never leave it."                     ← the claim, stated precisely
```

```
⚠ THE EVALUATION PROBLEM IS THE HARD PART, AND IT'S STRUCTURAL:
   YOU CANNOT LOOK AT THE DATA. Not "shouldn't" — the whole design
   premise is that we never see it.

   → no click logs with content
   → no labelled queries from real libraries
   → no error analysis on failures
   → a regression is INVISIBLE unless it's designed to be visible

FOUR SUBSTITUTES, none sufficient alone:
 1. PROXY CORPORA — public/licensed photo sets with authored queries.
    ⚠ distribution gap: stock photos ≠ blurry indoor phone snaps
 2. ⭐ FEDERATED / AGGREGATE METRICS — did the user tap a result?
    scroll past everything? reformulate? Sent as DP-noised aggregates,
    no content. ⚠ REFORMULATION RATE is the best privacy-safe signal
    available: it means we failed, and it needs no content at all.
 3. OPT-IN DONOR PANELS — users who explicitly consent to share a
    library for evaluation. Small, precious, and NOT representative.
 4. SYNTHETIC libraries built to mimic the real distribution.

→ SO: expect a WIDER quality gap between offline and production than
  in any other system in this pack, and design for GRACEFUL DEGRADATION
  rather than assuming you'll detect regressions quickly.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Searches that end in a tap without reformulation** (DP aggregate) | Success, measured without content |
| Offline | Recall@k on proxy corpora and donor panels, **per device tier** | The gate before rollout |
| Signals | Reformulation rate · zero-result rate · tap position, by encoder version | Privacy-safe production quality |
| **Guardrails** | Backfill battery and thermal budget · index size · on-device p95 · **zero content egress** | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **⭐ both numbers** `A1b` `I1a` `I1d`

```
PER-USER INDEX — why on-device is even possible:
  50,000 photos × 384 dims × 1 B (int8) = 19.2 MB
  + a flat or small-graph index           ≈ ~25 MB total
  → FITS COMFORTABLY. And at 50k vectors, ANN is unnecessary —
    a BRUTE-FORCE scan is ~milliseconds. ⚠ No HNSW needed at all.
    (G1 needs 2 TB and a sharded fleet. Inverted.)

CLOUD ALTERNATIVE — why it's not merely expensive, it's infeasible:
  10⁹ users × 10⁴ photos = 10¹³ IMAGE ENCODES for the initial backfill
  → at any plausible per-image cost this is prohibitive, and it
    recurs on every model update (§6).
  On-device: the compute is the USER'S BATTERY, amortised, and free
  to us.

ON-DEVICE BACKFILL: ~10k photos × ~10 ms = ~100 s of NPU time
  → overnight, on charge, on wifi. ⚠ Thermal and battery budget is a
    real product constraint, not a footnote.

CLOUD SIDE: training, distillation and MODEL DELIVERY only —
  a ~tens-of-MB model pushed through the app-update / CDN path.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Where content is processed | Cloud embed + index vs on-device embed + cloud index vs **on-device both** | Inversion makes uploaded vectors uploaded images; 10¹³ encodes is infeasible |
| Search structure | HNSW vs **brute-force scan** | 50k vectors scan in milliseconds |
| Query handling | Everything semantic vs **metadata filter first, then semantics** | "Rome last summer" is a filter |
| Device spread | One encoder vs **tiered encoders by device class** | Capping everyone at the oldest device is a real cost (FU 6) |
| Model updates | Hard swap vs **versioned dual-space migration** | A mixed library makes cross-space similarity meaningless |

### The system — *draw this in 90 seconds* `A2a`

```
══ ON DEVICE (everything that touches content) ═════════════════════════════
┌─ 1. IMAGE ENCODER — distilled, int8 ───────────────────────────
│ TASK: embed each NEW photo (~10 ms on NPU), backfill on charge
│ OUTPUT: a 384-d int8 vector per photo, tagged with ENCODER VERSION
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. LOCAL INDEX (~19 MB) ──────────────────────────────────────
│ vectors + metadata (date, place, album) · ⚠ brute-force scan
│ OUTPUT: searchable library, offline
└──────▲─────────────────────────────────────────────────────────
       │
┌─ 3. QUERY UNDERSTANDING ───────────────────────────────────────
│ TASK: split "beach with the dog in Rome last summer" into a
│       METADATA FILTER (Rome, Jun–Aug) + a SEMANTIC part (beach, dog)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. TEXT ENCODER (same version as the photos it searches) ─────
│ OUTPUT: a query vector → top-k by cosine within the filtered set
└────────────────────────────────────────────────────────────────
 RESULTS — works offline
       │ ⚠ ONLY THIS CROSSES THE BOUNDARY:
       │   DP-noised aggregate metrics. No content. No vectors.
       ▼
══ CLOUD ════════════════════════════════════════════════════════════════════
 model training · distillation · proxy-corpus evaluation · MODEL DELIVERY
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `I1b`

- **The mixed-space migration:** *"A model update leaves a library half in the old space and half in the new. Cross-space similarity is meaningless, so I version the index and query both spaces until the backfill finishes."*
- **The battery budget:** *"The backfill runs only on charge, on wifi, idle and below a thermal cap. A feature that warms the phone gets blamed for the phone."*
- **"We only upload vectors":** *"Embedding inversion means that claim doesn't hold. I'd state the privacy claim precisely in the product."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **On-device embedding *and* index** | "Embedding inversion means a cloud vector index is a cloud image index with extra steps. And 10¹³ encodes is infeasible anyway — ethics and economics agree" | A smaller encoder ⇒ lower ceiling on quality |
| **2** | **Brute-force, not ANN** | "50k vectors is a millisecond scan. Building an HNSW here is solving G1's problem in a place that doesn't have it" | None, really — until libraries get much larger |
| **3** | **Metadata filter before semantics** | "'Rome last summer' is a structured filter; running it as a semantic query is strictly worse and much slower" | Query understanding has to be good enough to split intent |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Web-scale image–text pairs** (licensed / alt-text) | Contrastive pre-training for the teacher | Stock-photo bias; noisy captions |
| **Proxy corpora** of personal-style photos with authored queries | The offline gate | ≠ blurry indoor phone snaps |
| **Opt-in donor panels** | The closest thing to real libraries | Small; consented; not representative |
| **Synthetic libraries** | Distribution-matched test sets | Synthetic ≠ real |
| **DP aggregate metrics** | Production quality signals, per version and tier | Noisy; no content by design |
| **Query templates** in many languages | Query-understanding training | Real phrasing varies |

### 5.2 Building the training set (e) `F1c`

- **Shift toward personal photos:** up-weight indoor, low-light, pets, children, documents, receipts and screenshots — the distribution people search, not the one the web publishes.
- **Hard negatives:** same scene, different subject ("beach without a dog") so the model learns the detail that the query names.
- **Balance people-related data** across skin tones, ages and cultures — quality on "my grandmother's birthday" shouldn't depend on who the grandmother is.
- **Multilingual captions**, since the text encoder serves every locale.
- **Screen harmful content** out of the corpus with hash matching and classifiers before training.

### 5.3 The model and its objective (f) `A2b` `F1a`

**CLIP-style dual encoder** — image encoder and text encoder into a shared space, trained with a **contrastive (InfoNCE) loss**. The reason it fits: **the text encoder runs at query time only** (once per query, cheap), and **the image encoder runs once per photo, offline**, so the expensive side is amortised across the library's lifetime.

| Model | Objective |
|---|---|
| **Teacher dual encoder** (cloud) | Contrastive InfoNCE on image–text pairs |
| **Student image + text encoders** (device) | **Distillation**: match the teacher's embeddings + keep the contrastive loss; trained per device tier |
| **Query parser** | Slot filling: place, date range, person, semantic remainder |

**⚠ Both encoders must be updated together.** They share an embedding space; a new image encoder with an old text encoder produces nonsense. That's what makes §6 hard.

### 5.4 Fine-tune an existing model (f) `F2c`

Start from a pretrained image–text model as the teacher; **LoRA-tune** it on the personal-photo-shifted set (§5.2). Then distil into small students per device tier, using **quantisation-aware training** so int8 costs little quality. **The binding constraint is the oldest supported device**, not the newest — a product decision that directly caps search quality.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · TEACHER    pretrained dual encoder → LoRA on personal-photo-shifted data
STAGE 2 · STUDENTS   distil per device tier → quantisation-aware training → int8
STAGE 3 · EVAL       proxy corpora + donor panels, PER TIER · battery/thermal on devices
STAGE 4 · SHIP       model in the app update, behind a flag
STAGE 5 · MIGRATE    versioned re-encode on device, recent photos first (§6)
STAGE 6 · WATCH      DP aggregates per version → continue, halt or roll back
```

---

## 6 · SERVING + THE UPDATE PROBLEM *(min 32–38)* — probe **(g)** · **the hardest part** `F3a` `F3b` `A3b` `I1b`

```
⚠ IMPROVING THE MODEL MEANS RE-EMBEDDING EVERY PHOTO ON EVERY DEVICE.

  · image and text encoders SHARE a space → they must move together
  · so a model update = a full library re-encode = hours of NPU per
    device, × a billion devices
  · ⚠ AND DURING THE MIGRATION, A LIBRARY IS MIXED: some photos in
    the old space, some in the new. Cross-space similarity is MEANINGLESS.

WHAT TO DO:
 · ⭐ VERSION THE INDEX and keep BOTH spaces during migration —
   query both with the matching text encoder, merge results by rank
   (RRF-style), and retire the old space when the backfill completes
 · re-encode INCREMENTALLY on charge + wifi, prioritised by recency
   (recent photos are searched far more than old ones)
 · ⚠ THEREFORE: UPDATE RARELY AND MAKE EACH ONE COUNT — model
   cadence is constrained by MIGRATION COST, not by research velocity
 · which is a strong argument for investing in the FIRST encoder:
   you don't get to iterate your way out of a weak one

⚠ AND THE FLEET IS PERMANENTLY HETEROGENEOUS: old devices, deferred
  updates, low storage. You will ALWAYS be serving several versions.
```

- **The objective** `A3b`: maximise search quality **subject to** a per-device battery and thermal budget and zero content egress; migration order maximises *searched* photos re-encoded per unit of battery — so recency first.
- **Efficiency** `F3a`: int8 halves the index and speeds the scan; brute force avoids index builds. The trade is a quality ceiling set by the smallest encoder, which tiered encoders soften.
- **Scale** `F3b`: **the fleet is the compute** — a billion NPUs doing the encoding; the cloud ships models, not inference.
- **Reliability** `I1b`: the versioned index makes every update **reversible without a re-encode**; an interrupted migration leaves both spaces queryable; the backfill resumes where it stopped.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Offline** | Proxy corpora with authored queries — **acknowledge the distribution gap** |
| **⭐ Privacy-safe production signal** | **Reformulation rate** (we failed), tap-through position, zero-result rate — DP-noised aggregates, no content |
| **Opt-in panels** | Small, consented, precious — and **not representative** |
| **Per-version** | Metrics sliced by **encoder version and device tier** — the fleet is heterogeneous by construction |

```
ROLLOUT OF A NEW ENCODER:  proxy + panel eval per tier → battery tests on real
  devices → 1% of the fleet, by tier → watch DP reformulation and zero-result
  rates per version → widen, or HALT and fall back to the old space
  (still on device, so rollback costs no re-encode)
```

**Success** = reformulation and zero-result rates down for the new version on every device tier, backfill within the battery budget, and no content leaving the device. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Device tier** | Users on older, cheaper phones get a smaller encoder and worse search | Tiered encoders; per-tier metrics with a parity target |
| **Skin tone and age** | People-related queries work worse for under-represented groups | Balanced training data; panel evaluation sliced by group |
| **Language** | Text encoder weaker in some locales | Multilingual captions; per-locale query sets |
| **Proxy-corpus bias** | Stock and Western-centric photos define "good" | Donor panels across regions; synthetic libraries |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | < 100 ms on-device; backfill within a battery and thermal budget | On-device p95 · mAh per 1,000 photos |
| **Reliability** `I1b` | Versioned index with rollback; resumable backfill; works offline | Migration completion · rollback events |
| **Usability** `I1c` | Filter chips you can edit; "what matched" labels; instant results offline | Reformulation rate · tap position |
| **Scalability** `I1d` | The fleet does the compute; the cloud only ships models | Model download success · tier coverage |
| **Privacy** `I1e` | **On-device embedding and index**; only DP aggregates leave; the privacy claim worded **precisely**; face grouping separate and consent-gated | Content egress = 0 · DP budget |
| **Safety** `I1f` | Sensitive photos (medical, IDs) never surfaced in shared or "memories" contexts by search; no search over children's faces outside consented grouping | Sensitive-surface incidents |
| **Security** `I1h` | ⚠ **Embedding inversion** — vectors treated as sensitive as images; the index inside device encryption and **excluded from unencrypted backups**; the shipped encoder assumed extractable | Index-at-rest encryption · backup audits |
| **Ethics / bias** `I1i` | Quality parity across device tiers, skin tones and languages | Per-tier and per-group recall |
| **Explainability** `I1j` | **What matched** shown per result (filter vs semantic); a plain statement of where search runs | % results with a matched label |

> ⭐ *"Privacy is the architecture here, so production readiness starts with a claim we can state precisely and keep: content and its embedding stay on the device."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **OCR over screenshots and documents:** a large share of real queries ("the plumber's receipt"), needing an on-device text pipeline.
- **Video:** multiplies the encode cost; needs frame sampling and a bigger battery budget.
- **Federated learning:** plausible for the encoder, but **federated evaluation first** (FU 9).

**When a device can't run the encoder — the labels-only pipeline** `A5b` `F2a`:

```
a device below the smallest tier, low storage, or a failed model download
  → a tiny on-device CLASSIFIER assigns a fixed set of labels
    (beach, dog, food, document, …) + on-device OCR keywords
  → search = METADATA FILTER + LABEL MATCH + keyword match
  → no embeddings, no backfill beyond the classifier, still fully on device
  → offer cloud search ONLY as an explicit opt-in with the trade stated
  "Worse at nuance, honest about privacy, and never silently in the cloud."
```

**Top 3 failure modes:**
1. **Mixed-space library during migration** — cross-space similarity is meaningless; results silently degrade.
2. **Quality regression nobody can see** — no content access, so a bad encoder ships and the only signal is a slow rise in reformulation rate.
3. **Thermal/battery complaints** — the backfill cooks the phone and the feature gets blamed for the device.

**Q1 ships:** the **on-device encoder + local index + metadata pre-filter**, and the **privacy-safe metrics pipeline**.
> *"The metrics pipeline in Q1, because without it a regression is genuinely invisible — and on this product you don't get content access later to go back and diagnose it."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why on-device? Cloud would let you use a much better model."** `A5a` `I1e`
> Two independent arguments that happen to agree, which is worth saying out loud. **Privacy**: this corpus is photographs of the user's children, medical documents, screenshots of bank statements — and the claim "the photo never leaves the device, we only upload vectors" is **not the guarantee it sounds like**, because **embedding inversion is real**: from a 512-dimensional vector you can reconstruct a recognisable approximation of the original image. So a cloud vector index is a cloud image index with extra steps, and I wouldn't let that pass unchallenged in a design review. **Cost**: cloud-embedding a billion libraries is **10¹³ image encodes** for the initial backfill alone, which is prohibitive at any plausible price and *recurs on every model update*. On-device inference is paid for by the user's battery, amortised, and free to us. When the ethics and the economics point the same direction, take the win — and the real cost is a smaller encoder, which caps quality.

**2. "How big is the index and what ANN algorithm do you use?"** `A1b` `A5a`
> About **19 megabytes** — fifty thousand photos at 384 dimensions in int8 — and **no ANN algorithm at all**. At fifty thousand vectors, a brute-force cosine scan is a few milliseconds on a modern phone, so building an HNSW graph would be solving G1's problem in a place that doesn't have it, at the cost of memory, build time, and recall loss. That's the inversion worth naming: **Lens is ten billion images in one shared index where ANN scale is the entire difficulty; Photos is ten thousand images per user where the corpus is trivial and the hard problems are elsewhere** — getting a good encoder onto a phone, and updating it across a billion devices. Recognising which problem you actually have is most of the value here, and a candidate who reaches for HNSW has pattern-matched to the wrong question.

**3. "The user searches 'photos from Rome last summer'. Walk me through it."** `A2a` `I1c`
> Mostly **not as a semantic query**, which is the point. "Rome" and "last summer" are **structured filters** — location metadata and a date range — and running them through an embedding is strictly worse: slower, less precise, and it can't express "summer" as an interval. So query understanding splits the intent first: apply the metadata filter to narrow the candidate set, then run semantic similarity over whatever remains, which is usually a small fraction of the library. That ordering matters for quality as much as speed — an embedding might surface Roman-looking architecture from Athens, and the user asked for Rome. The general principle is the same as shopping search: **a large share of apparently-natural-language queries are ninety percent structured filtering**, and treating them semantically is a common and expensive mistake.

**4. "You improved the model. How do you roll it out?"** `I2a` `I1b`
> This is the hardest part of the design, because **improving the model means re-embedding every photo on every device**. The image and text encoders share an embedding space, so they must move together, and during migration a library is **mixed** — some photos in the old space, some in the new — and cross-space similarity is **meaningless**, so naive rollout silently corrupts results. So: **version the index and keep both spaces during migration**, querying each with its matching text encoder and merging by rank; **re-encode incrementally** on charge and wifi, prioritised by recency since recent photos are searched far more; and retire the old space when backfill completes. The strategic consequence is the important half: **model cadence is constrained by migration cost, not by research velocity** — you update rarely and make each one count, which is a strong argument for investing heavily in the *first* encoder, because you don't get to iterate your way out of a weak one.

**5. "How do you know if search quality is good?"** `F1b` `I2b`
> With difficulty, and I'd name that as a first-class problem rather than a caveat — **you cannot look at the data**, and that's the design premise, not a policy I could negotiate. No click logs with content, no labelled queries from real libraries, no error analysis on failures. Four substitutes, none sufficient alone: **proxy corpora** with authored queries, acknowledging that stock photos are a poor match for blurry indoor phone snaps; **federated, DP-noised aggregate metrics** — and the best privacy-safe signal available is **reformulation rate**, because it means we failed and it requires no content at all; **opt-in donor panels** of users who explicitly consent to share a library for evaluation, which are small, precious and unrepresentative; and **synthetic libraries**. The practical consequence: expect a **wider offline-to-production gap than anywhere else in this pack**, and design for graceful degradation rather than assuming you'll detect a regression quickly.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Half your fleet is on old phones that can't run the good encoder. What do you do?"** `F2a` `I1i`
> Accept a **permanently heterogeneous fleet** and design for it, rather than pretending a single model version exists. Concretely: **tiered encoders** — a small one for old devices, a larger one for capable hardware — with the understanding that they produce **different embedding spaces**, so a user's index is tied to their device tier and quality genuinely differs across the fleet. That's uncomfortable and it's honest; the alternative is capping everyone at the oldest device's ceiling, which is a real product cost paid by the majority to serve a shrinking minority. Two consequences I'd plan for. **Metrics must be sliced by device tier and encoder version**, or the aggregate hides a large population having a materially worse experience. And **device upgrade becomes a migration event** — a user moving to a new phone should get the better encoder, which means a full re-encode on a device that's already busy restoring a library, so it needs to be scheduled rather than immediate. The cloud fallback for very old devices is worth considering as an **explicit opt-in** with the privacy trade stated plainly, rather than as a silent default.

**7. "A user searches 'my daughter at the beach'. What's happening, and what are you worried about?"** `I1e` `I1i`
> Two systems are involved and only one of them is the semantic search. "Beach" is the **embedding query**; "my daughter" is a **face-grouping** lookup, which is a different feature with a different consent model and much higher stakes. I'd be careful to keep them architecturally separate: face grouping is **clustering within a private library**, not recognition against any global identity set, it's **consent-gated and opt-in**, and the labels ("this cluster is Maya") are user-supplied rather than inferred from anywhere else. What I'd worry about: **the cluster-merge failure**, where two children look similar and the grouping conflates them — which is a small quality bug and an emotionally significant one; **children's data specifically**, which carries additional legal obligations in most jurisdictions; and **the export surface**, since face vectors leaving the device would be far more sensitive than scene embeddings. The answer to "would you build face recognition" is **no** — grouping within a library, yes, with consent; identification against an external set, out of scope by design, for the same reason as in G1.

**8. "Your reformulation rate went up 15%. What happened?"** `I2a` `F1c`
> Something regressed, and with no content access the diagnosis is a process of elimination across the dimensions I *can* slice. First: **is it version-correlated?** If it tracks a new encoder rollout, that's the answer and I'd halt the migration — which is exactly why the index is versioned and both spaces are retained, since I can roll back without a full re-encode. Second: **device tier**, since a regression concentrated on one tier points at the small encoder or a quantization issue rather than the model. Third: **query-length or language distribution**, because a change in *who's querying* can move the metric without anything having broken — a launch in a new locale, for instance. Fourth: **zero-result rate and tap position** alongside reformulation, since reformulation after zero results is a coverage problem and reformulation after tapping nothing is a ranking problem. What I'd emphasise is that this diagnosis is only possible because those slices were **designed into the metrics pipeline in Q1** — retrofitting per-version, per-tier aggregation after a regression means waiting a full release cycle to learn anything.

**9. "Could you use federated learning to improve the encoder on real data?"** `F3c` `I1e`
> In principle yes, and I'd be cautious about it in practice for reasons beyond the usual. The appeal is obvious — it addresses the central problem, which is that we're training on stock photos and serving blurry indoor snaps. The obstacles: **contrastive training needs negatives**, and the natural negatives are other users' photos, which is exactly what federated learning is supposed to avoid — so you're limited to within-library negatives, which are a much weaker and differently-distributed signal. **Gradient leakage is real**: gradients from a small local batch can leak information about the images that produced them, so this needs secure aggregation and DP noise, which cost quality. And the **device cost** is significant — on-device training is far more expensive than inference, on hardware already budgeted for a backfill. So my position: **federated evaluation before federated training** — the aggregate metrics in §7 are lower-risk, cheaper, and address the more urgent problem, which is that we currently can't tell whether anything works. Federated training is a reasonable Q4 investment once there's a measurement loop to prove it helped.

**10. "The product team wants cloud search so results appear on the web too. Respond."** `I1e` `A3b`
> That's a legitimate product need and I'd engage with it rather than refuse on principle — but I'd insist the privacy trade is stated **accurately**, because the usual framing is wrong. The claim that will be made is "we only upload embeddings, not photos," and that is **not a meaningful guarantee**: embedding inversion reconstructs a recognisable approximation, so an uploaded index is an uploaded library with extra steps, and shipping that as a privacy assurance is the thing I'd block. What I'd offer instead, in order: **on-device search with cloud-rendered results** for libraries already synced to cloud storage, where the content is already there and the index adds no new exposure; **explicit, granular opt-in** with plain language about what's uploaded, for users who want cross-device search and understand the trade; and **encrypted-index approaches** where the server holds vectors it cannot read — which is genuinely harder and materially slower, and worth scoping rather than dismissing. The framing for the decision-maker: **this is a positioning choice, not a technical one** — if the product's differentiator is privacy, the cloud index costs the differentiator; if it isn't, the trade may well be worth making, but it should be made deliberately and described honestly.

---

## ⭐ THE L6 LINE

> **"The scale is upside down — the per-user corpus is 19 MB and ANN is unnecessary; the hard problems are a phone-sized encoder and updating it across a billion devices. Privacy is the architecture, not a layer: embedding inversion means a cloud vector index is a cloud image index with extra steps, and 10¹³ encodes is infeasible anyway."**

## 🧭 HOOKS — *steer by what you mention*

- *"The scale is upside down"* → invites the G1 comparison / systems branch
- *"Embedding inversion is real"* → invites the privacy/security branch (**good at L6**)
- *"You cannot look at the data"* → invites the evaluation-under-constraint branch (**good at L6**)
- *"Model cadence is constrained by migration cost"* → invites the deployment branch
- *"Ethics and economics agree — take the win"* → invites the product-strategy branch
- *"'Rome last summer' is a filter, not a semantic query"* → invites the query-understanding branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the two numbers"* — 19 MB vs 10¹³ encodes regenerates the whole argument.
- **Running out of time?** → *"I have five minutes — I'd rather cover the update problem and evaluating blind than finish the encoder detail. Sound right?"*

**Name and defer:**
- "**Face and pet grouping** is a separate, consent-gated feature — clustering within a library, never global recognition."
- "**OCR over screenshots** is a large share of real queries and needs a different pipeline (G11)."
- "**Video** multiplies the encode cost and needs frame sampling."
- "**Memories/curation** — surfacing photos unprompted — is a different product with the Discover-feed sensitivity problem."
- "**Cross-device sync** of the index is a distributed-systems problem with a privacy constraint attached."
- "**Federated learning** is plausible for the encoder but I'd do federated *evaluation* first."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: on-device, brute force | §0, §3 | *"The scale is upside down — 19 MB per user, ANN unnecessary."* |
| **A1b** | Latency and capacity budgets | §1, §3, FU 2 | *"50k × 384 × 1 B = 19 MB; 10¹³ encodes in the cloud."* |
| **A1c** | Degrade without lowering quality | §4, §6 | *"An interrupted migration leaves both spaces queryable."* |
| **A2a** | The full stack | §4 diagram, FU 3 | Encoder → local index → query split → text encoder → results; only DP aggregates leave |
| **A2b** | Model components | §5.3 | Teacher dual encoder, tiered students, query parser |
| **A2c** | Data sources | §5.1 | Image–text pairs, proxy corpora, donor panels, synthetic libraries, DP aggregates |
| **A2d** | Staged pipeline | §5.5 | Teacher → students → eval per tier → ship → migrate → watch |
| **A3a** | Top-down, unprompted | §0 | *"The scale is upside down; privacy is the architecture"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 10 | *"Maximise quality subject to a battery budget and zero content egress."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Cloud library? Oldest device? Labels? Languages and screenshots? |
| **A5a** | Alternatives compared | §4, FU 1, 2 | Cloud vs on-device; HNSW vs brute force; one vs tiered encoders |
| **A5b** | A full alternative pipeline on demand | §9 | The labels-only pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss | §5.3 | *"Contrastive InfoNCE for the teacher; distillation + contrastive for the students."* |
| **F1b** | Metrics and their trade-offs | §2, FU 5 | *"Reformulation rate means we failed, and needs no content."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 8 | Device tier, skin tone, language, proxy-corpus bias |
| **F2a** | Something fails → a fallback | §9, FU 6 | Labels-only search; tiered encoders |
| **F2b** | Device too weak → design for it | §4 | Tiered encoders and the battery budget, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA teacher → QAT-distilled students per tier |
| **F3a** | Efficiency with a quality trade | §6 | int8 + brute force; the smallest encoder caps quality |
| **F3b** | Scale levers | §6 | The fleet is the compute |
| **F3c** | Extension, with cost and maturity risk | §9, FU 9 | OCR, video, federated learning |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | On-device p95; battery per 1,000 photos |
| **I1b** | Reliability | §6, §8, FU 4 | Versioned index, rollback without re-encode |
| **I1c** | Usability | §2, §8, FU 3 | Editable filter chips; offline |
| **I1d** | Scalability | §3, §8 | The cloud ships models, not inference |
| **I1e** | Privacy | §8, FU 1, 7, 10 | On-device; DP aggregates only; the claim stated precisely |
| **I1f** | Safety | §8 | Sensitive photos never surfaced in shared contexts |
| **I1h** | Security | §8 | **Embedding inversion**; index excluded from unencrypted backups |
| **I1i** | Ethics / bias | §7, §8, FU 6, 7 | Parity across device tiers, skin tones, languages |
| **I1j** | Explainability | §2, §8 | What matched, per result |
| **I2a** | Offline → online | §7, FU 4, 8 | Proxy → 1% by tier → DP watch → widen or roll back |
| **I2b** | The success definition | §7, FU 5 | *"Reformulation down on every tier, battery in budget, no egress."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) on-device stack → §4 · (d) data → §5.1 · (e) training set → §5.2 · (f) model → §5.3–5.4 · (g) update problem → §6 · (h) success → §2, §7 · (i) extension → §9.
