# Q6 · RAG / Retrieval for an Agent — **MY NOTES**
### The points I had doubts on, answered

> Companion to [`Q6_rag_retrieval_intv.md`](Q6_rag_retrieval_intv.md). Skim on a revision pass; go back to the sheet for the full answer.
>
> **⚠ Three of these were corrections to the sheet, not just explanations:** §6+7 (it ran HNSW and PQ together as if they were alternatives — **a genuine error**), §2 (ACLs are group-based, which is *why* the design works), §5 (the four slices were listed without saying what each catches). All three are now fixed in the sheet.

**Jump:** [1 long context](#1--why-is-long-context-simpler-and-cheaper-for-a-small-stable-corpus) · [2 groups not users](#2--isnt-it-group-permissions-not-per-user) · [3 freshness](#3--why-does-freshness-change-the-architecture) · [4 answer accuracy](#4--shouldnt-answer-accuracy-be-a-metric) · [5 the four slices](#5--the-four-special-slices--what-each-one-catches) · [6+7 HNSW vs PQ](#67--hnsw-or-pq-and-why-500-hops--the-sheet-was-wrong) · [8 ACL re-check + reranker](#8--why-acl-again--and-could-the-reranker-be-a-dnn) · [9 connectors, queue, DLQ](#9--connectors-durable-queue-dlq) · [10 filtered ANN](#10--filtered-ann-and-graph-fragmentation) · [11 tombstones](#11--what-is-a-tombstone) · [12 golden set in prod](#12--why-run-the-golden-set-continuously-against-production) · [13 measuring the panels](#13--how-do-you-actually-measure-those-panels) · [14 silent recall collapse](#14--silent-recall-collapse--explain) · [15 the cache leak](#15--the-cache-leak-explained) · [16 hallucination SLO](#16---hallucination-was-missing-from-the-nfrs--i-added-it)

---

## 1 · Why is long context simpler *and* cheaper for a small stable corpus?

| | **RAG** | **Long context** |
|---|---|---|
| Build | Chunker, embedder, vector index, hybrid retriever, reranker, fusion | **Put the documents in the prompt** |
| Keep running | Re-embed on change · index freshness · DLQ · parser failures · embedding-version migrations | Nothing |
| Failure modes | Retrieval misses · bad chunk boundaries · stale index · ACL propagation | The model misreads something |
| Accuracy | ⚠ Capped by **recall@k** — a miss is unrecoverable | ⭐ **No retrieval step to miss** |

**Cheaper needs the arithmetic**, because it's counterintuitive. 200 docs × 2k tokens = 400k tokens. Expensive per call — **but with prefix caching that corpus is a stable prefix**: prefill once, then pay a heavily discounted cached rate (often ~10% of input cost). Against RAG's embedding pipeline + index hosting + reranker inference + the engineering time to keep it correct, long context frequently wins on **total** cost at this size.

> ⭐ **And the accuracy point is stronger than the cost point:** RAG's ceiling is recall@k. Long context has **no retrieval step**, so that ceiling doesn't exist.

**RAG earns its complexity on three axes** — scale (can't put 50M docs in any window) · churn (re-prefilling an hourly-changing corpus is absurd) · ⭐ **permissions** (you cannot put content in context the user isn't allowed to see, and **long context has no equivalent of a pre-filter**). Ours has all three. **The point is to say that rather than assume it.**

---

## 2 · Isn't it group permissions, not per-user?

**Yes — and the sheet's wording is loose.** In practice ACLs are **principal sets dominated by groups**:

```
user's principals = {u:bob} ∪ {g:eng, g:eng-leads} ∪ {g:all-staff} ∪ {domain:corp}
```

> ⭐ **And that's exactly why the design works.** Per-*user* ACLs would mean a per-user predicate over millions of users. **Group-based means the principal set is bounded at a few hundred entries** — small enough to push into the query as a required disjunction and to probe as a bitmap at each HNSW hop.

If permissions really were per-user and unbounded, pre-filtering would be infeasible and you'd need **partitioned indexes** instead. So *"user permissions"* in the clarifying question means **per-user *effective* permissions, resolved from groups**.

⚠ **Both the complications are group-shaped too:** **nested groups** make resolution a graph walk (hence resolve *once* per query, cached), and **deny rules** break the clean "any overlap" test and force real evaluation. *(G6's sheet gets this right — "transitive principal set"; Q6's is sloppier.)*

---

## 3 · Why does freshness change the architecture?

Three regimes, genuinely different systems:

| Freshness | Architecture |
|---|---|
| **Days** | **Batch rebuild.** Nightly job re-embeds, rebuilds, atomic swap. No incremental anything |
| **Minutes** | **Incremental + streaming.** Change feed per connector, per-doc embed, upsert into a live index — ⚠ which HNSW handles awkwardly (insertions degrade graph quality, so you need periodic rebuilds *anyway*) |
| **Seconds** | ⭐ **A separate fresh-path index** — a small recent-docs index queried alongside the main one and merged, because you cannot maintain 200M vectors at second-level freshness |

**What actually changes:** a nightly rebuild needs no change-detection, no incremental indexing, no insertion-degradation handling, no DLQ for a streaming pipeline. **Each step down the ladder adds a subsystem.** That's why it's a minute-one question — the answer decides whether you're building **one index or three**.

---

## 4 · Shouldn't answer accuracy be a metric?

**Yes, and it's deliberately demoted.** The sheet has it as *"then groundedness, then end-to-end"* — behind recall and the attribution split.

> ⚠ **End-to-end accuracy is the metric that hides which half is broken.** A 70% score could be retrieval missing 30% of the time, or the generator ignoring what it was given — **different teams, different fixes, different weeks of work.**

The eval set has all three, in this order:
1. **Recall@100** — the ceiling. Fix first; nothing downstream can exceed it.
2. **Groundedness** — is every claim supported by a retrieved span? Catches hallucination-over-correct-context.
3. **End-to-end accuracy** — the user-facing number, reported **alongside the attribution split** so it's actionable.

> ⭐ **Say it as:** *"End-to-end accuracy is what the user experiences and the worst thing to debug with. I'd report it, and gate on recall plus groundedness, because those tell me **where** to work."*

---

## 5 · The four special slices — what each one catches

Each catches a failure the main eval set **structurally cannot**:

**① Unanswerable** — questions whose answers genuinely **aren't in the corpus**. Correct behaviour: *"I couldn't find anything on that."*
> ⭐ **A system scoring 0% here is hallucinating on every one** — fabricating from irrelevant chunks. Cheapest hallucination detector available, and almost nobody builds it. **An abstention rate of zero means the system is lying somewhere.**

**② Permission, as different users, in CI** — the *same* query as Alice and as Bob, asserting restricted docs never surface for Bob. **In the build, blocking.**
> ⚠ It's a **security property**, and security properties get regressed by **ordinary refactors nobody thinks to re-test**. A dashboard tells you after; a build failure tells you before.

**③ Freshness assertions** — insert a doc, assert retrievable within the SLO; delete one, assert **gone** within 60 s. Tests the **pipeline**, not the model — **the only thing that catches a silently broken connector**.

**④ Multi-hop** — questions needing ≥2 retrievals (*"compare our refund policy with the enterprise SLA"*). Single-shot retrieval finds **the average of two things and neither**. Measures whether the **agent loop** works, which per-query nDCG cannot see.

> **The pattern: each slice tests a property the aggregate metric averages away.**

---

## 6+7 · HNSW **or** PQ? And why 500 hops? — ⚠ **the sheet was wrong**

### ⚠ The genuine error I found

The sheet said *"vectors + graph must be resident"* **and** *"PQ"* as if they described one design. They don't — **but they're orthogonal, and the standard production answer uses both**:

```
HNSW-WITH-PQ (the usual combination):
   GRAPH (adjacency lists)   → RAM   ← ⚠ PQ does NOT remove this
   PQ-COMPRESSED vectors     → RAM   ← fast approximate distances
                                       DURING the walk
   full-precision vectors    → optional, on disk, to re-score survivors
```

> ⭐ **PQ compresses what is stored at each node, not the index structure.** The walk is still a dependent graph traversal, so the latency argument holds **unchanged**.

**The four designs, to be precise:**

| Design | Structure | Vectors |
|---|---|---|
| **HNSW, int8** | Graph in RAM | int8 in RAM |
| **HNSW + PQ** ← the sheet's intent | Graph in RAM | PQ codes in RAM |
| ⭐ **IVF-PQ** | **Inverted lists — NO graph** | PQ codes in RAM |
| **DiskANN** | Graph **on disk** | PQ in RAM, full vectors on disk, **batched reads** |

**IVF-PQ is the graph-less one** — it clusters and scans selected lists, which is why G1's ladder names it for the disk-resident case.

### On `m = 96` vs `64` — you were right that it's a design choice

PQ at `m` subquantizers with 8-bit codes = **m bytes per vector**. Both numbers in the pack are internally consistent; they just chose different `m`:

```
Q6: 200M chunks, m=96  → 200M × 96 B = 19.2 GB   ✓
G1: 10⁹ images, ~65 GB → 10⁹ × 65 B ⇒ m ≈ 64      ✓
```

Typical: 32 / 48 / 64 / 96 — trading compression against recall, often with `m | d` (768/96 = 8 dims per subvector). **Sheet now says "m is a tuning knob" explicitly.**

### On "~500 hops"

**Order of magnitude**, from HNSW's parameters: **M ≈ 16–32** neighbours/node, **efSearch ≈ 100–200** ⇒ nodes *visited* in the high hundreds to low thousands. Raise efSearch for recall and it rises. **Not a precise count** — and ⭐ **the argument survives at 200 or at 2000, because the real point is *serial***:

> ⚠ A dependent chain runs at **queue depth 1**, so you get `1/latency ≈ 10k reads/s` — **1–2% of an NVMe's IOPS**. You're bounded by **latency, never throughput**. The SSD's single greatest advantage is unusable.

### And the "chunk text on disk" point is separate

The **index** holds vectors + graph. The **document text** sits in a blob store, fetched only for the ~100 survivors in **one batched read**. ⭐ **That's why 614 GB is vectors-only** — the part people conflate when the memory number sounds impossible.

---

## 8 · Why ACL again? And could the reranker be a DNN?

### The re-check exists because the index's ACL copy is a **cache**

Pre-filtering uses ACLs **indexed as terms** — denormalised at ingest, so they lag the permission service by the propagation SLO:

```
10:00:00  Bob removed from g:eng
10:00:00  permission service: Bob ∉ g:eng     ← truth, immediately
10:00:00  index still has g:eng on 40k docs   ← cache, STALE
10:00:45  re-index completes
```

So: **pre-filter for correctness and recall** (inaccessible docs are never candidates → protects top-k) · **re-check for safety** (closes the lag). Cheap — ~100 docs against a service, not millions. ⭐ **And the two layers fail for different reasons**, which is what makes it defence in depth rather than redundancy.

### The reranker could be a DNN — the question is *where the interaction happens*

| | Mechanism | Cost |
|---|---|---|
| **Cross-encoder transformer** | Query and doc **tokens attend to each other** | ~120 ms/100 — the accuracy ceiling |
| **DNN over features** | Query/doc features **concatenated** (like G13's ranker) | Far cheaper, much weaker on text semantics |
| **Late interaction (ColBERT)** | Token embeddings precomputed, cheap similarity at query time | Middle ground; bigger index |

For *text relevance*, the cross-encoder's token-level interaction is what earns the 120 ms — a feature DNN can't model *"does this passage answer this question."* ⭐ **But the sheet's §6 cascade is exactly where a cheap DNN belongs**: 200 → cheap reranker → 30 → strong reranker → 10.

---

## 9 · Connectors, durable queue, DLQ

**Connectors** — per-source integrations pulling content **and permissions**: Drive, Confluence, Slack, SharePoint, Jira. Each needs auth, pagination, **change detection**, **ACL extraction**, format handling.
> ⭐ *In an enterprise deployment this is usually **most of the actual engineering**, and it's unglamorous.*

**Durable queue** — a broker (Kafka / Pub/Sub / SQS) holding one message per document-to-index.
- **Durable** = a message survives a worker crash. Not acked until the work completes ⇒ a pod dying mid-embed causes a **redelivery**, not a silently missing document.
- **Decouples** connectors (fast, bursty — a backfill enqueues 50M docs) from processing (slow, expensive — parse, embed, index).

**DLQ (dead-letter queue)** — where a message goes after N failed attempts, instead of retrying forever or being dropped.
> ⚠ Real corpora are full of password-protected PDFs, corrupt files, 500 MB spreadsheets — **"parsing fails constantly" is the normal state, not an exception.** The DLQ makes failures **countable and inspectable**: it converts a **silent coverage gap** into a monitored number. And a chunk never indexed is exactly the **unrecoverable first-stage recall miss**.

---

## 10 · Filtered ANN and graph fragmentation

**Yes — exactly that.** The graph was built for **unfiltered** traversal: hop to your nearest neighbours. With a predicate, many neighbours are ineligible:

```
Unfiltered:            every hop lands on a usable node → ~500 hops, done
Filtered (0.1% pass):  ~999 of 1000 neighbours INELIGIBLE
   → either scan far more of the graph        (blows latency)
   → or terminate early with few eligible     (⚠ SILENT RECALL COLLAPSE)
```

⚠ **And it's worst for the most restricted users** — precisely the ones the design exists to protect.

**Mitigations (from G6):** **partitioned indexes** per major permission group — a narrow group queries a small dense index where the filter is nearly a no-op (strongest where permission structure is stable) · **filtered-HNSW** variants maintaining eligible-neighbour lists or widening adaptively · **falling back to lexical** past a selectivity threshold, since posting-list intersection **degrades gracefully where graph traversal doesn't**.

> ⭐ **And measure recall by selectivity band** — this failure is invisible in an aggregate dominated by broadly-accessible documents.

---

## 11 · What is a tombstone?

**Document (or chunk) deletion — and your instinct is right: a set with O(1) membership.** In practice a **compressed bitmap (Roaring)** or a hash set of doc IDs, sometimes Bloom-backed when large.

**⚠ Why defer the real delete:** removing a node from an HNSW graph is genuinely hard. That node was a **hop in other nodes' neighbour lists**, so pulling it out means **repairing those edges** — and doing it live risks **fragmenting the graph**, degrading recall for queries that needed to route through that region.

**So two phases:**
1. **Tombstone immediately** — write the ID into the deny set; filter every result against it. Microseconds to write, **seconds to propagate.** The doc stops being **retrievable**.
2. **Compact asynchronously** — during segment merges / index rebuilds, physically drop it and rebuild cleanly.

**⚠ Two things worth knowing:**
- **Tombstones accumulate.** Deleted-but-not-compacted nodes still occupy graph slots, so you return **fewer live results than you asked for** and must over-retrieve. At ~30% tombstoned, recall *and* latency degrade ⇒ **compaction cadence is a real operational knob**.
- **For legal erasure a tombstone isn't enough** — the bytes are still there. Compaction must be **deadline-bound and verifiable**.

---

## 12 · Why run the golden set continuously against production?

> ⭐ **Because retrieval quality degrades via changes nobody associates with search.**

```
Tuesday:  the docs team upgrades a PDF parser
          → extraction subtly breaks on tables
          → table content vanishes from chunks
          → every table-related query silently fails

⚠ Nobody ran a search test. SEARCH WASN'T DEPLOYED.
```

Other versions: a connector's OAuth token expires and that source stops syncing · an upstream schema change drops a field the chunker depended on · a permission-system migration changes group semantics.

**Release-time testing cannot catch these, because search didn't release.** So: a few hundred gold-chunk queries, **every hour, against production**, with recall drops as an **alert** rather than a next-quarter discovery. ⭐ And that's also why **DLQ rate, parse-failure-by-doc-type, and zero-result rate** are on the panel — **they move before the quality metric does.**

---

## 13 · How do you actually measure those panels?

**ACL rejection rate** — at query time you already know how many candidates the predicate excluded and how many the re-check dropped. Emit `acl_prefiltered_total` and `acl_recheck_dropped_total` as a fraction of candidates.
> ⚠ A spike means either a permission change landed **or the filter is over-filtering** — and the **re-check counter going non-zero** tells you propagation lag is biting.

**Index freshness lag** — timestamp both ends: each doc carries `source_modified_at` and `indexed_at`; report **p50/p95/max** of the difference over recently-indexed docs.
> ⭐ Plus a **synthetic canary**: inject a doc every minute with a known ID, poll until retrievable, record the time. That measures **end-to-end** freshness *including the queue*, which a timestamp diff misses when the queue is backed up.

**DLQ rate** — the broker reports depth natively. Track **messages entering the DLQ per minute, broken down by failure reason and document type** — which is what turns *"DLQ is up"* into *"every .docx since Tuesday is failing."*
> ⚠ **Alert on the derivative, not the absolute** — the rate is never zero.

---

## 14 · "Silent recall collapse" — explain

```
A parser library upgrade changes .docx handling
   → extraction returns EMPTY for .docx
   → "successful" ingestion of 0-length text
   → embedded (a vector for empty text!) and indexed
   → ⚠ THE PIPELINE REPORTS SUCCESS. DLQ IS EMPTY. NO ERROR ANYWHERE.
   → every .docx is now unretrievable
```

⚠ **Why nothing fires: the failure isn't an exception, it's an *absence*.** The documents are "indexed." Queries just never match them — **and a query returning irrelevant results looks exactly like a hard query.**

**What catches it:**
- ⭐ **Golden set against production** (§12) — if gold chunks live in .docx, recall drops immediately
- **Per-document-type coverage** — count indexed docs by extension; .docx going flat is the signal
- ⭐ **Zero-length-extraction checks** — **refuse to index empty text** rather than embedding it. Cheap, and it **converts silence into a DLQ entry**
- **Zero-result-rate monitoring** — an underrated leading indicator that usually moves first

> **The general lesson, shared with Q8:** *a data bug doesn't crash — it silently produces a worse system.*

---

## 15 · The cache leak, explained

```
cache_key = hash(normalized_query)              ← ⚠ BROKEN

Alice (admin) searches "Q3 compensation plan"
   → results cached, INCLUDING the confidential doc
Bob (intern) searches the same string
   → CACHE HIT → Bob gets Alice's results
   → ⚠ A LEAK WITH NO RETRIEVAL INVOLVED.
     The ACL layer was NEVER CONSULTED.
```

> ⚠ **Every careful thing in the retrieval path — the pre-filter, the bitmap probe, the authoritative re-check — is bypassed entirely by a cache hit.** One line of code, total compromise.

**The fix:**

```
cache_key = hash(normalized_query + ACL_GROUP_HASH + filters)
```

⭐ **Group hash rather than user ID** — that's what preserves most of the cache benefit, since users with *identical* permission sets can legitimately share entries, and in an enterprise most people do.

**⚠ Two related traps:**
- **Negative caching** — caching "no results" is fine **per ACL hash** and dangerous globally: it suppresses results for a user who *does* have access.
- **Invalidation on permission change** — Bob leaves a team and cached entries under his old group hash are now wrong. Either include a **permission-version epoch** in the key, or keep TTLs inside your revocation SLO.

---

## 16 · ⚠ Hallucination was missing from the NFRs — I added it

**The gap I found:** the NFR table had latency, recall, freshness and deletion — **nothing about whether the answer is true.** For a RAG system that is the omission that matters most.

**But "hallucination rate" is not directly measurable at scale** — it needs a human to judge truth on every answer. So the SLO has to be phrased as something computable:

| New SLO | Target | Why this phrasing |
|---|---|---|
| ⭐ **Groundedness** | **≥ 98% of claims traceable to a retrieved span** | **Automatically checkable** — for each claim, is there a supporting span? |
| **Abstention rate on unanswerable** | **> 0, and tracked** | A system that never declines is **fabricating** on what it cannot answer |

> ⭐ **And groundedness decomposes the failure correctly, which is the real reason to prefer it:**
> - **Ungrounded claim** ⇒ **the generator's fault** — it asserted something no retrieved span supports.
> - **Grounded but wrong claim** ⇒ **the corpus is wrong** — a completely different bug, in a different team's area.
>
> *That's the same attribution logic as §4's retrieval-vs-generation split, applied one level down to the claims inside an answer.*

**The two together cover both halves of hallucination:**
- **Groundedness** catches *fabrication on answerable questions* — making things up when the evidence was right there.
- **Abstention** catches *fabrication on unanswerable questions* — ⚠ which groundedness alone **cannot** see, because a confident answer built from loosely-related retrieved chunks can be technically "grounded" and still wrong.

> ⚠ **Zero abstention is a red flag, not a win.** It's the §5① slice turned into a production SLO.

---

## 🔁 THE NINE THINGS I KEPT GETTING WRONG

1. ⭐ **HNSW and PQ are orthogonal, not alternatives** — the graph stays in RAM; PQ compresses what's *at each node*. **IVF-PQ** is the graph-less one.
2. **ACLs are group-based, and that's *why* pre-filtering works** — a bounded principal set fits in the query.
3. **614 GB is vectors-only.** Chunk text lives in a blob store, fetched for the ~100 survivors.
4. **The ACL re-check isn't redundant** — the index's copy is a cache that lags the permission service.
5. **The cache key must include the ACL group hash** — otherwise a hit bypasses every permission control you built.
6. **"Silent" means absence, not error.** The pipeline reports success; the docs just aren't findable.
7. **End-to-end accuracy is the worst thing to debug with** — gate on recall + groundedness, which say *where* to work.
8. **Long context isn't just simpler, it has no recall ceiling** — RAG earns its place on scale, churn, and permissions.
9. ⭐ **The NFRs had no truth metric.** Phrase it as **groundedness** (automatically checkable) plus a **non-zero abstention rate** — because "hallucination rate" needs a human on every answer.
