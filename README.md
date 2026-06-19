# Berman's Loop Library — paste-ready `/loop` prompts for Claude Code

**🔗 Live site: https://az9713.github.io/berman-loop-library/**

22 repeatable AI-agent workflows from [Matt Berman / Forward Future's Loop
Library](https://signals.forwardfuture.ai/loop-library/), turned into
**copy-and-paste `/loop` prompts** you can drop straight into a Claude Code session.

Berman published the loop *concepts* (trigger → action → verify → stop). The missing
piece for practitioners was the actual runnable prompt text — that's what this provides.

## What's here

| File | What it is |
|------|------------|
| [`index.html`](index.html) | Filterable card grid of all 22 loops with one-click **Copy** buttons. Browse by domain, filter by setup tier / scheduling mode. **This is the site.** |
| [`berman-loops.md`](berman-loops.md) | The same content as a Markdown reference, with deeper notes on scheduling, self-pacing, and `/loop` vs `/goal`. |
| [`mockups.html`](mockups.html) | Three candidate layouts (card grid / sidebar / accordion) behind a tab switcher — kept for reference. |

## How to use a loop

1. Open the [site](https://az9713.github.io/berman-loop-library/) and find a loop.
2. Hit **Copy**.
3. Paste it into Claude Code and press Enter.

Each loop is tagged:

- **Tier A / B / C** — how much setup it needs (run anywhere → needs the right project → needs a placeholder or external tool).
- **Mode** — *self-paced* (runs until done, stops itself) vs *fixed interval* (`1d`, `30m`…).
- **⟳ Routines** — wants unattended/overnight running, so back it with a cloud Routine rather than an in-session `/loop`.

## `/loop` vs `/goal`, in one line

Self-paced `/loop` **waits between rounds and trusts itself** to decide it's done; `/goal`
**runs flat-out with a second model as the judge**. Use `/loop` when there's something
external to wait for; use `/goal` when you can state a clear pass/fail end condition. The
site and the Markdown doc both include a short decision tree.

---

*Loop concepts © Matt Berman / Forward Future. Prompt text reconstructed here for practitioner use. Built with Claude Code.*
