# G25 · Harmful Content Detection — **INTERVIEW SHEET**

> **Tier 1 · #4** — ⭐ **directly confirmed asked in a Google L6 loop** ("design a content moderation system").
> Full study file: [`../G25_harmful_content_detection.md`](../G25_harmful_content_detection.md) · [diagrams](../G25_harmful_content_detection_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the enforcement stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 WHICH PLATFORM? — *read this first*

**The prompt is platform-agnostic** (*"design a content moderation system"*), and so is the design. **Assume a large user-generated-content (UGC) platform where users upload content that other people view.** At Google, the canonical instance is **YouTube**: videos, comments, Shorts, live streams.

**⚠ Not Chrome.** Chrome is a browser. It doesn't **host** content, so there is nothing to remove or demote. Its safety problem is **Safe Browsing**: classifying URLs and sites as malicious or phishing and warning the user. That is a different system (URL reputation, not content moderation).

**The surface changes the design, so ask which one in minute one:**

| Surface (Google example) | What changes |
|---|---|
| **Video** (YouTube) | Expensive multimodal scoring (frames + audio + transcript); hash-matching on re-uploads; **this sheet's default** |
| **Comments / chat** (YouTube comments, live chat) | Text-only, **orders of magnitude more volume**, real-time for live chat; harassment dominates |
| **Ads** (Google Ads) | **Pre-publication** with an SLA — that's **[G27](../G27_policy_violation_review.md)**, not this sheet |
| **Reviews / UGC listings** (Maps reviews, Play Store reviews) | Spam and fake-review fraud dominate; actor-level signals matter more than content |
| **Private storage** (Drive, Photos) | Not "published" — mainly **CSAM hash-scanning**, with very different privacy and legal constraints |
| **Search results** (SafeSearch) | **Filtering, not removal** — content belongs to third-party sites; the action is demote/hide |

> **Numbers in this sheet are illustrative.** 5M items/day is roughly the scale of **video uploads** to a YouTube-sized platform (~500 hours of video/min ≈ ~4M videos/day). **Comments run billions/day**, which makes the "≥99% must be automatic" arithmetic even more extreme.

> ⭐ **Say it in the room:** *"I'll assume a UGC platform like YouTube, with video as the primary surface. Comments and ads change the volume and the latency model enough that I'd treat them as variants — tell me if you had one of those in mind."*

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Surface, pre/post-publication, **review capacity**, appeals obligation (§1) |
| **Explainability** is easy points | A **statement of reasons** per action, model evidence for reviewers, transparency reports (§8) |
| **Security** beyond the gateway | Adversarial evasion, **mass-reporting attacks**, reviewer-account compromise (§8) |
| **Troubleshoot before being asked** | Raise **video cost** and **novel harm** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **It's a policy problem with a classifier attached** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Creator / viewer / reviewer views · **prevalence by views** · overturn rate | I1c, F1b, I2b |
| **9–12** | **CAPACITY + THRESHOLDS** | — | 5M/day vs 30k reviews → **≥99.4% automatic** · capacity sets thresholds | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Classifiers vs a policy-reading LLM · hash → cascade → band → queue → ladder | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | The taxonomy · κ · appeals as labels · per-policy objective · distillation | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Automated pre-publication, **human post-publication** · fail-closed per policy | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Temporal eval · **shadow → label-only → enforce** · dialect bias | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Live streams · low-resource languages · the novel-harm path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The mistake I want to avoid is treating this as a classification problem. It is a policy problem with a classifier attached — and the ordering matters, because the highest-leverage artifact in this system isn't the model, it's the policy taxonomy: a precise, versioned, enumerated set of categories with definitions, examples, and adjudicated edge cases.**
>
> **Without it, annotators disagree, the model learns their disagreement as noise, and every metric is uninterpretable because nobody can say what was being measured.**
>
> **Which leads to the design position: 'one model, one threshold, harmful or not' fails on contact with reality — harm is not one thing."**

**Then three commitments:**
1. **N policies, each with its own threshold and its own action ladder.** CSAM, terrorism, self-harm, hate, harassment, spam, regulated goods differ by **orders of magnitude in prevalence**, in cost of error, and in what enforcement is even appropriate.
2. **⚠ The borderline band is the product, not an edge case** — and **review capacity sets the thresholds, not the reverse.**
3. **Prevalence — the fraction of *views* that are violating** — is the headline, because harm scales with reach.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what creators, viewers and reviewers see and what I measure, the capacity arithmetic, the architecture, data and models, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked** (after naming the platform, 🎯 above):
0. **Which surface — video, comments, ads, reviews?** *(Changes volume, latency, and whether it's pre- or post-publication.)*
1. **Pre-publication or post-publication?** *(Pre = latency is a product feature and you need a *pending default*; post = prevalence is the metric.)*
2. **What's the review capacity — how many human reviewers?** *(An **input** to the design, not an afterthought.)*
3. **What's the appeals obligation?** *(A legal requirement in several jurisdictions **and** the best label source.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | YouTube-style **video**, published on upload; comments as a variant |
| Volume | ~5M uploads/day |
| Review capacity | ~300 reviewers × ~100 items/day ≈ **30k reviews/day**, split by language and category |
| Policies | ~N policies: CSAM, terrorism, self-harm, hate, harassment, spam, regulated goods… each **versioned** |
| Appeals | Required (e.g. EU DSA), with a published turnaround |
| Hash databases | Industry hash lists for known CSAM (NCMEC) and terrorist content (GIFCT) |

**FR:** per-policy scoring · multimodal (text/image/video) · borderline routing to humans · an action ladder per policy · appeals with a path back to training · policy versioning · a statement of reasons per action.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Scoring latency | Sub-second automated screening at upload; async for expensive models |
| **Prevalence (violating share of views)** | **Per-policy targets** ← the headline |
| **Review queue depth** | **Bounded** — never permanently backlogged |
| **Appeal turnaround** | Days, **and published** |
| Severe-policy recall (CSAM, terrorism) | **Near-1 at detection.** False positives are acceptable for **flagging**, but ⚠ **irreversible actions (account deletion, a law-enforcement report) need human verification first** |
| Availability | Hash matching and severe-policy screening **fail closed** (hold the upload); low-severity policies fail open |

> **Say the two SLO reasons out loud:** *"Queue depth unbounded means the effective policy is 'whatever review got to' — chosen by nobody. And **a right without a timeline is not a right**, so appeal turnaround gets published."*

**Scope / NON-GOALS:**
- ❌ **A single "is this harmful" model with one threshold.** Position, with the reason.
- ❌ Writing the policy (I build the system; policy people own the content) · misinformation/truth adjudication (G28 — *no ground truth for "true"*) · account-level abuse (G26).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What each party sees (b):**

```
CREATOR   uploads → video is LIVE (after seconds of automated screening)
          if actioned: a NOTICE naming the policy + what happened + [Appeal]
          → appeal decided by a different reviewer within the published window

VIEWER    most content: nothing visible
          sensitive-but-allowed: a LABEL (context panel) or an INTERSTITIAL
          (a click-through warning screen: "This video may be disturbing")
          violating: removed, or REACH REDUCED (not recommended)

REVIEWER  a queue ordered by expected harm, content BLURRED by default,
          the model's evidence (which frames, which words) beside it
```

**The metrics:**

```
⚠ THE HEADLINE IS PREVALENCE — VIEW-WEIGHTED, NOT ITEM-WEIGHTED

  a violating post seen 10,000,000 times
  vs one seen 3 times
     → item-level precision/recall treats them EQUALLY.
     → HARM SCALES WITH REACH, so the metric must too.

  PREVALENCE = violating share of VIEWS.  Per policy.
  ⚠ measured on a RANDOM SAMPLE of viewed content, labelled by humans —
    you can't measure what you missed from what you flagged

SECONDARY:
  · precision / recall PER POLICY, against human labels (never aggregated —
    prevalence spans six orders of magnitude across policies)
  · APPEAL OVERTURN RATE ← the over-enforcement signal
  · REVIEW-queue depth and age ← if unbounded, thresholds are wrong
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Prevalence per policy**, view-weighted | Harm that actually reached people |
| Over-enforcement | **Appeal overturn rate per policy** | The cleanest signal we're wrong in our own direction |
| Model | Precision / recall per policy · calibration per policy | Thresholds need trustworthy probabilities |
| **Guardrails** | Queue depth + age bounded · appeal turnaround met · overturn rate by language / dialect · reviewer wellbeing | Hard limits |

---

## 3 · CAPACITY + THRESHOLDS *(min 9–12)* `A1b` `I1a` `I1d`

```
5,000,000 items/day uploaded
    300 reviewers × ~100 items/day  ≈  30,000 reviews/day

  30,000 / 5,000,000  =  0.6%
  →  ⚠ AT LEAST 99.4% MUST BE DECIDED AUTOMATICALLY.

THIS IS ARITHMETIC, NOT A PREFERENCE. It means:
  · the auto/human split is fixed before any modelling choice
  · thresholds are set so the borderline band ≈ 30k items/day
  · the model's job is to make those 30k the MOST VALUABLE 30k

⚠ THE DIRECTION PEOPLE GET BACKWARDS:
  wrong: pick thresholds on a P-R curve → discover the queue is
         10× over capacity → permanent backlog → the effective
         policy is "whatever review got to," chosen by NOBODY
  right: REVIEW CAPACITY IS A HARD INPUT. Thresholds follow.
```

**How a threshold works (two per policy):**

```
         t_low                  t_high
  0 ──────┼──────────────────────┼────── 1      P(violates policy p)
  AUTO-ALLOW │  BORDERLINE BAND   │ AUTO-ACTION
             │  → human review    │
  t_high from PRECISION (no human in the loop) · t_low from the RECALL target

  Σ over policies (band volume) ≤ 30k/day   → a JOINT allocation:
  raise t_low on low-severity policies (spam) first; spend reviews on severe ones
  ⚠ one item can sit in several bands → dedupe, route once with all its policies
  ⚠ capacity isn't one pool: queues are split by LANGUAGE and CATEGORY
```

**Cost:** video is the expensive surface (frames + audio + transcript), so the cascade (§4) keeps the expensive fusion model on the uncertain few percent; track **$ per 1,000 items per stage**.

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| **Trained per-policy classifiers** ✅ for volume | Cheap at 5M items/day; calibrated per policy | A new or changed policy needs **weeks of labels** |
| **Policy-following LLM** (prompted with the policy text and adjudicated examples) | Handles a new policy **in hours**; reads context and combinations | Far more expensive per item; harder to calibrate |
| One model, one threshold | Simple | Wrong across six orders of magnitude of prevalence (FU 1) |

> *"Hybrid: classifiers for volume; the LLM on the borderline band, on new policies before labels exist, and as a labeller that bootstraps the next classifier."*

### The system — *draw this in 90 seconds* `A2a`

```
 UPLOAD
   │
┌─ STAGE 0 · HASH MATCHING (PhotoDNA-style) ─────────────────────
│ a PERCEPTUAL hash survives resizing / re-encoding; match against
│ known CSAM (NCMEC) and terrorist (GIFCT) hash lists
│ ⚠ the cheapest, highest-precision NON-ML component; no human ever
│   needs to see known material again
│ OUTPUT: known-bad → block + report · else continue
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ STAGE 1 · CHEAP CLASSIFIERS ON EVERYTHING ────────────────────
│ per modality: text · image · sampled video frames · audio
│ OUTPUT: per-policy scores; most items end here as clearly fine
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ STAGE 2 · EXPENSIVE MULTIMODAL FUSION, uncertain items only ──
│ ⚠ combination harms live HERE: benign image + benign caption
│   = harassment, and neither unimodal model fires
│ OUTPUT: calibrated P(violates p) for every policy p
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ STAGE 3 · PER-POLICY THRESHOLDS → THREE ZONES ────────────────
│ auto-allow │ BORDERLINE BAND (sized to review capacity) │ auto-act
│ OUTPUT: an action, or a review ticket carrying all its policies
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ STAGE 4 · HUMAN REVIEW QUEUE ─────────────────────────────────
│ ordered by P(violating) × expected REACH × severity
│ language- and category-specialised; blurred by default
│ OUTPUT: a decision + a label
└──────┬─────────────────────────────────────────────────────────
       ▼
 ACTION LADDER (per policy): nothing → reduce reach → label / interstitial
   → remove → account action → report        (least → most severe, irreversible)
       ▼
 APPEALS ─────────────────────────────────────────► TRAINING DATA
 ⚠ the best labels you have: user-initiated, adversarially selected,
   on cases you got wrong
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `F2a` `A1c`

- **Video is too expensive to score fully:** *"So I cascade hard: hash, then cheap per-modality models, and fusion only on the uncertain slice. Most videos need only the cheap pass."*
- **A harm the model has never seen:** *"The model won't catch it, so there's a **fast manual rule path** that ships in hours, plus hash-matching on the specific instances, plus reviewer escalation (FU 5)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **N policies, not one classifier** | "Prevalence spans six orders of magnitude and the cost ordering doesn't match it — one threshold is simultaneously too harsh on mild and too permissive on severe" | N models to train, monitor and version |
| **2** | **Capacity sets thresholds** | "If review handles 30k/day, thresholds produce 30k/day. The alternative is a queue chosen by nobody" | You cannot independently pick a precision-recall point |
| **3** | **Hash matching before any ML** | "Exact-match on known-bad is near-perfect precision and ~free — reaching for a model first is a strict downgrade" | Only covers known content; useless against novel |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **The policy taxonomy** | Definitions, examples, adjudicated edge cases, severity, action ladder | **A policy change is a metric change** — versioned |
| **Reviewer decisions** | The main training labels | Inter-rater agreement (κ) is the noise floor |
| **Appeals and overturns** | The best labels: on cases the system got wrong | Small; weight them up |
| **Random prevalence sample** | Human-labelled views, unbiased | Expensive; the only way to measure prevalence and recall |
| **User reports** | Detection signal | Noisy, and **brigaded** (§8) |
| **Hash databases** | Known-bad instances | Known content only |

### 5.2 Choosing the training set (e) `F1c`

```
1. MEASURE κ     inter-rater agreement per policy FIRST — if humans agree 70%
                 of the time, "85% accuracy" against them is uninterpretable
2. OVERSAMPLE    combination cases (benign + benign = harm): rare naturally
3. WEIGHT UP     appeals and overturned decisions
4. DEDUPE        hash-matched re-uploads (one instance, not 10,000 copies)
5. SPLIT BY TIME train on the past, test on the future: adversaries adapt, and a
                 random split flatters badly
6. VERSION       every label stamped with the policy version it was made under
```

### 5.3 The taxonomy, the labels, and the objective (f) `A2b` `F1a`

**The taxonomy IS the architecture:**
```
policy_id · version · definition · positive examples · negative examples
· adjudicated edge cases · severity · legal status · action ladder
```

**Labels and their bias:**
- **Annotator disagreement is the noise floor.** Measure κ *first*.
- **⚠ Self-harm inverts the asymmetry** — for most policies a false negative is worse; here an aggressive removal can cut someone off from support. Different action ladder: **connect to resources** (a crisis panel), keep recovery content up, remove only instructions and encouragement.
- **Severity ≠ prevalence.** CSAM is extremely rare and maximally severe; spam is extremely common and mildly harmful.

**The model and its objective:**
- **Encoders:** per-modality encoders from a pretrained multimodal model, adapted with **LoRA** rather than trained from scratch.
- **Heads:** one **per policy**, trained with **class-weighted (or focal) log loss**, because prevalence is tiny and a plain loss learns to predict "fine" everywhere.
- **Calibrated per policy**, because thresholds come from capacity and need probabilities that mean something.
- **Fusion stage** trained on **oversampled combination cases**.

### 5.4 Fine-tune and distil (f) `F2c`

The policy-reading LLM labels new or borderline cases; those labels are **distilled** into the cheap stage-1 classifiers. That's how a new policy goes from "LLM only, expensive" to "classifier at volume" in weeks rather than months.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · POLICY     a new / changed policy version → re-adjudicate the golden set
STAGE 2 · LABELS     reviewer decisions + appeals + LLM labels, stamped with the version
STAGE 3 · TRAIN      LoRA encoders + per-policy heads + fusion → calibrate per policy
STAGE 4 · THRESHOLD  set t_low / t_high per policy from precision, recall AND capacity
STAGE 5 · ROLL OUT   shadow → label-only → enforce (§7)
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
⚠ VIDEO: AUTOMATED CHECKS BEFORE PUBLISHING, HUMANS AFTER
  ~5M uploads/day can't wait for a human. So:
  UPLOAD → processing (seconds–minutes): hash matching + high-confidence
           classifiers run HERE → clear violations BLOCKED before anyone sees them
  → PUBLISHED
  → post-publication: ongoing classifiers · user reports · the borderline
    band goes to HUMAN REVIEW, ordered by expected reach
  (true pre-publication human review: ads (G27), monetisation, some
   high-risk categories or untrusted new accounts)

PRE-PUBLICATION SURFACES:
  latency is a PRODUCT FEATURE
  ⚠ THE PENDING DEFAULT: while uncertain, is it live or held?
     decided per RISK TIER × ACTOR TRUST (see G27)

POST-PUBLICATION (most UGC):
  ⚠ HOSTING ≠ AMPLIFYING — reducing distribution is a distinct,
     much cheaper action than removal
```

- **Scale and cost** `F3a` `F3b`: hash first; **int8** cheap classifiers on everything; the fusion model only on the uncertain few percent.
- **The queue objective** `A3b`: `P(violating) × expected reach × severity`, not score alone and not FIFO, because the reviews are the scarce resource.
- **Reliability** `I1b`: ⚠ **fail-closed vs fail-open, per policy** — if the CSAM hash service is down, **hold** uploads; for spam, publish and score later.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline** | **Prevalence per policy** (view-weighted), tracked over time |
| **Over-enforcement** | **Appeal overturn rate per policy** — the cleanest available signal |
| **Capacity health** | Queue depth **and age**; if unbounded, thresholds are wrong |
| **Model** | Precision/recall **per policy**, never aggregated |
| **Adversarial** | **Temporal evaluation** — train on past, test on future. Adversaries adapt; a random split flatters badly |
| **Reviewer** | Agreement rate, throughput, and **wellbeing** — this work causes real harm to the people doing it |

**⚠ How you roll out enforcement** `I2a`: you can't A/B-test leaving harmful content up. So a new model runs in **shadow** (scores, doesn't act), then **label-only** (interstitials, no removals), then enforces. Holdouts are used only on **low-severity** policies such as spam.

**Success** = prevalence down per policy, overturn rate flat or down, the queue bounded, appeals within the published window. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Dialect / language** | Some dialects and reclaimed terms misread as violating → one community over-enforced | Overturn rate and precision **sliced by language and dialect**; native-language reviewers |
| **Annotator disagreement** | The model learns reviewer noise | κ per policy; fix definitions, not reviewers (FU 7) |
| **Adversarial shift** | Evasion moves faster than retraining | Temporal splits; actor-level signals |
| **Reach** | Item-weighted metrics ignore how far harm spread | View-weighted prevalence; reach in the queue order |
| **Selection** | Only flagged items get reviewed | The random prevalence sample |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Sub-second automated screening at upload; the video cascade keeps most items on the cheap pass; cost per 1,000 items per stage | p95 per stage · $/1k items |
| **Reliability** `I1b` | ⚠ **Fail-closed vs fail-open, per policy**: if the CSAM hash service is down, **hold** uploads; for spam, publish and score later. The queue is never permanently backlogged | Queue depth + age · fail-mode events |
| **Usability** `I1c` | Creators get a **clear notice naming the policy** and an appeal with a published turnaround; viewers get interstitials instead of removal for sensitive-but-allowed content | Appeal turnaround · notice clarity audits |
| **Scalability** `I1d` | ≥ 99.4% automatic (§3); hash matching first; cheap per-modality models; fusion only on the uncertain slice | Automation rate · stage-2 share |
| **Privacy** `I1e` | Reviewers see only what they need (blurred by default, access-logged); retention rules for removed content; legal holds; private surfaces only hash-scanned | Access audits · retention compliance |
| **Safety** `I1f` | The point of the system: prevalence per policy; near-1 recall on severe policies, with **human verification before irreversible actions** | Prevalence · severe-policy recall |
| **Security** `I1h` | ⚠ **Adversarial evasion** (FU 8); ⚠ **mass-reporting attacks** that try to get legitimate content removed → detect report brigading; reviewer-account compromise | Report-abuse rate · evasion trend |
| **Ethics / bias** `I1i` | **Over-enforcement on dialects and languages**; self-harm's inverted ladder; **reviewer wellbeing** (FU 10); the same rules for creators of every size | Overturn rate by language / dialect |
| **Explainability** `I1j` | A **statement of reasons** per action (required under the EU Digital Services Act); reviewers see **model evidence** (which frames, which words); an immutable decision log; public transparency reports | % actions with a stated policy · audit coverage |

> ⭐ *"Moderation decisions are taken against people, so explainability is an obligation here, not a feature: the creator gets the policy and an appeal, the reviewer gets the evidence, and the regulator gets the log."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Live streams:** can't wait for async scoring, so sample frames and audio in near-real time with cheaper models, and act faster (cut the stream or reduce its reach).
- **Low-resource languages:** cross-lingual models are less mature, so expect more over- and under-enforcement there, slice the metrics for it, and lean on reviewers who speak the language.

**When the classifiers miss a new kind of harm, the fallback pipeline** `A5b` `F2a`:

```
a new harm pattern (coded language, a new format, a viral challenge)
  → FAST MANUAL RULES: keyword / pattern / hash rules ship in HOURS, accepting
    poor precision while the incident is active
  → HASH every confirmed instance so re-uploads are stopped at stage 0
  → the POLICY-READING LLM scores the borderline traffic with the new definition
  → its labels + reviewer decisions → a new classifier head in weeks (§5.4)
  → monitor PATTERN NOVELTY: the share of flagged content unlike known clusters
```

**Top 3 failure modes:**
1. **Permanent backlog** → the effective policy becomes "whatever review got to," chosen by nobody and changing daily with staffing.
2. **Novel harm** — a new format or coded language the model has never seen. Needs a **fast manual rule path** that ships in hours without a model release.
3. **Over-enforcement on a subgroup** — dialect or language misread as violating. Global precision looks fine; one community is being silenced.

**Q1 ships:** the **policy taxonomy with versioning**, hash matching, per-policy thresholds sized to capacity, and the appeals pipeline.
> *"Taxonomy first, because without it nobody can say what the model is even measuring — and appeals early, because it's simultaneously a legal obligation and the best training data the system will ever produce."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not one model with one threshold?"** `A5a` `F1b`
> Because harm is not one thing, and the differences are structural rather than cosmetic. **Prevalence spans six orders of magnitude** — CSAM is vanishingly rare, spam is everywhere — and critically **the cost ordering doesn't match the prevalence ordering**. A single threshold is therefore simultaneously too aggressive on the mild categories and too permissive on the severe ones. The policies also differ in what enforcement is even *appropriate*: removal is right for terrorism and wrong for self-harm, where an aggressive action can cut someone off from support. So the architecture is N policies, each with its own threshold and its own **action ladder**, and the taxonomy that defines them is the highest-leverage artifact in the system.

**2. "How do you pick the thresholds?"** `A1b` `A3b`
> Backwards from review capacity, which is the direction most teams get wrong. The usual sequence is to optimise a precision-recall trade-off, then discover the borderline band produces ten times what review can handle, and the queue backlogs permanently — at which point **the effective policy is "whatever review got to," chosen by nobody** and drifting with staffing. The correct construction: review capacity is a **hard input**. If review handles 30,000 items a day, thresholds are set so the borderline band produces roughly 30,000 a day, and **the model's job becomes making those 30,000 the most valuable 30,000** — which is why the queue is ordered by `P(violating) × expected reach × severity` rather than by score. Queue depth then becomes a diagnostic: if it grows without bound, the thresholds are wrong.

**3. "What's your headline metric?"** `F1b` `I2b`
> **Prevalence — the fraction of views that are violating, per policy.** Not item-level precision and recall, because harm scales with *reach*: a violating post seen ten million times matters enormously more than one seen three times, and an item-weighted metric treats them identically. So an item-level number can look excellent while the violating content that actually got distributed goes unmeasured. Alongside it I'd track **appeal overturn rate** as the over-enforcement signal, because precision measured against your own labels can't tell you that you're systematically wrong in the same direction your annotators are.

**4. "Multimodal — how do you actually combine the signals?"** `A2b` `F1a`
> Per-modality models first — text, image, video frames, audio, transcript — then a **fusion stage**, and the fusion stage is where the genuinely hard cases live. The reason is that increasingly the harm is in the **combination**: a benign image with a benign caption can together be harassment, and neither unimodal model fires. So fusion isn't an ensemble for accuracy, it's the component that handles a distinct failure class — which means it needs training data **oversampled for combination cases**, since they're rare in a natural sample and the model won't learn them otherwise. Cost-wise I'd cascade hard: cheap per-modality on everything, expensive fusion only on the uncertain subset.

**5. "A new kind of harmful content appears that your model has never seen."** `F2a` `A5b`
> The model won't catch it, and labelled data is weeks away, so the honest answer is about the *response path* rather than the model. First, a **fast manual rule path** that ships in hours without a model release — keyword, hash, or pattern rules, accepting poor precision temporarily because reach matters more than elegance during an active incident. Second, **hash matching** to stop re-uploads of the specific instances immediately. Third, **reviewer escalation** as the detection mechanism — experienced reviewers notice novelty far sooner than a model retrains, which is an argument for keeping the review function staffed as a *detection capability* rather than treating it as pure cost. And I'd monitor **pattern novelty** directly — the share of flagged content that doesn't resemble known clusters — as the leading indicator that something new has arrived.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your precision is 95%. Is that good?"** `F1b` `I1i`
> I can't tell without three things, and asking for them is the answer. **Which policy** — 95% precision on spam is fine and on CSAM-adjacent removal it's catastrophic, because the categories differ by orders of magnitude in the cost of a false positive. **Precision against what** — measured against annotators who agree with each other only 70% of the time, 95% isn't a meaningful number; inter-rater agreement is the ceiling and I'd want it published alongside. And **on which slice** — global precision can be 95% while being 60% on a particular dialect or language, which means one community is being systematically over-enforced while the aggregate looks healthy. That's the failure I'd most want instrumented, because it's invisible to every headline metric and it's the one that becomes a public story.

**7. "Reviewers keep disagreeing with each other. How do you fix it?"** `F1c` `F2a`
> First I'd reject the framing that it's a reviewer problem — **persistent disagreement is almost always a policy problem**, and the fix is the taxonomy rather than the training. Concretely: measure κ per policy, find the categories where it's lowest, and pull the specific items reviewers split on; they'll cluster around a definitional gap the policy never adjudicated. Then the policy owner resolves those into **adjudicated edge cases with worked examples**, and the policy version increments — which invalidates comparisons against prior numbers, and the system needs to know that. What I'd resist is raising agreement by narrowing reviewer discretion to the point where they're pattern-matching rather than judging, because then you've just moved the ambiguity into a rule nobody can apply either. And I'd note the consequence for modelling: **reviewers are the labelling function**, so their disagreement sets the model's achievable ceiling — which makes κ a *model* metric, not an ops metric.

**8. "An adversary is deliberately evading your classifier."** `I1h` `F1c`
> Assume it, design for it, and stop treating detection as the primary defence. Three things follow. **Temporal evaluation** — train on the past, test on the future — because a random split includes tomorrow's evasion patterns in training and reports performance against an adversary that has already moved on; the honest number is always the temporal one. **Features the attacker can't cheaply change** — behavioural and network signals (account age, posting velocity, shared infrastructure) are far more expensive to vary than surface text, which is why actor-level signals beat content-level ones under adversarial pressure. And **raise the cost rather than chase the pattern**: hash matching on known instances, rate limits, and friction on new accounts shift the economics, whereas a classifier update invites one more iteration. The strategic framing I'd give: this is an **economic contest, not an accuracy contest** — I'm trying to make the attack unprofitable, not impossible.

**9. "How do you handle a policy change — say hate speech gets redefined?"** `I2a` `A2d`
> As a **metric change**, which is the thing teams underestimate. The moment the definition moves, every historical number for that policy becomes incomparable, so the policy gets a **version** and every decision, label and metric is stamped with it — otherwise the quality trend after a policy change is measuring the instrument rather than the content. Operationally: re-adjudicate a **golden set** under the new definition, re-measure inter-rater agreement (a redefinition often *lowers* κ before it raises it), then re-label training data for the affected category, which is expensive and is the real cost of a policy change. I'd also expect the **appeal overturn rate to spike** during transition and would not read that as model degradation. And there's a fairness point worth raising unprompted: content actioned under the old definition is still actioned, so I'd want an explicit decision about **retroactivity** rather than letting it be settled by whichever system happens to re-score first.

**10. "Reviewers are seeing traumatic content all day. Is that your problem?"** `I1i` `I1e`
> Yes, and I'd say so directly — it's a design constraint, not an HR footnote, and treating it as someone else's problem is both wrong and self-defeating. It's a design constraint because reviewer wellbeing directly determines **label quality**: fatigued and traumatised reviewers agree less with each other, which raises the noise floor on every model trained from their judgements. So the mitigations are also quality investments: **route by severity** so the most graphic categories are concentrated rather than randomly distributed across everyone; **blur and grayscale by default** with click-to-reveal, which measurably reduces harm at near-zero accuracy cost; **rotate people off severe queues** on a schedule; cap exposure per shift; and use **hash matching aggressively** precisely so that known-bad content is removed without a human ever seeing it again — which is the single largest reduction in human exposure available. I'd also track reviewer agreement and throughput as **wellbeing indicators**, since both degrade before anyone reports a problem.

---

## ⭐ THE L6 LINE

> **"It's a policy problem with a classifier attached. Review capacity sets the thresholds, not the reverse — and the headline metric is prevalence by views, because harm scales with reach."**

## 🧭 HOOKS — *steer by what you mention*

- *"The taxonomy is the highest-leverage artifact"* → invites the data/annotation branch (Q27 territory)
- *"Review capacity is a hard input"* → invites the operations/economics branch
- *"Prevalence is view-weighted"* → invites the metrics branch
- *"Appeals are the best labels we have"* → invites the feedback-loop branch
- *"Reviewers are the labelling function"* → invites the org/leadership branch (**good at L6**)
- *"Hosting is not amplifying"* → invites the action-ladder / policy-design branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the capacity arithmetic"* — 5M vs 30k regenerates the whole design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the adversarial and fairness failures than finish the model detail. Sound right?"*

**Name and defer:**
- "**Pre-publication review** with an SLA is a different operational design — the pending default per risk tier."
- "**Misinformation** is deliberately out of scope: there's no ground truth for 'true', and the answer is provenance rather than a truth classifier."
- "**Coordinated inauthentic behaviour** is an actor-level problem, not a content-level one."
- "**Regional legal variation** means the same content has different obligations per jurisdiction."
- "**Reviewer tooling** — batching, keyboard-first, context pre-fetched — is a large throughput lever people ignore."
- "**On-device pre-screening** shifts the privacy and cost profile for messaging surfaces."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: hash → cheap classifiers → fusion on the uncertain few | §0, §4 | *"Hash matching before any ML; cheap-wide, expensive-narrow."* |
| **A1b** | Capacity and latency budgets | §1, §3 | *"30,000 reviews against 5,000,000 uploads: at least 99.4% must be automatic."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §6 | Cascade video; fail-closed only where the policy demands it |
| **A2a** | The full stack | §4 diagram | Hash → cheap → fusion → thresholds → queue → ladder → appeals |
| **A2b** | Model components | §5.3 | Per-modality LoRA encoders, per-policy heads, a fusion stage |
| **A2c** | Data sources | §5.1 | Taxonomy, reviewer labels, **appeals**, random prevalence sample |
| **A2d** | Staged pipeline | §5.5 | Policy version → labels → train → threshold from capacity → roll out |
| **A3a** | Top-down, unprompted | 🎯, §0 | *"It's a policy problem with a classifier attached"* + the plan |
| **A3b** | A principled objective for the trade-off | §3, §6 | *"Queue by P(violating) × reach × severity; thresholds from capacity."* |
| **A4a** | Clarify first, assumptions explicit | 🎯, §1 | Surface · pre/post · review capacity · appeals |
| **A5a** | Alternatives compared | §4, FU 1 | Classifiers vs a policy-reading LLM; one model vs N policies |
| **A5b** | A full alternative pipeline on demand | §9 | The novel-harm path: rules → hashes → LLM → new head |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Class-weighted or focal loss per policy, calibrated per policy."* |
| **F1b** | Metrics and their trade-offs | §2, §7 | *"Prevalence by views, not item precision; overturn rate as the over-enforcement signal."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | κ, dialect over-enforcement, adversarial shift, selection |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 5 | Novel harm → the manual rule path |
| **F2b** | Too slow / too costly → tiering | §4 | The video cascade, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.3–5.4 | LoRA encoders; LLM labels distilled into classifiers |
| **F3a** | Quantization with a quality trade | §6 | int8 cheap stage, fusion only on the uncertain slice |
| **F3b** | Scale via cheap stages and precomputation | §3, §6 | Hash first; ≥ 99.4% automatic |
| **F3c** | Extension, with cost and maturity risk | §9 | Live streams; low-resource languages |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Per-stage latency; $ per 1,000 items |
| **I1b** | Reliability | §6, §8 | Fail-closed vs fail-open per policy |
| **I1c** | Usability | §2, §8 | Notices naming the policy; appeals; interstitials |
| **I1d** | Scalability | §3, §8 | Automation rate; cascade |
| **I1e** | Privacy | §8 | Blurred, access-logged review; retention rules |
| **I1f** | Safety | §1, §8 | Near-1 severe recall; humans before irreversible actions |
| **I1h** | Security | §8, FU 8 | Evasion; **mass-reporting attacks** |
| **I1i** | Ethics / bias | §7, §8, FU 10 | Dialect over-enforcement; reviewer wellbeing |
| **I1j** | Explainability | §8 | **Statement of reasons**; model evidence; transparency reports |
| **I2a** | Offline → online, safely | §7 | Temporal eval → shadow → label-only → enforce |
| **I2b** | The success definition | §7 | Prevalence down, overturns flat, queue bounded |

**🧭 Probe map:** (a) assumptions → 🎯 + §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
