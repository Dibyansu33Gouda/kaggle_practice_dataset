# ⚽ FIFA World Cup 2026 — Player Performance Analysis

A self-directed data analysis project on a Kaggle player-match performance dataset for the FIFA World Cup 2026. Built entirely with **pandas** and **NumPy** — no ML libraries. Part of my ongoing B.Tech CSE learning practice (NIST University, 2025 batch).

## 📁 Repository Contents

| File | Description |
|---|---|
| `fifa_world_cup_2026_player_performance.csv` | Raw dataset — 54,600 rows, one row per player per match |
| `cleaned_player_data.csv` | Cleaned output, exported from Notebook 1 |
| `kaggle_dataset.ipynb` | **Notebook 1** — data cleaning, missing value handling, integrity checks |
| `player&team.ipynb` | **Notebook 2** — player, position, and team-level aggregations |
| `notebook_explained.md` | Beginner-friendly, cell-by-cell walkthrough of Notebook 1 |
| `phase2_pandas_masterclass.md` | Deep-dive study guide on groupby, named aggregation, and weighted averages used in Notebook 2 |

## 🧹 Notebook 1 — Cleaning & EDA

1. Initial inspection (shape, dtypes, describe)
2. Missing values audit — structural imputation (goalkeeper-only vs. outfield-only stats handled separately, not blindly filled)
3. Duplicate check on composite key (`player_id`, `match_id`)
4. Dtype fixes — `match_date` → datetime, low-cardinality columns → category
5. Derived columns — `xg_diff`, `xa_diff`, `goal_contributions`, `calc_pass_accuracy`, `actions_per_90`
6. Data integrity sanity check — match-level goal sums vs. the dataset's own `total_goals_tournament` metadata column
7. Export to `cleaned_player_data.csv`

**Finding:** thousands of players showed a mismatch between summed match-level goals and the pre-computed tournament total column — flagged and documented rather than silently trusted.

## 📊 Notebook 2 — Player, Position & Team Aggregation

- Per-player tournament totals via named aggregation (`groupby().agg()`)
- Position-level performance using **weighted averages** (weighted by `minutes_played`, not naive `.mean()`) to avoid distorting stats with small-sample appearances
- Per-90-minute normalization for fair cross-player comparison
- Team-level offensive and defensive rankings, with deduplication applied before aggregating team/match-level stats to avoid double-counting across teammates

## 🛠️ Requirements

```
pandas
numpy
```

## ▶️ Usage

```bash
git clone https://github.com/Dibyansu33Gouda/kaggle_practice_dataset.git
cd kaggle_practice_dataset
jupyter notebook kaggle_dataset.ipynb
```

Run Notebook 1 first — Notebook 2 loads `cleaned_player_data.csv`, which Notebook 1 generates.

## 🎯 Why this repo exists

- Practice writing production-quality pandas/NumPy code, not just "getting an answer"
- Build a habit of sanity-checking data instead of trusting it blindly
- Track hands-on learning progress alongside coursework
