---
name: meshy-wrapper
description: Generate, refine, rig, animate and export 3D assets with Meshy AI through the Meshy MCP server. Use when the user wants a 3D model, mesh, texture, character rig or animation created from a text prompt or a photo, needs a printable OBJ/STL/3MF, a game-ready FBX, an AR USDZ, or asks about Meshy credits, a running Meshy job, or a model they generated earlier. Also for requests like "generate a 3D model", "make a mesh from this image", "rig this character", "retexture this model", "model for 3D printing", "game-ready asset", "erzeuge ein 3D-Modell", "erstelle ein Mesh aus diesem Bild", "Charakter riggen", "Modell texturieren", "für den 3D-Druck", "Meshy Credits".
---

# meshy-wrapper

Drives [Meshy AI](https://www.meshy.ai) through the official Meshy MCP server: text or
images in, textured and optionally rigged 3D assets out.

## Read the server's own guide first

The Meshy MCP server ships a **workflow guide** (cost table + scenario playbooks A–J)
that the host injects as MCP server instructions when the server connects. That guide is
the authority on *which tool chain to run for which use case* — printing, game engine,
character animation, AR, retexture, conversion, UV unwrap, Creative Lab.

**This skill does not repeat it.** What follows is the layer around it: money, state,
where files land, and how Meshy fits into the rest of this plugin ecosystem.

If you do not see the server's guide in context, the MCP is not connected — fix that
before planning any work (see Troubleshooting).

## Mental model

Three things to hold onto:

**1. Every generation is a task, and the `task_id` is the only handle.** Tools return a
`task_id` immediately; the work happens server-side over minutes. Tasks chain: a preview
task feeds `meshy_text_to_3d_refine`, a textured task feeds `meshy_process_multicolor`, a
model task feeds `meshy_rig`, a rig task feeds `meshy_animate`. Losing a `task_id` does not
lose the work — `meshy_list_tasks` and `meshy_list_models` recover it, across sessions.

**2. Credits are the user's money.** Nearly every tool debits a shared account balance. A
wrong `target_formats` guess or a re-run "just to see" is a real charge. Treat spending as
an action requiring confirmation, always.

**3. Some choices are frozen at creation.** `target_formats` and `pose_mode` are set when
the task is created and cannot be changed afterwards — a model generated in a-pose cannot
be rigged well later, and a format not requested up front costs another task to obtain.
Ask about the *end use* before the first paid call, not after.

## Cost governance

The server's guide states the rule and carries the full price table. Apply it strictly:

- **Check `meshy_check_balance` before starting a multi-step plan.** It is free. A
  36-credit Creative Lab run against a 12-credit balance fails halfway and wastes what it
  already spent.
- **Quote the total, not the step.** "Generate + refine + rig" is 20 + 10 + 5 = 35 credits;
  present the sum and wait for a yes before the first call.
- **A failed task can still have consumed credits.** Check `consumed_credits` on the
  result rather than assuming a failure was free, and say so before re-running.
- **Never re-run a paid tool to "check something".** Use the free tools instead:
  `meshy_check_balance`, `meshy_get_task_status`, `meshy_list_tasks`, `meshy_list_models`,
  `meshy_analyze_printability`, `meshy_send_to_slicer`.
- **Cheap paths first.** Format-only change → `meshy_convert` (1 credit), not `meshy_remesh`
  (5). Real-world scale → `meshy_resize` (1 credit), not a regeneration.

## Waiting for tasks

`meshy_get_task_status` defaults to `wait=true` and auto-polls with backoff until the task
settles. **Call it once and let it block** — do not build your own poll loop, do not call
it repeatedly, do not sleep between calls.

- Its ceiling is 300 s. A long generation may return TIMEOUT while still running fine;
  call it again with the same `task_id` to resume waiting. TIMEOUT is not failure.
- Progress stalling at 95 % is normal finalization (30–120 s). **Do not cancel** — a
  cancel here forfeits the credits already spent.
- `wait=false` is for a quick status peek, e.g. when reporting on a job started earlier.

## Tool map

Grouped by what they cost you, since that is what governs the plan:

| Free | Paid — generation | Paid — post-processing |
| --- | --- | --- |
| `meshy_check_balance` | `meshy_text_to_3d` (5–20) | `meshy_text_to_3d_refine` (10) |
| `meshy_get_task_status` | `meshy_image_to_3d` (5–30) | `meshy_retexture` (10) |
| `meshy_list_tasks` | `meshy_multi_image_to_3d` (5–30) | `meshy_remesh` (5) |
| `meshy_list_models` | `meshy_text_to_image` (3–9) | `meshy_rig` (5, incl. walk + run) |
| `meshy_cancel_task` | `meshy_image_to_image` (3–12) | `meshy_animate` (3, custom only) |
| `meshy_download_model` | `meshy_creative_lab` (36) | `meshy_convert` (1) · `meshy_resize` (1) |
| `meshy_analyze_printability` | | `meshy_uv_unwrap` (5) |
| `meshy_send_to_slicer` | | `meshy_repair_printability` (10) |
| | | `meshy_process_multicolor` (10) |

Downloading is free and re-runnable — a completed task can be downloaded again later in
another format *if* that format was in `target_formats` at creation.

Two easily-wasted credits:

- `meshy_animate` is for **custom** animations only. Walking and running come free with
  `meshy_rig`; paying 3 credits for them is pure waste.
- `meshy_text_to_3d` alone yields an **untextured preview**. If the user wants a finished
  asset, budget the refine step into the quote up front rather than surprising them with a
  second charge.

## Where files land

`meshy_download_model` writes to **`{cwd}/meshy_output/`** — the current working directory,
which in an agent session is the user's project repo. Each download gets its own
`{timestamp}_{prompt-slug}_{task-id}/` folder plus thumbnails, a per-project
`metadata.json`, and a global `history.json` index.

Consequences worth acting on:

- **Warn before the first download in a git repo, and offer to add `meshy_output/` to
  `.gitignore`.** Generated meshes are large binaries; committing them by accident is
  hard to undo cleanly.
- Use the `save_to` argument (absolute path) when the asset belongs somewhere specific —
  a Unity `Assets/` folder, a Blender project directory.
- Pass `parent_task_id` when downloading a chained result (refine, rig, animate) so the
  files land in the same project folder as their ancestor instead of a fresh one.
- Ask which single format the user needs. Do **not** download every format "to be safe" —
  it is slow and clutters the repo, even though it costs no credits.

## Recipes

### Meshy → Blender (`agent-blender-wrapper`)

Meshy makes the asset; Blender edits and stages the scene. Meshy has no scene graph, no
lighting, no compositing — do not try to art-direct inside Meshy.

1. Generate with `target_formats: ["glb"]` (or `["blend"]` if only Blender will consume it).
2. `meshy_download_model` with `save_to` pointing at the Blender project directory.
3. Hand off to the blender skill and import via `execute_blender_code`.

If the user asks to "put the generated model into the scene", that second half is the
blender skill's job, not this one.

### Meshy → Unity (`agent-unity-wrapper`)

1. Generate with `target_formats: ["fbx"]`, `topology: "quad"`, and a `target_polycount`
   the target platform can carry.
2. Texture it (refine or `meshy_retexture`) *before* remeshing for topology.
3. Download with `save_to` inside the Unity project's `Assets/` tree so Unity imports it
   on next focus.

### Reference image → 3D (usually better than text → 3D)

The server's own guidance prefers this route, and it is cheaper to iterate on:

1. `meshy_text_to_image` to draft the design (3–9 credits) — cheap enough to iterate.
2. Show the user the image and iterate there until they approve.
3. `meshy_image_to_3d` on the approved image.

Iterating on 2D images costs a fraction of iterating on 3D generations. Reach for this
whenever the user's description is vague or stylistic.

### Recovering earlier work

"What happened to the dragon I generated yesterday?" → `meshy_list_tasks`
(`status: "SUCCEEDED"`, newest first) or `meshy_list_models` for the workspace view. Then
`meshy_download_model` with the recovered `task_id`. No regeneration, no credits.

## Pitfalls

- **`meshy_rig` needs ≤ 300,000 faces and a t-pose source.** Both are decided upstream: a
  model generated with `pose_mode: "a-pose"` rigs badly, and the fix is regenerating —
  another full charge. If rigging or animation is anywhere in the user's stated goal, set
  `pose_mode: "t-pose"` on the *first* generation.
- **`meshy_uv_unwrap` needs GLB with ≤ 40,000 faces.** Denser meshes are rejected with a
  400; remesh down first.
- **`meshy_process_multicolor` needs a textured input.** Running it on a preview mesh
  wastes 10 credits.
- **`3mf` is not downloadable unless it was produced.** Request it via `target_formats` at
  creation, or produce it with `meshy_convert` / `meshy_process_multicolor`.
- **`meshy_send_to_slicer` returns launch commands; it does not launch anything.** Running
  the returned command starts desktop software on the user's machine — surface the command
  and let the user confirm before executing it via Bash.
- **Never base64-encode images yourself.** Pass `file_path` / `file_paths` /
  `reference_file_paths` with absolute paths; the server reads and encodes them. Manual
  encoding blows up the context for no benefit.
- **`symmetry_mode` is deprecated** (API change 2026-05-11) and no longer affects output.
  Do not offer it as a lever.
- **Meshy is not a mesh editor.** "Make the handle 2 cm shorter", "remove that vertex",
  "merge these two objects" are Blender jobs. Meshy regenerates; it does not edit.

## Troubleshooting

**Every tool fails with an auth error.** The API key is missing or wrong. On Claude Code it
comes from the plugin's `userConfig` prompt at enable time — re-run
`/plugin config agent-meshy-wrapper` (or reinstall) to set it. On Codex it comes from the
`MESHY_API_KEY` environment variable of the process that launched the MCP. Keys start with
`msy_`. Note that API access requires a paid Meshy plan — a free account authenticates in
the web app but not against the API.

**Tools fail with an insufficient-credits error.** `meshy_check_balance` confirms it. The
user tops up at meshy.ai; there is nothing to retry locally.

**"Server not loaded" right after install.** Cold `npx` start: the package downloads before
the first JSON-RPC byte, which can exceed the host's handshake timeout. Reconnect the MCP
server (`/mcp` → reconnect); it starts fast once npm has cached it. To warm the cache
ahead of time: `npx -y @meshy-ai/meshy-mcp-server@0.4.0 --help`. Do not otherwise run the
server by hand — the host owns that process.

**A response looks truncated.** The server caps responses at `CHARACTER_LIMIT` characters
(default 25,000). Narrow the query — `limit`, a specific `task_type`, or
`response_format: "json"` — rather than re-requesting the same broad listing.
