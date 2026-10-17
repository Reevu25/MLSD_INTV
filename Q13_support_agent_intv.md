# Q13 · Customer Support Agent — **INTERVIEW SHEET**

> **Tier 2 · #15** — the most common applied-agent prompt.
> Full study file: [`../Q13_support_agent.md`](../Q13_support_agent.md) · [diagrams](../Q13_support_agent_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the agent stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving and actions · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | What actions can it take? What's the contact mix? What does a wrongful action cost? (§1) |
| **Explainability** is easy points | Every action logged with **the policy passage that justified it**; citations to customers (§8) |
| **Security** beyond the gateway | **Social engineering via chat** (someone impersonating the customer) and injected instructions (§8) |
| **Troubleshoot before being asked** | Raise **low confidence** (escalate early) and **double refunds** (idempotency) yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Deflection is the wrong north star** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · wrongful-action gate · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The conversation + the handoff · **resolution rate**, repeat-contact anti-metric | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Contact mix → **the well-understood head is the scope** · cost per contact | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Alternatives · intent → verify → retrieve → **tiered actions** → warm handoff | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Resolved transcripts · labels at 7 days · SFT + preference on outcomes | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING: AUTH + ESCALATION** | **(g)** | Capability credentials · compensating transactions · ⚠ **escalation is a product surface** | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Replay · agent-assist → read-only → reversible → financial | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Voice, multilingual, regulated domains · agent-assist fallback · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Let me start with the metric, because it drives everything and it's the most common place this design goes wrong. The obvious north star is deflection rate — the fraction of contacts handled without a human. It's the wrong one, and optimising it produces a predictably bad product: the fastest way to raise deflection is to make it hard to reach a human, so you get a bot that traps people, resolves nothing, and generates a second angry contact the metric doesn't attribute back.**
>
> **I'd optimise resolution rate — the customer's problem is actually fixed, measured by no repeat contact within seven days — with CSAT and escalation quality as co-primary, and deflection as a diagnostic only.**
>
> **Second: this system touches money and accounts, so the design centre is authorization, not answer quality. A chatbot's worst case is a bad answer. This agent's worst case is a wrongful $5,000 refund or an account change on the wrong customer."**

**Then two commitments:**
1. **Blast-radius tiering + capability-scoped credentials + compensating transactions** — *anything the agent does alone is reversible; anything irreversible requires a human.*
2. **⚠ Escalation is a product surface, not a failure.** The most hated support experience is being handed to a human and asked to repeat everything. A **warm handoff** — transcript, what was tried, the diagnosis, verified identity — is where a large fraction of the value lives.

> **Scope it:** *"An agent that resolves the well-understood, high-volume contact types end to end, and escalates everything else early and well. That's a much better product than one that attempts everything."*

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, the customer's experience and the metric, capacity, the architecture, data and training, actions and escalation, evaluation and rollout, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What actions can it take — read-only, or can it refund/cancel/modify?** *(This *is* the design; everything else follows.)*
2. **What's the contact mix?** *(The head of the distribution is the scope; the tail is escalation.)*
3. **What's the cost of a wrongful action vs a missed deflection?** *(They're not symmetric and the org usually only measures one.)*
4. **Which channels and languages?** *(Chat, email, voice; each changes latency and failure modes.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Business | A consumer store with subscriptions (orders, refunds, renewals) |
| Volume | ~50k contacts/day across chat and email; the top ~20 intents are ~75% |
| Actions | Existing support APIs: look up, resend, re-ship, refund, cancel, change address |
| Humans | A support team that receives escalations and owns quality |
| Policy | A knowledge base of refund and returns policy (the source of truth) |
| Languages | Top languages first, with human escalation for the rest |

**FR:** resolve high-volume contact types end to end · **identity verification before any account action** · tiered actions with approval · **warm handoff with full context** · escalate early on low confidence · log every action immutably.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| First response | < 3 s (streaming) |
| **Resolution rate** | The headline — *no repeat contact within 7 days* |
| **Wrongful-action rate** | **~0 on irreversible actions** — hard gate |
| Escalation latency | < 30 s once triggered, **with full context attached** |
| Containment of PII | Never surfaced to the wrong customer — **security-grade** |
| Cost | Cost per **resolved** contact vs a human-handled one |

**Scope / NON-GOALS:**
- ❌ **Deflection rate as the objective.** Position, with the reason.
- ❌ **Attempting every contact type.** Early, good escalation beats broad, bad coverage.
- ❌ Replacing the support org · outbound sales · the knowledge base's *content* (I consume it).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What the customer and the human agent see (b):**

```
CUSTOMER: "I was charged twice for my subscription this month"
AGENT:    verifies identity (a code sent to the email on file)
          "I can see two charges on Oct 3 for $9.99. The second is a duplicate.
           I've refunded $9.99 to your card — it will appear in 3–5 days.
           (Policy: duplicate charges are refunded in full — Billing §2.1)"
          [Talk to a person]   ← always visible, always honoured

WHEN IT ESCALATES, the human sees, before saying a word:
  verified customer · transcript · what was tried · the agent's diagnosis +
  confidence · the policy passage · a suggested next step
  → the customer never repeats themselves
```

**The metrics:**

```
⚠ DEFLECTION IS GAMEABLE BY MAKING THE PRODUCT WORSE.

  fastest way to raise deflection: make the human hard to reach.
  → bot traps the customer, resolves nothing, they contact again
    angrier — and the SECOND contact is not attributed to the failure.

AVAILABLE                  MEANINGFUL
deflection rate  ───────►  RESOLUTION RATE
                           (no repeat contact within 7 days)
handle time                CSAT on the resolved contact
containment                ESCALATION QUALITY (did the human have to
                            re-ask everything?)

⚠ AND TRACK REPEAT-CONTACT RATE AS THE ANTI-METRIC: a deflection that
  produces a contact tomorrow is a NEGATIVE, not a success.
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Resolution rate** (no repeat contact in 7 days) | The problem was actually fixed |
| Co-primary | CSAT on resolved contacts · **escalation quality** (did the human re-ask?) | Experience on both paths |
| Anti-metric | **Repeat-contact rate** | Rising alongside deflection = the trap firing |
| **Guardrails** | **Wrongful-action rate ≈ 0** · PII exposure attempts · time-to-human | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
CONTACT DISTRIBUTION IS HEAVILY HEAD-WEIGHTED:
  "where is my order" · "reset my password" · "cancel my subscription"
  · "why was I charged" · "return this item"
  → the top ~20 intents are typically 70–80% of volume

⚠ SO THE SCOPE IS THE HEAD, DELIBERATELY:
  automate 20 intents end-to-end and escalate the rest well
  > attempt 500 intents at 60% quality.

  and the economics follow: a resolved contact saves a full agent
  handle-time; a BADLY-handled one costs handle-time PLUS the repeat
  contact PLUS the CSAT damage — so a wrong automation is worse than
  no automation, which is the reverse of the usual coverage instinct.

PER-TURN LATENCY (first token < 3 s):
  intent + confidence ~50 ms · account lookup ~200 ms · policy retrieval
  ~200 ms · LLM generation (streamed) ~1–2 s · a tool call ~300 ms
COST: LLM tokens per conversation vs a human handle-time → track cost per
  RESOLVED contact (a cheap unresolved conversation is not a saving)
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Deployment | Full automation of everything vs **narrow end-to-end on the head** vs agent-assist only | Narrow automation of the head, with agent-assist for the rest (§9) |
| Objective | Deflection vs **resolution** | Deflection is gameable by making the product worse |
| Action control | Prompt instructions vs **capability-scoped credentials** | A credential is a guarantee; a prompt is a request |
| Policy answers | Model memory vs **retrieved, cited policy** | A plausible wrong policy stated confidently is the company's statement |

### The system — *draw this in 90 seconds* `A2a`

```
 CUSTOMER MESSAGE
       │
┌─ 1. INTENT + CONFIDENCE ───────────────────────────────────────
│ ⚠ low confidence ⇒ ESCALATE EARLY, not "try anyway"
│ OUTPUT: intent + calibrated confidence (or an escalation)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. IDENTITY VERIFICATION ─────────────────────────────────────
│ ⚠ BEFORE any account-specific read or write; STEP-UP for sensitive
│   changes (address, payment method)
│ OUTPUT: a verified customer id (scopes everything below)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RETRIEVAL — policy docs, KB, account state ────────────────
│ ⚠ ACL-SCOPED TO THIS CUSTOMER (cross-customer leakage is the worst bug)
│ OUTPUT: account facts + the governing policy passage
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. TIERED ACTIONS — capability-scoped credentials ────────────
│ T0 READ account state                         → free
│ T1 REVERSIBLE (resend email, re-issue tracking) → log
│ T2 BOUNDED FINANCIAL (refund under $X, within policy) → log + cap
│ T3 IRREVERSIBLE / LARGE (big refund, closure, deletion) → HUMAN
│ ⚠ + a COMPENSATING TRANSACTION for every T1/T2 action
│ OUTPUT: the action's result (idempotent), or a request for approval
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⭐ WARM HANDOFF (when escalating) ──────────────────────────
│ transcript · what was tried · diagnosis · VERIFIED IDENTITY ·
│ suggested next step — the customer NEVER repeats themselves
└──────┬─────────────────────────────────────────────────────────
       ▼
 RESOLUTION → 7-day repeat-contact check → training signal
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **Low confidence:** *"The worst conversation is a confident attempt at the wrong thing. Below a calibrated confidence threshold, the agent **escalates with a warm handoff** immediately rather than trying."*
- **Double refunds:** *"Tool calls time out and get retried, so every action carries an **idempotency key**: a retried refund is the same refund, never a second one."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Resolution rate, not deflection** | "Deflection is raised fastest by making the human hard to reach — it's gameable by making the product worse" | Lower headline automation numbers |
| **2** | **Narrow scope, early escalation** | "20 intents end-to-end beats 500 at 60%. A wrong automation costs more than no automation" | Visibly lower coverage in month one |
| **3** | **Every autonomous action reversible** | "Anything the agent does alone has a compensating transaction; anything irreversible needs a human" | Some genuinely safe actions still need a human |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Historical transcripts + outcomes** | How good human agents resolve each intent | PII; bad habits copied along with good ones |
| **The policy knowledge base** | The single source of truth for answers | Stale docs are the most common root cause |
| **Account and order APIs** | Live state, scoped to the customer | Must be ACL-scoped |
| **Action logs** | What was done, by whom, with what result | Rare, high-consequence events |
| **Outcomes** | Repeat contact at 7 days, CSAT, escalation notes | Lagging, by design |

### 5.2 Choosing the training set (e) `F1c`

```
1. LABEL BY OUTCOME   a transcript is a GOOD example only if the issue stayed
                      resolved for 7 days AND CSAT was positive — not merely
                      because the chat ended
2. PII                scrub names, addresses, payment details; exclude customers
                      who opted out of data use
3. COVERAGE           stratify by intent, language and channel, so the head
                      doesn't drown the escalation patterns
4. POLICY CURRENCY    drop transcripts that applied a policy since changed
5. SPLIT BY TIME      train on older months, evaluate on recent ones
```

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Component | Model | Objective |
|---|---|---|
| **Intent + confidence** | A small classifier | Multi-class log loss, then **calibrated**, because escalation thresholds use the number |
| **Policy retrieval** | RAG over the KB (Q6) | Recall of the governing passage; answers must cite it |
| **The agent** | An LLM with tool use | Correct tool calls, policy-grounded answers, honest escalation |

### 5.4 Fine-tune an existing model (f) `F2c`

Fine-tune an existing instruction-tuned LLM with **LoRA**: **SFT** on outcome-filtered transcripts (including correct **escalation** turns, so it learns when to stop), then a **preference stage** (DPO) where the preferred response is the one that led to resolution and good CSAT, and the rejected one led to a repeat contact or a wrong action. Policy facts come from retrieval, never from fine-tuning, so a policy change doesn't need a retrain.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · INTENT      pick the next head intent; write its tool scopes and tier
STAGE 2 · DATA        outcome-filtered transcripts for it; policy passages checked
STAGE 3 · TRAIN       LoRA SFT → DPO on outcomes → calibrate the confidence
STAGE 4 · REPLAY      historical contacts replayed; humans review ACTIONS taken
STAGE 5 · ROLL OUT    agent-assist → read-only → reversible → bounded financial (§7)
```

---

## 6 · SERVING: AUTHORIZATION + ESCALATION *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

**Authorization** — *"The design centre is authorization, not answer quality."*
- **Capability-scoped credentials** — the agent holds a token that permits `refund(amount ≤ $X, order_id ∈ this_customer)`. **An injected or confused agent cannot exceed it, because the API rejects it** — not because a prompt told it not to.
- **⚠ Identity verification precedes every account-specific operation.** The worst bug in this system isn't a wrong answer, it's **acting on the wrong customer's account** or surfacing their data.
- **Compensating transactions** for every autonomous action: a refund can be reversed, an email re-sent, a subscription re-instated. If an action has no compensating transaction, it's **T3 by definition**.
- **Rate limits per session and per agent** — catches the runaway loop, which is a bug far more often than an attack.

**Escalation — the value, not the failure:**
```
⚠ THE MOST HATED EXPERIENCE IN SUPPORT: being handed to a human and
  asked to repeat everything.

THE WARM HANDOFF CARRIES:
  · full transcript
  · WHAT WAS ALREADY TRIED (and what the customer already confirmed)
  · the agent's diagnosis + confidence
  · VERIFIED IDENTITY (so the human doesn't re-verify)
  · suggested next action + the relevant policy passage

→ this is what makes the agent a COLLEAGUE to the support org rather
  than a WALL in front of it — which matters politically as much as
  operationally, because the support org can kill this product.

ESCALATE EARLY ON: low intent confidence · detected frustration ·
  any T3 action · repeat contact on the same issue · explicit request
  ⚠ "let me talk to a human" is ALWAYS honoured, immediately.
```

- **The control objective** `A3b`: an action's control is set by **reversibility and amount** (tier), and escalation by **confidence × consequence**, so autonomy is spent only where a mistake can be undone.
- **Efficiency** `F3a` `F3b`: a small, quantized intent classifier on every turn; the LLM only when needed; cached policy retrieval for the head intents.
- **Reliability** `I1b`: if the agent, a tool or the model is unavailable, **route straight to a human with the context so far**; idempotency keys on every action.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline** | **Resolution rate** (no repeat contact in 7 days) |
| **Co-primary** | CSAT on resolved contacts · **escalation quality** (did the human re-ask?) |
| **Hard gate** | **Wrongful-action rate** on financial/account operations — near-zero |
| **Anti-metric** | **Repeat-contact rate** — rising while deflection rises is the trap firing |
| **Panels** | Escalation rate by intent · confidence calibration · action-tier distribution · **PII exposure attempts** · handoff-context completeness |
| **Offline** | Replay historical contacts; **human review of a sample**, especially of *actions taken* |

**Rollout** `I2a`: **agent-assist first** (the agent drafts, a human sends, so every draft is a labelled example) → read-only intents → T1 reversible actions → T2 bounded financial actions, each step gated on the wrongful-action rate of the step before. A/B tests per intent, measured at 7 days.

**Success** = resolution rate up and repeat contacts down **together**, CSAT up on both the automated and the escalated path, wrongful actions at ~0. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Metric gaming** | Deflection rewards trapping customers | Resolution + repeat-contact anti-metric |
| **Copied habits** | Transcripts include bad human practice | Outcome-filtered training data |
| **Unequal treatment** | Refunds or escalations differ by language, dialect or name | Audit outcomes by group |
| **Head bias** | Rare intents and languages handled worse | Early escalation; per-intent metrics |
| **Survivor bias** | Customers who give up never complain | Retention vs a matched cohort (FU 9) |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | First response < 3 s streamed; cost per **resolved** contact vs a human-handled contact | p95 first token · cost per resolution |
| **Reliability** `I1b` | Any agent or tool failure → **immediate warm handoff**; idempotency keys on every action; compensating transactions | Failed-handoff rate · duplicate-action rate |
| **Usability** `I1c` | "Talk to a person" **always** honoured; the warm handoff; plain answers with policy citations | Time-to-human · escalation quality |
| **Scalability** `I1d` | Head intents automated first; human queues routed by intent and language; per-intent rollouts | Automation coverage · queue time |
| **Privacy** `I1e` | Identity verified before account reads; retrieval **ACL-scoped** to the verified customer; PII redacted in logs and training data | Cross-customer exposure (0) · redaction recall |
| **Safety** `I1f` | Wrongful-action gate; no advice beyond policy; vulnerable situations (bereavement, fraud victims) escalated | Wrongful-action rate · sensitive-case escalation |
| **Security** `I1h` | ⚠ **Social engineering via chat** (someone impersonating the customer to redirect a refund or change an address) → **step-up verification** for sensitive changes; injected "ignore policy" instructions made moot by capability credentials; rate limits | Takeover attempts blocked |
| **Ethics / bias** `I1i` | Refunds and escalations **audited by language and group**; accessibility; no dark patterns hiding the human | Outcome parity · accessibility audits |
| **Explainability** `I1j` | Every action logged with **the policy passage that justified it**; citations shown to customers; the handoff summary for humans; an audit trail for disputes | % actions with a cited policy |

> ⭐ *"A support agent spends the company's money and touches customers' accounts, so production-readiness is: verified identity, reversible actions, a human always one tap away, and a policy citation behind every decision."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Voice:** ASR errors become an upstream failure mode (a misheard order number), and the latency budget tightens to conversational speed.
- **Multilingual:** changes the retrieval corpus and escalation routing, not the architecture; quality is uneven by language, so roll out per language.
- **Regulated domains** (health, financial advice): *saying* things becomes the regulated action, so abstention becomes the default and human review moves before sending (FU 10).

**When the agent shouldn't act alone, the fallback pipeline** `A5b` `F2a`:

```
a new intent, low confidence, or a wrongful-action spike on a tier
  → AGENT-ASSIST MODE: the agent drafts the reply AND the proposed action
  → a human reviews, edits, and sends (or rejects)
  → every accept / edit / reject becomes a labelled example
  → that intent graduates back to autonomy only after its accept rate and
    wrongful-action rate clear the bar
```

**Top 3 failure modes:**
1. **The deflection trap** — metric up, repeat contacts up, CSAT down, and the dashboard says success.
2. **Wrongful action** — a refund to the wrong account, or a cancellation the customer didn't ask for. One of these is a headline.
3. **Cold handoff** — the agent escalates but the human starts from zero; the customer experiences the bot as pure delay.

**Q1 ships:** the **top 5 intents, read-only plus T1 reversible actions**, resolution-rate measurement, and the **warm handoff**.
> *"Read-only and reversible first. The financial actions come once the wrongful-action rate on the reversible tier is demonstrated near-zero — and the handoff ships in Q1 because it's what makes the support org an ally rather than an opponent."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why isn't deflection the right metric?"** `F1b` `I2b`
> Because it's **gameable by making the product worse**, and the gaming is easy and invisible. The fastest way to raise deflection is to make reaching a human harder — hide the escalation path, add friction, keep the conversation going — and you get a bot that traps people, resolves nothing, and generates a second contact tomorrow that the metric doesn't attribute back to the failure. So deflection goes up, customer experience goes down, and the dashboard says you're winning. I'd optimise **resolution rate** — the problem is actually fixed, operationalised as no repeat contact within seven days — with **CSAT** and **escalation quality** as co-primary, and I'd track **repeat-contact rate as an explicit anti-metric**. Deflection stays as a diagnostic, because it's useful for capacity planning; it just doesn't get to be the target.

**2. "What can the agent do on its own?"** `A3b` `I1h`
> Anything **reversible**, and nothing that isn't. Concretely: **T0** reads of account state, **T1** reversible operations like resending a confirmation or re-issuing tracking, and **T2** bounded financial actions — a refund under a threshold, within an existing policy, on a verified account. **T3** — large refunds, account closure, data deletion, anything without a compensating transaction — requires a human. The enforcement is **capability-scoped credentials**, not prompt instructions: the agent holds a token permitting `refund(amount ≤ $X, order_id ∈ this_customer)`, so a confused or injected agent that tries to exceed it gets rejected by the API. That's the difference between a request and a guarantee. And the rule that makes the tiering self-defining: **if an action has no compensating transaction, it's T3 by definition** — which is a much more durable criterion than a hand-maintained list.

**3. "The agent can't solve it. Now what?"** `I1c` `A5b`
> Escalate — and treat the escalation as a **product surface rather than a failure**, because the most hated experience in support is being handed to a human and asked to repeat everything. The **warm handoff** carries the full transcript, what was already tried and confirmed, the agent's diagnosis with its confidence, the **verified identity** so the human doesn't re-verify, and a suggested next action with the relevant policy passage. That's where a large fraction of the actual value lives: even on contacts the agent can't resolve, it has done the intake, the verification, and the diagnosis, which is real handle-time saved. It also matters politically — it makes the agent a **colleague to the support org rather than a wall in front of it**, and the support org is entirely capable of killing this product if it makes their job worse. And I'd escalate **early** on low confidence rather than attempting and failing, because a failed attempt costs the customer's patience and the human's context.

**4. "How do you stop it refunding the wrong customer?"** `I1e` `I1h`
> **Identity verification before any account-specific operation**, enforced structurally rather than by convention — the retrieval layer is **ACL-scoped to the verified customer**, so account data for anyone else simply isn't reachable, and the action credentials are scoped to that customer's order IDs. This is the same argument as pre-filtering in retrieval: filtering after the fact means the data was retrieved, and any code path that skips the filter exposes it. Then **compensating transactions** so a mistake is recoverable, **per-session rate limits** to catch runaway loops, and an **immutable audit log** so that after an incident you can answer what else the session touched. I'd also flag the subtler version of this failure, which is **cross-customer PII in the conversation** — the agent summarising something from the wrong record — and that's why retrieval scoping matters more than output filtering.

**5. "How do you know if it's actually working?"** `I2a` `I2b`
> The headline is **resolution rate at seven days**, which is deliberately lagging because that's what makes it honest — a same-session "resolved" flag measures whether the conversation ended, not whether the problem was fixed. Alongside it, **CSAT on resolved contacts** and **escalation quality**, which I'd measure by asking whether the human had to re-ask things already established. The hard gate is **wrongful-action rate** on financial and account operations, which needs to be near-zero and which I'd measure by **human review of a sample of actions taken**, not just conversations — because actions are rare, high-consequence, and invisible in an aggregate CSAT number. And I'd watch the **anti-metric**: repeat-contact rate rising at the same time as deflection is the trap firing, and it's the pattern that tells you the product is getting worse while the primary dashboard improves.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "The support org thinks you're automating them out of a job. How do you handle that?"** `A3a` `I1i`
> I'd take it seriously rather than manage it, because they can kill this product and because the concern is partly correct. The honest framing: this automates the **well-understood head of the contact distribution** — password resets, order status, standard returns — which is the least interesting work and the highest volume, and it makes the remaining contacts **harder on average** because the easy ones are gone. That has a real consequence for the support org I'd raise unprompted: **handle time per contact will go up and that's expected, not a regression**, so if their team is measured on average handle time, that metric needs to change *before* deployment or the numbers will make them look worse for doing more valuable work. Then I'd make them the beneficiaries: the **warm handoff** is designed for them, the agent does intake and verification, and I'd want their escalation feedback to be the primary quality signal — which is both true and makes them the owners of the standard rather than its victims. The thing I'd refuse is to sell it internally as headcount reduction, because that's how you get an organisation that routes around the system.

**7. "A customer is angry and the agent keeps trying to help. What's wrong?"** `I1c` `F2a`
> The **escalation trigger is missing a signal it should have**, and frustration is the clearest one. An agent optimising for resolution will keep attempting, which is exactly wrong when the customer has already decided they want a human — persistence reads as obstruction. So: **detected frustration is an escalation trigger** alongside low confidence and repeat contact, and **an explicit request for a human is always honoured immediately**, with no retention attempt, because a retention attempt at that moment is the single most damaging thing the system can do. I'd also flag the metric interaction, since this is where the deflection trap actually bites: any system measured on containment will have quiet pressure to soften exactly these triggers, one reasonable-looking increment at a time. That's why the escalation triggers need to be an **owned, reviewable artifact** rather than tunable parameters, and why repeat-contact rate sits on the dashboard next to deflection — so the trade is visible rather than buried.

**8. "Your agent gives an answer that contradicts policy. Who's liable?"** `I1j` `A2b`
> Practically, the company — a statement made by our agent to our customer is our statement, and "the model said it" is not a position that survives contact with a regulator or a court. So the design has to make policy adherence **structural rather than behavioural**. Three things: **ground every policy claim in a retrieved passage** with the citation carried through, so an answer that can't cite isn't given; **keep the policy corpus as the single source of truth** rather than allowing the model to answer from parametric memory, which is the main mechanism by which a plausible wrong policy gets stated confidently; and **log the retrieved passage alongside the response** immutably, so after an incident you can determine whether the retrieval was wrong or the generation ignored it — the same attribution split as in RAG, and it decides whether you fix the corpus or the model. I'd also treat **contradicting-policy as a monitored category** with human review, not merely a CSAT signal, because customers usually don't know the policy well enough to complain.

**9. "Six months in, resolution rate is flat but volume is down 30%. Good or bad?"** `I2b` `F1b`
> Ambiguous, and disentangling it is the interesting part — because a 30% volume drop could be the best or worst outcome available. **The good version**: the agent is resolving issues so well that customers don't come back, *and* the upstream product improved because contact analysis surfaced root causes. **The bad version**: customers have learned that contacting support is useless and have given up — **silent churn**, which looks identical on a contact-volume chart and is much worse than the contacts it replaced. The measurements that separate them: **retention and repeat-purchase** among customers who contacted versus a matched cohort who didn't; **CSAT trend** among those who did contact; and **channel shift** — are complaints appearing in app-store reviews, social, or chargebacks instead? The structural point I'd make: **contact volume is an input metric masquerading as an outcome metric**, and a support organisation optimising for low volume has an obvious degenerate solution. What I'd actually want on the wall is **resolution rate plus retention**, because that pair can't be gamed in the same direction.

**10. "Would you use the same agent architecture for a regulated domain — say health or financial advice?"** `F3c` `A5a`
> No, and the reason isn't model quality — it's that the **action tiering and the abstention policy have to be rebuilt around a different cost function**. In consumer support, T2 bounded financial actions are safe because they're reversible and small; in a regulated domain, **giving advice is itself the regulated action**, so the tier that matters isn't "what can it do to the account" but "what can it *say*," and much of what this design treats as free (T0 reads, explanatory answers) becomes T3. Concretely I'd change three things: **abstention becomes the default** rather than a low-confidence fallback, with a much higher bar for answering at all; **every response requires a cited, approved source**, with unsourced generation disabled rather than discouraged; and **human review moves from sampling to pre-publication** for any advice-bearing response, which changes the economics completely and probably means the agent's role narrows to intake, triage and retrieval rather than answering. The general principle I'd state: **the architecture transfers, the thresholds don't** — and the most common failure in moving an agent across domains is porting the tier definitions along with the code.

---

## ⭐ THE L6 LINE

> **"Deflection is gameable by making the product worse — I optimise resolution rate with repeat-contact as the anti-metric. And the design centre is authorization, not answer quality: anything the agent does alone is reversible, and escalation is a product surface rather than a failure."**

## 🧭 HOOKS — *steer by what you mention*

- *"Deflection is gameable by making the product worse"* → invites the metrics/org branch (**good at L6**)
- *"The design centre is authorization"* → invites the agent-safety branch (Q10)
- *"If it has no compensating transaction, it's T3"* → invites the transactional-systems branch
- *"The warm handoff makes us a colleague, not a wall"* → invites the org/change-management branch (**good at L6**)
- *"Ground every policy claim in a retrieved passage"* → invites the RAG crossover (Q6)
- *"Handle time going up is expected, not a regression"* → invites the metric-migration branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the action tiers"* — T0–T3 regenerates the whole design.
- **Running out of time?** → *"I have five minutes — I'd rather cover escalation and the wrongful-action gate than finish the intent model. Sound right?"*

**Name and defer:**
- "**Multilingual support** changes the retrieval corpus and the escalation routing, not the architecture."
- "**Voice** adds ASR error as an upstream failure mode and changes the latency budget."
- "**Proactive support** — reaching out before the customer contacts — is a different product with an uplift problem."
- "**Knowledge-base quality** caps everything; stale policy docs are the most common root cause."
- "**Agent-assist** (human in the loop, agent suggesting) is often the better first deployment than full automation."
- "**Contact-reason analytics** feeding product fixes is where the largest long-run value actually is."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: automate the head, escalate the rest well | §0, §3 | *"20 intents end-to-end beats 500 at 60%."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"First token < 3 s; the top 20 intents are ~75% of volume."* |
| **A1c** | Degrade without lowering quality for everyone | §4, §6 | Escalate early on low confidence; a human on any failure |
| **A2a** | The full stack | §4 diagram | Intent → verify → retrieve → tiered actions → warm handoff |
| **A2b** | Model components | §5.3 | Calibrated intent classifier, policy RAG, a tool-using LLM |
| **A2c** | Data sources | §5.1 | Transcripts + outcomes, the KB, account APIs, action logs |
| **A2d** | Staged pipeline | §5.5 | Intent → data → train → replay → staged rollout |
| **A3a** | Top-down, unprompted | §0 | *"Deflection is the wrong north star"* + the scoping line + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Tier by reversibility and amount; escalate on confidence × consequence."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Actions · contact mix · cost of a wrongful action · channels |
| **A5a** | Alternatives compared | §4, FU 10 | Full vs narrow vs assist; deflection vs resolution; prompts vs credentials |
| **A5b** | A full alternative pipeline on demand | §9 | Agent-assist mode |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Calibrate the intent confidence — the escalation threshold uses it."* |
| **F1b** | Metrics and their trade-offs | §2 | *"Deflection is gameable by making the product worse."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Metric gaming, copied habits, unequal treatment, survivor bias |
| **F2a** | The agent underperforms → a fallback | §4, §9, FU 7 | Early escalation; agent-assist |
| **F2b** | Latency / reliability problems → tiering | §4 | Idempotency keys; escalate rather than retry blindly |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA SFT on outcome-filtered transcripts, then DPO on outcomes |
| **F3a** | Efficiency with a quality trade | §6 | A small quantized classifier per turn; the LLM only when needed |
| **F3b** | Scale levers | §6 | Cached policy retrieval; per-intent rollout |
| **F3c** | Extension, with cost and maturity risk | §9 | Voice, multilingual, regulated domains |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | Cost per **resolved** contact |
| **I1b** | Reliability | §6, §8 | Handoff on failure; idempotency; compensating transactions |
| **I1c** | Usability | §2, §8, FU 3 | "Talk to a person" always honoured; the warm handoff |
| **I1d** | Scalability | §8 | Head-first automation; routed human queues |
| **I1e** | Privacy | §8, FU 4 | ACL-scoped retrieval; identity first |
| **I1f** | Safety | §8 | Wrongful-action gate; vulnerable cases escalated |
| **I1h** | Security | §8 | **Social engineering via chat**; step-up verification |
| **I1i** | Ethics / bias | §7, §8 | Outcome parity by language and group |
| **I1j** | Explainability | §8, FU 8 | The policy passage behind every action |
| **I2a** | Offline → online | §7 | Replay → agent-assist → read-only → reversible → financial |
| **I2b** | The success definition | §7, FU 9 | *"Resolution up and repeat contacts down, together."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
