# agent-meshy-wrapper

Turns Meshy AI into a 3D asset pipeline the agent can actually run. Describe a
model or hand over a photo, and Claude generates it, textures it, rigs it,
converts it to the format your engine or slicer wants, and drops the file where
you need it — while keeping you in control of what it spends.

## Key features

- **Zero-setup MCP server** — the official Meshy MCP server is declared inline
  in the plugin manifest and launched via `npx` on install. No manual server, no
  global install, no config file.
- **API key prompted at install** — the key is a `userConfig` option, so Claude
  Code asks for it when you enable the plugin and stores it in your OS keychain.
  Nothing to hand-edit, nothing that can end up in a repository.
- **Text or image in, finished asset out** — text-to-3D, image-to-3D and
  multi-image-to-3D, plus AI texturing, retexturing and remeshing.
- **Characters that move** — auto-rigging (walk and run included) and custom
  animations, with the t-pose and polycount constraints handled up front instead
  of after you have paid for the wrong mesh.
- **Ready for its destination** — FBX for Unity or Unreal, USDZ for AR, GLB for
  the web, OBJ or 3MF for the printer, including multi-color 3MF for AMS/MMU
  printers and a free printability check before you slice.
- **Spends deliberately** — the skill quotes the full credit cost of a plan
  before the first paid call, checks your balance, prefers the 1-credit
  conversion over the 5-credit remesh, and iterates on cheap 2D drafts instead
  of expensive 3D re-generations.
- **Plays with the rest of your toolchain** — hands generated assets off to
  `agent-blender-wrapper` for scene work and drops game-ready meshes straight
  into a Unity project's `Assets/` folder.
- **Works in Claude Code and Codex** — ships both a `.claude-plugin` and a
  `.codex-plugin` manifest from one repository.

## Requirements

- **Node.js 18+** on the host machine; `npx` launches the server on demand.
- A **Meshy account on a paid plan** — API access is not available on the free
  tier. Create a key at https://www.meshy.ai/api-keys (keys start with `msy_`).
- Credits in that account. Generations, texturing, rigging and printing
  post-processing all debit it.
- Codex users set `MESHY_API_KEY` in the environment; Claude Code prompts for it.

## A good fit if you want to

- Get a usable 3D asset out of a sentence or a photo without opening a modelling
  tool.
- Produce a print-ready or game-ready file in the right format on the first try.
- Rig and animate a character conversationally.
- Keep an eye on what each generation costs before it happens.
