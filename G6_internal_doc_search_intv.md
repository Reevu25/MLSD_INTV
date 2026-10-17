# G6 · Internal Document Search — **INTERVIEW SHEET**

> **Tier 3 · #24** — RAG with ACLs; the enterprise variant. **Read [`Q6`](Q6_rag_retrieval_intv.md) first** — this sheet covers only what's *different*.
> Full study file: [`../G6_internal_doc_search.md`](../G6_internal_doc_search.md) · [diagrams](../G6_internal_doc_search_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the employee sees · (c) ACL-aware retrieval · (d) datasets · (e) training data that respects ACLs · (f) the models and their objectives · (g) serving: the read boundary, caching, revocation · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Permission model? Revocation SLO? Answer or retrieve? How many source systems? (§1) |
| **Explainability** is easy points | Every claim cited, **every citation openable by this user**; an audit log of what was read for whom (§2, §8) |
| **Security** beyond the gateway | **Cache keys**, **side channels** (counts, latency), the agent's **confused-deputy** credential, injected text inside indexed docs (§6, §8) |
| **Troubleshoot before being asked** | Raise **filtered-ANN fragmentation** and **revocation lag** yourself (§3, §4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **Retrieval quality and access control are the same system — and ACL wins** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · zero leaks as the SLO · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + THE TWO FAILURES** | **(b)**, (h) | Post-filtering: a quality failure *and* a leak channel | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Smaller corpus, harder constraint · **principal set bounded at a few hundred** | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | ACL as index terms · filter during traversal · authoritative re-check | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | ⚠ **Weights have no ACL** · role-stratified judgements · the models | A2b–d, F1a, F1c, F2c |
| **32–38** | **SERVING** | **(g)** | ⚠ **The summariser** · cache key with the ACL hash · revocation < 60 s | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Permission tests as **CI** · recall **for restricted users** | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Agents, five SaaS sources · **most-restrictive fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"The constraint that makes this different from every other search problem is that retrieval quality and access control are the same system, and access control wins every conflict. In web search a bad result is embarrassing. Here a bad result is a salary spreadsheet shown to someone who shouldn't see it — a security incident with legal and contractual consequences, and for an enterprise product it's the thing that loses the account.**
>
> **So the design centre is one rule: ACL filtering happens before ranking, never after. And I want to be precise about why, because 'filter afterwards' is the intuitive design and it fails in two distinct ways.**
>
> **The first is a quality failure: post-filtering destroys top-k. Retrieve a hundred, filter down to three, and a user with narrow access gets a nearly empty page for a query the corpus can answer — their experience degrades exactly in proportion to how restricted they are, which is backwards."**

**Then the serious one — the leak-channel argument:**
> ⚠ **"Any signal derived from a document the user cannot see is a leak channel. Result counts leak. Ranking scores leak. 'Related documents' leaks. And in an agent the summariser is the worst of all — if the model reads a document the user cannot open and paraphrases it into an answer, you have leaked the contents in a form no ACL check will ever catch, because the output is newly-generated text that no longer carries the document's identity.**
>
> **So the invariant is stronger than filtering results: the model must never see a document the user could not open themselves."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the employee sees and the two failures, the shape of the filter, the retrieval architecture, the data and models — including why weights can't hold restricted content — serving, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **How are permissions modelled — users, groups, nested groups, deny rules?** *(Determines whether the principal set is bounded.)*
2. **What's the ACL propagation SLO from the source systems?** *(Revocation lag is the exposure window.)*
3. **Does it answer, or just retrieve?** *(If it generates, the summariser leak is the dominant risk.)*
4. **How many source systems, with what permission models?** *(The ACL is only as good as each connector's fidelity — FU 8.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Customer | One enterprise tenant: ~100k employees |
| Corpus | Tens of millions of documents from several SaaS sources (docs, wiki, tickets, chat) |
| Permissions | Users, nested groups, sharing links, some deny rules — owned by the sources |
| Product | Search **and** generated answers with citations; also callable by agents |
| Tenancy | Many tenants on shared infrastructure; **nothing crosses a tenant boundary** |

**FR:** hybrid retrieval over connected sources · **ACL-aware retrieval** · freshness with fast revocation · citations · agent-callable.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **Permission correctness** | **Zero leaks.** A security property, not a quality metric |
| **ACL revocation** | **< 60 s** — same fast path as deletion |
| Recall@k **for a restricted user** | Comparable to an unrestricted one — *this is what pre-filtering buys* |
| Query latency p95 | ≤ 300 ms (retrieval); answers stream after |

**Scope / NON-GOALS:**
- ❌ **Post-filtering.** Position, with both reasons.
- ❌ **Letting the model read anything the user can't open.** The invariant.
- ❌ Being the permission source of truth (I consume it; the source systems own it).

---

## 2 · PRODUCT + THE TWO FAILURES *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the employee sees (b):**

```
[ what's our parental leave policy for contractors? ]

  Contractors are covered by the vendor's policy, not ours; the
  exception is … [1] [2]                        ← every claim cited
  [1] HR Policy Hub › Leave › Contractors         ← openable by THIS user
  [2] Vendor Management FAQ (updated 3 days ago)

  When nothing accessible answers:
  "I couldn't find anything about that."          ← NOT "there are documents
                                                     you can't access"
```

```
POST-FILTERING FAILS TWICE, AND THEY'RE DIFFERENT FAILURES:

1. QUALITY FAILURE — post-filtering destroys top-k.
   retrieve 100 → filter to 3 → a narrow-access user gets a nearly
   empty page for a query the corpus CAN answer.
   ⚠ and the 97 that should have filled those slots were NEVER RETRIEVED.
   → experience degrades IN PROPORTION TO HOW RESTRICTED YOU ARE,
     which is exactly backwards.

2. ⚠ LEAK CHANNEL — the serious one.
   ANY SIGNAL DERIVED FROM AN UNSEEABLE DOCUMENT LEAKS:
     · result counts ("47 results" vs "3 results")
     · ranking scores · "related documents" · latency differences
     · autocomplete and query suggestions
     · ⭐ THE SUMMARISER — worst of all (§6)

⭐ THE INVARIANT: THE MODEL MUST NEVER SEE A DOCUMENT THE USER
   COULD NOT OPEN THEMSELVES.
   (stronger than "don't show it" — it's "don't read it")
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **Gate** | **Permission tests in CI: leaks = 0** (same query as different users, canary docs) | Blocking |
| **North star** | **Search/answer success for restricted users**, close to unrestricted users | Pre-filtering works for the people it's for |
| Quality | Recall@k by **filter-selectivity band** · answer groundedness · citation accuracy | Where the failure hides |
| **Guardrails** | Revocation p99 < 60 s · abstention rate (zero means something leaks) · p95 latency | Hard limits |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **the constraint, not the scale** `A1b` `I1a` `I1d`

```
CORPUS: enterprise-scale, not web-scale — tens of millions of docs.
  → ANN is comfortable. THE SCALE IS NOT THE PROBLEM.

THE PROBLEM IS THE FILTER, AND ITS SHAPE:

  USER'S TRANSITIVE PRINCIPAL SET
    = {user_id} ∪ {direct groups} ∪ {groups of groups} ∪ {domain}
    ⚠ BOUNDED AT A FEW HUNDRED ENTRIES in practice — which is what
      makes pushing it into the query FEASIBLE.
    (if it were unbounded, this design wouldn't work)

  → resolve it once per query (cached, with its own invalidation)
  → push it into the posting-list traversal as a REQUIRED DISJUNCTION
  → ACL BITMAP per principal for O(1) membership during HNSW traversal

⚠ NESTED GROUPS make resolution a graph walk; DENY RULES break the
  clean "any overlap" test and force real evaluation. Both are
  reasons to resolve ONCE per query rather than per document.

THE HIDDEN COST: a selective filter (0.1% of docs) FRAGMENTS the HNSW
  graph — latency blows up or recall silently collapses for exactly
  the most restricted users (FU 6) → partitioned indexes for large
  stable groups; lexical fallback past a selectivity threshold
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Where the ACL applies | Post-filter vs **inside retrieval** (index terms + bitmaps) vs per-group partitioned indexes | Inside retrieval by default; partitions for large stable groups |
| Generation safety | Retrieve broadly, filter the output vs **an invariant at the read boundary** | A paraphrase carries no document identity |
| Source of truth | Trust the index ACL vs **authoritative re-check on survivors** | The index is an optimisation; the permission service is the truth |
| Cache key | Query vs **query + ACL-group hash** | Query-only keys leak across users |
| Agent credential | A broad service account vs **attenuated to the requesting user** | Otherwise every agent query is a privilege escalation |

### The system — *draw this in 90 seconds* `A2a`

```
 QUERY + USER
       │
┌─ 1. RESOLVE THE TRANSITIVE PRINCIPAL SET ──────────────────────
│ {u:bob, g:eng, g:eng-leads, g:all-staff, domain:corp}
│ OUTPUT: a few hundred principals, cached with invalidation
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. RETRIEVAL WITH THE ACL PREDICATE *INSIDE* ─────────────────
│ BM25: principal set as a REQUIRED DISJUNCTION in posting lists
│ ANN:  ACL BITMAP probed at each HNSW hop
│ ⚠ INACCESSIBLE DOCS ARE NEVER CANDIDATES
│ OUTPUT: RRF-fused ~100, all accessible
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. AUTHORITATIVE RE-CHECK vs the permission service ──────────
│ closes the PROPAGATION-LAG window on the survivors
│ OUTPUT: ~100 verified twice
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. CROSS-ENCODER RERANK ──────────────────────────────────────
│ OUTPUT: ~10
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. ⚠ GENERATION — reads ONLY these ~10 ───────────────────────
│ the summariser cannot leak what it never received
│ OUTPUT: answer + citations, each openable by this user
└────────────────────────────────────────────────────────────────

 INDEXING: each doc indexed WITH its principal set as terms
           ⚠ an ACL change is an INDEX UPDATE with a 60 s SLO
 AUDIT:    every doc id read, per query, per principal — immutable
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `I1b` `F1c`

- **Selective filters fragment ANN:** *"For the most restricted users, the HNSW graph is mostly ineligible nodes — recall can collapse silently. I'd measure recall by selectivity band."*
- **Revocation lag:** *"The index can be a minute behind; the authoritative re-check on ~10 docs closes that window cheaply."*
- **Evaluating as an admin:** *"Admin evals make post-filtering look fine. Evaluate as restricted users."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **ACL as index terms, filtered during traversal** | "The principal set is bounded at a few hundred, so it fits in the query — the filter is applied *during* retrieval, which fixes top-k and the leak at once" | Filtered ANN is harder; an ACL change is an index update |
| **2** | **The model never reads what the user can't open** | "A paraphrase carries no document identity — no downstream ACL check can catch it" | The agent sometimes can't answer a question the corpus could |
| **3** | **Authoritative re-check on survivors** | "The index ACL is a performance optimisation; the permission service is the truth. Cheap on ~10 docs" | An extra dependency in the hot path |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Connected sources** + their ACLs | The corpus and its permissions | Each source models permissions differently |
| **Query and click logs** | Relevance signal | Queries are sensitive themselves ("layoffs", "complaint about my manager") |
| **Role-stratified judgements** | Relevance as seen by different access levels | Raters must have access to what they judge |
| **Synthetic test principals** | Users with narrow, broad and odd access | Needed for permission CI |
| **Seeded canary documents** | Known-narrow ACLs that must never surface elsewhere | Must be re-seeded per connector |
| **Public and synthetic enterprise corpora** | Training data for shared models | Not our customers' data |

### 5.2 Training data that respects ACLs (e) `F1c`

- **⚠ Weights have no ACL.** A model fine-tuned on restricted documents can reproduce their content to anyone who queries it. So **no shared model is trained on restricted content** — not the generator, not the reranker.
- **Within a tenant:** adaptation (synthetic queries generated from documents, tenant LoRA) uses only **broadly accessible** documents (all-staff ACL), and never crosses tenants.
- **Query logs** don't feed autocomplete or "popular searches" across ACL boundaries — a query is itself information.
- **Judgements are stratified by role and filter selectivity**, so the eval sees what restricted users see.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Model | Task | Objective |
|---|---|---|
| **BM25 + dense retriever** | Candidates, with the ACL predicate inside | Contrastive (query, relevant passage) on public and synthetic data |
| **Cross-encoder reranker** | Order ~100 → ~10 | Pairwise ranking loss on role-stratified judgements |
| **Generator** | Answer with citations | Grounded answering; trained to **cite every claim** and to **abstain** when nothing accessible answers |
| **Permission evaluation** | Allow or deny | **Not a model** — deterministic, owned by the permission service |

### 5.4 Fine-tune an existing model (f) `F2c`

LoRA-tune the embedding model and reranker on public enterprise-like corpora plus **synthetic queries an LLM generates from all-staff documents inside the tenant**. Keep tenant adapters inside the tenant boundary. Fine-tune the generator for citation and abstention behaviour on public data. Restricted content reaches the model only at **read time, after the ACL check**.

### 5.5 The staged pipeline `A2d`

```
STAGE 1 · CONNECT     per-source connectors → normalise to one principal model
                      (refuse to index what can't be represented faithfully)
STAGE 2 · INDEX       content + principal-set terms + ACL bitmaps
STAGE 3 · SEED        canary documents with narrow ACLs
STAGE 4 · TEST (CI)   same query as different users · canaries · agent leak tests
STAGE 5 · DEPLOY      per connector: shadow → internal dogfood → tenant opt-in
```

---

## 6 · SERVING — THE READ BOUNDARY, CACHING, REVOCATION *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
⚠ WHY THE SUMMARISER IS THE WORST LEAK CHANNEL:

  an ACL check works on DOCUMENT IDENTITY. A generated paraphrase
  has NO document identity — it's new text.

  So if the model reads a salary spreadsheet the user can't open and
  says "engineering compensation averages around X", you have leaked
  the contents and:
    · no output filter can catch it (it's not a quotation)
    · no citation check can catch it (there's no citation)
    · no audit can reconstruct it (the doc id never appears)
    · ⚠ AND THE USER HAS NO IDEA IT HAPPENED

→ THEREFORE THE INVARIANT IS AT THE *READ* BOUNDARY, NOT THE OUTPUT
  BOUNDARY. "Retrieve broadly, generate carefully" is NOT SAFE here.

AND IN AN AGENT LOOP IT'S WORSE: the agent retrieves several times,
  and content from turn 2 is in context at turn 7. So the ACL check
  is per-retrieval, and context carries the union of everything read.

CACHING: ⚠ THE CACHE KEY MUST INCLUDE THE USER'S ACL-GROUP HASH.
  key on (normalised query + ACL hash + filters).
  ⚠ CACHING BY QUERY ALONE LEAKS ACROSS USERS — the classic
    enterprise-RAG vulnerability, and it's a one-line bug.

PROPAGATION: an ACL revocation must reach the index in < 60 s.
  · tombstone/deny-set checked at query time → seconds
  · index rebuild → asynchronous
  ⚠ REVOCATION IS MORE URGENT THAN A GRANT. Asymmetric SLOs.
```

- **The objective** `A3b`: maximise relevance **subject to the read-boundary invariant**; when nothing accessible answers, **abstain without hinting** — a correct "I don't know" beats a helpful leak.
- **Efficiency** `F3a`: cache by **ACL-group hash**, not user id, so users with identical access share entries; the authoritative re-check runs on ~10 docs, not ~100. The trade: group-hash keys fragment the cache for users with unusual access.
- **Scale** `F3b`: partitioned indexes for large, stable permission groups; the principal-set resolver cached with its own invalidation.
- **Reliability** `I1b`: **fail closed** — if the permission service is unreachable, unverified documents are never read; the user gets results verified so far or a clear error, never an unchecked answer.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

```
EVAL — permission correctness is tested as CI, not monitored:
  · ⭐ run the SAME QUERY AS DIFFERENT USERS and assert restricted
    docs never appear — in the build, blocking
  · seeded canary documents with known-narrow ACLs that must never
    surface for anyone else
  · recall@k measured FOR RESTRICTED USERS, not just admins
    ⚠ evaluating as an admin is how post-filtering's quality failure
      stays invisible
  · agent-loop leak test: multi-turn, assert no cross-ACL content
  · recall BY SELECTIVITY BAND (FU 6)

ROLLOUT, per connector: fidelity tests on its permission model → shadow
  indexing → internal dogfood → tenant opt-in → GA. A connector that fails
  fidelity tests is NOT indexed.
```

**Success** = zero leaks in CI and in audits, restricted users' recall close to unrestricted users', revocations under 60 s at p99, and grounded, cited answers. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Admin evaluation** | Evals run with broad access hide the restricted-user failure | Role-stratified evaluation |
| **Access-level quality gap** | Junior or contract staff get worse results | Recall by selectivity band, with a parity target |
| **Popular-source bias** | One heavily used source dominates training and ranking | Per-source evaluation slices |
| **Language** | Multinational tenants; English-centric models | Per-language judgements |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | p95 ≤ 300 ms with the ACL predicate inside retrieval; the re-check on ~10 docs | p95 · $ per 1,000 queries |
| **Reliability** `I1b` | **Fail closed** without the permission service; per-connector sync monitored | Fail-closed events · connector lag |
| **Usability** `I1c` | The same quality for restricted users; **honest abstention**; an access-request flow as a separate product | Restricted-user success · abstention rate |
| **Scalability** `I1d` | Bounded principal sets; partitioned indexes for big groups; many tenants on shared infra | Recall by selectivity · tenants per cluster |
| **Privacy** `I1e` | **Weights hold no restricted content**; query logs never cross ACL boundaries; tenant isolation; data minimisation vs audit needs | Cross-tenant tests = 0 leaks |
| **Safety** `I1f` | Answers grounded in cited documents; abstain rather than guess on HR, legal or compensation questions | Ungrounded-claim rate |
| **Security** `I1h` | ⚠ Cache keys with the ACL hash; **side channels** (counts, latency, suggestions); agent credential **attenuated to the requester**; **injected text inside indexed docs** treated as data; public sharing links | Leak tests · red-team findings |
| **Ethics / bias** `I1i` | No access-level quality gap; no role-based personalisation that lets people infer what others can see | Recall parity by access level |
| **Explainability** `I1j` | Every claim cited, every citation **openable by this user**; an immutable audit log of every document read per query | % claims cited · audit coverage |

> ⭐ *"I can show you every document this system read on behalf of every user, and that each was authorised. That's the claim the design makes provable."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Agents searching on a user's behalf:** context accumulates the union of everything read, and the agent's credential must be attenuated to the requester (FU 9).
- **Five SaaS sources with five permission models:** normalise to one principal model; most-restrictive where semantics don't map (FU 8).
- **Legal hold and eDiscovery:** interact with deletion and tombstones, and often conflict with data minimisation.

**When a connector's permissions can't be trusted — the most-restrictive pipeline** `A5b` `F2a`:

```
a connector fails fidelity tests, its ACL sync lags past SLO, or a leak is suspected
  → that source's documents drop to RETRIEVAL-ONLY (links, no generation over them)
  → or index only documents whose ACL maps exactly (explicit users, all-staff)
  → suspected leak: QUARANTINE the source from generation, replay the audit
    log to scope exposure, notify per policy
  → restore through the fidelity tests and permission CI
  "Under-serving is recoverable. Over-serving isn't."
```

**Top 3 failure modes:**
1. **Summariser leak** — paraphrased content from an unseeable document; undetectable after the fact.
2. **Cache leak** — query-only cache key serves one user's results to another.
3. **Revocation lag** — someone leaves a team and their access persists in the index for an hour.

**Q1 ships:** **ACL-as-index-terms with retrieval-time filtering**, the **authoritative re-check**, and **permission tests in CI**.
> *"CI tests in Q1 because this is a security property, and security properties are regressed by ordinary refactors. A dashboard tells you after; a build failure tells you before."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why not just filter the results at the end?"** `A5a` `I1h`
> It fails in two distinct ways and the second is the serious one. **Quality**: retrieve a hundred, filter to three, and a user with narrow access gets a nearly empty page for a query the corpus could answer — and the ninety-seven documents that *should* have filled those slots were never retrieved. So the experience degrades exactly in proportion to how restricted someone is, which is backwards. **Security**: any signal derived from a document the user cannot see is a leak channel — result counts, ranking scores, "related documents," even latency differences. In an enterprise, "there are forty-seven documents matching `project falcon acquisition`" is frequently the sensitive bit, independent of the contents. So the filter has to be **inside** the retrieval: the user's transitive principal set is bounded at a few hundred entries, which makes it feasible to push into the posting-list traversal as a required disjunction and to probe an ACL bitmap at each HNSW hop.

**2. "What's the worst leak channel?"** `I1h` `A1a`
> **The summariser**, and it's worse than the others by a category. An ACL check operates on **document identity** — but a generated paraphrase has no document identity; it's new text. So if the model reads a salary spreadsheet the user can't open and answers "engineering compensation averages around X," the contents have leaked and no output filter catches it (it isn't a quotation), no citation check catches it (there's no citation), no audit reconstructs it (the document id never appears in the output), and the user has no idea it happened. That's why the invariant has to sit at the **read boundary rather than the output boundary**: *the model must never see a document the user could not open themselves.* "Retrieve broadly, generate carefully" is a reasonable instinct and it is **not safe here**. And in an agent loop it's harder still, because content retrieved at turn two is still in context at turn seven, so the check is per-retrieval and the context carries the union of everything read.

**3. "How do you handle permission changes?"** `A1b` `I1b`
> As an **index update with a hard SLO**, and with asymmetric urgency. Revocation is more urgent than a grant: a late grant is a mild annoyance, a late revocation is exposure. So the fast path is a **deny-set checked at query time** — seconds to propagate, the same tombstone pattern as deletion — with the actual index rebuild happening asynchronously. And because index ACLs are a **performance optimisation rather than the source of truth**, I keep an **authoritative re-check** against the permission service on the ~10 surviving documents before generation, which closes the propagation-lag window at negligible cost. The structural complications worth naming: **nested groups** make principal resolution a graph walk, and **deny rules** break the clean "any overlap" test — both are arguments for resolving the principal set **once per query** rather than evaluating per document.

**4. "Your cache is making this fast. What did you just break?"** `F3a` `I1h`
> Potentially everything, if the key is wrong — and this is a one-line bug with a very large blast radius. The cache key must include the **user's ACL-group hash**, not just the normalised query: `(query + ACL hash + filters)`. Caching by query alone means the first user's results are served to the second, and if their permissions differ, that's a direct leak with no retrieval involved — the classic enterprise-RAG vulnerability. Using the **group hash rather than the user id** is what preserves most of the cache benefit, since users with identical permission sets can legitimately share entries. I'd also flag the related trap: **negative caching**. Caching "no results" for a query is fine per-ACL-hash and dangerous globally, because it can suppress results for a user who *does* have access.

**5. "How do you test that permissions actually work?"** `I2a` `F1b`
> As **CI, not as a dashboard** — because this is a security property, and security properties get regressed by ordinary refactors that nobody thinks to check. Concretely: **run the same query as different users** and assert that restricted documents never appear, in the build, blocking. **Seeded canary documents** with deliberately narrow ACLs that must never surface for anyone else, which catches whole classes of indexing bug. **Multi-turn agent leak tests**, since the single-query case is the easy one. And — the one people skip — **measure recall@k for restricted users, not just for an admin**, because evaluating as an admin is precisely how post-filtering's quality failure stays invisible: your metrics look great and the experience is worst for the people with the narrowest access.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "Your ANN index has an ACL filter that matches 0.1% of documents. What happens?"** `F2b` `F1c`
> The HNSW graph **fragments**, and this is the genuine engineering difficulty behind "just pre-filter." Graph traversal works by hopping between neighbours, but if only one in a thousand nodes passes the filter, most hops land on ineligible nodes and the search either scans far more of the graph than budgeted — blowing the latency — or terminates early having found few eligible neighbours, which is a **silent recall collapse** for exactly the most restricted users. Three mitigations. **Partitioned indexes** for major permission groups, so a query against a narrow group searches a small dense index where the filter is nearly a no-op — this is the strongest answer where permission structure is stable. **Filtered-HNSW variants** that maintain eligible-neighbour lists or widen the search adaptively when the filter is selective. And **falling back to lexical retrieval** when selectivity crosses a threshold, since posting-list intersection degrades gracefully where graph traversal doesn't. The thing I'd insist on is **measuring recall by selectivity band**, because this failure is invisible in an aggregate dominated by broadly-accessible documents.

**7. "A user asks a question only an inaccessible document can answer. What should happen?"** `I1c` `I1h`
> The system should say it **doesn't know** — and it must not hint. That sounds simple and there are two subtleties. First, the honest failure is *"I couldn't find anything about that"*, not *"there are documents about this that you can't access"* — because the second is itself a leak, confirming the existence and topic of restricted material, which is frequently the sensitive part. Second, this is exactly where a system will be tempted to be helpful in a way that breaks the invariant: suggesting a related accessible document that happens to reveal the shape of what's hidden, or offering to request access with a title in the request. My position is that **abstention is the correct behaviour and its rate should be measured** — a system whose abstention rate is zero is leaking somewhere. The product answer to the underlying need is a **separate, audited access-request flow** that the search system hands off to without disclosing content — which is a deliberate product surface rather than a fallback, and it's where the genuine user value in this case lives.

**8. "Documents are connected from five SaaS systems with different permission models. Now what?"** `A2c` `F3c`
> This is the real-world version of the problem and it's mostly an **integration and fidelity** problem rather than a retrieval one. Each source has its own model — one has nested groups, one has per-item sharing links, one has deny rules, one has org-unit inheritance — and **the ACL is only as good as the connector's fidelity**. So: **normalise to a common principal model** at ingest and refuse to index anything whose permissions can't be represented faithfully, because the failure mode of a lossy mapping is a **silent over-grant**, which is the worst possible error. Model **the most restrictive interpretation** where a source's semantics don't map cleanly — under-serving is recoverable, over-serving isn't. Keep a **per-connector ACL freshness SLO**, since a connector that syncs permissions daily is a day-long revocation window regardless of what the search layer promises. And treat **public sharing links** as their own principal type — they're the most common accidental over-grant in practice, and a document shared "to anyone with the link" inside an enterprise corpus is a genuine policy question rather than a technical one.

**9. "How does this change when an agent is doing the searching instead of a person?"** `F3c` `I1h`
> It gets harder in three specific ways. **The context accumulates**: the agent retrieves several times, and everything read at turn two is still in context at turn seven, so the ACL check is per-retrieval but the exposure is the *union* — and a single mis-scoped retrieval contaminates the rest of the session. **The agent may search on the user's behalf with its own credential**, which is the confused-deputy problem: if the agent holds broader access than the requester, every query is a potential privilege escalation, so the agent's credential must be **attenuated to the requesting user's principal set**, never broader. And **the output is further from the source** — a multi-hop answer synthesised from several documents has even less traceable identity than a single summary, making the leak harder to detect after the fact. So the mechanisms: per-retrieval ACL scoping tied to the requester, **provenance carried through the context** so I can audit what was read, and **citations required for claims**, which is both a quality feature and the only practical way to make the read set reviewable.

**10. "Legal asks you to prove no leak has ever occurred. Can you?"** `I1j` `I2b`
> Not in the strong sense, and I'd say so plainly rather than overclaim to a party who will rely on it. What I **can** prove is structural: an **immutable audit log** of every document id read in service of every query, with the requesting principal — so for any query I can reconstruct exactly what the model saw and demonstrate that every document was ACL-verified at read time. That's a genuine and useful claim, and it's why the audit log is a day-one requirement rather than an operational nicety. What I **cannot** prove is the absence of a leak through a channel nobody enumerated — a latency side channel, a result-count inference, or a summariser path that predates the invariant. So the honest framing: *"I can show you every document this system read on behalf of every user, and that each was authorised. I can't prove that no information was ever inferable through a channel we didn't anticipate — which is why the invariant sits at the read boundary, so the set of channels to worry about is as small as possible."* That's a defensible position; "no leak is possible" is not, and claiming it is how you lose credibility in the conversation that matters.

---

## ⭐ THE L6 LINE

> **"Retrieval quality and access control are the same system, and access control wins. Post-filtering fails twice — it destroys top-k for exactly the most restricted users, and every derived signal is a leak channel. The summariser is the worst of them, so the invariant is at the read boundary: the model must never see a document the user couldn't open."**

## 🧭 HOOKS — *steer by what you mention*

- *"The invariant is at the read boundary"* → invites the security-architecture branch (**good at L6**)
- *"A paraphrase carries no document identity"* → invites the generation/leak branch
- *"The principal set is bounded at a few hundred"* → invites the systems/implementation branch
- *"Recall measured as an admin hides the failure"* → invites the evaluation branch (**good at L6**)
- *"Filtered ANN fragments the graph"* → invites the ANN-internals branch
- *"The ACL is only as good as the connector's fidelity"* → invites the integration branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the two failures of post-filtering"* — quality + leak regenerates the design.
- **Running out of time?** → *"I have five minutes — I'd rather cover the summariser leak and CI testing than finish the ranking. Sound right?"*

**Name and defer:**
- "**Everything in [Q6](Q6_rag_retrieval_intv.md)** applies — chunking, hybrid, reranking, the attribution split — this sheet is the delta."
- "**Connector engineering** is most of the real work in an enterprise deployment and it's unglamorous."
- "**Access-request flows** are the product answer to the abstention case."
- "**Retention and legal hold** interact with the deletion path in ways that complicate tombstoning."
- "**Audit and eDiscovery** requirements often mandate logging that conflicts with data-minimisation."
- "**Personalisation by role** is valuable and must never become a channel for inferring what others can see."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: ACL inside retrieval | §0, FU 2 | *"ACL filtering happens before ranking, never after."* |
| **A1b** | Latency and capacity budgets | §1, §3, FU 3 | *"The principal set is bounded at a few hundred — that's what makes it fit in the query."* |
| **A1c** | Degrade without lowering quality | §6 | *"Fail closed — unverified documents are never read."* |
| **A2a** | The full stack | §4 diagram | Resolve principals → filtered retrieval → re-check → rerank → grounded generation |
| **A2b** | Model components | §5.3 | BM25 + dense, cross-encoder, citing and abstaining generator, rule-based permissions |
| **A2c** | Data sources | §5.1, FU 8 | Sources + ACLs, logs, role-stratified judgements, test principals, canaries |
| **A2d** | Staged pipeline | §5.5 | Connect → index → seed → test in CI → deploy per connector |
| **A3a** | Top-down, unprompted | §0 | *"Retrieval quality and access control are the same system"* + the plan |
| **A3b** | A principled objective for the trade-off | §6 | *"Maximise relevance subject to the read-boundary invariant; abstain without hinting."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Permission model · revocation SLO · answer or retrieve · sources |
| **A5a** | Alternatives compared | §4, FU 1 | Post-filter vs inside vs partitioned; output filter vs read boundary; cache keys |
| **A5b** | A full alternative pipeline on demand | §9 | The most-restrictive pipeline |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Pairwise reranking on role-stratified judgements; a generator trained to cite and abstain."* |
| **F1b** | Metrics and their trade-offs | §2, FU 5 | *"Recall measured as an admin hides the failure."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 6 | Admin evaluation, access-level gap, selectivity bands |
| **F2a** | Something fails → a fallback | §9 | The most-restrictive pipeline |
| **F2b** | Latency too tight → design for it | §3, §4, FU 6 | Fragmented ANN → partitions and lexical fallback, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | LoRA on public data + synthetic queries from all-staff docs, inside the tenant |
| **F3a** | Efficiency with a quality trade | §6, FU 4 | ACL-group-hash cache keys; re-check on ~10 docs |
| **F3b** | Scale levers | §6 | Partitioned indexes; cached principal resolution |
| **F3c** | Extension, with cost and maturity risk | §9, FU 8, 9 | Agents; multi-source connectors; legal hold |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8 | p95 with the ACL predicate inside |
| **I1b** | Reliability | §6, §8, FU 3 | Fail closed; revocation under 60 s |
| **I1c** | Usability | §2, §8, FU 7 | Equal quality for restricted users; honest abstention |
| **I1d** | Scalability | §3, §8 | Bounded principal sets; partitions; multi-tenant |
| **I1e** | Privacy | §5.2, §8 | **Weights have no ACL**; query logs; tenant isolation |
| **I1f** | Safety | §8 | Grounded answers; abstain on HR and legal questions |
| **I1h** | Security | §6, §8, FU 1, 2, 4, 9 | Summariser; cache keys; side channels; attenuated agent credentials |
| **I1i** | Ethics / bias | §7, §8 | No access-level quality gap; no inference through personalisation |
| **I1j** | Explainability | §2, §8, FU 10 | Openable citations; the audit log of everything read |
| **I2a** | Offline → online | §7 | Permission CI → shadow → dogfood → opt-in, per connector |
| **I2b** | The success definition | §7, FU 10 | *"Zero leaks, restricted-user recall near unrestricted, revocation under 60 s."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) retrieval → §4 · (d) data → §5.1 · (e) ACL-safe training data → §5.2 · (f) models → §5.3–5.4 · (g) serving → §6 · (h) success → §2, §7 · (i) extension → §9.
