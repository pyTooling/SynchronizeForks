# Synchronize Forks

A reusable GitHub Action synchronizing forked repositories of a GitHub organisation or user account with their upstream
repositories.

Forks don't update themselves. GitHub offers a *Sync fork* button per repository and branch, but no automation for a
whole namespace. This action reads a list of forks and their branches from simple configuration files and synchronizes
them one by one — usually from a scheduled workflow running once a day.

The algorithm was previously duplicated as an inline shell script in every consuming repository's workflow. This action
carries the algorithm; a consuming repository keeps only its configuration files.

## License

This GitHub Composite Action (source code) is licensed under [The MIT License](LICENSE.md).

---

SPDX-License-Identifier: MIT
