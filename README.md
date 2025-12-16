# F1 Data Management Project

## Project Overview

This project analyzes Formula 1 qualifying performance data from **2017-2025** to study the impact of the **2022 ground effect regulations** and **budget cap** on car performance and field competitiveness.

### Research Question

> _How did ground effect regulations and budget cap change car performance in Formula 1?_

---

## Data Sources Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA ACQUISITION LAYER                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Jolpica API  │  │ Wikipedia API│  │  OpenAI API  │          │
│  │ (REST)       │  │ (Scraping)   │  │ (LLM)        │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│         ▼                  ▼                  ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Qualifying   │  │  Season      │  │ Regulations  │          │
│  │ Results      │  │  Summaries   │  │ Extraction   │          │
│  │ 3853 records │  │  9 seasons   │  │ 36 records   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  DATA INTEGRATION│
                    │  (Merge/Join)   │
                    └────────┬────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   MongoDB       │
                    │   Storage       │
                    └─────────────────┘
```

---

## Prerequisites

```bash
pip install pandas numpy requests openai pymongo matplotlib
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

**Output:** `f1_regulations_openai.csv` (36 regulations)

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
| 7      | Data Preparation   | Convert times, calculate gaps                       |
| 8      | Data Integration   | Merge all datasets                                  |
| 9      | Data Quality       | ISO 25012 assessment                                |
| 10     | MongoDB            | Store data + run 2 queries                          |
| 11     | Analysis           | Field spread visualization                          |

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

### 4. Era Classification (Without Lambda)

```python
# Pattern: Vectorized conditional column creation
import numpy as np

# Instead of lambda:
# df['Era'] = df['Year'].apply(lambda x: 'Pre-2022' if x < 2022 else 'Post-2022')

# Use np.where (faster, no lambda):
df['Era'] = np.where(df['Year'] < 2022, 'Pre-2022', 'Post-2022')
```

### 5. MongoDB Storage Pattern

```python
from pymongo import MongoClient

def store_dataframe_to_mongodb(df, db_name, collection_name, connection_string):
    """
    Generic pattern for storing pandas DataFrame to MongoDB.
    """
    client = MongoClient(connection_string)
    db = client[db_name]
    collection = db[collection_name]

    # Clear existing data
    collection.delete_many({})

    # Insert records
    records = df.to_dict('records')
    collection.insert_many(records)

    return len(records)
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

| File                           | Type     | Description                              |
| ------------------------------ | -------- | ---------------------------------------- |
| `progetto_data_man_1.1.ipynb`  | Notebook | Main analysis notebook                   |
| `f1_regulations_openai.csv`    | Data     | 36 regulations extracted via GPT-4o-mini |
| `run_extract.py`               | Script   | Re-run OpenAI extraction                 |
| `walkthrough_mongo_quality.md` | Doc      | MongoDB + Data Quality guide             |
| `f1_qualifying_complete.csv`   | Data     | Exported qualifying results              |
| `f1_data_quality_report.csv`   | Data     | Quality metrics report                   |

---

## AI Model Used

| Model           | Provider | Purpose               | Cost                           |
| --------------- | -------- | --------------------- | ------------------------------ |
| **gpt-4o-mini** | OpenAI   | Regulation extraction | ~$0.15/M input, $0.60/M output |

**Total project cost:** < $0.01

---

## Key Metrics

| Metric                | Value                          |
| --------------------- | ------------------------------ |
| Qualifying records    | 3,853                          |
| Years covered         | 2017-2025                      |
| Regulations extracted | 36                             |
| Data sources          | 3 (Jolpica, Wikipedia, OpenAI) |
| Quality dimensions    | 5 (ISO 25012)                  |

---

## Authors

| Name                | Email                    |
| ------------------- | ------------------------ |
| **Romano Raffaele** | romano.raff10@gmail.com  |
| **Ilaria Mato**     | Ilaria.mato.02@gmail.com |
| **Tony Dwara**      | Tonydawrabou@gmail.com   |

---

_F1 Data Management Project - December 2024_
_University of Milano-Bicocca - Data Management Course_
