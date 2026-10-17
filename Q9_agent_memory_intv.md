# Q9 · Agent Memory — **INTERVIEW SHEET**

> **Tier 2 · #19** — personalization + privacy; very common, and one of the places L6 most reliably separates from L5.
> Full study file: [`../Q9_agent_memory.md`](../Q9_agent_memory.md) · [diagrams](../Q9_agent_memory_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the read and write paths · (d) datasets · (e) labels for the write gate · (f) the models and their objectives · (g) serving: conflict, decay, deletion · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Visible and editable? Regulatory posture? Does the agent act on memory? Shared devices? (§1) |
| **Explainability** is easy points | Every memory shows **its source turn and date**; answers say **which memories they used** (§2, §8) |
| **Security** beyond the gateway | **Memory turns prompt injection into a persistent attack** · cross-user leakage (§6) |
| **Troubleshoot before being asked** | Raise **why not long context**, **most turns should write nothing**, and **deletion reaching every copy** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A curation problem, not a storage problem** · the write path is a security boundary | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · write precision ≫ recall · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The memory UI · **the asymmetry** · correction rate · silent-wrong rate | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Tiny per user · **many small partitions, not one big index** · async writes | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Retrieval vs fine-tuning vs long context · ⚠ **the write path is the design** · the record | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Corrections as labels · extractor, gate, detector · **the asymmetric threshold** | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | ⭐ **The present wins** · decay · injection as *persistence* · deletion | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Hard gates in CI · opt-in cohort · **the long-horizon holdback** | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Procedural, shared, on-device memory · **explicit-only fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Let me frame it, because the framing determines what's hard. Memory is not a storage problem, it's a curation problem. Storing every conversation is trivial and it produces a bad product: the context fills with stale, contradictory, low-value material and the model's behaviour gets worse, not better.**
>
> **So the interesting engineering is on the write path — deciding what earns a memory — not the read path, which is retrieval and mostly solved.**
>
> **And the write gate is conservative by design, because a wrong memory is far worse than a missing one: a missing memory means the assistant asks a question; a wrong one means it acts confidently on a false premise, repeatedly, across every future session."**

**Then two more commitments:**
1. **Conflict and staleness** — *"memory is a claim about the past; the current turn is evidence about the present. **When they disagree, the present wins**, and the memory is **updated**, not left to silently coexist with its own contradiction."*
2. **⚠ The security reframe:** *"Memory turns prompt injection into a **persistent** attack. If an attacker gets an instruction written into memory it fires in every future session, invisibly, long after the malicious content is gone. **The write path is a security boundary, not just a quality filter.**"*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the user sees and how we measure it, capacity, the read and write paths, the data and models behind the write gate, conflict, decay and deletion, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Is memory visible and editable by the user?** *(**Yes — non-negotiable.** It changes the entire trust and correction model.)*
2. **What's the regulatory posture?** *(GDPR/CCPA-class ⇒ export, deletion, audit — and that's what makes this **retrieval**, not fine-tuning.)*
3. **Does the agent act on memory, or only converse?** *(Acting on a wrong memory is a blast-radius problem, not a quality one.)*
4. **One person per account, or shared devices and family accounts?** *(Attribution breaks quietly — FU 8.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | A consumer assistant: multi-session chat plus some tool actions (bookings, drafts) |
| Users | ~200k memory-enabled users at launch; design for 100M+ (the per-user shape doesn't change) |
| Memories | Up to **thousands** per heavy user, a few hundred bytes each; most users far fewer |
| Sources | User statements, tool results, and documents or web pages the agent reads |
| Visibility | Every memory visible, editable, deletable, exportable |
| Regulation | GDPR/CCPA-class: access, rectification, erasure, audit |

**FR:** extract candidate memories · **gate writes conservatively** · retrieve relevant memories per turn · **supersede on conflict** · decay/expire · **user-visible, editable, deletable** · export.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Write precision** | **High — deliberately at the cost of recall** |
| Retrieval latency | ≤ 50 ms, inside the turn budget |
| Write latency | **Async, seconds** — the write path is off the turn's critical path |
| **Deletion** | Effective **immediately**, and provable |
| **Injection persistence** | **Zero** — untrusted content can never write a durable memory |
| Cross-user leakage | **Zero** — security-grade, not best-effort |

**Scope / NON-GOALS:**
- ❌ **Storing everything.** Position, with the reason.
- ❌ **Per-user fine-tuning.** *"Weights don't scale per user, update slowly, and — decisively — aren't inspectable or deletable."*
- ❌ Conversation storage/transcripts (different system) · org-wide knowledge (that's RAG, Q6).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the user sees (b):**

```
IN THE CONVERSATION
  User:       Any ideas for dinner on Friday?
  Assistant:  Three vegetarian places near you in Seattle …
              Used 2 memories: vegetarian · lives in Seattle   ← tap to see or edit

SETTINGS → WHAT I REMEMBER ABOUT YOU
  You're vegetarian                   you said so · 11 Mar 2026   [edit] [delete]
  You live in Seattle (was: Boston)   you said so · 2 Sep 2026    [edit] [delete]
  Your daughter's name is Maya        you said so · 5 Jan 2026    [edit] [delete]
  [ Pause memory ]   [ Delete all ]   [ Export ]
  … and "forget that" works in the conversation itself
```

**Visibility is non-negotiable — and it pays three times:**

| | |
|---|---|
| **Privacy/compliance** | Access, rectification, erasure — the legal shape of the thing |
| **Correction signal** | The highest-value training signal available, and free |
| **Poisoning defence** | The only layer with **a human in the loop on persistence** — every automated defence is fallible |

> *"Rare alignment — the privacy-correct choice is also the quality-correct and security-correct one. Take it."*

```
⚠ THE ASYMMETRY THAT SETS EVERY THRESHOLD:

   MISSING memory → the assistant ASKS A QUESTION.  Mildly annoying.
   WRONG memory   → the assistant ACTS CONFIDENTLY ON A FALSE PREMISE,
                    repeatedly, across EVERY future session, and the
                    user may never work out why.

   → WRITE PRECISION ≫ WRITE RECALL. The gate is conservative by design.

METRICS:
  · write precision (sampled human review of what got written)
  · ⭐ CORRECTION RATE — the user fixing/deleting a memory is the
    single highest-value signal in the system: it deletes the wrong
    memory AND raises the write gate's bar for that category.
    ⚠ A correction is FREE LABELLED TRAINING DATA for the gate.
  · retrieval relevance (did the right memory surface?)
  · staleness (share of retrieved memories that are outdated)
  · ⚠ and the one nobody builds: SILENT-WRONG RATE — memories that are
    wrong but never corrected because the user never noticed.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Task success, memory cohort vs a memory-off holdback**, over months | Does memory make the assistant more useful? |
| Direct value | **Repeated-question rate** — asking what it was already told | Memory's most direct benefit |
| Quality | Write precision · correction rate · retrieval relevance · staleness | The write gate and the read path |
| **Guardrails** | Silent-wrong rate (sampled) · **injection persistence = 0** · **cross-user leakage = 0** · memory-off opt-outs | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **the inverted scale story** `A1b` `I1a` `I1d`

```
PER USER: a heavy user accumulates maybe THOUSANDS of memories,
  each a few hundred bytes → LOW MEGABYTES per user.
  → this is a TRANSACTIONAL workload, not a big-data one.

⚠ THE SCALE PROBLEM IS INVERTED vs Q6:
   Q6:  ONE index of 200,000,000 chunks
   Q9:  200,000 SMALL INDEXES of ~2,000 vectors each

   → per-query search is TRIVIAL (thousands of vectors, not billions)
   → the challenge is the NUMBER of tiny partitions, not the size of one
   → and user_id as PARTITION KEY is what makes export, deletion and
     isolation tractable — it's a correctness boundary, not a perf choice

THE TWO PATHS HAVE DIFFERENT BUDGETS:
   READ   one retrieval per turn → read QPS = turn QPS, ≤ 50 ms
   WRITE  ⚠ ASYNC after the turn — it can afford an LLM call, because
          the current turn already has the new fact in its context
   DELETE SYNCHRONOUS — a deleted memory must be gone by the next read

THREE STORES:
  row/document store  → the memory records (transactional, per-user)
  vector index        → embeddings only (embedding_ref on the record)
  append-only log     → the audit obligation

COST: the write path dominates — an LLM extractor per turn would be the
  biggest line → a cheap PRE-FILTER sends only the few turns that might
  hold a durable fact (§5.5); track $ per 1,000 turns
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Where memory lives | Per-user fine-tuning vs a long context window holding all history vs **retrieval over a per-user store** | Weights aren't inspectable or deletable; long context is storing-everything by another name |
| Representation | Prose summaries vs **structured claims** (+ a small prose escape hatch) | Claims can be deduplicated, conflict-checked, shown and deleted precisely |
| What to store | Everything vs **a gated, curated set** | Storing everything degrades behaviour |
| Updates | Overwrite vs **supersede with history** | Explainable and reversible |
| Prompt placement | As system-prompt instructions vs **as dated evidence** | A poisoned memory then reads as a claim, not a command |

### The system — *draw this in 90 seconds* `A2a`

```
══ READ PATH · per turn, on the critical path (≤ 50 ms) ═════════════════════
 CONVERSATION TURN
       │
┌─ 1. RETRIEVE — from THIS user's partition only ────────────────
│ TASK: rank this user's CURRENT memories (superseded ones filtered
│       out) against the turn: similarity + confidence + freshness
│ OUTPUT: top ~5–20 memories, each with its source and date
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. INJECT AS DATED EVIDENCE, NOT INSTRUCTIONS ────────────────
│ "On 2026-03-11 the user said they are vegetarian."
│ ⚠ never in the system prompt · inside its own token budget
│ OUTPUT: the model's context for this turn
└────────────────────────────────────────────────────────────────

══ WRITE PATH · after the turn, async — ⭐ WHERE THE DESIGN LIVES ═════════════
┌─ 3. EXTRACT ───────────────────────────────────────────────────
│ TASK: a cheap pre-filter, then an LLM turns the turn into
│       STRUCTURED CLAIMS (subject · predicate · object)
│ OUTPUT: 0–3 candidates — most turns produce NONE
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. ⚠ SOURCE_TRUST CHECK ──────────────────────────────────────
│ only USER STATEMENTS create durable semantic facts; tool results
│ and documents create EPISODIC, attributed memories
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⚠ INSTRUCTION DETECTOR ────────────────────────────────────
│ imperative content from a non-user source → REJECT
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. CONSERVATIVE WRITE GATE — precision ≫ recall ──────────────
│ TASK: durable? about the user? worth keeping? allowed category?
│ OUTPUT: write · drop (the default on doubt) · ask (sensitive)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 7. CONFLICT CHECK against existing memories ──────────────────
│ same subject + predicate, different object → SUPERSEDE (keep the
│ history) · high-stakes or ambiguous → ASK the user
└──────┬─────────────────────────────────────────────────────────
       ▼
 STORE — records partitioned by user_id · vector index · append-only audit log
       ▼
 ⭐ USER-VISIBLE MEMORY UI — see · edit · delete · export (§2)
```

### The record — *the fields to defend* `A2c` `I1e`

```json
{ "memory_id":"m_7f3a", "user_id":"u_881",     // ⚠ PARTITION KEY = isolation boundary
  "type":"semantic|episodic|procedural",
  "claim":{"subject":"user","predicate":"dietary_restriction","object":"vegetarian"},
  "text":"The user is vegetarian.",            // the injectable surface form
  "confidence":0.9,
  "provenance":{ "source":"user_statement|inferred|tool_result|document",
                 "source_trust":"trusted",     // ⚠ THE SECURITY FIELD
                 "session_id":"s_44","turn":12,
                 "created":"...","last_confirmed":"..." },
  "lifecycle":{ "decay_rate":0.02,"access_count":14,
                "superseded_by":null,"supersedes":["m_2c19"] },
  "privacy":{ "category":"dietary","sensitivity":"medium","user_visible":true } }
```

**Three fields worth ten seconds each:**
- **`source_trust`** — *"the security field. A memory sourced from a web page or tool result is **untrusted**, and untrusted memories can never carry instructions — only attributed facts. That single field is most of the poisoning defence."*
- **`supersedes` / `superseded_by`** — an update **history**, not an overwrite. Explainable and reversible.
- **`last_confirmed` separate from `created`** — *"what makes decay meaningful. A fact stated once two years ago and one reconfirmed last week are very different, and a single timestamp can't distinguish them."*

### ⚠ Raise these yourself, before the interviewer does `F2b` `A5a` `I1e`

- **Why not long context?** *"With million-token windows someone will ask why not paste the whole history in. Cost grows with history on every turn, and it's storing-everything by another name: the stale and the contradictory still reach the model, and nothing can be shown to the user or deleted as a fact."*
- **Most turns should write nothing:** *"The default on the write path is drop. If the store grows with every turn, the gate is broken."* And the write path runs **after** the turn, so its LLM calls never touch latency.
- **Deletion reaches every copy:** the record, its embedding, caches, the audit log, training sets and backups (§6).

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Retrieval, not per-user fine-tuning** | "Weights don't scale per user, update slowly, risk catastrophic forgetting — and decisively, **aren't inspectable or deletable**, which the product and the regulation both require" | Retrieval must fit in the context budget |
| **2** | **Structured claims, not prose** | "Prose memories can't be deduplicated, conflict-checked, shown to a user comprehensibly, or deleted precisely. A structured claim does all four" | Extraction is harder; keep a small unstructured escape hatch |
| **3** | **Supersede, don't overwrite** | "'Actually I moved to Seattle' — I don't delete Boston, I supersede it, so I can explain the change and roll it back if the correction was itself wrong" | Storage grows; retrieval must filter to current |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Conversation turns** (consented) | The raw material for extraction | Full of PII and **other people's** information |
| **Corrections, edits and deletions** from the memory UI | Free labels: a deleted or corrected memory was a bad write | Covers only the errors users **noticed** |
| **Sampled human review** of written memories | Direct write-precision labels | Reviewers see personal data: consent, redaction, access control |
| **Synthetic change scenarios** | "Moved Boston → Seattle", hypotheticals, quoted friends | Synthetic phrasing isn't real phrasing |
| **Poisoning red-team corpus** | Instruction-detector training and the CI gate | Must grow with every new attack |
| **Category and sensitivity policy** | What may be written, and how fast it decays | Versioned; set by policy, not learned |

### 5.2 Labels for the write gate (e) `F1c`

- **Positives:** memories the user kept, that were later **reconfirmed** (`last_confirmed` moved) or used without correction.
- **Negatives:** memories the user **deleted or corrected** — the gold — plus reviewer rejections.
- **Hard negatives to include deliberately:** hypotheticals ("if I were vegetarian…"), **other people's facts** ("my friend is vegan"), sarcasm, role-play, and instructions embedded in documents.
- **⚠ The bias in the labels:** corrections cover only errors users noticed, which skews toward memories that surface often. Add a **sampled review of rarely-retrieved memories**, the silent-wrong slice.
- **Sensitive categories** (health, relationships, finances) are **policy-gated**, never learned into the default write behaviour.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective and threshold |
|---|---|---|
| **Pre-filter** | Could this turn hold a durable fact? | Small classifier, tuned so the extractor sees only a small fraction of turns |
| **Extractor** | Turn → structured claims | An LLM with schema-constrained output; measured on claim-level precision |
| **Write gate** | write / drop / ask, per candidate | ⭐ **Write only when P(correct ∧ durable) > C_wrong / (C_wrong + C_missing).** A wrong memory costing ~10× a missing one puts the bar at **p > ~0.9**, set per category |
| **Instruction detector** | Imperative content from a non-user source? | Tuned for **recall** — reject on doubt |
| **Conflict check** | Does a new claim contradict a stored one? | Structured first: same subject + predicate, different object; an NLI model only for the prose escape hatch |
| **Retriever** | Which memories matter for this turn? | Similarity × confidence × decayed freshness, over current memories only |

### 5.4 Fine-tune an existing model (f) `F2c`

LoRA-tune **one shared** extractor-and-gate model from a base LLM on the correction data and reviewed examples, then distil it to a small model, since it runs on every pre-filtered turn. Say the distinction out loud: *"This is one model trained on aggregated, consented labels, not per-user weights, which stay a non-goal. Training data has a retention window, so a deletion leaves the training set at the next refresh."*

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · PRE-FILTER  cheap classifier: could this turn hold a durable fact? (most don't)
STAGE 2 · EXTRACT     LLM → structured claims, schema-constrained
STAGE 3 · TRUST       source_trust · instruction detector
STAGE 4 · GATE        conservative write gate, per-category thresholds
STAGE 5 · RECONCILE   conflict check → supersede · ask · write
STAGE 6 · STORE       record + embedding + audit entry → visible in the UI at once
```

---

## 6 · SERVING — CONFLICT, DECAY, INJECTION, DELETION *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b` `I1h`

```
⭐ THE CONFLICT RULE — say it as a rule:
   memory says X. This turn says Y. X ≠ Y.
   → THE PRESENT WINS.
      · supersede the memory, keep the history
      · if high-stakes or ambiguous, ASK: "I had you down as X —
        has that changed?"
      · ⚠ NEVER let both coexist silently — that's how the assistant
        becomes randomly inconsistent depending on which memory
        retrieval happens to surface.

DECAY: not all facts age alike.
   "is vegetarian"    → slow decay, stable preference
   "is in Lisbon"     → fast decay, obviously transient
   "kid's name"       → no decay
   → decay_rate per CATEGORY, and last_confirmed resets it.

⚠ INJECTION AS PERSISTENCE — the security reframe:
   ordinary injection is TRANSIENT. Injection that WRITES A MEMORY
   fires in EVERY future session, invisibly, long after the content
   is gone.
   DEFENCES: source_trust (only user statements → durable facts) ·
     instruction detector on the write path · inject memories as
     DATED EVIDENCE not instructions · USER-VISIBLE MEMORY (a human
     in the loop on persistence) · a poisoning red-team suite as a
     HARD GATE in CI.

DELETION — immediate, and provable:
   user deletes → TOMBSTONE synchronously (the next read can't see it)
   → purge the embedding and any cache that holds it
   → the audit log keeps IDs, events and hashes, never plaintext,
     so the log itself doesn't defeat erasure
   → content encrypted under a PER-USER KEY: "delete everything" =
     destroy the key, which reaches backups too (crypto-shredding)
   → out of the training set at the next refresh (§5.4)
```

- **The trade-off, as a rule** `A3b`: write only when P(correct ∧ durable) clears C_wrong / (C_wrong + C_missing); retrieve by relevance × confidence × decayed freshness; on conflict, the present wins.
- **Efficiency** `F3a`: the pre-filter keeps the LLM extractor to a small fraction of turns, and a distilled model serves it. The trade is a little recall, which this design spends deliberately. Per-user search is **exact, not ANN**: ~2,000 × 768 multiply-adds take tens of microseconds, so there's no per-user index to build or tune.
- **Scale** `F3b`: partition by user_id; memory grows with the number of users, not the size of any one index.
- **Reliability** `I1b`: memory is an enhancement, so if the store is down the turn is answered **without memory**, never failed. Writes are idempotent and queued; supersession history makes any bad write reversible.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE      extraction precision on labelled conversations · write precision on a
             reviewed held-out set · conflict scenarios (moves, hypotheticals,
             other people's facts) · retrieval relevance · staleness
HARD GATES   ⚠ poisoning red-team suite · cross-user leakage tests run AS DIFFERENT
 (in CI)       USERS (security-grade, like ACLs in Q6) · deletion tests (a deleted
               memory is never retrievable again)
ONLINE       dogfood → opt-in cohort → A/B: correction rate, repeated-question
             rate, memory-off opt-outs, "creepy" reports
LONG-HORIZON a memory-off holdback for months: task success, abandonment, retention
```

**Success** = the memory cohort beats the memory-off holdback on task success over months, with fewer repeated questions, a falling correction rate, and zero leakage or injection persistence. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Correction-label bias** | Only noticed errors get labelled | Sampled review of rarely-surfaced memories |
| **Inference and stereotyping** | Inferring gender, religion or health from behaviour | No inferred memories in sensitive categories; stated outranks inferred |
| **Language and dialect** | A weaker extractor in some languages writes more wrong memories for those users | Precision sliced per language; stricter thresholds where it's lower |
| **Heavy-user skew** | Heavy users dominate the labelled set | Stratify review by user activity |
| **Demo bias** | Memory demos well and can be net-negative in sustained use | The long-horizon holdback |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Retrieval ≤ 50 ms inside the turn; the write path async; extractor cost held down by the pre-filter | p99 retrieval · $ per 1,000 turns |
| **Reliability** `I1b` | Store down → answer **without memory**, never fail the turn; idempotent, queued writes; supersession makes bad writes reversible | Turns served without memory · write lag |
| **Usability** `I1c` | The memory UI: see, edit, delete, export, pause; "forget that" in conversation; **ask** on high-stakes conflicts | Correction rate · opt-outs |
| **Scalability** `I1d` | Partition by user_id; tiny exact-search indexes; scales with users, not with any one index | Partitions per node · tail latency |
| **Privacy** `I1e` | Visible, editable, deletable, exportable; **deletion synchronous and provable** (tombstone, purge, crypto-shredding); nothing sensitive written without an explicit statement | Deletion latency · sensitive writes |
| **Safety** `I1f` | Acting on memory is a blast-radius problem: a **high-stakes action** (pay, send, book) **confirms** rather than acting on a memory alone | Actions taken on wrong memories |
| **Security** `I1h` | ⚠ **Injection as persistence**: source_trust, instruction detector, dated evidence, red-team hard gate; **user_id isolation** on every query path, tested as different users | Red-team pass rate · leakage = 0 |
| **Ethics / bias** `I1i` | **Stated over inferred**; no stereotyping inference in sensitive categories; shared devices and family accounts (FU 8) | Inferred-memory share · "creepy" reports |
| **Explainability** `I1j` | Every memory shows **its source turn and date**; answers **cite the memories they used**; the supersession history explains every change | % answers with attribution |

> ⭐ *"Memory is the one feature where the privacy-correct, quality-correct and security-correct choices are the same: make it visible."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Procedural memory** (learned workflows rather than facts): a different write gate, because a learned habit can encode a mistake. It needs explicit confirmation before it changes behaviour.
- **Multi-agent and team memory:** attribution and authority across agents, and an ACL problem; it moves toward RAG.
- **On-device memory:** a privacy gain, but sync, deletion and cross-device consistency get harder.
- **Compaction over years** of history: hierarchical summaries, which must stay deletable fact by fact.

**When the write gate can't be trusted — the explicit-only fallback** `A5b` `F2a`:

```
write precision drops in review, or a poisoning incident is found
  → FREEZE automatic semantic writes (episodic, session-scoped only)
  → EXPLICIT-ONLY mode: "remember that …" plus a confirmation
  → QUARANTINE memories written since the last good gate version;
    re-review them before they can be retrieved again
  → existing memories stay readable, visible and deletable
  "Memory degrades to explicit, never to unvetted."
```

The same move covers **uncertain identity** (a shared device): write nothing durable, keep context to the session.

**Top 3 failure modes:**
1. **Confident wrong memory** — acted on repeatedly, and the user can't work out why the assistant believes it.
2. **Persistent injection** — the attack that survives the session and is invisible.
3. **Silent contradiction** — two conflicting memories coexist, and behaviour depends on which one retrieval surfaced.

**Q1 ships:** the **conservative write gate** + **structured schema with `source_trust` and supersession** + the **user-visible memory UI**.
> *"The UI in Q1 even though it looks like a later-phase feature — it's the privacy obligation, the correction signal, and the poisoning defence in one, which is the best return available."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why is the write path harder than the read path?"** `A1a` `F1b`
> Because reading is retrieval, which is a solved problem at this scale — a few thousand vectors per user makes ANN trivial. The difficulty is deciding **what earns a memory**, and the reason is an asymmetry: a **missing** memory means the assistant asks a question, which is mildly annoying; a **wrong** memory means it acts confidently on a false premise, repeatedly, across every future session, and the user may never work out why it believes that. So write precision matters far more than write recall, and the gate is conservative by design — I'd rather store a third of what I could and be right, than store everything and be occasionally, persistently wrong. Storing everything is the trivial version and it produces a worse product: context fills with stale, contradictory, low-value material and behaviour degrades.

**2. "What happens when a memory contradicts what the user just said?"** `A3b` `I1j`
> **The present wins** — memory is a claim about the past, the current turn is evidence about the present. Mechanically I **supersede** rather than overwrite: the old memory is retained with a `superseded_by` pointer, so I can explain the change if asked and roll it back if the correction was itself wrong. If the conflict is high-stakes or genuinely ambiguous, the right behaviour is to **ask** — "I had you down as vegetarian, has that changed?" — which is cheap and builds trust. What I'd never do is let both coexist silently: that's how the assistant becomes **randomly inconsistent depending on which memory retrieval happens to surface**, which is the most confusing possible failure because it's non-deterministic from the user's point of view and impossible for them to diagnose.

**3. "Why not just fine-tune a small adapter per user?"** `A5a` `I1e`
> Four reasons, and the last is decisive. Per-user **weights don't scale** — you'd be training and serving millions of adapters. They **update slowly**, so a correction the user made this morning doesn't take effect until the next training cycle, which fails the correction loop entirely. They risk **catastrophic forgetting** of general capability. And critically, **weights are not inspectable and not deletable** — you cannot show a user what the adapter believes about them, and you cannot remove a single fact from it on request. Since the product requires visible, editable memory and the regulation requires export and erasure, fine-tuning is **structurally non-compliant**, not merely inconvenient. Retrieval is inspectable, editable, and instantly deletable, which the product needed anyway — so the architecture is chosen by the requirement, not by performance.

**4. "How does memory change your security model?"** `I1h` `A2a`
> It turns prompt injection from a **transient** attack into a **persistent** one, which is a change in kind. Ordinary injection affects one session; injection that succeeds in **writing a memory** fires in every future session, invisibly, long after the malicious content is gone — and the user has no reason to suspect anything. So the write path is a **security boundary**, not a quality filter. Layered defences: **`source_trust`** — only user statements can create durable semantic facts, while tool results and document content create episodic, clearly-attributed memories that are never instructions; an **instruction detector** that rejects imperative content from non-user sources outright; **injecting memories as dated evidence rather than system-prompt instructions**, so a poisoned memory reads as a claim rather than a command; **user-visible memory**, which is genuinely strong because it puts a human in the loop on persistence; and a **poisoning red-team suite as a hard gate in CI**.

**5. "The user says memory is creepy. What did you get wrong?"** `I1i` `I1c`
> Usually one of three things, and they have different fixes. **Inferred rather than stated** — a memory the user never volunteered, derived from behaviour, which feels like surveillance even when it's accurate; the fix is to weight `source: user_statement` far above `inferred` for anything the assistant will surface. **Sensitive category** — health, relationships, finances — where an accurate inference can still be a harm, which is the same externalised-profile problem as the Discover feed, and the fix is category-level suppression regardless of confidence. Or **no visible control**, which is the one I'd fix first: a system that shows you what it thinks about you owes you a way to correct it, and the absence of that control is what converts "personalised" into "creepy." The general principle: **visibility is what makes memory feel like a tool rather than a dossier**, which is why I ship it in Q1 rather than treating it as a settings page for later.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "How do you know memory is actually helping, and not just feeling personal?"** `I2b` `I2a`
> With difficulty, because the obvious metrics measure the wrong thing — retrieval relevance tells you the memory system works, not that the **product** is better. What I'd want is a **long-horizon holdback**: a population with memory disabled or heavily restricted, compared over months on task success, session count, and whether users complete things they previously abandoned. That's slow and it's the only instrument that separates "the assistant remembers my name" from "the assistant makes me more effective." Alongside it, two cheaper proxies: **reduction in repeated questions** — is the assistant asking things it was told before? — which is memory's most direct value, and **correction rate trend**, because a falling correction rate means the write gate is improving. And I'd state the risk that motivates all this: memory is a feature that **demos extremely well and can be net-negative in sustained use** if write precision is poor, since a wrong durable belief compounds across sessions while a good one is invisible.

**7. "A memory is wrong and the user never notices. How would you ever catch that?"** `F1c` `F1b`
> This is the **silent-wrong rate**, and it's the failure nobody instruments because corrections only tell you about the errors users *caught*. Three approaches, none complete. **Sampled human review** of written memories against their source conversation — expensive, but it's the only direct measurement of write precision and I'd fund a small continuous sample rather than a one-off audit. **Consistency probes**: periodically check retrieved memories against current conversational evidence, and flag ones that contradict recent turns without having been superseded — a memory that's *never* confirmed despite many relevant opportunities is suspect. And **confidence-stratified review**, concentrating human attention on low-confidence and `inferred`-source memories, which is where errors concentrate. I'd also treat **`last_confirmed` going stale** as a signal in itself: a belief that has had many chances to be reconfirmed and never was should decay out rather than persisting indefinitely on the strength of one old statement.

**8. "Two users share a device, or a family account. What breaks?"** `I1e` `F2a`
> Attribution, and it breaks quietly — memories from one person get written under another's identity, which is simultaneously a **quality failure** (the assistant believes contradictory things and behaves inconsistently) and a **privacy breach** (one person's inferred facts surface to another). It's worse than a normal cross-user leak because there's no boundary violation to detect: the writes are legitimate under the account. Mitigations, in order of how much I'd trust them: **explicit profile switching**, which is the only reliable answer and is a product decision rather than an ML one; **confidence penalties on writes when the session shows signs of a different speaker** — vocabulary, topic discontinuity, device context — which is weak evidence and shouldn't gate on its own; and **sensitivity-aware defaults**, so high-sensitivity categories require a stronger identity signal before they're written at all. The design stance I'd take: **when identity is uncertain, don't write durable semantic facts** — degrade to episodic, session-scoped context, which fails safe.

**9. "Your memory store and the org's RAG corpus both feed context. How do they interact?"** `A2c` `I1h`
> They need to stay **architecturally separate with different trust levels**, and conflating them is a real mistake I'd flag. Memory is **user-scoped, user-editable, and semantically authoritative about the user** — "you are vegetarian" is a claim I should act on. The RAG corpus is **org-scoped, not user-editable, and authoritative about the world** — "the refund policy is 30 days." They fail differently and they need different provenance handling: a document retrieved from the corpus is `source_trust: untrusted` for memory-writing purposes, which is exactly the rule that stops a poisoned document becoming a durable belief about the user. The interaction that matters is **precedence**: when a memory and a retrieved document conflict, the document usually wins on facts about the world and the memory wins on facts about the user — and where that's ambiguous the honest behaviour is to surface both with attribution rather than silently pick. I'd also keep their **retrieval budgets separate**, so a long document can't crowd out the memories that make the response personal.

**10. "One quarter. Memory or no memory?"** `I2b` `I1f`
> I'd ship a **narrow, high-precision version** rather than either extreme, and I'd be explicit that the narrow version is the product decision rather than a compromise. Concretely: only **explicit user statements** as a source — no inference — restricted to a **small set of low-sensitivity categories** like preferences, names and stated constraints; **structured claims with supersession**; and the **user-visible UI from day one**. That's a memory system that is obviously useful, very unlikely to be embarrassing, and generates the correction data needed to widen the gate later. What I'd refuse to ship in that quarter is **inference-based memory across sensitive categories**, because that's where both the creepiness failure and the confident-wrong failure live, and getting it wrong early poisons user trust in the feature permanently — you don't get a second first impression on something that feels like surveillance. And I'd ship the **poisoning red-team gate** in the same quarter, not later, because persistence makes that failure much more expensive to discover in production than in CI.

---

## ⭐ THE L6 LINE

> **"Memory is a curation problem, not a storage problem — the write path is where the design lives, and it's a security boundary because memory turns injection into a persistent attack. A wrong memory is far worse than a missing one, so precision beats recall."**

## 🧭 HOOKS — *steer by what you mention*

- *"The write path is where the design lives"* → invites the extraction/gating branch
- *"Memory turns injection into a persistent attack"* → invites the security branch (Q10/Q19)
- *"The present wins"* → invites the conflict/consistency branch
- *"Weights aren't inspectable or deletable"* → invites the privacy/architecture branch (**good at L6**)
- *"Visibility pays three times"* → invites the product/org branch (**good at L6**)
- *"200,000 small indexes, not one big one"* → invites the systems/scale branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the write path"* — extract → trust → gate → conflict regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the injection-persistence problem than finish the retrieval detail. Sound right?"*

**Name and defer:**
- "**Multi-agent shared memory** raises an attribution and authority problem this design doesn't solve."
- "**Procedural memory** — learned workflows rather than facts — has a different write gate and different risks."
- "**Memory summarisation/compaction** over years of history is a real problem I'd handle with hierarchical summaries."
- "**Cross-session context budgeting** interacts with RAG for the same token budget."
- "**Federated / on-device memory** changes the privacy calculus and the deletion story materially."
- "**Memory for the org rather than the user** — team-level shared context — has an ACL problem attached."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a simple read path, the work on the write path | §0, §4 | *"Memory is a curation problem, not a storage problem — the design lives on the write path."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"Retrieval ≤ 50 ms inside the turn; writes async; deletes synchronous; low MB per user."* |
| **A1c** | Degrade without lowering quality | §6, §9 | *"Store down → answer without memory; gate in doubt → explicit-only."* |
| **A2a** | The full stack | §4 diagram | Retrieve → inject as evidence; extract → trust → detector → gate → conflict → store → UI |
| **A2b** | Model components | §5.3 | Pre-filter, extractor, write gate, instruction detector, conflict check, retriever |
| **A2c** | Data sources | §4 record, §5.1 | Turns, corrections, reviewed samples, change scenarios, red-team corpus |
| **A2d** | Staged pipeline | §5.5 | Pre-filter → extract → trust → gate → reconcile → store |
| **A3a** | Top-down, unprompted | §0 | *"A curation problem"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.3, §6 | *"Write only when P(correct) > C_wrong / (C_wrong + C_missing) — about 0.9."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Visible? Regulation? Acts or converses? Shared devices? |
| **A5a** | Alternatives compared | §4, FU 3 | Fine-tuning vs long context vs retrieval; prose vs claims; overwrite vs supersede |
| **A5b** | A full alternative pipeline on demand | §9 | Explicit-only memory, with quarantine |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"A precision-thresholded gate, a recall-tuned instruction detector, structured comparison for conflicts."* |
| **F1b** | Metrics and their trade-offs | §2 | *"A missing memory asks a question; a wrong one acts on a false premise in every session."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Correction-label bias, inference and stereotyping, language slices, demo bias |
| **F2a** | Something fails → a fallback | §9, FU 8 | Explicit-only mode; nothing durable when identity is uncertain |
| **F2b** | Latency too tight → design for it | §3, §4 | The write path moved off the turn, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | One shared LoRA-tuned extractor and gate on consented labels — not per-user weights |
| **F3a** | Efficiency with a quality trade | §6 | Pre-filter + distilled extractor; exact search instead of per-user ANN |
| **F3b** | Scale levers | §3, §6 | user_id partitioning; many tiny partitions |
| **F3c** | Extension, with cost and maturity risk | §9 | Procedural, team and on-device memory; compaction |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p99 retrieval; extractor $ per 1,000 turns |
| **I1b** | Reliability | §6, §8 | Answer without memory; idempotent writes; reversible supersession |
| **I1c** | Usability | §2, §8 | The memory UI; "forget that"; ask on high-stakes conflicts |
| **I1d** | Scalability | §3, §8 | Partition by user_id |
| **I1e** | Privacy | §6, §8, FU 3 | Immediate, provable deletion; crypto-shredding; nothing sensitive inferred |
| **I1f** | Safety | §8, FU 10 | Confirm before a high-stakes action that rests on a memory |
| **I1h** | Security | §6, §8, FU 4 | **Injection as persistence**; cross-user leakage tested as different users |
| **I1i** | Ethics / bias | §7, §8, FU 5 | Stated over inferred; no stereotyping inference; shared devices |
| **I1j** | Explainability | §2, §8 | Source and date on every memory; answers cite the memories used |
| **I2a** | Offline → online | §7 | Offline → CI hard gates → opt-in cohort → A/B → long-horizon holdback |
| **I2b** | The success definition | §7, FU 6 | *"Beats the memory-off holdback on task success, over months."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) read and write paths → §4 · (d) data → §5.1 · (e) gate labels → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
