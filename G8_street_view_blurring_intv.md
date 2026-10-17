# G8 · Street View Face & Plate Blurring — **INTERVIEW SHEET**

> **Tier 3 · #28** — asymmetric cost, cleanly argued. The "how do you know your recall" question is the separator.
> Full study file: [`../G8_street_view_blurring.md`](../G8_street_view_blurring.md) · [diagrams](../G8_street_view_blurring_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the user sees · (c) the pipeline · (d) datasets · (e) building the training and gold sets · (f) the models and their operating points · (g) the batch pipeline, retention and reports · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Which jurisdictions? Do we keep originals? Is there a report channel? Which capture types? (§1) |
| **Explainability** is easy points | A **per-image audit trail** (model version, detections, blur); recall **per stratum with confidence intervals** for a regulator (§8) |
| **Security** beyond the gateway | **The archive of unblurred originals** is the target · **deblurring attacks** on a weak blur · insider access (§8) |
| **Troubleshoot before being asked** | Raise **the cascade ceiling** and **the retention tension** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Recall isn't a metric, it's a legal requirement** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · substantiated recall as the SLO · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | ⚠ **How do you know what you missed?** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | 10¹⁰⁺ images, batch, **one pass, no second chance** · propagation | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Proposal → two verifiers → **geometric propagation** → padded blur | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Stratified gold sets · policy-versioned labels · **blur at p > ~10⁻⁴** | A2b–d, F1a, F1c, F2c |
| **32–38** | **BATCH PIPELINE + RETENTION + REPORTS** | **(g)** | ⚠ **Keeping originals is itself a liability** · reports as miss labels | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Recall per stratum with CIs · per region · per skin tone | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Other PII, indoor imagery · **hold-and-over-blur fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The framing that decides this design: recall is not a metric here, it is a legal requirement. In almost every ML system I trade precision against recall to optimise a business outcome. Here a missed face in a jurisdiction with strong data-protection law is a regulatory violation with per-instance exposure, and an over-blurred fire hydrant is a marginally worse photograph. The cost asymmetry is on the order of ten thousand to one.**
>
> **So I set the operating point far toward over-blurring, and the interesting engineering question becomes *how do I reach 99.9%-plus recall without blurring the whole world* — not where to put the threshold, which is already answered.**
>
> **Second, a structural constraint people miss: blurring is irreversible and applied at ingestion. There's no fixing it at serving time, because the asset users see is the blurred one. That makes this a batch problem with no second chance."**

**Then the two things to raise unprompted:**
1. ⚠ **The retention tension** — *"To re-run detection with a better model later, I must retain the unblurred originals. And a permanent archive of unblurred faces is itself a significant privacy liability. So the retention policy for originals is a real design decision with legal input, not an implementation detail."*
2. ⭐ **The measurement problem** — *"How do I know what recall I actually achieved? I can't human-review sixteen billion images, and by definition I don't know what I missed. **A recall claim I can't substantiate is worthless to the lawyer who has to defend it.**"*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and how recall is substantiated, the scale, the pipeline, the data and models, the batch pipeline with retention and reports, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Which jurisdictions, and what's the per-instance exposure?** *(Sets the operating point — this is a legal input, not a product one.)*
2. **Do we retain unblurred originals, and for how long?** *(⚠ A real decision with legal input — §6.)*
3. **Is there a takedown/report channel?** *(It's both an obligation and the best source of miss labels.)*
4. **Which capture types — car, trekker, indoor business, user-contributed?** *(Each changes the strata and the legal footing.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Imagery | ~10¹⁰–10¹¹ images across many countries, growing with every capture run |
| Capture | Registered, overlapping panoramas **with camera geometry** |
| Objects | Faces and licence plates in v1 |
| Publication | No image is published until it has been processed |
| Reports | A public "report a problem" flow with a legal clock |

**FR:** detect and blur **faces** and **licence plates** at ingestion · handle a user report/takedown · support re-processing when the model improves · **audit trail per image**.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Face recall** | **> 99.9%**, and **substantiated with a confidence interval** |
| **Plate recall** | > 99.9% |
| Precision | **Deliberately low priority** — over-blurring is acceptable |
| Throughput | Batch, **billions of images**, must keep up with capture |
| **Takedown latency** | Hours — it's a legal obligation with a clock |

**Scope / NON-GOALS:**
- ❌ **Optimising an F1 trade-off.** Position: the asymmetry is ~10,000:1; the threshold is already decided.
- ❌ **Fixing it at serving time.** Irreversible, applied at ingestion.
- ❌ Face *recognition* or identity (deliberately, absolutely) · other PII in imagery (house numbers, documents) as v1 · the capture fleet.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **⭐ the L6 move: substantiating a recall claim** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
A STREET-LEVEL PANORAMA
  faces and plates blurred — generously padded, not tight boxes
  [ Report a problem ]
     → "Blur my face / my car / my house"  → select the region
     → "Thanks. We'll review this and blur it, usually within hours.
        Blurring is permanent."                       ← honest about irreversibility

Internally, per image: model versions · detections · blur regions ·
  report history                                      ← the audit trail
```

```
⚠ THE HARD PROBLEM IS NOT THE DETECTOR, IT'S KNOWING WHAT YOU MISSED.
   · 16 billion images — you cannot human-review them
   · BY DEFINITION you don't know what the model missed
   · and "99.9% recall" with no substantiation is worthless to the
     lawyer who has to defend it

THREE MECHANISMS, and you need all three:

 1. ⭐ EXHAUSTIVELY-LABELLED GOLD SET, STRATIFIED BY DIFFICULTY
    humans mark EVERY face in a few thousand images — expensive,
    and it's the only way to measure recall rather than estimate it.
    ⚠ STRATIFY: tiny faces, motion blur, reflections, faces on
      posters/screens, extreme angles, night, crowds.
      A random sample is dominated by easy cases and OVERSTATES recall.
    → report recall WITH A CONFIDENCE INTERVAL, per stratum.

 2. USER REPORTS as a miss-label source — every "you didn't blur me"
    is a labelled false negative from the real distribution, which
    is exactly the data the gold set can't give you.

 3. ⚠ ADVERSARIAL/RED-TEAM SWEEPS — deliberately seek the hard cases
    rather than waiting for them.

⚠ AND THE CASCADE CEILING: if stage 1 misses it, no later stage can
   recover it. THE FIRST STAGE'S RECALL IS THE SYSTEM'S CEILING.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **Gate** | **Recall per stratum, per region, with confidence intervals**, on exhaustive gold sets | The claim we can defend |
| North star | **Substantiated recall ≥ 99.9%** on every stratum that matters legally | The requirement |
| Signals | Report volume and latency · stage-1 recall separately · red-team finds | Live miss signals |
| **Guardrails** | Over-blur rate (product quality) · takedown SLA · unprocessed images published = 0 | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
SCALE: ~10¹⁰–10¹¹ images, growing continuously.
  → BATCH, not real-time. No latency SLO in the usual sense —
    the constraint is KEEPING UP WITH CAPTURE.

⚠ ONE PASS, NO SECOND CHANCE:
  the blurred asset is what users see. There is no "fix it in
  ranking" here — the original is either retained (a liability)
  or gone (and the miss is permanent).

COST: a heavy detector on 10¹⁰ images is a real GPU bill.
  → CASCADE: cheap high-recall proposal on everything, expensive
    verification only on candidates.
    e.g. 10¹⁰ × ~5 ms ≈ 14k GPU-hours for stage 1; stage 2 on the
    candidate fraction with a heavier model is the same order
    → a backlog re-run is a real but ONE-TIME bill
  ⚠ BUT THE CASCADE IS ALSO THE RISK — stage 1 sets the ceiling,
    so stage 1 is tuned for RECALL AT ANY PRECISION COST, and the
    expensive stage exists to claw precision back, never recall.

⭐ THE STRUCTURAL WIN — GEOMETRIC PROPAGATION:
  panoramas are REGISTERED and overlapping. The same face appears in
  several adjacent captures at different scales and angles.
  → detect it in ANY ONE view, project into the others via known
    camera geometry, and blur it EVERYWHERE.
  → this converts N independent chances into ONE detection problem,
    and it's the single largest recall win available.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Structure | One heavy detector vs **a cascade** (cheap proposal → verification) | Cost — with stage 1 tuned so it never sets a low ceiling |
| Models | One detector vs **separate face and plate models** | Different failure modes, strata and gold sets |
| Per image vs scene | Independent per image vs **geometric propagation** across panoramas | N chances collapse to one detection |
| Blur | Tight box, light blur vs **padded region, irreversible blur** | Tight boxes leak; light blur can be partly reversed |
| Originals | Keep forever vs delete at once vs **a bounded, audited window** | Improvement vs liability — a legal decision (§6) |

### The system — *draw this in 90 seconds* `A2a`

```
 CAPTURE → PANORAMA STITCHING (camera geometry RETAINED)
       │
┌─ 1. STAGE 1 — CHEAP, HIGH-RECALL PROPOSAL ─────────────────────
│ TASK: propose every possible face or plate, on EVERY image
│ ⚠ tuned for recall at ANY precision cost — this is the CEILING
│ OUTPUT: many candidate boxes, mostly false positives
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. STAGE 2 — VERIFICATION, two separate models ───────────────
│ FACES — 15 px to full frame · occlusion · posters · reflections
│ PLATES — rigid, small, extreme angles, FORMATS VARY WORLDWIDE
│ ⚠ claws back PRECISION, never recall
│ OUTPUT: a probability per candidate → blur when p > ~10⁻⁴ (§5.3)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. ⭐ GEOMETRIC PROPAGATION ───────────────────────────────────
│ detected in ONE registered view → projected into ALL overlapping views
│ OUTPUT: blur regions for every view of the scene
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. BLUR — irreversible, applied to the asset ─────────────────
│ generous PADDING; strong enough that deblurring can't recover it
│ OUTPUT: the publishable image + its audit record
└──────┬─────────────────────────────────────────────────────────
       ▼
 PUBLISHED ASSET        +   ⚠ ORIGINALS: bounded window, then deleted (§6)
       ▲
 USER REPORT / TAKEDOWN ──► blur + ⭐ becomes a MISS LABEL
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `I1e`

- **The cascade ceiling:** *"Cost pressure will try to tighten stage 1. That silently lowers the system's recall ceiling, so stage-1 recall is measured on its own, per stratum."*
- **The retention tension:** *"Improving later needs originals; keeping originals is a liability. I'd want legal to set a bounded window."*
- **Deblurring:** *"A light Gaussian or pixelation blur can be partly reversed by a model. The blur has to destroy the information, not hide it."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Operating point far toward over-blurring** | "~10,000:1 asymmetry — a missed face is a regulatory violation, an over-blurred hydrant is a marginally worse photo" | Visible over-blurring; some legitimate content degraded |
| **2** | **Geometric propagation across panoramas** | "The same face appears in several registered views — detect it once, blur it everywhere. N chances collapse to one problem" | Requires camera geometry retained through the pipeline |
| **3** | **Separate face and plate models** | "Different failure modes entirely — plates are rigid, high-contrast and format-variable; faces vary 15 px to full-frame with occlusion" | Two models, two gold sets, two operating points |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Exhaustive gold sets**, per region and stratum | The only true recall measurement | Expensive; needs many positives per stratum |
| **Annotated captures** for training | Faces and plates in our own imagery | Annotators see a lot of personal imagery |
| **Plate examples per country format** | Localised plate detection | Formats change; new regions |
| **User reports** | Real-distribution misses | Biased toward salience (§6) |
| **Red-team sweeps** | Deliberately hard cases | Only as good as their imagination |
| **The policy definition** | What counts as a face (posters, reflections, screens) | Versioned; must precede annotation (FU 6) |

### 5.2 Building the training and gold sets (e) `F1c`

- **Stratify everything by difficulty:** tiny (< 20 px), motion blur, reflections, posters and screens, extreme angles, night, crowds, occlusion.
- **Hard-case mining:** stage-1 low-confidence regions, reported misses and red-team finds are up-weighted in training.
- **Policy-versioned labels:** annotators follow an adjudicated definition, measured by inter-annotator agreement; a recall number is only comparable within a policy version.
- **Balance people:** skin tone, age (children's faces are small) and clothing (headscarves, helmets), so recall doesn't depend on who is in the picture.
- **Per-region plates and scenes:** a detector trained on one region's plates and streets degrades elsewhere.

### 5.3 The models and their operating points (f) `A2b` `F1a`

| Model | Task | Objective and operating point |
|---|---|---|
| **Stage-1 proposal** | Every possible face or plate | A light detector with focal loss; threshold set so **recall on the hardest strata** stays above target |
| **Face verifier** | Is this a face (by our policy)? | A heavier detector/classifier; calibrated probability |
| **Plate verifier** | Is this a plate (any country format)? | Same, with per-region data |
| **Propagation** | Same object in other views | **Geometry, not learning** — known camera poses |

```
⭐ THE DECISION RULE — the same shape as fraud (G29):
   blur when  p > C_FP / (C_FP + C_FN)
   with C_FN ≈ 10,000 × C_FP  →  p > 1 / 10,001  ≈  10⁻⁴
   → the verifier only removes candidates it is ALMOST CERTAIN aren't faces
```

### 5.4 Fine-tune an existing model (f) `F2c`

Start from a pretrained detection backbone and fine-tune it on the stratified, policy-labelled data (full fine-tuning for the verifiers; LoRA is enough for per-region plate adaptation). **Distil** the verifier into the stage-1 proposal network, tuned for recall, so the cheap model inherits as much of the expensive one's coverage as possible.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · POLICY      versioned definition of face / plate (posters? reflections?)
STAGE 2 · GOLD        exhaustive, stratified, per region — BEFORE tuning
STAGE 3 · TRAIN       verifiers → distil the proposal net → per-region plate adapters
STAGE 4 · GATE        recall per stratum with CIs ≥ previous version in EVERY stratum
STAGE 5 · PROCESS     new captures + prioritised backlog re-run (FU 9)
STAGE 6 · LEARN       reports + red-team finds → hard cases → next training round
```

---

## 6 · BATCH PIPELINE + RETENTION + REPORTS *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
⚠ THE RETENTION TENSION — raise it unprompted:

  TO IMPROVE:  re-running a better detector over old imagery requires
               the UNBLURRED ORIGINALS.
  BUT:         a permanent archive of unblurred faces is itself a
               major privacy liability and an attractive target.

  → THIS IS A LEGAL AND POLICY DECISION, not an implementation detail.
  → the defensible middle: retain originals for a BOUNDED window with
    strict access control and audit, re-process within it, then
    delete. ⚠ And accept that after the window, a miss is permanent.

USER REPORTS — the channel does double duty:
  1. a LEGAL OBLIGATION with a clock (takedown in hours)
  2. ⭐ THE BEST MISS-LABEL SOURCE YOU HAVE — a real false negative
     from the real distribution, which the gold set cannot provide
  → so the report pipeline feeds TRAINING, and report volume by
    stratum is a live recall signal
  ⚠ but it's BIASED: people report themselves, near their homes, in
    countries with strong privacy norms. It measures SALIENCE as much
    as it measures misses — so it supplements the gold set, never
    replaces it.
```

- **The objective** `A3b`: minimise expected cost with C_FN ≈ 10,000 × C_FP — which *is* the p > ~10⁻⁴ rule — **subject to** keeping up with capture. Precision work only ever happens on strata where it costs no recall.
- **Efficiency** `F3a`: the cascade spends the heavy model only on candidates. The trade is the ceiling risk, policed by measuring stage-1 recall separately.
- **Scale** `F3b`: embarrassingly parallel — shard by capture run and region; the backlog re-run is a one-time bill, prioritised by jurisdiction, traffic and predicted miss rate.
- **Reliability** `I1b`: **nothing is published unprocessed**; if stage 2 fails for an image, publish with **all stage-1 proposals blurred** (fail toward over-blur); every run is idempotent and versioned per image.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

> *"This is where I'd spend disproportionate effort, because a recall claim I can't substantiate is worthless."*

```
⚠ WHY A RANDOM SAMPLE OVERSTATES RECALL:
   most faces in a random image are easy — frontal, well-lit,
   reasonable scale. The detector gets those. A random-sample recall
   estimate is therefore dominated by cases you were always going to
   catch, and it hides the strata where you fail.

→ EXHAUSTIVE LABELLING, STRATIFIED BY DIFFICULTY:
   tiny (<20 px) · motion-blurred · reflections in glass · faces on
   posters, billboards and screens · extreme angle · night/low light ·
   crowds · partially occluded
   → recall REPORTED PER STRATUM, each with a CONFIDENCE INTERVAL
   → and the headline is the WEIGHTED number with the weights stated

⚠ SAMPLE SIZE MATTERS AND PEOPLE SKIP IT: to distinguish 99.9% from
  99.5% recall you need a lot of labelled positives. Say so — it's
  what makes the claim defensible rather than aspirational.

ROLLOUT OF A NEW MODEL: gold sets per region and stratum (no stratum
  may regress beyond its CI) → shadow on recent captures, diffing blur
  regions against the old model → human review of the diffs → publish
  → prioritised backlog re-run while originals still exist
```

**Success** = substantiated recall above 99.9% in every legally relevant stratum and region, takedowns inside the legal clock, and over-blur falling only where it cost no recall. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Skin tone** | Face detectors have historically had lower recall on darker skin | Balanced data; **recall per skin-tone group** with CIs, a parity target |
| **Age** | Children's faces are small and often missed | A "small face" stratum; age-balanced gold sets |
| **Region** | Plate formats, architecture, lighting differ | Per-region gold sets and plate adapters |
| **Report bias** | Reports reflect salience and privacy norms | Gold sets remain the measure; reports supplement |
| **Random-sample bias** | Easy cases inflate recall | Stratified, exhaustive gold sets |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Keeps up with capture; GPU-hours per million images; backlog re-runs costed | Throughput vs capture rate · $ per million images |
| **Reliability** `I1b` | **Nothing published unprocessed**; fail toward over-blur; idempotent, versioned runs | Unprocessed publications = 0 |
| **Usability** `I1c` | Blur that looks clean (padded, smooth), not crude; a simple report flow with a clear promise | Over-blur complaints · report completion |
| **Scalability** `I1d` | Sharded by capture run and region; propagation scales with overlap | Backlog size · days behind capture |
| **Privacy** `I1e` | **Originals kept only for a bounded window**, then deleted; face **recognition** excluded by design and by enforcement | Originals past window = 0 |
| **Safety** `I1f` | Takedowns within the legal clock; sensitive locations (shelters, clinics) blurred more broadly on request or policy | Takedown SLA |
| **Security** `I1h` | ⚠ **The originals archive** is the prize: encryption, least-privilege access, audited reads; **irreversible blur** against deblurring models; annotator access controlled | Archive access audits · deblur red-team |
| **Ethics / bias** `I1i` | **Recall parity across skin tones and ages**; per-region quality; annotator wellbeing | Recall gap by group |
| **Explainability** `I1j` | **Per-image audit trail**; recall reported **per stratum with CIs** and the methodology; policy versions published internally | Audit coverage · regulator queries answered |

> ⭐ *"Here is the number, here is the interval, here is the sampling methodology, and here is where it's weakest. That's the only recall claim worth making."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Other PII in imagery** — house numbers, visible documents, distinctive vehicles: the natural v2, with a fuzzier policy definition.
- **Indoor and business imagery:** different norms and often different legal footing.
- **New capture types** (trekker, aerial): the strata change completely, so new gold sets come first.

**When recall can't be substantiated — the hold-and-over-blur pipeline** `A5b` `F2a`:

```
a new region, a new camera, or a new stratum with no gold set yet
  → HOLD publication for that region / camera until a gold set exists
  → or publish with EVERY stage-1 proposal blurred, no verification
    (maximum over-blur, recall = the proposal net's ceiling)
  → human review of a sample before release
  → build the gold set → measure → then switch to the normal cascade
  "An ugly image I can defend beats a pretty one I can't."
```

**Top 3 failure modes:**
1. **Unsubstantiated recall claim** — the number exists, the methodology doesn't survive scrutiny, and the regulator is the one who finds out.
2. **Cascade ceiling** — stage 1 was tuned on easy cases; a whole stratum (night, reflections) is missed and no later stage can recover it.
3. **Retention trap** — either originals were deleted and misses are permanent, or they were kept and the archive is now the liability.

**Q1 ships:** the **exhaustively-labelled stratified gold set** and the **report pipeline**, before any model tuning.
> *"The measurement before the model — because the operating point is already decided by the asymmetry, and the only open question is whether I can prove I hit it."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Where do you set the threshold?"** `A3b` `F1b`
> It's already decided by the cost asymmetry, which is roughly **ten thousand to one**. A missed face in a jurisdiction with strong data-protection law is a regulatory violation with per-instance exposure; an over-blurred fire hydrant is a marginally worse photograph. So I set the operating point far toward over-blurring and I'd say that the *threshold* question is the uninteresting one. The interesting engineering question is **how to reach 99.9%-plus recall without blurring the whole world** — which is a question about cascade design, geometric propagation, and hard-case coverage, not about a precision-recall curve. That reframe matters because a candidate who spends time on the threshold is answering a question the domain already settled.

**2. "How do you know you achieved 99.9% recall?"** `F1b` `I2b`
> This is the hard problem, and I'd spend disproportionate effort here, because **a recall claim I can't substantiate is worthless to the lawyer who has to defend it**. I can't human-review sixteen billion images, and by definition I don't know what I missed. So: an **exhaustively-labelled gold set** where humans mark *every* face in a few thousand images — expensive, and the only way to measure recall rather than estimate it — **stratified by difficulty**, because a random sample is dominated by easy frontal faces and systematically overstates recall while hiding the strata where I fail. Strata being tiny faces, motion blur, reflections in glass, faces on posters and screens, extreme angles, night, crowds. Recall reported **per stratum with confidence intervals**, and I'd note that distinguishing 99.9% from 99.5% requires a lot of labelled positives — saying that is what makes the claim defensible rather than aspirational. Then **user reports** as real-distribution miss labels, and **adversarial sweeps** that seek hard cases rather than waiting for them.

**3. "Why separate models for faces and plates?"** `A5a` `F1a`
> Because they're different problems with different failure modes, and a single model compromises on both. **Plates** are rigid, high-contrast, geometrically simple, usually small in frame, often at extreme angles — and critically their **formats vary enormously worldwide**, so it's partly a localisation problem and a detector trained on one region's plates degrades elsewhere. **Faces** vary hugely in scale — a fifteen-pixel face is a legitimate detection target — and appear partially occluded, in reflections, on posters, on billboards and on screens. The strata that matter differ, the augmentations that help differ, and the operating points can legitimately differ since the legal exposure isn't identical. So: two models, two gold sets, two sets of stratified recall numbers — and two **policy definitions**, because "is a face on a poster a face for our purposes" is a policy question that has to be adjudicated before annotation.

**4. "Blurring is irreversible. What does that change?"** `A1c` `I1e`
> It makes this a **batch problem with no second chance**, which rules out the usual escape hatches. There's no fixing it at serving time, because the asset users see *is* the blurred one — so I can't retrieve-broadly-and-filter-later, and I can't ship a mediocre model and improve the ranking. It also creates the tension I'd raise unprompted: **to re-run a better detector over old imagery, I have to retain the unblurred originals** — and a permanent archive of unblurred faces is itself a significant privacy liability and an attractive target. So the retention policy is a **real design decision with legal input**, not an implementation detail. The defensible middle is retaining originals for a bounded window under strict access control and audit, re-processing within it, and deleting after — accepting that beyond the window a miss is permanent. That's a trade I'd want made explicitly rather than by default.

**5. "What's the biggest single recall win available?"** `A2a` `F3a`
> **Geometric propagation across overlapping panoramas**, and it's structural rather than a modelling improvement. Street-level captures are registered and overlapping — the same face appears in several adjacent panoramas at different scales, angles and lighting. So if I detect it in **any one view**, I can project it into all the others using the known camera geometry and blur it everywhere. That converts N independent chances to fail into **one detection problem**, and because the views differ in exactly the ways that cause misses — scale, angle, motion blur — the chance that *all* views fail is far lower than any single view's miss rate. It also means effort spent preserving camera geometry through the stitching pipeline pays for itself in recall. I'd flag the related discipline: **blur generously with padding around the region** rather than a tight box, because a slightly-off box with a tight blur leaks the thing you were trying to hide.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Is a face on a billboard a face? A reflection? A statue?"** `F1c` `I1j`
> These are **policy questions, not modelling ones**, and treating them as modelling is how the recall number becomes meaningless. If annotators disagree about whether a billboard face counts, then the gold set's labels encode their disagreement, the measured recall is uninterpretable, and the model learns noise. So the definition has to be **adjudicated and versioned before annotation begins** — the same argument as the policy taxonomy in harmful-content detection. My default positions, with the reasoning: a **billboard or poster face is a real photograph of a real person**, so I'd blur it — the privacy interest doesn't disappear because the medium is printed. A **reflection in glass** is an identifiable person and gets blurred. A **statue or painting** is not a person, so it doesn't need blurring, but given the ~10,000:1 asymmetry I'd rather over-blur than build a classifier to distinguish them. And a **face on a screen** — someone's laptop visible through a window — is both a real person and a second privacy problem entirely. The meta-point: each of these decisions changes the recall denominator, so a recall number is only comparable against a stated policy version.

**7. "Your cascade's first stage is cheap. Isn't that where you'll lose the recall?"** `A1c` `F3a`
> Yes, and that's the structural risk of a cascade in a recall-critical system — **stage one's recall is the ceiling for the whole pipeline**, because nothing downstream can recover what was never proposed. So the design discipline is that **stage one is tuned for recall at essentially any precision cost**: a very low threshold, generous proposals, and an explicit acceptance that most candidates are false positives. The expensive stage exists to **claw back precision, never recall** — and I'd state that as an invariant, because the natural pressure during cost optimisation is to tighten stage one, which silently lowers the ceiling. To keep it honest I'd measure **stage-one recall separately and per stratum**, not just end-to-end, so a regression in the proposal stage is visible before it becomes a published miss. And I'd validate the cascade specifically on the hard strata, because a cheap detector tuned on average imagery is exactly the thing that fails on night, reflections and tiny faces — the strata where the aggregate won't notice.

**8. "A regulator asks for your false-negative rate in their country. What do you say?"** `I1j` `I2b`
> I'd want to be able to answer with a **real number, a methodology and an interval** — and building that capability is why the gold set is a Q1 deliverable. Concretely: recall measured on an exhaustively-labelled, difficulty-stratified sample **drawn from that country's imagery**, because a global number isn't responsive — plate formats, architecture, lighting and population density all differ, and recall genuinely varies by region. So gold sets need to be **per-region**, which is expensive and is the cost of being able to answer this question at all. Alongside it: the **report-and-takedown volume and latency** for that jurisdiction, and the **re-processing history** showing which imagery has been run through which model version. What I would not do is quote a global aggregate or an unqualified "99.9%" — because the first thing a competent regulator asks is how it was measured, and an answer that doesn't survive that question is worse than admitting uncertainty. The defensible position is *"here is the number, here is the interval, here is the sampling methodology, and here is where it's weakest."*

**9. "You improve the detector. Do you re-process eleven years of imagery?"** `F3b` `I1e`
> It depends on whether the originals still exist, which is exactly why the retention decision in §6 is load-bearing rather than an implementation detail. If they do, re-processing is **expensive but mechanical**, and I'd prioritise it rather than doing it uniformly: **by jurisdiction** (strongest legal exposure first), **by traffic** (imagery people actually view carries the real-world risk), and **by predicted miss rate** — running the new detector's *proposal* stage cheaply to find imagery where the old model likely missed something, then re-processing only that. That turns a full re-run into a targeted one. If the originals are gone, the honest answer is that **those misses are permanent**, and the only remaining channel is user reports — which is a real cost of a short retention window and should have been weighed when that window was set. I'd also note the cost-benefit asymmetry: re-processing is a **one-time compute bill against an ongoing legal exposure**, so the calculation usually favours doing it, and the harder constraint is typically the storage and access-control burden of having kept the originals at all.

**10. "The blurring is visibly degrading the product. Product wants it dialled back."** `A3b` `I1c`
> I'd engage with the specifics and hold the operating point, and the way to do that is to attack the *cause* of the over-blurring rather than the threshold. The asymmetry hasn't changed — a missed face is a regulatory violation with per-instance exposure, over-blurring is a worse photograph — so trading recall for aesthetics is not a trade the expected value supports, and I'd want that stated plainly rather than negotiated incrementally. But the complaint is usually legitimate in its particulars, and there are real wins that don't cost recall: **tighter but padded regions** rather than large blur areas, better **blur quality** (a well-implemented blur is much less visually disruptive than a crude one), and — the biggest one — **improving precision on the specific strata driving the complaints**, which is often a small number of systematic false positives like circular signs read as faces or text panels read as plates. Fixing those is a targeted modelling task with no recall cost. What I'd escalate rather than decide alone: if product genuinely wants a lower recall target, that's a **legal and executive decision with named accountability**, not an ML threshold — and I'd want it recorded as such, because the person who lowers it should be the person who owns the exposure.

---

## ⭐ THE L6 LINE

> **"Recall isn't a metric here, it's a legal requirement — the asymmetry is ~10,000:1, so the threshold question is already answered. The interesting problems are reaching 99.9% without blurring the world (geometric propagation across registered panoramas) and *substantiating* the claim — because a recall number I can't defend is worthless."**

## 🧭 HOOKS — *steer by what you mention*

- *"Recall is a legal requirement"* → invites the cost-asymmetry branch
- *"How do I know what I missed?"* → invites the measurement/statistics branch (**good at L6**)
- *"Geometric propagation across registered panoramas"* → invites the classical-CV branch
- *"Retaining originals is itself a liability"* → invites the privacy/policy branch (**good at L6**)
- *"Is a billboard face a face — that's a policy question"* → invites the annotation/taxonomy branch (G25)
- *"Stage one's recall is the ceiling"* → invites the cascade-design branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the asymmetry"* — 10,000:1 regenerates every design decision.
- **Running out of time?** → *"I have five minutes — I'd rather cover how I substantiate the recall claim than finish the detector. Sound right?"*

**Name and defer:**
- "**Other PII in imagery** — house numbers, visible documents, distinctive vehicles — is the natural v2 and has a fuzzier policy."
- "**Face recognition is deliberately out of scope**, and excluding it needs enforcement, not just absence."
- "**Adversarial evasion** is mostly not a concern here — nobody is trying to avoid being blurred."
- "**Interior and business imagery** has different norms and often different legal footing."
- "**Indoor/aerial/satellite** capture changes the strata completely."
- "**The annotation workforce** sees a lot of imagery; that's its own operational and wellbeing question."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the threshold is decided; recall is the work | §0 | *"Recall isn't a metric here, it's a legal requirement."* |
| **A1b** | Throughput and cost budgets | §1, §3 | *"10¹⁰ × ~5 ms ≈ 14k GPU-hours for stage 1 — keep up with capture."* |
| **A1c** | Degrade without lowering quality | §6, FU 4, 7 | *"Stage 2 fails → publish with all stage-1 proposals blurred."* |
| **A2a** | The full stack | §4 diagram, FU 5 | Proposal → two verifiers → geometric propagation → padded blur → reports |
| **A2b** | Model components | §5.3 | Proposal net, face verifier, plate verifier, geometric propagation |
| **A2c** | Data sources | §5.1 | Gold sets, annotated captures, plate formats, reports, red team, the policy |
| **A2d** | Staged pipeline | §5.5 | Policy → gold → train → gate → process → learn |
| **A3a** | Top-down, unprompted | §0 | *"~10,000:1 asymmetry"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.3, §6, FU 1, 10 | *"Blur when p > C_FP / (C_FP + C_FN) ≈ 10⁻⁴."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Jurisdictions · originals · reports · capture types |
| **A5a** | Alternatives compared | §4, FU 3 | One detector vs cascade; one vs two models; tight vs padded blur; retention options |
| **A5b** | A full alternative pipeline on demand | §9 | Hold-and-over-blur |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and operating point | §5.3, FU 3 | *"Focal-loss proposal net tuned on the hardest strata; calibrated verifiers."* |
| **F1b** | Metrics and their trade-offs | §2, FU 2 | *"A random sample is dominated by easy cases and overstates recall."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 6 | Skin tone, age, region, report salience, random-sample bias |
| **F2a** | Something fails → a fallback | §9 | Hold-and-over-blur |
| **F2b** | Cost too high → design for it | §4 | The cascade, and its ceiling raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Fine-tuned verifiers; LoRA plate adapters; distilled proposal net |
| **F3a** | Efficiency with a quality trade | §6, FU 5, 7 | Heavy model only on candidates; stage-1 recall policed |
| **F3b** | Scale levers | §6, FU 9 | Sharded batch; prioritised backlog re-runs |
| **F3c** | Extension, with cost and maturity risk | §9 | Other PII; indoor; new capture types |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Throughput vs capture; $ per million images |
| **I1b** | Reliability | §6, §8 | Nothing published unprocessed; fail toward over-blur |
| **I1c** | Usability | §2, §8, FU 10 | Clean padded blur; a simple report flow |
| **I1d** | Scalability | §3, §8 | Sharded by capture run; propagation |
| **I1e** | Privacy | §6, §8, FU 4, 9 | Bounded retention of originals; no recognition |
| **I1f** | Safety | §8 | Takedowns on the legal clock; sensitive locations |
| **I1h** | Security | §8 | **The originals archive**; irreversible blur vs deblurring |
| **I1i** | Ethics / bias | §7, §8 | Recall parity across skin tones and ages |
| **I1j** | Explainability | §8, FU 6, 8 | Per-image audit; per-stratum recall with CIs and methodology |
| **I2a** | Offline → online | §7 | Gold sets → shadow diff → review → publish → backlog |
| **I2b** | The success definition | §7, FU 2, 8 | *"Substantiated recall above 99.9% in every relevant stratum and region."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) pipeline → §4 · (d) data → §5.1 · (e) sets → §5.2 · (f) models → §5.3–5.4 · (g) batch, retention, reports → §6 · (h) success → §2, §7 · (i) extension → §9.
