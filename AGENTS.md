# AGENTS.md

Guidance for AI agents and humans who change this repository.

## Intent

This repository holds one reusable GitHub Actions workflow: `.github/workflows/review.yml`.
Repositories in the `jitsi` GitHub organization call it to get a Claude review on pull requests.

The workflow uses no Anthropic API key.
It uses Workload Identity Federation (WIF).
The review job exchanges its GitHub OIDC token for an Anthropic token that lives 10 minutes or less.
The Anthropic federation rule accepts only tokens from `review.yml` at a `v*` tag of this repository.

The repository is public. Every change must assume that an attacker reads the code and the Actions logs.

## Layout

| Path | Purpose |
| --- | --- |
| `.github/workflows/review.yml` | The reusable workflow. All logic, the prompt and the output schema are in this file. |
| `.github/CODEOWNERS` | Requires a code owner review for every change. |
| `README.md` | How to set up Anthropic, protect this repository and call the workflow. |

A called workflow cannot read other files of this repository at run time.
So `review.yml` keeps the prompt, the JSON schema and the scripts inline.

## Jobs

1. `gate` decides if a review runs. It reads permissions from the GitHub API and pins the head SHA.
2. `review` holds the Anthropic token. It has `contents: read` and `id-token: write`, and no GitHub write access.
3. `post` posts the review and removes the `claude-review` label. It has no `id-token` permission.

## Security rules

Do not weaken a rule below without a written reason in the pull request.

- The federation rule trusts `job_workflow_ref` at a `refs/tags/v` tag. Write access to this repository lets a user mint tokens.
  Tags are immutable through a tag ruleset. Callers pin an exact tag, for example `@v1.0.0`.
- The 4 Anthropic IDs are in `env` in `review.yml`. They are identifiers, not credentials.
- Never add an Anthropic API key, a `secrets:` input or a fallback to an API key.
- The `review` job never installs, builds or runs PR code.
  The workflow checks out the PR head into `claude-review-pr/head`, below the base branch checkout.
  The working directory stays on the base branch.
- Both checkouts use `persist-credentials: false`.
- Pass `github_token: ${{ github.token }}` to `claude-code-action`.
  Without it, the action gets a Claude GitHub App token with write access.
- Claude gets the tools `Read`, `Grep` and `Glob` only. No Bash, no write tools, no MCP servers.
- `--setting-sources user` and `--strict-mcp-config` stop the CLI from loading `.claude/` and `.mcp.json` from a checkout.
- `permissions.blockReadsOutsideWorkingDirectories` is true. Deny rules block `/proc`, and `.git` in both checkouts.
  `$RUNNER_TEMP` holds the identity token and the credentials cache. It must stay unreadable.
- `show_full_output` stays false. Actions logs of public repositories are public.
- Do not upload `execution_file`. Upload only `review.json`.
- The secret scan runs in `review` before the upload and again in `post` before it posts the review.
- Pin every action by full commit SHA, with the version in a trailing comment.
- No `actions/cache` in the `review` job.
- Never put `${{ }}` expressions with PR text (title, body, comment, branch name) directly in a `run:` script.
  Pass them through `env:`.

## Change process

1. Open a pull request. A code owner must approve it.
2. Check the workflow with `actionlint` and `zizmor`.
3. After the merge, push a new immutable tag, for example `v1.0.1`.
4. Callers update their pin. Dependabot can do this.

## Scope

The review posts comments only. It never approves or requests changes.
The workflow does not run lint, tests or builds. Other workflows do that.
