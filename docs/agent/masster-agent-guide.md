# MASSter Agent Guide

Use this guide when inspecting, analysing, extending, or automating a MASSter
workflow. Treat the public package namespace as the stable interface:

```python
import masster

sample = masster.Sample("run.sample5")
study = masster.Study("study-folder")
```

Do not rely on private modules or mutate dataframe internals unless a task
explicitly requires a low-level implementation change.

## Start with inspection

Before choosing an operation, inspect state and check its prerequisites. These
helpers are read-only and JSON-serializable; they neither load data nor run
processing.

```python
state = masster.describe_state(sample_or_study)
workflow = masster.describe_workflow(sample_or_study)
report = masster.validate_state(sample_or_study)
ready = masster.validate_workflow(sample_or_study, "find_ms2")
```

Only call an operation after its workflow report is valid. If it is invalid,
use the returned error messages to identify the missing prerequisite. Do not
invent missing data or mark workflow stages complete manually.

## Core object model

`Sample` represents one acquisition. Its principal tables are:

- `features_df`: detected chromatographic features.
- `id_df`: library matches for features.
- `lib_df`: currently loaded identification library.

`Study` combines multiple samples. Its principal tables are:

- `samples_df`: sample metadata and processing statistics.
- `features_df`: combined sample-level features.
- `consensus_df`: aligned and merged consensus features.
- `id_df` and `lib_df`: study-level identification results and library.

Use `feature_uid`, `sample_uid`, and `consensus_uid` as stable identifiers.
Feature and consensus `mz` values are in Da; their `rt` values are in minutes.
Obtain the full contract instead of guessing column names:

```python
masster.schema_names()
masster.describe_schema("features")
masster.validate_schema("consensus", study.consensus_df)
```

## Workflow order

For a `Sample`, use this normal progression:

1. Load or construct a sample with MS1 data.
2. Run `find_features()`.
3. Optionally run `find_iso()`, `find_adducts()`, and `find_ms2()`.
4. Load a library with `lib_load()` before `identify()`.
5. Save or export results.

`find_iso()` is the canonical public method name. Do not substitute an older
or internal isotope-method name.

For a `Study`, use this normal progression:

1. Add or load samples.
2. Ensure sample-level features are available.
3. Run `align()`.
4. Run `merge()` to create `consensus_df`.
5. Optionally fill, integrate, associate MS2, and identify.
6. Save or export results.

Check each step with `validate_workflow()` because valid alternatives exist,
such as loading a partially processed object from disk.

## Parameters and safety

Inspect parameter contracts before changing them:

```python
masster.describe_parameters(sample)
masster.validate_parameters("find_features_defaults", {"noise": 100})
```

Prefer documented public methods. Preserve existing `history`, stable IDs, and
table columns. Do not overwrite raw data, delete result tables, or force
alignment/merge state to bypass validation unless the user explicitly requests
a destructive repair and its target is verified.

## Programmatic agent tools

Use the small `masster.agent` namespace when an agent is called from code:

```python
report = masster.agent.inspect(sample)
readiness = masster.agent.check(sample, "find_iso")
result = masster.agent.execute(sample, "find_iso")
```

`inspect()` and `check()` are read-only. `execute()` is explicit and only
dispatches allowlisted processing operations after a successful preflight. It
does not expose arbitrary methods, exports, or filesystem operations. See the
executable examples in `examples/agent/`.

## Public distribution and version matching

The public distribution publishes this guide, the portable skill, examples,
and `agent-manifest.json` together as a versioned agent bundle. Before using a
bundle, compare `agent-manifest.json["masster_version"]` with
`masster.__version__`. Use the release-matched bundle for reproducible or
offline agent environments.

## Implementation work

When changing MASSter itself:

- Preserve public names and backwards-compatible aliases; `find_iso()` remains
  canonical.
- Keep description and validation helpers read-only and JSON-serializable.
- Update semantic schemas when a public table contract changes.
- Add focused tests for empty, partial, valid, and malformed objects.
- Run the relevant tests and linting. Documentation builds may be offline; set
  `MASSTER_DOCS_INTERSPHINX=1` only when external cross-reference inventories
  should be fetched.

For user-facing API details, read `docs/source/api/schema.rst` and
`docs/source/user_guide/data_model.rst` after this guide.
