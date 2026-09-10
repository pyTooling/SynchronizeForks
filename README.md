[![GitHub Workflow - Build and Test Status](https://img.shields.io/github/actions/workflow/status/pyTooling/SynchronizeForks/.github%2Fworkflows%2FPipeline.yml?branch=v1&logo=githubactions)](https://GitHub.com/pyTooling/SynchronizeForks/actions/workflows/Pipeline.yml)
[![Sourcecode License](https://img.shields.io/badge/code-MIT%20License-green?longCache=true&style=flat-square&logoColor=fff)](LICENSE.md)

# Synchronize Forks

A reusable GitHub Action synchronizing the branches and tags of forked repositories of a GitHub organisation or user
account with their upstream repositories.

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
        uses: pyTooling/SynchronizeForks@v1
        with:
          github-token: ${{ secrets.GH_TOKEN }}
```

That is the whole workflow. The forks are expected in the namespace the workflow is running in
(`${{ github.repository_owner }}`) and the configuration files are read from the repository's root directory; see
[Input Parameters](#input-parameters) to change either.

`@v1` is the major-version branch, moved to each release by the [pipeline](#pipeline). Pin `@v1.0.0` instead to hold
a single release.

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

`<upstream>=<fork>:<branches>[:<tagPatterns>]`

| Element         | Format                        | Meaning                                                                                                                                 |
|-----------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `<upstream>`    | `<organisation>/<repository>` | The repository the fork was created from. It's reported in the action's progress and error output, so it should name the real upstream.  |
| `<fork>`        | `<repository>`                | The forked repository. No organisation or account, because that's the `target-organisation` parameter.                                   |
| `<branches>`    | `<branch>[,<branch>,...]`     | Comma separated list of branches to synchronize. May be empty when tag patterns are given.                                               |
| `<tagPatterns>` | `<pattern>[,<pattern>,...]`   | *Optional.* Comma separated list of tag names or regular expressions. See [Tag Synchronization](#tag-synchronization).                   |

A line starting with `#` is a comment and skips that fork. Empty lines are ignored.

```
OSVVM/OSVVM=OSVVM:main,dev:v\d+\.\d+.*
OSVVM/OSVVM-Scripts=OSVVM-Scripts:main,dev
OSVVM/AXI4=OSVVM-AXI4:main,dev
OSVVM/Ethernet=OSVVM-Ethernet:main
#OSVVM/AvalonST=OSVVM-AvalonST:main
```

Read for the `PLC2` namespace, those five lines synchronize `PLC2/OSVVM` (branches `main` and `dev`, plus every tag
matching `v\d+\.\d+.*`) from `OSVVM/OSVVM`, `PLC2/OSVVM-Scripts` from `OSVVM/OSVVM-Scripts`, `PLC2/OSVVM-AXI4` from
`OSVVM/AXI4`, `PLC2/OSVVM-Ethernet` (branch `main` only) from `OSVVM/Ethernet`, and skip `PLC2/OSVVM-AvalonST`.

Both Unix and Windows line endings are accepted, and a missing final newline doesn't drop the last entry.

### The Log

Every organisation becomes a collapsible group, and each fork reports the branches it synchronized:

```
🏭 OSVVM
  📂 OSVVM/OSVVM ⇒ PLC2/OSVVM
    ✅ gh repo sync PLC2/OSVVM --branch main
    ✅ gh repo sync PLC2/OSVVM --branch dev
    🏷️ v2.1.0 — created from OSVVM/OSVVM@a1b2c3d
    🟰 12 tag(s) already up to date
  📂 OSVVM/AXI4 ⇒ PLC2/OSVVM-AXI4
    🌱 dev — created from OSVVM/AXI4@a1b2c3d
  📂 OSVVM/Ethernet ⇒ PLC2/OSVVM-Ethernet
    ❌ gh repo sync PLC2/OSVVM-Ethernet --branch main
    ↪ failed to sync: HTTP 404: Not Found
  🚫 #OSVVM/AvalonST=OSVVM-AvalonST:main

Summary:
  Synchronized branches:  2
  Created branches:       1
  Created tags:           1
  Skipped entries:        1
  Errors:                 1

Not synchronized:
  ❌ OSVVM/Ethernet ⇒ PLC2/OSVVM-Ethernet:main
```

### The Symbols

| Symbol | Meaning                                                        |
|:------:|----------------------------------------------------------------|
| 🏭     | An organisation from the index file — a collapsible log group.  |
| 📂     | A fork, and the upstream repository it follows.                 |
| ✅     | A branch was synchronized.                                      |
| 🌱     | A branch was created in the fork.                               |
| 🏷️     | A tag was created in the fork.                                  |
| 🟰     | Tags that already point at the same object as upstream.         |
| ☢️     | A tag moved upstream — refused, see [below](#tag-synchronization). |
| ℹ️     | Nothing to do: no tags upstream, or no tag matched.             |
| 🚫     | A commented out organisation or fork.                           |
| 🚧     | Dry-run: what would have happened.                              |
| ❌     | An error.                                                       |

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
| `created-tags` | Number of tags created in a fork.                                       |
| `skipped`      | Number of skipped organisations and repositories (commented out lines). |
| `errors`       | Number of counted errors.                                               |

In dry-run mode, `synchronized` counts the branches that *would* have been synchronized and the configured tag
patterns are printed. No repository is read, so a missing branch isn't detected and `created` and `created-tags` stay
`0`.

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

## Tag Synchronization

`gh repo sync` knows branches only — a fork's tags are never updated by it, which is why a fork drifts behind its
upstream in releases even while its branches are current. `Paebbels/ghdl` is a live example: `ghdl/ghdl` has 46 tags,
the fork has 19.

The optional fourth field of a configuration line says which tags to follow:

```
ghdl/ghdl=ghdl:master:v\d+\.\d+.*
OSVVM/OSVVM=OSVVM:main,dev:nightly,v\d+\.\d+\.\d+
antonblanchard/microwatt=microwatt::v\d+\.\d+
```

A pattern is either a fixed tag name or a regular expression, and it has to match the **whole** tag name. Patterns are
comma separated, and a line may carry tags without any branch, as the third line shows. Matching is done with
`grep -P`, so PCRE syntax including `\d`, `\w` and `{n,m}` is available. Beware that a fixed name is a regular
expression too: `v1.0` matches `v1x0` as well.

For every matching tag of the upstream repository:

* the fork doesn't have it → it's created, pointing at the same object:  
  `🏷️ v2.1.0 — created from OSVVM/OSVVM@a1b2c3d`
* the fork has it, at the same object → counted as up to date, reported as one line per repository:  
  `🟰 12 tag(s) already up to date`
* the fork has it, at a **different** object → the tag moved upstream. It's reported as an error and **left alone**,
  because rewriting it would silently discard whatever the fork's tag points at. Both sides are resolved to the commit
  they point at, with its date, so the report says what each tag means and which of the two is older:

  ```
      ☢️ v1.0.0 — moved in 'OSVVM/OSVVM'
        ↪ fork:     90e6af7  2024-03-11 14:22:05 UTC
        ↪ upstream: d3d07ba  2025-07-02 09:41:18 UTC
  ```

  The run continues with the next tag. Delete the tag in the fork to let the next run recreate it.

  An annotated tag is dereferenced, so the commit shown is the one the tag ultimately points at rather than the tag
  object. When both sides resolve to the *same* commit, the tag object itself was recreated — a re-signed or
  re-worded tag over unchanged code — and the report says so.

Tags are never deleted from the fork, and a tag that exists only in the fork is left untouched.

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
* a tag moved in the upstream repository, or a tag can't be created in the fork,
* the tags of a repository can't be read, or the runner's `grep` has no `-P` support.

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

The configuration file format is backwards compatible — a line without the tag field behaves exactly as before. Two
fields are worth a look while converting: `<upstream>` is reported in the progress and error output, so a copied
placeholder there makes the log name the wrong repository, and adding tag patterns is what stops the fork from drifting
behind in releases.

## Pipeline

[`Pipeline.yml`](.github/workflows/Pipeline.yml) runs the action against the fixtures in `tests/` in dry-run mode —
three jobs, no token, no repository touched. Beyond that it releases itself, with reusable workflows from
[pyTooling/Actions](https://github.com/pyTooling/Actions):

* **`PrepareJob.yml`** classifies the run — which branch or tag, a regular or a merge commit, a release commit or a
  release tag — so the two jobs below know whether they apply.
* **`TagReleaseCommit.yml`** turns a merge commit on `main` into a tag named after the release pull-request's title
  (`v1.0.0`), and starts this pipeline again on that tag. A tag pushed with the automatic `GITHUB_TOKEN` doesn't
  trigger a workflow, which is why the pipeline is dispatched explicitly — and why this file has to be called
  `Pipeline.yml`.
* **`PublishReleaseNotes.yml`** runs on the tagged pipeline, once the same three jobs pass, and publishes the release
  page. Its description is the body of the pull-request that produced the tagged merge commit; the workflow finds that
  pull-request from the merge commit itself.

* **`UpdateVersionBranch.yml`** — a local reusable workflow, running beside the release page. Consumers pin a major
  version (`pyTooling/SynchronizeForks@v1`), so each release has to move that branch. It opens a pull-request from
  `main` to `v<major>`, titled `Updating v1 from v1.0.1`, which is reviewed and merged like any other.

  A new major gets a new branch: `v2.0.0` creates `v2` **from `v1`**, not from `main` — a branch created from `main`
  would already be identical to it, and there would be nothing to open a pull-request about. Adding version branches
  is therefore just a matter of tagging a new major, and tags come from release pull-request titles.

  The branch prefix is an input, so the same workflow produces `r1` for a repository that names its branches that
  way. If the version branch is already level with `main`, nothing is opened; if a pull-request from the previous
  release is still open, it is retitled rather than duplicated.

  **References to this repository are rewritten for the branch.** A workflow, action or badge on `v1` has to
  reference `v1`, not `main` — otherwise the branch runs someone else's code and its badge shows someone else's
  status. Where a rewrite is needed, it becomes a commit on an `update/v1` branch and the pull-request is opened from
  there; where nothing needs rewriting, the pull-request is a plain merge of `main`. Only **self**-references are
  touched — `pyTooling/Actions@r8` and `actions/checkout@v7` are separate decisions and are left alone.

So a release is one merge: open a `dev` → `main` pull-request titled `vMM.mm.pp`, write the release notes in its
description, and merge it. What follows is automatic, except for the version-branch pull-request, which waits for a
review.

## Dependencies

* [GitHub CLI (`gh`)](https://cli.github.com/), pre-installed on GitHub-hosted runners.
* `grep` with PCRE support (`-P`), for tag patterns only. GNU grep on the Linux runners has it.

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
