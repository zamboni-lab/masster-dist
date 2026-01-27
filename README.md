# masster

[![PyPI - Version](https://img.shields.io/pypi/v/masster)](https://pypi.org/project/masster/)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/masster)](https://pypi.org/project/masster/)

## About this repository (`masster-dist`)

This repository is the **distribution repository** for MASSter:
- GitHub Releases here contain **platform/Python-specific wheels**.
- The **clear** pure-Python wheel (`py3-none-any`) is published to **PyPI** and is not mirrored here.
- Source code and development happen in the main project repo (`zamboni-lab/masster`).

**MASSter** is a Python package for the analysis of metabolomics experiments by LC-MS/MS data, with a main focus on the challenging tasks of untargeted and large-scale studies.  

## Installation

Install from this repository’s GitHub Releases (wheels). Pick the wheel that matches:
- your OS (`win_amd64`, `manylinux_2_28_x86_64`, `macosx_...`)
- your Python minor version (`cp311`, `cp312`, `cp313`)

### `pip`

Option A: install from a downloaded wheel file:
```bash
pip install ./masster-<version>-cp<py>-cp<py>-<platform>.whl
```

Option B: install directly from a GitHub Release URL:
```bash
pip install https://github.com/zamboni-lab/masster-dist/releases/download/<tag>/masster-<version>-cp<py>-cp<py>-<platform>.whl
```

### `uv`

Option A: install from a downloaded wheel file:
```bash
uv pip install ./masster-<version>-cp<py>-cp<py>-<platform>.whl
```

Option B: install directly from a GitHub Release URL:
```bash
uv pip install https://github.com/zamboni-lab/masster-dist/releases/download/<tag>/masster-<version>-cp<py>-cp<py>-<platform>.whl
```

Release assets include `SHA256SUMS.txt` for verification.

## Background and motivation

MASSter is actively used, maintained, and developed by the Zamboni Lab at ETH Zurich. The project started in 2025 because many needs were unmet by common software packages (mzMine, MS-DIAL, W4, etc.), such as performance, scalability, sensitivity, robustness, speed, rapid implementation of new features, and integration into ETL systems.

As of January 2026, the codebase includes over 60,000 lines of code and offers a wide range of possibilities. MASSter has become the workhorse for all of our work, from R&D to routine analyses, from small to large-scale studies, from DDA to DIA, across all vendors.

We decided to make MASSter public primarily for these reasons:
* **FAIRification of data processing**: We want others to be able to reproduce our results starting from raw data, using fully parameterized scripts that do not depend on human interaction.
* **Empowering other labs**: To allow other labs to immediately benefit from the latest developments we implement.

## Content

MASSter is designed to handle LC-MS/MS data, regardless of whether it is DDA, DIA/SWATH, or DIA/ZTScan. We still rely on the potent and fast feature detection provided by OpenMS, but we have redesigned everything else: centroiding, RT alignment, adduct and isotopomer detection, merging of multiple samples, gap-filling, quantification, extraction of the best MS2 spectra across the entire study, visualization, and export to mzTab-M, MGF, parquet, etc.

What is **not** included in MASSter is:
* downstream data analysis: Simple approaches like PCA, UMAP, or ANOVA are integrated for QC, but we didn't include methods for normalization or pathway analysis, etc.
* annotation by MS2 with spectral matching: by design, we outsource MS2 annotation to external tools, like LipidOracle or timaR. Masster has import functions to read the results of annotation and embed them into visualization and export files.
* support for ion mobility, or MS3 data.
* a GUI: we stick to Python scripts. Interactive data exploration is possible by notebooks (Jupyter or Marimo)

## Architecture

MASSter defines classes for Spectra, Chromatograms, Libraries, Samples, and Studies (a Study is a collection of samples, such as an LC–MS sequence). Users will typically work with a single `Study` object at a time. `Sample` objects are created when analyzing a batch (and saved for caching), or used for development, troubleshooting, or generating illustrations.

The analysis can be done in scripts (without user intervention, e.g., by the integrated Wizard), or interactively in notebooks like [marimo](https://marimo.io/) or [jupyter](https://jupyter.org/).

MASSter was engineered to maximize quality, sensitivity, scalability, and speed. While Python can be slower than other languages, considerable effort was spent on optimization, including the systematic use of Polars, NumPy vectorization, multiprocessing, and chunking. To maximize speed and maintain a small memory footprint, MASSter reads raw data directly, only accessing the fractions relevant for analysis. For example, the analysis of DIA/ZTScan data is effectively as fast as DDA data, even though the files are 10–50x larger. MASSter uses optimized HDF5 files to store results and facilitate portability across operating systems and Python versions. MASSter has been tested on studies with over 3,000 LC–MS/MS samples (≈1 million MS2 spectra) and autonomously completed analyses within a few hours.

A design principle of MASSter is to provide excellent results without requiring extensive optimization or technical knowledge. **In most cases, running the code or the Wizard with default settings is sufficient.**

## Prerequisites

You will need to install Python (3.11-3.13).
**Reading vendor formats relies on .NET libraries and is only possible on Windows**. On Linux or macOS, you will need to use mzML or mz5 data. The latter is recommended over mzML, in particular for large files (DIA).

### mzML vs mz5 (in MASSter)

- **mzML**: widely supported XML format. In MASSter it is read via OpenMS/pyOpenMS. It can be fastest for on-demand access when loaded in-memory (`ondisk=False`), but it can become very large and slow/heavy to parse for big runs.
- **mz5**: HDF5-based format produced by ProteoWizard (`msconvert --mz5`). In MASSter it is read with an on-demand HDF5 reader. It is often a better choice than mzML for large acquisitions (e.g., DIA) and for cross-platform access.
- **Reproducible benchmarks**: benchmark scripts live in the main project repository (`zamboni-lab/masster`). Results depend on file size and whether mzML is loaded on-disk vs in-memory.

MASSter uses its own HDF5 file format to save results (i.e., `.sample5` and `.study5`), which is portable across platforms and Python versions. Consequently, any results generated by MASSter are accessible on all operating systems.

**It is recommended to use data in either the vendor's raw formats (WIFF and Thermo RAW) or mzML/mz5 in profile mode (both accessible via Pwiz).** MASSter includes a sophisticated and sufficiently fast centroiding algorithm that works well across the full dynamic range and only acts on relevant spectra. In our tests with data from different vendors, this centroiding performed significantly better than most vendor implementations (which are primarily proteomics-centric).

**Direct access to Bruker *.d, Agilent *.d, and SCIEX *.wiff2 formats is not supported**, at least not in the public version.

## Getting started
**The quickest way to use or learn MASSter is to use the integrated Wizard**, which handles most tasks automatically.

The Wizard only needs to know the location of the MS files and where to store the results.
```python
from masster import Wizard
wiz = Wizard(
    source=r'..\..\folder_with_raw_data',    # where to find the data
    folder=r'..\..folder_to_store_results',  # where to save the results
    ncores=6                                 # optional; don't exceed 8 as the process is limited by disk I/O
    )
wiz.test_and_run()
```

This will trigger the analysis of the raw data and create a script to process all samples and assemble the study. The complete processing workflow will be saved as `1_processing.py` in the output folder. The Wizard will run a initial test and, if successful, execute the full workflow using parallel processes. Once processing is complete, navigate to the output folder to see the results.

If you want to interact with your data, we recommend using [marimo](https://marimo.io/) or [jupyter](https://jupyter.org/) to open the `*.study5` file. For example:

```bash
# Use marimo to open the script created by the Wizard
marimo edit '..\\..\\folder_to_store_results\\2_notebook.py'
# or, if you use uv to manage an environment with masster
uv run marimo edit '..\\..\\folder_to_store_results\\2_notebook.py'
```

### Basic Workflow for analyzing LC-MS study with 1-1000+ samples
In MASSter, the main object for data analysis is a `Study`, which consists of a bunch of `Samples`.
```python
from masster import Study
# Initialize the Study object with the default folder
study = Study(folder=r'D:\...\mylcms')

# Load data from folder with raw data, here: WIFF
study.add(r'D:\...\...\...\*.wiff')
# Load all compatible files from default folder
study.add()

# Perform retention time correction
study.align(rt_tol=2.0)
study.plot_alignment()
study.plot_rt_correction()
study.plot_bpc()

# Find consensus features
study.merge(min_samples=3)   # This will keep only features found in 3 or more samples
study.plot_consensus_2d()

# Retrieve information
study.info()

# Retrieve EICs for quantification
study.fill()

# Integrate EICs according to consensus metadata
study.integrate()

# Export results
study.export_excel()
study.export_csv()
study.export_mgf()
study.export_mztab()
study.export_parquet()

# Save the study to .study5
study.save()

# Some of the plots...
study.plot_samples_pca()
study.plot_samples_umap()
study.plot_heatmap()

# Load human metabolome (without RT) and annotate by MS1
study.lib_load('hsapiens')
study.identify()
study.get_id()
# Plot features with putative identification
study.plot_consensus_2d(show_only_features_with_id=True,
           colorby="has_ms2",
           tooltip="id")

# Import LipidOracle results (MS2 annotation)
study.import_oracle('lipidoracle-folder')

# Import timaR results (MS2 annotation)
study.import_tima('tima-folder')

# To learn more about available methods...
dir(study)
```
The information is stored in Polars DataFrames:
```python
# Information on samples
study.samples_df
# Information on consensus features
study.consensus_df
```

### Analysis of a single sample
For troubleshooting, exploration, or creating figures for a single file, you can open and process a single file:  
```python
from masster import Sample
sample = Sample(filename='...') # Full path to a *.raw, *.wiff, *.mzML, or *.sample5 file
# Peek into sample
sample.info()

# Process
sample.find_features(chrom_fwhm=0.5, noise=200) # For Orbitrap data, set noise to 1e5
sample.find_adducts()
sample.find_ms2()

# Access data
sample.features_df

# Save results
sample.save() # Stores to *.sample5, our custom HDF5 format
sample.export_mgf()
sample.export_csv()
sample.export_excel()
sample.export_mztab()

# Some plots
sample.plot_bpc()
sample.plot_tic()
sample.plot_2d()
sample.plot_feature_stats()

# Explore methods
dir(sample)
```

## Disclaimer

**MASSter is research software under active development.** While we use it extensively in our lab and strive for quality and reliability, please be aware:
- **No warranties**: The software is provided "as is" without any warranty of any kind, express or implied.
- **Backward compatibility**: We do not guarantee backward compatibility between versions. Breaking changes may occur as we improve the software.
- **Performance**: While optimized for our workflows, performance may vary depending on your data and system configuration.
- **Support**: This is an academic project with limited resources. At the moment, we do not provide external user support.
- **PyPI distribution**: PyPI publishes the **clear** pure-Python wheel only.
- **GitHub Releases**: Platform/Python-specific wheels are distributed via GitHub Releases in this repository (not via PyPI/TestPyPI).
- **Source for releases**: This distribution repository does not contain source code; see the main project repository (`zamboni-lab/masster`) for sources and issue tracking.
- **Production use**: If you plan to use MASSter in production or critical workflows, thorough testing with your data is recommended.

## License
GNU Affero General Public License v3

See the [LICENSE](LICENSE) file for details.

### Third-Party Licenses
This project uses several third-party libraries, including pyOpenMS which is licensed under the BSD 3-Clause License. For complete information about third-party dependencies and their licenses, see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Citation
If you use MASSter in your research, please cite this repository.
