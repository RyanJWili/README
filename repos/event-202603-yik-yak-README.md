# Yik Yak Event Matchmaking

Matchmaking engine for ~68k Yik Yak event users. Produces romantic, friend, and group matches with 99.8% user coverage.

## Quick Start

1. **Clone** the repo
2. **Secrets** — copy `.env.example` to `.env` and fill in values:
   ```
   cp .env.example .env
   ```
3. **Install dependencies:**
   ```bash
   uv sync          # preferred
   # or
   pip install -r requirements.txt
   ```
4. **Data** — obtain CSVs from the team and place them in:
   - `data/yik-yak/` — user profile exports (raw and cleaned)
   - `data/ditto/` — Ditto school reference data
5. **Run EDA:**
   ```bash
   uv run jupyter notebook EDA.ipynb
   ```
6. **Run matching:**
   ```bash
   uv run python main.py --csv data/yik-yak/yik-yak-profiles-cleaned_v2.csv --max-attempts 3 -v
   ```

## Project Structure

```
matching/          Matching engine (loader, scorer, engine, runner)
scripts/           Data cleaning scripts (school backfill, Ditto school cleanup)
workflow/          GOALS, BLUEPRINT, EXECUTION, BACKLOG (process docs)
docs/              Matching strategy document
data/              Source CSVs and reference data (not in repo)
output/            Match results (not in repo)
EDA.ipynb          Exploratory data analysis notebook
main.py            CLI entry point for the matching engine
```

## Data Pipeline

1. **Export** — Yik Yak profiles exported to CSV
2. **Clean** — School backfill from `yikyakschool`, gender/ethnicity cleanup
3. **EDA** — Exploratory analysis in `EDA.ipynb`
4. **Match** — `main.py` runs the matching engine, outputs to `output/`

## Key Outputs

| File | Description |
|---|---|
| `output/romantic_matches.csv` | Romantic match pairs with compatibility scores |
| `output/friend_matches.csv` | Friend match pairs |
| `output/group_assignments.csv` | Group activity assignments |
