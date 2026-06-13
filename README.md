# Enterprise Deposit Data Quality & Anomaly Detection Pipeline

**An end-to-end data-quality and risk-surfacing pipeline for institutional banking records — from raw regulatory data to an automated cleaning engine, an unsupervised ML anomaly detector, and executive Tableau dashboards.**

`Python` · `pandas` · `scikit-learn` · `Isolation Forest` · `Tableau` · `Jupyter`

---

## Overview

This project takes three large, genuinely messy, publicly available U.S. banking/financial datasets and runs them through a four-phase pipeline that mirrors how a real data-quality and risk function operates:

1. **Source** realistic, regulatory-grade data (no Kaggle, millions of rows, documented messiness).
2. **Clean & score** each dataset with an automated engine that produces a 0–100 *Data Health Score*.
3. **Detect anomalies** with an unsupervised Isolation Forest and surface the top 1% highest-risk records for human audit.
4. **Visualize** the results in interactive Tableau dashboards built for non-technical stakeholders.

The pipeline is column-agnostic: dataset columns are mapped to logical roles in a central config block, so the same code runs across all three datasets.

---

## Results at a glance

| Dataset | Rows processed | Health score (before → after) | Anomalies flagged | Anomaly rate |
|---|---|---|---|---|
| **HMDA** — mortgage loan-level records | 13.5M | 88.3 → 99.9 | 134,944 | 1.0% |
| **CFPB** — consumer complaints | 15.7M | 92.7 → 93.1 | 444,022 | 2.8% |
| **SEC EDGAR** — XBRL financial facts | 3.7M | 94.6 → 95.0 | 19,082 | 0.5% |

The HMDA cleaning step alone removed 74,764 duplicate rows and imputed tens of millions of missing financial values via segment-based medians, lifting the null rate from 23.1% to 0.25%.

---

## Datasets

All three come from official U.S. government / regulator sources and were chosen for scale (100k+ rows, in practice millions), multi-featured financial records, and documented real-world messiness — missing values, sentinel/exempt codes, mixed types, and heavy-tailed outliers.

- **HMDA Modified LAR (primary)** — Home Mortgage Disclosure Act loan-level register, published by FFIEC / CFPB. Loan amount, income, property value, interest rate, product tier, geography, and demographics. *Messiness:* `Exempt` strings in numeric fields, sentinel codes, missing income/property value, right-tailed loan amounts.
- **CFPB Consumer Complaint Database** — complaint-level records with product, issue, company, channel, response timeliness, and a free-text narrative. *Messiness:* large blocks of missing narrative/sub-issue/ZIP, inconsistent dates, redacted PII, mixed-type ZIP codes.
- **SEC EDGAR Financial Statement Data Sets** — quarterly XBRL archives across four tab-delimited files (SUB, NUM, TAG, PRE) requiring multi-file joins on `adsh`. *Messiness:* custom vs. standard tags, mixed units and scales, very large files.

---

## How it works

### Phase 2 — Automated cleaning & quality engine
- **Audit & Health Score** — scans for null cells, exact-duplicate rows, and structural type mismatches, returning a weighted 0–100 score (50% nulls, 30% duplicates, 20% mixed types).
- **Type standardization** — strips `$`, commas, and accounting parentheses from currency fields, coerces to float, parses timestamps, normalizes categorical casing/whitespace.
- **Segment-based imputation** — fills missing financial values with the median of the row's own business segment/tier (not a blunt global mean), falling back to the global median only when a whole segment is empty.
- **Safe deduplication** — drops duplicates on an explicit key subset, keeps the first occurrence, and reports the count removed.

### Phase 3 — Unsupervised anomaly detection
- **Robust scaling** — `RobustScaler` centers on the median and scales by the IQR so a handful of giant deposits don't dominate the transform — critical for heavy-tailed financial data.
- **Model** — `IsolationForest` (200 trees, contamination set to the expected anomaly rate). Unsupervised, no labels required.
- **Scoring & audit queue** — appends `is_anomaly` (1 = normal, −1 = anomalous) and `anomaly_score` (higher = riskier), then exports the top 1% highest-risk records as a ranked manual-review queue.

### Phase 4 — Tableau dashboards
One dashboard per dataset (HMDA Lending Risk, CFPB Complaint Anomalies, SEC Filing Anomalies), each following the same layout grammar: a KPI strip on top, insight charts in the middle, and an analyst audit queue at the bottom. A single color rule runs across every view — **Anomalous = red, Normal = navy** — so red always means *look here*. Each dashboard is driven entirely by its `*_deposits_scored.csv` output.

---

## Repository structure

```
.
├── phase2_cleaning_engine.ipynb      # Phase 2 — cleaning & Data Health Score
├── phase3_anomaly_detection.ipynb    # Phase 3 — Isolation Forest scoring
├── phase2_cleaned_output/            # health summaries + run summary JSON
├── phase3_anomaly_output/            # audit queues + run summary JSON
├── Field Guide.docx                  # full project blueprint (Phases 1–3)
├── Phase4_Dashboard_Spec.docx        # Tableau build guide (Phase 4)
└── *.twbx                            # Tableau workbooks (see note below)
```

> **Note on large files.** The raw datasets and full cleaned/scored CSVs run into the gigabytes (the largest is 8 GB) and exceed both GitHub's 100 MB file limit and Git LFS's 2 GB limit, so they are excluded via `.gitignore` and are **not** part of this repository. Download links for the raw data are in `Field Guide.docx`; the notebooks regenerate every output. The small run-summary JSONs and health summaries are included so results are reproducible and inspectable without the bulk data.
>
> The Tableau workbooks (`*.twbx`) are tracked with **Git LFS** (see `.gitattributes`) so the dashboards remain part of the repo.

---

## Tech stack

Python (pandas, NumPy, scikit-learn), Jupyter, and Tableau. No exotic dependencies — the cleaning and detection logic is pure pandas / NumPy / scikit-learn.

```bash
pip install pandas numpy scikit-learn jupyter
```

Open `phase2_cleaning_engine.ipynb` to run the cleaning engine, then `phase3_anomaly_detection.ipynb` to score anomalies. Each phase writes its outputs and a run-summary JSON to the corresponding `*_output/` folder.

---

## Author

**Khoi Van** · Denison University · Portfolio project in data analytics and applied machine learning.
