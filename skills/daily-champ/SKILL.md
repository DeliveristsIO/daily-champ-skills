---
name: daily-champ
description: |
  Drive Daily Champ through its MCP server: today's plan, the areas board, the
  worklog lanes, timers, estimates, repeats and the archive.
  Use for ANY question about what to work on, what is running right now, what is
  planned, what slipped, or what got done.
triggers:
  # Direct
  - daily champ
  - /daily-champ
  # What is on
  - what should I work on
  - what is on today
  - what am I working on
  - what is running
  - my day
  - my board
  - my worklog
  # Adding
  - add a task
  - new task
  - put it on my list
  - plan tomorrow
  - plan my day
  # Doing
  - start the timer
  - stop the timer
  - start the clock
  - mark done
  - tick it off
  - reopen
  - move to in progress
  - move to todo
  - park it
  # Planning
  - how long will it take
  - set an estimate
  - repeat every
  - every monday
  - push it to
  - archive it
  # Looking back
  - my streak
  - what did I get done
  - where did my time go
invocable: true
argument-hint: "[action] [args...]"
---

# Daily Champ

Daily Champ is a day tracker: a **board** of areas holding tasks, a **worklog**
of lanes holding the work in hand, and a **day** that says what is on today. You
reach all of it through MCP tools — no CLI, no curl.

## If the tools are not there

Check first: call `get_today`. If it resolves, you are connected; skip this
section.

If Daily Champ tools are missing, add the server to your own config, then hand
the sign-in to the human — that part is theirs and cannot be automated.

**Prefer the browser sign-in.** It is standard MCP OAuth: no secret is copied,
nothing lands in a config file, and the session renews itself.

1. **Add the server**, no header:
   - Claude Code — `claude mcp add --transport http --scope user daily-champ https://daily-champ.deliverists.io/mcp`
   - Codex — `codex mcp add daily-champ --url https://daily-champ.deliverists.io/mcp`, then `codex mcp login daily-champ`
   - OpenCode — under `"mcp"` in `opencode.json`: `"daily-champ": { "type": "remote", "url": "https://daily-champ.deliverists.io/mcp" }`
2. **Restart the session**, then tell the human to run `/mcp`, pick
   **daily-champ**, choose **Authenticate**, sign in and authorize. Wait for
   them — you cannot do this step.
3. **Verify** with `get_today`.

**Or a token**, for a headless machine or a client with no OAuth. Ask the human
to open Daily Champ → **Settings** → **API tokens** → new token, copy it once,
and `export DAILY_CHAMP_TOKEN=…`. Then add the server with the header:

- Claude Code — `claude mcp add --transport http --scope user daily-champ https://daily-champ.deliverists.io/mcp --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"`
- Codex — `codex mcp add daily-champ --url https://daily-champ.deliverists.io/mcp --bearer-token-env-var DAILY_CHAMP_TOKEN`
- A client with no MCP support — `npx mcp-remote https://daily-champ.deliverists.io/mcp --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"` as the stdio command.

Both credentials go on the same `Authorization: Bearer` header and reach the
same tools. A token belongs to one person and one device; revoking it in
Settings cuts that device off at once. There is no account subdomain to
remember either way.

### The transport, if you are driving it by hand

Stateless streamable HTTP. No `Mcp-Session-Id` is issued or required, and
`initialize` is optional — a bare `tools/call` POST works on its own:

```bash
curl -s https://daily-champ.deliverists.io/mcp \
  -H "Authorization: Bearer $DAILY_CHAMP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":"1","method":"tools/call","params":{"name":"get_today","arguments":{}}}'
```

## The one concept to get right

**A task is one thing with two halves.** An *item* lives on the board, in an
area. A *sitting* lives in the worklog, in a lane, on a day. Most tasks have
only one half — typed straight onto a day, or parked on the board and never
pulled in.

You never have to choose. **Every tool takes `task_id` and accepts either id.**
Reads hand back both (`id:` is the sitting, `item:` is the board task), so
whatever you just read you can quote straight back.

A write that needs the missing half makes it. `set_repeat` on a task that only
exists in the worklog files it under **Inbox** first, because a repeat belongs to
the task and not to one sitting of it. `move_task` and `schedule_task` on a board
task pull it into the worklog.

## Rules the app enforces, so you do not have to

- **One clock per person.** `start_timer` stops whatever was running and banks
  its minutes. The result says what it stopped.
- **Work in progress is capped.** Moving into a starting lane when it is full
  comes back as *"In progress is full at 7. Finish something before starting
  more."* — a sentence, not an error. Do not retry it; finish something or pick
  another lane.
- **Ending work stops the clock.** Ticking a task off, or moving it out of a
  starting lane, banks the running time and tells you how much.
- **A future day puts a task down.** `schedule_task` to a date ahead of today
  takes it out of the lanes; it lands back in To do that morning on its own.
- **Estimates are minutes.** Always. `set_estimate` takes minutes, `get_task`
  reports minutes.
- **Lanes are the user's own rows**, not an enum. `todo`, `in_progress` and
  `done` always resolve, and so does any lane the user has named themselves.
- **Archive, never delete.** There is no delete tool. `archive_task` puts a task
  away with its lane, its clock and its tracked time intact, and
  `restore: true` brings all of it back.

## Working rules for you

- **Read before you write.** `get_today` or `get_board` first, then act on the
  ids you were given. Do not guess an id.
- **Dates are ISO only.** You work out what "tomorrow" or "next Monday" means in
  the user's timezone; `schedule_task` takes `YYYY-MM-DD` and nothing else. Its
  refusal tells you today's date, so one retry is enough.
- **Never invent a number.** Streaks, tracked minutes and counts come from tool
  results. Do not estimate them.
- **You are the coach.** There are no coaching tools and you do not need any —
  the app's own coach is a smaller model. Give the advice yourself.
- **Say what you changed.** Quote the tool's own sentence back; it already names
  the lane, the clock and the day.

## The tools

<!-- tools:start -->

| Tool | | Arguments | What it does |
| --- | --- | --- | --- |
| `get_today` | read | — | What is on today: the task with the clock running, everything planned for the day, and how much of it is done. Start here when asked what to work on. |
| `get_board` | read | `include_done?` | The whole board: every area with the tasks parked in it, and the worklog lanes with the work in hand. Use it to see what exists before adding something new. |
| `search_tasks` | read | `query` | Find a task, an area or a card by name. Unfinished work is always findable; finished work drops out after a week. |
| `get_task` | read | `task_id` | Everything about one task, from either end: where it lives on the board, where it sits in the worklog, its estimate, its clock, its repeat and its steps. |
| `get_stats` | read | `days?` | How the run is going: the current streak, the best one, what got finished each of the last few days, and which areas the time went into. |
| `create_task` | write | `title` `area?` `date?` `estimate_minutes?` `lane?` | Add a task. With no area it lands on a day as work to do; with an area it is parked on the board until it is pulled in. Several lines in one title become several tasks. |
| `complete_task` | write | `task_id` | Tick a task off. Both halves finish together, and a running clock is stopped and banked as part of the same move. |
| `reopen_task` | write | `task_id` | Put a finished task back to work. It returns to the lane its state belongs in, not necessarily the one it left. |
| `start_timer` | write | `task_id` | Start the clock on a task. Several clocks may run at once, so anything already running keeps running. |
| `stop_timer` | write | `task_id?` | Stop the clock and bank what it counted. The task stays where it is — this does not finish it. |
| `move_task` | write | `task_id` `lane` | Move a task between the worklog lanes. A board task that has never been pulled in is pulled in by this. Moving into a lane that starts work also starts it, and moving out of one puts it back down. |
| `schedule_task` | write | `task_id` `date` | Put a task on a day, pulling it in from the board if it is not in the worklog yet. A date in the future puts it down until that morning, when it comes back on its own. |
| `set_estimate` | write | `task_id` `minutes` | Say how long a task should take, in minutes. The clock counts down against it, and changing it re-arms the alert that fires when the time is up. |
| `set_repeat` | write | `task_id` `days?` `monthly?` | Make a task come back on the days you choose. A task that only exists in the worklog is filed on the board first, because a repeat belongs to the task, not to one sitting of it. |
| `archive_task` | write | `task_id` `restore?` | Put a task away, or take it back out. Nothing is destroyed: its lane, its clock and its time all come back on restore. There is no delete. |

<!-- tools:end -->

## Workflows

**Plan my day.** `get_today` for what is already on; `get_board` for what is
waiting. Add what is missing with `create_task`, and pull anything from the board
with `schedule_task(task_id: …, date: <today>)`. Give each one a
`set_estimate` so the clock has something to count against.

**What am I working on?** `get_today`. The first line after the date says
`Running now:` with the elapsed and the budget, or `No clock running.`

**Start on something.** `move_task(task_id: …, lane: "in_progress")` then
`start_timer(task_id: …)`. If the lane refuses, say which lane is full and offer
to finish something.

**Park this and pick up that.** `move_task(task_id: …, lane: "todo")` on the
first — that alone stops its clock and banks the time — then move and start the
second.

**Push it to another day.** `schedule_task(task_id: …, date: "2026-09-15")`.
Resolve the date yourself and say the weekday back to the user.

**Make it weekly.** `set_repeat(task_id: …, days: [1, 3, 5])`. Days are numbers,
Sunday is 0. Add `monthly: true` for the 1st of the month as well. An empty
`days` with no `monthly` stops the repeat.

**Catch up on what slipped.** `get_board(include_done: false)` shows the lanes as
they stand; anything in progress from an earlier day has already been carried
onto today by the app. `get_stats` gives the streak and where the time went.

## Gotchas

- **A refusal is not a failure.** The full-lane message and "already done" come
  back as ordinary results. Only a real error is marked as one.
- **`stop_timer` does not finish anything.** Use `complete_task` for that — it
  stops the clock too.
- **`set_estimate` needs a sitting.** A board task that was never pulled in has
  nowhere to put an estimate; the refusal tells you to `schedule_task` it first.
- **`create_task` with an `area` does not put it on a day.** It parks it on the
  board. Leave `area` out to plan it for today.
- **Several titles in one call.** `create_task` splits on newlines and
  semicolons, so a pasted list becomes a list of tasks. It returns one line per
  task made.
- **Finished work drops out of search after a week.** Unfinished work is always
  findable.
- **`search_tasks` needs two characters.** Shorter queries match nothing.
- **Writes are rate limited per token.** Past the ceiling you get JSON-RPC
  `-32000` with a Retry-After. Stop and tell the user; do not loop.
- **There is no `delete_task`, no `create_area`, no coaching tool, and no way to
  accept or sign off a shared task from here.** Do not assume a tool exists —
  the table above is generated from the live server.
