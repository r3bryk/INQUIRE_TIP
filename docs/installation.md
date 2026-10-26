# Installation and setup

## Prerequisites: software and tools

| Software | Version used | Required? | Where to get it | Notes |
|---|---|---|---|---|
| **Anaconda Distribution** (includes Python, the conda package manager, Anaconda Navigator and JupyterLab) | any recent | yes (or Miniconda) | https://www.anaconda.com/download | Free for individual/academic use; check your institution's license terms. Miniconda (https://docs.anaconda.com/miniconda/) is a smaller alternative without the Navigator. |
| **Python** | 3.11.14 | yes | installed into the environment (see below) | The notebooks were developed and tested with Python 3.11.14 (Anaconda build, 64-bit, Windows). |
| **JupyterLab** | 4.4.7 | yes | installed into the environment | Runs **locally** in your web browser. **No account or login is needed.** |
| **Web browser** | any modern browser | yes | Chrome, Firefox, Edge, Safari | Used by JupyterLab. |
| **Internet connection** | – | for installation only | – | The pipeline itself runs offline. |
| Git or GitHub Desktop | any | optional | https://git-scm.com, https://desktop.github.com | Only to clone the repository; you can also download it as a ZIP file from GitHub. |
| Spreadsheet program (Excel, LibreOffice Calc) | any | optional | – | To prepare input files and to inspect output CSV/XLSX files. |

**Tested on:** Microsoft Surface Laptop 5, Intel Core i7-1265U (12th generation), 16 GB RAM, Windows 11 (64-bit), Python 3.11.14.

**Recommended:** 16 GB RAM. Part 2 uses all processor cores for cross-validation, permutation tests and stability selection (`n_jobs=-1`). No graphics card (GPU) is needed.

**Operating system:** developed and tested on Windows 11. The code uses only platform-independent paths (`pathlib`), so it should also run on macOS and Linux (not tested).

## Setting up the Python environment

An *environment* is an isolated Python installation with exactly the package versions needed for this pipeline, so it does not interfere with other Python projects on your computer.

### Option A – command line (recommended)

1. Open the **Anaconda Prompt** (Windows: Start menu → "Anaconda Prompt"; macOS/Linux: a normal terminal).
2. Go to the repository folder, e.g.
   ```
   cd C:\path\to\INQUIRE_TIP
   ```

3. Create the environment from the file `environment.yml` (takes a few minutes):
   ```
   conda env create -f environment.yml
   ```

4. Activate it:
   ```
   conda activate inquire_tip
   ```

5. Start JupyterLab from the repository folder:
   ```
   jupyter lab
   ```

JupyterLab opens in your browser. Navigate to `notebooks/`.

Alternative to step 3 (pip only):
```
conda create -n inquire_tip python=3.11
conda activate inquire_tip
pip install -r requirements.txt
```

### Option B – Anaconda Navigator (graphical)

1. Open **Anaconda Navigator** → **Environments** → **Import** → select `environment.yml` → name it `inquire_tip` → **Import**.
2. Go to **Home**, choose `inquire_tip` in the "Applications on" drop-down, and click **Launch** under JupyterLab.

### Check the installation

In a new notebook cell run:
```
import sys, pandas, numpy, sklearn, xgboost, shap
print(sys.version, pandas.__version__, sklearn.__version__, xgboost.__version__, shap.__version__)
```
It should print Python 3.11.x (tested: 3.11.14), pandas 2.3.3, scikit-learn 1.7.2, xgboost 3.2.0 and shap 0.51.0.

## Python packages

Only the packages below are imported by the three notebooks. All other packages in a typical Anaconda installation are not needed.

| Package | Version | Used in part | Purpose |
|---|---|---|---|
| pandas | 2.3.3 | 1, 2, 3 | Tables (DataFrames): reading, merging, reshaping, writing CSV/XLSX |
| numpy | 2.3.3 | 1, 2, 3 | Numerical arrays, logarithms, random numbers |
| scipy | 1.16.3 | 1, 2, 3 | `skew`, `rankdata`, Spearman, Mann-Whitney U, hypergeometric (Fisher) tests |
| scikit-learn | 1.7.2 | 2, 3 | Elastic Net (`ElasticNet`, `ElasticNetCV`), cross-validation, R² / ROC AUC, pipelines |
| xgboost | 3.2.0 | 2 | XGBoost models (`XGBRegressor`); built-in SHAP values as fallback |
| shap | 0.51.0 | 2 | SHAP values (`TreeExplainer`) of the final XGBoost model |
| joblib | 1.5.2 | 2 | Parallel permutation tests and stability selection |
| tqdm | 4.67.1 | 2 | Progress bars |
| ipywidgets | 8.1.7 | 2 | Needed by `tqdm.auto` to draw progress bars in JupyterLab |
| openpyxl | 3.1.5 | 1, 2, 3 | Reading and writing Excel (.xlsx) files |
| matplotlib | 3.10.7 | 3 | Figures (PDF, SVG, PNG) |
| jupyterlab | 4.4.7 | all | Notebook interface |
| ipykernel | 6.30.1 | all | Runs Python inside JupyterLab |

Python standard-library modules (no installation needed): `pathlib`, `datetime`, `logging`, `random`, `warnings`, `collections`, `textwrap`.

## Folder structure

Keep the repository structure; then the default working-folder path in the notebooks (`DATA_DIR = Path("../data")`, i.e., the `data` folder next to `notebooks`) works without changes. Output subfolders are created automatically.

```
INQUIRE_TIP/
├── notebooks/                              ← open and run the notebooks from here
├── example_data/                           ← synthetic test data (copy into data/)
└── data/                                   ← DATA_DIR (working folder)
    ├── tox_vs_dust_merged.csv              ← input, Parts 1 and 3
    ├── feature_metadata_GCxGC-MS.xlsx      ← input, Part 1
    ├── feature_metadata_LC-MS.xlsx         ← input, Part 1
    ├── feature_metadata_classified.xlsx    ← input, Parts 1 and 3
    └── inquire_tip_outputs/                ← OUT_DIR
        ├── <yymmdd>_*.csv / .xlsx          ← Part 1 outputs (read by Parts 2 and 3)
        ├── endpoint_models/<endpoint>/     ← Part 2 outputs per endpoint
        ├── cross_endpoint_summary/         ← Part 2 cross-endpoint outputs (read by Part 3)
        └── inquire_tip_figures/            ← Part 3 figures
            └── cv_predictions/             ← Part 3 cache of cross-validated predictions
```

If you prefer another location, change `DATA_DIR` in Parts 1–3 to the full path of your working folder, e.g., `Path(r"D:/projects/INQUIRE_TIP/data")`. Use forward slashes `/` or a raw string `r"..."` on Windows. The input file names are set in the same cells (`TOX_DUST_FILE`, `META_UMU_FILE`, `META_VU_FILE`, `META_CLASS_FILE`; see [parameters.md](parameters.md)).

**Log files:** every notebook writes a log file (copy of all printed messages) into the folder from which it runs (`notebooks/`): `inquire_tip_pt1_preprocessing_<yymmdd_HHMMSS>.log`, `inquire_tip_pt2_modeling_<yymmdd_HHMMSS>.log` and `inquire_tip_pt3_visualization_<yymmdd_HHMMSS>.log`. Date and time are part of the name, so every run (e.g., every endpoint in Part 2) keeps its own log. Keep them as a record of each run, but do not upload them to a public repository (they contain sample IDs and results).

## Reproducibility

- **Fixed random seeds.** `GLOBAL_SEED = 1` is set in the setup cell of Parts 1 and 2 (Python `random` and numpy) and in Part 3. All random steps derive their seeds from it: CV fold splits (`GLOBAL_SEED + repeat`), permutations (`GLOBAL_SEED + 1000 + i`), stability-selection subsamples (`GLOBAL_SEED + i`), Mann-Whitney permutations, XGBoost (`random_state`) and the jitter of the Part 3 plots. Runs on the same input data and package versions give the same results.
- **One endpoint per run in Part 2.** Restart the kernel before every endpoint (*Kernel → Restart Kernel and Run All Cells* down to `# ML end`), so nothing from the previous endpoint is reused. The first code cell (`%reset -f`) also clears all variables.
- **Same Part 1 run for Parts 2 and 3.** Part 2 uses the most recent Part 1 run unless `PT1_DATE` is set; set the same `PT1_DATE` in Part 3 if you use an older run.
- **Order of execution matters.** Always run each notebook top to bottom. Several cells use results of earlier cells.
- **Package versions.** Use the versions in `requirements.txt`. Other versions of scikit-learn, xgboost or shap can change the numbers slightly (e.g., the choice of alpha or the order of ties).
- **Record what you ran.** Keep the log files and note any parameter changes (see [parameters.md](parameters.md)). Part 1 also writes its settings to `<yymmdd>_preprocessing_settings.csv`, and Part 2 writes the run settings into `stats_<endpoint>.csv`.
