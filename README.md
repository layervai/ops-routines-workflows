# ops-routines-workflows

Public mirror of reusable GitHub Actions workflows from [`layervai/ops-routines`](https://github.com/layervai/ops-routines) — specifically the **dependency age-check** supply-chain gates.

This repo exists so **public** consumer repos in the `layervai` organization can call these workflows. GitHub blocks public → internal reusable-workflow calls regardless of `access_level`, so the workflows need to live somewhere public to be callable from public repos like [`qurl-python`](https://github.com/layervai/qurl-python) and [`qurl-integrations`](https://github.com/layervai/qurl-integrations).

## What's in here

Five self-contained reusable workflows — each has its bash logic **inlined** as heredoc content. No cross-repo checkout, no secrets, no GitHub App tokens.

| Workflow | Purpose |
|---|---|
| `age-check-actions.yml` | Block PRs adding `uses:` pins to recently-committed action SHAs |
| `age-check-docker.yml` | Block PRs adding recently-published Docker base images (Dockerfiles + compose YAML) |
| `age-check-go.yml` | Block PRs adding recently-published Go modules (go.sum diff) |
| `age-check-npm.yml` | Block PRs adding recently-published npm packages (package-lock/pnpm-lock/yarn.lock) |
| `age-check-pip.yml` | Block PRs adding recently-published PyPI packages (requirements.txt, poetry.lock, pyproject.toml) |

Each fails closed on any newly-added pin younger than `min_age_days` (7 for source-package ecosystems; 14 for Docker by default).

## What's NOT in here

- `lib/*.sh` canonical scripts (live in private `layervai/ops-routines`)
- Test harness (127 tests in `ops-routines/tests/`)
- CVE-verify logic, signed-commit library, routine prompts, runbooks
- Docker reusable uses `github.com/google/go-containerregistry/cmd/crane` internally — that's the only external binary dependency

The bash is commodity defensive infrastructure: `git diff` changed manifest files → curl public registries → classify ages → emit GH annotations. Nothing sensitive.

## Consumer setup

In your consumer repo:

```yaml
# .github/workflows/dependency-age-check-pip.yml
name: Dependency Age Check (pip)

on:
  pull_request:
  workflow_dispatch:

concurrency:
  group: dep-age-pip-${{ github.head_ref || github.ref_name }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  age-check:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'age-check-bypass') }}
    uses: layervai/ops-routines-workflows/.github/workflows/age-check-pip.yml@<SHA> # vX.Y.Z
    with:
      min_age_days: 7
    permissions:
      contents: read
```

Same pattern for the other ecosystems. No `secrets:` block needed — the workflows are self-contained.

## Bypass

Add the `age-check-bypass` label to a PR to skip the check. Use only for CVE-driven emergency updates.

## How these files are generated

They come from a generator in [`layervai/ops-routines`](https://github.com/layervai/ops-routines):

1. `lib/_preflight.sh` and `lib/age-check-<eco>.sh` are the canonical bash sources.
2. `templates/workflows/age-check-<eco>.yml.tmpl` is the YAML scaffolding.
3. `scripts/embed-lib-into-workflows.py` substitutes the scripts into the templates.
4. The output lands here as `.github/workflows/age-check-*.yml`.

**Do not hand-edit** the workflows in this repo — edit the lib/templates in `ops-routines` and regenerate.

## License

TBD (same as parent `layervai/ops-routines`).
