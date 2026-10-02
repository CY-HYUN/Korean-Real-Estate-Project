# Methodology Details

Deep-dive companion to the main [README](../README.md). Everything here is written against the actual scripts in this repository — file references point to real files, and code descriptions match what the code does (not an idealized version of it).

## 1. Data collection (Zigbang API)

Four scraper scripts polled four Zigbang endpoints in August 2024. Each is a standalone script with a simple sequential loop: request an ID, skip non-200 responses and empty payloads, append the `item` object to a list, and export the list to Excel at the end.

| Script | Endpoint | ID range | IDs polled |
| --- | --- | --- | --- |
| `src/data_collection/project ex.py` | `apis.zigbang.com/v2/store/article/stores/{id}` | 800,000–846,000 | 46,000 |
| `src/data_collection/project ex1.py` | `apis.zigbang.com/v2/store/article/stores/{id}` | 700,001–720,001 | 20,000 |
| `src/data_collection/project ex2.py` | `apis.zigbang.com/property/apartments/{id}/v1` | 10,001–21,795 | 11,794 |
| `src/data_collection/project ex3.py` | `apis.zigbang.com/v3/items/{id}` | 39,640,000–39,740,000 | 100,000 |
| **Total** | | | **177,794** |

Design choices in the actual code:

- **Sequential polling** (one request at a time) to stay under the API's informal rate tolerance.
- **`try/except` around each request** so a single network failure does not kill a multi-hour run; failures are printed to the console and skipped.
- **Status-code check before parsing** to separate removed listings (non-200) from empty payloads (`item` missing).

The scripts do **not** implement checkpointing, retry queues, or failure-log files — a run that crashed was restarted from an adjusted ID range (the range comments in the scripts record this). Console logs from the 2024 runs were not retained, which is why the README reports committed row counts rather than scrape success rates.

### Observed response schema (store endpoint)

Fields actually consumed downstream: `id`, `address`, `lat`, `lng`, `pricePerSize`, and `recentlyTransaction` — a nested object whose `rentList` array holds per-transaction records (`netArea.m2`, `grossArea.m2`, `floor`, `type`, `utime` as Unix time). See `notebooks/직방 데이터 전처리 코드.ipynb` cells 4–10 for the exploration that established this schema.

## 2. Processing pipeline

The numbered `project ex*.py` files after the scrapers form a linear pipeline (each reads the previous step's Excel output):

| Step | Script | What it does |
| --- | --- | --- |
| Flatten | `project file.py` | Expands each listing's nested `recentlyTransaction.rentList` into one row per transaction (1-to-many), copying the parent listing's columns onto every row (`ex4.py` is a related step that groups `rentList` records by net area) |
| Merge | `project_concat.py` | Concatenates the expanded per-range one-room files into one table |
| Subway flag | `project ex5.py` | Adds `subwayYN` ('O'/'X') from the `subways` field |
| Seoul filter | `project ex6.py` | Keeps rows where `local1` contains 서울 |
| Split by type | `project ex7.py` | Splits into sale (매매) / monthly rent (월세) / jeonse (전세) files — these are the `seoul_oneroom_*.xlsx` files committed in `data/` |
| Geocoding | `project ex8.py` | Reverse-geocodes lat/lng to district/neighborhood via geopy Nominatim (**note:** `geopy` is not in `requirements.txt`; install it separately if re-running this step) |

Two implementation notes, honestly stated:

- The nested JSON strings are parsed with `eval()` (`project file.py`, line 19). This works because the data is self-collected and trusted; `json.loads` or `ast.literal_eval` is the correct choice for anything untrusted.
- The pipeline scripts read/write hardcoded `C:/Users/user/Documents/python/project_ex/...` paths from the original 2024 machine. They are kept unmodified as the record of how the committed datasets were produced; edit the paths to `data/` if you want to re-run them.

## 3. Committed datasets (data dictionary)

`data/` holds 45 Excel files (34 MB), all git-tracked. Row counts below were verified with `pandas.read_excel`.

### Scraped listing datasets (Zigbang pipeline output)

| File | Rows | Cols | Content |
| --- | --- | --- | --- |
| `seoul_oneroom_monthly_rent_data.xlsx` | 39,486 | 20 | One-room monthly-rent listings |
| `seoul_oneroom_lease_data.xlsx` | 12,631 | 24 | One-room jeonse listings |
| `seoul_oneroom_Sale_data.xlsx` | 655 | 24 | One-room sale listings |
| `서울시_아파트_매매.xlsx` | 5,475 | 18 | Apartment sale listings |
| `서울시_아파트_월세.xlsx` | 11,591 | 18 | Apartment monthly-rent listings |
| `서울시_상가_월세.xlsx` | 53,264 | 16 | Commercial monthly-rent listings |
| `서울시_상가_매매.xlsx` | 442 | 16 | Commercial sale listings |
| `서울시_상가_전세.xlsx` | 26 | 16 | Commercial jeonse listings |
| **Total** | **123,570** | | |

### Macro and city datasets (2013–2023 unless noted)

- Bank of Korea: GDP growth, GDP & expenditure (2012–2022), base interest rate, CPI index + inflation, KRW/USD exchange rate, GNI per capita (USD)
- Korea Real Estate Board: monthly sale/jeonse/monthly-rent **price indices** and **median prices**, each for 4 housing types (apartment, detached, row-house, composite) — 24 files
- Seoul Open Data: population migration (in/out by year), population statistics, subway station-catchment land-value index, investment ROI by property type (2014–2023), transaction counts and areas for all property types

## 4. Analysis and visualization scripts

- `src/visualization/` (14 scripts): matplotlib line/panel charts of the macro indicators. Scripts fetch their input from the project's GitHub data mirror (`raw.githubusercontent.com/aaqq8/SteadyEstate/...`) so they run from any clone; identical files sit in `data/` if you prefer offline runs (swap the URL for a local path).
- `src/analysis/` (5 scripts): Seoul-specific analyses — migration flows, subway land-value index, ROI by property type, transaction counts/areas.
- `notebooks/` (4 notebooks):
  - `서울시 아파트 매매.ipynb` — district-level choropleth + marker-cluster map of apartment sale listings; its checked-in output contains the per-district listing counts cited in the README (은평구 403 … 금천구 77, 25 districts, 5,475 listings in that snapshot).
  - `서울시 아파트 월세.ipynb` — same for apartment monthly rent.
  - `원룸, 빌라, 오피스텔 지도.ipynb` — Folium maps for one-room/villa/officetel listings (per sales type), GeoJSON district boundaries from the `southkorea-maps` project.
  - `직방 데이터 전처리 코드.ipynb` — the preprocessing walkthrough (schema exploration, JSON flattening experiments) with outputs preserved.

Rendering notes:

- The Folium maps save to standalone HTML (`m.save(...)`) — open the saved file in a browser.
- Matplotlib scripts call `plt.show()` and do not save files; add `plt.savefig(...)` if you want artifacts. The preview images in `docs/img/` were rendered from the committed data with the same aggregation logic as the corresponding scripts.
- Korean axis labels need a CJK font: `plt.rcParams['font.family'] = 'Malgun Gothic'` on Windows (`AppleGothic` on macOS, `NanumGothic` on Linux).

## 5. Migration insight — computation

The README's headline (net domestic out-migration every year 2013–2023, cumulative −961,881) comes from `data/서울시_인구이동.xlsx`, Seoul row, yearly 총전입 (in) minus 총전출 (out):

| Year | 2013 | 2014 | 2015 | 2016 | 2017 | 2018 | 2019 | 2020 | 2021 | 2022 | 2023 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Net | −100,550 | −87,831 | −137,256 | −140,257 | −98,486 | −110,230 | −49,588 | −64,850 | −106,243 | −35,340 | −31,250 |

## 6. Known cleanup debt

- `real_estate_project-main/` is the original unorganized 2024 workspace (kept for provenance); the curated copies live in `src/`. Several files exist in both places — edit only the `src/` copies.
- A few `" (1)"`-suffixed duplicate scripts remain in `src/visualization/`; the non-suffixed file is canonical.
- `docs/` previously served as an empty placeholder; it now holds this file and the README preview images.
