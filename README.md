# PaySim Fraud Detection Case Study

## Approach
The workflow was designed to balance **predictive performance**, **leakage control**, and **operational usability**.

### 1. Data review and leakage controls
I first reviewed class imbalance, fraud by transaction type, time structure, and candidate feature behavior.  
Several columns were intentionally excluded or treated carefully:

- `isFlaggedFraud` was excluded because it behaves like a narrow pre-existing alert rather than a neutral predictor.
- `nameOrig` and `nameDest` were not used directly to avoid memorization / identity leakage.
- Historical behavioral features were constructed only from prior activity.

A **time-based split** was used instead of a random split:
- final **7 days** reserved as holdout
- earlier period used for training / tuning

### 2. Baseline model
A simple **logistic regression** baseline was built using:
- `step`
- `type`
- `log_amount`

This provided an interpretable benchmark and confirmed that even simple transaction-level features contain useful signal.

### 3. Feature engineering and selection
Candidate behavioral features were created and screened, including:
- `hour_of_day`
- first-seen indicators
- prior transaction counts
- recent 24-hour counts

For the logistic benchmark, I selected a compact set that balanced:
- interpretability
- incremental lift
- low redundancy

The strongest logistic model used:
- `step`
- `type`
- `log_amount`
- `hour_of_day`
- `orig_is_first_seen`
- `dest_is_first_seen`
- `log_orig_prior_tx_count`
- `log_dest_prior_tx_count`

### 4. Advanced model
I then trained an **XGBoost** model on the same feature family.

Hyperparameter tuning used a two-stage process:
1. **Random search** with rolling time-based cross-validation
2. **Narrowed grid search** on the most influential parameters (`max_depth`, `learning_rate`, `n_estimators`, `subsample`)

To reduce variance and avoid unstable settings, candidates were filtered using:
- `test_pr_auc_range` = max fold PR AUC − min fold PR AUC
- `relative_gap` = (mean train PR AUC − mean test PR AUC) / mean test PR AUC

This favored candidates that were both strong and temporally stable.

### 5. Threshold selection
Both the logistic model and XGBoost were first evaluated as ranking models, then converted into alerting systems using threshold analysis based on:
- precision / recall
- F1 / F2
- alert volume / alert rate
- expected cost

I did not use a default 0.5 threshold because the business objective is asymmetric:
- false negatives are much more costly than false positives

---

## Key Results
| Model | ROC AUC | PR AUC |
|---|---:|---:|
| Best Logistic Regression | 0.9510 | 0.4532 |
| Final XGBoost | 0.9708 | 0.6352 |

The final XGBoost model delivered a **material improvement**, especially in **PR AUC**, indicating that non-linear interactions among transaction, temporal, and behavioral-history features are important.

---

## Main Trade-offs
- **Interpretability vs performance:** logistic regression was easier to explain, but XGBoost produced much stronger ranking performance.
- **Feature richness vs practicality:** I screened a broader candidate pool, but kept the main benchmark feature set compact to reduce redundancy and keep the notebook readable.
- **Search breadth vs compute:** the advanced-model tuning was narrowed in a structured way so it remained practical on local compute while still covering the most important hyperparameters.

---

## Next Steps
If more time or compute were available, the next extensions would be:
- richer amount-based behavioral features
- additional rolling time windows
- SHAP-based interpretation of the final XGBoost model
- deeper threshold-stability analysis across time slices
