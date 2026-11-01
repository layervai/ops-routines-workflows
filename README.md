# ops-routines-workflows

Public mirror of reusable GitHub Actions workflows from [`layervai/ops-routines`](https://github.com/layervai/ops-routines) — specifically the **dependency age-check** supply-chain gates.

This repo exists so **public** consumer repos in the `layervai` organization can call these workflows. GitHub blocks public → internal reusable-workflow calls regardless of `access_level`, so the workflows need to live somewhere public to be callable from public repos like [`qurl-python`](https://github.com/layervai/qurl-python) and [`qurl-integrations`](https://github.com/layervai/qurl-integrations).

## What's in here

Six self-contained reusable workflows — each has its bash logic **inlined** as heredoc content. Public-registry checks need no extra secrets. Private action or Go-module checks use the optional scoped credentials declared by each workflow.

| Workflow | Purpose |
|---|---|
| `age-check-actions.yml` | Block PRs adding `uses:` pins to recently-committed action SHAs |
| `age-check-docker.yml` | Block PRs adding recently-published Docker base images (Dockerfiles + compose YAML) |
| `age-check-go.yml` | Block PRs adding recently-published Go modules (go.sum diff) |
| `age-check-npm.yml` | Block PRs adding recently-published npm packages (package-lock/pnpm-lock/yarn.lock) |
| `age-check-pip.yml` | Block PRs adding recently-published PyPI packages (requirements.txt, poetry.lock, pyproject.toml) |
| `age-check-precommit.yml` | Block PRs adding recently-committed pre-commit hook revisions |

Each fails closed on any newly-added pin younger than `min_age_days` (7 for source-package ecosystems; 14 for Docker by default).

## What's NOT in here

- `lib/*.sh` canonical scripts (live in private `layervai/ops-routines`)
- Test harness (127 tests in `ops-routines/tests/`)
- CVE-verify logic, signed-commit library, routine prompts, runbooks
- Docker reusable uses `github.com/google/go-containerregistry/cmd/crane` internally — that's the only external binary dependency

The bash is commodity defensive infrastructure: `git diff` changed manifest files → curl public registries → classify ages → emit GH annotations. Nothing sensitive.

## Caller permissions and private action references

Callers must declare job-level `permissions`, with `contents: read` for age checks. Add
`pull-requests: write` only when enabling `comment_on_failure`. The reusables
inherit those permissions, so a read-only caller can run without a startup
failure.

The Actions checker accepts an optional `AGE_CHECK_ACTIONS_READ_TOKEN` secret with read
access to referenced private or internal repositories. Commit-date lookups use that
token; provenance checks and optional PR comments keep the caller's `GITHUB_TOKEN`.
The token must cover every repository being dated. Public pins retain the
anonymous exact-SHA fallback when the API read fails; unreadable private pins fail closed.

Actions, Docker, and Go also accept the matching `age-check-bypass-actions`,
`age-check-bypass-docker`, or `age-check-bypass-go` label. Apply an approved
bypass in the reusable, keeping the caller job present so required checks
report a result. Scoped and global labels require verified-CVE approval. The checker does not
verify the label actor. If a consumer guards only the global label name, add
the three scoped names to that guard before enabling scoped bypasses. The
automatic CVE verifier still applies only the global label; npm, pip, and
pre-commit recognize only that label.

## Consumer setup

In your consumer repo:

```yaml
# .github/workflows/dependency-age-check-pip.yml
name: Dependency Age Check (pip)

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]
  workflow_dispatch:

concurrency:
  group: dep-age-pip-${{ github.head_ref || github.ref_name }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  age-check:
    uses: layervai/ops-routines-workflows/.github/workflows/age-check-pip.yml@414821a809395384de83d853a4bfd5b46e6bfc55 # v0.15.0
    with:
      min_age_days: 7
    permissions:
      contents: read
```

Use the same event types for the other ecosystems. Default public checks need no `secrets:` block; private reference reads use the optional credentials described above.

### Optional: PR comments with retry-after dates (v0.6.0+)

Set `comment_on_failure: true` and the workflow will post a sticky PR comment listing each pin that failed the age check, with its eligible-after date and days remaining. The comment is updated in place on subsequent runs and deleted once the PR clears.

Requires the calling job to grant `pull-requests: write` (Dependabot PRs need this granted explicitly — the default Dependabot token is read-only). If the token can't write, the post-step soft-fails to a `::notice` and the run still succeeds; retry-after dates are still in the Job Summary.

```yaml
jobs:
  age-check:
    uses: layervai/ops-routines-workflows/.github/workflows/age-check-pip.yml@414821a809395384de83d853a4bfd5b46e6bfc55 # v0.15.0
    with:
      min_age_days: 7
      comment_on_failure: true
    permissions:
      contents: read
      pull-requests: write
```

## Bypass

Add the `age-check-bypass` label to a PR to skip all age checks. The Actions, Docker, and Go checks also honor their matching scoped label. Use only for approved CVE-driven emergency updates.

Keep `labeled` and `unlabeled` in the caller's pull-request event types. Adding a bypass label then reports the named child check as `SKIPPED`; removing it reruns the check. Do not put a bypass `if:` on the caller job: that hides the child check required by branch protection. A manual rerun keeps its original event payload and cannot pick up a newly changed label.

## How these files are generated

They come from a generator in [`layervai/ops-routines`](https://github.com/layervai/ops-routines):

1. `lib/_preflight.sh` and `lib/age-check-<eco>.sh` are the canonical bash sources.
2. `templates/workflows/age-check-<eco>.yml.tmpl` is the YAML scaffolding.
3. `scripts/embed-lib-into-workflows.py` substitutes the scripts into the templates.
4. The output lands here as `.github/workflows/age-check-*.yml`.

**Do not hand-edit** the workflows in this repo — edit the lib/templates in `ops-routines` and regenerate.

## License

TBD (same as parent `layervai/ops-routines`).
