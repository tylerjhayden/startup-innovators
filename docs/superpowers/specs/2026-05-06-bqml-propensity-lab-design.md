# BQML B2B Propensity Lab — Design Spec

**Date:** 2026-05-06
**Author:** Chief & Minerva
**Status:** Approved, ready for implementation plan

## Overview

A self-built, hands-on parallel of Google's *Predict Visitor Purchases with BigQuery ML* lab. Instead of predicting whether a Google Merchandise Store visitor purchases, we train a logistic regression model to predict whether a B2B account will become a paying customer, using Chief's real account list enriched with synthetic firmographic and intent signals.

**Primary purpose:** Learn BigQuery ML end-to-end (data load → model creation → evaluation → prediction) with both Console and CLI workflows. Secondary purpose: produce a transferable propensity scoring template Chief can later apply to live prospecting work.

## Context

- **GCP project:** `startup-innovators` (created 2026-05-06, billed to Tyler Hayden Consulting)
- **APIs enabled:** `bigquery`, `bigquerystorage`, `aiplatform`
- **Default account:** `tylerjhayden@gmail.com`
- **Source data:** `/Users/tylerhayden/Unbound/Unbound Sales/Prospecting/upload lists/Dream Company Master List Mar 9.csv` (162 rows, 10 columns)
- **Reference lab:** Google Cloud Skills Boost — *Predict Visitor Purchases with BigQuery ML*

## Goals

1. Hands-on practice with the four canonical BQML workflow stages: explore → train → evaluate → predict
2. Console + CLI parity — every step executed both ways at least once
3. Reusable artifacts: enriched dataset, BQ tables/view/model, SQL files
4. Pedagogical engagement: Chief types every script, query, and CLI command himself

## Non-goals

- Production-grade modeling (no cross-validation, no hyperparameter tuning beyond stretch goals)
- Real-world feature engineering (synthetic signals are deliberately simple; weak signals are noisy by design)
- Operationalizing predictions back into Chief's CRM or sales workflow (separate project)

## Architecture

### Local layout

```
startup-innovators/
├── data/
│   ├── source/dream_company_master.csv     # Untouched copy of source
│   ├── enriched/accounts_enriched.csv      # Source + 10 synthetic columns
│   └── splits/
│       ├── train.csv                       # ~130 rows (~80%)
│       └── predict.csv                     # ~32 rows (~20%)
├── scripts/
│   ├── enrich_accounts.py                  # Adds synthetic columns
│   └── split_data.py                       # Train/predict split
├── sql/
│   ├── 01_explore.sql
│   ├── 02_create_model.sql
│   ├── 03_evaluate.sql
│   ├── 04_predict_industry.sql
│   └── 05_predict_per_account.sql
├── docs/superpowers/specs/                 # This spec
└── README.md                               # Lab manual (exercises + hints)
```

### BigQuery layout

| Object | Type | Source |
|---|---|---|
| `propensity_lab` | Dataset | New, US multi-region |
| `propensity_lab.accounts_train` | Table | `train.csv` upload |
| `propensity_lab.accounts_predict` | Table | `predict.csv` upload |
| `propensity_lab.training_data` | View | Selects label + features from `accounts_train` |
| `propensity_lab.propensity_model` | BQML model | Logistic regression |

## Data pipeline

### Source schema (10 columns, as-is)
`Company Name`, `Employees`, `Website`, `Domain`, `Company Street`, `Company City`, `Company State`, `Company Country`, `Annual Revenue`, `Source`

Dirty data to handle: `Annual Revenue` has currency formatting (`"$1,499,999,000"`), `Employees` has thousands separators (`"4,900"`), missing values shown as `"--"` or empty.

### Synthetic columns (10 added)

Note: `current_arr_usd` is *what this account pays us* (recurring revenue), distinct from the source `Annual Revenue` field which is the *company's overall revenue*.

| Column | Type | Generation rule |
|---|---|---|
| `current_arr_usd` | NUMERIC | $0 if not customer; else log-normal sample, $25k–$500k range |
| `industry` | STRING | Uniform sample from `{SaaS, Retail, Financial Services, Manufacturing, Healthcare, Media, Other}` |
| `engaged_l90d` | BOOL | 75% true if customer, 25% if prospect |
| `meetings_l180d` | INT | Poisson λ=4 if customer, λ=0.5 if prospect |
| `new_exec_l90d` | BOOL | 40% if customer, 15% if prospect |
| `news_initiative_signal` | BOOL | 40% if customer, 15% if prospect |
| `competitor_in_job_posts` | BOOL | 25% for both (noise feature) |
| `funding_or_expansion_l180d` | BOOL | 40% if customer, 15% if prospect |
| `tech_stack_match` | BOOL | 25% for both (noise feature) |
| `prior_customer` | BOOL | 50% if customer, 5% if prospect |

### Generator logic

1. Read source CSV, clean numeric fields (`$`/commas stripped, `--` → empty)
2. Assign `is_customer` per row using a base rate of ~30%, with a slight bias toward larger employee counts (gives firmographics a real-world-ish signal)
3. Sample `current_arr_usd` log-normally for customers, else 0
4. For each row, sample synthetic features per the rule table above
5. Flip ~5% of `is_customer` labels (irreducible error injection)
6. Re-derive `current_arr_usd` after flip — flipped customers get a sample, flipped prospects get $0
7. Write `data/enriched/accounts_enriched.csv`

**Determinism:** `random.seed(42)` at top of script. Re-running produces identical output.

**Dependencies:** Python stdlib only (`csv`, `random`, `math`). No pandas.

### Split logic

`split_data.py` reads enriched CSV, seeded shuffle, takes first 80% → train, rest → predict. Both files retain all columns including `current_arr_usd`.

## Lab tasks

Each task ships in two flavors: Console click-path and CLI command. README walks through both.

### Task 0 — Load data into BQ
*New task; the Google lab skips this since its data is pre-loaded.*

- **Console:** BigQuery → create dataset `propensity_lab` → create table → upload `train.csv` → autodetect schema. Repeat for `predict.csv`.
- **CLI:** `bq mk --dataset --location=US propensity_lab` then `bq load --autodetect --source_format=CSV propensity_lab.accounts_train data/splits/train.csv`.

### Task 1 — Explore data + save training view
*Mirrors Google lab Task 1.*

- Run `SELECT IF(current_arr_usd > 0, 1, 0) AS label, industry, employees, …` against `accounts_train`
- Save query result as view `propensity_lab.training_data`
- **Console:** SQL editor → Run → Save → Save view
- **CLI:** `bq mk --use_legacy_sql=false --view='…' propensity_lab.training_data`

### Task 2 — Create model
*Mirrors Google lab Task 2.*

```sql
CREATE MODEL propensity_lab.propensity_model
OPTIONS(model_type='LOGISTIC_REG', input_label_cols=['label']) AS
SELECT * FROM propensity_lab.training_data;
```

- **Console:** paste, Run. ~10–30s for 130 rows.
- **CLI:** `bq query --use_legacy_sql=false < sql/02_create_model.sql`
- **Bonus:** inspect model in Console (Details / Training / Evaluation / Schema tabs); CLI equivalent `bq show --model propensity_lab.propensity_model`

### Task 3 — Evaluate model
*Mirrors Google lab Task 3.*

```sql
SELECT * FROM ML.EVALUATE(
  MODEL propensity_lab.propensity_model,
  TABLE propensity_lab.training_data
);
```

Returns `precision`, `recall`, `accuracy`, `f1_score`, `log_loss`, `roc_auc`.
**Expected:** ROC-AUC in 0.75–0.85 range. Outside that range = generator or model issue.

### Task 4 — Predict per industry
*Mirrors Google lab Task 4 (per country); we substitute industry since most accounts are US.*

```sql
SELECT industry, SUM(predicted_label) AS predicted_buyers
FROM ML.PREDICT(
  MODEL propensity_lab.propensity_model,
  TABLE propensity_lab.accounts_predict
)
GROUP BY industry
ORDER BY predicted_buyers DESC
LIMIT 10;
```

### Challenge — Top-10 accounts by predicted probability
*Mirrors Google lab challenge.*

```sql
SELECT
  company_name,
  industry,
  predicted_label,
  predicted_label_probs[OFFSET(0)].prob AS p_buy
FROM ML.PREDICT(
  MODEL propensity_lab.propensity_model,
  TABLE propensity_lab.accounts_predict
)
ORDER BY p_buy DESC
LIMIT 10;
```

This produces a prospecting hot list — model's ranked guess at who to call first.

## Workflow model

### What Minerva writes
- This spec
- `README.md` (lab manual: exercises, hints, verification commands)

### What Chief writes (with Minerva's coaching)
- All Python in `scripts/`
- All SQL in `sql/`
- All CLI commands he runs

### Coaching modes Chief can request per task
- **Spec mode** — Minerva describes behavior, Chief derives code. Highest difficulty.
- **Show me mode** — Minerva shows exact code in chat, Chief types it into file. Medium. *(Default for utilities and boilerplate.)*
- **Walk me through mode** — Minerva gives one block at a time with narration, Chief types and asks. Highest learning. *(Default for synthetic generator and BQML statements.)*
- **Review my work** — Chief writes, Minerva reviews/debugs/explains.

### Hard constraint
Minerva does NOT use Write or Edit to drop code into `scripts/` or `sql/`. Code lives in chat until Chief types it into a file. (Stored as memory: `feedback_typing_to_learn.md`.)

## Stretch goals (post main path)

1. Train `BOOSTED_TREE_CLASSIFIER` and compare ROC-AUC to logistic regression
2. Use `ML.FEATURE_IMPORTANCE` to inspect which signals carry weight
3. Export predictions to CSV via `bq extract`
4. Cross-validation via `ML.EVALUATE` with held-out split
5. Hyperparameter tuning with `ML.HYPERPARAMETER_TRIALS`

## Success criteria

The lab is complete when:

1. `data/enriched/accounts_enriched.csv` exists with 162 rows + 20 columns (10 source + 10 synthetic)
2. `data/splits/train.csv` (~130 rows) and `data/splits/predict.csv` (~32 rows) exist
3. BQ tables `propensity_lab.accounts_train` and `propensity_lab.accounts_predict` loaded
4. View `propensity_lab.training_data` created
5. Model `propensity_lab.propensity_model` trained
6. `ML.EVALUATE` returns ROC-AUC in 0.75–0.85 range
7. `ML.PREDICT` produces both per-industry summary and per-account top-10
8. Every task executed at least once via Console AND once via CLI

## Risks & open questions

- **Small dataset (162 rows):** logistic regression should fit fine; boosted trees risk overfitting. Note in stretch goals.
- **No formal test/holdout discipline:** the 80/20 split is for the prediction exercise, not statistical validation. Real models would cross-validate.
- **Synthetic industry is uncorrelated with label:** per-industry prediction grouping is a workflow exercise, not a real insight.
- **Source CSV has dirty data:** the cleaning step is a learning exercise, but mistakes there will silently degrade the model. README will include verification commands to catch issues early.
- **Git not initialized:** project is not a git repo. Decision deferred — Chief to confirm whether to `git init` before implementation begins.
