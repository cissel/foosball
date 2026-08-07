# foosball

NFL and fantasy football data tools. Two parts: classic NFL analytics plots (R, standalone) and a full fantasy football "moneyball" pipeline for Room 40 (Python + R, VORP/ADP draft prep, live draft tools, projections).

All scripts assume this repo is cloned to `~/foosball`. Outputs write to `outputs/`, trained models to `models/`.

---

## Setup

**R**: tidyverse, nflreadr, nflfastR, ggplot2, patchwork, plotly, fflr, ggimage, ggthemes, httr, rvest, wdman, tidytext, jsonlite (per-script, see each file's library() calls).

**Python**: pandas, numpy, scipy, scikit-learn, lightgbm, matplotlib, plotly, dash, requests, joblib.

---

## NFL Analytics (R, standalone scripts)

Not part of the fantasy pipeline. Each pulls its own data and can run independently.

### `r/epaMap.R`
![EPA Map](images/epaMap.png)
Team offense vs defense EPA per play, full season, plotted as a 2D scatter with team logos. Top-right teams are good on both sides of the ball. Best single-image "who's actually good" gut check, since EPA per play strips out garbage-time counting stats.

### `r/nflQbStats.R`
![QB Stats](images/nflQbStats.png)
CPOE (completion % over expected) vs aggressiveness scatter for QBs. High CPOE means accurate beyond what the throw difficulty predicts. High aggressiveness means throwing into tighter windows. Top-right quadrant is the rare accurate-and-aggressive passer.

### `r/nflRbStats.R`
![RB Stats](images/nflRbStats.png)
Rush yards over expected (RYOE) scatter for RBs, from NFL Next Gen Stats. Isolates a back's own contribution from the yards his offensive line created (expected yards is a per-play model based on defenders in the box, blocking, etc).

### `r/nflWrStats.R`
![WR Stats](images/nflWrStats.png)
Advanced WR receiving leaderboard: separation, YAC over expected, target share. Shows who wins from route-running/tracking skill versus who wins from raw target volume.

### `r/nflTeStats.R`
![TE Stats](images/nflTeStats.png)
Same as WR stats, filtered to TE. TEs get compared to a different baseline than WRs since role and target rate differ a lot by position.

### `r/nflOlStats.R`
![OL Stats](images/nflOlStats.png)
Pressure rate vs sack rate scatter for offensive linemen (pass-block reps only). Bottom-left is the good corner: pressures allowed and sacks allowed both low.

### `r/room40map.R`
![Room 40 Map](images/room40map.png)
League standings/roster visualization pulled live from the Sleeper API for league 1259616442014244864 (Room 40).

### `r/targetShare.R`
![Target Share](images/tgtShr.png)
Target share breakdown by player/team from full-season play-by-play. Useful for spotting who's actually the WR1 in a given offense versus who has the name recognition.

### `r/scorekeepe.R`
![Fantasy Scoreboard](images/fantasyScoreboard.png)
Weekly Room 40 fantasy scoreboard, pulled from Sleeper. Reads `outputs/sleeper_proj_pts.csv` and `outputs/players.json` (both fetched by other scripts in this repo).

### `r/nflPlaye.R`
Pulls the full Sleeper NFL player directory to `outputs/players.json`. Utility script, feeds `room40map.R` and `scorekeepe.R`.

### `r/nextNFL.R`
Pulls the next upcoming NFL game from the season schedule with a countdown. Small utility, no plot.

---

## Fantasy Moneyball Pipeline

Room 40 league settings: 12 teams, full PPR, QB/RB/RB/WR/WR/TE/FLEX/K/DEF/BN x5, no keepers. Built to answer "who do I draft and when" with data instead of gut feel, same rigor as the SPY trading model (VORP instead of raw projections, empirical league splits instead of assumed ones, leakage-safe time-based train/val splits).

Pipeline runs in phases. Later phases depend on earlier phases' output files.

### Phase 1: Feature building (R)

- **`r/fetchNflCareerData.R`**: builds the player-season panel with Room 40's exact scoring rules applied to raw nflverse stats. Output: `outputs/fantasy/career_panel.csv`.
- **`r/fetchNflSnapShare.R`**: snap share + target share per player-season. Separates "starter getting real volume" from "name-value backup."
- **`r/fetchNflSchemeFeatures.R`**: team-season play-calling tendency (pass rate over expected, pace).
- **`r/mergeNflCoachingStaff.R`**: merges coaching staff history into one team/season table.
- **`r/buildNflCoachSignature.R`**: builds a coach-level scheme signature that follows a coordinator across team changes (rather than resetting to league-average when a team hires someone new).
- **`r/mergeNflPhase1Features.R`**: joins usage features with coach scheme signature into the complete Phase 1 feature table.
- **`r/nflPhase1Diagnostics.R`**
  ![Phase 1 Diagnostics](images/phase1_diagnostics.png)
  4-panel sanity check on the Phase 1 feature build before modeling. Confirms the pipeline isn't quietly broken before spending time training on it.

### Phase 2: Projection model (Python + R)

- **`python/trainNflFantasyModel.py`**: trains the season-long fantasy point model (Ridge + LightGBM per position). Time-based train/val split, never random, since this is a time series problem.
- **`python/buildNflFlexSplit.py`**: derives Room 40's real FLEX slot allocation (RB/WR/TE) from actual historical scoring rather than guessing. Came out ~95% WR / 5% RB / 0% TE in a full-PPR league, replacing a stale hardcoded assumption.
- **`r/nflPhase2Diagnostics.R`**
  ![Phase 2 Diagnostics](images/phase2_model_diagnostics.png)
  5-panel model diagnostic: actual vs predicted by position, residuals, feature importance, Spearman comparison, MAE by position. Read this before trusting any projection number.
- **`python/buildNflAgingCurve.py`**
  ![Aging Curves](images/aging_curves.png)
  Position-specific career aging curves using the delta method (tracks each player's own year-over-year change, not a cross-sectional average, which avoids survivorship bias from only good old players still being in the league). Shows when a position typically peaks and declines.

### Phase 3: 2026 projections + VORP (Python)

- **`python/fetchNflAdpAndRoster.py`**: pulls current ADP from Sleeper.
- **`python/buildNfl2026Projections.py`**: applies the Phase 2 model to every player with current ADP, using real 2025 usage as input.
- **`python/buildNflVorp.py`**: the core "moneyball" layer. Converts projections to Value Over Replacement Player using Room 40's actual roster math and empirical FLEX split, then joins ADP to surface market inefficiencies (good value relative to where players are actually being drafted).
- **`r/nflPhase3Diagnostics.R`**
  ![Phase 3 VORP Diagnostics](images/phase3_vorp_diagnostics.png)
  4-panel check on the VORP/ADP layer before using it to draft.

### Phase 4: Draft day tools (Python)

- **`python/buildNflDraftBoard.py`**
  ![Draft Board](images/draft_board_2026.png)
  Tiered cheat sheet, one column per position. Tiers come from real gaps in projected value (1D KMeans on VORP), not arbitrary round cutoffs. Breakout candidates are starred.
- **`python/buildNflBigBoard.py`**
  ![Big Board](images/big_board_2026.png)
  Same data as the draft board but flattened into one position-agnostic ranked list. Use this if you just want "who's next" regardless of position.
- **`python/nflDraftLive.py`**: tracks who's been drafted during a live draft so the board always shows best-available. JSON state file, no UI.
- **`python/buildNflPositionalScarcity.py`**
  ![Positional Scarcity](images/positional_scarcity.png)
  Shows, round by round, which position has the steepest value cliff coming up. A big drop means that position is scarce and worth reaching for now; a flat line means it's safe to wait.
- **`python/buildNflDraftStrategySim.py`**
  ![Draft Strategy Simulation](images/draft_strategy_sim.png)
  Simulates full 12-team snake drafts under named strategies (zero RB, hero RB, robust RB, balanced, best-player-available) and scores each by the resulting STARTING LINEUP's total VORP, not just total value drafted (bench depth you can't start doesn't count). Answers which strategy actually wins.
- **`python/buildNflStrategyGuide.py`**
  ![Strategy Guide](images/strategy_guide.png)
  Turns the strategy sim into a round-by-round printable card: "if I'm running zero RB, what position do I actually take in round 4."
- **`python/nflMockDraftLog.py`**: grades a completed Sleeper mock draft against the VORP board and logs it so multiple mocks can be compared.
- **`python/nflDraftWatch.py`**: live CLI draft assistant. Polls a real Sleeper draft (mock or real) and prints pick recommendations ranked by marginal starting-lineup VORP gain after every pick.
- **`python/nflDraftDashboard.py`**
  ![Live Draft Dashboard](images/nflDraftDashboard.png)
  Browser UI version of the draft watcher (Dash/Plotly). Same recommendation engine as the CLI watcher, meant to run on your laptop next to the Sleeper tab during a live draft.

### Live draft tools: quick start (no coding experience needed)

Both `nflDraftWatch.py` (plain text in a terminal) and `nflDraftDashboard.py` (a webpage) watch a Sleeper draft in real time and tell you who to pick. Everything runs on your own laptop, nothing is uploaded anywhere. Pick whichever one sounds friendlier — the dashboard looks nicer, the CLI watcher is a little more lightweight.

**Before you start:** these tools need a `vorp_2026.csv` file (the actual player rankings/model output) sitting in `outputs/fantasy/` inside this folder. That file isn't included in the download since it's the output of the whole modeling pipeline, not source code. Ask James to send you his `outputs/fantasy` folder (zip it up, share via Discord/Drive/USB, whatever's easiest) and drop it into your copy of the repo before running anything below.

**One-time setup (5 minutes):**

1. Install Python from [python.org](https://www.python.org/downloads/) if you don't already have it (Windows/Mac installer, just click through the defaults). On the install screen, check the box that says "Add Python to PATH."
2. Download this repo: click the green "Code" button on the GitHub page, then "Download ZIP," and unzip it somewhere you'll remember (like your Desktop).
3. Open a terminal in that folder:
   - **Windows**: open the unzipped folder in File Explorer, click the address bar, type `cmd`, hit Enter.
   - **Mac**: open Terminal, type `cd ` (with a space), drag the unzipped folder into the terminal window, hit Enter.
4. Install the required packages (copy/paste this whole line, hit Enter, wait for it to finish):
   ```
   pip install pandas numpy requests matplotlib scikit-learn dash plotly
   ```

**Find your draft ID:**

Open your Sleeper mock or real draft in a browser. Look at the URL, it'll look something like:
```
https://sleeper.com/draft/nfl/1234567890123456789
```
The long number at the end is your draft ID. Copy it.

**Find your draft slot:**

This is just which pick number you go in round 1 (e.g. if you pick 5th, your slot is `5`).

**Run the CLI watcher:**
```
python python/nflDraftWatch.py 1234567890123456789 5
```
(replace the number with your draft ID and `5` with your slot). Leave this terminal window open during the draft — it'll print new picks and your recommendations automatically every few seconds. Press Ctrl+C to stop it when the draft ends.

Example output when it's your turn:
```
[14:32:07] Pick #34 is up next (slot 10).
>>> YOUR TURN NOW <<<
Your roster so far (2 picks): Amon-Ra St. Brown (WR), Omarion Hampton (RB)
Still need: QB (0/1), RB (1/2), WR (1/2), TE (0/1)

Top 8 recommendations right now (by marginal starting-lineup VORP gain):
        full_name position team  adp_overall       vorp  value_gap  marginal_gain
       Drake Maye       QB   NE        101.0 138.400604      100.0          138.4
      Chris Olave       WR   NO         38.0  88.806879       29.0           88.8
           Bo Nix       QB  DEN        105.0  67.374755       88.0           67.4
```
`marginal_gain` is the number that matters — it's how much your BEST STARTING LINEUP improves if you draft that specific player right now, not just their raw stat value. Higher is better.

**Run the dashboard instead (nicer visuals):**
```
python python/nflDraftDashboard.py 1234567890123456789 5
```
Then open a web browser and go to `http://127.0.0.1:8877` — that's a page only your own computer can see, nobody else on the internet can reach it. Leave the terminal window open in the background while you use the dashboard in your browser; closing the terminal shuts the dashboard down. Refresh the page any time to see it re-poll the draft.

**Troubleshooting:**
- `command not found: python` → try `python3` instead of `python` in the commands above.
- Nothing happens / hangs at "Draft not started yet" → the draft hasn't begun on Sleeper's end yet, or you've got the wrong draft ID. Double check the URL.
- `ModuleNotFoundError` → you missed the `pip install` step above, or need to re-run it.

### Phase 5: Breakout candidates + rookies (Python + R)

- **`r/fetchNflBreakoutSignals.R`**: pulls raw signal inputs (efficiency, usage trend) for 2nd/3rd-year breakout scoring.
- **`python/buildNflBreakoutScore.py`**: composite z-scored breakout score for young skill players. Flags candidates who haven't "arrived" yet by finishing production but show strong underlying signals.
- **`r/fetchNflRookieData.R`**: draft capital + combine testing for rookies with no NFL usage history.
- **`python/buildNflRookieLandingSpot.py`**: measures how much target/snap share is vacated on a rookie's new team, since a clear path to touches matters independent of talent.
- **`python/buildNflRookieTrainingPanel.py`** / **`python/trainNflRookieModel.py`**: separate small-sample model (Ridge only) predicting rookie-year points from draft capital, combine, and landing spot, since rookies have no prior-year usage for the main model.
- **`python/scoreNfl2026Rookies.py`**: scores the incoming rookie class with the rookie prior model.

---

## Notes

- Scripts write to `outputs/` on first run; this directory is gitignored except for the example images in `images/`.
- Trained models ship in `models/` (16 `.pkl` files, ~2MB) so the draft-day tools work out of the box without retraining.
- Live draft tools (`nflDraftWatch.py`, `nflDraftDashboard.py`, `nflDraftLive.py`) need a real or mock Sleeper draft ID to run against.
- All plots use a dark navy theme (`#02233F` / `#1c2e4a`) for consistency across the repo.
