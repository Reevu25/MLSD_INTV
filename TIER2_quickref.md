# TIER 2 — QUICK REFERENCE
### High probability · know the opening and the diagram

> One page per question: **the opening**, **the diagram**, **the defensible choices**, **the number**.
> Full sheets are `*_intv.md`. For Tier 2 the realistic target is: **deliver the opening cleanly, draw the diagram, and hold the three choices.** Depth comes from the interviewer.

> ⭐ **Graded against Google's L6 form (3 parameters, 10 sub-parameters):** all ten Tier-2 sheets are rewritten in the same mould as Tier 1 ([R1](R1_prompt_suggestions_intv.md) / [R2](R2_video_query_search_intv.md)), on the same 60-minute spine (see [TIER1](TIER1_quickref.md)): **§0** opening · **§1** requirements (questions asked unprompted + an assumptions table) · **§2** product + metrics · **§3** capacity · **§4** architecture (alternatives, a TASK/OUTPUT stack, problems raised before you're asked) · **§5** data, training-set selection and the objective · **§6** serving · **§7** eval, rollout and bias · **§8 production-readiness sweep** (all nine pointers: performance · reliability · usability · scalability · privacy · safety · security · ethics/bias · explainability, at ≈ min 44–48) · **§9** extension + failures, with a fallback pipeline for when the main design fails. Every heading and follow-up carries its rubric codes, and each sheet ends with a 32-pointer coverage map. This page stays the skim layer.

---

## 11 · G2 — YouTube Video Search
*Search ranking, the third classic.*

**OPENING.** *"For the large majority of video-search queries, **the pixels are the least useful modality**. Title, description and the ASR transcript carry most of the retrievable signal; engagement carries most of the ranking signal. So: **text-first retrieval**, and visual budget only on the ~15% where text demonstrably fails."*

**TWO COMMITMENTS.** Settle **the unit of retrieval** (video-level everywhere, **moment-level only for the head**) · **⚠ the label is watch-through, not clicks — a click plus a three-second watch is a negative**.

```
QUERY → query understanding (is this visual? is this a moment query?)
  → [BM25 over title+desc+TRANSCRIPT ∥ DENSE ∥ ENGAGEMENT PRIOR] → RRF
  → (HEAD ONLY) moment-level transcript chunks
  → MULTI-OBJECTIVE RE-RANK (+ negative head: fast abandon)
  → results + timestamp deep-links
```

**DEFENSIBLE CHOICES**
- **Text-first, visual only where it pays** — pixels are largely redundant with accompanying text. *Cost: visual queries on tail content served poorly.*
- **Head/tail indexing asymmetry** — moment-level over the whole corpus is 100× the index for content nobody searches. *Cost: a tail video's best moment isn't findable.*
- **Keep BM25** — video queries are often literal; BM25 over a transcript is hard to beat. *Cost: two indexes.*

**NUMBER.** Moment-level for the head (~1% of 10⁹) ≈ **5×10⁸ chunks**; over the whole corpus it's **5×10¹⁰**. That's the cost decision.
**⭐** *A bad ASR transcript makes a good video invisible — and it concentrates in under-invested languages.*

---

## 12 · Q8 — Pretraining Data Pipeline
*"Design the data pipeline" is a standard MLOps prompt.*

**OPENING.** *"**The data pipeline is the model.** Architecture is commoditised and compute is fixed by budget — the largest lever anyone has is what tokens go in. And **the mixture is a decision that must be measured, not argued**: the pipeline's most important output isn't the corpus, it's **a fast ablation loop at 1B scale** that tells us which corpus to build."*

**THE FAILURE TO DESIGN AGAINST.** ⚠ *"A data bug doesn't crash — it silently produces a slightly worse model. You find out three weeks and three million dollars later."*

```
10 PB raw → lang ID + heuristic quality → 2 PB → exact dedup → 1 PB
   → ⚠ FUZZY DEDUP (MinHash + BANDED LSH) → 300 TB   ← THE COST
   → model-based quality filter → 100 TB → DECONTAMINATION → 95 TB
   → mixture + curriculum → 60 TB           ▲
                                   1B ABLATION LOOP feeds back
```

**DEFENSIBLE CHOICES**
- **Fuzzy dedup despite the cost** — near-duplicates are most of a crawl and exact hashing misses all of them. *Cost: a PB-scale shuffle with severe bucket skew.*
- **Ablation-driven mixture** — nobody predicts it from first principles. *Cost: 1B conclusions don't always transfer.*
- **Provenance at ingest** — retrofitting origin onto trillions of tokens is impossible. *Cost: schema overhead from day one.*

**NUMBER.** Banded LSH: `P(collision) = 1 − (1 − s^r)^b`. ⚠ **Bucket skew**: boilerplate buckets are enormous — a few reducers run for days.
**⭐** *Over-filtering is as damaging as under-filtering — and every data-quality stat improves while it happens.*

---

## 13 · Q12 — LLM-as-Judge
*The inevitable follow-up to any eval answer.*

**OPENING.** *"**A judge is a model, not a prompt.** It has training data, systematic biases, it drifts, it needs versioning — and the part everyone skips, **it needs its own evaluation**. If I can't tell you how accurate the judge is, every number downstream is decoration."*

**THE ONE TO BE GRADED ON.** ⭐ *"The moment a judge becomes a training reward, it stops being a measurement instrument and becomes **an adversarial target**."*

```
RUBRIC REGISTRY (versioned; κ per dimension + HUMAN CEILING stored in it)
  → ONE SCORING LIBRARY (shared by all 5 consumers — a correctness req.)
  → CALIBRATED FAMILY: TEACHER ──labels──► STUDENT (~8B)
       ▲ ~2% escalation      ⚠ VALIDATE STUDENT AGAINST HUMANS, NOT TEACHER
  → calibration loop (500 stratified, labelled WITH OVERLAP)
  → drift + gaming monitors (anchor set · canaries · verifiable slice)
```

**DEFENSIBLE CHOICES**
- **Pairwise over pointwise** — "is A better than B" has far better inter-rater reliability than "is this a 7". *Cost: needs a comparison.*
- **Distil, validate against humans** — 95% agreement with the teacher means you distilled its biases. *Cost: student needs its own labels.*
- **Specialists, not one general judge** — a general judge is mediocre everywhere. *Cost: N judges to calibrate.*

**NUMBER.** 70B teacher ≈ **$0.004/score** → **$40k per RL run**, $400k for filtering. Distilled 8B ≈ **30× cheaper**.
**⭐** *Human inter-rater agreement is the **ceiling** — 85% against 70%-consistent labels is uninterpretable.*

---

## 14 · G1 — Visual Search (Lens)
*Core Google product; ANN at scale.*

**OPENING.** *"There are **three different products** hiding inside 'visual search' — **visually similar**, **semantically similar**, and **same product / same instance** — and they need different labels, losses, indexes and evaluation. A single embedding space cannot serve all three, because **the notion of distance is genuinely different**."*

**THE LABELS COMMITMENT.** ⚠ *"Co-clicks are position- and popularity-biased, so a model trained naively retrieves **popular** things rather than **matching** things — and the metric approves, because it's built from the same clicks. **Grading the homework with your own answer key.**"*

```
CAMERA → REGION PROPOSAL/CROP → ⭐ INTENT CLASSIFIER (which of the three?)
  → EMBEDDING (on-device or cloud) → ANN over 10⁹ → top ~100
  → ⭐ GEOMETRIC VERIFICATION (RANSAC)  ← same-product, AND the abstention
  → re-rank (availability, merchant trust) → results, or an honest "no match"
```

**DEFENSIBLE CHOICES**
- **Separate spaces + intent router** — contradictory invariances. *Cost: multiple indexes; a router that can be wrong.*
- **Geometric verification on top-k** — embeddings say "looks like"; RANSAC says "**is**". *Cost: ~80 ms.*
- **Abstain rather than assert** — a confidently wrong product sends money to the wrong merchant. *Cost: visible no-match rate.*

**NUMBER.** 10⁹ × 512-d fp32 = **2 TB** → int8 **512 GB** → PQ **~65 GB**.
**⭐** *Evaluate on a **human-labelled, non-click-derived** set, sliced head vs tail.*

---

## 15 · Q13 — Customer Support Agent
*The most common applied-agent prompt.*

**OPENING.** *"Start with the metric, because it's where this goes wrong. **Deflection is the wrong north star** — the fastest way to raise it is to make reaching a human harder, so you get a bot that traps people and generates a second angry contact the metric doesn't attribute back. I'd optimise **resolution rate**: no repeat contact within seven days."*

**TWO COMMITMENTS.** **The design centre is authorization, not answer quality** (worst case is a wrongful $5,000 refund) · ⚠ **escalation is a product surface, not a failure**.

```
MESSAGE → intent + confidence (low ⇒ ESCALATE EARLY) → IDENTITY VERIFICATION
  → retrieval ⚠ ACL-SCOPED TO THIS CUSTOMER
  → TIERED ACTIONS: T0 read │ T1 reversible │ T2 bounded financial │ T3 HUMAN
     ⚠ + a COMPENSATING TRANSACTION for every autonomous action
  → ⭐ WARM HANDOFF (transcript · what was tried · diagnosis · verified identity)
```

**DEFENSIBLE CHOICES**
- **Resolution, not deflection** — deflection is gameable by making the product worse. *Cost: lower headline automation.*
- **Narrow scope, early escalation** — 20 intents end-to-end beats 500 at 60%. *Cost: visibly lower coverage.*
- **Every autonomous action reversible** — no compensating transaction ⇒ it's T3 by definition. *Cost: some safe actions still need a human.*

**NUMBER.** Top ~20 intents are **70–80% of volume**. A *badly* handled contact costs handle-time **plus** the repeat **plus** the CSAT damage.
**⭐** *The warm handoff makes the agent a colleague to the support org rather than a wall — and they can kill this product.*

---

## 16 · G22 — Delayed Conversion Prediction
*The Ads-team deep dive; the cleanest censored-label question.*

**OPENING.** *"**A negative label is never final.** 'Has not converted yet' mixes 'will never convert' with 'will convert tomorrow', and at training time I can't tell them apart — so recent data is contaminated with false negatives and the model **systematically under-estimates**. And it's a **calibration** bias, so AUC won't move while the platform under-bids."*

**KILL BOTH NAIVE ANSWERS.** ❌ *"Wait 30 days"* = a model trained on a world that's gone. ❌ *"One-day window"* = worst bias in the **highest-value verticals**. ✅ **Model the delay.**

```
⭐ THE CENSORED LIKELIHOOD — write it on the board:
   observed at delay d :  p · f(d; λ)
   NOT YET, elapsed e  :  (1 − p) + p · S(e; λ)
                           ↑ never      ↑ WILL, hasn't yet ← the whole idea

CLICK → joint model {head: p, head: λ} → calibrated pCVR → × bid
      → labels stream in 30–60 d → importance weighting → ⚠ MUTABLE ROWS
```

**DEFENSIBLE CHOICES**
- **Model the delay, don't pick a window** — lets recent data contribute *correctly weighted*. *Cost: a parametric assumption.*
- **Per-vertical λ** — B2B and impulse retail differ by orders of magnitude. *Cost: sparser per-vertical data.*
- **Fresh + corrected over clean + stale** — creatives change hourly. *Cost: always correcting an estimate.*

**NUMBER.** The diagnostic plot: **predicted vs actual, bucketed by how long ago the click happened.** Healthy is flat; **naive slopes down on recent clicks.**
**⭐** *Attribution **defines** the label — changing it is a metric change, and it must be stamped on every row.*

---

## 17 · Q4 — RL Post-Training
*Standard at L6.*

**OPENING.** *"The output isn't a service — it's **a better policy plus the evidence it's better**. Three things in order: **a reward we can trust** (a fast pipeline optimising a hackable reward produces a worse model faster), **rollout throughput**, and **an eval that gates weights** — because a run costs ~$60k."*

**THE REWARD COMMITMENT.** ⭐ **Verifiable first, decomposed always** — *"a single scalar is how hacking hides."*

```
TASK POOL ⚠ curated for GRADIENT (all-pass/all-fail ⇒ zero variance ⇒ no learning)
  → ACTOR FLEET → ⚠ ENVIRONMENT FLEET (the usual bottleneck)
     ⚠ THE GRADER LIVES OUTSIDE THE ENVIRONMENT
  → REWARD SERVICE (decomposed, components logged separately)
  → LEARNER (PPO/GRPO) + FROZEN REFERENCE ⚠ KL IS THE LEASH
  → ⚠ EVAL GATE → promote or discard
```

**DEFENSIBLE CHOICES**
- **Verifiable reward where possible** — a checker is deterministic and ungameable. *Cost: limits you to checkable domains.*
- **GRPO over PPO** — drops the value network. *Cost: needs a group per prompt; higher rollout cost.*
- **Curate for gradient** — all-pass/all-fail tasks burn budget and teach nothing. *Cost: ongoing curation as the policy improves.*

**NUMBER.** **RL is bottlenecked on generating experience, not gradient steps** — and for agents the **environment fleet**, not the GPUs.
**⭐** *"Goodhart isn't a warning here, it's a **schedule**." RM score climbing while the verifiable slice is flat is **proof**.*

---

## 18 · G4 — Query Autocomplete
*Classic Google, deceptively deep.*

**OPENING.** *"Two things. The **latency budget is brutal and the QPS enormous** — this fires on every keystroke, so autocomplete QPS is **10–20× search QPS** at **50 ms**, which rules out any neural model at serving time. And — the framing I'd build around — **a suggestion is a publication.** We're putting words in the user's mouth and appearing to endorse them."*

**THE ARCHITECTURAL CONSEQUENCE.** ⭐ *"I **retrieve** suggestions from a vetted set of real queries; I do not **generate** them — a generated suggestion makes the defence surface unbounded."* And: **showing nothing is a perfectly good output.**

```
PREFIX → normalise → TRIE/FST over the VETTED QUERY SET (~1–5 ms)
       → thin scoring (popularity · trending · locale)
       → ⭐ SAFETY LAYER (blocklists · NAMED-ENTITY rules · ⚠ DEFAULT SUPPRESS)
       → suggestions, or nothing

OFFLINE: logs → aggregate → ⚠ k-ANONYMITY → spam filter → SAFETY VETTING
       → score → build FST → push to EVERY replica
```

**DEFENSIBLE CHOICES**
- **Retrieve, never generate** — keeps the candidate set finite and **reviewable in advance**. *Cost: no coverage for novel phrasings.*
- **Replicate, don't shard** — the corpus fits; sharding adds a hop to a 50 ms budget. *Cost: updates fan out.*
- **Suppress by default on doubt** — an empty list costs a keystroke. *Cost: visible gaps on names and breaking news.*

**NUMBER.** A 20-character query = **20 requests**. Do that arithmetic out loud — it decides the architecture.
**⭐** *k-anonymity is simultaneously the privacy control and the quality control.*

---

## 19 · Q9 — Agent Memory
*Personalization + privacy; where L6 separates.*

**OPENING.** *"**Memory is not a storage problem, it's a curation problem.** Storing everything is trivial and produces a worse product. So the engineering is on the **write path** — deciding what earns a memory — not the read path, which is mostly solved. And the gate is conservative by design: **a wrong memory is far worse than a missing one.**"*

**THE SECURITY REFRAME.** ⚠ *"Memory turns prompt injection into a **persistent** attack — it fires in every future session, invisibly. **The write path is a security boundary.**"*

```
TURN ──► READ (easy): retrieve top-k, ⚠ INJECT AS DATED EVIDENCE not instruction
     └─► ⭐ WRITE PATH (the design):
          extract structured claims → ⚠ SOURCE_TRUST (only user statements
          create durable facts) → instruction detector → CONSERVATIVE GATE
          → conflict check ⇒ ⭐ SUPERSEDE, keep history
     → store partitioned by user_id → ⭐ USER-VISIBLE MEMORY UI
```

**DEFENSIBLE CHOICES**
- **Retrieval, not per-user fine-tuning** — weights aren't inspectable or deletable. *Cost: must fit the context budget.*
- **Structured claims, not prose** — prose can't be deduped, conflict-checked, shown, or precisely deleted. *Cost: harder extraction.*
- **Supersede, don't overwrite** — explainable and reversible. *Cost: storage grows.*

**NUMBER.** Scale is **inverted vs Q6**: not one index of 200M chunks, but **200,000 small indexes of ~2,000 vectors**.
**⭐** *Visibility pays three times: privacy obligation, correction signal, and the best poisoning defence. Rare alignment — take it.*

---

## 20 · G7 — Shopping Search Ranking
*Multi-objective with a real three-party conflict.*

**OPENING.** *"**Three parties with genuinely conflicting objectives** — buyer, seller, platform — and in shopping, relevance and business value **demonstrably do not** point the same way. So the decision I'd defend hardest is **how the conflict is expressed**: not a blended score, but **constrained optimisation** — maximise revenue subject to a relevance floor, a price floor, a seller-quality floor."*

**WHY.** ⭐ *"With a blended score, a high enough bid buys any position, and six months later nobody can tell whether quality regressed from the model or from auction pressure. **A constraint is a thing you can point at in a design review; a weight is not.**"*

```
QUERY → STRUCTURED INTENT {category, colour, size, price_max}
  ⚠ ~90% of this query is STRUCTURED FILTERING, not semantic matching
  → filter + semantic over the NORMALISED catalogue
  → PRODUCT DEDUPLICATION (N sellers → one result)
  → ⭐ maximise E[revenue] s.t. RELEVANCE ≥ floor, PRICE ≥ floor, QUALITY ≥ floor
  → results → click → purchase → ⚠ RETURN (30-day lag)
```

**DEFENSIBLE CHOICES**
- **Constraints, not a blended weight** — a bid shouldn't buy any position. *Cost: leaves revenue on the table, visibly.*
- **Purchase-*without-return*** — optimising purchase sells things people send back. *Cost: a 30-day gate lag.*
- **Catalogue before model** — wrong attributes can't be recovered by any ranker. *Cost: unglamorous, hard to fund.*

**NUMBER.** The **shadow price** of the relevance floor tells you exactly what it costs in revenue — which makes the trade a conversation.
**⭐** *Attributes are **claims, not facts** — supplied by parties with an incentive to mis-state them.*

---

### THE TIER-2 PATTERN MAP

| Pattern | Appears in |
|---|---|
| **The label is the whole question** | G2 (watch-through) · G7 (without-return) · G22 (censored) |
| **Constraints beat weights** | G7 (floors) · Q4 (multiplicative gates) · Q13 (action tiers) |
| **Precompute, then serve a lookup** | G4 (FST) · G2 (head moments) · Q12 (distilled student) |
| **Calibration against a human ceiling** | Q12 (κ vs inter-rater) · G1 (non-click-derived gold set) |
| **The write path is the security boundary** | Q9 (memory) · Q13 (authorization) · G4 (vetted set) |
| **Measure, don't argue** | Q8 (ablation loop) · Q12 (κ) · G22 (recency plot) |
