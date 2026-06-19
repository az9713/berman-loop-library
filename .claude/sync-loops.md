You maintain a public mirror of Matt Berman's Loop Library as paste-ready `/loop` prompts for
Claude Code. Your job is to keep this repo in sync with the source as new loops are published.

## Task

1. Fetch the source: https://signals.forwardfuture.ai/loop-library/
   List every loop's **number** and **title**.

2. Compare that list to the loops already in this repo — the `LOOPS` array in `index.html`
   (currently 001–022). Detect changes by **number/title**, not by full prompt text (the source
   page renders summaries, so exact text isn't reliable).

3. **If there are NO new loops**, make NO changes and open NO pull request. Print a one-line
   summary like "No new loops; library still at NNN." and stop.

4. **If there are new loops** (e.g. 023+), for EACH new one:
   - Write a paste-ready `/loop` prompt in the SAME style as the existing entries. Embed the
     trigger / action / verification / stop condition in the prompt text.
   - Choose the mode: **self-paced** (no interval) for any "stop when X" goal; a **fixed
     interval** (`1d`, `30m`, …) only for clock-based polling/maintenance.
   - Assign a **Tier** (A = runs anywhere · B = needs the right project · C = needs a placeholder
     or external tool) and fit it into one of the existing **categories**.
   - Add a prerequisite note and a `⟳ Routines` flag if it wants unattended/overnight running.
   - Add the loop to BOTH `index.html` (the `LOOPS` array, with `id`, `name`, `cat`, `tier`,
     `mode`, `uatt`, `prompt`, `note`) AND `berman-loops.md` (its own section plus a row in the
     quick-index table). Keep the two files consistent.

5. Open a pull request on a new branch titled **"Sync new loops from Forward Future Loop
   Library"**. In the PR body:
   - Summarize which loops were added.
   - Add a review checklist: verbatim-text fidelity, tier accuracy, category fit, mode choice.
   Do **NOT** push to `main` directly — a human reviews the reconstructed prompts before they
   reach the public site.

## Report the outcome on the run summary

At the very end, append a one-line verdict to GitHub's run summary so the result is visible
without expanding logs. The `GITHUB_STEP_SUMMARY` environment variable holds a file path — append
a markdown line to it with a bash command, for example:

- No changes: `echo "✅ Checked source — N loops found, none new. No changes." >> "$GITHUB_STEP_SUMMARY"`
- New loops:  `echo "🔔 Added loops NNN–MMM and opened PR #X." >> "$GITHUB_STEP_SUMMARY"`
- Source unreachable: `echo "⚠️ Could not fetch/parse the source page. No changes made." >> "$GITHUB_STEP_SUMMARY"`

Fill in the real numbers/PR link. Do this on every run, whatever the outcome.

## Rules

- Prefer **additive** changes. Never delete or rewrite an existing prompt unless the source has
  clearly removed or renamed that loop.
- If the source page can't be fetched or parsed, do nothing and report the failure — do not open
  a speculative or empty PR.
- Match the existing tone and formatting exactly; this content is published on a public site.
