# Berman's Loop Library → paste-ready `/loop` prompts

22 loops from [Matt Berman's Loop Library](https://signals.forwardfuture.ai/loop-library/),
each rewritten as a command you can paste straight into Claude Code. Prompts are
reconstructed faithfully from the library summaries — tune the wording to your project.

> **Pasting note:** the prompts below are hard-wrapped onto several lines for easy
> reading. They paste fine as a single command — terminals use bracketed paste, so
> the line breaks come in as one block; just press Enter once.

---

## Three ways to schedule

|                            | Cloud (Routines)   | Desktop task        | `/loop` (in-session) |
| :------------------------- | :----------------- | :------------------ | :------------------- |
| Runs on                    | Anthropic cloud    | Your machine        | Your machine         |
| Requires machine on        | No                 | Yes                 | Yes                  |
| Requires open session      | No                 | No                  | **Yes**              |
| Persists across restarts   | Yes                | Yes                 | Restored on `--resume` if unexpired |
| Access to local files      | No (fresh clone)   | Yes                 | Yes                  |
| MCP servers                | Per-task config    | Config + connectors | Inherits session     |
| Permission prompts         | No (autonomous)    | Configurable        | Inherits session     |
| Minimum interval           | 1 hour             | 1 minute            | 1 minute             |

### When to use which

- **`/loop`** — quick polling and iterate-until-done work *while you're sitting in the
  session*. Fastest to start, inherits your project, tools, and permissions. But it only
  fires while the session is open and idle, has **no catch-up** for missed fires, and
  **recurring tasks expire after 7 days**. Perfect for testing any loop here, and for
  "run until X" work in one sitting.
- **Routines (cloud)** — anything that must run **unattended / overnight** without your
  machine on. Survives restarts, no 7-day limit. Trade-off: runs on a fresh clone (no
  access to your local working tree) and needs per-task MCP/connector setup. Use for the
  nightly loops below.
- **Desktop task** — overnight/recurring work that **needs your local files or tools**
  (the thing Routines can't see). Machine must be on, but no session required.

> Rule of thumb: testing → `/loop`. Unattended + cloud-friendly → Routines. Unattended +
> needs local files → Desktop task.

---

## `/loop` cheat-sheet

- `/loop 5m <prompt>` — **fixed interval**. Runs every 5m; **never self-stops**. Units `s/m/h/d`.
- `/loop <prompt>` — **self-paced**. Claude picks the gap (1m–1h) each round and **stops itself
  when the goal is provably met**. Use this for every "stop when X" loop.
- `/loop 1d <prompt>` — daily. Runs ~once/day for ~7 days, then expires.
- **Clock times** (e.g. 3am) aren't a `/loop` option — `/loop` only does intervals. Ask in
  natural language instead: *"every day at 3am, <do the thing>"* → Claude schedules it via cron.
- **Jitter:** a daily `0 3 * * *` can fire up to 3:30. Want it tight? Phrase it as 3:03am.
- Stop a waiting `/loop` with **`Esc`**.

---

## What `/loop` without an interval actually means

Omitting the interval does **not** mean "loop constantly." It means **self-paced**: you
decide nothing about cadence, and Claude stops itself when the job is done. Each round, Claude:

1. Runs the prompt (does a chunk of real work),
2. Looks at what it found,
3. **Picks its own delay** before the next round (1 minute to 1 hour),
4. **Ends the loop entirely** once the goal is provably met — no next wakeup is scheduled.

So there's no fixed cadence and no "runs forever." It's a worker that keeps going until the
job is done, pacing itself based on whether work remains.

Contrast with `/loop 30m …` (**fixed interval**): that fires every 30 minutes mechanically,
*never* decides it's finished, and only stops on `Esc` or the 7-day expiry.

**Two examples from the loops below:**

- **005 · 100% Test Coverage** — Round 1: coverage 72% → add tests for the biggest gap →
  re-run. Gaps remain, so Claude goes straight into the next round (nothing to wait for).
  Rounds 2–4 climb the number. Coverage hits 100% → **Claude stops scheduling and the loop
  ends.** A fixed interval couldn't do this — it can't know when coverage is "done"; only
  self-paced can self-terminate.
- **003 · Sub-50ms Page-Load** — each round: measure → optimize the slowest path → re-measure.
  Tight gaps while pages are still >50ms; once all pages are under 50ms, it stops.

### How Claude picks the delay between rounds

There's **no fixed formula** — it's the model's judgment each round, and it prints the chosen
delay *and the reason* at the end of every iteration (clamped to **1 minute–1 hour**). The one
question it answers is: *"When is there likely to be new state worth acting on?"* That makes
the cadence genuinely task-dependent:

- **Local grind, nothing to wait for** (005 Coverage, 011 Test-Suite Speed) — the next chunk
  can start immediately, so gaps are negligible; it essentially loops straight back until done.
- **Polling external async state** (004 Error Sweep, a CI/PR loop) — the gap matches *how fast
  that state changes*: short while a build runs or a PR is active, stretching out once things
  go quiet. No point re-checking every 60s if the source updates hourly.
- **Waiting on a rough ETA** (a deploy/build that takes ~8 min) — it waits roughly that long
  instead of burning rounds checking every minute.

Two caveats:

1. **It's heuristic, not deterministic.** The same task can get slightly different gaps run to
   run. Want a specific cadence? Say so in the prompt (*"check roughly every 2 minutes"*) or use
   the fixed-interval form (`/loop 2m …`).
2. **For polling, Claude may skip the guessing** and use the **Monitor tool** — it runs a
   background script and streams each output line back as it appears, so it reacts the moment
   the watched thing emits output instead of waiting a guessed interval. More responsive and
   more token-efficient than interval polling.

**Rule of thumb:** any loop whose prompt says **"Stop when X"** wants the no-interval
(self-paced) form — it's the only mode that can actually *reach* the stop condition and quit.
Fixed intervals (`1d`, `30m`) are for work that should just keep polling on a clock, like the
nightly and monitoring loops.

> **Bare `/loop`** (no interval *and* no prompt) is also self-paced, but it runs Claude Code's
> built-in maintenance prompt — finish unfinished work → tend the branch's PR → cleanup
> passes — instead of one you typed.

---

## `/loop` (self-paced) vs `/goal` — two ways to "keep going until done"

Both keep the session working across turns without you re-prompting, and both can stop on
their own — so they look like the same tool. They aren't. The difference is **what starts the
next turn** and **who decides you're finished**.

| | `/loop <prompt>` (self-paced) | `/goal <condition>` |
|---|---|---|
| Next turn starts | after a **delay** Claude picks (1m–1h) | **immediately**, back-to-back, no gap |
| You give it | a **prompt** (the action), re-run each round | a **condition** (the end state); Claude derives the steps |
| Decides it's done | the **working model** stops scheduling | a **separate evaluator** model (Haiku) checks after every turn |
| Built for | **polling / waiting** on state that changes over time (CI, a deploy, logs) | **cranking** continuous work to a verifiable finish, fast |
| Stops when | work done, or you press `Esc` | evaluator confirms the condition, or `/goal clear` |
| Bound / expiry | recurring expires after 7 days | no interval; add "or stop after N turns" to the condition |
| Needs | — | v2.1.139+, workspace trust (it's a Stop hook under the hood) |

**Two questions decide it:**

1. **Is the work waiting on something external?** (a build/CI, a deploy, logs to appear, a time
   of day) → **`/loop`**. Poll on an interval; if it's nightly/unattended, back it with a Routine.
2. If not — **can you state a clear pass/fail end condition?** (tests green, coverage 100%,
   queue empty, file under N lines) → **`/goal`** (runs back-to-back, a second model verifies).
   - **No crisp finish line?** → **`/loop` (self-paced)** — let it iterate and stop itself.

**The one-line distinction:** self-paced `/loop` **waits between rounds and trusts itself** to
decide it's done; `/goal` **runs flat-out with no waiting and a second model as the judge**.

Why the judge matters: with `/loop`, the same model doing the work also decides when to stop —
convenient, but it can declare victory early. `/goal` hands the yes/no to a *fresh* evaluator
that only reads what Claude has surfaced in the transcript, so "done" means *demonstrated*, not
asserted. Write the condition so the proof lands in the chat — e.g. *"`npm test` exits 0 and
`git status` is clean"* — because the evaluator can't run commands itself.

**Picking between them for these loops:**
- **Waiting on external/async state** (004 Error Sweep, anything polling CI or a deploy) →
  `/loop`. There's literally something to wait *for*; the gap is the point.
- **Local grind to a checkable end state** (005 Coverage → 100%, 011 faster suite, 002
  architecture) → either works, but **`/goal` is usually tighter**: no inter-round delay and an
  independent check on "is it really done." Phrase the stop condition as the goal:
  `/goal coverage is 100% per the coverage report`.
- **Periodic maintenance with no end** (001/008 nightly) → neither; that's a fixed interval or
  a Routine.

> Related: a **Stop hook** is the durable, every-session version of `/goal` (lives in settings,
> can run a script for deterministic checks). **Auto mode** removes per-*tool* prompts within a
> turn; `/goal` removes per-*turn* prompts — they compose.

---

## Tier key

- **A** — paste and run almost anywhere; minimal setup.
- **B** — prompt is complete, but only useful **inside the right project** (a repo with
  tests / logs / a built site / git history). Run it from that directory.
- **C** — needs a **placeholder filled** (`[N]`, `[LEVEL]`…) or an **external tool/service**
  (Slack, a DB, audio generation, a code-review tool). Won't run verbatim.

---

# The 22 loops

## 001 · Overnight Docs Sweep — *Tier B · Routines recommended*
Keep docs in sync with code, nightly.
```
/loop 1d Review the codebase for documentation that no longer matches
recent code changes. Update the affected docs, then open a pull request.
Stop when the documentation matches the current implementation.
```
**Unattended:** this is a nightly job — run it via Routines so it fires without an open session.
Natural language: *"every night at 3am, review yesterday's code changes, update any stale docs, and open a PR."*

## 002 · Architecture Satisfaction — *Tier B*
Iterative refactor with test + self-review after each step.
```
/loop Refactor the codebase one step at a time. After each change, run the
tests and self-review the diff. Track progress in ARCHITECTURE_PROGRESS.md.
Stop when the architecture is satisfactory and all checks pass.
```

## 003 · Sub-50ms Page-Load — *Tier B*
Performance loop with a hard target.
```
/loop Measure page load times under consistent, repeatable conditions.
Find the slowest path, optimize it, and re-measure — no regressions.
Stop when every page loads in under 50ms.
```

## 004 · Production Error Sweep — *Tier C (logs + Slack) · Routines recommended*
Triage prod errors, fix, notify.
```
/loop 30m Review production logs for actionable errors. For each, trace the
root cause, apply a fix, and post a summary to Slack with links to the PRs.
Stop when the errors are fixed or the logs are clean.
```
**Needs:** read access to production logs + a Slack MCP/connector.
**Unattended:** monitoring should run when you're away → Routines with the log + Slack connectors configured.

## 005 · 100% Test Coverage — *Tier B*
Close coverage gaps until full.
```
/loop Run the coverage report. Find the largest gap, add tests for it, and
re-run. Treat the coverage report as the source of truth. Stop when
coverage reaches 100%.
```

## 006 · SEO/GEO Visibility — *Tier B (live site)*
Audit and fix discoverability by impact.
```
/loop Audit the site for crawlability, indexation, content structure, and
search rankings. Prioritize fixes by impact, apply the top one, and re-run
the benchmark. Stop when priority pages are indexable, answer-ready, and
technically sound.
```

## 007 · Logging Coverage — *Tier B*
Add tested logs to silent paths.
```
/loop Identify important code paths that lack logging. Add useful, tested
log statements one path at a time and verify them. Stop when every
important path emits useful, tested logs.
```

## 008 · Nightly Changelog — *Tier B · Routines recommended*
Daily changelog from yesterday's commits.
```
/loop 1d Review the previous day's code changes. For each user-relevant
change, add an entry to CHANGELOG.md. Stop when every user-relevant change
from the previous day is accounted for.
```
**Unattended:** nightly → Routines. Natural language: *"every day at 3am, update CHANGELOG.md from yesterday's commits."*

## 009 · Quality Streak — *Tier C (fill `[N]`)*
Run scenarios until a clean streak.
```
/loop Run a realistic end-to-end scenario. If it fails, document the
failure and add a regression test that guards against it. Stop after [N]
consecutive successful scenarios.
```

## 010 · Full Product Evaluation — *Tier C (fill `[N]`)*
Broad capability sweep against a quality bar.
```
/loop Create and run [N] realistic scenarios covering all major
capabilities, each with defined success criteria. Fix any failures and
re-run the affected scenarios. Stop when every scenario meets the defined
quality bar.
```

## 011 · Test-Suite Speed — *Tier B*
Faster tests, same behavior.
```
/loop Profile the test suite for the slowest tests, speed them up without
reducing coverage or changing behavior, and re-time the suite. Stop when it
is meaningfully faster with no coverage or behavior regression.
```

## 012 · Repository Cleanup — *Tier B (git repo)*
Make repo state intentional.
```
/loop Inspect branches, pull requests, commits, and worktrees. Recover any
valuable unmerged work and remove stale items, showing evidence for each
removal. Stop when the repository state is intentional.
```

## 013 · Stale-Safe Batch Release — *Tier B*
Ship only complete changes.
```
/loop Review all pending changes, exclude anything incomplete, and combine
the valid, complete changes into a single release. Stop when only current,
complete changes are staged to ship.
```

## 014 · Production Data Cleanup — *Tier C (DB access)*
Purge invalid records, improve classification.
```
/loop Review production records against the allowed definition. Remove
invalid records and improve the classification logic that let them through.
Stop when every remaining record meets the allowed definition.
```
**Needs:** access to the production datastore (MCP/connector or CLI).

## 015 · Post-Release Baseline — *Tier B · Routines/event recommended*
Benchmark after release, record baseline.
```
/loop 1d Run the standard benchmark suite and record the results as the new
baseline, including revision, environment, and test conditions.
```
**Better as an event trigger:** fire this *after a release* rather than on a timer — a Routine on a GitHub release event, or natural language: *"after each deploy to main, run the benchmark suite and save the results as the new baseline."*

## 016 · Ticket-to-PR-Ready — *Tier C (fill `[TICKET]`) · usually one-shot*
Bug report → review-ready patch.
```
/loop Take bug report [TICKET]. Reproduce the failure, isolate the root
cause, and apply the smallest credible fix. Produce: failure summary,
reproduction steps, root cause, fix summary, files changed, verification
proof, risks, follow-ups, suggested PR title, and PR description draft.
Stop when the patch is review-ready.
```
**Note:** this is really a single pass — self-paced `/loop` will stop after one ticket. For a queue, point it at "the next open bug labeled X" and let it loop.

## 017 · Customer AI Deployment — *Tier C (customer infra)*
Drive a customer workflow goal → production.
```
/loop For customer workflow [NAME], define the business goal, build or
update the deployment, run a dry run, then monitor production. Produce a
customer-facing update and reusable lessons. Stop when the deployment meets
its business goal in production.
```

## 018 · Product Update Podcast — *Tier C (audio generation) · Routines recommended*
Nightly feature-update episode.
```
/loop 1d Identify meaningful product changes from the last day and generate
a 3–5 minute podcast episode explaining the new features to users. Verify
the script against the source material for accuracy.
```
**Needs:** an audio-generation tool/MCP. **Unattended:** nightly → Routines.

## 019 · Clodex Adversarial-Review — *Tier C (Clodex + `[LEVEL]`)*
Review PR to a severity bar, iterate.
```
/loop Run a Clodex adversarial code review on the current pull request at
severity threshold [LEVEL]. Fix every finding at or above the threshold and
re-run. Stop when the pull request reaches the configured review bar.
```
**Needs:** the Clodex review tool configured.

## 020 · Loop Harness Verification — *Tier C (advanced) · Routines recommended*
Work in an isolated worktree; a separate pass verifies before shipping.
```
/loop 1h Do the next unit of work in an isolated git worktree. Then, in a
separate verification pass, check the output against the defined criteria.
Ship only output that is independently verified.
```
**Note:** Berman's original is a real harness (scheduler + isolated worker session + separate verifier). This single-prompt version approximates it; for the full pattern use Routines/Desktop to wake it on a cadence with git worktree isolation.

## 021 · Boeing 747 Benchmark — *Tier B (Three.js project)*
Model until visually convincing via a fixed camera rig.
```
/loop Improve the Three.js model of a Boeing 747. After each change, render
the standard camera-inspection views and compare against reference photos.
Stop when you are 100% satisfied with the visual realism.
```

## 022 · War Loops: Autonomous Frontend Designer — *Tier C (design capture)*
Clone a design, build, and score fidelity.
```
/loop Capture the design from [URL or image]. Produce both a static and a
coded build, then measure each against the original across design, motion,
and responsiveness. Stop when the builds match the reference across every
measured fidelity axis.
```
**Needs:** a way to capture the reference (browser/screenshot MCP or a provided image).

---

## Quick index

| # | Loop | Tier | Mode | Unattended → |
|--:|------|:----:|------|--------------|
| 001 | Overnight Docs Sweep | B | `1d` | Routines |
| 002 | Architecture Satisfaction | B | self-paced | — |
| 003 | Sub-50ms Page-Load | B | self-paced | — |
| 004 | Production Error Sweep | C | `30m` | Routines |
| 005 | 100% Test Coverage | B | self-paced | — |
| 006 | SEO/GEO Visibility | B | self-paced | — |
| 007 | Logging Coverage | B | self-paced | — |
| 008 | Nightly Changelog | B | `1d` | Routines |
| 009 | Quality Streak | C | self-paced | — |
| 010 | Full Product Evaluation | C | self-paced | — |
| 011 | Test-Suite Speed | B | self-paced | — |
| 012 | Repository Cleanup | B | self-paced | — |
| 013 | Stale-Safe Batch Release | B | self-paced | — |
| 014 | Production Data Cleanup | C | self-paced | — |
| 015 | Post-Release Baseline | B | `1d` | Routines/event |
| 016 | Ticket-to-PR-Ready | C | self-paced | — |
| 017 | Customer AI Deployment | C | self-paced | — |
| 018 | Product Update Podcast | C | `1d` | Routines |
| 019 | Clodex Adversarial-Review | C | self-paced | — |
| 020 | Loop Harness Verification | C | `1h` | Routines |
| 021 | Boeing 747 Benchmark | B | self-paced | — |
| 022 | Autonomous Frontend Designer | C | self-paced | — |
