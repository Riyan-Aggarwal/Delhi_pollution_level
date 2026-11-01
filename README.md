# Diwali, stubble burning and Delhi's air

**Question:** How much of the October–November PM2.5 rise in Delhi comes from Diwali night, and how much from the season's weather?

**Answer, in one line:** Diwali night adds only about 1–4% of the seasonal rise. It is by far the worst night of the autumn (city-wide PM2.5 peaked at 1,331 µg/m³ in 2025), but it lasts about 12 hours, while the roughly +130 µg/m³ seasonal rise lasts two months.

**Website:** https://riyan-aggarwal.github.io/Delhi_pollution_level/ (or open `docs/index.html`, which also works offline)

DA2402 Data Curation and Visualisation, IIT Madras, Project #2 (Delhi slice; Diwalis of 12 Nov 2023, 31 Oct 2024 and 20 Oct 2025).

## What's in this repository

```
notebooks/
  01_collection.ipynb    downloads OpenAQ + Open-Meteo, checks the hand-downloaded CPCB files, robots.txt checks, first look at raw files
  02_cleaning.ipynb      formats, duplicates (incl. across sources), outliers, missing values -> clean table
  03_analysis.ipynb      statistics, weather models, Diwali-night excess, all charts
raw/                     files exactly as downloaded (never edited) - 46 MB
  cpcb/                  36 hourly CSVs: 12 stations x 2023-2025
  openaq/                1,008 daily CSV.gz files for 2025 + station list (JSON)
  openmeteo/             3 JSON files of hourly weather
interim/                 each source parsed into a table, before cleaning
clean/                   delhi_pm25_hourly.parquet (+ .csv): 11 sites x 3 years x 1,848 hours
metadata/
  provenance.csv         every raw file: URL, access time, licence, size, SHA-256
  robots_checks.csv      robots.txt of every host we contacted
  cleaning_log.csv       one line per cleaning step, with rows affected
  data_dictionary.csv    column, dtype, unit, source, allowed range
  station_match_cpcb_openaq.csv, cpcb_site_ids.csv, cpcb_download_log.csv, openaq_*.csv
docs/                    the website (GitHub Pages serves this folder)
```

## Data sources

| Source | Used for | Access | Licence |
|---|---|---|---|
| [CPCB CAAQMS Data Repository](https://airquality.cpcb.gov.in/ccr/#/repository/data) | Hourly PM2.5, 2023–2025 | Hand download (captcha): State Delhi → City Delhi → station → 1H | Government of India public data |
| [OpenAQ](https://docs.openaq.org) API v3 + S3 archive | Station list, 2025 PM2.5 (cross-check) | Script, free API key | OpenAQ terms of use |
| [Open-Meteo Historical Weather](https://open-meteo.com/en/docs/historical-weather-api) | Hourly weather incl. boundary-layer height | Script | CC BY 4.0 |

OpenAQ has no Delhi government-station data from late 2022 to February 2025, which is why CPCB is the main PM2.5 source.

## How to run it

Run the three notebooks in order, top to bottom.

1. Put a free OpenAQ API key in a file called `.env` in the repository root:
   ```
   OPENAQ_API_KEY=your-key
   ```
   In Google Colab, add it under **Secrets** as `OPENAQ_API_KEY` instead. The `.env` file is ignored by git, so the key never gets published.
2. The CPCB files are already in `raw/cpcb/`. To re-download them, follow the steps in Section 5 of `01_collection.ipynb`.
3. Install the packages:
   ```
   pip install pandas numpy scipy statsmodels plotly requests pyarrow python-dotenv
   ```
4. Run `01_collection` → `02_cleaning` → `03_analysis`. Downloads are cached, so re-running is quick.
