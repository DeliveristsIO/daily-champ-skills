# Install the Daily Champ skill

I want you to install Agent Skills for Daily Champ for me. Execute all steps
autonomously, except the one marked **HUMAN** — stop and wait for that one.

**OBJECTIVE:** connect this agent to Daily Champ over MCP and install the skill
docs, so I can ask about my day and change it in plain language.

**DONE WHEN:** the `daily-champ` skill is installed, the MCP server is in this
agent's config, and calling `get_today` returns my day.

## TODO

- [ ] Step 1 — get a token (human)
- [ ] Step 2 — connect the MCP server
- [ ] Step 3 — install the skill docs
- [ ] Step 4 — restart and verify

---

## Step 1 — get a token · **HUMAN**

Only the account holder can do this. If you are an agent reading this file, tell
the user to do it and wait for them.

1. Open Daily Champ and sign in.
2. **Settings** → **API tokens** → **New token**. Name it after this machine.
3. Copy the token. It is shown once and never again.
4. Export it in the shell this agent runs in:

```bash
export DAILY_CHAMP_TOKEN='paste-it-here'
```

Put that line in your shell profile so it survives a new terminal. The token is
one device's access — revoking it in Settings cuts this machine off and nothing
else. There is no OAuth browser step and no account subdomain to remember.

## Step 2 — connect the MCP server

Run the one for this client.

**Claude Code**

```bash
claude mcp add --transport http --scope user daily-champ \
  https://daily-champ.deliverists.io/mcp \
  --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"
```

`--scope user` makes it available in every project. Drop it for this project
only.

**Codex**

```bash
codex mcp add daily-champ \
  --url https://daily-champ.deliverists.io/mcp \
  --bearer-token-env-var DAILY_CHAMP_TOKEN
```

Codex reads the variable at call time, so the token never lands in
`~/.codex/config.toml`.

**Editing the config by hand** — `~/.claude.json` under `mcpServers`, or a
project-level `.mcp.json`:

```json
{
  "mcpServers": {
    "daily-champ": {
      "type": "http",
      "url": "https://daily-champ.deliverists.io/mcp",
      "headers": {
        "Authorization": "Bearer ${DAILY_CHAMP_TOKEN}"
      }
    }
  }
}
```

**Other clients**

- **OpenCode** — under `"mcp"` in `opencode.json`:
  `"daily-champ": { "type": "remote", "url": "https://daily-champ.deliverists.io/mcp", "headers": { "Authorization": "Bearer {env:DAILY_CHAMP_TOKEN}" } }`
- **Cursor / VS Code** — the same block in `.cursor/mcp.json` or `.vscode/mcp.json`.
- **An agent with no MCP support** — run it over stdio:
  `npx mcp-remote https://daily-champ.deliverists.io/mcp --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"`

## Step 3 — install the skill docs

```bash
npx skills add DeliveristsIO/daily-champ-skills
```

That installs the skill using the [Agent Skills](https://agentskills.io)
standard, detecting the agent for you (Claude Code, Cursor, Codex, VS Code,
Goose, Amp and others). Use `-a claude-code` to name one, `-g` to install
globally. Check it landed with `npx skills list`.

**Or, in Claude Code, do steps 2 and 3 at once** — the plugin carries the server
config with it:

```
/plugin marketplace add DeliveristsIO/daily-champ-skills
/plugin install daily-champ@deliverists
```

You still need `DAILY_CHAMP_TOKEN` exported from step 1.

## Step 4 — restart and verify

Restart the session so the new server and skill are picked up, then ask:

> what's on today?

The agent should call `get_today` and come back with the date, whether a clock
is running, and the tasks planned. If it does not, check the raw surface:

```bash
curl -s https://daily-champ.deliverists.io/mcp \
  -H "Authorization: Bearer $DAILY_CHAMP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":"1","method":"tools/list"}' | grep -o start_timer
```

`start_timer` in the output means the server is reachable and the token is good.
A `401` means the token is wrong, expired or revoked — go back to step 1.

**EXECUTE NOW:** start with Step 1 by asking the user for a token, then carry on
through Step 4 without further prompting.

---

## Manual installation

Do not execute this section unless explicitly asked.

```bash
git clone https://github.com/DeliveristsIO/daily-champ-skills ~/.daily-champ-skills
mkdir -p ~/.claude/skills
ln -sfn ~/.daily-champ-skills/skills/daily-champ ~/.claude/skills/daily-champ

# per project instead:
mkdir -p .claude/skills
ln -sfn ~/.daily-champ-skills/skills/daily-champ .claude/skills/daily-champ

# update:
cd ~/.daily-champ-skills && git pull
```

Symlinks pick up changes with no reinstall.
