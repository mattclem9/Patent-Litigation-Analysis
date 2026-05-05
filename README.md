# U.S. Patent Litigation Analysis (1972–2020)

![Status](https://img.shields.io/badge/Status-Active%20Research-blue)

This is an ongoing empirical research project analyzing federal patent litigation cases from 1972–2020 using the United States Patent and Trademark Office's Patent Litigation Docket Reports Data. The project builds a comprehensive data pipeline encompassing case deduplication, cause of action classification, geographic extraction, and entity resolution of litigation parties. Preliminary visualizations are complete; entity resolution and longitudinal analysis are ongoing.

---

## Data Source

**USPTO Patent Litigation Docket Reports Data (1972–2020)**  
[https://www.uspto.gov/ip-policy/economic-research/research-datasets/patent-litigation-docket-reports-data](https://www.uspto.gov/ip-policy/economic-research/research-datasets/patent-litigation-docket-reports-data)

| File | Rows | Description |
|------|------|-------------|
| `cases.csv` | 96,966 | Core case records: case number, court, judges, cause of action, filing/closing dates |
| `names.csv` | 626,465 | Party names (plaintiffs and defendants) linked to cases |
| `attorneys.csv` | ~1,839,849 | Attorney names, contact info, and roles per case |
| `patents.csv` | ~159,402 | Patent numbers associated with cases |
| `pacer_cases.csv` | ~97,000 | PACER court metadata used for cross-referencing and validation |

> Raw data files are not included in this repository due to size constraints. All raw data is publicly available from the USPTO link above. Cleaned and processed files are available on [Zenodo — link forthcoming].

---

## Processing Pipeline

### 1. Case Deduplication
**Notebook:** `notebooks/Cases_deduplication.ipynb`

A composite identifier was constructed for each record using `case_number + district_id + first 3 characters of case_name (lowercased)`. This approach is consistent with the USPTO technical documentation, which notes that case numbers alone do not uniquely identify cases across districts.

- 96 records had no case name and could not be initially assigned an identifier. A cross-reference mapping was applied to borrow case names from records sharing the same case number, reducing null identifiers from 96 to 78. The remaining 78 were assigned identifiers using `case_number + district_id + "noname"`.
- 163 duplicate identifiers were exported for manual review. All confirmed duplicates represented the same underlying case differing only in data completeness. The record with the greatest number of populated fields was retained.
- **Final dataset: 96,881 unique cases** (reduced from 96,966).

### 2. Cause of Action Classification
**Notebook:** `notebooks/Case-cause.ipynb`

Raw `case_cause` strings (e.g., `35:271 Patent Infringement`) were parsed using regex to extract the U.S. Code title and section number. A ~110-entry lookup table maps normalized codes to human-readable labels.

Key cleaning decisions:
- Entries where `case_cause` contained corrupted numeric values (e.g., Excel date serial numbers) were classified as **Unknown**. Four specific cases affected: *Mikohn Gaming Corporation v. Paltronics Inc.*, *Storage Technology Corporation v. Certance LLC*, *Spectrum Laboratories Inc. v. Tech International Inc. et al.*, and *Wireless Divide Technology LLC v. Amimon Inc. et al.*
- Several data entry errors were identified and corrected by looking up individual cases: `32:102` → `35:102` (Conditions for Patentability), `35:217` → `35:271` (Patent Infringement), `38:2011` → `28:1651` (All Writs Act).
- The **Unknown** category comprises: undisclosed cause of action codes (e.g., `0:0`, `99:9999`), blank entries, and the four corrupted values above.
- Causes with fewer than 200 cases are collapsed into **Other** in visualizations for readability.
- The dominant cause of action is **35:271 Patent Infringement** (~50,000+ cases).

### 3. Geographic Extraction
**Notebook:** `notebooks/us-state-territory-extractor.ipynb`

U.S. state and territory was extracted from the `court_name` field using regex matching against all 50 states plus D.C., Puerto Rico, Guam, the U.S. Virgin Islands, and American Samoa. Special handling was implemented to ensure `West Virginia` is matched before `Virginia` to prevent misclassification. Achieved **100% coverage** across all cases.

### 4. Party Name Cleaning & Entity Resolution
**Notebook:** `notebooks/names-clean.ipynb` *(Ongoing)*

This is the most complex component of the pipeline. The names file contains 626,465 observations representing case/party pairs. Processing steps include:

**Deduplication:**  
Records were deduplicated on `party_type + name + identifier`. The ~157 remaining duplicate groups (largely attributable to multiple attorneys listed per party in PACER) were resolved by retaining one row per unique party per case. Approximately 120 records lacking identifiers after the `case_row_id` mapping — likely due to case deduplication removing certain row numbers — were resolved manually by matching on case number and case name.

**Name Standardization:**  
Leading symbols (`.`, `-`, `&`) were stripped from name fields. Fuzzy matching using `thefuzz` was applied to names appearing 100+ times to identify and consolidate near-duplicate name variants (e.g., `Apple Inc.` / `Apple Inc` / `Apple, Inc.`). A Union-Find algorithm was used to chain transitive matches into single canonical names, stored in a new `name_consolidated` field.

**Entity Type Classification:**  
A derived `entity_type` field was created to classify each party as `Organization`, `Individual`, or `Placeholder`. This field does not exist in the original USPTO data and represents an original contribution of this project. Classification was performed through a layered strategy:

1. Regex matching on ~80 organizational keywords and legal suffixes (Inc, LLC, Corp, S.A., N.V., GmbH, L.P., etc.) including multilingual variants
2. Slash/space pattern matching for European legal suffixes (S.A., N.V., B.V., C.V., etc.)
3. Indicator word matching for organizational terms unlikely to appear in individual names (Systems, Wireless, Partnership, Software, etc.)
4. Numeric character detection (names containing numbers are almost always organizations)
5. Geographic last-word rules (names ending in America, Japan, Korea, Europe, etc.)
6. NER-based classification using spaCy `en_core_web_sm` on high-frequency names (appearing 5+ times), followed by manual review of 600 Organization and 455 Uncertain classifications
7. Manual review of 1,000 single-word entries classified as Individual — no genuine individual names were found, so all single-word entries (except `Does`/`DOES` variants) were reclassified as Organization or Placeholder

`Placeholder` entries include unnamed defendants (`Does`, `Unknown Parties`), role titles (`Special Master`, `Facilitative Mediator`), and collective designations (`All Defendants`, `Service List`).

**Entity Resolution (Ongoing):**  
Splink 4.x is being used for probabilistic deduplication of the ~175,000 unique consolidated names, blocked on first-three-token groups to reduce false positive matches between individuals sharing common first names.

---

## Preliminary Results & Visualizations

All outputs are saved in `results/`.

### Cases by State — Choropleth Maps
**Files:** `results/Heat Maps/`

Choropleth maps showing total patent cases filed by U.S. state and territory (1972–2020). Available in both linear and log scale. The log scale map is recommended for comparing mid-tier states given the extreme concentration in Texas and California.

| State | Cases |
|-------|-------|
| Texas | 18,892 |
| California | 16,905 |
| Delaware | 10,444 |
| New York | 5,494 |
| Illinois | 5,041 |

The Eastern District of Texas alone accounts for 9,744 cases — the single busiest patent litigation venue in the dataset.

### Cases Filed Per Year — Time Series
**File:** `results/Time Series Line Graph/cases-per-year-line-graph-ploty.png`

Annual case filings from 1972–2020 annotated with two major legislative events:
- **Federal Circuit established (1982)** — creation of a specialized appellate court for patent cases
- **America Invents Act enacted (2011)** — major patent reform legislation

The chart shows a sharp increase in filings following the AIA, peaking around 2013–2014 before declining — consistent with the academic literature on NPE litigation trends.

### Cause of Action — Bar Chart
**File:** `results/Cause of Action Bar Chart/cause-of-action-bar-chart.png`

Horizontal bar chart of cases by mapped U.S. Code cause of action. Causes with fewer than 200 cases are collapsed into Other. 35:271 Patent Infringement dominates at roughly 4× the next most common cause.

---

## Repository Structure
Patent-Litigation-Analysis/
├── notebooks/          # Jupyter notebooks for each processing stage
├── results/            # Output visualizations (PNG and HTML)
│   ├── Heat Maps/
│   ├── Time Series Line Graph/
│   └── Cause of Action Bar Chart/
├── docs/               # Methodology documentation
├── src/                # Utility scripts
└── data/               # Not tracked in git (see Data Source above)
├── raw/
└── processed/

---

## Processed Data

Intermediate and final cleaned files are stored in `data/processed/` (not tracked in git).

| File | Description |
|------|-------------|
| `cases/cases_cleaned_v4.csv` | Deduplicated cases with normalized identifiers, state/territory, and cause of action labels |
| `names/names_cleaned_v3.csv` | Cleaned and deduplicated party names with consolidated name and entity type fields |
| `names/clusters_for_review.csv` | Name clusters flagged for manual review |
| `names/ner_organizations_review.csv` | NER-detected organization names reviewed for entity classification |

---

## Methodology

Full methodology documentation including data cleaning decisions, manual review notes, and classification rationale is available in `docs/Patent-litigation-methodology.docx`.

---

## Requirements
Python 3.11+
pandas
numpy
matplotlib
plotly
thefuzz
spacy
splink>=4.0
re
Jupyter Notebook


```

---

## Status & Roadmap

- [x] Case deduplication
- [x] Cause of action classification
- [x] Geographic extraction
- [x] Preliminary visualizations (choropleth maps, time series, bar chart)
- [x] Name deduplication and standardization
- [x] Entity type classification
- [ ] Entity resolution via Splink (ongoing)
- [ ] Corporate group consolidation via OpenCorporates/Wikidata
- [ ] Longitudinal analysis of party appearance frequency
- [ ] Analysis of most litigious corporate entities

---

## Citation

Raw data:  
Toole, Andrew, Richard Miller, and Ted Sichelman. "Technical Documentation for Patent Litigation Docket Reports Data, 1972–2020." USPTO Economic Working Paper No. 2024-1. March 2024. Available at SSRN: https://ssrn.com/abstract=4780166
