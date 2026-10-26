# Codebook: output files and variables

All outputs are written to `data/inquire_tip_outputs/` (`OUT_DIR`). Part 1 files carry the date of the run as prefix (`<yymmdd>_`). Files with `_w_metadata` in the name are the same table with the columns of the classified feature metadata added (`rt1_(min)`, `rt2_(sec)`, `feature_id`, `feature_name`, `formula`, `inchikey`, `class`, `n_detected_samples`, `detection_frequency`, `concentration_type`; see [input_data.md](input_data.md)).

## 1. File map

| Folder | File | Written by | Read by | Section |
|---|---|---|---|---|
| `inquire_tip_outputs/` | `<yymmdd>_UMU_&_VU_feature_metadata_clean.csv` | Part 1 | – | 2.1 |
| | `<yymmdd>_dust_feature_filter.csv` | Part 1 | – | 2.2 |
| | `<yymmdd>_endpoint_modeling_plan.csv` | Part 1 | – | 2.3 |
| | `<yymmdd>_dust_detfilt_log10.csv` | Part 1 | Part 2 | 2.4 |
| | `<yymmdd>_dust_detfilt_log10_scale().csv` | Part 1 | – | 2.4 |
| | `<yymmdd>_dust_raw_numeric.csv` | Part 1 | Part 2 | 2.4 |
| | `<yymmdd>_tox_endpoints_unscaled.csv` | Part 1 | Parts 2, 3 | 2.5 |
| | `<yymmdd>_feature_metadata_class_clean.csv` | Part 1 | Part 2 | 2.6 |
| | `<yymmdd>_feature_metadata_dust_masked.csv` | Part 1 | Part 2 | 2.6 |
| | `<yymmdd>_preprocessing_settings.csv` | Part 1 | Part 2 | 2.7 |
| | `<yymmdd>_preprocessing_description.xlsx` | Part 1 | – | 2.8 |
| `endpoint_models/<endpoint>/` | `univariate_screen_<endpoint>.csv` (+ `_w_metadata`) | Part 2 | – | 3.1 |
| | `elastic_net_coefficients.csv`, `elastic_net_coefficients_selected.csv` (+ `_w_metadata`) | Part 2 | – | 3.2 |
| | `stability_rank.csv`, `stable_features.csv`, `high_confidence_features.csv` (+ `_w_metadata`) | Part 2 | – | 3.3 |
| | `xgboost_cv_results.csv` | Part 2 | – | 3.4 |
| | `shap_rank.csv` (+ `_w_metadata`) | Part 2 | – | 3.5 |
| | `fsaci_rank.csv`, `fsaci_rank_w_metadata.csv`, `fsaci_top120_w_metadata.csv`, `fsaci_top120_stable_w_metadata.csv` | Part 2 | Part 3 (`fsaci_rank.csv`) | 3.6 |
| | `FSACI_<endpoint>.csv` | Part 2 | Part 2 (cross-endpoint) | 3.7 |
| | `stats_<endpoint>.csv`, `stats_<endpoint>_T.csv` | Part 2 | Part 2 (cross-endpoint), Part 3 | 3.8 |
| `cross_endpoint_summary/` | `endpoint_model_summary.csv`, `_T.csv`, `_T.xlsx` | Part 2 | Part 3 | 4.1 |
| | `compound_global_summary.csv`, `_w_metadata.csv`, `_w_metadata.xlsx` | Part 2 | Part 3 | 4.2 |
| | `compound_endpoint_summary.csv`, `_w_metadata.csv`, `_w_metadata.xlsx` | Part 2 | – | 4.3 |
| `inquire_tip_figures/` | figures, `fig3_class_enrichment_table<s>.csv`, `figure_notes<s>.txt` | Part 3 | – | 5 |
| `inquire_tip_figures/cv_predictions/` | `cv_predictions_<endpoint>.csv` | Part 3 | Part 3 (cache) | 5 |
| `notebooks/` | `inquire_tip_pt<1-3>_<name>_<yymmdd_HHMMSS>.log` | Parts 1–3 | – | – |

## 2. Part 1 outputs

### 2.1 `<yymmdd>_UMU_&_VU_feature_metadata_clean.csv`
Both platform metadata files merged (one row per feature), whitespace removed, `_<number>` suffixes removed from `inchikey`. Same 10 columns as the input metadata. Reference only.

### 2.2 `<yymmdd>_dust_feature_filter.csv` – one row per feature (all features)
| Column | Meaning |
|---|---|
| `ML_feature_id` | Feature ID |
| `detection_frac` | Share of samples with a detection (value > 0; below-LOD values count as not detected) |
| `n_unique` | Number of distinct values (after removing the sign of below-LOD values) |
| `kept` | `True` = passed the detection-frequency filter (`MIN_DETECT_FRAC`, `MIN_UNIQUE_VALUES`) and is modeled |

### 2.3 `<yymmdd>_endpoint_modeling_plan.csv` – one row per endpoint
| Column | Meaning |
|---|---|
| `endpoint` | Endpoint (output name) |
| `n_total`, `n_valid`, `n_nan` | Samples in total, with a value, without a value |
| `n_unique` | Number of distinct values |
| `most_common_value`, `n_most_common_value`, `%_most_common_value` | Most common value, how many samples have it, and their share (%) |
| `most_common_is_min` | `True` if the most common value is the lowest value (typical for a < LOQ substitute or 0 = no effect) |
| `skewness` | Skewness of the values (NaN for ≤ 2 distinct values) |
| `suggested_task` | `regression`, `classification` or `skip` (rules in [01_preprocessing.md](01_preprocessing.md), cell 20) |
| `suggested_transform` | `none`, `log10` (regression) or `binary` (classification/skip) |
| `reason` | Why this task was suggested |

### 2.4 Dust matrices – one row per sample (`Sample`), one column per feature
| File | Content |
|---|---|
| `<yymmdd>_dust_detfilt_log10.csv` | **Modeling matrix**: kept features, missing → 0, below-LOD handled (`NEG_HANDLING`), log10(x + c) with c = half the smallest detected value per feature. Not scaled (Part 2 scales per endpoint). |
| `<yymmdd>_dust_detfilt_log10_scale().csv` | The same after z-scaling (mean 0, SD 1 over all samples), with 10 metadata rows on top (`rt1_(min)` … `concentration_type`, incl. `class`), then one row per sample. Reference only. |
| `<yymmdd>_dust_raw_numeric.csv` | All features, raw values as in the input (negative = below LOD; missing kept as empty), text entries set to missing. Used by the univariate screen. |

### 2.5 `<yymmdd>_tox_endpoints_unscaled.csv` – one row per sample
`Sample` + the 11 endpoints (output names, see [input_data.md](input_data.md)), numeric, text entries set to missing, **not** transformed (transformations are done per endpoint in Part 2).

### 2.6 Metadata
`<yymmdd>_feature_metadata_class_clean.csv`: classified metadata of all features (11 columns as the input), IDs as clean text, exact duplicate rows removed, first row kept for conflicting duplicates. `<yymmdd>_feature_metadata_dust_masked.csv`: the same, only for the kept (modeled) features.

### 2.7 `<yymmdd>_preprocessing_settings.csv` – columns `setting`, `value`
Rows: `TOX_DUST_FILE`, `META_CLASS_FILE`, `FIRST_FEAT_COL`, `NAN_TO_ZERO`, `NEG_HANDLING`, `MIN_DETECT_FRAC`, `MIN_UNIQUE_VALUES`, `LOG_TRANSFORM_X`, `GLOBAL_SEED`, `n_samples`, `n_endpoints`, `n_dust_features_raw`, `n_dust_features_kept`. Written last; Part 2 uses only runs with this file.

### 2.8 `<yymmdd>_preprocessing_description.xlsx`
Sheets `dust` and `endpoints`; columns `Item` (step), `Explanation` (what was done and why, with the settings and numbers of this run) and `Notebook cell` (first line of the cell). Ready-made text for a methods section.

## 3. Part 2 outputs per endpoint (`endpoint_models/<endpoint>/`)

### 3.1 `univariate_screen_<endpoint>.csv` – one row per screened feature, sorted by `p_mannwhitney`
| Column | Meaning |
|---|---|
| `ML_feature_id` | Feature ID |
| `lab` | Platform tag from the ID (`UMU` = GCxGC-MS, `VU` = LC-MS in the INQUIRE data; `other`) |
| `n_detected` | Samples with a detection (value > 0) |
| `det_frac_top`, `det_frac_rest` | Share of samples with a detection in the top group / in the other samples |
| `median_top`, `median_rest` | Median concentration (raw scale) in the top group / in the other samples |
| `spearman_rho`, `p_spearman`, `q_spearman` | Spearman correlation with the endpoint (all samples), two-sided p-value, BH q-value |
| `mw_auc_top` | Probability that a top sample has a higher concentration than another sample (0.5 = no difference) |
| `p_mannwhitney`, `q_mannwhitney` | One-sided Mann-Whitney p-value (permutation p for approximate p < 0.05), BH q-value |
| `p_mannwhitney_approx` | Mann-Whitney p-value from the normal approximation |
| `p_fisher`, `q_fisher` | One-sided Fisher's exact p-value (detected more often in the top group), BH q-value |
| `hit_spearman` | q_spearman < `FDR_LEVEL` and ρ > 0 |
| `hit_mannwhitney`, `hit_fisher` | q < `FDR_LEVEL` |
| `n_tests_hit` | Number of tests (0–3) with a hit |

### 3.2 Elastic Net coefficients – one row per feature, sorted by `enet_importance`
| Column | Meaning |
|---|---|
| `enet_coef` | Coefficient of the Elastic Net fitted on all samples (scaled features; modeled target). + = higher concentration goes with a higher endpoint value |
| `enet_importance` | \|`enet_coef`\| |

`elastic_net_coefficients_selected.csv` contains only features with `enet_coef` ≠ 0.

### 3.3 Stability selection – one row per feature
| Column | Meaning |
|---|---|
| `stability_score` | Share of the `N_STABILITY_RUNS` subsample fits in which the feature had a non-zero coefficient (0–1) |

`stable_features.csv`: score ≥ `STABILITY_THRESHOLD` (0.6); `high_confidence_features.csv`: score ≥ `HIGH_CONF_THRESHOLD` (0.8).

### 3.4 `xgboost_cv_results.csv`
`endpoint`, `metric` (Q² or AUC), `CV_mean`, `CV_sd` (mean and SD of the CV metric over the repeats; NaN if XGBoost was skipped).

### 3.5 `shap_rank.csv` – one row per XGBoost input feature
| Column | Meaning |
|---|---|
| `shap_importance` | Mean \|SHAP value\| over all samples (model scale) |
| `shap_direction` | Rank correlation between the feature value and its SHAP value (+1 = higher concentration always pushes the prediction up) |

### 3.6 FSACI ranking – one row per feature, sorted by `FSACI`
| Column | Meaning |
|---|---|
| `FSACI_rank` | Rank (1 = highest FSACI) |
| `enet_coef`, `enet_importance` | See 3.2 |
| `stability_score` | See 3.3 |
| `shap_importance`, `shap_direction` | See 3.5 (0 if not an XGBoost feature) |
| `in_xgboost` | `True` if the feature was an XGBoost input |
| `enet_score` | Percentile rank of `enet_importance` among features with a non-zero coefficient (0 if zero) |
| `shap_score` | Percentile rank of `shap_importance` among features with SHAP > 0 |
| `effect_score` | Weighted mean of `enet_score` and `shap_score` (`W_ENET`, `W_SHAP`); `enet_score` only if XGBoost has no signal |
| `direction` | +1 / −1: sign of `enet_coef`, or of `shap_direction` if the coefficient is 0 |
| `FSACI_any_direction` | `stability_score` × `effect_score` |
| `FSACI` | = `FSACI_any_direction` for `direction` = +1; 0 for −1 (with `PRIORITIZE_POSITIVE_ONLY = True`) |

`fsaci_rank_w_metadata.csv` has the key columns first. `fsaci_top120_w_metadata.csv`: the top `TOP_N_FSACI` features with FSACI > 0; `fsaci_top120_stable_w_metadata.csv`: of these, those with stability ≥ 0.6.

### 3.7 `FSACI_<endpoint>.csv`
`ML_feature_id`, `FSACI`, `FSACI_rank`, `direction` (see 3.6). Input for the cross-endpoint summary.

### 3.8 `stats_<endpoint>.csv` (one row) and `stats_<endpoint>_T.csv` (transposed)
54 columns: endpoint description (`Endpoint`, `Task`, `Target`, `Metric`, `Chance level`, `Endpoint count`, `N active (classification)`, mean, median, SD, min, max, `N unique values`, `% at most common value`, `Total features`), Elastic Net (`best alpha`, `best l1_ratio`, `training metric`, `CV metric mean`/`SD`, `permutation observed`, `permutation p-value`, `signal`, `selected compounds (≠ 0)`, `selected positive`, `selection rate (%)`), stability selection (`Stability runs`, number of features per stability range, `Features with stability > 0`, `Stable features (score >= 0.6)`, `Stability selection rate (%)`), XGBoost (`N features`, `CV metric mean`/`SD`, `permutation observed`, `permutation p-value`, `signal`), SHAP/FSACI (`SHAP used in FSACI`, `Model has signal`, `FSACI positive only`, `Features with FSACI > 0`, `FSACI max score`, `FSACI top feature`, `FSACI top score`, `FSACI top feature stability score`) and run settings (`CV folds x repeats`, `N permutations`, `Fast mode`). Every column is explained in the `explanation` sheet of `endpoint_model_summary_T.xlsx` (4.1).

## 4. Part 2 cross-endpoint outputs (`cross_endpoint_summary/`)

### 4.1 `endpoint_model_summary.csv` – one row per endpoint
All 54 columns of `stats_<endpoint>.csv` plus:

| Column | Meaning |
|---|---|
| `Best CV metric`, `Best model` | Higher CV metric of Elastic Net and XGBoost, and which model |
| `dMetric (XGBoost-Elastic Net)` | XGBoost − Elastic Net CV metric (> 0: non-linear effects) |
| `Elastic Net overfit (train - CV)` | Training − CV metric |
| `Elastic Net selected (%)`, `XGBoost input (%)` | Selected / XGBoost input features as % of all features |
| `Permutation significant` | `Yes` if any model has p < 0.05 |
| `Overall quality` | Good / Moderate / Weak / Poor (Q² ≥ 0.30 / ≥ 0.15 / > 0; AUC ≥ 0.75 / ≥ 0.65 / > 0.55) |
| `Best CV metric (models with signal)` | Best CV metric among models with signal (empty = none) |
| `Minimum CV metric required` | `MIN_Q2` or `MIN_AUC` |
| `Included in compound ranking`, `Exclusion reason` | Inclusion in the global ranking and why not |
| `Endpoint block` | Block of endpoints measuring the same biology |
| `Endpoint weight` | Model quality 0–1 (Q², or (AUC − 0.5)/0.5); 0 if not included |

`endpoint_model_summary_T.csv`/`.xlsx`: transposed (one column per endpoint); the `.xlsx` has an `explanation` sheet for every row.

### 4.2 `compound_global_summary(_w_metadata)` – one row per compound in the top set of at least one included endpoint, sorted by `global_priority_score`
| Column | Meaning |
|---|---|
| `global_rank` | Rank (1 = highest global priority score) |
| `global_priority_score` | Weighted mean of the block scores (0–1), weights = block weights |
| `n_blocks_hit` | Blocks in which the compound is among the top `MAX_RANKED_PER_ENDPOINT` compounds of at least one endpoint |
| `n_blocks_top10`, `n_blocks_top25`, `n_blocks_top50` | Blocks in which the compound is in the top 10 / 25 / 50 of at least one endpoint |
| `main_block` | Block contributing most to the global score |
| `block_score_<block>` | Highest weighted endpoint score in the block / block weight (0–1) |
| `block_best_rank_<block>` | Best FSACI rank in the block (within the top set) |
| `score_<endpoint>` | Endpoint score = 1 − (rank − 1) / n for the top n compounds; 0 otherwise |
| `rank_<endpoint>` | FSACI rank for the endpoint (empty = not prioritized) |

### 4.3 `compound_endpoint_summary(_w_metadata)` – one row per compound with FSACI > 0 for at least one endpoint
| Column | Meaning |
|---|---|
| `<endpoint>` (one column per endpoint) | FSACI rank among the prioritized compounds of that endpoint (empty = not prioritized) |
| `n_endpoints_prioritized` | Endpoints (all) with FSACI > 0 |
| `n_included_endpoints_prioritized` | Included endpoints with FSACI > 0 |
| `best_rank_included_endpoints` | Best rank among the included endpoints |

The `.xlsx` versions of 4.2 and 4.3 have three sheets: `data`, `endpoints` (per endpoint: task, metric, best CV metric, minimum, inclusion, reason, block, weight, `N prioritized compounds (FSACI > 0)`, `N compounds used in global score`, `Block weight`) and `explanation` (every column explained).

## 5. Part 3 outputs (`inquire_tip_figures/`)

Figures: see [03_visualization.md](03_visualization.md).

`fig3_class_enrichment_table<s>.csv` – one row per endpoint × class:

| Column | Meaning |
|---|---|
| `endpoint`, `cls` | Endpoint and chemical class (small classes pooled as "Other classes") |
| `k`, `n_top` | Compounds of this class among the top compounds; number of top compounds |
| `K`, `N` | Features of this class among all features of the endpoint; number of features |
| `fold_enrichment` | (k / n_top) / (K / N) |
| `p`, `q` | One-sided hypergeometric p-value; BH q-value over all cells |
| `log2_fe` | log2 fold enrichment, 0–3 (only over-representation; used for the colors) |

`figure_notes<s>.txt`: caption drafts with key numbers per figure. `cv_predictions/cv_predictions_<endpoint>.csv`: sample ID, `measured_model_scale` (target on the modeling scale) and `predicted` (cross-validated Elastic Net prediction).
