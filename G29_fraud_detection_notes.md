# G29 · Fraud Detection — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G29_fraud_detection_intv.md`](G29_fraud_detection_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **Already answered in the sheet itself** (🎯 WHICH PLATFORM?): which platform we assume (Google as the merchant, e.g. Play purchases), a concrete stolen-card fraud timeline, and the purchase → settlement → chargeback money flow.

**Jump:** [1 ⭐ why PR-AUC, not ROC-AUC](#1--why-pr-auc-and-not-roc-auc) · [2 ⭐ the decision rule: when to block](#2--the-decision-rule-when-to-block) · [3 entities + ring detection](#3--what-is-an-entity-and-what-is-ring-detection)

---

## 1 · Why PR-AUC, and not ROC-AUC?

**Short answer:** ROC-AUC divides false positives by the **huge number of legitimate transactions**, so it barely moves when the model blocks thousands of real customers. PR-AUC divides by **what the model flagged**, which is exactly what reviewers and blocked customers experience.

### The two curves, side by side

| | ROC curve | PR curve |
|---|---|---|
| Axes | TPR (recall) vs **FPR** | **Precision** vs recall |
| False-positive term | **FPR = FP ÷ all legitimate** (≈ 999,000) | **Precision = TP ÷ all flagged** (TP + FP) |
| Asks | "What fraction of legit customers did we bother?" | "Of the ones we flagged, how many were really fraud?" |
| Random model scores | **0.5**, whatever the fraud rate | **= the fraud rate** (0.001 here) |

### The worked numbers (from the sheet)

```
ONE DAY: 1,000,000 transactions · 1,000 fraudulent (0.1%)
(fraud labels for that day arrive as chargebacks, 30–90 days later)
Model flags 10,000 and catches 500.

  ROC view:  FPR       = 9,500 / 999,000 = 0.95%   ← "looks excellent"
  PR view:   PRECISION = 500 / 10,000    = 5%      ← 19 of every 20 flags
                                                     is a REAL CUSTOMER
             RECALL    = 500 / 1,000     = 50%
```

### ⭐ Where ROC hides the difference that matters

Two models catch the same 500 frauds. Model B blocks half as many real customers:

```
              false positives   FPR (ROC)        PRECISION (PR)
  Model A         9,500          0.95%               5.0%
  Model B         4,750          0.48%               9.5%
                                 ↑ both "tiny":       ↑ nearly DOUBLED:
                                 ~0.5 pp apart        4,750 fewer blocked
                                                      customers, half the
                                                      review workload
```

On ROC the two look almost the same, because both FPRs sit at the far left of the curve. On PR the gap is obvious, and that gap is thousands of real customers declined at checkout.

### Three more reasons

1. **Only the top of the ranking matters.** We act on the riskiest ~1% of transactions. ROC-AUC averages over **every** threshold, including ones where we'd flag half of all customers, which we would never run. PR concentrates on the high-score region we actually operate in.
2. **Most legitimate transactions are easy.** A grocery-sized purchase on a 10-year-old account is trivially "not fraud". ROC-AUC gets lots of credit for ranking fraud above those, which inflates it toward 0.99 even when the model is weak on the hard cases.
3. **PR-AUC has an honest baseline.** A random model gets PR-AUC ≈ 0.001 here, so 0.20 means "200× better than random". A ROC-AUC of 0.97 sounds great but doesn't tell you whether precision is 50% or 2%.

### ⚠ Caveats worth saying

- **PR-AUC depends on the fraud rate.** If fraud prevalence drifts from 0.1% to 0.2%, PR-AUC rises with no model change. Compare models **on the same evaluation window**, and report prevalence alongside.
- **Evaluate at the true prevalence.** If you downsampled legitimate transactions for training, the eval set must still have the real 1-in-1,000 mix, or precision comes out wildly optimistic.
- **Neither is the objective.** PR-AUC is the **offline ranking metric**. The **operating point** comes from the cost matrix (`act when P(fraud) × C_FN > (1 − P(fraud)) × C_FP`), and the business metric is **expected loss in currency**.
- **Often more useful than either AUC:** **precision at the review capacity** (e.g. precision in the top 2,000 flags/day), or **recall at a fixed precision**. These are single points on the curve you'll actually run at.

> ⭐ **Say:** *"ROC-AUC's false-positive rate divides by the million legitimate customers, so blocking 9,500 of them shows up as under 1%. Precision divides by what we flagged, and it says 19 of every 20 flags are real customers. With 0.1% fraud, PR-AUC is the honest headline. But the threshold still comes from the cost matrix, not from either curve."*

---

## 2 · The decision rule: when to block

`ACT WHEN:  P(fraud) × C_FN  >  (1 − P(fraud)) × C_FP`

**In words:** for each transaction, compare the **expected cost of letting it through** with the **expected cost of blocking it**, and do whichever is cheaper.

```
p = the model's probability that THIS transaction is fraud

If we ALLOW it:  we lose C_FN, but only if it IS fraud     → expected cost = p × C_FN
If we BLOCK it:  we lose C_FP, but only if it's LEGITIMATE → expected cost = (1 − p) × C_FP

BLOCK when blocking is cheaper in expectation:   p × C_FN  >  (1 − p) × C_FP
```

| Term | Meaning | What's in it |
|---|---|---|
| **C_FN** | Cost of a **false negative**: we allowed a fraud | The amount (refunded via chargeback) + the dispute fee + investigation cost |
| **C_FP** | Cost of a **false positive**: we blocked a real customer | The lost margin on the sale + **the chance they never come back × their lifetime value** + the support call |

### Rearranged, it's just a threshold

```
p × C_FN  >  C_FP − p × C_FP
p × (C_FN + C_FP)  >  C_FP

        BLOCK when   p  >  p* = C_FP / (C_FN + C_FP)
```

### Worked example: a $99.99 Play purchase

```
C_FN = $99.99 lost + ~$20 dispute fee + ~$5 investigation       ≈ $125
C_FP = ~$30 lost margin
       + 20% chance a wrongly-declined customer leaves × $250 LTV  = $50
       + ~$5 support contact                                     ≈ $85

p* = 85 / (85 + 125) ≈ 0.40   → block only if the model says ≥ 40% fraud

⚠ Drop the LTV term (C_FP = $35):  p* = 35 / 160 ≈ 0.22
   → blocks almost twice as eagerly, and loses customers worth more than
     the fraud it stops. That's the sheet's "put a real LTV number in and
     the optimum moves toward PERMISSIVENESS".
```

### Four consequences

1. **There's no single global threshold.** C_FN scales with the amount, so p* differs per transaction. A $0.99 purchase barely justifies blocking at any p; a $499 Google Store phone (goods shipped, can't be recovered) justifies blocking at a much lower p.
2. **The model's probabilities must be calibrated.** The rule plugs `p` straight into a cost formula, so "0.4" has to mean "40% of transactions scored 0.4 are fraud". If training downsampled the legitimate class, correct the scores back to the true rate first (the same correction as [G21 notes §10](G21_ad_click_prediction_notes.md#10--negative-downsampling-correction--the-derivation)), then check calibration with Platt or isotonic scaling.
3. **More actions = the same comparison over more options.** Each rung of the ladder has its own expected cost; pick the cheapest:

   | Action | Expected cost |
   |---|---|
   | Allow | `p × C_FN` |
   | Step-up (3-D Secure) | `(1 − p) × friction cost` (a few buyers abandon) + `p × small residual` (the liability shifts to the bank) |
   | Send to review (if delivery can wait) | `review cost` + delay cost |
   | Decline | `(1 − p) × C_FP` |

   Step-up is cheap on both sides, so it wins across a wide middle band. You might step up from p ≈ 0.05 and only decline above p ≈ 0.4.
4. **This is why not F1.** F1 picks the threshold that balances precision and recall as *counts*. It has no idea a missed fraud costs $125 and a blocked customer $85, so its threshold has no relation to money.

> ⭐ **Say:** *"For each transaction I compare the expected cost of allowing it, p times the fraud cost, with the expected cost of blocking it, one minus p times the cost of losing a good customer. That gives a threshold of C_FP over C_FN plus C_FP. It's different per transaction because the amount changes C_FN, and it only works if the probabilities are calibrated."*

---

## 3 · What is an "entity", and what is ring detection?

**Entity** = any real-world thing that different transactions can **share**: an **account**, a **payment card** (hashed), a **device** (fingerprint), an **IP / network**, an **email / phone**, a **shipping address**. Each is a node in the fraud graph; "used together in a transaction or login" is an edge.

**Entity aggregates** = rolling numbers per entity, precomputed in batch and stored in a KV store, e.g. *"device D7: 3 distinct accounts in the last 24 h"*, *"card C3: added to 5 accounts in 30 days"*. Checkout **looks these up** in milliseconds instead of walking the graph.

**Entity resolution** = deciding two records are the same entity ("12 Main St Apt 3" = "12 Main Street #3"). Get it wrong and a ring splits into innocent-looking pieces.

**Ring detection** = finding groups of accounts run by **one fraud operation**, through the entities they share. A batch job over the whole graph:

```
build the graph → find clusters (connected components / community detection):
   many NEW accounts tied together by FEW devices, cards first seen the same day
→ score each cluster (size, density, account ages, confirmed chargebacks inside)
→ write back ring_id + ring_risk per account / device / card → read at checkout
```

**Why it matters:** one purchase from a new account looks normal, but one account in a 50-account ring sharing 3 devices doesn't. It lets you **block the whole ring at once**, and **one confirmed chargeback casts suspicion on the rest of the ring weeks before their own chargebacks arrive**, which softens the delayed-label problem.

⚠ **Legitimate rings exist** (a family sharing a tablet, a university behind one IP), so combine shared entities with account age, velocity and confirmed labels.

> ⭐ **Say:** *"Individual transactions are weak signals; the graph is where the fraud is. I precompute per-entity aggregates and ring scores in batch, and checkout only looks them up."*
