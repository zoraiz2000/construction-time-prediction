# Construction cycle-time prediction

Predicts how many calendar days a home will take to build (`construction_end_date` minus `construction_start_date`), using only information that would be known when construction starts.

## Python version

This project was run with **Python 3.9.9**.

## Installation

From the project root:

```bash
pip install -r requirements.txt
```

Then open JupyterLab:

```bash
jupyter lab
```

## Required input files

Place the raw extracts here (do not overwrite them):

- `data/build_history.csv`
- `data/homes_to_predict.csv`

Cleaning writes processed copies under `data/processed/`. Modeling and final scoring read:

- `data/processed/build_history_train.csv`
- `data/processed/homes_to_predict_cleaned.csv`

## Notebook order

Run these from the project root, top to bottom, in this order:

1. `01_data_audit.ipynb` — inspect the raw files; no cleaning
2. `02_data_cleaning.ipynb` — write cleaned files under `data/processed/`
3. `03_modeling.ipynb` — train/validation, baselines, and model selection
4. `04_final_predictions.ipynb` — fit the selected model and score new homes

## Predictions

`04_final_predictions.ipynb` writes **`predictions.csv` in the project root** (not under `data/`). It has 578 rows with `home_id` and `predicted_cycle_days`.

## Final model

**CatBoost core**, without `start_year` and without `selected_options_value`.

January–June 2024 validation MAE: **33.20 days** (floor-plan baseline 36.81; global baseline 50.73).

## AI logs

Drop assignment or tool logs in `ai_logs/` if needed. That folder is otherwise empty.
