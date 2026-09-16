# 1. Problem and constraints

## The task

A supplier sends a datasheet for a product. Someone has to turn that document
into catalog rows: the product's type, and a value for each attribute that type
defines. Output quality matters more than throughput, because the catalog feeds
other systems and a wrong value propagates quietly.

The catalog holds a few hundred thousand products, several hundred product
types, and several hundred distinct attribute names. Any one product type
allows only a small subset of those attributes, usually a handful.

## Three properties that drove the design

### The label space is large and it moves

New attribute values appear constantly. A classifier trained over today's value
vocabulary is stale the moment a supplier introduces a new value, and retraining
to learn one value is an absurd cost. Anything that treats values as a fixed
label set fights the problem instead of fitting it.

### Training text and production text are not the same distribution

Everything measurable came from the catalog's own product descriptions. Those
descriptions were often written from the same source material as the attribute
values, so they state the answers almost word for word. A real datasheet uses
the supplier's language, its own abbreviations, and a table layout.

This means every accuracy number produced from catalog text is an upper bound
on production accuracy, and the honest thing to do is label it as such rather
than quote it as a production number.

### A confident wrong answer costs more than no answer

The system writes into a catalog. A human reviewer catching a low confidence
row is cheap. A wrong value auto accepted into the catalog is expensive and may
not be noticed for months. Every threshold, cap and gate in the system exists
to push errors toward the cheap failure.

## Constraints that came from outside

- **Reproducibility.** The same model version and the same input must give the
  same output. This ruled out anything that silently changes predictions
  between runs.
- **A frozen contract.** Several constants, including the fusion weight and the
  routing thresholds, were fixed by an agreed specification. Changing one is a
  design review, not a code change. This shaped how improvements ship: behind
  flags that reproduce the old behaviour exactly.
- **CPU only.** No GPU at inference time.
- **An upstream service owns document parsing.** This service never opens a
  PDF. It consumes a structured handoff record.

## What success looks like

Not "predict everything". The goal is to make each row either trustworthy
enough to accept without a person, or clearly marked for the person who has to
handle it, with enough evidence attached that handling it is fast.
