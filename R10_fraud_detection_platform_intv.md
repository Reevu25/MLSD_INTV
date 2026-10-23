# R10 · A Fraud Detection System — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md)–[R9](R9_internal_doc_search_agent_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **Expect these probes:** (a) clarifying questions · (b) the goal and the actions · (c) data sources and features, real-time and batch · (d) model choice · (e) cold start · (f) class imbalance · (g) the learning loop with delayed labels · (h) evaluation, offline and online · (i) architecture for training, serving and monitoring · (j) adversarial abuse and model exploitation.
>
> **What's reused, and what's new.** [G29](G29_fraud_detection_intv.md) is the deep dive on **payment** fraud, and R10 reuses its core machinery by reference: the **cost-matrix decision rule** (G29 §2), **PR-AUC over ROC-AUC**, **"you never learn about your blocks"** and the random allow-through (G29 §5.2), the **entity graph as a lookup, not a traversal** (G29 §3), **GBDT + rules** in the hot path (G29 §4), and the **step-up rung as a label generator** (G29 §6). **New here:** the prompt spans **four event types** (transactions, logins, signups, ad clicks), so R10 designs **one shared risk platform with a model and a cost matrix per event type**, and it gives full sections to the four things this prompt asks for explicitly: **cold start**, **imbalance**, the **delayed-label learning loop**, and **adversarial exploitation**.
>
> **Related sheets:** [G22 delayed conversion](G22_delayed_conversion_intv.md) (modelling label delay) · [G21 ad click prediction](G21_ad_click_prediction_intv.md) (invalid traffic, calibration) · [G25 harmful content](G25_harmful_content_detection_intv.md) (thresholds against review capacity) · [R7](R7_contextual_bandit_recsys_intv.md) (propensities, exploration) · [00 playbook §7.7](00_L6_MLSD_PLAYBOOK.md) (calibration after downsampling).

---

## 📋 THE QUESTION (verbatim)

> **R10. Design a fraud detection system**
>
> **Scenario**
> You are designing an end-to-end fraud detection system for an online platform (e.g., e-commerce marketplace, payments, account signup, or ad traffic). The system should detect and prevent fraudulent activity while minimizing impact on legitimate users.
>
> **Requirements**
> - **Goal:** Predict whether an event (transaction / login / signup / ad click) is fraudulent and decide what action to take.
> - **Latency:** Support near-real-time decisioning (e.g., sub-second to a few seconds) for high-risk actions.
> - **Cold start:** Handle new users / new devices / new merchants with little or no historical data.
> - **Imbalanced data:** Fraud rate is low (e.g., <1%), so the dataset is highly class-imbalanced.
> - **Actions:** Decide between actions such as allow, step-up verification (2FA / OTP), manual review, or block.
> - **Learning loop:** Incorporate delayed labels (chargebacks, user reports, investigation outcomes) and retrain/refresh models.
>
> **What to cover**
> - Data sources and feature engineering (real-time + batch)
> - Model choice(s) and how you handle cold start + imbalance
> - Evaluation metrics and offline/online validation
> - System architecture for training, serving, monitoring
> - Abuse/adversarial considerations and how you prevent model exploitation

### The question's checklist, and where each item is earned

| The question asks for | Where |
|---|---|
| Goal: predict per event, and decide the action | §2 (the decision rule), §6.1 (the ladder per event type) |
| Near-real-time decisioning | §1 (budgets per event), §3, §6 |
| Cold start: new users, devices, merchants | §5.4 |
| Class imbalance | §5.5 |
| Actions: allow · step-up · review · block | §2, §6.1 |
| Learning loop with delayed labels | §5.6 |
| Data sources and features, real-time + batch | §5.1, §5.2 |
| Model choices | §4, §5.3 |
| Evaluation metrics, offline and online validation | §7 |
| Architecture for training, serving, monitoring | §4 diagram, §7.4 |
| Abuse and preventing model exploitation | §8 (security row), §9.2 |

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | *"Which event types, and what does each error cost, in currency?"*, plus investigator capacity and who absorbs the loss (§1) |
| **Explainability** is easy points | **Reason codes** per decision, mapped to human language; an immutable decision log; graph evidence for investigators (§8) |
| **Security** beyond the gateway | **Probing is model testing:** card testing and login stuffing teach attackers the thresholds; **trust farming** poisons history; **fake disputes** poison labels (§9.2) |
| **Troubleshoot before being asked** | Raise **"you never learn about your blocks"**, **outages (step up, don't block)**, and **label delay** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **One risk platform, a model per event type** · errors priced in currency · you never learn about your blocks | A1a, A3a |
| **1–5** | **REQUIREMENTS** | **(a)** | Event types · costs · latency per event · capacity · non-goals | A4a, A1b |
| **5–9** | **PRODUCT + DECISION RULE** | **(b)** | What users see · **the cost matrix** · PR-AUC, not ROC | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Volume per event type · a lookup, not a traversal · $ per 1,000 decisions | A1b, I1a, I1d |
| **12–20** | **ARCHITECTURE** | **(d) (i)** | Shared platform · per-event models · the decision engine · problems raised yourself | A2a, A5a, A1c, F2b |
| **20–34** | **DATA, MODELS, COLD START, IMBALANCE, LOOP** | **(c) (d) (e) (f) (g)** | Features real-time + batch · GBDT + graph · **cold start** · **imbalance** · **delayed labels** | A2b, A2c, A2d, F1a, F1c, F2c |
| **34–38** | **SERVING + ACTIONS** | **(b) (i)** | The ladder per event type · review queue · fail to step-up | F3a, F3b, A3b, I1b |
| **38–44** | **EVALUATION + MONITORING** | **(h) (i)** | Temporal splits · allow-through · champion/challenger · monitors | I2a, I2b, F1b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | **(j)** | Nine dimensions, with security as the adversarial row | I1a–I1j |
| **48–52** | **ADVERSARIES + FAILURES** | **(j)** | **Preventing exploitation** · the new-attack pipeline · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The prompt lists four kinds of event (a transaction, a login, a signup, an ad click), and they share the hard parts but not the economics. So I'd build one risk platform: shared identity and entity resolution, one entity graph, one feature store with real-time velocity counters and batch graph aggregates, and one decision engine. On top of it sit separate models and separate cost matrices per event type, because a missed stolen-card purchase, a missed account takeover and a fake ad click cost completely different amounts and are labelled on completely different clocks.**
>
> **Second, both errors are priced in currency. A miss costs the fraud loss; a false alarm costs friction or a lost customer, including their lifetime value. So the action is whichever of allow, step-up, review or block has the lowest expected cost, and that's why accuracy and ROC-AUC are the wrong metrics at a 0.1% fraud rate.**
>
> **Third, two structural traps. Labels arrive late (chargebacks take weeks), and the system never learns whether the events it blocked were really fraud. So the learning loop needs fast proxy labels, a small budgeted allow-through for unbiased labels, and step-up as the cheapest way to turn an uncertain block into a label. And because the adversary adapts, every evaluation is split by time, and the system is designed so probing it teaches the attacker as little as possible."**

**Then say the plan (`A3a`):** *"Clarifying questions, what users see and the decision rule, capacity, the shared platform, then features, models, and the three things this prompt names: cold start, imbalance and the delayed-label loop; then serving and actions, evaluation and monitoring, a production-readiness sweep, and how I'd stop the model being exploited."*

---

## 1 · REQUIREMENTS *(min 1–5)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Which event types are in scope, and which matters most?** *(Assume all four, with transactions as the deepest.)*
2. **What does each error cost, in currency, and who absorbs it?** *(The decision rule needs C_FN and C_FP per event type, including a legitimate customer's lifetime value.)*
3. **What's the latency budget per event, and can any decision be deferred?** *(Checkout and login must decide in-line; ad clicks can be filtered before billing; physical goods can be held for review.)*
4. **How large is the investigation team?** *(It bounds the manual-review rung.)*
5. **Which verification methods exist?** OTP, authenticator, bank 3-D Secure, ID checks? *(They define the step-up rung.)*
6. **Are labels shared across products?** *(A ring caught on signups is evidence for transactions.)*

**The four event types:**

| Event | Fraud looks like | Fraud rate (illustrative) | Label, and its delay | Decision latency |
|---|---|---|---|---|
| **Transaction** | Stolen card, friendly fraud | ~0.1–0.5% | **Chargeback, 30–90 days**; investigator outcome, days | **In-line, p99 ≤ 200 ms** |
| **Login** | Account takeover (ATO) | ~0.01% | **User report, hours**; step-up failure, seconds | **In-line, p99 ≤ 150 ms** |
| **Signup** | Fake or bulk accounts | Up to a few % | **Later abuse or ban, days to weeks** | **In-line, p99 ≤ 300 ms** |
| **Ad click** | Bots, click farms (invalid traffic) | Several % | **Advertiser refund, honeypot hits, conversion anomalies, days** | **Async: filter before billing, seconds to hours** |
| **Merchant onboarding** (marketplace) | Shell or fraudulent sellers | Low | **Chargeback clusters, refunds, weeks** | **Minutes to days**, with KYB review |

**State assumptions:**

| Assumption | Value |
|---|---|
| Platform | A marketplace with payments and ads, ~2×10⁸ users |
| Costs | C_FN and C_FP supplied per event type, with customer **LTV** from the business |
| Investigators | A fixed team; capacity bounds the review queue |
| Compliance | Card data tokenised under **PCI-DSS**; declines explainable; regional privacy law |

**FR:** score every event in its budget · a cost-derived action per event type · a prioritised review queue · fast proxy labels and slow truth joined back · a budgeted random allow-through · a manual rule path that ships in hours · reason codes.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **In-line decisions** | p99 ≤ 150–300 ms per event type (above) |
| **Objective** | **Expected loss per event**, in currency, per event type |
| **Offline headline** | **PR-AUC** per event type, plus precision at review capacity |
| **Chargeback rate** | Well under the card networks' monitoring threshold (~1%): a cliff, not a slope (G29) |
| **Allow-through** | Small, non-zero, **budgeted** |
| Availability | On an outage: **allow low-risk, step up the rest**, never a blanket block |

**Scope / NON-GOALS:**
- ❌ **Zero fraud.** The economically correct fraud rate isn't zero (G29 FU 6).
- ❌ **Optimising accuracy, F1 or ROC-AUC.**
- ❌ AML and sanctions screening (a different regulatory regime), chargeback dispute handling.

---

## 2 · PRODUCT + DECISION RULE *(min 5–9)* — probe **(b)** `I1c` `F1b` `I2b`

**What each party sees:**

```
A LEGITIMATE USER (almost everyone)        nothing: the event just succeeds
A RISKY-BUT-UNSURE EVENT                   STEP-UP: "Enter the code we sent" /
                                           "Confirm in your banking app"
A HIGH-RISK EVENT                          declined, with a recovery path
                                           (verify identity, retry, contact)
PHYSICAL GOODS / NEW MERCHANT PAYOUT       HELD for review: the order or payout
                                           waits; the user isn't rejected
INVESTIGATOR                               a queue ordered by
                                           P(fraud) × exposed value × RING SIZE,
                                           with the graph and reason codes
RISK OWNER                                 expected loss per event · approval rate
                                           · step-up rate · chargeback rate
```

**The decision rule: the cheapest expected action** (G29 §2, generalised to four actions):

```
p = calibrated P(fraud) for this event
EXPECTED COST of each action:
  ALLOW     p × C_FN                               (a miss)
  STEP-UP   friction cost × (1 − p)  +  p × C_FN × P(fraudster passes step-up)
  REVIEW    reviewer cost + delay cost  (only while the queue has CAPACITY)
  BLOCK     (1 − p) × C_FP                         (C_FP includes the LOST CUSTOMER'S LTV)
  → take the action with the lowest expected cost
  → for one transaction this collapses to G29's:  block when p > C_FP / (C_FN + C_FP)

WORKED EXAMPLE, a $99.99 digital purchase (G29's numbers):
  C_FN ≈ $125 (amount + dispute fee + investigation)
  C_FP ≈ $85 (margin + 20% churn × $250 LTV + support)
  → block only above p ≈ 0.40 · step-up wins across a wide middle band,
    e.g. 0.05 < p < 0.40, because it's cheap on both sides
  ⚠ drop LTV from C_FP and the block threshold falls to ~0.22: twice as eager,
    losing customers worth more than the fraud stopped

THE COSTS DIFFER BY EVENT TYPE, so the same p means different actions:
  transaction   C_FN = amount + fee           C_FP = margin + LTV × churn
  login (ATO)   C_FN = the account's value    C_FP = a few seconds of 2FA
                → step-up is nearly free, so it's used aggressively
  signup        C_FN = expected abuse cost    C_FP = a new user lost at verification
                of a fake account
  ad click      C_FN = advertiser overbilled  C_FP = a publisher's lost revenue
                → no user-facing action: the click is just not billed
```

**Why not accuracy or ROC-AUC:** at a 0.1% fraud rate, "always say legitimate" is 99.9% accurate, and ROC-AUC's false-positive denominator is the huge legitimate population, so it barely moves when 19 of every 20 flags are good customers. **PR-AUC** and **precision at the operating point** are the right offline views, and **expected loss** is the real objective (G29 §2 has the full arithmetic).

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Expected loss per event**, per event type: fraud loss + LTV-weighted false declines + friction | The real objective |
| Offline | **PR-AUC** · precision at review capacity · recall at a fixed false-positive rate · **calibration** | Ranking quality and trustworthy probabilities |
| Online | Approval rate · step-up rate and **step-up pass rate** · review yield per hour · time to detect a new attack | Users and operations |
| **Guardrails** | **Chargeback rate** vs the network cliff · **false-decline rate**, LTV-weighted · decline-rate parity across groups · p99 latency | Hard limits |

> **Say:** *"Both errors have prices, so the action is an expected-cost decision per event type, and step-up wins the whole uncertain middle because it's cheap on both sides and it generates a label."*

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* `A1b` `I1a` `I1d`

```
EVENTS PER DAY (illustrative, 2×10⁸ users)
  signups        10⁶       ≈    12/s
  logins         2×10⁸     ≈ 2,300/s avg,  ~10⁴/s peak
  transactions   5×10⁷     ≈   600/s avg,  ~2×10³/s peak
  ad clicks      2×10⁹     ≈ 2.3×10⁴/s avg, ~10⁵/s peak   ← the volume driver
  → ad clicks get a CHEAPER model (fewer features, a smaller GBDT) and an
    async path; transactions get the RICHEST model, because each one is worth most

⚠ THE GRAPH CAN'T BE WALKED IN 150 ms → the online path is a LOOKUP (G29 §3):
  ring detection, graph embeddings and entity reputation run in BATCH
  (minutes–hours) → written to the online feature store as entity aggregates
  velocity counters (per card, device, IP, account, merchant) update in SECONDS
  from the stream

FEATURE STATE
  ~10⁹ entities (accounts, devices, cards, IPs, merchants) × ~200 B of aggregates
  ≈ 200 GB → a sharded, replicated KV store; one batched read per event

MODEL COST
  a GBDT over a few hundred features ≈ 1 ms of CPU per event
  ad clicks: 10⁵/s × ~0.3 ms (smaller model) ≈ 30 CPU cores at peak
  transactions + logins: ~1.2×10⁴/s × 1 ms ≈ 12 cores → cheap; the feature
  fetch and the batch graph jobs are the bill → track $ per 1,000 decisions

THE IN-LINE BUDGET (transaction, p99 ≤ 200 ms; ~60 ms planned)
  entity resolution (hash card, fingerprint device)         5 ms
  feature fetch: one batched KV read                        15 ms
  rules engine (known patterns, blocklists)                  3 ms
  model (GBDT) + calibration                                 3 ms
  decision engine (expected cost per action)                 2 ms
  network, logging (async)                                  30 ms
  → plenty of headroom: what eats it is a slow dependency, hence timeouts (§6.3)
```

---

## 4 · ARCHITECTURE *(min 12–20)* — probes **(d) (i)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| One model or many | One model for all events · **shared platform, a model per event type** | Different labels, delays, costs and actions; shared entities, graph and features (G29 FU 9) |
| Model family | Rules · logistic regression · **GBDT + rules** · GNN in the hot path · anomaly detection | GBDT: tabular, robust to imbalance, explainable for declines; rules ship in hours; GNNs and anomaly scores run **offline as features** (G29 §4) |
| Graph signal | Walk the graph per event · **precomputed entity aggregates** | 150 ms budgets; rings change over hours |
| Threshold | Maximise F1 · **expected cost per action** | Both errors have prices |
| Uncertain cases | Block · **step-up** | Cheap on both sides, and it produces a label |
| Labels | Chargebacks only · **fast proxies + slow truth + allow-through** | Freshness, plus unbiased labels for what's blocked |
| Retraining | Online learning · **daily or weekly retrains on matured labels + real-time velocity features + a fast rule path** | Velocity features carry the freshness; rules carry live attacks; models carry the learning |

### The system — *draw this in 90 seconds* `A2a`

```
══ ONLINE · per event (p99 150–300 ms in-line; async for ad clicks) ═════════════
 EVENT: transaction / login / signup / ad click / merchant action
┌─ 1. ENTITY RESOLUTION ─────────────────────────────────────────
│ link to account, device (fingerprint + attestation), card (hashed),
│ IP / ASN, email, phone, merchant · OUTPUT: entity IDs
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. FEATURES: one batched read from the online feature store ──
│ REAL-TIME: velocity counters (cards per device in 1 h, failed logins
│   in 10 min, signups per IP today) · session behaviour
│ BATCH: entity reputation · ring_risk · graph embeddings · account age
│ EVENT: amount, item, merchant category, geo, time
│ OUTPUT: one feature vector (+ missing flags for new entities)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. RULES + MODEL (per event type) ────────────────────────────
│ rules: blocklists, known patterns, hard limits (instant, auditable)
│ model: GBDT for this event type → calibrated P(fraud) + REASON CODES
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. DECISION ENGINE ───────────────────────────────────────────
│ cheapest expected action (§2) using THIS event type's costs:
│ allow · step-up · review (if capacity) · block · ring action
│ + a small, budgeted RANDOM ALLOW-THROUGH on would-be blocks
│ + randomness near thresholds (§9.2) · OUTPUT: the action
└──────┬─────────────────────────────────────────────────────────
       ▼
 ACTION → user / payment / billing · DECISION LOG (features, scores, action)

══ NEARLINE (seconds–minutes) ═══════════════════════════════════════════════════
 event stream → velocity counters → online feature store
 step-up outcomes, user reports → fast labels · review queue → case outcomes

══ OFFLINE (hours–days) ═════════════════════════════════════════════════════════
 entity graph → ring detection · GNN embeddings · anomaly scores → feature store
 labels joined as they MATURE (chargebacks, investigations, later abuse)
 → point-in-time training sets → per-event GBDTs on TEMPORAL splits → calibrate
 → shadow → champion / challenger → ramp
 FAST PATH: a manual RULE for a live attack ships in HOURS, no model release
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F2a`

- **You never learn about your blocks:** *"A blocked event produces no chargeback and no label, so over-blocking confirms itself. A small, budgeted random allow-through is the only unbiased signal, and step-up turns most would-be blocks into labelled tests (G29 §5.2)."*
- **Outages** `F2a`: *"If the feature fetch or model times out, I don't block: allow low-value, low-risk events and step up the rest. A block caused by an outage is a lost customer for nothing."*
- **Label delay:** *"Chargebacks arrive in 30–90 days, so recent data looks cleaner than it is. Train on matured windows, add fast proxies, and never evaluate on immature labels (§5.6)."*
- **The latency squeeze** `F2b` `A1c`: *"If a richer model doesn't fit, I don't drop it for everyone: score with the GBDT in-line and run the expensive model async, holding fulfilment or payout where the business allows."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Shared platform, per-event models** | "The infrastructure is shared; the economics aren't" | More models to own; a shared feature contract |
| **2** | **Expected cost, not F1** | "Both errors have prices; C_FP includes lifetime value" | Needs real LTV numbers from the business |
| **3** | **GBDT in the hot path, graph offline** | "Tabular, imbalanced, and declines must be explainable" | Some graph signal left on the table |
| **4** | **Budgeted allow-through** | "The only unbiased label for what I block" | Deliberate, measured fraud loss |

---

## 5 · DATA, MODELS, COLD START, IMBALANCE, LOOP *(min 20–34)* — probes **(c) (d) (e) (f) (g)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 Data sources `A2c`

| Source | Real-time or batch | What it gives | Watch out for |
|---|---|---|---|
| **Event payloads** | Real-time | Amount, item, merchant, geo, time, credentials attempted | Attacker-controlled fields |
| **Device signals** | Real-time | Fingerprint, **attestation** (genuine app on a genuine device?), emulator / root signals | Spoofable unless attested |
| **Network** | Real-time | IP, ASN (data centre vs home), proxy / VPN flags | Shared IPs (offices, universities) |
| **Behaviour in session** | Real-time | Typing cadence, navigation speed, copy-paste into card fields | Accessibility tools look unusual |
| **Account and payment history** | Batch | Age, past purchases, past disputes, verified contact details | New entities have none |
| **Entity graph** | Batch | Shared devices, cards, IPs, addresses, payout accounts | Entity-resolution errors create guilt by association |
| **Labels** | Delayed | Chargebacks, user reports, investigator outcomes, step-up results, later bans, advertiser refunds | Each on its own clock (§5.6) |
| **Allow-through sample** | Delayed | Outcomes for would-be blocks | Costs real loss; small |

### 5.2 Feature engineering, real-time + batch (c)

| Family | Examples | Computed |
|---|---|---|
| **Velocity** | Cards added to this device in 1 h · failed logins from this IP in 10 min · signups per subnet today · transactions per card per hour | **Streaming, seconds** |
| **Deviation from normal** | Amount vs this account's usual · new geo vs usual · first time with this merchant category | Real-time vs batch baselines |
| **Entity reputation** | Device / IP / email-domain / BIN risk · merchant dispute rate | Batch, refreshed hourly |
| **Graph** | Component size · distance to a confirmed-fraud node · ring_risk · accounts per device | **Batch** (G29 §3) |
| **Account trust** | Account age · verified phone / 2FA · tenure of the payment instrument · past successful step-ups | Batch |
| **Session behaviour** | Time from page load to purchase · typing rhythm · navigation path | Real-time |
| **Learned embeddings** | Transaction-sequence embedding per account · GNN node embedding per entity | Batch, stored compressed |

**Rules for features in an adversarial setting:** prefer features that are **expensive to fake** (attestation, account age, graph structure, instrument tenure) over cheap ones (user agent, declared fields); log every feature **as served** so training uses exactly what the model saw (R6 §5.3).

### 5.3 Model choices (d) `A2b` `F1a`

```
PER EVENT TYPE: a GBDT on the features above + a RULES ENGINE
  rules     known patterns, blocklists, hard limits: instant and auditable;
            the fast path for live attacks
  GBDT      handles mixed tabular features and missing values natively,
            robust to imbalance with weighting, fast on CPU, explainable
            (per-decision attributions → reason codes)
  LOSS      COST-WEIGHTED log loss (fraud rows weighted toward their C_FN,
            e.g. by amount), then CALIBRATION, because the decision engine
            multiplies the probability by costs

OFFLINE, PRODUCING FEATURES (not in the hot path):
  · GNN / graph embeddings and ring detection over the entity graph
  · unsupervised ANOMALY scores (e.g. an isolation forest, autoencoder error)
    → catch NOVEL patterns before labels exist; fed to the GBDT and to
      investigators
  · a pretrained transaction-SEQUENCE model per account (§5.7)

WHY NOT A DEEP MODEL IN THE HOT PATH: declines must be explained to users and
  sometimes regulators; tabular data with heavy missingness favours GBDTs;
  latency budgets are tight. Deep models contribute as FEATURES
```

### 5.4 Cold start: new users, new devices, new merchants (e) `F1c` `F2a`

> **The principle:** no history doesn't mean no information. A new entity still has **context** (device, network, how it arrived), **connections** (what it shares with known entities) and **behaviour** (what it does in this session), and its risk can be bounded by **graduated trust** while history builds.

| New… | Signals that exist on day 0 | Policy until history builds |
|---|---|---|
| **User** | Device attestation and reputation · IP / ASN · email domain and age · phone carrier type (VoIP vs mobile) · signup velocity from the same network · **behaviour in this session** · **graph links** to known entities | **Graduated trust:** lower spending and payout limits for the first N days · step-up rather than block · higher review rate for high-value first purchases |
| **Device** | Attestation (a genuine app on genuine hardware?) · emulator / root signals · how many accounts it has touched · the fingerprint's history if seen before | **New device + old account = ATO risk:** step up the login · new device + new account = signup scrutiny |
| **Merchant** | KYB / KYC onboarding checks · business category priors · **links to banned merchants** (shared bank account, address, owner, device) · listing patterns | **Payout holds** and rolling reserves for the first weeks · velocity caps on sales volume · ramp limits that rise with clean history |

**In the model:**
- **Missing-value flags** instead of zeros, so the GBDT learns "no history" as its own condition; **account-age and history-length features** let it learn how much to trust history.
- **Hierarchical priors:** a new merchant inherits its **category's** dispute rate, shrunk toward its own as data arrives (the smoothed-rate idea from R6 §5.2).
- **Graph features carry most of the weight:** a brand-new account that shares a device with three banned accounts isn't a cold start at all.
- **A separate new-entity model** is worth it when new entities dominate an event type (signups always do): trained only on first-N-days events, with features that exist on day 0.
- ⚠ **Don't punish newness itself:** every good user was new once. Newness raises the **step-up** rate, not the block rate, and the false-decline rate for new users is a guardrail.

### 5.5 Class imbalance, explicitly (f) `F1a` `F1b`

```
THE PROBLEM: ~1 fraud per 1,000 events. A model can be 99.9% accurate by saying
  "legitimate" every time, and gradient training is dominated by easy negatives

1. MEASURE IT RIGHT     PR-AUC · precision at review capacity · recall at a
                        fixed false-positive rate · expected loss. Never accuracy;
                        ROC-AUC only as a sanity check
2. RESAMPLE OR WEIGHT   keep ALL fraud, DOWNSAMPLE legitimate events (e.g. keep
   (choose one)         r = 1 in 20) → much faster training, little signal lost
                        OR weight fraud rows up (cost-weighted: by amount × C_FN)
                        ⚠ either way, CALIBRATE back to the true rate before
                          costs are applied:  p = p' / (p' + (1 − p')/r)
                          (playbook §7.7), then isotonic on fresh, unsampled data
3. HARD NEGATIVES       keep the legitimate events the model finds CONFUSABLE
                        (high score, not fraud) at full weight: they define the
                        boundary; easy negatives add little
4. MORE POSITIVES       investigator confirmations, failed step-ups, confirmed
                        rings (every account in a confirmed ring), labels from
                        other event types for the same entities
5. WHAT RARELY HELPS    SMOTE-style synthetic fraud for tabular data: interpolated
                        fake fraud doesn't look like real fraud, and fraud
                        changes faster than any synthetic distribution
   (for a deep model: focal loss down-weights easy negatives, but GBDT +
    weighting is usually enough)
⚠ LABEL NOISE matters more when positives are rare: friendly fraud tagged as
  stolen-card, unreported fraud tagged as legitimate → use chargeback reason
  codes, and treat "no chargeback yet" as UNKNOWN until labels mature (§5.6)
```

### 5.6 The learning loop: delayed labels, retraining, refresh (g) `F1c` `A2d`

```
EVERY LABEL HAS A CLOCK
  step-up failed / passed         seconds    weak (a fraudster may pass; a
                                             real user may give up)
  user report ("not me")          hours      strong for ATO; noisy for payments
  investigator outcome            days       strong, capacity-limited; also a
                                             GRAPH SEED for the whole ring
  later abuse / ban (signups)     days–weeks strong
  advertiser refund (ad clicks)   days       medium
  chargeback (transactions)       30–90 days the slow truth; includes friendly fraud

HOW THEY'RE USED
  · MATURITY WINDOWS: a transaction enters training as "legitimate" only after
    its chargeback window has passed; before that it's UNKNOWN, not negative
    (or use a delayed-feedback model that learns the delay distribution, G22)
  · FAST PROXIES keep the model fresh: investigator outcomes and failed step-ups
    are weighted by how well they've predicted chargebacks historically
  · CONFIRMED RINGS label their neighbours: one confirmed case raises the
    ring_risk of every linked entity long before their own labels arrive
  · ALLOW-THROUGH outcomes are the unbiased slice, weighted 1/propensity

CADENCE: which part of the system carries which kind of freshness
  seconds    velocity counters and blocklists (the stream)
  hours      manual RULES for a live attack · entity reputation, ring_risk
  daily      retrain the fast-label models (ATO, signup) on fresh labels
  weekly     retrain the transaction model on matured chargebacks
  every push temporal-split evaluation → shadow → champion / challenger
```

### 5.7 Fine-tune an existing model `F2c`

If a pretrained **transaction-sequence model** exists (a model of how accounts normally behave, trained across products), adapt it per event type with small adapters (LoRA-style) and feed its embedding to the GBDT as a feature. The GBDT stays the explainable decision-maker; the sequence model adds behavioural signal it couldn't build itself (G29 §5.4).

### 5.8 The staged pipeline `A2d`

```
STAGE 1 · EVENTS     every event, device signal and decision → the stream + log
STAGE 2 · FEATURES   velocity in seconds; graph and reputation in batch → feature store
STAGE 3 · LABELS     fast proxies now, slow truth as it matures; allow-through outcomes
STAGE 4 · TRAIN      per-event GBDTs, cost-weighted, on TEMPORAL splits → calibrate
STAGE 5 · DEPLOY     shadow → champion / challenger on a traffic slice → ramp
STAGE 6 · MONITOR    drift, decline and step-up rates, chargeback rate, attack signals
FAST PATH: a manual rule ships in hours for a live attack
```

---

## 6 · SERVING + ACTIONS *(min 34–38)* — probes **(b) (i)** `F3a` `F3b` `A3b` `I1b`

### 6.1 The action ladder, per event type `A3b`

| Event | Allow | Step-up | Manual review | Block / act |
|---|---|---|---|---|
| **Transaction** | Completes | 3-D Secure / OTP: friction for the user, and liability shifts to the bank | **Physical goods only:** hold the shipment (digital goods are delivered instantly, so there's no window) | Decline with a recovery path; card or account action on ring evidence |
| **Login** | Logged in | 2FA / passkey re-prompt: almost free, so used widely | Rarely in-line | Lock the session, force a password reset, notify the owner |
| **Signup** | Account created | Phone verification, CAPTCHA | Bulk-signup clusters reviewed as a ring | Reject, or create the account on **probation** (limited features) |
| **Ad click** | Billed | — (no user to challenge) | Investigation of suspicious publishers | **Not billed** (filtered); the publisher is suspended on ring evidence |
| **Merchant** | Payouts flow | Extra verification documents | **KYB review** of high-risk sellers | Payout hold, reserve, suspension |

- **The decision objective** `A3b`: per event, the action with the lowest expected cost (§2) using that event type's costs. **Manual review is capacity-bounded:** if the queue is full, the next-best action takes over, and the queue is ordered by `P(fraud) × exposed value × ring size` (G29 §6).
- **Step-up earns its place three times:** it cuts the false-positive cost, it deters the fraudster, and its outcome **is a label** for a case that would otherwise have been blocked blind.

### 6.2 Efficiency and scale `F3a` `F3b`

- **Efficiency** `F3a`: a GBDT on CPU (~1 ms); a **smaller, cheaper model for ad clicks**, where volume is highest and each event is worth least; graph embeddings stored **quantised** in the feature store; the expensive anomaly and graph models run in batch.
- **Scale** `F3b`: entity aggregates in a sharded, replicated KV store with one batched read per event; velocity counters partitioned by entity; stateless scoring services autoscaled per event type; batch graph jobs on a distributed engine.

### 6.3 Reliability: fail safely, by risk `I1b`

```
TIER 0 · FULL         rules + model + decision engine, all features fresh
TIER 1 · STALE BATCH  graph / reputation features old → use them with a flag;
                      velocity still live
TIER 2 · NO MODEL     model or feature store timing out → RULES ONLY: allow
                      low-value, low-risk events; STEP UP the rest
TIER 3 · NO RISK      the whole risk service down → allow below a small amount
   SERVICE            limit, step up above it (a fixed, pre-agreed policy)
⚠ NEVER a blanket block: an outage-driven block costs customers for nothing.
  Ad clicks are the exception that's easy: billing waits until scoring catches up
```

---

## 7 · EVALUATION + MONITORING *(min 38–44)* — probes **(h) (i)** `I2a` `I2b` `F1b` `F1c`

### 7.1 Offline

```
⚠ TEMPORAL SPLITS, always: train on weeks 1…N, test on week N+k, after a GAP
  as long as the label delay. A random split puts tomorrow's attack patterns
  into training and grades the model against an adversary that already moved
METRICS per event type: PR-AUC · precision at review capacity · recall at a fixed
  false-positive rate · CALIBRATION (the decision engine multiplies p by costs)
  · EXPECTED LOSS replayed with the cost matrix
SLICES: new vs established entities (cold start) · amount bands · geo · device
  type · merchant category · and the ALLOW-THROUGH sample, the only unbiased slice
  for what the old policy blocked
⚠ IMMATURE LABELS: never compute recall on transactions younger than the
  chargeback window; they look like misses that aren't
```

### 7.2 Online validation

- **Shadow:** the new model scores live events alongside the champion; compare score distributions, would-be actions, latency.
- **Champion / challenger:** the challenger **decides** for a few percent of traffic (randomised by user or entity), so its actions are real; compare **expected loss**, approval rate, step-up rate and pass rate, and the **matured** chargeback rate weeks later (G29 §7).
- **Why not a plain A/B on fraud rate:** labels take weeks, and blocked events have no labels, so the **allow-through sample** and step-up outcomes are what make the comparison fair before chargebacks arrive.
- **Success** = expected loss per event down **at the same or better approval rate**, chargeback rate under the cliff, false declines for new users not up, decline rates fair across groups. `I2b`

### 7.3 Monitoring and response `F1c`

| Signal | Catches | Response |
|---|---|---|
| **Score distribution per event type** (mean, tail share) | Model or feature breakage; a new attack | Alert; compare with the champion |
| **Decline, step-up and review rates** | Over-blocking drift; a threshold bug | Auto-rollback beyond a band |
| **Step-up pass rate** | Attackers passing step-up (SIM-swap, OTP phishing) | Stronger method for that slice |
| **Chargeback rate**, matured, vs the cliff | The real loss | Tighten thresholds on the worst slice |
| **Feature drift** (PSI, null rates) and **training–serving skew** | Upstream changes | Block model pushes |
| **Velocity spikes** per entity, subnet, BIN | Live attacks (card testing, credential stuffing) | **Manual rule in hours** |
| **Pattern novelty**: share of flagged activity unlike known clusters | New attack types | Investigators → graph seeds |
| **Latency, timeouts, fallback tier share** | Degradation | Capacity, not thresholds |

**Bias sources and mitigations:**

| Bias | Mechanism | Mitigation |
|---|---|---|
| **Selection** | Only allowed events get labels | Allow-through; step-up outcomes; investigated samples of blocks |
| **Label delay** | Recent fraud not yet labelled | Maturity windows; fast proxies; delay modelling |
| **Adversarial shift** | Attackers adapt faster than retraining | Temporal evaluation; velocity and graph features; novelty monitoring |
| **⚠ Proxy discrimination** | Postcode, device price or language standing in for protected groups | Decline-rate parity audits; feature review; an appeal route |
| **Newness penalty** | New users blocked for being new | Step-up instead of block; false-decline guardrail for new users |
| **Guilt by association** | A wrong entity link flags innocent accounts | Precision on links; legitimate shared entities (families, offices) |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p99 150–300 ms in-line per event type; async for ad clicks; **expected loss per event** as the objective; $ per 1,000 decisions | p99 per event · expected loss |
| **Reliability** `I1b` | Fail by risk: rules-only, then allow-small / step-up-large; **never a blanket block**; versioned rollouts with fast rollback | Fallback tier share · outage-mode decisions |
| **Usability** `I1c` | Step-up instead of decline where possible; a **recovery path** for every decline; new users not punished for being new | Step-up completion · false-decline complaints |
| **Scalability** `I1d` | Shared entity layer and feature store across event types; a cheap model for the highest-volume event; batch graph jobs | Fetch p99 · cores per 10k events/s |
| **Privacy** `I1e` | Card data tokenised under **PCI-DSS**; device and behavioural signals collected with disclosure and retention limits; cross-product signals only within policy | PCI audit · retention compliance |
| **Safety** `I1f` | Protect users, not just the platform: **ATO alerts** to account owners, **scam warnings** where the real user is being manipulated (P2P), fast account recovery | ATO dwell time · scam-loss reports |
| **Security** `I1h` | Auth and rate limits, and **beyond the gateway**: ⚠ **probing** (card testing and credential stuffing double as model testing) · ⚠ **trust farming** (aged accounts with fake good history) · ⚠ **label poisoning** (fake disputes, mass false reports) · ⚠ the model and features treated as **secrets** (§9.2) | Probe-pattern alerts · rule hit rates |
| **Ethics / bias** `I1i` | **Proxy discrimination** audited as decline-rate parity by group; LTV in C_FP so the system doesn't over-block; an appeal route | Decline-rate parity · appeal outcomes |
| **Explainability** `I1j` | **Reason codes** per decision from model attributions, mapped to human language (users get a category, not the rule); an **immutable decision log** for regulators; investigators see the graph evidence | % decisions with a reason code · audit completeness |

> ⭐ *"A fraud decision is a decision about a person: it needs a price on both errors, a recovery path, a fairness audit, and on an outage it should default to step-up, not to no."*

---

## 9 · ADVERSARIES + FAILURES *(min 48–52)* — probe **(j)** `F3c` `F2a` `A5b`

### 9.1 The fallback pipeline when a new attack appears `A5b` `F2a`

```
a NEW pattern: the model hasn't seen it, and its labels are weeks away
  → VELOCITY and ANOMALY signals first: a spike in cards per device, a
    fast-growing graph component, a burst of signups from one subnet
  → INVESTIGATORS confirm a few cases → GRAPH SEEDS → the whole ring is suspect
  → a MANUAL RULE ships in hours (no model release), scoped to the slice
  → STEP-UP on the matching slice while evidence builds (not blanket blocks)
  → confirmed labels flow into the next retrain; the rule is retired once the
    model covers it (rules accumulate otherwise)
```

### 9.2 Preventing model exploitation — *the prompt's last bullet, in full* `I1h`

| Attack on the model | How it works | Defence |
|---|---|---|
| **Probing the thresholds** | Card testing and credential stuffing: many small attempts to learn what passes | **Rate limits per entity** (card, device, IP, BIN) · **randomness near thresholds** (the same attempt doesn't always get the same answer) · **identical messages** for every decline · act on probing **itself**: a burst of small failures is a signal |
| **Learning from reason codes** | Using the decline reason to adjust the next attempt | Users get a **category** ("we couldn't verify this payment"), never the rule or the features |
| **Fast feedback** | Seeing immediately which attempts got through | **Delayed, batched enforcement** for rings: let a suspected ring run under watch and ban it **in waves**, so the attacker can't tell which signal gave them away |
| **Mimicking good users** | Faking the features the model trusts | **Expensive-to-fake features** (attestation, account age, instrument tenure, graph structure) over cheap ones (user agent, declared fields) |
| **Trust farming** | Aging accounts and making small legitimate purchases to build a good history, then cashing out | Velocity on **change** (sudden jumps in amount or category) · graph links between "aged" accounts · trust that rises slowly and drops fast |
| **Label poisoning** | Fake disputes or mass false reports to corrupt training labels | Trust-weight reporters · cap any single entity's influence on labels · investigate label spikes |
| **Model theft / insider risk** | Exfiltrating thresholds or the feature list | Access control and audit on the model, rules and features; scores never exposed through public APIs |
| **Adaptation** | Attackers shift as soon as something works | **Temporal evaluation** · **novelty monitoring** · the fast rule path · **honeypots** (decoy accounts, cards or ads that only attackers touch) for early warning |

> **Say:** *"The model is under attack the moment it ships, so I'd make probing expensive and uninformative: rate limits, randomness near thresholds, identical decline messages, ring bans in waves, and features that are expensive to fake."*

### 9.3 Extensions `F3c`

- **Real-time graph models:** a GNN scoring the live neighbourhood instead of nightly aggregates. The **cost** is latency and serving complexity; the **maturity risk** is explainability for declines. Start with streaming updates to a few graph features (accounts per device in the last hour) rather than live message passing.
- **Behavioural biometrics** (typing and touch patterns) for ATO. Strong signal, but **privacy and accessibility** risks: opt-in where required, and never the sole reason for a block.
- **LLM-based scam detection** on P2P payment notes or chats (G29 §9): the real user authorises the payment, so the action is a **warning**, not a block; on-device or high-risk-only for privacy and cost.

### Top 3 failure modes

1. **Over-blocking drift:** self-confirming and invisible without allow-through, because blocked events never produce labels.
2. **Evaluating against yesterday's attacker:** a random split hides adaptation entirely; only temporal evaluation shows it.
3. **Teaching attackers the thresholds:** deterministic decisions plus specific reasons turn card testing into model testing.

**Q1 ships:** the shared entity layer and feature store with **velocity counters**, per-event GBDTs with **cost-weighted loss and calibration**, the **expected-cost decision engine with step-up**, the **budgeted allow-through**, the manual rule path, and **temporal-split evaluation**.
> *"Step-up and the allow-through first: step-up converts uncertain blocks into cheap, labelled tests, and the allow-through is the only way the system ever learns whether its blocks were right."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Fraud is 0.1% of events. How do you train and evaluate on that?"** `F1a` `F1b`
> Start with the evaluation, because that's where imbalance does the most damage: a model that always says "legitimate" is 99.9% accurate, and ROC-AUC barely moves when nineteen of every twenty flags are good customers, because its denominator is the huge legitimate population. So I'd report **PR-AUC**, **precision at the review capacity**, **recall at a fixed false-positive rate**, and above all **expected loss** with the cost matrix. For training, keep every fraud example and **downsample the legitimate ones**, say one in twenty, which speeds training a lot and loses little, or equivalently weight fraud rows up, ideally by their cost. Either way the model's probabilities are now wrong in scale, so I **correct them back** with p = p′ / (p′ + (1 − p′)/r) and fit isotonic calibration on fresh, unsampled data, because the decision engine multiplies p by currency. I'd keep **hard negatives**, the legitimate events the model finds confusable, at full weight, since they define the boundary. And I'd get **more positives**: investigator confirmations, failed step-ups, whole confirmed rings. What I wouldn't lean on is synthetic oversampling like SMOTE: interpolated fake fraud doesn't look like real fraud, and real fraud moves faster than any synthetic distribution.

**2. "A brand-new user, on a new device, buying from a merchant that joined yesterday. How do you decide?"** `F1c` `F2a` `I1c`
> No history isn't no information. **The device** has attestation (is this a genuine app on genuine hardware?) and maybe a fingerprint seen elsewhere. **The network** has a reputation: a home connection or a data centre. **The account** has an email age and a phone type, mobile or VoIP. **The session** has behaviour: did they paste a card number in two seconds? And **the graph** has links: does this device, card or payout account connect to anything already banned? Often the "new" entity turns out to share infrastructure with known fraud, and then it isn't a cold start at all. For the merchant, onboarding checks and **category priors** stand in for history, shrunk toward its own record as data arrives. Then the policy is **graduated trust**: lower limits for the first days, **step-up rather than block** when the score is uncertain, and for the merchant, **payout holds and rolling reserves** that relax as clean history builds. The thing to avoid is punishing newness itself: every good customer was new once, so newness raises the step-up rate, not the block rate, and I'd guard the false-decline rate for new users specifically.

**3. "Chargebacks take 60 days. How does the system keep learning?"** `F1c` `A2d`
> By giving each kind of freshness to the part of the system that can carry it. **Seconds:** velocity counters and blocklists update from the stream, so a card-testing burst is visible immediately, with no retrain. **Hours:** investigators confirm a few cases of a new pattern, those become graph seeds that raise the risk of the whole ring, and a **manual rule** ships without a model release. **Days:** models for fast-labelled events (logins, signups) retrain on fresh labels. **Weekly:** the transaction model retrains on **matured** chargebacks. The rule that keeps training honest is **maturity**: a transaction younger than its chargeback window is **unknown**, not legitimate, otherwise recent data looks cleaner than it is and the model learns that new patterns are safe. Fast proxies (failed step-ups, investigator outcomes) are weighted by how well they've predicted chargebacks historically, or I model the delay directly (G22). And the **allow-through** sample, weighted by its propensity, is the unbiased slice that tells me whether my blocks were right.

**4. "How do you stop fraudsters reverse-engineering your model?"** `I1h` `F1c`
> Assume they're probing from day one, because card testing and credential stuffing *are* model testing. So I'd make probing **expensive** and **uninformative**. Expensive: rate limits per card, device, IP and card range, and treating a burst of small failures as a fraud signal in itself. Uninformative: **identical decline messages** whatever the reason; reason codes kept internal, with users seeing only a category; **randomness near the thresholds**, so the same attempt doesn't always get the same answer; and for rings, **delayed enforcement**, watching a suspected ring and banning it in waves, so the attacker can't tell which signal gave them away. On the feature side, I'd lean on things that are **expensive to fake** (device attestation, account age, instrument tenure, graph structure) rather than cheap declared fields. And I'd watch for the slower attacks: **trust farming**, where accounts are aged with small legitimate purchases before cashing out, and **label poisoning** through fake disputes or mass false reports, handled by trust-weighting reporters and capping any one entity's influence on labels. Honeypots, decoy accounts and cards only attackers would touch, give early warning.

**5. "How do you choose between allow, step-up, review and block?"** `A3b` `I1c`
> By expected cost, per event type. For each action I compute what it's expected to cost given the calibrated probability p: allowing costs p × C_FN; blocking costs (1 − p) × C_FP, where C_FP includes the lost customer's lifetime value; step-up costs a little friction for everyone challenged plus the fraud that still gets through; review costs reviewer time and delay, and is only available while the queue has capacity. Then I take the cheapest. With realistic costs, **step-up wins the whole uncertain middle**, because it's cheap on both sides and it produces a label, so block is reserved for high p, and allow covers the low end. The costs differ by event type, so the same p means different actions: on a login, step-up is nearly free, so it's used aggressively; on an ad click there's no one to challenge, so the action is simply not billing the click. And because C_FN scales with the amount, a $2 purchase almost never justifies a block, while a $2,000 one does at a much lower p.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "You block an event. Was it fraud?"** `F1c` `I2a`
> You'll never know, and that's the deepest problem in the system. A blocked transaction produces no chargeback and no label, so the training data contains outcomes only for what the old policy **allowed**. If the model believes a pattern is fraud and always blocks it, that belief can never be contradicted, and over-blocking confirms itself. The mitigations, in order of value: a small **random allow-through** on would-be blocks, a deliberate, budgeted fraud loss that's the only unbiased signal; **step-up instead of blocking**, which is cheaper, since a completed challenge is itself evidence; and **manual investigation of a sample of blocks**, which buys labels without the loss. I'd budget the allow-through explicitly and log its propensity, because it's the line item that gets cut first and is impossible to reconstruct later. It's the same bandit-feedback problem as R7, with money attached.

**7. "Why not one model for transactions, logins, signups and ad clicks?"** `A5a` `A2b`
> Because they share infrastructure but not economics. The **labels** run on different clocks: chargebacks in 30–90 days, takeover reports in hours, fake-account bans in days to weeks, ad refunds in days. The **costs** differ by orders of magnitude, and so do the **actions**: step-up is nearly free on a login, costly at checkout, and impossible on an ad click. The **signals** differ too: payments are transactional and graph-shaped, takeover is behavioural and session-shaped, click fraud is about traffic patterns. One model would blur all of that. What I'd share is everything underneath: **entity resolution, the graph, the feature store and the decision engine**, and **cross-event features**, because a ring caught on signups is strong evidence for transactions from the same devices. Shared platform, separate models, separate cost matrices, separate thresholds.

**8. "Fraudsters age accounts for months before cashing out. What catches them?"** `I1h` `F1a`
> Trust farming is designed to beat history-based features, so I'd stop treating history as one number. Three signals. **Change, not level:** an account that made $5 purchases for six months and suddenly buys $800 of gift cards on a new device is a velocity-of-change pattern, regardless of its age. **Shape of the history:** farmed accounts often have suspiciously regular, low-value, low-diversity activity, and many of them were created together, so **graph links between the aged accounts** (shared devices, cards, creation dates) give the farm away. **Asymmetric trust:** trust rises slowly with clean history and drops fast on anomalies, and high-value first-of-a-kind actions get step-up regardless of age. And the cash-out moment is where the ring acts together, so ring-level detection and wave bans matter here more than anywhere.

**9. "The step-up rate doubled overnight. Attack or bug?"** `I2a` `F2a`
> Separate those quickly, in order. **Did anything ship?** A model push, a rule, a threshold or cost change; the decision log shows the version per decision, and if the jump is on the new version, roll back first. **Did a feature break?** Null rates and PSI on the feature store, and training–serving skew; a broken velocity counter that defaults to a high value looks exactly like an attack. **Is it concentrated?** An attack shows up in a slice: one card range, one subnet, one merchant, one device type. A bug usually shows up everywhere. **What's the step-up pass rate?** If real users are passing at the normal rate, the model is over-challenging good traffic; if the pass rate collapses, there really are more fraudsters being stopped. The structural point: per-version metrics, feature monitors and slice breakdowns are what make this a fifteen-minute diagnosis instead of a day.

**10. "Ad-click fraud has no chargeback. Where do labels come from?"** `F1c` `A2c`
> From several weaker sources, combined. **Honeypots:** decoy ads or invisible links that no real user would click, so every click on them is invalid by construction. **Conversion anomalies:** traffic that clicks but never converts, or converts in impossible patterns, compared with the advertiser's normal funnel. **Advertiser refunds and complaints**, which are delayed and noisy. **Investigator review** of suspicious publishers and click farms. And **behavioural signals** with known ground truth, like clicks from data-centre IPs or known bot frameworks. The action is also different: there's nobody to challenge, so suspected clicks are simply **not billed**, and since billing can wait, scoring can run async with heavier models. The cost matrix balances advertisers overbilled against publishers underpaid, and both sides need to trust it, so the filtering rules and their aggregate impact are reported to both.

---

## ⭐ THE L6 LINE

> **"One risk platform with shared entities, graph and features, and a model and cost matrix per event type. Every action is the cheapest expected one, step-up wins the uncertain middle and generates labels, the loop runs on matured labels plus fast proxies and a budgeted allow-through, and the system is built so probing it teaches the attacker as little as possible."**

## 🧭 HOOKS — *steer by what you mention*

- *"Shared platform, separate economics"* → invites the architecture branch (**good at L6**)
- *"Both errors have prices"* → invites the decision-rule branch
- *"You never learn about your blocks"* → invites the selection-bias branch (**good at L6**)
- *"No history isn't no information"* → invites the cold-start branch
- *"Unknown, not negative, until labels mature"* → invites the learning-loop branch
- *"Probing is model testing"* → invites the adversarial branch
- *"Step-up converts a block into a labelled test"* → invites the actions branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to one event: what's it worth, what does each mistake cost, and when do I find out the truth?"* Costs, labels and actions regenerate the design.
- **Running out of time?** → *"I have five minutes. I'd rather cover preventing exploitation and the learning loop than finish the feature list. Sound right?"*

**Name and defer:**
- "**Payment-specific detail** (the card network flow, 3-D Secure liability shift, the chargeback cliff) is G29."
- "**AML and sanctions screening** is a different regulatory system."
- "**Chargeback representment**, contesting disputes, is an operations workflow."
- "**Account recovery** after a takeover needs its own flow and abuse controls."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"One risk platform, a model and cost matrix per event type; the graph as a lookup."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"p99 150–300 ms in-line; ~60 ms planned; ad clicks at 10⁵/s get a cheaper async model."* |
| **A1c** | Degrade without lowering quality | §4, §6.3 | *"Score in-line with the GBDT; run the expensive model async and hold fulfilment where allowed."* |
| **A2a** | The full stack | §4 diagram | Entity resolution → features → rules + model → decision engine → action; nearline and offline loops |
| **A2b** | Model components | §5.3 | Per-event GBDTs + rules; GNN, anomaly and sequence models offline as features |
| **A2c** | Data sources | §5.1, FU 10 | Event payloads, device, network, behaviour, history, graph, labels, allow-through |
| **A2d** | Staged pipeline | §5.6, §5.8 | Events → features → labels → train → deploy → monitor; the fast rule path |
| **A3a** | Top-down, unprompted | §0 | *"Shared platform; both errors priced; two structural traps"* + the plan |
| **A3b** | A principled objective for the trade-off | §2, §6.1, FU 5 | *"The action with the lowest expected cost, per event type, review only while capacity allows."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Event types, costs in currency, latency per event, investigator capacity, step-up methods |
| **A5a** | Alternatives compared | §4, FU 7 | One model vs per-event; rules vs GBDT vs GNN vs anomaly; F1 vs expected cost |
| **A5b** | A full alternative pipeline on demand | §9.1 | The new-attack pipeline: velocity and anomalies → investigators → rule → step-up → retrain |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and method per subproblem | §5.3, §5.5, FU 1 | *"Cost-weighted GBDT, downsampling with calibration back to the true rate, hard negatives."* |
| **F1b** | Metrics and their trade-offs | §2, §7.1, FU 1 | *"PR-AUC and precision at capacity, never accuracy; expected loss is the objective."* |
| **F1c** | Bias sources and mitigations | §5.4, §5.6, §7.3, FU 6 | Selection, label delay, adversarial shift, proxy discrimination, newness penalty, guilt by association |
| **F2a** | Something fails → a fallback | §6.3, §9.1, FU 9 | Rules-only, then allow-small / step-up-large; the new-attack pipeline |
| **F2b** | Latency too tight → design for it | §4 | In-line GBDT, async heavy model, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.7 | A pretrained transaction-sequence model adapted per event type, as a GBDT feature |
| **F3a** | Efficiency with a quality trade | §6.2 | A smaller model for ad clicks; quantised embeddings; heavy models in batch |
| **F3b** | Scale levers | §3, §6.2 | Shared sharded feature store, one batched read, stateless autoscaled scorers |
| **F3c** | Extension, with cost and maturity risk | §9.3 | Real-time graph models; behavioural biometrics; LLM scam detection |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p99 per event type; expected loss; $ per 1,000 decisions |
| **I1b** | Reliability | §6.3, §8 | *"Never a blanket block on an outage: rules-only, then allow small, step up the rest."* |
| **I1c** | Usability | §2, §8, FU 2 | Step-up over decline; a recovery path; newness doesn't mean blocked |
| **I1d** | Scalability | §3, §8 | Shared entity layer; a cheap model for the highest-volume event |
| **I1e** | Privacy | §8 | PCI-DSS tokenisation; disclosed device and behavioural signals; retention limits |
| **I1f** | Safety | §8 | ATO alerts; scam warnings; fast recovery |
| **I1h** | Security | §8, §9.2, FU 4, 8 | **Probing, trust farming, label poisoning**, model secrecy |
| **I1i** | Ethics / bias | §7.3, §8 | Decline-rate parity; LTV in C_FP; appeals |
| **I1j** | Explainability | §8 | Reason codes mapped to human language; an immutable decision log |
| **I2a** | Offline → online | §7, FU 9 | Temporal splits after a label-delay gap → shadow → champion/challenger → matured chargebacks |
| **I2b** | The success definition | §7.2 | *"Expected loss down at the same or better approval rate, chargebacks under the cliff, fair declines."* |

**🧭 Probe map:** (a) clarifying → §1 · (b) goal and actions → §2, §6.1 · (c) data and features → §5.1, §5.2 · (d) models → §4, §5.3 · (e) cold start → §5.4 · (f) imbalance → §5.5 · (g) learning loop → §5.6 · (h) evaluation → §7 · (i) training, serving, monitoring → §4, §7.3 · (j) adversaries → §8, §9.2.
