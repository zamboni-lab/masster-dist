---
name: masster
description: Use when working with MASSter mass-spectrometry analyses, including Sample, Study, or Wizard workflows, feature detection, alignment, consensus merging, gap filling, integration, quality control (qc_* columns and consensus_qualify), isotope/adduct/MS2 processing, compound identification (identify, identify2, TIMA, LipidOracle, .lib3/.lib5 libraries), .sample5/.study5 v4.0 SQLite and HDF5 storage, semantic schemas, validation, automation, or related repository changes. Read the repository-backed guide before choosing operations or modifying public workflow behavior.
---

# MASSter

Read [the MASSter Agent Guide](../../docs/agent/masster-agent-guide.md) before
working with MASSter objects or APIs.

## Key rules

- **Public namespace**: Use `masster` (`Sample`, `Study`, `Wizard`, `Lib`,
  `Spectrum`, `Chromatogram`, `masster.agent`, and the top-level `describe_*` /
  `validate_*` schema helpers).
- **Preflight first**: Inspect `Sample` and `Study` state with
  `describe_state()`, `describe_workflow()`, `validate_state()`, and
  `validate_workflow()` (or `masster.agent.inspect()` / `masster.agent.check()`)
  before running state-changing operations.
- **Canonical workflow & method names**:
  - `Sample`: `load()` -> `find_features()` -> optional `features_filter()` ->
    `find_ms2()`, `find_adducts()`, `find_iso()` (canonical; do not use
    `find_isotopes`) -> `lib_load()` + `identify()` / `identify2()` (or
    `import_tima()` / `import_oracle()`) -> `save()` / export -> `close()`.
  - `Study`: `add()` / `add_sample()` / `load()` (use `set_samples_color()`, not
    `set_sample_color`) -> optional `features_filter()` -> `align()` ->
    `merge()` -> `fill()` -> `integrate()` -> `consensus_qualify()` -> optional
    `consensus_filter()` -> `find_ms2()` -> `lib_load()` + `identify()` /
    `identify2()` (or `import_tima()` / `import_oracle()`, `lib_filter()`,
    `lib_compare()`, `id_update()`) -> `save()` / export.
  - `Wizard`: `create_scripts()` (preserves existing scripts in the output
    folder) -> `test_only()`, `test_and_run()`, or `run(skip_study=...)`.
- **Storage & libraries**:
  - `.sample5` and `.study5` default to v4.0 SQLite (`masster.sqlite5`) while
    loading legacy HDF5 files transparently (and supporting `masster-light` for
    read-only/post-processing workflows).
  - `lib_load()` and `lib_save()` support `.lib3` (SQLite) and `.lib5` (HDF5)
    libraries in addition to built-in models, `.csv`, `.json`, `.msp`, and
    `.mgf`.
- **QC columns**: Use canonical `qc_*` names (`qc_coherence`, `qc_sanity`,
  `qc_gaussianity`, `qc_prominence_sc`, `qc_area_diff_10`, `qc_area_diff_20`,
  `qc_area_diff_left_10`, `qc_area_diff_left_20`, `qc_area_diff_right_10`,
  `qc_area_diff_right_20`, `qc_oddity`, `qc_oddity_reason`, `qc_cv`,
  `qc_to_blank`, `qc_dratio`, `qc_drift`, `ms2_prec_purity`, and `stars`).
  Legacy `chrom_*` names are mapped automatically on load, and retired columns
  (`qc_quality`, `qc_prominence`, `qc_height_scaled`, and 50% area diffs) are
  dropped.
- **Units**: `mz` is in Da; `rt` in `features_df` and `consensus_df` is in
  minutes; library `rt`, `id_df.rt_delta`, `chrom_fwhm`, and `rt_tol` parameters
  are in seconds.
- **Parameters**: Workflow methods take `**kwargs`. `help(study.merge)` (or `?` in
  Jupyter, hover in marimo) shows the key parameters first, then every parameter
  with type and default, generated from the matching `*_defaults` class. Use it
  instead of guessing names or defaults.
