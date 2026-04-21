# Suspicious Activity Detection in P2P Transactions

## Overview

This case study develops a transaction-level suspicious-activity scoring workflow using the PaySim dataset. The objective is to identify fraudulent transactions under severe class imbalance while keeping false positives operationally manageable.

The analysis is organized into three stages:

1. **Exploratory data analysis and feature construction**
2. **Baseline modeling and incremental improvement**
3. **Advanced modeling and tuning**

The workflow emphasizes:
- leakage-aware feature design
- time-based evaluation
- interpretable baseline modeling
- behavioral feature engineering
- cost-aware threshold selection

---

## Problem Setting

Fraud detection in this dataset is a rare-event problem. The target variable, `isFraud`, represents only a very small fraction of all transactions, so overall accuracy is not an appropriate primary evaluation metric. Instead, the analysis focuses on:

- **PR AUC** as the main ranking metric
- **ROC AUC** as a secondary ranking metric
- **expected cost** for threshold selection
- **alert volume / alert rate** for operational interpretability

The case also assumes asymmetric business cost:

- **False negative cost = 500**
- **False positive cost = 5**

This makes recall-sensitive modeling and thresholding especially important.

---

## Dataset and Initial Findings

The dataset contains transaction-level records with:
- time index: `step`
- transaction attributes: `type`, `amount`
- entity identifiers: `nameOrig`, `nameDest`
- balance variables
- fraud-related fields: `isFraud`, `isFlaggedFraud`

Several key findings emerged during EDA:

- Fraud is concentrated entirely in `TRANSFER` and `CASH_OUT`.
- `isFlaggedFraud` is a very narrow auxiliary flag and was excluded from modeling.
- Raw transaction amount is highly right-skewed, so `log_amount` was used instead.
- Time structure is important, and the final evaluation was performed using a **forward-looking holdout period**.
- Raw sender and receiver identifiers were excluded from direct modeling because of memorization and leakage risk.

---

## Modeling Design

### Time-based split
The final 7 days of the dataset were reserved as the **holdout set**. This creates a realistic future-period evaluation design:

- earlier transactions used for training
- later transactions used for final testing

For the advanced model, the training portion was further split into:
- **development-train**
- **validation**
- **holdout**

This allowed:
- rolling time-based cross-validation for tuning
- validation-based final model selection
- untouched holdout reporting

---

## Feature Engineering

A broader candidate feature pool was constructed during exploratory analysis, including:

- `step`
- `type`
- `amount`
- `log_amount`
- `hour_of_day`
- sender/receiver novelty indicators
- prior sender/receiver activity counts
- prior rolling 24-hour activity counts
- additional cumulative and rolling amount features

After screening for interpretability, redundancy, and practical value, a smaller subset was carried forward into modeling.

### Selected behavioral features
The final modeling workflow focused on:
- `hour_of_day`
- `orig_is_first_seen`
- `dest_is_first_seen`
- `log_orig_prior_tx_count`
- `log_dest_prior_tx_count`
- `log_orig_prior_24h_tx_count`
- `log_dest_prior_24h_tx_count`

These features were chosen because they provided a good balance of:
- behavioral relevance
- interpretability
- manageable redundancy
- computational practicality

---

## Baseline Model

The baseline model was a **logistic regression** using:

- `step`
- `type`
- `log_amount`

### Baseline performance
- **ROC AUC:** 0.8948
- **PR AUC:** 0.1118

This confirmed that even a simple transaction-level feature set contains meaningful predictive signal.

---

## Improved Logistic Regression

Selected feature blocks were evaluated systematically in different combinations:

- temporal refinement: `hour_of_day`
- behavioral novelty: first-seen indicators
- prior activity count
- recent 24-hour activity count

### Best logistic-regression benchmark
The strongest logistic-regression model was:

- `step`
- `type`
- `log_amount`
- `hour_of_day`
- `orig_is_first_seen`
- `dest_is_first_seen`
- `log_orig_prior_tx_count`
- `log_dest_prior_tx_count`

### Best logistic benchmark performance
- **ROC AUC:** 0.9510
- **PR AUC:** 0.4532

### Interpretation
The logistic-regression comparison showed that:

- `hour_of_day` added substantial value beyond the baseline
- novelty indicators added major incremental lift
- prior-count history provided the final improvement to the best logistic model
- recent 24-hour count features did not provide consistent additional value once the strongest blocks were already included

This produced a strong, compact, and interpretable benchmark.

---

## Advanced Model

The advanced model was designed around **XGBoost** to capture:

- non-linear effects
- feature interactions
- threshold-like behavior
- more complex combinations of transaction and behavioral features

### Intended tuning approach
The intended advanced-model workflow was:

1. random search over a broad hyperparameter space  
2. rolling time-based cross-validation on the development-training set  
3. stability screening using fold-level PR AUC dispersion and train-test generalization gap  
4. validation-based final model selection  
5. optional second-stage narrowed grid search around the strongest stable candidates  

This tuning design was chosen to reflect AML-style monitoring requirements, where temporal stability matters in addition to average predictive performance.

### Compute limitation note
Because the notebook was run in a local compute environment, the full advanced-model search procedure was computationally expensive, especially when combined with:

- repeated XGBoost training
- rolling time-based cross-validation
- multiple hyperparameter candidates
- a potentially large second-stage narrowed grid

As a result, the full advanced-model expansion was treated as a compute-limited extension rather than the main completed modeling path. The logistic-regression benchmark and feature-engineering workflow were completed fully, while the XGBoost tuning process was documented as the intended next step.

If additional compute time were available, the next step would be to complete the rolling-CV random search, apply stability filtering, and finalize the XGBoost model through validation-based selection.

---

## Threshold Selection

The logistic-regression benchmark was first evaluated as a **ranking model**, then converted into an alerting model through threshold selection.

Thresholds were reviewed using:
- precision
- recall
- F1
- F2
- alert volume
- alert rate
- expected cost

This made it possible to choose thresholds that are statistically reasonable and operationally defensible.

### Logistic benchmark threshold notes
The expected-cost curve around the best logistic threshold was relatively flat, meaning multiple nearby thresholds were defensible depending on business preference:

- lower thresholds improved recall but increased alert volume
- higher thresholds improved precision and reduced review burden

This reflects a realistic monitoring trade-off rather than a single purely mechanical threshold choice.

---

## Main Conclusions

This case study shows that:

1. A simple logistic-regression baseline already captures meaningful fraud signal.
2. A small number of carefully engineered behavioral features can produce substantial lift.
3. Time-aware evaluation is essential for realistic fraud-monitoring assessment.
4. Threshold selection should be tied to business cost and alert burden, not a default 0.5 cutoff.
5. A stronger non-linear model was designed with a production-relevant tuning framework, even though the full search was not completed because of compute limitations.

---

## Future Work

If additional time and compute were available, the next stage of work would include:

- completing the XGBoost rolling-CV random search and validation-based final selection
- richer cumulative amount features
- multi-window behavioral features
- ratio features comparing recent activity to longer-run history
- deeper threshold-stability analysis across periods
- a second-stage narrowed grid search for the advanced model

---

## Deliverables

The final workflow produces:
- an interpretable logistic-regression benchmark
- a documented advanced-model tuning framework
- threshold comparisons
- a scored holdout output file containing:
  - `transaction_id`
  - `label`
  - `score`
  - `threshold_alert`

---

## Summary

Overall, this notebook demonstrates that a fraud-detection workflow can be both **practically useful** and **methodologically disciplined** by combining:

- leakage-aware feature construction
- time-based validation
- interpretable baseline modeling
- behaviorally meaningful feature engineering
- cost-aware operational decisioning

The completed logistic-regression benchmark provides a strong final reference model, while the advanced XGBoost workflow is clearly defined as the next compute-dependent extension.
