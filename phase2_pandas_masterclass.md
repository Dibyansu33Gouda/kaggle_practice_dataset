# Phase 2 Masterclass: Groupby, Aggregation & Weighted Statistics in Pandas

*A complete classroom-style guide built around your actual FIFA World Cup 2026 project code.*

---

## 1. Introduction

Every dataset you'll ever touch has this shape: many rows, but the *real* questions live one level above the rows. Your CSV has 54,600 rows — one row per player per match. But nobody actually cares about row #38,201. People care about "who scored the most goals this tournament?" That's not a row. That's a **summary across many rows.**

The tool that turns "many rows" into "one meaningful number per group" is `groupby`. Everything in Phase 2 — player totals, position averages, team rankings — is one skill wearing different clothes: **split the data into meaningful piles, then calculate something for each pile.**

Real-life analogy: think of a teacher with a stack of 200 exam papers from 4 different classes. The teacher doesn't care about paper #147 individually — they care about "what's the average score for Class A?" To answer that, they physically sort papers into 4 piles (one per class), then compute an average *within* each pile. That sorting-then-computing is the entire chapter you're about to master.

---

## 2. Why Do We Need This?

**Before `groupby` existed (or if you tried to avoid it):** you'd have to manually filter for each group and calculate separately.

```python
spain_players = df[df['team'] == 'Spain']
spain_avg = spain_players['player_rating'].mean()

france_players = df[df['team'] == 'France']
france_avg = france_players['player_rating'].mean()

# ...repeat this 30+ times, once per team
```

This is slow to write, slow to run, and breaks the moment a new team appears in the data. `groupby` does all of this — for every group, automatically — in one line:

```python
df.groupby('team')['player_rating'].mean()
```

**Why this problem exists in the first place:** raw data is almost always stored at the finest possible detail (one row = one event, one transaction, one match). But *decisions* are made at a coarser level (per customer, per team, per position). Groupby is the bridge between "how data is stored" and "how humans think about it."

---

## 3. Mental Model — Split → Apply → Combine

This is the single mental picture to carry forever:

```
   RAW DATA                SPLIT                  APPLY              COMBINE
┌─────────────┐      ┌───────────────┐      ┌──────────────┐   ┌───────────────┐
│ P1  Spain  2│      │ Spain pile:   │      │ sum() each   │   │ Spain:   5     │
│ P2  France 1│  ──▶ │  P1=2, P4=3   │  ──▶ │ pile         │──▶│ France:  1     │
│ P3  Brazil 4│      │ France pile:  │      │              │   │ Brazil:  4     │
│ P4  Spain  3│      │  P2=1         │      │              │   │                │
└─────────────┘      │ Brazil pile:  │      │              │   └───────────────┘
                     │  P3=4         │      │              │
                     └───────────────┘      └──────────────┘
```

Three steps, always in this order:
1. **Split** — `groupby('team')` physically groups rows sharing the same team value
2. **Apply** — a function runs *separately* on each group (`.sum()`, `.mean()`, `.count()`, `.agg()`)
3. **Combine** — results are stitched back into one clean output table

Internally, pandas doesn't loop through your data 30 times for 30 teams. It builds an index mapping each row to its group in a single pass, then runs the aggregation function against each group's rows. This is why `groupby` is dramatically faster than manually filtering and looping — you already discovered this the hard way with your original per-team filtering instinct.

---

## 4. Core Concepts

### 4.1 `groupby()` on a single column

```python
df.groupby('player_id')['goals'].sum()
```

- **Definition:** groups rows sharing the same `player_id` value into one "pile," then sums `goals` inside each pile.
- **Return value:** a `Series`, indexed by `player_id`, one value per group.
- **Important note:** `groupby()` alone returns a lazy `GroupBy` object — it does nothing until you call an aggregation function on it (`.sum()`, `.mean()`, `.agg()`, etc.). Calling `df.groupby('player_id')` by itself and printing it just shows you a memory address, not data. This trips up almost every beginner once.

### 4.2 Named Aggregation — `.agg()`

Your actual code:
```python
player_agg = df.groupby(['player_id', 'player_name', 'team']).agg(
    total_matches_played=('match_id', 'count'),
    sum_minutes=('minutes_played', 'sum'),
    sum_goals=('goals', 'sum'),
    meta_total_goals=('total_goals_tournament', 'first')
).reset_index()
```

- **Syntax pattern:** `new_column_name=('source_column', 'function_name')`
- **Purpose:** run *multiple different* calculations on *multiple different* columns, in a single pass over the data, with clean output column names — instead of three separate `.groupby()` calls glued together afterward.
- **Why not just do three separate groupbys and merge them?** You *could*, but that's three full passes over 54,600 rows instead of one, and you'd need three separate `merge()` calls to stitch the results back together — more code, more chances for bugs, slower.

**Function-by-function breakdown:**

| Function | What it does to a pile | When to use it |
|---|---|---|
| `sum` | Adds every value in the pile | Totals: goals, minutes, passes |
| `count` | Counts non-null rows in the pile | Number of matches played (row = 1 match) |
| `mean` | Simple average of the pile | Only safe when every row deserves equal weight |
| `first` | Grabs the first value seen in the pile | Metadata repeated identically on every row (see 4.3) |
| `nunique` | Counts distinct values in the pile | "How many different teams did this player appear under?" |

### 4.3 Why `first` and not `sum` for `total_goals_tournament`

**The trap:** `total_goals_tournament` isn't per-match data — it's a single tournament-wide number that got **copied onto every match row** for that player. If a player played 5 matches, the value `7` (their season total) appears identically on all 5 rows.

If you `sum()` this column per player, you get `7 × 5 = 35` — a meaningless number that's just the real value multiplied by however many matches they played. `first` correctly grabs just one copy of that repeated value.

**Rule of thumb:** ask yourself, "does this value change per row, or is it the same constant repeated?" Constants → `first`. Genuinely per-row measurements → `sum` or `mean`.

### 4.4 Multi-column `groupby` — the landmine

```python
df.groupby(['player_id', 'player_name', 'team'])
```

**Mental model:** grouping by multiple columns is like sorting by "customer AND store location" instead of just "customer." You now get one pile per **unique combination**, not one pile per player.

**What happens if `team` isn't perfectly constant per player?**

```
Row: player_id=P001, team='Spain'   goals=2
Row: player_id=P001, team='Spain'   goals=1
Row: player_id=P001, team='SPAIN'   goals=1     ← inconsistent casing/typo!
```

Grouping by `['player_id', 'team']` creates **two piles** for the same real player: one for `'Spain'` (3 goals) and one for `'SPAIN'` (1 goal) — because to pandas, `'Spain'` and `'SPAIN'` are different strings, full stop. Your one player silently becomes two "different" entities in your output, each carrying an incomplete goal count.

**This is exactly why your discrepancy count changed between notebooks** — grouping by extra columns can silently fragment a group without ever raising an error. No crash, no warning — just quietly wrong numbers.

**Defense:** before adding extra columns to a groupby "just to keep them in the output," verify they're truly constant per your main key:
```python
df.groupby('player_id')['team'].nunique().gt(1).sum()
```
If this returns `0`, it's safe to add `team` to the groupby — it'll never split a real player into two piles. If it's above `0`, investigate before trusting any output built on that groupby.

### 4.5 Weighted Averages — why `.mean()` alone lies to you

Your code:
```python
df['weighted_rating'] = df['player_rating'] * df['minutes_played']
position_agg = df.groupby('position').agg(
    total_minutes=('minutes_played', 'sum'),
    sum_weighted_rating=('weighted_rating', 'sum')
)
position_agg['avg_rating_weighted'] = position_agg['sum_weighted_rating'] / position_agg['total_minutes']
```

**The problem with plain `.mean()`:** imagine two Forwards. Forward A played the full 90 minutes and earned a 9.0 rating. Forward B came on as a substitute for 2 minutes and got a 6.0 rating (small sample, less meaningful). A plain average treats both ratings as equally important:

```
(9.0 + 6.0) / 2 = 7.5
```

But Forward B's rating is based on almost no playing time — it shouldn't count as much as Forward A's full-match performance. A **weighted average** fixes this by scaling each rating by how much playing time backs it up:

```
(9.0 × 90 + 6.0 × 2) / (90 + 2) = 822 / 92 ≈ 8.93
```

Forward A's result now correctly dominates the average, because it's backed by 45× more playing time.

**The formula, in words:** *weighted average = (sum of each value × its weight) ÷ (sum of the weights).* In your code: multiply rating by minutes first (`weighted_rating`), sum those products per group, then divide by total minutes per group. This is a standard technique any real analyst reaches for whenever samples carry different amounts of "evidence" behind them — GPA calculations, stock portfolio returns, survey results, all use exactly this pattern.

### 4.6 Per-90 Normalization

```python
position_agg['avg_distance_per_90'] = (position_agg['sum_weighted_distance'] / position_agg['total_minutes']) * 90
```

**Why:** a player who played 45 minutes naturally covers less total distance than one who played 90 — that's just arithmetic, not skill. To compare players fairly regardless of how long they were on the pitch, sports analytics normalizes every stat to "per 90 minutes played" — as if everyone played a full match. Multiplying the per-minute rate by 90 converts "distance per minute" into "distance if this rate held for a full match."

### 4.7 Dedup-Before-Groupby (Team Defense Logic)

```python
match_level_df = df.drop_duplicates(subset=['match_id', 'team'])
team_defensive = match_level_df.groupby('team').agg(
    total_goals_conceded=('goals_opponent', 'sum')
)
```

**The trap this avoids:** your raw `df` has one row **per player per match** — meaning a single match for one team is represented by ~14 rows (all 14 players who featured). The `goals_opponent` value (goals the *team* conceded that match) is identical across all 14 of those rows — it's team-level data duplicated onto every player row, same pattern as `total_goals_tournament`.

If you summed `goals_opponent` directly on the raw `df` grouped by `team`, you'd multiply every match's real goals-conceded value by however many players featured that match — wildly overcounting. **Deduplicating to one row per match-team combination first** strips out that duplication, so each match's goals-conceded number gets counted exactly once.

**General rule:** whenever a column represents something true about a *group* (a match, a team) rather than something true about the *individual row* (a player), check whether it's been duplicated across multiple rows before aggregating it — and deduplicate first if so.

---

## 5. Teacher–Student Discussion

**Student:** "Why can't I just use `.mean()` for everything? It's simpler."

**Teacher:** Simplicity that gives you the wrong answer isn't actually simpler — it's a shortcut to a bug. `.mean()` treats every row as equally trustworthy. The moment your rows represent unequal amounts of evidence — 2 minutes of playing time vs 90, or a duplicated team-level stat vs a genuine per-player stat — plain `.mean()` silently produces a misleading number with zero warning. The extra step of weighting or deduplicating isn't complexity for its own sake; it's the difference between a real answer and a confidently wrong one.

**Student:** "But my discrepancy count changed between two notebooks and nothing errored. How was I supposed to catch that?"

**Teacher:** That's the uncomfortable truth about `groupby` bugs — they are almost never loud. Pandas doesn't know your intent; it just executes exactly what you asked, even if what you asked was subtly different from what you meant. The only real defense is the habit you're building right now: before trusting any grouped result, ask "is my grouping key actually as unique as I think it is?" and verify it with a `nunique()` check, every time, before you build on top of it.

**Student:** "So should I always add every related column to my `groupby()` list, just to have them in the output?"

**Teacher:** No — and this is the core lesson of Section 4.4. Only add a column to your groupby key if you've *verified* it's truly constant within your main grouping key. Otherwise you're not adding convenience, you're adding a landmine that only detonates when the data has an inconsistency you haven't found yet.

---

## 6. Think Like a Data Analyst

When a professional analyst sees a `groupby` task, the internal checklist runs like this, before writing a single line of code:

1. **What is my true unique key?** (One row per player? Per match? Per player-match?)
2. **Is the column I'm about to aggregate genuinely per-row, or is it a repeated group-level value?** (This decides `sum` vs `first`.)
3. **Do all rows deserve equal weight, or does sample size vary?** (This decides plain `.mean()` vs weighted average.)
4. **If I add extra columns to my groupby key, are they provably constant within the main key?** (Verify with `nunique()` before trusting the output.)
5. **Does my result "smell right"?** (Sanity-check against a known constraint — like you did comparing `sum_goals` against `total_goals_tournament`.)

Step 5 is the one beginners skip most — and it's the one that catches almost everything else.

---

## 7. Practical Examples

### Beginner
**Problem:** Total assists per player.
```python
df.groupby('player_id')['assists'].sum()
```
Groups by player, adds up assists within each group. Straightforward — every row is genuinely a per-match measurement, no traps here.

### Intermediate
**Problem:** Average pass accuracy per position, done correctly (not naively).
```python
pos_agg = df.groupby('position').agg(
    total_success = ('successful_passes', 'sum'),
    total_attempts = ('total_passes', 'sum')
)
pos_agg['true_accuracy'] = pos_agg['total_success'] / pos_agg['total_attempts'] * 100
```
**Why not just `df.groupby('position')['pass_accuracy'].mean()`?** That would average each match's *already-computed percentage* equally, regardless of how many passes each match involved — a player with 2 passes (100% accuracy) would count exactly as much as a player with 80 passes (75% accuracy). Computing the true ratio from raw totals (successes ÷ attempts) avoids that distortion entirely — this is the same "unequal evidence" problem from Section 4.5, just applied to ratios instead of averages.

### Professional
**Problem:** Rank teams by defensive performance without double-counting.
```python
match_level = df.drop_duplicates(subset=['match_id', 'team'])
defense = match_level.groupby('team').agg(
    matches=('match_id', 'count'),
    conceded=('goals_opponent', 'sum')
)
defense['conceded_per_match'] = defense['conceded'] / defense['matches']
```
Notice the deduplication happens *before* the groupby — this is the exact pattern from Section 4.7, and it's the step that separates a correct professional analysis from a plausible-looking but wrong beginner one.

---

## 8. Common Beginner Mistakes

| Mistake | Why it happens | Fix |
|---|---|---|
| Using `sum` on a repeated metadata column | Not recognizing the column is duplicated per row | Use `first`, or verify uniqueness first |
| Adding extra columns to `groupby()` "for convenience" | Assuming they're constant without checking | Run `.nunique()` check before adding to the key |
| Plain `.mean()` on uneven sample sizes | Not questioning whether every row deserves equal weight | Use a weighted average (Section 4.5) |
| Aggregating team/match-level stats directly on player-level rows | Not realizing the value is duplicated across teammates | Deduplicate to the correct grain first (Section 4.7) |
| Assuming no error = correct result | `groupby` bugs are silent by nature | Always sanity-check totals against an independent number |

---

## 9. Frequently Asked Classroom Questions

**Student:** "Why doesn't `df.groupby('team')` show me anything when I print it?"

**Teacher:** Because `groupby()` alone is lazy — it only builds the group structure internally, it doesn't compute anything yet. Nothing happens until you chain an aggregation function like `.sum()` or `.agg()` onto it. Think of it as "I've sorted the piles" — you haven't done any math on them yet.

**Student:** "Why did my discrepancy count go *down* when I added more columns to groupby — isn't that a good sign?"

**Teacher:** No — and this is a dangerous instinct to unlearn. A number moving in the "hoped for" direction isn't evidence it's correct. As shown in Section 4.4, splitting one real player into two incomplete piles can accidentally make partial sums match a metadata value more often, purely by chance — not because the underlying inconsistency was fixed.

---

## 10. Compare Similar Concepts

| Function | Question it answers | Multiplicity handling |
|---|---|---|
| `.sum()` | "What's the total?" | Adds every value, duplicates included |
| `.mean()` | "What's the plain average?" | Equal weight to every row |
| Weighted average (manual) | "What's the average, accounting for unequal evidence?" | Weights each value before averaging |
| `.first()` | "What's the repeated constant value?" | Ignores duplicates, grabs one copy |
| `.nunique()` | "How many distinct values exist?" | Counts unique values, ignores repeats |
| `.count()` | "How many non-null rows exist?" | Counts rows, not values |

---

## 11. Real Data Analysis Scenarios

- **E-commerce:** total revenue per customer — same pattern as `sum_goals` per player.
- **Hospital records:** average recovery time per department, weighted by number of patient-days — identical logic to weighting ratings by minutes played.
- **Employee records:** grouping by `employee_id` + `department` — same landmine as grouping by `player_id` + `team`: if an employee transferred departments mid-year, you'd silently fragment their record unless you check first.
- **Sales data:** a "total store revenue for the day" column duplicated across every transaction row that day — same trap as `goals_opponent`, requiring deduplication before aggregating at the store level.

---

## 12. Interview Perspective

**Common question:** "What's the difference between `.groupby().mean()` and a weighted average, and when would you use each?"
**What they're really testing:** whether you understand that `.mean()` assumes equal-weight samples, and can recognize when that assumption breaks (uneven sample sizes, uneven exposure/duration).

**Trick question:** "If I group by two columns instead of one, will I always get more accurate results?"
**Best answer:** No — more granular grouping only helps if the extra column is genuinely meaningful to the analysis *and* doesn't fragment your real key. Otherwise it can silently produce more, smaller, less trustworthy groups. Always verify the extra column's uniqueness relationship to your primary key first.

**Follow-up they might ask:** "How would you detect that kind of fragmentation in a groupby before it causes a bug?"
**Best answer:** `df.groupby(primary_key)[extra_column].nunique()` — if any group has more than 1 unique value, that column isn't safe to add to the groupby key without further investigation.

---

## 13. Practice While Learning

**Pause and predict:** given this data —
```
player_id | team    | goals
P01       | Spain   | 2
P01       | Spain   | 1
P02       | France  | 3
```
What does `df.groupby('player_id')['goals'].sum()` return? What does `df.groupby(['player_id','team'])['goals'].sum()` return? *(Same answer in this case — because `team` is constant per player here. Now imagine row 2's team was `'France'` by data-entry error — predict how the second groupby's output changes.)*

---

## 14. Challenge Problems

**Easy:** Using `.agg()`, compute total goals and total assists per player in one call.

**Medium:** A column `stadium_capacity` is duplicated across every player row for a given match. Write the correct way to compute the *average stadium capacity across all matches* (not accidentally multiplied by player count).

**Hard:** You suspect some `player_id`s have inconsistent `nationality` values across their rows (typos, formatting differences). Write code that lists every `player_id` where this is true, along with the distinct `nationality` values found for each.

---

## 15. Summary

- `groupby` is always **split → apply → combine** — nothing more mysterious than that.
- `.agg()` with named aggregation lets you compute multiple different summaries in one pass.
- `first` is for repeated constants; `sum`/`mean` are for genuine per-row measurements — confusing the two silently corrupts results.
- Multi-column `groupby` only stays safe if every extra column is verified constant within your main key — always check with `.nunique()` before trusting it.
- Plain `.mean()` assumes every row is equally trustworthy — when sample sizes or exposure differ, use a weighted average instead.
- Team/match-level values duplicated across player rows must be deduplicated to the correct grain *before* aggregating, or you'll multiply the real number by however many rows share it.
- No `groupby` bug throws an error — the only defense is sanity-checking your output against an independent number, every time.
