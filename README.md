# Line Builder

A 7-stage data pipeline that aggregates expert betting predictions from multiple sources, enriches them with historical performance data, and applies a statistical filtering stack — defense-adjusted projections, Bayesian hit-rate smoothing, and expected-value gating — to surface only the highest-confidence betting opportunities for a given day. The system runs fully from a single command and produces structured Excel outputs at each stage, turning hours of manual research into a repeatable, auditable process.

---

## Why Private?

The pipeline generates the actual outputs I use for daily betting research. The source code is private because it's a working, actively maintained system — not a demo — and exposing the full production implementation would hand a direct competitor a calibrated, battle-tested starting point. The architecture and methodology are described here in full for technical evaluation purposes.

---

## Technical Architecture

### Pipeline Overview

```
Step 1 → candidates_{date}.csv                  Expert picks, normalized
Step 2 → candidates_{date}_with_teams.csv        ESPN IDs + team abbreviations resolved
Step 3 → scraped_data_{date}.xlsx                ESPN game logs, 75+ per-player/team metrics
Step 4 → defense_ranks_{date}.csv                Opponent defensive ratios by stat category
Step 5 → line_builder_output_{date}.xlsx         Weighted projections + raw edge vs. line
         ↑ [MANUAL: user fills Odds column]
Step 6 → true_edge_output_{date}.xlsx            Bayesian-filtered, EV-qualified bets
Step 7 → qualified_bets_bettingpros_{date}.xlsx  Market context from BettingPros
```

All intermediate files are date-stamped and retained. Steps 1–5 run sequentially via a single orchestrator (`calculations/main.py`); steps 6–7 are triggered after the manual odds entry step.

---

### Stage-by-Stage Breakdown

#### Step 1 — Expert Pick Aggregation (`unified_stage1_scraper.py`)

A unified scraper fan-out layer that runs four source scrapers in parallel:

- **OddsTrader** — consensus picks with line and position data
- **Pickswise** — expert-selected player props and game bets
- **SportyTrader** — model-generated predictions
- **TheSportsGeek** — editorial picks

Each source scraper handles its own Selenium/BeautifulSoup parsing and normalizes output to a shared schema: `Player`, `Sport`, `Stat`, `Line`, `Position` (Over/Under), `Bet Type`. Results are deduplicated and written as a multi-section CSV with separate Player Props and Game Bets blocks.

#### Step 2 — Entity Resolution (`players_and_teams_lookup_for_scraper.py`)

Fuzzy name matching resolves each player name and team abbreviation against ESPN's entity graph (`espn_ids.py`). The resolved ESPN IDs are needed in Step 3 to query the correct game log URLs. Any unresolved players are flagged to the console for manual lookup before the scraper runs.

#### Step 3 — ESPN Game Log Scraping (`gamelog_scraper_v5.9.6.py`)

The largest and most complex component (~2,000 LOC). For each player and team in the candidate set, it fetches full-season game logs from ESPN using the resolved IDs, extracting 75+ metrics including:

- **Players**: raw box score stats (PTS, AST, REB, STL, BLK, 3PM, MIN, TOI) and derived per-36-minute normalizations
- **Teams**: game-level margin, total score, win/loss, opponent context

Results are written to a two-sheet Excel file: `Player Gamelogs` and `Team Gamelogs`. Game logs are indexed sequentially (`Game_Index`) to enable the non-overlapping window logic used downstream.

#### Step 4 — Opponent Defense Scraping (`defense_ranks_scraper.py`)

Scrapes season-to-date opponent defensive stats from Basketball-Reference (with equivalent sources for other sports). Produces a ranked ratio for each team × stat combination: how much more or less of a given stat their opponents produce relative to league average.

#### Step 5 — Line Builder: Weighted Projection + Edge Calculation (`line_builder.py`)

The core statistical layer. For each candidate bet:

**Projection (Sharp Bettor Weights):**

Projections are built from three non-overlapping historical windows with fixed weights calibrated to prioritize true talent over noise:

| Window      | Games         | Weight |
|-------------|---------------|--------|
| Season      | All – last 10 | 75%    |
| Recent      | Games 6–10    | 20%    |
| Very Recent | Last 5 games  | 5%     |

NBA player stats use per-36-minute normalizations to account for variable minutes; low-minute games (<20 MIN) are excluded from all windows.

**Defense Adjustment:**

The weighted projection is adjusted based on the opponent's defensive rank for that specific stat. The adjustment is linear and symmetric around rank 15 (league median = neutral), with stat-specific maximum adjustment caps derived from sharp bettor research. This is applied only to the final projection, not the historical windows.

**Edge Calculation:**

`Edge = Projection − Betting Line`

A raw edge filter (≥2.0 points for both player props and game bets) gates what advances to the Excel output. The file is structured for manual odds entry before Step 6.

#### Step 6 — True Edge Validation + Bayesian Filter (`validate_true_edge.py`)

After the user fills in the `Odds` column, this stage converts raw edge into expected value and applies a full statistical filter stack:

**Bayesian Hit Rate:**

For each bet, the same three historical windows are used to compute hit rates (how often the player/team exceeded the line historically). Each window's hit rate is smoothed toward 0.50 using Laplace/Bayesian smoothing — equivalent to adding 10 "neutral" games to each window. This shrinks extreme hit rates on small samples while converging to the observed rate once ~30+ games are available.

Smoothed window rates are combined with the same 75/20/5 weights to produce a single composite hit rate.

**Coefficient of Variation (CV) Gate:**

CV (standard deviation / mean) is computed over the recent window. High CV indicates stat volatility; the EV threshold is raised by 1–6 percentage points depending on the CV bracket and sport/market type, making the filter progressively stricter for volatile performers.

**EV Calculation + Tier Assignment:**

```
EV = (hit_rate × win_amount) − ((1 − hit_rate) × stake)
```

Each qualifying bet is assigned a tier based on its composite Bayesian hit rate:

| Tier | Hit Rate Threshold |
|------|--------------------|
| T1   | ≥ 58.5%            |
| T2   | ≥ 54.0%            |
| T3   | ≥ 49.5%            |

Base EV thresholds vary by sport and market (6–12%), with CV-based additions on top.

**Deduplication:** A hard cap of 1 bet per game (across all market types) is enforced. Within a game, the highest-EV bet is retained.

#### Step 7 — BettingPros Market Enrichment (`bettingpros_info_fetcher.py`)

The final output is enriched with current market data from BettingPros: line movement, consensus percentages, and book-level availability. This step provides the context needed to assess whether an identified edge is already priced in or still available.

---

## Technology Stack

- **Language**: Python 3, no frameworks
- **Scraping**: Selenium (dynamic pages) + BeautifulSoup (HTML parsing)
- **Data**: pandas, numpy, openpyxl
- **Sources**: ESPN (game logs), Basketball-Reference (defense stats), OddsTrader, Pickswise, SportyTrader, TheSportsGeek, BettingPros
- **Output format**: Date-stamped Excel files with named sheets

---

## Running the Pipeline

```bash
# Full pipeline for today
python3 calculations/main.py

# Full pipeline for a specific date
python3 calculations/main.py 2026-03-10
```

The orchestrator prompts before re-running any step for which an output file already exists, making it safe to resume mid-pipeline or re-run individual stages.

---

## Repository Structure

```
calculations/
  main.py                   Orchestrator — runs all 7 steps via subprocess
  config.py                 All tunable thresholds, weights, and parameters
  line_builder.py           Weighted projection + defense adjustment engine
  bayesian.py               Bayesian hit-rate and CV calculations
  validate_true_edge.py     EV calculation, tier filtering, deduplication
scrapers/
  unified_stage1_scraper.py Fan-out runner for all source scrapers
  gamelog_scraper_v5.9.6.py ESPN game log scraper (2,000+ LOC)
  defense_ranks_scraper.py  Opponent defensive ratio scraper
  bettingpros_info_fetcher.py BettingPros market data enrichment
player_team_lookup/
  espn_ids.py               Fuzzy name matching → ESPN player/team IDs
  csv_parser.py             Multi-section candidates CSV parser
outputs/                    All intermediate and final outputs (date-stamped)
```

---

I'm happy to do a live walkthrough or answer any questions about the implementation. Reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.
