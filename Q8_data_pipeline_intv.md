# Q8 · Pretraining Data Pipeline — **INTERVIEW SHEET**

> **Tier 2 · #12** — *"design the data pipeline"* is a standard MLOps prompt, and Google weights MLOps heavily.
> Full study file: [`../Q8_data_pipeline.md`](../Q8_data_pipeline.md) · [diagrams](../Q8_data_pipeline_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product (the corpus) looks like · (c) the pipeline · (d) the sources · (e) deciding what goes in · (f) the filtering models and their objectives · (g) running it · (h) how you know the corpus is better · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Pretraining or post-training? Licensing and deletion? Do we have an ablation loop? (§1) |
| **Explainability** is easy points | **Provenance for every token**, data cards, "which documents trained this checkpoint" (§8) |
| **Security** beyond the gateway | **Data poisoning** of the crawl, and dataset supply-chain integrity (§8) |
| **Troubleshoot before being asked** | Raise **dedup bucket skew** and the **silent filter bug** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **The data pipeline is the model** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · reproducibility + provenance SLOs | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | A versioned corpus + its data card · ⚠ **the corpus's metric is a 1B ablation** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + COST** | — | 10 PB in → 60 TB out · **fuzzy dedup is the cost** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · the funnel, stage by stage · skew and silent bugs | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Sources · over-filtering · filter models and objectives · mixture | A2b–d, F1a, F1c, F2c |
| **32–38** | **RUNNING IT** | **(g)** | Content-addressed stages · incremental builds · the mixture objective | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Ablation → mid-scale check → full run · contamination gate · language bias | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Multimodal + post-training data · the regression bisect · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The sentence I'd want you to remember: the data pipeline IS the model. Architecture is nearly commoditised and compute is fixed by budget — the largest lever anyone has on model quality is what tokens go in. So this deserves the best engineers, not the ones nobody else wanted.**
>
> **Second, it's a genuine distributed-systems problem, not an ETL script. Ten petabytes in, sixty terabytes out, and the expensive step — fuzzy deduplication — is a petabyte-scale shuffle that needs its own capacity plan.**
>
> **Third, the mixture is a decision that must be measured, not argued. Which sources, at what weights, in what order — those move benchmark numbers by several points and nobody can predict them from first principles. So the pipeline's most important output isn't the corpus; it's a fast ablation loop at 1B scale that tells us which corpus to build."**

**Then the failure mode to design against:**
> **⚠ "A data bug doesn't crash — it silently produces a slightly worse model. You find out three weeks and three million dollars later. So the pipeline needs the same discipline as production code: versioning, tests, monitoring, and a way to evaluate itself."**

And: **provenance and compliance are hard requirements, and a day-one schema decision** — *"retrofitting provenance onto a finished corpus is effectively impossible."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the training team receives and how a corpus is judged, the funnel's capacity, the architecture, sources and filtering models, how it runs, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Pretraining corpus, or post-training (SFT/preference) data?** *(Same infrastructure, very different quality bar and scale.)*
2. **What are our licensing and deletion obligations?** *(This is a **schema** decision, not a legal footnote.)*
3. **Do we have a fast small-scale ablation capability?** *(If not, that's the first thing I build.)*
4. **What model, languages and modalities is it for?** *(A 100B multilingual text model and an image-text model need different corpora.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Target | Pretraining corpus for a ~100B-class multilingual LLM (the model in Q1) |
| Sources | 10 PB raw web crawl + code + books + papers + licensed + some synthetic |
| Output | ~60 TB of text (~15T tokens at ~4 bytes/token), versioned |
| Languages | 100+, heavily skewed toward a few |
| Obligations | Per-document licences, crawl opt-outs respected, deletion requests honoured |
| Cadence | A full build per quarter; incremental crawl updates weekly |

**FR:** ingest at PB scale · quality filtering · **exact + fuzzy dedup** · decontamination against eval sets · mixture weighting and curriculum · **provenance per document** · PII handling · deletion support · **versioned, reproducible corpora**.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Throughput | 10 PB processed in **weeks, not months** |
| **Reproducibility** | Same (source snapshot + pipeline version) → **identical corpus hash** |
| **Provenance coverage** | **100%** — every token traceable to a source and licence |
| Deletion | A document removable **and provably absent** from the next build |
| **Ablation loop** | 1B-scale train + eval in **< 1 day** |
| Cost | Tracked in **$ per build** and per ablation |

**Scope / NON-GOALS:**
- ❌ **A linear ETL.** It's a versioned, continuously-audited system whose output determines model quality.
- ❌ **Maximum filtering.** Over-filtering is as damaging as under-filtering (§5).
- ❌ Model architecture/training (Q1) · the eval harness itself (Q5, though this pipeline *consumes* it).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the training team receives (b):**

```
CORPUS v42  ·  hash 9f3c…  ·  pipeline v17  ·  sources snapshot 2026-09
  15.1T tokens · mixture: web 52% · code 18% · papers 9% · books 7% · …
  provenance: 100% · licence-cleared: 100% · contamination vs eval suites: clean
  per-language retention table · per-stage token yield vs v41
  1B ABLATION vs v41:  reasoning +1.2 ±0.4 · code +0.6 ±0.5 · multilingual −0.1 ±0.3
  DATA CARD: what's in it, what was removed and why, known gaps
  rebuild: corpus build --sources 2026-09 --pipeline v17   (→ same hash)
```

**The metrics:**

```
⚠ THE CORPUS HAS NO INTRINSIC METRIC.
  "1.2 trillion tokens, 94% passed quality filters" tells you NOTHING
  about whether the model will be better.

THE ONLY REAL METRIC IS DOWNSTREAM:  train a 1B model, evaluate it.
  → so the PIPELINE'S MOST IMPORTANT OUTPUT is a FAST ABLATION LOOP:
    1B-scale train + eval in under a day, so mixture decisions are
    MEASURED rather than ARGUED in a meeting.

PIPELINE HEALTH METRICS (necessary, not sufficient):
  · token yield per stage (a sudden change = a bug, not an improvement)
  · dedup removal rate · contamination hits
  · provenance coverage (must be 100%)
  · per-source retention rate ← ⚠ the over-filtering detector

⚠ AND THE TRAP: filter aggressiveness improves every data-quality stat
  while making the model WORSE. The stats cannot see it. Only the
  ablation can.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **1B ablation scorecard** vs the previous corpus, with noise floors | Is this corpus better for the model? |
| Pipeline health | Token yield per stage and source · dedup rate · per-language retention | Bugs show up here first |
| Compliance | Provenance coverage 100% · deletion verified · contamination clean | Hard gates |
| **Guardrails** | Build time · $ per build · reproducible hash | Operational limits |

---

## 3 · CAPACITY + COST *(min 9–12)* — **⭐ the funnel and where the money goes** `A1b` `I1a` `I1d`

```
THE FUNNEL:
  10 PB raw crawl
   ↓  language ID + heuristic quality       → ~2 PB
   ↓  EXACT dedup (hash)                    → ~1 PB
   ↓  ⚠ FUZZY DEDUP (MinHash + LSH)         → ~300 TB   ← THE COST
   ↓  model-based quality filter            → ~100 TB
   ↓  decontamination vs eval sets          → ~95 TB
   ↓  mixture + curriculum                  →  ~60 TB trained on

⚠ FUZZY DEDUP IS A PETABYTE-SCALE SHUFFLE and dominates the plan:
   · MinHash signature per document (cheap, embarrassingly parallel)
   · BANDED LSH: split signature into b bands of r rows
       P(collision) = 1 − (1 − s^r)^b   ← tune the S-curve for your
       target similarity threshold
   · then group by band-bucket → ⚠ THE SHUFFLE, and the bucket
     distribution is heavily SKEWED (boilerplate = enormous buckets)
   · → cap bucket size / salt hot buckets, or a few reducers run
     for days while the rest finish in minutes

COST: compute is dominated by the dedup shuffle and the model-based filter
  over ~1 PB; storage by the raw crawl. Running the quality model over a
  petabyte is why it's DISTILLED small (§6). Track $ per build and $ per ablation.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Dedup | Exact only vs **exact + fuzzy (MinHash/LSH)** vs embedding-based semantic dedup | Near-duplicates are most of a crawl, so fuzzy; semantic dedup is costlier and can remove useful paraphrase |
| Quality filter | Heuristics only vs **heuristics + a model-based filter** | Model filters catch more, but **over-filter** non-standard text — so they're ablated, not trusted |
| Mixture | Argued in a meeting vs **measured by ablation** | Nobody predicts mixture effects from first principles |
| Provenance | Post-hoc annotation vs **attached at ingest** | Retrofitting trillions of tokens is impossible |

### The system — *draw this in 90 seconds* `A2a`

```
 SOURCES: crawl · code · books · papers · licensed · SYNTHETIC
   ⚠ each with LICENCE + PROVENANCE attached AT INGEST
       │
┌─ 1. EXTRACT — HTML → text, PDF, code ──────────────────────────
│ ⚠ extraction quality caps everything downstream
│ OUTPUT: documents + provenance records
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. LANGUAGE ID + HEURISTIC QUALITY ───────────────────────────
│ length, symbol ratio, perplexity · per-language thresholds
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. EXACT DEDUP (content hash) — cheap ────────────────────────
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. FUZZY DEDUP — MinHash + BANDED LSH ────────────────────────
│ ⚠ PB-scale shuffle · bucket skew · the step that will actually break
│ OUTPUT: one representative per near-duplicate cluster
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. MODEL-BASED QUALITY FILTER ────────────────────────────────
│ ⚠ OVER-FILTERING IS AS BAD AS UNDER-FILTERING: it strips dialects,
│   technical prose, low-resource languages
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. DECONTAMINATION vs EVAL SETS — n-gram match + CANARY STRINGS
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 7. PII DETECTION + REDACTION · safety filters ────────────────
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 8. MIXTURE + CURRICULUM  ← ⚠ SET BY ABLATION ─────────────────
│ OUTPUT: a VERSIONED, CONTENT-ADDRESSED CORPUS → training
└──────┬─────────────────────────────────────────────────────────
       ▲
 ⚠ the 1B ABLATION LOOP feeds back into stages 5 and 8
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **Bucket skew:** *"Boilerplate makes enormous LSH buckets, so a few reducers run for days. I'd characterise the bucket distribution on a subset first, then **cap and salt** hot buckets, accepting slightly lower recall exactly where duplicates matter least."*
- **The silent filter bug:** *"A regex change can drop 15% of a source with no error. **Golden-document tests** and **per-stage, per-source yield alerts** catch it before a training run does (FU 4)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Fuzzy dedup despite the cost** | "Duplicates get memorised and waste compute; near-duplicates are most of a crawl and exact hashing misses all of them" | A PB-scale shuffle with severe bucket skew |
| **2** | **Ablation-driven mixture, not argued** | "Mixture moves benchmarks by several points and nobody predicts it from first principles" | 1B-scale conclusions don't always transfer to 100B |
| **3** | **Provenance at ingest, not retrofitted** | "'Where did this token come from, under what licence, can we remove it' is a day-one schema decision — retrofitting is effectively impossible" | Schema overhead on every record from day one |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The sources (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Web crawl** | Scale and breadth | Boilerplate, spam, PII, **poisoning**, opt-outs |
| **Code** | Reasoning and structure | Licences; secrets in repos |
| **Books and papers** | Long-form, high-quality prose | Licensing |
| **Licensed / curated sets** | Domains the crawl lacks | Contract terms per document |
| **Synthetic** | Scarce formats and languages | Self-consumption; contamination-like gains |
| **Eval suites** | What must be **excluded** (decontamination) | Canary strings needed in every private task |

### 5.2 Deciding what goes in (e) `F1c`

```
⚠ OVER-FILTERING IS AS DAMAGING AS UNDER-FILTERING — and it's the
  counterintuitive half, because every data-quality statistic IMPROVES
  as you filter harder.

  aggressive quality filters strip:
    · dialects and non-standard registers
    · dense technical prose (scores "low quality" on perplexity heuristics)
    · low-resource languages (the classifier was trained on English web)
    · code with unusual formatting
  → a CLEANER corpus that produces a WORSE model.

  → the only detector is the ABLATION, and the panel that hints at it
    is PER-SOURCE RETENTION RATE.
```

So: per-language thresholds and classifiers, per-source retention watched on every build, and every filter change justified by an ablation, not by a cleaner-looking statistic.

### 5.3 The models inside the pipeline, and their objectives (f) `A2b` `F1a`

| Model | Objective | The catch |
|---|---|---|
| **Language ID** | Multi-class log loss | Short and code-mixed text is misrouted |
| **Quality classifier** | Binary log loss: "reference-quality" text (e.g. curated sources) vs random crawl | ⚠ It learns the **style** of the reference set, which is how dialects and technical prose get thrown out |
| **PII detector** | Token-level tagging (names, emails, IDs) | Over-redaction breaks code and maths |
| **Safety filters** | Per-category classifiers | Removing all toxic text leaves a model that can't recognise toxicity; balance by policy |
| Dedup | Not a model: MinHash + LSH | Bucket skew |

### 5.4 Fine-tune rather than train from scratch (f) `F2c`

The quality and PII classifiers start from a pretrained **multilingual** encoder and are adapted with **LoRA per language family**, because one English-trained filter is exactly what over-filters low-resource languages. Then they're **distilled** into small models cheap enough to run over a petabyte (§6).

### 5.5 Mixture, curriculum, and the staged build `A2d`

**Mixture:** source weights + ORDER (curriculum), both measured at 1B and extrapolated with care. Repetition: a few epochs on high-quality data beats one epoch on more low-quality data — up to a point that is empirical, not theoretical.

```
STAGE 1 · SNAPSHOT   freeze source versions; attach licence + provenance
STAGE 2 · PROCESS    stages 1–7, each output content-addressed and cached
STAGE 3 · MIX        candidate mixtures → 1B ABLATIONS (< 1 day each)
STAGE 4 · VALIDATE   the winning mixture at an intermediate scale
STAGE 5 · PUBLISH    corpus hash + data card + per-stage yield report → training
```

---

## 6 · RUNNING IT *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

- **Content-addressed stages:** every stage's output is keyed by **(input hash + stage version)**, so a change to stage 5 re-runs only stages 5–8, and two builds can be diffed. Same idea as `harness_ver` in the eval cache key.
- **Incremental builds** `F3b`: weekly crawl deltas flow through the cached stages; only the dedup step needs the global view.
- **Cheap filters at petabyte scale** `F3a`: the quality model is **distilled and quantized** to a fastText-sized classifier; the bigger model only labels its training data. The trade (a slightly blunter filter) is checked by ablation.
- **The mixture objective** `A3b`: maximise the ablation scorecard **subject to** per-language floors, a repetition ceiling and a synthetic-fraction cap, with the scorecard weighting owned as a reviewable artifact.
- **Reliability** `I1b`: idempotent, checkpointed stages; **golden-document tests** on every pipeline change; **per-stage, per-source yield alerts** on deviation; a reproducible corpus hash.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
1B ABLATION         candidate corpus vs the previous version, same eval suite,
                    published noise floors; per-language results, not one aggregate
INTERMEDIATE SCALE  validate the winning mixture before the full budget, because
                    1B conclusions don't always transfer to 100B
CONTAMINATION GATE  n-gram overlap + canary strings, on EVERY build, BLOCKING
FRESH HELD-OUT      evals the mixture was never tuned against, so the corpus
                    doesn't overfit the ablation suite
FULL RUN            the corpus version is pinned in the checkpoint's metadata
```

**Success** = the new corpus beats the old one on the ablation scorecard beyond the noise floor, **without** losing any language or domain, with provenance at 100% and contamination clean. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **English-centric filters** | Quality classifiers score other languages as low quality | Per-language classifiers and thresholds; per-language retention |
| **Over-filtering** | Dialects and technical prose look "low quality" | Ablation decides; retention panels |
| **Source dominance** | One language or domain swamps the mixture | Mixture floors and upsampling with a repetition ceiling |
| **Self-consumption** | Synthetic data narrows the distribution | Cap the synthetic fraction; prefer verifiable synthetic data |
| **Ablation overfit** | Everyone optimises the same small suite | Fresh held-out evaluations |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | 10 PB in weeks; ablation < 1 day; $ per build and per ablation | Build wall-clock · $/build |
| **Reliability** `I1b` | Idempotent checkpointed stages; golden-document tests; yield alerts on deviation; reproducible hash | Yield anomalies caught · rebuild reproducibility |
| **Usability** `I1c` | A **data card** per corpus; one-command rebuild; diffs between versions for researchers | Time to answer "what changed?" |
| **Scalability** `I1d` | Sharded MinHash/LSH with salted buckets; incremental builds; distilled filters | Reducer skew · cost per PB |
| **Privacy** `I1e` | PII detection and redaction; **deletion provably absent** from the next build; crawl opt-outs respected | Deletion verification · PII recall |
| **Safety** `I1f` | Illegal content (e.g. CSAM) removed by hash matching before anything else; dangerous-capability content handled by policy; toxicity **balanced**, not stripped, so the model can still recognise it | Policy-violating content found in audits |
| **Security** `I1h` | ⚠ **Data poisoning**: attackers can plant content on crawled pages or expired domains (shown practical in 2023 research) → source reputation, snapshot integrity checks, anomaly detection on new sources; access control on licensed data | Poisoning red-team results |
| **Ethics / bias** `I1i` | **Over-filtering dialects and low-resource languages** is an equity failure that no aggregate shows; licensing and consent respected | Per-language retention · per-language ablation |
| **Explainability** `I1j` | **Provenance for every token**; "which documents contributed to this checkpoint" answerable; per-stage yield reports; the data card | Provenance coverage (100%) |

> ⭐ *"A corpus is a product shipped to every model trained on it, so its production-readiness is provenance, reproducibility and the languages it quietly drops."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Multimodal corpora:** image-text pairs need **alignment filtering** (does the caption describe the image?), licensing per image, and **CSAM hash scanning** before anything else. Cost: a vision model over billions of images. Risk: alignment scores favour literal captions and drop useful loose ones.
- **Post-training data** (SFT, preference pairs): the same infrastructure with a far higher quality bar, human review, and tight decontamination.

**When the ablation says a new build is worse, the fallback pipeline** `A5b` `F2a`:

```
corpus v43 loses to v42 beyond the noise floor
  → DIFF the two builds: per-stage and per-source token yield, per-language retention
  → BISECT pipeline versions: content-addressed stages make each step re-runnable
  → REVERT the offending stage (or source snapshot) and rebuild only downstream stages
  → training stays on v42 meanwhile; the regression becomes a golden-document test
```

**Top 3 failure modes:**
1. **⭐ Silent quality regression from a filter bug** — *the defining failure.* A regex change drops 15% of a source, no error, no crash; you discover it three weeks and three million dollars later when the model is slightly worse and nobody knows why.
2. **Dedup shuffle skew** — a few boilerplate buckets are enormous; a handful of reducers run for days. The job doesn't fail, it just never finishes.
3. **Contamination found late** — every benchmark comparison retroactively invalid.

**Q1 ships:** the **1B ablation loop**, **provenance schema**, and **corpus versioning with content addressing**.
> *"The ablation loop first, because without it every mixture decision is an argument rather than a measurement — and those arguments are expensive and unresolvable."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Walk me through fuzzy deduplication at petabyte scale."** `A1b` `F3b`
> **MinHash plus banded LSH**, and the interesting part is the shuffle rather than the algorithm. For each document I compute a MinHash signature — cheap and embarrassingly parallel. Then **banded LSH**: split the signature into `b` bands of `r` rows each, and hash each band; two documents that agree on any band land in the same bucket, giving `P(collision) = 1 − (1 − s^r)^b`, an S-curve I tune so the transition sits at my target similarity threshold. Then **group by band-bucket**, which is the expensive step — a genuine petabyte-scale shuffle. The failure nobody anticipates is **bucket skew**: boilerplate (cookie notices, licence headers, navigation chrome) produces enormous buckets, so a handful of reducers run for days while the rest finish in minutes, and the job doesn't fail — it just never finishes. So I'd **cap bucket size and salt hot buckets**, accepting slightly lower recall on the most-duplicated content, which is exactly the content where missing a few duplicates matters least.

**2. "Why not just filter more aggressively? Cleaner data is better data."** `F1c` `I1i`
> That's the intuition I'd push back on hardest, because **over-filtering is as damaging as under-filtering** and it's much harder to detect. Aggressive quality filters systematically strip dialects and non-standard registers, dense technical prose that scores badly on perplexity heuristics, low-resource languages whose quality classifier was trained mostly on English web text, and code with unusual formatting. The result is a **cleaner corpus that produces a worse model** — and critically, **every data-quality statistic improves** while this happens, so the pipeline's own metrics tell you it's going well. The only instrument that sees it is the **1B ablation**, and the panel that hints at it is **per-source retention rate** — if a filter change drops a language or a domain from 80% retention to 30%, that's a finding regardless of what the aggregate quality score says.

**3. "How do you decide the mixture?"** `A3b` `I2a`
> By **measuring it, not arguing about it** — which is the single most important process decision in this system. Source weights and ordering move benchmark numbers by several points, and nobody can predict them from first principles, so a meeting about mixture is unresolvable and expensive. The answer is a **fast ablation loop at ~1B scale** — train and evaluate in under a day — so that a proposed mixture change becomes a measurement. That's why I'd say the pipeline's most important output isn't the corpus, it's the loop that tells you which corpus to build. The honest caveat: **1B-scale conclusions don't always transfer to 100B**, because some data types (code, multilingual, long-context) show different returns at scale, so I'd validate the final mixture at an intermediate scale before committing the full training budget, and I'd treat the ablation as a *ranking* device rather than an absolute predictor.

**4. "A bug drops 15% of one source and nobody notices for three weeks. Prevent it."** `I1b` `F2a`
> This is **the defining failure of the system** — a data bug doesn't crash, it silently produces a slightly worse model, and you find out three weeks and three million dollars later. The discipline is to treat the pipeline like production code. **Token yield per stage, per source**, monitored with alerting on *deviation* rather than absolute thresholds — a sudden change in retention rate is a bug until proven otherwise, and that includes changes that look like improvements. **Golden-document tests**: a fixed set of documents with known expected outcomes through every stage, run on every pipeline change, so a regex that now drops valid content fails a test rather than a training run. **Content-addressed, versioned corpora** so two builds are comparable and a diff is meaningful. And **pipeline version in the corpus hash**, so you can always answer which code produced which data — the same argument as `harness_ver` in the eval cache key.

**5. "How do you handle a deletion request after the model is trained?"** `I1e` `A2d`
> Honestly, and the honest answer has an uncomfortable part. Removing a document from the **corpus** is straightforward if provenance exists: content-addressed records, a per-document licence and source field, and a re-buildable pipeline mean the next build provably excludes it. Removing it from **trained weights** is not possible — you can't unlearn a document from a checkpoint in any way that's currently reliable — so the realistic answer is that deletion applies to the corpus and takes effect at the **next training run**, with the retention policy and re-training cadence stated up front rather than discovered during an incident. That's exactly why **provenance is a day-one schema decision**: without a per-document origin record you can't even establish whether a document was included, let alone exclude it. And it's why I'd push back on any proposal to treat provenance as a post-hoc annotation project — retrofitting origin onto trillions of tokens isn't a big job, it's an impossible one.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How much synthetic data would you use, and what breaks?"** `F3c` `F1c`
> Some, deliberately, with a hard ceiling and a monitored composition — and I'd frame the risk precisely rather than gesturing at "model collapse." Synthetic data is genuinely valuable where real data is scarce or badly distributed: underrepresented languages, rare reasoning formats, structured tasks with verifiable answers. The failure is **self-consumption** — training on outputs of a model trained on your own distribution narrows it, and the effect compounds across generations because each round's tails get clipped. So: **cap the synthetic fraction** and track it as a first-class corpus statistic; prefer synthetic data with **verifiable ground truth** (code that runs, maths that checks) since verification breaks the self-reference; **never train on your own model's unfiltered output**; and monitor **diversity statistics** of the corpus over time rather than only quality statistics, because collapse shows up as narrowing before it shows up as degradation. And the ablation remains the arbiter — if a synthetic fraction helps at 1B and I can't explain why, I'd be suspicious rather than pleased, because that's the pattern contamination also produces.

**7. "Decontamination — how do you actually know your eval sets didn't leak?"** `I2a` `I1h`
> Three mechanisms with different strengths, and the third is the one people miss. **N-gram overlap** against eval sets is the workhorse — 13-gram matching catches verbatim inclusion — but it misses paraphrase and reformatting, so it gives you a floor, not certainty. **Canary strings** planted in every private eval task are the only check that gives you a *definitive* answer: grep the corpus for the canary and you know. They cost nothing and should have been added before the eval existed, which is the catch. And the **fresh-vs-old delta** — comparing model performance on an old public benchmark against a freshly authored set of equal difficulty — is a *behavioural* contamination estimate that works even when you can't scan the training data, which matters for third-party models and for data you licensed rather than crawled. The organisational point: decontamination has to run **on every build and block**, because contamination doesn't degrade the model, it **invalidates the measurement** — and a benchmark you can no longer trust is worse than one you never had.

**8. "Two teams want different mixtures. How do you resolve it?"** `A3b` `I2b`
> With the ablation, and I'd make that a **standing process rather than a one-time arbitration** — because this disagreement recurs every quarter and re-arguing it is the actual cost. Concretely: both proposals get run at 1B against a shared eval suite with **published noise floors**, and the decision rule is agreed *before* the results, including what "better" means across a multi-dimensional scorecard. That last part is where these disputes actually live — one team's mixture wins on code and loses on multilingual, and without a pre-agreed weighting the argument just moves to which benchmark matters. So the resolution is partly technical (run the experiment) and partly organisational (**own the scorecard weighting as an explicit, reviewable artifact**). I'd also note the failure mode of ablation-driven decisions: if everyone optimises the same small eval suite, the corpus converges on that suite's preferences, so I'd hold **fresh held-out evaluations** that the mixture decisions were not tuned against.

**9. "Your crawl is 60% of one language and 0.1% of another. Do you rebalance?"** `I1i` `F1c`
> Yes, and upweighting low-resource languages is one of the clearest wins available — but the interesting part is that **the filtering stage is usually the bigger culprit than the crawl**. Quality classifiers trained predominantly on English web text systematically score other languages as low quality, so a language can enter the pipeline at 2% and leave at 0.1% because the *filter* removed it, not because the crawl lacked it. That's why **per-source and per-language retention rate** is on the panel: it separates "we didn't collect it" from "we collected and then discarded it," which have completely different fixes. Then: **per-language quality classifiers** rather than one global model, **upsampling with a repetition ceiling** (repetition helps up to a point that's empirical), and ablations that report **per-language** results rather than an aggregate that the majority language dominates. I'd also flag the equity framing plainly — this is a case where a defensible-looking engineering default produces a systematically worse product for a large population, and it won't appear in any aggregate metric.

**10. "You have one quarter and the training run starts in six weeks. What do you actually build?"** `A3a` `A1c`
> The **ablation loop first**, even though it produces no corpus — because without it every mixture and filtering decision for the next six weeks is an argument rather than a measurement, and those arguments consume more calendar time than the loop costs to build. Then, in order: **provenance schema and corpus versioning**, because both are impossible to retrofit and cheap to add at the start; **exact dedup and heuristic filtering**, which are simple, high-yield, and unlikely to be wrong; and **decontamination**, which is blocking and non-negotiable since a contaminated corpus invalidates every measurement I'd use to make the remaining decisions. **Fuzzy dedup** is the one I'd stage carefully — it's the biggest engineering lift and the likeliest to blow the schedule on shuffle skew, so I'd run it on a subset first to characterise the bucket distribution before committing to a full-corpus run. What I would *not* do is spend the six weeks tuning the model-based quality filter, because that's precisely the decision the ablation loop is supposed to make for me, and tuning it by intuition is how over-filtering gets shipped.

---

## ⭐ THE L6 LINE

> **"The data pipeline is the model. The mixture must be measured with a fast ablation loop rather than argued, over-filtering is as damaging as under-filtering, and a data bug doesn't crash — it silently produces a worse model three weeks and three million dollars later."**

## 🧭 HOOKS — *steer by what you mention*

- *"Fuzzy dedup is a petabyte-scale shuffle"* → invites the distributed-systems branch
- *"The mixture must be measured, not argued"* → invites the experimentation/process branch (**good at L6**)
- *"Over-filtering is as damaging as under-filtering"* → invites the quality/equity branch (**good at L6**)
- *"Provenance is a day-one schema decision"* → invites the compliance/legal branch
- *"A data bug doesn't crash"* → invites the reliability/testing branch
- *"The corpus has no intrinsic metric"* → invites the evaluation crossover (Q5)

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me walk the funnel again"* — 10 PB → 60 TB regenerates every stage.
- **Running out of time?** → *"I have five minutes — I'd rather cover the mixture loop and the silent-failure problem than finish the dedup detail. Sound right?"*

**Name and defer:**
- "**Post-training data** — SFT and preference pairs — shares this infrastructure with a much higher quality bar."
- "**Tokenizer coupling**: changing the tokenizer invalidates the prepared corpus, which is an ordering constraint people hit late."
- "**Multimodal corpora** add alignment and licensing problems text doesn't have."
- "**Curriculum ordering** — not just weights but sequence — has real effects and is expensive to ablate."
- "**Long-context data** has to be constructed deliberately; natural documents are mostly short."
- "**Extraction quality** (HTML→text, PDF) caps everything downstream and is unglamorous, high-yield work."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a funnel of cheap filters before expensive ones | §0, §3 | *"Ten petabytes in, sixty terabytes out; fuzzy dedup is the cost."* |
| **A1b** | Throughput and cost budgets | §1, §3 | *"10 PB in weeks; a 1B ablation in under a day."* |
| **A1c** | Degrade without lowering quality | §4, FU 10 | Characterise skew on a subset; cap and salt buckets |
| **A2a** | The full stack | §4 diagram | Extract → language/heuristics → exact → fuzzy → quality → decontam → PII → mixture |
| **A2b** | Model components | §5.3 | Language ID, quality classifier, PII tagger, safety filters |
| **A2c** | Data sources | §5.1 | Crawl, code, books, papers, licensed, synthetic, eval suites |
| **A2d** | Staged pipeline | §5.5 | Snapshot → process → mix → validate → publish |
| **A3a** | Top-down, unprompted | §0 | *"The data pipeline IS the model"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 8 | *"Maximise the ablation scorecard subject to language floors and a repetition ceiling."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Pretrain vs post-train · licensing · ablation capability · target model |
| **A5a** | Alternatives compared | §4 | Exact vs fuzzy vs semantic dedup; heuristic vs model filters; argued vs measured mixture |
| **A5b** | A full alternative pipeline on demand | §9 | Diff → bisect → revert → rebuild downstream |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"The quality classifier learns the reference set's style — that's how dialects get removed."* |
| **F1b** | Metrics and their trade-offs | §2 | *"The corpus has no intrinsic metric; the only real one is a 1B ablation."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Over-filtering, English-centric filters, self-consumption |
| **F2a** | Something underperforms → a fallback | §4, §9, FU 4 | The silent filter bug; the regression bisect |
| **F2b** | Too slow / costly → tiering | §4, §6 | Skew handling; distilled filters |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA per language family on a multilingual encoder |
| **F3a** | Efficiency with a quality trade | §6 | *"A distilled, quantized fastText-sized filter, checked by ablation."* |
| **F3b** | Scale via caching and incremental builds | §6 | Content-addressed stages keyed by input hash + stage version |
| **F3c** | Extension, with cost and maturity risk | §9, FU 6 | Multimodal corpora; synthetic data |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Build time; $ per build |
| **I1b** | Reliability | §6, §8 | Golden documents; yield alerts; reproducible hash |
| **I1c** | Usability | §2, §8 | The data card; one-command rebuild; version diffs |
| **I1d** | Scalability | §3, §8 | Salted LSH; incremental builds |
| **I1e** | Privacy | §8, FU 5 | PII redaction; deletion provably absent |
| **I1f** | Safety | §8 | Illegal content removed first; toxicity balanced, not stripped |
| **I1h** | Security | §8 | **Data poisoning**; supply-chain integrity |
| **I1i** | Ethics / bias | §7, §8, FU 9 | Per-language retention as equity |
| **I1j** | Explainability | §8 | Provenance per token; checkpoint audit |
| **I2a** | Offline → online | §7 | 1B ablation → mid-scale → full run; contamination gate |
| **I2b** | The success definition | §7 | *"Better beyond the noise floor, with no language or domain lost."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) sources → §5.1 · (e) what goes in → §5.2 · (f) filter models → §5.3–5.4 · (g) running it → §6 · (h) success → §2, §7 · (i) extension → §9.
