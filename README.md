# reusable-workflows

Reusable GitHub Actions workflows for HYGEOS projects, following the
[GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
model: workflows are defined once here and called from each project instead of
being copied around.

Available workflows:

- [`pypi_build.yml`](#pypi_buildyml--build-and-verify-a-python-package) —
  build and verify a Python package before publishing to PyPI
- [`github_release.yml`](#github_releaseyml--publish-a-github-release-on-tag-push) —
  publish a GitHub release or pre-release on version tag push
- [`tests.yml`](#testsyml--run-the-project-test-suite) —
  run the project test suite on its supported Python range
- [`docs_github_pages.yml`](#docs_github_pagesyml--build-and-publish-sphinx-docs) —
  build the Sphinx docs and publish them to GitHub Pages

## Setting up a project

### PyPI publishing

Save this as `.github/workflows/push_pypi.yml` in the project:

```yaml
name: Publish to PyPI

on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"

jobs:
  build:
    uses: hygeos/reusable-workflows/.github/workflows/pypi_build.yml@v1

  publish:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/${{ needs.build.outputs.package-name }}
    permissions:
      id-token: write
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/

      - name: Publish to PyPI (Trusted Publishing)
        uses: pypa/gh-action-pypi-publish@release/v1
```

The tag filter uses `+` (one or more digits) so that pre-release tags such as
`v1.0.0dev1` or `v1.0.0rc1` do not trigger a publish.

**Migrating from the old standalone `push_pypi.yml`:** replace the whole
content of the file with the caller above. Keep the filename: it is the one
registered in the PyPI Trusted Publisher configuration, so nothing needs to
change on pypi.org or in the repo's environments.

**One-time prerequisites** (per project, unchanged from the previous
standalone workflow):

- Create the `pypi` environment in the repo Settings > Environments.
- Configure a Trusted Publisher on pypi.org for the project (PyPI project
  Settings > Publishing > Add a new publisher) with:
  owner = `<org/user>`, repo = `<repo>`, workflow = `push_pypi.yml`,
  environment = `pypi`.

### GitHub releases

Save this as `.github/workflows/github_release.yml` in the project:

```yaml
name: GitHub Release

on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+*"  # trailing * also matches dev/rc suffixes

jobs:
  release:
    uses: hygeos/reusable-workflows/.github/workflows/github_release.yml@v1
    permissions:
      contents: write
```

`permissions: contents: write` on the calling job is required to create the
release, since the default token permissions may be read-only.

No other setup is needed. For the release description to be taken from the
project's `CHANGELOG.md`, follow the
[expected changelog format](#changelog-format); otherwise GitHub's
auto-generated notes are used.

### Tests

Save this as `.github/workflows/tests.yml` in the project:

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    uses: hygeos/reusable-workflows/.github/workflows/tests.yml@v1
```

No other setup is needed for most projects: the workflow detects the
environment manager, the tests directory and the Python versions from
`pyproject.toml` (see the [reference](#testsyml--run-the-project-test-suite)).

If some tests require a GPU, decorate them with `@pytest.mark.gpu` — they are
skipped in CI, which runs pytest with `-m "not gpu"` — and register the marker
in the committed `pyproject.toml` (a gitignored local `pytest.ini` is
invisible to CI):

```toml
[tool.pytest.ini_options]
markers = ["gpu: tests requiring a GPU (skipped in CI)"]
```

### Docs (GitHub Pages)

Save this as `.github/workflows/docs_github_pages.yml` in the project:

```yaml
name: Docs

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  docs:
    uses: hygeos/reusable-workflows/.github/workflows/docs_github_pages.yml@v1
    permissions:
      contents: write
```

`permissions: contents: write` is required to push the built HTML to the
`gh-pages` branch. On pull requests the docs are built as a check but not
published; publication happens when the change reaches `main`.

**One-time prerequisite** (per project): in the repo Settings > Pages, set the
source to "Deploy from a branch" with branch `gh-pages` / `/ (root)` (the
branch is created by the first publishing run).

## Workflow reference

### `pypi_build.yml` — build and verify a Python package

Called on a version tag push (e.g. `v1.0.3`), it:

1. checks that the tagged commit is on the `main` (or `master`) branch;
2. checks that the tag matches the `version` declared under `[project]` in
   `pyproject.toml` (which must be static, along with `name`);
3. builds the sdist + wheel with `python -m build`;
4. uploads `dist/` as the `python-package-distributions` artifact (1 day
   retention) for the caller's publish job.

| Input | Default | Description |
|---|---|---|
| `python-version` | `"3.12"` | Python version used to build the package |

| Output | Description |
|---|---|
| `package-name` | Project name read from `pyproject.toml` |

**Why doesn't it publish too?** PyPI Trusted Publishing requires the publish
job to run in a workflow file belonging to the publishing repository itself —
it rejects publish jobs running inside a reusable workflow from another
repository ([pypi/warehouse#11096](https://github.com/pypi/warehouse/issues/11096)).
So the short publish job stays in each project.

### `github_release.yml` — publish a GitHub release on tag push

Called on a version tag push — including pre-release tags — it:

1. checks that the tagged commit is on the `main` (or `master`) branch;
2. classifies the tag: plain `vX.Y.Z` tags become **releases**, suffixed tags
   such as `v0.2.0dev1` or `v1.0.0rc1` become **pre-releases**;
3. publishes the GitHub release, using as description the tag's section from
   the changelog when available (see format below), otherwise GitHub's
   auto-generated notes (with a workflow warning).

| Input | Default | Description |
|---|---|---|
| `changelog-file` | `"CHANGELOG.md"` | Path to the changelog file, relative to the repo root |

Unlike PyPI publishing, the whole job runs inside the reusable workflow (the
`GITHUB_TOKEN` is available to it), so the caller is minimal.

#### Changelog format

The description is taken from the section whose heading matches the tag name
exactly, i.e. `## <tag>` (`v` prefix included), up to the next `## ` heading —
the format used by
[geoclide's CHANGELOG.md](https://github.com/hygeos/geoclide/blob/main/CHANGELOG.md):

```markdown
## v3.0.3
Release date: 07-06-2025

* Fix bug in `vec2ang` function with float32 x, y, z vector components
```

If the file or the section is missing, the release is still created with
auto-generated notes.

### `tests.yml` — run the project test suite

Called on push / pull_request, it:

1. detects the environment manager: **pixi** when `pyproject.toml` has a
   `[tool.pixi]` section (or a `pixi.toml` exists), otherwise **pip/uv**
   installing the `[project]` dependencies (plus pytest);
2. detects the tests directory: `./tests` at the repo root, else a single
   `<package>/tests` match, else fails asking for the `tests-path` input;
3. derives the Python matrix from the min and max
   `Programming Language :: Python :: 3.X` classifiers, falling back to the
   `requires-python` bounds (min from `>= 3.X`; max from a `< 3.Y` bound if
   present, otherwise min only) — pixi projects get one extra "locked" entry
   running in the environment from the committed `pixi.lock`;
4. runs `pytest <tests-path> -m "not gpu"` on each matrix entry, so tests
   decorated with `@pytest.mark.gpu` are skipped.

| Input | Default | Description |
|---|---|---|
| `tests-path` | `""` | Path of the tests directory, relative to the repo root (empty = auto-detect `./tests` or `<package>/tests`) |
| `pytest-args` | `-m "not gpu"` | Arguments passed to pytest |

**Pixi projects:** pytest must be available in the *default* pixi environment,
and the workspace `platforms` must include `linux-64`. The "locked" matrix
entry tests the environment developers actually use (installed from the
committed `pixi.lock`, cached); the min/max entries pin the Python version
with `pixi add "python==3.X.*"` and re-solve from scratch, so both ends of
the declared support range are actually tested — a failure on the minimum
version usually means the declared range is stale.

### `docs_github_pages.yml` — build and publish Sphinx docs

Called on push / pull_request / workflow_dispatch, it:

1. detects the environment manager (same rule as `tests.yml`): **pixi** when
   `pyproject.toml` has a `[tool.pixi]` section (or a `pixi.toml` exists),
   otherwise **pip/uv** installing the project with its docs extras;
2. detects the Sphinx source directory: `docs/conf.py`, else
   `docs/source/conf.py`, else fails asking for the `docs-source` input;
3. picks the pixi environment: `docs` when declared in
   `[tool.pixi.environments]`, otherwise `default`;
4. if `<docs-source>/_notebooks/*.py` jupytext percent scripts exist, converts
   them to notebooks, moves them into the Sphinx source directory and executes
   them in the build environment;
5. builds once: with pixi, in the environment installed from the committed
   `pixi.lock` (cached); with uv, on the `python-version` input — testing the
   supported Python range is the job of the tests workflow, not this one;
6. runs `sphinx-build -b html` directly (any `docs/Makefile` is bypassed);
7. publishes with `ghp-import -n -p -f` to the `gh-pages` branch, never on
   `pull_request` events.

| Input | Default | Description |
|---|---|---|
| `docs-source` | `""` | Sphinx source directory containing `conf.py` (empty = auto-detect `docs` or `docs/source`) |
| `extras` | `"docs"` | Comma-separated extras installed on the pip/uv path (ignored with pixi) |
| `pixi-environment` | `""` | Pixi environment for the build (empty = `docs` if declared, else `default`) |
| `python-version` | `"3.13"` | Python version on the pip/uv path (ignored with pixi, which uses its lock) |
| `publish` | `true` | Publish to GitHub Pages |

Notes: projects using `nbsphinx` need the pandoc *binary* — the workflow
apt-installs it on the runner, so no project setup is needed (conda pandoc in
a pixi environment also works and is harmless duplication). Notebook execution
needs a Jupyter kernel: make sure `ipykernel` is reachable from the docs
environment (add it to the `docs` extra rather than relying on transitive
installs). Overlapping publishing runs force-push `gh-pages` and the last one
wins; add a `concurrency` group in the caller if that ever matters.

## Versioning

Callers reference the major-version tag `v1`, which is a *moving* tag (like
`actions/checkout@v4`): after a backward-compatible fix on `main`, re-point it
so that all projects pick the fix up on their next run:

```bash
git tag -f v1
git push -f origin v1
```

For a breaking change (renamed input/output, artifact name, required setup),
tag `v2` instead and let projects switch to `...@v2` explicitly, so that
existing callers keep working on `v1`.
