# 3. Models and math

## Representation

A pretrained sentence encoder, BAAI `bge-small-en-v1.5`, maps each product
description to a 384 dimensional vector. Vectors are L2 normalized, so an inner
product is a cosine similarity. Inference is CPU only, in batches of 128, which
amortizes per batch overhead and is two to four times faster than small batches
with identical output.

The model is used off the shelf. Nothing is fine-tuned. A fine-tuning scaffold
exists in the codebase but is not wired into the service, for two reasons:
it needs labelled pairs the project does not have yet, and any change to the
encoder invalidates the index and every value statistic built from it.

The encoder revision is pinned to an exact commit hash. An unpinned model would
quietly change the embeddings while the stored artifacts continued to describe
vectors that no longer exist. The alternative encoders that were considered are
recorded in configuration with a note that enabling one requires a full
re-evaluation: a smaller and weaker model, and a larger one roughly three times
slower.

## Retrieval

Catalog embeddings live in a FAISS `IVFFlat` index.

| Parameter | Value | Why |
| --- | --- | --- |
| Metric | Inner product | Vectors are L2 normalized, so this is cosine |
| `nlist` | 512 | Voronoi cells the vectors are partitioned into |
| `nprobe` | 16 | Cells searched per query |
| `top_k` | 50 | Neighbours returned |
| Training sample | 100,000 vectors, fixed seed | Trains the cell centroids |

Exact search would be accurate and unnecessary. The vote needs the
neighbourhood, not a perfectly ordered list. Raising `nprobe` to 32 was tried
and did not pay for itself.

Rebuild cost is dominated by encoding the catalog, on the order of an hour and
a half of CPU time. Building the index from those vectors takes seconds. That
asymmetry is why the index is treated as an artifact to version rather than
something to regenerate casually.

## Product type consensus

```
pt_conf = votes for the winning type / total votes
```

over the 50 neighbours. Above 0.80 is high consensus, below 0.60 is ambiguous.

## The novelty gate

The gate measures something the vote cannot: the similarity of the single
nearest catalog product.

Vote share is a share of a ballot. It stays high whenever the neighbours agree
with each other, however poor all of them are. Measured here, out of catalog
text reached a vote share of 0.80 and adjacent industrial text reached 1.00,
while their nearest neighbour similarity never exceeded 0.76. Held-out catalog
products sat at 0.832 at the first percentile. A threshold of 0.80 on top-1
similarity separates the two populations with a margin.

The cost is that the gate refuses 0.67% of genuine catalog products. That is
the right direction to be wrong in, and the threshold is configurable, with 0
disabling the gate.

## Value scoring

Every combination of product type, attribute and value becomes a cluster of the
embeddings of the products that use it. Each cluster stores a mean and a
covariance estimated with Ledoit-Wolf shrinkage, which is what makes a
covariance usable when the number of examples is close to the number of
dimensions. The inverse is precomputed.

A candidate scores as

```
conf_embed(A, v) = exp(-D^2 / (2 * sigma^2))
```

where `D^2` is the squared Mahalanobis distance from the query to that cluster
and `sigma` is a per product type temperature. Mahalanobis rather than
Euclidean because a value whose examples are tightly grouped should punish a
distant query harder than a value whose examples are spread out.

Clusters with fewer than five members do not get an inverse covariance at all.
They fall back to squared Euclidean distance, and their confidence is capped
downstream.

## The scale mismatch between those two branches

This is the most interesting measurement in the project.

Inverting a 384 by 384 covariance estimated from 8 to 28 points produces `D^2`
in the thousands. The identity branch is bounded near 0 to 4. On one product
type the medians were about 10,653 and 0.40, a factor of roughly 26,600. After
the exponential, every low sample cluster scores near 1.0 and every well
estimated one scores near 0.0, so the ranking is decided by whether a cluster
has fewer than five members rather than by similarity. Shrinkage does not close
a gap that size at this ratio of samples to dimensions.

Measured on 300 held-out products and 2,766 graded answers:

| Candidate pool | n | Mahalanobis | Euclidean |
| --- | --- | --- | --- |
| Every cluster the same kind | 1,627 | 76.8% | 53.8% |
| Mixed kinds | 1,139 | 11.7% | 32.4% |

Mahalanobis is far stronger on a uniform pool and collapses on a mixed one.
Ranking only ever happens within one attribute, so the fix is to choose per
attribute: Mahalanobis when that attribute's candidate pool is homogeneous,
identity when it is mixed. The policy is implemented and selectable. The
default stays on the original behaviour so existing runs reproduce exactly,
because changing a default belongs to a release, not to the commit that
discovered the problem.

## Calibrating sigma

For each product type in the validation split, a grid search picks the sigma
that minimizes

```
val_loss(sigma) = Brier(conf_embed, correct) + lambda * ECE(conf_embed, correct, bins = 10)
```

with `lambda` at 0.5 and a grid of eight candidates spanning 0.5 to 300.

Brier alone rewards sharpness and tolerates miscalibration, while ECE alone is
satisfied by a model that is uniformly unsure. The combination asks for scores
that are both informative and honest about their own reliability.

One implementation detail is worth recording, because it made the sweep
practical: `D^2` does not depend on sigma. Computing it once per query and
cluster pair and replaying the exponential for each candidate sigma makes an
eight point grid roughly ten times cheaper than rescoring from scratch.

The result is a table of per type sigma values, stored as an artifact next to
the index and the clusters.

## The online feedback math

Implemented, tested, and switched off by default. It is described here because
the decision to disable it only makes sense alongside what it does.

When a reviewer confirms value `v` for attribute `A` on query embedding `q`:

```
mu_new = (N * mu_old + q) / (N + 1)
N      = N + 1
```

When a reviewer corrects a confidently wrong prediction from `v_wrong` to
`v_true`, the wrong cluster is pushed away from the query and the right one gets
the confirm update:

```
mu_wrong = mu_old - lambda * (q - mu_old),   lambda = 0.01
```

Updates append to a crash-safe log immediately and mutate the in-memory store at
once. The large centroid file is not rewritten per update. It is regenerated on
demand, and on startup the log is replayed on top of the last snapshot so no
update is lost. A cross-platform advisory file lock guards both the append and
the in-memory mutation so two reviewers never interleave a torn write.

The reason this is off by default is in [decision 9](04-decisions.md).
