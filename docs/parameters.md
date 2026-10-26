# Parameters reference

All user-settable parameters, one table per part. "Cell" = first line (title) of the notebook cell. After changing a parameter, re-run the notebook from the top (and the later parts).

## Part 1 – INQUIRE_TIP_Pt1_Preprocessing

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `GLOBAL_SEED` | Imports & setup | `1` | Never for reproducible results |
| `plan_endpoint(censor_frac, min_class_n, skew_log)` | Imports & setup | `0.35`, `15`, `1.0` | Rules of the modeling plan: share of samples at the lowest value for classification; minimum class size; skewness above which a log10 scale is used (same function in Part 2) |
| `DATA_DIR` | Check if necessary columns exist | `Path("../data")` | Working folder elsewhere |
| `TOX_DUST_FILE` | Check if necessary columns exist | `DATA_DIR / "tox_vs_dust_merged.csv"` | Other file name |
| `META_UMU_FILE` | Check if necessary columns exist | `DATA_DIR / "feature_metadata_GCxGC-MS.xlsx"` | Other file name |
| `META_VU_FILE` | Check if necessary columns exist | `DATA_DIR / "feature_metadata_LC-MS.xlsx"` | Other file name |
| `META_CLASS_FILE` | Check if necessary columns exist | `DATA_DIR / "feature_metadata_classified.xlsx"` | Other file name |
| `OUT_DIR` | Check if necessary columns exist | `DATA_DIR / "inquire_tip_outputs"` | Other output folder (then also in Parts 2–3) |
| `META_COLS` | Check if necessary columns exist | 10 metadata columns | Other metadata columns |
| endpoint column list in `check_cols_before_load(TOX_DUST_FILE, ...)` | Check if necessary columns exist | `sample_base` + 11 raw endpoint names | Other endpoints |
| `MAX_SHOW_PER_COL`, `MAX_SHOW_PER_COL_META` | Whitespace cleanup | `None` (show all) | Limit the number of changed values printed per column |
| `TOX_COL_RENAME` | Harmonize column names | raw → standard names of the 11 endpoints | Other endpoints or names |
| `SMPL_COL_MRGD` | Define tox endpoint columns | `"Sample"` | Never (name after renaming `sample_base`) |
| `LAST_TOX_COL` | Define tox endpoint columns | `"antiestro (ng/g)"` | Other last endpoint (warning only) |
| `FIRST_FEAT_COL` | Define tox endpoint columns | `"2427_0.69_UMU"` | **Your first feature column** |
| candidate thresholds `[0.02, 0.05, 0.10, 0.15, 0.20, 0.30]` | Detection-frequency threshold check | – | Other thresholds to compare |
| `NAN_TO_ZERO` | Dust feature preprocessing | `True` | Keep `True` |
| `NEG_HANDLING` | Dust feature preprocessing | `"abs"` | `"zero"`: below-LOD estimates = not detected |
| `MIN_DETECT_FRAC` | Dust feature preprocessing | `0.10` | Detection-frequency filter (typically 0.10–0.30) |
| `MIN_UNIQUE_VALUES` | Dust feature preprocessing | `5` | Minimum distinct values per feature |
| `LOG_TRANSFORM_X` | Dust feature preprocessing | `True` | Rarely |

## Part 2 – INQUIRE_TIP_Pt2_Modeling

### Data and run settings

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `GLOBAL_SEED` | Imports & setup | `1` | Never for reproducible results |
| `make_tuned_enet(..., inner_splits)` | Imports & setup | `3` | Inner CV folds for re-tuning alpha |
| `ENetTopKSelector(k, min_features)` | Imports & setup | `k` = XGBoost features, `10` | Rarely |
| `DATA_DIR`, `OUT_DIR` | Load preprocessed data | as in Part 1 | Must match Part 1 |
| `PT1_DATE` | Load preprocessed data | `None` | Use an older Part 1 run, e.g. `"261018"` |
| `ENDPOINT_SHORT` | Endpoint selection | `"AhR"` | **Every run** |
| `ENDPOINTS` | Endpoint selection | 11 short → output names | Other endpoints |
| `TASK_OVERRIDE` | Endpoint selection | `None` | Force `"regression"` or `"classification"` |
| `FORCE_RUN` | Endpoint selection | `True` | `False`: stop at endpoints marked "skip" |
| `FAST_MODE` | Endpoint selection | `False` | `True` for a quick test |
| `N_SPLITS` | Endpoint selection | `5` | CV folds |
| `N_REPEATS` | Endpoint selection | `10` (`3` in fast mode) | CV repeats |
| `N_PERM` | Endpoint selection | `100` (`20` in fast mode) | Permutations (smallest p = 1 / (N_PERM + 1)) |
| `N_STABILITY_RUNS` | Endpoint selection | `100` (`50` in fast mode) | Stability-selection runs |
| `SUBSAMPLE_FRACTION` | Endpoint selection | `0.75` | Share of samples per stability run |
| `STABILITY_THRESHOLD` | Endpoint selection | `0.6` | "Stable" features (XGBoost input, stable lists) |
| `HIGH_CONF_THRESHOLD` | Endpoint selection | `0.8` | "High-confidence" features |
| `XGB_MAX_FEATURES` | Endpoint selection | `200` | Maximum XGBoost input features |
| `TOP_N_FSACI` | Endpoint selection | `120` | Size of the saved top list |
| `PRIORITIZE_POSITIVE_ONLY` | Endpoint selection | `True` | `False`: FSACI also for compounds linked to lower endpoint values |

### Univariate screen

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `SCREEN_TOP_FRAC` | Univariate screen | `0.10` | Share of samples in the top group |
| `SCREEN_MIN_TOP_N` | Univariate screen | `8` | Minimum size of the top group |
| `SCREEN_MIN_DETECTED` | Univariate screen | `5` | Minimum number of detections of a screened feature |
| `FDR_LEVEL` | Univariate screen | `0.10` | BH q-value threshold for a hit |
| `SCREEN_SHOW_N` | Univariate screen | `20` | Rows shown per table |
| `SCREEN_MW_PERM` | Univariate screen | `200000` | Permutations for the Mann-Whitney p-values |
| `SCREEN_MW_REFINE_P` | Univariate screen | `0.05` | Permutation p-values only below this approximate p |

### Models and FSACI

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `L1_RATIOS` | Train Elastic Net | `[0.1, 0.3, 0.5, 0.7, 0.9]` | Rarely |
| `N_ALPHAS` | Train Elastic Net | `60` | Rarely |
| `alphas_eval` | Elastic Net performance | every 3rd alpha (20 values) | Rarely (alpha grid of the nested CV) |
| `xgb_params` | Prepare XGBoost input matrix & configuration | `n_estimators=300, max_depth=3, learning_rate=0.03, subsample=0.8, colsample_bytree=0.5, min_child_weight=3, reg_lambda=1.0` | Rarely; objective is set from the task |
| `W_ENET`, `W_SHAP` | FSACI | `0.5`, `0.5` | Weights of Elastic Net and SHAP in the effect score |

### Cross-endpoint summary

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `MIN_Q2` | Cross-endpoint statistics summary | `0.10` | Minimum CV Q² (regression) for inclusion |
| `MIN_AUC` | Cross-endpoint statistics summary | `0.60` | Minimum CV AUC (classification) for inclusion |
| `ENDPOINT_BLOCKS` | Cross-endpoint statistics summary | cytotoxicity, inflammation, AhR, androgen_receptor, estrogen_receptor, thyroid_receptor | Other endpoints or grouping |
| `endpoint_order` | Cross-endpoint statistics summary | 11 endpoints | Order in the tables |
| quality thresholds ("Overall quality") | Cross-endpoint statistics summary | Q² ≥ 0.30 / 0.15 / > 0; AUC ≥ 0.75 / 0.65 / > 0.55 | Descriptive rating only (fixed in the code) |
| `MAX_RANKED_PER_ENDPOINT` | Cross-endpoint compound summary | `100` | Compounds per endpoint in the global score |
| `TOP_N_LIST` | Cross-endpoint compound summary | `[10, 25, 50]` | Columns `n_blocks_top<N>` |
| `KEY_META_COLS` | Cross-endpoint compound summary | `feature_name, class, inchikey, formula` | Metadata columns placed after the ID |

## Part 3 – INQUIRE_TIP_Pt3_Visualization

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `DATA_DIR`, `OUT_DIR` | Setup | as in Parts 1–2 | Must match |
| `PT1_DATE` | Setup | `None` | Same Part 1 run as in Part 2 |
| `TOX_DUST_FILE`, `META_CLASS_FILE` | Setup | as in Part 1 | Must be the same files as in Part 1 |
| `SAMPLE_COL_RAW` | Setup | `"sample_base"` | Other sample ID column |
| `NEG_HANDLING`, `GLOBAL_SEED`, `N_SPLITS` | Setup | `"abs"`, `1`, `5` | Must match Parts 1–2 |
| `RECOMPUTE_CV` | Setup | `False` | `True` to recompute the cached CV predictions (Fig 2) |
| `IDENTIFIED_ONLY` | Setup | `False` | `True`: Figs 3–5 with identified compounds only (suffix `_identified`) |
| `UNKNOWN_NAME_TEXT` | Setup | `"Feature"` | Text that marks unknown features in `feature_name` (case-sensitive) |
| `NO_NAME_IS_UNKNOWN` | Setup | `True` | Features without a name count as unknown |
| `N_TOP_ENRICH` | Setup | `100` | Fig 3: top compounds per endpoint |
| `MIN_CLASS_SIZE` | Setup | `15` | Fig 3: smaller classes pooled as "Other classes" |
| `N_TOP_COMPOUNDS` | Setup | `6` | Fig 4: compounds per endpoint |
| `N_TOP_HEATMAP` | Setup | `30` | Fig 5: compounds from the global ranking |
| `FIG_WIDTH_FULL` | Setup | `7.09` (in; 180 mm) | Journal format |
| `FIG_FORMATS` | Setup | `["pdf", "svg", "png"]` | Other formats |
| `ENDPOINTS`, `ENDPOINT_LABELS`, `ENDPOINT_UNITS`, `ENDPOINT_BLOCKS` | Setup | 11 endpoints, 6 blocks | Other endpoints (blocks as in Part 2) |
| `SERIES`, `BLOCK_COLORS` and other colors | Setup | colorblind-checked palette | Other colors |
