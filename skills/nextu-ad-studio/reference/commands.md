# `nextu` command reference

Mirrors the CLI exactly. Global flags work on every command: `--url <baseUrl>`, `--json` (raw JSON for parsing), `-h/--help`, `--version`. (`--token <token>` also exists but is discouraged — see `login` below; a token passed as an argument lands in shell history, `ps` output and CI logs.) `--json` prints the raw API response instead of formatted text — use it when scripting.

Precedence for site/token: `--flag` > env (`NEXTU_BASE_URL` / `NEXTU_API_TOKEN`) > `~/.nextu/config.json` > default (`https://nextu.studio`).

## Auth & status

| Command | Purpose |
|---|---|
| `nextu login [--url <baseUrl>]` | Prompts for the token (input not echoed) and stores it in `~/.nextu/config.json` (0600 on POSIX; ACL tightened via `icacls` on Windows). Refuses untrusted/non-https hosts. Verifies against `/api/mcp/options`. |
| `nextu login --token-stdin` | Same, but reads the token from stdin — use this in CI: `echo "$NEXTU_TOKEN" \| nextu login --token-stdin`. |
| `nextu logout` | Delete the local config. |
| `nextu status` | Show current site / token (masked) / health / whether the MCP endpoint is armed, and where each value came from. |
| `nextu health` | Public site health (backend configured? FX fresh?). No token needed. |

## Read

| Command | Purpose |
|---|---|
| `nextu projects` | List your projects (title · output function · current step · updated · id). |
| `nextu project <id>` | Read one project: metadata + a bounded summary of its `data` (top-level keys + size, never the full blob). |
| `nextu options` | Legal values: category ids + names, output functions, video models, direction styles, aspect ratios, resolutions, and the marketing-concept list per category. |

## Create & configure

| Command | Purpose |
|---|---|
| `nextu create [--title <t>] [--output copy\|photo\|video]` | Create a project; prints its id. |
| `nextu set-inputs <id> [flags]` | S1–S5 inputs. Flags: `--output copy\|photo\|video`, `--category <N>`, `--name <s>`, `--desc <s>`, `--kol` / `--no-kol`, `--kol-name <s>`, `--kol-style <s>`, `--kol-notes <s>`. Only the flags you pass are updated. |
| `nextu set-settings <id> [flags]` | S5.5 generation settings. Flags: `--video-model grok\|seedance`, `--direction oneShot\|imagePromo\|unboxing`, `--unboxing-person` / `--no-unboxing-person`, `--image-ar <r>`, `--video-ar <r>`, `--image-res <r>`, `--video-res <r>`, `--duration <N>` (or a value that clears it), `--mode manual\|auto`. |

## Bring in a product

| Command | Purpose |
|---|---|
| `nextu extract <id> <url>` | Import name / description / images / logo from a product page URL. Non-destructive: only fills blank fields, de-dupes images, sets logo only if empty. |
| `nextu attach <id> <path> --role product\|kol\|logo` | Upload a **local** image file (≤10 MB; .jpg/.png/.webp/.gif). `product` appends (max 12), `kol` and `logo` replace. Remote MCP servers can't do this — reading the local disk is CLI-only. |

## Generate (prepare → your model → save)

| Command | Purpose |
|---|---|
| `nextu prepare <id> <step> [--language zh-TW\|en] [--concept <c>] [--save-images <dir>]` | Get the generation prompt + save-contract for a step, without calling any paid model. `step` ∈ `s6_copy` \| `s7_script` \| `s8_product_prompt` \| `s12_scene_photo`. `--concept` (s6_copy/s7_script only) picks a marketing concept by ConceptKey or name. `--save-images` writes attached reference images to a folder. |
| `nextu save <id> <step> <result>` | Write your generated result back. Provide the result as the 3rd argument, or `--file <path>`, or `--stdin`. `step` must match the `prepare` step. Format must satisfy the contract `prepare` printed (422 lists any violations). |

Steps: `s6_copy` = ad copy (JSON) · `s8_product_prompt` = product-photo prompt (text) · `s12_scene_photo` = scene-photo prompt (text; needs a saved `s7_script`) · `s7_script` = video script (JSON envelope). Concept applies to `s6_copy` and `s7_script`.

## Derive (deterministic; no model)

| Command | Purpose |
|---|---|
| `nextu derive <id> <step> [flags]` | Compute a downstream step deterministically using Next U's own logic (zero drift, no generation). `step` ∈ `s14_starting_frames` \| `s16_video_prompts` \| `s18_music` \| `s10_person_photo` \| `s11a_outfit`. |

Step notes for `derive`:
- `s14_starting_frames` / `s16_video_prompts` / `s18_music` — parsed from the saved `s7_script`; video projects only.
- `s10_person_photo` — spokesperson "character identity sheet" prompt (template fill). Needs a spokesperson (`--kol`/`--name`, or `attach --role kol`). Flags: `--bg <pure white|light grey|dark grey studio|navy blue>`, `--angles 微笑,開心` (Chinese expressions only: 微笑/開心/溫柔/滿足/驚喜/憤怒/悲傷; empty = none). Invalid `bg`/`angles` are rejected (must match the website's options).
- `s11a_outfit` — outfit-change prompt (template fill). Needs a spokesperson. Flag: `--accessories` / `--no-accessories` (else auto-derived from product category / uploaded accessory images).

## Plan & estimate (read-only, no charge)

| Command | Purpose |
|---|---|
| `nextu plan <id>` | The one-shot plan + **cost estimate**: which tools to call in what order, which are paid, the estimated total credits, and your current balance / whether it's sufficient. Run before spending. Video scene count is exact only after `derive s16_video_prompts`. |

## Generate media (PAID — deducts the account's Next U credits)

Asynchronous: each returns a `jobId` (after charging); poll for the result, which writes back into the project. Every response reports `cost` (credits spent this call; 0 on an idempotent re-hit) and `balance`.

| Command | Purpose |
|---|---|
| `nextu generate-image <id> <9\|11\|13\|15\|112> [--scene N]` | Generate one image for a slot: 9=product · 11=person · 13=scene · 112=outfit-changed person · 15=per-scene start frame (**requires `--scene N`**, 0-based). Prompt + reference images come from the matching authoring step (S8/S10/S12/S14 …). |
| `nextu generate-music <id>` | Generate background music (S19). Requires `derive s18_music` first. Provider (Minimax 50 / Suno 20 credits) follows the project's setting. |
| `nextu generate-video <id> <sceneIndex>` | Generate one video scene (S17). Requires `derive s16_video_prompts` first. **Generate scenes in order 0,1,2…** — oneShot/unboxing chain the previous scene's last frame server-side. |
| `nextu generation <id> <jobId>` | Poll an image/music/video job. `done=true` → result URL (written back). Failed → auto-refunded. `written=false` on a done video means the scene changed since start — the URL is still returned. |

## Compose the final master (S22 — free, compute only)

| Command | Purpose |
|---|---|
| `nextu compose <id>` | Stitch all scene videos + music + subtitles into the final master. **No credits** (internal FFmpeg step). Best after `derive s21_edit`. Idempotent on the same materials. |
| `nextu get-compose <id> <jobId>` | Poll the compose job. `done=true` → playable/downloadable master URL (also persisted to the works library, cross-device). |

## Exit codes

`0` success; non-zero on error. With `--json`, machine-readable output is printed even on failure so a Skill can branch on `applied`, `error`, or HTTP status.
