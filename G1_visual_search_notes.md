# G1 · Visual Search — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G1_visual_search_intv.md`](G1_visual_search_intv.md). These are the specific lines that didn't land first time. Skim this on a revision pass; go back to the sheet for the full answer.

**Jump:** [1 recall vs precision@1](#1--why-recall-for-shopping-but-precision1-as-the-headline) · [2 "Position, with the reason"](#2--what-position-with-the-reason-means) · [3 the 4-byte arithmetic](#3--how-is-one-embedding--4-bytes) · [4 int8 shards vs PQ nodes](#4--int8--shard-across-nodes-pq--fewer-nodes-why) · [5 the 500 × 100 µs figure](#5--where-does-500-serial-ssd-reads-at-100-µs-come-from) · [6 hash sharding](#6--what-is-hash-sharding-and-why-not-semantic) · [7 RANSAC](#7--why-ransac--why-not-just-metadata) · [8 §5 is the encoder](#8--5-is-training-the-embedding-model-right) · [9 the small on-device embedding](#9--what-is-the-small-embedding-for-the-common-path) · [10 intent labels](#10--the-intent-classifier-needs-labelled-data) · [11 why context beats the photo](#11--rules-first-intent-correlates-more-with-where-the-user-is-than-what-they-photographed--explain)

---

## 1 · Why **recall** for Shopping, but **precision@1** as the headline?

They measure **different stages**. The sheet says it at §2: *"same-product precision@1 ← the headline · recall@k for the retrieval stage."*

| Stage | Metric | Why |
|---|---|---|
| Retrieval (ANN → ~100) | **recall@k** | It's the **ceiling** — a miss is unrecoverable |
| End-to-end product | **precision@1** | One right answer; the UX shows one |

**Why recall is the *business* metric if Shopping monetises.** Recall = of all products someone photographs, what fraction can we find in the catalogue **at all**. That's the ceiling on revenue: at 60% recall, **40% of camera queries can never monetise** — not because ranking was poor, but because the matching listing was never retrieved. Recall defines the *addressable* share of queries.

**And "not a modelling preference"** means the recall target isn't picked off a P-R curve to balance F1 — it's **set by the business**, because it determines how much query volume is monetisable. *(Same structural move as G8's "recall is not a metric, it's a legal requirement": an external constraint has already decided the operating point, so the modelling question becomes how to hit it, not where to put it.)*

**Why @1 specifically** — two reasons, second is sharper:
1. **The UX returns an identification, not a list.** Point the camera, the product says *"this is the sofa."* No scanning ten results.
2. ⭐ **For same-product there is exactly one correct answer** — so precision@10 is **bounded above by 0.1 by construction** and is uninterpretable. The metric family for a single-ground-truth problem is precision@1 (= top-1 accuracy), or MRR for partial credit.

> **Contrast:** G7 shopping search and G13 recommendation have **many** legitimately good items and graded relevance → **nDCG@10** is the right shape there.

---

## 2 · What does "**Position, with the reason**" mean?

A convention used across all 30 sheets. A non-goals list holds two different kinds of item:

- **A scope cut** — *"face recognition is out, video is out."* Say it fast, move on.
- **A position** — a stance you'd actually **defend**, where the obvious design is the one you're rejecting.

The marker flags the second kind and says: **don't just state it, justify it in the same breath.** *"I won't do X"* reads as a simplification; *"I won't do X **because** Y"* reads as judgement.

```
❌ "One embedding space for all three products is a non-goal."
   → sounds like you're ducking work

✅ "I'm deliberately not building one embedding space for all three.
   The notion of distance is genuinely different — same-product must be
   invariant to lighting and angle but SENSITIVE to small design
   differences; semantic similarity needs the exact opposite. One space
   is mediocre at all three, and that's not a capacity problem a bigger
   model fixes."

✅ "I won't evaluate on click-derived labels alone — co-clicks are
   position- and popularity-biased, so the metric is built from the same
   bias the model learned. That's grading the homework with your own
   answer key."
```

**Why those two are flagged:** they're the places an interviewer is most likely to say *"wait, why not just…"* — having the reason ready converts an apparent gap into a demonstration of judgement.

---

## 3 · How is one embedding = 4 bytes?

It isn't. **The 4 B is per *dimension*** — fp32 means each of the 512 numbers is a 32-bit float.

The chain reads `images × dims-per-image × bytes-per-dim`:

```
one embedding  = 512 dims × 4 B  = 2,048 B ≈ 2 KB
10⁹ embeddings = 10⁹ × 2 KB              ≈ 2 TB
   (10⁹ × 512 × 4 = 2.05 × 10¹² bytes)  ✓
```

| Precision | Bytes/dim | Per embedding | 10⁹ embeddings |
|---|---|---|---|
| fp32 | 4 | 2 KB | **2 TB** |
| int8 | 1 | 512 B | **512 GB** (4×) |
| PQ (~64 subquantizers) | — | ~64 B **total** | **~65 GB** (~32×) |

⚠ **PQ breaks the per-dimension pattern** — it chops the vector into *m* subvectors and stores a **codebook index** per subvector, so size is set by *m*, not by the dimension count. That's why it compresses past what int8 can, and why it costs **real recall** rather than a fraction of a point.

> **⭐ Memorize:** a **512-d fp32 embedding is 2 KB; at int8 it's 512 B.** Same arithmetic in Q6 (200M × 768 × 4 B = 614 GB) and G3 (50k × 384 × 1 B = 19 MB).

---

## 4 · "int8 → shard across nodes", "PQ → fewer nodes" — why?

It's one question: **how many machines does the index need?** Divide by RAM per box — and remember the index is **vectors + the HNSW graph (~1.5×)**.

Assume ~256 GB usable RAM per serving box:

| Precision | Vectors | × graph | Nodes | × replicas (2–3) |
|---|---|---|---|---|
| fp32 | 2 TB | ~3 TB | ~12 | **~30 machines** |
| int8 | 512 GB | ~770 GB | **~3–4** | ~10 machines |
| PQ | ~65 GB | ~100 GB | **1** | 2–3 machines |

- **"fp32 doesn't fit"** — not impossible, **unreasonable**. 12× the hardware for an index that fits on 3, every query fans out to 12 shards, and p99 is set by the slowest of 12. Nobody ships it.
- **"int8 → shard, resident"** — 512 GB exceeds one box so it *must* shard, but only a handful of ways. **"Resident"** is load-bearing (see §5).
- **"PQ → fewer nodes"** — ~65 GB fits on **one** box, so sharding becomes *optional*: you replicate for availability and QPS, not because the data doesn't fit. No scatter-gather, no merge, no cross-shard tail latency.

> ⭐ **The decision rule: int8 is nearly free — take it always. PQ is a *memory-forced* choice**, not a default. At 10⁹ images int8's ~10 machines is fine, so PQ is optional; at 10¹⁰ it wouldn't be. And PQ's recall loss is recoverable by **retrieving wider + re-ranking** (geometric verification re-scores exactly) — *if* k stays affordable.

---

## 5 · Where does "~500 serial SSD reads at ~100 µs" come from?

**~500** — HNSW is a greedy graph walk. With typical **M ≈ 16–32** neighbours/node and **efSearch ≈ 100–200**, nodes *visited* lands in the **high hundreds to low thousands**. So ~500 is an order-of-magnitude figure, and it rises as you raise efSearch for recall. ⚠ Each visited node needs **two** things — its **vector** (for the distance) and its **neighbour list** (where to go next) — so 2 reads/hop unless they're stored adjacently.

**~100 µs** — SSD **random-read latency**. Modern enterprise NVMe is better (~20–80 µs); SATA is ~100–200 µs. So 100 µs is a conservative round number.

```
500 hops × 100 µs = 50 ms          vs  RAM: 500 × ~100 ns = 50 µs  (~1000×)
```

### ⭐ But the real argument is *serial*, not the latency number

An NVMe drive does **500k–1M IOPS** — but only at **high queue depth**, many requests in flight. A dependent chain can't do that:

> **You cannot issue hop *k+1* until hop *k* returns, because hop *k*'s contents tell you where to go next.**

So you're bounded by **latency, never throughput**. Queue depth 1 = `1/100 µs` = **10,000 reads/sec — 1–2% of the drive's capability.** The SSD's single greatest advantage is unusable. That's why it's *"latency-infeasible"* rather than *"slow"* — a structural property of the access pattern, not a hardware generation you can wait out.

**Honest caveats:** modern NVMe narrows it (~15 ms, still over budget, less dramatic) · page cache helps the small upper layers that cache well and not the big bottom layer where the hops are · and ⭐ **this is exactly why DiskANN is built differently** — it keeps compressed vectors in RAM for the *walk* and hits SSD only for **full vectors of a known candidate set**: a **batched, parallel** read that *can* use queue depth. Best evidence the argument is right.

---

## 6 · What is "hash sharding", and why not semantic?

**⚠ Hashing does NOT group similar things — it deliberately destroys grouping.**

```
shard = hash(image_id) % N
```

A hash scatters IDs **uniformly and arbitrarily**: two visually identical sofas land on different shards as often as not. **No locality of any kind — that's the feature.** *(Semantic sharding is the alternative that groups: furniture on shard 1, electronics on shard 2.)*

**Semantic sounds better** — a sofa query would hit only shard 1. It fails twice:

**Hotspots.** Traffic isn't uniform across categories. 40% clothing queries ⇒ that shard takes 40% of load while others idle. You must provision each shard for *its own* peak, can't rebalance by adding nodes (the skew is in the data placement), and traffic shifts seasonally so the scheme goes stale.

**⚠ Recall cliffs — the one that breaks correctness.** Routing semantically requires **classifying the query image** first. But objects don't respect your taxonomy — is a futon furniture or bedding? A smartwatch electronics or accessories? The classifier picks one shard, and **the genuinely nearest neighbours on the unqueried shard are never retrieved.** "Cliff" because it's discontinuous: category centres work, boundaries fall off sharply, and **average recall looks fine while a specific query class is quietly broken** — and a first-stage miss is unrecoverable.

> ⭐ **With hashing there's no classifier, so no misrouting.** Every shard searched, every candidate considered, **recall equals an unsharded index**. You pay N× total work to buy correctness that doesn't depend on a routing decision being right. *Boring and even beats clever and skewed.*

**Scatter-gather** = query all N shards in parallel, each returns local top-k, merge. **Replicate 2–3×** = each shard on 2–3 machines, for availability (a lost node would otherwise silently drop 1/N of every candidate set) and QPS.

---

## 7 · Why RANSAC? Why not just metadata?

**What it does.** Extract local features (SIFT/ORB-style keypoints) from both images, match by descriptor — many matches are wrong. RANSAC repeatedly samples a minimal subset, fits a **homography**, and counts agreeing matches. If a **single consistent transform explains many keypoints**, the images show the **same physical object** under a different camera pose. No consensus ⇒ different objects that merely look alike.

**Why it's the right tool:** it answers what embeddings structurally cannot. An embedding says *"these look similar"*; RANSAC says *"these are the same rigid object."* Two sofas from one product line are near-identical in embedding space and distinguishable by actual geometry — stitching, leg placement, proportions. It's also **robust to large outlier fractions**, which is the regime here.

**Alternatives worth naming:**

| Approach | Note |
|---|---|
| **SuperPoint + SuperGlue / LightGlue** | The modern answer — a GNN matches keypoints jointly. Better on hard cases, needs GPU in the path |
| **MAGSAC++ / GC-RANSAC** | Direct upgrades to RANSAC; same role, better estimates |
| **Re-ranking transformers** | Cross-attention over patch tokens; learns verification end-to-end, no explicit geometry |
| **DELF / DELG** | Learned local features, still verified geometrically |

> **In the room:** say **"geometric verification"** as the concept, RANSAC as the classic instance, and note learned matchers are current SOTA. The concept is what's being graded.

**⚠ Why not metadata/specs — you don't have them at query time.** The user **photographed a sofa in their living room**: no SKU, no brand string, no spec sheet attached to a photograph. Metadata lives on the **catalogue** side and is **what you're trying to find** — resolving image → SKU *is* the problem, not an input to it.

Metadata does three real jobs, all **after** matching: narrowing when the user also supplies text, re-ranking survivors (price, availability, merchant trust), and verifying the catalogue's own attributes. *(And per G7: supplied attributes are **claims, not facts** — merchants mis-state them.)*

---

## 8 · §5 — that's training the embedding model, right?

**Yes** — §5 trains the **image encoder** that produces the vectors going into the ANN index. Two things to hold onto:

**⚠ Hard negatives are the whole game.** Random negatives teach *"sofa ≠ bicycle"*, which the model already knows. The ones that matter are **different SKUs from the same product line** — same colour, same silhouette, one design detail apart. Sources: **catalogue structure** (same brand, same category, different model number — free and surprisingly strong), **mining the current index** for high-scoring non-matches, and **confirmed production errors**. ⚠ Beware **false negatives in mining** — a mined "negative" that's the same product under another listing poisons training.

**⭐ Augmentation defines the invariances** — the sharpest line in the section. *What you augment over is what the model learns to ignore.* Augment lighting/angle/crop/background ⇒ invariant to them.

> **Which is why same-product and semantic-similarity need OPPOSITE augmentation policies** — same-product must stay *sensitive* to small design differences that a semantic model should ignore. **That's the training-level version of "one embedding space can't serve all three."**

---

## 9 · What's the "small embedding for the common path"?

**The big index is in the cloud** — you had this right.

The on-device model is a **small, distilled encoder** doing three jobs:
- **Crop / region proposal** — what is the user pointing at? Interactive, must feel instant.
- **Intent classification** — routing (but see §10).
- **An on-device cache** for repeat / near-repeat queries — point at the same object twice, no round trip.

⚠ It produces embeddings in a **small model's space**, which is **not** the space the 10⁹-image index was built in — so **it cannot query that index.** A real search embeds in the cloud (or uploads the crop) and searches there.

**Why split at all:** crop and intent need to feel instant and shouldn't cost an upload; the heavy work — full-corpus ANN + geometric verification — needs the big model and the big index, which only exist in the cloud.

> ⭐ **Contrast with G3 (Photos)** — the instructive comparison. There the on-device model **is the whole system**, because the corpus is 10k images/user (~19 MB) and fits on the phone. Here it's 10⁹ shared images (~512 GB), which can never be on-device, so on-device is only ever a front-end. **Same technique, opposite architectural role, because the scale is inverted.**

---

## 10 · The intent classifier needs labelled data — where from?

*(This one changed the sheet — it's now a ⚠ note in §4 and follow-up 10.)*

**⚠ Intent is not a property of the image.** The same sofa photo is same-product if they're shopping and semantic if they're decorating — so it **can't be labelled from the pixels**, two annotators will disagree, and that disagreement becomes the model's noise floor *(G25's taxonomy argument)*.

**Label sources, increasing honesty:**
- **Human annotation on (image + context)** — needs an adjudicated, versioned definition of the three classes *first*, or κ is low and the number is meaningless.
- **⭐ Behavioural, from production** — tapped through to a merchant ⇒ same-product; browsed a similar-category grid ⇒ semantic. Free, at scale, real distribution — but **circular**: you only observe behaviour on results you showed, so you need a seed system first.
- **Weak supervision from the surface** — *"Shopping tab ⇒ same-product"*. Noisy, no annotation, and the realistic bootstrap.

**⭐ The cheaper path: most of the signal is in the context, not the pixels.**

| Signal | Tells you | Cost |
|---|---|---|
| **Entry surface** (Shopping tab vs design app) | Usually decisive alone | Free |
| **Accompanying text** ("where can I buy this") | Explicit | Free |
| **Coarse object category** | sofa → shoppable · plant → semantic | ~Free (detection already runs) |
| **Session history** | Already tapped a merchant link | Free |

A small decision tree over those needs **no intent labels at all** and is debuggable.

### ⭐ The strongest version: don't classify — run both

Query the same-product **and** semantic indexes in parallel, and let the **surfacing** reflect confidence:

```
confident homography found  → "This is the X.  Buy it"
no geometric consensus      → "Similar items"
```

That's genuinely better, because **geometric verification already IS an intent detector** — a confident match means a same-product answer *exists*; no consensus means it doesn't, whatever a classifier predicted. **You've replaced a prediction about user intent with evidence about what the corpus can deliver.** Cost: two retrievals, affordable because retrieval is cheap and verification is the expensive stage.

> **The L6 move here** is challenging your own box rather than defending it — same shape as **G17's "notice it isn't an online ML problem."** Recognising that an expensive component **may not need to exist** reads better than justifying it.

---

## 11 · "Rules first… intent correlates more with *where the user is* than *what they photographed*" — explain

The claim: **you can predict what someone wants better from where they are in the product than from what's in their photo.**

### Why the photo is weak evidence

Same image — a sofa — three users:

| User | What they want |
|---|---|
| Browsing a retail app | **Same-product** — "buy this exact sofa" |
| In an interior-design app | **Semantic** — "show me sofas like this" |
| Tagging their own photos | **Visual** — "more images with this palette" |

⚠ **Identical pixels, three different intents.** The photo tells you *what the object is*; it tells you almost nothing about *what the user plans to do with it*. So an image-only classifier is trying to predict something **the image doesn't contain** — which is exactly why the labels are unobtainable and annotators disagree.

### Why context is strong evidence

The **entry surface** alone often settles it, with no image at all:

- **Shopping tab** → they're shopping → same-product
- **Plant-ID flow** → they want to know what it *is* → semantic
- **Design tool** → visual similarity

⭐ **The user has already told you their intent by choosing where to stand.** Surface choice is a **declared** intent; the photo is at best a weak correlate of one.

Same for the rest: **text** (*"where can I buy this"* — explicit, done) · **session history** (tapped a merchant link two queries ago ⇒ shopping) · **coarse object category** — the only *image*-derived signal in the list, and a weak prior (sofas and shoes usually shoppable; plants and animals usually not).

### Why that makes "rules first" the sequencing

> ⭐ **The high-signal features are free and need no labels. The expensive, un-labelable feature is the weak one.**

So a decision tree over surface + text + category + history captures most of the available signal at **zero labelling cost**, and it's **debuggable** — which a classifier isn't. Training a model buys you the **residual**, and the residual is the genuinely ambiguous slice. For *that* slice, don't train either: **run both retrievals and let geometric verification arbitrate** (§10).

### ⭐ The general pattern worth carrying

> **Before training a model to predict something, check whether the user has already told you.**
> Surface, explicit text, prior actions are **declarations, not inferences** — and a declaration beats a prediction every time.

Same instinct as **G17's** *"conditioned on an item, not a user, so it precomputes"* and **G4's** *"a trie problem with a thin ML layer, not an ML problem with a cache."* **Recognising where the ML genuinely earns its place is the move.**

---

## 🔁 THE SIX THINGS I KEPT GETTING WRONG

1. **4 B is per *dimension*, not per embedding.** 512-d fp32 = 2 KB.
2. **Hash sharding scatters; it does *not* group.** Grouping is the thing it's avoiding.
3. **Recall and precision@1 aren't competing** — different stages. Recall = the ceiling; precision@1 = the product outcome.
4. **The on-device embedding can't query the cloud index** — different space. It's a front-end, not a shortcut.
5. **"Position, with the reason"** = a stance to defend, not a scope cut. Say the *because* in the same breath.
6. **Intent lives in the context, not the pixels** — the surface is a *declaration*; the photo is a weak correlate. Check what the user already told you before training anything.
