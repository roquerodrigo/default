# CLAUDE.md

This repository is data, not an application: the extensionless files at the root
(`appdaemon`, `integration`, `netdaemon`, `plugin`, `python_script`, `template`, `theme`)
are the JSON lists HACS shows in its store. `blacklist`, `removed` and `critical` track
repositories that were taken out. The Python under `scripts/` only validates and edits
these lists.

## Downstream consumers

- HACS clients read the published lists from `https://data-v2.hacs.xyz/<category>/repositories.json`
  (`scripts/check/existing.py` and `scripts/check/removed.py` query that same endpoint).
- Pushes to `master` that touch `critical` or `removed` upload them to that data store
  (`.github/workflows/upload-critical.yml`, `upload-removed.yml`). A malformed entry there
  reaches every HACS install, so both workflows re-validate against the JSON schema first.
- The upstream project is `hacs/default`; contributor PRs target its `master`.

## Editing the lists

- Every list must stay sorted case-insensitively (`sorted(..., key=str.casefold)`), not
  plain `sort`. CI fails otherwise (`scripts/is_sorted.py`). Fix with `scripts/sort`.
- Entries are `owner/repo` strings matching `^[\w\.-]+/[\w\.-]+$`
  (`tools/jsonschema/repositories.schema.json`). No URLs, no `.git` suffix.
- An "add" PR must add exactly one repository to exactly one category file.
  `scripts/changed/category.py` and `scripts/changed/repo.py` diff against a fresh clone of
  `hacs/default` at `/tmp/repositories/default` and exit with "Bad data" on anything else.
- Never delete an entry by hand. Use `scripts/remove_repository` (interactive) or
  `scripts/remove_archived_repo <owner/repo>`; both call `scripts/remove_repo.py`, which
  removes it from its category, appends it to `blacklist` and adds a record to `removed`
  (`repository` and `removal_type` required, per `tools/jsonschema/removed.schema.json`).
  `removed` itself is append-ordered, not sorted.
- Repositories owned by `custom-cards` or `custom-components` can be removed even if not
  present in any category (special case in `remove_repo.py`).
- Banned publishers live in `REMOVED_PUBLISHERS` in `scripts/remove_publishers.py`;
  `scripts/check/owner.py` imports that list to reject new submissions from them.
- `critical` entries require `repository`, `reason` and `link` (a URI).
- `scripts/add_repository` is a stub that only prints "Not implemented".

## CI checks (`.github/workflows/checks.yml`)

Every PR must be editable by maintainers (`scripts/check/edits.py`). The submission checks
only run when the PR carries the label `New default repository`. They verify: the PR author
is the owner or has at least a
third of the top contributor's commits, the repository has a GitHub release, it is not
already listed or previously removed, `hacs/action` passes, and (integrations only)
hassfest passes on the single `manifest.json` found in the repository.

## Running scripts locally

Run from the repository root; the check scripts import `scripts.helpers...`, so use module
form, e.g. `REPOSITORY=owner/repo python3 -m scripts.check.releases`. `owner.py` and
`edits.py` also need `GITHUB_EVENT_PATH` pointing at a pull request event payload, and the
GitHub API checks read `GITHUB_TOKEN`.
