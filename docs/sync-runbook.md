# Sync-Loops runbook — how to run & manage the GitHub Action

A quick reference for operating the **Sync Loop Library** workflow in this repo. Each task lists
the **browser** way (click path) and the **terminal** way (`gh` command). Run terminal commands
from inside the repo folder. Install the CLI once from <https://cli.github.com/> if you don't have it.

| Fact | Value |
|------|-------|
| Workflow name | **Sync Loop Library** |
| Workflow file | `.github/workflows/sync-loops.yml` |
| Schedule | Weekly — Monday 09:00 UTC (+ manual button) |
| Auth secret | `CLAUDE_CODE_OAUTH_TOKEN` (your Max token) |
| Task instructions | `.claude/sync-loops.md` |

---

## ▶️ Run it now (manual trigger)

**Browser:** repo → **Actions** tab → left sidebar **Sync Loop Library** → **Run workflow** button
(top-right) → keep branch `main` → **Run workflow**.

**Terminal:**
```
gh workflow run sync-loops.yml
```

---

## 👀 Watch / peek into a run

**Browser:** **Actions** tab → click the running entry → **All jobs → sync** → click the
**Run the sync agent** step to expand its live log.

**Terminal:**
```
gh run watch                      # live-tail the most recent run until it finishes
gh run list --workflow=sync-loops.yml   # list recent runs + status (grab the ID)
gh run view --log                 # full log of the latest run
gh run view <run-id> --log        # full log of a specific run
```

---

## ✅ See the result at a glance

Every run prints a one-line verdict on its **Summary** tab (e.g. "✅ 22 loops, none new"):

**Browser:** **Actions** tab → click the run → read the top of **Summary**.

**Terminal:**
```
gh run view                       # shows status + conclusion of the latest run
```

---

## 🔔 Did it find new loops? (check for a PR)

When new loops appear, the run opens a pull request instead of changing the site directly.

**Browser:** repo → **Pull requests** tab.

**Terminal:**
```
gh pr list                        # open PRs
gh pr view <number>               # read one
gh pr checkout <number>           # pull it locally to inspect/edit
gh pr merge <number> --squash     # merge it (rebuilds the Pages site)
```

---

## ⏸️ Pause / ▶️ resume the weekly schedule

**Browser (pause):** **Actions** tab → **Sync Loop Library** → **•••** (top-right) → **Disable workflow**.
Re-enable from the same menu.

**Terminal:**
```
gh workflow disable sync-loops.yml
gh workflow enable  sync-loops.yml
```

> Note: *disable* stops everything including the manual button. To keep the button but stop only the
> weekly auto-run, instead comment out the `schedule:` block in `.github/workflows/sync-loops.yml`
> and push.

---

## 🕐 Change how often it runs

Edit the cron line in `.github/workflows/sync-loops.yml` (times are **UTC**) and push:

```yaml
on:
  schedule:
    - cron: "0 9 * * 1"     # current: Mondays 09:00 UTC
```

| Want | cron |
|------|------|
| Daily 6am UTC | `0 6 * * *` |
| Twice weekly (Mon & Thu) | `0 9 * * 1,4` |
| 1st of each month | `0 9 1 * *` |

---

## 🚑 If a run fails

On failure you get a GitHub **email** and a new **issue** with a link to the run. To diagnose:

```
gh run view --log            # read the failing step's output
```

Common causes:

| Symptom in the log | Fix |
|--------------------|-----|
| `Could not fetch an OIDC token` | `id-token: write` missing from `permissions:` (already set). |
| Auth / 401 / invalid token | Token expired → regenerate: `claude setup-token`, then `gh secret set CLAUDE_CODE_OAUTH_TOKEN`. |
| Can't create PR / 403 | Claude GitHub App not installed on this repo → reinstall: <https://github.com/apps/claude>. |
| Opened a junk/empty PR | Shouldn't happen (instructions forbid it) — close the PR and tell Claude to tighten `.claude/sync-loops.md`. |

---

## 🔧 One-time setup (only if starting fresh or after a wipe)

```
claude setup-token                     # generate a Max-subscription token (no API billing)
gh secret set CLAUDE_CODE_OAUTH_TOKEN   # paste it when prompted
gh secret list                          # verify it's stored
```
Then install the Claude GitHub App on this repo: <https://github.com/apps/claude>
(grant Contents, Issues, Pull requests).

---

## ✏️ Change what the sync does

The agent's instructions live in **`.claude/sync-loops.md`** (plain English). Edit and push — the
next run uses the new version. The workflow file (`sync-loops.yml`) only controls *when/how* it runs;
the `.md` controls *what work it does*.
