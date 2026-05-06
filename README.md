# BQML B2B Propensity Lab

A self-built parallel of Google Cloud Skills Boost's *Predict Visitor Purchases with BigQuery ML* lab, reframed as a B2B account propensity model. Same workflow — explore, train, evaluate, predict — different domain.

This is the lab manual. The full design lives in [`docs/superpowers/specs/2026-05-06-bqml-propensity-lab-design.md`](docs/superpowers/specs/2026-05-06-bqml-propensity-lab-design.md).

## The lab in one breath

Take a 162-row account list. Add ten synthetic firmographic and intent columns. Split 80/20. Load both halves into BigQuery. Train a logistic regression model on the train half. Evaluate it. Score the predict half. Rank accounts by buy-probability. Run every step once in the Console and once on the CLI.

## How this lab is run

Chief types every Python script, every SQL query, and every CLI command. Minerva coaches in chat using one of four modes per task:

- **Spec mode** — Minerva describes behavior; Chief derives the code. Hardest, highest learning.
- **Show mode** — Minerva pastes exact code in chat; Chief types it into the file. Default for boilerplate.
- **Walk-me-through mode** — Minerva narrates one block at a time. Default for novel constructs (BQML SQL, label-conditioned sampling).
- **Review mode** — Chief writes; Minerva reviews and debugs.

Chief flips modes per task. Minerva never uses `Write` or `Edit` on `scripts/` or `sql/`.

## Prerequisites

Run this checklist before Milestone 0. Every box must be ticked.

- [ ] `gcloud auth login` completed for `tylerjhayden@gmail.com`
- [ ] `gcloud config get-value project` returns `startup-innovators`
- [ ] `gcloud auth application-default login` completed (ADC for client libraries)
- [ ] `gcloud auth application-default set-quota-project startup-innovators` set
- [ ] `bq ls` succeeds (BigQuery API reachable)
- [ ] `gitleaks version` returns a version number
- [ ] `git config core.hooksPath` returns `.githooks`
- [ ] `python3 --version` returns 3.9+

One-liner sanity check:

```bash
gcloud config get-value project \
  && bq ls >/dev/null \
  && gitleaks version >/dev/null \
  && [ "$(git config --default '' core.hooksPath)" = ".githooks" ] \
  && echo OK
```

If you don't see `OK`, fix the failing prereq before going further.

## Repository layout

```
startup-innovators/
├── data/
│   ├── source/dream_company_master.csv     # Untouched source (162 rows)
│   ├── enriched/accounts_enriched.csv      # Source + 10 synthetic columns (Milestone 0)
│   └── splits/{train,predict}.csv          # 80/20 split (Milestone 0)
├── scripts/
│   ├── enrich_accounts.py                  # Milestone 0a
│   └── split_data.py                       # Milestone 0b
├── sql/
│   ├── 01_explore.sql                      # Milestone 2
│   ├── 02_create_model.sql                 # Milestone 3
│   ├── 03_evaluate.sql                     # Milestone 4
│   ├── 04_predict_industry.sql             # Milestone 5
│   └── 05_predict_per_account.sql          # Milestone 6
├── docs/superpowers/specs/                 # Design spec + plan
└── README.md                               # This file
```

## Milestones

Each milestone has three sections: **Build**, **Hints**, **Verify**. Build is the spec. Hints are tiered — read only as far as you need. Verify is the gate to the next milestone.

---

### Milestone 0a — Enrich the source CSV

**Build.** Write `scripts/enrich_accounts.py`. Read `data/source/dream_company_master.csv`. Clean the dirty fields. Add ten synthetic columns per the generation rules in the spec. Write `data/enriched/accounts_enriched.csv` with all 20 columns: the 10 source columns first (cleaned, in source order), then the 10 synthetic columns in spec table order — so `current_arr_usd` is column 11. Booleans must be written lowercase (`true`/`false`) so BigQuery autodetect picks them up as BOOL. Numerics must be written without `$` or commas. Output must be deterministic across runs.

**Hints.**

1. The label-conditioned sampling is the whole point — features for customers come from a different distribution than features for prospects. That's the signal the model will learn.
2. Stdlib only — `csv`, `random`, `math`. No pandas.
3. Dirty fields: `Annual Revenue` has `$` and commas; `Employees` has commas; missing values appear as `--` or empty string. Clean to integers / floats / empty.
4. Determinism comes from `random.seed(42)` at the top.
5. Generation order matters: assign `is_customer` first (with mild employee-count bias), THEN sample features conditioned on the label, THEN flip ~5% of labels, THEN re-derive `current_arr_usd` from the post-flip label.

**Verify.**

```bash
wc -l data/enriched/accounts_enriched.csv
# Expect: 163 (header + 162)

head -1 data/enriched/accounts_enriched.csv | tr ',' '\n' | wc -l
# Expect: 20

# Label distribution sanity check
awk -F, 'NR>1 && $11+0 > 0 {n++} END {print n " customers of " NR-1 " rows"}' data/enriched/accounts_enriched.csv
# Expect: ~45–55 customers of 162 rows (~30% positive base rate)
```

The awk verify only works if (a) `current_arr_usd` is column 11 and (b) cleaned numerics contain no commas. Both are required by the Build above.

---

### Milestone 0b — Split into train and predict

**Build.** Write `scripts/split_data.py`. Read the enriched CSV. Seeded shuffle. First 80% to `data/splits/train.csv`, rest to `data/splits/predict.csv`. Both files keep all 20 columns including `current_arr_usd` (the label source).

The predict half is unseen by the model — it's the closest thing to a holdout in this lab, even though we use it for scoring rather than formal evaluation. Stretch goal 4 evaluates against it for an honest generalization estimate.

**Hints.**

1. Use `random.seed(42)` again for reproducibility.
2. `random.shuffle()` mutates in place — apply it to the row list after reading.
3. Header line goes into both output files.
4. Round the split point with `int(0.8 * n_rows)` — for 162 rows that's 129 train / 33 predict.

**Verify.**

```bash
wc -l data/splits/train.csv data/splits/predict.csv
# Expect: 130 train.csv (header + 129), 34 predict.csv (header + 33)

# No row leaks across the split
diff <(tail -n +2 data/splits/train.csv | sort) <(tail -n +2 data/splits/predict.csv | sort) | head
# Expect: no shared rows
```

Commit and push when both files exist.

---

### Milestone 1 — Load CSVs into BigQuery

**Build.** Create dataset `propensity_lab` (US multi-region). Load `train.csv` into table `accounts_train`. Load `predict.csv` into table `accounts_predict`. Run it once via the Console, once via `bq` CLI.

**Hints.**

1. **Console path:** BigQuery → SQL workspace → click `startup-innovators` → ⋮ → Create dataset → name `propensity_lab`, location `us` (multi-region). Then click the dataset → Create table → upload, autodetect schema.
2. **CLI path:** `bq mk --dataset --location=US propensity_lab`, then `bq load --autodetect --source_format=CSV --skip_leading_rows=1 propensity_lab.accounts_train data/splits/train.csv`.
3. Autodetect picks up BOOL only for lowercase `true` / `false`. Python's `str(bool)` returns `True` / `False` (capitalized) — those autodetect as STRING. Milestone 0a's Build pins lowercase output for this reason; if you skipped that, drop and reload after fixing.
4. Drop and reload if you mess up: `bq rm -ft propensity_lab.accounts_train`.

**Verify.**

```bash
bq ls propensity_lab
# Expect: 2 tables (accounts_train, accounts_predict)

bq show --schema --format=prettyjson propensity_lab.accounts_train | head -30
# Expect: 20 fields, sensible types

bq query --use_legacy_sql=false 'SELECT COUNT(*) AS n FROM propensity_lab.accounts_train'
# Expect: 129

bq query --use_legacy_sql=false 'SELECT COUNT(*) AS n FROM propensity_lab.accounts_predict'
# Expect: 33
```

---

### Milestone 2 — Explore + create the training view

**Build.** Write `sql/01_explore.sql`. First a `SELECT` that pulls features and the derived label (`IF(current_arr_usd > 0, 1, 0) AS label`). Run it. Then wrap it in `CREATE OR REPLACE VIEW propensity_lab.training_data AS ...` so the model has a single object to train on.

**Hints.**

1. Drop `current_arr_usd` from the feature list — it would leak the label perfectly.
2. Drop high-cardinality identifiers (`company_name`, `domain`, `website`, `company_street`) and the source-tracking column (`source`) — they don't generalize.
3. Keep `industry`, `employees`, `annual_revenue`, and the seven synthetic boolean/int features.
4. `company_state` (up to 50 levels) and `company_country` (likely near-constant for this US-heavy list) are judgment calls. On 129 train rows, BQML's one-hot encoding of `company_state` will produce many single-row dummies that overfit. Consider dropping both, or at least dropping `company_country` if >95% are US.
5. **Console:** SQL editor → Run → Save → Save view (pick dataset `propensity_lab`, name `training_data`). Note: Console "Save view" wraps the *current query* as a view. The CLI path uses the explicit `CREATE OR REPLACE VIEW` DDL from your SQL file. Same end state, different mechanic.
6. **CLI:** `bq query --use_legacy_sql=false < sql/01_explore.sql`.

**Verify.**

```bash
bq ls propensity_lab
# Expect: training_data now appears alongside the two tables

bq query --use_legacy_sql=false '
  SELECT label, COUNT(*) AS n
  FROM propensity_lab.training_data
  GROUP BY label
'
# Expect: two rows, label 0 and label 1, ~70/30 split
```

---

### Milestone 3 — Train the model

**Build.** Write `sql/02_create_model.sql`. One `CREATE OR REPLACE MODEL` statement against `propensity_lab.propensity_model`. Logistic regression. Label column is `label`. Source: `propensity_lab.training_data`. Disable BQML's automatic eval-split so Milestone 4's training-set evaluation is honest (otherwise BQML auto-holds out 10–20% for early stopping, and "evaluating on training data" silently means evaluating on a mix).

**Hints.**

1. `CREATE MODEL` syntax: `CREATE OR REPLACE MODEL <name> OPTIONS(...) AS <SELECT>`. The `OPTIONS` block is where the model type and label column live.
2. The three options you need: `model_type`, `input_label_cols`, and `data_split_method`. Look up the literal values in the BQML docs (you want logistic regression, label `'label'`, and no split).
3. `input_label_cols` is a list (note the brackets) even with one label.
4. BQML auto-handles categorical encoding for STRING columns and standardization for numerics. No preprocessing needed.
5. `OR REPLACE` lets you re-run the script after iterating on the view.
6. **Console:** paste, Run. ~10–30 seconds for 130 rows.
7. **CLI:** `bq query --use_legacy_sql=false < sql/02_create_model.sql`.
8. Inspect after training: Console → click model → Details / Training / Evaluation / Schema tabs. CLI: `bq show --model propensity_lab.propensity_model`.

**Verify.**

```bash
bq show --model propensity_lab.propensity_model | head -20
# Expect: Model type LOGISTIC_REGRESSION, label column 'label', non-zero training run count
```

---

### Milestone 4 — Evaluate the model

**Build.** Write `sql/03_evaluate.sql`. One `SELECT * FROM ML.EVALUATE(...)` against the model and the training view.

**Hints.**

1. `ML.EVALUATE(MODEL ..., TABLE ...)` is a table-valued function. Wrap it in `SELECT * FROM ML.EVALUATE(...)`.
2. Returned columns for binary `LOGISTIC_REG`: `precision`, `recall`, `accuracy`, `f1_score`, `log_loss`, `roc_auc`, plus a `threshold` column (BQML's default 0.5). The first four are computed at that threshold; `log_loss` and `roc_auc` are threshold-free.
3. **In-sample evaluation is optimistic.** You're scoring the model on the same data it trained on. Expect the ROC-AUC here to be **0.05–0.15 higher** than what the model would get on a true holdout. Stretch goal 4 measures the gap.
4. Sanity floor for accuracy: a constant "no-buy" predictor scores ~0.70 (since ~30% of rows are positive). Beat that by a wide margin or your model isn't learning.
5. Expected in-sample ROC-AUC: **0.88 to 0.97**. Below 0.85 means something is off in the generator or the feature set. Stretch goal 4's holdout AUC will likely land in 0.80–0.90.
6. **Console:** SQL editor → paste → Run.
7. **CLI:** `bq query --use_legacy_sql=false < sql/03_evaluate.sql`.

**Verify.**

```bash
bq query --use_legacy_sql=false --format=pretty < sql/03_evaluate.sql
# Expect: roc_auc in 0.88–0.97 range; accuracy comfortably above 0.70
```

---

### Milestone 5 — Predict aggregated by industry

**Build.** Write `sql/04_predict_industry.sql`. Use `ML.PREDICT` against `accounts_predict`, then group by `industry`, summing `predicted_label`. Order descending. Limit 10.

**Hints.**

1. `ML.PREDICT(MODEL ..., TABLE ...)` returns every input column plus `predicted_label` and `predicted_label_probs`.
2. `predicted_label` is the model's hard 0/1 call. Summing it gives "predicted buyer count per industry."
3. Wrap the `ML.PREDICT` call as a subquery, then `SELECT industry, SUM(predicted_label) AS predicted_buyers ... GROUP BY industry`.
4. Industry was generated uniformly with no label correlation, so the ranking is workflow practice, not insight. Expected output: numbers proportional to industry sample sizes.

**Verify.**

```bash
bq query --use_legacy_sql=false --format=pretty < sql/04_predict_industry.sql
# Expect: <=10 rows, industries with predicted_buyers descending
```

---

### Milestone 6 — Top-10 accounts by buy probability

**Build.** Write `sql/05_predict_per_account.sql`. Per-account ranking: `company_name`, `industry`, `predicted_label`, and the probability of the positive class. Order by probability descending. Limit 10.

**Hints.**

1. The output is the prospecting hot list — model's ranked guess at who to call first.
2. You need the probability of class `1` per row. `predicted_label` won't help (it's the hard call); you want the underlying probability that produced it.
3. `predicted_label_probs` is a `STRUCT` array — one struct per class, each with `label` and `prob` fields. You need to extract the struct *for class 1*, not "the first struct" — array order isn't guaranteed by class identity.
4. Tempting but unsafe: `predicted_label_probs[OFFSET(0)].prob`. `OFFSET(0)` is whichever class BQML lists first, not necessarily the positive class. If you go this route, verify with `SELECT predicted_label_probs FROM ML.PREDICT(...) LIMIT 1` first.
5. Safe pattern: a correlated subquery that unnests and filters by class label.

**Verify.**

```bash
bq query --use_legacy_sql=false --format=pretty < sql/05_predict_per_account.sql
# Expect: 10 rows, p_buy descending, top scores roughly in 0.6–0.95 range
```

---

## Stretch goals

Optional reps after the main path lands. Each is a separate commit.

1. **Boosted trees.** Train `BOOSTED_TREE_CLASSIFIER` (e.g. as `propensity_lab.propensity_tree`) on the same view. Compare ROC-AUC. Watch for overfitting on 130 rows.
2. **Feature importance for the boosted tree.** `SELECT * FROM ML.FEATURE_IMPORTANCE(MODEL propensity_lab.propensity_tree)`. Which synthetic signals carry weight? `competitor_in_job_posts` and `tech_stack_match` were generated as pure noise — they should rank low. (`ML.FEATURE_IMPORTANCE` is tree-only and will error against the logistic model — use `ML.WEIGHTS` for that one.)
3. **Logistic-regression coefficients.** `SELECT * FROM ML.WEIGHTS(MODEL propensity_lab.propensity_model)`. Cross-check signs and magnitudes against the generator's label-conditioning rules.
4. **Export predictions.** Materialize predictions as a table, then `bq extract --destination_format=CSV ... gs://...` to a GCS bucket and download.
5. **Holdout evaluation.** `ML.EVALUATE(MODEL ..., TABLE propensity_lab.accounts_predict)` for the honest generalization estimate. Compare to the in-sample ROC-AUC from Milestone 4 — the gap is your overfitting estimate.
6. **Hyperparameter tuning.** `ML.HYPERPARAMETER_TRIALS` to sweep `l1_reg` and `l2_reg`.

## Glossary — BQML SQL functions

| Function | What it does |
|---|---|
| `CREATE MODEL ... OPTIONS(model_type=...)` | Trains a model. `model_type` selects the algorithm (`LOGISTIC_REG`, `LINEAR_REG`, `BOOSTED_TREE_CLASSIFIER`, `KMEANS`, etc.). `input_label_cols` names the target column. |
| `ML.EVALUATE(MODEL ..., TABLE ...)` | Returns metrics for the model against any table with the same schema. For binary classifiers: `precision`, `recall`, `accuracy`, `f1_score` (all at the `threshold` column's value, default 0.5), plus `log_loss` and `roc_auc` (threshold-free). |
| `ML.PREDICT(MODEL ..., TABLE ...)` | Scores every row of the input table. Returns the input columns plus `predicted_label` (hard call at the default 0.5 threshold) and `predicted_label_probs` (`STRUCT` array of `{label, prob}` per class). |
| `ML.FEATURE_IMPORTANCE(MODEL ...)` | Tree-based models only (boosted trees, random forest). Errors on `LOGISTIC_REG`. |
| `ML.WEIGHTS(MODEL ...)` | Per-feature coefficients for linear models. The logistic-regression equivalent of feature importance. |

## Security

This is a public repo. Hygiene rules:

- Pre-commit secret scanning via [gitleaks](https://github.com/gitleaks/gitleaks). After cloning:
  ```bash
  brew install gitleaks
  git config core.hooksPath .githooks
  ```
- No service account keys in the repo. Auth is via Application Default Credentials: `gcloud auth application-default login`.
- `data/source/dream_company_master.csv` is sample data, not real prospect intel.

## License

MIT — see [LICENSE](LICENSE).
