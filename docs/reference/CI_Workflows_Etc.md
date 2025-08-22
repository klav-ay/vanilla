# CI workflows, checks and GitHub verification (FRI 22nd Aug 2025)

This document records the CI/workflow findings for this repository and provides exact commands you can run locally (or I can run if you provide a short-lived PAT) to inspect branch protection and workflow status on GitHub.

## Quick plan

- Summarize workflows found locally.
- Show which checks can be done locally vs which require GitHub API access.
- Provide exact `gh` and `curl` commands to inspect branch protection and workflow runs.
- Provide a short collaborator notification and recovery steps (for forced-pushes / rewritten history).

## Workflows found locally

Files under `.github/workflows/` in this repo (present at time of writing):

- `ci-quick-smoke.yml` — lightweight smoke tests (workflow file present)
- `verify-export.yml` — runs in-process export verification and uploads artifacts
- `ci-smoke-puppeteer.yml` — Puppeteer/Chromium smoke export (has workflow_dispatch)
- `server-tests-pr.yml` — server-side test workflow (PR/dispatch)
- `WORKFLOWS.md` — human-readable summary of repository workflows

Docs and references in the repo mention CI requirements such as installing Chromium and setting `CHROME_PATH` for Puppeteer tests.

## What you can check locally (no token required)

- List workflows (local files):

```bash
ls -1 .github/workflows
```

- Inspect workflow text and triggers:

```bash
sed -n '1,240p' .github/workflows/ci-smoke-puppeteer.yml
sed -n '1,240p' .github/workflows/verify-export.yml
# or open WORKFLOWS.md
sed -n '1,240p' .github/workflows/WORKFLOWS.md
```

- Search repo for CI-related references:

```bash
grep -n "CHROME_PATH\|chromium\|actions/checkout\|workflow_dispatch" -R . | sed -n '1,200p'
```

These local checks confirm which workflows exist and the repo's expectations, but they do not tell you whether the workflows run successfully on GitHub or what branch protection rules are configured.

## What requires GitHub access / token

- Branch protection rules (who can push, required status checks, required reviewers, allow/deny force-push)
- Recent workflow runs and their statuses as recorded by GitHub Actions
- Whether branch protection would have prevented the forced pushes

To query those you can use the GitHub CLI (`gh`) or the REST API via `curl` with a PAT (`GITHUB_TOKEN`) that has repo access. `gh` is more convenient if available and authenticated.

### GitHub CLI examples (run locally when authenticated)

List workflows in the remote repo:

```bash
gh repo clone klav-ay/vanilla temp-vanilla && cd temp-vanilla
# or operate in this repo if you are authenticated
gh workflow list
```

List recent runs for a workflow (replace name if different):

```bash
gh run list --workflow ci-quick-smoke.yml
```

Get branch protection for `main` using the API via `gh`:

```bash
# returns protection JSON
gh api repos/klav-ay/vanilla/branches/main/protection
# show the required status checks and review settings
gh api repos/klav-ay/vanilla/branches/main/protection --jq '.required_status_checks, .required_pull_request_reviews, .enforce_admins, .allow_force_pushes'
```

Note: `gh api` requires you to be authenticated (`gh auth login`) and authorized for the repo. `--jq` needs `jq` installed locally for JSON filtering (or use `--jq` provided by `gh` if supported).

### GitHub REST API examples (curl) — needs a PAT in $GITHUB_TOKEN

Replace `OWNER` and `REPO` with `klav-ay` and `vanilla` (or appropriate owner). Ensure `$GITHUB_TOKEN` is set and has `repo` (and ideally `admin:repo_hook`) scopes.

Check branch protection for `main`:

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/klav-ay/vanilla/branches/main/protection | jq '.'

# to check whether force-push is allowed (if the key exists)
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/klav-ay/vanilla/branches/main/protection | jq '.allow_force_pushes, .required_status_checks, .required_pull_request_reviews'
```

List workflows on the repo and find the workflow id or filename:

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/klav-ay/vanilla/actions/workflows | jq '.'
```

List runs for a workflow (using filename) and show the latest run status:

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/klav-ay/vanilla/actions/workflows/ci-smoke-puppeteer.yml/runs" | jq '.workflow_runs[0] | {id, head_branch, status, conclusion, html_url}'
```

Check a specific run (replace RUN_ID):

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/klav-ay/vanilla/actions/runs/RUN_ID | jq '.'
```

## Notes about permissions & scope

- To read branch protection rules, your token must have appropriate repo access (for private repos `repo` scope).
- For administrative-level branch protection fields, you may need higher permissions (repo admin).
- `gh` uses the same token under the hood if you authenticate via `gh auth login`.

## Repository history rewrite notes (context for collaborators)

- We removed `client/.next` from history and force-pushed rewritten branches to `origin` to shrink repo size.
- The `origin` remote was re-added to `https://github.com/klav-ay/vanilla.git` in this workspace.
- Backup branches created locally before rewrite:
  - `backup/pre-filter-repo`
  - `backup/remove-next-before-filter`
    These were pushed to `origin` as safety copies.

## Collaborator notification (copy/paste)

Title: Repo history rewritten — please re-sync your local clone

Message:

> We removed generated Next.js build artifacts (`client/.next`) from the repository history and force-pushed rewritten branches to `origin` to shrink repo size. This rewrites commit history; you must re-align your local clones.
>
> Recommended (easy) action for most contributors:
>
> ```bash
> # reclone fresh (recommended)
> git clone https://github.com/klav-ay/vanilla.git
> ```
>
> If you have local branches with un-pushed commits you want to preserve:
>
> ```bash
> # keep a backup of your current work
> git checkout -b my-work-backup
> # fetch rewritten remote and reset your branch to remote/main (or another branch)
> git fetch origin
> git checkout main
> git reset --hard origin/main
> # re-apply your local commits via cherry-pick or by creating a branch and rebasing
> ```

## Final checklist for you

- [ ] Run the `curl` or `gh` commands above locally to inspect branch protection and workflow runs (requires token or gh auth).
- [ ] Notify collaborators using the message above (I can send or draft fuller message if you want).
- [ ] Optionally verify that CI runs for PRs and `main` after the force-push (open a test PR or re-run a workflow manually via Actions UI).

---

If you want, I can run the `curl` checks here now — provide a short-lived `GITHUB_TOKEN` with repo access (I will use it only to query and will not store it). Otherwise, run the `curl` / `gh` examples above locally and share the results if you want help interpreting them.
