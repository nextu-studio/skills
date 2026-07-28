# Next U — Agent Skills

Skills that teach a coding agent (Claude Code, Cursor, and other Agent-Skills / MCP-capable clients) to drive **[Next U](https://nextu.studio)**, the AI ad-creation studio, through the `nextu` CLI.

This directory is a self-contained bundle — its contents are the intended root of the public `nextu-ai/skills` repo, and it doubles as a Claude Code **plugin marketplace**.

## What's inside

```
.claude-plugin/
  marketplace.json          # Claude Code marketplace (one plugin: nextu-ad-studio)
  plugin.json               # plugin manifest; skills auto-discovered from skills/
skills/
  nextu-ad-studio/
    SKILL.md                # the skill: setup + the prepare→generate→save loop
    reference/
      commands.md           # full command/flag reference
      video-pipeline.md     # S7 script → S12 → derive S14/S16/S18
```

## Prerequisite: the `nextu` CLI + a token

The skill calls the `nextu` CLI, so it must be installed and logged in:

```bash
npx nextu-cli --help             # or: npm i -g nextu-cli
nextu login                           # prompts for the token; input is NOT echoed
nextu status
```

In CI, pipe the token instead of passing it as an argument (`echo "$NEXTU_TOKEN" | nextu login --token-stdin`) —
a command-line argument lands in shell history, `ps` output and CI logs.

Get a token from your Next U account under **Settings → CLI**.

## Install the skill (pick one)

### 1. Claude Code plugin marketplace (recommended)

```
/plugin marketplace add nextu-ai/skills
/plugin install nextu-ad-studio@nextu
```

(For a local checkout instead: `/plugin marketplace add ./agent-skills`.)

### 2. `npx skills add` (Agent-Skills tooling)

```bash
npx skills add nextu-ai/skills
```

### 3. Manual copy

Copy the skill folder into your project (or user) skills directory:

```bash
cp -r skills/nextu-ad-studio <your-project>/.claude/skills/
# or, user-wide:
cp -r skills/nextu-ad-studio ~/.claude/skills/
```

Any Agent-Skills-aware client that reads `SKILL.md` from a skills directory will then pick it up.

## What the skill does

Next U never runs a paid model — **your agent's model does the generation**. The skill teaches the loop `nextu prepare` (get prompt + save-contract) → generate with your own model → `nextu save`, plus project setup (product URL extraction, local-image upload, valid-option lookup) and the deterministic video pipeline (`nextu derive`). It is deliberately anti-drift: the exact output format for each step comes from the save-contract that `prepare` prints at runtime, so the skill stays correct even as formats evolve server-side.

## License

MIT (see the hosting repo).
