# 8. Evaluation

## Protocol

- **Split at the product level**, stratified by product type, 80/10/10 with a
  fixed seed. Row level splitting leaks: one product's rows would land on both
  sides and the encoder would meet the same description twice.
- **Product types with fewer than three products go entirely to training**,
  since they cannot supply both a validation and a test example.
- **Each query product is excluded from its own neighbour list.** Without this
  the system retrieves itself and the metric measures memorization.
- Reported per run: product type accuracy, top-1 and top-3 value accuracy,
  expected calibration error over ten bins, Brier score, a coverage against
  precision sweep over thresholds, per product type breakdowns, a confusion
  table, failure cases and per query latency.

## Results

Held-out products, self-exclusion on. Roughly 13,000 query products and 143,000
graded attribute answers.

| Measure | Result |
| --- | --- |
| Product type accuracy | 95.5% |
| Value correct at rank 1 | 60.4% |
| Value correct in top 3 | 85.1% |
| Expected calibration error | 0.107 |
| Brier score | 0.235 |
| Coverage at the 0.85 threshold | 12.1% |
| Precision within that coverage | 82.4% |

The 25 point gap between rank 1 and top 3 is the number that justifies showing
a reviewer three candidates.

## The threshold sweep

The sweep is the artifact that makes the auto accept threshold a decision rather
than a guess. At the bottom of the range, coverage is 98.7% at 60.8% precision:
accept everything and be wrong two times in five. At 0.85, coverage falls to
12.1% and precision rises to 82.4%.

The bar this needs to clear is 95% precision. Nothing in the current sweep
reaches it, which is the honest conclusion: the threshold cannot be set
correctly from catalog text alone, and the missing input is real reviewed
decisions.

## Experiment one: the fusion normalization

The fusion formula assumed both signals were present. Most rows had no rule
signal, so their real signal was scaled by the smaller weight and could never
exceed 0.30, below every routing threshold.

Measured on the same held-out set, with the fix off and then on:

| Measure | Before | After |
| --- | --- | --- |
| Rows auto accepted | 0 | 17,367 |
| Precision of those rows | none | 82.4% |
| Expected calibration error | 0.414 | 0.107 |
| Product type accuracy | 0.957 | 0.957 |
| Top-3 accuracy | 0.851 | 0.851 |
| Top-1 accuracy | 0.604 | 0.604 |

The last three lines are identical because the change affects the final score
only, not the order candidates come out in. Nothing got worse.

The before run was produced on the same day with the fix switched off, and it
reproduced the earlier baseline exactly, which is what makes this a like for
like comparison rather than two different setups.

## Experiment two: the distance metric

Described in full in [models and math](03-models-and-math.md). Mahalanobis
scored 76.8% against 53.8% for Euclidean on homogeneous candidate pools, and
11.7% against 32.4% on mixed ones. The conclusion was to choose per attribute
pool rather than globally, implemented as a selectable policy with the original
behaviour kept as the default.

## What the numbers do not cover

- **Supplier datasheet text.** Everything above is catalog text, which is
  cleaner and closer to the answers. The production number is unknown.
- **Whether the confidence is a probability.** Calibration error is reported,
  it is not zero, and the score is described as a score.
- **Attributes the document never mentions.** The evaluation grades what was
  predicted, and the system currently predicts something for every attribute the
  type allows. Abstention is not yet measured because it is not yet implemented.
