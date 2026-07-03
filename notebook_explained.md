# 📘 What This Notebook Does — Explained Simply

**Notebook name:** Notebook 1: Data Cleaning & Initial EDA
**Dataset:** FIFA World Cup 2026 Player Performance (a CSV file full of football player match stats)

Think of this notebook like a chef prepping vegetables before cooking. The "raw" dataset is messy — it has missing values, wrong data types, maybe some duplicate entries. Before you can do any cool analysis or build a model, you need to clean it up. That's exactly what this notebook does, step by step. Nothing fancy is being predicted here — it's all about making the data trustworthy.

Let's go through it cell by cell, in plain words.

---

## 1. Import (Cell 2)
```python
import pandas as pd
import numpy as np
import warnings
```
This just loads the tools needed:
- **pandas** → the main library for working with tables of data (like Excel, but in code)
- **numpy** → helps with number crunching (math operations)
- **warnings** → lets us silence annoying warning messages later

Nothing happens with the data yet — this is just "getting your toolbox ready."

---

## 2. Loading the Dataset (Cell 4)
```python
Raw_file_path = r"D:\fifa_world_cup_2026_player_performance.csv"
raw_df = pd.read_csv(Raw_file_path)
df = raw_df.copy()
```
Here the CSV file is read into a pandas DataFrame (basically a spreadsheet-like table in Python), called `raw_df`.

Then a **copy** of it is made called `df`. Why copy it? So that the original untouched data (`raw_df`) is always safe. If something goes wrong while cleaning, you can always go back to `raw_df` and start over, instead of re-reading the file from disk.

It then prints the shape of the data (rows × columns) so you know how big the dataset is.

---

## 3. Inspection (Cell 6)
```python
df.info(memory_usage='deep')
display(df.head(3))
display(df.describe().T[...])
```
This is like giving the dataset a health checkup before touching anything:
- `df.info()` → shows column names, data types, and how much memory the data uses
- `df.head(3)` → shows the first 3 rows so you can visually see what the data looks like
- `df.describe()` → gives quick statistics (average, max, count, etc.) for numeric columns

This step doesn't change anything — it's purely for understanding what you're working with.

---

## 4. Missing Values Audit & Structural Handling (Cell 8)
This is one of the more important cells. Here's the idea in simple terms:

1. First, it counts how many missing (empty/NaN) values exist in each column, and what percentage that is of the whole column.
2. Then it looks at **why** some values might be missing — and this dataset has a clever, logical reason:
   - Some columns only make sense for **goalkeepers** (like `saves`, `clean_sheet`, `penalty_saves`). If a player is NOT a goalkeeper, these will naturally be empty — because an outfield player was never in a position to make a "save." So instead of just guessing a number, the code fills these with `0.0` for outfield players, because "0 saves" makes logical sense — the player simply isn't a goalkeeper.
   - Similarly, some columns only make sense for **outfield players** (like `crosses`, `shots_on_target`). These get filled with `0.0` specifically for goalkeepers.
3. Then it does two **safety checks** (called `assert` statements):
   - Makes sure there are still exactly 4 unique player positions (nothing got corrupted)
   - Makes sure critical columns like `player_id`, `match_id`, `goals` have **zero** missing values — because these are too important to have gaps

**Why this matters for a beginner:** This teaches you that missing data isn't always a mistake — sometimes it's missing for a *real reason*, and the smart way to fill it in is to think about *why* it's missing, not just plug in an average value everywhere.

---

## 5. Duplicate Check (Cell 10)
```python
composite_key = ['player_id', 'match_id']
duplicate_mask = df.duplicated(subset=composite_key, keep=False)
```
This checks if the same player appears more than once for the same match (which shouldn't happen — that would mean the data got duplicated somehow).

- If duplicates are found, it shows a preview of them, then removes the duplicates (keeping only the first occurrence).
- If no duplicates are found, it just confirms that every (player, match) pair is unique.

Think of `player_id` + `match_id` together like a unique ID card — no two rows should share the exact same card.

---

## 6. Datatype Fixes (Cell 12)
Right now, computers don't automatically know that a column full of dates is actually a "date" — by default it might just treat it as plain text. This cell fixes that:

- `match_date` is converted from plain text into a proper **datetime** format, so Python understands it as an actual calendar date (which allows sorting by date, filtering by month, etc.)
- Columns like `position`, `preferred_foot`, `nationality`, `team` etc. are converted to a special type called **category**. This is used when a column only has a handful of repeating values (like "Goalkeeper", "Defender", "Midfielder", "Forward"). It saves memory and speeds up grouping operations.
- Columns like `goals`, `assists`, `shots`, `yellow_cards` are converted to whole numbers (**Int64**), since you can't score half a goal!

**Why it matters:** Correct data types = correct calculations. If a "goals" column was stored as text instead of a number, you couldn't do math on it (like summing total goals).

---

## 7. Derived Columns / Feature Engineering (Cell 14)
This is where new, useful columns are **created** from existing ones — this is called **feature engineering**. Four new columns are added:

1. **`xg_diff`** = `goals - expected_goals_xg` → tells you if a player scored *more* or *fewer* goals than statistically expected. A positive number means they're finishing better than expected (a "clinical finisher").
2. **`xa_diff`** = `assists - expected_assists_xa` → same idea but for assists (playmaking performance).
3. **`goal_contributions`** = `goals + assists` → a simple combined measure of how much a player directly contributed to scoring.
4. **`calc_pass_accuracy`** = `(successful_passes / total_passes) × 100` → the percentage of passes that were successful. It safely avoids "divide by zero" errors using `np.where`, so if a player made 0 total passes, it just returns 0 instead of crashing.
5. **`actions_per_90`** = normalizes a player's total actions (passes + tackles + interceptions) to what they would be *if they played a full 90-minute match*. This is important because comparing a player who played 90 minutes to a substitute who played 10 minutes wouldn't be fair otherwise.

**Why it matters:** Raw stats alone don't always tell the full story. These new columns turn raw numbers into more meaningful insights.

---

## 8. Sanity Checks & Data Integrity Audits (Cell 16) — *3rd-last cell*
This cell double-checks that the data makes internal sense — like a teacher cross-checking a student's math homework.

Here's what happens step by step:
1. It groups all rows by `player_id` and adds up each player's `goals` across all their matches → this gives a **computed total**.
2. It compares this computed total against a column already in the dataset called `total_goals_tournament` (which is supposed to be the "official" reported total).
3. It calculates the difference (**discrepancy**) between the two.
4. If there are no mismatches → it prints a success message confirming the data is internally consistent.
5. If there ARE mismatches → it shows you which players have inconsistent numbers, and even leaves a note saying: trust the row-by-row calculated total more than the possibly-wrong summary metadata.

**Why it matters:** This is a trust exercise. Even after cleaning, you want to verify the numbers actually add up logically before you use this data for anything serious like a report or a machine learning model.

---

## 9. Export Cleaned Dataset (Cell 18) — *2nd-last cell*
```python
CLEANED_OUTPUT_PATH = "cleaned_player_data.csv"
df.to_csv(CLEANED_OUTPUT_PATH, index=False)
```
After all that cleaning, this cell **saves** the cleaned-up DataFrame as a brand-new CSV file called `cleaned_player_data.csv`.

- `index=False` is important — it tells pandas *not* to add an extra unnamed column with row numbers when saving. Without this, every time you reload the file, you'd get a messy extra column.
- It then prints a confirmation message with the final shape (rows × columns) of the exported file.

**Why it matters:** This is the "final product" of the notebook — a clean, ready-to-use file that a future notebook (like "Notebook 2") can load directly without repeating all this cleaning work again.

---

## 10. Key Takeaways & Cleaning Audit Log (Cell 19) — *last cell*
This final cell is just a **markdown summary** (no code) — like an executive summary at the end of a report. It recaps everything that was done, organized neatly:

1. **Structural Imputation & Missingness Decisions** — recaps the goalkeeper/outfield-specific 0.0 filling logic from Cell 8, and confirms no missing values remain in critical ID columns.
2. **Deduplication Audit** — recaps the duplicate-checking done in Cell 10, confirming the (player_id, match_id) pairs are unique.
3. **Type Optimization** — recaps the datatype fixes from Cell 12 (dates converted properly, categories used for memory savings, integers enforced for count columns).
4. **Engineered Features for Notebook 2** — recaps the new columns created in Cell 14 (`xg_diff`, `xa_diff`, `actions_per_90`), explicitly mentioning that these are being prepared for use in a *future* notebook.
5. **Sanity Check Audit** — recaps the integrity check from Cell 16, confirming row-level and summary-level goal totals match.

**Why it matters:** This is basically the notebook "signing off" — a clean written record of every decision made and why, so that anyone (including future-you) can read this section alone and understand exactly what happened to the data, without having to re-read every code cell.

---

## 🎯 Big Picture Summary (in one paragraph)

This notebook takes a messy, raw CSV file of football player stats and turns it into a clean, trustworthy dataset. It does this by: loading the data safely (keeping a backup copy), inspecting it, intelligently filling in missing values based on real-world logic (goalkeepers vs. outfield players), removing duplicate records, fixing data types so calculations work correctly, engineering new useful stat columns, running sanity checks to make sure the numbers are internally consistent, and finally exporting a clean CSV file — with a written summary at the end explaining every decision. This cleaned file is meant to be the starting point for a "Notebook 2" that will likely do deeper analysis or modeling.
