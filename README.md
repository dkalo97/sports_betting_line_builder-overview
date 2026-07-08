# Line Builder

A 7-stage data pipeline that aggregates expert betting predictions from multiple sources, enriches them with historical performance data, and applies a statistical filtering stack — defense-adjusted projections, Bayesian hit-rate smoothing, and expected-value gating — to surface only the highest-confidence betting opportunities for a given day. The system runs from a single orchestrator command through the projection stage and produces structured Excel outputs at each stage, turning hours of manual research into a repeatable, auditable process.

---

## Why Private?

The pipeline generates the actual outputs I use for daily betting research. The source code is private because it's a working, actively maintained system — not a demo — and exposing the full production implementation would hand a direct competitor a calibrated, battle-tested starting point. The architecture and methodology are described here in full for technical evaluation purposes.

---

## Current Status

| Sport | Status |
|-------|--------|
| NBA | Live — real-money bets |
| MLB | Paper-trading validation |
| NFL | Planned — paper-trading rollout later in 2026 |
| NHL | Planned — paper-trading rollout later in 2026 |
| Soccer | Out of scope for now — three-outcome (1X2) markets need a different modeling approach than the current binary hit-rate framework |

The system is under active, iterative development. Thresholds and formulas are deliberately kept conservative (flat rather than adaptive) until enough graded bets accumulate to validate a more complex version empirically — see the CV and qualification-gate notes in Step 6 below for a concrete example of that discipline.

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

All intermediate files are date-stamped and retained. Steps 1–5 run sequentially via a single orchestrator (`calculations/main.py`). Steps 6–7 are run as separate standalone commands once the Odds column is filled in — they are not auto-chained by the orchestrator (see "Running the Pipeline" below).

---

### Stage-by-Stage Breakdown

#### Step 1 — Expert Pick Aggregation (`unified_stage1_scraper.py`)

An orchestrator that runs each source scraper as its own subprocess, then standardizes and deduplicates the combined output. Three source scrapers are currently active:

- **OddsTrader** — consensus player-prop picks (2+ star confidence threshold) with line and position data
- **Pickswise** — expert-selected player props and game bets
- **TheSportsGeek** — editorial picks

A fourth scraper (**SportyTrader**, model-generated predictions) exists in the codebase but is currently disabled pending a fix for upstream site changes.

Each source scraper handles its own Selenium/BeautifulSoup parsing in its own file and returns a shared schema: `Player`, `Sport`, `Stat`, `Line`, `Position` (Over/Under), `Bet Type`. The orchestrator standardizes each source's output, deduplicates across sources, and writes a multi-section CSV with separate Player Props and Game Bets blocks.

#### Step 2 — Entity Resolution (`players_and_teams_lookup_for_scraper.py`)

A small internal pipeline resolves each candidate to the identifiers needed downstream. A CSV parser extracts player and team names from the multi-section candidates file; team names are resolved to abbreviations using sport-aware matching (to avoid cross-sport collisions — e.g. "Atlanta" meaning the Braves vs. the Hawks); player names are matched fuzzily against ESPN's entity graph, consulting a curated alias file for ambiguous cases (e.g. mapping a scraped name to its canonical "Jr." form) and a shared name-normalization helper for accented characters. Any unresolved players are flagged to the console for manual lookup before the gamelog scraper runs.

#### Step 3 — ESPN Game Log Scraping (`gamelog_scraper_v5.9.6.py`)

The largest and most complex component (~2,000 LOC). For each player and team in the candidate set, it fetches full-season game logs via ESPN's internal JSON API (`site.web.api.espn.com/apis/common/v3/sports/…`), extracting 75+ metrics including:

- **Players**: raw box score stats (PTS, AST, REB, STL, BLK, 3PM, MIN, TOI) and derived per-36-minute normalizations
- **Teams**: game-level margin, total score, win/loss, opponent context

Results are written to a two-sheet Excel file: `Player Gamelogs` and `Team Gamelogs`. Game logs are indexed sequentially (`Game_Index`) to enable the non-overlapping window logic used downstream.

#### Step 4 — Opponent Defense Scraping (`defense_ranks_scraper.py`)

Scrapes season-to-date opponent defensive stats from Basketball-Reference (NBA) and Baseball-Reference (MLB). Produces a ratio for each team × stat combination: how much more or less of a given stat their opponents produce relative to league average. For MLB, per-team opponent batting data is fetched from each team's B-Ref pitching page to compute exact TB, 2B, and 3B allowed ratios — more accurate than using H as a proxy. NHL and NFL don't have a defense-ranks source yet, so defense adjustment is currently limited to NBA and MLB (see Step 5).

#### Step 5 — Line Builder: Weighted Projection + Edge Calculation (`line_builder.py`, `run_line_builder.py`, `team_props.py`)

The core statistical layer. For each candidate bet:

**Projection (Sharp Bettor Weights):**

Projections are built from three non-overlapping historical windows with fixed weights calibrated to prioritize true talent over noise:

| Window      | Games (NBA / MLB / NHL) | Games (NFL)   | Weight |
|-------------|--------------------------|---------------|--------|
| Season      | All − last 10            | All − last 5  | 75%    |
| Recent      | Games 6–10                | Games 3–5     | 20%    |
| Very Recent | Last 5 games              | Last 3 games  | 5%     |

NBA player stats use per-36-minute normalizations when building the projection (hit-rate calculation in Step 6 always uses the raw stat, to keep the historical comparison honest) to account for variable minutes; individual games under 20 minutes played are excluded from all windows.

**Defense Adjustment:**

The weighted projection is scaled directly by the opponent's defensive ratio for that stat — a ratio of 0.932, for example, means the opponent allows 93.2% of league-average production, applied as a straight multiplier against the projection. For team totals, the ratio is applied to the blended 50/50 offense/defense projection rather than the raw season average, to avoid double-counting the opponent's strength. Team-level defense adjustment (spreads/totals) is currently gated to NBA and MLB only, since no defense-ranks source exists yet for NHL/NFL.

**Edge Calculation:**

`Edge = Projection − Betting Line`

A raw edge filter gates what advances to the Excel output. The default is 2.0 points for player props and spreads, 3.0 for totals, with lower per-(sport, stat) overrides for MLB's low-scoring counting stats — as low as 0.5 for hits, home runs, RBIs, and stolen bases. A separate floor-and-ceiling contract per stat also screens out scraper data errors (e.g. a stray decimal point, or a stub placeholder line where the real market line is materially different) before any projection is built. The file is structured for manual odds entry before Step 6.

#### Step 6 — True Edge Validation + Bayesian Filter (`validate_true_edge.py`)

After the user fills in the `Odds` column, this stage converts raw edge into expected value and applies the statistical filter stack:

**Bayesian Hit Rate:**

For each bet, the same three historical windows are used to compute hit rates (how often the player/team exceeded the line historically). Each window's hit rate is smoothed toward 0.50 using Laplace/Bayesian smoothing — equivalent to adding 10 "neutral" games to each window. This shrinks extreme hit rates on small samples while converging to the observed rate once ~30+ games are available. Smoothed window rates are combined with the same 75/20/5 weights used in Step 5 to produce a single composite hit rate.

**Qualification Gate:**

A bet qualifies only if it clears two independent thresholds: a flat minimum true edge (5% for player props, 8% for team bets), and a sport-specific minimum Bayesian hit-rate floor — 49.5% for NBA/NFL/NHL, and a lower 35% floor for MLB, where most lines sit at 0.5 for rare events (home runs, RBIs) and even a strong projection lands in the 35–48% range.

**Volatility Signal (CV):**

Coefficient of variation (standard deviation / mean) is computed over the recent window as a volatility signal and surfaced as a warning at two severity levels. A design for using CV to dynamically raise the required edge on volatile performers exists in configuration but is deliberately held inactive until enough graded bets accumulate to validate it empirically — flat, uncalibrated thresholds are preferred over guessing at a second free parameter.

**Deduplication:** Bets are deduplicated same-player-same-stat, then same-player-across-stats, keeping the highest-EV version each time. A hard cap of 1 bet per game (across all market types) is then enforced; bets displaced by that cap are written to a Runner-Up sheet rather than discarded.

#### Step 7 — BettingPros Market Enrichment + MLB Context (`bettingpros_info_fetcher.py`, `mlb_context.py`)

The final output is enriched with current market data from BettingPros: line movement, consensus percentages, and book-level availability — the context needed to assess whether an identified edge is already priced in or still available. For MLB specifically, a supporting module adds game-context signals sourced from the MLB Stats API and a weather API: starting pitchers, batting-order slot, park factors, and weather effects (including wind blow-in/blow-out and roof state for retractable-roof stadiums).

---

## Paper-Trading & Result Tracking

Every qualified bet is logged and tracked against real-world outcomes — the system grades its own predictions rather than assuming the statistical filters work.

- **Ledger** — every bet that clears Steps 6–7 is appended to a JSON ledger with a flat 1-unit stake (the ground truth for P&L) and, in parallel, an informational Kelly-sized stake for comparison. MLB bets also get a 0–10 confidence score derived from contextual signals (venue, line movement, batting order, park factors, weather).
- **Automated resolution** — MLB bets are resolved automatically against the MLB Stats API on a daily schedule (macOS `launchd`), with a secondary catch-up job that covers missed runs if the machine was asleep or off at the scheduled time.
- **Backtesting** — a standalone tool retroactively re-applies the Kelly-sizing formula to historically resolved bets, to compare hypothetical Kelly P&L against the actual flat-stake ledger.
- **Model diagnostics** — a reporting tool breaks down why bets get filtered out at each pipeline stage, and cross-tabs win rate / P&L by hit-rate tier, stat type, and confidence level once enough resolved bets exist.

This closes the loop: rather than trusting the filtering stack (defense adjustment, Bayesian smoothing, EV gating) on faith, every recommendation is tracked to a real result and used to recalibrate the model over time.

---

## Technology Stack

- **Language**: Python 3, no frameworks
- **Scraping**: Selenium (dynamic pages) + BeautifulSoup (HTML parsing) + ESPN/BettingPros internal JSON APIs
- **Data sources**: ESPN (game logs), Basketball-Reference / Baseball-Reference (defense stats), OddsTrader, Pickswise, TheSportsGeek (active pick sources), BettingPros (market enrichment), MLB Stats API (game context + automated result resolution), a public weather API (MLB game-context)
- **Data**: pandas, numpy, openpyxl
- **Output format**: Date-stamped Excel files with named sheets
- **Automation**: macOS `launchd` scheduling for daily automated result resolution, with a catch-up poller
- **Tooling**: ruff (linting), mypy (type checking), pytest (241 tests across 17 modules), Makefile (`make check` runs all three)

---

## Running the Pipeline

```bash
# Steps 1-5, orchestrated, for today
python3 calculations/main.py

# Steps 1-5, orchestrated, for a specific date
python3 calculations/main.py 2026-03-10
```

The orchestrator prompts before re-running any of Steps 1–5 for which an output file already exists, making it safe to resume mid-pipeline or re-run individual stages.

After filling in the `Odds` column in the Step 5 output, Steps 6 and 7 are run as separate standalone commands:

```bash
python3 calculations/validate_true_edge.py 2026-03-10
python3 scrapers/bettingpros_info_fetcher.py \
    outputs/true_edge_output_2026-03-10.xlsx \
    outputs/qualified_bets_bettingpros_2026-03-10.xlsx \
    --date 2026-03-10
```

---

## Repository Structure

```
calculations/
  main.py                     Orchestrator — runs Steps 1-5 via subprocess
  config.py                   All tunable thresholds, weights, and parameters
  line_builder.py             Weighted projection + defense-ratio adjustment engine
  run_line_builder.py         Step 5 entry point
  team_props.py               Team bet screening (spread/total/ML) with opponent-side matching
  bayesian.py                 Bayesian hit-rate and CV calculations
  validate_true_edge.py       Step 6 entry point — EV calculation, qualification gate, deduplication
  utils.py                    Shared math utilities (odds conversion, EV, matchup IDs)
scrapers/
  unified_stage1_scraper.py   Step 1 orchestrator — runs each source scraper, standardizes + dedupes output
  oddstrader_scraper.py       OddsTrader consensus picks
  pickswise_scraper.py        Pickswise expert picks
  thesportsgeek_scraper.py    TheSportsGeek editorial picks
  sportytrader_scraper.py     SportyTrader model predictions (currently disabled)
  gamelog_scraper_v5.9.6.py   ESPN game log scraper (2,000+ LOC)
  defense_ranks_scraper.py    Opponent defensive ratio scraper (NBA + MLB)
  mlb_context.py              MLB enrichment: starting pitchers, lineups, park factors, weather
  bettingpros_info_fetcher.py BettingPros market data enrichment (Step 7)
player_team_lookup/
  players_and_teams_lookup_for_scraper.py  Step 2 orchestrator
  csv_parser.py                Multi-section candidates CSV parser
  team_abbr_lookup.py          Sport-aware team name → abbreviation resolution
  espn_ids.py                  Fuzzy name matching → ESPN player IDs
tools/
  log_paper_bets.py           Logs qualified bets to a paper-trade ledger; computes confidence + Kelly stake
  mlb_result_checker.py       Resolves MLB paper bets via the MLB Stats API (scheduled + on-demand)
  kelly_backtest.py           Retroactive Kelly-sizing backtest against resolved bets
  pipeline_stats.py           Rejection-funnel and win-rate/P&L analytics across pipeline runs
  install_scheduler.sh / uninstall_scheduler.sh / run_checker.sh / catch_up_checker.sh
                               macOS launchd automation for the daily result checker
data/                          Paper-trade ledger and result cache (gitignored)
tests/                         241 pytest tests across 17 modules covering core calculation and scraper logic
outputs/                       All intermediate and final outputs (date-stamped)
```

---

I'm happy to do a live walkthrough or answer any questions about the implementation. Reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.
