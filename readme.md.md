# FIFA World Cup 2026 — Player Performance Analysis

Exploratory data analysis project on a player-match performance dataset (Kaggle) covering the FIFA World Cup 2026. Built entirely with **pandas** and **NumPy** — no ML libraries.

## Dataset

`fifa_world_cup_2026_player_performance.csv` — 54,600 rows, one row per player per match. 75 columns spanning player bio data, match context, and performance metrics (xG, passes, defensive actions, physical tracking data, tournament totals).

## Project Structure

```
├── fifa_world_cup_2026_player_performance.csv   # raw dataset
├── cleaned_player_data.csv                      # output of Notebook 1
├── 01_cleaning_eda.ipynb                        # data cleaning & audit
├── 02_analysis.ipynb                            # analysis & insights
└── README.md
```

## Notebook 1 — Cleaning & EDA

1. Initial inspection (shape, dtypes, describe)
2. Missing values audit — structural (not blind) imputation for goalkeeper-only vs outfield-only columns
3. Duplicate check on composite key (`player_id`, `match_id`)
4. Dtype fixes — `match_date` → datetime, low-cardinality columns → category
5. Derived columns — `xg_diff`, `xa_diff`, `goal_contributions`, `calc_pass_accuracy`, `actions_per_90`
6. Data integrity sanity check — match-level `goals` aggregated per player vs. reported `total_goals_tournament`
7. Export to `cleaned_player_data.csv`

**Key finding:** Aggregated match-level goals did **not** match the `total_goals_tournament` metadata column for 3,547 players. Granular match-level sums were treated as the source of truth for all downstream analysis.

## Notebook 2 — Analysis

- Positional performance comparison (groupby)
- Age and market value segmentation (`pd.cut`)
- Pivot tables — stage × position × rating
- Time trend across the tournament
- xG over/under-performance ranking
- NumPy-based correlation matrix and percentile filtering

## Requirements

```
pandas
numpy
```

## Usage

```bash
git clone https://github.com/yourusername/repo-name.git
cd repo-name
jupyter notebook 01_cleaning_eda.ipynb
```

Run Notebook 1 first — Notebook 2 imports `cleaned_player_data.csv`, which Notebook 1 generates.
