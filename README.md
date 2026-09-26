# agent-meshy-wrapper

A Claude Code **skill** plugin. Pairs the official [Meshy MCP server](https://github.com/meshy-dev/meshy-mcp-server) with a skill so Claude can generate, refine, rig, animate and export 3D assets from text or images — tracking async jobs and credit cost — instead of hand-driving the Meshy API.

The MCP server is declared inline in both plugin manifests. When the plugin is installed, the host agent launches it automatically via `npx` — no manual server setup. The server is pinned to an exact version (`@meshy-ai/meshy-mcp-server@0.4.0`) so `npx` resolves the same cached package on every start.

## Prerequisites

### 1. Node.js 18+

`npx` (bundled with npm) launches the server on demand without a global install.

### 2. A Meshy account with API access

The API key is the plugin's only configuration, and every generation spends credits from the account behind it.

1. Sign up at https://www.meshy.ai — **API access requires a paid plan**. A free account works in the web app but is rejected by the API.
2. Create a key at https://www.meshy.ai/api-keys. Keys start with `msy_`.

## Install

```
/plugin marketplace add seretos-agents/modular-software-factory
/plugin install agent-meshy-wrapper@modular-software-factory
```

### Claude Code — the key is prompted for you

The manifest declares the key as a [`userConfig`](https://code.claude.com/docs/en/plugins-reference#user-configuration) option, so Claude Code asks for it when the plugin is enabled and injects it into the server's environment. Nothing to hand-edit.

The value is marked `sensitive`, so it is stored in the OS keychain (or `~/.claude/.credentials.json` where no keychain is available) — **not** in `settings.json`, and never in the repository.

To change or re-enter it later:

```
/plugin config agent-meshy-wrapper
```

Or non-interactively at install time:

```
claude plugin install agent-meshy-wrapper@agent-marketplace --config api_key=msy_...
```

### Codex — set the key in the environment

Codex has no `userConfig` equivalent, so the Codex manifest ships **no** `env` block and the server inherits `MESHY_API_KEY` from the process that launches it. Export it in the shell or profile that starts Codex:

```bash
export MESHY_API_KEY=msy_...
```

```powershell
$env:MESHY_API_KEY = "msy_..."
```

## Credits

Nearly every Meshy tool debits your account balance — roughly 1 credit for a format conversion, 5–20 for a text-to-3D generation, 36 for a full Creative Lab product run. The server publishes the price table to the agent as MCP instructions and requires cost confirmation before spending; the skill reinforces that and adds a balance check before multi-step plans.

`meshy_check_balance` is free and answers "how much is left".

## Downloaded files

`meshy_download_model` saves into **`meshy_output/`** under the current working directory, one timestamped folder per project plus thumbnails and a `history.json` index. In a code repository that means generated meshes land in your worktree — add `meshy_output/` to that project's `.gitignore`, or pass an absolute `save_to` path to place the asset where it belongs.

## What the skill teaches

See `skills/meshy-wrapper/SKILL.md`. The server itself injects a workflow guide (cost table plus scenario playbooks for printing, game engines, character animation, AR, retexture, conversion, UV unwrap and Creative Lab), so the skill deliberately does not repeat it. It covers the layer around it: credit governance, task/`task_id` lifecycle and recovery across sessions, where downloads land, handoff recipes into `agent-blender-wrapper` and `agent-unity-wrapper`, and the constraints that are frozen at task-creation time.

## Troubleshooting

**Every tool errors with an auth failure.** The key is missing, wrong, or belongs to a free account. On Claude Code re-run `/plugin config agent-meshy-wrapper`; on Codex check `MESHY_API_KEY` in the launching environment.

**Insufficient credits.** `meshy_check_balance` confirms the balance; top up at meshy.ai. Nothing to retry locally.

**"Server not loaded" right after install.** Cold `npx` start: the package downloads before the first JSON-RPC byte, which can exceed the host's handshake timeout. Reconnect the MCP server (`/mcp` → reconnect) — it starts cleanly once npm has cached it. To warm the cache ahead of time:

```
npx -y @meshy-ai/meshy-mcp-server@0.4.0 --help
```

Do not otherwise run the server manually — the host owns that process.

**A task returns TIMEOUT.** `meshy_get_task_status` caps its wait at 300 seconds. The task is still running server-side; call the tool again with the same `task_id`. Progress sitting at 95 % is normal finalization — cancelling there forfeits credits already spent.
