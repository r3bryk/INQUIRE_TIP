# Part 1 – INQUIRE_TIP_Pt1_Preprocessing

## Purpose

Part 1 prepares the data for the models of Part 2. It

1. reads the merged tox vs. dust dataset and the feature metadata, checks the mandatory columns and cleans whitespace,
2. renames the endpoint columns to standard names and splits the file into an **endpoint matrix** (samples × endpoints) and a **dust matrix** (samples × features),
3. checks samples (duplicates, missing IDs) and endpoint values (missing, text, zeros, negatives),
4. preprocesses the dust features: missing → 0, **detection-frequency filter**, handling of **below-LOD** values, **log10** transformation and **z-scaling**,
5. suggests a **modeling plan** per endpoint (regression, classification or skip),
6. saves the inputs for Part 2 (all files with the date prefix `<yymmdd>_`) and an Excel file that describes every preprocessing step with the numbers of the run.

## Inputs

From `data/` (`DATA_DIR`); formats and mandatory columns in [input_data.md](input_data.md):

| Input | File (default) |
|---|---|
| Merged tox vs. dust dataset | `tox_vs_dust_merged.csv` (`TOX_DUST_FILE`) |
| Feature metadata, GCxGC-MS | `feature_metadata_GCxGC-MS.xlsx` (`META_UMU_FILE`) |
| Feature metadata, LC-MS | `feature_metadata_LC-MS.xlsx` (`META_VU_FILE`) |
| Classified feature metadata | `feature_metadata_classified.xlsx` (`META_CLASS_FILE`) |

## Outputs

All in `data/inquire_tip_outputs/` (`OUT_DIR`), with the date of the run as prefix (`<yymmdd>_`):

| File | Content | Used by |
|---|---|---|
| `<yymmdd>_dust_detfilt_log10.csv` | **Modeling matrix**: samples × kept features, after the detection-frequency filter and log10 (not scaled) | **Part 2** |
| `<yymmdd>_dust_raw_numeric.csv` | Raw dust data, all features, numeric (text → missing) | **Part 2** (univariate screen) |
| `<yymmdd>_tox_endpoints_unscaled.csv` | Endpoint matrix: samples × 11 endpoints, renamed, numeric, not transformed | **Parts 2 and 3** |
| `<yymmdd>_feature_metadata_class_clean.csv` | Classified metadata, IDs cleaned, duplicate rows removed | **Part 2** |
| `<yymmdd>_feature_metadata_dust_masked.csv` | Classified metadata of the kept (modeled) features only | **Part 2** |
| `<yymmdd>_preprocessing_settings.csv` | Settings and key numbers of the run; written last, so Part 2 only uses complete runs | **Part 2** |
| `<yymmdd>_dust_detfilt_log10_scale().csv` | Scaled matrix (z-scores) with the metadata rows on top (one column per feature) | reference |
| `<yymmdd>_dust_feature_filter.csv` | Detection frequency, number of distinct values and kept yes/no per feature | reference, methods |
| `<yymmdd>_endpoint_modeling_plan.csv` | Distribution summary and suggested task per endpoint | reference, methods |
| `<yymmdd>_UMU_&_VU_feature_metadata_clean.csv` | Merged platform metadata, cleaned (whitespace, InChIKey suffixes) | reference |
| `<yymmdd>_preprocessing_description.xlsx` | One row per preprocessing step (sheets `dust` and `endpoints`) with explanation and numbers of this run | methods section |

Log file in `notebooks/`: `inquire_tip_pt1_preprocessing_<yymmdd_HHMMSS>.log`. All columns are explained in [codebook.md](codebook.md).

## Notebook sections in run order

All three notebooks start the same way: a title cell, *Reset environment* (`%reset -f`, clears all variables of a previous run) and *Imports & setup* (packages, logging, helper functions for safe reading of files and checking columns, `GLOBAL_SEED = 1`).

**0. Title.** Purpose of the notebook (no computation).

**1. Reset environment.** Clears all variables of a previous run.

**2. Imports & setup.** Loads the packages, starts logging (messages appear in the notebook and in the log file), defines the helper functions (file and column checks, whitespace cleanup, `plan_endpoint()` for the modeling plan, `r_scale()` = z-scaling like R's `scale()`), and sets `GLOBAL_SEED = 1`.

**3. Check if necessary columns exist in raw input data files.** Sets `DATA_DIR`, the input file names and `OUT_DIR` (created if missing). Reads only the header row of the merged file and of both platform metadata files and checks the mandatory columns. Stops with a list of missing and available columns if a column is missing.

**4. Load raw input data.** Loads the merged file and both platform metadata files, merges the metadata (one table) and checks that every `ML_feature_id` occurs only once.

**5. Whitespace cleanup.** Removes leading/trailing spaces from column names (stops if this would create duplicate names) and from all text values; lists every changed value.

**6. Harmonize column names.** Renames the raw endpoint columns to the standard names (`TOX_COL_RENAME`; warns if a column is not found) and collapses repeated spaces in column names.

**7. Drop `_<number>` suffixes from InChIKeys.** Removes suffixes such as `_37.79` from InChIKeys in the merged platform metadata, shows the changed keys and saves the cleaned metadata (`<yymmdd>_UMU_&_VU_feature_metadata_clean.csv`).

**8. Define tox endpoint columns.** Defines the sample ID column (`SMPL_COL_MRGD = "Sample"`), the endpoint columns (all columns between the sample ID and `FIRST_FEAT_COL`) and the feature columns (from `FIRST_FEAT_COL` to the end). Warns if the last endpoint column is not `LAST_TOX_COL`.

**9. Tox sample QC.** Counts rows, unique sample IDs, duplicated IDs and rows without ID. Rows without ID are dropped later; of duplicated IDs only the first row is used.

**10. Raw tox content QC.** Per endpoint: number of missing values, `<LOD` text, negative values, zeros, empty cells, "not tested" entries and other non-numeric entries (diagnostics only).

**11. Tox QC summary.** Converts the endpoints to numbers (text → missing, with a warning) and reports per endpoint the number of values, missing values (%), negatives, median and mean.

**12. Create tox matrix.** Endpoint matrix (sample ID + endpoints) with a distribution summary (quantiles) and the skewness per endpoint.

**13. Create sample list.** Sorted list of unique sample IDs; the same samples and order are used for the dust and endpoint data.

**14. Create dust predictor matrix.** Dust matrix (samples × features), rows without ID and duplicated IDs removed, all columns converted to numbers (text → missing, with a warning).

**15. Dust matrix diagnostics.** Per feature: missing values, distinct values, detections (> 0), negative values, quantiles and skewness; number of constant features (SD = 0); number of features per detection-frequency class. Diagnostics only.

**16. Detection-frequency threshold check.** Shows how many features (per platform) each candidate `MIN_DETECT_FRAC` (2–30%) would keep, and the distribution of detection frequencies per platform. Use it to choose `MIN_DETECT_FRAC` in the next cell. Features without a `_UMU` or `_VU` tag are listed.

**17. Dust feature preprocessing.** The central preprocessing step:

1. missing values → 0 (= not detected; `NAN_TO_ZERO`);
2. **detected = value > 0**; negative values (below LOD) count as **not detected**;
3. **detection-frequency filter**: keep features detected in ≥ `MIN_DETECT_FRAC` of the samples (default 10%) **and** with ≥ `MIN_UNIQUE_VALUES` distinct values (default 5). Rarely detected features cannot be linked reliably to toxicity with this number of samples and mainly add noise; they are still tested in the univariate screen of Part 2;
4. **below-LOD values**: `NEG_HANDLING = "abs"` (default) keeps them as (less precise) concentration estimates (minus sign removed); `"zero"` sets them to 0. A QC line reports how often a below-LOD estimate is higher than the smallest real detection of the same feature (expected when LODs differ between samples);
5. **log10(x + c)** per feature, c = half of the smallest **detected** value of that feature (`LOG_TRANSFORM_X`). Non-detects (0) all get the lowest value log10(c), below every detection; below-LOD estimates stay close to that floor.

Saves `<yymmdd>_dust_feature_filter.csv` and reports the median skewness after the transformation.

**18. Scale dust predictor matrix.** Z-scores like R's `scale()`: (x − mean) / SD with the sample SD (n − 1); features with SD = 0 are dropped. (Part 2 scales again on the samples of each endpoint.)

**19. Create response matrix.** Endpoint matrix in the same sample order as the dust matrix; stops if samples or order differ.

**20. Final pre-ML tox endpoints QC.** Per endpoint: number of values and missing values, distinct values, most common value and its share, whether it is the lowest value, skewness and the **suggested modeling task** (`plan_endpoint()`):

- **classification** (active vs. inactive) if ≥ 35% of the samples share the **lowest** value (e.g., the < LOQ substitute or 0 = no effect) and both classes have ≥ 15 samples;
- **skip** if one of these classes has < 15 samples;
- otherwise **regression**, on log10 scale if the skewness is > 1 and all values are ≥ 0.

Saves `<yymmdd>_endpoint_modeling_plan.csv`. Part 2 recomputes the plan on the samples of each endpoint and can override it (`TASK_OVERRIDE`, `FORCE_RUN`).

**21. Final pre-ML QC.** Shapes of X and Y; warns if X contains missing values (models would fail). Missing values in Y are expected (dropped per endpoint in Part 2).

**22. Save preprocessed datasets.** Loads and checks the classified metadata (`META_CLASS_FILE`; IDs as clean text; exact duplicate rows removed; for conflicting duplicates the first row is kept with a warning). Lists dust features without metadata (with near-matches to help find typos). Saves all files for Part 2 (see Outputs); `<yymmdd>_preprocessing_settings.csv` is written last.

**23. Preprocessing description.** Writes `<yymmdd>_preprocessing_description.xlsx`: one sheet for the dust data and one for the endpoints, one row per step (Item, Explanation, Notebook cell), with the settings and numbers of this run filled in. Useful for the methods section.

## User-settable parameters

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `DATA_DIR` | Check if necessary columns exist | `Path("../data")` | Your working folder is elsewhere |
| `TOX_DUST_FILE`, `META_UMU_FILE`, `META_VU_FILE`, `META_CLASS_FILE` | Check if necessary columns exist | see [input_data.md](input_data.md) | Your files have other names |
| `TOX_COL_RENAME` | Harmonize column names | 11 endpoints | Other endpoints or names |
| `FIRST_FEAT_COL` | Define tox endpoint columns | `"2427_0.69_UMU"` | **Always check**: name of the first feature column of your merged file |
| `LAST_TOX_COL` | Define tox endpoint columns | `"antiestro (ng/g)"` | Other last endpoint (warning only) |
| `NAN_TO_ZERO` | Dust feature preprocessing | `True` | Keep `True` (models cannot handle missing values) |
| `NEG_HANDLING` | Dust feature preprocessing | `"abs"` | `"zero"` to treat below-LOD estimates as non-detects |
| `MIN_DETECT_FRAC` | Dust feature preprocessing | `0.10` | Use the threshold check (cell 16); typical 0.10–0.30 |
| `MIN_UNIQUE_VALUES` | Dust feature preprocessing | `5` | Rarely |
| `LOG_TRANSFORM_X` | Dust feature preprocessing | `True` | Rarely (concentrations span orders of magnitude) |

All parameters of all parts: [parameters.md](parameters.md).

## Checks to look at

| Cell | What to look for | Interpretation |
|---|---|---|
| Check columns | "column check passed" for every file | Otherwise fix the column names in the input file |
| Load raw input data | Rows/columns as expected; "Duplicate ML_feature_id values: 0" | Duplicates → fix the platform metadata |
| Whitespace cleanup | Changed values listed | Information only |
| Define tox endpoint columns | The 11 endpoints listed; number of feature columns | Wrong endpoints → check `FIRST_FEAT_COL` |
| Tox sample QC | Duplicate IDs and rows without ID = 0 | Otherwise check the merged file |
| Raw tox content QC / Tox QC summary | Non-numeric entries set to missing; missing values per endpoint | Many missing values → fewer samples for that endpoint |
| Dust matrix diagnostics | Constant features; detection-frequency classes | Many never-detected features are normal for NTS data |
| Detection-frequency threshold check | Features kept per platform for each threshold | Choose `MIN_DETECT_FRAC` so that enough features of both platforms are kept |
| Dust feature preprocessing | Features kept per platform; "Median feature skewness after preprocessing" | Median skewness near 0–1 is fine |
| Create response matrix | "Sample alignment verified" | Otherwise the run stops |
| Final pre-ML tox endpoints QC | `suggested_task` and `reason` per endpoint | "skip" endpoints give unreliable results in Part 2 |
| Save preprocessed datasets | "Dropped from dust matrix (no metadata): 0"; duplicate metadata rows | Features without metadata are modeled but have no names in the outputs |

## Common errors

| Message | Cause | Solution |
|---|---|---|
| `FileNotFoundError: ... file not found` | `DATA_DIR` or a file name is wrong | Set `DATA_DIR` and the file names |
| `ValueError: [<dataset>] missing columns: ...` | Column names in the input differ from the expected ones | Rename the columns in the input file (case-sensitive) or adapt the check |
| `ValueError: Column '2427_0.69_UMU' not found in merged tox vs. dust dataset` | `FIRST_FEAT_COL` does not match your data | Set `FIRST_FEAT_COL` to your first feature column |
| `ValueError: No tox endpoint columns found between Sample and the first feature column` | Column order differs | Put the endpoint columns between the sample ID and the features |
| `ValueError: Duplicate column names after harmonizing` | Two columns differ only by spaces | Rename one of them |
| `ValueError: No dust features left after the detection-frequency filter` | `MIN_DETECT_FRAC` too high | Lower `MIN_DETECT_FRAC` |
| `ValueError: X and Y have different row counts` / `Sample order mismatch` | Sample IDs inconsistent | Check `sample_base` |
| `PermissionError` when saving | Output file is open in Excel | Close the file and re-run the cell |
