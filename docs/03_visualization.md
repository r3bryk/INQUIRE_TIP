# Part 3 – INQUIRE_TIP_Pt3_Visualization

## Purpose

Part 3 makes figures from the saved outputs of Parts 1 and 2 (no variables from Parts 1–2 are needed):

| Figure | Content |
|---|---|
| Fig 1 | Model performance per endpoint: cross-validated Q² (regression) and AUC (classification) of Elastic Net and XGBoost, with permutation p-values |
| Fig 2 | Measured vs. cross-validated predicted values (regression) and ROC curves (classification) for the endpoints used in the compound ranking |
| Fig 3 | Chemical class enrichment among the top prioritized compounds per endpoint |
| Fig 4 | Endpoint vs. concentration of the top compounds of each endpoint (one figure per endpoint) |
| Fig 5 | Cross-endpoint heatmap of the top compounds of the global ranking |

Every figure is saved as PDF (vector, editable text), SVG and PNG (600 dpi), 180 mm wide (double column), and a caption draft with the key numbers is written to `figure_notes.txt`.

**All features or identified compounds only.** With `IDENTIFIED_ONLY = False` (default) Figs 3–5 show all features, including unknown features. With `IDENTIFIED_ONLY = True` they show identified compounds only (unknown features removed from the top lists **and** from the background of the enrichment test); these files get the suffix `_identified`. Run the notebook once per setting to get both sets. Figs 1 and 2 do not depend on this setting (the models use all features) and are saved without suffix.

## Inputs

| Input | Source |
|---|---|
| `cross_endpoint_summary/endpoint_model_summary.csv` | Part 2, cross-endpoint statistics summary (required) |
| `cross_endpoint_summary/compound_global_summary_w_metadata.csv` | Part 2, cross-endpoint compound summary (Fig 5; skipped with a warning if missing) |
| `endpoint_models/<endpoint>/fsaci_rank.csv`, `stats_<endpoint>.csv` | Part 2, every endpoint |
| `<yymmdd>_tox_endpoints_unscaled.csv` | Part 1 (run `PT1_DATE`, or the most recent run) |
| `tox_vs_dust_merged.csv` (`TOX_DUST_FILE`) | Raw input (concentrations for Fig 4; the modeling data are rebuilt from it) |
| `feature_metadata_classified.xlsx` (`META_CLASS_FILE`) | Raw input (names and classes) |

## Outputs

In `inquire_tip_outputs/inquire_tip_figures/` (`FIG_DIR`); `<s>` = empty or `_identified`:

| File | Content |
|---|---|
| `fig1_model_performance.{pdf,svg,png}` | Fig 1 |
| `fig2_measured_vs_predicted.{pdf,svg,png}` | Fig 2 |
| `fig3_class_enrichment<s>.{pdf,svg,png}` | Fig 3 |
| `fig3_class_enrichment_table<s>.csv` | Numbers behind Fig 3 (counts, fold enrichment, p, q) |
| `fig4_<endpoint>_top_compounds<s>.{pdf,svg,png}` | Fig 4, one file per included endpoint |
| `fig5_cross_endpoint_heatmap<s>.{pdf,svg,png}` | Fig 5 |
| `figure_notes<s>.txt` | Caption drafts with the key numbers of each figure |
| `cv_predictions/cv_predictions_<endpoint>.csv` | Cache of the cross-validated predictions of Fig 2 (reused unless `RECOMPUTE_CV = True`) |

Log file in `notebooks/`: `inquire_tip_pt3_visualization_<yymmdd_HHMMSS>.log`.

## Notebook sections in run order

**0. Title.** Purpose; run the cross-endpoint cells of Part 2 first.

**1. Setup.** Packages, logging, paths (`DATA_DIR`, `OUT_DIR`, `PT1_DATE`, input files), settings that must match Parts 1–2 (`NEG_HANDLING`, `GLOBAL_SEED`, `N_SPLITS`), the `IDENTIFIED_ONLY` switch, figure settings, endpoint names, labels, units and blocks, a fixed color per endpoint block (colorblind-checked palette) and the figure style (Arial/Helvetica 7 pt, editable text in PDF/SVG).

**2. Load saved outputs.** Loads the endpoint model summary (stops if it is missing or from an older version of Part 2), the global compound ranking, the FSACI ranking of every endpoint, the classified metadata (empty classes → "Unclassified"), the endpoint values (Part 1) and the raw concentrations of the modeled features. Counts identified and unknown features (overall and among the prioritized compounds per endpoint).

**3. Rebuild modeling data.** Repeats the preprocessing of Part 1 for the modeled features (missing → 0, below-LOD handling, log10(x + c), z-scaling on each endpoint's samples) and the endpoint transformation of Part 2. A QC table compares the numbers of samples and features with Part 2; a warning means different settings or input files.

**4. Fig 1 – model performance.** One panel for the regression endpoints (Q²) and one for the classification endpoints (AUC): Elastic Net (filled circles) and XGBoost (open diamonds), mean ± SD over the CV repeats; solid line = chance level; dashed line = minimum effect size for the compound ranking; colored = endpoint used in the compound ranking, gray = not used; permutation p-values on the right.

**5. Fig 2 – measured vs. predicted / ROC.** For every endpoint used in the compound ranking: cross-validated Elastic Net predictions (one CV repeat with the settings of Part 2; alpha re-tuned inside every training fold). Regression: measured vs. predicted on the modeling scale with the 1:1 line; classification: ROC curve. The predictions are cached in `cv_predictions/` (set `RECOMPUTE_CV = True` to recompute). The Q²/AUC of this single repeat can differ slightly from the mean over the repeats of Part 2 (both are given in the caption draft).

**6. Fig 3 – class enrichment.** Per endpoint: the top `N_TOP_ENRICH` (100) compounds by FSACI (FSACI > 0) vs. all features used for that endpoint. Fold enrichment = share of a class among the top compounds / share among all features (colored only if > 1, capped at 8-fold); numbers = compounds of that class among the top compounds; * = q < 0.05 (one-sided hypergeometric test, BH correction across all cells). Classes with on average fewer than `MIN_CLASS_SIZE` (15) features are pooled as "Other classes". Endpoints without prioritized compounds are left out.

**7. Fig 4 – endpoint vs. concentration.** For the top `N_TOP_COMPOUNDS` (6) compounds of each included endpoint. Regression: endpoint (modeling scale) vs. log10 concentration; filled = detected, open colored = below-LOD estimate, open gray at "n.d." = not detected; line = linear fit to the detected values; Spearman ρ over all samples. Classification: log10 concentration in inactive vs. active samples; bar = median of the detected values; AUC = probability that an active sample has a higher concentration than an inactive one. With `IDENTIFIED_ONLY = True` the titles also show the FSACI rank among all prioritized features.

**8. Fig 5 – cross-endpoint heatmap.** The top `N_TOP_HEATMAP` (30) compounds of the global ranking: class and name (with global rank), endpoint score per included endpoint (color; number = FSACI rank; gray = not among the endpoint's top compounds) and the global priority score (bars).

## Reading the figures

- Fig 4 shows single-compound associations of compounds that were **selected on the same data**: they look stronger than they would in independent data. A compound ranked high by the multivariate models can show a weak or even negative single-compound association, because the models credit each compound given all others. The figure illustrates the data; it does not establish causality.
- Fig 3 tests over-representation of classes among the top compounds relative to the modeled features, not relative to all chemicals; classes depend on the classification in the metadata file.

## User-settable parameters

| Parameter | Cell | Default | Change when |
|---|---|---|---|
| `DATA_DIR` | Setup | `Path("../data")` | Your working folder is elsewhere |
| `PT1_DATE` | Setup | `None` (most recent Part 1 run) | Part 2 used an older Part 1 run (`PT1_DATE` in Part 2) |
| `TOX_DUST_FILE`, `META_CLASS_FILE`, `SAMPLE_COL_RAW` | Setup | as in Part 1; `"sample_base"` | Must be the same files as in Part 1 |
| `NEG_HANDLING`, `GLOBAL_SEED`, `N_SPLITS` | Setup | `"abs"`, 1, 5 | Must match Parts 1–2 |
| `RECOMPUTE_CV` | Setup | `False` | `True` after re-running Part 2 (otherwise old cached predictions are reused if the samples match) |
| `IDENTIFIED_ONLY` | Setup | `False` | `True` for figures with identified compounds only |
| `UNKNOWN_NAME_TEXT`, `NO_NAME_IS_UNKNOWN` | Setup | `"Feature"`, `True` | Other naming of unknown features |
| `N_TOP_ENRICH`, `MIN_CLASS_SIZE` | Setup | 100, 15 | Fig 3 |
| `N_TOP_COMPOUNDS` | Setup | 6 | Fig 4 |
| `N_TOP_HEATMAP` | Setup | 30 | Fig 5 |
| `FIG_WIDTH_FULL`, `FIG_FORMATS` | Setup | 7.09 in (180 mm), pdf/svg/png | Journal requirements |
| `ENDPOINTS`, `ENDPOINT_LABELS`, `ENDPOINT_UNITS`, `ENDPOINT_BLOCKS` | Setup | 11 endpoints, 6 blocks | Other endpoints (blocks as in Part 2) |

## Checks to look at

| Cell | What to look for | Interpretation |
|---|---|---|
| Load saved outputs | Endpoints in the summary and used in the ranking; tox endpoint file name (Part 1 run); identified vs. unknown counts | Wrong Part 1 run → set `PT1_DATE` |
| Rebuild modeling data | "Samples and features match part 2 for all endpoints" | Otherwise check `NEG_HANDLING`, `PT1_DATE` and the input files |
| Fig 2 | "cached CV predictions loaded" vs. "computed" | After a new Part 2 run set `RECOMPUTE_CV = True` |
| Fig 3 | "Significant enrichments (q < 0.05)"; endpoints left out | Information |
| Fig 4 / Fig 5 | Warnings about endpoints without (identified) prioritized compounds | Figure skipped or column left out |

## Common errors

| Message | Cause | Solution |
|---|---|---|
| `FileNotFoundError: ... endpoint_model_summary.csv not found` | Cross-endpoint cells of Part 2 not run | Run them (Part 2, cells 19–20) |
| `ValueError: Columns missing in endpoint_model_summary.csv ...` | Summary from an older version of Part 2 | Re-run the cross-endpoint statistics summary cell |
| `FileNotFoundError: <yymmdd>_tox_endpoints_unscaled.csv not found` | Wrong `PT1_DATE` | Set `PT1_DATE` to an existing Part 1 run |
| `ValueError: Sample column 'sample_base' not found` | Other sample ID column in the raw file | Set `SAMPLE_COL_RAW` |
| `RuntimeError: No endpoint is used in the compound ranking: nothing to plot` | No endpoint passed the inclusion rules in Part 2 | See Part 2, cross-endpoint statistics summary |
| `ValueError: No compounds left to plot ...` | `IDENTIFIED_ONLY = True` and no identified compound in the global ranking | Use `IDENTIFIED_ONLY = False` |
| Fonts differ from Arial | Arial not installed | Matplotlib falls back to Helvetica, Liberation Sans or DejaVu Sans |
