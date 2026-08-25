# GameBoard

Voice-driven scorekeeping. Six board types, natural-language voice control, and
a Flask + SQLite backend where every board belongs to the person who created it.

## Run it

    npm install
    pip3 install -r backend/requirements.txt

    npm run dev        # API on :5055, app on :5173

Open http://localhost:5173 and sign in.

### One port (phone + laptop on the same wifi)

    npm run serve      # builds the app and serves everything from :5055

Then open `http://<your-computer-ip>:5055` on the other device.

### Ports, and why you should not have to care

GameBoard listens on **5055**. Port 8080 is deliberately avoided: a Spring Boot
app already uses it on this machine, and a dev server left proxying `/api`
there returns a stranger's 404 that looks like a GameBoard bug.

**The frontend now finds its own backend.** On boot it probes, in order:

1. `VITE_API_BASE`, if you set one
2. the base it used last time
3. same origin (`/api`) — correct when Flask serves the built app
4. `http://<this-host>:5055/api` — the backend's own default

It keeps the first one that answers `/health` as GameBoard. So a stale proxy
target no longer breaks anything: the app routes around it and remembers the
working base. Only if *nothing* answers do you get an error, and it names every
address it tried.

Override the port with `PORT=… npm start`.

## Accounts

**Your name is your account.** Sign in with the same name on any device and
your boards are there; sign in with a different name and you get a clean slate.

You must sign in before anything else — there is nothing meaningful to show
before the app knows whose boards to load.

Boards are private per account, enforced in SQL (`scoreboards.owner_id`) rather
than hidden in the UI. Another account asking for your board gets a 404, so the
API never even confirms it exists. There is no password: this is a scoreboard
for people around a table, not a public service.

## Board types

Chosen in step 1 of **New Board**. (There was a separate "Board Types" gallery
page listing the same six; it duplicated the picker, so it was removed.)

| Board | For |
|---|---|
| Leaderboard | Ranked rows. Quizzes, class points, anything "who's ahead". |
| Scoresheet | Rounds x players grid. Card games, multi-round play. |
| Scoreboard | Big blocks. Readable across a room or on a TV. |
| Tournament Bracket | Single elimination, automatic seeding and byes. |
| Sports Scoreboard | Two sides, game clock, periods, full-screen scorebug. |
| Counter | Tap a tile, the number goes up. |

## Voice

Hold **Space** to talk, or turn on **Hands-free**. Parsed on-device — no API
key, no per-call cost.

**Scoring** — "add 5 points to player 1", "give Rahul ten", "remove 3 from
Alice", "set Bob to 20", "Jhanavi 25 Rahul 18", "everyone gets 3"

**Asking** — "who's on top", "who's winning", "who's last", "what's the score",
"how many points does Alice have", "who's in second", "what's the gap", "what
round is it", "how much time is left". Answers are spoken and shown on screen
with the standings.

**Control** — "next round", "undo", "end game", "start the clock"

## Tournaments

A tournament is a roster and a board type. Each game inside it is an ordinary
board carrying `tournament_id`, so standings are aggregated from the same
`scores` rows the boards use — the table can never drift from its games.

- **Start game N** opens the next board, pre-filled with the roster and already
  started, so you can score straight away.
- Standings show points, wins and games played. A draw awards no win, and an
  unfinished game contributes points but no win.
- Ties share a rank and skip the next, matching the leaderboard.
- Deleting a tournament keeps its games — ending a series is not deleting
  history.

## Analysis dashboard

Every board has one at **`/games/{id}/analysis`** — the chart icon on the board
screen, or "Analysis" inside a history entry.

Ported from the standalone analytics page, with its inputs removed: it used to
ask for a scoreboard id in the query string and carry its own API base and
connection indicator. All three are already known here, so they are gone. The
panels are the same:

- KPIs — players, rounds played, total points, leader gap
- Leaderboard, ranked by the backend (click a row to inspect that player)
- Player insight — average, best/worst round, consistency, last round, trend,
  gap to leader, rounds played
- Score progression — cumulative totals per round; click a name to hide a line
- Achievements and a game timeline of lead changes

Every number comes from `GET /api/scoreboards/{id}/analysis`. Nothing is
re-totalled or re-ranked in the browser. The chart uses Recharts, already a
dependency, rather than a CDN script.

## Backend

Flask + SQLite, from `github.com/neghaaloor/cognizant-demo`. Its API contract
(`docs/API_CONTRACT.md`), lifecycle (`SETUP -> ACTIVE -> ENDED`), error codes
and `success` envelope are followed as written. Points always live in the
`scores` table and totals are always derived with `SUM()`, so a stored total
can never drift from the round-by-round history.

    npm run test:api          # 74 tests

### What GameBoard added

The upstream backend has no notion of accounts, which is exactly why every
sign-in used to show the same history. Added, in its own style:

- `users` table, `POST /api/session` — sign in by name, case-insensitive.
- `scoreboards.owner_id` — enforced inside `get_scoreboard_row()`, the function
  every service already calls first, so ownership cannot be forgotten.
- `board_id` / `config` / `board_state` columns — which of the six board types
  renders a board, and the state the relational model doesn't cover (bracket
  tree, game clock). Points never go here.
- `PUT /api/scoreboards/{id}/rounds/{roundId}/scores/{playerId}` — set or nudge
  one player's score. The upstream `POST /scores` is a one-shot round
  submission and rejects a second write; the running-total boards need to keep
  changing a number. Same table, same `SUM()`-derived totals.
- `players.colour`.
- `tournaments` table plus `scoreboards.tournament_id`, and
  `/api/tournaments` — the upstream backend has no concept of a series.
- `GET /api/scoreboards` now returns each board's ranked leaderboard, rounds
  played, total points and winner. The history screen needs final standings,
  and re-deriving them from an empty list response is why finished games used
  to show every player on zero.

### Multi-device

This backend has no push channel by design; its contract says to poll
`/summary`. The board screen polls every 3 seconds, so a score entered on one
device appears on another within a few seconds.

### Known limits

- The name is the account; anyone who knows it can sign in as you.
- The built app is served *by* the API server, so a reload with the server down
  won't load.
- Adding a round writes 0 for anyone unscored, to satisfy the backend's
  `ROUND_INCOMPLETE` rule honestly. Empty scoresheet cells already display 0.
- The Players screen is a read-only roll-up; players belong to a board on the
  server rather than a global roster.

## Credits

The backend is derived from **[neghaaloor/cognizant-demo](https://github.com/neghaaloor/cognizant-demo)**
— its Flask + SQLite scorekeeper, API contract, lifecycle, error codes and test
suite. Those files are kept close to the original; the GameBoard additions are
marked `ADDED for GameBoard` where they appear:

- accounts (`users`, `owner_id`) so boards are private per person
- board-type columns so the six board types persist
- `PUT .../scores/{playerId}` for the running-total boards
- `tournaments` and `scoreboards.tournament_id`

> **Note on licensing:** that upstream repository has no LICENSE file, which
> under GitHub's terms means the author reserves all rights. If you are not one
> of its authors, get their permission before publishing this repository
> publicly, or replace `backend/` with your own implementation of the same
> documented contract.

The frontend (React, Vite, Tailwind), the offline voice parser and the six board
types were written for this project.
