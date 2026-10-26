# Part 2 – INQUIRE_TIP_Pt2_Modeling

## Purpose

Part 2 links the dust features to the toxicity endpoints. For **one endpoint per run** it

1. selects the samples with a value for the endpoint, chooses the task (regression or classification) and transforms the endpoint,
2. runs a **univariate screen** (each feature tested on its own),
3. fits an **Elastic Net** model and evaluates it with repeated, nested cross-validation (CV) and a **permutation test**,
4. runs **stability selection** (Elastic Net on many random subsamples),
5. fits **XGBoost** on the most stable features, evaluated with CV (feature selection redone inside every training fold) and a permutation test,
6. computes **SHAP** values of the final XGBoost model,
7. combines stability, Elastic Net and SHAP into the **FSACI** (Final Stability-Adjusted Compound Index) ranking of the compounds,
8. saves all results and one statistics table for the endpoint.

When all endpoints are done, two cells at the end of the notebook build the **cross-endpoint summary**: which endpoints have a reliable model, and a global ranking of the compounds over all reliable endpoints.

The method is explained in [method_and_limitations.md](method_and_limitations.md).

## How to run Part 2

**One endpoint per run.**

1. In the *Endpoint selection* cell set `ENDPOINT_SHORT` (one of `cell_viability`, `IL6_0.1`, `IL6_1.0`, `IL8_0.1`, `IL8_1.0`, `andro`, `antiandro`, `estro`, `antiestro`, `AhR`, `TR`).
2. *Kernel → Restart Kernel*, then run all cells from the top **down to the `# ML end` cell** (e.g., select the `# ML end` cell and use *Run → Run All Above Selected Cell*).
3. Repeat for every endpoint. Each run writes its own folder `endpoint_models/<ENDPOINT_SHORT>/` and its own log file.

**Cross-endpoint summary (once, after all endpoints).**

4. Restart the kernel, run the first four cells (title, *Reset environment*, *Imports & setup*, *Load preprocessed data*), then the cell *Cross-endpoint statistics summary* and after it the cell *Cross-endpoint compound summary* (both below `# ML end`). They read the files saved for all endpoints; no endpoint model needs to be in memory.

**Quick test.** `FAST_MODE = True` uses fewer CV repeats (3 instead of 10), permutations (20 instead of 100) and stability runs (50 instead of 100). Use it to test the pipeline or new data; use `FAST_MODE = False` for results. With 20 permutations the smallest possible p-value is 1/21 = 0.048, so the permutation tests can hardly reach p < 0.05.

**Endpoints marked "skip".** If the modeling plan says "skip" (too few active samples), the run stops unless `FORCE_RUN = True` (default). With `FORCE_RUN = True` the endpoint is modeled as classification and a warning says that the results are unreliable. Such endpoints normally have no model with signal and are excluded from the cross-endpoint ranking.

## Inputs

| Input | Source |
|---|---|
| `<yymmdd>_preprocessing_settings.csv`, `<yymmdd>_tox_endpoints_unscaled.csv`, `<yymmdd>_dust_detfilt_log10.csv`, `<yymmdd>_dust_raw_numeric.csv`, `<yymmdd>_feature_metadata_class_clean.csv`, `<yymmdd>_feature_metadata_dust_masked.csv` | Part 1 (`OUT_DIR`); all six files of **one** Part 1 run: the most recent complete run, or the run given by `PT1_DATE` |
| `endpoint_models/<endpoint>/stats_<endpoint>.csv`, `FSACI_<endpoint>.csv` | Part 2 endpoint runs (read by the cross-endpoint cells) |

## Outputs

**Per endpoint** in `inquire_tip_outputs/endpoint_models/<ENDPOINT_SHORT>/`. Files marked "+ meta" are also saved as `<name>_w_metadata.csv` with the feature metadata (name, class, InChIKey, ...) added:

| File | Content |
|---|---|
| `univariate_screen_<endpoint>.csv` (+ meta) | Univariate tests per feature (Spearman, Mann-Whitney, Fisher) with BH q-values and hits |
| `elastic_net_coefficients.csv` (+ meta) | Elastic Net coefficient per feature (all features) |
| `elastic_net_coefficients_selected.csv` (+ meta) | Features with a non-zero coefficient |
| `stability_rank.csv` (+ meta) | Stability score per feature |
| `stable_features.csv` (+ meta) | Features with stability ≥ `STABILITY_THRESHOLD` (0.6) |
| `high_confidence_features.csv` (+ meta) | Features with stability ≥ `HIGH_CONF_THRESHOLD` (0.8) |
| `xgboost_cv_results.csv` | XGBoost CV metric (mean, SD) |
| `shap_rank.csv` (+ meta) | Mean \|SHAP\| and SHAP direction per XGBoost feature |
| `fsaci_rank.csv`, `fsaci_rank_w_metadata.csv` | **Full FSACI ranking** (all features), with all components; the metadata version has the key columns first |
| `fsaci_top120_w_metadata.csv` | Top `TOP_N_FSACI` (120) features with FSACI > 0 |
| `fsaci_top120_stable_w_metadata.csv` | Of these, the stable ones (stability ≥ 0.6) |
| `FSACI_<endpoint>.csv` | FSACI, FSACI rank and direction per feature (read by the cross-endpoint cells) |
| `stats_<endpoint>.csv`, `stats_<endpoint>_T.csv` | One row (or column) with all statistics of the endpoint models and the run settings |

**Cross-endpoint** in `inquire_tip_outputs/cross_endpoint_summary/`:

| File | Content |
|---|---|
| `endpoint_model_summary.csv`, `endpoint_model_summary_T.csv`, `endpoint_model_summary_T.xlsx` | All endpoint statistics side by side + model quality, inclusion in the compound ranking, endpoint block and weight. The `.xlsx` has the sheets `data` and `explanation` (every row explained) |
| `compound_global_summary.csv`, `compound_global_summary_w_metadata.csv` / `.xlsx` | **Global ranking** of the compounds over all included endpoints (global priority score, blocks, per-endpoint scores and ranks). The `.xlsx` has the sheets `data`, `endpoints` and `explanation` |
| `compound_endpoint_summary.csv`, `compound_endpoint_summary_w_metadata.csv` / `.xlsx` | FSACI rank of every prioritized compound for **every** endpoint (also the excluded ones) |

Log file in `notebooks/`: `inquire_tip_pt2_modeling_<yymmdd_HHMMSS>.log`. All columns are explained in [codebook.md](codebook.md).

## Notebook sections in run order

### A. Set-up and data

**0. Title.** Purpose of the notebook.

**1. Reset environment.** Clears all variables of a previous run (`%reset -f`).

**2. Imports & setup.** Packages, logging, `GLOBAL_SEED = 1` and the helper functions used by all later cells: `plan_endpoint()` (same as Part 1), `r_scale()`, `make_cv()` (shuffled K-fold; stratified for classification), `score_metric()` (Q² or ROC AUC), `repeated_cv_metric()`, `permutation_test_metric()`, `make_tuned_enet()` (Elastic Net that re-tunes alpha on its training data), `ENetTopKSelector` (feature selection inside CV folds), `add_meta()` and `save_with_meta()`.

**3. Load preprocessed data.** Finds the Part 1 run (`PT1_DATE`, or the most recent run with a settings file), checks that all six files of that run exist, logs the Part 1 settings, loads the data, aligns samples between X and Y and stops if the modeling matrix contains missing values.

### B. One endpoint

**4. Endpoint selection.** All run settings in one place (endpoint, task override, CV, permutations, stability selection, XGBoost, FSACI). For the selected endpoint:

- drops samples without a value for this endpoint;
- recomputes the modeling plan (`plan_endpoint()`) on these samples; `TASK_OVERRIDE` can force a task;
- builds the **model target**: classification → 1 = value above the lowest value (active), 0 = at the lowest value (if no block of values sits at the minimum, the median is the cut-off); regression → log10(y), or log10(y + c) with c = half the smallest positive value if zeros occur, or the raw values if the endpoint is not skewed;
- scales the dust matrix again on these samples (`r_scale()`) and drops features that are constant within them;
- creates the output folder `endpoint_models/<ENDPOINT_SHORT>/`.

**5. Univariate screen.** Tests each feature on its own, with the **raw** dust data and a lower detection threshold (features detected in ≥ `SCREEN_MIN_DETECTED` = 5 samples), so that rare compounds present mainly in the most active samples are not lost by the detection-frequency filter of Part 1. Three tests per feature, each with Benjamini-Hochberg (BH) correction:

- **Spearman** correlation with the endpoint (all samples; a hit requires a positive correlation);
- **Mann-Whitney U**, one-sided: higher concentration in the **top group** (the `SCREEN_TOP_FRAC` = 10% samples with the highest endpoint values, at least `SCREEN_MIN_TOP_N` = 8) than in the rest. P-values below 0.05 are recomputed with `SCREEN_MW_PERM` = 200,000 permutations, because the usual approximation gives too small p-values for sparsely detected features;
- **Fisher's exact** test, one-sided: detected more often in the top group.

If the cut-off of the top group falls on a large block of tied values (e.g., most samples at the lowest value, as for an endpoint marked "skip"), the top group is the samples strictly above that value and a warning is shown. The cell shows a p-value histogram (pure noise = flat), the number of hits per test (q < `FDR_LEVEL` = 0.10) and the top features per test. The screen is independent of the models and is not used in FSACI; it is a second, simpler line of evidence.

**6. Train Elastic Net.** `ElasticNetCV` with shuffled 5-fold CV chooses the mixing parameter `l1_ratio` (0.1–0.9) and the regularization strength alpha (60 values over three orders of magnitude below alpha_max, the smallest alpha that sets all coefficients to zero). Warns if the best alpha lies at the edge of the grid (largest alpha = no linear signal).

**7. Elastic Net performance.** Training metric (optimistic by definition) and the **repeated CV metric**: `N_REPEATS` repeats of `N_SPLITS`-fold CV, one pooled out-of-fold Q² or AUC per repeat. The CV is **nested**: alpha is re-tuned inside every training fold (inner 3-fold CV, `l1_ratio` fixed), so the test samples never influence the model.

**8. Permutation test – Elastic Net.** The same nested CV on `N_PERM` copies of the endpoint with shuffled values. p = (number of permuted scores ≥ observed + 1) / (`N_PERM` + 1); the smallest possible p is 1/101 = 0.0099 with 100 permutations. **Elastic Net has signal** if the CV metric is above chance (Q² > 0, AUC > 0.5) **and** p < 0.05.

**9. Elastic Net coefficients & outputs.** Coefficients of the model fitted on all samples (sign kept: + = higher concentration goes with a higher endpoint value), number of non-zero and positive coefficients, top 20; saves the coefficient tables.

**10. Stability selection.** `N_STABILITY_RUNS` Elastic Net fits (tuned alpha and `l1_ratio`) on random subsamples of `SUBSAMPLE_FRACTION` (75%) of the samples, in parallel; for classification each subsample keeps the active/inactive ratio. **Stability score** = share of runs in which a feature gets a non-zero coefficient. Shows the number of stable (≥ 0.6) and high-confidence (≥ 0.8) features and the distribution of the scores.

**11. Save stability outputs.** Saves the stability ranking and the stable and high-confidence feature lists.

**12. Prepare XGBoost input matrix & configuration.** XGBoost gets the stable features (stability ≥ 0.6), ordered by stability and then by \|Elastic Net coefficient\|, at most `XGB_MAX_FEATURES` (200). If fewer than 10 features are stable, the 10 most stable features are used; if no feature was ever selected, the XGBoost and SHAP steps are skipped (outputs are still saved). Shows the XGBoost parameters (shallow trees, slow learning rate, row and column subsampling).

**13. XGBoost CV & permutation test.** Repeated CV and permutation test as for the Elastic Net. Inside every training fold the features are **selected again** (`ENetTopKSelector`: Elastic Net with alpha re-tuned, non-zero coefficients first, at most as many features as XGBoost uses), so the CV metric is not inflated by features chosen on all samples. **XGBoost has signal** with the same rule as the Elastic Net.

**14. Train final XGBoost model and compute SHAP.** Fits XGBoost on all samples and computes the SHAP values (`shap.TreeExplainer`; if the shap package fails, XGBoost's built-in SHAP values are used). **SHAP importance** = mean \|SHAP value\| per feature; **SHAP direction** = rank correlation between the feature value and its SHAP value (+ = higher concentration pushes the prediction up).

**15. FSACI.** For every feature: **FSACI = stability score × effect score**. Effect score = percentile rank (among features with a non-zero value) of \|Elastic Net coefficient\| and, only if XGBoost has signal, of the SHAP importance (weights `W_ENET` = `W_SHAP` = 0.5). Direction = sign of the Elastic Net coefficient (else the SHAP direction). With `PRIORITIZE_POSITIVE_ONLY = True`, features linked to **lower** endpoint values get FSACI = 0 (their score is kept in `FSACI_any_direction`). Warns if neither model has signal (the ranking is then exploratory).

**16. Save model outputs.** Saves the XGBoost CV results, the SHAP ranking, the full FSACI ranking (with metadata, key columns first), the top 120 (and top 120 stable) features and `FSACI_<endpoint>.csv` for the cross-endpoint cells.

**17. Endpoint statistics summary.** One table with the endpoint distribution, all model statistics (Elastic Net, stability selection, XGBoost, SHAP/FSACI) and the run settings; saved as `stats_<endpoint>.csv` and transposed as `stats_<endpoint>_T.csv`.

**18. `# ML end`.** Marker: the endpoint run ends here.

### C. Cross-endpoint summary (after all endpoints)

**19. Cross-endpoint statistics summary.** Reads `stats_<endpoint>.csv` of all endpoints, adds derived columns (best CV metric and model, overfitting = training − CV metric, rough quality rating) and decides which endpoints enter the compound ranking:

- **included** if at least one model has signal **and** its CV metric reaches the minimum effect size (`MIN_Q2` = 0.10 for regression, `MIN_AUC` = 0.60 for classification), so that "significant but tiny" endpoints are left out;
- **endpoint weight** = model quality on a 0–1 scale: best CV Q² (regression) or (best CV AUC − 0.5) / 0.5 (classification) among the models with signal;
- **endpoint blocks** (`ENDPOINT_BLOCKS`): endpoints measuring the same biology (e.g., the four IL-6/IL-8 endpoints = inflammation) count once in the global score.

Saves `endpoint_model_summary.csv`, its transposed version and an `.xlsx` with an explanation sheet.

**20. Cross-endpoint compound summary.** Reads the FSACI files of all endpoints and builds:

- the **compound endpoint summary**: FSACI rank of every compound with FSACI > 0 for every endpoint (also the excluded ones);
- the **global ranking** over the included endpoints: each endpoint contributes its top `MAX_RANKED_PER_ENDPOINT` (100) compounds with an endpoint score = 1 − (rank − 1) / n (1 for rank 1, close to 0 for the last); block score = best weighted endpoint score in the block; **global priority score** = weighted mean of the block scores (weights = block weights). A compound that ranks high for well-modeled endpoints gets a high score; `n_blocks_hit` tells whether it is endpoint-specific (1 block) or active in several assays.

Saves both tables as `.csv` and, with metadata, as `.xlsx` with the sheets `data`, `endpoints` and `explanation`.

## User-settable parameters

Main parameters (all of them, incl. the univariate screen and XGBoost settings, in [parameters.md](parameters.md)):

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `DATA_DIR` | Load preprocessed data | `Path("../data")` | Your working folder is elsewhere |
| `PT1_DATE` | Load preprocessed data | `None` (most recent run) | To use an older Part 1 run, e.g. `"261018"` |
| `ENDPOINT_SHORT` | Endpoint selection | `"AhR"` | **Every run**: the endpoint to model |
| `TASK_OVERRIDE` | Endpoint selection | `None` | To force `"regression"` or `"classification"` |
| `FORCE_RUN` | Endpoint selection | `True` | `False` to stop at endpoints marked "skip" |
| `FAST_MODE` | Endpoint selection | `False` | `True` for a quick test |
| `N_SPLITS`, `N_REPEATS`, `N_PERM` | Endpoint selection | 5, 10, 100 | More repeats/permutations for more precise estimates |
| `N_STABILITY_RUNS`, `SUBSAMPLE_FRACTION` | Endpoint selection | 100, 0.75 | Rarely |
| `STABILITY_THRESHOLD`, `HIGH_CONF_THRESHOLD` | Endpoint selection | 0.6, 0.8 | Rarely |
| `XGB_MAX_FEATURES` | Endpoint selection | 200 | Rarely (more features than samples overfit) |
| `TOP_N_FSACI` | Endpoint selection | 120 | Size of the top list saved |
| `PRIORITIZE_POSITIVE_ONLY` | Endpoint selection | `True` | `False` to also rank compounds linked to lower endpoint values |
| `MIN_Q2`, `MIN_AUC` | Cross-endpoint statistics summary | 0.10, 0.60 | Stricter or looser inclusion of endpoints |
| `ENDPOINT_BLOCKS` | Cross-endpoint statistics summary | 6 blocks | Other endpoints or grouping |
| `MAX_RANKED_PER_ENDPOINT` | Cross-endpoint compound summary | 100 | Compounds per endpoint in the global score |

## Checks to look at

| Cell | What to look for | Interpretation |
|---|---|---|
| Load preprocessed data | `Pt1 run used: <yymmdd>`; Part 1 settings; "Samples aligned" | Wrong run → set `PT1_DATE` |
| Endpoint selection | Samples dropped; suggested task and reason; target; number of active samples (classification) | "FORCE_RUN" warning → results unreliable |
| Univariate screen | p-value histogram vs. "expected if noise"; hits per test; "No feature passes BH correction" warning | Many more small p-values than expected = real association; none = no detectable single-compound link |
| Train Elastic Net | Best alpha and `l1_ratio`; warning "Best alpha = largest alpha" | Largest alpha = no linear signal; low `l1_ratio` = diffuse signal spread over many correlated features |
| Elastic Net performance | Training vs. CV metric; CV SD | Judge the model by the CV metric only; a large gap = overfitting |
| Permutation test – Elastic Net | p-value; "Elastic Net has signal" | No signal → compound ranking exploratory |
| Stability selection | Number of stable features; distribution of scores | Very few stable features = weak or diffuse signal |
| XGBoost CV & permutation test | CV metric vs. Elastic Net; "XGBoost has signal" | XGBoost better → non-linear effects; worse → the linear model is sufficient |
| SHAP | "shap package failed" warning | Built-in XGBoost SHAP values are used instead (same meaning) |
| FSACI | Features with FSACI > 0; top feature and its stability; "Neither model has signal" warning | Rankings of endpoints without signal are excluded from the global ranking |
| Cross-endpoint statistics summary | Included/excluded endpoints and reasons; weights | Excluded endpoints give no reliable compound ranking |
| Cross-endpoint compound summary | Blocks used; compounds per endpoint; "Main block of the top 50 compounds" | One block taking (almost) all top compounds → check weights/blocks |

## Common errors

| Message | Cause | Solution |
|---|---|---|
| `FileNotFoundError: No Pt1 outputs found ...` | Part 1 not run, or `DATA_DIR` wrong | Run Part 1; check `DATA_DIR` |
| `FileNotFoundError: Pt1 files missing for run <yymmdd>` | Part 1 run incomplete (stopped before the save cell) or wrong `PT1_DATE` | Re-run Part 1 completely or fix `PT1_DATE` |
| `ValueError: ... NaNs in the modeling matrix` | `NAN_TO_ZERO = False` in Part 1 | Set `NAN_TO_ZERO = True` and re-run Part 1 |
| `ValueError: Unknown ENDPOINT_SHORT` | Typo in `ENDPOINT_SHORT` | Use one of the keys of `ENDPOINTS` |
| `RuntimeError: Endpoint '<x>' skipped: ...` | Plan "skip" and `FORCE_RUN = False` | Accept, or set `FORCE_RUN = True` (results unreliable) |
| `RuntimeError: Too few samples in one class for 5-fold CV` | Fewer than 5 active or inactive samples | Endpoint cannot be modeled |
| `ValueError: Top group cannot be formed ...` | Almost all samples share one value | Endpoint cannot be screened (skip the screen cell) |
| `FileNotFoundError: No endpoint statistics files were found` | Cross-endpoint cells run before any endpoint | Run the endpoints first |
| `RuntimeError: No endpoint passes the inclusion rules` | No endpoint has a model with signal and sufficient effect size | Report this; optionally lower `MIN_Q2` / `MIN_AUC` (state it) |
| `RuntimeError: Run the 'Cross-endpoint statistics summary' cell first` | Cells run in the wrong order | Run cell 19 before cell 20 |
| `PermissionError` when saving | Output file is open in Excel | Close the file and re-run the cell |
