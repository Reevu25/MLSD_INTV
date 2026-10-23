# R12 · App-Store App Recommendations — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md)–[R11](R11_chatbot_structured_unstructured_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **Expect these probes:** (a) clarifying questions · (b) surfaces and what the user sees · (c) architecture, offline + online · (d) candidate generation and ranking · (e) features and model choices · (f) labels and metrics · (g) real-time adaptation · (h) cold start, exploration, feedback loops · (i) sponsored placements and policy · (j) monitoring, A/B tests, reliability.
>
> **What's reused, and what's new.** [G15](G15_play_store_recommendations_intv.md) is the deep dive on this domain's **labels**, and R12 reuses it by reference: **install measures the listing, not the app** (G15 §2), the **delayed-label bridge** with retention-weighted installs (G15 §5.2), **PMI-normalised co-installs** against popularity skew, **content features for permanent cold start**, the **funded new-app exploration quota**, and **category baselines** (G15 FU 6). **New here**, because this prompt asks for it: **real-time adaptation within minutes**, **new-user cold start**, **sponsored placements** blended safely with organic results, **category pages** (and search as an extension), explicit **feedback-loop mitigation**, and the **monitoring and reliability** plan.
>
> **Related sheets:** [R6 real-time recsys](R6_realtime_recsys_intv.md) (streaming features, freshness tiers) · [R4 watch-next](R4_watch_next_losses_optimizers_intv.md) (losses, MMoE, position bias) · [R5 product or video](R5_product_or_video_recsys_intv.md) (sponsored slots, self-preferencing) · [G21 ad click prediction](G21_ad_click_prediction_intv.md) (the ads auction) · [G17 similar items](G17_similar_items_intv.md) (PMI).

---

## 📋 THE QUESTION (verbatim)

> **R12. Design an app-store app recommendation system**
>
> You are building an app recommendation system for a mobile app store.
>
> **Goal**
> Recommend apps to a user on surfaces such as:
> - Home feed / "Recommended for you"
> - Category pages
> - Search results (optional extension)
>
> **Requirements (assume if not specified)**
> - Personalized ranking for each user.
> - Cold start support for new users and new apps.
> - Real-time adaptation to recent user actions (clicks/installs) within minutes.
> - Business constraints: respect policy/safety filters and optionally support sponsored placements.
>
> **Input signals**
> You may use:
> - User events: impressions, clicks, installs, uninstalls, time spent, ratings
> - App metadata: category, text description, tags, developer, price, locale, device compatibility
> - Context: country, language, device type, time
>
> **Output**
> For a given user and context, produce a ranked list of apps (top-K) with latency suitable for an online product.
>
> **Describe:**
> - Overall architecture (offline + online)
> - Candidate generation and ranking approach
> - Feature engineering and model choices
> - Data/label definitions and evaluation metrics
> - Cold-start, exploration, and feedback-loop mitigation
> - Monitoring, A/B testing, and reliability considerations

### The question's checklist, and where each item is earned

| The question asks for | Where |
|---|---|
| Surfaces: home, category pages, (search) | §2, §6.1, §9 |
| Personalised ranking per user | §4, §5.3, §5.4 |
| Cold start: new users **and** new apps | §5.5 |
| Real-time adaptation within minutes | §5.6, FU 2 |
| Policy and safety filters | §4 diagram (box 4), §8 |
| Sponsored placements | §6.2, FU 5, FU 8 |
| The given input signals, turned into features | §5.1, §5.2 |
| Top-K at online latency | §3 |
| Architecture, offline + online | §4 |
| Candidate generation and ranking | §5.3, §5.4 |
| Labels and evaluation metrics | §2, §7 |
| Exploration and feedback-loop mitigation | §5.7 |
| Monitoring, A/B testing, reliability | §6.4, §7 |

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | *"Installs, retention or revenue: what are we judged on?"* and *"How must sponsored apps relate to organic ranking?"* (§1) |
| **Explainability** is easy points | A reason on every card, an **"Ad" label** on sponsored ones, and **per-app impression attribution** for developers (§8) |
| **Security** beyond the gateway | **Install farms, fake reviews and bot-faked opens** poisoning the labels; developers gaming metadata (§8) |
| **Troubleshoot before being asked** | Raise **the install trap**, **already-installed apps reappearing**, and **sponsored apps crowding out organic value** yourself (§4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Install measures the listing** · real-time in features, not weights · ads in separate, labelled slots | A1a, A3a |
| **1–5** | **REQUIREMENTS** | **(a)** | Objective · surfaces · sponsored policy · assumptions · non-goals | A4a, A1b |
| **5–8** | **PRODUCT + LABELS** | **(b) (f)** | Three surfaces · the label ladder · D7-retained installs per visit | I1c, F1b, I2b |
| **8–10** | **CAPACITY + BUDGETS** | — | Small corpus, popularity skew · ~3k QPS · p95 ≤ 200 ms | A1b, I1a, I1d |
| **10–18** | **ARCHITECTURE** | **(c)** | Offline + nearline + online · sources → ranker → re-rank + ads · problems raised yourself | A2a, A5a, A1c, F2b |
| **18–32** | **DATA, FEATURES, MODELS, COLD START, REAL-TIME, LOOPS** | **(d) (e) (g) (h)** | Signals → features · two-tower + PMI + content · MMoE · **new users and apps** · **minutes** · exploration | A2b, A2c, A2d, F1a, F1c, F2c |
| **32–37** | **SERVING: SURFACES, ADS, SCALE** | **(i)** | Home vs category · **sponsored slots** · caching · fallback | F3a, F3b, A3b, I1b |
| **37–44** | **EVALUATION, A/B, MONITORING** | **(j)** | Matured-cohort gate · proxy↔truth · ads holdback · monitors | I2a, I2b, F1b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | — | Search · re-engagement · retention-floored fallback · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"First, the label. Installs are the obvious target and the wrong one: an install measures the store listing, the icon and the screenshots, not whether the app is any good, and an install-optimised store fills with apps people uninstall the next morning. So I'd optimise retained installs: install, then opened on day one, then still used at day seven, with an uninstall within 24 hours as an explicit negative. Retention arrives late, so I train on installs weighted by how often similar installs were retained, and gate releases on the real retention once it matures.**
>
> **Second, 'adapt within minutes' is a feature problem, not a retraining problem. A click or an install streams into the user's session features within seconds, an installed app disappears from the candidates immediately, and the candidates are recomputed on strong events like installs. The models retrain daily; the freshness lives in the features.**
>
> **Third, two business constraints shape the final list: policy and safety filters are hard constraints applied before ranking, not scores to trade; and sponsored apps come from a separate auction into fixed, labelled slots, held to the same filters and a relevance floor, so ad spend never quietly bends the organic ranking."**

**Then say the plan (`A3a`):** *"Clarifying questions, the surfaces and the label ladder, capacity, the architecture, then signals and features, candidate generation and the ranker, cold start for new users and new apps, real-time adaptation, exploration and feedback loops, then serving with sponsored slots, evaluation and monitoring, a production-readiness sweep, and extensions."*

---

## 1 · REQUIREMENTS *(min 1–5)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **What are we judged on: installs, retention, or revenue?** *(If installs, renegotiate before designing; G15 FU 1.)*
2. **Which surfaces first?** *(Assume the home feed, then category pages; search as an extension.)*
3. **How must sponsored apps relate to organic results?** *(Assume a separate auction, fixed labelled slots, a relevance floor, no influence on organic ranking.)*
4. **Is new-app (developer) reach a goal?** *(Decides the size of the exploration budget.)*
5. **Are minors among the users, and which regions?** *(Age-appropriate filtering; regional rules for gambling, loans; EU rules on self-preferencing.)*
6. **Which usage signals may we use?** *(Time spent and opens only with consent; the on-device installed-apps list only where policy allows.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Users | ~10⁹ devices; ~10⁸ home-feed loads + ~10⁸ category-page loads a day |
| Catalogue | A few million apps; thousands of new ones a day |
| Output | Home: ~10 rows of ~10 apps; category page: a ranked list of ~50 |
| Signals | The prompt's events (impressions, clicks, installs, uninstalls, time spent, ratings), metadata and context |
| Sponsored | Up to 2 labelled ad slots per row of 10, from a separate ads system |
| Freshness | Clicks and installs reflected **within minutes**; a new app rankable **within hours** |

**FR:** personalised top-K per user, surface and context · policy, safety, age and device-compatibility filters · already-installed apps removed · cold start for users and apps · real-time adaptation · sponsored slots · a reason per card · "not interested".

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Latency** | **p95 ≤ 200 ms** per surface request |
| **Freshness** | Install / uninstall → candidates updated **≤ 1 min**; clicks → session features **≤ 10 s** |
| **Gating metric** | **D7-retained installs per visit**, matured cohort, category-relative (G15) |
| **Policy** | Policy-violating, malware-flagged or age-inappropriate impressions **= 0** |
| **New-app reach** | A monitored guardrail |
| Availability | 99.9%; fallback to **retention-floored charts**, never the raw install chart |

**Scope / NON-GOALS:**
- ❌ **Optimising installs** (G15).
- ❌ **Per-event online learning** of the whole model: real-time adaptation through features is enough (R6).
- ❌ The ads auction itself (G21) and app review / policy enforcement (upstream).

---

## 2 · PRODUCT + LABELS *(min 5–8)* — probes **(b) (f)** `I1c` `F1b` `I2b`

**What the user sees, per surface:**

```
HOME FEED
  Recommended for you
  [ Running app ]  [ Budget app ]  [ Ad · Fitness tracker ]  [ Puzzle app ]  …
     "Because you installed Strava"                ↑ labelled, fixed slot
  New & notable for you                           ← EXPLORATION row (new apps)
  [ new app D ]  [ new app E ]  [ new app F ]
  Top in Health & Fitness                         ← category row: diversity
  long-press any card → "Not interested"          ← an immediate negative

CATEGORY PAGE: "Productivity"
  1  Notes app       ★4.6   personalised WITHIN the category, quality-weighted
  2  Calendar app    ★4.4
  3  Ad · Task app          ← labelled slot
  …  sub-category diversity (notes, calendars, to-do, email)

AFTER AN INSTALL (minutes later): the installed app is gone from every row;
  "People who installed Strava also kept…" appears near the top
```

**The label ladder** (G15 §2): each rung is better and arrives later.

```
IMPRESSION → CLICK (details page) → INSTALL → D1 OPEN → D7 RETAINED → D30 RETAINED
 immediate ←────────────────────────────────────────────────────→ 30 days
 measures the LISTING                          measures the APP
NEGATIVES: uninstall within 24 h (explicit) · "not interested" · a low rating
  · repeated impressions without a click
TIME SPENT and RATINGS (with consent): quality signals that refine D7/D30
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **North star** | **D7-retained installs per visit**, matured cohort, **relative to the category baseline** | The apps were worth installing |
| Fast proxy | Install + D1 open, **importance-weighted** by historical retention (G15 §5.2) | Iteration speed |
| Offline | Recall@k (retrieval) · NDCG@k on retention-weighted labels · per-head AUC and calibration | Screens, never decides |
| Ecosystem | **New-app reach** · category concentration · catalogue coverage | Is the loop narrowing? |
| Sponsored | Ad revenue · ad install-and-retain rate · **organic installs displaced** | Ads earn their slots |
| **Guardrails** | Uninstall-within-24 h · policy impressions = 0 · p95 latency · **proxy↔truth correlation** | Ship blockers |

> **Say:** *"Installs are available; retained installs are meaningful. I iterate on installs weighted by how often similar ones were kept, and I gate on real D7 retention once it matures."*

---

## 3 · CAPACITY + BUDGETS *(min 8–10)* `A1b` `I1a` `I1d`

```
TRAFFIC   ~2×10⁸ surface loads/day (home + category) ≈ 2.3k QPS avg, ~7k peak
EVENTS    ~10⁸ installs/day ≈ 1.2k/s · ~10⁹ impressions/day · clicks between
CORPUS    a few million apps: small next to 10⁹ videos
          → app embeddings: 5×10⁶ × 64-d × 4 B ≈ 1.3 GB → in memory per serving node
          → retrieval is NOT the hard part (G15 §3); labels, cold start and
            popularity skew are

THE BUDGET (p95 ≤ 200 ms; ~120 ms planned)
  user + session features (one batched read)                  15 ms
  candidate sources in parallel (ANN over ~5×10⁶ + lookups)    25 ms
  filters: installed · incompatible · policy · age · region    5 ms
  pre-ranker (2,000 → 400) + MMoE ranker (400)                 50 ms
  re-rank: diversity, quotas, exploration, sponsored slots     10 ms
  network / serialisation                                      15 ms

RANKER LOAD: ~7k QPS × 400 candidates ≈ 2.8×10⁶ scorings/s at peak → the cost line
```

---

## 4 · ARCHITECTURE *(min 10–18)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Objective | Installs · revenue · **retained installs via the delayed-label bridge** | G15 §4 |
| Retrieval | Two-tower on IDs only · **two-tower with content features in the app tower + PMI co-install + other sources** | Content lets new apps be retrieved on day one |
| Ranker | GBDT · **MMoE multi-task DNN with negative heads** | Conflicting heads (install vs uninstall); real-time sequence features |
| Real-time adaptation | Retrain every few minutes · **streaming features + event-triggered candidate refresh** | Most of the freshness value at a fraction of the risk (R6) |
| Policy | A penalty term in the score · **hard filters before ranking** | A finite penalty eventually loses to a big enough engagement gain |
| Sponsored | Boost paid apps in the organic score · **a separate auction into fixed, labelled slots** | Disclosure, and an organic ranker that measures user value honestly |
| Surfaces | One ranker everywhere · **one ranker with a surface feature + per-surface re-ranking** | Shared learning, different intent and layout |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE (daily) ══════════════════════════════════════════════════════════════
 events (impressions WITH position + surface, clicks, installs, opens, uninstalls,
   time spent, ratings) + app metadata + policy status
   → fraud removal → labels (retention-weighted installs; matured D7/D30)
   → train: two-tower (retrieval) · MMoE ranker · content encoder · weights
   → app embeddings → ANN index · PMI co-install table · trending per country

══ NEARLINE (seconds–minutes) ═══════════════════════════════════════════════════
 event stream → SESSION features (recent clicks, installs, categories viewed)
   → online feature store (≤ 10 s)
 INSTALL / UNINSTALL → the user's installed set updated (≤ 1 min) and a
   candidate refresh triggered
 NEW APP → policy check → content embedding → ANN fresh tier (within hours)

══ ONLINE · per request (p95 ≤ 200 ms) ══════════════════════════════════════════
┌─ 1. FEATURES ──────────────────────────────────────────────────
│ user long-term + SESSION features · context (country, language, device, time,
│ surface) · the device's compatibility profile
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CANDIDATE SOURCES, IN PARALLEL, EACH WITH A QUOTA ─────────
│ two-tower ANN · PMI co-install ("installed X, kept Y") · "next install"
│ after the most recent install · content-similar · trending in country ·
│ category top (category pages) · ⭐ NEW-APP EXPLORATION
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. HARD FILTERS (before any scoring) ─────────────────────────
│ already installed · device-incompatible · not offered in this country ·
│ policy / malware flagged · age-inappropriate · "not interested" list
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. PRE-RANKER → MMoE RANKER ──────────────────────────────────
│ heads: click · install · D1 open · D7 retained · ⚠ uninstall < 24 h
│ + shallow position tower (training only)
│ score = P(install) × P(D7 retained | install) − w·P(uninstall < 24 h)
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. RE-RANK + SPONSORED ───────────────────────────────────────
│ category diversity (MMR) · per-category quotas · exploration slots ·
│ SPONSORED SLOTS filled by the ads auction (same filters, a relevance floor,
│ deduplicated against organic), labelled "Ad"
└──────┬─────────────────────────────────────────────────────────
       ▼
 ROWS / LIST with reasons → LOGS (served features, positions, propensities)
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **The install trap** `F1c`: *"An install-optimised store fills with apps people abandon. Retained installs are the objective; installs are a weighted proxy, checked against the truth weekly (G15)."*
- **Already-installed apps reappearing:** *"The installed set is updated within a minute of an install, and the filter runs on every request, so a user never sees the app they just installed."*
- **Sponsored apps eating organic value:** *"Ads go in fixed slots from a separate auction with a relevance floor, and an ads-blind holdback measures whether they add value or just displace organic installs (§7)."*
- **The latency squeeze** `F2b` `A1c`: *"If the ranker can't score 400 in budget, I'd tighten a distilled pre-ranker and cache candidates per user between strong events, rather than shrink the ranker for everyone."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Retained installs, not installs** | "Install measures the listing; retention measures the app" | A delayed gating metric |
| **2** | **Real-time in features, not weights** | "A click changes the next request; weights move daily" | A taste shift that needs new weights waits a day |
| **3** | **Policy as filters, not penalties** | "Unsafe apps are ineligible, not just less likely" | No graded trade-off for borderline cases: those go to policy review |
| **4** | **Ads in separate, labelled slots** | "Ad spend never bends the organic ranking" | Some revenue left on the table vs blending |

---

## 5 · DATA, FEATURES, MODELS, COLD START, REAL-TIME, LOOPS *(min 18–32)* — probes **(d) (e) (g) (h)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 Data and label definitions `A2c`

| Signal (from the prompt) | Becomes | Delay | Watch out for |
|---|---|---|---|
| **Impressions** (with position, surface) | The denominator; exposure for propensities | Instant | Position bias; below-the-fold isn't seen |
| **Clicks** (details page) | Click head label; a session feature | Seconds | The listing, not the app |
| **Installs** | Install head label, **importance-weighted** by retention | Minutes | Install fraud; incentivised installs |
| **Opens** (with consent) | **D1 open**, **D7 / D30 retained** labels | 1–30 days | Matured cohorts only; unmatured rows are **censored**, not negative (G15 §5.2) |
| **Uninstalls** | **Uninstall < 24 h** = an explicit negative label | Hours | Storage clean-ups (one-off mass uninstalls) |
| **Time spent** (with consent) | A quality weight on retention labels; a user feature | Days | Games vs utilities differ hugely: normalise per category |
| **Ratings** | A quality feature (Bayesian-smoothed per app); a negative label when low | Days | Fake reviews; rating prompts biased toward happy users |
| **App metadata** | Content features: category, description, tags, developer, price, locale, compatibility | Static | Developer-written, optimised for installs |
| **Context** | Country, language, device type and tier, time | Instant | Device tier correlates with what's even installable |

**Cleaning first:** install farms, bot opens and fake reviews are removed **before** anything becomes a label (§8).

### 5.2 Feature engineering `F1a`

| Group | Features |
|---|---|
| **User, long-term** (batch, daily) | Installed-and-retained apps (an embedding of them) · category affinities, retention-weighted · paid-app history · tenure · device tier |
| **User, session** (streaming, seconds) | Last ~20 clicks and installs · categories browsed this session · search queries this session · time since last install |
| **App** (batch) | Category, tags · **content embedding** (description + screenshots) · developer track record (their other apps' retention) · smoothed rating · **category-relative D7 retention** · price · size · update recency · install velocity (trending) |
| **User × app** | Similarity to the user's retained apps · affinity to the app's category · developer already installed? · uninstalled this app before? · times shown recently (fatigue) |
| **Context** | Country, language, device type, OS version, time of day · **surface** · **position** (training only, via the shallow tower) |

### 5.3 Candidate generation `A2b` `F1a`

| Source | How | Strength | Quota (home) |
|---|---|---|---|
| **Two-tower ANN** | User tower (long-term + session + context) · app tower (**content features + ID**) · sampled softmax with partial logQ (R4 §5.2) | Personalised; content in the app tower retrieves new apps | ~40% |
| **PMI co-install** | "Installed X **and kept it** → also kept Y", normalised by popularity (G17) | Precise for the head; avoids the top-50 chart | ~20% |
| **"Next install"** | A sequence model over recent installs | Captures the moment ("just got a running watch") | ~10% |
| **Content-similar** | Nearest neighbours in content-embedding space | Cold-start apps | ~10% |
| **Trending in country** | Install velocity, fraud-filtered | Freshness; local hits | ~10% |
| **⭐ New-app exploration** | Fresh apps sampled by Thompson sampling (§5.7) | Keeps the catalogue renewing | ~10% |
| **Category top** | Within-category quality ranking | Category pages | (category surface) |

### 5.4 Ranking `A2b` `F1a`

```
PRE-RANKER: a small model distilled from the ranker, 2,000 → 400

MMoE RANKER (R4 §5.4): shared experts, one gate per head
  heads:  P(click) · P(install) · P(D1 open | install) · P(D7 retained | install)
          ⚠ P(uninstall < 24 h | install)
  losses: BCE per head, each masked to rows that have its label (D7 only on
          matured rows) · installs IMPORTANCE-WEIGHTED by the app's historical,
          category-relative retention (G15 §5.2) · a shallow POSITION tower,
          dropped at serving (R4 §5.5)
  inputs: the features in §5.2, with TARGET ATTENTION over the user's recent
          installs (which past apps matter for THIS candidate, R6 §5.4)

THE SCORE:  P(install) × P(D7 retained | install)  −  w · P(uninstall < 24 h)
            all heads calibrated (playbook §7.7), weights owned by product
RE-RANK:    MMR over category and developer · per-category quotas on mixed
            rows · exploration slots · sponsored slots (§6.2)
```

### 5.5 Cold start: new users and new apps (h) `F1c` `F2a`

**New users (a fresh phone, no history):**

| Signal | Why it helps |
|---|---|
| **Country, language, device model and tier, OS version** | Strong priors: what's popular, compatible and affordable here |
| **The first session's clicks and searches**, streamed in seconds | After 2–3 interactions, session features carry the personalisation (§5.6) |
| **Onboarding** (optional "what are you into?") | Explicit, but many skip it |
| **Installed apps on the device** (only where policy and consent allow) | Very strong, and **privacy-sensitive** (FU 10) |
| **A category-mix bandit** | Explores categories on the first rows instead of guessing |

**Policy:** the first home feed is **popular-and-retained in this country and device tier**, deliberately **spread across categories**; the blend shifts toward personal models as history builds.

**New apps (zero installs), reusing G15 §6:**
- **Content embedding** from a pretrained image–text encoder, **LoRA-tuned on retention-weighted co-installs**, so a new app sits near apps its likely users kept (§5.8).
- **Developer track record:** the strongest prior; a studio with three retained hits is evidence about the fourth.
- **Category priors** for retention, shrunk toward the app's own as data arrives.
- **A funded exploration quota** with logged propensities (§5.7), because without impressions an app never earns data.
- **Graduation** into the normal sources once its retention estimate is tight.

### 5.6 Real-time adaptation within minutes (g) `A2d`

```
FRESHNESS LIVES IN FEATURES AND CANDIDATES, NOT WEIGHTS (R6)
  seconds   a CLICK → session features (recent clicks, categories viewed)
            → the next request's ranker sees them
  ≤ 1 min   an INSTALL → the installed set updates (the app is filtered out
            everywhere) → a CANDIDATE REFRESH is triggered: "next install"
            and "installed X, kept Y" sources keyed on the new app
            an UNINSTALL → the app's neighbours are pushed down; a negative
            for "not interested"-style suppression
  hours     trending per country · new apps into the fresh ANN tier
  daily     the two-tower model, the ranker, the content encoder retrain
            (warm-started); retention weights refresh weekly

READ-YOUR-WRITES: the app sends its last few actions with each request, so an
  install counts on the very next refresh even if the stream is a few seconds
  behind (R6 §4)

e.g. 09:00 installs a running app → 09:01 home: "installed X, kept Y" brings a
  hydration tracker and a training-plan app up; the running app is gone
```

### 5.7 Exploration and feedback-loop mitigation (h) `F1c`

| Loop | How it hurts | Mitigation |
|---|---|---|
| **Popularity** | Raw co-installs recommend the top-50 chart to everyone, and installs "approve" | **PMI** normalisation · partial logQ · recall **by popularity decile** · category concentration guardrail |
| **Exposure** (selection) | Only shown apps get labels; unshown apps never prove themselves | **Exploration slots with logged propensities** → unbiased labels and OPE (R3 §7) |
| **Rich-get-richer** | New apps never get impressions | The **new-app exploration quota**: Thompson sampling on each new app's D1-open posterior, prior from its category and developer (R6 §5.5) |
| **Position** | Top cards installed regardless of quality | Shallow position tower; a randomised-order slice to measure κ (R4 §5.5) |
| **Install-proxy drift** | The model satisfies installs without retention | Importance weighting + the **proxy↔truth correlation check** (G15 FU 5) |
| **Category tilt** | High-retention categories crowd out others on mixed rows | Category-relative weights and per-category quotas (G15 FU 6) |

**Long-term check:** a **holdback** on the previous policy for months, because a slow drift toward high-install, low-value apps is invisible inside a 30-day window.

### 5.8 Fine-tune an existing model `F2c`

LoRA-tune a pretrained **image–text encoder** on app descriptions and screenshots, with **retention-weighted co-installs** as the positive pairs (G15 §5.4). Its embeddings feed the two-tower app tower, the content-similar source and the ranker. Every daily run **warm-starts** from the previous checkpoint.

### 5.9 The staged pipeline `A2d`

```
STAGE 1 · LOG      impressions (position, surface) → clicks → installs → opens → uninstalls
STAGE 2 · CLEAN    fraud removal (farms, bots, fake reviews) · policy status join
STAGE 3 · LABEL    retention weights per app × segment, category-relative · matured D7/D30
STAGE 4 · TRAIN    two-tower · MMoE ranker · content encoder (daily, warm-started)
STAGE 5 · INDEX    app embeddings → ANN + fresh tier · PMI table · trending
STAGE 6 · GATE     offline screens → shadow → A/B on the proxy → matured-D7 gate → ramp
NEARLINE: session features in seconds; installed set and candidate refresh within a minute
```

---

## 6 · SERVING: SURFACES, ADS, SCALE *(min 32–37)* — probe **(i)** `F3a` `F3b` `A3b` `I1b`

### 6.1 One ranker, different surfaces

| | **Home feed** | **Category page** |
|---|---|---|
| **Intent** | Browse: "show me something good" | Narrower: "something in Productivity" |
| **Candidates** | All sources, quota'd | Apps in the category, plus personalised sources filtered to it |
| **What dominates the score** | Personal fit, with diversity across categories | **Quality within the category** (category-relative retention, ratings), personalised second |
| **Diversity** | Across categories and developers (rows) | Across **sub-categories** (notes, calendars, to-do) |
| **Exploration** | A dedicated "New & notable for you" row | 1–2 new apps per page, labelled |
| **Model** | The same ranker, with **surface as a feature**; re-ranking rules per surface | |

### 6.2 Sponsored placements `A3b` `I1i`

```
A SEPARATE SYSTEM, MERGED AT THE END (G21's auction)
  1. ELIGIBLE ADS pass the SAME hard filters: policy, malware, age, device
     compatibility, country, already installed
  2. RELEVANCE FLOOR: an ad must clear a minimum P(install AND D7 retained)
     for THIS user, from the organic ranker's own heads, so an irrelevant
     ad loses its slot however much it bids
  3. AUCTION ranks eligible ads by  bid × P(install) × quality
     (quality includes retention and ratings, so an app people abandon pays more
      per slot, or wins less)
  4. FIXED SLOTS (e.g. positions 3 and 8 of a row), always labelled "Ad"
  5. DEDUPE: an app that's both organic and sponsored appears once, in the
     better position, labelled "Ad" only if it's in an ad slot
  6. CAPS: frequency per user per app; a ceiling on ads per page

⚠ THE ORGANIC RANKER NEVER SEES BIDS: ad spend can buy a labelled slot, never a
  better organic position. That keeps the organic ranker an honest measure of
  user value, and it's required for disclosure (and, for gatekeeper app stores
  in the EU, by the DMA's ban on self-preferencing for first-party apps too)
⚠ DISPLACEMENT: an ad that takes an install the user would have made
  organically is revenue, not value → measured by an ads-blind holdback (§7)
```

### 6.3 Efficiency and scale `F3a` `F3b`

- **Efficiency** `F3a`: the app corpus and its embeddings (~1.3 GB) sit **in memory** on every serving node; a distilled pre-ranker cuts the heavy ranker's work; content embeddings are computed offline per app; int8 storage for the ANN.
- **Scale** `F3b`: stateless ranking services autoscaled per surface; user features in a sharded online store; **candidate cache per user**, refreshed on strong events (install, uninstall) and expired after minutes; trending and category tops precomputed per country and device tier.

### 6.4 Reliability: the degradation ladder `I1b`

```
TIER 0 · FULL        personalised sources → MMoE → re-rank + ads
TIER 1 · CACHED      ranker slow → the user's cached candidate list, re-filtered
TIER 2 · CHARTS      user features or ranker down → category charts per country
                     and device tier, RETENTION-FLOORED (never the raw install
                     chart, which rewards the listing)
TIER 3 · EDITORIAL   everything down → curated lists per country
EVERY TIER: policy, age, compatibility and already-installed filters still apply;
  sponsored slots are DROPPED rather than shown unfiltered
```

---

## 7 · EVALUATION, A/B, MONITORING *(min 37–44)* — probe **(j)** `I2a` `I2b` `F1b` `F1c`

### 7.1 Offline

```
TIME-SPLIT evaluation (never random): train to day N, test on N+k
RETRIEVAL  recall@k of the apps users went on to install AND keep · sliced by
           popularity decile, new vs established apps, new vs established users
RANKER     per-head AUC + calibration · NDCG@k with retention-weighted gains
           (a kept install scores higher than an install uninstalled the next day)
SLATE      category diversity · new-app share · coverage · concentration
COUNTERFACTUAL  SNIPS / doubly robust on the exploration slots (logged propensities)
⭐ trust an offline metric only as far as it has predicted past A/B outcomes
```

### 7.2 A/B testing

- **Randomise by user**; ramp 1% → 5% → 50%; ≥ 2 weeks (novelty, weekly cycles); CUPED for variance.
- **Two clocks** (G15 §7): iterate on the **weighted-install proxy** during the test; make the **ship decision on the cohort's matured D7** a week later.
- **Guardrails:** uninstall-within-24 h, new-app reach, category concentration, policy impressions = 0, p95 latency, ad revenue.
- **Interleaving** for ranker-vs-ranker comparisons: a fast, low-variance first read.
- **Sponsored changes need their own design:** an **ads-blind holdback** (a small group with no ads) measures whether ads **add** installs and retention or merely **displace** organic ones; ad-load changes are tested against total value, not ad revenue alone.
- **Long-term holdback** on the old policy for months.
- **Success** = D7-retained installs per visit up, category-relative; uninstall-24 h not up; new-app reach held; ads adding net value. `I2b`

### 7.3 Monitoring

| Signal | Catches | Response |
|---|---|---|
| **Proxy↔truth correlation**, per category | Stale retention weights; the model gaming installs | An incident: refresh weights, or freeze and gate on matured D7 only (G15 FU 5) |
| **Install velocity anomalies** per app | Install farms, incentivised campaigns | Fraud review; labels quarantined |
| **Uninstall-within-24 h rate**, per source | A source or ranker version recommending bad apps | Roll back the version; demote the source |
| **Freshness**: event-to-feature lag, installed-set update lag | Already-installed apps reappearing; stale sessions | Scale the stream; client-sent events cover the gap |
| **Policy / age filter hit rates** | A broken filter or policy feed | Page: policy impressions must stay at zero |
| **Concentration, new-app reach, coverage** | The feedback loop narrowing | Increase exploration; check PMI and logQ settings |
| **Score drift, PSI on features, calibration** | Upstream breakage, drift | Block pushes; recalibrate |
| **Latency, fallback-tier share** | Degradation | Capacity, not thresholds |

**Bias sources and mitigations** `F1c`: popularity, exposure, rich-get-richer, position and category tilt are in §5.7; add **device tier** (low-end devices see fewer compatible apps and generate worse data → per-tier evaluation) and **language** (thinner metadata in some locales → per-locale slices).

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 200 ms per surface; corpus in memory; a distilled pre-ranker; $ per 1,000 surface loads | p95 per surface · $/1k loads |
| **Reliability** `I1b` | Cached candidates → retention-floored charts → editorial; filters at every tier; ads dropped rather than unfiltered | Fallback share per tier |
| **Usability** `I1c` | Installed apps never shown; reasons on cards; "not interested" acts on the next load; category pages that stay on-topic | Uninstall-24 h · dismissals |
| **Scalability** `I1d` | Stateless rankers per surface; sharded user features; precomputed per-country charts; candidate cache per user | QPS headroom · stream lag |
| **Privacy** `I1e` | **Which apps someone uses reveals health, finance, dating, religion** → usage signals only with consent; sensitive categories never used to infer interests or shown in reasons; the on-device installed-apps list only where policy allows | Sensitive-inference audits |
| **Safety** `I1f` | Policy-violating and malware-flagged apps **filtered before ranking**; **age-appropriate** recommendations for minors; gambling, loans and dating restricted by region and age | Policy / age-inappropriate impressions = 0 |
| **Security** `I1h` | Auth and rate limits, and **beyond the gateway**: ⚠ **install farms, incentivised installs, bot-faked opens and fake reviews** poison labels → fraud removal before training, velocity anomalies, developer-level penalties · ⚠ **metadata gaming** (keyword-stuffed descriptions) → content-quality checks | Fraudulent installs caught |
| **Ethics / bias** `I1i` | **New-app reach** funded; **sponsored apps disclosed, capped and kept out of organic ranking**; **no self-preferencing** of first-party apps (EU DMA for gatekeeper stores); per-tier and per-locale parity | New-app reach · first-party share vs organic baseline |
| **Explainability** `I1j` | "Because you installed …" on cards; "Ad" labels; **per-app impression attribution** for developers (which source, which stage, which surface); policy actions distinguished from ranking changes (G15 FU 7) | % cards with a reason · developer disputes resolved |

> ⭐ *"The store is a market maker: users should get apps they keep, developers should see why they rank where they do, and advertisers should buy a labelled slot, never a better organic position."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* `F3c` `F2a` `A5b`

### Extensions `F3c`

- **Search results** (the prompt's optional surface): the query adds explicit intent, so it's a **search-ranking** problem. Retrieval combines BM25 on titles and developer names with semantic retrieval over descriptions; **navigational queries** (an exact app name) must return that app first, whatever personalisation says; ranking blends query relevance with retained-install quality and light personalisation; sponsored results are capped and labelled. The **cost** is a second retrieval stack; the **maturity risk** is semantic search surfacing look-alike apps for brand queries, so exact-name matching wins ties.
- **Re-engagement of installed apps:** an **uplift** problem, since recommending an app the user would open anyway is worthless (G15 §9).
- **Subscriptions:** D30 can mean "forgot to cancel", so subscription apps need their own retention semantics.

### The full alternative pipeline: retention-floored, non-personalised `A5b` `F2a`

```
WHEN THE BRIDGE BREAKS (the proxy↔truth correlation drops) or personalisation
is unavailable:
  → freeze retention weights at the last validated set
  → rank within category and country by SMOOTHED D7 retention (shrunk toward
    the category mean), quality-floored, compatibility-filtered
  → keep the exploration row and the policy filters
  → gate releases on matured D7 only
  "Slower iteration on the truth beats fast iteration on a broken proxy" (G15 §9)
```

### Top 3 failure modes

1. **The install trap:** an install-optimised ranker fills the store with apps users abandon, and the install dashboard approves.
2. **Sponsored drift:** ad spend leaking into organic ranking, or ad load rising until ads mostly displace organic installs, with revenue up and user value down.
3. **New-app starvation:** no exploration budget, so only incumbents are reachable and the catalogue stops renewing.

**Q1 ships:** retention-weighted installs with **D1 open** and the **proxy↔truth check** (G15's quick win), the multi-source candidates with **PMI** and the **new-app exploration quota**, **hard policy filters**, **streaming session features with the installed-set update**, and sponsored slots **with the ads-blind holdback** from day one.

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not optimise installs?"** `F1b` `A5a`
> Because an install measures the **store listing**, not the app: a polished icon and good screenshots get the install, and the app is uninstalled the next morning. An install-optimised ranker converges on exactly that, a store full of hyper-casual games and utilities with great install rates and terrible retention, because that's what the objective asked for. So the objective is **retained installs**: install, then opened on day one, then still used at day seven, with **uninstall within 24 hours as an explicit negative**. Retention arrives one to four weeks late, so I train on installs **weighted by how often similar installs were retained**, category-relative, and gate releases on matured D7. And I keep a standing check that the weighted proxy still predicts retention, because when that correlation breaks, the system quietly optimises the listing again (G15).

**2. "A user installs an app. How does their feed change within minutes?"** `A2d` `I1c`
> The freshness lives in features and candidates, not in retraining. Within seconds, the install event streams into the user's **session features** (recent installs, categories), and the app sends its last few actions with each request, so the very next refresh counts the install even if the stream lags. Within a minute, the user's **installed set** updates, so that app disappears from every row and every surface, and a **candidate refresh** is triggered for sources keyed on the new app: "people who installed X and kept it also kept Y", and a "next install" sequence model that captures the moment, like a hydration tracker after a running app. The ranker then scores those candidates with the new session features, using target attention over recent installs. An **uninstall** does the reverse: it pushes that app's neighbours down. The models themselves retrain daily, because the relationship between features and outcomes changes slowly; it's the user's state that changes in minutes.

**3. "A brand-new user on a fresh phone opens the store."** `F2a` `F1c`
> They still bring context: country, language, device model and tier, OS version. That says a lot about what's popular, compatible and affordable here, so the first feed is **popular-and-retained apps for this country and device tier**, deliberately spread across categories, because the first rows are a measurement as much as a recommendation. A **category-mix bandit** decides how much of each category to show. Every click and search streams into **session features** within seconds, so after two or three interactions the feed is already personal. An optional onboarding step helps when people use it. The on-device **installed-apps list** would be the strongest signal of all, but it's privacy-sensitive, so I'd use it only where platform policy and consent allow (FU 10). The blend shifts from these priors toward the personal models as history builds, with no cliff.

**4. "A brand-new app with zero installs. How does it get shown?"** `F2c` `I1i`
> It's retrieved and ranked on **content**: the app tower of the two-tower model takes content features, and a pretrained image–text encoder, LoRA-tuned on **retention-weighted co-installs**, embeds the description and screenshots, so the new app lands near apps its likely users kept. The ranker uses **developer track record**, the strongest single prior, plus **category retention priors**. But modelling isn't enough: without impressions it never earns data. So it gets a **funded exploration quota**, Thompson sampling on its D1-open posterior with a prior from its category and developer, with every exploratory impression's **propensity logged**. A good app earns exposure within days, and it graduates into the normal sources once its retention estimate is tight. New-app reach is a monitored guardrail, because a store that only surfaces incumbents stops getting new apps (G15).

**5. "How do you add sponsored apps without hurting users?"** `A3b` `I1i`
> By keeping them structurally separate from organic ranking. Ads come from **their own auction** into **fixed, labelled slots**. Before the auction, every eligible ad passes the **same hard filters** as organic apps: policy, malware, age, compatibility, country, already installed. Then a **relevance floor**: the ad must clear a minimum predicted chance of being installed and kept by this user, taken from the organic ranker's own heads, so an irrelevant app can't buy its way in however much it bids. The auction ranks by bid × install probability × quality, with retention and ratings in quality, so apps people abandon pay more or win less. An app that's both organic and sponsored appears once. And critically, the **organic ranker never sees bids**: spend buys a labelled slot, never a better organic position. To check ads add value rather than displace installs that would have happened anyway, I'd run an **ads-blind holdback** and compare total retained installs, not just ad revenue.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "The feed recommends the top-50 chart to everyone."** `F1c` `A5a`
> Popularity skew in the collaborative signal, and the metrics approve, because those apps really do get installed. A handful of apps account for most installs, so raw co-install counts and an uncorrected retrieval model both drift there. Fixes: **PMI-normalise** co-installs so "kept together more than chance" beats "both popular"; apply **logQ partially** in the two-tower loss, since a full correction restores the popularity prior (R4 FU 1); keep **source quotas** so no single source dominates; lean on **content and session features**, which don't carry this bias; and evaluate **recall by popularity decile**, with **category concentration** as a guardrail, so the failure is visible rather than averaged away (G15 FU 4).

**7. "How should the home feed and category pages differ?"** `I1c` `A2a`
> Same ranker, different intent. The **home feed** is browsing, so personal fit matters most, with diversity **across categories** and a dedicated exploration row. A **category page** has narrower intent, so **quality within the category** matters more: category-relative retention and ratings come first, personalisation second, and diversity runs across **sub-categories** (notes, calendars, to-do) rather than categories. I'd keep one ranker with **surface as a feature**, so both surfaces share learning, and put the differences in the candidate sources and the re-ranking rules. One trap to name: comparing scores across categories on the home feed has no principled exchange rate between "kept a game" and "kept a banking app", which is why mixed rows use **category-relative** scores and per-category quotas (G15 FU 6).

**8. "The ads team wants twice as many sponsored slots."** `A3b` `I2b`
> I'd treat it as an experiment with a total-value readout, not an ad-revenue one. More slots raise ad revenue almost by definition, so revenue alone can't decide it. What decides it is whether **total retained installs** and **user return rate** hold. I'd run the change against the current ad load, with an **ads-blind holdback** as the baseline, and read: retained installs per visit across organic and sponsored, **displacement** (how many sponsored installs would have happened organically anyway), the uninstall-within-24 h rate on sponsored installs, and longer-term store engagement. If doubling slots mostly displaces organic installs and raises abandonment, the extra revenue is borrowed from the store's trust. The relevance floor stays fixed throughout, because loosening it is the quiet way ad quality falls.

**9. "Your proxy↔truth correlation just dropped."** `I2a` `F2a`
> It's an incident, because every model change since depends on that proxy (G15 FU 5). First, **freeze**: keep the last validated retention weights and gate releases on **matured D7 only** until it's understood. Then diagnose the three usual causes. **Stale weights:** an app's retention changed after an update or a monetisation change, so last quarter's estimate is wrong. **A category-mix shift:** weights calibrated on one distribution applied to another, say a seasonal surge in games. **Goodhart:** the model learned to win installs without retention, which shows up as proxy-up, truth-flat, often concentrated in one source or one cluster of apps. Fixes follow the cause: refresh weights on a faster cadence, re-fit per category, or add a guardrail and retrain. Meanwhile the fallback ranks by smoothed retention, which is slower but honest.

**10. "Is using the installed-apps list a privacy problem?"** `I1e` `I1i`
> It can be, and I'd treat it as a sensitive signal by default. The set of apps on someone's phone can reveal health conditions, finances, sexuality, religion or politics. So: use it **only where platform policy and user consent allow** it, and prefer the store's **own** install history over reading the device; **never infer or display sensitive categories**, so no "because you have a fertility app" reasons and no sensitive interests in the user profile; **aggregate** it into embeddings rather than storing raw lists where possible; honour **deletion** and opt-outs, which is easier when user features are recomputed from history rather than baked into per-user parameters; and consider **on-device** scoring for the most sensitive signals. The test I'd apply: would the user be surprised or uncomfortable to learn this signal was used? If yes, it needs explicit consent or it doesn't belong in the model.

---

## ⭐ THE L6 LINE

> **"Optimise apps people keep, not installs: retention-weighted installs as the proxy, matured D7 as the gate, uninstalls as negatives. Adapt within minutes through streaming features and candidate refreshes, keep policy as hard filters and sponsored apps in separate, labelled slots behind a relevance floor, and fund exploration so new apps and new users aren't left to a popularity loop."**

## 🧭 HOOKS — *steer by what you mention*

- *"Install measures the listing, not the app"* → invites the labels branch (**good at L6**)
- *"Real-time lives in features, not weights"* → invites the real-time branch
- *"Content in the app tower"* → invites the new-app cold-start branch
- *"Ad spend buys a labelled slot, never an organic position"* → invites the sponsored branch (**good at L6**)
- *"Policy as filters, not penalties"* → invites the safety branch
- *"An ads-blind holdback"* → invites the experimentation branch
- *"The store is a market maker"* → invites the ecosystem branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the label ladder: impression, install, D1 open, D7 retained."* It regenerates objectives, labels and evaluation.
- **Running out of time?** → *"I have five minutes. I'd rather cover cold start, sponsored placement and monitoring than finish the feature list. Sound right?"*

**Name and defer:**
- "**The delayed-label bridge in depth** (importance weights, category baselines, the correlation check) is G15."
- "**The ads auction** (bidding, pricing, calibration) is G21."
- "**App review and policy enforcement** are upstream; I consume their verdicts as filters."
- "**In-store search** is an extension with its own retrieval stack."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"Retained installs as the objective; real-time through features; policy as filters; ads in separate slots."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"~7k QPS at peak; ~120 ms of a 200 ms p95; ~2.8×10⁶ scorings/s."* |
| **A1c** | Degrade without lowering quality | §4, §6.4 | *"Cached candidates, then retention-floored charts, never the raw install chart."* |
| **A2a** | The full stack | §4 diagram | Offline → nearline → online: sources → filters → ranker → re-rank + ads |
| **A2b** | Model components | §5.3, §5.4 | Two-tower with content, PMI, next-install sequence model, MMoE, content encoder |
| **A2c** | Data sources | §5.1 | The prompt's events, metadata and context, each turned into a label or feature |
| **A2d** | Staged pipeline | §5.6, §5.9 | Log → clean → label → train → index → gate; nearline freshness tiers |
| **A3a** | Top-down, unprompted | §0 | *"The label; real-time in features; two business constraints"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.4, §6.2, FU 5, 8 | *"P(install) × P(D7 \| install) − w·P(uninstall); ads behind a relevance floor."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Objective, surfaces, sponsored policy, developer reach, minors, consent |
| **A5a** | Alternatives compared | §4, FU 1, 6 | Installs vs retention; ID-only vs content two-tower; penalties vs filters; blended vs separate ads |
| **A5b** | A full alternative pipeline on demand | §9 | The retention-floored, non-personalised pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, §5.4 | *"Sampled softmax with partial logQ; MMoE with per-head BCE and importance-weighted installs."* |
| **F1b** | Metrics and their trade-offs | §2, §7.1, FU 1 | *"Installs are available; retained installs are meaningful."* |
| **F1c** | Bias sources and mitigations | §5.7, §7.3, FU 6 | Popularity, exposure, rich-get-richer, position, proxy drift, category tilt, device tier |
| **F2a** | Something fails → a fallback | §6.4, §9, FU 9 | The degradation ladder; freeze weights and gate on matured D7 |
| **F2b** | Latency too tight → design for it | §4 | Distilled pre-ranker and cached candidates, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.8 | LoRA image–text encoder on retention-weighted co-installs |
| **F3a** | Efficiency with a quality trade | §6.3 | Corpus in memory; distilled pre-ranker; int8 ANN |
| **F3b** | Scale levers | §6.3 | Per-user candidate cache; precomputed charts per country and tier |
| **F3c** | Extension, with cost and maturity risk | §9 | Search; re-engagement uplift; subscriptions |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 ≤ 200 ms per surface |
| **I1b** | Reliability | §6.4, §8 | Filters at every tier; ads dropped rather than unfiltered |
| **I1c** | Usability | §2, §8, FU 2, 7 | Installed apps never shown; reasons; surface-appropriate ranking |
| **I1d** | Scalability | §3, §8 | Stateless per-surface rankers; candidate caches |
| **I1e** | Privacy | §8, FU 10 | Consent for usage signals; no sensitive inference; the installed-apps list |
| **I1f** | Safety | §8 | Policy, malware and age filters before ranking |
| **I1h** | Security | §8 | **Install farms, bot opens, fake reviews, metadata gaming** |
| **I1i** | Ethics / bias | §6.2, §8, FU 4, 5 | New-app reach; disclosed ads; no self-preferencing |
| **I1j** | Explainability | §8 | Reasons, "Ad" labels, per-app attribution for developers |
| **I2a** | Offline → online | §7, FU 9 | Time-split screens → A/B on the proxy → matured-D7 gate → holdbacks |
| **I2b** | The success definition | §7.2, FU 8 | *"D7-retained installs up, uninstalls not up, new-app reach held, ads adding net value."* |

**🧭 Probe map:** (a) clarifying → §1 · (b) surfaces → §2, §6.1 · (c) architecture → §4 · (d) candidates and ranking → §5.3, §5.4 · (e) features and models → §5.2, §5.4 · (f) labels and metrics → §2, §5.1, §7 · (g) real-time → §5.6 · (h) cold start, exploration, loops → §5.5, §5.7 · (i) sponsored and policy → §6.2, §8 · (j) monitoring, A/B, reliability → §6.4, §7.
