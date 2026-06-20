# `/loop` vs `/goal` — and when (not) to combine them

Two Claude Code commands keep a session working without you prompting each step. They look
similar, people mix them freely, and **most of those combos are redundant.** This explains the
real difference, the one combination that genuinely composes, and how to pick.

**Sources:**
- `/goal` — <https://code.claude.com/docs/en/goal>
- `/loop` & scheduled tasks — <https://code.claude.com/docs/en/scheduled-tasks>

---

## The core principle

`/goal` and self-paced `/loop` are **two answers to the same question** — *"keep working until
done"* — that differ on a **single axis**:

> **What starts the next turn?**
> - **`/goal`** → the previous turn finishing. Immediate, back-to-back, **no waiting**. A separate
>   fast model judges your condition after every turn and stops when it's met.
> - **`/loop`** → **a clock.** A fixed interval (`5m`, `1d`), or a self-paced 1m–1h gap Claude
>   picks. Built to **wait** for something between turns.

Everything else follows from that. If your reason to continue is *"there's more work to do right
now,"* that's `/goal`. If it's *"something external might have changed since last time,"* that's
`/loop`.

Two consequences worth remembering:
- **`/goal`'s evaluator can't run tools** — it only reads what Claude has surfaced in the
  transcript. So it can't *watch* or *poll* anything; it can only *judge*.
- **One `/goal` per session** (a new one replaces the old). You can have **many** `/loop`/cron tasks.

---

## The rule of thumb

| Your situation | Use | Why a combo is wrong here |
|---|---|---|
| Drive to a **checkable end state**, no reason to pause — coverage → 100%, tests green, queue empty | **`/goal` alone** | Adding `/loop` injects idle waits between bursts of work that should be continuous. It *slows you down.* |
| **Poll / wait** on external or time-based state — deploy finished? new CI failures? 3 pm yet? | **`/loop` alone** | `/goal` can't run tools or wait — it only reads the transcript, so it can't watch anything. |
| Same task, and you typed *both* "loop" and "goal" | **Pick one** | Redundant. Claude resolves natural language to a single control structure anyway. |

---

## The one combo that's real: time **outside**, condition **inside**

The only non-redundant nesting is `/loop <interval> /goal <condition>` — *"periodically drive to
this end state."* (`/loop` can take a command as its prompt, e.g. `/loop 20m /review-pr 1234`.)

It's meaningful **only when external change makes the goal re-relevant over time:**

```
/loop 1h /goal the open-bug queue labeled "ready" is empty
```

New tickets arrive through the day (the loop's **cadence**), and each hour you fully drain
whatever's there (the goal's **completion**). The loop supplies repetition; the goal supplies
"finish the batch."

The **reverse never composes.** `/goal … /loop …` isn't a thing: `/goal` takes a *condition*
(text the evaluator reads), not a command it can run. You can't put a loop "inside" a goal.

Even this valid combo is often better as a **Stop hook** (a durable, every-session `/goal`) or a
**Routine / cron task** — reach for the nest only when you want *this session* doing periodic
convergence.

---

## Running both at once — legitimately

Not nested — **parallel, on different targets:**

```
/goal migrate the auth module until all call sites compile and tests pass
/loop 10m check if the staging deploy went green and ping me
```

`/goal` *drives your work*; a separate `/loop` *watches an unrelated external thing*. Two
concerns, two mechanisms — fine.

> **The test:** same completion target → redundant, pick one. Different targets (drive vs. watch)
> → orthogonal, run both.

---

## Why the "combos" in tutorials look free-form

When someone says *"loop it until the goal is X"* in natural language, Claude doesn't spin up two
controllers — it **resolves the intent to a single mechanism** (usually one self-paced `/loop` or
one `/goal`). The mixing is harmless but mostly cosmetic; the extra keyword rarely adds a second
control loop. The skill isn't stacking them — it's picking the right *one*, and reserving the
`/loop … /goal …` nest for true "periodic drive-to-done."

---

## Quick mental compiler

| You say… | It's… |
|---|---|
| "until \<a condition I can check\>" | `/goal` |
| "every \<time\>" / "wait for \<external thing\>" | `/loop` |
| "every \<time\>, drive \<condition\> to done" | `/loop … /goal …` *(rare)* |
| both words, one task | you meant **one** |

---

## How this maps to the Loop Library

Most of [the 22 loops](../index.html) are `/loop` prompts: **self-paced** for any "stop when X"
goal, **fixed interval** for clock-based polling/maintenance. Several would work equally well —
or better — as a `/goal` (e.g. *100% Test Coverage* → `/goal coverage is 100% per the coverage
report`), because they have a crisp, checkable finish line and no reason to wait between turns.
See [`berman-loops.md`](../berman-loops.md) for the per-loop mode choices.
