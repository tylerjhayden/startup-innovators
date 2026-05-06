# startup-innovators

A hands-on BigQuery ML lab. Predict B2B account propensity to buy, using a small account list enriched with synthetic firmographic and intent signals.

This is a self-built parallel of Google Cloud Skills Boost's *Predict Visitor Purchases with BigQuery ML* lab — same workflow (explore → train → evaluate → predict), different domain (B2B propensity instead of e-commerce visitors).

## Status

Scaffolded. Lab manual and scripts in progress — see [the design spec](docs/superpowers/specs/2026-05-06-bqml-propensity-lab-design.md) for the full plan.

## Stack

- BigQuery ML (logistic regression)
- Python stdlib for data prep
- gcloud / bq CLI for Console-parity workflow

## Security

This is a public repo. Hygiene rules:

- Pre-commit secret scanning via [gitleaks](https://github.com/gitleaks/gitleaks). After cloning, run:
  ```bash
  brew install gitleaks
  git config core.hooksPath .githooks
  ```
- No service account keys in the repo. Use Application Default Credentials: `gcloud auth application-default login`.
- The included account list (`data/source/dream_company_master.csv`) is sample data, not real prospect intel.

## License

MIT — see [LICENSE](LICENSE).
