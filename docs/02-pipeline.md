# 2. The pipeline

Four layers. Layer 1 belongs to the upstream document service and is described
here only as a contract.

## Layer 1: the input contract

One record per document, with six fields:

| Field | Meaning |
| --- | --- |
| `source_type` | Where the text came from, for example a datasheet or a catalog record |
| `text` | The raw text of the document |
| `structured_fields` | Extracted label and value pairs, for example "Output" and "4 to 20mA" |
| `normalized_units` | Quantities parsed into canonical units |
| `reference_condition_units` | Units that only apply under a stated reference condition |
| `source_ref` | A reference back to the source document |

Fixing this contract early was the single most useful integration decision.
Both services built against it independently and the first end to end call
worked.

## Layer 2: the rule engine

Three tiers, tried in order.

**Tier 1, exact part number.** A part number that matches the catalog exactly
is terminal: it identifies the product outright, so the semantic layers are
skipped entirely and the pipeline returns that match.

**Tier 2, manufacturer.** A fuzzy match on manufacturer name using a token set
ratio with a minimum score of 90. Token set matching tolerates word order and
extra words, so "Acme Controls Inc" matches "Acme Controls".

**Tier 3, numeric.** Compares quantities with their units against known values.

**The guardrail.** Every value a rule produces is checked against the catalog's
table of values that are valid for that attribute. A rule hit whose value is not
in that table is not returned as an answer and is not raised as an error.
It is demoted to zero confidence and flagged, and the semantic layer decides
instead. Rules must never invent a value the catalog does not contain.

Rule confidences are fixed rather than learned: 1.0 for an exact part number,
0.85 for a fuzzy manufacturer match, 0.65 for a partial match, 0 for no match.

## Layer 3: retrieval, consensus and value scoring

**Encode.** The product description becomes a 384 dimensional vector.

**Retrieve.** An approximate nearest neighbour index returns the 50 closest
catalog products.

**The novelty gate.** If the single closest product is not similar enough, the
pipeline stops and returns a rejection with the reason "unknown product type".
No attributes are predicted. This runs before the vote, because a vote among
poor neighbours still looks confident.

**Vote.** The 50 neighbours vote on the product type. The confidence is the
winning type's share of the ballot. Above 0.80 is high consensus, below 0.60 is
ambiguous and caps everything downstream.

**Score values.** The product type determines which attributes are in play. For
each attribute, every value the catalog has used for it is scored against the
query vector, and the top three are returned.

## Layer 4: fusion, caps and routing

**Fuse.**

```
confidence = alpha * rule_score + (1 - alpha) * semantic_score,  alpha = 0.7
```

divided by the weight actually applied when one of the two signals is absent.

**Caps, applied after fusion.**

| Condition | Cap |
| --- | --- |
| Product type vote below 0.60 | 0.75 |
| Winning value came from a cluster with fewer than five examples | 0.70 |

When both fire, the lower cap wins. Each cap raises a flag on the prediction so
a reviewer can see why the score was held down rather than guessing.

**Route.**

| Confidence | Destination |
| --- | --- |
| 0.85 and above | Auto accept into staging |
| 0.50 to 0.85 | Review queue |
| Below 0.50 | Held and flagged |

**Resolve attribute identifiers.** Attribute names are resolved to identifiers
in the context of the predicted product type, because the same attribute name
can exist under more than one identifier and the consumer writes by identifier.

## Reason codes

Every routed attribute carries codes explaining which part of the evidence was
weak:

| Code | Meaning |
| --- | --- |
| `RULE_NO_MATCH` | No deterministic rule supports the value |
| `MISSING_FIELD` | Neither a rule nor the semantic layer found any evidence |
| `LOW_SIMILARITY` | Semantic evidence exists but sits below the review floor |
| `AMBIGUOUS_VALUE` | The runner-up scores almost as high as the winner |
| `LOW_SAMPLE_VALUE` | Too few catalog examples for the winning value |
| `AMBIGUOUS_PRODUCT_TYPE` | The type vote was split and the score was capped |
| `UNKNOWN_PRODUCT_TYPE` | Nothing in the catalog resembles the product |

Two constants matter here and both came from a bug. Ambiguity is **relative**:
a runner-up within 5% of the winner is ambiguous. And any score below an
absolute floor of 0.01 is treated as no evidence rather than weak evidence.
The first version used an absolute margin, so two scores of 0.0001 and 0.0002
were reported as a close call between plausible values, when the truth was that
the system had nothing at all. Reason codes that lie are worse than no reason
codes, because a reviewer calibrates their trust on them.

## Submission identity

A submission is keyed by brand, supplier and SKU, hashed into a stable
identifier. The same triple always produces the same identifier, so a re-sent
document maps onto the rows it produced before instead of creating duplicates.

Two deliberate details:

- **Catalog product IDs are not part of the key.** Most supplier documents do
  not carry one, and a single product ID can cover several configured SKUs.
- **Option markers in a SKU are stripped.** Every configuration of one product
  shares an identity.
