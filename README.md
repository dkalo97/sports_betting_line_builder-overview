# Line Builder

An 8-stage (+5.5) data pipeline that aggregates expert betting predictions from multiple sources, enriches them with historical performance data, and applies a statistical filtering stack — defense-adjusted projections, Bayesian hit-rate smoothing, expected-value gating, and a confidence-scoring layer — to surface only the highest-confidence betting opportunities for a given day. The architecture now spans three sport-families (traditional team sports, tennis, and soccer), each with its own day-scheduling and source scrapers, sharing a common projection/validation/logging core. The system runs from a single orchestrator command through the projection stage and produces structured Excel outputs at each stage, turning hours of manual research into a repeatable, auditable process.

---

## Why Private?

The pipeline generates the actual outputs I use for daily betting research. The source code is private because it's a working, actively maintained system — not a demo — and exposing the full production implementation would hand a direct competitor a calibrated, battle-tested starting point. The architecture and methodology are described here in full for technical evaluation purposes.

---

## Current Status

| Sport | Status |
|-------|--------|
| NBA | Live — real-money bets |
| MLB | Paper-trading validation, ongoing |
| Tennis (ATP/WTA) | Paper-trading validation, ongoing — full pipeline built out since NBA/MLB, including its own market-enrichment and result-resolution sources |
| Soccer | Pipeline built through the projection and validation stages; not yet paper-trading — confidence scoring and automated result resolution aren't wired up for it yet, and real bet volume is still low this early in a tracked season |
| NFL | Pipeline built end-to-end; not yet producing real candidates — blocked purely on the season's own pick sources posting lines, not a code gap |
| NHL | Not yet built |

The system is under active, iterative development. Thresholds and formulas are deliberately kept conservative (flat rather than adaptive) until enough graded bets accumulate to validate a more complex version empirically — see the CV and qualification-gate notes below for a concrete example of that discipline. A companion internal document tracks every "extra chance to qualify" mechanism the model has accumulated (alternate-line generation, cross-market reconciliation) against its own real, tagged performance data, specifically to catch the model quietly drifting back toward a simpler, previously-abandoned approach that didn't work — no enhancement is assumed to be a net positive without evidence.

---

## Technical Architecture

### Pipeline Overview

```
Stage 1   → candidates_{date}.csv                  Expert picks, normalized
Stage 2   → candidates_{date}_with_teams.csv        Player/team IDs resolved
Stage 3   → scraped_data_{date}.xlsx                Historical game logs, dozens of per-player/team metrics
Stage 4   → defense_ranks_{date}.csv                Opponent-quality ratios by stat category
Stage 5   → line_builder_output_{date}.xlsx         Weighted projections + raw edge vs. line
          ↑ [Stage 5.5: sportsbook odds auto-filled where a provider covers it;
             manual review/override otherwise]
Stage 6   → true_edge_output_{date}.xlsx            Bayesian-filtered, EV-qualified bets
Stage 7   → qualified_bets_{date}.xlsx              Market context enrichment
Stage 8   → paper-trade ledger                      Confidence-scored and logged for tracking
```

All intermediate files are date-stamped and retained. Each sport-family has its own orchestrator running Stages 1-5 sequentially; Stages 6-8 are run once the Odds column has been reviewed, not auto-chained, since that review step needs a human in the loop.

---

### Stage-by-Stage Breakdown

#### Stage 1 — Expert Pick Aggregation

An orchestrator per sport-family runs each source scraper as its own subprocess, then standardizes and deduplicates the combined output into a shared schema (`Player`/`Team`, `Stat`, `Line`, `Position`, `Bet Type`). Traditional team sports draw from 3 active consensus/editorial pick sources; tennis and soccer each draw from 4-5 of their own sport-specific sources, since neither shares any scraper code, output shape, or scheduling with the team-sports pipeline (tennis and soccer both scrape the evening before their slate, to catch early kickoffs the same-day team-sports pipeline doesn't need to worry about).

#### Stage 2 — Entity Resolution

A shared internal pipeline resolves each candidate to the identifiers needed downstream, dispatching internally to the right lookup mechanism for whichever sport-family the input belongs to. Team names are resolved via sport-aware matching (to avoid cross-sport collisions — e.g. a short code that happens to also be a substring of an unrelated team's name in a different league); player names are matched fuzzily against the relevant entity graph, consulting curated alias data for known ambiguous cases and a shared name-normalization helper for accented/transliterated names. Any unresolved players are flagged to the console for manual lookup before the game-log scraper runs.

#### Stage 3 — Historical Game Log Scraping

The largest and most complex component by far for team sports (thousands of lines of scraping logic). For each player and team in the candidate set, it fetches full-season game logs via each sport's own real data source, extracting dozens of metrics — raw box-score stats and derived per-minute normalizations for individual sports, game-level margin/total/opponent context for team-level bets. Tennis and soccer each use their own dedicated real data sources instead, matching the stat shape each sport actually needs (serve/return stats for tennis; match-level team stats for soccer).

#### Stage 4 — Opponent Quality Scraping

Scrapes season-to-date opponent-quality ratios: how much more or less of a given stat an opponent allows relative to a reference average. The reference pool itself varies by sport — traditional team sports and tennis each use one shared pool; soccer computes a separate ratio pool per domestic league plus a synthetic cross-league pool for continental competition, since a "0.9" ratio in one league isn't automatically comparable to a "0.9" in a much stronger one. A current-vs-prior-season blend is used early in a new season, weighted up as the current season's own sample grows large enough to stand on its own.

#### Stage 5 — Line Builder: Weighted Projection + Edge Calculation

The core statistical layer. For each candidate bet:

**Projection (Sharp Bettor Weights):**

Projections are built from three non-overlapping historical windows with fixed weights calibrated to prioritize true talent over noise — roughly a 75/20/5 split across season-long, recent, and very-recent windows, with sport-specific window sizes tuned to each sport's own game count and season length. Team-sport player stats use per-minute normalizations to account for variable playing time; individual games under a minimum playing-time threshold are excluded from all windows.

**Defense Adjustment:**

The weighted projection is scaled directly by the opponent's quality ratio for that stat — a ratio below 1.0 suppresses the projection, above 1.0 boosts it, applied as a straight multiplier. For team totals, the ratio is applied to a blended offense/defense projection rather than the raw season average, to avoid double-counting the opponent's strength.

**Edge Calculation:**

`Edge = Projection − Line`

A raw edge filter gates what advances, with sport- and stat-specific overrides for markets where a much smaller absolute edge is still meaningful (e.g. low-scoring counting stats). A separate floor-and-ceiling contract per stat also screens out scraper data errors (a stray decimal point, a placeholder line materially different from the real market) before any projection is built. When the model's own projection clears the scraped line by more than the qualifying threshold, some sports also generate tighter intermediate candidate lines between the scraped line and the projection, rather than only ever recommending the single original (often lopsided-priced) line — each additional candidate is tracked separately from a primary recommendation so its own performance can be checked against it later, not assumed to be equivalent.

#### Stage 5.5 — Sportsbook Odds Auto-Fill

Where a real-time odds provider covers the sport, the blank Odds column from Stage 5 is auto-filled directly from live sportsbook data, with a fallback provider for markets a primary source can't match. Coverage varies by sport-family and market — tennis-tier tournaments and a handful of leagues aren't covered by every provider, in which case the column stays blank for manual entry. A conflict-of-interest exclusion keeps my own employer's sportsbook out of every auto-fill source, checked manually instead.

#### Stage 6 — True Edge Validation + Bayesian Filter

After the Odds column is reviewed, this stage converts raw edge into expected value and applies the statistical filter stack:

**Bayesian Hit Rate:** the same historical windows are used to compute hit rates (how often the player/team actually exceeded the line historically), smoothed toward 50% to shrink extreme rates on small samples while converging to the observed rate once enough games accumulate. Smoothed window rates are combined with the same weighting scheme used in Stage 5 into one composite hit rate.

**Qualification Gate:** a bet qualifies only if it clears two independent thresholds — a flat minimum true edge, and a sport-specific minimum Bayesian hit-rate floor. The hit-rate floor in particular is tuned per sport: a sport where most lines sit near a rare-event boundary needs a much lower floor than one where lines cluster near a coin flip, or a real edge would never be able to clear it.

**Volatility Signal (CV):** coefficient of variation over the recent window is computed as a volatility signal and surfaced as a warning. A design for using it to dynamically raise the required edge on volatile performers exists in configuration but is deliberately held inactive until enough graded bets accumulate to validate it empirically — flat, uncalibrated thresholds are preferred over guessing at a second free parameter.

**Cross-Market Reconciliation:** when a qualifying moneyline candidate's own projected margin is unusually large (or a qualifying spread candidate's margin is unusually close to even), the system also evaluates the equivalent bet in the other market for the same matchup, keeping only whichever side wins on real expected value once both have real prices — a way of letting a mispriced favorite/underdog surface its more efficiently-priced alternative, without ever inventing a signal that wasn't already implied by the same underlying projection.

**Deduplication:** bets are deduplicated same-entity-same-stat, then across stats for the same entity, keeping the highest-EV version each time. A hard cap of one bet per game (across all market types) is then enforced; bets displaced by that cap are written to a runner-up sheet rather than discarded outright.

#### Stage 7 — Market Enrichment

The output is enriched with current market context: line movement, consensus percentages, book-level availability, and (for the traditional team-sports pipeline) game-context signals like starting lineups, park/venue factors, and weather effects for outdoor venues — the context needed to assess whether an identified edge is already priced in or still available, and the raw inputs the next stage's confidence score is built from.

#### Stage 8 — Confidence Scoring + Logging

Every qualified bet is scored using the market context Stage 7 just gathered — line movement direction, public-money splits, contextual factors relevant to that sport. For the most mature sport-family, this score is also a real second gate: a bet scoring below a minimum confidence threshold is rejected outright even though it already cleared Stage 6, on the theory that a real edge with strongly unfavorable context around it is less trustworthy than the raw statistical edge alone suggests. Other sport-families currently compute a score without gating on it yet, reflecting how much real graded history exists to validate a hard cutoff against — a gate is only added once there's real evidence it improves outcomes, not by default.

---

## Paper-Trading & Result Tracking

Every qualified bet is logged and tracked against real-world outcomes — the system grades its own predictions rather than assuming the statistical filters work.

- **Ledger** — every bet that survives Stage 8 is appended to a ledger with a flat 1-unit stake (the ground truth for P&L) and, in parallel, an informational Kelly-sized stake for comparison, scaled against a shared daily exposure cap so no single day's edges compound into an outsized bankroll swing.
- **Automated resolution** — the most mature sport-families are resolved automatically against their own real results data on a daily schedule, with a secondary catch-up job that covers missed runs if the machine was asleep or off at the scheduled time. Newer sport-families don't have this wired up yet — a real, tracked gap, not an oversight.
- **Backtesting** — a standalone tool retroactively re-applies the Kelly-sizing formula to historically resolved bets, to compare hypothetical Kelly P&L against the actual flat-stake ledger.
- **Model diagnostics** — a reporting tool breaks down why bets get filtered out at each pipeline stage, and cross-tabs win rate/P&L by hit-rate tier, stat type, confidence level, and — as of the most recent iteration — by which specific mechanism produced the recommendation (an original scraped line vs. a tightened alternate line vs. a cross-market reconciliation), once enough resolved bets exist to make each comparison meaningful.

This closes the loop: rather than trusting the filtering stack on faith, every recommendation is tracked to a real result and used to recalibrate the model over time — including whether a given enhancement is actually earning its keep, checked against real tagged outcomes rather than assumed.

---

## Technology Stack

- **Language**: Python 3, no frameworks
- **Scraping**: Selenium (dynamic pages) + BeautifulSoup (HTML parsing), coordinated browser automation for JS-heavy or bot-protected sites, plus several sources' own internal JSON APIs called directly
- **Data sources**: major sports-reference sites (game logs, opponent-quality data) per sport-family, several independent pick/tipster sources per sport, sportsbook APIs for market context and odds auto-fill, official league statistics APIs and a public weather API for game-context enrichment
- **Data**: pandas, numpy, openpyxl
- **Output format**: Date-stamped Excel files with named sheets
- **Automation**: scheduled daily jobs for automated result resolution, with a catch-up poller for missed runs
- **Tooling**: ruff (linting), mypy (type checking), pytest (2,200+ tests), Makefile (`make check` runs all three)

---

## Running the Pipeline

```bash
# Traditional team sports, Stages 1-5, orchestrated, for today
python3 calculations/main_american.py

# For a specific date
python3 calculations/main_american.py 2026-03-10

# Tennis / soccer each have their own orchestrator, day-before scheduled by default
python3 calculations/main_tennis.py
python3 calculations/main_soccer.py
```

Each orchestrator prompts before re-running any stage for which an output file already exists, making it safe to resume mid-pipeline or re-run individual stages. After reviewing the Odds column in the Stage 5 output, Stages 6-8 are run as separate standalone commands per sport-family.

---

## Repository Structure

```
calculations/
  main_american.py / main_tennis.py / main_soccer.py   Per-sport-family orchestrators
  helpers/                     Core projection/statistical engine, shared across sport-families
    config.py                  All tunable thresholds, weights, and per-sport parameters
    line_builder.py            Weighted projection + opponent-quality adjustment engine
    bayesian.py                Bayesian hit-rate and volatility calculations
    team_props.py              Team-bet screening with opponent-side matching
  run_line_builder.py          Stage 5 entry point (shared, dispatches by sport)
  validate_true_edge.py        Stage 6 entry point (shared, dispatches by sport)
scrapers/
  american/                    Traditional team-sports source scrapers + game-log/opponent-quality scrapers
  tennis/                      Tennis-specific source scrapers, odds provider, market-enrichment source
  soccer/                      Soccer-specific source scrapers, opponent-quality scraper (multi-league pools)
  bettingpros_fetcher.py       Shared market-enrichment engine (Stage 7)
player_team_lookup/
  players_and_teams_lookup_for_scraper.py   Stage 2 orchestrator, shared across sport-families
  helpers/                     Fuzzy name matching, team-abbreviation resolution, CSV parsing
tools/
  log_paper_bets.py            Stage 8: confidence scoring + ledger logging
  mlb_result_checker.py / tennis_result_checker.py     Automated result resolution
  analysis/                    Backtesting and diagnostic reporting tools
data/                          Paper-trade ledger and result cache (gitignored)
tests/                         2,200+ pytest tests covering the calculation engine and scraper logic
outputs/                       All intermediate and final outputs (date-stamped)
```

---

I'm happy to do a live walkthrough or answer any questions about the implementation. Reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.
