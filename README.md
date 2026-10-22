# UmojaHack Africa 2022

Samuel Mensah ([@simiel](https://github.com/simiel)) — competition notebooks for two Zindi challenges at UmojaHack Africa 2022.

This repository is the working record of those challenges. The main project predicts the cost of short-term insurance claims for Zimnat, so the insurer can forecast claim spend instead of only reacting to it. A second folder takes on AirQo’s faulty air-quality sensor problem: flag readings that come from a device that has failed, so those sensors can be isolated and fixed. Everything here is analysis, model experiments, and the CSV files written for Zindi.

## Monthly insurance claim prediction

Each training row is one claim. The target, `target`, is the claim amount. Predictors are mostly categorical — product, agent, class of business, risk type, client type, renewal frequency, primary cause, secondary cause, and branch — plus `sum_insured` and `loss_date`. A separate policies table carries cover dates and the same policy attributes.

| File | Rows | Role |
| --- | ---: | --- |
| `data/train.csv` | 9,376 | Claims with `target` |
| `data/test.csv` | 4,019 | Claims to score |
| `data/policies.csv` | 30,371 | Policy transactions |
| `data/SampleSubmission.csv` | 4,019 | `claim_id`, `target` |

Claim amounts in train run from 0 to 7,868,590.62 (mean 41,464.71). 1,252 claims are zero. There are 11 branches. The starter notebook’s own EDA notes that June has the most claims and December the fewest, that 2018 is the heaviest year and 2021 the lightest, and that branches 1 and 2 dominate both volume and `sum_insured` outliers.

The metric used throughout is mean absolute error. Lower is better.

### Starter

`Zimnat_starternotebook.ipynb` is the official starter, with CSV paths pointed at `data/`. `original/` keeps that notebook and the data files as they shipped. The starter fills missing categoricals with the mode, label-encodes them, adds day, month, and year from `loss_date`, and fits a linear regression. On its 30% holdout the saved output is MAE **58,703.30**. Submitted unchanged, that starter scores **75,003.60** on the Zindi test set.

### Experiments

Work after the starter lives in `notebook/`.

- `1.ipynb` one-hot encodes every categorical column on the concatenated train and test frames.
- `investigate.ipynb` plots claim amount against sum insured and left-joins policies onto claims (policy number, product, agent, class of business, client type, renewal frequency) to inspect policy history.
- `2.ipynb` one-hot encodes categoricals, adds loss-date parts, target-encodes `product`, trains `XGBRegressor` and `LGBMRegressor` (500 trees), writes a 50/50 blend, and fits a linear model on the two boosters’ predictions (`META.csv`).
- `final.ipynb` is the latest training notebook. It one-hot encodes the categoricals, adds day, day of week, week, month, and year from `loss_date`, target-encodes `product` and `primary_cause`, and trains:
  - `XGBRegressor` — learning rate 0.3, up to 10,000 trees, early stopping on MAE, 25% holdout
  - `CatBoostRegressor` — learning rate 0.3, up to 9,000 trees, early stopping on MAE
  - an equal-weight blend of the two, saved as `notebook/FINALBLENDxxxx4.csv`

The submission filenames are the search history around that setup. Committed notebooks train, at various points, linear regression, ridge, random forest, k-nearest neighbors, SVR, XGBoost, LightGBM, and CatBoost, crossed with one-hot, count, and target encoding. XGBoost files also vary learning rate, max depth, gamma, subsample, the `gblinear` booster, objective, and early-stopping length. Blends are saved alongside the single models.

Policy history is explored in `investigate.ipynb`. The policy-count merge in `notebook/2.ipynb` is commented out, and `final.ipynb` trains on the claim columns only.

### Scores written down in the repo

These are Zindi test MAE figures recorded in `Book1(AutoRecovered).xlsx` or in a commit message. They are the scores that were logged, not a full leaderboard.

| Experiment | Test MAE | Where it is recorded |
| --- | ---: | --- |
| Starter notebook (linear regression) | 75,003.60 | `Book1(AutoRecovered).xlsx` |
| Starter with ridge | 75,002 | same spreadsheet |
| Starter with lasso | 75,003.66 | same spreadsheet |
| One-hot categoricals + linear regression | 64,415.17 | spreadsheet and commit `652f175` |
| One-hot LightGBM | 48,112.50 | commit `2f44b61` |

The XGBoost, CatBoost, and blend submissions that came after the LightGBM run are in `notebook/`. Their Zindi scores were not written back into the spreadsheet.

`notebook/catboost_info/catboost_training.json` is the training log for the CatBoost run in `final.ipynb`. On that log’s evaluation split, MAE reached **45,763.87** at iteration 211. The log continues through iteration 1,011. That number is local evaluation MAE from the training log, separate from the Zindi test scores above.

## Faulty air-quality sensors (AirQo)

`umojahack-africa-2022-beginner-challenge/` is the UmojaHack Africa 2022 beginner challenge. AirQo’s low-cost sensors, deployed across Uganda, fail from calibration drift, wear, and harsh sites, and bad readings flow into policy decisions. The task is binary classification of `Offset_fault` for any device type.

| File | Rows |
| --- | ---: |
| `train.csv` | 297,177 |
| `test.csv` | 127,361 |

Columns are `Sensor1_PM2.5`, `Sensor2_PM2.5`, `Temperature`, `Relative_Humidity`, and `Datetime`. The starter summary places the training window from 15 October 2021 to 21 January 2022, with a 0.96 correlation between the two PM2.5 sensors and a −0.99 correlation between temperature and humidity. Labels are imbalanced: 186,903 non-fault and 110,274 fault.

`Airqo_StarterNotebook.ipynb` and `One.ipynb` are the same starter. It zero-fills missing values, adds day, month, year, and hour, and fits `RandomForestClassifier(max_depth=3)`. The saved holdout report (30% split) shows accuracy **0.662**, with fault-class recall **0.11** — the baseline predicts the non-fault class almost everywhere. `Baseline.csv` is that submission: 7,488 of 127,361 test rows marked as faults.

`Signoff.csv`, `Signoff2.csv`, and `Signoff3.csv` are later binary submissions from this folder. They mark 33,813, 35,787, and 37,802 test rows as faults. `catboost_info/` is a CatBoost log for a 3,000-iteration logloss run that finishes at a training logloss of 0.118. The cells that produced the sign-off files and that CatBoost run are not in the checked-in notebooks; those notebooks are the starter.

## Layout

```
data/                          Zimnat train, test, policies, data dictionary, sample submission
original/                      starter notebook and Zimnat files as shipped
Zimnat_starternotebook.ipynb   starter, reading from data/
notebook/                      experiment notebooks, submission CSVs, CatBoost logs
umojahack-africa-2022-beginner-challenge/
                               AirQo data, starter, baseline and sign-off files
Book1(AutoRecovered).xlsx      early Zindi test-score log for the claim models
```

## Tech stack

Python, in Jupyter notebooks. The modeling notebooks import:

- pandas, NumPy, matplotlib, seaborn
- scikit-learn — linear regression, ridge, random forest, k-nearest neighbors, holdout splits, and MAE. An earlier committed revision of `notebook/2.ipynb` also trains `SVR` (`SVR1.csv`)
- XGBoost, LightGBM, CatBoost
- `category_encoders` — `CountEncoder` and `TargetEncoder`

There is no `requirements.txt` and no pinned versions.

## Getting started

No environment variables are used. Notebooks read local CSVs.

1. Use a Python environment with Jupyter and the libraries above.
2. Claim starter: open `Zimnat_starternotebook.ipynb` from the repository root. It reads `data/train.csv`, `data/policies.csv`, `data/test.csv`, and `data/SampleSubmission.csv`.
3. Claim models: open `notebook/final.ipynb` (or `notebook/2.ipynb`). They read the same files through `../data/` and write submission CSVs into `notebook/`.
4. Sensor challenge: from `umojahack-africa-2022-beginner-challenge/`, open `Airqo_StarterNotebook.ipynb`. It reads `train.csv`, `test.csv`, and `SampleSubmission.csv` in that directory and writes `Baseline.csv`.

There is no build script and no test suite.

## Project status

Completed competition entry. Commits run from 19 March 2022 through 29 April 2022. The tree is the hackathon workspace — notebooks, submission files, and training logs — kept as it stood at the end of the event.

## What this demonstrates

- Moving a Zindi tabular problem past the starter. Label encoding plus linear regression (test MAE 75,003.60) is replaced by a real encoding comparison. One-hot encoding with linear regression reaches a recorded test MAE of 64,415.17, and one-hot LightGBM reaches 48,112.50.
- A broad, kept-on-disk model search for a heavy-tailed claim-severity target: linear models, ridge, random forest, k-nearest neighbors, SVR, and three boosting libraries, with XGBoost hyperparameter variants saved as their own submission files.
- Training against the competition metric. `final.ipynb` fits XGBoost and CatBoost with MAE and early stopping on a holdout, then blends them. `notebook/2.ipynb` also stacks a linear model on the XGBoost and LightGBM predictions.
- Using the second table. Policy transactions are joined and profiled in `investigate.ipynb` before the final model settles on claim-level features.
- A second problem type in the same event: imbalanced sensor-fault classification, with a baseline whose holdout report shows the failure mode (fault recall 0.11) and later submissions that flag several times as many faults.
