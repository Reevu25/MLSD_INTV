# TIER 3 — QUICK REFERENCE
### Worth having · don't be surprised

> One page per question: **the opening**, **the diagram**, **the defensible choices**, **the number**.
> Full sheets are `*_intv.md`. For Tier 3 the target is **not to be surprised** — read the opening, know the diagram exists, recognise the pattern. Several of these are variants of questions you already know well; the cross-references say which.

> ⭐ **Graded against Google's L6 form (3 parameters, 10 sub-parameters):** all ten Tier-3 sheets are rewritten in the same mould as Tiers 1 and 2 ([R1](R1_prompt_suggestions_intv.md) / [R2](R2_video_query_search_intv.md)), on the same 60-minute spine (see [TIER1](TIER1_quickref.md)): **§0** opening · **§1** requirements (questions asked unprompted + an assumptions table) · **§2** product + metrics · **§3** capacity · **§4** architecture (alternatives, a TASK/OUTPUT stack, problems raised before you're asked) · **§5** data, training-set selection and the objective · **§6** serving · **§7** eval, rollout and bias · **§8 production-readiness sweep** (all nine pointers, at ≈ min 44–48) · **§9** extension + failures, with a fallback pipeline. For the non-classic prompts the sections map naturally: in Q19 "serving" is containment at runtime, in Q1 it's running the job, in Q26 the update ladder, in G23 running experiments. Every heading and follow-up carries its rubric codes, and each sheet ends with a 32-pointer coverage map. This page stays the skim layer.

---

## 21 · Q19 — Prompt Injection Defence
*Explicitly named in 2026 interview research. The deep dive behind Q10's T2.*

**OPENING.** *"The honest position first: **prompt injection has no reliable model-level solution today.** An LLM processes instructions and data in the same channel — that's a property of how they work, not a defect — so any defence relying on the model distinguishing content from instructions is probabilistic and fails under adversarial pressure. **So I don't design to prevent injection. I design so that a successful injection is insufficient to cause harm.**"*

**THE REASONING TOOL.** 🔺 **The lethal trifecta** — *private data + untrusted content + an exfiltration channel. Remove any one. The cheapest is almost always **the exfiltration channel**.*

```
THREE PATTERNS, increasing strength and cost:
 1. TAINT TRACKING    ⚠ taint spreads — DECLASSIFICATION is where
                        the security actually lives
 2. ⭐ DUAL-LLM        privileged model NEVER sees untrusted text;
                        quarantined model returns TYPED values only
 3. CONTROL/DATA-FLOW plan derived BEFORE untrusted data is read
    SEPARATION
```

**DEFENSIBLE CHOICES**
- **Containment over detection** — probabilistic defences fail under pressure. *Cost: you accept injections will succeed.*
- **Cut exfiltration first** — ⚠ including markdown **image rendering and link generation**, which leak silently on render. *Cost: an allowlist.*
- **Dual-LLM where stakes justify it** — deterministic, not probabilistic. *Cost: no free-form passthrough.*

**NUMBER.** Every channel the model reads is a vector — web, email, docs, **tool responses, other agents' output**, file names, EXIF, **invisible text**, and the RAG corpus itself.
**⭐** *"Cut capability, not controls." And: untrusted content can **never** write durable memory — that's what makes an attack permanent.*

---

## 22 · G17 — Similar Items / "You May Also Like"
*Cheap to learn; two definitional questions carry it.*

**OPENING.** *"Two questions decide this, and both are about **definitions rather than models**. **Similar in what sense?** — substitutes, complements, stylistic, same-category — and **the placement decides**: on a product page the user is comparing (substitutes), in the cart substitutes are useless and complements convert. And: **co-occurrence data is dominated by popularity** — raw counts return the best-seller for almost every item."*

**THE STRUCTURAL SIMPLIFICATION.** ⭐ *"This is conditioned on an **item, not a user** — so it's **precomputable**. Top-K offline, a KV lookup at serve time, **no online model in the hot path**."*

```
OFFLINE: logs → ⭐ PMI/LIFT, not raw counts  + MINIMUM SUPPORT FLOOR
         → blend content similarity for the TAIL → top-K per relation type
         → KV store:  item_id → [neighbours]
ONLINE:  request(item, PLACEMENT) → pick relation → lookup (~1 ms)
         → light personalised/availability re-rank
```

**DEFENSIBLE CHOICES**
- **PMI/lift over raw co-occurrence** — the best-seller co-occurs with everything. *Cost: rare pairs need a support floor.*
- **Precompute + KV** — noticing it isn't an online ML problem *is* the design. *Cost: staleness between refreshes.*
- **Separate relations by placement** — substitutes after purchase converts terribly. *Cost: multiple lists.*

**NUMBER.** 10⁷ items × K=50 × ~16 B ≈ **8 GB**. A hash lookup, not a model.
**⭐** *Evaluate on the **tail** — aggregate CTR approves a system that returns the top-50 chart to everyone.*

---

## 23 · Q2 — Coding Agent
*Very likely if the team is DevInfra.*

**OPENING.** *"A coding agent is **a deterministic, observable control system wrapped around a stochastic core**. The model reasons, the orchestrator controls, the sandbox executes, the verifier judges — and almost all the interesting engineering is **in the loop and its bounds**, not in the prompt."*

**THREE PRIORITIES.** **A trustworthy verifier** (tests: a signal you can ship on *and* train on) · **context as a managed budget** (an agent turn is ~4× a chat turn) · **bounded autonomy**.

```
TASK → ORCHESTRATOR (⚠ termination · cost cap · NO-PROGRESS DETECTOR)
     → CONTEXT MANAGER (⚠ pinned plan slot; silent truncation is the #1 bug)
     → MODEL → SANDBOX ⚠ THE GRADER LIVES OUTSIDE IT
     → VERIFIER (tests · lint · types) → PR with diff + rationale
     → TRACE STORE → EVAL HARNESS → RL LOOP  ← what makes it IMPROVE
```

**DEFENSIBLE CHOICES**
- **Scope to test-backed tasks** — verifiability is what makes eval and RL tractable. *Cost: excludes greenfield.*
- **Grader outside the sandbox** — otherwise the agent edits the tests, reliably. *Cost: an extra boundary.*
- **PR-only, never direct commit** — a PR is reviewable and revertible. *Cost: slower perceived autonomy.*

**NUMBER.** Gate on **`pass^k`, not `pass@k`** — a user gets one attempt. 70% `pass@5` with 25% `pass^5` feels broken in production.
**⭐** *The agent's capability ceiling is set by **the repo's verifiability**, not by the model.*

---

## 24 · G6 — Internal Document Search
*RAG + ACLs. **Read [`Q6`](Q6_rag_retrieval_intv.md) first** — this is only the delta.*

**OPENING.** *"**Retrieval quality and access control are the same system, and access control wins every conflict.** So the design centre is one rule: **ACL filtering happens before ranking, never after** — and post-filtering fails in two distinct ways."*

**THE TWO FAILURES.** **Quality**: retrieve 100, filter to 3, and a narrow-access user gets an empty page — *degradation in proportion to how restricted you are*. **Security**: ⚠ *any signal derived from an unseeable document leaks* — counts, scores, related-docs, and worst of all **the summariser**.

```
QUERY + USER → resolve TRANSITIVE PRINCIPAL SET (bounded, few hundred)
  → retrieval with the ACL predicate ⚠ INSIDE the traversal
     (BM25: required disjunction · ANN: bitmap probed at each hop)
  → fuse → AUTHORITATIVE RE-CHECK → rerank → ~10
  → ⚠ GENERATION READS ONLY THESE — all ACL-verified twice
```

**DEFENSIBLE CHOICES**
- **ACL as index terms, filtered during traversal** — the principal set is bounded, so it fits in the query. *Cost: filtered ANN is harder.*
- **The model never reads what the user can't open** — a paraphrase carries no document identity. *Cost: sometimes can't answer.*
- **Authoritative re-check on survivors** — the index ACL is an optimisation, not truth. *Cost: a hot-path dependency.*

**NUMBER.** Cache key **must** include the user's ACL-group hash. Query-only keys leak across users — the classic enterprise-RAG bug.
**⭐** *The invariant is at the **read** boundary, not the output boundary. Permission tests belong in **CI**, not a dashboard.*

---

## 25 · G3 — Photos Natural-Language Search
*Multimodal + on-device. **G1 inverted** — same imagery, opposite scale.*

**OPENING.** *"Two things invert what you'd expect. **The scale is upside down**: Lens is 10 billion images in one index; Photos is 10 thousand per user × a billion users — **19 MB per library**, which fits on the phone. So ANN scale isn't the problem; a phone-sized encoder and **updating it across a billion devices** is. And **privacy is the architecture, not a layer**."*

**THE CLAIM TO CHALLENGE.** ⚠ *"'The photo never leaves the device, we only upload vectors' is **not** the guarantee it sounds like — **embedding inversion is real**. A cloud vector index is a cloud image index with extra steps."* ⭐ *And cloud-embedding a billion libraries is **10¹³ encodes** — infeasible anyway. **Ethics and economics agree; take the win.***

```
ON DEVICE: photo → distilled encoder (~10 ms) → LOCAL INDEX (~19 MB)
           query → text encoder → ⚠ BRUTE-FORCE SCAN (no ANN needed)
           + METADATA PRE-FILTER ("Rome last summer" is a FILTER)
CROSSES THE BOUNDARY: only DP-noised aggregate metrics. No content.
```

**DEFENSIBLE CHOICES**
- **On-device embedding *and* index** — inversion + 10¹³ encodes. *Cost: smaller encoder ⇒ lower quality ceiling.*
- **Brute-force, not ANN** — 50k vectors is a millisecond scan. *Cost: none, until libraries grow.*
- **Metadata filter before semantics** — running a filter as a semantic query is strictly worse. *Cost: query understanding must split intent.*

**NUMBER.** ⚠ **A model update = re-embedding every photo on every device** — and during migration a library is **mixed**, where cross-space similarity is meaningless.
**⭐** *You **cannot look at the data** — so reformulation rate is the best privacy-safe signal, and expect a wide offline/production gap.*

---

## 26 · Q1 — Train a 100B Model
*Only if infra-leaning; high variance.*

**OPENING.** *"The deliverable isn't a cluster — it's **one checkpoint that meets a quality bar, produced once, inside a fixed budget**. So: **it's a single-shot job**, which means everything learnable at small scale must be learned there; **the bottleneck is goodput, not FLOPs**; and **data quality dominates**, because compute is fixed by budget."*

```
⭐ DO THE ARITHMETIC:  FLOPs ≈ 6ND = 6 × 1e11 × 2e12 = 1.2e24
   4096 H100 @ MFU 0.45 → 1.8e18 FLOP/s → ~7.7 DAYS perfect uptime
   ⚠ MEMORY: params 200 GB + grads 200 + ADAM 800 + master 400 ≈ 1.6 TB
     vs 80 GB/GPU → SHARDING IS NOT OPTIONAL

PARALLELISM: 4096 = TP 8 (⚠ INSIDE the node) × PP 8 (across) × DP 64 (+ZeRO)
   ordered by COMMUNICATION COST
```

**DEFENSIBLE CHOICES**
- **TP inside the node, PP across** — TP does 2 all-reduces per layer. *Cost: PP's bubble needs many microbatches.*
- **Goodput, not MFU** — MTBF is half a day. *Cost: checkpointing bandwidth.*
- **De-risk at small scale** — muP transfers hyperparameters from 1B. *Cost: small-scale conclusions don't always transfer.*

**NUMBER.** ~880k GPU-hours ≈ **$2–4M for one run**. That's why it's single-shot, and why the **kill criterion must exist before the sunk cost argues**.
**⭐** *Inference cost should constrain training **today** — GQA is decided here and sets the KV cache forever.*

---

## 27 · G15 — Play Store Recommendations
*Cold start + delayed labels.*

**OPENING.** *"The label is the whole question. **Install is the available objective and it's the wrong one** — it measures **the store listing, not the app**. A polished icon gets an install and the app is uninstalled the next morning. And an install-optimised ranker has a **named failure**: the store fills with hyper-casual apps with great install rates and terrible retention, because that's what the objective asked for."*

**THE RESOLUTION.** ⭐ *"The delayed-label bridge: **install + D1 open as the fast proxy, importance-weighted by the historical retention of that app and category**, with D7/D30 as the slow truth."* Plus: **this is a permanent cold-start domain** — new apps *are* the ecosystem.

```
USER → SOURCES WITH QUOTAS (collab ⚠PMI-normalised · content · ⭐NEW-APP EXPLORATION)
  → MULTI-TASK RANKER: p(click) · p(install) · ⭐p(D1 open) · p(D7)
    ⚠ NEGATIVE HEAD: p(uninstall < 24h)
    features: ⚠ CONTENT + METADATA carry the cold start
  → impressions → install (min) → D1 (day) → D7/D30 (weeks)
  → ⭐ STANDING CHECK: does the proxy still predict the truth?
```

**DEFENSIBLE CHOICES**
- **Retention as gate, install as proxy** — install measures the listing. *Cost: 7–30 day lag.*
- **Importance-weight the proxy** — an estimate of the label you can't see yet. *Cost: weights go stale.*
- **Content over collaborative** — new apps have zero history. *Cost: weaker head personalisation.*

**NUMBER.** **D1 open is the cheapest large win** — one day of lag, and it already removes most of the installed-and-never-opened population.
**⭐** *Uninstall-within-24-hours is an **explicit negative**, not a weak positive.*

---

## 28 · G8 — Street View Face & Plate Blurring
*Asymmetric cost, cleanly argued.*

**OPENING.** *"**Recall is not a metric here, it is a legal requirement.** A missed face is a regulatory violation with per-instance exposure; an over-blurred fire hydrant is a marginally worse photograph. **The asymmetry is ~10,000:1.** So the threshold question is already answered, and the interesting question is **how to reach 99.9%+ without blurring the whole world**."*

**TWO THINGS TO RAISE UNPROMPTED.** ⚠ **The retention tension** — improving later requires keeping unblurred originals, which is itself a major liability. ⭐ **The measurement problem** — *"a recall claim I can't substantiate is worthless to the lawyer who has to defend it."*

```
CAPTURE → STAGE 1 cheap high-recall ⚠ THIS STAGE IS THE SYSTEM CEILING
        → STAGE 2 expensive verification (SEPARATE face and plate models)
        → ⭐ GEOMETRIC PROPAGATION across overlapping REGISTERED panoramas
             detect in ONE view → project into ALL → N chances become one
        → BLUR (irreversible, generous padding)
        ▲ USER REPORT ──► blur + ⭐ becomes a MISS LABEL
```

**DEFENSIBLE CHOICES**
- **Operating point far toward over-blurring** — the asymmetry decides it. *Cost: visible over-blurring.*
- **Geometric propagation** — the single largest recall win available. *Cost: needs camera geometry retained.*
- **Separate face and plate models** — entirely different failure modes. *Cost: two gold sets, two operating points.*

**NUMBER.** **Exhaustively-labelled gold set, stratified by difficulty** — a random sample is dominated by easy faces and **overstates recall**.
**⭐** *"Is a billboard face a face?" is a **policy** question — adjudicate and version it **before** annotation, or the recall number means nothing.*

---

## 29 · Q26 — Continuous Learning from Production
*The retraining/monitoring prompt.*

**OPENING.** *"This is **a closed-loop control system, not a training pipeline.** The model we deploy shapes the traffic that trains the next model — so **every in-loop metric is computed on a distribution the model itself created**, and the system can improve on every metric we own while the product gets worse."*

**THE FIRST DECISION.** ⭐ *"Not the labeller or the trainer — **the frozen holdout that lives outside the flywheel**. Drawn from traffic the model doesn't influence, never updated by the loop, refreshed only by a deliberate human act. **I'd build that before the pipeline that feeds it.**"*

```
SIGNALS RANKED BY GAMEABILITY:
  ⭐ VERIFIABLE (CI passed, refund posted) — gold. ⚠ The highest-value work
     is CHANGING THE PRODUCT so it emits more of these.
  IMPLICIT (retry, edit distance, escalation) — dense, confounded
  EXPLICIT thumbs — 0.1–1%, biased toward anger

⭐ THE UPDATE LADDER — velocity tiered by ROLLBACK PATH:
  index MINUTES │ prompts/configs HOURS │ routers DAILY
  │ adapters WEEKLY │ ⚠ BASE WEIGHTS MONTHLY
```

**DEFENSIBLE CHOICES**
- **Frozen holdout before the pipeline** — no fixed point ⇒ no way to detect a slow slide. *Cost: months before it's useful.*
- **Velocity tiered by rollback** — each layer moves as fast as its rollback allows, no faster. *Cost: slower adaptation where it matters.*
- **Propensity logged on every decision** — ⚠ it cannot be reconstructed later. *Cost: hot-path logging discipline.*

**NUMBER.** ⚠ **Join completeness is a monitored metric** — a silent drop in label joins looks *exactly* like a change in the metric itself.
**⭐** *"Continuous learning" does **not** mean continuous weight updates — most of the value lives in the top two rungs.*

---

## 30 · G23 — Ads Evaluation Framework
*The Ads deep dive on experimentation.*

**OPENING.** *"I want to challenge the premise: **'run an A/B test' does not work in an auction.** User-level randomisation assumes the arms are independent, and with a shared auction and shared budgets **they are not**. So the measured difference contains an **interference term that won't exist at 100%** — and the direction is predictable: **naive A/B systematically over-states** any allocation-affecting change."*

**TWO STRUCTURAL MOVES.** **A ladder of methods**, with rung selection as the engineering judgement · ⭐ *"The framework's job isn't to run experiments — it's to **make the offline metric predict the online outcome**, and that's itself measurable."*

```
THREE INTERFERENCE CHANNELS: (1) COMPETITION (2) BUDGET (3) LEARNING

THE LADDER            WHEN                          BIAS  VAR   COST
 1. offline counterfactual screening                high  low   free
 2. user A/B         ONLY if it doesn't touch       low*  low   cheap
                     allocation or bidding
 3. ⭐ GEO/CLUSTER    any allocation change —        low   HIGH  expensive
                     each market is a self-
                     contained auction
 4. switchback       no geography available         med   med   medium
 5. long-horizon holdback the slow ground truth     lowest high ongoing
```

**DEFENSIBLE CHOICES**
- **Geo randomisation for allocation changes** — interference moves *inside* the unit. *Cost: tens of units ⇒ power collapses.*
- **Budget isolation per arm** — removes channel 2 (budget) structurally. *Cost: real platform work.*
- **Offline↔online correlation as the KPI** — it sets everyone's iteration rate. *Cost: a meta-metric, hard to fund.*

**NUMBER.** Geo tests randomise on **markets (tens)**, not users (millions) — so **CUPED and matched pairs are load-bearing**, not optional. And **state the MDE per method**.
**⭐** *Advertisers adapt over weeks to months — so the honest result is "+X% at two weeks, +Y% at six months."*

---

### THE TIER-3 PATTERN MAP

| Pattern | Appears in |
|---|---|
| **Assume the defence fails; contain instead** | Q19 (injection) · Q2 (grader outside) · G8 (report channel) |
| **The measurement is the hard part** | G8 (substantiating recall) · G3 (can't see the data) · G23 (interference) |
| **Recognise it isn't an online ML problem** | G17 (KV lookup) · G3 (brute force) · G4 (trie) |
| **Delayed labels with a proxy↔truth check** | G15 (retention) · Q26 (joins) · G23 (long horizon) |
| **The thing you can't retrofit** | Q26 (propensity) · Q1 (provenance) · Q2 (traces) · G8 (originals) |
| **Policy decisions disguised as ML ones** | G8 (is a billboard a face) · G25 (taxonomy) · G23 (rung selection) |
