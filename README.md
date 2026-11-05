# buildingway-ci

Shared GitHub Actions for Dalamud plugin projects: reusable workflows that a project calls from
thin caller workflows, and the composite action they use to install a pinned Dalamud.

| Workflow | Jobs (check names are `<caller job> / <job>`) | Inputs |
|---|---|---|
| `test.yml` | `test`; `revert-check` on pull requests (informs only; don't require it) | `test-project` (required), `revert-check` (default true), `dotnet-version` |
| `pr-build.yml` | `build`: format check, Release build, plugin artifact | `plugin` (required), `dotnet-version` |
| `workflow-lint.yml` | `actionlint`, `zizmor` | none |
| `release.yml` | `tag`, `build`, `publish`: tag main's `<Version>`, build, GitHub release, repo.json PR | `plugin` (required), `channel`, `checks`, `dotnet-version` |
| `changelog.yml` | `changelog` on pull requests: fails when plugin code changed but the changelog didn't (informs only; skipped for the `no-changelog` label and Dependabot) | `paths` (required), `changelog` (default `CHANGELOG.md`) |
| `mutation.yml` | `mutation`: Stryker.NET; pull requests test only changed code unless labelled `mutation` (informs only) | `test-project` (required), `dotnet-version` |

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

`changelog.yml` and `mutation.yml` read the caller's pull request labels, so the caller must
trigger on the label events (`labeled`, plus `unlabeled` for the changelog) and the repo needs the
`no-changelog` and `mutation` labels. Keep the caller's own triggers: a schedule for nightly
mutation runs, and a path filter naming the code under test.

`test.yml`'s `revert-check` runs `dotnet buildingway revert-check`, so the caller's
`.config/dotnet-tools.json` must list `Buildingway.Tools`
([buildingway-tools](https://github.com/Cellar-Mander/buildingway-tools)); set
`revert-check: false` if it doesn't.

## release.yml

The caller runs on a pushed `v*` tag and on `workflow_dispatch` with a `channel` choice. It must
set `concurrency: release` at its top level (two runs would race on repo.json; the reusable
workflow can't set the same group without cancelling itself) and grant the three permissions
the jobs need:

```yaml
# .github/workflows/release.yml in the plugin repo
name: Release
on:
  push:
    tags: [ 'v*' ]
  workflow_dispatch:
    inputs:
      channel:
        description: Channel to publish main's <Version> to
        type: choice
        options: [ testing, stable ]
        default: testing
concurrency: release
permissions: {}
jobs:
  release:
    permissions:
      actions: write        # dispatches the required checks on the repo.json PR
      contents: write       # pushes the tag, the release and the repo.json branch
      pull-requests: write  # opens the repo.json PR
    uses: Cellar-Mander/buildingway-ci/.github/workflows/release.yml@<sha> # v0.2.0
    with:
      plugin: MyPlugin
      channel: ${{ inputs.channel }}
```

The project needs `<plugin>/<plugin>.csproj` with `<Version>`, a Keep a Changelog `CHANGELOG.md`
with `## Unreleased` and `## <version>` sections, `repo.json` on main (a one-entry array once
released), `.github/dalamud.env` for install-dalamud, and in Settings → Actions → General
"Allow GitHub Actions to create and approve pull requests".

`checks` lists the workflow files of every required check (default
`test.yml workflow-lint.yml pr-build.yml`). Pushes and PRs made with `github.token` start no
workflows, so the release dispatches these on the repo.json branch; a required check missing
from the list never reports and blocks the PR, so each needs `workflow_dispatch` too.

What each step guards against is in the comments in `release.yml`.

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
