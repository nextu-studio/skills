# Next U — Agent Skills

Skills for **[NEXT U](https://nextu.studio)** ads, with two execution environments:

- `nextu-ad-mcp`: connected remote MCP tools in ChatGPT and other MCP clients; account login uses browser OAuth. No terminal or CLI token in the conversation.
- `nextu-ad-studio`: local `nextu` CLI in coding agents or ChatGPT Work with terminal access, an installed CLI and a local NEXT U login.

This directory is a self-contained bundle — its contents are mirrored to the `nextu-studio/skills` repo (manual sync), and it doubles as a Claude Code **plugin marketplace**.

## What's inside

```
.claude-plugin/
  marketplace.json          # Claude Code marketplace (one plugin: nextu-ad-studio)
  plugin.json               # plugin manifest; skills auto-discovered from skills/
skills/
  nextu-ad-mcp/
    SKILL.md                # remote MCP workflow and account connection
    agents/openai.yaml      # UI metadata and remote tool dependency
    references/video-workflow.md
  nextu-ad-studio/
    SKILL.md                # the skill: setup + the prepare→generate→save loop
    reference/
      commands.md           # full command/flag reference
      video-pipeline.md     # S7 script → S12 → derive S14/S16/S18
```

Root `plugin.json` and `mcp.json` provide a portable Agent Plugins package. The
existing Claude manifest remains available. The MCP endpoint is public HTTPS;
the package contains no credentials or pre-authorized account mapping.

## ChatGPT with remote MCP

1. Follow [OpenAI's current connection instructions](https://developers.openai.com/plugins/deploy/connect-chatgpt). Custom MCP availability depends on your account and workspace policy.
2. Connect `https://nextu.studio/api/mcp`, sign in to NEXT U in the browser and review authorization. Installing this package does not complete OAuth.
3. Load the MCP skill or plugin through the interface supported by your ChatGPT environment, then enable the connected NEXT U tools in a new conversation. Standalone/local skills and package installation availability differ by surface; see [OpenAI's skills documentation](https://learn.chatgpt.com/docs/build-skills).
4. Ask for a connection check or estimate first. For local photos, upload them to a NEXT U project on the website before asking the remote tools to use that project. A ChatGPT attachment is not automatically uploaded to NEXT U.

The versioned package is downloadable from `https://nextu.studio/skills/nextu-ad-studio-0.2.0.zip` after deployment. This source bundle is **not a published or approved OpenAI directory listing**. Public submission/review and account OAuth remain separate steps.

## CLI skill prerequisite: the `nextu` CLI + a token

The skill calls the `nextu` CLI, so it must be installed and logged in:

```bash
npx nextu-cli --help             # or: npm i -g nextu-cli
nextu login                           # prompts for the token; input is NOT echoed
nextu status
```

In CI, pipe the token instead of passing it as an argument (`echo "$NEXTU_TOKEN" | nextu login --token-stdin`) —
a command-line argument lands in shell history, `ps` output and CI logs.

Get a token at **<https://nextu.studio>** → sign in → **Studio** → **Settings (設定)** → the **CLI** tab
→ generate. The value is shown **once** — copy it before closing the dialog.

## Install the skill (pick one)

### 1. Claude Code plugin marketplace (recommended)

```
/plugin marketplace add nextu-studio/skills
/plugin install nextu-ad-studio@nextu
```

(For a local checkout instead: `/plugin marketplace add ./agent-skills`.)

### 2. `npx skills add` (Agent-Skills tooling)

```bash
npx skills add nextu-studio/skills
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

The pipeline has two halves and the skill drives both.

**Authoring (free)** — Next U never runs a paid model here; **your agent's model does the generation**. The loop is `nextu prepare` (get prompt + save-contract) → generate with your own model → `nextu save`, plus project setup (product URL extraction, local-image upload, valid-option lookup) and the deterministic video pipeline (`nextu derive`). Output comes in four languages (zh-TW / en / ja / ko).

**Media generation (paid)** — product/person/scene images, per-scene video and background music use NEXT U's vendor pipeline. Credits are deducted when generation starts; failures are refunded automatically. Review `nextu plan` / `plan_pipeline` and resolve incomplete estimates before spending. There is no bring-your-own-API-key.

**Final composition (free)** — `compose` / `compose_video` joins the existing media into the final master and deducts **0 NEXT U credits**. It queues work and creates an output. Credit unit value is not the price of a finished ad.

It is deliberately anti-drift: the exact output format for each step comes from the save-contract that `prepare` prints at runtime, so the skill stays correct even as formats evolve server-side.

## License

MIT (see the hosting repo).
