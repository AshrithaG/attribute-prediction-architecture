# 4. Architecture decisions

Each decision is written as options, choice, reason and trade-off. The last
four were reversals, which are the ones worth reading.

## 1. Retrieval and statistics, not a trained classifier

**Options.** Fine-tune a classifier over attribute values. Train per attribute
models. Retrieve similar catalog products and reuse their values.

**Choice.** Retrieval, with per value statistics on top.

**Why.** The value vocabulary changes constantly. A classifier needs retraining
to learn a value that retrieval picks up as soon as one example exists.
Retrieval also hands the reviewer something a classifier cannot: the specific
catalog products that supported the answer.

**Trade-off.** Accuracy is bounded by how well the catalog's own text separates
the classes, and the index must be rebuilt when the catalog shifts.

## 2. Predict the product type first, then its attributes

**Choice.** One product type decision gates everything downstream.

**Why.** It turns a several hundred way problem into a handful per row, and the
type is the easiest thing to read off a description. The cheapest decision
carries the expensive ones.

**Trade-off.** A wrong type poisons every attribute under it, which is why it
gets its own confidence bands, its own cap, and its own rejection path.

## 3. Return three candidates, not one

**Choice.** Each attribute returns a ranked top three.

**Why.** The consumer is a person clearing a queue, and picking from a short
list beats typing into an empty box. The measured gap between first guess and
top three is 25 points. Optimizing the single answer would have been optimizing
the wrong thing.

## 4. Fixed rule confidences rather than learned ones

**Choice.** Rule tiers emit constants: 1.0, 0.85, 0.65, 0.

**Why.** These are contract values that other teams reason about. A learned
rule confidence would move under them for no practical gain, since the tiers
are already ordered by how much evidence they represent.

## 5. Rules may never invent a value

**Choice.** Every rule produced value is validated against the catalog's table
of values allowed for that attribute. A value that is not there is demoted to
zero confidence and flagged, not returned and not raised as an error.

**Why.** A rule firing on a plausible looking string is exactly the confident
wrong answer the system exists to avoid. Demoting rather than raising keeps the
semantic layer in play, so the attribute still gets an answer.

## 6. Two signals, and a fusion rule that survives a missing one

**Choice.** Combine rule and semantic scores at a fixed weighting, and divide by
the weight actually applied when one signal is absent.

**Why.** The original formula assumed both signals were present. Rules only fire
for part numbers, manufacturers and a few clear cases, so most rows had no rule
signal, and their one real signal was multiplied by the smaller weight. That
capped them below every routing threshold: they could not be auto accepted and
could not even reach the review queue. They fell into the bottom bucket
regardless of what the model thought.

**Trade-off.** This changes a frozen contract value, so it ships behind a flag
that reproduces the old behaviour exactly, and the before and after were
measured in the same run.

## 7. A novelty gate on similarity, not on vote share

**Choice.** Refuse to predict when the closest catalog product is not similar
enough, independent of neighbour agreement.

**Why.** Vote share cannot detect a product the catalog has never seen, because
neighbours agree happily even when all of them are poor. The two populations
separate cleanly on top-1 similarity and not at all on vote share.

**Trade-off.** It refuses a small fraction of genuine products.

## 8. Confidence is a score, not a probability

**Choice.** Report calibration error next to accuracy, and refuse to call the
number a probability until reviewer decisions exist to calibrate against.

**Why.** Thresholds at 0.85 invite everyone to read "85% likely correct". It is
not that yet. The dependency is worth stating plainly rather than papering over,
because the people who would act on the misreading are the client's.

## 9. Learning from reviewers is deliberate, not automatic

**Options.** Update value statistics online as corrections arrive, or fold them
into a new version on demand.

**Choice.** Online updates off by default. Retraining is a command, and it never
promotes its own output.

**Why.** Reproducibility was a client requirement. Online updates change
predictions without changing the version number, so two identical documents get
different answers on different days and nothing explains why. A correction still
takes effect immediately as an exact match rule, which is the fast path that
actually matters, without moving the statistical model underneath it.

**Trade-off.** Improvements land more slowly and need a person to run and
approve them.

## 10. A registry with an approval gate and a rollback

**Choice.** Versions are registered with their metrics and a manifest.
Promotion compares against the active version's headline metrics with a
tolerance of 0.005, requires a named approver, and is appended to a promotion
log. Rollback restores the previous pointer.

**Why.** Newer is not better. The gate turns a regression into an explicit
decision, and the log answers what was live on a given date.

## 11. Idempotent delivery keyed by the caller's key

**Choice.** The caller sends an idempotency key. The first call computes and
stores the response. Retries return the stored response, marked as replayed.

**Why.** The upstream service retries on transient failures. Without this, a
retry creates a second prediction and a second queue item for one document.

## 12. An append-only decision log

**Choice.** Reviewer decisions are inserted, never updated in place. The current
state of a row is derived from the log.

**Why.** "Who decided what, when, and what did the model say at the time" is the
question that gets asked months later, usually when something is wrong.

## 13. Staging is gated on a complete review

**Choice.** A submission's staging rows are only available once every attribute
on it has been decided.

**Why.** A half reviewed submission written to the catalog is worse than one
that waits. It looks complete and it is not.

## 14. CPU only, pinned encoder, model baked into the image

**Choice.** No GPU. The encoder is pinned to an exact revision, copied into the
container at build time, and the container runs with model hub access disabled.

**Why.** Latency is already milliseconds per document on CPU. The index and
statistics are only valid for the exact embeddings that built them. Baking the
model in also means the container starts identically in an environment with no
outbound network, which is where it will actually run.

## 15. Artifacts are mounted, not baked

**Choice.** The opposite decision for the index, clusters and calibration table:
they are mounted at runtime.

**Why.** They are gigabytes, and they change on a different cadence than the
code. A new model version should not require a new image.

## 16. Attribute identifiers resolved per product type

**Choice.** Resolve names to identifiers in the context of the predicted type.

**Why.** The same attribute name exists under more than one identifier, and the
consumer writes by identifier. Resolving by name alone writes the right word to
the wrong field, which is invisible until someone reads the catalog.

## 17. Reason codes on every routed row

**Choice.** Each row carries codes describing which evidence was weak.

**Why.** A confidence number tells a reviewer how much to trust a row, not what
to do about it. In aggregate the codes also turn the queue into a diagnosis of
where the system is weak, which is how the abstention gap was found.

## 18. Reversal: the rebuild was abandoned and the original service restored

**What happened.** A second version of the service was built as a cleaner
rewrite. On inspection against the integration contract it dropped two of the
six input fields, had no authentication, was not safe under the upstream
service's retries because it wrote on every call, and was not what the upstream
connector had been built against.

**Choice.** Ship the original, keep the rewrite in the repository, unshipped.

**Why.** A cleaner internal structure does not outweigh a contract the other
team already implements. The rewrite is not deleted, because the parts of it
that were genuinely better are worth keeping visible.

**Cost of the detour.** The restructure had broken the container build, the
dependency list, the CI gates and ten scripts including the whole artifact
rebuild chain. Restoring it was most of a day.

## 19. Reversal: ambiguity became relative, with an evidence floor

**What happened.** The first reason code implementation flagged
`AMBIGUOUS_VALUE` whenever the top two scores were within a fixed absolute
margin. It fired constantly on rows where both scores were near zero, where the
real problem was no evidence at all.

**Choice.** Ambiguity is relative to the winner, at 5%, and scores below an
absolute floor of 0.01 get a no evidence code instead.

**Why.** Reviewers calibrate their trust on these codes. A code that cries
ambiguity on empty evidence trains people to ignore it.

## 20. Reversal: evaluation now excludes each product from its own vote

**What happened.** The evaluation queried an index that contained the query
product, so every product retrieved itself as its own nearest neighbour.

**Choice.** Exclude the query product from its own neighbour list.

**Why.** The earlier numbers measured memorization. The corrected numbers are
lower and they are the ones that carry forward. A metric you cannot defend is a
liability, especially one a client is making decisions against.
