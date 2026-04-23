# PaySim Fraud Detection Case Study — Short README

## Objective
Build a transaction-level suspicious-activity scoring workflow for the PaySim dataset under severe class imbalance, while keeping false positives operationally manageable.

## Approach
The modeling workflow emphasized leakage control, time-aware evaluation, and operational thresholding.

The final feature set was built from:
- transaction type
- log-transformed transaction amount
- timeline position (`step`)
- hour-of-day
- destination / origin novelty indicators
- prior entity transaction-count features

To reduce leakage risk, the primary model excluded:
- `oldbalanceOrg`
- `newbalanceOrig`
- `oldbalanceDest`
- `newbalanceDest`

Raw identifiers (`nameOrig`, `nameDest`) were also not used directly. Instead, they were used only to derive prior-history and first-seen features.

A time-based split was used:
- **train** for model fitting / tuning
- **validation** for model selection and threshold selection
- **holdout** for final one-time reporting

## Models
### Baseline
The baseline was a **logistic regression** using:
- `step`
- `type`
- `log_amount`

Candidate feature blocks were then added in a structured way. The best logistic model used:
- `step`
- `type`
- `log_amount`
- `hour_of_day`
- `orig_is_first_seen`
- `dest_is_first_seen`
- `log_orig_prior_tx_count`
- `log_dest_prior_tx_count`

The logistic threshold was selected on validation and then locked before holdout testing.

### Advanced model
The advanced model was **XGBoost**. Tuning used:
1. random search with rolling time-based CV inside the training set,
2. stability filtering using fold variability and train-validation gap,
3. a narrowed grid search around the strongest stable candidates,
4. validation-based comparison of the top candidates,
5. validation-based threshold selection.

The final XGBoost candidate used:
- `max_depth = 4`
- `learning_rate = 0.05`
- `n_estimators = 400`
- `subsample = 1.0`
- `colsample_bytree = 0.7`
- `colsample_bylevel = 0.8`
- `gamma = 0.5`

The selected threshold was **0.93**.

## Key trade-offs
The main trade-off was **recall vs operational burden**. Lower thresholds increased recall but created many more alerts and false positives. Thresholds were therefore selected using validation-based operational metrics rather than a default 0.5 cutoff.

I tracked:
- PR AUC as the primary ranking metric
- ROC AUC as a secondary ranking metric
- expected cost
- alert volume
- incremental alerts against the negative class
- false omission rate

The cost function used:
- **FN = 500**
- **FP = 5**

## Final holdout results
### Best Logistic Regression
- ROC AUC: **0.9510**
- PR AUC: **0.4532**
- Precision: **0.0508**
- Recall: **0.9256**
- F2: **0.2082**
- Alert volume: **33,803**
- Expected cost: **229,435**

### Final XGBoost
- ROC AUC: **0.9726**
- PR AUC: **0.6406**
- Precision: **0.2608**
- Recall: **0.8339**
- F2: **0.5792**
- Alert volume: **5,929**
- Expected cost: **175,915**

Overall, XGBoost materially outperformed the logistic benchmark on holdout. It achieved much stronger ranking quality, substantially higher precision, far lower alert volume, and lower expected cost.

## Model interpretation and practical findings
SHAP analysis showed that the final XGBoost model was driven primarily by:
- transaction type
- transaction amount
- destination novelty
- destination-side prior history
- temporal structure (`step` and `hour_of_day`)

The top 20 highest-risk holdout transactions were all true frauds, with precision **1.00** in that top-risk slice. These cases were dominated by large `CASH_OUT` transactions, often with sparse prior history and first-seen destination patterns.

Segment analysis showed the model was strongest in the most relevant groups:
- strongest type-level performance in `TRANSFER` / `CASH_OUT`
- better performance when `dest_is_first_seen = 1`
- strongest performance in the highest transaction-amount band

## Conclusion
The final workflow combined:
- leakage-aware feature engineering
- time-based validation
- interpretable benchmarking
- stability-aware XGBoost tuning
- business-oriented threshold selection

The final XGBoost model provided the best balance of ranking quality and operational usability on the holdout set.
