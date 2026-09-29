# Claude Remediation Prepare

GitHub Action that validates a slash-command trigger comment on a PR (e.g.
`/claude-fix`), checks the commenter's permission and branch safety, and
gathers PR context (metadata, diff, review/issue comments) for an automated
Claude remediation run. It is the first half of the remediation flow; the
second half — committing the fix and opening a nested PR — is
[`howdycom/open-remediation-pr`](https://github.com/howdycom/open-remediation-pr).

The prepare script is zero-dependency TypeScript running on plain `node` via
type stripping — no repo toolchain install required on the runner.

Licensed under the [MIT License](LICENSE).

## Trigger commands

A reviewer (or author) requests remediation by commenting on the PR:

| Comment | Scope | Meaning |
|---|---|---|
| `/claude-fix` as a **reply to an inline review comment** | `single` | Fix just that one finding. Must be a reply to an existing inline comment, otherwise the run disables itself with a skip reason. |
| `/claude-fix all` as a **top-level PR comment** | `all` | Fix all eligible findings on the PR. |

The command must be the first non-empty line of the comment. The prefix is
configurable via `command_prefix`.

## Safety gates

Every gate that fails disables the run (`enabled: false` + `skip_reason`)
instead of failing the job, so skipped triggers stay green:

1. **Permission** — the commenter must have `write`, `maintain`, or `admin`
   on the repo (checked via the collaborators API).
2. **Same-repo PRs only** — forks are rejected, so remediation never exposes
   secrets to forked code.
3. **Protected branches** — PRs targeting `main`/`master` (configurable) are
   rejected.
4. **Sensitive paths** — PRs touching `.github/workflows/`,
   `.github/actions/` (configurable prefixes), or build-tooling manifests
   matched by basename at any depth (`Makefile`, `package.json`,
   `pyproject.toml`, `uv.lock`, `poetry.lock`, `package-lock.json`,
   `requirements.txt`, `conftest.py`, `setup.py`, `.npmrc` — configurable)
   are rejected. The nested remediation PR's base is the source PR's own
   head branch, and remediation runs the repo's own test/build/format
   commands, so unreviewed changes to these files could execute with
   repository secrets before a human reviews them.

## Context bundle

On success the action writes a `context_dir` (under `RUNNER_TEMP` by default)
containing everything the remediation job needs:

| File | Contents |
|---|---|
| `request.json` | Trigger (actor, comment, scope) plus the target review comment for `single` scope. |
| `pr.json` | Source PR metadata (`gh pr view`). |
| `pr.diff` | Full PR diff. |
| `review-comments.json` | All inline review comments. |
| `issue-comments.json` | All top-level PR comments. |

## Usage

Wire the trigger events to this action, then gate the LLM invocation and
the nested PR on `enabled`:

```yaml
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  prepare:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
    outputs:
      enabled: ${{ steps.prepare.outputs.enabled }}
    steps:
      - uses: howdycom/claude-remediation-prepare@v1
        id: prepare
        with:
          github_token: ${{ github.token }}

  remediate:
    needs: prepare
    if: needs.prepare.outputs.enabled == 'true'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      # ... check out the PR head, invoke your Claude action scoped to
      # non-executing tools (see Security notes), then:
      - uses: howdycom/open-remediation-pr@v1
        with:
          branch: ${{ steps.prepare.outputs.remediation_branch }}
          # ... base, title, body
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github_token` | yes | — | Token with repo read access, used by the `gh` CLI for permission checks, PR metadata, diff, and comments. |
| `command_prefix` | no | `/claude-fix` | Comment prefix that triggers remediation. |
| `protected_branches` | no | `main,master` | Comma-separated PR head branches that must never be targeted. |
| `sensitive_path_prefixes` | no | `.github/workflows/,.github/actions/` | Comma-separated path prefixes that block remediation when the source PR touches them. |
| `sensitive_file_names` | no | `Makefile,makefile,…` (see above) | Comma-separated basenames, matched at any depth, that block remediation when the source PR touches them. |

## Outputs

| Output | Description |
|---|---|
| `enabled` | `"true"` if remediation should proceed, `"false"` if skipped. |
| `skip_reason` | Human-readable reason (only when `enabled` is `"false"`). |
| `command_scope` | `"single"` (one inline finding) or `"all"`. |
| `context_dir` | Directory containing the context bundle. |
| `head_ref` / `head_sha` | Source PR head branch / SHA. |
| `pr_number` / `pr_title` / `pr_url` | Source PR identity. |
| `remediation_branch` | Suggested branch name, unique per run. |
| `trigger_url` | URL of the comment that triggered the run. |

## Requirements

- `gh` CLI and `node` 22+ on the runner (both preinstalled on
  GitHub-hosted runners; no `setup-*` step needed).
- Pin to a tag (`@v1`), never to `main`.

## Security notes for remediation consumers

This action only handles trigger validation and context gathering — the LLM
invocation itself (e.g. `anthropics/claude-code-action`) is wired up in each
consuming repo's own workflow, including its `--allowedTools` Bash allowlist.
Scope that allowlist to **non-executing commands only** (formatters and
linters: `black`, `isort`, `prettier`, `eslint --fix`, `flake8`, `mypy`, plus
read-only `git diff`/`git status`) — never test or build execution (`pytest`,
`npm run test`, `npm run build`, `npm ci`, `uv run`, `make test`, etc.).

Why: the agent's API key is present in the environment inherited by whatever
the Bash tool executes. A test/build command that reads its own environment
(intentionally or via a compromised dependency/test file) can exfiltrate the
key. Formatters/linters never execute the target code, so they don't have
this exposure — verification that a fix actually works should happen via the
normal CI that runs on the resulting draft PR, not inside the remediation job
itself.

## Versioning

Changes are tagged with semver (`v1`, `v1.1`, …). The major tag (`v1`) moves
to the latest compatible release; breaking changes bump the major version.
Don't reference `main` from a consumer workflow.

## Contributing

Changes go through a PR, not direct pushes to `main`. `npm run build` is a
typecheck (`tsc --noEmit`) over `prepare.ts`; keep the script dependency-free
so it runs on a bare runner. This action reads commenter permissions and PR
contents in consuming repos, so review matters here more than usual.
