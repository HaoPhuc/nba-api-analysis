# nba_api calls: what to use and what to drop

Audit of the 145 calls in the `nbaapi analysis/data understanding/` notebooks. Tested **2026-10-25** with **nba_api 1.12.0** on the **2025-26** season.

## At a glance

Every call was re-run with the notebooks' own inputs (Wembanyama, Durant, the Spurs, and the notebooks' `game_id` `0042500405`). The results were then compared against each other value by value. Together these 145 calls are the whole library: 139 `stats` endpoints, 4 `live` and 2 `static`.

| Verdict | Calls | What it means |
|---|---:|---|
| **Use** | **80** | 63 core calls, plus 17 optional ones that are useful but niche |
| **Drop** | **65** | 28 return no data · 30 duplicate another call · 7 aren't analysis data |

---

## Start here

These calls cover most analyses. Everything else in this document is detail.

| I want… | Call |
|---|---|
| Player / team IDs | `stats.static.players`, `stats.static.teams` (offline, no request) |
| A player table (bio, draft, college) | `playerindex(historical_nullable=1)` |
| Every game result in a season | `leaguegamelog` (one call, every team) |
| Every player's game log for a season | `playergamelogs` (one call, every player) |
| Season stats for all players | `leaguedashplayerstats` |
| Season stats for all teams | `leaguedashteamstats` |
| Tracking: speed, drives, passing, rebounding… | `leaguedashptstats` |
| Every shot with x/y | `shotchartdetail` |
| Every event in a game | `playbyplayv3`, or live `playbyplay` (richer) |
| A full box score for one game | `boxscoretraditionalv3` + the other `…v3` box scores |
| Lineups and on/off | `leaguedashlineups`, `teamplayeronoffsummary` |
| Standings | `leaguestandingsv3` |
| A player's career | `playercareerstats` |
| The schedule | `scheduleleaguev2` |
| Play types | `synergyplaytypes` |

**Tip:** `leaguedashplayerstats` and `leaguedashteamstats` take `measure_type_detailed_defense` (Base, Advanced, Misc, Scoring, Usage, Four Factors, Defense) and `per_mode_detailed` (Totals, PerGame, Per36, …). One endpoint therefore gives you seven different stat tables.

---

## Usable calls by topic

All calls are in `nba_api.stats.endpoints` unless noted.

### Players & teams

| Call | What you get |
|---|---|
| `stats.static.players` | Player name ↔ ID (5,198 players, offline) |
| `stats.static.teams` | Team abbreviation ↔ ID (offline) |
| `playerindex` | One row per player: position, height, weight, college, draft, years active |
| `commonplayerinfo` | One player's bio, including birthdate and seasons of experience |
| `commonteamroster` | Team roster for a season, plus the coaching staff |
| `playerawards` | Every award a player has won |

### Games, schedules & game logs

| Call | What you get |
|---|---|
| `leaguegamelog` | Every game of a season, one row per team (`'T'`) or per player (`'P'`) |
| `playergamelogs` | Player game logs with 72 columns; all season types |
| `teamgamelogs` | Team game logs with 57 columns; all season types |
| `leaguegamefinder` | Search games across seasons with filters |
| `scheduleleaguev2` | The full season schedule (1,400 games) |

Optional:

- `scoreboardv3` gives one date's games.
- `scoreboardv2` is worth keeping **only** for its conference standings on any date.
- `playergamestreakfinder` finds streaks such as 3+ games with 20+ points.

### Box score for one game

All of these take `game_id`. They return one row per player, plus team totals.

| Call | What you get |
|---|---|
| `boxscoretraditionalv3` | Counting stats, with starters vs bench |
| `boxscoreadvancedv3` | Ratings, pace, possessions, TS%, usage, PIE |
| `boxscorefourfactorsv3` | eFG%, FTA rate, TOV%, OREB% (own and opponent) |
| `boxscoremiscv3` | Points off turnovers, second-chance, fast-break and paint points |
| `boxscorescoringv3` | Shot mix and assisted/unassisted shares |
| `boxscoreusagev3` | Each player's share of team stats |
| `boxscoreplayertrackv3` | Speed, distance, touches, passes, contested shots |
| `boxscorehustlev2` | Deflections, charges, screen assists, box-outs |
| `boxscoredefensivev2` | Matchup defence per defender |
| `boxscorematchupsv3` | Who guarded whom, for how long, and the result |
| `boxscoresummaryv3` | Officials, arena, attendance, line score, last 5 meetings |

`boxscorehustlev2` and `boxscoredefensivev2` are the current versions; there is no V3 for either.

### Play-by-play & shots

| Call | What you get |
|---|---|
| `playbyplayv3` | One row per event (24 columns) |
| `live.nba.endpoints.playbyplay` | Richer events (56 columns): x/y, possession, linked assist/steal/block. Works for past games too. |
| `shotchartdetail` | Every shot with location and zone. Pass `context_measure_simple='FGA'` to include misses. |

Optional: `shotchartlineupdetail` returns the shots taken while one 5-man lineup was on the floor.

### Season stats: every player

| Call | What you get |
|---|---|
| `leaguedashplayerstats` | **The main season table** (see the tip above) |
| `leaguedashplayerbiostats` | Age, height, weight and draft for everyone who played (582 players) |
| `leaguedashplayerclutch` | Stats in clutch time |
| `leaguedashplayershotlocations` | FG by court zone |
| `leaguedashplayerptshot` | Shot tracking by closest defender, dribbles, touch time and shot clock |
| `leaguedashptdefend` | Opponent FG% when this player defends |
| `leaguehustlestatsplayer` | Season hustle stats |
| `playerestimatedmetrics` | Estimated ratings, pace and rebound/assist rates |

### Season stats: every team

| Call | What you get |
|---|---|
| `leaguedashteamstats` | **The main team table** |
| `leaguedashptstats` | **Tracking.** `pt_measure_type` = SpeedDistance, Rebounding, Passing, Drives, CatchShoot, PullUpShot, Defense, Possessions, … Set `player_or_team='Player'` for player rows. |
| `leaguedashteamptshot` | Team shot tracking |
| `leaguedashoppptshot` | Opponent shot tracking |
| `leaguedashptteamdefend` | Opponent FG% when defended |
| `leaguedashteamshotlocations` | Team FG by zone |
| `leaguedashteamclutch` | Team clutch stats |
| `leaguehustlestatsteam` | Team hustle stats |
| `teamestimatedmetrics` | Team estimated ratings and pace |

### Splits (player & team dashboards)

Each dashboard returns an "Overall" row that's identical to `leaguedashplayerstats`/`leaguedashteamstats`. Only the split tables are new.

| Call | Splits |
|---|---|
| `playerdashboardbygeneralsplits` | Home/away, wins/losses, month, before/after the All-Star break, starter/bench, rest days |
| `playerdashboardbygamesplits` | By half, quarter and score margin |
| `playerdashboardbyshootingsplits` | By distance, area, shot type, and **who assisted** |
| `playerdashptpass` | Passes to and from each teammate (a pass network) |
| `teamdashboardbygeneralsplits` | The team version of general splits. Don't use `plus_minus='Y'`: it returns wrong values. |
| `teamdashboardbyshootingsplits` | The team version of shooting splits |

Optional:

- `playerdashboardbyclutch` gives 10 different clutch definitions.
- `playerdashboardbylastngames` covers the last 5/10/15/20 games.
- `playerdashboardbyteamperformance` splits by team score differential.
- `playerdashboardbyyearoveryear` is per-season and overlaps `playercareerstats`.

### Lineups, on/off & matchups

| Call | What you get |
|---|---|
| `leaguedashlineups` | Every lineup's stats; filter by team with `team_id_nullable` |
| `teamplayeronoffsummary` | Team net, offensive and defensive rating with each player on vs off |
| `teamplayeronoffdetails` | The full stat line with each player on vs off |
| `leagueseasonmatchups` | Season-long head-to-head matchup stats |
| `synergyplaytypes` | Play types (Isolation, Transition, P&R, …): points per possession, frequency, percentile |

Optional:

- `leaguelineupviz` gives lineup ratings with a minutes cut-off.
- `matchupsrollup` shows who a defender guarded, by position.
- `playervsplayer` gives stats with a specific opponent on vs off the floor.

### Career, history & franchises

| Call | What you get |
|---|---|
| `playercareerstats` | Season and career totals (regular season, playoffs, All-Star, college), plus highs |
| `teamyearbyyearstats` | A franchise's every season: record and playoff result |
| `franchisehistory` | Totals for every franchise, including defunct ones |
| `franchiseplayers` | Every player's totals with one franchise |
| `teamdetails` | Arena, owner, titles, Hall of Famers, retired numbers |

Optional: `alltimeleadersgrids` lists all-time career leaders by stat.

### Standings & playoffs

| Call | What you get |
|---|---|
| `leaguestandingsv3` | Standings with seeding, clinch flags, home/road and splits |
| `iststandings` | NBA Cup group standings |
| `commonplayoffseries` | Playoff games with series ID and game number |

### Draft & combine

| Call | What you get |
|---|---|
| `drafthistory` | Every pick: round, number, team, school |
| `draftcombinestats` | Combine measurements, athletic drills and shooting in one table |

Optional: `draftcombinespotshooting` and `draftcombinenonstationaryshooting` give made/attempted shots per spot. Both are sparse: only 43 of 81 players have data.

### New tracking metrics (optional)

- `gravityleaders` gives each player's gravity score: how much defensive attention the player draws.
- `dunkscoreleaders` has one row per dunk, with vertical, hang time and style scores.

### Live

- `live.nba.endpoints.boxscore` (optional) is for games **in progress**. For finished games it repeats `boxscoretraditionalv3`.

---

## Calls to drop

### Return no data (28)

**The server sends nothing back.** These 11 are deprecated in nba_api. They return either `{}` or the NBA.com homepage instead of data.

| Call | Use instead |
|---|---|
| `boxscoreadvancedv2`, `boxscorefourfactorsv2`, `boxscoremiscv2`, `boxscorescoringv2`, `boxscoreusagev2` | The matching `…v3` |
| `playbyplay`, `playbyplayv2` | `playbyplayv3` |
| `leagueleaders`, `assistleaders` | `leaguedashplayerstats` |
| `playoffpicture` | `leaguestandingsv3` |
| `assisttracker` | — |

**Server error (HTTP 500) for every input tried.** There are 15 of these. Many hang for about 30 seconds before failing. nba_api doesn't mark them deprecated, so they might come back. Re-test them later.

| Call | Use instead |
|---|---|
| `playerdashptreb`, `teamdashptreb`, `teamdashptpass` | `leaguedashptstats` (Rebounding / Passing) |
| `playerdashptshots`, `teamdashptshots` | `leaguedashplayerptshot`, `leaguedashteamptshot` |
| `gamerotation` | Substitution events in live `playbyplay` |
| `teamgamestreakfinder` | Compute from `teamgamelogs` |
| `defensehub` | `leaguedashptdefend` |
| `teamandplayersvsplayers` | `playervsplayer` |
| `draftboard` | `drafthistory` |
| `playercareerbycollege` | `playerindex` (filter on `COLLEGE`) |
| `teamhistoricalleaders` | `franchiseplayers` |
| `winprobabilitypbp`, `glalumboxscoresimilarityscore`, `videodetails` | — |

**Return 0 rows.** Both were tried on two seasons.

| Call | Use instead |
|---|---|
| `playerdashptshotdefend` | `leaguedashptdefend` |
| `playerfantasyprofilebargraph` | `leaguedashplayerstats` |

### Duplicates (30)

These still work, but another call returns the same data. Rows marked *(verified)* were compared value by value.

| Drop | Use instead | Why |
|---|---|---|
| `boxscoretraditionalv2` | `boxscoretraditionalv3` | Identical *(verified)*; deprecated |
| `boxscoresummaryv2` | `boxscoresummaryv3` | Mostly empty for games after 2025-04-10 |
| `hustlestatsboxscore` | `boxscorehustlev2` | Identical *(verified)* |
| `playergamelog` | `playergamelogs` | Subset *(verified)*; regular season only |
| `teamgamelog` | `teamgamelogs` | Subset; its `TOV` leaves out team turnovers |
| `cumestatsplayergames`, `cumestatsteamgames` | game logs | Just a list of game IDs *(verified)* |
| `cumestatsplayer`, `cumestatsteam` | game logs | Sums over chosen games; aggregate the logs yourself |
| `playerprofilev2` | `playercareerstats` | Same career tables *(verified)* |
| `commonallplayers` | `playerindex` | Same players; fewer columns |
| `commonteamyears` | `franchisehistory` | A team list with first/last year |
| `teaminfocommon` | `leaguestandingsv3` | One team's record and 4 ranks |
| `franchiseleaders` | `franchiseplayers` | Its top row *(verified)* |
| `teamplayerdashboard` | `leaguedashplayerstats(team_id_nullable=…)` | Identical *(verified)* |
| `teamdashlineups` | `leaguedashlineups` | Part of the league call's output *(verified)* |
| `leagueplayerondetails` | `teamplayeronoffdetails` | Identical *(verified)* |
| `leaguestandings` | `leaguestandingsv3` | V3 has 13 more columns |
| `scheduleleaguev2int` | `scheduleleaguev2` | Same games; adds international broadcasters |
| `playernextngames` | `scheduleleaguev2` | The next few games on the schedule |
| `shotchartleaguewide` | `shotchartdetail` | Its league-average table |
| `homepageleaders`, `homepagev2`, `leaderstiles` | `leaguedashplayerstats` | Home-page top-5 widgets |
| `playercompare` | `leaguedashplayerstats` | Two players' rows side by side |
| `draftcombineplayeranthro`, `draftcombinedrillresults` | `draftcombinestats` | Identical subsets *(verified)* |
| `fantasywidget`, `infographicfanduelplayer` | `leaguedashplayerstats` / box scores | Fantasy points are a formula over box-score stats |
| live `scoreboard` | `scoreboardv3` | Today's games only |

### Not analysis data (7)

- `videodetailsasset`, `videoevents`, `videoeventsasset` and `videostatus` return video links and availability flags, not stats.
- live `odds` has today's betting odds only, with no history.
- `teamvsplayer` ignores `team_id` and returns all 30 teams, and its on/off tables are empty.
- `playercareerbycollegerollup` puts colleges into an NCAA-style bracket; a novelty.

---

## Watch out for

1. **The notebooks' `game_id` is a playoff game.** `leaguegamefinder` returns every season type by default, so "the last game of the season" is Finals Game 5. Pass `season_type_nullable='Regular Season'` if you want the regular season.
2. **Dates come in four formats:** `Apr 10, 2026`, `APR 12, 2026`, `2026-06-13T00:00:00` and `2026-06-13`. Parse them before joining tables.
3. **`MIN` means different things:** `"39:10"` in box scores, 240 player-minutes in `teamgamelog`, and 48.0 game minutes in `teamgamelogs`.
4. **V2 box scores leave stats blank (NaN) for players who didn't play; V3 uses 0.** Fill with 0 before summing.
5. **`playerindex(season=…)` returns only 241 players.** For everyone who played that season, use `leaguedashplayerbiostats` (582).
6. **`leaguegamefinder(game_id_nullable=…)` is ignored by the API.** Filter by game ID in pandas instead.
7. **Deprecated calls will be removed.** nba_api already warns on 13 endpoints. All of them are in the drop lists except `scoreboardv2`.

## Notebook fixes

- `player-comparisons-and-matchups.ipynb` uses `playergamelog` without importing it, so it raises `NameError`.
- `shot-charts.ipynb` uses `leaguedashlineups` without importing it, so it raises `NameError`.
- The same setup cell is pasted into 24 notebooks. Load it once from `example-inputs/inputs.ipynb` instead.
- None of the notebooks have saved outputs, so these failures weren't visible.

---

## Appendix: every call A–Z

| Call | Verdict | Use instead |
|---|---|---|
| `alltimeleadersgrids` | Optional |  |
| `assistleaders` | Dead | `leaguedashplayerstats` |
| `assisttracker` | Dead |  |
| live `boxscore` | Optional |  |
| `boxscoreadvancedv2` | Dead | `boxscoreadvancedv3` |
| `boxscoreadvancedv3` | Core |  |
| `boxscoredefensivev2` | Core |  |
| `boxscorefourfactorsv2` | Dead | `boxscorefourfactorsv3` |
| `boxscorefourfactorsv3` | Core |  |
| `boxscorehustlev2` | Core |  |
| `boxscorematchupsv3` | Core |  |
| `boxscoremiscv2` | Dead | `boxscoremiscv3` |
| `boxscoremiscv3` | Core |  |
| `boxscoreplayertrackv3` | Core |  |
| `boxscorescoringv2` | Dead | `boxscorescoringv3` |
| `boxscorescoringv3` | Core |  |
| `boxscoresummaryv2` | Duplicate | `boxscoresummaryv3` |
| `boxscoresummaryv3` | Core |  |
| `boxscoretraditionalv2` | Duplicate | `boxscoretraditionalv3` |
| `boxscoretraditionalv3` | Core |  |
| `boxscoreusagev2` | Dead | `boxscoreusagev3` |
| `boxscoreusagev3` | Core |  |
| `commonallplayers` | Duplicate | `playerindex` |
| `commonplayerinfo` | Core |  |
| `commonplayoffseries` | Core |  |
| `commonteamroster` | Core |  |
| `commonteamyears` | Duplicate | `franchisehistory` |
| `cumestatsplayer` | Duplicate | `playergamelogs` |
| `cumestatsplayergames` | Duplicate | `playergamelogs` |
| `cumestatsteam` | Duplicate | `teamgamelogs` |
| `cumestatsteamgames` | Duplicate | `teamgamelogs` |
| `defensehub` | Dead | `leaguedashptdefend` |
| `draftboard` | Dead | `drafthistory` |
| `draftcombinedrillresults` | Duplicate | `draftcombinestats` |
| `draftcombinenonstationaryshooting` | Optional |  |
| `draftcombineplayeranthro` | Duplicate | `draftcombinestats` |
| `draftcombinespotshooting` | Optional |  |
| `draftcombinestats` | Core |  |
| `drafthistory` | Core |  |
| `dunkscoreleaders` | Optional |  |
| `fantasywidget` | Duplicate | `leaguedashplayerstats` |
| `franchisehistory` | Core |  |
| `franchiseleaders` | Duplicate | `franchiseplayers` |
| `franchiseplayers` | Core |  |
| `gamerotation` | Dead | live playbyplay |
| `glalumboxscoresimilarityscore` | Dead |  |
| `gravityleaders` | Optional |  |
| `homepageleaders` | Duplicate | `leaguedashplayerstats` |
| `homepagev2` | Duplicate | `leaguedashplayerstats` |
| `hustlestatsboxscore` | Duplicate | `boxscorehustlev2` |
| `infographicfanduelplayer` | Duplicate | box scores |
| `iststandings` | Core |  |
| `leaderstiles` | Duplicate | `leaguedashplayerstats` |
| `leaguedashlineups` | Core |  |
| `leaguedashoppptshot` | Core |  |
| `leaguedashplayerbiostats` | Core |  |
| `leaguedashplayerclutch` | Core |  |
| `leaguedashplayerptshot` | Core |  |
| `leaguedashplayershotlocations` | Core |  |
| `leaguedashplayerstats` | Core |  |
| `leaguedashptdefend` | Core |  |
| `leaguedashptstats` | Core |  |
| `leaguedashptteamdefend` | Core |  |
| `leaguedashteamclutch` | Core |  |
| `leaguedashteamptshot` | Core |  |
| `leaguedashteamshotlocations` | Core |  |
| `leaguedashteamstats` | Core |  |
| `leaguegamefinder` | Core |  |
| `leaguegamelog` | Core |  |
| `leaguehustlestatsplayer` | Core |  |
| `leaguehustlestatsteam` | Core |  |
| `leagueleaders` | Dead | `leaguedashplayerstats` |
| `leaguelineupviz` | Optional |  |
| `leagueplayerondetails` | Duplicate | `teamplayeronoffdetails` |
| `leagueseasonmatchups` | Core |  |
| `leaguestandings` | Duplicate | `leaguestandingsv3` |
| `leaguestandingsv3` | Core |  |
| `matchupsrollup` | Optional |  |
| live `odds` | Not data |  |
| live `playbyplay` | Core |  |
| `playbyplay` | Dead | `playbyplayv3` |
| `playbyplayv2` | Dead | `playbyplayv3` |
| `playbyplayv3` | Core |  |
| `playerawards` | Core |  |
| `playercareerbycollege` | Dead | `playerindex` |
| `playercareerbycollegerollup` | Not data |  |
| `playercareerstats` | Core |  |
| `playercompare` | Duplicate | `leaguedashplayerstats` |
| `playerdashboardbyclutch` | Optional |  |
| `playerdashboardbygamesplits` | Core |  |
| `playerdashboardbygeneralsplits` | Core |  |
| `playerdashboardbylastngames` | Optional |  |
| `playerdashboardbyshootingsplits` | Core |  |
| `playerdashboardbyteamperformance` | Optional |  |
| `playerdashboardbyyearoveryear` | Optional |  |
| `playerdashptpass` | Core |  |
| `playerdashptreb` | Dead | `leaguedashptstats` |
| `playerdashptshotdefend` | Dead | `leaguedashptdefend` |
| `playerdashptshots` | Dead | `leaguedashplayerptshot` |
| `playerestimatedmetrics` | Core |  |
| `playerfantasyprofilebargraph` | Dead | `leaguedashplayerstats` |
| `playergamelog` | Duplicate | `playergamelogs` |
| `playergamelogs` | Core |  |
| `playergamestreakfinder` | Optional |  |
| `playerindex` | Core |  |
| `playernextngames` | Duplicate | `scheduleleaguev2` |
| `playerprofilev2` | Duplicate | `playercareerstats` |
| `static.players` | Core |  |
| `playervsplayer` | Optional |  |
| `playoffpicture` | Dead | `leaguestandingsv3` |
| `scheduleleaguev2` | Core |  |
| `scheduleleaguev2int` | Duplicate | `scheduleleaguev2` |
| live `scoreboard` | Duplicate | `scoreboardv3` |
| `scoreboardv2` | Optional |  |
| `scoreboardv3` | Optional |  |
| `shotchartdetail` | Core |  |
| `shotchartleaguewide` | Duplicate | `shotchartdetail` |
| `shotchartlineupdetail` | Optional |  |
| `synergyplaytypes` | Core |  |
| `teamandplayersvsplayers` | Dead | `playervsplayer` |
| `teamdashboardbygeneralsplits` | Core |  |
| `teamdashboardbyshootingsplits` | Core |  |
| `teamdashlineups` | Duplicate | `leaguedashlineups` |
| `teamdashptpass` | Dead | `leaguedashptstats` |
| `teamdashptreb` | Dead | `leaguedashptstats` |
| `teamdashptshots` | Dead | `leaguedashteamptshot` |
| `teamdetails` | Core |  |
| `teamestimatedmetrics` | Core |  |
| `teamgamelog` | Duplicate | `teamgamelogs` |
| `teamgamelogs` | Core |  |
| `teamgamestreakfinder` | Dead | `teamgamelogs` |
| `teamhistoricalleaders` | Dead | `franchiseplayers` |
| `teaminfocommon` | Duplicate | `leaguestandingsv3` |
| `teamplayerdashboard` | Duplicate | `leaguedashplayerstats` |
| `teamplayeronoffdetails` | Core |  |
| `teamplayeronoffsummary` | Core |  |
| `static.teams` | Core |  |
| `teamvsplayer` | Not data |  |
| `teamyearbyyearstats` | Core |  |
| `videodetails` | Dead |  |
| `videodetailsasset` | Not data |  |
| `videoevents` | Not data |  |
| `videoeventsasset` | Not data |  |
| `videostatus` | Not data |  |
| `winprobabilitypbp` | Dead |  |
