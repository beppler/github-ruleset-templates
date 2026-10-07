# GitHub Ruleset Templates

Importable GitHub **repository rulesets** that protect the default branch (`main`) so changes only land through pull requests.

There are two templates. They are identical except that the second one also requires CI tests to pass before a PR can be merged.

| Template | File | Requires tests |
| --- | --- | --- |
| Protect main | `protect-main.json` | No |
| Protect main + tests | `protect-main-with-tests.json` | Yes |

## What both templates enforce

- **No direct pushes** to the default branch. Changes must go through a pull request.
- **Restrict deletions**: the branch can't be deleted.
- **Block force pushes**: history can't be rewritten.
- **Require linear history**: no merge commits on `main`. PRs are merged with squash or rebase.
- **Require a pull request** with:
  - 1 required approval
  - Dismiss stale approvals when new commits are pushed
  - Review required from Code Owners
- **Bypass list**: the *Repository admin* role can bypass the rules **for pull requests only**. This is the "break glass" option for emergencies: you still open a PR, but you can use **Merge without waiting for requirements to be met**. Direct pushes stay blocked.

### Additional rule in `protect-main-with-tests.json`

- **Require status checks to pass** before merging, with the check named `test` and **Require branches to be up to date before merging** enabled.

```json
{
  "type": "required_status_checks",
  "parameters": {
    "strict_required_status_checks_policy": true,
    "do_not_enforce_on_create": false,
    "required_status_checks": [
      { "context": "test" }
    ]
  }
}
```

## How to use

1. Open the target repository on GitHub, then go to **Settings → Rules → Rulesets**.
2. Click **New ruleset → Import a ruleset** and select one of the JSON files from this repository.
3. Review the settings and click **Create**.
4. Go to **Settings → General → Pull Requests** and uncheck **Allow merge commits**. Otherwise the Merge button is shown but fails because of the linear history rule.

You need admin access to the repository. Rulesets are available on public repositories on all plans, and on private repositories with GitHub Pro, Team or Enterprise.

## Before importing

### CODEOWNERS

Both templates require review from Code Owners, so the repository needs a `CODEOWNERS` file on the default branch (in `.github/`, the repo root, or `docs/`). Example:

```text
* @your-github-username
```

Every owner must have **Write** access or higher, or the entry is ignored. If you don't want this requirement, set `require_code_owner_review` to `false`.

### Set the status check name (tests template only)

Replace `"test"` with the exact name of your CI job, as shown in the PR's checks list. With GitHub Actions this is the job's `name:` (or its key if there is no `name:`). For matrix jobs each combination is a separate check, for example `test (ubuntu-latest, 20)`.

Your workflow must run on `pull_request`. If it doesn't, the required check stays "Expected" forever and blocks every PR. A typo in the name has the same effect.

## Things to know

### Solo repositories

The PR author can't approve their own PR. If you are the only collaborator, the 1-approval requirement can't be satisfied normally, so you merge your own PRs using the admin bypass. Note that **bypass skips every rule in the ruleset**, including the status checks. If you want tests enforced against yourself too:

- remove the entry from `bypass_actors`,
- set `required_approving_review_count` to `0`,
- set `require_code_owner_review` to `false`.

### Bypass modes

| Mode | Effect |
| --- | --- |
| `pull_request` (used here) | Bypass actors still open a PR, but can merge it without meeting the requirements. Direct pushes stay blocked. |
| `always` | Bypass actors can also push directly to `main`. Use only if you really need it. |

The bypass entry uses `actor_id: 5`, which is the **Repository admin** role. After an emergency merge, check that CI passes on `main` (make the workflow also run on `push` to `main`).

### Updating PR branches

Because of linear history, prefer `git rebase main` over `git merge main` when updating a PR branch.

### Existing protections

Classic branch protection rules stack with rulesets, and the most restrictive result wins. Remove old branch protection rules on `main` to avoid confusion.

### Testing

Use **Enforcement status: Evaluate** first if you want to see what would be blocked without enforcing it. To verify the active ruleset, `git push origin main` should be rejected with a message saying changes must be made through a pull request.

## Customization cheat sheet

| Goal | Change |
| --- | --- |
| More approvals | `required_approving_review_count` |
| Require conversations to be resolved | `required_review_thread_resolution: true` |
| Require approval of the most recent push | `require_last_push_approval: true` |
| Less friction when `main` moves | `strict_required_status_checks_policy: false` |
| Protect another branch | Edit `conditions.ref_name.include`, e.g. `["refs/heads/release/*"]` |
| Trial run without enforcing | `"enforcement": "evaluate"` |
