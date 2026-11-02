# MASSter Agent Guide

Use this guide when inspecting, analysing, extending, or automating a MASSter
workflow. Treat the public package namespace as the stable interface:

```python
import masster

sample = masster.Sample("run.sample5")
study = masster.Study("study-folder")
wizard = masster.Wizard(source="raw-folder", folder="study-folder")
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

`Sample` represents one acquisition. Its principal tables and attributes are:

- `scans_df` and `ms1_df`: scan metadata and MS1 peaks (when loaded).
- `features_df`: detected chromatographic features.
- `id_df`: library matches for features.
- `lib_df` and `lib_ms2`: currently loaded identification library and MS2 fragments.
- `polarity`, `timestamp` (ISO-8601 acquisition time), `parameters`, and `history`.

Call `sample.close()` when finished with a raw/open acquisition to release open
file handles.

`Study` combines multiple samples. Its principal tables are:

- `samples_df`: sample metadata (`sample_type` in `"sample"`, `"qc"`, `"blank"`,
  `"ms2"`; `sample_group`; `sample_batch`; `sample_sequence`; `sample_timestamp`;
  `sample_color`) and per-sample counts (`num_features`, `num_ms1`, `num_ms2`).
- `features_df`: combined sample-level features.
- `consensus_df`: aligned and merged consensus features.
- `consensus_mapping_df` and `consensus_ms2`: feature-to-consensus links and
  consensus-associated MS2 spectra.
- `id_df`, `lib_df`, and `lib_ms2`: study-level identification results and library.
- `diff_df`: differential comparison definitions.

Use `feature_uid`, `sample_uid`, and `consensus_uid` as stable identifiers
(alongside numeric `feature_id`, `sample_id`, `consensus_id`, and `lib_id`).
Obtain the semantic table contract instead of guessing column names:

```python
masster.schema_names()
masster.describe_schema("features")
masster.validate_schema("consensus", study.consensus_df)
```

### Units

- `mz` in `features_df`, `consensus_df`, and `lib_df` is in **Da**.
- `rt`, `rt_start`, `rt_end`, `chrom_start`, and `chrom_end` in `features_df`
  and `consensus_df` are in **minutes**.
- Library `rt` (in `.csv`/`.json`/`lib_df`), `rt_delta` in `id_df`, `chrom_fwhm`
  in `find_features()`, and `rt_tol` parameters in `align()`, `merge()`, and
  `identify()` are in **seconds**.

## Storage formats

- **`.sample5` and `.study5` (format v4.0, SQLite)**: `sample.save()` and
  `study.save()` write SQLite v4.0 files by default (`masster.sqlite5`). Tables
  are stored as zstd-compressed Parquet blobs partitioned by read key (`ms1` by
  RT window, study `features` by `sample_id`), with lossless delta/grid
  encodings for spectra and chromatograms, atomic writes via temporary files,
  lock-free reads, in-place `ms2_all` catalogue storage, and partial loading
  (`sqlite5.load_sample`, `sqlite5.read_ms1`, `sqlite5.load_study`). Legacy HDF5
  files (`sample5` v1/v2, `study5` v1–v3) are auto-detected on load by header
  and can still be written with `sample.save(version="v2")` or
  `study.save(version="v3")`. Both `masster` and the read-only `masster-light`
  package load v4.0 SQLite and legacy HDF5 files transparently.
- **`.lib3` (SQLite) and `.lib5` (HDF5) libraries**: `lib_load()` accepts
  built-in metabolic models (`"hsapiens"`/`"human"`, `"scerevisiae"`/`"yeast"`,
  `"ecoli"`, `"core"`, `"aa"`, `"chebi"`), `.csv`, `.json`, `.msp`/`.msp.zip`,
  `.mgf`/`.mgf.zip`, `.lib5` (HDF5; build with `tools/msp2lib.py`), and `.lib3`
  (compact SQLite; build with `tools/msp2sqlite.py`). Save a loaded library with
  `sample.lib_save("lib.lib3")` or `study.lib_save("lib.lib3")`.

## Workflow order

### Single `Sample`

1. Load or construct a sample with MS1 data (`.wiff`/`.wiff2`, `.raw`, `.mzML`,
   `.mz5`, or `.sample5`; pass `sample_idx` for multi-sample `.wiff` files).
2. Run `find_features(noise=..., chrom_fwhm=..., chrom_peak_snr=...)`.
3. Optionally filter with `features_filter()` / `features_select()`, and run
   `find_ms2()`, `find_adducts()`, and `find_iso()`.
4. For identification:
   - MS1/RT matching: `lib_load()` then `identify()`.
   - MS1/MS2 spectral matching: `lib_load()` then `identify2(mslevel=[1, 2])`.
   - External MS2 annotations: `import_tima()` (folder or `.zip`) or
     `import_oracle()`.
5. Inspect with `info()`, `plot_*()`, or `dashboard()`, then `save()` or
   export (`export_mgf()`, `export_excel()`, `export_csv()`, `export_mztab()`,
   `export_features()`, `export_network()`, `export_history()`).

`find_iso()` is the canonical public method name. Do not substitute
`find_isotopes` or internal helper names.

### Multi-sample `Study`

1. Add or load samples (`add()`, `add_sample()`, or `load()`). `add()` orders
   `samples_df` chronologically by `sample_timestamp` into `sample_sequence` and
   assigns sample colors (`set_samples_color()` is canonical). Import or update
   sample metadata with `metadata_import()` or `samples_metadata.csv`.
2. Ensure sample-level features are available; optionally pre-filter with
   `study.features_filter(study.features_select(qc_prominence_sc=2.0))`.
3. Run `align(algorithm="kd", rt_tol=...)`.
4. Run `merge(method="qt" | "greedy", min_samples=..., rt_tol=..., mz_tol=..., pull_orphans=...)`
   to create `consensus_df`. `pull_orphans` recovers MS2 spectra for orphan
   consensus features from `.sample5` MS2 catalogues (skipping DIA/ZTScan runs).
5. Run `fill()` and `integrate(boundary_refine=...)` (use
   `boundary_refine="area"` when cumulative-area boundary refinement is desired).
6. Run `consensus_qualify()` to recompute integration-window QCs, calculate
   study-level QC metrics, and assign `stars`; optionally filter with
   `consensus_filter()` / `consensus_select()`.
7. Optionally associate MS2 with `find_ms2()` and identify compounds via
   `lib_load()` + `identify()` / `identify2()`, or `import_tima()` (folder or
   `.zip`) / `import_oracle()`, followed by `lib_filter()`, `lib_compare()`, and
   `id_update()`.
8. Inspect with `info()`, `plot_*()`, or `dashboard()`, then `save()` or export
   (`export_excel()`, `export_parquet()`, `export_csv()`, `export_mgf()`,
   `export_ms2()`, `export_mztab()`, `export_network()`, `export_history()`).

Check each step with `validate_workflow()` because valid alternatives exist,
such as loading a partially processed object from disk.

### Automated `Wizard`

For end-to-end batch processing from raw/open files to a finished study:

1. Construct `wiz = masster.Wizard(source=..., folder=..., polarity=..., num_cores=...)`.
2. Call `wiz.create_scripts()` to generate `1_processing.py`, `2_annotation.py`
   (Marimo annotation notebook), `3_analysis.py`, and `pyproject.toml`. Existing
   scripts in `folder` are preserved individually so user edits survive re-runs;
   `noise` and `chrom_fwhm` are sampled from the data only when `1_processing.py`
   is written.
3. Run `wiz.test_only()` to validate on one file, `wiz.test_and_run()` to test
   then execute the full batch, or `wiz.run(skip_study=...)` directly. Worker
   pools automatically retry unfinished files with fewer workers if a worker
   process dies.

## Quality control (`qc_*`) columns

All QC columns in `features_df` and `consensus_df` use the `qc_` prefix:

- **Chromatographic shape & window**: `qc_coherence`, `qc_sanity`,
  `qc_gaussianity`, `qc_prominence_sc`, `qc_area_diff_10`, `qc_area_diff_20`,
  `qc_area_diff_left_10`, `qc_area_diff_left_20`, `qc_area_diff_right_10`,
  `qc_area_diff_right_20` (and `*_mean` for `qc_area_diff_*` on `consensus_df`).
  At the study level, `integrate()` and `consensus_qualify()` compute these on
  the actual integrated window (`chrom_start` to `chrom_end`).
- **Oddity**: `qc_oddity` (0–1 shape-based score on `consensus_df`; 0–1
  cross-sample deviation score on `Study.features_df`) and `qc_oddity_reason`.
- **Study/sample-type QCs** (on `consensus_df` via `consensus_qualify()`):
  `qc_cv` (RSD across `"qc"` samples, or `"qc"` + `"sample"` if no QCs exist),
  `qc_to_blank` (signal enrichment over `"blank"` samples), `qc_dratio`
  (`SD(qc) / SD(sample)`), `qc_drift` (relative QC area drift across
  `sample_sequence`), `ms2_prec_purity` (precursor isolation purity), and
  `stars`.

Legacy `chrom_*` and `oddity` column/filter names are translated automatically
on load and in `features_select()` / `consensus_select()` / `consensus_filter()`
(`masster.qc_columns`), while retired columns (`qc_quality`, `qc_prominence`,
`qc_height_scaled`, and 50% area diffs) are dropped on load. Always use the
canonical `qc_*` names in new code.

## Parameters and safety

Inspect parameter contracts before changing them:

```python
masster.describe_parameters(sample)
masster.validate_parameters("find_features_defaults", {"noise": 100})
```

Workflow methods (`find_features`, `align`, `merge`, `fill`, `integrate`, ...) take
`**kwargs`, so read their parameters from the docstring: `help(study.merge)`,
`study.merge?` in Jupyter, or hover in marimo. It lists the key parameters first and then
every parameter with its type and default (generated from the `*_defaults` class, so it is
always current). Pass them as keyword arguments or as a ready-made `merge_defaults(...)`
via `params=`.

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

- Target Python 3.12–3.14 and Polars 1.38+ / 2.x (`engine="streaming"`,
  `with_row_index`; prefer native Polars expressions over `map_elements` so
  obfuscated builds do not fail on Python bytecode inspection).
- Use `spawn` multiprocessing pools where Polars has already run (`fork` can
  hang after multithreaded Polars execution).
- Use `MassterLogger` and `masster.logger.tqdm` (no `loguru`); keep compiled
  Numba kernels in `src/masster/_kernels.py` with complete NumPy fallbacks.
- Preserve public names and backwards-compatible aliases; `find_iso()` and
  `set_samples_color()` remain canonical.
- Keep description and validation helpers read-only and JSON-serializable, and
  update semantic schemas when a public table contract changes.
- Add focused tests for empty, partial, valid, and malformed objects, and run
  relevant tests and linting. Documentation builds may be offline; set
  `MASSTER_DOCS_INTERSPHINX=1` only when external cross-reference inventories
  should be fetched.

For user-facing details, read `docs/source/api/schema.rst`,
`docs/source/user_guide/data_model.rst`,
`docs/source/quality_control_assurance.md`, `docs/source/sqlite5_format.md`,
and `docs/source/lib3_format.md` after this guide.
