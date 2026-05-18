# Yahoo Fantasy Sports API Reference

## Base URL
`https://fantasysports.yahooapis.com/fantasy/v2`

## Authentication
All requests require an OAuth 2.0 Bearer token in the Authorization header.

## Response Format
All responses use `format=json` parameter. The Yahoo Fantasy API wraps all responses in a `fantasy_content` envelope.

## Common Response Patterns

### Array Pattern
Yahoo returns resources as a list (array) where:
```
[metadata_dict, {subresources_dict}]
```
- **index 0**: Metadata/properties array (list of single-key dicts)
- **index 1**: Sub-resources dict (e.g., `{roster: ...}`, `{standings: ...}`, `{player_stats: ...}`)

### Numeric-Keyed Container
Collections are stored as dicts with string/numeric keys:
```
{"0": {resource_type: [metadata, subresources]}, "1": {...}, "count": "N"}
```

## Endpoints

### User Games
```
GET /users;use_login=1/games?format=json
```
Response: `fantasy_content.users.{0: {user: [metadata, {games: {0: {game: [metadata, sub]}, ...}]}}}`

### User Leagues
```
GET /users;use_login=1/games;game_keys={key}/leagues?format=json
```
Response: Same nested structure as games.

### League Metadata
```
GET /league/{league_key}?format=json
```

### League Standings
```
GET /league/{league_key}/standings?format=json
```
Response format:
```
fantasy_content.league = [
  {league_metadata},                                    # [0]
  {standings: [{teams: {"0": {team: [props, stats]}, ...}}]}  # [1]
]
```
Each team is an array: `[props_array, {team_stats: ...}, {team_points: ...}, {team_standings: ...}]`

Props_array contains dicts like `{name: "..."}`, `{team_key: "..."}`, etc.

### League Scoreboard (Weekly Matchups)

```
GET /league/{league_key}/scoreboard?format=json
```
Returns matchups for the current week.

```
GET /league/{league_key}/scoreboard;week={week}?format=json
```
Returns matchups for a specific week.

```
GET /league/{league_key}/scoreboard;week={week}?format=json&matchup_metadata=true
```
Includes per-stat-category breakdown in each matchup.

Response structure:
```
fantasy_content.league = [
  {league_metadata},                                    # [0]
  {scoreboard: [
    {week: "N"},
    {matchups: {0: {matchup: [props, {teams: {0: {team: [team_id_props, {team_stats: ...}]}, 1: {...}}}]}, count: N}}
  ]}                                                    # [1]
]
```

#### Matchup Props (index 0)
List of single-key dicts containing:
- `week`: "1" (current week)
- `week_start`: "2024-03-25"
- `week_end`: "2024-03-31"
- `status`: "postevent" (one of "preevent", "postevent", "midevent")
- `is_playoffs`: "0" or "1"
- `is_consolation`: "0" or "1"
- `is_tied`: "0" or "1"
- `winner_team_key`: "{team_key}" (present if status is postevent)

#### Matchup Teams Structure
Each matchup contains a teams container at `matchup[1].teams`:
```
{teams: {
  "0": {team: [
    {team_key: "b1.l.7500.t.1"},
    {team_id: "1"},
    {name: "Team Name"},
    {is_owned_by_current_login: "0" or "1"},
    {team_points: {
      total: "55.0",
      week: "1"
    }},
    {team_projected_points: {total: "48.5"}},
    {team_stats: {stats: [{stat: {stat_id: "7", value: "55"}}, ...]}}
  ]},
  "1": {team: [
    ...
  ]}
}}
```

#### Matchup-Specific Notes
- Team array in a matchup includes `team_points` and `team_projected_points` but NOT full team props (no `managers`, `logos`, `waiver_priority`, etc.)
- `team_points.total`: The current score for that team this week
- `team_stats` contains per-stat-category totals (the raw stat values before scoring)
- With `matchup_metadata=true`, additional fields appear:
  - `stat_winner`: "1" or "0" for each stat category indicating which team won it
  - `stat_value`: The raw stat value for each category
  - `team_points` includes a breakdown by stat category with `points_earned`
- `status` values indicate: `preevent` = not yet started, `midevent` = in progress, `postevent` = completed

### Team Matchup

```
GET /team/{team_key}/matchup;week={week}?format=json
```
Returns a specific team's matchup for the given week.

Response structure:
```
fantasy_content.team = [
  [props],                                              # [0]
  {matchups: [
    {week: "N"},
    {matchups: {0: {matchup: [props, {teams: {0: {team: [...]}, 1: {...}}}]}, count: "1"}}
  ]}                                                    # [1]
]
```

Structure follows the same pattern as the league scoreboard matchup, scoped to a single team's matchup for the week.

### Team Metadata
```
GET /team/{team_key}?format=json
```
Response: `fantasy_content.team = [props_array, {sub_resources}]`

### Team Roster
```
GET /team/{team_key}/roster?format=json&week={week}
```
Response:
```
fantasy_content.team = [
  [props],
  {roster: {0: {players: {0: {player: [props, {player_stats}]}, count: N}}}}
]
```

### League Players (Free Agents/Waivers)
```
GET /league/{league_key}/players;status=FA;start=0;count=25?format=json
```
Parameters:
- `status`: FA (free agents), W (waivers), A (all)
- `start`: Offset for pagination
- `count`: Results per request (max 25)

Response:
```
fantasy_content.league = [
  {league_metadata},
  {players: {0: {player: [props, sub]}, count: N}}
]
```

### Player Stats
```
GET /league/{league_key}/players;player_keys={player_key}/stats;type=season?format=json
```
Note: The stats sub-resource is appended with a semicolon separator.

Alternative: `/player/{player_key}/stats;type=season?format=json`

Response:
```
fantasy_content = {
  "league": [
    {league_metadata},
    {players: {0: {player: [props, {player_stats: {stats: [{stat: {stat_id: "...", value: "..."}}, ...]}}]}}}
  ]
}
```

## Player Object Structure

Each player is returned as:
```
player: [props_list, {sub_resources_dict}]
```

### props_list (index 0)
List of single-key dicts containing:
- `name`: `{full: "Player Name", first: "First", last: "Last", ascii_first: "First", ascii_last: "Last"}`
- `player_key`: "l.10001.p.12345"
- `player_id`: "12345"
- `display_position`: "SP"
- `primary_position`: "SP"
- `status`: "IL" (if injured), null/absent otherwise
- `status_code`: "IL10" / "IL" / null
- `editorial_team_full_name`: "Toronto Blue Jays"
- `editorial_team_abbr`: "Tor"
- `uniform_number`: "16"
- `image_url`: "https://s.yimg.com/.../headshots/12345.jpg"
- `eligible_positions`: `{"position": "SP"}`, or a list `[{"position": "SP"}, {"position": "RP"}]`

### eligible_positions
Can be either:
- `{"position": "1B"}` - single position dict
- `[{"position": "OF"}, {"position": "RF"}, {"position": "LF"}]` - list of position dicts
- dicts
- `["1B", "OF"]` - list of strings

### sub_resources (index 1)
- `player_stats`: Dict containing player statistics
  - `stats`: `[{"stat": {"stat_id": "7", "value": "55"}}, ...]`
  - `coverage_type`: "season" or "week"
  - `week`: current week number
- `ownership`: Dict containing ownership/percent_owned data
  - `ownership.percent_owned.value`: "50"
  - `ownership.ownership_type`: "email" or "public" or "private"

## Statistics Format
Individual stats come as:
```json
{
  "stats": [
    {"stat": {"stat_id": "7", "value": "55"}},
    {"stat": {"stat_id": "12", "value": "12"}},
    {"stat": {"stat_id": "28", "value": "8"}},
    {"stat": {"stat_id": "50", "value": "120.1"}}
  ]
}
```

### Known Stat IDs
| ID | Category | Type |
|----|----------|------|
| 7  | R (Runs) | Hitting |
| 12 | HR (Home Runs) | Hitting |
| 13 | RBI (Runs Batted In) | Hitting |
| 16 | SB (Stolen Bases) | Hitting |
| 4  | AVG (Batting Average) | Hitting |
| 5  | OBP (On-Base Percentage) | Hitting |
| 3  | BA (Batting Average) | Hitting |
| 55 | OPS (On-Base + Slugging) | Hitting |
| 50 | IP (Innings Pitched) | Pitching |
| 26 | ERA (Earned Run Average) | Pitching |
| 27 | WHIP (Walks + Hits per IP) | Pitching |
| 57 | K (Strikeouts) | Pitching |
| 83 | SV (Saves) | Pitching |
| 89 | HLD (Holds) | Pitching |
| 28 | W (Wins) | Pitching |
| 32 | QS (Quality Starts) | Pitching |
| 22 | K (Strikeouts) | Hitting) | Hitting |

## Known Gotchas

1. **List-wrapping**: Teams and players are often wrapped in lists even when there's only one.
2. **Numeric keys**: Dicts use `"0"`, `"1"`, etc. as keys instead of arrays.
3. **Props array**: Properties are spread across multiple single-key dicts rather than one flat dict.
4. **Name field**: `name` can be a dict `{full: ...}` or a list `[{full: ...}, {ascii_first: ...}]`.
5. **eligible_positions**: Varies between dict (`{position: "OF"}`) and list (`[{position: "OF"}, ...]`).
6. **status field**: Can be "IL" (injured), "W" (waiver), or absent for active.
7. **league_scoring_type**: Field like `"head"` for H2H leagues.
8. **team_stats**: The stats array inside team_stats has entries like `{"stat": {"stat_id": "7", "value": "55"}}`.
9. **Percent owned**: Inside `ownership.percent_owned.value`.
10. **Auto-refresh**: 401 responses trigger automatic token refresh and retry.
11. **Stats sub-resource**: Stats are typically accessed by appending `/stats;type=season` to the resource URL.
12. **Player key format**: `{league_key}.p.{player_id}` e.g., `"b1.l.7500.p.12345"`.
13. **Matchup team array**: Teams inside a matchup have a reduced props list (no `managers`, `logos`, `waiver_priority`, etc.) compared to the full team endpoint.
14. **Scoreboard double-wrapping**: The scoreboard response wraps matchups in `scoreboard -> [week_meta, {matchups: {...}}]`, so index 0 of the scoreboard array is the week metadata dict.
15. **Matchup status strings**: `status` values in matchup props use `"preevent"`, `"midevent"`, `"postevent"` (not `"upcoming"`, `"in_progress"`, `"completed"`).
