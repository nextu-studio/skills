# Next U video pipeline (S7 → S12 → S14/S16/S18)

For **video** projects. One true generation step (the script), one optional scene-photo step, then deterministic derivations. Only the script uses your model; everything downstream is parsed from it server-side, so it matches the website exactly.

> **Scope:** this file covers the **authoring half only** — the prompts and the storyboard. It is
> free and no media exists at the end of it. Rendering the actual frames, scene videos, music and
> the final master is the **production half**, and the CLI does all of it (`nextu plan` →
> `generate-image` / `generate-video` / `generate-music` → `compose`). See **SKILL.md §6**.
> Don't finish this file and send the user to the website — they don't have to leave the terminal.

## 0. Prerequisites

- Project `output` = `video` (`nextu create … --output video` or `nextu set-inputs <id> --output video`).
- A product name (set / extracted).
- Recommended before the script — set the shoot settings so the script is written for them:

```bash
nextu set-settings <id> \
  --video-model grok \          # grok | grok15 | seedance | seedance25
  --direction imagePromo \      # oneShot | imagePromo | unboxing
  --video-ar 9:16 \
  --video-res 1080p \
  --duration 15                 # seconds
# unboxing style with a person on camera:
#   nextu set-settings <id> --direction unboxing --unboxing-person
```

Use `nextu options` to see the legal `--video-model` / `--direction` / aspect-ratio / resolution values, and the marketing-concept list for the product's category.

## 1. Script — `s7_script` (generate)

The only LLM step. `prepare` returns the prompt + a save-contract; generate the script with your own model per that contract, then save.

```bash
nextu prepare <id> s7_script            # optionally --concept "…" and/or --language en
#   → generate the JSON envelope your model produces, following the contract, then:
nextu save <id> s7_script --file script.json     # or --stdin, or inline
```

The saved script is a JSON envelope whose `rawMarkdown` lays out the shoot as **Step 4** (per-scene starting frames), **Step 5** (per-scene motion / video prompts), **Step 6** (audio & music), and **Step 7** (logo). The exact envelope shape is defined by the `prepare` contract — follow it; do not hand-author the structure from memory. Saving a new script clears any previously derived downstream fields.

## 2. Scene photography prompt — `s12_scene_photo` (generate, optional)

A scene / environment photography prompt, seeded from the script's scene breakdown. Needs a saved `s7_script`.

```bash
nextu prepare <id> s12_scene_photo
#   → generate the prompt text per contract, then:
printf '%s' "$PROMPT" | nextu save <id> s12_scene_photo --stdin
```

## 3. Derive downstream steps (no generation)

Each reads the saved script and runs Next U's own parser server-side — identical to what the website computes. Nothing for you to write.

```bash
nextu derive <id> s14_starting_frames   # from Step 4: per-scene first-frame prompts (+ brand-logo injection),
                                        #   scene assets, resets scene index, clears S15 reference selections
nextu derive <id> s16_video_prompts     # from Step 5: per-scene motion / video prompts (+ records source script)
nextu derive <id> s18_music             # from Step 6: background-music prompt (+ lyrics if enabled).
                                        #   If the script has no music / BGM is off, this is a no-op.
```

All three are **video-only** (they refuse on photo/copy projects). Each returns `applied` = the fields written; re-running after the script is unchanged is safe.

## Typical end-to-end

```bash
id=$(nextu create --title "Tiger cap promo" --output video --json | jq -r .id)
nextu extract   "$id" "https://shop.example/tiger-cap"
nextu set-inputs "$id" --category 6
nextu set-settings "$id" --video-model grok --direction imagePromo --video-ar 9:16 --duration 15
nextu prepare   "$id" s7_script            # → generate → save
nextu save      "$id" s7_script --file script.json
nextu derive    "$id" s14_starting_frames
nextu derive    "$id" s16_video_prompts
nextu derive    "$id" s18_music
```

The project — script, scenes, frame prompts, motion prompts, music — is now populated and synced to the user's Next U account. **This is the halfway point, not the finish line:** nothing has been rendered yet. Continue into the production half (SKILL.md §6), which the CLI runs end to end:

```bash
nextu plan "$id"                         # ordered steps + estimated credits + balance (no charge)
# → generate-image (start frames) → generate-video (scenes, in order) → generate-music → compose
```

The user can also open the web app and render there instead — but they don't have to.

## Person & outfit prompts (S10 / S11A) — deterministic derives

When the ad has a spokesperson, two more prompt steps are available as **`derive`** (template fills, no generation):

```bash
# Spokesperson "character identity sheet" prompt. Needs a KOL (set --kol/--name, or attach --role kol).
nextu derive <id> s10_person_photo                       # defaults: bg="pure white", expression="微笑"(smile)
nextu derive <id> s10_person_photo --bg "navy blue" --angles 微笑,開心

# Outfit-change prompt (dress the person in the product). Needs a KOL.
nextu derive <id> s11a_outfit                            # simple (clothing only) or full (clothing + accessories),
nextu derive <id> s11a_outfit --accessories              #   auto-derived from category / uploads, or forced with the flag
```

`s10` mirrors the autopilot exactly; `s11a` mirrors the manual S11A step (which the autopilot deliberately skips because it needs user-supplied outfit references). Both write the prompt back to the project (`personPhotoPrompt` / `outfitPrompt`) — they produce **prompts, not pictures**. Rendering those images is the paid production half, and the CLI does it: `nextu generate-image <id> 11` (person, S11) and `nextu generate-image <id> 112` (outfit-changed, S11B).

## Notes

- If a non-clothing product's script needs "wearable" wording, note that the CLI path uses only deterministic category rules (it never calls a classifier LLM), so ambiguous wearables may need the category set explicitly (`nextu set-inputs <id> --category <apparel id>`).
