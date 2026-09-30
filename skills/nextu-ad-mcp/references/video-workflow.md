# Video workflow with NEXT U MCP

Read current descriptors and use `get_project_options` before selecting settings.
This is the dependency order; skip completed steps only when the project and
runtime contracts establish that their outputs are still usable.

1. Set product facts and video settings. Use a product URL or website-uploaded
   assets; the remote server cannot read a local file path.
2. Prepare/save `s7_script` using its runtime JSON save-contract and target language.
3. Prepare/save `s8_product_prompt` for product-photo instructions when needed.
   If the chosen direction needs a person, derive `s10_person_photo`; derive
   `s11a_outfit` only for an intended outfit change with required website-uploaded
   references. Do not add a spokesperson to a product-only request.
4. Prepare/save `s12_scene_photo` when the script calls for scene photography.
5. Derive `s14_starting_frames`, `s16_video_prompts` and `s18_music` from the saved
   script. Scene indexes start at zero. Do not invent a scene count or infer one
   from a truncated project summary.
6. Call `plan_pipeline`. Report the estimate, balance and warnings. Incomplete
   estimates need prerequisites, then another plan. Obtain authorization for
   paid outputs before starting any generation.
7. Generate required product/person/scene images with `generate_image` steps
   `9`, `11`, `13`, `112` as applicable. Generate starting frames with step `15`
   and the corresponding `sceneIndex`. Store each job ID and finish its
   `get_generation` polling before dependent media uses the result.
8. Generate video scenes with `generate_video({projectId, sceneIndex})` in order,
   waiting for each result when the next scene depends on its ending frame.
   Generate requested background music with `generate_music`, then retrieve it
   through `get_generation`. Honor the selected settings and authorized budget.
9. Derive `s21_edit` and use `set_project_settings` for intended scene subtitles
   or subtitle enablement. Do not silently replace existing editing choices.
10. Call `compose_video({projectId})`, store its job ID, and poll
    `get_compose({projectId, jobId})`. Composition costs **0 NEXT U credits**;
    paid image/video/music generation is charged when it starts and failures
    are automatically refunded. Return the actual final URL on success.

`get_generation` may write assets back and finalize/refund jobs. `get_compose`
reads the composition status; the worker creates the final master separately.
If a tool returns pending, preserve the IDs and report the pending step after a
bounded polling batch. If it returns failed/canceled, report the actual error;
do not promise a finished asset or blindly start another paid generation.
