# reusable-workflows

Reusable GitHub Actions workflows for HYGEOS projects, following the
[GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
model: workflows are defined once here and called from each project instead of
being copied around.

## Available workflows

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

## Using it from a project

Save this as `.github/workflows/push_pypi.yml` in the project:

```yaml
name: Publish to PyPI

on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"

jobs:
  build:
    uses: hygeos/reusable-workflows/.github/workflows/pypi_build.yml@main

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

### One-time prerequisites (per project)

Unchanged from the previous standalone workflow:

- Create the `pypi` environment in the repo Settings > Environments.
- Configure a Trusted Publisher on pypi.org for the project (PyPI project
  Settings > Publishing > Add a new publisher) with:
  owner = `<org/user>`, repo = `<repo>`, workflow = `push_pypi.yml`,
  environment = `pypi`.

### Migrating a project that has the old standalone `push_pypi.yml`

Replace the whole content of its `.github/workflows/push_pypi.yml` with the
caller above. Keep the filename: it is the one registered in the PyPI Trusted
Publisher configuration, so nothing needs to change on pypi.org or in the
repo's environments.

## Versioning

Callers currently reference `@main`. Once the workflows are stable, tag a
release here (e.g. `v1`) and pin callers to it
(`.../pypi_build.yml@v1`) so that later changes cannot break existing
projects.
