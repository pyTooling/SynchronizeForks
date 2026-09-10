[![GitHub Workflow - Build and Test Status](https://img.shields.io/github/actions/workflow/status/pyTooling/SynchronizeForks/.github%2Fworkflows%2FVerification.yml?branch=dev&logo=githubactions)](https://GitHub.com/pyTooling/SynchronizeForks/actions/workflows/Verification.yml)
[![Sourcecode License](https://img.shields.io/badge/code-MIT%20License-green?longCache=true&style=flat-square&logoColor=fff)](LICENSE.md)

# Synchronize Forks

A reusable GitHub Action synchronizing forked repositories of a GitHub organisation or user account with their upstream
repositories.

Forks don't update themselves. GitHub offers a *Sync fork* button per repository and per branch, but no automation for
a whole namespace. This action reads a list of forks and their branches from simple configuration files and
synchronizes them one by one — usually from a scheduled workflow running once a day.

The algorithm used to be an inline shell script, copied into every consuming repository's workflow. This action carries
the algorithm; a consuming repository keeps only its configuration files.

## Usage

A repository using this action contains a workflow and its configuration files:

```
SynchronizeForks/
├── .github/
│   └── workflows/
│       └── Synchronize.yml
├── .ALL.repos
├── OSVVM.repos
└── _Others.repos
```

### The Workflow

The action needs write access to the contents of every listed fork, so a workflow's automatic `GITHUB_TOKEN` is **not**
sufficient — it's scoped to the repository the workflow is running in. Create a personal access token with that access
and store it as a repository secret (`GH_TOKEN` below).

```yaml
name: Synchronize forked repositories

on:
  push:
  schedule:
    # Every day at 05:50 (UTC+1) — check the upstream repositories for updates.
    - cron: '50 4 * * *'

jobs:
  Synchronize:
    runs-on: ubuntu-latest
    steps:
      - name: ⏬ Checkout
        uses: actions/checkout@v6

      - name: 🔄 Synchronize forked repositories
        uses: pyTooling/SynchronizeForks@main
        with:
          github-token: ${{ secrets.GH_TOKEN }}
```

That is the whole workflow. The forks are expected in the namespace the workflow is running in
(`${{ github.repository_owner }}`) and the configuration files are read from the repository's root directory; see
[Input Parameters](#input-parameters) to change either.

> [!NOTE]
> No version tag has been released yet, so the examples reference `@main`. Once released, pin a version tag as usual.

### The Entry Point: `.ALL.repos`

The index file is the entry point. It lists the **upstream organisations** to be synchronized, one per line. For each
listed name, a matching `<organisation>.repos` file is read from the same directory — so `OSVVM` below reads
`OSVVM.repos`.

A line starting with `#` is a comment and skips that organisation, including its whole file. Empty lines are ignored.

```
# Upstream organisations, one per line. Each needs a matching '<organisation>.repos' file.
OSVVM
VHDL

# Single repositories from various organisations and private accounts.
_Others
```

The last entry is a convention rather than a rule: single repositories from many different accounts don't deserve one
file each, so they are collected in an `_Others` (or `_Misc`) file.

### An Organisation File: `OSVVM.repos`

Each of these files lists one fork per line, in the format:

`<upstream>=<fork>:<branches>`

| Element      | Format                                            | Meaning                                                                                                                                 |
|--------------|---------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `<upstream>` | `<organisation>/<repository>`                     | The repository the fork was created from. It's reported in the action's progress and error output, so it should name the real upstream.  |
| `<fork>`     | `<repository>`                                    | The forked repository. No organisation or account, because that's the `target-organisation` parameter.                                   |
| `<branches>` | `<branch>[,<branch>,...]`                         | Comma separated list of branches to synchronize.                                                                                        |

A line starting with `#` is a comment and skips that fork. Empty lines are ignored.

```
OSVVM/OSVVM=OSVVM:main,dev
OSVVM/OSVVM-Scripts=OSVVM-Scripts:main,dev
OSVVM/AXI4=OSVVM-AXI4:main,dev
OSVVM/Ethernet=OSVVM-Ethernet:main
#OSVVM/AvalonST=OSVVM-AvalonST:main
```

Read for the `PLC2` namespace, those five lines synchronize `PLC2/OSVVM` (branches `main` and `dev`) from
`OSVVM/OSVVM`, `PLC2/OSVVM-Scripts` from `OSVVM/OSVVM-Scripts`, `PLC2/OSVVM-AXI4` from `OSVVM/AXI4`,
`PLC2/OSVVM-Ethernet` (branch `main` only) from `OSVVM/Ethernet`, and skip `PLC2/OSVVM-AvalonST`.

Both Unix and Windows line endings are accepted, and a missing final newline doesn't drop the last entry.

### The Log

Every organisation becomes a collapsible group, and each fork reports the branches it synchronized:

```
🏭 OSVVM
  📂 OSVVM/OSVVM ⇒ PLC2/OSVVM
    ✅ gh repo sync PLC2/OSVVM --branch main
    ✅ gh repo sync PLC2/OSVVM --branch dev
  📂 OSVVM/AXI4 ⇒ PLC2/OSVVM-AXI4
    🌱 dev — created from OSVVM/AXI4@a1b2c3d
  📂 OSVVM/Ethernet ⇒ PLC2/OSVVM-Ethernet
    ❌ gh repo sync PLC2/OSVVM-Ethernet --branch main
    ↪ failed to sync: HTTP 404: Not Found
  🚫 #OSVVM/AvalonST=OSVVM-AvalonST:main

Summary:
  Synchronized branches:  2
  Created branches:       1
  Skipped entries:        1
  Errors:                 1

Not synchronized:
  ❌ OSVVM/Ethernet ⇒ PLC2/OSVVM-Ethernet:main
```

### Input Parameters

| Parameter             | Required | Default                          | Description                                                                                                          |
|-----------------------|:--------:|----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `github-token`        |  **yes** |                                  | GitHub token used to synchronize the forked repositories. It needs write access to the contents of every listed fork. |
| `target-organisation` |    no    | `${{ github.repository_owner }}` | GitHub organisation or user account owning the forked repositories.                                                  |
| `directory`           |    no    | `'.'`                            | Directory containing the configuration files.                                                                        |
| `index-file`          |    no    | `'.ALL.repos'`                   | Name of the index file listing the organisations to be synchronized.                                                 |
| `force`               |    no    | `false`                          | Hard reset the fork's branch to the upstream branch, discarding commits that aren't in the upstream repository.       |
| `create-missing-branches` | no   | `false`                          | Create a listed branch that doesn't exist in the fork yet from the upstream repository's branch head.                |
| `dry-run`             |    no    | `false`                          | Print the synchronization commands instead of running them.                                                          |
| `fail-on-error`       |    no    | `true`                           | Let the action fail if at least one error was counted.                                                               |

### Output Parameters

| Parameter      | Description                                                             |
|----------------|---------------------------------------------------------------------------|
| `synchronized` | Number of successfully synchronized branches.                           |
| `created`      | Number of branches created in a fork.                                   |
| `skipped`      | Number of skipped organisations and repositories (commented out lines). |
| `errors`       | Number of counted errors.                                               |

In dry-run mode, `synchronized` counts the branches that *would* have been synchronized. Dry-run reads no repository,
so a missing branch isn't detected and `created` stays `0`.

## Missing Branches

`gh repo sync` updates a branch; it can't create one. A branch listed in an `<organisation>.repos` file that the fork
doesn't have yet — a branch added upstream after the fork was created, or a fork made with *Copy the default branch
only* — therefore can't be synchronized at all.

With `create-missing-branches: true`, the action creates it from the upstream repository's branch head, and the next
run synchronizes it like any other branch:

```
  📂 OSVVM/AXI4 ⇒ PLC2/OSVVM-AXI4
    🌱 dev — created from OSVVM/AXI4@a1b2c3d
```

**It's off by default**, because this is where `<upstream>` stops being decoration: it's the repository the new
branch's head is read from, and a stale or copy-pasted upstream would create the branch from the wrong repository.
Check that field, then enable it per repository. While it's disabled, a missing branch is a counted error naming the
parameter.

No clone, fetch or push is involved. GitHub keeps a fork and its upstream in one object network, so the upstream's
commit is addressable through the fork and the branch is created with a single API call. If the branch exists in
neither repository — usually a typo in the configuration file — it's a counted error.

## Error Handling

Each of these is counted, reported as a GitHub Actions error annotation, and lets the action fail at the end — unless
`fail-on-error` is disabled:

* the index file doesn't exist,
* an `<organisation>.repos` file listed in the index file doesn't exist,
* a repository line is malformed — no `=`, no `:`, or an empty `<upstream>`, `<fork>` or branch list,
* `gh repo sync` fails for a fork's branch. Its output is quoted below the failed command, and the annotation names the
  upstream repository, the fork and the branch,
* a branch exists neither in the fork nor in the upstream repository, or creating it fails,
* a branch is missing from the fork while `create-missing-branches` is disabled.

A failing fork doesn't stop the run: every other fork is still synchronized, and the summary lists what wasn't.

## Users

| Repository                                                                | Namespace   | Synchronizes                                        |
|---------------------------------------------------------------------------|-------------|-----------------------------------------------------|
| [Paebbels/SynchronizeForks](https://github.com/Paebbels/SynchronizeForks) | `Paebbels`  | GHDL, OSVVM, VHDL and various single repositories.  |
| [VHDL/Synchronize](https://github.com/VHDL/Synchronize)                   | `VHDL`      | OSVVM.                                              |
| [PLC2/Synchronize](https://github.com/PLC2/Synchronize)                   | `PLC2`      | OSVVM, VHDL, and vendor and infrastructure forks.   |

Each of them still carries its own copy of the inline script and is converted to this action once a version is
released.

## Migrating an existing repository

A repository that carries the inline script keeps its `*.repos` files and replaces the two script steps of its workflow
with the `uses:` step shown above. `targetOrganisation=<name>`, the constant that had to be edited in every copy,
becomes the `target-organisation` parameter — and can be dropped where the namespace is the one the workflow runs in.

The configuration file format is unchanged. One field is worth a look while converting: `<upstream>` is reported in the
progress and error output, so a copied placeholder there makes the log name the wrong repository.

## Dependencies

* [GitHub CLI (`gh`)](https://cli.github.com/), pre-installed on GitHub-hosted runners.

## Contributors

* [Patrick Lehmann](https://GitHub.com/Paebbels) (Maintainer)
* [and more...](https://GitHub.com/pyTooling/SynchronizeForks/graphs/contributors)

### Credits

This action is the algorithm of [Paebbels/SynchronizeForks](https://github.com/Paebbels/SynchronizeForks), extracted
from the workflow it lived in. Its copyright is carried over accordingly: Patrick Lehmann from 2024, The pyTooling
Authors from 2026.

See also: [Syncing a fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork)

## License

This GitHub Composite Action (source code) is licensed under [The MIT License](LICENSE.md).

---

SPDX-License-Identifier: MIT
