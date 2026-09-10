# Install the Daily Champ skill

I want you to install Agent Skills for Daily Champ for me. Execute all steps
autonomously, except the one marked **HUMAN** — stop and wait for that one.

**OBJECTIVE:** connect this agent to Daily Champ over MCP and install the skill
docs, so I can ask about my day and change it in plain language.

**DONE WHEN:** the `daily-champ` skill is installed, the MCP server is in this
agent's config, and calling `get_today` returns my day.

## TODO

- [ ] Step 1 — connect the MCP server
- [ ] Step 2 — install the skill docs
- [ ] Step 3 — sign in (human) and verify

---

## Step 1 — connect the MCP server

Daily Champ speaks standard MCP OAuth, so there is **nothing to copy**. Add the
server with no header and the sign-in happens in a browser.

**Claude Code**

```bash
claude mcp add --transport http --scope user daily-champ https://www.dailychamp.net/mcp
```

`--scope user` makes it available in every project. Drop it for this project
only.

**Codex**

```bash
codex mcp add daily-champ --url https://www.dailychamp.net/mcp
codex mcp login daily-champ
```

**OpenCode**

```bash
opencode mcp add daily-champ --type remote --url https://www.dailychamp.net/mcp
opencode mcp auth daily-champ
```

Or by hand under `"mcp"` in `opencode.json` (global:
`~/.config/opencode/opencode.json`):

```json
{
  "mcp": {
    "daily-champ": { "type": "remote", "url": "https://www.dailychamp.net/mcp" }
  }
}
```

**pi** — pi has no built-in MCP, so the adapter is the way in:

```bash
pi install npm:pi-mcp-adapter
```

Restart pi, then add the server under `"mcpServers"` in `.mcp.json` (this
project) or `~/.config/mcp/mcp.json` (every project):

```json
{
  "mcpServers": {
    "daily-champ": { "url": "https://www.dailychamp.net/mcp" }
  }
}
```

The adapter registers one `mcp` proxy tool; call it with
`{ "search": "get_today" }` to discover, then
`{ "tool": "get_today", "args": {} }` to call. Sign-in is browser OAuth like
the others: `/mcp` → **daily-champ** → authenticate.

**By hand** — `~/.claude.json` under `mcpServers`, or a project-level
`.mcp.json`:

```json
{
  "mcpServers": {
    "daily-champ": {
      "type": "http",
      "url": "https://www.dailychamp.net/mcp"
    }
  }
}
```

**Other clients**

- **OpenCode** — under `"mcp"` in `opencode.json`:
  `"daily-champ": { "type": "remote", "url": "https://www.dailychamp.net/mcp" }`
- **Cursor / VS Code** — the same block in `.cursor/mcp.json` or `.vscode/mcp.json`.
- **An agent with no MCP support** — see the token fallback below.

## Step 2 — install the skill docs

The repository is private, so this reads it over your own GitHub access. If
`gh auth status` is not green, or the command cannot find the repository, go
straight to **Manual installation** at the bottom.

```bash
npx skills add DeliveristsIO/daily-champ-skills
```

That installs the skill using the [Agent Skills](https://agentskills.io)
standard, detecting the agent for you (Claude Code, OpenCode, pi, Cursor, Codex,
VS Code, Goose, Amp and others). Use `-a claude-code` to name one, `-g` to
install globally. Check it landed with `npx skills list`.

**Or, in Claude Code, do steps 2 and 3 at once** — the plugin carries the server
config with it:

```
/plugin marketplace add git@github.com:DeliveristsIO/daily-champ-skills.git
/plugin install daily-champ@deliverists
```

You still need `DAILY_CHAMP_TOKEN` exported from step 1.

## Step 3 — sign in and verify · **HUMAN for the sign-in**

Restart the session so the new server and skill are picked up. Then the human
authorizes — an agent cannot do this part:

> `/mcp` → **daily-champ** → **Authenticate** → sign in → **Authorize**

Then ask:

> what's on today?

The agent should call `get_today` and come back with the date, whether a clock
is running, and the tasks planned. If it does not, check the raw surface:

```bash
curl -s https://www.dailychamp.net/mcp \
  -H "Authorization: Bearer $DAILY_CHAMP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":"1","method":"tools/list"}' | grep -o start_timer
```

`start_timer` in the output means the server is reachable and the credential is
good. A `401` means the sign-in did not complete — run `/mcp` again.

**EXECUTE NOW:** start with Step 1, carry on through Step 3, and stop only to
ask the user to authorize in their browser.

---

## Fallback: an API token

For a headless machine, CI, or a client with no OAuth support. Both credentials
go on the same `Authorization: Bearer` header and reach the same tools.

1. Open Daily Champ → **Settings** → **Devices** → **New device token**. Name it
   after this machine.
2. Copy it — it is shown once and never again — and export it, ideally from your
   shell profile so it survives a new terminal:

```bash
export DAILY_CHAMP_TOKEN='paste-it-here'
```

3. Add the server with the header:

```bash
# Claude Code
claude mcp add --transport http --scope user daily-champ \
  https://www.dailychamp.net/mcp \
  --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"

# Codex — reads the variable at call time, so the token never lands in config.toml
codex mcp add daily-champ \
  --url https://www.dailychamp.net/mcp \
  --bearer-token-env-var DAILY_CHAMP_TOKEN

# OpenCode
opencode mcp add daily-champ --type remote --url https://www.dailychamp.net/mcp \
  --header "Authorization: Bearer {env:DAILY_CHAMP_TOKEN}"
# (or by hand: the "remote" block with "oauth": false and the header, see SKILL.md)

# pi — into ~/.config/mcp/mcp.json or .mcp.json
{
  "mcpServers": {
    "daily-champ": {
      "url": "https://www.dailychamp.net/mcp",
      "headers": { "Authorization": "Bearer ${DAILY_CHAMP_TOKEN}" }
    }
  }
}

# stdio-only agents
npx mcp-remote https://www.dailychamp.net/mcp \
  --header "Authorization: Bearer $DAILY_CHAMP_TOKEN"
```

A token is one device's access. Revoking it under **Settings → Devices** cuts
that machine off and nothing else. An agent you signed in through the browser
lives under **Settings → Agents**, where **Disconnect** ends its access at once.

---

## Manual installation

Do not execute this section unless explicitly asked — except when the commands
above could not reach the private repository, in which case this is the way in.
It needs an SSH key with access to the DeliveristsIO organisation.

```bash
git clone git@github.com:DeliveristsIO/daily-champ-skills.git ~/.daily-champ-skills
mkdir -p ~/.claude/skills
ln -sfn ~/.daily-champ-skills/skills/daily-champ ~/.claude/skills/daily-champ

# per project instead:
mkdir -p .claude/skills
ln -sfn ~/.daily-champ-skills/skills/daily-champ .claude/skills/daily-champ

# update:
cd ~/.daily-champ-skills && git pull
```

Symlinks pick up changes with no reinstall.
