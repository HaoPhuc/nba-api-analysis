# nba_api call audit: which calls are worth using

This audit covers every call in the `nbaapi analysis/data understanding/` notebooks: 139 `stats.endpoints`, 4 `live.nba.endpoints` and 2 `stats.static` modules, which together make up the whole nba_api library. Each call was re-run with the same parameters the notebooks use, and its output was compared against every other call that looked like it might return the same data.

| | |
|---|---|
| Tested on | 2026-10-25, `nba_api` 1.12.0 |
| Inputs | Victor Wembanyama (1641705), Kevin Durant (201142), Spurs (1610612759), season `2025-26` |
| `game_id` | `0042500405`, 2026 Finals Game 5 (what the notebooks' "last game of the season" cell picks) |

## Result

| Verdict | Calls | Meaning |
|---|---:|---|
| ✅ Core | 63 | Works and returns data no other call gives you more cheaply |
| ➕ Optional | 17 | Works and is unique, but niche or sparse |
| ❌ Dead | 28 | Returns nothing: an empty `{}`, the NBA.com homepage HTML, HTTP 500, or 0 rows |
| 🔁 Duplicate | 30 | Works, but another call returns the same data (checked value by value where marked *verified*) |
| 🚫 Not analysis data | 7 | Video links, betting odds, novelty widgets, or results that ignore their filters |

So **80 calls are usable and 65 can be removed** from the notebooks.

---

## 1. Usable calls

Import them from `nba_api.stats.endpoints` unless the table says otherwise. In "Grain", *one row per* describes the main dataset.

### Reference / IDs

| Call | Grain | Notes |
|---|---|---|
| ✅ `stats.static.players` | player (5,198, all-time) | Offline: no HTTP request. Use it for name → ID lookup. |
| ✅ `stats.static.teams` | team (30) | Offline. Abbreviation/name → ID. |
| ✅ `playerindex` | player | The player dimension table: position, height, weight, college, country, draft, from/to year. `historical_nullable=1` returns all 5,197 players. |
| ✅ `commonplayerinfo` | one player | Adds birthdate, season experience, and headline stats. |
| ✅ `commonteamroster` | player on a team-season | Roster with age, experience and school, plus a **Coaches** table with the full staff. |
| ✅ `playerawards` | award | All of a player's awards, All-NBA teams, and Player of the Week/Month. |

### Games, schedule & game logs

| Call | Grain | Notes |
|---|---|---|
| ✅ `leaguegamelog` | team-game or player-game (whole league) | **One call per season** for every team (`player_or_team_abbreviation='T'`) or every player (`'P'`). Regular season by default. |
| ✅ `playergamelogs` | player-game | Superset of `playergamelog`: 72 columns (BLKA, PFD, DD2, TD3, ranks), all season types, and `measure_type_player_game_logs_nullable` for Advanced/Usage. Leave `player_id_nullable` empty to get every player (26,651 player-games for 2025-26 regular season in one call). |
| ✅ `teamgamelogs` | team-game | Superset of `teamgamelog` (57 columns, all season types). |
| ✅ `leaguegamefinder` | team-game or player-game | Flexible search across seasons with many filters. **Returns every season type by default** (preseason `1…`, regular `2…`, playoffs `4…`, NBA Cup final `6…`). |
| ✅ `scheduleleaguev2` | game | Full season schedule: 1,400 games with arena, week, series and broadcasters. |
| ➕ `scoreboardv3` | game on a date | One date at a time: line score, game and team leaders. Use the schedule plus game logs for season-level work. |
| ➕ `scoreboardv2` | date | Keep it **only** for `East/WestConfStandingsByDay` (standings as of any date), which V3 doesn't have. Otherwise skip it: it's deprecated, its LineScore is incomplete (1 of 2 teams), and it has known 2025-26 line-score bugs. |
| ➕ `playergamestreakfinder` | streak | Streaks matching stat thresholds, e.g. 3+ games with 20+ points. |

### Single-game box scores (`game_id`)

| Call | Grain | What it adds |
|---|---|---|
| ✅ `boxscoretraditionalv3` | player-game | Counting stats, plus starter/bench splits |
| ✅ `boxscoreadvancedv3` | player-game | Off/def/net rating, pace, possessions, PIE, TS%, usage |
| ✅ `boxscorefourfactorsv3` | player-game | eFG%, FTA rate, TOV%, OREB%, for the team and its opponent |
| ✅ `boxscoremiscv3` | player-game | Points off turnovers, second-chance, fast-break and paint points; blocks against; fouls drawn |
| ✅ `boxscorescoringv3` | player-game | Shot-mix and assisted/unassisted percentages |
| ✅ `boxscoreusagev3` | player-game | Each player's share of team stats |
| ✅ `boxscoreplayertrackv3` | player-game | Speed, distance, touches, passes, contested shots, rim defence |
| ✅ `boxscorehustlev2` | player-game | Deflections, charges, screen assists, box-outs, loose balls (this V2 is the current version) |
| ✅ `boxscoredefensivev2` | defender-game | Matchup defence totals per defender (no V3 exists) |
| ✅ `boxscorematchupsv3` | offence × defence pair | Who guarded whom, for how long, and the result |
| ✅ `boxscoresummaryv3` | game | Officials, arena, attendance, line score, last 5 meetings, other stats |

### Play-by-play & shots

| Call | Grain | Notes |
|---|---|---|
| ✅ `playbyplayv3` | event | The stats-site play-by-play (24 columns). |
| ✅ `live.nba.endpoints.playbyplay` | event | **Richer** than V3: 56 columns including x/y, area, possession, qualifiers, wall-clock time, linked assist/steal/block/foul-drawn players, and one row per player substitution. Works for past games too (tested back to 2019-20). |
| ✅ `shotchartdetail` | shot | Every shot with location and zone (pass `context_measure_simple='FGA'` to include misses), plus a league-average-by-zone table. |
| ➕ `shotchartlineupdetail` | shot by lineup | Shots taken while a given 5-man `group_id` was on the floor. |

### League-wide season stats: players (one row per player)

| Call | Notes |
|---|---|
| ✅ `leaguedashplayerstats` | **The main season table.** `measure_type_detailed_defense` = Base / Advanced / Misc / Scoring / Usage / Four Factors / Defense; `per_mode_detailed`; `team_id_nullable` gives a player's stats for one team only. |
| ✅ `leaguedashplayerbiostats` | Age, height, weight, college, country and draft for **every player who appeared that season** (582 in 2025-26), with a few advanced stats. |
| ✅ `leaguedashplayerclutch` | Season stats in clutch time. |
| ✅ `leaguedashplayershotlocations` | FGM/FGA/FG% by court zone (columns come back as a MultiIndex). |
| ✅ `leaguedashplayerptshot` | Shot tracking with filters for closest defender, dribbles, touch time and shot clock. Replaces the dead `playerdashptshots`. |
| ✅ `leaguedashptdefend` | Opponent FG% when defended, compared with normal FG%. Replaces the empty `playerdashptshotdefend`. |
| ✅ `leaguehustlestatsplayer` | Season hustle stats. |
| ✅ `playerestimatedmetrics` | Estimated ratings, pace and rebound/assist rates (some overlap with LDPS *Advanced* `E_*` columns). |

### League-wide season stats: teams (one row per team)

| Call | Notes |
|---|---|
| ✅ `leaguedashteamstats` | The main team table (same `measure_type` options). |
| ✅ `leaguedashptstats` | **The main tracking call.** `pt_measure_type` = SpeedDistance, Rebounding, Passing, Drives, CatchShoot, PullUpShot, Defense, Possessions, Elbow/Post/PaintTouch, Efficiency; `player_or_team='Player'` or `'Team'`. Replaces the dead `teamdashptreb`, `teamdashptpass` and `playerdashptreb`. |
| ✅ `leaguedashteamptshot` | Team shot tracking (same filters as the player version). Replaces `teamdashptshots`. |
| ✅ `leaguedashoppptshot` | The same shot tracking from the opponent's side. |
| ✅ `leaguedashptteamdefend` | Team opponent FG% when defended. |
| ✅ `leaguedashteamshotlocations` | Team FGM/FGA by zone. |
| ✅ `leaguedashteamclutch` | Team clutch stats. |
| ✅ `leaguehustlestatsteam` | Team hustle stats. |
| ✅ `teamestimatedmetrics` | Team estimated ratings and pace. |

### Splits: player & team dashboards

In every dashboard, the `Overall…` dataset is the same row you get from `leaguedashplayerstats`/`leaguedashteamstats` (*verified*), so only the **split** datasets are new.

| Call | Unique splits |
|---|---|
| ✅ `playerdashboardbygeneralsplits` | Home/away, wins/losses, month, before/after the All-Star break, starter/bench, days of rest |
| ✅ `playerdashboardbygamesplits` | Half, quarter, score margin, actual margin |
| ✅ `playerdashboardbyshootingsplits` | Distance, area, assisted/unassisted, shot type, **assisted-by player** |
| ✅ `playerdashptpass` | Passes made to and received from each teammate (a pass network) |
| ➕ `playerdashboardbyclutch` | 10 clutch definitions (the league call uses one) |
| ➕ `playerdashboardbylastngames` | Last 5/10/15/20 games, game-number buckets |
| ➕ `playerdashboardbyteamperformance` | By team score differential and points scored/allowed |
| ➕ `playerdashboardbyyearoveryear` | Per-season rows with ranks (largely overlaps `playercareerstats`) |
| ✅ `teamdashboardbygeneralsplits` | Team version of general splits. **Don't pass `plus_minus='Y'`**: the library warns it returns deltas instead of values. |
| ✅ `teamdashboardbyshootingsplits` | Team shooting splits, including assisted-by |

### Lineups, on/off & matchups

| Call | Grain | Notes |
|---|---|---|
| ✅ `leaguedashlineups` | lineup | All lineups, optionally filtered by team (515 for SAS). Superset of `teamdashlineups`. |
| ➕ `leaguelineupviz` | lineup | Advanced lineup ratings with a `minutes_min` cut-off. |
| ✅ `teamplayeronoffdetails` | player × on/off | Team stats with each player on and off the floor. |
| ✅ `teamplayeronoffsummary` | player × on/off | Compact on/off net, offensive and defensive rating. |
| ✅ `leagueseasonmatchups` | offence × defence (season) | Season-long head-to-head matchup stats. |
| ➕ `matchupsrollup` | defender × position | Who a defender guarded, by position. |
| ➕ `playervsplayer` | player | Stats with a specific opponent on and off the floor. |
| ✅ `synergyplaytypes` | player or team × play type | Synergy play types (Isolation, Transition, P&R, …): PPP, frequency, percentile. |

### Career, history & franchise

| Call | Grain | Notes |
|---|---|---|
| ✅ `playercareerstats` | player-season | Regular season, playoffs, All-Star, college; career totals; season highs and career highs. |
| ✅ `teamyearbyyearstats` | team-season | A franchise's whole history in one call: record, playoff results, ranks. |
| ✅ `franchisehistory` | franchise | Totals for every franchise, including defunct ones. |
| ✅ `franchiseplayers` | player × franchise | Every player's totals with one franchise (franchise leaders can be derived from it). |
| ✅ `teamdetails` | team | Background, arena, owner, championships, Hall of Famers, retired numbers. |
| ➕ `alltimeleadersgrids` | top-N | All-time career leaders by stat (`topx` sets N). |

### Standings & playoffs

| Call | Notes |
|---|---|
| ✅ `leaguestandingsv3` | Superset of `leaguestandings`: adds playoff seeding, play-in clinch, total points, neutral-site record. |
| ✅ `iststandings` | NBA Cup (In-Season Tournament) group standings. |
| ✅ `commonplayoffseries` | Playoff games with `SERIES_ID` and game number. |

### Draft & combine

| Call | Notes |
|---|---|
| ✅ `drafthistory` | Every pick: round, pick, team, school. |
| ✅ `draftcombinestats` | One-call combine summary: anthropometrics, athletic drills and shooting. Superset of the anthro and drill calls. |
| ➕ `draftcombinespotshooting` | Made/attempted/% per spot. Sparse: 43 of 81 players in 2023. |
| ➕ `draftcombinenonstationaryshooting` | Off-dribble and on-the-move shooting. Equally sparse. |

### Tracking leaderboards (new metrics)

| Call | Notes |
|---|---|
| ➕ `gravityleaders` | Gravity score (on/off-ball, perimeter/interior) per player. |
| ➕ `dunkscoreleaders` | One row per dunk with biomechanics: vertical, hang time, ball speed, style subscores. |

### Live (cdn.nba.com)

| Call | Notes |
|---|---|
| ➕ `live.nba.endpoints.boxscore` | Box score for a **game in progress**. For finished games it repeats `boxscoretraditionalv3`/`miscv3` (*verified*: points, rebounds, assists, FGM and +/- identical for the same players). |

---

## 2. Calls to remove

### ❌ Dead: no data comes back (28)

| Call | What happens | Use instead |
|---|---|---|
| `boxscoreadvancedv2`, `boxscorefourfactorsv2`, `boxscoremiscv2`, `boxscorescoringv2`, `boxscoreusagev2` | Server returns `{}`, so nba_api raises `KeyError: 'resultSet'`; the library marks them deprecated | the matching `…v3` |
| `playbyplay`, `playbyplayv2` | Same empty `{}` | `playbyplayv3` or live `playbyplay` |
| `leagueleaders` | Returns NBA.com homepage HTML (`JSONDecodeError`); deprecated | `leaguedashplayerstats` |
| `assistleaders` | Homepage HTML; deprecated | `leaguedashplayerstats` / `leaguedashteamstats` |
| `assisttracker` | Homepage HTML; deprecated | none |
| `playoffpicture` | Homepage HTML; deprecated | `leaguestandingsv3` |
| `gamerotation` | HTTP 500 for every game tried (2019-20, 2023-24, 2025-26) | Rebuild stints from substitution events in live `playbyplay` |
| `winprobabilitypbp` | HTTP 500 for every game tried | none |
| `playerdashptreb` | HTTP 500, both seasons | `leaguedashptstats(player_or_team='Player', pt_measure_type='Rebounding')` |
| `playerdashptshots` | HTTP 500, both seasons | `leaguedashplayerptshot` with its closest-defender/dribble/touch filters |
| `playerdashptshotdefend` | 200 OK but 0 rows (2025-26 and 2023-24) | `leaguedashptdefend` |
| `teamdashptpass`, `teamdashptreb`, `teamdashptshots` | HTTP 500, both seasons | `leaguedashptstats(pt_measure_type='Passing' / 'Rebounding')`, `leaguedashteamptshot` |
| `teamgamestreakfinder` | HTTP 500, with and without filters | Compute streaks from `teamgamelogs` |
| `defensehub` | HTTP 500 | `leaguedashptdefend`, `leaguehustlestatsplayer` |
| `teamandplayersvsplayers` | HTTP 500 | `playervsplayer`, `teamplayeronoffdetails` |
| `glalumboxscoresimilarityscore` | HTTP 500 (a G League alumni tool) | none |
| `draftboard` | HTTP 500 (2023 and 2025) | `drafthistory` |
| `playercareerbycollege` | HTTP 500 (Texas, Kentucky) | `playerindex` filtered on `COLLEGE` + `playercareerstats` |
| `teamhistoricalleaders` | HTTP 500 | `franchiseplayers` |
| `videodetails` | HTTP 500 | (not analysis data anyway) |
| `playerfantasyprofilebargraph` | 200 OK but 0 rows (2025-26 and 2024-25) | `leaguedashplayerstats` (has `NBA_FANTASY_PTS`) |

nba_api 1.12.0 does **not** mark the HTTP-500 endpoints as deprecated, so they may be a server-side outage rather than permanent removals. Re-test before writing them off for good. Note that several of them **hang for about 30 seconds** before failing, which slows down a notebook run.

### 🔁 Duplicates: same data as another call (30)

| Drop | Keep | Evidence |
|---|---|---|
| `boxscoretraditionalv2` | `boxscoretraditionalv3` | *Verified*: all stats identical for all 30 players once V2's DNP blanks are read as 0. Deprecated ("no longer published as of 2025-26"). |
| `boxscoresummaryv2` | `boxscoresummaryv3` | For this game, its officials, other-stats, last-meeting and season-series datasets are empty (V3 has them filled) and its line score has 1 of 2 teams; the library warns that data is missing after 2025-04-10. |
| `hustlestatsboxscore` | `boxscorehustlev2` | *Verified* identical values; older column naming. |
| `playergamelog` | `playergamelogs` | *Verified*: 25 shared columns identical; it has fewer columns and regular season only. |
| `teamgamelog` | `teamgamelogs` | Subset. Watch out: its `TOV` **excludes team turnovers** (it disagrees with every other game log in 61% of games) and `MIN` is player-minutes (240). |
| `cumestatsplayergames` | `playergamelogs` | Returns only `MATCHUP, GAME_ID`: *verified* as the same 64 games. |
| `cumestatsteamgames` | `teamgamelogs` | Only `MATCHUP, GAME_ID`: the same 82 games. |
| `cumestatsplayer`, `cumestatsteam` | `playergamelogs` / box scores | Sums over chosen games; aggregate the logs yourself. |
| `playerprofilev2` | `playercareerstats` | *Verified* identical career tables; adds only preseason totals and a "next game" row. |
| `commonallplayers` | `playerindex` / `static.players` | The same players (5,197 historical either way); `playerindex` adds bio and draft columns. |
| `commonteamyears` | `static.teams` + `franchisehistory` | A team-ID list with first and last year. |
| `teaminfocommon` | `leaguestandingsv3` + `leaguedashteamstats` | One team's record and 4 stat ranks. |
| `franchiseleaders` | `franchiseplayers` | *Verified*: its 5 leaders are exactly the max of `franchiseplayers` for PTS/AST/REB/BLK/STL. |
| `teamplayerdashboard` | `leaguedashplayerstats(team_id_nullable=…)` | *Verified* identical for all 20 players. |
| `teamdashlineups` | `leaguedashlineups(team_id_nullable=…)` | *Verified*: its 250 lineups are identical to the 515-lineup league call. |
| `leagueplayerondetails` | `teamplayeronoffdetails` | *Verified* identical to the on-court dataset. |
| `leaguestandings` | `leaguestandingsv3` | V3 has 79 of the same columns plus 13 more (loses only the pre/post-All-Star records). |
| `scheduleleaguev2int` | `scheduleleaguev2` | The same 1,400 games; only international broadcaster columns differ (1,142 columns versus 206). |
| `playernextngames` | `scheduleleaguev2` | The next N games from the schedule. |
| `shotchartleaguewide` | `shotchartdetail` (`LeagueAverages`) | The same 20 zones; FG% matches in 19 of 20 zones. |
| `homepageleaders`, `homepagev2`, `leaderstiles` | `leaguedashplayerstats(per_mode_detailed='PerGame')` | Home-page top-5 widgets; all three return the identical top-5 list. |
| `playercompare` | `leaguedashplayerstats` | Two players' season rows side by side. |
| `draftcombineplayeranthro`, `draftcombinedrillresults` | `draftcombinestats` | *Verified* identical subsets. |
| `fantasywidget`, `infographicfanduelplayer` | `leaguedashplayerstats` / box scores | Fantasy points are a formula over box-score stats you already have. |
| `live.nba.endpoints.scoreboard` | `scoreboardv3` | Today's games only. |

### 🚫 Not analysis data or unreliable (7)

| Call | Why |
|---|---|
| `videodetailsasset`, `videoevents`, `videoeventsasset` | Video URLs and thumbnails, with no tables (`get_data_frames()` is empty). |
| `videostatus` | Whether video exists for a date. |
| `live.nba.endpoints.odds` | Today's betting odds as nested JSON, with no history. Only useful if you snapshot it daily. |
| `teamvsplayer` | **Ignores `team_id`**: returns all 30 teams, and its on/off-court datasets are empty. |
| `playercareerbycollegerollup` | Colleges seeded into an NCAA-style bracket; a novelty. |

---

## 3. Gotchas found during the audit

- **The notebooks' `game_id` is a playoff game.** `LeagueGameFinder` returns every season type, so "the Spurs' last game" is Finals Game 5 (`0042500405`), not the regular-season finale. Pass `season_type_nullable='Regular Season'` if that's what you intended.
- **Units differ between similar calls:** `MIN` is `"39:10"` (box scores), total player-minutes (`teamgamelog`), or decimal game minutes (`teamgamelogs`). `GAME_DATE` comes in four formats: `"Apr 10, 2026"` (`playergamelog`), `"APR 12, 2026"` (`teamgamelog`), `"2026-06-13T00:00:00"` (`…gamelogs`), and `"2026-06-13"` (`leaguegamelog`, `leaguegamefinder`). Parse dates before joining.
- **V2 box scores leave DNP stats as NaN, while V3 uses 0.** Fill with 0 before comparing or summing.
- **`playerindex(season='2025-26')` / `commonallplayers(is_only_current_season=1)` return only 241 players**, not everyone who played. For "everyone who played in season X", use `leaguedashplayerbiostats` or `leaguedashplayerstats` (582).
- **Deprecation clock:** nba_api 1.12.0 raises `DeprecationWarning` for 13 endpoints and says they "will be removed in a future version": 12 are in the ❌/🔁 lists, and the 13th is `scoreboardv2`, kept above only for standings-by-day.
- `leaguegamefinder(game_id_nullable=…)` is ignored by the API (library warning); filter client-side.

## 4. Issues in the notebooks themselves

- `player-comparisons-and-matchups.ipynb` calls `playergamelog.PlayerGameLog` without importing `playergamelog`, so it raises `NameError` when run top to bottom.
- `shot-charts.ipynb` calls `leaguedashlineups.LeagueDashLineups` without importing it, so it also raises `NameError`.
- The same setup cell (IDs, season, `game_id`) is copied into 24 notebooks; `example-inputs/inputs.ipynb` already holds it, so it could be `%run` or imported instead.
- None of the notebooks have saved outputs, so the failures above weren't visible in them.
