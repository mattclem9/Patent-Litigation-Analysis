# U.S. Patent Litigation Analysis (1972–2020)

This project is an ongoing effort to build a robust data pipeline and visualization suite for federal patent litigation cases. While core processing modules are functional, entity resolution and longitudinal analysis are currently being refined.

---
## Data Provenance & Versioning
.

## Data Sources https://www.uspto.gov/ip-policy/economic-research/research-datasets/patent-litigation-docket-reports-data

| File | Rows | Description |
|------|------|-------------|
| `data/raw/cases.csv` | ~96,966 | Core case records: case number, court, judges, cause of action, filing/closing dates |
| `data/raw/names.csv` | ~626,470 | Party names (plaintiffs and defendants) linked to cases |
| `data/raw/attorneys.csv` | ~1,839,849 | Attorney names, contact info, and roles per case |
| `data/raw/patents.csv` | ~159,402 | Patent numbers associated with cases |
| `data/raw/pacer_cases.csv` | ~97,000 | PACER court records used for cross-referencing and validation |

---

## Processing Pipeline

### 1. Case Deduplication (`notebooks/Cases_deduplication.ipynb`)

Creates a composite identifier from `case_number + district_id + first 3 chars of case_name` to detect duplicate records. When duplicates exist, the most complete row (by non-null field count) is retained. Reduced the dataset from 96,966 to **96,881 unique cases**.

### 2. Cause of Action Classification (`notebooks/Case-cause.ipynb`)

Parses raw `case_cause` strings (e.g., `35:271 Patent Infringement`) using regex to extract the U.S. Code title and section, then maps them to human-readable labels via a ~110-entry lookup table. The dominant cause of action is **35:271 Patent Infringement** (~50,000 cases).

### 3. State/Territory Extraction (`notebooks/us-state-territory-extractor.ipynb`)

Extracts the U.S. state or territory from the court name string using regex matching against a full list of 54 states and territories. Achieved **100% coverage** (no unmatched rows). Adds a `U.S. State/Territory` column used in geographic visualizations.

### 4. Name Cleaning (`notebooks/names-clean.ipynb`) *Ongoing*

Cleans and deduplicates party names from `names.csv`. Uses NER-based organization detection and fuzzy clustering to identify name variants that refer to the same entity. Produces review files for manual validation of uncertain clusters.

---

## Visualizations

All outputs are in `results/`.

### Cases by State — Heat Map
**Files:** `results/Heat Maps/`

Choropleth maps of the 50 states + D.C. and territories showing total patent cases filed. Available in both linear scale and log scale (for readability given Texas/California dominance). Exported as static PNG and interactive HTML.

Top states by case volume:
| State | Cases |
|-------|-------|
| Texas | 18,892 |
| California | 16,905 |
| Delaware | 10,444 |
| New York | 5,494 |
| Illinois | 5,041 |

The Texas Eastern District Court alone accounts for 9,744 cases — the single busiest patent court in the dataset.

### Cases Filed Per Year — Time Series
**File:** `results/Time Series Line Graph/cases-per-year-line-graph-ploty.png`

Line graph of annual case filings from 1972–2020 with two key legislative markers:
- **Federal Circuit established (1982)** — red vertical line
- **America Invents Act (2011)** — green vertical line

### Cause of Action — Bar Chart
**File:** `results/Cause of Action Bar Chart/cause-of-action-bar-chart.png`

Horizontal bar chart of cases grouped by their mapped U.S. Code cause of action. Causes with fewer than 200 cases are collapsed into an "Other" category.

---

## Processed Data
## Data Provenance & Versioning

The data in this repository has been transformed from the original USPTO Patent Litigation records.

Intermediate and final cleaned files are in `data/processed/`.

| File | Description |
|------|-------------|
| `cases/cases_cleaned_v4.csv` | Final cleaned cases dataset with normalized case numbers, court names, state/territory, and cause of action labels |
| `names/names_cleaned_v3.csv` | Final cleaned party names |
| `names/clusters_for_review.csv` | Name clusters flagged for manual review |
| `names/ner_organizations_review.csv` | NER-detected organization names for review |

---

## Methodology

See `docs/Patent-litigation-methodology.docx` for full methodology documentation.

---

## Requirements

- Python 3.11+
- pandas, numpy, matplotlib, seaborn, plotly, re
- Jupyter Notebook 
