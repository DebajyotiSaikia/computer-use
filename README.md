# Perseus Pisces

**Cross-platform desktop AI automation agent. Records, replays, and executes tasks like a human.**

Perseus Pisces drives a real computer the way a person does: it captures the
screen, asks an LLM what to do next, and performs human-like mouse and keyboard
input — all gated by an auditable whitelist engine and driven from an embedded
local web UI.

- **Website:** <https://computer.deb0.com>
- **Download:** [latest release](https://github.com/DebajyotiSaikia/computer-use/releases/latest)
- **Binary:** `perseus` — a single static binary with **zero external dependencies**

> This repository hosts the **documentation** and the **installers/binaries**
> (published via [Releases](https://github.com/DebajyotiSaikia/computer-use/releases))
> for Windows, macOS, and Linux. Full docs also live at
> [computer.deb0.com](https://computer.deb0.com).

---

## Download & install

Grab the build for your platform from the
[latest release](https://github.com/DebajyotiSaikia/computer-use/releases/latest):

| Platform              | Asset                       |
| --------------------- | --------------------------- |
| Windows (x64)         | `perseus-windows-amd64.exe` |
| macOS (Apple Silicon) | `perseus-darwin-arm64`      |
| macOS (Intel)         | `perseus-darwin-amd64`      |
| Linux (x64)           | `perseus-linux-amd64`       |

**Windows** (PowerShell):

```powershell
.\perseus-windows-amd64.exe start
```

**macOS / Linux:**

```sh
chmod +x ./perseus-darwin-arm64      # or the asset you downloaded
./perseus-darwin-arm64 start
```

Starting opens the local control UI at <http://127.0.0.1:7523>.

### Platform notes

- **macOS** — downloaded binaries are quarantined by Gatekeeper; clear it with
  `xattr -d com.apple.quarantine ./perseus-darwin-arm64`. Grant the app
  **Screen Recording** and **Accessibility** permissions (System Settings →
  Privacy & Security) so it can capture the screen and control input.
- **Linux** — requires X11 (`libX11`, `libXtst`) at runtime; under Wayland it
  falls back to `ydotool` (input) and `grim` (screenshots).
- **Windows** — no extra dependencies.

---

## What it is

- **Human-like input** — mouse moves along randomized Bézier curves with jitter
  and overshoot; keystrokes carry natural 40–220 ms delays.
- **Whitelist + audit** — every action clears a glob-rule whitelist and is
  written to an append-only NDJSON log before it executes.
- **Record & replay** — capture tasks as a [semantic action graph](docs/recording.md)
  and replay them with perceptual-hash state matching and drift detection.
- **Multi-provider** — Anthropic, OpenAI, and GitHub Copilot, all with vision.
- **MCP server** — expose the desktop as Model Context Protocol tools over stdio.
- **Local web UI** — a dark control room embedded in the binary; no CDN, no
  frameworks.
- **Zero dependencies** — GUI primitives are OS syscalls; no containers, no
  package managers.

---

## Documentation

- [Architecture](docs/architecture.md) — subsystems, the agent loop, and the dependency graph
- [Record & replay](docs/recording.md) — the semantic action graph, state matching, and drift detection
- [CLI reference](docs/cli.md) — every command and flag
- [Configuration](docs/configuration.md) — config schema and at-rest key encryption
- [MCP server](docs/mcp.md) — expose the desktop as Model Context Protocol tools
- [Security policy](SECURITY.md) — threat model and how to report a vulnerability

---

## Quick usage

```sh
# Start the agent + local web UI
perseus start

# Authenticate a provider
perseus auth anthropic        # prompts for an API key
perseus auth copilot          # GitHub OAuth device flow

# Run a task headlessly
perseus start --no-ui --prompt "Open Notepad and type a haiku"

# Expose the desktop to any MCP client (e.g. Claude Desktop)
perseus mcp

# Inspect the audit log
perseus audit --tail
```

See the [CLI reference](docs/cli.md) for all commands.

---

## Providers

| Provider       | Auth                    | Vision | Reasoning                 |
| -------------- | ----------------------- | ------ | ------------------------- |
| Anthropic      | `x-api-key`             | yes    | extended thinking budgets |
| OpenAI         | `Authorization: Bearer` | yes    | `reasoning_effort`        |
| GitHub Copilot | OAuth 2.0 device flow   | yes    | —                         |

Credentials are encrypted at rest with AES-256-GCM (see
[configuration](docs/configuration.md)).

---

## Security

Perseus controls real input and can run shell commands, so it is built
defensively: loopback-only server, default-deny whitelist for privileged
actions, argv-only shell execution, redacted audit logs, and encrypted secrets.
See [SECURITY.md](SECURITY.md) for the full policy and how to report issues.
