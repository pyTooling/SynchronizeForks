# Test Fixtures

Configuration files used by [`Verification.yml`](../.github/workflows/Verification.yml). The action is run in dry-run
mode against them, so no repository is synchronized and no token is needed.

* `Valid` — one organisation, one commented out organisation, three repositories of which one is commented out, and
  four branches in total.
* `Invalid` — an organisation without its `<organisation>.repos` file, and two malformed repository lines.
