# claude-review

A reusable GitHub Actions workflow that posts a Claude review on pull requests in the `jitsi` organization.

The workflow uses no Anthropic API key.
The review job exchanges its GitHub OIDC token for an Anthropic token that lives 10 minutes or less.
This exchange is Workload Identity Federation (WIF).

## Use the workflow in a repository

Add this file as `.github/workflows/claude-review.yml` on the default branch:

```yaml
name: Claude review
on:
  pull_request_target:
    types: [opened, reopened, synchronize, ready_for_review, labeled]
  issue_comment:
    types: [created]
permissions: {}
jobs:
  review:
    uses: jitsi/claude-review/.github/workflows/review.yml@v1.0.0
    permissions:
      contents: read
      pull-requests: write
      id-token: write
```

Then do these steps:

1. Merge the file to the default branch.
   `pull_request_target` and `issue_comment` always run the workflow from the default branch.
2. Create a label with the name `claude-review` in the repository.
3. Optional: add `.github/claude-review.md` with review guidance for the repository.
   Claude also reads `AGENTS.md` and `CLAUDE.md` if they exist.

The repository needs no secret and no variable.
Pin an exact tag. A branch or a moving tag does not get an Anthropic token.

## Triggers

A maintainer is a user with `write` permission or higher on the repository.
The workflow reads this permission from the GitHub API.

| Trigger | Draft PR | Ready PR | Who can start it |
| --- | --- | --- | --- |
| PR opened, reopened, new push, or ready for review | No review | Review | The PR author and the event sender are maintainers. |
| `claude-review` label added | Review | Review | The user who adds the label is a maintainer. |
| A comment that starts with `/review` | Review | Review | The commenter is a maintainer. |

- The workflow removes the `claude-review` label after the run.
- A new push cancels an automatic review that is in progress on the same PR.
- PRs from bots get no automatic review. A maintainer can start one with the label or `/review`.
- PRs from outside contributors get a review only when a maintainer starts it.

## Output

The workflow posts 1 GitHub review with the state `COMMENT`.
The review has a summary and 1 inline comment for each finding.
If GitHub rejects an inline comment, the findings go into the review body.
The review never approves a PR and never requests changes.

## How it works

```
 PR event, claude-review label, or /review comment
        │
        ▼
 gate job ── not a maintainer, or automatic run on a draft ──▶ skip
 (permission check through the API, pin the head SHA)
        │
        ▼
 review job (id-token: write, contents: read)
   GitHub OIDC token ──▶ Anthropic token exchange
   Claude reads the pinned SHA ──▶ secret scan ──▶ review.json artifact
        │
        ▼
 post job (pull-requests: write, no id-token)
   secret scan ── match ──▶ fail, post nothing
        │
        ▼
 review on the PR, label removed
```

### Security controls

- The federation rule accepts only the `review` job of `review.yml` at a `v*` tag of this repository.
- A fork has a different owner ID, so its token exchange fails.
- The `review` job has no GitHub write access. It passes `github_token` to `claude-code-action`, so the action does not get a Claude GitHub App token.
- Claude has the tools `Read`, `Grep` and `Glob` only. It cannot run commands or write files.
- The working directory is the base branch. The PR head is in `claude-review-pr/head`.
- `--setting-sources user` and `--strict-mcp-config` stop Claude Code from loading `.claude/` or `.mcp.json` from a checkout.
  A PR cannot add a hook that runs in the job.
- Claude cannot read files outside the working directory, `/proc`, or the `.git` directories.
- The job never installs, builds or runs PR code.
- Actions logs do not show the Claude transcript. The artifact holds only `review.json`.
- If the review contains `sk-ant-`, `ghs_` or a JWT, the workflow fails and posts nothing.
- The workflow pins all actions by commit SHA.

### Token lifetime

A GitHub OIDC token expires about 5 minutes after GitHub issues it.
The Anthropic token lives for the smaller of 600 seconds and 2 times the remaining life of the OIDC token.
`claude-code-action` writes a new OIDC token every 4 minutes, and Claude Code exchanges it again before expiry.
So a review that takes more than 10 minutes works. Do not raise the token lifetime for this reason.

## One-time setup

### Anthropic

An admin of the Anthropic organization does these steps:

1. Create 1 dedicated workspace for `jitsi` PR reviews.
2. Set a monthly spend limit on the workspace, for example 100 USD.
3. Create the service account `gha-jitsi-pr-review` with the Developer role, and add it to the workspace.
4. Reuse the GitHub Actions federation issuer (`https://token.actions.githubusercontent.com`) of the organization.
5. Create 1 federation rule with the settings below.

| Setting | Value |
| --- | --- |
| Audience | `https://api.anthropic.com` |
| Claim | `repository_owner_id` is the numeric ID of the `jitsi` organization (`3671647`). |
| Condition | `claims.job_workflow_ref.startsWith("jitsi/claude-review/.github/workflows/review.yml@refs/tags/v") && claims.event_name in ["pull_request_target", "issue_comment"]` |
| Target | The service account `gha-jitsi-pr-review` |
| Workspace | The dedicated workspace |
| OAuth scope | `workspace:inference` |
| Token lifetime | 600 seconds |

Put the 4 IDs in `env` of the `review` job in `.github/workflows/review.yml`.
The IDs are the organization UUID, the workspace ID (`wrkspc_…`), the service account ID (`svac_…`) and the rule ID (`fdrl_…`).
These IDs are identifiers, not credentials.

### GitHub

Do not install the Claude GitHub App on the `jitsi` organization.

Protect this repository:

1. Add a tag ruleset for `v*` that blocks update and deletion of tags.
   Then a tag that a caller pins cannot change.
2. Add a branch ruleset for `master` that needs a pull request and a code owner review.
3. Keep the write access to this repository small.
   A user with write access can push a tag and mint tokens.

## Release a new version

1. Merge the change to `master` through a reviewed pull request.
2. Push a new tag, for example `v1.0.1`. Do not move an existing tag.
3. Update the pin in the caller workflows. Dependabot can open these pull requests.

## Limits

- The review does not run lint, tests or builds.
- The `/review` comment pins the head SHA when the `gate` job runs, not when the maintainer wrote the comment.
- If the review text contains a string that looks like a credential, the run fails. A maintainer can start the review again.

See [AGENTS.md](AGENTS.md) for the rules that apply to changes in this repository.
