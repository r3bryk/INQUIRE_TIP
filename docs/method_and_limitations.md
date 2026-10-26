# Method, reference run and limitations

## Model in brief

### What is prioritized

For each toxicity endpoint, INQUIRE_TIP ranks the chemical features (identified compounds and unknown features) whose concentrations in the samples go together with **higher** endpoint values (more toxic activity). The data typically have far more features than samples (p ≫ n) and many correlated features (compounds from the same sources), so the method combines regularized models, resampling and strict validation, and ranks compounds by how **consistently** (stability) and how **strongly** (effect) they contribute to a model that has been shown to predict the endpoint better than chance.

### Preprocessing (Part 1)

- **Detection-frequency filter:** features detected (value > 0) in fewer than `MIN_DETECT_FRAC` (10%) of the samples or with fewer than 5 distinct values are not modeled; they cannot be linked reliably to toxicity with ~150 samples and mainly add noise. They are still tested in the univariate screen.
- **Below-LOD values** (negative numbers, i.e., concentration estimates below the LOD) count as not detected for the filter and are then kept as low concentration estimates (`NEG_HANDLING = "abs"`).
- **log10(x + c)**, c = half of the smallest detected value of the feature, then **z-scaling** (as R `scale()`), repeated on the samples of each endpoint in Part 2. Non-detects get the lowest value, below all detections.
- Endpoint values are not imputed: each endpoint uses all samples with a value.

### Modeling task per endpoint

Endpoint data often contain a block of identical lowest values: the substitute for values below the limit of quantification (LOQ/√2; Hornung and Reed 1990) or 0 = no effect. Such values carry no information on *how* low the activity is. Therefore (`plan_endpoint()`):

- **classification** (active = above the lowest value vs. inactive) if ≥ 35% of the samples share the lowest value and both classes have ≥ 15 samples;
- **skip** if a class has < 15 samples (can be run anyway with `FORCE_RUN = True`, results unreliable);
- otherwise **regression**, on log10 scale if the skewness is > 1.

For classification the Elastic Net is fitted to the 0/1 target (a linear probability model) and both models are evaluated by the ROC AUC; XGBoost uses a logistic objective.

### Elastic Net with nested, repeated cross-validation

The Elastic Net (Zou and Hastie 2005) combines L1 (lasso: sparse selection) and L2 (ridge: correlated features share weight) penalties. `ElasticNetCV` chooses the mix (`l1_ratio` 0.1–0.9) and the strength alpha (60 values over three orders of magnitude below alpha_max) by 5-fold CV. A low `l1_ratio` means a diffuse signal spread over many correlated features; a high one a specific signal of few features.

Performance is estimated by **repeated, nested CV**: 10 repeats of shuffled (stratified for classification) 5-fold CV, with alpha re-tuned inside every training fold (inner 3-fold CV). Tuning on all samples and then cross-validating would let the test samples leak into the model and inflate the result (Varma and Simon 2006). Per repeat, all out-of-fold predictions are pooled into one metric: **Q²** (R² of the out-of-fold predictions; 0 = no better than predicting the mean, negative = worse) for regression and **ROC AUC** (0.5 = chance) for classification. Pooling is more stable than averaging per-fold R² on ~30 test samples.

### Permutation test and "signal"

The same nested CV is repeated on 100 copies of the endpoint with shuffled values (Ojala and Garriga 2010). p = (number of permuted scores ≥ observed + 1) / (100 + 1), so the smallest possible p is 0.0099 (report as p ≤ 0.01). A model **has signal** if its CV metric is above chance **and** p < 0.05.

### Stability selection

The Elastic Net (tuned alpha and `l1_ratio`) is refitted on 100 random subsamples of 75% of the samples (class ratio kept for classification). The **stability score** of a feature is the share of fits in which it gets a non-zero coefficient (after Meinshausen and Bühlmann 2010). Features with a score ≥ 0.6 are "stable", ≥ 0.8 "high-confidence". Correlated features from the same source can split the selections between them, which lowers their individual scores.

### XGBoost and SHAP

XGBoost (Chen and Guestrin 2016) is fitted to the stable features (ordered by stability and \|coefficient\|, at most 200) to capture non-linear effects and interactions, with shallow, regularized trees (300 trees, depth 3, learning rate 0.03, row subsampling 0.8, column subsampling 0.5). Its CV and permutation test repeat the **feature selection inside every training fold** (Elastic Net with re-tuned alpha), so that features chosen with the real endpoint values on all samples do not inflate the CV metric or invalidate the permutation test. SHAP values (Lundberg and Lee 2017; TreeExplainer, Lundberg et al. 2020) of the final model give each feature's contribution to each prediction; SHAP importance = mean \|SHAP\|, SHAP direction = rank correlation between feature value and SHAP value.

### FSACI

The **Final Stability-Adjusted Compound Index** of a feature for one endpoint is

**FSACI = stability score × effect score**,

where the effect score is the percentile rank (among features with a non-zero value) of \|Elastic Net coefficient\| and, if XGBoost has signal, the mean of this and the percentile rank of the SHAP importance (weights 0.5/0.5). Percentile ranks are used instead of min-max scaling so that one extreme value does not squash all others; the product means a compound must be **both** stable and influential to rank high. The direction is the sign of the Elastic Net coefficient (else of the SHAP direction); with `PRIORITIZE_POSITIVE_ONLY = True` compounds linked to lower endpoint values get FSACI = 0. FSACI ranges from 0 to 1 and ranks compounds **within** an endpoint.

### Univariate screen

Independently of the models, every feature detected in ≥ 5 samples (raw data, no detection-frequency filter) is tested on its own: Spearman correlation with the endpoint, one-sided Mann-Whitney U test (top 10% samples vs. the rest; permutation p-values) and one-sided Fisher's exact test (detection in the top group), each with Benjamini-Hochberg correction (Benjamini and Hochberg 1995). It shows whether single compounds are associated with the endpoint and catches rare compounds present mainly in the most active samples. It is not used in FSACI.

### Cross-endpoint ranking

An endpoint enters the global ranking if at least one model has signal **and** reaches a minimum effect size (CV Q² ≥ 0.10 or CV AUC ≥ 0.60); a statistically significant but tiny effect is of little practical use. Endpoint weight = model quality on a 0–1 scale (Q², or (AUC − 0.5)/0.5). Endpoints measuring the same biology form a block that counts once (cytotoxicity; inflammation = IL-6 and IL-8 at both doses; AhR; androgen receptor; estrogen receptor; thyroid receptor). Each included endpoint contributes its top 100 compounds (FSACI > 0) with an endpoint score from 1 (rank 1) to near 0; block score = best weighted endpoint score in the block; **global priority score** = weighted mean of the block scores. Limiting every endpoint to its top 100 prevents endpoints with very long lists from dominating, and the weighted mean favors compounds that rank high over compounds that merely appear in many lists.

## Reference run

INQUIRE house dust, analyzed by GCxGC-MS (non-polar constituents) and LC-MS (polar constituents) based non-target screening; default settings (`FAST_MODE = False`).

**Data and preprocessing**

| Item | Value |
|---|---|
| Samples | 167 (no duplicated or missing IDs) |
| Endpoints | 11; missing values per endpoint 0–16 |
| Dust features | 27,017 (5,372 GCxGC-MS, 21,645 LC-MS); 1,626 never detected |
| Below-LOD values | 390,261 values in 5,325 GCxGC-MS features |
| Features kept (detected in ≥ 10% of samples, ≥ 5 distinct values) | 7,853 (GCxGC-MS 4,925; LC-MS 2,928) |
| Identified features among the modeled features | 2,298 of 7,853 |

**Models per endpoint** (CV = mean over 10 repeats of 5-fold CV; p = permutation p-value, 100 permutations; – = XGBoost not run because no feature was ever selected)

| Endpoint | Task (metric) | n | Elastic Net CV | p | XGBoost CV | p | Included | Weight | FSACI > 0 |
|---|---|---|---|---|---|---|---|---|---|
| Cytotoxicity (`cell_viability`) | classification (AUC) | 167 | 0.695 | ≤ 0.01 | 0.680 | ≤ 0.01 | yes | 0.390 | 49 |
| IL-6, 0.1 mg/ml | regression (Q²) | 151 | 0.009 | 0.050 | −0.064 | 0.069 | no (Q² < 0.10) | 0 | 671 |
| IL-6, 1 mg/ml | regression (Q²) | 151 | 0.243 | ≤ 0.01 | 0.200 | ≤ 0.01 | yes | 0.243 | 1,389 |
| IL-8, 0.1 mg/ml | regression (Q²) | 151 | 0.035 | ≤ 0.01 | 0.060 | ≤ 0.01 | no (Q² < 0.10) | 0 | 601 |
| IL-8, 1 mg/ml | regression (Q²) | 151 | 0.334 | ≤ 0.01 | 0.242 | ≤ 0.01 | yes | 0.334 | 2,110 |
| Androgenic | regression (Q²) | 164 | −0.013 | 0.059 | −0.312 | 0.41 | no (no signal) | 0 | 19 |
| Anti-androgenic | regression (Q²) | 159 | −0.029 | 0.73 | – | – | no (no signal) | 0 | 0 |
| Estrogenic | classification (AUC), plan "skip" | 152 | 0.479 | 0.16 | – | – | no (no signal) | 0 | 0 |
| Anti-estrogenic | regression (Q²) | 157 | −0.023 | 0.39 | −0.261 | 0.54 | no (no signal) | 0 | 6 |
| AhR | regression, log10 (Q²) | 161 | 0.274 | ≤ 0.01 | 0.211 | ≤ 0.01 | yes | 0.274 | 95 |
| TR agonism | classification (AUC) | 152 | 0.610 | 0.020 | 0.593 | 0.020 | yes | 0.220 | 11 |

**Cross-endpoint ranking:** 5 endpoints in 4 blocks (cytotoxicity, inflammation, AhR, thyroid receptor); 323 compounds in the global ranking, of which 4 rank in the top set of two blocks. As a plausibility check, PAHs and fused aromatics were strongly over-represented among the 95 compounds prioritized for AhR activity (18 of 95, 5.7-fold, q = 2 × 10⁻⁷; Part 3, Fig 3), consistent with PAHs being AhR agonists.

How to read these numbers: Q² values of 0.25–0.35 mean that the dust composition explains about a quarter to a third of the variation of the endpoint in new samples – a real but partial explanation. The Elastic Net was as good as or better than XGBoost for most endpoints, i.e., the signal is mainly linear. For the receptor endpoints other than AhR and TR, no model predicted better than chance with this number of samples; their compound lists are not interpretable.

## Limitations and how results should be reported

- **Associations, not causation.** A compound ranks high if its concentrations go together with the endpoint. Compounds from the same source (e.g., combustion, plastics) co-occur, so a non-toxic compound can rank high because it travels with the active one. Confirm candidates with the literature, effect-directed analysis or tests of the pure compounds.
- **Correlated features share credit.** The Elastic Net and stability selection may pick one of several correlated features or spread the weight over all of them; a moderate stability score does not mean a compound is irrelevant.
- **Many features, few samples.** With ~7,900 features and ~150–167 samples, the models can capture only the strongest and most common patterns. Rare compounds (detected in < 10% of samples) are not modeled; check the univariate screen for them.
- **Semi-quantitative data.** Many features are normalized peak areas, not concentrations, and response factors differ between compounds. The models compare samples within a feature (after log-transformation and scaling), not features with each other. Below-LOD estimates are less precise.
- **Unknown features** can rank high; they need identification before interpretation (Part 3 can show identified compounds only).
- **Endpoint preprocessing before the pipeline** (LOQ/√2 substitution, removal of outliers and negative values) cannot be undone. Classification of censored endpoints uses less information than a regression.
- **Validation is internal.** Cross-validation and permutation tests show that a model predicts better than chance within this data set. An external test set was not available; Q²/AUC may be lower in other populations.
- **FSACI is a ranking score**, not an effect size or a toxic potency; it is not comparable between endpoints, and its top positions depend on the tie-breaking of many small values. Report ranks together with the stability score and the model performance of the endpoint.
- **Global priority score** depends on the inclusion thresholds (`MIN_Q2`, `MIN_AUC`), the endpoint blocks and weights; report these settings.
- **Permutation p-values** have a floor of 1/(N_PERM + 1) (0.0099 with 100 permutations) and are computed for one CV split; report them as p ≤ 0.01.
- **Fig 4** shows associations of compounds that were selected on the same data and therefore looks optimistic.

**Report:** the CV metric (mean ± SD) and permutation p-value of each endpoint; which endpoints were included and why; the settings (detection-frequency threshold, `NEG_HANDLING`, CV repeats, permutations, stability threshold, inclusion thresholds, blocks); and the compound rankings as candidates for confirmation. The file `<yymmdd>_preprocessing_description.xlsx` (Part 1) and the `explanation` sheets of the Part 2 `.xlsx` files provide ready-made descriptions.

## Optional checks for users

- Compare the Elastic Net CV metric with the training metric (`Elastic Net overfit (train - CV)`): large gaps are expected with p ≫ n; judge only the CV metric.
- Re-run an endpoint with another `GLOBAL_SEED` to see how much the ranks of the top compounds change.
- Re-run Part 1 with `MIN_DETECT_FRAC = 0.20` or `NEG_HANDLING = "zero"` to check the sensitivity of the rankings to preprocessing.
- Compare the FSACI top compounds with the univariate screen hits of the same endpoint: compounds found by both are the most robust candidates.

## References

- Benjamini Y, Hochberg Y (1995) Controlling the false discovery rate: a practical and powerful approach to multiple testing. *Journal of the Royal Statistical Society B* 57(1):289–300.
- Chen T, Guestrin C (2016) XGBoost: a scalable tree boosting system. *Proceedings of the 22nd ACM SIGKDD International Conference on Knowledge Discovery and Data Mining*, 785–794.
- Hornung RW, Reed LD (1990) Estimation of average concentration in the presence of nondetectable values. *Applied Occupational and Environmental Hygiene* 5(1):46–51.
- Lundberg SM, Lee S-I (2017) A unified approach to interpreting model predictions. *Advances in Neural Information Processing Systems* 30:4765–4774.
- Lundberg SM, et al. (2020) From local explanations to global understanding with explainable AI for trees. *Nature Machine Intelligence* 2:56–67.
- Meinshausen N, Bühlmann P (2010) Stability selection. *Journal of the Royal Statistical Society B* 72(4):417–473.
- Ojala M, Garriga GC (2010) Permutation tests for studying classifier performance. *Journal of Machine Learning Research* 11:1833–1863.
- Pedregosa F, et al. (2011) Scikit-learn: machine learning in Python. *Journal of Machine Learning Research* 12:2825–2830.
- Varma S, Simon R (2006) Bias in error estimation when using cross-validation for model selection. *BMC Bioinformatics* 7:91.
- Zou H, Hastie T (2005) Regularization and variable selection via the elastic net. *Journal of the Royal Statistical Society B* 67(2):301–320.
