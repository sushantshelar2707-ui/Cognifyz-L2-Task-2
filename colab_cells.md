# Level 2 — Google Colab cells (copy one by one, top to bottom)

Paste each numbered block into its **own Colab code cell** and run in order.
Lines marked *optional text cell* can be added with Colab's **+ Text** button for a polished submission.


---

 *Optional text cell:* ** Level 2 — Data Transformation & Aggregation**


### ▶ CODE CELL 1

```python

# Setup
import os
import pandas as pd
import numpy as np

pd.set_option("display.max_columns", None)
pd.set_option("display.width", 140)

print("pandas version:", pd.__version__)
```


---

 *Optional text cell:* **Task 2.1 — Data Filtering**


### ▶ CODE CELL 2

```python

# Load dataset (Level 1 output first, raw fallback second)
CANDIDATE_PATHS = [
    "level1/output/trains_cleaned.csv",
    "trains_cleaned.csv",
    "Railway_info.csv",
    "data/raw/Railway_info.csv",
    "trains.csv",
    "data/raw/trains.csv",
]

def load_dataset():
    """Return (dataframe, source_path). Falls back to a Colab upload widget."""
    for path in CANDIDATE_PATHS:
        if os.path.exists(path):
            print(f" Loading dataset from: {path}")
            return pd.read_csv(path), path
    try:  # Google Colab upload fallback
        from google.colab import files
        print("Dataset not found in this session — please upload your CSV:")
        uploaded = files.upload()
        name = next(iter(uploaded))
        return pd.read_csv(name), name
    except ImportError:
        raise FileNotFoundError(
            "Could not locate the dataset CSV. Upload it (Colab: folder icon) and re-run."
        )

df, source_path = load_dataset()

# Defensive column resolution (robust to header naming)
def find_column(frame, *patterns):
    """First column whose lower-cased name contains any of the patterns."""
    for pattern in patterns:
        for col in frame.columns:
            if pattern in col.lower():
                return col
    return None

train_col  = find_column(df, "train no", "train_no", "train id", "train code", "train") or df.columns[0]
source_col = find_column(df, "source", "origin") or find_column(df, "from")
dest_col   = find_column(df, "destination") or find_column(df, "to")
day_col    = find_column(df, "day") or "days"

# If we fell back to RAW data, apply Level 1 standardization inline
if "cleaned" not in os.path.basename(source_path).lower():
    print("Raw data loaded — applying Level 1 standardization inline.")
    def _std(v):
        return v if pd.isna(v) else " ".join(str(v).split()).upper()
    for col in {source_col, dest_col, find_column(df, "train name")}:
        if col:
            df[col] = df[col].apply(_std)

# Normalize day names for reliable filtering (' saturday' / 'SATURDAY' -> 'Saturday')
# and repair corrupted tokens found in the raw feed (e.g. 'Fridayd' -> 'Friday')
DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]

def normalize_day(value):
    v = str(value).strip().capitalize()
    for day in DAYS:
        if v.startswith(day):
            return day
    return v

_clean = df[day_col].fillna("UNKNOWN").astype(str).str.strip().str.capitalize()
n_dirty = int((~_clean.isin(DAYS)).sum())
df[day_col] = _clean.apply(normalize_day)
print(f"Repaired {n_dirty:,} corrupted day value(s) (e.g. 'Fridayd' -> 'Friday').")

print(f"Rows: {len(df):,} | Columns: {df.shape[1]}")
df.head(10)
```


### ▶ CODE CELL 3

```python

# Task 2.1 | Filter trains operating on a specific day
TARGET_DAY = "Saturday"   # <- change to any day: Monday ... Sunday

print("Services per day in the dataset:")
print(df[day_col].value_counts())

day_df = df[df[day_col] == TARGET_DAY].copy()
print(f"\n Trains operating on {TARGET_DAY}: {len(day_df):,}")
day_df.head(10)
```


### ▶ CODE CELL 4

```python

# Task 2.1 | New dataframe: trains starting from a specific station
# Default = busiest source station; set any UPPER-CASE station name to explore others
SOURCE_STATION = df[source_col].value_counts().idxmax()

station_df = df[df[source_col] == SOURCE_STATION].copy()
print(f" Trains starting from {SOURCE_STATION}: {len(station_df):,}")
station_df.head(10)
```


---

 *Optional text cell:* **Task 2.2 — Grouping and Aggregation**


### ▶ CODE CELL 5

```python

# Task 2.2 | Trains originating per source station
trains_per_source = (
    df.groupby(source_col)[train_col]
      .count()
      .rename("train_count")
      .sort_values(ascending=False)
)

print("Trains originating per source station (top 10):")
print(trains_per_source.head(10))
print(f"\nStations covered: {len(trains_per_source):,}")
```


### ▶ CODE CELL 6

```python

# Task 2.2 | Average trains per day, per source station
# Count trains per (station, day) pair, then average over the days each station runs
per_day = df.groupby([source_col, day_col])[train_col].count()

avg_trains_per_day = (
    per_day.groupby(level=0)
           .mean()
           .round(2)
           .rename("avg_trains_per_day")
           .sort_values(ascending=False)
)

print("Average trains per operating day, per source station (top 10):")
print(avg_trains_per_day.head(10))
```


---

 *Optional text cell:* **Task 2.3 — Data Enrichment**


### ▶ CODE CELL 7

```python

# Task 2.3 | Categorize operating days: Weekday vs Weekend
def categorize_day(day):
    if day in {"Saturday", "Sunday"}:
        return "Weekend"
    if day in {"Monday", "Tuesday", "Wednesday", "Thursday", "Friday"}:
        return "Weekday"
    return "Unknown"   # never silently mislabel unexpected values

df["Day_Category"] = df[day_col].apply(categorize_day)

print("Day_Category distribution:")
print(df["Day_Category"].value_counts())
df.head(10)
```


### ▶ CODE CELL 8

```python

# Export Level 2 outputs
os.makedirs("level2/output", exist_ok=True)

df.to_csv("level2/output/trains_enriched.csv", index=False)
day_df.to_csv(f"level2/output/trains_{TARGET_DAY.lower()}.csv", index=False)
station_file = f"level2/output/trains_from_{SOURCE_STATION.lower().replace(' ', '_')}.csv"
station_df.to_csv(station_file, index=False)

print("Saved Level 2 outputs:")
print("   level2/output/trains_enriched.csv   (full enriched dataset)")
print(f"   level2/output/trains_{TARGET_DAY.lower()}.csv   ({TARGET_DAY} filter)")
print(f"   {station_file}   ({SOURCE_STATION} filter)")

try:  # auto-download
    from google.colab import files
    files.download("level2/output/trains_enriched.csv")
except ImportError:
    pass
```


---



