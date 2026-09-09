---
name: daily-champ
description: |
  Drive Daily Champ through its MCP server — the whole app, not a corner of it:
  today and any other day, the areas board, the worklog lanes, cards and their
  plans, timers, estimates, repeats, steps, checklists, sharing and the archive.
  Use for ANY question about what to work on, what is running right now, what is
  planned, what slipped, what got done, or what somebody else is waiting on.
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
  - remind me at
  - due at
  - give it a time
  - push it to
  - archive it
  - plan this card
  - start a card
  - break it into steps
  # People
  - share this with
  - ask them to
  - sign it off
  - who is waiting on me
  # Looking back
  - my streak
  - what did I get done
  - where did my time go
invocable: true
argument-hint: "[action] [args...]"
---

# Daily Champ

Daily Champ is a day tracker: a **board** of areas holding tasks, a **worklog**
of lanes holding the work in hand, **cards** for projects with an end, and a
**day** that says what is on. You reach all of it through MCP tools — no CLI, no
curl — and the tools cover everything the app can do, deleting included.

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
to open Daily Champ → **Settings** → **Devices** → new device token, copy it once,
and `export DAILY_CHAMP_TOKEN=…`. Then add the server with the header:

- Claude Code — `claude mcp add --transport http --scope user daily-champ https://daily-champ.deliverists.io/mcp --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"`
- Codex — `codex mcp add daily-champ --url https://daily-champ.deliverists.io/mcp --bearer-token-env-var DAILY_CHAMP_TOKEN`
- A client with no MCP support — `npx mcp-remote https://daily-champ.deliverists.io/mcp --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"` as the stdio command.

Both credentials go on the same `Authorization: Bearer` header and reach the
same tools. A token belongs to one person and one device; revoking it under
**Settings → Devices** cuts that device off at once. A browser sign-in is
listed under **Settings → Agents** instead, and disconnecting it there kills
the agent's access the moment you click it. There is no account subdomain to
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

- **Several clocks may run at once.** `start_timer` leaves anything already
  running alone, and the result names what is still running beside it.
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
- **A time written into a title is the hour it is due.** `@9` or `@16:45`
  anywhere in a title — on `create_task`, or on a rename — comes out of the name
  and becomes when the task is due. A task with no day of its own is put down on
  the first day it can still happen: today while the hour is ahead, tomorrow once
  it has gone by. A task already sitting on a day keeps that day and takes the
  hour on it. The 24-hour clock only, so `@25` and an address like `bob@12` are
  left alone. `update_task(deadline:)` is the explicit form, for when the day
  matters as much as the hour.
- **A deadline nudges the person ten minutes before it.** One push, to whatever
  they have turned notifications on for under **Settings** — you cannot turn that
  on for them, and the button beside it sends a test one so they can prove it
  works.
- **Work already in hand sits at the bottom of its area.** `get_board` reads an
  area the way the person is looking at it: what is still waiting first, then
  whatever has been pulled into a lane, then what is done. The top of an area is
  always what to pick up next.
- **Lanes are the user's own rows**, not an enum. `todo`, `in_progress` and
  `done` always resolve, and so does any lane the user has named themselves.
  `create_lane` adds one, and its two flags are what give it meaning: a lane
  that starts work starts the clock and counts against the WIP limit, one that
  finishes work ticks the task off.
- **Areas and lanes lose nothing when deleted.** `delete_area` hands its tasks
  to the Inbox, `delete_lane` moves its work to whichever lane matches where
  each task had got to, and neither the Inbox nor the last lane can go.
- **Archive first, delete last.** `archive_task` puts a task away with its lane,
  its clock and its tracked time intact, and `restore: true` brings all of it
  back. `delete_task` cannot be undone — reach for it only when the human has
  said to destroy something, and say what will go before you call it.
- **A task has two halves, and delete respects which id you hold.** A worklog id
  deletes that sitting and leaves the board task; a board id deletes the board
  task and leaves the days its past sittings are on; `everywhere: true` takes
  both. Every other tool takes either id and does the same thing.

- **Cards are projects, areas are not.** An area is a standing part of someone's
  life and never ends; a card runs between two dates and has a plan spread over
  them. `plan_card` with no `tasks` drafts a plan and saves nothing — show it to
  the human, then call it again with the tasks they kept. Only today and tomorrow
  are written out as real tasks; the rest arrive each morning.
- **Finishing a card is a status, not a delete.** `update_card(status:
  "achieved")` or `"abandoned"` closes it and keeps everything. `delete_card`
  destroys its tasks, their sittings, its plan and its reviews.

- **A day is more than its tasks.** `get_day` shows the sections it is laid out
  in — checklists and notes — and `apply_template` lays a fresh one out. A note
  is the first line on a text section, so it is written with
  `add_checklist_item` and reworded with `update_checklist_item`.
- **A day's status is not yours to set.** Pending today, scheduled ahead, win or
  loss behind, worked out from its own tasks every time it is saved. Finish the
  tasks and the day follows.

- **Work handed to someone runs at both ends.** `update_task(assignee:)` asks
  them; they answer with `respond_to_task`, and declining needs a reason.
  Finishing it does not close it — it goes back to whoever asked, who closes it
  with `sign_off_task(decision: "approve")` or reopens it with `"send_back"` and
  a reason. Every reason is written onto the task's thread.
- **An @ name on a thread only reaches someone who can already see the task.**
  `add_comment` naming anybody else tells nobody, quietly — `share_task` first,
  then mention them.
- **Two tools reach outside the account, and both say so.** `share_task` with an
  address nobody here has sends that person an invitation email. `share_link`
  mints a URL that shows the task to anyone holding it, with no sign-in. Say what
  will happen and get a yes before either; `share_link(revoke: true)` kills a
  link at once.

## Working rules for you

- **Read before you write.** `get_today` or `get_board` first, then act on the
  ids you were given. Do not guess an id.
- **Dates are ISO only.** You work out what "tomorrow" or "next Monday" means in
  the user's timezone; `schedule_task` takes `YYYY-MM-DD` and nothing else. Its
  refusal tells you today's date, so one retry is enough. An hour — in a
  `deadline` or written into a title — is read as the hour where the person is,
  never where the server is, so you never convert one yourself.
- **Never invent a number.** Streaks, tracked minutes and counts come from tool
  results. Do not estimate them.
- **You are the coach.** The app's own coach is a smaller model. `plan_day` and
  `plan_card` reach it for a draft, which is worth it when the human wants the
  app's own read; otherwise give the advice yourself.
- **Say what you changed.** Quote the tool's own sentence back; it already names
  the lane, the clock and the day.

## The tools

<!-- tools:start -->

| Tool | | Arguments | What it does |
| --- | --- | --- | --- |
| `get_today` | read | — | What is on today: the task with the clock running, everything planned for the day, and how much of it is done. Start here when asked what to work on. |
| `get_board` | read | `include_done?` `include_archived?` | The whole board: every area with the tasks parked in it, and the worklog lanes with the work in hand. Use it to see what exists before adding something new — every heading carries the id you need to change it. |
| `search_tasks` | read | `query` | Find a task, an area or a card by name. Unfinished work is always findable; finished work drops out after a week. |
| `get_task` | read | `task_id` | Everything about one task, from either end: where it lives on the board, where it sits in the worklog, its estimate, its clock, its repeat, its steps, who it is shared with and what has been said on it. |
| `get_stats` | read | `days?` | How the run is going: the current streak, the best one, what got finished each of the last few days, and which areas the time went into. |
| `get_cards` | read | `card?` | Cards are the projects with an end: a title, a window of days, and a plan spread across them. With no id this lists them; with one it opens that card, its plan, its tasks and its reviews. |
| `get_day` | read | `date?` `thread?` | One day in full: how it was called, what was planned, and the sections it is laid out in — checklists and notes both. Ask for the thread to see what the day's coach has said. |
| `list_notifications` | read | `unread_only?` `limit?` | What has happened that involves other people: work shared with you, asked of you, accepted, declined or signed off. Reading them here does not mark them read. |
| `create_task` | write | `title` `area?` `date?` `estimate_minutes?` `lane?` | Add a task. With no area it lands on a day as work to do; with an area it is parked on the board until it is pulled in. Several lines in one title become several tasks. |
| `update_task` | write | `task_id` `title?` `description?` `notes?` `note?` `tag?` `deadline?` `assignee?` | Change a task's wording, its deadline, its tag or who it is for. A rename carries across both halves on its own, so it does not matter which id you hold. |
| `complete_task` | write | `task_id` | Tick a task off. Both halves finish together, and a running clock is stopped and banked as part of the same move. |
| `reopen_task` | write | `task_id` | Put a finished task back to work. It returns to the lane its state belongs in, not necessarily the one it left. |
| `start_timer` | write | `task_id` | Start the clock on a task. Several clocks may run at once, so anything already running keeps running. |
| `stop_timer` | write | `task_id?` | Stop the clock and bank what it counted. The task stays where it is — this does not finish it. |
| `move_task` | write | `task_id` `lane` | Move a task between the worklog lanes. A board task that has never been pulled in is pulled in by this. Moving into a lane that starts work also starts it, and moving out of one puts it back down. |
| `file_task` | write | `task_id` `area?` `card?` | Put a task in an area or on a card. This is where it is parked on the board, not which lane it sits in — move_task does lanes. |
| `schedule_task` | write | `task_id` `date` | Put a task on a day, pulling it in from the board if it is not in the worklog yet. A date in the future puts it down until that morning, when it comes back on its own. |
| `copy_task` | write | `task_id` `date?` `each_day_for?` | Make another go at the same task on a later day. The copy starts fresh: no clock, nothing banked, not done. Use schedule_task instead to move the one that exists. |
| `set_estimate` | write | `task_id` `minutes` | Say how long a task should take, in minutes. The clock counts down against it, and changing it re-arms the alert that fires when the time is up. |
| `set_repeat` | write | `task_id` `unit?` `interval?` `days?` `monthday?` `at?` `until_on?` `times?` | Make a task come back. A task that only exists in the worklog is filed on the board first, because a repeat belongs to the task, not to one sitting of it. |
| `archive_task` | write | `task_id` `restore?` | Put a task away, or take it back out. Nothing is destroyed: its lane, its clock and its time all come back on restore. Reach for this before delete_task. |
| `delete_task` | write | `task_id` `everywhere?` | Destroy a task for good. Prefer archive_task, which keeps everything and can be undone. The id says which half goes: a worklog id takes that sitting and leaves the board task, a board id takes the board task and leaves the days its past sittings are on. |
| `add_step` | write | `task_id` `titles` | Break a task down. Steps belong to the task itself, so every sitting of it shows the same list. A worklog-only task is filed on the board first. |
| `update_step` | write | `step_id` `done?` `title?` | Tick a step off, put it back, or reword it. |
| `delete_step` | write | `step_id` | Take a step off a task for good. |
| `create_area` | write | `title` `color?` | Add a column to the board. An area is a standing part of someone's life — Home, Work, Health — not a project with an end, which is what a card is for. |
| `update_area` | write | `area` `title?` `color?` | Rename an area or change its colour. |
| `delete_area` | write | `area` | Take a column off the board. Nothing in it is lost — every task in it moves to the Inbox first. The Inbox itself cannot go. |
| `create_lane` | write | `name` `starts_work?` `finishes_work?` | Add a column to the worklog. A lane is placement, and its two flags are what make it mean something: a lane that starts work starts the task's clock running against the WIP limit, and one that finishes work ticks the task off. |
| `update_lane` | write | `lane` `name?` `starts_work?` `finishes_work?` | Rename a worklog lane or change what landing in it means. Changing the flags does not re-file the work already sitting there; it changes what the next move into it does. |
| `delete_lane` | write | `lane` | Take a lane off the worklog. The work in it is not lost — each task moves to whichever remaining lane matches where it had got to. The last lane cannot go. |
| `create_card` | write | `title` `description?` `start_date?` `end_date?` `area?` `most_per_day?` `rest_weekday?` | Start a card: a project with an end, as against an area, which is a standing part of life. With no dates it opens a 90-day window from today. |
| `update_card` | write | `card` `title?` `description?` `status?` `start_date?` `end_date?` `area?` `most_per_day?` `rest_weekday?` | Change a card, including closing it: achieved when it worked, abandoned when it did not. Neither touches the tasks already written out of its plan. |
| `delete_card` | write | `card` | Destroy a card and everything written out of it. Its tasks go too, and so do their sittings, its plan, its reviews and its coach thread. To stop a card without losing any of that, set its status to abandoned instead. |
| `plan_card` | write | `card` `tasks?` | Draft or save a card's plan. Called with no tasks it asks the app's own coach for a draft and hands it back without saving anything, so it can be shown to the person first. Called with tasks it saves them, and writes today's and tomorrow's into real tasks; the rest are written each morning as they come. |
| `review_card` | write | `card` `suggest?` `period_start?` `period_end?` `progress_rating?` `are_items_effective?` `what_worked?` `what_to_improve?` `notes?` | Write a review of how a card is going over a stretch of days. Ask the app's own coach for something to react to first with suggest, which drafts a review without saving it. |
| `plan_day` | write | `date?` `tasks?` | Draft or fill a day. Called with no tasks it asks the app's own day coach what to put on the day, given what is already there, and saves nothing. Called with tasks it puts them on the day. |
| `apply_template` | write | `template?` `date?` | Lay a day out from a template — its sections and whatever they always start with. With no template named this lists the templates there are and changes nothing. Applying one replaces the sections already on that day. |
| `delete_section` | write | `section_id` | Take a section off a day, with everything on its checklist. Tasks are not in a section — delete_task takes those. |
| `add_checklist_item` | write | `section_id` `contents` | Put something on one of a day's checklists — the sections a day is laid out in, as against the tasks on it. |
| `update_checklist_item` | write | `item_id` `done?` `content?` | Tick something off a day's checklist, put it back, or reword it. |
| `delete_checklist_item` | write | `item_id` | Take a line off a day's checklist for good. |
| `respond_to_task` | write | `task_id` `decision` `reason?` | Answer a task somebody has asked you to do. Accepting takes it on; declining hands it back with the reason, which is written into its thread. Only the person it was given to can answer. |
| `sign_off_task` | write | `task_id` `decision` `reason?` | Say whether work you asked somebody for is done. Approving closes it; sending it back reopens it with the reason on its thread. Only the person who asked can do either. |
| `add_comment` | write | `id` `body` | Say something on a task's or a card's thread. Everyone it is shared with sees it, and anyone named with an @ is told. |
| `delete_comment` | write | `comment_id` | Take back something you said. The line stays on the thread marked as deleted, so nobody is left answering a comment that vanished. Only its author can. |
| `share_task` | write | `id` `with` `level?` | Let somebody else in on a task or a card, to look at or to work on. Somebody already in the workspace is told in the app; an email nobody here has is sent an invitation, so check the address before you call this. |
| `unshare_task` | write | `id` `from` | Take somebody off a task or a card. They lose sight of it at once; anything they already wrote on its thread stays. |
| `share_link` | write | `id` `revoke?` | Make a link that shows a task or a card to anybody who has it, with no sign-in. Say the link back to the person before it goes anywhere. Called with revoke it kills the link instead, at once and for everyone. |
| `ask_coach` | write | `id` `message` | Put a question to the app's own coach about one task or one card. It answers from what it can see of that thing and remembers the exchange on its thread. It is a smaller model than you — reach for it when the person wants the app's own read, not for advice you can give yourself. |
| `ask_day_coach` | write | `message` `date?` | Put a question to the app's own coach about a whole day. It answers from what is on that day and remembers the exchange, which is what plan_day's draft then reads. |
| `undo` | write | — | Take back the last change, whoever made it — this reverses the person's own last action in the app just as readily as your own. It goes back one step only, and it does not reach delete_task, delete_card or anything that was said to somebody else. |

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

**Make it weekly.** `set_repeat(task_id: …, unit: "week", days: [1, 3, 5])`.
Days are numbers, Sunday is 0. `interval: 2` makes it every other week,
`unit: "month"` with `monthday: "last"` takes the last day of the month, and
`at`, `until_on` and `times` add a reminder time and an ending. Calling it with
no `unit` and no `days` stops the repeat. A repeating task stays one task on the
board and comes back with a fresh sitting on each day it is due.

**Give it a time.** `create_task(title: "Call the plumber @9")` puts the task on
the first day nine o'clock can still happen and has it nudge the person ten
minutes before. `update_task(task_id: …, title: "Call the plumber @16:45")` does
the same to a task that already exists, and
`update_task(task_id: …, deadline: "2026-09-15T09:00")` is the way to say the day
as well as the hour.

**Catch up on what slipped.** `get_board(include_done: false)` shows the lanes as
they stand; anything in progress from an earlier day has already been carried
onto today by the app. `get_stats` gives the streak and where the time went.

**Start a project.** `create_card` opens a 90-day window. Then `plan_card` with
no `tasks` drafts a plan from the app's own coach and saves nothing — read it
back, and call `plan_card` again with the tasks they kept. Today's and
tomorrow's become real tasks; the rest arrive each morning.

**Break something down.** `add_step(task_id: …, titles: [...])`, then
`update_step(step_id: …, done: true)` as each one is finished. `get_task` shows
the list and the count.

**Hand something to someone.** `update_task(task_id: …, assignee: "…")` asks
them, `share_task` lets them see it, and `get_task` says where it has got to.
When they finish it, it comes back to the owner for `sign_off_task`.

**Take that back.** `undo` reverses the last change — the human's own as readily
as yours, and one step only. It does not reach anything that was deleted or said
to somebody else.

## Gotchas

- **A refusal is not a failure.** The full-lane message and "already done" come
  back as ordinary results. Only a real error is marked as one.
- **`stop_timer` does not finish anything.** Use `complete_task` for that — it
  stops the clock too.
- **`set_estimate` needs a sitting.** A board task that was never pulled in has
  nowhere to put an estimate; the refusal tells you to `schedule_task` it first.
- **`create_task` with an `area` does not put it on a day.** It parks it on the
  board. Leave `area` out to plan it for today — or write a time into the title,
  which files it in the area and puts it on a day as well.
- **Several titles in one call.** `create_task` splits on newlines and
  semicolons, so a pasted list becomes a list of tasks. It returns one line per
  task made.
- **Finished work drops out of search after a week.** Unfinished work is always
  findable.
- **`search_tasks` needs two characters.** Shorter queries match nothing.
- **Writes are rate limited per token.** Past the ceiling you get JSON-RPC
  `-32000` with a Retry-After. Stop and tell the user; do not loop.
- **Do not assume a tool exists.** The table above is generated from the live
  server and is the whole surface. Anything not in it is not there — signing in,
  billing, workspace settings and device tokens are all deliberately out.
- **`undo` goes back one step, not many.** It undoes the last gesture, whoever
  made it, and it does not reach a delete, a comment or an email.
- **Reading notifications does not mark them read.** `list_notifications` leaves
  them as they were, so the human still sees the badge.
- **`ask_coach` and `ask_day_coach` cost a rate-limited request** and answer with
  a smaller model than you. Use them when the human wants the app's own read.
  Give your own advice for free.
