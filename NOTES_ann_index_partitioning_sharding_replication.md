# NOTES · Partitioning, Sharding, Replication and Load Balancing for an ANN Index

> **What this covers:** how a large approximate-nearest-neighbour (ANN) index is split across machines and served under load. It uses one running example: **R2's moment index** (video moments, split by vertical). The same ideas apply to G13's video index, R8's chunk index, and any embedding retrieval at scale.
>
> **The one-liner to carry into an interview:** *"Partition to cut what a query searches, shard to make each partition fit and balance its data, replicate to absorb peak load and survive failures, and load-balance across replicas to keep the tail short."*

---

## 1 · Why one machine isn't enough — *three different problems*

```
                    ┌─────────────────────────────────────────┐
  all queries ────► │  ONE INDEX SERVER                       │
                    │  3–6×10⁹ moment vectors ≈ 200–400 GB    │
                    └─────────────────────────────────────────┘

  PROBLEM 1 · SIZE        400 GB doesn't fit in one machine's RAM
  PROBLEM 2 · LOAD        peak QPS exceeds what one machine can serve
  PROBLEM 3 · FAILURE     the machine dies → search is down for everyone

  AND A FOURTH, SPECIFIC TO R2:
  PROBLEM 4 · MIXED SPACES  dance and skate vectors come from DIFFERENT encoder
                            adapters, so they aren't comparable; a dance query
                            should never even look at skate vectors
```

**Each mechanism solves a different one:**

| Mechanism | Solves | In one line |
|---|---|---|
| **Partitioning** | 4 (and helps 1, 2) | Split by **meaning**: which index a query searches |
| **Sharding** | 1 | Split a partition into **pieces that fit one machine** |
| **Replication** | 2, 3 | **Copies** of each shard, to serve more queries and survive failures |
| **Load balancing** | 2 (the tail) | **Which copy** answers each request |

> ⚠ **Vocabulary:** many teams say "partition" and "shard" interchangeably. In these notes, **partition** is the *logical* split (by vertical, tenant or language) and **shard** is the *physical* piece of one partition on one machine. Say which you mean in the interview.

---

## 2 · The vocabulary

| Term | Meaning | In R2 |
|---|---|---|
| **Partition** | A logically separate index, chosen by a property of the data | One per vertical: dance, skate, gymnastics… |
| **Shard** | One physical slice of a partition, held by one machine | Dance has 4 shards of ~28 GB each |
| **Replica** | An identical copy of one shard | Each dance shard has 3 copies |
| **Router** | Decides which partition a query goes to | A vertical classifier on the query clip |
| **Load balancer** | Picks one replica of each shard for each request | Least-loaded healthy replica |
| **Scatter-gather** | Send the query to many shards ("scatter"), merge their answers ("gather") | Query all 4 dance shards, merge into the top 1,000 |
| **Fan-out** | How many shards one query touches | 4 for dance, 1 for skate |

---

## 3 · PARTITIONING — *split by meaning*

```
                         ┌────────────┐
       query clip ─────► │   ROUTER   │  "this is a dance clip"
                         └─────┬──────┘
             ┌─────────────────┼──────────────────┐
             ▼                 ▼                  ▼
   ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
   │ DANCE partition  │ │ SKATE        │ │ GYMNASTICS       │
   │ 1.5×10⁹ moments  │ │ 3×10⁸        │ │ 5×10⁸            │
   │ ~110 GB          │ │ ~22 GB       │ │ ~36 GB           │
   │ dance adapter v7 │ │ skate v3     │ │ gym v5           │
   └──────────────────┘ └──────────────┘ └──────────────────┘
        ▲ searched            ✗ never touched by a dance query
```

**Common ways to partition:**

| Partition by | Why | Example |
|---|---|---|
| **Domain / vertical** | Different encoders, so different embedding spaces; smaller search space | R2: dance vs skate |
| **Tenant / customer** | Isolation: one customer's data never mixes with another's | A SaaS search product |
| **Access domain** | Restricted collections physically separate | R8: HR and legal docs in their own index |
| **Language / region** | Different models; data residency laws | EU data stays in EU |
| **Freshness tier** | A small, fast-updating index plus a big daily one | R6, G13: the fresh tier |

**What it buys:** a query searches less, so it's faster and more precise. Each partition can use its own model version, and scale on its own. A problem in one doesn't touch the others.

**What it costs:**
- **Routing can be wrong.** A clip classified as gymnastics that's really breaking misses its results. Mitigation: if the router is unsure, search the top 2 partitions and merge (that's why scores need calibrating across partitions).
- **Hot partitions.** If dance gets 70% of traffic, its partition needs far more replicas than the rest (§5).

---

## 4 · SHARDING — *split one partition so it fits*

The dance partition is ~110 GB. With ~32 GB of usable RAM per index machine, it needs **4 shards**. The question is **which moment goes to which shard**. There are two strategies, with opposite trade-offs.

### 4.1 Hash (random) sharding

```
  new moment m ──► shard = hash(moment_id) mod 4
                         │
      ┌──────────────────┼──────────────────┬──────────────────┐
      ▼                  ▼                  ▼                  ▼
 ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
 │ SHARD 0  │      │ SHARD 1  │      │ SHARD 2  │      │ SHARD 3  │
 │ ~28 GB   │      │ ~28 GB   │      │ ~28 GB   │      │ ~28 GB   │
 │ a RANDOM │      │ a RANDOM │      │ a RANDOM │      │ a RANDOM │
 │ quarter  │      │ quarter  │      │ quarter  │      │ quarter  │
 │ of dance │      │ of dance │      │ of dance │      │ of dance │
 └──────────┘      └──────────┘      └──────────┘      └──────────┘
  each shard has its OWN ANN index (HNSW or IVF-PQ) over its quarter

  THE QUERY: neighbours could be in ANY shard → ask ALL of them

           query q
              │ SCATTER (in parallel)
   ┌──────────┼──────────┬──────────┐
   ▼          ▼          ▼          ▼
 shard 0   shard 1    shard 2    shard 3
 top-1000  top-1000   top-1000   top-1000   ← each returns its LOCAL top-K
   └──────────┴────┬─────┴──────────┘
                   ▼ GATHER: merge 4,000 by score → global top-1,000
```

- **Data is perfectly balanced:** hashing spreads moments evenly, with no hot cluster of popular or similar items on one machine.
- **Load is perfectly balanced:** every shard receives every query.
- **The cost is fan-out:** every query touches every shard, and **the query is as slow as its slowest shard** (§7).
- ⚠ **Each shard must return the full K, not K/4.** The true global top-1,000 might all sit on one shard; asking each for 250 could miss most of them.
- **Why the merge is correct:** each shard's local top-K contains every global top-K item that lives on that shard, so the union of the four local lists contains the global top-K. That's exact if each shard's search is exact; with ANN, recall is each shard's ANN recall.

### 4.2 Semantic (cluster) sharding

```
  BUILD: k-means the dance vectors into 4 regions of embedding space
         (in practice: many fine clusters, grouped onto 4 machines)
 ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 │ SHARD A      │  │ SHARD B      │  │ SHARD C      │  │ SHARD D      │
 │ power moves  │  │ footwork     │  │ freezes      │  │ toprock      │
 │ (centroid A) │  │ (centroid B) │  │ (centroid C) │  │ (centroid D) │
 └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

  THE QUERY: compare q to the 4 centroids → search only the NEAREST 1–2 shards
           query q (an airflare)
              │ nearest centroids: A (power moves), then C
              ├──────────────► shard A ✓
              └──────────────► shard C ✓      shards B, D: not touched
```

- **Less fan-out:** a query touches 1–2 shards, not 4, which means less total work at high QPS.
- **Hot shards:** if everyone searches airflares this week, shard A takes most of the load and needs more replicas than B, C or D.
- **Boundary misses:** a true neighbour just over the line in shard B is never found. Mitigation: probe more shards when the query sits near a boundary.
- **Uneven data:** clusters aren't equal-sized; rebalancing means re-clustering.

### 4.3 Which to use

| | **Hash sharding** | **Semantic sharding** |
|---|---|---|
| Data balance | ✅ Even by construction | ⚠ Clusters are uneven |
| Load balance | ✅ Every shard sees every query | ⚠ Trending topics make hot shards |
| Fan-out per query | ⚠ All shards | ✅ 1–2 shards |
| Recall | ✅ No boundary loss | ⚠ Neighbours across a boundary missed |
| Adding a shard | Re-hash (consistent hashing moves ~1/N of items) | Re-cluster |
| **Default when** | **A few to tens of shards** per partition (R2, G13, R8) | **Very many shards**, where fan-out to all becomes the bottleneck |

> **Say:** *"Within a partition I'd hash-shard: balanced data, balanced load, no recall lost at boundaries. The price is fan-out, which is fine at a handful of shards. Semantic sharding only wins when there are so many shards that asking all of them is the bottleneck."*

---

## 5 · REPLICATION — *copies for load and failure*

```
                        DANCE PARTITION
   ┌─────────────┬─────────────┬─────────────┬─────────────┐
   │  SHARD 0    │  SHARD 1    │  SHARD 2    │  SHARD 3    │
   ├─────────────┼─────────────┼─────────────┼─────────────┤
   │ replica 0a  │ replica 1a  │ replica 2a  │ replica 3a  │  zone 1
   │ replica 0b  │ replica 1b  │ replica 2b  │ replica 3b  │  zone 2
   │ replica 0c  │ replica 1c  │ replica 2c  │ replica 3c  │  zone 3
   └─────────────┴─────────────┴─────────────┴─────────────┘
     4 shards × 3 replicas = 12 machines
     every replica in a column is IDENTICAL; replicas are spread across zones
```

**What replication buys:**
- **Read throughput:** three replicas of a shard serve ~3× the queries.
- **Failover:** a replica dies, and the other two keep serving.
- **Zone resilience:** a whole datacenter zone goes down, and two copies of every shard survive.

**What it costs:**
- **Machines:** 3× the memory bill.
- **Write fan-out:** every new vector is written to all replicas of its shard.
- **Consistency:** replicas must serve the same index version; otherwise two identical queries get different results depending on which replica answered.

**Two ways to keep replicas consistent:**

| Approach | How | Used for |
|---|---|---|
| **Rebuild + atomic swap** | Build the new shard index offline, ship it to every replica, flip all to version N+1 together | The big daily index |
| **Streaming upserts** | Each new vector written to every replica of its shard via a log; replicas apply it in order | The fresh tier (minutes-old moments) |

---

## 6 · LOAD BALANCING — *which replica answers*

```
            query q (for dance)
                   │
          ┌────────▼────────┐
          │  QUERY FRONTEND │   one request per SHARD, each to ONE replica
          └────────┬────────┘
    ┌──────────────┼──────────────┬──────────────┐
    ▼              ▼              ▼              ▼
  LB 0           LB 1           LB 2           LB 3     (a load-balancing
  ├─ 0a (busy)   ├─ 1a ◄ ✓      ├─ 2a          ├─ 3a     decision per shard)
  ├─ 0b ◄ ✓      ├─ 1b          ├─ 2b ✗ down   ├─ 3b ◄ ✓
  └─ 0c          └─ 1c          └─ 2c ◄ ✓      └─ 3c
    │              │              │              │
    └──────────────┴──────┬───────┴──────────────┘
                          ▼  merge → top-1,000 → reranker
```

**Strategies, from simplest to best:**

| Strategy | How it picks | Weakness |
|---|---|---|
| Round robin | Next replica in turn | Ignores that one replica is slow right now |
| Random | Any replica | Same |
| **Least outstanding requests** | The replica with the fewest in-flight queries | Needs live load information |
| **Power of two choices** | Pick 2 replicas at random, send to the less loaded | Nearly as good as least-loaded, with very little coordination: a strong default |

**Plus the essentials:**
- **Health checks:** a replica that fails checks is removed from rotation (2b above).
- **Zone awareness:** prefer replicas in the caller's zone, which is cheaper and faster.
- **Hedged requests:** if a shard hasn't answered within its p95 time, send the same request to a second replica and take whichever answers first. That cuts the tail at a few percent extra load (§7).

> **This is the answer to "what about peak hours":** load is spread by **replicas plus a load balancer**, not by how vectors are assigned to shards. Hashing balances the *data*; replication and load balancing balance the *queries*.

---

## 7 · THE TAIL-LATENCY TRAP — *why fan-out needs hedging*

```
  every dance query waits for ALL 4 shards → it's as slow as the SLOWEST one

  if each shard is fast 99% of the time (its p99 = 20 ms):
    P(all 4 fast) = 0.99⁴ ≈ 0.96
    → ~4% of QUERIES hit at least one slow shard
    → the QUERY's p96 ≈ one SHARD's p99

  with 20 shards:  0.99²⁰ ≈ 0.82 → ~18% of queries are slow
  ⚠ the more shards, the more a rare per-shard slowdown becomes a common one

  FIXES
    HEDGED REQUESTS   a shard slower than its p95 → ask a second replica too
    PARTIAL RESULTS   a shard that misses the deadline is dropped; merge the
                      other 3 and log "degraded recall" for that query
    FEWER, BIGGER     fewer shards per partition, if memory allows
      SHARDS
```

> **Say:** *"Fan-out turns one shard's p99 into the query's p96, so I'd hedge requests to a second replica and let a query proceed without a straggler shard rather than wait for it."*

---

## 8 · THE WHOLE PICTURE — *query path and write path together*

```
════ QUERY PATH ══════════════════════════════════════════════════════════════════
 query clip → encode (dance adapter v7) → q
       │
   ROUTER ──► dance partition (unsure? also search the 2nd-best partition)
       │
   SCATTER to 4 shards; per shard, the LOAD BALANCER picks one healthy replica
       │        ┌ shard 0 → 0b ┐
       ├───────►│ shard 1 → 1a │ each: local ANN search → local top-1,000
       │        │ shard 2 → 2c │ (2b is down, skipped by health checks)
       │        └ shard 3 → 3b ┘ (3b slow? HEDGE to 3a at the p95 mark)
       ▼
   GATHER: merge 4×1,000 by score → global top-1,000 → temporal reranker

════ WRITE PATH ══════════════════════════════════════════════════════════════════
 new dance moment (id = m91f3)
       │ embed with dance adapter v7
       ▼
   hash(m91f3) mod 4 = 2  ──►  SHARD 2  ──►  write to 2a, 2b, 2c  (via a log)
                                              2b down? it replays the log on recovery

════ DAILY REBUILD ═══════════════════════════════════════════════════════════════
 build shard indexes v(N+1) offline → ship to every replica → health-check
   → flip ALL replicas to v(N+1) together → keep vN for rollback
```

---

## 9 · SIZING IT — *the two formulas*

```
SHARDS per partition   = ceil( partition size  /  usable RAM per machine )
REPLICAS per shard     = ceil( peak QPS for the partition  /  QPS one replica serves )
                         + 1 spare, if it must survive losing a replica AT PEAK

  ⚠ with HASH sharding every query hits EVERY shard, so each shard sees the
    partition's FULL QPS, not a quarter of it

WORKED EXAMPLE (illustrative):
  dance: ~110 GB, 32 GB usable per machine          → 4 shards
         peak 200 QPS, one replica serves ~70 QPS   → 3 replicas (4 with a spare)
         → 12 machines (16 with spares)
  skate: ~22 GB                                      → 1 shard
         peak 20 QPS                                → 2 replicas (failover, not load)
         → 2 machines
  ⭐ HOT PARTITIONS GET MORE REPLICAS: scaling is per partition, which is
    another reason to partition at all
```

---

## 10 · FAILURES AND WHAT HAPPENS

| Failure | What the system does | What the user sees |
|---|---|---|
| **One replica dies** | Health check removes it; the other replicas take its share | Nothing |
| **A whole zone goes down** | Replicas in the other zones serve | Nothing, if replicas were spread across zones |
| **Every replica of one shard down** | Merge the other 3 shards' results; log degraded recall | Slightly worse results, never an error |
| **One shard is slow** | Hedge to another replica; drop it at the deadline | Nothing, or slightly worse results |
| **Router misclassifies the vertical** | If confidence is low, search the top 2 partitions | Usually nothing |
| **A partition gets hot** (a viral dance trend) | Autoscale **its** replicas; cache popular queries | Nothing, if scaling is fast enough |
| **A partition outgrows its shards** | Add a shard; consistent hashing moves only ~1/N of vectors | Nothing (done offline, swapped in) |
| **Replicas on different index versions** | Version pinned per request; all flip together | Nothing; otherwise inconsistent results |

---

## 10.5 · DELETES AND UPDATES — *why ANN indexes make them awkward*

**The problem:** ANN structures are built for reads, not removals.

| Index type | Deleting a vector | Consequence |
|---|---|---|
| **HNSW** (graph) | Removing a node cuts the edges other nodes use to navigate; repairing them is expensive | Usually **not truly deleted**: tombstoned instead |
| **IVF / IVF-PQ** (inverted lists) | Remove the entry from its cluster's list: cheap | Easy to delete, but the cluster centroids slowly go stale |
| **Exact / flat** | Remove the row | Easy, at small scale only |

**The standard recipe: four layers.**

```
1. TOMBSTONE (seconds)    mark the ID deleted in a fast DELETED SET, replicated
                          to every replica; the vector stays in the index for now
2. FILTER AT QUERY TIME   every result is checked against the deleted set before
                          it's returned → a deleted item can never be served,
                          however far behind the index is
   + OVER-FETCH           if ~10% of a shard is tombstoned, ask it for ~1.1×K so
                          filtering doesn't leave fewer than K results
3. COMPACT / REBUILD      tombstones still cost: HNSW still walks through them,
   (hours–daily)          wasting time and hurting recall. Past a threshold
                          (e.g. > 10–20% of a shard deleted), rebuild that shard
                          from the source of truth minus deletions, offline, then
                          swap atomically across its replicas (§5)
4. PURGE EVERYWHERE       caches, the fresh tier, other replicas, snapshots and
                          backups: a "deleted" vector still in last week's
                          snapshot comes back on a restore
```

**Updates = delete + insert.** A changed item gets a new version: tombstone the old vector, insert the new one, and check the version at query time (exactly R8's registry check).

**Deleting by parent.** In R2, a takedown removes a **video**, but the index holds **moments**. Keep a secondary map `video_id → [moment_ids]`, so one takedown tombstones all ~30 of its moments at once.

**Why it matters in an interview:** deletes are often **legal or privacy obligations** (a takedown, a user deleting their data, content made private), with an SLA. So the guarantee must come from **layer 2, the query-time filter**, not from how fast the index gets rebuilt. That's the same principle as R8's "the registry is the gate; the index can lag".

> **Say:** *"HNSW doesn't delete cleanly, so I tombstone in a replicated deleted set, filter every result against it and over-fetch to compensate, rebuild a shard once tombstones pass a threshold, and purge caches and snapshots too. The query-time filter is what makes the guarantee; the rebuild is housekeeping."*

---

## 11 · INTERVIEW LINES — *and where they earn credit*

| Say | Earns |
|---|---|
| *"Partition by vertical to cut what a query searches; each vertical has its own embedding space anyway."* | **F3b**, A2a |
| *"Hash-shard within a partition: balanced data, no recall lost at boundaries, at the cost of fan-out."* | **F3b**, A5a |
| *"Each shard returns the full top-K, not K over N, or the merge misses results."* | F1a |
| *"Replicas plus a load balancer absorb peak QPS; hashing only balances data."* | **I1d**, I1b |
| *"Fan-out turns one shard's p99 into the query's p96, so I hedge and accept partial results."* | **I1b**, A1c |
| *"Replicas spread across zones, one index version per request, rebuilt offline and swapped atomically."* | **I1b** |
| *"Shards = size ÷ RAM; replicas = peak QPS ÷ per-replica QPS, plus a spare."* | **A1b**, I1d |

**Where each sheet uses this:** R2 (moment index, partitioned by vertical) · G13 (video index, "hash-sharded, replicated 2–3×") · G1 (visual search, HNSW + PQ) · R8 (chunk index; restricted collections as separate partitions) · R6 (no sharding needed at 50k items: the whole index fits in every server).
