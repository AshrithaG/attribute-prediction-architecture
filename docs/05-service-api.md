# 5. Service and API

A FastAPI application. Every endpoint except the health and metrics endpoints
requires a bearer token, supplied to the process through an environment
variable.

## Endpoints

| Method and path | Purpose |
| --- | --- |
| `POST /predict` | Predict attributes for one document handoff record |
| `GET /review/queue` | Items waiting for a person, paged |
| `POST /review/decisions` | Record one accept, correct or reject |
| `GET /submissions` | Submissions, filterable by state |
| `GET /submissions/{record_key}` | One submission with its predictions and decisions |
| `GET /submissions/{record_key}/staging` | Catalog-ready rows, once review is complete |
| `GET /stats` | Counts by routing outcome and review state |
| `POST /feedback` | Online cluster update, disabled by default |
| `GET /healthz` | Liveness, with artifact and configuration fingerprints |
| `GET /metrics` | Prometheus metrics |

## Idempotency

`POST /predict` accepts an `Idempotency-Key` header. The key becomes the record
key. If a response is already stored under that key, it is returned verbatim
with a `replayed` flag, and the pipeline is not run again. When no key is sent,
the source reference is used, and failing that a random key is generated so
anonymous calls never collide.

Storing the response rather than recomputing it also means a retry is cheap,
which matters because the upstream connector retries on every transient error.

## Error semantics

The distinction that mattered to the upstream team is permanent against
transient, because it decides whether they retry.

| Situation | Status | Upstream behaviour |
| --- | --- | --- |
| Bad or missing token | 401 | Permanent, no retry |
| Malformed record | 422 | Permanent, no retry |
| Unknown product type | 200 with a rejection body | Delivered successfully, nothing to retry |
| Review state conflict | 409 | Permanent |
| Unknown record key | 404 | Permanent |
| Persistence disabled but a store endpoint called | 503 | Transient |

A rejection is deliberately not an error. The document was processed correctly
and the answer is "this is not something I can predict", which is a result, not
a failure.

## Response shape

A prediction response carries, beyond the per attribute rows:

- the resolved product type and its vote confidence
- the resolved manufacturer
- a catalog match when an exact part number identified the product outright
- the submission identity and the record key
- the neighbouring catalog products that supported the prediction
- whether the item was queued for review
- whether this response was replayed
- a rejection block when the novelty gate fired

Each attribute row carries the attribute identifier, the predicted value, the
confidence, the routing outcome, the reason codes, the alternatives, and the
flags saying which caps were applied.

## Metrics

Prometheus counters, gauges and histograms, prefixed with the service name:

| Metric | Type | What it answers |
| --- | --- | --- |
| `requests_total` | Counter | Traffic by endpoint and outcome, including replays |
| `predict_latency_ms` | Histogram | Per document latency |
| `conf_final` | Histogram | Distribution of final confidence |
| `pt_conf` | Histogram | Distribution of product type consensus |
| `conf_drift_kl` | Gauge | Divergence of the live confidence distribution from the evaluation baseline |
| `predictions_routed_total` | Counter | Rows by routing outcome |
| `review_decisions_total` | Counter | Reviewer decisions by kind |
| `review_queue_pending` | Gauge | Depth of the review queue |
| `feedback_total` | Counter | Online updates, when enabled |

The drift gauge is the one that earns its place. It compares the shape of live
confidence scores against the distribution seen during evaluation, which is the
earliest signal that production text has drifted away from catalog text.

## Health

`GET /healthz` reports the model version, the number of loaded clusters, the
encoder identity, and a configuration fingerprint. It also verifies at startup
that the encoder currently loaded matches the one the artifacts were built with,
and refuses to start when it does not. A service that silently serves an index
built by a different encoder produces confident nonsense.
