# Architecture

Orion is a single Go binary (zero external dependencies) that drives a
real desktop through OS primitives, decides actions with an LLM, gates every
action through a whitelist, and audits everything. This document maps the
subsystems and how they fit together.

Project site: <https://computer.deb0.com> · Source:
<https://github.com/DebajyotiSaikia/computer-use>

## The agent loop

Each iteration mirrors how a person works at a computer:

```
screenshot → LLM decision → whitelist → execute (human-like input) → audit
```

1. **Capture** — grab the screen via OS calls and encode it to PNG.
2. **Reason** — send the screenshot plus conversation history to the provider;
   receive structured JSON actions.
3. **Clear** — evaluate each action against the whitelist (allow / deny / ask).
4. **Act** — execute with Bézier mouse paths and randomized keystroke timing.
5. **Audit** — write the screenshot thumbnail, decision, and result to NDJSON
   before the next step.

The loop pauses on errors or on `ask` decisions and resumes on user input. It
ends on `task_complete`, `task_fail`, cancellation, or an iteration cap.

## Package layout

```
main.go              entry point; embeds static/ assets
cmd/                 CLI (start, record, replay, auth, audit, config, whitelist, mcp)
internal/
  types/             shared Action/Decision types (breaks import cycles)
  util/              platform paths, image/pHash, manual RFC 6455 WebSocket, UUID
  gui/               GUI interface + per-platform syscalls + human input
  provider/          Anthropic, OpenAI, GitHub Copilot (device-flow OAuth)
  audit/             append-only NDJSON logger with secret redaction
  whitelist/         glob-rule engine, pause/timeout, session overrides
  agent/             the loop, system prompt, action execution
  config/            JSON config, AES-256-GCM at-rest key encryption
  session/           session lifecycle + session-scoped memory
  memory/            user-level persistent memory (confirm-to-write)
  recording/         semantic action graph, recorder, replayer (pHash matching)
  mcp/               MCP server (JSON-RPC 2.0 over stdio) exposing OS tools
  server/            loopback HTTP API + WebSocket hub + REST handlers
static/              embedded dark-theme web UI
tools/icongen/       brand icon generator (dev tool, not in the product binary)
```

## Dependency graph

`internal/types` sits at the bottom (Action, ActionType, Decision) so the
provider, agent, whitelist, audit, and recording packages can all reference a
single canonical `Action` without forming an import cycle.

```
types  ← util ← gui, provider, whitelist, audit, config, memory
session ← provider, whitelist, types
agent   ← gui, provider, whitelist, audit, session, memory, config, types, util
recording ← gui, types, provider, util, whitelist
mcp     ← gui, whitelist, audit, types, util
server  ← (everything)        cmd ← (everything) + server      main ← cmd
```

The whitelist engine talks to the UI through a callback hook, so it never
imports the server.

## GUI backends (build-tagged)

Exactly one platform file compiles per target:

| Platform | File             | Mechanism                                                   |
| -------- | ---------------- | ----------------------------------------------------------- |
| Windows  | `gui_windows.go` | Pure `syscall` (BitBlt, `mouse_event`, `SendInput`)         |
| macOS    | `gui_darwin.go`  | cgo / CoreGraphics, AppKit, ApplicationServices             |
| Linux    | `gui_linux.go`   | cgo / X11 + XTest, with a Wayland `ydotool`/`grim` fallback |

Human-like input (`human.go`, shared) moves the mouse along randomized cubic
Bézier curves with Gaussian jitter and a small overshoot, and types with
normally-distributed inter-key delays (40–220 ms).

## Recording and replay

Record-and-replay with drift detection is the core differentiator — a **semantic
action graph**, not a coordinate macro. See **[Record & replay](recording.md)**
for the full data model, matching thresholds, branching, parameters, and a
worked example.

- **Recorder** decorates the GUI layer; every executed action is captured as a
  semantic step (role, label, context, fallback coordinates) plus a perceptual
  hash of the screen before and after.
- **Replayer** matches live screen state to the recording by average-hash
  similarity. High similarity replays deterministically; genuine mismatches fall
  back to the LLM, constrained to the recording's scope. Drift across a run is
  detected and surfaced to the UI.

## World model (observation sources)

Orion uses **progressive disclosure**: cheap context is always present, richer
sources are pulled only on demand. Every turn includes the screenshot, a pruned
accessibility tree, and lightweight metadata (active window, cursor, modifiers).
Deeper sources are read-only tools the model invokes when the view is ambiguous.

Sources are grouped by capability and tagged with cost so the agent prefers the
cheapest source that answers a question and escalates only when needed:

| Capability    | Sources                                                                | Cost                |
| ------------- | ---------------------------------------------------------------------- | ------------------- |
| Vision        | `screenshot`, `ocr`                                                    | medium              |
| Accessibility | `get_accessibility_tree` (pruned, actionable)                          | low                 |
| OS            | `get_window_info`, `get_input_state`, `list_processes`, `get_hardware` | very-low–medium     |
| Clipboard     | `read_clipboard`                                                       | very-low            |
| Filesystem    | `list_files`, `read_file`                                              | low                 |
| Browser/Net   | `get_dom`, `get_network_requests`, `get_console_logs`                  | unavailable (stubs) |

The catalog (`agent/sources.go`) carries availability so unavailable bridges
fail clearly. Structured accessibility is Windows-only today; macOS/Linux return
the window node. All sources are allowed by default and audited.

## Providers

A provider implements authentication, model listing, and completion. Requests
use only `net/http` with TLS verification and exponential-backoff retries.
Vision is sent as base64 PNG; actions are parsed from the model's text. See
[configuration](configuration.md) for credentials and [MCP](mcp.md) for the
tool-server mode.

## Local web UI

`net/http` serves the embedded `static/` assets on `127.0.0.1`. A hand-written
RFC 6455 WebSocket hub streams live events (screenshots, pending actions,
whitelist prompts, agent messages) to the browser. The UI is vanilla
HTML/CSS/JS — no frameworks, no CDN.
