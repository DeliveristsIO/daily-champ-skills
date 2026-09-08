# Daily Champ skills

Talk to [Daily Champ](https://daily-champ.deliverists.io) from Claude Code,
Codex, Cursor and anything else that speaks MCP.

This repository is private. Every command below reads it over your own GitHub
access, so `gh auth status` should be green and your SSH key loaded first.

```bash
npx skills add DeliveristsIO/daily-champ-skills
```

Then follow [`install.md`](install.md) — it is written to be handed to your agent
and executed. The only step you do yourself is signing in through the browser
when prompted; there are no keys to copy.

**Claude Code, in one step** — the plugin bundles the skill *and* the MCP server
config:

```
/plugin marketplace add git@github.com:DeliveristsIO/daily-champ-skills.git
/plugin install daily-champ@deliverists
```

If either command cannot reach the repository, clone it over SSH and link the
skill by hand — see **Manual installation** in [`install.md`](install.md).

## What you can ask for

| | |
| --- | --- |
| **What is on** | "what should I work on?" · "what's running?" · "what's left today?" |
| **Adding** | "add stretching, 15 minutes" · "put 'call the plumber' in Home" |
| **Doing** | "start the timer on the invoice" · "park that and pick up the report" · "tick it off" |
| **Planning** | "push the dentist to Thursday" · "make the bins every Monday and Thursday" · "give that 45 minutes" |
| **Looking back** | "what's my streak?" · "where did my time go this week?" |

## Skills

| Skill | Covers |
| --- | --- |
| `daily-champ` | The whole MCP surface — the day, the areas board, the worklog lanes, timers, estimates, repeats and the archive. |

## Requires

A Daily Champ account, authorized in your browser through standard MCP OAuth —
no keys to copy, and no password anywhere near your agent. For a headless
machine or a client without OAuth there is an API token fallback: **Settings →
Devices**, one per device, revocable from the same page. Agents you signed in
through the browser are listed under **Settings → Agents** and disconnect from
there.

Built by [Deliverists.IO](https://deliverists.io).
