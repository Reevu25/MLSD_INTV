# G25 · Harmful Content Detection — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`G25_harmful_content_detection_intv.md`](G25_harmful_content_detection_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **⚠ One of these was a wrong premise, not a gap:** §8. YouTube video is **not** human-reviewed before publishing. Automated checks run pre-publication; **human review is post-publication**.

**Jump:** [1 which platform?](#1--which-platform-is-this-chrome) · [2 near-1 recall, FPs ok?](#2--severe-policy-recall-near-1-false-positives-acceptable) · [3 ⭐ thresholds + capacity](#3--what-is-a-policy-threshold-and-how-does-review-capacity-set-it) · [4 queue depth, precision/recall per policy](#4--queue-depth-and-precisionrecall-per-policy) · [5 PhotoDNA + label/interstitial](#5--photodna-and-labelinterstitial) · [6 band = all policies?](#6--per-policy-thresholds-is-the-band-contributed-by-all-policies) · [7 ⭐ self-harm inverts](#7--why-self-harm-inverts-the-asymmetry) · [8 ⚠ is video pre-publication?](#8---is-video-reviewed-before-publishing)

---

## 1 · Which platform is this? Chrome?

**Not Chrome.** Assume a **large UGC platform**, canonically **YouTube**. Chrome's problem is **Safe Browsing**, which is *URL reputation* (is this site phishing or malware?), not judging user-posted content.

| Surface | Character |
|---|---|
| **Video** | Default. ~4M uploads/day (≈ the sheet's 5M) |
| Comments / live chat | Billions/day, text, latency-critical for live |
| Ads | Pre-publication review → that's **G27** |
| Maps / Play reviews | Text + spam/fake-review flavour |
| Drive / Photos | CSAM hash scanning |
| SafeSearch | **Filter**, not remove |

⭐ **Ask it as clarifying question 0:** *"Which surface: video, comments, ads, reviews?"* It changes volume, latency, and whether review is pre- or post-publication.

---

## 2 · Severe-policy recall near-1. False positives acceptable?

**Why recall-first:** the asymmetry is extreme. A missed CSAM item harms a child again every time it's viewed, and there are **legal reporting obligations** (in the US, to NCMEC). A false positive is a benign item removed until appeal: bad, but recoverable.

⚠ **The nuance: "FPs acceptable" applies to *flagging*, not to irreversible actions.** In 2022 a father's medical photos of his child (taken for a doctor) were flagged as CSAM. His Google account was deleted and a police report filed, even after police cleared him.

```
DETECTION:  near-1 recall — flag aggressively, accept many FPs
  ↓
ACTION:     irreversible steps (account deletion, law-enforcement
            report) → HUMAN VERIFICATION first
```

> ⭐ **High recall at detection, high precision before irreversible action.** Two different thresholds. Conflating them turns an aggressive detector into an unjust system.

---

## 3 · What is a policy threshold, and how does review capacity set it?

Each policy `p` has its own classifier score `s_p(item) ∈ [0,1]` and **two thresholds**, giving three zones:

```
         t_low                  t_high
  0 ──────┼──────────────────────┼────── 1
  AUTO-ALLOW │  BORDERLINE BAND   │ AUTO-ACTION
             │  → human review    │
```

| Threshold | Set by | Why |
|---|---|---|
| **`t_high`** | **Precision** (e.g. ≥99% on a labelled set) | No human in the loop, so it must be right |
| **`t_low`** | **Recall** target (severity-dependent) | Anything below is never reviewed, so misses ship |
| **Band** | Whatever's left | Its size = reviewer workload |

**⚠ The constraint:**

```
Σ_p (items in policy p's band per day)  ≤  review capacity (e.g. 30k/day)
```

If the band produces more than capacity, the queue grows forever. To fit, you trade:

| Lever | Cost |
|---|---|
| Raise `t_low` | Lower **recall** (more harm ships) |
| Lower `t_high` | Lower **precision** (more wrongful removals) |
| Order the queue by `P(viol) × reach × severity` | Low-value items wait / age out |

⭐ **So thresholds are not picked off a PR curve in isolation.** They are a **constrained optimisation: maximise harm prevented, subject to band ≤ capacity.** That's *"capacity sets thresholds."*

**Example:** hate band 50k + spam band 40k vs 30k capacity. Spam is low severity → raise spam's `t_low` (accept lower spam recall), spend the reviews on hate.

---

## 4 · Queue depth, and precision/recall per policy

**Yes, the human review queue.** *Depth* = items waiting; *age* = how long the oldest has waited. Unbounded depth ⇒ the band outruns reviewers ⇒ thresholds are wrong (§3).

```
precision_p = flagged-for-p AND truly p  ÷  all flagged-for-p
              "when we flag hate speech, how often is it really hate speech?"

recall_p    = truly p AND flagged  ÷  all truly p
              "of all the hate speech out there, how much did we catch?"
```

**Yes, it's the model's flagging,** with two clarifications:

- **Two levels.** *Model-level* (classifier at its thresholds) vs *system-level* (after human review, which is what actually ships). Reviewers fix some model errors and add some of their own.
- ⚠ **Recall needs a random sample.** Precision you can measure from what you flagged. Recall you **can't**, because you never saw what you missed. Label a **random sample of all content** to estimate how much violating content exists. Same sample gives **prevalence**, which is why prevalence sampling is a standing cost.

**Why per policy, never aggregated:** prevalence spans ~6 orders of magnitude across policies. Aggregate precision is dominated by spam and says nothing about CSAM.

---

## 5 · PhotoDNA, and label/interstitial

**PhotoDNA** = Microsoft's **perceptual hash** (with Dartmouth, 2009) for known CSAM.

| | Cryptographic hash (SHA-256) | Perceptual hash (PhotoDNA) |
|---|---|---|
| Change one pixel | Completely different hash | ~same hash |
| Resize / recompress / light crop | Different | **Still matches** |
| Answers | "Exact same file?" | **"Visually the same image?"** |

**How it's used:** NCMEC and others keep hash DBs of **known, confirmed** material. Hash every upload, check against the DB. A match is near-certain detection: **no ML, and no human has to view it again.**

"PhotoDNA-style" = *perceptual hash vs known-bad DB*. Siblings: **CSAI Match** (Google, video), **GIFCT** hash-sharing DB (terrorist content, cross-company).

⭐ **Why Stage 0:** near-perfect precision, essentially free, spares reviewers. **Limit: known content only.** Novel content needs the ML stages.

**Label / interstitial:** the middle rungs, where content **stays up with context or friction**.

- **Label:** context attached, e.g. an info panel, "graphic content," "state-affiliated media."
- **Interstitial:** a **click-through warning screen before viewing**: *"This video may be disturbing. [Proceed]."* Still available, but nobody sees it by accident.

For **legitimate but sensitive** content (news violence, medical, documentary): removing it is wrong, but so is showing it unannounced.

```
ACTION LADDER: nothing → reduce reach → label/interstitial → remove → account action → report
               (least severe, most reversible)                     (most severe, least reversible)
```

---

## 6 · Per-policy thresholds: is the band contributed by all policies?

**Yes on both.** That's why capacity is an **allocation** problem.

```
band(hate)      → 18k/day ┐
band(harass)    → 9k/day  │
band(self-harm) → 4k/day  ├─► ONE shared review queue — 30k/day
band(spam)      → 25k/day │   56k arriving > 30k capacity
band(...)       → ...     ┘   → which t_low to raise? what to review first?
```

- **Allocate by marginal value:** where does one more review prevent the most harm? Severe policies get priority; low-severity ones get a higher `t_low`.
- ⚠ **One item can be in several bands** (borderline harassment *and* hate). **Dedupe and route once**, carrying all candidate policies, or it costs capacity twice.
- ⚠ **Capacity isn't one pool.** Reviewers specialise by **language** (Thai queue needs Thai reviewers) and **category** (CSAM needs specially trained teams with wellbeing support). So it's several queues, and a policy's threshold can be limited by *its* team's capacity.

---

## 7 · Why self-harm inverts the asymmetry

**For most policies, removal is the safe direction:** a false negative means harm reaches viewers.

**Self-harm breaks that, because of who is posting:** often **a person in crisis**.

- Removing their post can **silence someone reaching out**: isolating them at the worst moment, signalling their experience is unspeakable.
- Aggressive removal also catches **recovery content** (people sharing how they got through it, support communities), which is among the most valuable material on the topic and looks similar to a classifier.

So over-removal **causes harm**. A different ladder:

| Content | Action |
|---|---|
| Person expressing distress | ⭐ **Crisis resources panel**, keep it up |
| Recovery stories / support communities | **Keep up** (maybe a resources panel) |
| Graphic depictions | **Reduce reach / interstitial** |
| **Method instructions / encouragement** | **Remove** — the genuinely harmful part |
| Signals of imminent risk | **Escalate** to a specialised team |

**Viewer side:** **suicide contagion** is real, so even acceptable content is often **not recommended**. *Hosting ≠ amplifying.*

> ⭐ **The "safe" direction depends on the policy.** Severity alone doesn't set the threshold; **who is harmed by each kind of error** does.

---

## 8 · ⚠ Is video reviewed before publishing?

**No, that was my wrong premise.** ~4M uploads/day vs tens of thousands of reviewer-hours: **human review of every video before publishing is impossible.**

What actually happens:

```
UPLOAD
  → PROCESSING (transcoding, seconds–minutes)
       ⭐ AUTOMATED screening BEFORE public:
          hash matching (CSAI Match, Content ID), high-confidence classifiers
       → clear violations BLOCKED before anyone sees them
  → PUBLISHED
  → POST-PUBLICATION:
       ongoing classifiers · user flags · view-weighted prioritisation
       → borderline band → HUMAN REVIEW QUEUE
```

| | Pre-publication | Post-publication |
|---|---|---|
| **Automated** | ✅ hash + high-confidence classifiers | ✅ ongoing |
| **Human** | ❌ (except below) | ✅ the borderline band |

⭐ **This is why view-weighted prevalence is the headline metric:** harm accumulates while a borderline video sits in the queue, so order the queue by **expected reach**.

**True pre-publication human review** exists only where volume is low and stakes high: **ads (G27)**, monetisation, some high-risk categories or untrusted/new accounts. Hence *"pending default by risk tier × actor trust."* That's the G25 ↔ G27 boundary.

**Why video cascades hard:** frames + audio + transcript is expensive, so most videos get only the cheap pass; the expensive multimodal models run on the uncertain slice.

---

## 🔁 THE EIGHT THINGS I KEPT GETTING WRONG

1. **The platform is YouTube-style UGC, not Chrome.** Chrome is Safe Browsing (URL reputation). Ask *"which surface?"* first.
2. ⚠ **"FPs acceptable" is for flagging, not for irreversible action.** High recall at detection, human-verified precision before account deletion or a law-enforcement report.
3. ⭐ **Two thresholds per policy:** `t_high` by precision (auto-action), `t_low` by recall (never-reviewed). The band between is reviewer workload.
4. ⭐ **Capacity sets thresholds:** Σ bands across policies ≤ capacity, so it's a **joint constrained optimisation**, not independent PR points per policy.
5. **Queue depth = the human review queue.** Unbounded depth means the thresholds are wrong, not that you need patience.
6. ⚠ **Recall can't be measured from what you flagged.** It needs a random labelled sample, which is also how prevalence is measured. Always per policy.
7. **PhotoDNA = perceptual hash** (survives resize/recompress) against known-bad DBs. **Interstitial = click-through warning**; content stays up.
8. ⚠ **Video is post-publication for humans.** Automated checks run before going live; human review happens after, ordered by reach. Pre-publication human review is ads (G27). **Self-harm** inverts the usual asymmetry: resources, not removal.
