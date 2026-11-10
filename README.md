# GitHub Ruleset Templates

Ready-to-import rulesets that lock down your `main` branch so all changes go through pull requests.

## The three templates

| What you want | File |
| --- | --- |
| Protect `main` (PRs only, no delete, no force push, linear history) | `protect-main.json` |
| Require 1 approval (plus an extra review for Copilot-authored PRs) | `require-reviews.json` |
| Require CI tests to pass | `require-tests.json` |

You can import any combination. They stack, and when rules overlap, the stricter one wins.

## Setup

1. In your repository, go to **Settings → Rules → Rulesets**.
2. Click **New ruleset → Import a ruleset** and pick one of the JSON files.
3. Review the settings, then click **Create**.
4. Go to **Settings → General → Pull Requests** and uncheck **Allow merge commits**. Without this, merging fails because of the linear history rule.

You need admin access. Rulesets work on public repos on any plan, and on private repos with Pro, Team, or Enterprise.

## Before importing

### Create a CODEOWNERS file

`protect-main.json` requires review from Code Owners, so you need a `CODEOWNERS` file on the default branch (in `.github/`, the repo root, or `docs/`):

```text
* @your-github-username
```

Every owner listed needs **Write** access or higher. Solo repository? You can skip this file — the code-owner requirement only applies to files that have an owner, so with no `CODEOWNERS` file it does nothing and PRs merge without review.

### Match the test name to your CI (tests template only)

Replace `"test"` with the exact name of your CI job as it appears in the PR's checks list. For GitHub Actions, that's the job's `name:` (or its key if there's no `name:`). Matrix jobs produce separate checks like `test (ubuntu-latest, 20)`.

⚠️ Your workflow must run on `pull_request`, and the name must match exactly. Otherwise the check stays "Expected" forever and blocks every PR.

## What the rules do

With all three imported, every change to `main` must:

- Go through a pull request (direct pushes are blocked)
- Pass the `test` CI check, with the branch up to date first
- Get 1 approval, and Code Owner review; new commits dismiss old approvals

Admins can skip the review and test requirements via the bypass (see below), but never the PR requirement itself.

Plus, `main` itself:

- Can't be deleted
- Can't have its history rewritten (no force pushes)
- Can't have merge commits — merge PRs with squash or rebase

### The admin "break glass" option

Repository admins can still merge a PR that doesn't meet the requirements (**Merge without waiting for requirements to be met**), but direct pushes stay blocked. This exists so emergencies don't require changing settings.

Note: bypass is per-ruleset, and it skips *all* rules in a ruleset that lists the bypass actor. All three templates use the same entry (Repository admin, PR-only mode), so an admin's **Merge without waiting** skips reviews *and* the test check. To keep that safe, make your workflow also run on `push` to `main`, so checks skipped during an emergency merge still run afterwards.

| Bypass mode | Effect |
| --- | --- |
| `pull_request` (used here) | Admins open a PR, then can merge without meeting requirements. Direct pushes stay blocked. |
| `always` | Admins can also push directly to `main`. Avoid unless you really need it. |

## Good to know

- **Solo repositories**: Don't import `require-reviews.json` and skip the `CODEOWNERS` file — zero approvals are required, so you can open a PR and merge normally. If you do keep a `CODEOWNERS` file, you can't approve your own PR; merge those with the admin bypass, or set `require_code_owner_review` to `false` in `protect-main.json`.
- **Updating a PR branch**: Use `git rebase main`, not `git merge main`, to keep history linear.
- **Old-style branch protection**: Classic branch protection rules stack with rulesets (stricter wins). Remove old rules on `main` to avoid confusion.
- **Try before enforcing**: Set **Enforcement status: Evaluate** to see what would be blocked without actually blocking it. To confirm a ruleset is active, `git push origin main` should be rejected with a pull-request message.

## Customization cheat sheet

| Goal | Change |
| --- | --- |
| Require more approvals | `required_approving_review_count` |
| Require conversations to be resolved | `required_review_thread_resolution: true` |
| Require approval after the latest push | `require_last_push_approval: true` |
| Less friction when `main` moves | `strict_required_status_checks_policy: false` |
| Protect another branch | Edit `conditions.ref_name.include`, e.g. `["refs/heads/release/*"]` |
| Trial run without enforcing | `"enforcement": "evaluate"` |
