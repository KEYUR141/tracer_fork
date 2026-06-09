# Contributing to TRACER

Thank you for your interest in contributing.

TRACER is a research-grade routing library with formal parity guarantees. The
project lives or dies on those guarantees holding up under measurement, so the
bar for changes is evidence, not intuition. Please read the standards below
before opening an issue or a PR. They exist to keep contributions focused and
to respect everyone's time, including yours.

## Contribution standards (read first)

TRACER is benchmark-driven. Every change that can affect routing behavior must
be justified with numbers, not opinion. Three rules follow from that.

1. **Adding a model needs a benchmark, not a name.** "We should add model X"
   is not a contribution. A model proposal must show, on a reproducible eval,
   that the new surrogate improves the coverage-vs-agreement frontier over the
   models already in the zoo (see "Adding a new surrogate model" below). An
   issue or PR that proposes a model without that evidence will be closed with
   a pointer back to this section. You are welcome to reopen it once you have
   the numbers.

2. **Changing a core parameter, threshold, or gate needs an ablation.** The
   acceptor threshold, the parity gate, the calibration logic, the default
   `FitConfig` values, and the split strategy are load-bearing for the parity
   guarantee. A change to any of them must include before/after numbers on the
   eval showing the frontier does not regress (and, ideally, improves). "It
   feels like it should be configurable" is not a justification. If you believe
   a parameter should be user-tunable, show a case where a different value wins
   on the benchmark.

3. **Bug reports need a reproduction.** The strongest reports name the affected
   files and lines and include a minimal script that reproduces the behavior.
   See issues #29, #30, and #31 for the standard we are looking for.

If your idea is exploratory ("how would TRACER handle X?", "is this a good fit
for Y?"), open a **GitHub Discussion** rather than an issue. Issues are for
actionable, evidenced work.

## What "benchmarked" means here

Run against a fixed, public eval so results are reproducible by a maintainer:

- Use a known dataset (the repo's test fixtures, or a public set such as
  Banking77) and a fixed `seed`.
- Report the metric that matters: **coverage at the target teacher agreement**
  (how much traffic the surrogate can take while holding parity), plus the
  realized teacher agreement on a held-out split (not the calibration split).
- Compare against the current behavior on `main`, not against nothing.
- Include the exact command and config you ran so the numbers can be checked.

A change that does not move (or that hurts) the frontier will not be merged,
no matter how clean the code is.

## Setup

```bash
git clone https://github.com/adrida/tracer
cd tracer
pip install -e ".[dev]"
```

## Running tests

```bash
pytest tests/ -v
```

Tests use synthetic data and run in temporary directories. No external
dependencies or API keys required.

## Quick sanity check

```bash
tracer demo
```

## Project structure

```
src/tracer/
  __init__.py            <- public exports (fit, update, load_router, report, embed, types)
  api.py                 <- public API (fit, update, load_router, report)
  config.py              <- FitConfig, EmbeddingConfig
  types.py               <- TraceRecord, QualitativeReport, ArtifactManifest, ...
  fit/
    pipeline.py          <- global / L2D / RSB pipeline construction + calibration
    surrogate.py         <- model zoo (LogReg, SGD, MLP, RF, ET, DT, GBT, XGB) + selection
  analysis/
    qualitative.py       <- XAI report: slices, boundary pairs, examples, deltas
    html_report.py       <- self-contained HTML audit report generator
  embeddings/
    index.py             <- FAISS wrapper + embed_texts (sentence-transformers)
    embedder.py          <- Embedder class (sentence-transformers, HTTP, callable)
  traces/
    loader.py            <- JSONL loader / writer + validation
  policy/
    artifacts.py         <- manifest, pipeline, qualitative report I/O
  runtime/
    router.py            <- production Router class
    serve.py             <- lightweight HTTP prediction server (stdlib only)
  cli/
    main.py              <- tracer CLI entry point (fit, report, update, demo, serve)
    _ui.py               <- terminal formatting and progress display
```

## Adding a new surrogate model

The surrogate zoo is not a popularity contest. We add a model only when it
earns a place on the frontier.

1. Add a factory to the `_candidates()` dict in `src/tracer/fit/surrogate.py`.
   The model must implement the scikit-learn `fit` / `predict` / `predict_proba`
   interface.
2. Include a benchmark in the PR description showing the new model wins (higher
   coverage at the target agreement, or equal coverage at lower latency) on a
   reproducible eval, against the existing zoo. Show the command and seed.
3. Account for the dependency cost. A new heavy dependency needs to pay for
   itself in the numbers, and should be optional where possible.

A PR that adds a model without (2) will be closed with a request for the
benchmark.

## Changing the gate, calibration, or default config

These are the parts that back the parity guarantee. A PR here must include an
ablation: the frontier on `main` versus the frontier with your change, on the
same eval and seed. State clearly which guarantee you are strengthening and
show it does not regress coverage or realized agreement.

## Adding a new pipeline family

Implement a `build_<name>(split, target_ta) -> dict` function in
`src/tracer/fit/pipeline.py` following the same structure as `build_global`,
`build_l2d`, and `build_rsb`. Register it in the `builders` dict inside
`fit_frontier`. Include a benchmark showing where the new family wins.

## Submitting a PR

1. Fork the repo and create a branch from `main`.
2. Make your changes with tests.
3. Run `pytest tests/ -v`. All tests must pass.
4. Open a pull request with a clear description of what changed, why, and (for
   anything touching routing behavior) the benchmark that backs it.
