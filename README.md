# Diwali, stubble burning and Delhi's air

Course project for **DA2402 Data Curation and Visualisation** at IIT Madras (2026).

**Dashboard:** https://riyan-aggarwal.github.io/Delhi_pollution_level/
**All charts in one file:** [`dashboard_figures.pdf`](dashboard_figures.pdf)

## Question

How much of the October–November PM2.5 rise in Delhi comes from Diwali night, and how much from the season's weather?

We looked at 11 monitoring sites in Delhi over three autumns (15 Sep – 30 Nov), around the Diwalis of 12 Nov 2023, 31 Oct 2024 and 20 Oct 2025.

## Answer in short

Diwali night adds only about **1–4%** of the seasonal rise. It is by far the worst night of the autumn (city-wide PM2.5 peaked at **1,331 µg/m³** in 2025), but it lasts about 12 hours, while the roughly **+130 µg/m³** seasonal rise lasts two months.

## Data

| Source | What we took | How | Licence |
|---|---|---|---|
| [CPCB CAAQMS Data Repository](https://airquality.cpcb.gov.in/ccr/#/repository/data) | Hourly PM2.5, 2023–2025 | Downloaded by hand (the site has a captcha) | Government of India public data |
| [OpenAQ](https://docs.openaq.org) API v3 + S3 archive | Station list, 2025 PM2.5 as a cross-check | Python script, free API key | OpenAQ terms of use |
| [Open-Meteo Historical Weather](https://open-meteo.com/en/docs/historical-weather-api) | Hourly weather, including mixing-layer height | Python script | CC BY 4.0 |

## Repository layout

```
notebooks/
  01_collection.ipynb    getting the data
  02_cleaning.ipynb      cleaning and joining it into one table
  03_analysis.ipynb      analysis and all the charts
raw/                     files exactly as downloaded (never edited)
interim/                 each source read into a table, before cleaning
clean/                   delhi_pm25_hourly.parquet, the final table
metadata/
  cleaning_log.csv       every cleaning step, the rows it affected and why
  data_dictionary.csv    every column of the final table
docs/                    the dashboard website
dashboard_figures.pdf    every dashboard chart with its caption
```

## How to run

1. Install the packages:
   ```
   pip install pandas numpy scipy statsmodels plotly requests pyarrow python-dotenv
   ```
2. Get a free OpenAQ API key and put it in a file called `.env` in this folder (it is not uploaded to GitHub):
   ```
   OPENAQ_API_KEY=your-key
   ```
3. Run the three notebooks in order. The CPCB files are already in `raw/cpcb/`.

To view the dashboard on your own computer, run `python -m http.server` inside `docs/` and open http://localhost:8000. The maps need this, because they do not load when `index.html` is opened directly as a file.

## Team

| Member | Work |
|---|---|
| Riyan Aggarwal | Data visualisation |
| Shaurya Gupta | Data collection |
| Prakhar Srivastava | Data cleaning |
