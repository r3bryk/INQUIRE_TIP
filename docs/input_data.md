# Input data: files, formats and mandatory columns

All input files go into the working folder `data/` (`DATA_DIR`). File names are set in the cell *Check if necessary columns exist in raw input data files* of Part 1 (and in the setup cell of Part 3). If your files have other names, change them there.

Part 1 first checks the **header row** of every input file for the mandatory columns. It stops with a clear error message if a column is missing. Column names are case-sensitive; leading/trailing spaces in column names and text values are removed automatically (Part 1, *Whitespace cleanup*).

| File (default name) | Variable | Read by | Required? |
|---|---|---|---|
| `tox_vs_dust_merged.csv` | `TOX_DUST_FILE` | Parts 1 and 3 | yes |
| `feature_metadata_GCxGC-MS.xlsx` | `META_UMU_FILE` | Part 1 | yes |
| `feature_metadata_LC-MS.xlsx` | `META_VU_FILE` | Part 1 | yes |
| `feature_metadata_classified.xlsx` | `META_CLASS_FILE` | Parts 1 and 3 | yes |

## Terms used

- **Feature** – one chemical signal of the non-target screening (a compound, identified or not), with a unique ID (`ML_feature_id`). Each feature is one column of the merged file.
- **Endpoint** – one toxicity test result per sample (e.g., AhR activity). Each endpoint is one column of the merged file.
- **Identified / unknown feature** – a feature with a compound name (and usually an InChIKey) vs. a feature named `Feature_<ID>`.
- **Below-LOD value** – a concentration estimate below the limit of detection, marked with a minus sign (see below).

## 1. Merged tox vs. dust dataset (`tox_vs_dust_merged.csv`)

One CSV file (comma-separated) with **one row per sample** and three blocks of columns, in this order:

| Block | Columns | Content |
|---|---|---|
| Sample ID | `sample_base` | Unique sample ID (text). Rows without ID are dropped; for duplicated IDs only the first row is used. |
| Endpoints | the 11 endpoint columns (see 1.1), between `sample_base` and the first feature column | Endpoint values per sample (numbers; empty = not measured) |
| Features | from `FIRST_FEAT_COL` (default `2427_0.69_UMU`) to the **last column** | Concentration per sample and feature |

The endpoint block is defined by position: every column between the sample ID column and `FIRST_FEAT_COL` is an endpoint, and every column from `FIRST_FEAT_COL` to the end is a feature. Set `FIRST_FEAT_COL` (Part 1, *Define tox endpoint columns*) to the name of the first feature column of your file.

### 1.1 Endpoint columns

Part 1 checks for these 11 columns (raw names) and renames them to the names used in all outputs (`TOX_COL_RENAME`). Part 2 and Part 3 refer to them by a short name (`ENDPOINTS`).

| Raw column name (input) | Name in outputs | Short name (Parts 2–3) | Endpoint |
|---|---|---|---|
| `%.reduced.cell.viability.per.mg_BEAS2B` | `reduced cell viability/mg BEAS-2B (effect cut-off = 20%)` | `cell_viability` | Cytotoxicity: reduced cell viability per mg dust in BEAS-2B human bronchial epithelial cells (effect cut-off 20%) |
| `%.expr.of.IL6_0.1mg/ml_BEAS2B` | `relative IL-6 expression (BEAS-2B 0.1 mg/ml)` | `IL6_0.1` | Inflammation: relative IL-6 expression in BEAS-2B cells, 0.1 mg/ml dust |
| `%.expr.of.IL6_1mg/ml_BEAS2B` | `relative IL-6 expression (BEAS-2B 1 mg/ml)` | `IL6_1.0` | Inflammation: relative IL-6 expression, 1 mg/ml dust |
| `%.expr.of.IL8_0.1mg/ml_BEAS2B` | `relative IL-8 expression (BEAS-2B 0.1 mg/ml)` | `IL8_0.1` | Inflammation: relative IL-8 expression, 0.1 mg/ml dust |
| `%.expr.of.IL8_1mg/ml_BEAS2B` | `relative IL-8 expression (BEAS-2B 1 mg/ml)` | `IL8_1.0` | Inflammation: relative IL-8 expression, 1 mg/ml dust |
| `androgen.activity.(ng/g)` | `andro (ng/g)` | `andro` | Androgen receptor agonist activity (ng/g dust) |
| `anti-androgen.activity.(µg/g)` | `antiandro (µg/g)` | `antiandro` | Androgen receptor antagonist activity (µg/g dust) |
| `AhR.activity.(ng/g)` | `AhR (ng/g)` | `AhR` | Aryl hydrocarbon receptor activity (ng/g dust) |
| `TR.agonism.(ng/g)` | `TR agonism (ng/g)` | `TR` | Thyroid receptor agonist activity (ng/g dust) |
| `estrogen.activity.(pg/g)` | `estro (pg/g)` | `estro` | Estrogen receptor agonist activity (pg/g dust) |
| `anti-estrogen.activity.(ng/g)` | `antiestro (ng/g)` | `antiestro` | Estrogen receptor antagonist activity (ng/g dust) |

**Values:** numbers; empty cells = not measured. Text entries (e.g., `n.t.`, `<LOD`) are counted in *Raw tox content QC* and then set to missing. Missing values are **not** imputed: in Part 2 each endpoint uses all samples that have a value for it.

**Preprocessing before the pipeline (INQUIRE data, as reported by the data provider):** values below the limit of quantification (< LOQ) were replaced by LOQ/√2; outliers among these < LOQ values (IQR rule) were set to missing; negative values (for cell viability: cell proliferation) were set to missing. Original values are not available to the pipeline. Because many samples can share the same substitute value, Part 1 checks each endpoint for such a block of identical lowest values and then suggests a classification task instead of a regression (see [01_preprocessing.md](01_preprocessing.md) and [method_and_limitations.md](method_and_limitations.md)).

### 1.2 Feature columns

**Column names = feature IDs** (`ML_feature_id`), unique, identical to the IDs in the metadata files. In the INQUIRE data the IDs have these formats:

| Platform | ID format | Example |
|---|---|---|
| GCxGC-MS (non-polar constituents) | `<number>_<2nd-dimension retention time (s)>_UMU`, optionally followed by `_<nn>` | `2427_0.69_UMU`, `1321_1.63_UMU_01` |
| LC-MS (polar constituents) | `<retention time (min)>_<m/z>_VU_<n>` | `9.98_400.22769_VU_2` |

The platform tags `_UMU` and `_VU` (followed by `_<suffix>` or at the end of the ID) are only used to count features per platform in the QC tables of Parts 1 and 2. IDs without one of these tags work in the same way and are reported as platform "other". If your IDs use other tags, the per-platform counts show "other"; the models are not affected.

**Values** (concentrations; estimated concentrations or normalized peak areas, see `concentration_type` in the metadata):

| Value | Meaning | Handling (Part 1) |
|---|---|---|
| > 0 | detected | log10-transformed |
| 0 or empty (NaN) | not detected | NaN → 0 (`NAN_TO_ZERO = True`) |
| < 0 | concentration estimate **below the limit of detection** (LOD), marked with a minus sign (INQUIRE: GCxGC-MS features only) | counts as **not detected** for the detection-frequency filter; value kept as \|value\| (`NEG_HANDLING = "abs"`) or set to 0 (`"zero"`) |
| text | invalid entry | set to missing (then 0) with a warning |

Values of different features are not comparable (different response factors and units); the pipeline log-transforms and scales every feature separately.

## 2. Feature metadata per platform (`feature_metadata_GCxGC-MS.xlsx`, `feature_metadata_LC-MS.xlsx`)

One Excel file per platform (first sheet), **one row per feature**. Part 1 merges both, removes whitespace and `_<number>` suffixes from InChIKeys and saves the result (`<yymmdd>_UMU_&_VU_feature_metadata_clean.csv`); this merged table is a cleaned reference and is not used by the later steps.

| Column | Mandatory? | Content |
|---|---|---|
| `ML_feature_id` | yes | Feature ID, identical to the column name in the merged file |
| `rt1_(min)` | yes (may be empty) | Retention time (GCxGC-MS: 1st dimension; LC-MS: retention time) |
| `rt2_(sec)` | yes (may be empty) | 2nd-dimension retention time (GCxGC-MS only) |
| `feature_id` | yes | Original feature ID of the platform |
| `feature_name` | yes | Compound name, or `Feature_<ID>` for unknown features |
| `formula` | yes (may be empty) | Molecular formula |
| `inchikey` | yes (may be empty) | InChIKey; a suffix `_<number>` (e.g., `CRDAMVZIKSXKFV-YFVJMOTDSA-N_37.79`, used to separate isomers with the same key) is removed |
| `n_detected_samples` | yes (may be empty) | Number of samples with a detection, as reported by the platform |
| `detection_frequency` | yes (may be empty) | Detection frequency (%), as reported by the platform |
| `concentration_type` | yes (may be empty) | Type of value, e.g., `estimated concentration`, `normalized peak area` |

## 3. Classified feature metadata (`feature_metadata_classified.xlsx`)

The same columns as above **plus `class`** (chemical class, e.g., `PAHs & Fused Aromatics`, `Phthalates`), for **all** features (both platforms), one row per feature. Column order: `ML_feature_id`, `rt1_(min)`, `rt2_(sec)`, `feature_id`, `feature_name`, `formula`, `inchikey`, `class`, `n_detected_samples`, `detection_frequency`, `concentration_type`.

This file provides the compound names and classes in all outputs of Parts 2 and 3 (`*_w_metadata` files, figures) and the classes for the class enrichment (Part 3, Fig 3). Exact duplicate rows are removed; for IDs with conflicting duplicate rows the first row is kept (Part 1 warns). `class` may be empty (unknown features); Part 3 then uses "Unclassified".

**Unknown features:** Part 3 treats a feature as unknown if its `feature_name` contains `Feature` (`UNKNOWN_NAME_TEXT`) or is empty (`NO_NAME_IS_UNKNOWN = True`); see [03_visualization.md](03_visualization.md).

## Using other endpoints or another data set

The pipeline works with any number and kind of endpoints. To use other endpoints, adapt:

1. Part 1, *Check if necessary columns exist*: the list of endpoint columns checked in the merged file.
2. Part 1, *Harmonize column names*: `TOX_COL_RENAME` (raw name → output name; endpoints not listed keep their raw name).
3. Part 1, *Define tox endpoint columns*: `LAST_TOX_COL` (only used for a warning) and `FIRST_FEAT_COL`.
4. Part 2, *Endpoint selection*: `ENDPOINTS` (short name → output name).
5. Part 2, *Cross-endpoint statistics summary*: `ENDPOINT_BLOCKS` and `endpoint_order`.
6. Part 3, setup cell: `ENDPOINTS`, `ENDPOINT_LABELS`, `ENDPOINT_UNITS` and `ENDPOINT_BLOCKS` (same blocks as in Part 2).

Higher endpoint values must mean **more** activity (the default FSACI keeps only compounds linked to higher values, `PRIORITIZE_POSITIVE_ONLY = True`). For an endpoint where lower values mean more toxicity, invert it before Part 1 (e.g., use "reduced viability" instead of "viability").
