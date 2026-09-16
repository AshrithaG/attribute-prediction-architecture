# 11. Gaps and roadmap

## Edge cases the system handles

| Case | Behaviour |
| --- | --- |
| Product type absent from the catalog | Rejected by the novelty gate, no attributes predicted |
| Two similar product types | Vote lands between the bands, confidence capped, routed to review |
| Value with very few examples | Simpler distance, confidence capped, reason code attached |
| Exact part number match | Terminal, returns the identified product without semantic scoring |
| A rule producing a value not in the catalog | Demoted to zero and flagged, semantic layer decides |
| The same document delivered twice | Stored response replayed |
| The same product arriving in two documents | Mapped to one submission identity |

## Gaps

**Abstention.** The system ranks candidates for every attribute the product type
allows, whether or not the document mentions that attribute. It should say
nothing instead. This is the clearest remaining weakness and it inflates the
apparent error rate, because a row that should have been empty is counted as a
wrong value.

**The extracted label and value pairs are underused.** The upstream service
already provides them. Matching those labels to attribute names directly, then
using the semantic layer for what the labels do not cover, is a more direct path
to the answer than inferring everything from the description.

**Values as quantities.** Ranges, either-or lists and units are scored as text.
Comparing them as quantities is a better fit for the data.

**Calibration.** The confidence is not a probability and should not be described
as one until reviewed decisions exist to calibrate against.

**Thin product types.** Some types define only two or three attributes, so even
a perfect system can only fill those. This is a data question for the catalog
owner rather than a modelling one, and it is measurable: a sizeable share of
catalog rows carry an attribute their product type does not list, so nothing can
reach them.

## Deliberately out of scope

- **Document parsing.** A separate service extracts text and structured fields.
- **The reviewer interface.** The queue is an API and a table. Someone still has
  to build the screen.
- **Industry standard taxonomy mapping.** The integration is built and tested.
  The mapping file is a third party dependency that had not arrived.
- **Cloud deployment and writeback into the catalog system.** Owned elsewhere.

Naming what is out matters as much as naming what is in. Most of the risk in
this project sat in the seams between those pieces rather than inside any one of
them.

## What I would do next, in order

1. **Measure on real supplier datasheets.** Twenty to thirty documents whose
   correct values are already known. Every number in this write-up is
   provisional until this exists.
2. **Implement abstention**, using the extracted label and value pairs to decide
   which attributes the document actually speaks to.
3. **Match labels to attributes directly**, and combine that with the existing
   value scoring.
4. **Score values as quantities**, including ranges and units.
5. **Calibrate the auto accept threshold** against real reviewed decisions, then
   enable auto accept for the attributes that clear the bar, rather than for
   everything at once.
6. **Move the prediction store** from a single file database to a shared one,
   which the portable schema already allows.
