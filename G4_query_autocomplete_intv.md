# G4 · Query Autocomplete — **INTERVIEW SHEET**

> **Tier 2 · #18** — classic Google, and deceptively deep. The safety framing is what separates answers.
> Full study file: [`../G4_query_autocomplete.md`](../G4_query_autocomplete.md) · [diagrams](../G4_query_autocomplete_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the serving path · (d) datasets · (e) building the candidate set · (f) the offline models and their objectives · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Personalisation? The safety obligation for named entities? Freshness? (§1) |
| **Explainability** is easy points | Every suppression logged with **the rule that fired**; a "report this prediction" link (§8) |
| **Security** beyond the gateway | **Manufactured query volume** to plant a suggestion (§8) |
| **Troubleshoot before being asked** | Raise the **QPS arithmetic** and the **trending-vs-vetted** trade yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A suggestion is a publication** · 50 ms on every keystroke | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · the harmful-suggestion gate · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The box · acceptance **and** downstream success · **showing nothing is fine** | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | ⚠ **10–20× search QPS** — the arithmetic that rules out models | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Retrieve vs generate · trie/FST → thin scoring → **safety layer** | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Logs as the corpus · k-anonymity · offline scorer · spell correction | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING + SAFETY** | **(g)** | **Replicate, don't shard** · caching · atomic pushes · the safety subsystem | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Replay + interleaving · safety audits · the feedback loop | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | New locales, entity completion · the trending fast lane · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Two things make this different from everything else in the search stack, and I want both on the table before I design anything.**
>
> **First, the latency budget is brutal and the QPS is enormous, because this fires on every keystroke. A twenty-character query generates twenty requests, so autocomplete QPS is roughly ten to twenty times search QPS, with maybe fifty milliseconds of server budget instead of hundreds. That rules out almost every interesting model at serving time — this is a precomputation-and-trie problem with a thin ML layer on top, not an ML problem with a cache in front of it.**
>
> **Second — and I'd build the whole design around this — a suggestion is a publication. We're putting words in the user's mouth and appearing to endorse them. If the box completes a person's name with an accusation, that's a news story and a defamation exposure, and 'that's what people actually search for' is not a defence, because we chose to surface it."**

**Then the architectural consequence, stated immediately:**
> ⭐ **"So I retrieve suggestions from a vetted set of real queries; I do not generate them.** The moment a model composes a suggestion nobody ever typed, the space of things I have to defend becomes unbounded and no filter can be complete. Retrieval keeps the candidate set **finite, auditable, and reviewable in advance.**"
>
> And: **"showing nothing is a perfectly good output."** An empty list costs a keystroke or two; a harmful suggestion costs a great deal. **That asymmetry sets every threshold in the system.**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the box and how it's measured, the QPS arithmetic, the architecture, the data and offline models, serving and the safety subsystem, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Is personalisation in scope?** *(Changes the cache story completely — a personalised suggestion can't be shared across users.)*
2. **What's the safety obligation for named entities?** *(People, groups, religions — the highest-risk slice by far.)*
3. **How fresh must suggestions be?** *(Breaking-news queries appear in minutes and are the highest-risk slice *and* the most valuable.)*
4. **Which locales and input methods?** *(Blocklists and prefix logic don't transfer across languages.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | The web search box, desktop and mobile |
| Load | ~10–20× search QPS (every keystroke is a request) |
| Corpus | Tens of millions of **vetted** past queries per locale |
| Personalisation | A light re-rank over shared candidates, with user controls |
| Freshness | Trending queries eligible within minutes, through a stricter lane |
| Locales | Many, each with its own policy and blocklist |

**FR:** rank completions for a prefix in ≤50 ms · typo tolerance · freshness · locale/language · **safety filtering as a first-class stage** · personalisation (optional).

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Latency** | **p99 ≤ 50 ms** end-to-end — *per keystroke* |
| **QPS** | **10–20× search QPS** |
| Freshness | New trending query surfaces in **minutes** |
| **Harmful-suggestion rate** | **~0** — a hard gate, and showing nothing is an acceptable fallback |
| Availability | On any failure, **fail to empty**, never to unfiltered suggestions |

**Scope / NON-GOALS:**
- ❌ **Generating suggestions with a model.** Position, with the reason (unbounded defence surface).
- ❌ **A neural model on the serving path.** The arithmetic rules it out.
- ❌ Search ranking itself (G2) · query understanding for retrieval · answer generation.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the user sees (b):**

```
[ how to tie a|                         ]
   how to tie a tie
   how to tie a tie step by step
   how to tie a bowline knot
   how to tie a scarf
                          Report inappropriate predictions   ← feedback link

[ <a person's name>|                    ]
   (only vetted, non-accusatory completions — or NOTHING)
```

**The metrics:**

```
ACCEPTANCE RATE is the obvious metric and it's INCOMPLETE:

  a suggestion accepted that leads to a bad search is not a win —
  it SAVED KEYSTROKES AND WASTED THE SESSION.

MEASURE THE PAIR:
  · acceptance rate (did they take it?)
  · ⚠ DOWNSTREAM SUCCESS (did the resulting search succeed?)
  · keystrokes saved · time-to-search

⚠ AND THE ASYMMETRY THAT SETS EVERY THRESHOLD:
     empty suggestion list   → costs a keystroke or two
     harmful suggestion      → news story · defamation · real harm
   → SHOWING NOTHING IS A PERFECTLY GOOD OUTPUT.
   → so safety thresholds are set FAR to the conservative side, and
     "coverage" is explicitly NOT a goal that can outvote safety.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Accepted suggestions that led to a successful search** | Saved effort that wasn't wasted |
| Efficiency | Keystrokes saved · time-to-search | The user benefit |
| Offline | MRR of the eventually-issued query, replayed over prefix logs | The ranking gate |
| **Guardrails** | **Harmful-suggestion rate ≈ 0** (audited) · p99 per keystroke · "report" rate | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **⭐ the arithmetic that decides the architecture** `A1b` `I1a` `I1d`

```
A 20-CHARACTER QUERY = 20 REQUESTS.
  → autocomplete QPS ≈ 10–20× SEARCH QPS
  → and the budget is ~50 ms, not ~300 ms

⚠ DO THIS OUT LOUD: at search-scale QPS × 15, with a 50 ms budget,
  you cannot run a neural model per request. Not "it'd be expensive" —
  the fleet doesn't exist.

→ THEREFORE: PRECOMPUTE. The serving path is
     PREFIX → TRIE/FST LOOKUP → small candidate set → cheap scoring
  and the ML lives OFFLINE, in building and scoring the candidate set.

INDEX SIZE: the suggestion corpus is SMALL — tens of millions of
  vetted queries, not billions of documents.
  → ⚠ IT FITS ON ONE MACHINE → REPLICATE, DON'T SHARD (§6)

COST: dominated by the request rate, not compute → edge caching of head
  prefixes is the biggest lever; track cost per million keystrokes
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Candidates | Generate with a model vs **retrieve from vetted real queries** | Generation makes the defence surface unbounded |
| Serving model | A neural ranker per request vs **a thin linear / GBDT scorer** | The QPS × 50 ms arithmetic rules out the neural path |
| Index | Plain trie vs **FST** | FST shares suffixes too: smaller, cache-friendlier |
| Distribution | Shard vs **replicate** | The corpus fits on one machine; sharding adds a hop to a 50 ms budget |
| Personalisation | Personalised retrieval vs **a light re-rank over shared candidates** | Keeps the cache, limits privacy exposure |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE (where all the ML lives) ═════════════════════════════════════════
 query logs → aggregate → ⚠ k-ANONYMITY (distinct TRUSTED actors)
   → spam / bot filtering → SAFETY VETTING (entity rules, policy classifiers)
   → score → build the FST → push ATOMICALLY to every replica
 trending stream → a STRICTER, time-boxed fast lane (§9)

══ ONLINE · per keystroke (≤ 50 ms total) ═══════════════════════════════════
 KEYSTROKE (prefix)
       │
┌─ 1. NORMALISE — case, accents, whitespace, IME / transliteration ──
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. TRIE / FST LOOKUP over the VETTED QUERY SET ───────────────
│ top-N candidates by precomputed score · + a fuzzy path for typos
│ ⚠ ~1–5 ms. This is the whole retrieval.
│ OUTPUT: ~10–20 candidates
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. THIN SCORING LAYER (linear / small GBDT) ──────────────────
│ popularity · recency / trending · location · language · light
│ personalisation (re-rank only)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. ⭐ SAFETY LAYER — a FIRST-CLASS SUBSYSTEM ──────────────────
│ blocklists (exact + pattern) · NAMED-ENTITY rules (person + accusation,
│ group + slur) · a versioned policy taxonomy
│ ⚠ DEFAULT = SUPPRESS. Empty is fine.
│ OUTPUT: suggestions, or NOTHING
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `A1b` `F2b` `F2a`

- **The QPS arithmetic:** *"20 keystrokes per query means 10–20× search QPS at 50 ms, so no model runs per request. Precompute, and keep serving to a lookup plus a linear score."*
- **Trending vs vetted:** *"The most valuable suggestions (breaking news) arrive exactly when they're least vetted. They go through a **stricter fast lane**, time-boxed, with human review for anything entity-adjacent (FU 5)."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Retrieve, never generate** | "A generated suggestion nobody ever typed makes the defence surface unbounded — retrieval keeps the candidate set finite and **reviewable in advance**" | No coverage for genuinely novel phrasings |
| **2** | **Replicate, don't shard** | "The suggestion corpus is small enough to fit on one machine; sharding adds a network hop to a 50 ms budget for no benefit" | Every replica holds the full index; updates fan out |
| **3** | **Suppress by default on any doubt** | "An empty list costs a keystroke; a harmful suggestion costs a great deal. Coverage doesn't get to outvote safety" | Visible gaps, especially on names and breaking news |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Query logs** (aggregated) | The candidate corpus and its popularity | Rare queries are private; spam; self-generated volume |
| **Acceptance + downstream logs** | Which suggestions helped | Position bias; the feedback loop |
| **Typed → corrected pairs** | Data for spell correction | Locale-specific |
| **Entity knowledge base** | People, groups, organisations | Needed for named-entity rules |
| **Policy taxonomy + blocklists** | What may never be suggested | Versioned; locale-specific |
| **"Report prediction" feedback** | User-flagged problems | Can be brigaded |

### 5.2 Building the candidate set (e) `F1c`

**The corpus is query logs — and that has consequences:**
- **⚠ k-anonymity threshold** — a query must have been issued by at least *k* distinct users before it can ever be suggested. This is simultaneously a **privacy** control (no suggesting one person's unique query back to the world) and a **quality** control (filters typos and one-offs). **The single highest-value rule in the offline pipeline.** Count **distinct trusted actors**, not cookies.
- **Spam and bot filtering** — an adversary who can issue queries can try to manufacture a suggestion. Deduplicate by actor, not just by string.
- **Feedback loop** — suggesting a query causes it to be issued more, which raises its score, which suggests it more. **The system creates the popularity it measures.** Score on volume from sessions where the query was **typed, not accepted**, and track the self-generated share.
- **Safety vetting before eligibility:** every candidate passes entity rules and category policy offline, so the online layer is a backstop, not the only defence.

### 5.3 The offline models and their objectives (f) `A2b` `F1a`

| Model | Objective | Note |
|---|---|---|
| **Candidate scorer** | **Pairwise ranking loss** (LambdaMART) on "accepted **and** the search succeeded" | Precomputed into the FST; online only adds context features |
| **Spell correction** | Noisy channel: `P(intended \| typed) ∝ P(typed \| intended) · P(intended)` | The prior comes from the same logs |
| **Risky-pair classifier** | High-recall detection of entity + claim pairings (accusation, medical or legal claim) | Runs offline on every candidate |

**Personalisation:** valuable but it **breaks the shared cache**, so it's applied as a light re-rank over a globally-cached candidate set rather than a personalised retrieval.

### 5.4 Fine-tune an existing model (f) `F2c`

Models earn their place **offline**. A pretrained LLM, fine-tuned with **LoRA** on policy-labelled examples, makes a strong **risky-pair classifier** and catches phrasings a blocklist misses. Tune it for **recall** (anything flagged goes to suppression or human review), because a missed accusation is the failure that defines this system.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · AGGREGATE   logs → counts per query per locale, by distinct trusted actors
STAGE 2 · FILTER      k-anonymity · spam · bots · self-generated volume discounted
STAGE 3 · VET         entity rules · LoRA risky-pair classifier · category policy
STAGE 4 · SCORE       the ranking model → precomputed scores
STAGE 5 · PUBLISH     build the FST → canary replicas → atomic swap everywhere
```

---

## 6 · SERVING + SAFETY *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
⚠ REPLICATE, DON'T SHARD — the opposite of the usual instinct.
  The suggestion index is tens of millions of entries, not billions of
  documents. It FITS. Sharding would add a network hop and a
  scatter-gather to a 50 ms budget, for no capacity benefit.
  → every replica holds the complete trie; scale with MORE REPLICAS.

FST (finite-state transducer) over a plain trie: shares suffixes as
  well as prefixes → a much smaller, cache-friendlier structure,
  which matters when it must be resident on every replica.

CACHING: the prefix distribution is EXTREMELY head-heavy — a small
  set of prefixes covers most traffic → cache aggressively at the
  edge; personalisation is applied AFTER the cached lookup.

UPDATES: rebuild the FST offline, push atomically. Never mutate
  in place — a half-updated index can surface a suggestion that was
  supposed to have been removed.
```

**The safety subsystem — not a filter:**

| Layer | What it does |
|---|---|
| **k-anonymity** | Nothing rare is ever a candidate |
| **Blocklists** | Exact and pattern, versioned like a policy |
| **Named-entity rules** | **The highest-risk slice**: person + accusation, group + slur, individual + medical/legal claim |
| **Category policy** | Self-harm, violence, election-related, health — each with its own action (suppress vs replace vs hard-block) |
| **Manual override, fast** | Ships in **minutes**, not a release cycle — breaking news creates harmful prefixes with no warning |
| **Audit** | Every suppression logged; the blocklist is reviewable and versioned |

- **The ranking objective** `A3b`: rank by P(accepted **and** successful search) **subject to** a near-zero harm threshold; anything above it is suppressed, whatever its score.
- **Efficiency** `F3a` `F3b`: the FST is compact enough to sit in memory on every replica (scores stored in a few bits); edge caching absorbs the head.
- **Reliability** `I1b`: atomic swaps with the last-good index kept; on any failure, **fail to an empty list**, never to unfiltered suggestions.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE      replay real prefix logs: MRR of the query the user finally issued,
             keystrokes saved — per locale
SAFETY AUDIT sampled human review of shown suggestions, weighted to entity-
             adjacent prefixes; every new FST checked before the swap
ONLINE       interleaving or A/B on acceptance AND downstream search success
             ⚠ the surface shapes the queries it's measured on, so read
               downstream success, not acceptance alone
ROLLOUT      canary replicas → atomic fleet-wide swap → instant rollback
```

**Success** = more accepted suggestions that lead to successful searches, keystrokes saved, harmful-suggestion rate at ~0 in audits, and a flat or falling "report" rate. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Self-reinforcement** | Suggested queries get issued more | Score on typed volume; self-generated share; exploration slot |
| **Position** | The top suggestion gets accepted anyway | Interleaving; position-aware evaluation |
| **Over-suppression** | Rules can hide legitimate queries about some groups | Audit suppression rates by entity type and group |
| **Locale** | Rules and data are English-first | Per-locale policies, blocklists and thresholds |
| **Manufactured popularity** | Attacks inflate a query | Trusted-actor counting; velocity anomaly checks |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p99 ≤ 50 ms per keystroke at 10–20× search QPS; cost per million keystrokes | p99 · edge cache hit rate |
| **Reliability** `I1b` | Atomic FST swaps; last-good index; **fail to empty, never to unfiltered**; replicas | Swap failures · fail-safe events |
| **Usability** `I1c` | Typo tolerance; IME and transliteration support; stable lists that don't jump on every keystroke | Keystrokes saved · time-to-search |
| **Scalability** `I1d` | **Replicate, don't shard**; edge caching; a compact FST | QPS headroom per replica |
| **Privacy** `I1e` | **k-anonymity** by distinct trusted actors; personal history only as a light re-rank, with controls; nothing private resurfaced on a shared screen; log retention | k-threshold audits · control usage |
| **Safety** `I1f` | The safety subsystem (§6): named-entity rules, category policy, **suppress by default**, a minutes-fast override | Harmful-suggestion rate (audited) |
| **Security** `I1h` | ⚠ **Manufactured query volume** to plant a suggestion → trusted-actor counting, velocity and network anomaly detection, a longer eligibility window; blocklist integrity (access control, audit) | Planted-suggestion attempts caught |
| **Ethics / bias** `I1i` | **Defamation** risk on named entities; **over-suppression** of legitimate topics about some groups audited; per-locale policies | Suppression rate by group · locale coverage |
| **Explainability** `I1j` | Every suppression logged with **the rule that fired**; the blocklist as a versioned policy artifact; a **"report inappropriate predictions"** link | % suppressions with a logged rule |

> ⭐ *"A suggestion is a publication, so production-readiness here is mostly editorial: a vetted, auditable set, a default of silence, and a way to pull anything within minutes."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **New locales and scripts:** tokenisation, normalisation and **input methods** break first; blocklists and entity rules **don't transfer**. Gate each launch on that locale's own policy, blocklist and k-threshold (FU 9).
- **Entity-aware completion:** completing to a canonical entity ("Paris, France") rather than a string; a richer product, with the entity KB as a new dependency.
- **Zero-prefix suggestions** (before any typing): a different and more sensitive problem, closer to R1's home-screen chips.

**For breaking news, the trending fast lane** `A5b` `F2a`:

```
a query spikes in the last hour
  → a HIGHER k-threshold over a shorter window (trusted actors only)
  → MANDATORY named-entity screening; no personalisation
  → HUMAN REVIEW for anything entity-adjacent (a small, feasible slice)
  → TIME-BOXED elevation: it decays out unless re-confirmed
  → the manual override stays the backstop
  "Being slightly late to a trend is acceptable; being wrong is not."
```

**Top 3 failure modes:**
1. **A harmful suggestion on a named entity** — the news-story failure, and the one that defines the design.
2. **Manufactured suggestions** — an adversary generates query volume to plant a completion. k-anonymity by *actor* is the defence.
3. **Self-reinforcing popularity** — the system suggests, users click, the score rises. It creates the trend it reports.

**Q1 ships:** the **trie/FST serving path with k-anonymity and the safety layer**, plus the fast manual-override path.
> *"Safety is not a later phase here. The failure mode is a news story, and the override path has to exist before the first incident rather than after it."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not use an LLM to generate suggestions?"** `A5a` `A1b`
> Two reasons, and the second is the one I'd lead with. **The arithmetic**: this fires on every keystroke, so autocomplete QPS is roughly 10–20× search QPS with a ~50 ms budget — you cannot run a generative model per request, and that's not a cost objection, it's that the fleet doesn't exist. **The safety argument** is more fundamental: the moment a model composes a suggestion that nobody ever typed, **the space of things I have to defend becomes unbounded**, and no filter is complete against an unbounded generator. Retrieval from a vetted set of real queries keeps the candidate set **finite, auditable, and reviewable in advance** — I can enumerate what could possibly be shown and have humans review the risky slices. That's a property I'd be very reluctant to give up on a surface where a single bad output is a news story. Where a model *does* earn its place is **offline**: scoring candidates, clustering near-duplicates, and detecting risky entity pairs.

**2. "A suggestion is defaming someone. How did that happen and how do you prevent it?"** `I1f` `I1j`
> It happened because people searched it and the system treated frequency as endorsement — and I'd reject "that's what people actually search for" as a defence, because **we chose to surface it**; that's what makes a suggestion a publication rather than a statistic. Prevention is layered: **k-anonymity** so nothing rare surfaces at all; **named-entity rules** as the highest-risk slice — detect person-plus-accusation, group-plus-slur, individual-plus-medical-or-legal-claim patterns and suppress by default; **suppress-by-default on doubt**, since an empty list costs a keystroke and this costs a great deal; and a **fast manual override** that ships in minutes, because breaking news creates harmful prefixes with no warning and a release cycle is far too slow. I'd also make the blocklist a **versioned, auditable policy artifact** with an owner, rather than a config file — because it's a publication policy, and it will be scrutinised as one.

**3. "What's your latency budget and how do you make it?"** `A1b` `F3b`
> **50 ms p99 end-to-end, per keystroke**, and I make it by doing essentially no work at serving time. The path is: normalise the prefix, one **trie or FST lookup** over the vetted query set returning a small candidate set, a **thin scoring layer** — linear or a small GBDT over popularity, recency, location and language — and the safety filter. That's single-digit milliseconds of actual computation. All the ML lives **offline**: building the candidate set, scoring it, vetting it, and compiling the FST. I'd use an **FST rather than a plain trie** because it shares suffixes as well as prefixes, giving a much smaller and more cache-friendly structure — which matters because it has to be resident on every replica. And the prefix distribution is extremely head-heavy, so edge caching absorbs most traffic before it reaches the service at all.

**4. "How do you shard this?"** `A5a` `I1d`
> I wouldn't — and that's the interesting answer. The suggestion index is **tens of millions of vetted queries**, not billions of documents, so it fits comfortably on one machine. Sharding would add a network hop and a scatter-gather to a **50 ms budget** in exchange for capacity I don't need. So: **replicate, don't shard** — every replica holds the complete FST, and I scale with more replicas, which also gives clean failure isolation and trivial load balancing. Updates are handled by **rebuilding the structure offline and pushing atomically**, never mutating in place, because a half-updated index could surface a suggestion that was supposed to have been removed — which on this surface is the failure that matters. It's a nice inversion of the usual instinct, and the reason is that here the corpus is small and the *latency* is the binding constraint, rather than the reverse.

**5. "A trending query is both the most valuable suggestion and the least vetted. How do you handle that?"** `A3b` `F2a`
> That's the hardest trade in the system and I'd name it as such rather than pretend it resolves cleanly. Breaking news produces exactly the queries users most want completed, in exactly the window where we know least about them — and it's also when harmful prefixes appear (a person's name suddenly paired with an allegation). My approach: a **separate, more conservative pipeline for trending candidates** — a higher k-anonymity threshold, mandatory named-entity screening, and no personalisation; **time-boxed elevation** so a trending suggestion decays out automatically rather than persisting if nobody revisits it; **human review in the loop for entity-adjacent trends**, which is a small enough slice to be feasible; and the **fast override** as the backstop. And I'd be explicit about the default: on this surface, **being slightly late to a trend is an acceptable cost and being wrong is not**, so the thresholds sit on the conservative side even though that's visibly worse on a coverage metric.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Someone is deliberately manufacturing query volume to plant a suggestion. Detect it."** `I1h` `F1c`
> This is a real attack and the naive k-anonymity threshold doesn't stop it, because *k distinct users* is easy to fake — so the first fix is to count **distinct actors with trust weighting**, not distinct sessions or cookies, and to deduplicate by inferred identity rather than by request. Then the detection signals: **velocity anomaly**, since organic trends have a characteristic growth curve and manufactured ones are usually too fast or too uniform; **geographic and temporal concentration**, because real interest is distributed and a campaign usually isn't; **behavioural signature** — manufactured queries often have no downstream click, or an identical one; and **network structure**, since the accounts or IPs involved tend to share infrastructure, which is the same graph argument as fraud detection. The mitigation I'd emphasise is economic rather than detective: **raise the cost** by requiring sustained volume across distinct trusted actors over a longer window before a suggestion becomes eligible, which makes the attack expensive without needing to catch every instance. And every confirmed attempt becomes a permanent blocklist entry plus a training example.

**7. "Your system suggests a query, people click it, its score goes up. Isn't that circular?"** `F1c` `A3b`
> Yes, and it's worth stating plainly: **the system creates the popularity it measures.** A suggested query is issued far more often than an unsuggested one, so its score rises, so it's suggested more — the ranking converges on whatever it happened to surface early, and "what people search for" becomes partly an artefact of what we showed them. Three responses. **Damp the feedback** by scoring on volume from sessions where the query was *typed rather than accepted*, which is a cleaner signal of organic intent even though it's a smaller sample. **Measure the self-generated share** — what fraction of a suggestion's volume arrived via the suggestion itself — and treat a high ratio as a reason to discount rather than promote. And **hold an exploration slot** so candidates that would never break into the head get occasional exposure, which is the only way to learn whether they'd have been good. The deeper point I'd make: this surface doesn't just reflect search behaviour, it **shapes** it, and that's an argument for treating the ranking as a product decision with an owner rather than a pure popularity readout.

**8. "How does personalisation interact with your caching story?"** `I1e` `F3b`
> Badly, which is why I'd apply it narrowly. The reason autocomplete is cheap is that the prefix distribution is extremely head-heavy and a **shared cache** absorbs most traffic — but a personalised suggestion is by definition not shareable, so naive personalisation destroys the cache and with it the entire latency and cost argument. So the design is: **retrieve and cache globally**, then apply a **light personalised re-rank** over that cached candidate set — reordering by the user's own history, location and language, which is cheap and preserves the cache. What I'd resist is *personalised retrieval*, where the candidate set itself differs per user, because that's both the expensive version and the one with the worst privacy properties. There's also a safety interaction worth naming: personalisation can resurface a user's own sensitive past queries in a context where someone else is looking at the screen — the same externalised-profile harm as the Discover feed — so a user's private history shouldn't drive a *public-looking* suggestion box without a clear control.

**9. "What breaks when you launch in a new language or script?"** `I1i` `F3c`
> More than the model, and this is where a Western-centric design shows. **Tokenisation and normalisation** assumptions break first: prefix matching is meaningless in a language without spaces in the same way, and scripts with combining characters, multiple normalisation forms, or right-to-left rendering break naive prefix logic. **Input method editors** change the problem entirely — for Chinese or Japanese, the user is typing romanised input that transforms into characters, so "the prefix" is ambiguous and completion has to work at both levels. **Transliteration** matters for languages routinely typed in Latin script. On the safety side it's worse: **blocklists and named-entity rules do not transfer** — slurs, sensitive pairings and legal exposures are locale-specific, so a filter validated in English provides essentially no protection elsewhere, and assuming it does is how the news-story failure happens in a market nobody was watching. So each locale needs its own **policy taxonomy, its own reviewed blocklist, and its own k-threshold**, and I'd gate launch on that rather than on model quality.

**10. "One quarter, and leadership wants coverage metrics up. What do you do?"** `I2b` `A3a`
> I'd deliver coverage where it's safe and push back explicitly on coverage as a top-line target on this surface — and I'd make the argument in their terms rather than as a values objection. The asymmetry is the whole point: **an empty suggestion list costs a keystroke or two; a harmful suggestion costs a news cycle, a legal exposure, and a durable trust hit** — so the expected-value calculation does not support trading safety thresholds for coverage percentage points, and I'd want that written down before the quarter rather than argued during an incident. What I *can* deliver: coverage gains from **better typo tolerance** and **freshness**, which are pure wins with no safety cost, plus **locale coverage** in markets where we're under-serving. What I'd refuse is lowering the k-anonymity threshold or relaxing named-entity suppression, and I'd explain why in terms of the specific failure. The structural point for L6: **coverage is easy to measure and harmful-suggestion rate is a near-zero event**, so the metrics are asymmetric in visibility too — which means the safety threshold needs an owner outside the team measured on coverage, or it erodes one defensible increment at a time.

---

## ⭐ THE L6 LINE

> **"A suggestion is a publication — so I retrieve from a vetted set rather than generating, and showing nothing is a perfectly good output. And the arithmetic (10–20× search QPS at 50 ms) makes this a precomputation-and-trie problem with a thin ML layer, not an ML problem with a cache."**

## 🧭 HOOKS — *steer by what you mention*

- *"A suggestion is a publication"* → invites the safety/policy branch (**good at L6**)
- *"Retrieve, never generate"* → invites the bounded-defence-surface branch
- *"10–20× search QPS"* → invites the capacity/serving branch
- *"Replicate, don't shard"* → invites the distributed-systems branch
- *"The system creates the popularity it measures"* → invites the feedback-loop branch (**good at L6**)
- *"k-anonymity is both privacy and quality"* → invites the privacy branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me redo the QPS arithmetic"* — 20 keystrokes × search QPS regenerates the architecture.
- **Running out of time?** → *"I have five minutes — I'd rather cover the safety subsystem than finish the spell correction. Sound right?"*

**Name and defer:**
- "**Spell correction** is a noisy-channel model — `P(typed|intended)·P(intended)` — with the prior from the same logs."
- "**Entity-aware completion** (completing to a canonical entity rather than a string) is a richer product."
- "**Zero-prefix suggestions** — what to show before any typing — is a different and more sensitive problem."
- "**Query reformulation** at the other end of the session is the sibling problem."
- "**Voice and mobile** change the input distribution materially — fewer keystrokes, more full utterances."
- "**A/B testing here is hard** because the surface shapes the query distribution it's measured on."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: retrieval from a vetted set, no model online | §0, §4 | *"A precomputation-and-trie problem with a thin ML layer, not an ML problem with a cache."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"20 keystrokes = 20 requests: 10–20× search QPS at 50 ms."* |
| **A1c** | Degrade without lowering quality | §6 | *"Fail to an empty list, never to unfiltered suggestions."* |
| **A2a** | The full stack | §4 diagram | Normalise → FST lookup → thin scoring → safety layer; offline vetting |
| **A2b** | Model components | §5.3 | Offline scorer, spell correction, risky-pair classifier |
| **A2c** | Data sources | §5.1 | Query logs, acceptance + downstream logs, entity KB, blocklists |
| **A2d** | Staged pipeline | §5.5 | Aggregate → filter → vet → score → publish atomically |
| **A3a** | Top-down, unprompted | §0 | *"A suggestion is a publication"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 5 | *"Rank by accepted-and-successful, subject to a near-zero harm threshold."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Personalisation · named-entity obligation · freshness · locales |
| **A5a** | Alternatives compared | §4, FU 1, 4 | Generate vs retrieve; neural vs thin scorer; trie vs FST; shard vs replicate |
| **A5b** | A full alternative pipeline on demand | §9 | The trending fast lane |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"LambdaMART on accepted-and-successful; a noisy channel for spelling."* |
| **F1b** | Metrics and their trade-offs | §2 | *"An accepted suggestion that leads to a bad search saved keystrokes and wasted the session."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Self-reinforcement, over-suppression, locale, manufactured popularity |
| **F2a** | Something fails → a fallback | §4, §9 | The trending lane; the manual override |
| **F2b** | Latency too tight → design for it | §4 | The QPS arithmetic, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A LoRA risky-pair classifier tuned for recall |
| **F3a** | Efficiency with a quality trade | §6 | A compact FST with few-bit scores |
| **F3b** | Scale levers | §3, §6 | Replication; edge caching of head prefixes |
| **F3c** | Extension, with cost and maturity risk | §9, FU 9 | New locales and IMEs; entity completion |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p99 per keystroke; cost per million keystrokes |
| **I1b** | Reliability | §6, §8 | Atomic swaps; fail to empty |
| **I1c** | Usability | §2, §8 | Typo tolerance; IME support; stable lists |
| **I1d** | Scalability | §6, §8, FU 4 | Replicate, don't shard |
| **I1e** | Privacy | §5.2, §8 | **k-anonymity** as both privacy and quality |
| **I1f** | Safety | §6, §8, FU 2 | The safety subsystem; suppress by default |
| **I1h** | Security | §8, FU 6 | **Manufactured query volume** |
| **I1i** | Ethics / bias | §7, §8 | Defamation; over-suppression audits |
| **I1j** | Explainability | §8 | The rule behind every suppression; a report link |
| **I2a** | Offline → online | §7 | Replay → safety audit → interleaving → canary swap |
| **I2b** | The success definition | §7, FU 10 | *"More accepted-and-successful searches, harm at ~0."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) candidate set → §5.2 · (f) offline models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
