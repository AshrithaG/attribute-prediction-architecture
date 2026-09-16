# 10. Engineering practice

## Gates

Every pull request runs, on Python 3.12:

- `ruff` with a broad rule set, including docstring, annotation and naming rules
- `black` and `ruff format`
- `mypy --strict` over the source and scripts
- `pytest` with a branch coverage floor of 85%, currently at 93%
- a secrets scanner in the pre-commit hooks of the product repository

Turning these on partway through a project surfaced several hundred existing
violations. Two things made that manageable: excluding directories that are
data or prototypes rather than product code, and keeping intentional exceptions
narrow and documented, for example the mathematical unicode used in the scoring
code.

## Tests

Around 400 tests. The ones worth describing:

- **Contract tests** that run the real upstream delivery client against the
  real service in a container, covering a known product, an unknown product, a
  record carrying reference condition units, a wrong token, a malformed record,
  and a duplicate delivery. This is the test that proved the integration before
  either side shipped.
- **Idempotency tests** asserting a replay returns the stored response and
  creates no second queue item.
- **Review lifecycle tests** covering the queue, decisions, conflicts on double
  decisions, and the staging gate refusing partial submissions.
- **Registry tests** covering a blocked promotion, an approved one and a
  rollback.
- **A reproducibility test** asserting that the pinned encoder revision is what
  the artifacts were built with.
- One wall clock performance test that is skipped in CI, because shared runners
  are too slow for a timing assertion to mean anything. It runs locally.

## Merging the service into the product repository

The service had to move into the repository holding the upstream service, with
its history, and without carrying anything the client should not receive.

The import script splits the service directory into its own history, rewrites
that history so every file sits under a subdirectory and every excluded path is
absent from every commit, creates a branch, and merges. Excluded: all
documentation and office files, demos, archived code, reports, client data and
model artifacts. Then it verifies:

- no file outside the target subdirectory changed
- no excluded path exists in any imported commit
- no model binaries or catalog data are present
- no documentation files are present
- files the root ignore rules would otherwise swallow are tracked
- new files in those directories are still addable

It pushes nothing. Verification failures leave the branch in place for
inspection with the exact command to remove it.

Rewriting the history rather than squashing kept authorship intact, so every
contributor's commits survive the move.

## Mistakes worth recording

**A lint exclusion that was wider than intended.** The pattern excluding a top
level data directory also matched a source package with the same name, so two
modules were never linted. It surfaced only because the other repository's
hooks pass explicit file names, which bypass exclusion rules. Anchoring the
pattern to the project root fixed it. The general lesson: a check that silently
covers less than you think is worse than no check, and running the same tool two
different ways is how you find out.

**A debug log that evaluated its arguments eagerly.** Logging calls built
strings from a field that did not exist on the object, at a level that was
usually off. Guarding on whether the level is enabled fixed both the cost and
the crash.

**Formatter disagreement.** Two formatters disagreed on one long assertion
message. Rewriting the line so both agreed was cheaper than configuring around
it.

**A secrets scanner flagging a model revision hash.** It looks exactly like a
credential. An inline allowlist comment is the right answer, since the hash has
to stay in version control for reproducibility.
