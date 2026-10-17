# Q2 · Coding Agent — **INTERVIEW SHEET**

> **Tier 3 · #23** — very likely if the team is DevInfra or developer-facing.
> Full study file: [`../Q2_coding_agent.md`](../Q2_coding_agent.md) · [diagrams](../Q2_coding_agent_diagrams.html)
>
> 🧾 **Built against Google's L6 evaluation form** (full form in [R1](R1_prompt_suggestions_intv.md)): **A** System Architecture/Design · **F** AI Fluency · **I** AI Product Integration, 10 sub-parameters. Every section carries the rubric codes it earns; the [🧾 coverage map](#-rubric-coverage-map--every-pointer-as-it-applies-to-this-question) at the end gives the exact line for each pointer.
>
> **Expect these probes:** (a) assumptions · (b) what the developer sees · (c) the agent loop · (d) datasets · (e) building the task suites · (f) the models and their objectives · (g) bounded autonomy at runtime · (h) how you know it worked · (i) the extension.

---

## 🎯 APPLY R1'S FOUR LESSONS

| Lesson from the sample feedback | Here |
|---|---|
| Clarifying questions **unprompted**, minute 1 | Tests? PR or commit? Cost ceiling? Whose code — ours or customers'? (§1) |
| **Explainability** is easy points | Every PR carries **a rationale, the tests run, and a link to the trace**; a stopped run says **why it stopped** (§2, §8) |
| **Security** beyond the gateway | **Repo content is untrusted input** (injection via issues, READMEs, comments) · secrets in traces · new dependencies · sandbox escape (§8) |
| **Troubleshoot before being asked** | Raise **silent truncation**, **test gaming** and **flaky tests** yourself (§3, §4) |

---

## ⏱ THE TIMED SPINE — *60-minute slot*

| Min | Section | Probe | What you say here | Rubric codes |
|---|---|---|---|---|
| **0–1** | **OPENING** | — | ⭐ **A deterministic control system around a stochastic core** | A1a, A3a |
| **1–6** | **REQUIREMENTS** | **(a)** | Clarifying Qs **unprompted** · assumptions · `pass^k` as the gate · non-goals | A4a, A1b |
| **6–9** | **PRODUCT + METRICS** | **(b)**, (h) | The PR · ⚠ **`pass^k`, not `pass@k`** · review burden | I1c, F1b, I2b |
| **9–12** | **CAPACITY + BUDGETS** | — | Agent turn ≈ **4×** a chat turn · **context as a budget** · sandboxes | A1b, I1a, I1d |
| **12–22** | **ARCHITECTURE** | **(c)** | Orchestrator · context manager · sandbox · **verifier outside it** · trace store | A2a, A5a, A1c, F2b |
| **22–32** | **DATA & MODEL** | **(d) (e) (f)** | Fresh verifiable tasks · traces → SFT → RL on tests | A2b–d, F1a, F1c, F2c |
| **32–38** | **BOUNDED AUTONOMY** | **(g)** | Termination · cost cap · blast radius · checkpoint | F3a, F3b, A3b, I1b |
| **38–44** | **EVAL + ROLLOUT + BIAS** | **(h)** | Contamination · shadow PRs · revert rate as the slow truth | I2a, I2b, F1c |
| **44–48** | **PRODUCTION-READINESS SWEEP** | — | Nine dimensions, said as a checklist | I1a–I1j |
| **48–52** | **EXTENSION + FAILURES** | **(i)** | Untested monorepos · **suggestion-mode fallback** · top 3 failures | F3c, F2a, A5b |
| **52–60** | **PROBES** | — | Follow-ups below | — |

---

## 0 · THE OPENING — *memorize these three sentences* `A1a` `A3a`

> **"A coding agent is a deterministic, observable control system wrapped around a stochastic core. The model reasons, the orchestrator controls, the sandbox executes, the verifier judges — and almost all the interesting engineering is in the loop and its bounds, not in the prompt.**
>
> **Three things I'd optimise, in order. First, a trustworthy verifier: tests give me a signal I can both ship on and train on, and unlike a learned judge it's hard to game if I control the grading harness. Second, context as a managed budget — agent turns are roughly 4× the tokens of a chat turn, and silent truncation is the single most common cause of an agent forgetting its own plan mid-task. Third, bounded autonomy — every loop needs a termination condition, a cost cap, and a blast radius, or it will find a way to run forever or do something irreversible.**
>
> **And I'd scope it deliberately: repo-scoped, test-backed tasks first. Those are verifiable, which makes evaluation and RL tractable. Greenfield with no tests is a much harder reward problem and I'd defer it."**

**Then say the plan (`A3a`):** *"Clarifying questions and assumptions, what the developer sees and how we measure it, the context and cost budgets, the agent loop, the data and training flywheel, the bounds at runtime, evaluation, a production-readiness sweep, then extensions and failure modes."*

---

## 1 · REQUIREMENTS *(min 1–6)* — probe **(a)** `A4a` `A1b`

**⚠ Ask these BEFORE you're asked:**
1. **Does the repo have tests?** *(**Verifiability is the whole scoping decision** — test-backed tasks are a different product from greenfield.)*
2. **Does it open PRs, or commit directly?** *(Blast radius — a PR is reviewable; a push to main is not.)*
3. **What's the cost ceiling per task?** *(An agent with no cap will spend until something stops it.)*
4. **Whose code — internal repos or customers'?** *(Customer code in traces is a privacy and training-consent question.)*

**State assumptions:**

| Assumption | Value |
|---|---|
| Product | An **async** agent: takes an issue, returns a PR (not inline IDE completion) |
| Repos | Internal, mostly test-backed; many languages and build systems |
| Volume | Thousands of tasks a day across teams |
| Model | A strong hosted model, with a smaller one for cheap first passes |
| Review | Every PR reviewed by a human owner before merge |

**FR:** read/write repo files · run tests in a **sandbox** · iterate on failures · open a PR with a diff and rationale · **terminate on a condition** · emit a full trace.

**NFR / SLO:**

| Dimension | Target |
|---|---|
| **`pass^k`** on the eval suite | The gate — *not* `pass@k` |
| **Cost per task** | Capped, and **enforced**, not advisory |
| **Blast radius** | PR-only by default; **no direct push, no prod credentials** |
| Trace completeness | 100% — *the trace is the product's memory* |
| Wall-clock per task | Bounded (a task that runs forever is a failure, not a slow success) |

**Scope / NON-GOALS:**
- ❌ **Greenfield code with no tests.** Position: the reward problem is much harder; defer it.
- ❌ **Direct commits to main** · production access · infra changes.
- ❌ The base model itself (I'm building the system around it) · IDE UX.

---

## 2 · PRODUCT + METRICS *(min 6–9)* — probes **(b)**, (h) · **the L6 move lives here** `I1c` `F1b` `I2b`

**What the developer sees (b):**

```
PR #4821  "Fix off-by-one in pagination cursor"           opened by the agent
  Rationale: the cursor skipped the last item when page_size divided total;
             changed < to <= in cursor.py:112 and added a regression test.
  Tests run: 214 passed · 0 failed · lint ✓ · types ✓
  Diff: 2 files, +18 −3                                    ← reviewable size
  Trace: plan · 7 turns · commands · test output · $0.84   ← one click
  ⚠ Test files modified: 1 (new test only, none deleted)   ← flagged for review

OR, when it can't:
  "Stopped after 8 turns: same edit twice with no change in test outcome.
   Tried A and B; the failure is in the fixture, not the code. Suggest …"
```

```
⚠ pass@k SAYS THE CAPABILITY EXISTS. pass^k SAYS IT'S RELIABLE.

  pass@k = at least one of k attempts succeeds  ← flatters badly
  pass^k = ALL k attempts succeed               ← what users experience

  A USER GETS ONE ATTEMPT. A model at 70% pass@5 and 25% pass^5 feels
  broken in production and excellent on a leaderboard.
  → GATE ON pass^k.

AND MEASURE THE THINGS THAT AREN'T "did it pass":
  · COST per solved task (an agent that solves it for $40 is not a win)
  · TURNS to solution (and the distribution's tail)
  · ⚠ HUMAN REVIEW BURDEN — a 900-line diff that passes tests is
    NOT a success if nobody can review it
  · REVERT RATE / post-merge defect rate  ← the slow truth
```

**The metric stack:**

| Layer | Metric | Role |
|---|---|---|
| **Gate** | **`pass^k`** on a fresh, internal held-out suite | Reliability, not capability |
| **North star** | **Merged PRs not reverted within 30 days**, per engineer-hour saved | Work that stuck |
| Efficiency | Cost per solved task · turns · wall-clock | The bill |
| **Guardrails** | Review time · diff size · test-file modifications · revert rate | The cost pushed onto humans |

---

## 3 · CAPACITY + BUDGETS *(min 9–12)* — **context as a budget** `A1b` `I1a` `I1d`

```
⚠ AN AGENT TURN IS ~4× A CHAT TURN:
   system + tools + file contents + test output + prior turns
   → context fills FAST, and the default failure is SILENT TRUNCATION

⚠ SILENT TRUNCATION IS THE #1 CAUSE OF AN AGENT FORGETTING ITS OWN
  PLAN MID-TASK. Nothing errors. The agent just starts behaving as if
  it had never made a decision it made ten turns ago.

SO: CONTEXT IS A MANAGED BUDGET, not a limit you hit.
   · reserve a fixed slice for THE PLAN and the task statement —
     never evictable
   · summarise-and-evict old turns DELIBERATELY, with the summary
     written by the agent before eviction
   · file contents are RETRIEVED ON DEMAND, not accumulated
   · ⚠ TEST OUTPUT IS THE BIGGEST OFFENDER — truncate stack traces
     to the relevant frames, don't paste 4k lines of pytest output

COST: a hard task ≈ 30 turns × ~20k tokens re-read per turn
   ≈ 600k input tokens → PREFIX CACHING of the stable head (system,
   tools, plan) is the biggest single cost lever
   → cost per task is a REAL number and it belongs in the SLO.

SANDBOXES: one container per running task; pre-warmed images per repo,
   because building the environment often costs more wall-clock than
   the agent's reasoning — and TEST RUNTIME dominates the loop
```

---

## 4 · ARCHITECTURE *(min 12–22)* — probe **(c)** `A2a` `A5a` `A1c` `F2b`

### The decision first: alternatives `A5a`

| Decision | Options | Choice and why |
|---|---|---|
| Verifier | An LLM judge vs **tests + lint + types, owned by us** | Deterministic and hard to game; a judge only for style signals |
| Structure | Multi-agent (planner / coder / reviewer) vs **one bounded loop** | Errors compound across hops; split later if traces show a need |
| Autonomy | Direct commit vs **PR-only** | Reviewable and revertible |
| Context | Accumulate vs **a managed budget** with a pinned plan | Silent truncation is the #1 failure |
| Model | One large model vs **a cascade**: small first pass, escalate on failure | Most of the value at a fraction of the cost (FU 8) |

### The system — *draw this in 90 seconds* `A2a`

```
 TASK (issue, description, or failing test)
       │
┌─ 1. ORCHESTRATOR — the DETERMINISTIC part ─────────────────────
│ TASK: plan → act → observe → repeat, as a state machine
│ ⚠ TERMINATION: tests pass · turn cap · cost cap · no-progress
│ OUTPUT: the next model call, or STOP with a reason
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 2. CONTEXT MANAGER — the managed budget (§3) ─────────────────
│ pinned plan slot · summarised history · on-demand file retrieval
│ · truncated test output
│ OUTPUT: a context that fits, with the plan always in it
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 3. MODEL — reasoning + tool calls ────────────────────────────
│ OUTPUT: edits, commands, or "done"
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 4. SANDBOX — the blast radius ────────────────────────────────
│ a repo COPY, not the real repo · no network by default · no prod
│ credentials · secrets never mounted
│ OUTPUT: command results, test output
└──────┬─────────────────────────────────────────────────────────
       ▼
┌─ 5. VERIFIER — OUTSIDE the sandbox ────────────────────────────
│ run tests · lint · type-check from a source the agent can't edit
│ ⚠ or the agent will edit the tests to pass
│ OUTPUT: pass / fail + structured failures
└──────┬─────────────────────────────────────────────────────────
       ▼
 PR with diff + rationale + trace link
       ▼
┌─ 6. TRACE STORE → EVAL HARNESS → TRAINING ─────────────────────
│ ⚠ this second half is what makes the system GET BETTER instead
│   of just EXIST (§5.5)
└────────────────────────────────────────────────────────────────
```

### ⚠ Raise these yourself, before the interviewer does `F2b` `A1c` `F1c`

- **Silent truncation:** *"The default failure of a long loop is losing the plan with no error — so the plan has a pinned slot."*
- **Test gaming:** *"Deleting the failing test is the shortest path to green, so the grader lives outside the sandbox and test-file edits are flagged."*
- **Flaky tests:** *"A flaky test is a noisy reward and a false failure. Run each test several times before it enters a suite, and quarantine the flaky ones."*

### The defensible choices — know the trade for each

| # | Choice | Say this | The trade you accept |
|---|---|---|---|
| **1** | **Scope to test-backed tasks** | "Tests are a verifier I can ship on *and* train on, and hard to game if I own the harness" | Excludes greenfield, where much of the demand is |
| **2** | **Grader outside the sandbox** | "Otherwise the agent edits the tests to pass — and it will find that, reliably" | An extra boundary and a slower loop |
| **3** | **PR-only, never direct commit** | "A PR is reviewable and revertible; a push to main is an irreversible action by an agent" | Slower perceived autonomy |

---

## 5 · DATA & MODEL *(min 22–32)* — probes **(d) (e) (f)** `A2b` `A2c` `A2d` `F1a` `F1c` `F2c`

### 5.1 The datasets (d) `A2c`

| Source | What it gives | Watch out for |
|---|---|---|
| **Issue → merged PR pairs** from internal repos | Tasks with **fail-to-pass tests** (fails before the fix, passes after) | Only works where a test came with the fix |
| **Fresh tasks authored after the model's cutoff** | A clean, uncontaminated gate | Expensive to author |
| **Agent traces** | Plans, turns, commands, outcomes | Contain proprietary code and possibly secrets |
| **Review outcomes** | Approved · changes requested · reverted | Slow; the revert signal takes weeks |
| **CI history** | Which tests are flaky | Needed before any test becomes a reward |
| **Public benchmarks** (SWE-bench) | Comparability | Contaminated; repos unlike ours (FU 6) |

### 5.2 Building the task suites (e) `F1c`

- **Verifiable only:** a task enters the suite if a fail-to-pass test exists **and** passes deterministically on several reruns.
- **Contamination:** exclude tasks resolved before the model's cutoff from the gate; plant **canary strings** in private suites.
- **Stratify:** by repo, language, difficulty and change type, so one popular repo doesn't define "good".
- **For RL, keep tasks with mixed outcomes:** a task the policy always solves or never solves gives **no gradient** under group-relative methods — measure each task's pass rate first and train where it's in between.
- **Hold out whole repos**, not just tasks, to measure transfer to unseen codebases.

### 5.3 The models and their objectives (f) `A2b` `F1a`

| Component | Task | Objective |
|---|---|---|
| **Policy** (the agent model) | Solve the task in the loop | **SFT** on successful, approved, non-reverted traces, then **RL with a verifiable reward** |
| **The reward** | Score a finished run | **Multiplicative gates × pass**: tests pass **and** no test deleted or weakened **and** diff within bounds; any gate fails → 0 |
| **Router** | Small model first, or large? | A classifier on task features, trained on past pass/fail and cost |
| **Code retriever** | Find the files that matter | Contrastive on issue → files actually edited; plus structural search (symbols, call graph) |

### 5.4 Fine-tune an existing model (f) `F2c`

Start from a strong code model. Run **rejection-sampling fine-tuning** first: sample many attempts per task, keep the ones that pass every gate, and fine-tune on them. It's cheap, stable and uses the same verifier. Then RL on the mixed-outcome tasks; full-parameter for the shipped policy, **LoRA** for the router and the retriever, which are small and retrained often.

### 5.5 The staged pipeline — *the flywheel* `A2d`

```
STAGE 1 · TRACE     every run: plan, turns, commands, diffs, outcome
STAGE 2 · FILTER    passed + approved + not reverted → positive traces;
                    secrets and customer code handled per policy
STAGE 3 · SFT       rejection-sampled successes
STAGE 4 · RL        verifiable reward on mixed-outcome tasks
STAGE 5 · GATE      pass^k on the fresh internal suite · cost · diff size
STAGE 6 · ROLLOUT   shadow PRs → opt-in teams → GA (§7)
```

---

## 6 · BOUNDED AUTONOMY *(min 32–38)* — probe **(g)** `F3a` `F3b` `A3b` `I1b`

```
EVERY LOOP NEEDS FOUR BOUNDS — or it runs forever or does something
irreversible:

 1. TERMINATION CONDITION  tests pass · turn cap · ⚠ NO-PROGRESS
    DETECTOR (same file edited 5× with no test delta ⇒ stop)
 2. COST CAP               enforced by the orchestrator, not suggested
                           in the prompt
 3. BLAST RADIUS           sandbox · repo copy · no network · no prod
                           creds · PR not push  (Q10's tiering)
 4. HUMAN CHECKPOINT       for anything irreversible or large

⚠ THE FAILURE PEOPLE UNDER-ESTIMATE: the agent that "succeeds" by
  DELETING THE FAILING TEST. That's why the grader is outside the
  sandbox AND why the diff is reviewed — a green suite is necessary,
  not sufficient.
```

- **The objective** `A3b`: maximise `pass^k`, **subject to** the cost cap, a diff-size bound and zero test deletions — constraints the orchestrator enforces, not weights in a reward someone can tune away.
- **Efficiency** `F3a`: prefix caching of the stable head; the small-model first pass; truncated test output. The trade: the cascade adds latency on tasks that escalate, which is accepted for the cost saving on the many that don't.
- **Scale** `F3b`: a pre-warmed, autoscaled sandbox pool per repo image; a task queue with per-team quotas.
- **Reliability** `I1b`: the agent's state is **checkpointed every turn**, so a crashed sandbox resumes rather than restarts; tool calls are idempotent; a run that hits a bound ends in a **handoff with what it tried**, never a silent failure.

---

## 7 · EVAL, ROLLOUT, BIAS *(min 38–44)* — probe **(h)** `I2a` `I2b` `F1b` `F1c`

| | |
|---|---|
| **The gate** | **`pass^k`** on a held-out task suite, plus cost per solved task |
| **Trace store** | Every run: plan, turns, tool calls, diffs, test output, outcome. **This is the asset** |
| **Eval harness** | Replay tasks against a new model/prompt/orchestrator version — *the harness version is part of the cache key* (Q5) |
| **RL loop** | Tests give a **verifiable reward** — this is the rare agent domain where RL is genuinely tractable (Q4) |
| **Contamination** | ⚠ Public benchmarks are in the training data. **Author fresh tasks after the cutoff**, or measure memorisation |
| **Human-review burden** | Diff size, review time, revert rate — the metric nobody builds |

```
ROLLOUT:  offline pass^k on the fresh internal suite
          → SHADOW: drafts PRs on real issues, not opened; owners grade a sample
          → opt-in teams, PRs opened, every one reviewed
          → GA by change class (mechanical first) · revert rate as the slow truth
```

**Success** = `pass^k` up on a fresh internal suite, merged-and-not-reverted PRs per engineer-hour up, review time per PR flat or down, and cost per solved task within budget. `I2b`

**Bias sources and mitigations** `F1c`

| Bias | Mechanism here | Mitigation |
|---|---|---|
| **Benchmark-repo bias** | Public repos look nothing like ours | An internal suite as the real gate |
| **Language** | Python-heavy training and benchmarks | Suites stratified by language; per-language `pass^k` |
| **Verifiability selection** | Only tested code gets measured, so scores over-predict on untested code | Report coverage of the suite; FU 10 |
| **Contamination** | Benchmark solutions in the training data | Post-cutoff tasks; canaries; fresh-vs-old delta |
| **Reviewer bias** | Rubber-stamping agent PRs, or over-scrutinising them | Track review time and revert rate per reviewer |

---

## 8 · PRODUCTION-READINESS SWEEP *(min 44–48)* — `I1a` `I1b` `I1c` `I1d` `I1e` `I1f` `I1h` `I1i` `I1j`

> **Parameter 3 · Sub-parameter 1: production readiness across dimensions.** Say it as a checklist: every row is a separate pointer, and the sample candidate lost points on the rows they skipped.

| Dimension | For THIS system | Metric |
|---|---|---|
| **Performance** `I1a` | Bounded wall-clock; cost per solved task; prefix caching and the cascade | p95 wall-clock · $ per solved task |
| **Reliability** `I1b` | Per-turn checkpoints; no-progress detector; flaky tests quarantined; handoff on any bound | Crash-resume rate · runaway runs |
| **Usability** `I1c` | **Reviewable diffs**, a plain rationale, a useful handoff when it can't finish | Review time · acceptance rate |
| **Scalability** `I1d` | Autoscaled sandbox pool; pre-warmed images; per-team quotas | Queue wait · sandbox spin-up time |
| **Privacy** `I1e` | Customer code in traces: access control, retention limits, **no training on it without consent**; secrets scrubbed from traces | Secrets found in traces = 0 |
| **Safety** `I1f` | PR-only; no prod credentials; **human checkpoint** for auth, payments, migrations, infra | Agent changes merged without review = 0 |
| **Security** `I1h` | ⚠ **Repo content is untrusted** (issue text, READMEs, comments can carry injections); **no egress** so secrets can't leave; **new dependencies flagged** (typosquatting); sandbox escape hardening | Red-team pass rate · flagged deps |
| **Ethics / bias** `I1i` | **Licence compliance** — no verbatim copying of incompatibly licensed code; review load shared fairly, not dumped on one team | Licence-scan hits · review load per person |
| **Explainability** `I1j` | Rationale, tests run and a **trace link** on every PR; every stop says **why**; test-file edits called out | % PRs with rationale + trace |

> ⭐ *"The verifier makes this agent trainable; the bounds make it safe; the trace makes it debuggable. Production readiness is having all three before the first PR opens."*

---

## 9 · EXTENSION + FAILURES *(min 48–52)* — probe **(i)** `F3c` `F2a` `A5b`

**Extensions:**
- **Customer monorepos with no tests:** the verifier disappears, and retrieval and environment setup become the bottleneck (FU 10).
- **Multi-agent decomposition** (planner / coder / reviewer): only once traces show where a single loop fails, since errors compound across hops.
- **Inline IDE agent:** a different product with a seconds-level latency budget and no time for a full test run.

**When there's no verifier to trust — the suggestion-mode pipeline** `A5b` `F2a`:

```
no tests, flaky tests, or a repo the agent hasn't earned autonomy on
  → no autonomous PR: the agent writes a DRAFT patch + explanation for a human
  → or FIRST writes characterisation tests around the code it will touch,
    reviewed by a human — which creates the missing verifier
  → only once those tests exist and are stable: the normal loop resumes
  "The agent's capability ceiling is the repo's verifiability."
```

**Top 3 failure modes:**
1. **Silent context truncation** — the agent forgets its own plan; no error, just incoherent behaviour ten turns in.
2. **Test gaming** — deletes or weakens the failing test. Green suite, broken code.
3. **Runaway loop** — same edit, same failure, thirty turns, $40 spent, no progress.

**Q1 ships:** the **orchestrator with real bounds** + **sandbox with the grader outside** + the **trace store**.
> *"Traces in Q1 even though they deliver nothing to users — they're what the eval harness and the RL loop are built on, and you cannot reconstruct them retroactively."*

---

## 🔥 TOP 5 FOLLOW-UPS

**1. "Why `pass^k` instead of `pass@k`?"** `F1b` `I2b`
> Because **a user gets one attempt**. `pass@k` says at least one of k tries succeeded, which measures whether the capability *exists*; `pass^k` says all k succeeded, which measures whether it's *reliable*. A model at 70% `pass@5` and 25% `pass^5` looks excellent on a leaderboard and feels broken in production, and that gap is exactly the difference between a demo and a product. So I'd gate on `pass^k` for anything user-facing. And I'd add two metrics that "did it pass" misses entirely: **cost per solved task**, because solving it for forty dollars isn't a win, and **human-review burden** — a nine-hundred-line diff that passes tests is not a success if nobody can review it, and that's a cost the agent externalises onto the team.

**2. "How do you stop the agent from cheating on the tests?"** `I1h` `A3b`
> Structurally, not by asking it not to. **The grader lives outside the sandbox** — the agent can edit the repo copy it works in, but the tests that determine success are run from a source it cannot reach. That's the single most important boundary in the design, and without it the agent will find test-deletion reliably, because it's the shortest path to the reward. Then **the diff is reviewed**, so a green suite is necessary but not sufficient, and I'd specifically flag diffs that modify test files as requiring closer review. And I'd monitor for the softer version: tests weakened rather than deleted, assertions loosened, a test marked skip. The general principle is the one from RL: **if the policy can reach the grader, it will optimise the grader** — so the grader must be unreachable rather than protected by instruction.

**3. "The agent forgot what it was doing halfway through. Why?"** `F2b` `I1b`
> Almost certainly **silent context truncation**, which is the most common failure in agent loops and the least visible. An agent turn is roughly four times a chat turn — system prompt, tool definitions, file contents, test output, and every prior turn — so context fills fast, and the default behaviour when it overflows is to drop the oldest content. That's usually the **plan**, made at turn one. Nothing errors; the agent simply starts behaving as though it never made a decision it made ten turns ago. So context has to be a **managed budget**: a pinned, never-evictable slot for the plan and the task statement; deliberate summarise-then-evict of old turns, with the summary written *before* eviction rather than reconstructed after; file contents retrieved on demand rather than accumulated; and aggressive truncation of test output, which is the biggest single offender — nobody needs four thousand lines of pytest to know which assertion failed.

**4. "What bounds the loop?"** `A1a` `I1f`
> Four things, and all four are enforced by the orchestrator rather than requested in the prompt. A **termination condition** — tests pass, or a turn cap, or a **no-progress detector**: if the same file has been edited five times with no change in the test outcome, the agent is stuck and more turns won't help. A **cost cap**, enforced, because an agent without one will spend until something external stops it. A **blast radius** — a sandboxed repo copy, no network by default, no production credentials, and a PR rather than a direct push, so every action the agent takes alone is reviewable and revertible. And a **human checkpoint** for anything irreversible. The framing I'd use is Q10's: prompt-level restrictions are requests, orchestrator-level bounds are guarantees, and for a loop that can run for thirty turns you want guarantees.

**5. "How does this system get better over time?"** `A2d` `I2a`
> Through the second half of the design, which is the part people skip: the **trace store, the eval harness, and the RL loop**. Every run emits a complete trace — plan, turns, tool calls, diffs, test output, final outcome — and that store is the actual asset, because you can't reconstruct it retroactively. On top of it, an **eval harness** that replays a held-out task suite against any new model, prompt, or orchestrator version, so a change is a measurement rather than an argument. And then RL, which is unusually tractable here: **tests are a verifiable reward**, deterministic and hard to game if I own the harness, which is exactly the tier-1 signal that's missing in most agent domains. That's the real reason to scope to test-backed tasks — not that they're easier, but that **verifiability is what makes the flywheel possible at all**.

---

## 🔥🔥 FIVE HARDER ONES — *the L6 separators*

**6. "SWE-bench scores are in every model's training data. How do you evaluate honestly?"** `F1c` `I2a`
> By assuming contamination and designing around it, rather than reporting a number I can't defend. Three things. **Author fresh tasks after the model's cutoff** — real issues from real repos, resolved after the training data was collected — which is expensive and is the only source of a genuinely clean signal. **Use canary strings** in any private task set so I can grep a corpus for a definitive answer about leakage. And **measure the fresh-versus-old delta**: performance on a public benchmark against a freshly authored set of equal difficulty, where the gap *is* the contamination estimate and works without access to the training data. Beyond contamination there's a subtler problem I'd raise: public benchmarks are drawn from **popular open-source repos**, which look nothing like a typical enterprise codebase — unusual build systems, undocumented internal conventions, sparse tests. So even an uncontaminated public score over-predicts performance on the code people actually want fixed, and I'd want an **internal task suite** drawn from our own repos as the real gate.

**7. "The agent opens a PR that passes tests but the reviewer hates it. What's the metric failure?"** `F1b` `A3b`
> The verifier is measuring **correctness** and the reviewer is measuring **maintainability**, and only one of those is in the reward. So the agent optimises toward the checkable thing and drifts away from the uncheckable one — a nine-hundred-line refactor that passes, a fix that special-cases the test rather than fixing the cause, a new dependency added to avoid writing ten lines. That's classic Goodhart with a verifiable proxy, and the proxy being *honest* doesn't make it *complete*. What I'd add: **diff-size and blast-radius limits** as a constraint rather than a preference, since a bounded diff is both more reviewable and less likely to be a workaround; **lint, type-check and convention checks** in the verifier, which converts some of the maintainability signal into a checkable one; and **human-review burden as a tracked metric** — review time, revert rate, post-merge defect rate — so the cost the agent externalises onto the team is visible. The slow truth is **revert rate**, and I'd treat that the way G7 treats returns: the fast signal for iteration, the slow one for gating.

**8. "One task takes forty turns and forty dollars. Another takes three. Same suite. What do you do?"** `F3a` `I1a`
> Treat the distribution rather than the mean, because the tail is where both the cost and the failure live. First, **diagnose the tail**: long runs are usually one of three things — a genuinely hard task, a **no-progress loop** where the agent is cycling on the same edit, or a **context-truncation** case where it lost the plan and is rediscovering it. Those need different fixes and they're distinguishable from the trace, which is one of the concrete payoffs of storing traces. Second, **route by predicted difficulty**: a cheap first pass with a smaller model and a low turn cap, escalating to a stronger model only on failure — the same cheap-wide-then-expensive-narrow cascade as everywhere else in this pack, and it typically captures most of the value at a fraction of the cost. Third, **fail fast and hand off**: an agent that stops at turn eight with "I couldn't solve this, here's what I tried and where I got stuck" is more valuable than one that burns forty dollars and produces a bad diff — the **warm handoff** argument from the support agent applies here too.

**9. "Would you let this agent run without a human in the loop?"** `I1f` `A3b`
> For a bounded class of changes, yes — and I'd define the class by **reversibility and blast radius** rather than by confidence. Fully autonomous: changes that are reversible, verifiable, and low-reach — dependency bumps with a green suite, lint fixes, mechanical refactors, test additions. Those are tier-1 in Q10's terms and a human reviewing them adds cost without adding safety. Human-in-the-loop: anything touching **security, auth, data migrations, payment paths, or infrastructure**, regardless of how confident the agent is, because the cost asymmetry is enormous and a green test suite doesn't cover the failure that matters. The mechanism that makes autonomy safe isn't the model's accuracy, it's that **a PR is revertible** — so I'd rather have broad autonomy on a revertible surface than narrow autonomy on an irreversible one. And I'd insist on the monitoring the support agent needs too: **approval latency**, because if reviewers are rubber-stamping agent PRs in four seconds, the human checkpoint is theater and I'd rather know.

**10. "Your agent works on your repos. A customer's monorepo is ten million lines with no tests. What breaks?"** `F3c` `A2c`
> Nearly everything the design depends on, and I'd say so rather than promise it generalises. **The verifier disappears** — no tests means no deterministic reward, which breaks the eval harness, the RL loop, and the autonomy argument all at once; the product becomes suggestion rather than autonomous change. **Retrieval becomes the bottleneck**: at ten million lines the agent can't see the codebase, so the question shifts from "can it reason" to "can it find the three files that matter," which is a code-retrieval problem — and a large monorepo has internal conventions and cross-module coupling that neither embeddings nor grep handle well. **Build and environment setup** becomes the dominant engineering cost — most of the work is making the repo runnable in a sandbox at all, which is unglamorous and is where these deployments actually stall. So my sequencing would invert: **first make the repo verifiable** — characterisation tests around the areas the agent will touch, which is itself a good task for the agent under human review — then expand autonomy behind that. The general lesson: **the agent's capability ceiling is set by the repo's verifiability**, not by the model.

---

## ⭐ THE L6 LINE

> **"A coding agent is a deterministic control system around a stochastic core — the engineering is in the loop and its bounds. Scope to test-backed tasks because a verifier you own is a reward you can ship on *and* train on, keep the grader outside the sandbox, and gate on `pass^k` because users get one attempt."**

## 🧭 HOOKS — *steer by what you mention*

- *"Bounded autonomy — termination, cost, blast radius"* → invites the agent-safety branch (Q10)
- *"Silent truncation is the #1 cause"* → invites the context-engineering branch (Q23)
- *"Tests are a verifiable reward"* → invites the RL branch (Q4)
- *"The grader lives outside the sandbox"* → invites the reward-hacking branch (**good at L6**)
- *"Human-review burden is a cost the agent externalises"* → invites the org/metrics branch (**good at L6**)
- *"The capability ceiling is the repo's verifiability"* → invites the enterprise-deployment branch

## 🆘 RECOVERY & DEFER

- **Blanked?** → *"Let me go back to the four bounds"* — termination, cost, blast radius, checkpoint regenerates the loop.
- **Running out of time?** → *"I have five minutes — I'd rather cover the eval flywheel than finish the context manager. Sound right?"*

**Name and defer:**
- "**Code retrieval** at monorepo scale is its own design — embeddings plus structural search plus call graphs."
- "**Multi-agent decomposition** (planner/coder/reviewer) is tempting and usually premature; error compounds across hops."
- "**Sandbox escape** is standard container security applied here, and it's a real surface."
- "**IDE integration vs async PR agent** are different products with different latency budgets."
- "**Trace privacy** — customer code in the trace store — is a real constraint for an enterprise deployment."
- "**Flaky tests** poison the reward signal; detecting and quarantining them is prerequisite work."

---

## 🧾 RUBRIC COVERAGE MAP — every pointer, as it applies to this question

**Codes:** **A** = System Architecture/Design · **F** = AI Fluency · **I** = AI Product Integration. The number is the sub-parameter, the letter the pointer (the form skips I1g). The form's pointers were written for R1; each row restates the pointer **for this question**. ⚠ A pointer is ticked only for what you **say**.

### A · System Architecture / Design

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **A1a** | A practical critical path: the loop and its bounds | §0, FU 4 | *"The engineering is in the loop and its bounds, not in the prompt."* |
| **A1b** | Latency and cost budgets | §1, §3 | *"30 turns × 20k tokens ≈ 600k input tokens a task — prefix caching is the lever."* |
| **A1c** | Degrade without lowering quality | §4, §6 | *"A run that hits a bound ends in a handoff with what it tried."* |
| **A2a** | The full stack | §4 diagram | Orchestrator → context manager → model → sandbox → verifier → PR → traces |
| **A2b** | Model components | §5.3 | Policy, multiplicative-gate reward, router, code retriever |
| **A2c** | Data sources | §5.1, FU 10 | Issue → PR pairs, fresh tasks, traces, review outcomes, CI flakiness |
| **A2d** | Staged pipeline | §5.5, FU 5 | Trace → filter → SFT → RL → gate → rollout |
| **A3a** | Top-down, unprompted | §0 | *"A deterministic control system around a stochastic core"* + the plan |
| **A3b** | A principled objective for the trade-off | §6, FU 7, 9 | *"Maximise pass^k subject to cost, diff size and zero test deletions."* |
| **A4a** | Clarify first, assumptions explicit | §1 | Tests? PR or commit? Cost cap? Whose code? |
| **A5a** | Alternatives compared | §4 | LLM judge vs tests; multi-agent vs one loop; one model vs cascade |
| **A5b** | A full alternative pipeline on demand | §9 | Suggestion mode, and characterisation tests first |

### F · AI Fluency

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **F1a** | The right model and loss per subproblem | §5.3 | *"Rejection-sampled SFT, then RL with gates × pass as the reward."* |
| **F1b** | Metrics and their trade-offs | §2, FU 1, 7 | *"pass@k says the capability exists; pass^k says it's reliable."* |
| **F1c** | Bias sources and mitigations | §5.2, §7, FU 6 | Contamination, benchmark-repo bias, language, zero-gradient tasks |
| **F2a** | Something fails → a fallback | §9 | Suggestion mode when there's no verifier |
| **F2b** | Context or latency too tight → design for it | §3, §4, FU 3 | The managed context budget, raised **before** being asked |
| **F2c** | Fine-tune an existing model | §5.4 | Rejection sampling → RL; LoRA for router and retriever |
| **F3a** | Efficiency with a quality trade | §6, FU 8 | Prefix caching; the small-model cascade |
| **F3b** | Scale levers | §3, §6 | Pre-warmed, autoscaled sandbox pool |
| **F3c** | Extension, with cost and maturity risk | §9, FU 10 | Untested monorepos; multi-agent; inline IDE |

### I · AI Product Integration

| Code | The pointer, for this question | Where | The line that earns it |
|---|---|---|---|
| **I1a** | Performance | §3, §8, FU 8 | Wall-clock; $ per solved task |
| **I1b** | Reliability | §6, §8 | Per-turn checkpoints; flaky tests quarantined |
| **I1c** | Usability | §2, §8 | Reviewable diffs; a useful handoff |
| **I1d** | Scalability | §3, §8 | Sandbox pool; per-team quotas |
| **I1e** | Privacy | §8 | Customer code in traces; secrets scrubbed; no training without consent |
| **I1f** | Safety | §8, FU 4, 9 | PR-only; human checkpoint for auth, payments, migrations |
| **I1h** | Security | §8, FU 2 | **Repo content is untrusted**; no egress; flagged dependencies; grader outside |
| **I1i** | Ethics / bias | §8 | Licence compliance; fair review load |
| **I1j** | Explainability | §2, §8 | Rationale + tests + trace on every PR; every stop says why |
| **I2a** | Offline → online | §7, FU 5, 6 | Fresh suite → shadow PRs → opt-in teams → GA by change class |
| **I2b** | The success definition | §7, FU 1 | *"Merged and not reverted, per engineer-hour, at a bounded cost."* |

**🧭 Probe map:** (a) assumptions → §1 · (b) product → §2 · (c) loop → §4 · (d) data → §5.1 · (e) suites → §5.2 · (f) models → §5.3–5.4 · (g) bounds → §6 · (h) success → §2, §7 · (i) extension → §9.
