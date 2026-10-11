# What game data does SIHF publish?

Research for [issue #2](https://github.com/obaume/firststar/issues/2). Investigated on **2026-10-11** (season 2026/27, a Sunday with live games), against SIHF's own sites and the endpoints they call. Everything marked **Verified** was observed directly in HTTP responses on that date. Everything marked **Not verified** is inference or comes from an older or secondary document.

## Short answer

- **All six v1 senior leagues are covered by the same SIHF data**: National League, Sky Swiss League, MyHockey League, 1. Liga, 2. Liga and 3. Liga. Every game has a schedule, a venue (street address plus a venue ID that joins to lat/lng coordinates), line-ups for both teams, and a live status (period, intermission, final).
- **There is no running game clock.** SIHF publishes the current period (`1. Drittel`, `Pause`, `2. Drittel`, ...), event timestamps (goals, penalties, goalie changes) and an `Ende` status with the actual end time. That is enough to detect the final buzzer within a few minutes.
- **Line-ups arrive late.** They come from the game sheet entered at the rink, so they usually appear around an hour before puck drop. They are not published days ahead.
- **Access is unofficial.** There is no documented public API. The data comes from (a) a JSONP endpoint on `data.sihf.ch` that the old SIHF site used, and (b) the new Game Center at `sihf.ch/de/game-center/`, built from server-rendered HTML fragments with a SignalR push hub. Both are undocumented, rate-limited (HTTP 429 seen) and could change without notice.
- **Terms of use do not allow a commercial SaaS** to reuse the data without SIHF's prior consent: personal, non-commercial use only. Firststar needs a data agreement with SIHF before it relies on this data in production.
- **Implication:** technically, Team Managers would not need to enter schedules, venues or rosters, and votes could open automatically when SIHF reports `Ende`. Legally and operationally, Firststar should get SIHF's permission and keep a manual fallback (TM enters or overrides game time, venue and roster, and opens or closes the vote), because lower-league data is sometimes incomplete and the access path is unofficial.

## Leagues and IDs

**Verified.** These are the IDs used by both data sources for season `2027` (2026/27). Source: the `League` filter returned by `data.sihf.ch` and the league navigation at `https://www.sihf.ch/de/game-center/suche/`.

| League | League ID | Region IDs |
|---|---|---|
| National League | 1 | 1 (CH) |
| Sky Swiss League | 2 | 1 (CH) |
| MyHockey League | 100 | 1 (national) |
| 1. Liga | 3 | 5 ("überregional"; groups Ost/West are phases) |
| 2. Liga | 10 | 2 Ostschweiz, 3 Zentralschweiz, 4 Westschweiz (+ 1 National for finals) |
| 3. Liga | 18 | 2, 3, 4 |

Note: the MyHockey League (ID 100) is listed in the 2026/27 game center. Older seasons in `data.sihf.ch` use different league lists, so IDs should be read from the filter response, not hard-coded.

Game IDs are 14-digit strings, e.g. `20271105000075` (NL) or `20270009252355` (3L).

## Data sources

### 1. `data.sihf.ch` statistics API (JSONP)

**Verified.** This undocumented endpoint powered the previous sihf.ch results pages. No authentication. It returns JSONP: wrap a `callback` name and strip the wrapper to get the JSON.

**Schedule and results, per league and date:**

```
GET https://data.sihf.ch/Statistic/api/cms/cache300
    ?alias=results
    &searchQuery=1,2,3,10,18,100//1,2,3,10,18,100
    &filterQuery=<season>/<leagueId>/<regionId|all>/<phaseId|all>/<dd.mm.yyyy|all>
    &filterBy=Season,League,Region,Phase,Date
    &orderBy=gameLocalTime&orderByDescending=false&take=300&skip=-1
    &callback=externalStatisticsCallback&language=de
```

Each row contains:

- date and local start time
- home and away team (`id`, `name`, `acronym`, `clubId`)
- score and per-period scores
- OT/SO marker
- a `gameStatus` object: `id`, `name`, `startDateTime`, `endDateTime`, `percent`, `canceled`, `updateDateTime`
- the `gameId`
- the TV broadcaster

`availableExports` lists `Csv`, `Ics` and `Pdf`.

**Game detail:**

```
GET https://data.sihf.ch/Statistic/api/cms/gameoverview
    ?alias=gameDetail&searchQuery=<gameId>
    &callback=externalStatisticsCallback&language=de
```

Returns:

- `league`, `region`, `phase`
- `status` (same `gameStatus` object)
- `result`, including shots on goal per period
- `summary.periods[]`: goals and penalties with `time` and `timeInSeconds`
- `lineUps.homeTeam` / `lineUps.awayTeam`: player IDs by position (goalkeepers, defenders L/R, forwards L/C/R), captain and alternates, coach
- `players[]`: `id`, `fullName`, `jerseyNumber`, `teamId`, birth year
- `details.venue`: `id`, `name`, `street`, `zip`, `city`, `country`, `spectators`
- `details.referees` / `details.linesmen`

**Freshness.** Responses carry `cache-control: max-age=40`. During the live NL game Kloten v Zug, `status.updateDateTime` was 15:16:10 when fetched at 15:16:44 local, about 30 s behind. A live U14 game was about 3 min behind. Schedules for the whole season are already published: Jan 2027 games had `updateDateTime` values from July/August 2026.

**Rate limiting.** Bursts of roughly 100 sequential requests got `HTTP 429 Too Many Requests`. About one request per 1.5 s was fine.

### 2. New Game Center (`www.sihf.ch/de/game-center/`)

**Verified.** This is an Umbraco site with htmx and a PWA.

**Pages**

| URL | Contents |
|---|---|
| `/de/game-center/spiele/` | Games of the day across all leagues. Each game card has `data-game-id`, `data-league-id`, `data-region-id`, `data-venue-id`, `data-is-live`, and the time or status text. |
| `/de/game-center/game/<gameId>` | Overview: header, period scores, shots on goal, timeline, venue, officials. |
| `/de/game-center/game/aufstellung/<gameId>` | Line-ups. |
| `/de/game-center/eisfelder/` | Rink map with **217 rinks**. Each rink is a `<dialog data-lat data-lng data-name data-search data-link=".../eisfelder/<venueId>">`. |

**Fragment endpoints.** The page fetches HTML fragments from `/umbraco/GameCenter/...` (e.g. `GameDetail/GetTimeline`, `GameDetail/GetGameLineup`, `GameShort/Get`). Called directly, without the page context, they returned a 400 error fragment.

**robots.txt** disallows `/umbraco/`.

**Live push.** Live updates come over an Azure SignalR hub. `POST /hubs/game/negotiate` returns an anonymous access token for `sigr-sihf-web-001.service.signalr.net` (hub `gamehub`). Clients receive `gameUpdate` messages carrying `gameId`, `leagueId`, `regionId` and team/club IDs. These are change notifications: the page then re-fetches the fragment. Firststar could use them as a trigger to re-poll the `data.sihf.ch` game detail. **Not verified:** whether the payload carries the status itself (it was only read from the minified JS, not captured).

### Joining venues to coordinates

**Verified.** `details.venue.id` in the API and `data-venue-id` in the Game Center match the rink IDs on the `/eisfelder/` map. All 81 venues in use on 2026-10-11, across all leagues, were present in the 217-rink map.

Examples:

| Venue ID | Rink | Coordinates |
|---|---|---|
| 167 | SWISS Arena, Kloten | 47.441863, 8.583797 |
| 173 | Sportanlage Güttingersreuti, Weinfelden | 47.560688, 9.125188 |
| 55 | Seelandhalle Lyss | 47.067813, 7.312313 |

The venue also always carries a street address, so geocoding is a fallback. In one NL game out of 13 sampled, the address was incomplete.

**Not verified:** how accurate the coordinates are (building centroid vs. entrance). Spot checks look right to within about 100 m. That should be confirmed before choosing the rink-presence radius.

## Per-league findings

**Sample:** every final game on 3, 4 and 10 Oct 2026 in each senior league, read from the game-detail endpoint. "Both line-ups" means both teams had goalies and skaters listed. "Goals = score" means the number of goal events equalled the final score.

| League | Final games sampled | Both line-ups | Venue address | Goals = score | Plausible end time (90–200 min after start) | Other |
|---|---|---|---|---|---|---|
| National League (1) | 13 | 13 | 12 | 13 | 13 | live today: period status, SOG, events |
| Sky Swiss League (2) | 14 | 14 | 14 | 14 | 14 | 1 game `Abgebrochen` |
| MyHockey League (100) | 13 | 13 | 13 | 13 | 13 | |
| 1. Liga (3) | 18 | 18 | 18 | 18 | 18 | 1 game `Abgebrochen` |
| 2. Liga (10) | 33 | 33 | 33 | 33 | 33 | 1 game `Abgebrochen` |
| 3. Liga (18) | 49 | 49 | 48 | 49 | 42 | see anomalies below |

**3. Liga anomalies** (all **Verified**, game IDs given so they can be rechecked):

- A 3L qualification tournament on 3 Oct had 1-hour slots, so end times of about 60 min are legitimate. One game in it, `20270009263911`, has `endDateTime` 3 min *before* its scheduled start. The sheet was evidently closed or back-filled out of real time.
- The 3L Finale `20270009263912` (3 Oct, 18:00) was still `Wie geplant` (not started) eight days later, and its home and away line-ups were identical, which is clearly bogus.
- Lower leagues therefore occasionally have missing or wrong live data. Firststar must not depend on it without a manual override.

**Live state.** It works across all levels: on 2026-10-11 the NL game and even U14/U16 games showed live period states and minute-by-minute events. This follows from SIHF's "Reporter" electronic game report.

- The 2019/20 SIHF directive for Regio League clubs ([weisungen-reporter-rl_2019_20.pdf](https://www.sihf.ch/media/15789/weisungen-reporter-rl_2019_20.pdf)) makes the Reporter game report mandatory. It also says scorekeepers only accept line-ups from the player list kept in Reporter/MyHockey.
- **Not verified** that the 2026/27 rules are identical. The observed data is consistent with them.

### Game status values

**Verified.** These `gameStatus.id` / `name` pairs were seen over 3–11 Oct 2026 across all leagues:

| id | name | meaning |
|---|---|---|
| 0 | Wie geplant | scheduled, not started |
| 1 | 1. Drittel | 1st period |
| 2 | Pause | 1st intermission |
| 3 | 2. Drittel | 2nd period |
| 4 | Pause | 2nd intermission |
| 5 | 3. Drittel | 3rd period |
| 9 | Ende* | provisional end (presumably awaiting OT/SO or confirmation) |
| 12 | Ende | final; `endDateTime` = actual end time, `percent` = 100 |
| 14 | Abgebrochen | abandoned (`canceled: true`) |
| 18 | Abgesagt | called off (`canceled: true`) |

IDs 6–8, 10–11 and 13 were not seen. They presumably cover OT, shootout and other states (**Not verified**).

### Line-up timing

**Verified, single snapshot.** At 15:20 on 2026-10-11, of ten 2L/3L games later that day:

- only the 16:30 game (70 min out) had line-ups, last updated at 14:53, about 95 min before start;
- games 85 min or more away had none.

Line-ups therefore come from the rink-side game sheet shortly before the game. They are not a season roster. For a season roster, the game center has team pages (`/de/game-center/team/<league>-<region>-<teamId>`, player stats via `/GameCenter/Team/GetPlayerStats`). These were **not examined in depth**.

## Terms of use

**Verified.** Source: [sihf.ch/de/nutzungsbedingungen](https://www.sihf.ch/de/nutzungsbedingungen/), read 2026-10-11.

- "Eine Reihe von Textelementen, Bildern, Grafiken, Beiträgen und Daten auf der Website werden von Fotoagenturen, von Clubs und den Partnern an SIHF zur Verfügung gestellt. Deren Inhalt darf nicht ohne vorherige Genehmigung der aufgeführten Eigentümer oder Lieferanten weiterverwendet werden."
- Content may be retrieved and reused "für Ihren persönlichen, nichtkommerziellen Gebrauch". "Bei unberechtigter Weiterverwendung der Website oder eines ihrer Teile für kommerzielle Zwecke behält sich SIHF rechtliche Schritte vor."
- Names and logos are SIHF or partner trademarks and may not be used without consent.
- Using robots to fill in competitions and votings systematically is forbidden. This is about SIHF's own forms, not data reading, but worth knowing.
- `robots.txt` disallows `/umbraco/` (the Game Center fragment API) and adds `Content-Signal: ai-train=yes, search=yes, ai-input=yes`. The `data.sihf.ch` host is separate.

**Not verified / not found:** any published SIHF data-licensing offer, developer API or club data feed. Searches turned up nothing. Firststar should ask SIHF directly about a data licence; clubs, as SIHF members, may help. The third-party club tool MyIceHockey [states](https://myice.hockey/en/swiss-ice-hockey-und-my-ice-hockey-verlaengern-partnerschaft-um-drei-jahre/) a partnership with SIHF that includes importing Reporter game data, so such agreements do exist.

## What this means for Firststar

| Need | SIHF provides? | Team Manager must enter by hand? |
|---|---|---|
| Game schedule (date, time, opponent) | Yes, whole season, all six leagues | No. Allow an override for reschedules; `Abgesagt` status is published. |
| Venue + coordinates | Yes: venue ID → lat/lng from the rink map, plus street address | No. Allow an override; check coordinate accuracy for the geofence. |
| Line-ups, both teams | Yes, but only about 1 h before puck drop (from the game sheet) | Not normally. Fall back to the team's season roster or manual entry if missing at vote time. |
| Live period / final buzzer | Period status and `Ende` with end timestamp, about 30 s to a few minutes behind real time. No running clock. | Not normally. Votes can open automatically on `Ende` (or on `3. Drittel` + elapsed time). Keep a manual "open/close vote" button for games whose sheet is late or broken. |
| Permission to use the data commercially | **No.** Terms restrict use to personal, non-commercial use. | n/a. Needs an agreement with SIHF before launch. |

## Method and sources

All requests were made on 2026-10-11 between 15:10 and 15:25 CEST.

- `https://www.sihf.ch/de/game-center/`, `/spiele/`, `/eisfelder/`, `/eisfelder/74`, `/suche/`, `/navigation/`, `/sihf-tools/`
- game pages `/game/20271105000075`, `/game/20270009252355` and their `/aufstellung/` pages
- `https://www.sihf.ch/scripts/game-center.js`: fragment URLs, SignalR hub `/hubs/game`, `gameUpdate` handler
- `https://www.sihf.ch/hubs/game/negotiate` (POST)
- `https://www.sihf.ch/robots.txt`
- `https://www.sihf.ch/de/nutzungsbedingungen/`
- `https://data.sihf.ch/Statistic/api/cms/cache300?alias=results...` and `.../gameoverview?alias=gameDetail...`, for about 200 games in total
- [SIHF Weisungen Reporter Regio League 2019/20](https://www.sihf.ch/media/15789/weisungen-reporter-rl_2019_20.pdf): background only, not re-read for 2026/27
- [eishockeynews.de on the MyHockey League 2026/27](https://www.eishockeynews.de/artikel/mehrere-neuerungen-in-der-dritthoechsten-hockeyliga-1230834): secondary, used only to confirm MHL still exists; SIHF data confirms it as league 100
