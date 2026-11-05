# buildingway-ci

Shared GitHub Actions for Dalamud plugin projects: reusable workflows that a project calls from
thin caller workflows, and the composite action they use to install a pinned Dalamud.

| Workflow | Jobs (check names are `<caller job> / <job>`) | Inputs |
|---|---|---|
| `test.yml` | `test`; `revert-check` on pull requests (informs only; don't require it) | `test-project` (required), `revert-check` (default true), `dotnet-version` |
| `pr-build.yml` | `build`: format check, Release build, plugin artifact | `plugin` (required), `dotnet-version` |
| `workflow-lint.yml` | `actionlint`, `zizmor` | none |

| Action | Does |
|---|---|
| `.github/actions/install-dalamud` | Installs the Dalamud pinned in the project's `.github/dalamud.env` (see the comment in its `action.yml`) |

## Using them

Pin each to a full commit SHA with the tag in a comment, so Dependabot's `github-actions`
updates propose new versions as PRs:

```yaml
# .github/workflows/test.yml in the plugin repo
name: Tests
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
permissions:
  contents: read
jobs:
  test:
    uses: Cellar-Mander/buildingway-ci/.github/workflows/test.yml@<sha> # v0.1.0
    with:
      test-project: MyPlugin.Core.Tests
```

The caller's `permissions:` caps what the called jobs get; all of these need only
`contents: read`. A workflow that builds the plugin itself (a release, say) can use the action
directly after checking out:

```yaml
    - uses: Cellar-Mander/buildingway-ci/.github/actions/install-dalamud@<sha> # v0.1.0
```

`test.yml`'s `revert-check` runs the caller's `tools/revert-check`; set `revert-check: false` if
the project has none.

## Checking workflows locally

Install [actionlint](https://github.com/rhysd/actionlint) (with shellcheck) and
[zizmor](https://docs.zizmor.sh), then from the project's root run `actionlint && zizmor .`.
CI's zizmor config lives in `workflow-lint.yml`; it only disables rules, so a clean local run
means a clean CI run.

## Changing this repo

The reusable workflows pin this repo's own action to an earlier commit, since a reusable
workflow can't name the commit it runs from. To change the action: commit the change, then
bump the SHA in the workflows that use it in a later commit. The `self-pins` job fails while a
pin points at an older version of an action. Tag each release `vX.Y.Z` so Dependabot sees it.
