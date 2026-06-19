# GitHub Actions, explained — with a self-updating repo as the worked example

A from-scratch tutorial on GitHub Actions, then a complete real workflow: a scheduled
agent that checks [Matt Berman's Loop Library](https://signals.forwardfuture.ai/loop-library/)
for new loops and opens a pull request to add them to this repo.

---

## Part 1 — GitHub Actions in general

### What it is

**GitHub Actions** is automation that lives inside your repository. You describe *"when X
happens, run these steps on a fresh computer,"* and GitHub provides the computer and runs
them. No server to maintain — GitHub hosts the machines (called **runners**).

Common uses: run tests on every push, lint a PR, publish a release, deploy a site, or — as
here — run a scheduled maintenance job.

### The vocabulary (the whole model in seven words)

```
Workflow ── triggered by ──> Event
   └── contains Jobs ── run on ──> Runners
          └── made of Steps ── each is a ──> shell command (run:) or an Action (uses:)
```

| Term | What it is |
|------|-----------|
| **Workflow** | A `.yml` file in `.github/workflows/`. One file = one automation. A repo can have many. |
| **Event / trigger** (`on:`) | What starts the workflow: a `push`, a `pull_request`, a `schedule` (cron), or a manual `workflow_dispatch` button. |
| **Job** | A named unit of work. A workflow has one or more; by default they run in parallel. |
| **Runner** (`runs-on:`) | The throwaway VM the job runs on, e.g. `ubuntu-latest`. Starts clean every time, then is destroyed. |
| **Step** | One thing inside a job, done top-to-bottom. Either a shell command (`run:`) or a prebuilt **Action** (`uses:`). |
| **Action** | A reusable step someone published, referenced as `owner/name@version` — e.g. `actions/checkout@v4` clones your repo onto the runner. |
| **Secret** | An encrypted value (API key, token) stored in repo settings, read as `${{ secrets.NAME }}`. Never hard-code keys. |
| **`permissions:`** | What the job's automatic `GITHUB_TOKEN` is allowed to do (e.g. write contents, open PRs). Least-privilege by default. |

### A minimal workflow

```yaml
name: Hello                      # shows up in the Actions tab
on: [push]                       # event: run on every push
jobs:
  greet:                         # job id
    runs-on: ubuntu-latest       # runner
    steps:
      - run: echo "Hi from CI"   # a step that runs a shell command
```

Commit that to `.github/workflows/hello.yml`, push, and the **Actions** tab shows a run.

### Two step styles

```yaml
steps:
  - uses: actions/checkout@v4    # run a published Action (here: clone the repo)
  - run: npm test                # run a shell command on the runner
```

`uses:` pulls in someone's reusable logic; `run:` is your own commands. Most workflows mix both.

### How events fire

- **`push` / `pull_request`** — code-change driven (the usual CI).
- **`schedule`** — cron, in **UTC**. `"0 9 * * 1"` = 09:00 UTC every Monday.
  *Caveat:* scheduled runs can be delayed minutes under load, and GitHub disables schedules in
  repos with no activity for 60 days. They are best-effort, not real-time.
- **`workflow_dispatch`** — adds a **Run workflow** button in the Actions tab so you can trigger
  it by hand. Always include this on scheduled workflows so you can test without waiting for the clock.

### Where it runs / what it costs

Jobs run on GitHub-hosted runners. For **public repos, Actions minutes are free**; private repos
get a monthly free allotment. (Our repo is public → the runner is free; the only cost here is the
Claude API tokens the agent uses.)

---

## Part 2 — The worked example: a self-updating repo

**Goal:** once a week, check Berman's page. If it lists loops we don't have yet (he numbers them
sequentially — we're at 001–022), write paste-ready prompts for the new ones and **open a PR**.
If nothing's new, do nothing.

This needs an agent that can read the web, edit files, and open a PR — that's exactly what the
**Claude Code Action** does inside a workflow.

### The pieces we'll add

```
.github/workflows/sync-loops.yml   ← the workflow (when + how to run the agent)
.claude/sync-loops.md              ← the agent's instructions (what to actually do)
```

Keeping the instructions in their own file (rather than inline in the YAML) means you can edit
the task without touching the workflow, and the instructions are versioned in git.

### The workflow file

```yaml
name: Sync Loop Library

on:
  schedule:
    - cron: "0 9 * * 1"      # every Monday 09:00 UTC
  workflow_dispatch:          # ...and a manual "Run workflow" button

# What the job's GITHUB_TOKEN may do. The agent commits + opens a PR (contents,
# pull-requests) and the action fetches an OIDC token during auth (id-token).
permissions:
  contents: write
  pull-requests: write
  id-token: write

# Don't let two syncs run at once.
concurrency:
  group: sync-loops
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4          # clone our files onto the runner

      - name: Run the sync agent
        uses: anthropics/claude-code-action@v1
        with:
          # Auth via your Claude Max subscription — NO per-token API billing.
          # (Swap for `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}` to bill the API instead.)
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: "Read .claude/sync-loops.md and follow it exactly."
          claude_args: "--max-turns 40"     # safety cap on agent iterations
```

**Line by line:**

- `on.schedule.cron` — the weekly trigger (UTC). `on.workflow_dispatch` — the manual button.
- `permissions` — grants the run's built-in `GITHUB_TOKEN` write access to files and PRs (so the
  agent can push a branch and open the pull request) plus `id-token: write` (the action fetches an
  OIDC token during auth). Miss `id-token` and the run fails with *"Could not fetch an OIDC token."*
- `concurrency` — if a previous run is still going, don't start a second; avoids duplicate PRs.
- `actions/checkout@v4` — **required** because the agent needs our repo's files (and the
  instruction file) present on the runner.
- `anthropics/claude-code-action@v1` — runs Claude Code. Because we pass a `prompt`, it runs in
  **automation mode** (acts immediately) rather than waiting for an `@claude` mention.
  - `claude_code_oauth_token` — auth via your **Claude Max subscription** (generated by
    `claude setup-token`), so runs draw on your Max quota instead of metered API billing. The
    `anthropic_api_key` input is the pay-per-token alternative.
  - `prompt` — points the agent at our instruction file. (It could also be inline text, or a
    repo skill like `/sync-loops` if we packaged one under `.claude/skills/`.)
  - `claude_args` — passthrough CLI flags. `--max-turns 40` bounds how long it can iterate so a
    confused run can't loop forever (and rack up tokens).

### The instruction file (`.claude/sync-loops.md`)

```markdown
You maintain a public mirror of Matt Berman's Loop Library as paste-ready /loop prompts.

1. Fetch https://signals.forwardfuture.ai/loop-library/ and list every loop's NUMBER and TITLE.
2. Compare that list to the loops already in this repo — the `LOOPS` array in `index.html`
   (currently 001–022). Detect by number/title, not full text.
3. If there are NO new loops, make NO changes and open NO pull request. Say "no changes" and stop.
4. If there are new loops (023+), for EACH new one:
   - Write a paste-ready `/loop` prompt in the SAME style as the existing entries: embed the
     trigger/action/verification/stop condition; choose self-paced vs a fixed interval; assign a
     Tier (A/B/C) and one of the existing categories; add a prerequisite/Routines note if needed.
   - Add it to the `LOOPS` array in `index.html` AND to `berman-loops.md` (including its quick-index
     table), keeping the two files consistent.
5. Open a pull request on a new branch titled "Sync new loops from Forward Future Loop Library".
   In the PR body, summarize what changed and add a review checklist: verbatim-text fidelity,
   tier accuracy, category fit. Do NOT push to main directly.
6. Never delete or rewrite existing prompts unless the source clearly removed/renamed a loop.
   Prefer additive changes.
```

The agent reads this, does the work with its normal tools (web fetch, file edits, git), and the
GitHub app token lets it open the PR.

---

## Part 3 — Setup, step by step

You do this once. **You must be a repo admin** (you are — it's your repo).

### 1. Install the Claude GitHub App

The quickest way, from a Claude Code terminal in this repo:

```
/install-github-app
```

It walks you through installing the app and adding the secret. Manual alternative: install from
<https://github.com/apps/claude> and grant it **Contents**, **Issues**, and **Pull requests**
(read & write) on this repo.

### 2. Add an auth token as a secret

**Recommended (Claude Max — no API billing):** generate a subscription OAuth token and store it.
Usage counts against your Max plan, not a metered API account.

```bash
claude setup-token                      # run locally; logs in via your Max subscription
gh secret set CLAUDE_CODE_OAUTH_TOKEN   # paste the token; stored encrypted
```

Or in the browser: **repo → Settings → Secrets and variables → Actions → New repository secret**,
name `CLAUDE_CODE_OAUTH_TOKEN`.

**Alternative (pay-per-token):** if you'd rather bill the Anthropic API, store `ANTHROPIC_API_KEY`
(a key from <https://console.anthropic.com>) instead, and use the `anthropic_api_key` input in the
workflow. This is *separate* from your Max subscription and consumes API credit.

### 3. Commit the two files

```bash
git add .github/workflows/sync-loops.yml .claude/sync-loops.md
git commit -m "Add weekly Loop Library sync workflow"
git push
```

Once on the default branch, the schedule is live and the manual button appears.

---

## Part 4 — Running, watching, reviewing

### Trigger it by hand (don't wait for Monday)

In the browser: **Actions tab → Sync Loop Library → Run workflow**.

From the terminal with the `gh` CLI:

```bash
gh workflow list                       # see workflows and their IDs
gh workflow run sync-loops.yml         # trigger the workflow_dispatch run
gh run list --workflow=sync-loops.yml  # list recent runs + status
gh run watch                           # live-tail the latest run
gh run view --log                      # full logs of the latest run
```

### Review the result

- **No new loops** → the run succeeds and does nothing. Most weeks.
- **New loops found** → a **pull request** appears. Open it, read the reconstructed prompts
  (they're *our* wording, not Berman's verbatim), check the tier/category, then merge or tweak.
  Merging to `main` auto-rebuilds the GitHub Pages site.

This human-in-the-loop PR is deliberate: an unattended agent should never publish unreviewed
content straight to a public site.

---

## Part 5 — Costs, safety, and gotchas

- **Cost** = GitHub Actions minutes (free on this public repo) + model usage per run. With the
  `CLAUDE_CODE_OAUTH_TOKEN` (Max) path there's **no per-token API charge** — runs draw on your Max
  quota, and a weekly mostly-no-op sync barely touches it. (The `ANTHROPIC_API_KEY` path bills the
  API per token instead.) `--max-turns` caps the worst case either way.
- **Schedules are best-effort** — UTC only, can lag under load, and auto-pause after 60 days of
  repo inactivity. The manual button is your reliable fallback.
- **Secrets are write-only** — once set you can't read a secret back, only overwrite it. They're
  masked in logs.
- **Least privilege** — the workflow grants only `contents: write` and `pull-requests: write`.
  It can't, say, change repo settings or delete the repo.
- **PR, not push** — the agent proposes; you decide. Keep it that way for anything public-facing.
- **Pin third-party actions to a commit SHA** — `uses: owner/action@<40-char-sha>` instead of a
  mutable tag like `@v1`. A tag can be repointed at malicious code; a SHA can't. This matters most
  when the step receives a secret (ours gets the auth token). Re-pin periodically after reviewing
  changelogs — Dependabot can automate the bumps. Our `sync-loops.yml` pins the action this way.

---

## Cheat-sheet

| You want to… | Do this |
|--------------|---------|
| Add an automation | Create `.github/workflows/<name>.yml` |
| Run on a clock | `on: schedule: - cron: "0 9 * * 1"` (UTC) |
| Add a manual button | `on: workflow_dispatch:` |
| Clone your code onto the runner | step `uses: actions/checkout@v4` |
| Run a shell command | step `run: <command>` |
| Use a published Action | step `uses: owner/name@version` |
| Store a key safely | repo Settings → Secrets, read as `${{ secrets.NAME }}` |
| Auth CI on your Max plan (no API bill) | `claude setup-token` → secret `CLAUDE_CODE_OAUTH_TOKEN` → `claude_code_oauth_token:` input |
| Let the job open a PR | `permissions: { contents: write, pull-requests: write }` |
| Run Claude in CI | `uses: anthropics/claude-code-action@v1` with `prompt` + an auth token |
| Trigger from terminal | `gh workflow run <file.yml>` |
| Watch a run | `gh run watch` / `gh run view --log` |
