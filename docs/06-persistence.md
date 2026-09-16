# 6. Persistence and review

SQLite behind a small store class, written in portable SQL so the same schema
moves to a shared database later without a rewrite. The database file lives on
a mounted volume so it survives container restarts.

## Tables

**`predictions`.** One row per processed document, keyed by the record key,
holding the stored response, the submission identity, the model version, the
configuration fingerprint and the timestamp. The stored response is what makes a
retry a replay.

**`decisions`.** Append only. One row per reviewer action: what the model
predicted, what the reviewer decided, the corrected value when there was one,
who decided it and when. Indexed on the record key, on the natural key of
submission and attribute, and on the decision kind.

**`review_queue`.** One row per attribute awaiting a person, keyed by record key
and attribute name, with the queued time and status. Indexed on status and
queue time so the oldest pending work is cheap to find.

**`config_log`.** One row per configuration fingerprint ever served, so a
prediction can be traced back to the exact thresholds that produced it.

## Keys

Two keys, doing different jobs.

- **The record key** is the caller's idempotency key. It answers "have I seen
  this delivery before".
- **The natural key** is submission identity plus attribute identifier. It
  answers "is this the same catalog row as something I already wrote", which is
  the question that matters when the same product arrives through two different
  documents.

## Submission states

A submission is `pending_review`, `complete` or `rejected`. Completion is
derived from the decision log rather than stored as a flag that can drift out of
step with it.

## The staging gate

Staging rows are only served once every attribute on a submission has been
decided. Asking before then returns a conflict, not a partial result.

## Review operations

Recording a decision validates that the record and attribute exist, rejects a
second decision on the same attribute as a conflict, appends to the log,
updates the queue, refreshes the queue depth gauge, and returns the new
submission state along with how many attributes are still pending. The caller
learns whether their decision completed the submission without having to ask a
second question.

## Duplicate handling

Recording a prediction under a key that already exists returns nothing rather
than raising. The duplicate is the normal case under retries, not an exception.
