# 📈 Predict LTV/ROAS by Campaign — Notebook Guide

> **Notebook:** `Predict LTV by Campaign.ipynb`  
> **Goal:** Forecast **campaign-level LTV/ROAS** at horizons (D7, D14, D30) using daily media metrics + revenue cohorts.  
> **Audience:** Growth / UA analysts, data scientists.  
> **Why a Notebook?** It’s easier for non-engineers to run cells step‑by‑step, tweak parameters, and visualize results without packaging code or a CLI.

---

## ✨ What this does
- Pulls **media metrics** (cost, impressions, clicks, installs) and **cohort revenue** (daily revenue counts, LTV by day).
- Aggregates to **daily per campaign** and computes **ROAS** targets.
- Splits data by a **temporal cut-off** (e.g., `2025‑06‑15`) for backtest-like evaluation.
- Standardizes features and tries multiple regressors (Linear/Ridge/Lasso/BayesianRidge/Tree/GBM/RF/**XGBoost**).
- Evaluates with **R², MAE, RMSE, MAPE** and visualizes **predicted vs actual by campaign**.
- Trains **staged models**: predict `roas_d7` first → feed predictions as features to predict `roas_d14` and `roas_d30`.

---

## 🗂️ Inputs & Assumptions
- **Data sources**
  - **BigQuery**: campaign-level media metrics.
  - **MongoDB**: cohort revenue table with `ltv_0 … ltv_30` and `revenue_count_day_*` fields.
- **Join keys**: `event_date` (daily) + `campaign`.
- **Targets**: `roas_d7` (primary), then `roas_d14`, `roas_d30` using staged predictions.
- **Base numeric features**: all numeric columns except `event_date`, `campaign`, and target columns.

> You can change the **cut-off date** in the notebook (default is `2025-06-15`) to rebalance train/test windows.

---

## 🔧 Setup

### 1) Python environment
```bash
# Use any recent Python 3.9–3.12
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -U pip

# Core libraries
pip install pandas numpy matplotlib seaborn scikit-learn xgboost python-dotenv pymongo google-cloud-bigquery
pip install jupyterlab  # or notebook
```

### 2) Credentials
- **BigQuery**
  - Create a GCP service account with BigQuery read access.
  - Download the JSON key and set:
    ```bash
    set GOOGLE_APPLICATION_CREDENTIALS=path\to\key.json          # Windows
    export GOOGLE_APPLICATION_CREDENTIALS=path/to/key.json         # Mac/Linux
    ```
- **MongoDB** — create a `.env` file in your project root:
  ```ini
  MONGO_USER=...
  MONGO_PASS=...
  MONGO_HOST=...
  MONGO_DB=...
  MONGO_AUTHDB=admin
  ```

> The notebook uses `from google.cloud import bigquery`, `from pymongo import MongoClient`, and `from dotenv import load_dotenv` to connect securely.

---

## ▶️ How to run
1. Launch Jupyter and open **`Predict LTV by Campaign.ipynb`**:
   ```bash
   jupyter lab
   ```
2. Run cells **top to bottom**:
   - **Data Load & Aggregation**: builds daily campaign metrics and revenue LTV per day (`ltv_0...ltv_30`).
   - **Feature Build**: computes ROAS targets (`roas_d0/d1/d2/d7/d14/d30`) and selects base numeric features.
   - **Temporal Split**: `event_date < cut_off_date` → **train**, `>= cut_off_date` → **test**.
   - **Scaling**: `StandardScaler` on features.
   - **Modeling**: fits a list of models (Linear, Ridge, Lasso, BayesianRidge, DecisionTree, GradientBoosting, RandomForest, **XGBRegressor** with `learning_rate=0.05, max_depth=6, n_estimators=300`).
   - **Metrics**: stores per‑model `R²`, `MAE`, `MSE`, `RMSE`, `MAPE` in a results table.
   - **Staged Predictions**: predict `roas_d7` → reuse as feature to predict `roas_d14`, then include both to predict `roas_d30`.
   - **Visualization**: line charts per campaign with a vertical **cut‑off** marker ✂️.

> Tip: The cut-off chart uses `ax.axvline(cut_off_date, linestyle='--', linewidth=1.2)` and an annotation. Adjust as needed.

---

## 📊 Feature ideas (optional)
If you want to extend the model:
- **Ratios/efficiencies**: CTR, CVR, CPI, cost share by campaign, volatility windows.
- **Lagged features**: rolling means/volatility over 3–14 days.
- **Seasonality**: day‑of‑week, month, holidays.
- **Geo mix**: US vs non‑US splits (the notebook shows an example of `us_users` / `non_us_users`).

---

## 🧪 Evaluation
- **Primary**: `MAE` (minimize absolute error on ROAS).
- **Secondary**: `RMSE`, `MAPE`, and `R²` for stability checks.
- **Backtest**: time‑based split avoids leakage and simulates “train on past, predict future.”
- **Per‑campaign plots**: quickly reveal drift or underfit/overfit by creative/source.

---

## 🧰 Troubleshooting
- **`TypeError: XGBModel.fit() got an unexpected keyword argument ...`**  
  Remove `eval_metric`/`early_stopping_rounds` from `fit()` or upgrade `xgboost`. The notebook uses a simple `fit()` with defaults.
- **Auth errors** (BigQuery/Mongo)  
  Double‑check `GOOGLE_APPLICATION_CREDENTIALS` and `.env` values, network/VPC/firewall, and dataset/table names.
- **All‑zero ROAS**  
  Ensure costs aren’t zero (avoid division by zero) and that joins keep only valid `(event_date, campaign)` pairs.

---

## 🧭 Notebook map
- **Imports & clients**: `pandas`, `numpy`, `matplotlib`, `seaborn`, `sklearn`, `xgboost`, `bigquery`, `pymongo`, `dotenv`
- **Data build**: aggregate **cost**, **impressions**, **clicks**, **installs**; compute **ltv_0…ltv_30** and ROAS by horizon.
- **Split & scale**: `cut_off_date = 2025‑06‑15` (changeable), `StandardScaler`.
- **Models**: Linear/Ridge/Lasso/BayesianRidge, DecisionTree, GradientBoosting, RandomForest, XGBoost.
- **Results**: dataframe of metrics sorted by `MAE` + campaign charts.

---

## 🔒 Data & Privacy
- Don’t commit `.env` or service account keys.  
- Use least‑privilege roles. Mask or aggregate sensitive fields.

---

## 🗺️ Roadmap
- [ ] Add **Optuna** tuning for XGBoost (or LightGBM/CatBoost) to squeeze MAE further.
- [ ] Add **cross‑validation** by **time folds**.
- [ ] Ship a small **`requirements.txt`** and **docker** recipe.
- [ ] Optional: export predictions to a BI table (e.g., BigQuery) for dashboards.


---

## 🙌 Acknowledgements
Thanks to the UA & Data teams for feedback on metrics, feature definitions, and QA.

> _Last updated: 2025-08-21_
