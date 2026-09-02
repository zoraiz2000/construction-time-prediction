# Construction cycle-time prediction

## 1. Approach and final model

The target is construction cycle time in calendar days (`construction_end_date` minus `construction_start_date`). Predictions use only information available when construction starts.

Two baselines were evaluated on mature January–June 2024 validation: a global training-mean cycle time, and a floor-plan average by `plan_code`. CatBoost was chosen because the data are relatively small, tabular, and mix numeric and categorical features.

The selected model is **CatBoost core without `start_year` and without `selected_options_value`**.

| Model | MAE (days) |
|---|---:|
| Global-average baseline | 50.73 |
| Floor-plan-average baseline | 36.81 |
| **CatBoost core (selected)** | **33.20** |
| CatBoost with `start_year` | 35.49 |
| CatBoost plus `selected_options_value` | 31.71 |

Selected RMSE is 39.99 days and R² is about 0.45. Adding `start_year` worsened MAE to 35.49, so it was dropped: it did not improve mature-period validation and can absorb snapshot effects rather than a stable future pattern. `selected_options_value` improved MAE to 31.71 but was dropped because availability on the construction-start date could not be confirmed.

## 2. Data-quality issues and treatments

Raw files were `build_history.csv` (4,684 rows) and `homes_to_predict.csv` (578 rows). Originals were preserved. Cleaning is reproducible and writes only to `data/processed/`.

- **Duplicates.** History had 25 extra identical rows (4,659 unique homes).
- **Community labels.** History used 70 raw spellings of 14 communities; stripped and title-cased.
- **Garage size.** 259 history and 31 scoring rows used Single/Double/Triple; mapped to 1/2/3.
- **Basement.** Literal `"None"` (1,654 history, 260 scoring) means no basement, not missing; recoded to `No basement`.
- **Square footage.** 13 history and 3 scoring homes had `sqft` about 10× the plan typical; all 16 were divided by 10 after a plan-range check.
- **Missing numerics.** Beds (141 / 17) and lot size (97 / 12) were left as missing.
- **Dates and target.** Permit and start dates were consistent. Seven completed homes had end before start; those negative cycles were excluded from training, not rewritten.
- **Completed vs in-progress.** History had 4,105 Complete and 579 In Progress rows. In-progress homes have no observed cycle. Training uses 4,077 completed homes with `cycle_days > 0` (minimum 65 days).

## 3. Train/validation design

The cleaned target file contains only completed homes. Recent completes over-represent fast builds, because slower recent homes could still have been in progress at the July 2025 snapshot.

To reduce that completion-selection / right-censoring bias while keeping a future-like period, I used a maturity-based split:

- **Train:** 2021–2023 starts (2,839 homes).
- **Validate:** January–June 2024 starts (530 homes).
- **Held out of fitting and selection:** starts after 30 June 2024 (708 completes).

After selection, the final CatBoost model was fit on all mature completes through June 2024 (3,369 rows) with 159 trees and no early stopping, then used to generate 578 predictions in `predictions.csv`.

## 4. Excluded fields

`site_manager` remains a model feature.

| Field(s) | Why excluded |
|---|---|
| `home_id`, `lot_number` | Identifiers |
| `plan_name` | Duplicates `plan_code` |
| Raw sale, permit, and start dates | Replaced by start-time duration and calendar features |
| `construction_end_date`, `final_inspection_date` | Future information and target leakage |
| `trade_invoice_total`, `change_order_count` | Accumulate during construction |
| `construction_status` | Post-start; constant in eligible training rows |
| `data_source`, `train_eligible`, `exclude_reason` | Constant or cleaning metadata |
| `start_year` | Worsened validation; can capture snapshot bias |
| `selected_options_value` | Availability at prediction time unconfirmed |

## 5. Principal drivers and operational implications

On the selected model, the strongest predictive drivers were spec-home status (15.1), `sqft` (13.2), bathrooms (9.6), community (8.5), product line (7.7), basement type (6.8), construction-start month (6.3), and site manager (5.9). Importance is predictive, not causal.

Larger and more complex homes likely need longer scheduling buffers. Floor plan or product line alone is not enough: the plan-average baseline is still 3.6 MAE days worse than CatBoost. Community and season may capture local permitting, trade availability, or weather. Spec-home status may mark different selection or scheduling processes. Manager effects should not be interpreted without adjusting for project mix.

## 6. Site-manager analysis and confidence

Raw averages on mature 2021–June 2024 completes rank A. Boucher first and W. Halvorsen 17th, which ignores project mix.

A separate auxiliary CatBoost model used the same core features and early stopping but **dropped `site_manager`**. It is not the scoring model. That helper was trained on 2021–2023 and overestimated January–June 2024 cycle times by about **23 days** on average, so almost every manager looks “fast” versus the helper. Manager comparisons therefore use residuals **relative to the validation-wide average**, not the raw gap versus 2021–2023 expectations.

W. Halvorsen (44 validation builds) was about **22 days faster than the typical mix-adjusted January–June 2024 home**, not 45 days faster than peers. Versus the helper the residual is −45.3 days because that figure still includes the 23-day cohort overestimate. T. Beaulieu (25 builds) is close, with overlapping intervals. They form the leading group (Halvorsen ranked first in 52.6% of bootstraps). The evidence does not support one definitive winner.

This remains observational. Assignments are not random and may reflect crews, communities, workload, or project difficulty.

## 7. Limitations and additional work

Recent starts are still right-censored in a completed-only file. Selection used one mature temporal window. Some fields’ as-of-start availability is unconfirmed. Hyperparameters were not extensively tuned. Weather, trade availability, supply delays, and workload were unavailable. The manager analysis is associative, not causal.

With more time: model in-progress homes with survival methods; use rolling temporal validation; confirm feature timing with operations; add operational and weather variables; and report prediction intervals rather than only point estimates.

The selected CatBoost model produced a meaningful but moderate improvement over the floor-plan baseline (MAE 33.20 vs 36.81 days) while remaining reproducible and defensible.
