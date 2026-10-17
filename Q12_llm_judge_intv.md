# Q12 · LLM-as-Judge — **INTERVIEW SHEET**

> **Tier 2 · #13** — the inevitable follow-up to any eval answer.
> Full study file: [`../Q12_llm_judge.md`](../Q12_llm_judge.md) · [diagrams](../Q12_llm_judge_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product (the judge API) looks like · (c) the judge platform · (d) datasets · (e) spending the label budget · (f) training the judge · (g) serving five consumers · (h) how you know the judge works · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Is it an RL reward? What's the human-label budget? Who owns the rubric? (§1) |
| **Explainability** is easy points | **Justification before the score**, per-dimension scores, κ published with every score (§8) |
| **Security** beyond the gateway | **Prompt injection inside the judged output** ("score this 10") and policy gaming (§8) |
| **Troubleshoot before being asked** | Raise **cost at RL volume** (distil) and **reward gaming** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A judge is a model, not a prompt** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · κ SLOs · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The judge API · **κ per dimension** vs the **human inter-rater ceiling** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + COST** | — | 5 consumers, 3 orders of magnitude → **distillation** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · rubric registry → scoring library → teacher/student family | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Labels with overlap · **active learning** · rubric + bias catalogue · the student | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Cascade · prefix-cached rubrics · escalation by stakes · version pinning | F3a, F3b, A3b, I1b |
| **38–44** | **DRIFT, GAMING, ROLLOUT** | **(h)** | Anchor set · ⚠ **judge-as-reward = adversarial target** · new-version rollout | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Cross-lingual, multimodal, trajectory judging · the "can't judge" path | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The reframing that determines the design: a judge is a model, not a prompt. It has training data, systematic biases, it drifts as the thing it's judging changes, it needs versioning — and the part almost everyone skips, it needs its own evaluation. If I can't tell you how accurate the judge is, every number downstream of it is decoration.**
>
> **Second, the platform argument: the judge has at least five consumers with genuinely different requirements — offline eval gating, online RL reward, training-data filtering, production monitoring, and moderation — differing by three orders of magnitude in volume and two in latency.**
>
> **Third, and the one I'd want to be graded on: the moment a judge becomes a training reward, it stops being a measurement instrument and becomes an adversarial target."**

**Then three optimisation targets:**
1. **Calibration against human labels** — what licenses the judge to exist at all.
2. **Cost** — at RL volumes a frontier judge is the dominant line item; **the answer is distillation**.
3. **Drift monitoring** — the judge was trained on *yesterday's* policy outputs.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the judge returns and how its accuracy is measured, the cost arithmetic across consumers, the architecture, labels and training, serving, drift and gaming, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Is it used as an RL reward?** *(If yes it's an adversarial target and needs different defences.)*
2. **What's the human-label budget?** *(It's the ceiling on everything — no labels, no calibration.)*
3. **Who owns the rubric?** *(Rubrics are *policy*, and they drift.)*
4. **Which domains, languages and modalities?** *(κ drops hard outside the judge's dominant language.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Consumers | Offline gating · RL reward · data filtering · production monitoring · moderation |
| Domains | Several (coding help, factual QA, safety, creative), each with its own rubric |
| Labels | A fixed human-label budget per release, raters with **overlapping** assignments |
| Models | A strong teacher available; room to distil an ~8B student per domain |
| Languages | Multilingual, with English dominant |
| Rubric owners | Policy / product teams own content; the platform owns versioning and calibration |

**FR:** score on defined dimensions, **pointwise and pairwise**, with a justification · serve all consumers from **one scoring library and one rubric registry** · maintain calibration per dimension continuously · **rubric versioning** · detect drift · provide **uncertainty** so consumers can escalate.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Judge–human agreement** | **Cohen's κ ≥ 0.6 per dimension**, re-measured every release |
| **Human inter-rater agreement** | Measured and **published alongside — it's the ceiling** |
| **Position-bias disagreement** | < 5% of pairwise calls flip when order is swapped |
| **Calibration** | Reliability curve within ±0.1 of the diagonal |
| p99 latency (RL) | < 2 s · throughput ≥ 200 scores/s |
| Cost | $ per 1,000 scores, per consumer |

**Scope / NON-GOALS:**
- ❌ **One judge model for every domain.** *"A general judge is mediocre everywhere — I'd rather have a small set of calibrated specialists."*
- ❌ A single holistic quality score (unreliable and un-actionable).
- ❌ Writing the rubric content (policy owns content; I own versioning and calibration).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What a consumer gets back (b):**

```
score(rubric = "factual-qa@v7", prompt, response_A, response_B?, reference?)
→ {
    dimensions: { correctness: 1 (of 0–2), completeness: 2, clarity: 2 },
    preference: "A > B" (scored in BOTH orders; agreed),
    justification: "...written BEFORE the scores...",
    uncertainty: 0.18  → above a consumer's threshold, it ESCALATES
    judge: student-8b@v12 · rubric: factual-qa@v7
    calibration: κ 0.71 on this rubric (human ceiling 0.78), measured 2026-10-02
  }
```

**The metrics:**

```
⚠ "THE JUDGE AGREES WITH HUMANS 85% OF THE TIME" IS NOT A NUMBER YET.

  1 · AGAINST WHAT CEILING? Measure HUMAN INTER-RATER AGREEMENT first.
      85% against 70%-consistent labels is close to meaningless — the
      judge might be BETTER than any individual rater, or fitting noise,
      and you cannot tell.
  2 · κ, NOT RAW %. Under class imbalance (80% "acceptable"), an
      always-acceptable judge scores 80% having learned nothing.
  3 · PER DIMENSION AND PER DOMAIN, never aggregate — 85% overall can be
      95% on formatting and 60% on factual accuracy, hiding that the
      judge is useless at the dimension you care about.

κ < 0.6  →  ⚠ FIX THE RUBRIC FIRST, THE MODEL SECOND. Don't ship the metric.

FREE QUALITY SIGNALS (no labels needed):
  · position-bias FLIP RATE when you swap order
  · score-distribution COMPRESSION (everything gets a 7 ⇒ stopped discriminating)
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **κ per dimension and domain**, read against the human ceiling | The license to exist |
| Free signals | Both-orders flip rate · score compression · uncertainty on production items | Early warnings, no labels |
| Downstream | Do judged improvements agree with A/B outcomes and verifiable slices? | Is it measuring the right thing? |
| **Guardrails** | Anchor-set stability · canary failures · p99 for RL · $/1k scores | Hard limits |

---

## 3 · CAPACITY + COST *(min 9–12)* — **⭐ distillation isn't an optimisation, it's what makes it possible** `A1b` `I1a` `I1d`

```
FIVE CONSUMERS — volume spans 10³, latency spans 10²:
  offline eval/gating   10⁵/release   hours OK    HIGHEST accuracy
  ONLINE RL REWARD      10⁷/run       <2 s p99    adversarially attacked
  training-data filter  10⁸ batch     days OK     cheapest possible
  production monitoring 10⁶/day       minutes     sampled
  moderation            10⁶/day       <500 ms     high precision

THE ARITHMETIC THAT FORCES DISTILLATION:
  70B prompt-only judge, both-orders  ≈ $0.004 / score
    × 10⁷ scores per RL run           = $40,000   ← comparable to the
                                                     entire RL compute
    × 10⁸ for filtering               = $400,000  ← simply not viable

  distilled ~8B student               ≈ $0.00012 / score  → ~30× cheaper
                                        RL run ≈ $1,200

CASCADE: student scores everything → ~2% escalate to teacher →
         teacher-uncertain → humans
PLUS: PREFIX-CACHE THE RUBRIC — it's byte-identical on every call.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Scoring mode | Pointwise vs **pairwise** vs reference-based | Pairwise for reliability; reference-based where a gold answer exists; pointwise only when there's nothing to compare |
| Coverage | One general judge vs **domain specialists** | A general judge is mediocre everywhere |
| Model | Prompted frontier model vs **teacher + distilled student** | Only the student is affordable at RL and filtering volume |
| Verification | Judge everything vs **checker first, judge the residue** | A checker is ungameable; a judge is tier 3 (FU 8) |

### The system — *draw this in 90 seconds* `A2a`

```
┌─ 1. RUBRIC REGISTRY (versioned, calibration stored IN it) ─────
│ {rubric_id, version, dimensions[], output_contract, κ per dimension,
│  HUMAN_INTER_RATER_κ ← the ceiling, status: approved | provisional | deprecated}
│ OUTPUT: the exact contract every consumer scores against
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. ONE SCORING LIBRARY — shared by ALL five consumers ────────
│ ⚠ a CORRECTNESS requirement: different implementations ⇒ you
│   optimise one thing and measure another
│ judged text FENCED AS DATA (it may contain instructions to the judge)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MODELS = A CALIBRATED FAMILY, NOT ONE JUDGE ───────────────
│ TEACHER (strong, expensive) ──labels──► STUDENT (~8B, distilled)
│ STUDENT scores everything; uncertain items ESCALATE to the teacher (~2%)
│ ⚠ VALIDATE THE STUDENT AGAINST HUMANS, NOT AGAINST THE TEACHER
│ OUTPUT: per-dimension scores + justification + uncertainty
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. CALIBRATION LOOP (per release) ────────────────────────────
│ ~500 stratified items · humans label WITH OVERLAP → κ per dimension
│ vs the ceiling · + ACTIVE LEARNING on the label budget
│ OUTPUT: κ written back into the registry; status set
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. DRIFT + GAMING MONITORS ───────────────────────────────────
│ ANCHOR SET (scores must not move) · canaries · verifiable held-out slice
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **Cost at RL volume:** *"A frontier judge at 10⁷ scores is ~$40,000 a run. I don't degrade the gate; I **distil** a student for volume and keep the teacher for the uncertain ~2%."*
- **Reward gaming:** *"The moment this judge is a reward, the policy will find its blind spots. So reward and gate never share a judge model, and a verifiable slice watches for divergence (§7)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Pairwise over pointwise** | "'Is this a 7 or an 8' has terrible inter-rater reliability; 'is A better than B' has good reliability. Derive scalars with Bradley-Terry" | Needs a comparison; N² pairs if naive |
| **2** | **Distil, and validate against humans** | "95% agreement with the teacher means you faithfully distilled its length bias and self-preference — that's not a result" | Student needs its own human-label budget |
| **3** | **Specialists, not one general judge** | "Domain-specific rubrics and often domain-specific fine-tunes; a general judge is mediocre everywhere" | N judges to calibrate and version |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Human labels with overlap** | Ground truth **and** the inter-rater ceiling | Expensive; raters drift over months |
| **Teacher-labelled corpus** | Volume for distilling the student | Carries the teacher's biases |
| **Anchor sets** (model side and human side) | Frozen items whose scores must not move | Must stay out of training |
| **Canaries** | Responses that **must** score badly | Must be refreshed as gaming evolves |
| **Verifiable slices** | Items with checkable answers | The ungameable cross-check |
| **Production samples** | The current distribution | PII; consent |

### 5.2 Spending the label budget (e) `F1c`

```
THE LOOP (per release):
 1. SAMPLE ~500 items per rubric, STRATIFIED
 2. HUMANS label WITH OVERLAP → INTER-RATER AGREEMENT = THE CEILING
 3. Judge labels the same items
 4. Cohen's κ — PER DIMENSION, PER DOMAIN
 5. κ < 0.6 → FIX THE RUBRIC FIRST, THE MODEL SECOND
 6. RE-MEASURE EVERY RELEASE
 7. PUBLISH agreement alongside every judged score

⚠ SPEND THE LABEL BUDGET WITH ACTIVE LEARNING — but split it:
   ~⅓ RANDOM STRATIFIED  → MEASURE κ honestly
   ~⅔ UNCERTAINTY        → IMPROVE the judge
   (low confidence · BOTH-ORDERS DISAGREEMENT (free!) · ensemble split
    · judge-vs-programmatic disagreement)

⚠ YOU CANNOT MEASURE ON THE SAMPLE YOU OPTIMISED AGAINST — a pure
  uncertainty sample gives a BIASED (pessimistic) κ and could kill a
  judge that is fine.
```

### 5.3 The rubric and the bias catalogue `A2b` `F1a`

**Rubric design rules:**
- **Decompose into 4–6 separable dimensions** — a holistic score is unreliable *and* tells you nothing when it drops.
- **Binary or few-point scales** — fine-grained scales have poor inter-rater reliability for humans *and* models.
- **⚠ Justification BEFORE the score** — score first and the model rationalises afterwards.
- **Anchored examples** — the single cheapest accuracy improvement available.
- **⚠ Separate correctness from helpfulness** — or the judge rewards **fluent wrongness**.
- **Version every rubric** — *a rubric change is a metric change.*

| Bias | Control |
|---|---|
| **Position** | Score **both orders**; the flip rate is a free quality metric |
| **Length / verbosity** | Length-normalise; test at matched quality; verbose-but-empty canary |
| **Self-preference** | **Never let a model family judge itself** in a gating comparison |
| Style over substance | Anchored examples; wrong-but-fluent canary |
| Sycophancy | Neutral phrasing; never reveal which output is the candidate |
| Score compression | Few-point scales; monitor the distribution |

### 5.4 Training the student: the objective (f) `F2c`

Start the ~8B student from an existing instruction-tuned model and fine-tune it (LoRA, or full fine-tuning if the budget allows) on the teacher-labelled corpus plus the human labels:
- **SFT on the teacher's justifications, then its scores** (so the student also writes reasoning before verdicts);
- a **pairwise preference loss** (Bradley-Terry) for comparison rubrics;
- then **calibrate** the uncertainty (temperature or isotonic) on held-out human labels, because escalation and active learning both depend on it.

**⚠ Validate against humans, not the teacher.**

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · RUBRIC     a new rubric version, owned by policy → registry (provisional)
STAGE 2 · LABEL      human labels with overlap → ceiling; teacher labels for volume
STAGE 3 · TRAIN      distil the student → calibrate uncertainty
STAGE 4 · GATE       κ ≥ 0.6 per dimension vs humans · anchor set stable · canaries fail
STAGE 5 · PROMOTE    shadow-score beside the current judge → status approved
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

| Consumer | How it's served |
|---|---|
| **Offline gating** | Teacher (or student + teacher on uncertainty), both orders, human escalation; highest accuracy |
| **RL reward** | Student only, batched, p99 < 2 s; on failure **null-score-and-drop**, never a default score |
| **Data filtering** | Student, offline batch at 10⁸; the cheapest configuration |
| **Production monitoring** | Student on a sample; trends, not single scores |
| **Moderation** | Student with a high-precision threshold; uncertain items to humans |

- **Scale** `F3b`: the cascade; **prefix-caching the rubric**, which is byte-identical on every call; batching for offline consumers.
- **Efficiency** `F3a`: the student can be **quantized** for the RL and filtering tiers, with κ re-measured after quantizing.
- **The escalation objective** `A3b`: escalate when **uncertainty × stakes** crosses a per-consumer threshold, so the expensive teacher and humans are spent where an error costs most.
- **Reliability** `I1b`: every score is stamped with judge and rubric versions, and consumers **pin** a version, so a judge update can't silently change their numbers.

---

## 7 · DRIFT, GAMING, ROLLOUT *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

**Drift:** the **anchor set** — frozen items scored by every judge version. Their scores must not move. If they do, **the judge changed, not the model**. If the anchor is stable but production scores moved, it's genuine **distribution drift** — the policy improved off the calibration set.

**⚠ Gaming — the section to be graded on:**
```
A judge with 85% human agreement on NATURAL outputs can be near-useless
on outputs SPECIFICALLY OPTIMISED AGAINST IT. Six defences:

 1 · NEVER the same judge for reward and gating (different family, or
     a judge held out of the RL loop)
 2 · ENSEMBLE + UNCERTAINTY PENALTY — disagreement is exactly where
     over-optimisation lives
 3 · HOLD OUT A VERIFIABLE SLICE — judge score climbing while verifiable
     score is flat is PROOF, not suspicion
 4 · RETRAIN the judge on FRESH ON-POLICY outputs
 5 · JUDGE-HACKING CANARIES as a hard gate
 6 · ROTATE across an ensemble — no one fixed surface to attack

⭐ "Goodhart isn't a warning here, it's a SCHEDULE."
```

**Rolling out a new judge version** `I2a`: shadow-score it beside the current one on recent traffic and the anchor set; if the anchor moves, **re-baseline** historical numbers explicitly rather than letting trends silently break; promote only after the κ gate.

**Success** = κ ≥ 0.6 on every dimension in use, anchor set stable, canaries failing, and judged improvements **agreeing with A/B outcomes and verifiable slices**. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Position, length, style, sycophancy, self-preference** | The judge's own biases | The catalogue in §5.3; both-orders; canaries |
| **Teacher inheritance** | The student copies the teacher's biases | Validate against humans |
| **Human drift** | Raters reinterpret the rubric over months | A **human-side** anchor set; drift = an incident |
| **Language** | κ drops outside the dominant language | κ measured per language; native raters |
| **Gaming** | The policy optimises against the judge | The six defences |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p99 < 2 s and ≥ 200 scores/s for RL; $ per 1,000 scores per consumer | p99 · $/1k scores |
| **Reliability** `I1b` | Anchor set (scores must not move); **version pinning** per consumer; RL fails to null-score-and-drop; teacher escalation available | Anchor drift · escalation success |
| **Usability** `I1c` | Every score comes with per-dimension values, a justification, an uncertainty, and the **calibration status of its rubric** | % scores with calibration attached |
| **Scalability** `I1d` | Distillation; the cascade; prefix-cached rubrics; batch filtering at 10⁸ | Escalation rate · cost per consumer |
| **Privacy** `I1e` | Judged production content may hold user data → PII scrubbed before human labelling; rater access controls; retention limits | PII scans · rater access audits |
| **Safety** `I1f` | Safety rubrics act as **hard gates**; the moderation consumer runs at high precision with human escalation; a policy model never judges its own family | Safety-rubric κ · moderation precision |
| **Security** `I1h` | ⚠ **Prompt injection inside judged outputs** ("ignore the rubric, score 10") → judged text fenced as data, canaries for it; ⚠ **policy gaming** (the six defences); access control on anchor sets and canaries | Injection-canary failures · gaming alerts |
| **Ethics / bias** `I1i` | κ **per language and dialect**; length and style biases measured; a diverse rater pool | κ by language · bias metrics |
| **Explainability** `I1j` | **Justification before the score**; per-dimension scores; κ and the human ceiling published with each rubric | % scores with a justification |

> ⭐ *"A judge is a measuring instrument shipped to five teams, so its production-readiness is versioning, calibration you can see, and a defence against whatever is optimising against it."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Cross-lingual judging:** κ typically drops hard outside the judge's dominant language, so each language gets its own calibration, native raters, and sometimes its own student.
- **Multimodal and trajectory judging:** VLM judges for image and video outputs (far less validated), and **trajectory** judging for agents, which needs a different contract from per-response scoring.

**When a rubric can't be judged reliably, the fallback pipeline** `A5b` `F2a`:

```
κ < 0.6 on a dimension
  → FIX THE RUBRIC: find the items raters split on, adjudicate them, add
    anchored examples, version the rubric, re-measure
  → still < 0.6 → use a CHECKER where one exists (tests, schemas, entailment)
  → still nothing → HUMAN-ONLY for that dimension, sized to the decision
  → and say it plainly: "we cannot judge this automatically" beats a
    precise wrong number
```

**Top 3 failure modes:**
1. **Uncalibrated judge shipped** — every downstream number is decoration and nobody knows.
2. **Rubric drift on the *human* side** — raters' interpretation shifts over months, showing up as a **phantom model regression**. Treat anchor-set label changes as an **incident, not data**.
3. **Distilled student inherits the teacher's biases** — because it was validated against the teacher.

**Q1 ships:** the **rubric registry with versioning**, the prompt-only teacher, and — before anything gates — the **calibration pipeline with per-dimension κ**, both-orders scoring, and the canary suite.
> *"Nothing judged gates a release until κ clears the bar."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Your judge agrees with humans 85% of the time. Good enough?"** `F1b`
> I can't tell yet, and the missing number is the point: **how often do humans agree with each other?** If inter-rater agreement is 95%, then 85% is a real gap and the judge is missing something systematic. If humans agree only 70% of the time, then 85% agreement with 70%-consistent ground truth is close to meaningless — the judge might be *better* than any individual rater, or it might be fitting the noise, and you cannot distinguish those. So the first thing is overlapping assignments to measure the ceiling. Second, I'd distrust the aggregate: 85% overall can be 95% on formatting and 60% on factual accuracy, which hides that the judge is useless at exactly the dimension that matters — so **κ per dimension and per domain**. Third, raw percentage is the wrong statistic under class imbalance. **Cohen's κ, per dimension, against a measured human ceiling.**

**2. "Pointwise or pairwise?"** `A5a` `F1a`
> **Pairwise, and derive absolute scores from comparisons** rather than asking for them. The reason is empirical and applies to humans too: "is this a 7 or an 8" has terrible inter-rater reliability, while "is A better than B" has good reliability. Absolute scoring is also unstable across items and drifts over time, because the model has no fixed reference for what a 7 means. So I'd collect pairwise outcomes and fit a **Bradley-Terry-style model** to produce scalars when a consumer needs one. Two cases where I'd still use pointwise: when there's genuinely nothing to compare against — production monitoring of a single output — and when a **reference answer** exists, in which case reference-based scoring is stronger than both because it grounds the judgement. And layered on any of them: **rubric decomposition**, because scoring dimensions separately is what turns a metric into a diagnostic.

**3. "A frontier judge at RL volumes is unaffordable. Fix it."** `F3b` `F2c`
> Distillation, and it isn't an optimisation — it's what makes the consumer possible at all. A 70B prompt-only judge with both-orders scoring costs roughly $0.004 per score; at 10⁷ scores per RL run that's **$40,000, comparable to the entire RL compute budget**, and at filtering scale of 10⁸ it's $400,000 and simply not viable. So the **strong teacher labels a large corpus offline** — expensive but one-time — and I train a small **~8B student** on those labels, roughly 30× cheaper, bringing an RL run to about $1,200. Then a **cascade**: the student scores everything, low-confidence items escalate to the teacher at maybe 2% of volume, teacher-uncertain items escalate to humans. Plus **prefix-caching the rubric**, which is byte-identical on every single call. The critical discipline is in the validation: **the student must be validated against human labels, not against the teacher** — 95% agreement with the teacher means you faithfully distilled its length bias and self-preference along with the signal, and that is not a result.

**4. "What happens when the judge becomes an RL reward?"** `I1h` `F1c`
> It stops being a measurement instrument and becomes an **adversarial target** — a change in kind, not degree. Under optimisation pressure the policy finds the judge's blind spots reliably and faster than you expect, and a judge with 85% human agreement on natural outputs can be near-useless on outputs specifically optimised against it, because the policy has learned to live precisely in the 15%. Six defences: **never use the same judge for reward and for gating** — different family, or a judge held out of the RL loop, because otherwise the policy optimises against the thing that approves it and the gate has no independence left; **ensemble with an uncertainty penalty**, since disagreement is exactly where over-optimisation lives; **hold out a verifiable slice**, because judge score climbing while verifiable score stays flat is *proof* rather than suspicion; **retrain on fresh on-policy outputs**, since the judge goes stale precisely as the policy learns to exploit it; **judge-hacking canaries as a hard gate**; and **rotate across an ensemble** so there isn't one fixed surface to attack. The framing: **Goodhart isn't a warning here, it's a schedule.**

**5. "Your quality metric dropped 5 points. Did the model get worse?"** `F2a` `I2a`
> Something did, and the **anchor set** tells me which. I keep a frozen set of items scored by every judge version, and their scores must not change. If they did, **the judge changed, not the model** — a judge-model update, a rubric edit, a prompt-template change, a decoding-parameter drift — and every historical number produced by the old configuration is now incomparable, which is a silent catastrophe if undetected because your quality trend is measuring the instrument rather than the subject. If the anchor set is stable and production scores still moved, it's genuine **distribution drift**: the outputs being judged have moved away from the calibration set, which happens naturally as the policy improves. I'd confirm with embedding distance between the calibration set and current traffic, and with rising judge uncertainty on production items, which is usually the earliest signal. The fix is scheduled re-calibration on fresh human labels **drawn from the current distribution** — on a cadence, not on incident.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you spend a fixed human-label budget?"** `A3b` `I2a`
> Two pools, deliberately, because they answer different questions. A **random stratified slice** — maybe a third — measures κ honestly; a pure uncertainty sample would give a biased estimate of overall agreement, and **you cannot measure on the sample you optimised against**. The other two thirds go to **uncertainty sampling**: items where the judge is unconfident, where both-orders scoring disagreed, or where an ensemble split — that's where the information is, and it makes a fixed budget go several times further. I'd note that **both-orders disagreement is free**, since you already paid for both calls to control position bias, so it's the signal I'd lead with. And there's a prerequisite worth naming: **uncertainty sampling on a miscalibrated judge selects the wrong items**, because an overconfident judge reports 0.95 on things it's wrong about — so calibration isn't just an SLO, it's a precondition for the labelling strategy to work at all. Plus a slice for **overlapping assignments** (the ceiling) and an **anchor set** for human-side drift.

**7. "Raters' interpretation of the rubric shifted over six months. How would you even know?"** `F1c` `I1b`
> With the **anchor set on the human side**, which is the mirror of the model-side one and the thing almost nobody builds. I keep a fixed set of previously-labelled items and periodically send them back through the labelling process; if the labels come back different, **the ground truth moved, not the model** — and the signature in production is a **phantom model regression**, where quality appears to decline and no model change explains it. The stance I'd take is that a change in human labels on the anchor set is **an incident, not data**: everything downstream is calibrated to those labels, so the right response is to investigate why interpretation drifted — usually an under-specified rubric edge case that raters resolved differently over time, or turnover in the rater pool — and then to **adjudicate it explicitly and version the rubric**, rather than silently absorbing the shift. The organisational point: rubrics are policy, they drift like policy, and they need an owner and a review cadence for exactly that reason.

**8. "When should you NOT use a judge at all?"** `A5a` `A5b`
> Whenever something stronger is available, and I'd resist reaching for the judge reflexively. **If a checker exists, use it** — tests, a solver, schema validation, an entailment check against a retrieved span. A judge is tier 3 in the verification ladder, and reaching for it when tier 1 is available is a strict downgrade: you're trading a deterministic, ungameable signal for a probabilistic one. **Where correctness is checkable but quality isn't**, use both — the checker gates correctness, the judge scores the residue that genuinely requires judgement. That's also the principle behind **verifier-augmented judging**: give the judge tools, so "does this code work" is answered by running it rather than by opinion, and **every fact moved from the judge's opinion into a tool's output stops being gameable**. Humans are for calibrating the other two and for domains where κ can't clear the bar — not for scale. And the position I'd hold: if a domain's κ can't reach 0.6 even after fixing the rubric, the right answer is **"we cannot judge this automatically"** rather than shipping a precise wrong number, which is the more dangerous outcome.

**9. "Five consumers, one judge. Isn't that over-engineering?"** `A2a` `A3b`
> The **platform** is justified, but not by DRY — by a correctness requirement, and the distinction matters. If the judge scoring my RL rollouts implements the rubric differently from the judge gating my release, **I'm optimising one thing and measuring another**, and I find out when a release inexplicably fails and nobody can explain why. So **the scoring library and the rubric registry are shared, non-negotiably**. What is *not* shared is the model: volume spans three orders of magnitude and latency spans two, so **one judge model cannot serve all five well** — filtering wants the cheapest possible thing at 10⁸ scale, moderation wants sub-500 ms, gating wants maximum accuracy with both-orders scoring and human escalation. Hence a **calibrated family**: a strong teacher and a distilled student per domain, sharing one rubric contract. The line I'd use: **share the contract and its implementation; vary the model that serves it.**

**10. "Leadership wants the judge to replace human review entirely. Argue it."** `I2b` `A3a`
> I'd argue against the framing and offer the version that works. A judge cannot replace human review because **humans are the calibration**: the judge's κ is measured against human labels, so eliminating human labelling removes the only evidence that the judge is working, and the system immediately begins drifting with no detector. That's not a philosophical objection — it's that you'd be flying an instrument you can no longer check. What I *would* propose is a **volume reallocation rather than an elimination**: the judge handles the confident mass, humans handle the **uncertainty-sampled tail** — which is both where the model is weakest and where human judgement adds most — plus a **random stratified slice** for honest κ measurement, plus the **anchor set** for drift. That typically reduces human volume by an order of magnitude while *improving* the labels you get, because reviewers spend their time on genuinely hard cases rather than on obvious ones. The honest cost to state: the human budget can shrink a lot, but it **cannot go to zero**, and a plan that assumes it can is a plan to lose the ability to know whether anything works.

---

## ⭐ THE L6 LINE

> **"A judge is a model, not a prompt — it needs its own evaluation, and human inter-rater agreement is the ceiling on what its κ can mean. And the moment it becomes an RL reward it stops being a measurement and becomes an adversarial target."**

## 🧭 HOOKS — *steer by what you mention*

- *"Human inter-rater agreement is the ceiling"* → invites the annotation branch (Q27)
- *"Distillation isn't an optimisation, it's what makes it possible"* → invites the cost/economics branch
- *"Validate the student against humans, not the teacher"* → invites the distillation branch (Q25)
- *"Goodhart isn't a warning, it's a schedule"* → invites the RL/gaming branch (**good at L6**)
- *"A rubric change is a metric change"* → invites the versioning branch
- *"If a checker exists, use it — a judge is tier 3"* → invites the verification-ladder branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the calibration loop"* — the 7 steps regenerate most of the answer.
- **Running out of time?** → *"I have five minutes — I'd rather cover gaming and drift than finish the rubric detail. Sound right?"*

**Name and defer:**
- "**The eval harness that consumes this** is a separate design — contamination, caching, promotion gates."
- "**Reference-based judging** is stronger where a gold answer exists, with a canary for equivalent-but-different."
- "**Multi-turn / trajectory judging** needs a different contract than per-response scoring."
- "**Human annotation operations** — recruitment, training, wellbeing — is its own system."
- "**Constitutional / principle-based** judging is an alternative to rubric decomposition with different failure modes."
- "**Cross-lingual judging** — κ typically drops hard outside the judge's dominant language."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a cheap calibrated student, the teacher only where uncertain | §0, §3 | *"Distillation isn't an optimisation, it's what makes it possible."* |
| **A1b** | Latency and cost budgets | §1, §3 | *"$40,000 per RL run with a frontier judge; ~$1,200 with the student."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §6 | The cascade; null-score-and-drop for RL |
| **A2a** | The full stack | §4 diagram | Registry → scoring library → teacher/student → calibration → monitors |
| **A2b** | Model components | §5.3–5.4 | Decomposed rubric; teacher + distilled student |
| **A2c** | Data sources | §5.1 | Labels with overlap, teacher labels, anchors, canaries, verifiable slices |
| **A2d** | Staged pipeline | §5.5 | Rubric → label → train → κ gate → promote |
| **A3a** | Top-down, unprompted | §0 | *"A judge is a model, not a prompt"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 6 | *"Escalate when uncertainty × stakes crosses the consumer's threshold."* |
| **A4a** | Clarify first, assumptions explicit | §1 | RL reward? Label budget? Rubric owner? Languages? |
| **A5a** | Alternatives compared | §4, FU 2, 8 | Pointwise vs pairwise vs reference; general vs specialists; checker first |
| **A5b** | A full alternative pipeline on demand | §9 | Fix the rubric → checker → human-only → "can't judge" |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right method per subproblem | §5.3 | *"Justification before the score; separate correctness from helpfulness."* |
| **F1b** | Metrics and their trade-offs | §2 | *"κ, not raw %, per dimension, against the human ceiling."* |
| **F1c** | Bias sources and mitigations | §5.3, §7 | The bias catalogue; teacher inheritance; human drift |
| **F2a** | Something fails → a fallback | §4, §9, FU 5 | The anchor set tells judge drift from model drift |
| **F2b** | Too costly → tiering | §4 | Distil and cascade, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | SFT on rationales + pairwise loss, calibrated, validated on humans |
| **F3a** | Quantization with a quality trade | §6 | A quantized student, κ re-measured |
| **F3b** | Scale levers | §3, §6 | Cascade, prefix-cached rubrics, batching |
| **F3c** | Extension, with cost and maturity risk | §9 | Cross-lingual, VLM and trajectory judges |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §1, §8 | p99 for RL; $ per 1,000 scores |
| **I1b** | Reliability | §6, §8 | Version pinning; anchor set |
| **I1c** | Usability | §2, §8 | Scores with justification, uncertainty and calibration status |
| **I1d** | Scalability | §3, §8 | Distillation and the cascade |
| **I1e** | Privacy | §8 | PII scrubbed before human labelling |
| **I1f** | Safety | §8 | Safety rubrics as hard gates |
| **I1h** | Security | §8, FU 4 | **Injection inside judged outputs**; the six gaming defences |
| **I1i** | Ethics / bias | §7, §8 | κ per language; rater diversity |
| **I1j** | Explainability | §2, §8 | Justification first; κ published per rubric |
| **I2a** | Offline → online | §7 | Shadow a new judge; re-baseline on anchor moves |
| **I2b** | The success definition | §7, FU 10 | *"κ ≥ 0.6 everywhere, anchors stable, agreeing with A/B and verifiable slices."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) label budget → §5.2 · (f) training the student → §5.4 · (g) serving → §6 · (h) does it work → §2, §7 · (i) extension → §9.
