# Early Warning of High-Demand Hours in a Hospital Emergency Department

DATA 245 (Machine Learning) term project, San José State University, Fall 2026.

We predict, **one hour ahead**, whether an emergency department (ED) will have an unusually busy hour (more than 8 arrivals, the 90th percentile). The project compares rule baselines, an ARIMA model and five machine-learning classifiers on four years (2014–2017) of public hourly arrival data, using a strict time-ordered evaluation.

## Key results (test year 2017)

| Method | PR-AUC [95% CI] |
|---|---|
| ARIMA (Fourier terms) | 0.236 [0.217, 0.259] |
| Logistic Regression | 0.235 [0.217, 0.256] |
| Linear SVM | 0.233 [0.216, 0.253] |
| Hour-of-week baseline | 0.218 [0.202, 0.238] |
| RBF SVM / KNN | 0.215 |
| XGBoost | 0.208 [0.192, 0.227] |
| Previous-week / last-hour rules | 0.181 |

No-skill PR-AUC (the test positive rate) is 0.116.

The best models are about twice as good as chance, but only slightly better than a calendar-only baseline.

## Repository structure

```
.
├── Threshold_Rule_XGBoost.ipynb      # Main notebook: data reconstruction (v2), baselines and models, evaluation
├── Demo_next_hour_alert.ipynb        # Live demo: next-hour high-demand alert
└── data/
    └── ed_hourly_clean.csv           # Cleaned hourly series: 35,040 rows × 29 columns, incl. train/validation/test split
```

## Method

### 1. Data reconstruction
The source file is an Excel pivot table. Blank hourly cells are zero-arrival hours: on 100% of days, the hourly values add up to the recorded daily total. The 12 fully blank days and 32 absent calendar dates are kept as missing on a complete hourly grid.

### 2. Target and split
An hour is high demand if it has more than 8 arrivals (the 90th percentile of 2014–2016).

| Split | Years | Positive rate | Used for |
|---|---|---|---|
| Train | 2014–2015 | 4.7% | Fitting and hyperparameter tuning |
| Validation | 2016 | 7.4% | Choosing alert cutoffs |
| Test | 2017 | 11.6% | Final evaluation, used once |

### 3. Features
- **Calendar:** hour, weekday, month, weekend, US holidays, plus sine/cosine encodings
- **Lags:** 1, 2, 3, 24 and 168 hours
- **Rolling statistics** over past hours
- **Same-hour 7-day mean**

All features use only information available before the hour being predicted.

### 4. Baselines and models
- **Rules:** previous-week and last-hour
- **Hour-of-week frequency:** the calendar-only baseline
- **ARIMA(3,0,0)** with daily, weekly and yearly Fourier terms, converted into P(arrivals > 8)
- **Classifiers:** Logistic Regression, KNN, linear and RBF SVM, XGBoost

Hyperparameters were tuned with a 4-fold `TimeSeriesSplit` inside the training years, scored by PR-AUC.

### 5. Model improvement
- **Class imbalance**: we compared no reweighting, class weights, SMOTE and random undersampling. Resampling was applied to the training data only. None of them improved ranking quality, and class weights made the predicted probabilities poorly calibrated.
- **Demand drift**: retraining XGBoost on 2014–2016 raised its test PR-AUC from 0.208 to 0.229. Logistic Regression was unchanged.
- **Feature groups**: XGBoost with calendar features only reached 0.240. Adding the lag features lowered its score.
- **Alert cutoff policies**: cutoffs fixed on 2016 raised too many alerts in the busier year 2017. A rolling 28-day quantile cutoff kept alerts at the planned 6 per day and caught about half of all high-demand hours.

### 6. Evaluation
- **Primary metric:** PR-AUC
- **Other metrics:** ROC-AUC, recall, precision, specificity, F1, F2, balanced accuracy, MCC, Cohen's κ, accuracy, Brier score and log loss
- **Uncertainty:** day-level block bootstrap confidence intervals
- **Workload:** models are also compared at an equal number of alerts per day

## How to run

1. Open `Threshold_Rule_XGBoost.ipynb` in Google Colab.
2. Upload the original data file `Supplementary file 1.xlsx` from Dryad (link below) to `/content/`.
3. Run all cells in order. The notebook rebuilds `ed_hourly_clean.csv`, trains all models and produces the evaluation.

## Demo: next-hour alert

`Demo_next_hour_alert.ipynb` shows how the system would be used in practice: for a given hour, it estimates the probability of more than 8 arrivals, decides whether to raise an alert, and compares the decision with what actually happened.

- **Input:** `data/ed_hourly_clean.csv` only. The raw Excel file is not needed.
- **Model:** Logistic Regression without class weights, so its probabilities are calibrated (test PR-AUC 0.233). The alert cutoff (0.08) is chosen on the 2016 validation year. The saved model `ed_alert_model.joblib` is about 6 KB.
- **Run in Colab:**
  1. Open the notebook and run Cell 0, which clones this repository.
  2. Run Cell 1 to train the model and save it.
  3. Run Cell 2 to make a prediction for a single hour.
  4. Optionally, run Cell 3 to see a full day.
- **Run locally:** start from the repository root and skip Cell 0.
- **Change the hour:** edit the timestamp, e.g. `next_hour_alert("2017-03-13 17:00")`. Any hour from 2014-01-08 to 2017-12-30 works, except the few days with missing data.

Example output:
```
Hour starting   : Mon 2017-03-13 17:00
Inputs (past)   : last hour = 5, same hour last week = 12, 7-day same-hour mean = 5.4
P(> 8 arrivals): 0.15   (alert cutoff 0.08)
Decision        : ALERT - expect high demand
What happened   : 14 arrivals -> high demand
```

## Data source

Choudhury, A., & Urena, E. Hourly emergency department arrival data. Dryad Digital Repository. https://doi.org/10.5061/dryad.q57d4g4

Related paper: Choudhury, A., & Urena, E. (2020). Forecasting hourly emergency department arrival using time series analysis. *British Journal of Healthcare Management, 26*(1), 34–43.
