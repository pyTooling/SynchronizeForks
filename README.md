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

The token needs write access to the contents of every listed fork, so a workflow's automatic `GITHUB_TOKEN` is not
sufficient — it's scoped to the repository the workflow is running in. Store a personal access token as a repository
secret instead.

```yaml
name: Synchronize forked repositories

on:
  push:
  schedule:
    # Every day at 05:50 (UTC+1)
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

By default, the forks are expected in the namespace the workflow is running in (`${{ github.repository_owner }}`) and
the configuration files are read from the repository's root directory.

> [!NOTE]
> No version tag has been released yet, so the examples reference `@main`. Once released, pin a version tag as usual.

### Input Parameters

| Parameter             | Required | Default                            | Description                                                                                                                          |
|-----------------------|:--------:|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `github-token`        |  **yes** |                                    | GitHub token used to synchronize the forked repositories. It needs write access to the contents of every listed fork.                 |
| `target-organisation` |    no    | `${{ github.repository_owner }}`   | GitHub organisation or user account owning the forked repositories.                                                                  |
| `directory`           |    no    | `'.'`                              | Directory containing the configuration files.                                                                                        |
| `index-file`          |    no    | `'.ALL.repos'`                     | Name of the index file listing the organisations to be synchronized.                                                                 |
| `force`               |    no    | `false`                            | Hard reset the fork's branch to the upstream branch, discarding commits that aren't in the upstream repository.                       |
| `dry-run`             |    no    | `false`                            | Print the synchronization commands instead of running them.                                                                          |
| `fail-on-error`       |    no    | `true`                             | Let the action fail if at least one error was counted.                                                                               |

### Output Parameters

| Parameter      | Description                                                                     |
|----------------|---------------------------------------------------------------------------------|
| `synchronized` | Number of successfully synchronized branches.                                   |
| `skipped`      | Number of skipped organisations and repositories (commented out lines).         |
| `errors`       | Number of counted errors.                                                       |

In dry-run mode, `synchronized` counts the branches that *would* have been synchronized.

## Configuration File Formats

The `.ALL.repos` file is the entry point. It lists one upstream organisation per line, or a comment starting with `#`.
When many single repositories from various organisations or private accounts are synchronized, an `_Others` or `_Misc`
entry is recommended.

**Example:**
```
ghdl
OSVVM
#Skipped for now
_Others
```

Each listed organisation has a matching `<organisation>.repos` file containing one repository per line, or a comment
starting with `#`. A repository line has the following format:  
`<upstream>=<fork>:<branches>`

* `<upstream>` is formatted like `<organisation>/<repository>` or `<privateAccount>/<repository>`.  
  It documents where the fork came from; the upstream repository itself is resolved by `gh repo sync` from the fork.
* `<fork>` is formatted like `<repository>`.  
  An organisation or account is not required, because it's given by the `target-organisation` parameter.
* `<branches>` is a comma separated list of branch names like `<branch>,<branch>,<branch>`.

**Example:**
```
OSVVM/OSVVM=OSVVM:main,dev
OSVVM/AXI4=OSVVM-AXI4:main,dev
#OSVVM/AvalonST=OSVVM-AvalonST:main
```

Empty lines are ignored. Both Unix and Windows line endings are accepted, and a missing final newline doesn't drop the
last entry.

## Error Handling

Each of these is reported as a GitHub Actions error annotation and counted, and the action fails at the end unless
`fail-on-error` is disabled:

* the index file doesn't exist,
* an `<organisation>.repos` file listed in the index file doesn't exist,
* a repository line is malformed (no `=` or no `:`),
* `gh repo sync` fails for a fork's branch — its output is quoted below the command and in the annotation.

A failing repository doesn't stop the run: every other fork is still synchronized.

## Migrating an existing repository

A repository that carries the inline script keeps its `*.repos` files, and replaces the two script steps of its
workflow with the `uses:` step shown above. The configuration file format is unchanged.

## Dependencies

* [GitHub CLI (`gh`)](https://cli.github.com/), pre-installed on GitHub-hosted runners.

## Contributors

* [Patrick Lehmann](https://GitHub.com/Paebbels) (Maintainer)
* [and more...](https://GitHub.com/pyTooling/SynchronizeForks/graphs/contributors)

### Credits

This action is the algorithm of [Paebbels/SynchronizeForks](https://github.com/Paebbels/SynchronizeForks), extracted
from the workflow it lived in.

See also: [Syncing a fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork)

## License

This GitHub Composite Action (source code) is licensed under [The MIT License](LICENSE.md).

---

SPDX-License-Identifier: MIT
