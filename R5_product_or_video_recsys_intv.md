# R5 · A Product or Video Recommendation System — **INTERVIEW SHEET**

> ⭐ **A Google-style prompt, prepared against the same L6 evaluation form as [R1](R1_prompt_suggestions_intv.md)–[R4](R4_watch_next_losses_optimizers_intv.md):** 3 parameters (**A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration), 10 sub-parameters. Every section is tagged with the rubric codes it earns, and the [🧾 RUBRIC COVERAGE MAP](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each.
>
> **The prompt:** *"Design a recommendation system for a large consumer platform. The platform may recommend either products in an e-commerce feed or videos in a media feed. Cover: the main user-facing surfaces · online and offline data sources · candidate generation · ranking and re-ranking · feedback signals (clicks, views, purchases, watch time, likes, skips, hides, negative feedback) · model training and evaluation · cold start for new users and new items · online serving and latency · experimentation, monitoring and guardrails."*
>
> **Expect these probes:** (a) assumptions and surfaces · (b) feedback signals, and what counts as success · (c) candidate generation, ranking, re-ranking · (d) online and offline data · (e) labels from feedback · (f) models and training · (g) serving and latency · (h) experimentation, monitoring, guardrails · (i) cold start.
>
> **How this differs from its neighbours:** [G13](G13_youtube_recommendation_intv.md) is the video home feed and [R4](R4_watch_next_losses_optimizers_intv.md) is watch-next with every loss derived. This sheet is the **platform view**: one architecture, **two domains**, and exactly where they diverge. The default walk-through is **e-commerce**; if the interviewer picks videos, use the Videos column and draw depth from G13 / R4.
>
> **Related sheets:** [G7 shopping ranking](G7_shopping_ranking_intv.md) (purchase without return) · [G14 Discover feed](G14_discover_feed_intv.md) (suppression) · [G15 Play Store](G15_play_store_recommendations_intv.md) (the delayed-label bridge) · [G17 similar items](G17_similar_items_intv.md) (substitutes vs complements) · [G22 delayed conversion](G22_delayed_conversion_intv.md) (modelling the delay).

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | The first one is *"products or videos?"*, and the second is *"is there a marketplace, ads or an own brand?"*: they change the objective and the guardrails (§1) |
| **Explainability** is easy points | A reason on every card, a **"Sponsored"** label, and a **non-personalised feed option**: EU platforms of this size must offer one (§8) |
| **Security** beyond the gateway | **Fake reviews and click farms** boosting listings; **keyword-stuffed titles** gaming content retrieval; scraping (§8, FU 10) |
| **Troubleshoot before being asked** | Raise **delayed purchases and returns**, **a hide that must change the next page**, **cold start**, and **training–serving skew** yourself (§4) |

### This prompt's own checklist, and where each item is earned

| The prompt asks for | Where |
|---|---|
| Main user-facing surfaces | §2 (surface table, both domains) |
| Online and offline data sources | §5.1 (with an online / nearline / offline column) |
| Candidate generation · ranking · re-ranking | §4 (the stack), §5.3 (the models) |
| Feedback signals (clicks … negative feedback) | §5.2 (the signal table: polarity, delay, trap, use) |
| Model training and evaluation | §5.3, §5.6, §7 |
| Cold start, new users **and** new items | §5.4, §9 (the first-session pipeline), FU 2, 3 |
| Online serving and latency | §3, §6 |
| Experimentation, monitoring, guardrails | §7 |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **One platform, two domains** · feedback acts at three speeds · a feed is a slate under constraints | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | *"Products or videos?"* **unprompted** · assumptions · NFR · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | Surfaces · **the two label ladders** · net purchases, not GMV or CTR | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | (g) | ~10⁵ QPS · 5×10⁷ scorings/s · 7×10⁵ events/s into the session store | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Shared vs domain-specific · offline / nearline / online stack · four problems raised yourself | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f) (i)** | Data by freshness tier · **the signal table** · two-tower + sequence + ESMM ranker · **cold start** | A2b, A2c, A2d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | Budget · feature store, log-at-serve · page 2 re-ranked from page 1 · fallbacks | F3a, F3b, A3b, I1b |
| **38–44** | **EXPERIMENTS, MONITORING, GUARDRAILS** | **(h)** | Offline → A/B with **marketplace interference** · guardrails per domain · monitors | I2a, I2b, F1b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | — | First-session pipeline · shoppable video, conversational steering · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The prompt says products or videos, and I want to use that. The architecture is the same for both: candidate sources, a multi-task ranker, a re-ranker under constraints. So I'd build it once, as a platform. What differs is everything that decides whether it's any good: the objective, the labels and how late they arrive, what's allowed to be shown, and cold start. I'll ask which domain you want depth on, and mark where the other one diverges.**
>
> **Second, feedback isn't one signal. A click, a purchase, a return and a hide mean different things, arrive at different times, and should act at different speeds: a hide has to change the next page, not just tomorrow's model.**
>
> **Third, the feed is a slate under constraints (diversity, stock, policy, sponsored disclosure), not twenty independent predictions. And the metric isn't the click: for products it's purchases that aren't returned, and for videos it's watch time the viewer doesn't regret."**

**Then say the plan (`A3a`):** *"Clarifying questions, the surfaces and what success means, capacity, the architecture, data and feedback signals, models and cold start, serving, experimentation and guardrails, a production-readiness sweep, then failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Products or videos, and which surface first?** *(Assume e-commerce, the home feed first; product-page and post-purchase surfaces second.)*
2. **Is it a marketplace with third-party sellers? Are there sponsored items? Does the platform sell its own brand?** *(Each one adds a party whose interests diverge from the user's: §7, FU 8.)*
3. **What's the business objective, and what must not regress?** *(Assume net purchases plus return visits; guardrails on returns, hides, seller concentration and latency.)*
4. **Signed in? How much history does a typical user have?** *(Decides how much rides on cold start.)*
5. **Scale and latency?** *(Below.)*
6. **Regions?** *(EU users bring DSA obligations: transparency and a non-profiling option, §8.)*

**State assumptions:**

| Assumption | Products (default) | Videos |
|---|---|---|
| Users | ~3×10⁸ DAU, ~10 feed pages/day | Same order |
| Catalogue | ~5×10⁸ listings, ~10⁶ new/day; **price and stock change hourly** | ~10⁹ videos, millions of uploads/day |
| Surfaces in scope | **Home feed** (primary) · product page · cart / post-purchase | Home feed (primary) · watch-next (R4) |
| Objective | **Net purchases** (bought and not returned) + 30-day return visits | Satisfied watch time + 7-day return |
| Personalisation | History, respecting settings; sensitive categories excluded from inference | History; none on made-for-kids content |
| Market | Third-party sellers, sponsored slots, an own brand | Creators |

**FR:** a personalised feed under constraints · item-page "similar" (substitutes) and "goes with" (complements) · post-purchase complements and replenishment · **negative feedback that acts on the next page** · cold-start handling for users and items · a reason per card · "Sponsored" disclosure · a non-personalised option.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **First page** | **p95 ≤ 200 ms** server-side |
| **Next page** | Prefetched while the user scrolls; reflects page-1 actions |
| **Hide → effect** | Item and near-duplicates gone from the **next page**; aversion features updated in **seconds** |
| Item freshness | Price and stock **≤ 5 min** stale; a new listing eligible within **~1 h** |
| Availability | 99.9%; on failure, a degraded but relevant feed, never a blank |
| **Guardrails** | Return rate · hide rate · out-of-stock clicks · small-seller share · sponsored share · p95 latency |

**Scope / NON-GOALS:**
- ❌ **Optimising CTR or raw GMV.** CTR rewards thumbnails; GMV rewards expensive items whatever the fit.
- ❌ Search ranking (G7) · the ads auction itself (G21): it supplies the sponsored candidates, and the feed decides the **slots** · pricing.
- ❌ Email and push (the same models, a different send-time problem).

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) `I1c` `F1b` `I2b`

**The surfaces (a):**

| Surface | Products | Videos | Different problem? |
|---|---|---|---|
| **Home feed** | "Recommended for you": infinite scroll | Home feed | Low intent, broad interests → **exploration and diversity matter most** |
| **Item page** | "Similar items" = **substitutes**; "Goes with" = **complements** | Watch-next (R4) | Strong context (the item on screen) → item-to-item |
| **Post-action** | After purchase: complements, **not** substitutes; replenishment for consumables | After a watch: the series continues | ⚠ The fridge problem (FU 9) |
| **Cart / checkout** | Add-ons, low price, high attach rate | — | Must not distract from checkout |

**What the user sees (home feed, products):**

```
RECOMMENDED FOR YOU
  [img] Running shoes, $79     "Because you viewed trail runners"     ⋮
  [img] Water bottle, $12      "Goes with your recent purchase"       ⋮
  [img] Desk lamp, $34         "Popular in Home near you"   ← exploration slot
  [img] Headphones, $59        Sponsored                    ← disclosed, capped
  … mixed categories · never 3 in a row from one seller or brand
  ⋮ menu: Not interested · Don't show this brand · Already bought it  ← labels, act NOW
Settings: "Show a non-personalised feed"
```

**Two label ladders: one per domain, each rung later and more meaningful.**

```
PRODUCTS:  IMPRESSION → CLICK → DETAIL VIEW (dwell) → ADD TO CART / WISHLIST
           → PURCHASE (hours–days) → ⭐ NOT RETURNED (up to ~30 days)
           → repeat visit / repeat purchase (consumables)

VIDEOS:    IMPRESSION → CLICK → WATCH ≥ 10 s → WATCHED DEEP *for its length*
           → LIKE / SAVE / SHARE → ⭐ SURVEY "worth your time?" → return next week

NEGATIVES (both): skip (examined, passed over) · HIDE / not interested ·
           don't show this seller / brand / channel · report · RETURN (products)
           · abandon < 10 s (videos)

⚠ THE TRAPS: CTR rewards thumbnails. GMV rewards expensive items. Purchases
  that are returned are worse than none: they cost shipping, handling and trust.
  Watch time rewards long and sensational videos.
```

**The metric stack:**

| Layer | Products | Videos | Role |
|---|---|---|---|
| **North star** | **Net purchases per user** (not returned) + 30-day return visits | Satisfied watch time + 7-day return | Value that lasted |
| Fast proxies | CTR · detail-view dwell · **add-to-cart** (bridged to purchase, G15) | CTR · watch quantile · likes | Iteration speed |
| Offline | Recall@k · NDCG · per-head AUC, calibration · coverage | Same | Screens, never decides |
| **Guardrails** | Return rate · hide rate · out-of-stock clicks · small-seller share · sponsored share · latency | Regret · hides · borderline · new-creator reach | Ship blockers |

> **Say:** *"Add-to-cart is available today; a purchase that isn't returned is known a month later. I iterate on the first, weighted by how often it became the second, and I gate on the second."*

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — probe (g) `A1b` `I1a` `I1d`

```
TRAFFIC
  3×10⁸ DAU × ~10 pages = 3×10⁹ page loads/day ≈ 3.5×10⁴ QPS avg, ~10⁵ peak
  RANKER: 10⁵ × ~500 candidates = 5×10⁷ scorings/s at peak → the cost line
EVENTS
  ~200 events per user per day → 6×10¹⁰/day ≈ 7×10⁵ events/s into the stream
  → session features must be a STREAMING job, not a batch one
STATE
  item vectors: 5×10⁸ × 64-d int8 ≈ 32 GB per index replica, sharded
  user sequences: 3×10⁸ users × last 200 events × ~16 B ≈ 1 TB in the online store
  price + stock: ~5×10⁸ items, updated by change events (minutes), not daily
LATENCY BUDGET (first page, p95 ≤ 200 ms; sums to ~170 ms)
  request gate · auth · rate limit                  5 ms
  user + session + negative-list fetch             20 ms
  candidate sources, ALL IN PARALLEL               30 ms
  filters (stock, ships-to, hidden, policy)        10 ms
  pre-ranker (3,000 → 500)                         10 ms
  ranker (500, batched)                            50 ms
  re-rank + sponsored slotting                     15 ms
  network / serialisation                          30 ms
⭐ PAGE 2+: computed while the user reads page 1 → near-zero perceived latency,
   AND it can use what they did on page 1
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: what's shared, and what differs `A5a`

| Component | Shared platform | Products | Videos |
|---|---|---|---|
| Candidate sources | Two-tower ANN · item-to-item · session model · trending · fresh · exploration | + **co-purchase** (complements) · replenishment | + subscriptions · series |
| Ranker | Multi-task, MMoE, position tower | Heads: click · add-to-cart · **purchase (ESMM)** · **return** · hide | Heads: click · watch quantile · like · skip · hide |
| Hard filters | Hidden items, policy | **In stock · ships to user · current price** · age-restricted · already-bought durables | Policy · kids · region rights |
| Re-rank | MMR / caps / exploration slots | + **variant dedupe** (one shoe, not five colours) · sponsored slots | + creator caps · borderline demotion |
| Label delay | — | **Days** (purchase), **weeks** (return) | Minutes (watch), sparse (survey) |
| Other side | — | Sellers, own brand, advertisers | Creators |

| Decision | Options | Choice and why |
|---|---|---|
| Objective | CTR · GMV · **net purchases (+ satisfied watch time for videos)** | CTR and GMV both reward the wrong items |
| Candidates | One two-tower · **several sources, each with a quota** | Co-purchase knows complements; the session model knows today's mission; content knows new items |
| Purchase head | CVR on clicked rows · **ESMM over all impressions** | CVR on clicked rows has sample-selection bias and sparse labels (§5.3) |
| Freshness of user state | Daily batch features · **streaming session features** | Shopping is mission-based: today's intent beats last month's history |
| Negative feedback | A training label only · **a filter now + a feature in seconds + a label tomorrow** | A hide that takes a day to act is ignored feedback |

### The system — *draw this in 90 seconds* `A2a`

```
══ OFFLINE (daily) ═══════════════════════════════════════════════════════════════
 LOGS (impressions WITH POSITION + features AS SERVED) · CATALOGUE · ORDERS · RETURNS
   → labels (§5.2) → train two-tower, session model, ranker (warm-started)
   → item vectors → ANN index · co-view / co-purchase tables (G17) · segment popularity
══ NEARLINE (seconds–minutes) ════════════════════════════════════════════════════
 EVENT STREAM → session features · recent-item sequence · NEGATIVE LIST (hides)
 CATALOGUE CHANGES → price, stock, ships-to · NEW ITEMS → content embedding → FRESH TIER
══ ONLINE · per page ═════════════════════════════════════════════════════════════
┌─ 1. REQUEST + CONTEXT ─────────────────────────────────────────
│ TASK: auth, rate limit; fetch long-term profile, session state, negative list
│ OUTPUT: who, what they're doing now, what they've rejected
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CANDIDATE SOURCES, IN PARALLEL, EACH WITH A QUOTA ─────────
│ two-tower ANN · item-to-item from the last few items (co-view =
│ substitutes, co-purchase = complements) · SESSION MODEL (next item
│ in this mission) · segment-trending · FRESH tier · EXPLORATION ·
│ sponsored candidates from the ad server
│ OUTPUT: ~3,000 candidates, each tagged with its source
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. HARD FILTERS ──────────────────────────────────────────────
│ hidden items + near-duplicates · out of stock · doesn't ship here ·
│ already-bought DURABLES · age-restricted · policy-flagged
│ OUTPUT: only items the user could and would want to see
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. PRE-RANKER → 5. RANKER (multi-task) ───────────────────────
│ target attention over the user's history (which past items matter
│ for THIS candidate) · heads: click, add-to-cart, purchase (ESMM),
│ return, hide · shallow position tower
│ OUTPUT: calibrated heads → an additive value score
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 6. RE-RANK ───────────────────────────────────────────────────
│ variant dedupe · MMR across category / brand / seller · caps ·
│ exploration slots · sponsored slots (capped, labelled)
│ OUTPUT: a page of ~20, each with a reason + provenance
└──────┬─────────────────────────────────────────────────────────
       ▼
 CLIENT → EVENTS → the stream (page 2 is re-ranked from page 1's actions)
 ⚠ the logs are the output of the CURRENT policy: a feedback loop (§7)
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Delayed labels:** *"Purchases arrive in days and returns in weeks, so the purchase head trains on matured windows, and add-to-cart is a fast proxy weighted by how often it became a kept purchase (G15, G22)."*
- **Negative feedback at three speeds:** *"A hide filters the item and its variants from the very next page, updates aversion features within seconds, and becomes a training label tomorrow."*
- **Cold start:** *"New users get a context-and-session pipeline; new items get a content tower and a funded exploration quota (§5.4)."*
- **The latency squeeze** `F2b` `A1c`: *"If the ranker can't score 500 in 50 ms, I won't shrink it for everyone. I'd tighten a distilled pre-ranker and precompute page 2 while page 1 is read."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **One platform, domain-specific heads and filters** | "The boxes are shared; the objective and constraints aren't" | Domain teams share a codebase and must agree on interfaces |
| **2** | **Net purchases, not GMV or CTR** | "A returned purchase is worse than none" | A slower, delayed north star |
| **3** | **Streaming session state** | "Shopping is a mission; today's intent beats last month's history" | A streaming pipeline to run, with its own failure modes |
| **4** | **Negative feedback acts immediately** | "A hide that takes a day to act is ignored feedback" | A nearline negative list on the request path |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f) (i)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 Data sources, by freshness (d) `A2c`

| Source | Tier | What it gives | Watch out for |
|---|---|---|---|
| **Impression logs, with position and served features** | Offline | Labels and the exact features the model saw | Position bias; only shown items have labels |
| **Event stream**: clicks, views, carts, hides, skips | **Online / nearline** | Session intent within seconds | Bots; out-of-order events |
| **Orders and returns** | Offline, delayed | The real labels | Days to weeks late; gifts look like taste |
| **Catalogue**: title, images, attributes, category, brand, seller | Offline + change events | Content features; the only signal a new item has | Seller-written: keyword stuffing, wrong categories |
| **Price, stock, ships-to** | **Online** (minutes) | Hard filters and features | Stale stock means clicks that go nowhere |
| **Search queries** on the platform | Nearline | Explicit intent, the strongest cold-start signal | Privacy of queries |
| **Reviews and ratings** | Offline | Quality priors | **Fake reviews** (§8) |
| **User context**: locale, device, entry point | Online | Cold-start features | The entry point (an ad, a shared link) is strong, short-lived intent |
| **Exploration slice with propensities** | Offline | Unbiased labels and evaluation | Small and precious |

### 5.2 Feedback signals → labels (e) — *the signal table* `F1c`

| Signal | Domain | Polarity · strength | Delay | Trap | Used as |
|---|---|---|---|---|---|
| **Impression** (examined) | Both | The denominator | Instant | Below-the-fold ≠ seen | Exposure; propensity |
| **Click** | Both | + weak | Instant | Thumbnails, accidental taps | Label (click head), feature |
| **View**: detail dwell / video start | Both | + medium | Seconds | Dwell ≠ interest (comparing prices) | Label |
| **Watch time** | Videos | + strong, **if duration-normalised** | Minutes | Length and sensationalism bias (R4) | Quantile label |
| **Add to cart / wishlist** | Products | + strong fast proxy | Minutes | Carts abandoned; wishlists aspirational | Label + bridge to purchase |
| **Purchase** | Products | + strongest | Hours–days | Gifts; attribution window | Label (ESMM) |
| **Return / cancel** | Products | **− cancels the purchase** | Up to ~30 days | Size and fit vs dislike | Return head; corrects purchase labels |
| **Like / save / share** | Both | + strong, sparse | Instant | — | Label |
| **Skip**: swipe away < 2 s, or scrolled past while examined | Both | − weak | Instant | Context, mood; only if examined | Weak negative |
| **Hide / not interested** | Both | **− strong** | Instant | — | **Filter now** · aversion feature · label |
| **Don't show seller / brand / channel** | Both | − very strong, **broad** | Instant | — | Filter on the whole entity · label |
| **Report** | Both | − policy | Instant | Brigading | Trust & safety queue; never only a ranking signal |
| **Repeat purchase / rewatch** | Both | + for consumables; ⚠ not for durables | Days–weeks | The fridge problem (FU 9) | Replenishment source; durable filter |

**Labelling rules:**
- **Unexamined ≠ negative.** Items below the scroll depth are dropped, not labelled 0.
- **Purchase labels mature:** the purchase head trains on impressions ≥ 7 days old (the attribution window), and the return head on purchases ≥ 30 days old. Fresher data is used through the add-to-cart bridge, or a delay model (G22).
- **Gifts and one-off events** (holiday, baby shower) are down-weighted in taste, so one gift purchase doesn't define a profile.
- **Bots and fraud removed first:** click farms' events are poisoned labels, not users.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective |
|---|---|---|
| **Two-tower retrieval** | User and item embeddings; item vectors precomputed | Sampled softmax with in-batch + uniform negatives, **logQ-corrected** (R4 §5.2 derives it and its bias) |
| **Session / sequence model** | Next item in **this** mission, from the last ~50 events | Transformer over the event sequence (SASRec-style), next-item softmax; serves as a candidate source |
| **Item-to-item tables** | Substitutes (co-view) and complements (co-purchase) | PMI-normalised co-occurrence (G17); a complement isn't a "similar" item |
| **Multi-task ranker** | Value per candidate | MMoE; **target attention** over history (DIN-style: weight past items by relevance to the candidate); BCE per head; **ESMM** for purchase; shallow position tower |
| **Re-ranker** | The page as a set | Greedy MMR under hard constraints |

```
⭐ ESMM: THE PURCHASE HEAD WITHOUT SAMPLE-SELECTION BIAS (Alibaba, SIGIR 2018)
  The naive CVR model trains P(buy | click) on CLICKED rows only, then
  serves on ALL impressions: a train/serve mismatch, and very sparse labels.
  ESMM learns two things over the ENTIRE impression space:
      pCTR    = P(click | impression)
      pCTCVR  = P(click AND buy | impression) = pCTR × pCVR
      L = Σ_impressions  BCE(y_click, pCTR)  +  BCE(y_click · y_buy, pCTR × pCVR)
  pCVR is never supervised directly; it's learned through the product, on
  every impression, and shares embeddings with the dense click task.

THE VALUE SCORE (additive in calibrated heads, weights owned by product):
  products: V = pCTR · pCVR · (1 − pReturn) · u(price)  +  w_atc · pATC  −  w_hide · pHide
            u(price) = 1 (count purchases) or a mild, concave price term: pure
            GMV pushes expensive items whatever the fit
  videos:   V = pCTR · (w_w · ŷ_watch + w_l · pLike)  −  w_skip · pSkip  −  w_hide · pHide
```

### 5.4 Cold start: new users and new items (i) `F2a` `F1c`

**New user (zero history):**

| Step | What happens | Why |
|---|---|---|
| **Context at request** | Locale, device, time, **entry point** (the ad or link they came from), signed-out search queries | The entry point is the strongest intent a new user has |
| **Optional onboarding** | Three taps on interests; skippable | Explicit, cheap, but many skip it |
| **First page** | Segment-popular + trending, **deliberately spread across categories** | Page 1 is a measurement as much as a recommendation |
| **In-session** | Streaming session features → the session model; **page 2 reflects page 1** within seconds | Session signal beats segment priors after a handful of clicks |
| **Category bandit** | Thompson sampling on a Beta posterior per category to set the mix | Explores without wasting the page |
| **Graduation** | Blend weight moves from segment/session toward the personal model as history grows | No cliff between "new" and "known" |

**New item (zero interactions):**

| Step | What happens | Why |
|---|---|---|
| **Eligibility gate** | Listing quality, policy, counterfeit checks | Unverified items shouldn't get free exposure |
| **Content tower** | Image + title + attributes + price + category → item vector; trained with **ID dropout** so it works without an ID embedding | Places the item near its likely audience on day 0 |
| **Priors** | CTR / CVR priors from the k nearest items in content space and from the **seller's** history | The ranker needs a starting estimate |
| **Fresh tier + exploration quota** | Streaming upsert into the ANN; guaranteed impressions with **logged propensities**, capped; boost decays as the posterior tightens | It must earn real labels, and fast |
| **Videos** | Creator's subscribers first, then content-similar audiences | The creator graph is the warm start |

### 5.5 Fine-tune an existing model (f) `F2c`

A pretrained **image–text encoder**, LoRA-tuned contrastively on **co-purchase and co-view pairs**, so its embedding learns what shoppers treat as related (a tent and a sleeping bag), not only what looks alike. This content tower is what makes the new-item path work. Daily training **warm-starts** from yesterday's checkpoint.

### 5.6 The staged pipeline `A2d`

```
STAGE 1 · CLEAN       bots, click farms, fake-review rings removed FIRST
STAGE 2 · LABEL       examined-only · matured purchases · return join · gift
                      down-weighting · propensities on the exploration slice
STAGE 3 · TRAIN       two-tower · session model · ESMM ranker (warm-started) · calibrate
STAGE 4 · INDEX       item vectors daily · fresh tier streaming · co-view / co-purchase daily
STAGE 5 · GATE        offline screens → shadow → A/B with the guardrail panel
CADENCE: session state seconds · price/stock minutes · fresh items ~1 h · models daily
```

---

## 6 · SERVING *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
ONE FEATURE DEFINITION, TWO STORES:
  offline store (training) and online store (serving) computed from the same
  definitions; and the features the model ACTUALLY SAW are LOGGED AT SERVING
  TIME and used for training → training–serving skew can't creep in silently

PAGE 2 IS COMPUTED WHILE PAGE 1 IS READ:
  the candidate set is cached per session; page 2 re-ranks it with page-1
  actions (clicks pull neighbours up; hides remove them) → fast AND adaptive

STOCK AND PRICE ARE CHECKED TWICE: at ranking, and at render
  (a sold-out item on the page is a broken promise)
```

- **The re-rank objective** `A3b`: greedy MMR, `λ·V(v) − (1 − λ)·max_{u ∈ S} sim(v, u)` over category, brand and seller, then hard constraints: one variant per product, ≤ 2 per seller in any 10, exploration slots, and sponsored items in **fixed, labelled, capped slots**. Sponsored items never get a hidden boost to organic rank.
- **Efficiency** `F3a`: int8 item vectors (≈ 4× less index memory, gated on recall@k); a **pre-ranker distilled** from the ranker; batched GPU or CPU scoring.
- **Scale** `F3b`: ANN sharded and replicated; online store sharded by user; precomputed segment-popular lists; page 2 from cache.
- **Reliability** `I1b`: per-source timeouts with quota refill → if personal sources fail, the **first-session pipeline** (§9) → if the ranker fails, segment-popular under the same filters. **Never a blank, never a sold-out item.**

---

## 7 · EXPERIMENTATION, MONITORING, GUARDRAILS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
OFFLINE SCREENS (time-split)
  recall@k, sliced by popularity decile, new items and new users separately
  · NDCG@k · per-head AUC, normalised entropy, CALIBRATION · coverage
  · counterfactual replay (SNIPS) on the exploration slice
  ⭐ an offline metric earns its job by having predicted past A/B outcomes
SHADOW   latency, calibration, overlap, filter-violation rate (stock, policy)
A/B      randomise by USER · 1% → 5% → 50% · ≥ 2 weeks, AND read purchases
         after the 7-day window and returns after 30 · CUPED on heavy-tailed
         spend
  ⚠ MARKETPLACE INTERFERENCE: treatment and control share inventory and
    ad budgets. If treatment pushes a scarce item, control can't buy it;
    the measured lift is inflated.
    → for supply-affecting changes: randomise by ITEM or category cluster,
      or budget-split the sponsored side; run an A/A to check
LONG RUN a ~1% holdback for months: return visits, net purchases
```

**Success** = net purchases per user up, return visits up, every guardrail held. CTR up alone is **not** success. `I2b`

**Guardrails (ship blockers):**

| Products | Videos |
|---|---|
| Return rate · hide rate · out-of-stock clicks · complaint rate | Regret · hide rate · abandon rate |
| **Small-seller impression share** · seller concentration | **New-creator reach** · creator concentration |
| Sponsored share of impressions · own-brand share | Borderline impressions |
| p95 latency · cost per 1,000 pages | p95 latency · cost per 1,000 pages |

**Monitoring:**

| Layer | Monitor | Catches |
|---|---|---|
| **Data** | Event volume and lag per stream · null rates · label maturity | A broken pipeline before the model learns from it |
| **Skew** | Served-feature vs training-feature distributions | Training–serving skew (FU 10) |
| **Model** | Calibration per head and per segment · per-source contribution | Silent degradation; a dead source |
| **Product** | Impression concentration · new-item and small-seller share · coverage | The feedback loop narrowing the feed |
| **System** | p95 / p99 per stage · fallback rate · stale-stock rate | Latency creep; broken promises |

**Bias sources and mitigations** `F1c`:

| Bias | Mechanism | Mitigation |
|---|---|---|
| **Position** | Top slots are clicked regardless | Shallow position tower; randomised slice for propensities |
| **Selection / exposure** | Only shown items have labels | Exploration slice; ESMM over all impressions |
| **Popularity** | Positives skew to the head | logQ correction understood (R4 FU 1); quotas; sliced recall |
| **Delayed labels** | Recent purchases not yet arrived | Matured windows; add-to-cart bridge; delay modelling (G22) |
| **Price** | GMV-weighted labels favour expensive items | Count-based or concave price term |
| **Seller size** | Big sellers have more data, so better estimates, so more exposure | New-seller exploration; small-seller share guardrail |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | First page p95 ≤ 200 ms; page 2 prefetched; cost per 1,000 pages, with the ranker as the cost line | p95 per stage · $/1k pages |
| **Reliability** `I1b` | Source timeouts with refill; first-session fallback; segment-popular fallback; stock re-checked at render | Fallback rate · sold-out-on-page rate |
| **Usability** `I1c` | Diverse pages; one variant per product; reasons; *Not interested*, *Don't show this brand*, *Already bought it* act on the **next page** | Hide rate · time to first useful click |
| **Scalability** `I1d` | Sharded ANN + fresh tier; streaming session store at 7×10⁵ events/s; precomputed segment lists | QPS headroom · stream lag |
| **Privacy** `I1e` | Purchases reveal health, pregnancy, sexuality, finances: **sensitive categories are excluded from inference**, from reasons and from email (the widely reported retail case of inferring a pregnancy from purchases is the cautionary tale); shared devices; query and history deletion honoured | Sensitive-inference audits · deletion latency |
| **Safety** `I1f` | Recalled and dangerous products filtered; age-restricted items gated; **counterfeits** blocked at eligibility; for videos, borderline demotion and kids rules | Recalled-item impressions = 0 |
| **Security** `I1h` | OAuth, rate limits, and **beyond the gateway**: ⚠ **click farms and fake reviews** boosting listings → fraud filtering before training, trust-weighted signals; ⚠ **keyword-stuffed titles and wrong categories** gaming content retrieval → listing-quality models; ⚠ **scraping** of recommendations for competitive intel → rate and pattern detection | Fraud events caught · listing-quality violations |
| **Ethics / bias** `I1i` | Small-seller exposure funded; **no self-preferencing of the own brand** (EU DMA prohibits it for gatekeepers); sponsored items disclosed and capped; no sensitive attributes as features | Small-seller share · own-brand share vs organic baseline |
| **Explainability** `I1j` | A reason per card from **source provenance**; "Sponsored" labels; an additive value score with per-head attribution logged; **the main ranking parameters disclosed and a non-profiling feed option** (EU DSA obligations for very large platforms) | % cards with a reason · non-personalised option usage |

> ⭐ *"A recommendation feed sells three things at once, the user's interest, the seller's product and the platform's margin, and the user's interest has to win every tie."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* `F3c` `F2a` `A5b`

**Extensions** `F3c`:
- **Shoppable video:** both domains at once. A video feed whose items carry products. The shared platform pays off: the video ranker gains a **purchase head** (ESMM, from video impression to product purchase), and the product side gains video as a content source. The risks: purchase labels are even sparser per video, and creators' incentives shift toward selling. Start with tagged products in a quota of the feed.
- **Conversational steering** ("cheaper ones", "less of this brand"): an LLM turns the request into **constraints and filters** for the same ranker; it never names products itself, because it can hallucinate items that don't exist or aren't in stock. The cost is an LLM call per steer, and the maturity risk is mis-parsed constraints, so it's opt-in and logged.

**When there's no personal signal: the first-session pipeline** `A5b` `F2a`:

```
a new user · a signed-out user · a non-profiling choice · OR the personal sources failed
  → context sources: entry point (the ad / link), locale, device, time
  → segment-popular + trending, spread across categories (a measuring page)
  → the category bandit sets the mix; the SESSION model takes over after a
    handful of clicks; page 2 reflects page 1
  → the same filters and re-rank: stock, ships-to, policy, diversity, disclosure
  "Less personal, still relevant, and it learns within one session."
```

**Top 3 failure modes:**
1. **Proxy drift:** optimising CTR or GMV fills the feed with thumbnails or expensive items, and returns rise a month later.
2. **The fridge problem:** a just-bought durable keeps getting recommended, because "purchased" looks like "interested" (FU 9).
3. **Cold-start starvation:** new items and small sellers never get exposure, the catalogue's tail goes dark, and the platform stops attracting sellers.

**Q1 ships:** the two-tower + item-to-item + session sources, the **ESMM ranker with return and hide heads**, the three-speed negative feedback path, the new-item content tower with an exploration quota, and **logging features at serving time**.
> *"Logged serving features and an exploration slice in Q1, because both are impossible to add retroactively, and without them every later model is trained on skewed features and the old policy's choices."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Products or videos: what actually changes in your design?"** `A5a` `F1b`
> Less than people expect in the boxes, and almost everything in what's inside them. Shared: the funnel of candidate sources, a multi-task ranker, a constrained re-ranker; the two-tower with logQ correction; position debiasing; the feature store and streaming session state. Different: **the objective**, net purchases for products and satisfied watch time for videos. **The labels and their delays**: a purchase arrives in days and a return in weeks, against a watch that resolves in minutes, so the product side needs matured windows and a fast-proxy bridge. **The hard filters**: stock, shipping, current price and already-bought durables have no video equivalent, while videos have borderline content and kids rules. **Consumption**: you buy a fridge once but watch a creator weekly, so repeat-recommendation logic is opposite. **Intent**: shopping is mission-based, so the session model carries more weight, while video browsing is mood-based and broader. And **the other side of the market**: sellers, an own brand and advertisers bring conflicts of interest that creators mostly don't. I'd build one platform with domain-specific heads, filters and value functions, and own the value function per domain.

**2. "A brand-new user opens the app. What do they see on page 1, and on page 3?"** `F2a` `I1c`
> Page 1 is part recommendation, part measurement. I use what the request gives me: locale, device, time, and above all the **entry point**. If they arrived from an ad for running shoes, that's the strongest intent signal I'll have, so the first rows lean into it. The rest of page 1 is segment-popular and trending items **deliberately spread across categories**, because I need to learn which ones they respond to; a category bandit, Thompson sampling on per-category Beta posteriors, decides the mix. Every click streams into session features within seconds, so **page 2 is built from page 1**: the session model, trained on next-item prediction within missions, takes over after a handful of interactions. By page 3 the feed is mostly session-driven, still with an exploration slot. An optional three-tap interest picker helps, but I wouldn't depend on it. And there's no cliff: the blend shifts from segment and session toward the personal model as history accumulates.

**3. "A new product with no interactions: how does it get its first impressions?"** `F2c` `I1i`
> First it has to be **eligible**: listing quality, policy and counterfeit checks, because free exposure for an unverified listing is how counterfeits spread. Then a **content tower**: a pretrained image–text encoder, LoRA-tuned on co-purchase and co-view pairs, embeds the image, title and attributes, and the item tower is trained with ID dropout, so it produces a sensible vector with no ID history. That vector goes into a **streaming fresh tier** of the index within the hour. The ranker needs a starting estimate, so I seed **CTR and CVR priors** from the nearest items in content space and from the **seller's** track record. Then a **funded exploration quota**: guaranteed impressions, capped, with logged propensities, and a boost that decays as its own posterior tightens. I'd monitor time-to-first-sale for new items, sliced by seller size, because big sellers have priors that small ones don't, and that's where unfairness hides.

**4. "A user hides an item. What happens, and when?"** `I1c` `A2a`
> Three things, at three speeds. **Immediately**: the item goes on a nearline negative list that the hard filter reads on the next request, so it's gone from the next page, along with its **variants and near-duplicates**, because hiding a red shoe and then seeing the blue one feels like being ignored. **Within seconds**: aversion features update in the session state (hidden category, brand, price band), so the ranker pushes similar items down on page 2 without a retrain. **Tomorrow**: the hide is a strong negative label for the hide head, and its count is an input feature. The hide's scope matters: "not interested" is item-level, while "don't show this brand" is a filter on the whole entity and is honoured indefinitely. I'd also watch for the reverse failure: a burst of hides in one session might be a shared device or a mood, so aversions decay unless they're repeated.

**5. "Purchases take days and returns take weeks. How do you train on that?"** `F1c` `F1a`
> Three layers. **Matured windows**: the purchase head trains on impressions at least seven days old, inside the attribution window, and the return head on purchases at least thirty days old. That's clean but stale. **A fast-proxy bridge**: add-to-cart arrives in minutes, and I weight it by how often similar add-to-carts became kept purchases. That's how I iterate weekly without pretending a cart is a sale; it's the same bridge as installs-to-retention in G15. **Model the delay**: for the freshest data, a delayed-feedback model treats "no purchase yet" as unknown rather than negative, using an estimate of the conversion-delay distribution (G22). For the purchase head itself I use **ESMM**, pCTCVR = pCTR × pCVR over all impressions, so the purchase model isn't trained only on the clicked subset it never serves on. And in experiments, I read purchases after the window and returns after thirty days, because a week-one win can be a month-two loss.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Clicks are up 6% and purchases are flat. Ship it?"** `F1b` `I2b`
> No, and the pattern is diagnostic. More clicks without more purchases means the model got better at **attracting** attention without getting better at **satisfying** it: thumbnails, eye-catching prices with expensive shipping, items that look good in the feed and disappoint on the detail page. I'd check the detail-page dwell and add-to-cart rate per click, which should have fallen; the **return rate** and hide rate once they mature; whether the gain is concentrated in a category or a seller cluster; and novelty, by reading the trend over the test rather than the average. The fix is usually in the value function: if the click head's weight is too high relative to purchase and return, the ranker is doing what it was told. A flat north star with a rising proxy is a regression in disguise, because it spends user attention for nothing.

**7. "Your A/B shows +3% purchases. Is the result real?"** `I2a` `F1b`
> Possibly inflated, because a marketplace violates the independence an A/B assumes. Treatment and control share inventory and ad budgets: if treatment steers users toward a scarce item, control users find it sold out; if treatment shifts sponsored impressions, it spends budget that control would have used. Both make treatment look better than it would be at 100%. So for changes that move supply, I'd randomise by **item or category cluster** rather than by user, or **budget-split** the sponsored side so each arm has its own budgets, and run an A/A to calibrate. I'd also check the readout timing (purchases after the attribution window, returns after thirty days) and use CUPED, because spend is heavy-tailed. If the user-randomised and item-randomised estimates disagree, the gap *is* the interference, and the item-side number is the one to plan on.

**8. "The business wants to boost high-margin items, sponsored items and the platform's own brand."** `I1i` `A3b`
> These are three different requests, and I'd answer each as a policy decision, not a ranking tweak. **Sponsored items** go in **fixed, labelled, capped slots**, filled by the ad auction and held to the same relevance floor and filters, never a hidden boost in organic ranking, because undisclosed promotion is what destroys trust in a feed. **Margin** can enter the value function, but as an explicit, owned, small term with the user-side guardrails (returns, hides, return visits) as hard constraints, and its cost is reported, not buried in an average. **The own brand** is the sharpest: self-preferencing in ranking is exactly what the EU's DMA prohibits for gatekeepers, and it erodes seller trust everywhere else. So own-brand items compete on the same value function as everyone else, with an own-brand share monitored against an organic baseline. The principle: the user's interest wins ties, and every commercial lever is visible, bounded and owned.

**9. "Someone buys a fridge, and their feed fills up with fridges."** `F2a` `I1c`
> The classic e-commerce failure: the model reads "purchased a fridge" as "very interested in fridges", which is true until the moment they buy one. Three fixes. **Classify consumption**: durables (fridges, laptops) versus consumables (coffee, filters) versus collectibles (books, where more of the same is fine), learned from repeat-purchase rates per category. **After a durable purchase**, filter that category's substitutes for a period and switch the post-purchase surface to **complements**: a water filter for the fridge, a case for the phone, from co-purchase tables rather than co-view, because co-view encodes "considered instead of" and co-purchase encodes "bought together with". **For consumables**, a **replenishment** source timed to the typical repurchase interval. And a label fix: in training, purchase-of-a-durable shouldn't become "more of this category"; the substitutes the user *didn't* buy were considered and rejected, not desired.

**10. "Recommendations got worse after a feature-pipeline change, and nothing alerted."** `I1b` `F2a` `I1h`
> That's training–serving skew, the most common silent failure in recommenders. Somewhere, a feature at serving time no longer matches what the model trained on: a unit change, a default value for missing data, a join that started returning stale prices. The model doesn't crash; it quietly scores with wrong inputs. Prevention: **one feature definition** computed for both the offline and online stores, and **logging features at serving time**, training on what the model actually saw. Detection: **served-versus-training distribution monitors** per feature (null rates, means, quantiles), calibration per head on live traffic, and per-source contribution panels, any of which would have alerted here. Recovery: roll back the pipeline, not the model, and backfill. One more angle: a sudden shift in a feature's distribution can also be **adversarial**, a click farm inflating a popularity feature, so the same monitors feed fraud review.

---

## ⭐ THE L6 LINE

> **"One platform, two domains: the funnel is shared, the objective, labels, filters and cold start are not. Feedback acts at three speeds, so a hide changes the next page, and the north star is purchases that aren't returned or watches that aren't regretted, never the click."**

## 🧭 HOOKS — *steer by what you mention*

- *"One platform, two domains"* → invites the shared-vs-different branch (**good at L6**)
- *"Feedback acts at three speeds"* → invites the negative-feedback and streaming branch
- *"A returned purchase is worse than none"* → invites the delayed-label branch
- *"ESMM over all impressions"* → invites the sample-selection-bias branch
- *"The fridge problem"* → invites the substitutes-vs-complements branch
- *"Marketplace interference"* → invites the experimentation branch (**good at L6**)
- *"The user's interest wins every tie"* → invites the ads, margin and own-brand branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to what's shared and what differs."* The table in §4 regenerates the design.
- **Running out of time?** → *"I have five minutes. I'd rather cover cold start and guardrails than finish the re-ranker. Sound right?"* (Both are named items in this prompt.)

**Name and defer:**
- "**Search ranking** shares the ranker's features but has a query; it's G7."
- "**The ads auction** supplies sponsored candidates; pricing and bidding are G21."
- "**Email and push** reuse the models but add a send-time and fatigue problem."
- "**Pricing and promotions** affect every label but are a separate system, with fairness questions of their own."
- "**Long-term value / RL** over a user's lifetime is a research direction I'd scope separately."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path | §0, §4 | *"One platform: sources, a multi-task ranker, a constrained re-ranker; domain-specific heads and filters."* |
| **A1b** | Latency and capacity budgets | §1, §3 | *"5×10⁷ scorings/s; 7×10⁵ events/s; 170 ms of a 200 ms p95; page 2 prefetched."* |
| **A1c** | Degrade without lowering quality | §4, §6 | *"A distilled pre-ranker and page 2 computed while page 1 is read."* |
| **A2a** | The full stack | §4 diagram, FU 4 | Offline / nearline / online; sources → filters → rankers → re-rank → events |
| **A2b** | Model components | §5.3 | Two-tower, session model, item-to-item tables, ESMM ranker, re-ranker |
| **A2c** | Data sources | §5.1 | Logs, event stream, orders and returns, catalogue, price/stock, queries, reviews, context |
| **A2d** | Staged pipeline | §5.6 | Clean → label → train → index → gate |
| **A3a** | Top-down, unprompted | §0 | *"One platform, two domains"* + the plan |
| **A3b** | A principled objective for the trade-off | §5.3, §6, FU 8 | *"An additive value score with owned weights; MMR under hard constraints; sponsored in fixed, labelled slots."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Products or videos? Marketplace, ads, own brand? Objective? Regions? |
| **A5a** | Alternatives compared | §4, FU 1 | Shared vs domain-specific; CTR vs GMV vs net purchases; CVR vs ESMM; batch vs streaming |
| **A5b** | A full alternative pipeline on demand | §9 | The first-session pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3, FU 5 | *"ESMM over all impressions; a session transformer for missions; target attention over history."* |
| **F1b** | Metrics and their trade-offs | §2, §7, FU 6, 7 | *"Add-to-cart is available; a kept purchase is meaningful."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 5 | Position, selection, popularity, delayed labels, price, seller size |
| **F2a** | Something fails → a fallback | §5.4, §9, FU 9, 10 | First-session pipeline; the fridge fix; skew recovery |
| **F2b** | Latency too tight → design for it | §4 | Pre-ranker and page-2 prefetch, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.5, FU 3 | A LoRA-tuned image–text encoder on co-purchase pairs |
| **F3a** | Efficiency with a quality trade | §6 | int8 vectors gated on recall; a distilled pre-ranker |
| **F3b** | Scale levers | §3, §6 | Sharded ANN + fresh tier; streaming session store; cached segment lists |
| **F3c** | Extension, with cost and maturity risk | §9 | Shoppable video; LLM conversational steering as constraints |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | First page p95 ≤ 200 ms; cost per 1,000 pages |
| **I1b** | Reliability | §6, §8, FU 10 | Fallback chain; stock re-checked at render; skew monitors |
| **I1c** | Usability | §2, §8, FU 4, 9 | One variant per product; controls that act on the next page |
| **I1d** | Scalability | §3, §8 | Streaming at 7×10⁵ events/s; sharded index and store |
| **I1e** | Privacy | §8 | Sensitive categories excluded from inference, reasons and email |
| **I1f** | Safety | §8 | Recalled, dangerous, age-restricted and counterfeit items filtered |
| **I1h** | Security | §8, FU 10 | **Click farms, fake reviews, keyword stuffing, scraping** |
| **I1i** | Ethics / bias | §8, FU 3, 8 | Small-seller exposure; no own-brand self-preferencing; disclosed, capped sponsorship |
| **I1j** | Explainability | §8 | Reasons from provenance; "Sponsored"; a non-profiling option and disclosed parameters |
| **I2a** | Offline → online | §7, FU 7 | Sliced screens → shadow → user- or item-randomised A/B → holdback |
| **I2b** | The success definition | §7, FU 6 | *"Net purchases and return visits up, guardrails held; CTR alone is not success."* |

**🧭 Probe map:** (a) assumptions, surfaces → §1, §2 · (b) feedback and success → §2, §5.2 · (c) candidates, ranking, re-ranking → §4, §5.3 · (d) data sources → §5.1 · (e) labels → §5.2 · (f) models, training → §5.3, §5.5, §5.6 · (g) serving, latency → §3, §6 · (h) experiments, monitoring, guardrails → §7 · (i) cold start → §5.4, §9.
