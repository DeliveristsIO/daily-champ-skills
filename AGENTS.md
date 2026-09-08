# AGENTS.md

Distribution repo for the Daily Champ agent skill. The app itself lives in
`daily-champ`.

## Layout

```
.claude-plugin/     Claude Code plugin + marketplace manifests
skills/daily-champ/ SKILL.md — the one skill
install.md          Written as an executable agent prompt, not as prose
```

## The rule that matters

**The tool table in `SKILL.md` is generated, not written.** It sits between
`<!-- tools:start -->` and `<!-- tools:end -->`, and it comes from the live
registry in the app:

```bash
cd ../daily-champ && bin/rails mcp:skill
```

`test/agent/skill_parity_test.rb` in the app fails the build when the table is
stale, when a registered tool goes unmentioned, when the skill names a tool that
does not exist, or when a worked example passes an argument the schema has no
such key for. So a tool added, renamed or dropped in the app breaks the test
until the skill follows.

Everything outside the markers is written by hand — the concepts, the rules, the
workflows and the gotchas. That is the part the tool schemas cannot carry, and
it is the reason the skill exists.

## Adding a tool

1. Register it in `lib/agent/registry.rb` in the app.
2. Run `bin/rails mcp:skill` to refresh the table here.
3. If it covers something new, add a workflow and any triggers for it by hand.
4. Run `bin/rails test test/agent/` in the app.

## Checking it works

Drive the server directly — stateless streamable HTTP, so a bare `tools/call`
POST needs no handshake and no session id. The curl is in `install.md`, step 4.
