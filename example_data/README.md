# example_data/ – synthetic test data

Small **synthetic** input files in exactly the formats of the real input (same file structure, column names and column order), so the whole pipeline can be tested end to end. All values are random. The names and InChIKeys of the identified compounds are real compounds; sample IDs (`EX001`–`EX120`) and feature IDs are fictitious.

## Files

| File | Content |
|---|---|
| `tox_vs_dust_merged.csv` | 122 rows (120 samples + 2 test rows) × 612 columns: `sample_base`, the 11 endpoint columns (raw names), 600 dust features (280 GCxGC-MS, 320 LC-MS) |
| `feature_metadata_GCxGC-MS.xlsx` | Metadata of the 280 GCxGC-MS features (10 columns) |
| `feature_metadata_LC-MS.xlsx` | Metadata of the 320 LC-MS features (10 columns) |
| `feature_metadata_classified.xlsx` | Metadata of all 600 features + `class` (602 rows, incl. 2 duplicate test rows) |

## How to use

Copy the four files into `data/` and run the notebooks as described in the README (Part 2 once per endpoint; `FAST_MODE = True` is enough for a test). No settings need to be changed: the file names and `FIRST_FEAT_COL = "2427_0.69_UMU"` match the defaults of the notebooks.

## Built-in associations

| Endpoint | Planned behavior |
|---|---|
| AhR | Regression (log10); linked to a group of PAHs and related compounds |
| IL-6 / IL-8 at 1 mg/ml | Regression; linked to phthalates and other plasticizers |
| Cell viability | Classification (≈ 42% of samples at 0); linked to a group of unknown LC-MS features |
| TR agonism | Classification (≈ 53% of samples at the lowest value); linked to a group of unknown GCxGC-MS features |
| Estro | Modeling plan **skip** (only 9 samples above the lowest value); modeled anyway with `FORCE_RUN = True` |
| andro, antiandro, antiestro, IL-6 / IL-8 at 0.1 mg/ml | No or weak signal |

Expected result: AhR, IL-6 (1 mg/ml), IL-8 (1 mg/ml), cell viability and TR agonism pass the inclusion rules of the cross-endpoint summary; the other six endpoints are excluded. Exact numbers depend on the package versions.

## Built-in test cases

| Test case | Where | Handled in |
|---|---|---|
| Duplicated sample row (`EX017`) | merged file, last rows | Part 1, *Tox sample QC* (first row kept) |
| Row without sample ID | merged file, last row | Part 1, *Tox sample QC* (dropped) |
| Text entries in endpoint columns (`n.t.`, `<LOD`) | `andro`, `antiandro` | Part 1, *Raw tox content QC* / *Tox QC summary* (set to missing) |
| Missing endpoint values | all endpoints except cell viability | Part 2, *Endpoint selection* (samples dropped per endpoint) |
| Below-LOD values (negative numbers) | GCxGC-MS features | Part 1, *Dust feature preprocessing* (`NEG_HANDLING`) |
| Missing dust values (NaN) | 6 LC-MS features | Part 1, *Dust feature preprocessing* (`NAN_TO_ZERO`) |
| Text entry in a dust column (`n.d.`) | 1 LC-MS feature | Part 1, *Create dust predictor matrix* (set to missing, then 0) |
| Never-detected (constant) features | 15 LC-MS features | Part 1, *Dust matrix diagnostics* and detection-frequency filter |
| Rarely detected features | many LC-MS features | Part 1, detection-frequency filter |
| Feature IDs with an extra `_01` / `_02` suffix | 2 GCxGC-MS features | lab-tag detection (`_UMU`/`_VU` followed by a suffix) |
| InChIKeys with a `_<retention time>` suffix | Phenanthrene, Anthracene (GCxGC-MS metadata) | Part 1, *Drop `_<number>` suffixes from InChIKeys* |
| Feature names with leading/trailing spaces | ` Avobenzone`, `Hydrocortisone ` (LC-MS metadata) | Part 1, *Whitespace cleanup* |
| Exact duplicate metadata row | classified metadata | Part 1, *Save preprocessed datasets* (removed) |
| Conflicting duplicate metadata row (same ID, other class) | classified metadata | Part 1, *Save preprocessed datasets* (first row kept, warning) |
| Unknown features (`Feature_<ID>`) and isomers without InChIKey | all metadata files | Part 3, `IDENTIFIED_ONLY` |

The data were generated with a script (seed 2026) that is not part of the pipeline.
