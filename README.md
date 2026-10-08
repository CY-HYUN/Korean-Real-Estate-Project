# Seoul Real Estate Market Analysis

A data collection, integration, and visualization project (completed August 2024): scrapes Seoul property listings from the Zigbang real-estate API, merges them with 11 years of Korean macroeconomic indicators and Seoul open data, and turns the result into interactive Folium maps and a matplotlib macro-indicator chart.

**Team and my role.** A team project in which I did most of the work, and I assembled this repository from the team's files. The original team workspace and data mirror are `real_estate_project-main/` and `aaqq8/SteadyEstate`.

[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![Pandas](https://img.shields.io/badge/Pandas-2.0+-green.svg)](https://pandas.pydata.org/)
[![Folium](https://img.shields.io/badge/Maps-Folium-77B829.svg)](https://python-visualization.github.io/folium/)

## Results at a glance

All numbers below can be checked against files committed in this repository (row counts via `pandas.read_excel`, ID ranges from the scraper source, the migration totals from the committed dataset, district counts from a checked-in notebook output).

| Result | Number | Evidence in repo |
| --- | --- | --- |
| Listing/transaction rows collected and committed | **123,570** across 8 scraped datasets | `data/seoul_oneroom_*.xlsx`, `data/서울시_아파트_*.xlsx`, `data/서울시_상가_*.xlsx` |
| Zigbang API ID ranges set in the scrapers | **177,794** IDs across 4 endpoints | ID ranges in `src/data_collection/project ex.py`–`ex3.py` (request logs were not kept, so the share of IDs that returned a listing is not known) |
| Datasets integrated (macro + city + listings) | **45 Excel files, 34 MB**, 2013–2023 | `data/` |
| Headline insight: Seoul lost residents to domestic migration **every year 2013–2023** | cumulative net **−961,881** people | computed from `data/서울시_인구이동.xlsx` |
| Top district by apartment-sale listings | 은평구 Eunpyeong-gu (**403**), then 강서구 388, 강남구 351 | checked-in output of `notebooks/서울시 아파트 매매.ipynb` |

The ID count and the row count are not a ratio. One listing becomes several rows once its transaction history is flattened (1 listing → N rows), so 123,570 rows says nothing about how many of the 177,794 IDs were live.

![Seoul domestic migration 2013-2023](docs/img/seoul_migration.png)

*Seoul in/out domestic migration from `data/서울시_인구이동.xlsx` (the shaded band is the yearly net loss). This preview was drawn for the README; no script in `src/` draws it.*

## Quick start

Clone the repository and install the dependencies:

```bash
git clone https://github.com/CY-HYUN/Korean-Real-Estate-Project.git
cd Korean-Real-Estate-Project

# minimal set actually needed by the analysis/visualization scripts (verified):
pip install pandas openpyxl requests matplotlib folium
# or the full stack (adds jupyter, seaborn, plotly, geopandas, scipy, statsmodels):
pip install -r requirements.txt
```

Run the chart script and open the notebooks (both scripts below exit cleanly on Python 3.12, checked 2026-10-08):

```bash
python "src/visualization/GDP 금리 물가.py"    # 3-panel chart: CPI inflation, CPI index, KRW/USD
python "src/analysis/서울시_인구이동.py"        # loads the migration table into a DataFrame; prints nothing (inspect it in Spyder/Jupyter)
jupyter notebook notebooks/                    # Folium choropleth maps + preprocessing walkthrough
```

**Data notes (honest):**

- Every dataset is committed in `data/` (45 Excel files, 34 MB) — nothing external is needed to inspect the data.
- The analysis/visualization scripts download identical copies from the project's GitHub data mirror (`raw.githubusercontent.com/aaqq8/SteadyEstate`), so they run from any clone without path edits. Internet is required; the mirror was live on 2026-10-08.
- Chart labels are Korean. On a machine without a Korean matplotlib font they render as boxes — add `plt.rcParams['font.family'] = 'Malgun Gothic'` (Windows) or another CJK font first.
- The scrapers in `src/data_collection/` are the archival record of the August 2024 collection run. They hit the live Zigbang API and write to hardcoded paths from the original machine — edit the output paths before re-running them.

## What was built

```text
Zigbang API (4 endpoints, 177,794 IDs polled)
        │  src/data_collection/project ex.py, ex1.py, ex2.py, ex3.py
        ▼
Flatten nested transaction JSON (1 listing → N transaction rows)
        │  project file.py / ex4.py
        ▼
Enrich + filter: subway-adjacency flag → Seoul-only → split by sales type
        │  ex5.py → ex6.py → ex7.py  (ex8.py: reverse geocoding)
        ▼
Committed datasets (data/*.xlsx, 123,570 rows)
        +
Macro & city data: Bank of Korea (GDP, base rate, CPI, FX, GNI),
Korea Real Estate Board price indices, Seoul open data (migration, subway
land-value index, ROI by property type), 2013–2023
        │
        ▼
Analysis & visualization: interactive Folium maps (notebooks/) —
choropleth by district + marker clustering per listing — and one
matplotlib chart (src/visualization/GDP 금리 물가.py)
```

### Repository layout

```text
data/                       45 committed Excel datasets (scraped listings + macro/city data)
src/data_collection/        23 scripts — Zigbang scrapers, JSON flattening, merge/filter steps
src/visualization/          14 scripts — 1 draws the CPI/FX chart; 13 load one macro table each (GDP, rates, CPI, GNI) into a DataFrame
src/analysis/                5 scripts — each loads one Seoul table (migration, subway land value, ROI, transaction volumes) into a DataFrame
notebooks/                   4 notebooks — Folium district maps, Zigbang preprocessing walkthrough
docs/                       DETAILS.md (methodology + data dictionary), img/ (rendered previews)
real_estate_project-main/   legacy: original unorganized 2024 workspace, kept as-is
requirements.txt
```

The scraped one-room / apartment / commercial datasets break down as:

| Dataset | Rows |
| --- | --- |
| One-room monthly rent / jeonse / sale | 39,486 / 12,631 / 655 |
| Apartment sale / monthly rent | 5,475 / 11,591 |
| Commercial monthly rent / sale / jeonse | 53,264 / 442 / 26 |
| **Total** | **123,570** |

![Macro indicators chart](docs/img/economic_indicators.png)

*CPI inflation, CPI index, and KRW/USD (2013–2023), rendered from `data/물가상승률_물가지수_환율.xlsx` with the same aggregation as `src/visualization/GDP 금리 물가.py`.*

## Working with the data

Everything is plain Excel + pandas — no database or API keys needed. Example (verified against the committed files; output shown is the actual output):

```python
import pandas as pd

# Apartment sale listings scraped from Zigbang (committed in the repo)
df = pd.read_excel("data/서울시_아파트_매매.xlsx")
df["gu"] = df["address"].str.split().str[1]   # district from address
print(len(df))                                 # 5475
print(df["gu"].value_counts().head(5))
```

```text
gu
은평구    403
강서구    388
강남구    351
구로구    347
성북구    344
```

The one-room datasets carry richer fields per listing — `salesType`, `deposit`, `rent`, `전용면적M2` (exclusive area), `lat`/`lng`, `subwayYN`, and district columns (`local1`–`local3`) — see the data dictionary in [docs/DETAILS.md](docs/DETAILS.md).

## Key findings

- **Seoul shed population to domestic migration in all 11 observed years (2013–2023).** Cumulative net loss: 961,881 people; the largest single-year loss was 2016 (−140,257), the smallest 2023 (−31,250). Computed directly from the committed migration dataset.
- **Apartment-sale listings are spread across many districts, not concentrated in Gangnam.** Top districts by listing count: Eunpyeong (403), Gangseo (388), Gangnam (351), Guro (347), Seongbuk (344), out of 5,475 listings in 25 districts.
- **Monthly rent dominates the scraped rental market.** One-room listings: 39,486 monthly-rent rows vs 12,631 jeonse (lump-sum lease) vs 655 sale. Commercial listings are even more lopsided: 53,264 monthly rent vs 26 jeonse.

## Tech stack

- **Python 3.8+** — pandas, openpyxl for data processing
- **requests** — Zigbang API scraping and remote dataset fetch
- **matplotlib / seaborn** — static charts
- **Folium** — interactive choropleth maps with marker clustering (GeoJSON district boundaries)
- **Jupyter** — exploratory analysis and map notebooks

## Scope and limitations

- This repo contains **data collection, EDA, and visualization** — there is no predictive-modeling code (no regression/ARIMA) in it.
- Most scripts in `src/visualization/` and `src/analysis/` stop after loading a table (Spyder style, inspected in the variable explorer). The finished outputs are the Folium maps in `notebooks/` and the one matplotlib chart.
- The migration totals are simple yearly sums (in minus out) from the committed table; the per-year values are in [docs/DETAILS.md §5](docs/DETAILS.md). No committed script computes them.
- The scrape is a **point-in-time snapshot (August 2024)**; per-request success logs were not retained, so scrape-level success metrics are not reported here.
- `recentlyTransaction` JSON strings are parsed with `eval()` in the original scripts — acceptable for this trusted dataset, but `json.loads`/`ast.literal_eval` would be the production choice.
- `real_estate_project-main/` and a few `" (1)"`-suffixed files are preserved from the original team workspace; the curated copies live under `src/`.

Deeper methodology (API schema, per-script pipeline map, data dictionary) is in **[docs/DETAILS.md](docs/DETAILS.md)**.

## Data sources

- [Zigbang](https://www.zigbang.com/) — property listings (store, apartment, and one-room API endpoints)
- [Korea Real Estate Board](https://www.reb.or.kr/) — official price indices and transaction volumes
- [Bank of Korea](https://www.bok.or.kr/) — GDP, base rate, CPI, exchange rate, GNI (2013–2023)
- [Seoul Open Data Portal](https://data.seoul.go.kr/) — migration, subway land-value index, ROI statistics
- [southkorea-maps](https://github.com/southkorea/southkorea-maps) — GeoJSON district boundaries
