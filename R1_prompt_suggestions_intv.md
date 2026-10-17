# R1 · Prompt Suggestions for an LLM Chatbot (Gemini app) — **INTERVIEW SHEET**

> ⭐ **This is a Google recruiter's own sample prompt, and this sheet is built against the actual L6 evaluation form** they shared: 3 parameters, 10 sub-parameters, ~35 pointers. Every section below is tagged with the rubric codes it earns (e.g. `A1a`), and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-where-you-earn-it-and-the-line-that-earns-it) at the end lists every pointer with the exact line that earns it.
>
> **The interviewer's sub-questions** (they will probe these, in roughly this order): (a) assumptions · (b) what the final product looks like · (c) the inference stack · (d) relevant datasets · (e) how to downselect a training set · (f) if PEFT, what's the objective · (g) how to serve · (h) how you'll know it worked · (i) what if it's multimodal.
>
> **Baseline the form expects:** ask clarifying questions, state FR and NFR, ask probing questions, and **need no nudging, hints or guidance**.

---

## 🎯 BEAT THE SAMPLE CANDIDATE — four gaps in the feedback you were shown

The sample feedback is "mostly positive", but read it as a grader would. Four things cost that candidate:

| Gap in the sample | Evidence in the form | What you do instead |
|---|---|---|
| **Clarifying questions came late** | `A4a`: *"**upon triggering**, candidate did cover latency/surfaces"* | Ask them **in minute 1, unprompted**, then state assumptions out loud |
| **Explainability was blank** | `I1j` has no comment at all | Per-chip reason labels, logged provenance, ranker attributions, user controls (§8) |
| **Security stopped at the gateway** | `I1h`: validation, OAuth, rate limits only | Add **prompt injection via context** and **trend poisoning** (§8) |
| **Troubleshooting was reactive** | `F2a/F2b` read as answers to "what if retrieval underperforms / latency is too tight?" | **Raise both problems yourself** before the interviewer does (§4, §6) |

> ⭐ **The meta-rule of this form:** an interviewer can only tick a pointer for something you *said*. If you know it and didn't say it, it doesn't exist.

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Sub-question | What you say here | Rubric codes earned |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **Retrieval + ranking in the critical path; generation is the fallback** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | 5 clarifying Qs **unprompted** · assumptions · FR/NFR · scope to **home + post-turn** | A4a, A1b, A3a |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The two surfaces · north star = **downstream completion**, not CTR | I1c, I2b, F1b |
| **9–12** | **CAPACITY + LATENCY BUDGETS** | — | Per-surface P95 · per-stage budget · cost/request | A1b, I1a |
| **12–22** | **ARCHITECTURE** | **(c)** | Full inference stack · **retrieval vs generation trade** · tiered confidence-gated serving | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Datasets · downselection · two-tower + ranker · **PEFT objective** · staged pipeline | A2b, A2c, A2d, F1a, F2c |
| **32–38** | **SERVING** | **(g)** | Precompute home · prefetch post-turn · quantize · diversity rerank · scale | F3a, F3b, A3b, I1d |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Offline → shadow → A/B · **correct for shown-only labels** · bias sources | I2a, I2b, F1b, F1c |
| **44–48** | **PRODUCTION READINESS SWEEP** | — | Privacy · safety · security · ethics · **explainability** · reliability | I1b, I1e–I1j |
| **48–52** | **MULTIMODAL + FAILURES** | **(i)** | Image context · cost · embedding-maturity risk · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES / YOUR QUESTIONS** | — | Generation pipeline on demand · harder follow-ups · ask about their team | A5b |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Before I design anything, I want to fix one decision, because it determines the rest: what sits in the critical path. A suggestion is something Google is *recommending* the user type, so it carries a higher safety bar than an answer, and it has to appear fast, on surfaces where nobody waits for it.**
>
> **So my default is retrieval plus ranking over a controlled, policy-reviewed inventory of prompts in the critical path, and LLM generation as a gated fallback for contexts the inventory can't cover. Retrieval wins on latency, cost, safety control and evaluability; generation wins on coverage. I'll compare them properly in the architecture section.**
>
> **And the metric isn't click-through. A suggestion that gets tapped and leads nowhere is worse than no suggestion, so the north star is whether the conversation it starts actually succeeds."**

**Then say your plan out loud (this is `A3a`, organising top-down unprompted):**
> *"I'll go: clarifying questions and assumptions, the product, metrics, latency budgets, the inference stack, data and training, serving, evaluation, then a production-readiness sweep and multimodal. Stop me anywhere you'd like more depth."*

---

## 1 · REQUIREMENTS *(min 1–6)* — sub-question **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked (the sample candidate lost credit here):**
1. **Which surfaces?** Home / new-chat screen? Follow-ups after an answer? Typeahead as you type? *(Typeahead is ~50 ms and a different system, like [G4](G4_query_autocomplete_intv.md).)*
2. **What's the goal: engagement, feature discovery, or task success?** *(It decides the north-star metric.)*
3. **Latency: do suggestions block anything?** *(If they render asynchronously, the budget is looser.)*
4. **Personalisation allowed?** Signed-in only? Does it respect the user's activity setting?
5. **Scale, locales, modalities?** *(Multimodal: I'll design text first and extend at the end.)*

**Then state assumptions explicitly (sub-question a):**

| Assumption | Value |
|---|---|
| Product | Gemini app, signed-in users, mobile + web |
| **Surfaces in scope** | **Home (zero-state starter chips)** and **post-turn follow-up chips**. Typeahead out of scope |
| Scale | ~100M daily active users, ~3 app opens and ~10 turns per user per day |
| Output | **3–5 suggestions** per surface, each with a **category label** and reason |
| Personalisation | **Moderate**: locale, recent topics, coarse interests, only with activity setting on |
| Languages | Top ~40 locales; inventory quality varies by locale (a risk to monitor) |
| Existing assets | A pretrained text encoder, an internal LLM family, a safety classifier stack |

**FR:** home starter suggestions · post-turn follow-ups grounded in the conversation · multi-source candidates · ranking + diversity · in-path safety and privacy checks · category metadata + reason · user controls (dismiss, "not interested", turn off) · generation fallback.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Home latency** | **P95 300–500 ms** (renders async with skeleton chips) |
| **Post-turn latency** | **P95 200–400 ms**, measured from the **end** of the answer |
| Availability | 99.9%; on failure **show nothing**, never block the chat |
| **Unsafe suggestion rate** | ~0 on audited samples, a hard gate |
| **Privacy leakage rate** | 0 suggestions containing another user's PII or rare verbatim text |
| Cost | Track **$ per 1,000 requests**; generation fallback on ≤ ~10% of requests |

**Scope / NON-GOALS:**
- ❌ **Optimising raw CTR.** It's gameable by clickbait chips ("Tell me a joke"). Position, with the reason.
- ❌ Typeahead/autocomplete · the chat model's answer quality · ads in suggestions.
- ❌ Multimodal in the core design; **extended at the end** (§9).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — sub-questions **(b)**, **(h)** `I1c` `I2b` `F1b`

**What the final product looks like (b):**

```
HOME / NEW CHAT (zero state)                POST-TURN (after an answer)
┌────────────────────────────────┐          …model's answer about a Kyoto trip…
│  Hello, Priya                  │
│  [✈ Plan a 3-day Lisbon trip]  │          [Make it a day-by-day table]
│  [🎨 Create an image of …]     │          [Compare with Osaka]
│  [📄 Summarise a PDF]          │          [Find vegetarian restaurants]
│  [💡 New: try Deep Research]   │          ↑ grounded in THIS conversation
└────────────────────────────────┘
  discovery: what can Gemini do?            continuation: what next, here?
  3–5 chips, diverse categories             3–5 chips, specific to the context
  each with a category + reason             each with a category + reason
```

**The two surfaces are different problems:** home is **discovery** (low context, cacheable, teaches capabilities); post-turn is **continuation** (rich context, must be specific, computed per turn).

**Metrics (defined now, so the design can be judged against them):**

| Layer | Metric | Why |
|---|---|---|
| **North star** | **Downstream completion**: the suggested conversation succeeds (≥ 2 more turns, thumbs-up, copy/share, no regenerate or immediate abandon) | A tap that leads nowhere is worse than no chip |
| Engagement | **Acceptance rate** (taps per impression), dismiss rate | The proximate signal |
| Long-term | **Retention** (D7/D28: the share of users active again 7 / 28 days later) vs a long-term holdout | Suggestions should build habit, not novelty clicks |
| Offline, retrieval | **Recall@k** of the accepted prompt in the candidate set | Can the ranker even see the right answer? |
| Offline, ranking | **NDCG@5, MRR** | Is the best chip near the top? |
| Offline, slate | **Diversity** (intra-list distance, category entropy) · **coverage** (share of inventory/categories ever shown) | Avoids five near-identical chips and a shrinking catalogue |
| Model health | **Calibration** of P(accept) (ECE) | The confidence gate in §4 thresholds on it |
| **Guardrails** | P95 latency per surface · cost/request · **unsafe suggestion rate** · **privacy leakage rate** · reputation incidents | Hard limits, not trade-offs |

> **Say:** *"CTR is the available metric; downstream completion is the meaningful one. A 'Tell me a joke' chip wins CTR and loses everything else."*

> **D7 / D28, defined:** take the users who were active on day 0 (or first saw the feature that day). **D7** = the fraction active again **7 days later**, **D28** = **28 days later** ("active" = sent at least one prompt). E.g. 1M users on day 0 → 420k back on day 7 (**D7 = 42%**) → 300k on day 28 (**D28 = 30%**).
> Compare users who get suggestions with a **long-term holdout** who never do: 31.5% vs 30% at D28 = **+1.5 points that lasted**. ⚠ If D7 rises but D28 doesn't, it was **novelty**: new chips get tapped out of curiosity, then ignored. A gap that **survives to D28** means the suggestions built a **habit**.

---

## 3 · CAPACITY + LATENCY BUDGETS *(min 9–12)* `A1b` `I1a`

```
TRAFFIC
  home:      100M DAU × ~3 opens  = 3×10⁸/day  ≈ 3.5k QPS avg, ~10k peak
  post-turn: 100M DAU × ~10 turns = 10⁹/day    ≈ 12k QPS avg, ~35k peak
  generation fallback at ≤10% of post-turn     ≈ 3.5k QPS peak of LLM calls
    → the COST driver, which is why it's gated

HOME: precompute per user (or per segment) in a nightly batch
  100M users × 5 chips → a KV cache; online = lookup + light re-rank (~20–50 ms)
  → the 300–500 ms budget is easy; spend it on freshness, not compute

POST-TURN: per-stage budget for P95 ≤ 300 ms (target ~150 ms, rest is tail headroom)
  request gate (validate, OAuth, rate limit)        10 ms
  context filter (PII strip, sensitivity check)     20 ms
  context tower encode (small encoder, int8)        15 ms
  candidate generation, all sources in PARALLEL     15 ms   (ANN top-200)
  feature fetch                                     15 ms
  ranker (GBDT on ~200 candidates)                  10 ms
  diversity rerank (MMR)                             2 ms
  safety + privacy check on final ~10               20 ms
  network / serialisation                           40 ms
                                                  ≈ 150 ms

⭐ THE TRICK FOR POST-TURN: start computing while the ANSWER IS STILL STREAMING.
  The answer takes seconds; the user's prompt + the answer so far are enough
  context. The 200–400 ms budget is measured from the answer's END, so even a
  small-LLM fallback (~300–500 ms) can be hidden behind the stream.

COST: retrieval path ≈ fractions of a cent per 1,000 requests (CPU + ANN);
      generation ≈ 10–100× more per request → track $/1k requests per tier.
```

---

## 4 · ARCHITECTURE *(min 12–22)* — sub-question **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: retrieval vs generation in the critical path `A5a`

| Dimension | Retrieval + ranking (curated inventory) | LLM generation |
|---|---|---|
| **Control / safety** | Every prompt **pre-reviewed** and safety-tagged | Open-ended: needs an in-path filter on every output |
| **Cost** | CPU + ANN, ~free per request | An LLM call per request |
| **Latency** | Tens of ms | Hundreds of ms+ |
| **Coverage** | Limited to the inventory; **weak on specific contexts** | Any context, highly specific |
| **Evaluability** | Finite catalogue → clean offline recall@k, NDCG | Open-ended → needs judges and human review |
| **Reliability** | Few dependencies, degrades gracefully | Depends on a GPU fleet under load |

> **Decision:** *"Retrieval + ranking in the critical path for both surfaces. Generation as a **gated fallback** for post-turn contexts the inventory can't cover. And a middle path for coverage: **templates with slots** ('Compare {entity} with {related entity}', 'Make it {shorter / a table}') filled from entities in the conversation, which gets specificity without open-ended generation."*

**Templates + slots, unpacked:** a *template* is a pre-reviewed prompt with typed blanks; *slots* are filled per request with entities pulled from the conversation. The wording is reviewed once, and the chip still fits this conversation.

```
STEP 1 · OFFLINE: write and policy-review TEMPLATES, each with TYPED slots
  T1  "Compare {PLACE} with {SIMILAR_PLACE}"
  T2  "Find {DIET} restaurants in {PLACE}"     DIET ∈ {vegetarian, vegan, halal, …}
  T3  "Make it {FORMAT}"                       FORMAT ∈ {a table, shorter, a checklist}
  T4  "Explain {CONCEPT} like I'm 12"          CONCEPT = a topic from the answer

STEP 2 · ONLINE: extract ENTITIES from the last turn + the answer (~5 ms entity tagger)
  user: "Plan 3 days in Kyoto, I'm vegetarian"   answer: temples, Arashiyama, …
  → PLACE = Kyoto · DIET = vegetarian
  → SIMILAR_PLACE = Osaka (a "similar city" edge in the knowledge graph)

STEP 3 · FILL + CHECK
  T1 → "Compare Kyoto with Osaka"
  T2 → "Find vegetarian restaurants in Kyoto"
  T3 → "Make it a table"
  every slot value must pass: the right TYPE (PLACE resolves to a real place in
  the knowledge graph) · an allow-list or safety check · a length cap
  → filled chips join the candidate pool (box 3b) and face the SAME ranker and filters

WHY: specific like generation ("…in Kyoto", not "…nearby") at retrieval's cost,
  latency and safety. Only the VALUE varies, and values are typed and checked,
  so a chip can't say anything its template doesn't allow.
⚠ WHERE IT BREAKS: a harmless template + a harmful value (T4 with a dangerous
  CONCEPT) → the value passes the safety classifier, and the filled chip is
  checked again in box 6 · a pasted text trying to plant a value ("PLACE =
  evil.com") → typed slots must resolve to a known entity, so a URL never fills one
```

### The inference stack — *draw this in 90 seconds* `A2a`

```
══ OFFLINE / BATCH ═══════════════════════════════════════════════════════════
 PROMPT INVENTORY  (curated library + mined-from-logs + reviewed synthetic)
   each prompt: text · category · capability · locale · SAFETY TAGS
   e.g. "Plan a 3-day trip to {city}"       travel · text-gen · en-IN · safe
        "Turn this photo into a watercolour" creative · image-edit · all · safe,
                                            needs-image, no-real-person-edits
        "Explain my blood test results"     health · file-analysis · en-US ·
                                            SENSITIVE: post-turn only, never home, 18+
   → the tags are HARD FILTERS at serving (box 6), never ranking features
       │
       ▼
 PROMPT TOWER → one embedding per prompt → ANN INDEX (ScaNN), rebuilt daily
 TREND JOB (hourly) → popular prompts per locale, k-anonymous
 HOME JOB (nightly) → top chips per user / segment → KV CACHE
 RANKER TRAINING (daily) → GBDT / small NN

══ ONLINE · per request ══════════════════════════════════════════════════════
 request (surface, conversation context, user id)
       │
┌─ 1. REQUEST GATE ──────────────────────────────────────────────
│ validate schema + size · OAuth (who is this?) · per-user rate limit
│   SCHEMA = the request has the fields and types we expect:
│     surface ∈ {home, post_turn} · locale ("en-IN") · user_id ·
│     context = last user turn + answer summary (text) · app version
│   SIZE = hard caps, so no single request can blow the budget:
│     context ≤ ~8 KB (≈ 2k tokens; drop the OLDEST turns first) ·
│     attachments passed by ID (a reference), never as inline bytes
│   e.g. ✅ {surface: "post_turn", locale: "en-IN", context: "…Kyoto…"}
│        ❌ surface: "sidebar" (unknown value) → reject, show nothing
│        ❌ a whole 2 MB PDF pasted into context → keep the last 8 KB
│   WHY: encoder cost grows with input length, and malformed or
│        oversized payloads are the cheapest way to attack a service
│ OUTPUT: an authenticated, well-formed request
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CONTEXT FILTER ────────────────────────────────────────────
│ strip PII · detect sensitive conversations (self-harm, medical,
│ grief) → SUPPRESS suggestions entirely or show support resources
│ · strip instructions embedded in pasted text (prompt injection)
│ OUTPUT: a safe context, or "show nothing"
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MULTI-SOURCE CANDIDATE GENERATION (all in parallel) ───────
│ a. TWO-TOWER ANN: context tower → embedding → top-200 prompts
│ b. TEMPLATES + SLOTS filled from conversation entities
│ c. TRENDING in this locale · d. CAPABILITY PROMOS (new features)
│ e. GENERATION FALLBACK, only if the gate in step 6 says so
│ OUTPUT: ~200–300 candidate prompts, each tagged with its source
│   SOURCE = which of a–e produced it: ann · template · trending ·
│            promo · generated. One prompt found by two sources keeps both.
│   e.g. {"Find vegetarian restaurants in Kyoto", source: template,
│         template_id: T2}  ·  {"Try Deep Research", source: promo}
│   WHY TAG IT: the ranker uses it as a FEATURE (a promo and a close
│     match aren't equally likely to be tapped) · QUOTAS and caps per
│     source (≤ 1 promo chip) · the REASON on the chip ("Trending",
│     "New feature") · STRICTER checks for "generated" in box 6 ·
│     per-source contribution on a dashboard, so a source that
│     silently returns nothing gets noticed
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. LIGHTWEIGHT RANKER (GBDT or small NN) ─────────────────────
│ features: two-tower score · popularity · freshness · category ·
│ user/segment affinity · surface · locale
│ predicts P(accept) and P(conversation succeeds | accept)
│ OUTPUT: candidates sorted by expected value, with CALIBRATED scores
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. DIVERSITY RERANKER (MMR) ──────────────────────────────────
│ pick greedily: λ · relevance − (1 − λ) · max similarity to the
│ chips already picked · plus a cap per category
│ (MMR = Maximal Marginal Relevance; worked example in §6)
│ OUTPUT: ~10 diverse candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. SAFETY / PRIVACY CHECK + CONFIDENCE GATE ──────────────────
│ safety classifier + policy tags · PII / rare-string check ·
│ dedupe against chips already shown this session
│ GATE on the calibrated top score: high → serve · medium → call
│ generation (if budget) · low → serve fewer chips, or none
│ OUTPUT: 3–5 chips + metadata (category, reason, source, version)
└──────┬─────────────────────────────────────────────────────────
       ▼
 CLIENT renders chips · logs impression, POSITION, tap, dismiss, outcome
       │
       └──► interaction logs → tomorrow's training data (a feedback loop: §7)
```

### Tiered, confidence-gated serving — the answer to "latency is too tight" `A1c` `F2b` `I1b`

**Raise this yourself:** *"If the latency budget is too tight for the quality we want, I wouldn't lower quality across the board. I'd tier it, and let a calibrated confidence decide per request."*

| Tier | When | Latency | What the user gets |
|---|---|---|---|
| **0 · Cache** | Home; or a repeat context | ~10–50 ms | Precomputed chips |
| **1 · Retrieve + rank** | Default | ~150 ms | Ranked inventory chips |
| **2 · Generation fallback** | Top calibrated score **below τ** *and* budget left (or prefetched during streaming) | +300–500 ms, hidden by the stream | Generated chips, safety-filtered and reranked |
| **3 · Degrade** | Low confidence and no budget, or a dependency is down | — | **Fewer chips, safe defaults, or none** |

> ⭐ *"Showing nothing is a legitimate output. A bad suggestion costs trust; a missing one costs nothing."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Retrieval in the critical path, generation as a fallback** | "Control, cost, latency and evaluability favour retrieval; generation buys coverage" | Generic chips for unusual contexts, mitigated by templates and the fallback |
| **2** | **Calibrated confidence gate** | "Spend the expensive tier only where the cheap one is unsure" | Calibration becomes a maintained property, not a nice-to-have |
| **3** | **MMR diversity on the slate** | "Five paraphrases of one idea is one suggestion" | A small relevance loss on the top slot |
| **4** | **North star = downstream completion** | "A tap that leads nowhere is negative value" | Slower, noisier metric than CTR |

---

## 5 · DATA & MODEL *(min 22–32)* — sub-questions **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F2c`

### 5.1 Which datasets matter (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Curated prompt library** | Editorial, policy-reviewed prompts tagged by category, capability, locale, safety | Small; thin in non-English locales |
| **Interaction logs** | Impressions **with position**, taps, dismissals, ignores, typed-instead, downstream outcome | **Only shown chips have labels** (selection bias) |
| **Conversation context** | Last user turn, the answer, entities, intent, attachment modality | Contains PII and sensitive topics |
| **Popularity / trend signals** | What people ask now, per locale, aggregated | Must be **k-anonymous**; can be poisoned |
| **User / segment data** | Locale, device, new vs returning, coarse interests | Consent (activity setting); no sensitive attributes |
| **Capability catalogue** | Features to teach (Deep Research, image generation, Gems) | Promo vs relevance tension |
| **Reviewed synthetic prompts** | LLM-generated candidates per category, human-reviewed: **cold start** for new locales/features | Style homogeneity |

### 5.2 Downselecting to a training set (e)

```
FROM RAW LOGS TO TRAINING EXAMPLES:
  1. ELIGIBILITY   only impressions actually RENDERED IN VIEW; drop bots,
                   internal traffic, users with activity/training opted out
  2. CLEAN         strip PII; drop conversations flagged sensitive or
                   policy-violating; dedupe near-identical prompts (embedding sim)
  3. LABEL         positive = tapped AND the conversation succeeded
                   strong negative = DISMISSED · weak negative = shown, ignored
                   ⚠ NOT a negative: chips in positions the user never looked at
  4. DEBIAS        inverse-propensity weights from a small RANDOMISED-ORDER
                   traffic slice (position bias); downsample head prompts
                   (popularity feedback loop)
  5. BALANCE       cap examples per user (heavy users dominate); stratify by
                   locale and category so small locales aren't drowned
  6. SPLIT BY TIME train on weeks 1…N, validate on week N+1; keep the
                   randomised slice as an UNBIASED eval set
```

**Step 4, unpacked: inverse-propensity weights (position bias).**

```
THE PROBLEM: a chip in slot 1 gets tapped more because more people LOOK at slot 1,
  not because it's better.   P(tap) = P(seen | slot) × P(useful)
  Train on raw taps and the model learns "slot 1 = good", which entrenches
  whatever the old ranker put first.

STEP A · MEASURE how often each slot is seen: the PROPENSITY κ_slot
  On 1–2% of traffic, SHUFFLE the chip order at random. Chip quality is then the
  same on average in every slot, so any difference in tap rate is PURE position.
    randomised slice tap rate:  slot 1: 10% · 2: 6% · 3: 4% · 4: 3% · 5: 2.5%
    κ (relative to slot 1):     1.0      · 0.6  · 0.4  · 0.3  · 0.25

STEP B · WEIGHT each tapped example by 1/κ of the slot it was tapped in
  a tap in slot 1 counts 1 · a tap in slot 4 counts 1/0.3 ≈ 3.3
  WHY: only ~30% as many people looked at slot 4, so a tap there is stronger
  evidence. For every tap we saw, ~3.3 would have happened if everyone had looked.
  IN THE LOSS: L = Σ_i w_i · ℓ_i with w_i = 1/κ_slot(i); for the pairwise
  ranker, each (tapped > ignored) pair gets the tapped chip's weight
  THE ONE-LINE PROOF: E[tap / κ] = κ · P(useful) / κ = P(useful)
  → on average, the weighted taps are what we'd see if every chip were examined

CAVEATS
  · a tiny κ → a huge weight → noisy training: CLIP κ ≥ ~0.1 (weight ≤ 10)
  · κ differs by surface and device → estimate it per surface
  · the shuffled slice costs a little quality on 1–2% of traffic. It's also
    the unbiased eval set (step 6), so it pays twice
  · alternative with no randomisation: a shallow POSITION TOWER (R4 §5.5)
```

**Where position is handled in EACH model.** Position belongs to the **impression**: this chip, in this slot, for this context. The two models deal with that differently.

```
TWO-TOWER RETRIEVAL (box 3a): NO position tower
  · the towers only meet in a dot product, so neither the context side nor the
    prompt side can hold a property of the PAIR (the slot it was shown in)
  · in-batch negatives were never shown in this context, so they have no
    position to correct for
  → weight each POSITIVE by 1/κ_slot (IPS, as above) and stop there
  · retrieval only needs the right prompt somewhere in the top-200; fine
    ordering is the ranker's job, so this is enough

RANKER (box 4): position as a TRAINING-ONLY input
  GBDT:      add `slot` as a feature in training; at serving, set it to the SAME
             value (slot 1) for every candidate → position can't reorder them
  NEURAL:    a SHALLOW POSITION TOWER, trained jointly with the main ranker
               logit = z_main(context, prompt) + z_pos(slot, surface, device)
             · a LOGIT is the raw score before the sigmoid:
               P(tap) = σ(logit) = 1 / (1 + e^(−logit))
             · z_main = the MAIN ranker's output: how useful THIS prompt is
               for THIS context (the big network, all the real features)
             · z_pos  = a TINY network's output: how much the SLOT alone
               raises or lowers the tap chance (slot 1 = 0, the reference)
             e.g. "Compare Kyoto with Osaka", z_main = −1.5
               shown in slot 1: z_pos =  0.0 → σ(−1.5) = 18% tap chance
               shown in slot 4: z_pos = −1.2 → σ(−2.7) =  6% tap chance
               → same chip, same usefulness; the slot alone cut taps ~3×
               (e^(−1.2) ≈ 0.30 = κ for slot 4: adding logits ≈ multiplying
                P(useful) × P(seen), the same split as IPW above)
             · position features feed ONLY z_pos; both get the same gradient
               (p − y under the BCE term), so z_pos soaks up "taps explained by the slot" and
               z_main learns usefulness
             · ~10% dropout on the slot feature during training
             · at serving: DROP z_pos and rank by z_main
  ⚠ the catch: the old ranker put better chips higher, so slot and quality are
    correlated, and z_pos can absorb some real quality. Train on the shuffled
    slice too, where slot carries no quality signal, and check offline on it.
```

**Why downsample head prompts (the popularity feedback loop).**

```
THE LOOP: a popular prompt is shown more → tapped more → fills more of the
  training data → the model scores it higher → it's shown even more
  e.g. per day:  "Tell me a joke"            2,000,000 taps
                 "Compare Kyoto with Osaka"        300 taps
  → a few HEAD prompts dominate the gradient; the model learns "popular",
    not "fits this conversation"; specific TAIL prompts are under-trained;
    coverage shrinks (failure mode 2 in §9)

THE FIX: cap each prompt's examples per day (keep each with probability
  min(1, T / count), e.g. T = 10,000)
    "Tell me a joke"  → 10,000 kept (0.5%)  ·  "Compare Kyoto…" → all 300 kept
  → the model has to learn WHICH CONTEXT a prompt fits, not how popular it is

NOT THROWING POPULARITY AWAY: popularity and trend remain explicit ranker
  FEATURES (box 4), so the model can still use them, but they no longer
  drown out context fit.
CALIBRATION: downsampling shifts predicted P(accept) for head prompts down, so
  the isotonic calibration step (§5.3) is fit on UNSAMPLED recent data. The
  confidence gate needs honest probabilities.
SAME IDEA, OTHER LOSS: the logQ correction in the retrieval loss (§5.3)
  handles popularity on the NEGATIVE side.
```

### 5.3 The models `A2b` `F1a`

**Retrieval: two towers, dot-product similarity.**
- **Context tower:** encodes the surface, the last user turn and an answer summary, conversation entities, and user/segment features → a vector.
- **Prompt tower:** encodes the prompt text, category and capability → a vector, **precomputed** for the whole inventory and indexed in ScaNN.
- **Score** = dot product of the two normalised vectors. Online, only the context tower runs.
- **Loss: contrastive (sampled softmax) with in-batch negatives**, with a **logQ correction** so popular prompts aren't over-penalised as negatives, plus **hard negatives** from dismissed chips.

```
for a batch of (context_i, accepted_prompt_i) pairs, s(c, p) = c·p / τ
L = − Σ_i log [ exp(s(c_i, p_i) − log Q(p_i)) / Σ_j exp(s(c_i, p_j) − log Q(p_j)) ]
    every OTHER example's prompt in the batch serves as a negative for free
```

**How the model knows which is the positive: the ROW is the label.** Each training row is one logged pair (a context, the prompt the user accepted in it). Nobody stores a "negative" label; the loss builds a B × B score table and says *"for row i, the right answer is column i"*.

```
BATCH OF B = 3 LOGGED PAIRS
  row 1: c1 = a Kyoto trip plan     p1 = "Make it a day-by-day table"
  row 2: c2 = a Python stack trace  p2 = "Explain this error line by line"
  row 3: c3 = a cover-letter draft  p3 = "Make it more formal"

SCORE TABLE  S[i][j] = s(c_i, p_j)   (every context against every prompt)
              p1      p2      p3
   c1       [ 4.1 ]   0.3     1.2      ← row 1 is a 3-way choice; answer: column 1
   c2         0.2   [ 3.8 ]   0.5
   c3         1.0     0.4   [ 3.5 ]
   DIAGONAL = positives (the logged pairs) · OFF-DIAGONAL = negatives for free

  in code:
    C = context_tower(contexts); P = prompt_tower(prompts)   # B × d each
    S = C @ P.T / τ                     # the B × B table
    S = S − log Q[None, :]              # logQ: each COLUMN pays for its prompt's popularity
    S[same prompt, off-diagonal] = −∞   # "accidental hits": a popular prompt twice in a batch
    loss = cross_entropy(S, labels = [0, 1, …, B−1])

  the softmax over row 1 pushes 4.1 UP and pushes 0.3 and 1.2 DOWN, the 1.2 harder:
  a more confusable negative ("Make it more formal" for a trip plan) gets more push

⚠ THE ASSUMPTION: another row's prompt is a negative for THIS context. Almost
  always true (a cover-letter prompt for a Kyoto trip), but two Kyoto users in one
  batch would make each other's good prompts look like negatives ("false negatives").
  Large batches (thousands) make it rare; same-prompt duplicates are masked;
  dismissed chips are added as EXTRA columns, used only for their own row (hard negatives).
```

**"Two-tower ANN", unpacked:** *two towers* = two separate encoders that never see each other's input, so one side can be computed in advance; *ANN* = **approximate nearest-neighbour** search over that precomputed side.

```
OFFLINE (daily):  every prompt in the inventory → PROMPT TOWER → a 128-d vector
                  say ~10⁶ prompts across locales → one ANN index (ScaNN)

ONLINE (per request):
  context: the user planned a Kyoto trip; the answer listed temples
    → CONTEXT TOWER → q, a 128-d vector                  ~5 ms (small int8 encoder)
    → ANN: "which prompt vectors p have the largest q·p?" → top-200     ~2 ms
        0.82  "Make it a day-by-day table"
        0.79  "Find vegetarian restaurants nearby"
        0.77  "Suggest a packing list"
        …
        0.12  "Write a poem about the sea"    ← far away in the space: not returned
    → the RANKER rescores these 200 with richer features (§4, box 4)

WHY TWO TOWERS: the prompt side never depends on the request, so 10⁶ prompt
  vectors are computed ONCE; per request, only one encoder pass runs.
  The price: the towers meet only in a dot product, so they can't model fine
  context × prompt interactions. That's the ranker's job.
WHY APPROXIMATE: exact search streams 10⁶ × 128 × 4 B = 512 MB of vectors per
  request, which is tens of ms and memory-bound, over the 15 ms budget. ScaNN
  partitions the space into clusters, searches only the few nearest ones, and
  scores compressed vectors: ~1–3 ms at ~95–99% recall of the exact top-200.
  → recall@200 against exact search is a tracked offline metric
  (with only ~10⁴ prompts, exact search would be cheap enough: see R6)
```

**Ranking: GBDT (LambdaMART) or a small neural ranker**, trained with a **pairwise ranking loss**: within one shown slate, the tapped chip should outscore the ignored ones. Two heads: P(accept) and P(success | accept). Final score = P(accept) × P(success | accept), then **calibrated** (isotonic), because the confidence gate uses the number directly.

**Which loss for which head, precisely:**

```
HEAD 1 · P(accept): PAIRWISE + a POINTWISE term
  the label is a CHOICE among 3–5 chips shown together, so compare chips
  WITHIN the same slate. For tapped chip i and ignored chip j (s = ranker score):
      L_pair = −log σ(s_i − s_j)            (RankNet form; LambdaMART also weights
                                             each pair by how much swapping i and j
                                             would change NDCG)
  WHY PAIRWISE: anything that's the same for the whole slate (this user taps a
    lot, it's late at night, the answer was bad) CANCELS in s_i − s_j. The model
    learns which chip beats which, not which sessions are tappy.
  WHY NOT ONLY PAIRWISE: pairwise fixes the ORDER but not the SCALE. s_i − s_j
    is the same whether both chips are good or both are bad, yet the confidence
    gate needs a real probability. So either:
      L = L_pair + α · L_BCE    (a pointwise term keeps scores calibrated), or
      pairwise only, then ISOTONIC calibration on fresh unsampled data
  + IPW: weight each pair by 1/κ of the tapped chip's slot (§5.2)

HEAD 2 · P(success | accept): POINTWISE BCE, on tapped rows only
  usually one tap per slate, so there's no pair to compare. The label is
  absolute: did the conversation that followed succeed? → binary cross-entropy

FINAL:  score = P(accept) × P(success | accept), both calibrated
```

> **Say:** *"Pairwise for acceptance, because a tap is a choice among chips shown together and pairwise cancels everything about the session. Plus a pointwise term or isotonic calibration, because the gate needs a probability, not just an order. Pointwise for success, because it's an absolute outcome."*

**Isotonic calibration and IPW on pairs, unpacked:**

```
ISOTONIC CALIBRATION: turn raw ranker scores into honest probabilities
  1. take FRESH data (last few days) that was NOT downsampled: downsampling
     head prompts (§5.2) distorts tap rates, so it can't be used here
  2. bucket by raw score; measure the real tap rate in each bucket
        raw score      −3…−2   −2…−1   −1…0    0…1    1…2
        tap rate        2%      5%      4%     12%    25%
                                 └── 5% then 4%: going DOWN breaks the order (noise)
  3. "isotonic" = must never go down as the score goes up, so merge the
     offending neighbours into their average (pool-adjacent-violators):
        calibrated      2%     4.5%    4.5%    12%    25%
  4. serving: P(accept) = that step function applied to the raw score
  · the ORDER of chips doesn't change (the map only goes up), only the NUMBERS
  · vs Platt scaling (fit a sigmoid): isotonic assumes no shape, but needs more
    data; refit daily, per surface

IPW ON PAIRS: which pairs to form, and how much each counts
  slate:  slot 1 A (ignored) · slot 2 B (ignored) · slot 3 C (TAPPED) · slot 4 D · slot 5 E
  PAIRS:  C > A and C > B, the chips ABOVE the tap: the user saw them and skipped
          them, plus D, the one just BELOW (usually seen too)
          not C > E: slot 5 may never have been looked at, and "unseen" isn't
          "rejected" (the same rule as §5.2)
  WEIGHT: each pair × 1/κ of the TAPPED chip's slot = 1/κ₃ = 1/0.4 = 2.5
          a win from slot 3 is stronger evidence than a win from slot 1, because
          fewer people look at slot 3, and the TAP is the position-biased event
  LOSS:   Σ_pairs  w · ( −log σ(s_C − s_other) )
```

### 5.4 "Suppose you pick PEFT for training — what's the objective?" (f) `F2c` `F1a`

**Name where PEFT applies, then give each objective:**

| Where | PEFT method | Objective |
|---|---|---|
| **The context/prompt encoder** (fine-tune an existing pretrained encoder) | LoRA adapters; base frozen | **Contrastive InfoNCE** on (context, accepted prompt) pairs, in-batch + hard negatives (dismissed), logQ-corrected, **IPS-weighted** for position |
| **The generation-fallback LLM** | LoRA on a small LLM | **Stage 1, SFT:** next-token cross-entropy on context → suggestions that were *accepted and succeeded*. **Stage 2, preference (DPO):** chosen = accepted chip, rejected = a **dismissed chip from the same slate** |

**Why PEFT here:** cheap to train and iterate; **one adapter per surface or locale** on a shared base; the frozen base keeps general language understanding; adapters swap at serving without a new model fleet.

> **Say:** *"The objective, in words: make the model rank or produce what users accept **and** succeed with, measured on debiased labels. Concretely that's InfoNCE for the encoder adapters, and SFT then DPO on accepted-versus-dismissed pairs for a generator. I'd pair the DPO examples within the same slate so position and context are held fixed."*

### 5.5 The staged training pipeline `A2d`

```
STAGE 1 · INVENTORY BUILDING   curated + mined-from-logs (aggregated, k-anonymous,
                               PII-scrubbed, human-reviewed) + reviewed synthetic
                               → dedupe → category + safety tags
STAGE 2 · EXAMPLE GENERATION   logs → (context, slate, positions, taps, dismissals,
                               outcomes) → labels + IPS weights (§5.2)
STAGE 3 · INDEX + RANKER BUILD encoder adapters (PEFT) → embed inventory → ScaNN
                               index; train + calibrate the ranker
STAGE 4 · EVALUATE + DEPLOY    offline gates (recall@k, NDCG, calibration, safety
                               audit) → shadow → canary → A/B → ramp
    SHADOW  the new model runs on LIVE requests alongside the current one, but its
            chips are only LOGGED, never shown. Checks latency, errors, unsafe-flag
            rate and how often it agrees with production. Zero user risk.
    CANARY  the new model really SERVES a tiny slice (~1% of traffic, or one
            region) for hours, with AUTOMATIC ROLLBACK if latency, errors or the
            unsafe rate breach. Catches what shadow can't: real clients, real load.
    A/B     a controlled experiment (e.g. 5% vs 5%, 1–2 weeks): are acceptance
            AND downstream completion better, with every guardrail held?
    RAMP    5% → 25% → 50% → 100%, guardrails checked at each step; a small
            long-term holdout stays on the old version
    → canary asks "is it SAFE to serve?" (operations, hours);
      A/B asks "is it BETTER?" (product, weeks)
CADENCE: trend source hourly · index + ranker daily · encoder adapters weekly
```

---

## 6 · SERVING *(min 32–38)* — sub-question **(g)** `F3a` `F3b` `A3b` `I1d`

- **Home: precompute.** A nightly batch computes chips per user (or per segment for new users) into a KV cache; online does a light re-rank for time of day and recent activity. **A lookup, not a computation.**
- **Post-turn: prefetch during streaming.** Kick off suggestion computation when the answer starts streaming; finalise with the full answer.
- **Scale via precomputation:** prompt embeddings are precomputed; only the small context tower runs online; ANN index sharded by locale and replicated; GBDT ranker on CPU. `F3b`
- **Quantization:** int8/FP8 for the context encoder and the fallback LLM. Lower-precision weights (and activations) mean less memory and bandwidth per request, so it's **cheaper and faster, with a quality trade-off** that must be gated on the eval set. `F3a`
- **Diversity reranker (MMR):** `score = λ · relevance − (1 − λ) · max_sim(candidate, already picked)`, with λ tuned online (start ~0.7) and a per-category cap. `A3b`

  **MMR, unpacked.** *Maximal Marginal Relevance* (Carbonell & Goldstein, 1998, from search and summarisation) builds the slate **one chip at a time**. Each step picks the candidate with the best **marginal** value: how relevant it is, minus how much it repeats what's already picked. Picking the best *set* exactly is combinatorial; greedy is fast (~2 ms) and good enough.

  ```
  score(c) = λ · relevance(c) − (1 − λ) · max_{s ∈ picked} sim(c, s)
    relevance  the ranker's calibrated score (box 4)
    sim        cosine of the two prompts' embeddings (already computed by the prompt tower)
    λ          the knob: λ = 1 → pure relevance · λ = 0 → pure diversity

  WORKED EXAMPLE, post-turn after a Kyoto trip plan, λ = 0.7, pick 3:
    A "Make it a day-by-day table"           rel 0.90   format
    B "Turn it into a daily schedule"        rel 0.88   format   ← sim(A, B) = 0.95
    C "Find vegetarian restaurants in Kyoto" rel 0.80   food
    D "Compare Kyoto with Osaka"             rel 0.75   travel
    E "Suggest a packing list"               rel 0.70   travel
    (other similarities 0.15–0.40)

    pick 1: nothing picked yet → highest relevance → A
    pick 2: B = 0.7·0.88 − 0.3·0.95 = 0.33   ← punished for repeating A
            C = 0.7·0.80 − 0.3·0.20 = 0.50   ✅
            D = 0.7·0.75 − 0.3·0.30 = 0.44
    pick 3: D = 0.44 ✅ (beats E = 0.39 and B = 0.33)
  → MMR shows A, C, D. TOP-3 BY RELEVANCE would show A, B, C: two chips that
    say the same thing, so the user effectively gets two suggestions, not three.

  THE CATEGORY CAP is a HARD rule on top (e.g. ≤ 2 chips per category): MMR's
    penalty is soft, and a cap guarantees the slate spans at least two kinds of help.
  λ is tuned ONLINE: too high → near-duplicates; too low → diverse but less
    relevant chips. Watch acceptance AND category entropy as λ moves.
  (A determinantal point process, DPP, scores the whole set at once instead of
    greedily; R3 uses it. MMR is the simpler, standard first choice.)
  ```
- **Reliability:** every dependency has a timeout; on timeout, drop to the next tier down. The chat never waits for suggestions. `I1b`

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — sub-question **(h)** `I2a` `I2b` `F1b` `F1c`

**How you'll know it worked:**

```
OFFLINE GATES   recall@k (retrieval) · NDCG@5, MRR (ranker) · diversity · coverage
                · calibration (ECE) · safety audit on a sampled slate set
                ⚠ evaluated on the RANDOMISED slice, or with IPS weights, because
                  logs only label what the OLD system chose to show
SHADOW          compute new suggestions for live traffic, log, DON'T show:
                latency, safety flags, overlap with production
A/B             1% → 5% → 50%: acceptance, DOWNSTREAM COMPLETION, dismiss rate,
                guardrails (P95 latency, cost/request, unsafe rate, privacy leakage)
LONG-TERM       a holdout that never gets suggestions (or gets the old ones),
                read on D7/D28 RETENTION
RANKER SPEED-UP interleaving: two rankers' chips mixed in one slate; faster
                and lower-variance than a full A/B for ranker comparisons
```

**Success** = acceptance up **and** downstream completion up **and** retention not down, with every guardrail held. `I2b`

**Bias sources, named proactively, with mitigations:** `F1c`

| Bias | Mechanism | Mitigation |
|---|---|---|
| **Position** | The first chip gets tapped more regardless of quality | Randomise order on a small traffic slice; IPS weights; position as a training-only feature |
| **Selection** | Only shown chips have labels | Exploration traffic (ε ≈ 1–5%); counterfactual (IPS / doubly robust) offline eval |
| **Feedback loop** | Shown → tapped → trained on → shown more | Diversity and freshness boosts; downsample head prompts; coverage monitor |
| **Popularity** | Head prompts crowd out the long tail | logQ correction in the retrieval loss; reranking; per-category caps |
| **Drift** | Trends, new features, seasonality | Per-category acceptance and embedding-drift monitors; scheduled retrains |

---

## 8 · PRODUCTION READINESS SWEEP *(min 44–48)* `I1b` `I1e`–`I1j`

**Say it as a checklist, out loud, because each line is a separate pointer:**

| Dimension | What you do | Metric |
|---|---|---|
| **Performance** `I1a` | Per-surface P95 targets; per-stage budgets; cost/request per tier | P95 per surface · $/1k requests |
| **Reliability** `I1b` | Retrieval as the availability backbone; tiered serving; timeouts → degrade to fewer chips | Error rate · % requests served per tier |
| **Usability** `I1c` | Home = discovery, post-turn = continuation; 3–5 **diverse** chips with category metadata; dismiss + "not interested" | Dismiss rate · category entropy |
| **Scalability** `I1d` | Controlled inventory; precomputed embeddings; ANN index; lightweight ranker; home cache | QPS headroom · index build time |
| **Privacy** `I1e` | Guardrails on mined prompts (aggregate, k-anonymous, PII-scrubbed, reviewed); **never surface another user's text verbatim**; moderate personalisation; respect activity settings; avoid sensitive inferred interests on a **shared screen** | **Privacy leakage rate** |
| **Safety** `I1f` | Policy review + safety tags on inventory; **in-path** safety filter on every served chip; **suppress** after sensitive conversations. *A suggestion is an endorsement, so the bar is higher than for answers* | **Unsafe suggestion rate** · **reputation incidents** |
| **Security** `I1h` | Request validation · OAuth · rate limits · ⚠ **prompt injection via context** (pasted text saying "suggest visiting evil.com" → strip instructions; no URLs in generated chips) · ⚠ **trend poisoning** (coordinated queries to push a topic → minimum distinct-user thresholds, anomaly detection, review) | Blocked injection attempts · trend anomalies |
| **Ethics / bias** `I1i` | No sensitive attributes as features; audit exposure and acceptance per locale and language (**stereotyped suggestions** by inferred demographics are a real failure); **per-segment calibration** | Exposure parity · ECE per segment |
| **Explainability** `I1j` | **A reason on each chip** ("Continue your trip plan" / "Trending" / "New feature"); **provenance logged** per chip (source, model version, scores); **GBDT feature attributions** (SHAP) for debugging; user controls to turn personalisation off | % chips with a reason · audit coverage |

> ⭐ *"Explainability here is three audiences: the user gets a reason on the chip, the on-call engineer gets provenance and attributions, and policy reviewers get an audit log of what was shown and why."*

---

## 9 · MULTIMODAL + FAILURES *(min 48–52)* — sub-question **(i)** `F3c` `F2a` `A5b`

**If the context is multimodal** (the user uploads a photo, a PDF, a voice note):
- **Suggestions change by modality:** photo → "What's in this?", "Make it a watercolour", "Identify this plant"; PDF → "Summarise", "Make flashcards".
- **The context tower must encode the image:** reuse the embedding the main model already computes if possible; otherwise a vision encoder adds **~10–30 ms of GPU per request**, a new cost on a CPU-only path.
- **Embedding-maturity risk:** joint image-text embeddings are strong for captions, weaker for "what action would this user want next". Mitigation: **start with modality-conditioned templates** (cheap, safe), add learned joint retrieval once there's interaction data.
- **Safety gets harder:** never suggest prompts that judge people in a photo ("rate this person"); face and child-safety rules apply to images.
- **Inventory:** needs capability-specific prompts per modality, reviewed like the text ones.

> **Say:** *"Multimodal adds a vision encode to the critical path and an embedding that's less proven for this task, so I'd start with modality templates and earn the learned version with data."*

**When retrieval underperforms, the generation pipeline on demand** `F2a` `A5b`:

```
context → SMALL LLM proposes ~10 candidate chips (PEFT-tuned, §5.4)
        → SAFETY FILTER (same classifier + policy rules; no URLs, no PII)
        → DEDUPE (against each other, against inventory, against chips already shown)
        → RERANK (the same ranker, so scores are comparable with retrieved chips)
        → QUALITY GATE (calibrated score ≥ τ; must reference real Gemini capabilities,
                        no hallucinated features)
        → merge with retrieved chips → MMR → serve
```

**Top 3 failure modes:**
1. **An embarrassing or unsafe chip goes viral.** The reputational cost is asymmetric because a suggestion reads as Google's endorsement. → in-path filter, suppression after sensitive topics, audit sampling.
2. **Feedback-loop homogenisation.** The same 50 prompts for everyone, coverage collapses silently. → coverage and diversity monitors, exploration traffic.
3. **Privacy leak through mined prompts.** One user's specific question resurfaces for others. → k-anonymity, PII scrub, human review before anything mined enters the inventory.

**Q1 ships:** home chips from the curated library + two-tower retrieval + GBDT ranker + MMR + safety tags, and post-turn **templates with slots**; generation fallback in a second phase once its quality gate is proven in shadow. With panels for downstream completion, unsafe rate and P95 latency.

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just have the LLM generate suggestions every time?"**
> Because the critical path cares about five things generation is bad at: latency, cost at ~35k QPS, safety control, evaluability and availability. A curated inventory is pre-reviewed, so every chip has passed policy before anyone sees it; generation needs a filter on every output and still has a tail of surprises, and a suggestion reads as Google's endorsement. Retrieval is also evaluable offline with recall@k and NDCG on a finite catalogue. Where generation clearly wins is coverage of specific contexts, so I use it as a **gated fallback** behind a calibrated confidence threshold, hidden behind answer streaming, and I close most of the coverage gap more cheaply with templates and slots.

**2. "Retrieval quality is poor for long, specific conversations. What do you do?"**
> First diagnose which stage: if the right prompt isn't in the top-200, it's recall, so look at the context tower (is the answer summary in the input? are entities extracted?); if it's there but ranked low, it's the ranker. Then three fixes in order of cost: **templates with entity slots**, which add specificity with no generation; **fine-tuning the encoder** with PEFT on accepted vs dismissed pairs from these contexts; and the **generation fallback**, gated on low calibrated confidence. I'd measure recall@k sliced by conversation length, because the average will hide exactly this.

**3. "How do you handle position bias in your training data?"**
> Position bias means the first chip is tapped more whatever its quality, so naive training learns "whatever was first is good" and entrenches the old ranker. I'd run a small **randomised-order slice** of traffic to estimate the examination probability per position, then weight training examples by **inverse propensity**. I'd also include position as a feature during training and fix it to a constant at serving, and treat chips in positions the user likely never looked at as unlabelled rather than negative. For offline evaluation of a new ranker, I'd use the randomised slice or a doubly robust estimator, never raw logs.

**4. "Acceptance went up 8% in the A/B. Ship it?"**
> Not on that number alone. I'd check **downstream completion** first: if users tap more but the conversations that follow are shorter, get more regenerates or more abandons, we've built clickbait chips. Then the guardrails: unsafe rate, privacy leakage, P95 latency, cost per request. Then whether the gain is broad or concentrated in one category or locale, and whether **novelty** explains it, which the long-term holdout answers. Ship if completion and retention hold and guardrails are clean.

**5. "How do you build the prompt inventory without leaking user data?"**
> Mined prompts are the risk, because a user's specific question resurfacing for others is a privacy incident. So anything mined from logs must be **aggregated** across many users, pass a **k-anonymity** threshold on distinct users, be **PII-scrubbed**, be generalised where it's specific ("Plan a trip to {city}"), and pass **human review** before entering the inventory. Users with activity or training opted out are excluded at source. And I'd track a **privacy leakage rate** on audited samples as a hard guardrail.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your confidence gate depends on calibration. What happens when calibration drifts?"**
> The gate silently starts making the wrong trade: over-confident scores skip the fallback when it's needed, under-confident ones send too much traffic to the expensive tier and blow the cost budget. So calibration is a **monitored SLO**, not a training-time step: ECE per surface and segment on recent traffic, recalibration (isotonic on a fresh window) on a schedule and on alert, and a **cost-per-request alarm** as a second signal, because a drifting gate usually shows up in the bill before it shows up in quality.

**7. "Product wants to use suggestions to promote new features. How do you handle it?"**
> That's a policy decision disguised as a ranking one, so I'd make it explicit rather than bury a boost in the ranker. A **capability-promo source** with its own candidates, a **budgeted exposure share** (say, at most one promo chip per slate), and its own success metric (feature adoption *and* downstream completion). If promo chips are tapped but abandoned, the budget should shrink. And the budget is owned by product, reviewed like any other promise to users.

**8. "How would you evaluate the generation fallback offline?"**
> It's open-ended, so offline metrics on a fixed catalogue don't apply. I'd use a **calibrated LLM judge** for relevance and specificity against a rubric, validated against human ratings on a sample, plus deterministic checks: safety classifier, PII detector, URL/feature-hallucination checks, and duplication against the inventory. Then **shadow** it on the contexts where retrieval confidence was low, and compare acceptance and completion in an A/B restricted to that slice, since that's the only traffic it will ever serve.

**9. "After a conversation about a sensitive topic, what should happen?"**
> Usually **no suggestions**, or a single support-resource chip. Suggesting "Tell me more" after a conversation about self-harm, a diagnosis or a bereavement is actively harmful, and it's the kind of screenshot that becomes a reputation incident. So the context filter classifies sensitivity before any candidate generation and can short-circuit the whole stack. It's the same lesson as self-harm in content moderation: the safe action depends on the topic, and here the safe action is to say less.

**10. "How do you stop suggestions from making everyone's experience the same?"**
> That's the feedback loop: shown, tapped, trained on, shown more. Three levers: **exploration** traffic that shows lower-ranked or new prompts so they can earn labels; **diversity and freshness** in the reranker plus per-category caps; and **coverage as a tracked metric**, the share of the inventory and categories shown at least N times per week, by locale. When coverage shrinks while acceptance holds, the system is narrowing, and that's the early warning that retention will follow.

---

## ⭐ THE L6 LINE

> **"A suggestion is an endorsement, so the critical path is retrieval over a reviewed inventory, with generation as a calibrated, gated fallback, and the metric is whether the conversation it starts succeeds, not whether the chip was tapped."**

## 🧭 HOOKS — *steer by what you mention*

- *"A suggestion is an endorsement"* → invites the safety / reputation branch
- *"Showing nothing is a legitimate output"* → invites the tiered-serving and reliability branch
- *"CTR is available; downstream completion is meaningful"* → invites the metrics branch (**good at L6**)
- *"Only shown chips have labels"* → invites the bias / counterfactual-eval branch (**good at L6**)
- *"Compute post-turn chips while the answer streams"* → invites the serving branch
- *"Templates with slots get specificity without generation"* → invites the retrieval-vs-generation branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the decision that drives everything: what's in the critical path."* Retrieval vs generation regenerates the architecture.
- **Running out of time?** → *"I have five minutes. I'd rather do the production-readiness sweep than finish the training pipeline. Sound right?"* (The sweep is worth ~10 pointers.)

**Name and defer:**
- "**Typeahead** is a ~50 ms autocomplete problem with its own design (G4)."
- "**Personalised LLM-written greetings** on the home screen are a separate generation product."
- "**Cross-device continuity** of suggestions needs a session store; not in scope."
- "**Ads or sponsored suggestions** would need disclosure and a separate policy review."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, where you earn it, and the line that earns it

**Codes:** **A** = Parameter 1, System Architecture/Design · **F** = Parameter 2, AI Fluency · **I** = Parameter 3, AI Product Integration. The number is the sub-parameter, the letter the pointer.

### A · System Architecture / Design

| Code | Pointer (from the form) | Where | The line that earns it |
|---|---|---|---|
| **A1a** | Retrieval + ranking over generation in the critical path; generation as fallback | §0, §4 | *"Retrieval + ranking in the critical path; generation as a gated fallback."* |
| **A1b** | Per-surface latency budget (home P95 300–500 ms, post-turn P95 200–400 ms) | §1, §3 | *"Home P95 300–500, post-turn P95 200–400 from the end of the answer, and here's the per-stage budget."* |
| **A1c** | Tiered confidence-gated serving instead of compromising quality | §4 | *"I wouldn't lower quality across the board; I'd tier it and let calibrated confidence decide."* |
| **A2a** | Full inference stack (validation, OAuth, rate limits, context filter, multi-source candidates, ranker, diversity reranker, safety/privacy, 3–5 results + metadata) | §4 diagram | Walk the six boxes in order |
| **A2b** | Two-tower retrieval (context + prompt tower, dot product) + GBDT / light neural ranker | §5.3 | *"Context tower and prompt tower, dot product, then a GBDT ranker."* |
| **A2c** | Data sources (library, logs, context, trends, user/segment) | §5.1 | The datasets table |
| **A2d** | Staged training pipeline (inventory, examples, index + ranker, eval + deploy) | §5.5 | The four stages |
| **A3a** | Organise top-down unprompted | §0 | *"I'll go: questions, product, metrics, budgets, stack, data, serving, eval, readiness, multimodal."* |
| **A3b** | Diversity reranker with a relevance–λ diversity objective | §4, §6 | *"MMR: λ·relevance − (1−λ)·max similarity to chips already picked."* |
| **A4a** | Clarifying questions up front; scope as explicit assumptions | §1 | Ask the 5 questions **before being prompted**; state the assumptions table |
| **A5a** | Compare retrieval vs generation on control, cost, latency, coverage, evaluability | §4 | The comparison table |
| **A5b** | Design a full generation pipeline on demand (generate, safety filter, dedupe, rerank, quality gate) | §9 | The five-step pipeline |

### F · AI Fluency

| Code | Pointer | Where | The line that earns it |
|---|---|---|---|
| **F1a** | Models matched to subproblems: two-tower, contrastive loss, in-batch negatives; GBDT / neural ranker with pairwise loss | §5.3 | *"Contrastive loss with in-batch negatives and logQ correction; pairwise ranking loss for the ranker."* |
| **F1b** | Metrics and trade-offs: recall@k, NDCG, MRR, diversity, coverage, calibration | §2, §7 | Name each with **why** (recall for retrieval, NDCG/MRR for ranking, calibration for the gate) |
| **F1c** | Bias sources (position, selection, feedback, popularity) + mitigations | §7 | The bias table |
| **F2a** | Retrieval underperforming → generation fallback with safety, dedupe, rerank | §9, FU 2 | Raise it **yourself**: *"If retrieval underperforms on specific contexts…"* |
| **F2b** | Latency too tight → tiered confidence-gated serving | §4 | Raise it **yourself** |
| **F2c** | Fine-tune an existing encoder on accepted / dismissed pairs | §5.4 | *"LoRA on the existing encoder, InfoNCE on accepted pairs, dismissed as hard negatives."* |
| **F3a** | Quantization: lower-precision weights and activations, cheaper and faster with a quality trade-off | §6 | *"int8/FP8 for the encoder and fallback LLM, gated on the eval set."* |
| **F3b** | Scale via embedding index, lightweight rankers, precomputed embeddings | §6 | *"Only the context tower runs online; prompt embeddings are precomputed."* |
| **F3c** | Multimodal extension, noting cost and embedding-maturity risk | §9 | *"A vision encode on the critical path, and an embedding less proven for this task."* |

### I · AI Product Integration

| Code | Pointer | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance: per-surface latency + cost/request + P95 tracking | §3, §8 | *"$ per 1,000 requests per tier, P95 per surface."* |
| **I1b** | Reliability: availability in retrieval-vs-generation; tiered serving | §4, §6, §8 | *"Retrieval is the availability backbone; on timeout, drop a tier."* |
| **I1c** | Usability: home discovery vs post-turn; 3–5 diverse results with category metadata | §2 | The product mock-up |
| **I1d** | Scalability: controlled inventory, embedding index, precomputed embeddings | §6, §8 | *"Home is a lookup; the inventory is controlled."* |
| **I1e** | Privacy: guardrails + privacy leakage rate; moderate personalisation | §8, FU 5 | *"Privacy leakage rate is a hard guardrail."* |
| **I1f** | Safety: in-path filters, policy review, safety tags; unsafe rate + reputation metrics | §8 | *"A suggestion is an endorsement."* |
| **I1h** | Security: request validation, OAuth, rate limiting | §4, §8 | **Go further:** prompt injection via context, trend poisoning |
| **I1i** | Ethics / bias: identification, mitigation, calibration | §7, §8 | *"Exposure parity per locale, per-segment calibration, no sensitive attributes."* |
| **I1j** | Explainability *(blank in the sample: easy points)* | §8 | *"Three audiences: a reason on the chip, provenance for on-call, an audit log for policy."* |
| **I2a** | Offline (NDCG, MRR) → online (shadow, A/B on acceptance + downstream), correcting for shown-only labels | §7 | *"Offline on the randomised slice or with IPS, because logs only label what we showed."* |
| **I2b** | Success = acceptance/CTR + downstream completion + retention | §2, §7 | *"Acceptance up, completion up, retention not down, guardrails held."* |
