# Report: Diwali, stubble burning and Delhi's air

**Our question:** How much of the October–November PM2.5 rise in Delhi comes from Diwali night, and how much from the season's weather?

**Diwalis covered:** 12 Nov 2023, 31 Oct 2024, 20 Oct 2025.

**Dashboard (website):** https://riyan-aggarwal.github.io/Delhi_pollution_level/  
**Code and data:** https://github.com/Riyan-Aggarwal/Delhi_pollution_level

---

## 1. Getting the data (scraping and downloads)

### Weather: Open-Meteo
- We pulled hourly weather from the Open-Meteo Historical Weather API for all three years, 15 September to 30 November each year.
- Variables: temperature, humidity, rain, wind speed, wind direction, surface pressure and boundary-layer height (how deep the layer of air is that the smoke can mix into).
- We used one point in central Delhi (28.61 N, 77.21 E). The model behind Open-Meteo works on a grid several kilometres wide, so all our stations fall in essentially the same weather cell.
- This part went smoothly: all 1,848 hours per year came back, with no gaps.

### PM2.5: OpenAQ
- We used the OpenAQ API (v3, with a free key) to list every monitoring station in and around Delhi. It returned 99 locations. We kept only government monitors (run by CPCB, DPCC, IMD or IITM), as the brief asks.
- We then tried to get hourly PM2.5 for 2023, 2024 and 2025, both from OpenAQ's public archive on AWS and through the API.
- **Problem:** we only got 2025. For every Delhi government station, OpenAQ has no data at all from late 2022 to February 2025. We checked the archive listings for all our stations and also queried the API directly. Both came back empty for October–November 2023 and 2024.
- We downloaded the 2025 data anyway: one file per station per day, 14 September to 1 December 2025, for our 12 stations plus one extra location we suspected was a duplicate. That came to 1,008 daily files; another 19 station-days simply do not exist in the archive (the collection notebook lists them).
- All three sources were accessed on 31 October 2026 (UTC). The collection notebook ends with a summary table of every source: URL, how we got it, access date, licence, number of files and size.

### PM2.5: CPCB official website (to fill the gap)
- Because OpenAQ could not give us 2023 and 2024, we went to the original publisher: the Central Pollution Control Board's CAAQMS Data Repository.
- The repository is behind a captcha, so we did **not** write a scraper for it. We downloaded the files by hand, one per station per year, for 2023, 2024 and 2025.
- Steps we followed: airquality.cpcb.gov.in → Data → CAAQMS Data Repository → solve captcha → State: Delhi, City: Delhi, Station: (each of our 12) → Frequency: 1H → Submit → download the 2023, 2024 and 2025 rows.
- Each file is a CSV with one year of hourly readings: PM2.5, PM10, gases, and the station's own weather sensors.
- We picked 12 stations spread across the city (east, west, north, south, centre), run by different agencies:
  - Anand Vihar, R K Puram, Okhla Phase-2, Punjabi Bagh, Najafgarh and ITO
  - Pusa (DPCC) and Pusa (IITM), kept on purpose because they look like the same site under two names
  - IMD Lodhi Road, Jahangirpuri, Narela and Dwarka Sector 8
- We kept 2025 from both CPCB and OpenAQ on purpose. Having the same year from two sources lets us check whether they agree, and lets us practise matching the same station across two sources that spell its name differently.

### Rules and permissions we checked
- **OpenAQ:** no robots.txt. Its terms require an API key and say to use the official API or exports, not scraping. That is what we did.
- **Open-Meteo:** the API host's robots.txt says `Disallow: /`, which blocks search-engine crawlers. Open-Meteo's terms allow free non-commercial use (education is named) under the CC BY 4.0 licence, up to 10,000 calls a day. We made 3 calls.
- **CPCB:** no robots.txt, but the captcha tells us they do not want automated downloads. That is why we downloaded by hand instead of scraping.

### First look at the raw files
- **OpenAQ files** are in long format (one row per reading per pollutant), every 15 minutes, with times in Indian time. On Diwali day 2025 at Anand Vihar, only 66 of the 96 expected PM2.5 readings were there, and some PM10 values repeated exactly for several readings in a row.
- **CPCB files** are wide (one column per pollutant), hourly, and mostly complete: under 3% of PM2.5 hours missing for most stations in our window. Anand Vihar was the worst, at about 11% in 2023 and 2024. Every column name carries its unit in brackets (`PM2.5 (µg/m³)`, `WS (m/s)`), and the file name only has a site number, not the station name.
  - At first we thought the unit symbols were garbled (`Âµg/mÂ³`), but that turned out to be how the browser's developer view showed the file. When we checked the actual bytes, the file is proper UTF-8 and reads correctly in pandas. Lesson learned: check the file itself, not a preview of it.
- The highest PM2.5 readings (900+ µg/m³) all fall on Diwali nights or in Delhi's severe smog episode of mid-November 2024 (13 and 18 November). So these are real peaks, not errors, and we must not clean them away.
- **Open-Meteo** came back clean and complete. It used the model grid point nearest to our coordinates (28.58 N, 77.19 E, 217 m).

---

## 2. Storing the data

- We used three folders, as the brief asks:
  - `raw/` holds the files exactly as downloaded. We never edit these: every later step reads them and writes its output somewhere else.
  - `interim/` holds each source parsed into a tidy table, before any cleaning: `cpcb_hourly_parsed`, `openaq_pm25_hourly_parsed` and `openmeteo_hourly_parsed`.
  - `clean/` holds the final table, `delhi_pm25_hourly.parquet`, plus a CSV copy.
- Every raw file name carries the source and the date or year, for example `openaq_loc235_2025-10-20.csv.gz` and `cpcb_site301_anand_vihar_2024.csv`.
- **Why Parquet:** it saves each column's type with the data, so dates stay dates and categories stay categories when we read the file back. We checked this: after saving and re-reading, every column type and the shape (60,984 × 20) came back identical. It is also much smaller than CSV. We kept a CSV copy for anyone who wants to open the data in Excel.
- The data dictionary for the final table, with each column's type, unit, source and allowed range, is shown below and saved in `metadata/data_dictionary.csv`.

### Data dictionary (the final table, `clean/delhi_pm25_hourly.parquet`)

60,984 rows (11 sites × 3 seasons × 1,848 hours), 20 columns. Also saved as `metadata/data_dictionary.csv`.

| Column | Meaning | Type | Unit | Source | Allowed range | Observed range | Missing | Note |
|---|---|---|---|---|---|---|---|---|
| `station_key` | Monitoring site (11 sites; the two Pusa monitors merged into 'pusa') | category | – | CPCB station list | one of 11 site names | – | 0.0% |  |
| `ts_ist` | Hour the reading covers (start of the hour) | datetime64[ns] | IST | CPCB timestamp | 15 Sep - 30 Nov, 2023-2025 | – | 0.0% | 1,848 hours per site per year |
| `year` | Season year | int16 | – | derived from ts_ist | 2023-2025 | 2023 to 2025 | 0.0% |  |
| `days_from_diwali` | Days before (-) or after (+) that year's Diwali | int16 | days | derived from ts_ist | -58 to +41 | -58 to 41 | 0.0% | Diwali: 12 Nov 2023, 31 Oct 2024, 20 Oct 2025 |
| `pm25` | Hourly PM2.5 after cleaning; used in all analysis | float64 | µg/m³ | CPCB repository (gaps in 2025 filled from OpenAQ) | > 0, no upper limit | 0.4 to 1901 | 2.4% | Missing hours are gaps longer than 3 hours, left empty on purpose |
| `pm25_raw` | Hourly PM2.5 before outlier removal and gap filling | float64 | µg/m³ | CPCB repository / OpenAQ | any (still contains the errors) | -233 to 1901 | 3.0% | Kept for the before/after comparison |
| `pm25_flag` | Result of the outlier checks for that hour | category | – | derived | ok, impossible_value, stuck, isolated_spike, citywide_high_kept | – | 0.0% | First three were set to missing in pm25 |
| `pm25_source` | Where the cleaned pm25 value came from | category | – | derived | cpcb, openaq, interpolated, missing | – | 0.0% |  |
| `pm10` | Hourly PM10; context only, not used in the analysis | float64 | µg/m³ | CPCB repository | > 0 | 0.6 to 1000 | 3.9% | Never above exactly 1,000: readings look capped by the instrument |
| `st_temp_c` | Air temperature measured at the station | float64 | °C | CPCB repository (station sensor) | 0 - 45 | 2.2 to 50.8 | 19.7% | Check only; some readings above 45 left as recorded |
| `st_rh_pct` | Relative humidity measured at the station | float64 | % | CPCB repository (station sensor) | 0 - 100 | 14.2 to 99 | 19.7% | Check only |
| `st_wind_kmh` | Wind speed measured at the station | float64 | km/h | CPCB repository (station sensor, converted from m/s) | 0 - 60 | 0.4 to 102 | 19.9% | Check only; 40% of readings sit on the sensor minimum, some above 60 |
| `st_wind_dir_deg` | Wind direction measured at the station | float64 | degrees | CPCB repository (station sensor) | 0 - 360 | 1 to 360 | 22.3% | Check only |
| `temp_c` | Air temperature 2 m above ground | float64 | °C | Open-Meteo Historical Weather API | 0 - 45 | 11.2 to 38.7 | 0.0% | Same for every site in a given hour |
| `rh_pct` | Relative humidity 2 m above ground | int64 | % | Open-Meteo Historical Weather API | 0 - 100 | 22 to 99 | 0.0% |  |
| `rain_mm` | Rain in that hour | float64 | mm | Open-Meteo Historical Weather API | 0 - 50 | 0 to 16.1 | 0.0% |  |
| `wind_kmh` | Wind speed 10 m above ground | float64 | km/h | Open-Meteo Historical Weather API | 0 - 60 | 0 to 27.6 | 0.0% |  |
| `wind_dir_deg` | Direction the wind blows from (0 = north) | int64 | degrees | Open-Meteo Historical Weather API | 0 - 360 | 1 to 360 | 0.0% |  |
| `pressure_hpa` | Air pressure at the surface | float64 | hPa | Open-Meteo Historical Weather API | 950 - 1010 | 976 to 997 | 0.0% |  |
| `blh_m` | Boundary-layer height: depth of the air layer pollution mixes into | float64 | m | Open-Meteo Historical Weather API | 0 - 3000 | 10 to 2790 | 0.0% |  |

**Notes on the dictionary**
- **Missing values in `pm25`:** 2.4% of hours, all gaps longer than 3 hours that we deliberately left empty.
- **Station weather columns (`st_*`):** 20–22% of hours are missing. Two of them break their allowed range: 11 temperature readings above 45 °C (up to 50.8 °C, at Najafgarh and Punjabi Bagh) and 57 wind readings above 60 km/h (up to 102 km/h, all at Jahangirpuri). These come from the stations' own weather sensors, which we found unreliable (40% of wind readings sit on the sensor's minimum value). We only used these columns to check Open-Meteo, never in the analysis, so we left them as recorded and flag them here.
- **`pm10` never goes above exactly 1,000 µg/m³** (3 hours sit at that value), which suggests the instruments cap readings there. This is one reason we list possible capping as a limitation.

- Raw data is 46 MB in total, under the 50 MB limit, so it is included in the submission.

---

## 3. Cleaning

All of this is in notebook `02_cleaning.ipynb`. Every step, with what we did, the number of rows it touched and why, is written to `metadata/cleaning_log.csv` (24 steps).

### Fixing formats and inconsistencies
- **Column names:** CPCB names carry units in brackets (`PM2.5 (µg/m³)`). We renamed the 7 columns we kept to short names (`pm25`, `st_wind_ms` and so on) and moved the units into the data dictionary. We dropped the other 18 columns (gases, benzene-type compounds, solar radiation), because they are outside our question.
- **Dates:** CPCB timestamps were text. We parsed all 315,648 of them, and 0 failed.
- **Number strings:** we checked every value column for numbers stored as text. There were none (0 in each column).
- **Time zones:** OpenAQ times carry a `+05:30` offset; CPCB times have none but are Indian time. We converted OpenAQ to plain IST so the two line up, which affected 80,525 readings.
- **Time step:** OpenAQ reports every 15 minutes, CPCB every hour. We averaged OpenAQ's readings into hours.
- **Units:** CPCB measures wind in m/s and Open-Meteo in km/h. We converted CPCB wind to km/h (× 3.6), which affected 48,865 values.
- **Station names:** OpenAQ adds a number to every name (`Anand Vihar, New Delhi - DPCC-3379575`) and sometimes says "New Delhi" where CPCB says "Delhi". We cleaned all 13 OpenAQ names so they could be compared with CPCB's spelling.

### Duplicates
- **Exact duplicates:** we checked for repeated rows (same station and hour in CPCB, same sensor and time in OpenAQ). There were 0 in both.
- **The same station in two sources:** this was the interesting one. Names alone were not safe ("Pusa, Delhi - IMD" in OpenAQ is "Pusa, Delhi - IITM" in CPCB), so we compared values. Our rule: an OpenAQ location and a CPCB station are the same if their hourly PM2.5 agrees within 0.5 µg/m³ in at least 80% of the hours both have.
  - 11 of 12 stations passed this rule; 8 agreed in 99–100% of hours.
  - The twelfth, Pusa (IITM), agreed in only 16% of hours but was still clearly the same station (correlation 0.98, mean difference 5.5 µg/m³), so we accepted it under a second, looser rule.
  - **What we learned:** CPCB's hourly value is simply the average of OpenAQ's four 15-minute readings. OpenAQ republishes CPCB's data; it does not measure separately.
  - We kept CPCB and dropped the 20,120 duplicate OpenAQ hours. Where CPCB had a gap but OpenAQ had a reading, we used OpenAQ's value (123 hours in the final table).
  - One extra OpenAQ location (Lodhi Road, IITM) matched none of our stations and was dropped.
- **One site, two names:** the two Pusa monitors sit at the same coordinates. Their readings are similar (correlation 0.85–0.94) but not copies, so they are two instruments measuring the same air. Keeping both would count Pusa twice in city averages, so we merged them into one site by averaging. That left 11 sites.

### Outliers
We used fixed rules, then looked at every flagged value before deciding:

| Check | Rule | What we decided |
|---|---|---|
| Impossible values | PM2.5 of 0 or below | **Error**, 55 hours removed. These included negatives down to −233 and runs of exactly 0.00 at ITO and Lodhi Road in 2025. |
| Stuck sensor | Same value 4 or more hours in a row | **Error**, 43 hours removed. Some were on Diwali nights (Punjabi Bagh stuck at 364 for six hours on 1–2 Nov 2024). The level looked believable, but a flat line hides the real rise and fall of the night. |
| Isolated spike | More than 3× the station's own 24-hour median **and** more than 3× the median of the other stations that hour | **Error**, 28 hours removed. Only one station saw it, which points to a local fault. |
| City-wide high | More than 3× its own median, but other stations high too | **Real, kept** (154 hours). These are Diwali nights and smog episodes, exactly what we are studying. |

- **A rule we changed our minds about:** we started by treating anything above 1,000 µg/m³ as "outside the instrument's range". The data proved us wrong. On Diwali night 2025, nine stations went above 1,000 in the same hours (the highest was 1,901). A fault does not hit nine separate monitors at once, so we dropped that rule and let the city-wide test decide.
- **A physics check:** PM2.5 is part of PM10, so it should never be much bigger. 70 hours had PM2.5 more than 10 µg/m³ above PM10, mostly at Jahangirpuri and Pusa. They come from separate instruments and we cannot tell which one is wrong, so we flagged them but did not change them. We do not use PM10 in the analysis.

### Missing values
- After removing errors, about 3% of PM2.5 hours were missing. Most stations lost under 4%. The worst were Anand Vihar (11–12% in 2023–2024) and Lodhi Road (16% in 2025).
- **Pattern:** most gaps are short. 233 gaps were a single hour and 111 were 2–3 hours; only 8 lasted more than a day.
- **Tests:**
  - **Hour of day:** a chi-square test found no link between missingness and hour (p = 0.21).
  - **Station:** a strong link (p ≈ 0).
  - **Air quality:** the city was cleaner in hours when a station was missing (median 75 vs 119 µg/m³, Mann–Whitney p ≈ 0). That is because gaps are most common in September (5.3% of hours) and rarest in November (1.8%).
- **Our conclusion: MAR (missing at random).** The gaps depend on things we can see (which station, what time of season), not on the missing value itself. There is no sign that monitors fail *because* the air is very dirty.
- **What we did:**
  - Gaps of up to 3 hours were filled by straight-line interpolation (482 hours), because PM2.5 changes smoothly from hour to hour.
  - Longer gaps were left empty (1,487 hours, 2.4% of the table). Filling whole days would mean inventing data, and could hide or create a Diwali peak.
  - Daily or nightly averages are only computed when at least 75% of the hours are present.

### Checking the weather data
- We compared Open-Meteo with the CPCB stations' own weather sensors. Temperature agreed well (correlation 0.87) and humidity too (0.84).
- Station wind speed was much lower than the model's (1.7 vs 7.1 km/h on average). 40% of station wind readings sit on a single value, the sensor's minimum, so the station anemometers are not reliable.
- This is why we use Open-Meteo for the weather analysis.
- Some station weather values are also out of range (a 50.8 °C temperature, a 102 km/h wind). We left them unchanged and do not use them.

### The final table
- `clean/delhi_pm25_hourly.parquet` has 60,984 rows: 11 sites × 3 years × 1,848 hours.
- Where the PM2.5 values came from:
  - 58,892 straight from CPCB
  - 123 from OpenAQ
  - 482 interpolated
  - 1,487 left missing
- Each row also carries the original value (`pm25_raw`), an outlier flag and the weather for that hour.

---

## 4. Analysis

All of this is in notebook `03_analysis.ipynb`.

### City-wide PM2.5
- For each hour we averaged the 11 sites, but only when at least 8 of them reported. That way one station's gap cannot swing the city value.
- "Diwali night" means 18:00 on Diwali day to 06:00 the next morning.

### The seasonal rise
- Late September (15–30 Sep) averaged 47, 52 and 48 µg/m³ in 2023, 2024 and 2025.
- October–November averaged 180, 184 and 174.
- So the rise was **+133, +132 and +126 µg/m³**, almost the same every year.
- 89–95% of October–November days were above India's 24-hour standard of 60 µg/m³.

### How much of that is weather?
- We first looked at which weather moves with daily PM2.5 (Spearman correlation):
  - temperature: −0.74
  - rain: −0.46
  - wind: −0.36
  - humidity: −0.30
  - mixing-layer height: −0.21
- Temperature is by far the strongest. But temperature also falls steadily through the season, at the same time as stubble burning in Punjab and Haryana and winter heating increase, and we have no data on those. So we could not treat temperature as pure "weather".
- We therefore fitted two regression models of daily PM2.5 (on the log scale, leaving out Diwali and the day after):
  - **Ventilation only** (mixing height and wind): R² = 0.20. This model predicts only 3%, 17% and 41% of the rise in 2023, 2024 and 2025.
  - **Full weather** (adding temperature and humidity): R² = 0.63. This one predicts 73%, 72% and 122% of the rise. Over 100% means it over-shoots in 2025.
- We also tested each model on a year it had not seen. Trained on two years and tested on the third, the full model scored R² 0.44–0.63; the ventilation model scored −0.23 to 0.18.
- **Our reading:** the honest answer for "weather" is a range. Air that cannot carry pollution away (low mixing height, calm wind) explains a modest part of the rise. The rest moves with the calendar, and our data cannot cleanly split colder air from more smoke arriving at the same time of year.

### Diwali night
We compared each Diwali night with the nights 2–7 days before and after, and with what that night's weather predicts:

| | 2023 | 2024 | 2025 |
|---|---|---|---|
| Diwali night average (µg/m³) | 463 | 382 | 785 |
| Neighbouring nights | 271 | 209 | 142 |
| Expected from that night's weather | 258 | 110 | 94 |
| Excess over neighbours | +192 | +174 | +643 |
| Times the weather-expected level | 1.8× | 3.5× | 8.4× |
| Hourly peak (city-wide) | 679 | 650 | 1,331 |

- **Turning the excess into a share of the season:** Diwali night is 12 hours long and October–November is 1,464 hours. So we spread the excess over the season: excess × 12 / 1,464.
  - Using the smaller of the two excess estimates, Diwali night adds **+1.6, +1.4 and +5.3 µg/m³** to the October–November average.
  - That is **1.2%, 1.1% and 4.2%** of the seasonal rise.
  - Using the larger estimate changes this only slightly (up to 1.3%, 1.7% and 4.5%).
- **By station:** the Diwali jump ranged from −71 to +421 µg/m³ in 2023 and from +152 to +1,003 in 2025. The two stations on the city's outer edge, Narela and Najafgarh, barely rose in 2023 and 2024.

---

## 5. Charts and the website

- We made 26 interactive charts with Plotly. The website shows 22 of them: the before/after cleaning chart in Data and methods, and 21 results charts in five themes: the season, day and night, Diwali night, weather and stations. It opens with the data itself: the three sources and a preview of the clean table.
- The data-quality charts (8, 9, 18, 19, 20) stay in the notebook. On the website, only the before/after chart (8) is kept, in the Data and methods section, because the rubric asks for at least one before/after plot.
- After going through the course's visualisation lectures and the intro deck, we revised several charts:
  - **Chart 1** became a strip plot: every day is a dot, with a median tick. A box plot hides the shape of the data (lecture 6).
  - **Chart 2** became small multiples: one panel per year, on the same axes (lecture 7).
  - **Charts 4 and 12** got labels at the ends of the lines instead of a legend (Gestalt proximity, lecture 8).
- **Colours:** the three years use one blue in three shades (light to dark), plus dotted, dashed and solid lines, so they stay distinguishable in greyscale. Orange is used for one thing only: Diwali. We checked the colours with a colour-blindness simulator.
- **Gestalt choices:**
  - **Proximity:** the three years sit side by side in each group.
  - **Similarity:** the same colour means the same year in every chart.
  - **Alignment:** the three weather panels share one y-axis.
  - **One highlight colour:** orange is used only for the point of each chart.
- **Before/after cleaning:** Chart 8 shows ITO in late November 2025, with the zero readings that cleaning removed. Chart 9 shows how closely OpenAQ and CPCB agree.

| # | Chart | Type | Question it answers |
|---|---|---|---|
| 1 | Daily PM2.5 by month | Strip plot (univariate) | How big is the seasonal shift? |
| 2 | Daily PM2.5 through the season | Line, one panel per year | When does the rise happen, and where is Diwali? |
| 3 | Daily PM2.5 vs mixing height, wind, temperature | Scatter (bivariate) | Which weather goes with dirty air? |
| 4 | Hourly PM2.5 around Diwali | Line | How big and how long is the Diwali spike? |
| 5 | Diwali night vs neighbours vs weather-expected | Grouped bars | Is Diwali night more than the weather explains? |
| 6 | Observed rise vs weather-predicted vs Diwali | Grouped bars | **The answer to the question** |
| 7 | Diwali excess by station | Dot plot | Where in the city was it worst? |
| 8 | ITO before and after cleaning | Line | What did cleaning change? |
| 9 | CPCB vs OpenAQ agreement | Bar | Are the two sources the same data? |
| 10 | Calendar of daily PM2.5 | Heatmap | When does each season turn dark? (first "very poor" day: 13, 17, 18 Oct) |
| 11 | Days in each official AQI band | 100% bar | How bad is each month? (November: 98% very poor or severe) |
| 12 | Average PM2.5 by hour of day | Line | Daily rhythm (November: 296 at 1 am vs 153 at 4 pm) |
| 13 | Diwali night 2025, station × hour | Heatmap | When did each station peak? (all between 1 am and 5 am) |
| 14 | PM2.5 by wind direction | Polar bar | Which winds bring dirty air? (west 206, north-west 157) |
| 15 | Map of the 11 stations | Map | Where is it worst? (Jahangirpuri 216, Anand Vihar 213; Lodhi Road 135) |
| 16 | Station ranking by year | Dot plot | Is the ranking stable? (rank correlation 0.92–0.95) |
| 17 | Model prediction vs actual | Scatter | How well does the weather model fit? (R² 0.63) |
| 18 | Model vs station temperature | 2-D histogram | Is the modelled weather right? (r = 0.87, stations 2 °C warmer) |
| 19 | Missing hours by station and day | Heatmap | Where were the gaps? (16 full days, 11 at Lodhi Road) |
| 20 | Errors removed by station | Bar | Where did the errors come from? (Lodhi Road 38, ITO 30 of 126) |
| 21 | Every hour of every season | Heatmap (hour × day) | When is the air dirtiest? (night bands from late October; Diwali 2025 the darkest column) |
| 22 | 24-hour rose, after Florence Nightingale | Polar area chart | How does the daily cycle grow? (November peak 296 at 1 am; wedge area shows the value) |
| 23 | Week-by-week spread | Ridgeline | How does the distribution shift? (weekly median 42 → 280 µg/m³) |
| 24 | Each station, September to November | Slope chart | Which station rises most? (Punjabi Bagh +226; Lodhi Road least, +141) |
| 25 | Diwali night 2025 on the map | Animated map, after John Snow and Hans Rosling | How did the smoke spread? (all stations peak 1–5 am; Najafgarh 1,901) |
| 26 | Wind speed at every hour of every season | Heatmap (hour × day), aqua scale | When is the air still? (calmest around 1 am, 5.8 km/h; windiest around 3 pm, 9.7 km/h; 35% of November nights below 5 km/h) |

- **Website (dashboard):** https://riyan-aggarwal.github.io/Delhi_pollution_level/. It is a static web page (`docs/index.html`) hosted on GitHub Pages, and it doubles as our presentation. It follows the rubric's order: introduction, data and methods, results, conclusion, limitations, then team contributions. The arrow keys step through it one chart at a time, and a button in the top bar switches between light and dark mode. The charts and the Plotly library are saved inside `docs/assets/`; only the two maps need an internet connection, for their background tiles.
- **Charts in one file:** `dashboard_figures.pdf` in the repository has the dashboard's charts with their captions, for printing (it was made before the wind chart, 26, was added).

---

## 6. Conclusion

**Diwali night accounts for about 1–4% of Delhi's October–November PM2.5 rise. The other 96–99% comes with the season.**

- Diwali night is the worst single event of the autumn. City-wide PM2.5 peaked at 679, 650 and 1,331 µg/m³, and the night averaged 1.8 to 8.4 times what its weather predicted.
- But it lasts about 12 hours, while the seasonal rise of about 130 µg/m³ lasts two months.
- Weather explains between a small share of that seasonal rise (ventilation alone, 3–41%) and nearly all of it (with temperature, 72–122%). Our data cannot separate colder, stiller air from the stubble smoke and heating that arrive at the same time of year.

## 7. Limitations

- **No stubble-burning data.** We did not use any crop-fire data, so "the season" mixes weather, stubble smoke, heating and traffic.
- **Modelled weather at one point.** Open-Meteo weather is modelled on a grid several kilometres wide and we used one point for the whole city. The stations' own wind sensors were not good enough to replace it.
- **Only three Diwalis.** That is too few for a precise average effect, and 2025 behaved very differently from 2023 and 2024.
- **Possible capping before 2025.** No PM2.5 reading in 2023–2024 goes above about 1,000 µg/m³, and PM10 never goes above exactly 1,000.0, but in 2025 nine stations went above 1,000 together. Older peaks may have been capped by the instruments, which would make the 2023–2024 Diwali excess an underestimate.
- **Station choice.** We used 12 of the 59 Delhi stations CPCB lists, chosen to cover the whole city.
- **Association, not cause.** Our regressions show which weather goes with dirtier air; they do not prove what causes it.

## 8. Attributions

**Data**
- Central Pollution Control Board (CPCB), CAAQMS Data Repository, https://airquality.cpcb.gov.in/ccr/#/repository/data. Government of India public data; no licence is stated on the repository page.
- OpenAQ, API v3 (https://api.openaq.org) and AWS open-data archive (https://openaq-data-archive.s3.amazonaws.com). Used under OpenAQ's terms of use, which require attribution to OpenAQ.
- Open-Meteo Historical Weather API, https://open-meteo.com/en/docs/historical-weather-api. CC BY 4.0, free non-commercial use.

**Software:** Python, pandas, NumPy, SciPy, statsmodels, Plotly, requests, pyarrow, Jupyter.

**Terms we quoted**
- Open-Meteo terms: "You may only use the free API services for non-commercial purposes", with limits of "less than 10'000 API calls per day, 5'000 per hour and 600 per minute".
- OpenAQ terms of use: users must use "provided APIs, export functions, or other officially sanctioned methods for accessing data"; scraping is not allowed; "attribution to OpenAQ as the source data is also required".
- robots.txt checked on 31 Oct 2026 (UTC):
  - `api.openaq.org`: none published
  - `openaq-data-archive.s3.amazonaws.com`: none published
  - `archive-api.open-meteo.com`: `User-agent: * / Disallow: /`, which applies to crawlers, not documented API use
  - `open-meteo.com`: `Allow: /`
  - `airquality.cpcb.gov.in`: none published, but the site has a captcha, so we downloaded by hand

## 9. Contributions

| Member | What they did |
|---|---|
| Riyan Aggarwal | Data visualisation |
| Shaurya Gupta | Data collection |
| Prakhar Srivastava | Data cleaning |
