---
name: nextu-ad-studio
description: "Create advertising end-to-end with Next U (nextu.studio) from a coding agent — ad copy, product-photo prompts, video scripts, AND the actual media (product/person/scene images, per-scene video, background music) plus the final composed master video — driven by the `nextu` CLI. Use when the user wants to make ads or marketing content for a product, drive the Next U ad pipeline, generate ad images/video/music, turn a product URL or local photos into a finished ad, or otherwise use the `nextu` command. Two halves — the authoring half (copy/prompts/scripts) is FREE, your own model writes it; the production half (images/video/music/compose) runs through Next U's paid pipeline and deducts the account's Next U credits (never your own API keys — a logged-in Next U member with credits is required). Bilingual (defaults to Traditional Chinese; pass `--language en` for English)."
---

# Next U Ad Studio (via the `nextu` CLI)

Next U is an AI ad-creation studio. This skill drives it from the terminal with the `nextu` CLI.

## Two halves — free authoring, paid production

The pipeline splits in two. Know which half you're in:

- **Authoring half (FREE)** — copy, prompts, scripts, and deterministic derivations. **Next U never calls a paid model here; YOU generate with your own model** (`prepare` → generate → `save`), and `derive` runs Next U's own parser. Zero credits.
- **Production half (PAID)** — the actual media: images (`generate-image`), music (`generate-music`), per-scene video (`generate-video`), and the final `compose`. These run through **Next U's own paid vendor pipeline and deduct the account's Next U credits.** There is no "bring your own API key" — the user must be a logged-in Next U member with credits. Each paid command reports its cost and the remaining balance; insufficient balance returns a clear error. **Run `nextu plan <id>` first to see the ordered steps and the total estimated cost before spending.**

## The one idea to understand first (authoring half)

**In the authoring half, Next U never calls a paid model. YOU generate with your own model.** Every text step is a three-move loop:

1. `nextu prepare <id> <step>` → prints an **instruction**, a **save-contract** (回存合約), and a **generation prompt** (plus any product/reference images).
2. **You generate** the content with your own model, following the printed save-contract **exactly**.
3. `nextu save <id> <step> …` → validates and writes it back to the project.

> **Anti-drift rule:** the exact output format for each step is whatever the `prepare` save-contract says at runtime. Read it and follow it verbatim. Never hardcode or guess a format from memory — if a step's format changes server-side, the contract changes with it and you stay correct.

## Setup (once)

```bash
npx nextu-cli --help             # or: npm i -g nextu-cli
nextu login                           # prompts for the token; input is NOT echoed
nextu status                          # confirm site / token / endpoint is armed
```

For local dev add `--url http://localhost:3000`. In CI or any non-interactive context, pipe it
instead — **never** pass the token as a command-line argument:

```bash
echo "$NEXTU_TOKEN" | nextu login --token-stdin
```

> `--token <value>` still works for compatibility but is discouraged and prints a warning: an
> argument lands in shell history, `ps` output, CI logs and your own transcript, and deleting the
> config file does not take it back.

Token is stored at `~/.nextu/config.json` (0600). Env overrides: `NEXTU_API_TOKEN`, `NEXTU_BASE_URL`. Precedence: flag > env > config > default (`https://nextu.studio`). The token is only ever sent to trusted hosts (nextu.studio + subdomains over https, or localhost) — a hostile `--url` cannot exfiltrate it.

## Workflow

### 1. Create a project

```bash
nextu create --title "Tiger-print cap" --output copy    # output: copy | photo | video
# → prints the project id; use it in every later command
```

### 2. Give it a product (pick whatever the user has)

| The user has… | Use |
|---|---|
| a product **page URL** | `nextu extract <id> "https://…"` — auto-fills name/description/images/logo (non-destructive) |
| **local image files** | `nextu attach <id> ./cap.jpg --role product` — roles: `product` (append) · `kol` (spokesperson, replace) · `logo` (replace). **This is the CLI's unique power — a remote MCP server can't read the user's disk.** |
| just facts | `nextu set-inputs <id> --category 6 --name "…" --desc "…" [--kol --kol-name "…"]` |

You can combine them. `--category` must be a valid id — see step 3.

### 3. Check valid values before setting them

```bash
nextu options            # or: nextu options --json
```

Returns the legal category ids (with names), output functions, video models, direction styles, aspect ratios, resolutions, and **the marketing-concept list per category**. Always pick `--category`, `--video-model`, `--direction`, and any `--concept` from here — don't invent values.

### 4. Generate

Run `prepare`, generate with your own model per the contract, then `save`.

```bash
nextu prepare <id> s6_copy                          # ad copy (JSON envelope per contract)
#   → generate with your model, then:
printf '%s' "$YOUR_JSON" | nextu save <id> s6_copy --stdin
```

Steps you can generate (all go through the same `prepare` → generate → `save` loop):

- **`s6_copy`** — ad copy (headline / subheading / key benefits / CTA / social captions). JSON.
- **`s8_product_prompt`** — a product-photography prompt (studio 9-grid style). Plain text.
- **`s12_scene_photo`** — a scene / environment photography prompt. Plain text. Requires an existing `s7_script`.
- **`s7_script`** — a full short-video script (JSON envelope with `rawMarkdown` covering Step 4/5/6/7). Set video settings first (step 5) and read `reference/video-pipeline.md`.

Marketing concept (s6_copy / s7_script only): `prepare` shows the applied concept and the options for the category. To choose one: `nextu prepare <id> s6_copy --concept "Social Proof"` (or a ConceptKey like `core1`), then generate again.

Language: append `--language en` for English output (default is Traditional Chinese).

Vision steps attach product/spokesperson reference images — add `--save-images <dir>` to write them to disk, or read them from the `--json` `images[]` (base64) to feed your own vision model.

### 5. Video pipeline (only if output is video)

Set settings, generate the script, then **derive** the downstream steps deterministically (no model needed). See **`reference/video-pipeline.md`** for the full sequence. In short:

```bash
nextu set-settings <id> --video-model grok --direction imagePromo --video-ar 9:16 --duration 15
nextu prepare <id> s7_script   # → generate script per contract → nextu save <id> s7_script --file script.json
nextu derive  <id> s14_starting_frames   # first-frame prompts (parsed from your script)
nextu derive  <id> s16_video_prompts     # per-scene motion prompts
nextu derive  <id> s18_music             # background-music prompt
```

`derive` runs Next U's own parser server-side over the saved script — its output matches the website exactly, so there is nothing for you to generate and no drift.

### 6. Production — generate the media (PAID, deducts the account's credits)

This half runs Next U's paid vendor pipeline. **Always `nextu plan <id>` first** to see the ordered steps + total estimated cost + whether the balance is sufficient.

```bash
nextu plan <id>                          # ordered steps + estimated total cost + balance (no charge)
```

All generation is **asynchronous**: a `generate-*` / `compose` command returns a `jobId` immediately (after charging), then you poll for the result. The result also writes back into the project.

```bash
# Images — slots: 9 product · 11 person · 13 scene · 112 outfit-changed · 15 per-scene start frame
nextu generate-image <id> 9              # → jobId + cost + balance
nextu generation <id> <jobId>            # poll until "✅ 完成" (done); re-run every few seconds
nextu generate-image <id> 15 --scene 0   # step 15 needs --scene N (0-based), one per storyboard scene

# Music (S19) — needs `derive s18_music` first
nextu generate-music <id>
nextu generation <id> <jobId>

# Video (S17) — needs `derive s16_video_prompts` first. Generate scenes IN ORDER 0,1,2…
nextu generate-video <id> 0              # oneShot/unboxing auto-chain the previous scene's last frame — order matters
nextu generation <id> <jobId>            # poll until done before starting the next scene

# Final master (S22) — free (compute only, no credits). Best after `derive s21_edit`.
nextu derive <id> s21_edit               # scene order / subtitles / BGM pick / auto-compose flags
nextu compose <id>                       # → jobId
nextu get-compose <id> <jobId>           # poll until "✅ 母片完成" → playable/downloadable master URL
```

**Prompts and reference images come from the authoring half** — `generate-image` for slot 11 uses the S10 person prompt, slot 9 uses the S8 product prompt, slot 15 uses the S14 starting-frame prompt, etc. So the order is: finish authoring (prompts + derives) → then production. `plan` lays out the exact sequence for the project.

Zero-drift & no-loophole guarantees: prompts / reference-image selection / parameters / credit deduction are the **same code** the website uses — an MCP/CLI-generated ad equals what the website would produce, and every paid generation goes through Next U's credit ledger (no external API keys).

## Rules & gotchas

- **Follow the `prepare` save-contract exactly.** It is the source of truth for each step's format.
- **Every command takes `--json`** — use it when scripting so you parse structured data, not prose. `prepare --json` returns `{ instructions, contract, prompt, images[], appliedConcept, conceptOptions[] }`; `save`/`derive --json` return `{ applied: [...] }`.
- **`save`/`derive` return `applied`** = the fields actually written. Empty/duplicate is a no-op, not an error.
- **Ordering:** product name must exist before `prepare` (set it, extract it, or the extract must have filled it). `s12_scene_photo` and every `derive` step need a saved `s7_script` first. `s7_script` is best after `set-settings` (video model / direction / duration).
- **Errors are actionable:** 422 lists exactly which fields are wrong → fix and re-`save`. 404 → wrong project id (`nextu projects`). 401 → re-`login`. 503 → the site's MCP endpoint is dormant.
- **Non-destructive by default:** `extract` only fills blank fields and de-dupes images; re-running is safe.
- **All changes sync to the user's Next U account** and appear in the web app.

## Reference

- `reference/commands.md` — every command and flag, mirroring the CLI exactly.
- `reference/video-pipeline.md` — the full S7 → S12 → S14/S16/S18 video sequence with step semantics and prerequisites.
