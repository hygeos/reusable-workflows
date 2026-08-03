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
