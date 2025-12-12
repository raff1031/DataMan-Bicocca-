# F1 Data Management Project

## Overview

This project implements a comprehensive F1 (Formula 1) data management pipeline for analyzing racing data across two technical regulation cycles: 2017-2021 and 2022-2025. The system combines web scraping from the official Formula 1 website with the FastF1 and OpenF1 APIs to collect, store, and analyze race timing data.

## Authors

- Romano Raffaele (romano.raff10@gmail.com)
- Ilaria Mato (Ilaria.mato.02@gmail.com)

## Project Structure

The main notebook `f1_dataMan_v3.ipynb` is organized into the following sections:

### 1. Dependencies and Imports

```python
import pandas as pd
import fastf1 as ff1
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from bs4 import BeautifulSoup
import time
import re
import requests
import json
import os
```

**Required packages:**
- `pandas`: Data manipulation and analysis
- `fastf1`: Official F1 timing data API wrapper (historical data)
- `selenium`: Browser automation for web scraping
- `beautifulsoup4`: HTML parsing
- `requests`: HTTP requests for OpenF1 API

### 2. Cache Configuration

The project implements a two-tier caching strategy:

1. **FastF1 Cache**: Stores downloaded telemetry and timing data locally to avoid repeated API calls
2. **JSON Cache**: Stores scraped race calendar data for quick access

```python
cache_dir = 'f1_cache'
os.makedirs(cache_dir, exist_ok=True)
ff1.Cache.enable_cache(cache_dir)
```

### 3. Data Acquisition: Web Scraping

The `scrape_f1_races(years)` function extracts race calendar information from formula1.com.

**Technical Implementation:**

1. **Browser Automation**: Uses Selenium WebDriver in headless mode for JavaScript-rendered content
2. **Cookie Handling**: Automatically dismisses cookie consent banners using WebDriverWait
3. **Content Loading**: Scrolls the page to trigger lazy-loaded content
4. **Data Extraction**: 
   - Uses regex pattern `ROUND\s+(\d+)` to extract round numbers from link text
   - Extracts GP names from URL slugs (e.g., `/racing/2024/bahrain` -> "Bahrain")
5. **Deduplication**: Tracks seen rounds to prevent duplicate entries

**Output Format:**
```python
[(year, round_number, gp_name), ...]
# Example: [(2024, 1, 'Bahrain'), (2024, 2, 'Saudi Arabia'), ...]
```

### 4. Data Acquisition: FastF1 API

The `collect_fastest_laps_fastf1(races, save_file)` function retrieves detailed lap timing data.

**Technical Implementation:**

1. **Session Loading**: Loads only lap data (telemetry, weather, and messages disabled for performance)
2. **Fastest Lap Extraction**: Uses `session.laps.pick_fastest()` to identify the fastest lap
3. **Time Formatting**: Converts timedelta objects to readable format (M:SS.mmm)
4. **Error Handling**: Gracefully handles missing data or API failures
5. **Incremental Saving**: Saves progress to CSV after each race to prevent data loss

**Data Fields Collected:**
- Year
- Round
- GP (Grand Prix name)
- Driver (three-letter abbreviation)
- Team
- LapTime (formatted as M:SS.mmm)
- LapTime_sec (total seconds for numerical analysis)

### 5. Data Acquisition: OpenF1 API

The `get_fastest_laps_openf1(years, save_file)` function provides an alternative data source for recent seasons (2023+).

**Technical Implementation:**

1. **Rate Limiting Protection**: 
   - 1-second delay between requests
   - Exponential backoff on 429 (rate limit) responses
   - Automatic retry with configurable attempts
2. **Session Discovery**: Queries `/v1/sessions` endpoint filtered by year and session type
3. **Lap Data Retrieval**: Fetches lap times from `/v1/laps` endpoint
4. **Fastest Lap Calculation**: Client-side minimum calculation on lap_duration field

**API Endpoints Used:**
- `https://api.openf1.org/v1/sessions?year={year}&session_type=Race`
- `https://api.openf1.org/v1/laps?session_key={key}&is_pit_out_lap=false`

### 6. Data Storage

The project exports data in multiple formats:

1. **JSON Format** (race calendar):
   - `races_cycle1.json`: 2017-2021 race calendar
   - `races_cycle2.json`: 2022-2025 race calendar

2. **CSV Format** (timing data):
   - `fastest_laps_cycle1.csv`: Fastest laps for 2017-2021
   - `fastest_laps_cycle2.csv`: Fastest laps for 2022-2025

### 7. Data Quality Considerations

**Known Limitations:**

1. **Historical Data Availability**: FastF1 has varying data quality for older seasons (pre-2018)
2. **OpenF1 Coverage**: OpenF1 API only provides data from 2023 onwards
3. **API Stability**: FastF1 relies on multiple backend APIs (F1 official, Ergast) which may have intermittent availability

**Validation:**
- Expected race counts per season are documented for verification
- Scraped data can be cross-referenced with official F1 records

## Technical Cycles

The project analyzes two distinct technical regulation eras:

### Cycle 1: 2017-2021 (Hybrid Era Peak)
- Characterized by dominant Mercedes performance
- Stable technical regulations with incremental changes
- Total races: approximately 100 across 5 seasons

### Cycle 2: 2022-2025 (Ground Effect Era)
- New technical regulations introducing ground effect aerodynamics
- Increased competitive parity
- Expansion to 24 races per season

## Usage

### Initial Setup

1. Install dependencies:
```bash
pip install pandas fastf1 selenium beautifulsoup4 requests
```

2. Ensure Chrome WebDriver is installed and in PATH

### Running the Pipeline

1. Execute all cells in order
2. Scraping cells will populate JSON files
3. FastF1/OpenF1 cells will populate CSV files
4. Review output DataFrames for analysis

### Loading Cached Data

```python
# Load race calendar
with open('races_cycle1.json', 'r') as f:
    races = [tuple(x) for x in json.load(f)]

# Load timing data
df = pd.read_csv('fastest_laps_cycle1.csv')
```

## File Structure

```
Data Managament/
├── f1_dataMan_v3.ipynb      # Main analysis notebook
├── README.md                 # This documentation
├── f1_cache/                 # FastF1 cache directory
├── races_cycle1.json         # Scraped calendar 2017-2021
├── races_cycle2.json         # Scraped calendar 2022-2025
├── fastest_laps_cycle1.csv   # Timing data 2017-2021
└── fastest_laps_cycle2.csv   # Timing data 2022-2025
```

## References

- FastF1 Documentation: https://docs.fastf1.dev/
- OpenF1 API Documentation: https://openf1.org/
- Formula 1 Official Website: https://www.formula1.com/

## License

This project is developed for educational purposes as part of a Data Management course.
