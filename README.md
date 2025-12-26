# F1 Data Management Project

## Project Overview

This project analyzes Formula 1 qualifying performance data from **2017-2025** to study the impact of the **2022 ground effect regulations** and **budget cap** on car performance and field competitiveness.

### Research Question

> _How did ground effect regulations and budget cap change car performance in Formula 1?_

---

## Data Sources Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA ACQUISITION LAYER                      │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Jolpica API  │  │ Wikipedia API│  │  OpenAI API  │           │
│  │ (REST)       │  │ (Scraping)   │  │ (LLM)        │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Qualifying   │  │  Season      │  │ Regulations  │           │
│  │ Results      │  │  Summaries   │  │ Extraction   │           │
│  │ 3853 records │  │  9 seasons   │  │ 46 records   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ DATA INTEGRATION│
                    │  (Merge/Join)   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    SQLite       │
                    │    Database     │
                    │ (f1_data_       │
                    │ management.db)  │
                    └─────────────────┘
```

---

## Database Schema

The project uses **SQLite** for data storage. See [database_schema.md](database_schema.md) for full schema details.

| Table | Description |
|-------|-------------|
| `seasons` | F1 season years |
| `teams` | Teams per season with WCC and ATR | 
| `drivers` | Drivers with WDC, points and wins | 
| `events` | GPs with pole position | 
| `regulations` | FIA regulations (from OpenAI) | 
| `circuit_performance` | Pre/Post 2022 comparison (9 stable circuits) |

---

## Prerequisites

```bash
pip install pandas numpy requests openai matplotlib sqlite3
```

---

## Execution Sequence

### STEP 0: Environment Setup

```powershell
# Set OpenAI API key (PowerShell)
$env:OPENAI_API_KEY="sk-proj-your-key-here"

# Verify it's set
echo $env:OPENAI_API_KEY
```

### STEP 1: Extract Regulations via OpenAI (Optional)

> ⚠️ Skip if `f1_regulations_openai.csv` already exists

```bash
python run_extract.py
```

**What it does:**

1. Fetches Wikipedia summaries for each F1 season (2017-2025)
2. Sends each summary to GPT-4o-mini with prompt to extract regulations
3. Parses JSON response and saves to CSV

**Output:** `f1_regulations_openai.csv` (37 regulations)

---

### STEP 2: Run Notebook Cells in Order

Open `progetto_data_man_1.1.ipynb` and run cells in this exact sequence:

| Cell # | Section            | What it does                                        |
| ------ | ------------------ | --------------------------------------------------- |
| 1      | Imports            | Load libraries (pandas, numpy, requests, etc.)      |
| 2      | Jolpica API        | Fetch 3853 qualifying results with pagination       |
| 3      | Wikipedia API      | Scrape season summaries for 9 years                 |
| 4      | Keyword Extraction | Parse summaries for regulation keywords             |
| 5      | Load Regulations   | Load `f1_regulations_openai.csv` (OpenAI extracted) |
| 6      | ATR Data           | Calculate wind tunnel/CFD allocations               |
| 7      | WDC/WCC Data       | Fetch championship winners per season               |
| 8      | Data Preparation   | Convert times, calculate gaps                       |
| 9      | Data Integration   | Merge all datasets                                  |
| 10     | SQLite Storage     | Store data in `f1_data_management.db`               |
| 11     | Data Quality       | ISO 25012 assessment                                |
| 12     | Circuit Analysis   | Pre vs Post 2022 pole time comparison (9 circuits)  |
| 13     | Field Spread       | Visualization of field competitiveness              |

---

## Code Examples & Generalizations

### 1. API Data Fetching with Pagination

```python
def fetch_paginated_api(base_url, year, limit=100):
    """
    Generic pattern for fetching paginated API data.
    Used for Jolpica API to get all qualifying results.
    """
    all_results = []
    offset = 0

    while True:
        url = f"{base_url}/{year}/qualifying.json?limit={limit}&offset={offset}"
        response = requests.get(url)
        data = response.json()

        items = data['MRData']['RaceTable']['Races']
        if not items:
            break

        all_results.extend(items)
        offset += limit

        if offset >= int(data['MRData']['total']):
            break

        time.sleep(0.3)  # Rate limiting

    return all_results
```

### 2. Wikipedia API Scraping

```python
def scrape_wikipedia_summary(page_title):
    """
    Generic pattern for scraping Wikipedia page summaries.
    Uses Wikipedia REST API for structured data.
    """
    url = f"https://en.wikipedia.org/api/rest_v1/page/summary/{page_title}"
    headers = {'User-Agent': 'YourProject/1.0 (Educational)'}

    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json().get('extract', '')
    return None
```

### 3. LLM-Based Data Extraction (OpenAI)

```python
def extract_structured_data_via_llm(text, schema, model='gpt-4o-mini'):
    """
    Generic pattern for using LLM to extract structured data from text.
    Returns JSON that matches the provided schema.
    """
    import openai
    import os

    openai.api_key = os.getenv('OPENAI_API_KEY')

    prompt = f"""Extract data from this text and return as JSON array:
    Schema: {schema}
    Text: {text}
    Return only valid JSON."""

    response = openai.chat.completions.create(
        model=model,
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0.3
    )

    return json.loads(response.choices[0].message.content)
```

### 4. Era Classification (Vectorized)

```python
# Pattern: Vectorized conditional column creation
import numpy as np

# Use np.where (faster than lambda):
df['Era'] = np.where(df['Year'] < 2022, 'Pre-2022', 'Post-2022')
```

### 5. SQLite Storage Pattern

```python
import sqlite3

def store_dataframe_to_sqlite(df, db_path, table_name):
    """
    Generic pattern for storing pandas DataFrame to SQLite.
    """
    conn = sqlite3.connect(db_path)
    df.to_sql(table_name, conn, if_exists='replace', index=False)
    conn.close()
    return len(df)
```

### 6. Data Quality Assessment (ISO 25012)

```python
def assess_data_quality(df, name, expected_records=None):
    """
    Generic pattern for ISO 25012 data quality assessment.
    Returns scores for 5 dimensions.
    """
    total_cells = df.size

    # 1. COMPLETENESS: % non-null values
    completeness = (1 - df.isnull().sum().sum() / total_cells) * 100

    # 2. ACCURACY: expected vs actual
    accuracy = min(len(df) / expected_records * 100, 100) if expected_records else 100

    # 3. CONSISTENCY: duplicate check
    consistency = (1 - df.duplicated().sum() / len(df)) * 100

    # 4. TIMELINESS: data recency
    timeliness = 100 if df['Year'].max() >= 2025 else 90

    # 5. VALIDITY: type/format check
    validity = 100  # Implement specific checks

    return {
        'Completeness': completeness,
        'Accuracy': accuracy,
        'Consistency': consistency,
        'Timeliness': timeliness,
        'Validity': validity,
        'Overall': (completeness + accuracy + consistency + timeliness + validity) / 5
    }
```

---

## Project Files

| File                           | Type     | Description                                        |
| ------------------------------ | -------- | -------------------------------------------------- |
| `progetto_data_man_1.1.ipynb`  | Notebook | Main analysis notebook                             |
| `f1_data_management.db`        | Database | SQLite database with all tables                    |
| `f1_regulations_openai.csv`    | Data     | 46 regulations extracted via gpt-5-mini-2025-08-07 |
| `database_schema.md`           | markdow  | Full database schema documentation                 |

---

## AI Model Used

| Model           | Provider | Purpose               | Cost                           |
| --------------- | -------- | --------------------- | ------------------------------ |
| **gpt-5-mini-2025-08-07** | OpenAI   | Regulation extraction | ~$0.25/M input, $2.00/M output |

**Total project cost:** < $0.04

---

## Key Metrics

| Metric                | Value                          |
| --------------------- | ------------------------------ |
| Qualifying records    | 3,853                          |
| Years covered         | 2017-2025                      |
| Regulations extracted | 37                             |
| Data sources          | 3 (Jolpica, Wikipedia, OpenAI) |
| Quality dimensions    | 5 (ISO 25012)                  |
| Stable circuits analyzed | 9                           |

---

## Authors

| Name                | Email                    |
| ------------------- | ------------------------ |
| **Romano Raffaele** | romano.raff10@gmail.com  |
| **Tony Dwara**      | tonydawrabou@gmail.com   |
| **Ilaria Mato**     | Ilaria.mato.02@gmail.com |

---

_F1 Data Management Project - December 2025_
_University of Milano-Bicocca - Data Management Course_
