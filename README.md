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
│  │ Jolpica API  │  │  OpenAI API  │  │   FIA ATR    │           │
│  │ (REST)       │  │  (LLM)       │  │ (Allocations)│           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Qualifying   │  │ Regulations  │  │ Wind Tunnel  │           │
│  │ + WDC/WCC    │  │ Extraction   │  │ CFD Limits   │           │
│  │ 3853 records │  │ 46 records   │  │ 40 records   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ DATA INTEGRATION│
                    │ + Entity Resol. │
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
| `seasons` | F1 season years with WDC/WCC winners |
| `teams` | Teams per season with WCC and ATR | 
| `drivers` | Drivers with WDC, races count | 
| `events` | GPs with pole position | 
| `regulations` | FIA regulations (from OpenAI) | 
| `field_spread` | P1-P10 gap per GP (competitiveness) |
| `circuit_performance` | Pre/Post 2022 comparison (9 stable circuits) |

---

## Prerequisites

```bash
pip install pandas numpy requests openai matplotlib
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

The notebook uses OpenAI GPT to extract F1 regulations from parametric knowledge (no scraping).

**Output:** `f1_regulations_openai.csv` (46 regulations)

---

### STEP 2: Run Notebook Cells in Order

Open `progetto_data_man_1.1.ipynb` and run cells in this exact sequence:

| Cell # | Section            | What it does                                        |
| ------ | ------------------ | --------------------------------------------------- |
| 1      | Imports            | Load libraries (pandas, numpy, requests, etc.)      |
| 2      | Jolpica API        | Fetch 3853 qualifying results with pagination       |
| 3      | Entity Resolution  | Normalize team names across years                   |
| 4      | Load Regulations   | Load `f1_regulations_openai.csv` (OpenAI extracted) |
| 5      | ATR Data           | Calculate wind tunnel/CFD allocations               |
| 6      | WDC/WCC Data       | Fetch championship winners per season               |
| 7      | Data Preparation   | Convert times, calculate gaps                       |
| 8      | Data Integration   | Merge all datasets                                  |
| 9      | SQLite Storage     | Store data in `f1_data_management.db`               |
| 10     | Data Quality       | ISO 25012 assessment                                |
| 11     | Circuit Analysis   | Pre vs Post 2022 pole time comparison (9 circuits)  |
| 12     | Field Spread       | Visualization of field competitiveness              |

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

### 2. Entity Resolution Pattern

```python
# Team name normalization mapping
TEAM_ENTITY_MAP = {
    'Toro Rosso': 'RB F1 Team',
    'AlphaTauri': 'RB F1 Team',
    'Force India': 'Aston Martin',
    'Racing Point': 'Aston Martin',
    'Alfa Romeo': 'Sauber',
    'Renault': 'Alpine F1 Team',
}

def resolve_team_entity(team_name):
    """Resolve team name to canonical entity."""
    return TEAM_ENTITY_MAP.get(team_name, team_name)

df['Team_Entity'] = df['Team'].apply(resolve_team_entity)
```

### 3. LLM-Based Data Extraction (OpenAI)

```python
def extract_structured_data_via_llm(prompt, model='gpt-5-mini'):
    """
    Generic pattern for using LLM to extract structured data.
    Returns JSON that matches the provided schema.
    """
    import openai
    import os

    openai.api_key = os.getenv('OPENAI_API_KEY')

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
def assess_data_quality(df, name, exclude_cols=None):
    """
    Generic pattern for ISO 25012 data quality assessment.
    Supports excluding structural null columns.
    """
    if exclude_cols:
        df_check = df.drop(columns=exclude_cols, errors='ignore')
    else:
        df_check = df
    
    total_cells = df_check.size
    null_cells = df_check.isnull().sum().sum()
    completeness = (1 - null_cells / total_cells) * 100
    
    return {'Dataset': name, 'Completeness': completeness}
```

---

## Project Files

| File                           | Type     | Description                                        |
| ------------------------------ | -------- | -------------------------------------------------- |
| `progetto_data_man_1.1.ipynb`  | Notebook | Main analysis notebook                             |
| `f1_data_management.db`        | Database | SQLite database with all tables                    |
| `f1_regulations_openai.csv`    | Data     | 46 regulations extracted via gpt-5-mini-2025-08-07 |
| `f1_atr_allocations.csv`       | Data     | Wind tunnel/CFD allocations (2021+)                |
| `database_schema.md`           | Markdown | Full database schema documentation                 |

---

## AI Model Used

| Model           | Provider | Purpose               | Cost                           |
| --------------- | -------- | --------------------- | ------------------------------ |
| **gpt-5-mini-2025-08-07** | OpenAI   | Regulation extraction | ~$0.25/M input, $2.00/M output |

**Total project cost:** < $0.04

---

## Key Metrics

| Metric                   | Value                               |
| ------------------------ | ----------------------------------- |
| Qualifying records       | 3,853                               |
| Years covered            | 2017-2025                           |
| Regulations extracted    | 46                                  |
| Data sources             | 4 (Jolpica, OpenAI, FIA ATR, Entity Resolution) |
| Quality dimensions       | 5 (ISO 25012)                       |
| Stable circuits analyzed | 9                                   |

---

## Key Results

| Metric | Pre-2022 | Post-2022 | Interpretation |
|--------|----------|-----------|----------------|
| **Field Spread (P1-P10)** | 2.55s | 2.15s | **+16% more competitive** |
| **Pole Times** | Baseline | -1.2% avg | **Cars slightly faster** |

---

## Authors

| Name                | Email                    |
| ------------------- | ------------------------ |
| **Romano Raffaele** |  |
| **Tony Dwara**      |  |
| **Ilaria Mato**     |  |

---

_F1 Data Management Project - January 2026_
_University of Milano-Bicocca - Data Management Course_
