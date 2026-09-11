# db-github-workflows

Shared GitHub Actions building blocks for DesignBuilder repositories. Other
repos reference what lives here instead of each keeping its own copy, so a fix
to the release logic lands everywhere at once.

Two mechanisms are on offer, and the difference matters when you pick one:

| | Lives in | Referenced as | Use when |
|---|---|---|---|
| **Reusable workflow** | `.github/workflows/` | `uses:` on a **job** | You want a whole job — checkout, runner, permissions and all |
| **Composite action** | `actions/<name>/` | `uses:` on a **step** | You want one step inside a job you already have |

## Contents

- [`.github/workflows/release.yml`](.github/workflows/release.yml) — reusable
  workflow. Tags and releases the version declared in the repository.
- [`actions/read-version`](actions/read-version/action.yml) — composite action.
  Reads the declared version and the tag name it implies.

## Versioning and pinning

Callers pin a ref after the `@`. Prefer the moving major tag:

```yaml
uses: DesignBuilderSoftware/db-github-workflows/.github/workflows/release.yml@v1
```

`v1` is re-pointed at each backwards-compatible release, so callers get fixes
without editing anything. Pin a full SHA instead if a repository needs to be
certain nothing moves under it.

If this repository is private, other repositories cannot reference it until
its **Settings → Actions → General → Access** is set to allow access from
repositories in the DesignBuilderSoftware organisation.

## `release.yml`

Cuts a git tag and a GitHub Release from the version declared in the
repository, so the declared version and the tag can never drift apart. It is
idempotent: if the tag already exists, the job logs that and stops, which is
what makes it safe to fire on every push to `main`.

### Usage

This is [`db-process`](https://github.com/DesignBuilderSoftware/db-process)'s
own release workflow, reduced to a reference to this repository. Drop it in as
`.github/workflows/release.yml`:

```yaml
name: Release

# Cuts a git tag + GitHub Release whenever pyproject.toml's version changes on
# main, so the declared version and the tag can never drift apart.

on:
  push:
    branches: [main]
    paths: ['pyproject.toml']
  workflow_dispatch:

jobs:
  release:
    permissions:
      contents: write
    uses: DesignBuilderSoftware/db-github-workflows/.github/workflows/release.yml@v1
```

The `paths` filter is what makes this cheap — the job only starts when the
version file is touched. `workflow_dispatch` lets you re-run it by hand if a
release needs recreating.

Note that `permissions: contents: write` is set by the **caller**, on the job.
A called workflow can never hold more permission than the caller's token gives
it, so this cannot be defaulted from inside `release.yml`.

### Inputs

All optional.

| Input | Default | Description |
|---|---|---|
| `version-file` | `pyproject.toml` | File to read the version from. `pyproject.toml` (`[project].version`, falling back to `[tool.poetry].version`) and `package.json` (`.version`) are parsed; any other filename is read as plain text containing only the version. |
| `version` | *(empty)* | Explicit version, e.g. `1.2.3`. Overrides `version-file` for repositories that keep the version somewhere this workflow cannot parse. |
| `tag-prefix` | `v` | Prepended to the version to form the tag. Set to `''` for bare `1.2.3` tags. |
| `target` | the caller's SHA | Commit-ish the tag points at. |
| `title` | the tag | Release title. |
| `notes-file` | *(empty)* | Path to a file holding the release notes. When empty, notes are generated from commits and PRs since the last release. |
| `draft` | `false` | Create the release as a draft. |
| `prerelease` | `false` | Mark the release as a pre-release. |
| `fail-if-tag-exists` | `false` | Fail loudly instead of skipping when the tag is already present. |
| `runs-on` | `ubuntu-latest` | Runner label for the job. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `token` | no | Token used to create the tag and release. Defaults to the caller's `GITHUB_TOKEN`. Supply a PAT only if the published release needs to trigger *other* workflows — releases created with `GITHUB_TOKEN` deliberately do not. |

### Outputs

| Output | Description |
|---|---|
| `version` | The resolved version, without the tag prefix. |
| `tag` | The tag name (prefix + version). |
| `created` | `'true'` if a release was created, `'false'` if the tag already existed. |
| `url` | URL of the release; empty when nothing was created. |

### Fuller example

Reading the version from a plain `VERSION` file, hand-written notes, and a
follow-on job that only runs when a release was actually cut:

```yaml
name: Release

on:
  push:
    branches: [main]
    paths: ['VERSION', 'CHANGELOG.md']
  workflow_dispatch:

jobs:
  release:
    permissions:
      contents: write
    uses: DesignBuilderSoftware/db-github-workflows/.github/workflows/release.yml@v1
    with:
      version-file: VERSION
      notes-file: CHANGELOG.md
      prerelease: true

  announce:
    needs: release
    if: needs.release.outputs.created == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Shipped ${{ needs.release.outputs.tag }} at ${{ needs.release.outputs.url }}"
```

## `actions/read-version`

Use this when you only need the version inside a job you already have — a
build that stamps artefacts, say — rather than the whole release job.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - id: version
        uses: DesignBuilderSoftware/db-github-workflows/actions/read-version@v1
        with:
          version-file: pyproject.toml   # default
          tag-prefix: v                  # default

      - run: echo "building ${{ steps.version.outputs.version }} (${{ steps.version.outputs.tag }})"
```

Inputs: `version-file`, `tag-prefix`. Outputs: `version`, `tag`. Resolution
rules are identical to `release.yml`.

## Requirements

- Parsing `pyproject.toml` uses the runner's `python3` and its standard-library
  `tomllib`, so the runner needs **Python 3.11+**. Every current
  GitHub-hosted image satisfies this.
- Parsing `package.json` uses `jq`, preinstalled on GitHub-hosted runners.
- `release.yml` uses the `gh` CLI, also preinstalled.

## Maintaining this repository

See [CONTRIBUTING.md](CONTRIBUTING.md) for how `main`, the `release/vN`
branches and the moving `vN` tags fit together, what counts as a breaking
change to a caller's contract, and how to test a change before tagging it.

In short: develop on a branch off `main`, let
[`self-test.yml`](.github/workflows/self-test.yml) run against it, then tag an
immutable version and move the major pointer.

```bash
git tag -a v1.1.0 -m "v1.1.0" && git push origin v1.1.0
```

```bash
git tag -f v1 v1.1.0 && git push origin --force v1
```

One trap worth repeating here: version-resolution logic is deliberately
duplicated between `release.yml` and `actions/read-version/action.yml`, because
a reusable workflow cannot `uses: ./actions/...` — when called from elsewhere,
`./` resolves against the caller's checkout. Change one, change the other.

## Licence

MIT — see [LICENSE](LICENSE).
