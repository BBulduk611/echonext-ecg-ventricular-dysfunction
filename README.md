# Predicting left and right ventricular systolic dysfunction from routine ECG measurements (EchoNext)

Analysis code for the study **"Predicting left and right ventricular systolic dysfunction from routine electrocardiogram measurements: explainable machine learning in EchoNext"** (manuscript in preparation).

## Aim

To evaluate how well five routine ECG measurements (ventricular rate, atrial rate, PR interval, QRS duration, QTc), together with age and sex, predict two echocardiography-confirmed outcomes:

- **Primary:** left ventricular ejection fraction (LVEF) ≤45% (`lvef_lte_45_flag`)
- **Secondary:** moderate or severe right ventricular (RV) systolic dysfunction (`rv_systolic_dysfunction_moderate_or_greater_flag`)

## Data source

The study uses the **EchoNext** dataset, **version 1.0.0**, from PhysioNet (file `EchoNext_metadata_100k.csv`).

- Dataset: https://doi.org/10.13026/r9pp-3y42
- Access: restricted. Users must register on PhysioNet and sign the **PhysioNet Restricted Health Data Use Agreement 1.5.0**.

**This repository contains no data.** It contains no patient records, raw ECG waveforms or personal information, and it must not be used to redistribute the data. The `.gitignore` file blocks CSV files from being committed by mistake. The notebook writes aggregate results only (summary tables and figures).

## Repository contents

| File | Description |
|---|---|
| `EchoNext_LVEF_RV_analysis.ipynb` | Complete analysis pipeline (Jupyter notebook) |
| `requirements.txt` | Python packages and the versions used in the study |
| `LICENSE` | MIT license for the code |
| `.gitignore` | Prevents data files and outputs from being committed |

## Methods implemented in the notebook

1. **Analysis set.** Uses the predefined patient-level train/validation/test split of the dataset. ECGs labeled `no_split` (additional ECGs of validation/test patients) are excluded to prevent leakage, and patient overlap between splits is checked.
2. **Data cleaning.** Physiologically implausible values are set to missing: ventricular rate 20–300/min, atrial rate 20–600/min, PR 60–600 ms, QRS 40–260 ms, QTc 250–750 ms.
3. **Models.** For each outcome there are three predictor sets (age + sex, ECG only, age + sex + ECG) and two models:
   - **Logistic regression:** median imputation with missingness indicators, standardization, `LogisticRegression(max_iter=2000)`.
   - **XGBoost:** `learning_rate=0.03`, `max_depth=4`, `min_child_weight=5`, `subsample=0.8`, `colsample_bytree=0.8`, `reg_lambda=1.0`, `tree_method="hist"`. Early stopping after 100 rounds without improvement in validation AUROC (maximum 3000 trees). Missing values are handled natively.
4. **Performance on the test set.**
   - AUROC, AUPRC and Brier score with 1000-replicate bootstrap 95% CIs.
   - Paired bootstrap differences in AUROC.
   - Sensitivity, specificity, PPV and NPV (Wilson 95% CIs) at the Youden threshold selected on the validation set.
5. **Calibration.**
   - Calibration plots by deciles of predicted risk.
   - Calibration intercept and slope.
   - Platt recalibration on the validation set.
6. **Explainability.** SHAP values (`TreeExplainer`) for 5000 test records, with summary and dependence plots.
7. **Subgroup analyses.** By sex, age group and QRS duration.
8. **Sensitivity analyses.**
   - One ECG per patient in the training set.
   - Thresholds targeting 90% and 95% sensitivity.
   - Records with missing vs complete ECG measurements.
   - A logistic regression reference model using the predictors of a previous study (Kim et al., 2026).
9. **Simple ECG rules and decision curve analysis.**
   - Comparison with QRS/QTc rules at a sensitivity matched on the validation set.
   - Net benefit of the recalibrated model.

A fixed random seed (`SEED = 2026`) is used for all random operations.

## Notebook sections and manuscript outputs

| Notebook section | Output files (in `echonext_outputs/`) | Manuscript |
|---|---|---|
| 1. Data loading and analysis set | `echonext_results.xlsx` (Flow, Cleaning) | Results: sample |
| 2. Table 1 | `echonext_results.xlsx` (Table1_*) | Tables 1–2 |
| 3. Descriptive figure | `fig1_prevalence` | Figure 1 |
| 4. Models | `echonext_results.xlsx` (Performance, AUROC_comparisons) | Table 3 |
| 5. ROC and calibration | `fig2_roc`, `fig3_calibration` | Figures 2, 3A |
| 6. SHAP | `fig4_shap_beeswarm_*`, `fig5_shap_dependence_*` | Figures 4–5 |
| 7. Subgroups | `fig6_subgroups` | Figure 6 |
| 8. One ECG per patient | `echonext_results.xlsx` (Sensitivity) | Results |
| 10. Additional analyses | `reviewer_analyses.xlsx` | Calibration, Table 5, additional analyses |
| 11. Simple rules (validation threshold) | `rules_validation_threshold.xlsx` | Table 4 |
| 12. Recalibration | `fig3b_recalibration`, `recalibration.xlsx` | Figure 3B |
| 13. Decision curve analysis | `fig7_decision_curve`, `rules_and_dca.xlsx` | Figure 7 |

### Which rule comparison is reported

The notebook contains two comparisons with simple ECG rules. Only one of them is reported in the manuscript.

| | Section 11 (reported) | Section 13, first part (not reported) |
|---|---|---|
| How the model threshold is chosen | Matched to each rule's sensitivity on the **validation set** | Matched to each rule's sensitivity on the **test-set** ROC curve |
| Output | `rules_validation_threshold.xlsx` | `rules_and_dca.xlsx`, sheet `Rules_vs_model` |
| Used in manuscript | **Yes – Table 4** | **No** (earlier version, superseded) |

The earlier comparison selects the threshold on the same data used for evaluation, which can be optimistic. It was replaced by the validation-set approach and is kept unchanged only for transparency. The **decision curve analysis** in the second part of Section 13 (`DCA_selected`, `DCA_full`, `fig7_decision_curve`) **is** used in the manuscript (Figure 7).

## Verification against the manuscript

The notebook is saved without outputs. The study results were produced by running this code in Google Colab on the EchoNext data (version 1.0.0). Only formatting differs from the code that was run: comments, headings and printed messages were translated into English. This was checked by comparing the syntax trees of the code before and after translation.

All numerical results reported in the manuscript (Tables 1–5, the text and Figures 1–7) were checked against the aggregate output files of those runs. The additional analyses (Section 10) were run in two separate sessions and gave identical results.

**Independent rerun (November 2026).** This exact notebook was run from start to finish in a new Google Colab session with the package versions in `requirements.txt`. Every value in all five output workbooks (`echonext_results.xlsx`, `reviewer_analyses.xlsx`, `recalibration.xlsx`, `rules_and_dca.xlsx`, `rules_validation_threshold.xlsx`) was identical to the original results. Eight of the ten figures were pixel-identical. The two SHAP beeswarm plots (Figure 4) differed only in the drawing order of overlapping points, which `shap.summary_plot` randomizes. The SHAP values themselves (sheet `Importance`) and the dependence plots (Figure 5) were identical.

Key values for checking a rerun (test set, full XGBoost model, n=5442):

| Outcome | AUROC (95% CI) | AUPRC (95% CI) | ΔAUROC vs age + sex | Sensitivity / specificity at validation Youden threshold |
|---|---|---|---|---|
| LVEF ≤45% | 0.795 (0.780–0.811) | 0.478 (0.446–0.511) | 0.158 (0.139–0.178) | 75.8% / 68.5% |
| RV systolic dysfunction | 0.795 (0.774–0.814) | 0.268 (0.231–0.309) | 0.253 (0.226–0.281) | 74.7% / 70.6% |

Small differences can occur with package versions other than those in `requirements.txt`.

## How to run

### Option A – Google Colab (as in the study)

1. Open `EchoNext_LVEF_RV_analysis.ipynb` in Google Colab.
2. Run all cells. When asked, select your own copy of `EchoNext_metadata_100k.csv`.
3. The results are written to `echonext_outputs/`.

### Option B – Local computer

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt jupyter
jupyter notebook EchoNext_LVEF_RV_analysis.ipynb
```

1. Set `DATA_PATH` in cell 2 to the location of `EchoNext_metadata_100k.csv`.
2. Run all cells.

Settings in cell 2:
- `DATASET_VERSION = "1.0.0"`
- `SEED = 2026`
- `N_BOOT = 1000` bootstrap replicates
- `SHAP_SAMPLE = 5000`

The full run takes several minutes, mainly because of the bootstrap.

## Notes

- If `xgboost` or `shap` is not installed, the notebook falls back to `HistGradientBoostingClassifier` or permutation importance and prints a warning. The study results were produced with xgboost 3.4.1 and shap 0.54.1, so install the packages in `requirements.txt` to reproduce them.
- Package versions are saved to `echonext_outputs/software_versions.csv` at each run.
- In the study, sections 1–11 were run as one notebook. Sections 12 and 13 were run afterwards in the same way. All random operations use fixed seeds, so the order of execution does not affect the results.

## Citation

If you use this code, please cite the dataset and PhysioNet as required by the data providers:

- Elias P, Finer J. EchoNext: a dataset for detecting echocardiogram-confirmed structural heart disease from ECGs (version 1.0.0). PhysioNet. 2025. doi:10.13026/r9pp-3y42
- Hughes JW, Jing L, Finer J, et al. EchoNext-Mini: a dataset and baseline AI model for detecting structural heart disease from electrocardiograms. NEJM AI. 2026. doi:10.1056/aidbp2500516
- Pollard T, Moody BE, Lehman L, et al. PhysioNet as a global platform for biomedical research. Nat Health. 2026. doi:10.1038/s44360-026-00096-z

The citation of the article will be added after publication.

## License

The code is released under the MIT License (see `LICENSE`). The EchoNext data are **not** covered by this license and remain subject to the PhysioNet Restricted Health Data License 1.5.0.
