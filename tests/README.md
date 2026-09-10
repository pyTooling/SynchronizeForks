# Test Fixtures

Configuration files used by [`Verification.yml`](../.github/workflows/Verification.yml). The action is run in dry-run
mode against them, so no repository is synchronized and no token is needed.

* `Valid` — one organisation, one commented out organisation, four repositories of which one is commented out, and
  four branches in total. Two lines carry tag patterns, one of them without any branch.
* `Invalid` — an organisation without its `<organisation>.repos` file, and four malformed repository lines:
  a missing `:`, a missing `=`, an empty `<upstream>` and an empty `<fork>`.
