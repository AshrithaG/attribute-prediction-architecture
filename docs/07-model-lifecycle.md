# 7. Model lifecycle

## What a model version is

A version is a directory of artifacts, not a file of weights, because nothing is
trained. It contains:

- the FAISS index over catalog embeddings
- the product identifiers aligned to the index
- cluster statistics: mean, shrunk covariance and inverse per value
- the per product type calibration table
- a hash of the encoder that produced the vectors
- build information: encoder identity, dimensions, counts, index parameters and
  build timings
- a manifest with provenance and the evaluation metrics of that version

An `ACTIVE` pointer names the version being served. The service resolves its run
directory from the registry unless one is given explicitly.

## The build stages

Each stage is a script, and each writes artifacts the next one reads.

| Stage | What it does |
| --- | --- |
| Splits | Product level stratified split into train, validation and test |
| Index build | Encode the catalog and build the FAISS index |
| Cluster build | Build per value statistics from the training split |
| Calibration | Grid search per product type sigma on the validation split |
| Evaluation | Score the held-out split and write a report bundle |
| Serve | Run the API against a chosen version |
| Load test | Latency and throughput against a running service |

Splitting at the product level rather than the row level is a correctness
requirement, not a preference. A product's attribute rows must not straddle the
split, or the encoder sees the same description at both training and evaluation
time.

## The registry

| Operation | Behaviour |
| --- | --- |
| `register` | Record a version with its metrics and provenance in a manifest |
| `gate` | Compare a candidate against the active version on the headline metrics, with a tolerance of 0.005 |
| `promote` | Move the active pointer, requiring a named approver, appending to a promotion log |
| `rollback` | Restore the previous pointer |
| `history` | The promotion log |

The gate compares product type accuracy, top-1, top-3 and calibration error. A
candidate that is worse on any of them beyond the tolerance is blocked, and the
failures are reported individually rather than as a single pass or fail.

Promotion requires an approver name because "who signed off on this" is a
question that gets asked after an incident, and an automated promotion has no
answer to it.

## Retraining

Retraining is a command someone runs. It:

1. Builds a fresh candidate version from the catalog.
2. Folds recorded reviewer decisions into that version's cluster statistics.
3. Evaluates the candidate on the held-out split.
4. Registers it with its metrics and its provenance.

It stops there. It never promotes. The serving model never changes underneath
its version number, so the same version and input always give the same output.

Folding decisions is done against the candidate's own clusters, and a decision
whose cluster no longer exists is skipped rather than aborting the run, because
the catalog moves between builds.

## How a correction takes effect

Two paths, deliberately separated:

- **Immediately**, as an exact match rule. The same supplier wording maps to the
  corrected value on the next document, with no retraining.
- **Eventually**, as a shift in the statistics, but only through a new version
  that a person evaluated and approved.

The fast path is the one that matters day to day. The slow path is the one that
has to be reproducible.
