# G29 · Fraud Detection — **INTERVIEW SHEET**

> **Tier 1 · #7** — named in multiple "most commonly asked" lists.
> Full study file: [`../G29_fraud_detection.md`](../G29_fraud_detection.md) · [diagrams](../G29_fraud_detection_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the product looks like · (c) the decision stack · (d) datasets · (e) choosing the training set · (f) the training objective · (g) serving · (h) how you know it worked · (i) the extension.

---

## 🎯 WHICH PLATFORM? — *read this first*

**The prompt is platform-agnostic** (*"design a fraud detection system"*), and so is the design. **Assume payment fraud on purchases where Google is the merchant.** At Google, the canonical instance is **Google's own payments risk engine** behind purchases on **Google Play** (apps, in-app purchases, subscriptions), **YouTube Premium / Google One** subscriptions, and the **Google Store**.

**What "online, card-not-present" means:** a user buys something *from Google* inside an app or website and pays with a card **typed in or saved in their account**. No card is swiped, chipped or PIN-verified at a terminal, so **nothing proves the person paying actually holds the card**. Under card-network rules, **the merchant absorbs fraud losses on card-not-present purchases**. Strong authentication like **3-D Secure** (the bank's "confirm in your banking app" step) can shift that liability to the cardholder's bank.

**Who charges whom: the four parties and the money flow**

```
CARDHOLDER — the person whose card it is (maybe NOT the person paying!)
MERCHANT   — Google: sells the goods, runs THIS fraud model
ACQUIRER   — Google's payment processor / bank; connects Google to the card network
ISSUER     — the cardholder's bank: issued the card, runs ITS OWN fraud model
NETWORK    — Visa / Mastercard: routes messages and money between acquirer and issuer

STEP 1 · PURCHASE (milliseconds)
  buyer clicks "Buy $99.99"
     → GOOGLE runs OUR fraud model: allow / step-up (3-D Secure) / decline
     → if allowed: authorisation request  GOOGLE → ACQUIRER → NETWORK → ISSUER
     → ISSUER checks the card is valid, has funds, looks OK to ITS model → APPROVE
     → Google delivers the goods (for digital goods: instantly)

STEP 2 · SETTLEMENT (1–2 days)
  ISSUER ──$99.99──► NETWORK ──► ACQUIRER ──~$97 (minus ~2–3% fees)──► GOOGLE
  ISSUER bills the CARDHOLDER on their monthly statement

STEP 3 · CHARGEBACK (weeks later, only if the cardholder says "that wasn't me")
  CARDHOLDER disputes with ISSUER → ISSUER refunds the cardholder
     → ISSUER claws $99.99 back:  NETWORK → ACQUIRER → debits GOOGLE
     → plus a dispute fee (~$15–25) charged to GOOGLE
  Google can contest it ("representment"), but for stolen-card fraud it usually loses
  → NET: Google is out the money, the fee, AND the goods it already delivered
```

**Why this matters for the design:**
- **Our model runs in step 1, before authorisation.** The issuer runs its own model after ours, so there are **two fraud models in series**, and the issuer's approval does *not* mean the purchase is safe for Google.
- **The label (a chargeback) arrives in step 3, weeks later.** That's the delayed-labels problem in one picture.
- **3-D Secure is an action on the ladder, not just a check:** challenging a risky purchase adds friction for the buyer but moves the fraud loss off Google.
- ⚠ **There's a cliff, not just a per-transaction cost.** Card networks put merchants whose chargeback rate creeps toward ~1% into monitoring programmes with fines, and eventually can cut them off. That's a reason to watch the **chargeback rate** as a guardrail, not only expected loss.

**⚠ Not Google Pay at a third-party shop.** When you tap Google Pay at a merchant, that **merchant and the card issuer** carry the fraud liability, and Google's role is mostly tokenising the card. **⚠ Not AML** (a different regulatory regime) and **not ad-click fraud** (invalid traffic, where no payment is involved).

**The surface changes the design, so ask which one in minute one:**

| Surface (Google example) | What changes |
|---|---|
| **Digital goods** (Play apps, in-app purchases) | **Delivered instantly**, so there's no window to review before the loss; no shipping address; signals are device, account age, payment instrument. "Friendly fraud" (a child buys gems, the parent disputes). **This sheet's default** |
| **Physical goods** (Google Store hardware) | **Shipping address and drop addresses / reshippers** matter; the delivery delay gives a window for manual review |
| **Cloud sign-ups** (GCP, Workspace) | Stolen cards spin up VMs for crypto-mining; **the loss is compute burned**, not just the chargeback. Free-trial abuse |
| **Ads billing** (Google Ads) | Stolen cards fund campaigns that are often **scams themselves**, a double harm that links to [G27](../G27_policy_violation_review.md) |
| **P2P payments** (Google Pay UPI in India) | Scams where **the real user authorises** the payment: **no chargeback** exists, so the problem becomes detecting social engineering and warning the user |

> ⭐ **Say it in the room:** *"I'll assume card-not-present payment fraud where we're the merchant, like Google Play purchases, so a missed fraud costs us the chargeback. Physical goods, Cloud sign-ups and P2P change the loss model enough that I'd treat them as variants. Tell me if you had one of those in mind."*

**📌 A concrete fraud to picture: stolen cards on Google Play**

```
DAY 0   A fraudster buys 5,000 stolen card numbers on a dark-web market
        (leaked from some retailer's data breach).

DAY 1   CARD TESTING: makes $0.99 purchases with each card to see
        which ones still work. ~800 go through.

DAY 2   Creates 50 fresh Google accounts on emulators / cheap phones,
        adds the working cards, and buys high-value DIGITAL goods:
        in-game currency, Play gift cards, premium app subscriptions.
        Resells them (or the loaded game accounts) for real cash.
        → delivered instantly, so the goods are gone

DAY 30  The real cardholder sees "$99.99 Google Play" on their
        statement, and calls their bank: "I didn't make this."

DAY 35  CHARGEBACK: the bank pulls the $99.99 back from Google,
        plus a dispute fee (~$15–25).
        → Google has lost the money, the fee, AND the goods.
```

**Why it's a good example for this sheet:**
- **The cost of a miss** is the chargeback plus the fee plus the goods, which is the `C_FN` in the cost matrix.
- **The label arrives ~30+ days late.** Today's training data can't contain this attack yet (the delayed-labels problem).
- **Each purchase looks normal on its own:** a $99.99 in-game purchase is legitimate for millions of users. **The graph gives it away:** 50 new accounts sharing a few device fingerprints, all paying with cards first used on the same day. That's commitment #1 below.
- **The cost of a false positive:** a real parent buying a game for their child gets blocked and may never come back. That's the `C_FP` with lifetime value.

**Two other common kinds on the same surface:**
- **Account takeover:** the fraudster logs into a *real* user's account (phished password) and spends the card already saved there. The device is new, but the account is old and trusted.
- **Friendly fraud:** a child buys $500 of gems on a parent's phone, and the parent disputes it as "unauthorised". The card was never stolen, but it's still a chargeback.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Surface, the **cost of each error in currency**, real-time vs batch, investigation capacity (§1) |
| **Explainability** is easy points | **Reason codes** per decline, an immutable decision log, graph evidence for investigators (§8) |
| **Security** beyond the gateway | **Card testing is also model testing**: don't teach attackers your thresholds (§8) |
| **Troubleshoot before being asked** | Raise **outages** (step-up, don't decline) and **"you never learn about your blocks"** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | **ROC-AUC misleads** · both errors are priced in **currency** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · FR/NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The checkout ladder · **PR-AUC** · the **cost-matrix decision rule** | I1c, F1b, I2b |
| **9–12** | **CAPACITY** | — | 100 ms → **a lookup, not a traversal** · entities · ring detection | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Rules vs GBDT vs GNN · the decision stack · fail to step-up | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | ⚠ **You never learn about your blocks** · delayed labels · cost-weighted loss | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | The graduated ladder · queue ordering · precomputed aggregates | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | **Temporal** split · allow-through · champion/challenger · proxy bias | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | P2P scams · the new-pattern path · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"Three properties define this problem, and each has a consequence people get wrong. First, extreme class imbalance — fraud is maybe one in a thousand. The consequence: ROC-AUC is actively misleading here, because its false-positive-rate denominator is the enormous legitimate population, so a model can report 0.99 and be useless. PR-AUC is the right headline.**
>
> **But the more important consequence is the threshold — and this is what fraud gives you that almost no other ML problem does: both error types are denominated in currency. A missed fraud costs the loss plus the chargeback. A false positive costs the declined margin plus — the part usually omitted — the expected lifetime value of a legitimate customer blocked at checkout, who may simply never come back.**
>
> **So the operating point comes from an explicit cost matrix and an expected-value calculation, not from maximising F1. F1 is a statistical convenience that has no idea what a chargeback costs."**

**Then two more commitments:**
1. **Individual transactions are weak signals; the graph is where the fraud is.** A new account making an odd purchase is genuinely ambiguous. **A hundred accounts sharing a device fingerprint and a drop address is not.**
2. **⚠ You never learn whether the transactions you blocked were actually fraud** — no transaction, no chargeback, no label. The training data contains outcomes **only for what you allowed**.

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the buyer and the investigator see and how I set the threshold, capacity, the architecture, data and models, serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
0. **Which surface: digital goods, physical goods, Cloud sign-ups, or P2P?** *(It sets who pays for a miss, and whether there's time to review before the loss.)*
1. **What are the actual costs of each error, in currency?** *(The threshold is an EV calculation — without these it's guesswork dressed as optimisation.)*
2. **Real-time at checkout, or batch post-hoc?** *(Real-time severely constrains graph features.)*
3. **How large is the investigation team?** *(Capacity bounds the review tier, as in moderation.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Surface | **Google Play digital goods**, delivered instantly |
| Fraud rate | ~0.1% of transactions; labels (chargebacks) arrive in **30–90 days** |
| Decision point | Real-time at checkout, **before** authorisation (§🎯) |
| Costs | C_FN and C_FP supplied in currency, with **customer LTV** from the business |
| Investigators | A fixed team; capacity bounds the review queue |
| Compliance | Card data handled under **PCI-DSS**; declines must be explainable |

**FR:** real-time scoring · graph-aware features · **cost-derived thresholds with a graduated action ladder** · prioritised investigation queue · **random allow-through sampling** · fast adaptation · explainable declines.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| Decision latency | p99 ≤ **100 ms** — graph features must be **precomputed** |
| **PR-AUC** | The headline offline metric |
| **Expected loss per transaction** | The real objective, in currency |
| FP rate on legitimate customers | Tracked **LTV-weighted** |
| Random allow-through rate | Small, non-zero, **explicitly budgeted** |
| **Chargeback rate** | Well under the card networks' ~1% monitoring threshold (a cliff, not a slope) |
| Availability | On an outage: allow small purchases, **step up** larger ones — never a blanket decline |

**Scope / NON-GOALS:**
- ❌ **Optimising F1 or ROC-AUC.** Position, with reasons.
- ❌ **Blocking all fraud** — the economically correct fraud rate is **not zero**.
- ❌ Chargeback dispute handling · AML (different regulatory regime).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**What each party sees (b):**

```
BUYER at checkout ($99.99 in-game purchase)
  low risk     → purchase completes instantly (almost everyone)
  medium risk  → "Confirm in your banking app" (3-D Secure STEP-UP)
  high risk    → declined, with a recovery path: verify identity, retry, contact
  (physical goods only: "hold" = the order ships after a quick review)

INVESTIGATOR
  a queue ordered by P(fraud) × exposed value × RING SIZE, with the graph
  around each case and its reason codes; one decision can close a whole ring

FINANCE / RISK OWNER
  expected loss per transaction · approval rate · chargeback rate vs the cliff
```

**The metrics — do this arithmetic out loud:**

```
ONE DAY of purchases (illustrative volume: Google-scale is higher, but
every number below is a RATIO, so the conclusions hold at any volume):

1,000,000 transactions that day · 1,000 of them fraudulent (0.1%)
   (we only KNOW which 1,000 once their chargebacks arrive, 30–90 days later)
Model flags 10,000 of that day's transactions, and 500 of those are fraud.

  ROC view:  FPR = 9,500 / 999,000 = 0.95%   ← "looks excellent"
  PR view:   PRECISION = 500/10,000 = 5%     ← 19 of every 20 flags
                                                is a LEGITIMATE CUSTOMER
  RECALL   = 500/1,000 = 50%

⚠ ROC-AUC's denominator is the ENORMOUS legitimate population, so it is
  structurally insensitive to exactly the errors that hurt.

THE COST MATRIX — this domain's rare gift:
  C_FN = transaction loss + chargeback fee + investigation cost
  C_FP = lost margin + ⚠ EXPECTED LIFETIME VALUE OF A LOST CUSTOMER
                       + support cost + reputational effect

  ACT WHEN:   P(fraud) × C_FN  >  (1 − P(fraud)) × C_FP

⚠ Put a REAL LTV number in and the optimum moves substantially toward
  PERMISSIVENESS vs what an F1-tuned model would pick.
```

> **The organisational point:** *"A team measured on **fraud rate** will systematically over-block, because the cost lands on conversion and retention — somebody else's dashboard."*

**The decision rule, explained:** for each transaction, compare the expected cost of letting it through with the expected cost of blocking it, and do whichever is cheaper.

```
p = the model's probability that THIS transaction is fraud

ALLOW it:  costs C_FN only if it's fraud       → expected cost = p × C_FN
BLOCK it:  costs C_FP only if it's legitimate  → expected cost = (1 − p) × C_FP

BLOCK when   p × C_FN  >  (1 − p) × C_FP
rearranged:  p × (C_FN + C_FP) > C_FP   →   BLOCK when  p > C_FP / (C_FN + C_FP)

WORKED EXAMPLE, a $99.99 Play purchase:
  C_FN ≈ $99.99 + $20 dispute fee + $5 investigation            ≈ $125
  C_FP ≈ $30 lost margin + (20% churn × $250 LTV = $50) + $5     ≈ $85
  threshold = 85 / (85 + 125) ≈ 0.40   → block only if P(fraud) ≥ 40%

  ⚠ Drop the LTV term (C_FP = $35):  35 / 160 ≈ 0.22
    → blocks almost twice as eagerly, losing customers worth more than
      the fraud it stops
```

- **The threshold differs per transaction:** C_FN scales with the amount, so a $0.99 purchase almost never justifies a block, and a $499 phone justifies one at a much lower p.
- **The probabilities must be calibrated:** "0.4" has to mean 40% really are fraud. If training downsampled legitimate transactions, correct the scores back to the true fraud rate first.
- **More actions = the same comparison over more options:** step-up (3-D Secure) is cheap on both sides (a little friction, and the loss shifts to the bank), so it wins across a wide middle band. E.g. step up above p ≈ 0.05, decline only above p ≈ 0.4.
- **Why not F1:** it balances *counts* of errors and has no idea a missed fraud costs $125 while a blocked customer costs $85.

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **Expected loss per transaction** (fraud loss + LTV-weighted false declines) | The real objective, in currency |
| Offline | **PR-AUC** · precision at the review capacity · calibration | Ranking quality and trustworthy probabilities |
| Online | Approval rate · step-up completion · investigation yield per hour | The customer and ops view |
| **Guardrails** | **Chargeback rate** vs the network cliff · decline-rate parity across groups · p99 latency | Hard limits |

---

## 3 · CAPACITY *(min 9–12)* — **one number** `A1b` `I1a` `I1d`

```
⚠ You cannot traverse a billion-node graph in a 100 ms checkout budget.

SO THE ONLINE PATH IS A LOOKUP, NOT A TRAVERSAL:
  · PRECOMPUTE entity-level aggregates continuously —
    per device / address / card / account (ENTITY: defined below the box):
    rolling network statistics in a fast KV store
  · at decision time, LOOK UP and join into the feature vector
  · full graph analysis runs in BATCH (minutes–hours): RING DETECTION
    (defined below the box), aggregate refresh
  · accept SLIGHT STALENESS — ring structure changes over HOURS,
    not milliseconds

(Same shape as G17's precomputed neighbours and G21's feature fetch:
 at low latency and high volume it's a LOOKUP problem with a model attached.)

COST: a GBDT on CPU over ~a few hundred features is ~1 ms; the fetch and
  the batch graph jobs are the bill → track $ per 1,000 decisions
```

**What an "entity" is:** any real-world thing that different transactions can **share**. Each one is a **node** in the fraud graph, and "used together" links them:

| Entity | Identified by | Example precomputed aggregate (rolling window) |
|---|---|---|
| **Account** | Google account ID | Account age · cards added in last 7 d · past chargebacks |
| **Payment card** | Hashed card number (never stored raw) | Distinct accounts it was added to in last 30 d |
| **Device** | Device fingerprint (hardware, OS, app signals) | Distinct accounts that used it in last 24 h |
| **IP / network** | IP, subnet, ASN (data centre vs home broadband) | New accounts created from it today |
| **Email / phone** | Normalised email, phone number | Accounts sharing it |
| **Shipping address** (physical goods) | Normalised address | Distinct cards shipped to it in last 7 d |

**Entity resolution** = deciding two records are the **same** entity: "12 Main St Apt 3" = "12 Main Street #3"; one phone seen again after a reset of its advertising ID. Get it wrong and a ring splits into small, innocent-looking pieces.

```
THE GRAPH AROUND ONE CHECKOUT (the stolen-card example above):

 account A1 ─┐                     ┌─ card C1 (stolen)
 account A2 ─┼─── device D7 ───────┼─ card C2 (stolen)
 account A3 ─┘    (an emulator)    └─ card C3 (stolen)
      │
      └── IP 203.0.113.9 (a data centre, not a home)

At checkout for A3 paying with C3, the model does NOT walk this graph.
It LOOKS UP numbers a batch job already wrote to the KV store:
   device D7: 3 accounts in 24 h, 3 new cards · IP: data centre · ring_risk 0.92
```

**Ring detection** = finding groups of accounts run by **one fraud operation**, through the entities they share. It runs in **batch** over the whole graph:
1. **Build the graph:** accounts linked to the devices, cards, IPs and addresses they used.
2. **Find suspicious clusters:** connected components or community detection (e.g. label propagation): many **new** accounts tied together by **few** devices, cards first seen on the same day.
3. **Score each cluster:** size, density, account ages, and any **confirmed chargebacks inside it**. One confirmed fraud makes all its neighbours suspicious.
4. **Write the result back as entity aggregates** (`ring_id`, `ring_risk` per account, device, card), which checkout reads in milliseconds.

**Why it matters:** a single transaction from A3 looks normal, but one of 50 accounts sharing 3 devices doesn't; you can act on the whole ring at once; and one chargeback on A1 casts suspicion on A2 and A3 weeks before their own chargebacks arrive. ⚠ **Legitimate rings exist** (a family sharing one tablet, a university behind one IP), so combine shared entities with account age, velocity and confirmed labels.

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Option | Strength | Why not / when |
|---|---|---|
| Rules | Ship in hours; auditable | Brittle; attackers learn them |
| **GBDT** ✅ | Tabular, handles imbalance, **explainable** | Leaves some graph signal on the table |
| GNN in the hot path | Strongest graph signal | Unexplainable for declines; slow |
| Unsupervised anomaly detection | Catches **novel** patterns without labels | Noisy on its own |

> *"Rules for speed, GBDT for coverage, GNNs offline producing features, anomaly scores as features and as a signal for investigators."*

### The system — *draw this in 90 seconds* `A2a`

```
══ BATCH (minutes–hours) ════════════════════════════════════════════════════
 ENTITY GRAPH → ring detection · GNN embeddings · anomaly scores
   → written to the KV STORE as entity aggregates (ring_risk, counts, ages)

══ ONLINE · per checkout (≤ 100 ms) ═════════════════════════════════════════
 TRANSACTION / ACCOUNT EVENT
       │
┌─ 1. ENTITY RESOLUTION ─────────────────────────────────────────
│ link this purchase to its account, card (hashed), device, IP, email
│ OUTPUT: entity IDs
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. FEATURE ASSEMBLY — ⚠ A LOOKUP, NEVER A TRAVERSAL ──────────
│ transaction + account history + PRECOMPUTED GRAPH AGGREGATES:
│ component size · shared-attribute counts · distance to known fraud
│ · growth velocity · ring_risk
│ OUTPUT: one feature vector
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MODEL — GBDT (+ rules for known patterns) ─────────────────
│ OUTPUT: calibrated P(fraud) + REASON CODES (top feature attributions)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. COST-DERIVED DECISION ─────────────────────────────────────
│ cheapest action by expected cost, C_FP including LTV:
│ allow → STEP-UP AUTH → hold → decline → RING ACTION
│ + ⚠ RANDOM ALLOW-THROUGH on a small, budgeted sample of would-be blocks
│ OUTPUT: the action shown to the buyer
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. INVESTIGATION QUEUE ───────────────────────────────────────
│ ordered by P(fraud) × exposed value × COMPONENT SIZE;
│ fast-growing components escalated
│ OUTPUT: confirmations at RING level
└──────┬─────────────────────────────────────────────────────────
       ▼
 FEEDBACK: confirmations → labels AND GRAPH SEEDS · chargebacks (30–90 d) → slow truth
```

### ⚠ Raise these yourself, before the interviewer does `A1c` `F2b` `F2a`

- **Outages and timeouts:** *"If the feature fetch or the model times out, I don't decline. Low-value purchases are allowed; higher-value ones get **step-up auth**. A decline caused by an outage is a lost customer for nothing."*
- **You never learn about your blocks:** *"A blocked purchase produces no chargeback and no label, so over-blocking confirms itself. That's why the **random allow-through** is in the design from day one, as a budgeted line item."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Cost matrix, not F1** | "Both errors are priced in currency — the threshold is an EV calculation, and C_FP must include customer LTV" | Requires real LTV numbers from outside the team |
| **2** | **GBDT, not a GNN, in the hot path** | "Tabular, imbalanced, and **explainability is often a regulatory requirement** for declines. GNNs belong offline, producing features" | Leaves some graph signal on the table |
| **3** | **Budgeted random allow-through** | "The only unbiased signal about the population I block — without it, over-blocking is self-confirming" | Real, deliberate fraud loss |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Transactions and outcomes** | Amount, item, time, the action taken | Outcomes exist **only for allowed** purchases |
| **Chargebacks** | The slow truth (30–90 days) | Recent data **under-represents** fraud |
| **Investigator confirmations** | Fast, high-quality labels **and graph seeds** | Limited by capacity |
| **Customer reports** | Fast signal (account takeover often surfaces in hours) | Includes friendly fraud |
| **The entity graph** | Shared devices, cards, IPs, addresses | Entity-resolution errors create guilt by association |
| **The allow-through sample** | The only unbiased view of what we would have blocked | Costs real fraud loss |

### 5.2 Choosing the training set (e) `F1c`

**⚠ The selection-bias problem — volunteer this, most answers miss it:**
```
model blocks a pattern → no transaction → NO CHARGEBACK → NO LABEL
   → the belief is never contradicted → UNFALSIFIABLE
   → over-blocking is SELF-CONFIRMING and invisible

(= G20's bandit-feedback problem with money attached)

MITIGATIONS, in order of value:
  1. RANDOM ALLOW-THROUGH on would-be blocks — costs real fraud loss,
     and is the ONLY unbiased signal → a BUDGETED line item
  2. manual investigation of a sample of blocks — a label without the loss
  3. STEP-UP AUTH instead of blocking — cheapest: completion IS evidence
  4. treat the label set as CENSORED and model it
```

**Delayed labels:** chargebacks arrive **30–90 days** later → fast proxy (investigator confirmation, customer report) + importance weighting + slow truth. Trap: **recent data systematically under-represents fraud** because the chargebacks haven't arrived. **Split by time**: train on the past, test on the future.

### 5.3 The model and the objective (f) `A2b` `F1a`

- **Model:** a **GBDT** on transaction, account and **graph features**, plus rules for known patterns. Features are **network properties, not attributes**: component size, shared-attribute counts, **graph distance to a confirmed-fraud node**, growth velocity.
- **Loss:** **cost-weighted log loss** (fraud examples weighted toward their C_FN), then **calibration**, because the decision rule uses the probability directly.
- **The highest-leverage loop:** a confirmed fraud makes its whole neighbourhood suspicious, so **investigation findings feed back as graph seeds**.

### 5.4 Fine-tune an existing model (f) `F2c`

If a pretrained **transaction-sequence model** exists across Google surfaces (a model of how accounts normally spend), adapt it per surface with small adapters and feed its embedding to the GBDT as a feature. The GBDT stays the explainable decision-maker; the sequence model adds a signal it couldn't build itself.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · EVENTS      transactions, logins, device signals → entity resolution
STAGE 2 · GRAPH       batch ring detection + aggregates → KV store (minutes–hours)
STAGE 3 · LABELS      fast proxies now, chargebacks later; allow-through outcomes
STAGE 4 · TRAIN       cost-weighted GBDT on a temporal split → calibrate
STAGE 5 · DEPLOY      shadow → champion / challenger on a traffic slice → ramp
FAST PATH: a manual RULE ships in hours for a live attack, without a model release
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

**The graduated ladder:**

| Score band | Action | Cost |
|---|---|---|
| Low | **Allow** | None |
| Moderate | **⭐ STEP-UP AUTH** (3DS, OTP, biometric) | Small friction — and it **converts an uncertain block into a cheap test that also generates a label** |
| Elevated | **Hold for review** — delay fulfilment, not the purchase | Latency, not rejection |
| High | **Decline**, with a route to appeal | High — a *customer*, not a transaction |
| Very high + graph evidence | **Account/ring-level action** | Highest cost, highest value |

> **Step-up is the most under-used rung** and earns its place three times: cuts the FP cost, deters the fraudster, **and produces a label** on a case you'd otherwise have blocked blind.

- **The decision objective** `A3b`: per transaction, the action with the **lowest expected cost** (§2), with C_FP including LTV; the investigation queue ordered by `P(fraud) × exposed value × component size`.
- **Efficiency and scale** `F3a` `F3b`: a GBDT on CPU (~1 ms); graph work precomputed in batch; any embedding features stored compressed (quantized) in the KV store.
- **Reliability** `I1b`: on an outage, allow small, step up large; the rules engine as a backstop; versioned model rollouts with fast rollback.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **Headline offline** | **PR-AUC** |
| **The real objective** | **Expected loss per transaction**, in currency |
| **⚠ Temporal split** | Train on past, test on **future**. A random split includes tomorrow's patterns in training and reports performance against an adversary that already moved on |
| **LTV-weighted FP cost** | The customer-side cost, properly priced |
| **Allow-through outcomes** | The **only unbiased estimate** of what's in the blocked population |
| Also | Precision at the operating point · investigation yield per hour · time-to-detect new patterns · **pattern novelty** (share of flagged activity not resembling known clusters) |

**Rollout** `I2a`: shadow scoring → **champion / challenger** on a traffic slice (the challenger decides a few percent of purchases) → ramp. Blocked purchases have no labels, so the **allow-through sample** is what makes the comparison fair.

**Success** = expected loss per transaction down **at the same or better approval rate**, chargeback rate safely under the cliff, decline rates fair across groups. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Selection** | Only allowed purchases get labels | Random allow-through; investigated samples; step-up |
| **Label delay** | Recent fraud not yet labelled | Fast proxies; importance weighting; temporal splits |
| **Adversarial shift** | Fraudsters adapt faster than retraining | Temporal evaluation; graph and velocity features; pattern novelty |
| **⚠ Proxy discrimination** | Postcode, device price or language stand in for protected groups | Decline-rate parity audits; feature review |
| **Guilt by association** | A wrong entity link flags innocent accounts | Precision on links; legitimate-ring checks |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p99 ≤ 100 ms decision; **expected loss per transaction** as the objective; compute cost per 1,000 decisions | p99 · expected loss / txn |
| **Reliability** `I1b` | ⚠ **Fail-open vs fail-closed by amount**: on an outage, allow small, step up large (§4); rules engine as the backstop; versioned model rollouts | Outage-mode decisions · rollback time |
| **Usability** `I1c` | Step-up instead of decline where possible; a **clear path to recover** from a decline (verify, retry, contact) | Step-up completion rate · false-decline complaints |
| **Scalability** `I1d` | Precomputed entity aggregates in a KV store; ring detection in batch; GBDT on CPU | Aggregate freshness · fetch p99 |
| **Privacy** `I1e` | Card data **tokenised / hashed** and handled under **PCI-DSS**; cross-product signals only with consent; retention limits on device fingerprints | PCI audit · data-access logs |
| **Safety** `I1f` | Protect users, not just Google: **scam warnings** on P2P and on purchases that match known scam patterns (elderly users are targeted) | Scam-loss reports |
| **Security** `I1h` | ⚠ **Model probing** via card testing (below); insider access controls; the model and feature store treated as sensitive assets | Probe-pattern alerts |
| **Ethics / bias** `I1i` | ⚠ **Proxy discrimination**: postcode, device price or language can act as proxies for protected groups → audit **decline rates by group**; an appeal route; LTV in C_FP keeps the system from over-blocking | Decline-rate parity · appeal outcomes |
| **Explainability** `I1j` | **Reason codes** per decline (FU 7), mappable to human language; an immutable decision log for regulators; investigators see the graph evidence | % declines with a reason code |

> ⭐ *"A decline is a decision about a person, so it needs a reason, a recovery path and a fairness audit, and on an outage it should default to step-up, not to no."*

> ⚠ **Worth saying in full: don't teach attackers your thresholds.** Card testing is also *model* testing. Rate-limit, add randomness to step-up decisions, and keep exact reason codes internal: customers get a reason category, not the rule.

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extension: P2P scams.** On Google Pay UPI the **real user authorises** the payment, so there's no chargeback and no stolen card. An LLM reading the payment note or the in-app conversation can spot scam scripts. Cost: a model call per payment. Constraint: serious privacy limits, so run it **on-device** or only on high-risk payments, and act with a **warning**, not a block.

**When a new attack pattern appears, the fallback pipeline** `A5b` `F2a`:

```
a new pattern: the model hasn't seen it, chargebacks are 30–90 days away
  → GRAPH ANOMALIES first: a fast-growing component with odd sharing shows up
    before any label exists
  → INVESTIGATORS confirm a few cases → GRAPH SEEDS → the whole ring is suspect
  → a MANUAL VELOCITY RULE ships in hours (no model release)
  → STEP-UP on the matching slice while the evidence builds
  → the next model retrain absorbs the confirmed labels
```

**Top 3 failure modes:**
1. **Over-blocking drift** — self-confirming and invisible without allow-through.
2. **Adversarial adaptation** — fraudsters iterate fast; a random eval split hides it entirely.
3. **Entity-resolution error** — a wrong link creates **guilt by association**; precision on *links* matters as much as on scores.

**Q1 ships:** the **cost matrix with a real customer-LTV number**, PR-AUC evaluation, a GBDT on transaction + account features, and the ladder **with step-up auth**.
> *"Step-up is the cheapest large win — it converts a block into a test and generates a label at the same time. And the cost matrix ships first because it changes the operating point immediately and makes the trade visible to the people who own the customer relationship."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Our fraud model has 0.99 ROC-AUC. Good?"** `F1b`
> It tells you almost nothing at a 0.1% base rate, and I'd treat that number being quoted as the first thing to fix. ROC-AUC's false-positive-rate denominator is the enormous legitimate population, so it's structurally insensitive to exactly the errors that hurt: a million transactions, a thousand frauds, flag ten thousand to catch five hundred gives an FPR under 1% — which looks excellent — while precision is 5%, meaning **nineteen of every twenty flags is a legitimate customer blocked at checkout**. So I'd report **PR-AUC** as the headline and use the precision-recall curve to pick the operating point. And I'd go further: even PR-AUC is a proxy, because the actual objective is **expected loss in currency**, and this is one of the rare domains where both error types can be priced.

**2. "How do you set the threshold?"** `A3b` `F1a`
> From an explicit cost matrix, not from F1 — because F1 is a statistical convenience with no idea what a chargeback costs. Act when `P(fraud) × C_FN > (1−P) × C_FP`. The false-negative cost is easy: loss plus chargeback fee plus investigation. The false-positive cost is where most teams go wrong, because they price it as the declined margin when it's really **the customer**: someone declined on a legitimate purchase frequently doesn't come back and often tells other people. Put a real **expected-lifetime-value** figure into C_FP and the optimum moves substantially toward permissiveness relative to an F1-optimised model. That's the practical consequence, and it explains a common organisational failure: **a fraud team measured on fraud rate will systematically over-block**, because the cost lands on a metric nobody in that team owns.

**3. "You block a transaction. Was it fraud?"** `F1c` `I2a`
> You'll never know, and that's the deepest problem in the system. A blocked transaction generates no chargeback, so it produces no label — the training data contains outcomes **only for transactions you allowed**, and the labelled distribution is the output of your own past policy. If the model believes a pattern is fraudulent and always blocks it, **that belief is unfalsifiable**; it can drift into over-blocking with no signal that would ever contradict it. This is bandit feedback with money attached. Mitigations in order: **random allow-through** on would-be blocks — costs real fraud loss and is the only genuinely unbiased signal, so it's an exploration budget with a very concrete price; **manual investigation of a sample of blocks**, which buys a label without the loss; and **step-up auth instead of blocking**, cheapest of the three since a completed challenge is itself evidence. I'd budget the allow-through explicitly rather than letting it be quietly cut.

**4. "The graph is too big to query in 100 milliseconds."** `A1a` `F3b`
> Correct, and the resolution is that the online path is a **lookup, not a traversal**. I precompute entity-level aggregates continuously — for each device, address, card and account, rolling network statistics in a fast KV store — and at decision time fetch and join them into the feature vector. Full graph analysis runs in **batch**, minutes to hours, doing ring detection and refreshing aggregates. The online features are slightly stale, and that's almost always fine because **ring structure evolves over hours, not milliseconds**; for the fast-growth case a targeted real-time counter on high-risk attributes covers the gap. Same architectural shape as precomputed neighbour lists and ad feature-fetch: at low latency and high volume, it's a distributed-lookup problem with a model attached.

**5. "A completely new fraud pattern appears. How long until you catch it?"** `F2a` `A5b`
> Longer than I'd like, and the honest answer has stages. The model won't catch it initially — it's out of distribution — and **chargeback labels are 30–90 days away**, so the supervised path is slow by construction. What catches it faster: **graph anomalies**, since a new ring still shares infrastructure and a rapidly growing component with unusual attribute sharing is visible before any label exists; **investigator pattern recognition**, since experienced investigators notice novelty far sooner than a model retrains — a strong argument for keeping investigation staffed as a detection capability rather than a cost centre; and **velocity rules** as a blunt fast lever, since a manual rule ships in hours where a model takes weeks. So the design needs a **fast manual rule path** that doesn't require a model release, and I'd monitor **pattern novelty** directly as the leading indicator.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your fraud rate is 0.3% and the CFO wants zero."** `I2b` `A3a`
> Zero isn't the economically correct target, and I'd make that case with the cost matrix rather than by pushing back on principle. Beyond some point the friction required to eliminate the remaining fraud costs more in lost legitimate business than the fraud itself — each increment of recall is bought with a rapidly increasing number of blocked good customers, each costing a **lifetime value**, not a transaction margin. So the target is **minimum expected total cost**, and I'd show the curve: fraud loss falling, customer-experience cost rising, and the sum having a minimum well above zero. The reason this matters organisationally is that fraud rate is the metric most easily owned and optimised, and optimising it drives the system past the economic optimum without anyone noticing — the costs land on conversion and retention, which are somebody else's dashboard. Getting a fraud-rate *target derived from total cost* rather than minimised in isolation is the actual win.

**7. "You have to explain a decline to a regulator. Can you?"** `I1j` `I1i`
> Yes, and that requirement shapes the model choice rather than being bolted on afterwards — which is the main reason gradient-boosted trees remain the workhorse here despite graph neural networks being attractive. In several jurisdictions a declined customer is entitled to a reason, so I need per-decision attributions that are stable and expressible in human terms ("unusual velocity on a new device, shipping to an address linked to prior confirmed fraud") rather than a 512-dimensional embedding distance. Three consequences: **GNNs live offline**, producing ring detections and embeddings that become named features, rather than sitting in the hot path where their contribution is unexplainable; **feature names must be reason-code-mappable** by design; and **the decision path is logged immutably**, because the explanation has to be reconstructable months later when the complaint arrives. There's a fairness dimension too — if the attributions reveal the model is leaning on a proxy for a protected characteristic, I'd rather find that in our own review than in a regulator's.

**8. "Investigators are your bottleneck. Should you just hire more?"** `A3b` `I1d`
> Sometimes, but I'd first argue that investigation is mis-framed as a cost centre when it's actually a **detection capability and a label factory**, and that reframing changes the staffing math. Three things make an investigator-hour unusually valuable here: they act at **ring level**, so one investigation can shut down a hundred accounts rather than one transaction; their confirmations become **graph seeds**, so a single confirmed case propagates suspicion across an entire neighbourhood; and they're the **fastest detector of novel patterns**, which is exactly the failure mode the supervised model is worst at. So before hiring I'd improve the **ordering** — `P(fraud) × exposed value × component size`, with fast-growing components escalated — because the value of a review is enormously variable and per-transaction risk is the wrong key. Then tooling. Then headcount, justified in **expected loss prevented per investigator-hour** rather than as an ops expense.

**9. "Can you use the same model for account takeover and payment fraud?"** `A5a` `A2b`
> I'd resist it, for the same reason harmful-content detection doesn't use one classifier: these are different problems wearing similar clothes. The **label timing** differs — payment fraud surfaces via chargeback in 30–90 days, account takeover often surfaces within hours via a customer report. The **cost asymmetry** differs — a blocked payment loses a transaction and maybe a customer; a missed takeover loses the account and everything in it, and the reputational cost isn't symmetric. The **signal** differs — payment fraud is transactional and graph-structured, takeover is behavioural and session-structured (device change, impossible travel, interaction-pattern shift). And the **actions** differ: step-up auth is the right middle rung for payments, whereas for takeover the right action is often to invalidate sessions and force re-authentication. What I *would* share is the **entity-resolution layer and the graph aggregates**, because the same infrastructure serves both and a confirmed takeover is valuable graph evidence for payment risk. Shared features, separate models, separate thresholds.

**10. "Six months in, your model's performance looks flat. Is it working?"** `I2a` `F1c`
> Flat is ambiguous here in a way it isn't elsewhere, and disentangling it is the interesting part. Flat performance against an **adapting adversary** may mean you're running to stand still — which is success, not stagnation — or it may mean you've stopped detecting a pattern that has moved on and simply don't see it. The measurements that separate those: **temporal-split performance over successive windows**, since degradation shows up there before it shows in aggregates; **pattern novelty**, the share of confirmed fraud that doesn't resemble known clusters, which rising means new patterns are landing; and crucially the **random allow-through outcomes**, because that's the only unbiased window into what's getting through and it's the one number that can't be gamed by your own policy. I'd also check whether **expected loss per transaction** — the actual objective — moved, since PR-AUC can be flat while the cost picture improves or degrades. And I'd be explicit that a fraud model's flat line is not the same as a recommender's: **the baseline is adversarial and moving**, so "no change" requires interpretation rather than reassurance.

---

## ⭐ THE L6 LINE

> **"Both errors are priced in currency — so the threshold is an expected-value calculation with customer LTV inside C_FP, not an F1 optimisation. And you never learn whether the transactions you blocked were fraud, so over-blocking is self-confirming unless you budget a random allow-through."**

## 🧭 HOOKS — *steer by what you mention*

- *"ROC-AUC's denominator is the legitimate population"* → invites the metrics branch
- *"C_FP must include customer lifetime value"* → invites the business/economics branch (**good at L6**)
- *"The graph is where the fraud is"* → invites the features/graph branch
- *"A lookup, not a traversal"* → invites the systems/latency branch
- *"You never learn about your blocks"* → invites the causal-inference / exploration branch (**good at L6**)
- *"The optimal fraud rate is not zero"* → invites the org/leadership branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me redo the cost matrix"* — C_FN vs C_FP regenerates the whole design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the selection-bias problem than finish the feature list. Sound right?"*

**Name and defer:**
- "**Account takeover** is a related but distinct problem — behavioural signals, faster labels."
- "**Graph neural networks** belong offline producing embeddings; explainability keeps them out of the hot path."
- "**Real-time entity resolution** at scale is its own system — fuzzy matching under latency."
- "**Collusion / friendly fraud** (the legitimate customer *is* the fraudster) breaks the graph assumption."
- "**Model-stacking with rules** — rules for speed and auditability, model for coverage."
- "**Cross-institution signal sharing** is powerful and legally complicated."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: a lookup over precomputed graph aggregates | §3, §4 | *"The online path is a lookup, not a traversal."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"p99 ≤ 100 ms; the GBDT is ~1 ms; the fetch is the budget."* |
| **A1c** | Degrade without lowering quality for everyone | §4 | *"On an outage: allow small, step up large — never a blanket decline."* |
| **A2a** | The full stack | §4 diagram | Entity resolution → lookups → GBDT → cost decision → queue → feedback |
| **A2b** | Model components | §5.3 | GBDT on graph features; rules; offline GNN features |
| **A2c** | Data sources | §5.1 | Outcomes, chargebacks, confirmations, reports, graph, allow-through |
| **A2d** | Staged pipeline | §5.5 | Events → graph → labels → train → champion/challenger; the rule fast path |
| **A3a** | Top-down, unprompted | 🎯, §0 | *"Both errors are priced in currency"* + the plan |
| **A3b** | A principled objective for the trade-off | §2, §6 | *"Block when p > C_FP / (C_FN + C_FP), with LTV inside C_FP."* |
| **A4a** | Clarify first, assumptions explicit | 🎯, §1 | Surface · costs in currency · real-time vs batch · investigators |
| **A5a** | Alternatives compared | §4, FU 9 | Rules vs GBDT vs GNN vs anomaly detection |
| **A5b** | A full alternative pipeline on demand | §9 | The new-pattern path: anomalies → seeds → rules → step-up |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Cost-weighted log loss, then calibration, because the rule uses p directly."* |
| **F1b** | Metrics and their trade-offs | §2 | *"ROC's denominator is the legitimate population; 19 of 20 flags are customers."* |
| **F1c** | Bias sources and mitigations | §5.2, §7 | Selection (you never learn about blocks), label delay, proxies |
| **F2a** | The model underperforms → a fallback | §4, §9, FU 5 | New patterns → graph anomalies + rules |
| **F2b** | Latency / outages → tiering | §4 | Fail to step-up, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | A transaction-sequence model adapted per surface |
| **F3a** | Efficiency with a quality trade | §6 | GBDT on CPU; compressed embedding features |
| **F3b** | Scale via precomputation | §3, §6 | Batch ring detection; KV aggregates |
| **F3c** | Extension, with cost and maturity risk | §9 | P2P scams: an on-device LLM, warn not block |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §1, §8 | p99; expected loss per transaction |
| **I1b** | Reliability | §6, §8 | Fail open/closed by amount; rules backstop |
| **I1c** | Usability | §2, §8 | Step-up over decline; a recovery path |
| **I1d** | Scalability | §3, §8 | Precomputed aggregates; batch graph jobs |
| **I1e** | Privacy | §8 | Tokenised cards, PCI-DSS, consent |
| **I1f** | Safety | §8 | Scam warnings protect users |
| **I1h** | Security | §8 | **Card testing is model testing** |
| **I1i** | Ethics / bias | §7, §8 | Proxy discrimination audits; appeals |
| **I1j** | Explainability | §8, FU 7 | Reason codes; immutable decision log |
| **I2a** | Offline → online, correcting for unlabelled blocks | §7 | Temporal split → shadow → champion/challenger, with allow-through |
| **I2b** | The success definition | §7, FU 6 | *"Expected loss down at the same approval rate, under the chargeback cliff."* |

**🧭 Probe map:** (a) assumptions → 🎯 + §1 · (b) product → §2 · (c) stack → §4 · (d) data → §5.1 · (e) training-set selection → §5.2 · (f) objective → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
