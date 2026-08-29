# pi-interactive-subagents

Async subagents for [pi](https://github.com/badlogic/pi-mono), running in a terminal multiplexer pane. Spawn a sub-agent, keep working in the main session, and get the result steered back when it finishes. Fully non-blocking.

**Terminal multiplexer support.** Works with **tmux** (Linux/macOS) and **Psmux** (Windows / PowerShell). The tmux commands used by this extension are transparently compatible with Psmux — no platform-specific code is needed in the extension logic.

## How it works

`subagent()` returns immediately. The sub-agent runs in its own multiplexer pane — a right split off the parent pi pane, so pane creation never steals keyboard focus. A live widget above the input tracks every running sub-agent, and when one finishes, its result is steered into the main session as a notification that triggers a new turn.

```
╭─ Subagents ──────────────────────────── 2 running ─╮
│ 00:23  scout      active · bash 7m                 │
│ 00:45  scout-2    waiting 2m                       │
╰────────────────────────────────────────────────────╯
```

Spawn several in parallel — they run concurrently and steer results back independently as each finishes.

Panes are kept evenly sized: the extension re-applies an `even-horizontal` layout after every spawn and exit (debounced). The layout is a single constant, `SUBAGENT_TMUX_LAYOUT` in `pi-extension/subagents/tmux.ts` — change it to any named multiplexer layout (`main-vertical`, `tiled`, …).

If your shell startup is slow and launch commands get dropped before the prompt is ready, raise the delay:

```bash
export PI_SUBAGENT_SHELL_READY_DELAY_MS=2500   # default: 500
```

## Requirements

| Platform | Multiplexer | Shell | Notes |
|----------|-------------|-------|-------|
| **Linux** | tmux | bash | Standard setup |
| **macOS** | tmux | bash | Standard setup |
| **Windows** | [Psmux](https://github.com/dockyarded/psmux) | PowerShell 7+ (`pwsh`) preferred, PowerShell 5.x or bash (Git Bash) as fallback | Psmux is a tmux emulator for Windows — the `tmux` CLI commands work transparently |

### macOS — tmux permission prompt

If you see a macOS security prompt to allow **Terminal** access to **Full Disk Access**, click Allow. tmux needs this to manage panes.

```bash
tmux new -A -s pi 'pi'
```

### Windows — Psmux

On Windows, this extension runs inside [Psmux](https://github.com/dockyarded/psmux), a tmux-compatible multiplexer built on PowerShell. Psmux exposes a `tmux` shim so all `tmux` commands from the extension work identically to Linux/macOS — no special configuration needed.

The extension auto-detects the available shell in the Psmux pane:

1. **pwsh** (PowerShell 7+) — preferred, better `-File` behavior and `$LASTEXITCODE` handling
2. **powershell** / `powershell.exe` (Windows PowerShell 5.x) — fallback
3. **bash** (Git Bash / MSYS2) — last resort

Launch commands are written as `.ps1` scripts on Windows and `.sh` scripts on Unix. Bash syntax (`$?` exit codes, inline env vars, `echo`) is automatically converted to the PowerShell equivalent (`$LASTEXITCODE`, `$env:VAR=`, `Write-Output`) when targeting a PowerShell shell.

## Tools

| Tool | Description |
| --- | --- |
| `subagent` | Spawn a sub-agent in a dedicated multiplexer pane (async) |
| `subagent_message` | Message a sub-agent by name — steers it if running, resumes its session if finished |
| `subagents_list` | List available agent definitions |
| `ask_question` | *(sub-agent sessions only)* Ask the orchestrator a question and wait for the reply |

There is also a `/subagent <agent> <task>` command for spawning directly.

### Spawning

```typescript
subagent({ agent: "scout", task: "Analyze the auth module" });
subagent({ agent: "worker", name: "dark-mode", task: "Implement the dark mode toggle" });
```

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `agent` | string | required | Which agent to spawn (must be known and permitted) |
| `task` | string | required | Task prompt |
| `name` | string | agent name | Display name for the pane and widget. Must be unique — duplicates are auto-suffixed (`scout`, `scout-2`, …) |
| `model` | string | agent's model | Override the model for this spawn |
| `cwd` | string | agent's `cwd` | Working directory (see [Role folders](#role-folders)) |

### Messaging

`subagent_message` is addressed **by name only**. Names are unique per session and persist after a sub-agent finishes, so the same name works either way:

```typescript
subagent_message({ name: "scout", message: "Also check the auth middleware" });
```

- **Running** — the message is typed into the live pane (newlines flattened) and picked up at the next turn boundary. The call returns immediately; the eventual completion still arrives as a steer message.
- **Finished** — the session is resumed with the message as the follow-up task, like a fresh spawn: fire-and-forget, always autonomous, result steered back later. The resumed run reclaims its original name.

Every spawn records name → session file in `artifacts/<sessionId>/subagent-registry.json`, so names stay addressable across pi restarts. A nested sub-agent that spawns children gets its own registry keyed by its own session id. Resume is refused with a clear error (listing known names) if the name isn't registered, the session file is gone, or the session predates sandboxed resume.

**Resume replays the original sandbox.** At spawn time the fully-resolved loadout — tool allowlist, backing extensions, model, thinking level, system prompt, spawn whitelist, cwd — is snapshotted to `<session>.loadout.json`. Resume rebuilds the exact same restricted process from that snapshot rather than relaunching unrestricted.

### ask_question

A sub-agent can ask its orchestrator a single freeform question when requirements are ambiguous or a decision materially affects the work. The session **stays open** (parked as `waiting`) instead of exiting; the parent is notified with the sub-agent's name, replies via `subagent_message({ name, message })`, and the reply arrives as the sub-agent's next turn. Parallel questions are supported — each waiting sub-agent has its own name.

If the reply arrives while the sub-agent is still mid-turn, it is absorbed into the current turn — either way the question is marked answered and the session exits normally when the work is done. If the parent never replies, the pane stays open until a human closes it. Only available inside sub-agent sessions.

## Bundled agents

| Agent | Model | Tools | Role |
| ----- | ----- | ----- | ---- |
| **scout** | `openrouter/z-ai/glm-5.3` | `read`, `grep`, `find`, `ls` | Fast read-only codebase recon |
| **researcher** | `openrouter/z-ai/glm-5.3` | `web_search`, `web_fetch`, `safe_bash` | Web research, synthesized into a sourced brief |
| **worker** | `openrouter/z-ai/glm-5.3` | `read`, `write`, `edit`, `bash`, `web_search`, `web_fetch` + spawning | General implementer; may spawn `scout` and `researcher` |

All three are autonomous (`auto-exit: true`) and carry their identity in the system prompt (`system-prompt: append`).

## Custom agents

Place a `.md` file in `.pi/agents/` (project) or `~/.pi/agent/agents/` (global). Discovery priority: **project > global > package-bundled** — a project-local file overrides a bundled agent with the same name.

```markdown
---
name: my-agent
description: Does something specific
model: openrouter/z-ai/glm-5.3
thinking: medium
tools: read, edit, write, safe_bash, web_search
session-mode: lineage-only
auto-exit: true
---

You are a specialized agent that does X...
```

### Frontmatter reference

| Field | Type | Description |
| ----- | ---- | ----------- |
| `name` | string | Agent name (used in `agent: "my-agent"`) |
| `description` | string | Shown in `subagents_list` |
| `model` | string | Default model |
| `thinking` | string | `minimal`, `low`, `medium`, or `high` |
| `tools` | string | Strict tool allowlist. Built-ins: `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`. Extension-backed: `web_search`, `web_fetch`, `safe_bash`, `video_extract`, `youtube_search`, `google_image_search`. Only the extensions backing the listed tools are loaded into the child |
| `subagent_agents` | string | Comma-separated agent names this agent may spawn. **Presence of this field grants the spawning toolset** (`subagent`, `subagent_message`, `subagents_list`) and restricts spawn targets to the list. Omit it and the agent cannot spawn at all |
| `skills` | string | Comma-separated skill names to auto-load |
| `session-mode` | string | `standalone` (default), `lineage-only`, or `fork` — see below |
| `system-prompt` | string | `append` or `replace`: pass the body as the child's `--append-system-prompt` / `--system-prompt`. Omit and the body is prepended to the task prompt instead |
| `auto-exit` | boolean | Auto-shutdown when the agent finishes (see below) |
| `interactive` | boolean | Whether stall/recovery transitions wake the parent (see below) |
| `cwd` | string | Default working directory |
| `disable-model-invocation` | boolean | Hide from `subagents_list`; still spawnable by explicit name |
| `cli` | string | `claude` runs the agent via the Claude Code CLI instead of pi |

### session-mode

- `standalone` — fresh session, no lineage link to the caller (default)
- `lineage-only` — fresh session with `parentSession` linkage for discovery/fork UX, but no copied turns
- `fork` — child session seeded with the caller's conversation context

### auto-exit

With `auto-exit: true`, the session shuts down when the agent's turn ends — the agent just writes its final message and stops (there is no "done" tool). The last assistant message becomes the summary returned to the parent. Recommended for all autonomous agents.

Notes:

- **Manual input does not strand an auto-exit sub-agent.** If a human types into the pane, the session still closes once that turn completes normally — only an escape/abort leaves it open.
- **Auto-exit is suppressed while work is in flight:** the session parks as `waiting` instead of exiting when an `ask_question` is still unanswered, or when the agent's own child sub-agents are still running (a worker can stop after dispatching children and stays open until the last result returns).

### interactive

Controls whether `stalled`/`recovered` status transitions send a steer message to the parent session. Defaults to the inverse of `auto-exit`: autonomous agents get stall pings; user-driven agents stay quiet (the user is already working in that pane — the widget still updates). Set explicitly to override.

## Tool access control

Access is **whitelist-only**. Every sub-agent process is launched with `--no-extensions` (extension discovery disabled) and `--tools <allowlist>`; only the extensions backing the listed tools are loaded back in explicitly. There is no default toolset and no deny-list — an agent gets exactly what its frontmatter lists. The restriction survives resume via the loadout snapshot.

Spawns must name a known agent at **every** depth. A top-level session may spawn anything discoverable; a sub-agent may only spawn the agents in its `subagent_agents` list (enforced via `PI_SUBAGENT_ALLOWED`). There is no agentless spawn route, so a child can never escalate to a full-toolset profile by omitting its agent.

Extensions can register additional tools for sub-agents at runtime via `registerToolExtension(name, path)` on the `__pi_interactive_subagents` process global.

### Package extensions

Agents can declare external pi package extensions in their frontmatter using the `packages:` field:

```yaml
---
name: researcher
tools: safe_bash
packages: npm:@narumitw/pi-firecrawl, git:github.com/MasuRii/pi-rtk-optimizer
---
```

Each comma-separated entry is passed as a separate `-e` flag to the child `pi` process, loading the extension even though `--no-extensions` disables global discovery.

**Important:** If a package extension uses lazy loading (tools register only when first invoked), you must also list those tools in the `tools:` field so the agent can discover them:

```yaml
---
name: researcher
tools: safe_bash, firecrawl_search, firecrawl_scrape
packages: npm:@narumitw/pi-firecrawl
---
```

Without listing the tools in `tools:`, the agent's tool schema won't include them even though the extension is loaded.

## Role folders

`cwd` starts a sub-agent in a directory with its own config, so role-specific setups (CLAUDE.md, skills, extensions) apply:

```
project/
└── agents/
    ├── game-designer/   ← CLAUDE.md, .pi/…
    └── sre/             ← CLAUDE.md, .pi/…
```

```typescript
subagent({ agent: "worker", cwd: "agents/sre", task: "Review the deployment pipeline" });
```

Set a per-agent default with `cwd:` in frontmatter.

## Status widget & configuration

The widget tracks each sub-agent from a runtime activity snapshot written by the child: `starting`, `active` (turn/provider/tool work), `waiting` (open for input or another stage), `stalled` (no valid snapshot for too long), or `running` (fallback). Sub-agent sessions also show their own tools widget — toggle it with `Ctrl+Alt+O`. Completion messages expand with `Ctrl+O`.

Status display is configured via `config.json` in the extension directory (copy `config.json.example`; it's gitignored):

```json
{
  "status": { "enabled": true }
}
```

## Platform notes

### Windows (Psmux)

- The extension auto-detects the best available shell in the Psmux pane: **pwsh** (PowerShell 7+, preferred) → **powershell** (5.x) → **bash** (Git Bash/MSYS2).
- Launch commands are written as `.ps1` scripts on Windows (`.sh` on Unix). Bash syntax is automatically converted to PowerShell:
  - Inline env vars: `VAR=value` → `$env:VAR=value; `
  - Exit code sentinel: `$?` → `$LASTEXITCODE`
  - `echo` → `Write-Output`
  - `&&` chaining → `;` (PowerShell 5 compatible)
- The `tmux` CLI commands used by the extension are transparently handled by Psmux's `tmux` shim — no special configuration or flags needed.
- Claude Code `on-stop.sh` hooks are skipped on Windows (they require Bash/Python).
- Script files use `-NoProfile -NoLogo` on PowerShell to avoid slow profile loading and keep screen captures clean.
- Shell ready delay (`PI_SUBAGENT_SHELL_READY_DELAY_MS`) applies equally to PowerShell pane startup.

### macOS

- tmux may trigger a Full Disk Access prompt. Grant it to pane management.
- The `tmux` command must be installed (via Homebrew: `brew install tmux`).

## Acknowledgements

Forked from [HazAT/pi-interactive-subagents](https://github.com/HazAT/pi-interactive-subagents), which originated the subagent architecture, the multi-multiplexer surface layer, and the status widget; its supervision features were inspired by [RepoPrompt](https://repoprompt.com/). Additional modifications based on [amosblomqvist/pi-interactive-subagents](https://github.com/amosblomqvist/pi-interactive-subagents).

## License

MIT
