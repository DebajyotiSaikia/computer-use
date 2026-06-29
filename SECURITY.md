# Security Policy

Orion controls a real mouse, keyboard, and screen and can run shell
commands, so it is designed defensively. This document describes the security
model and how to report issues.

Project site: <https://computer.deb0.com>

## Supported versions

| Version | Supported |
| ------- | --------- |
| 0.1.x   | ✅        |

## Reporting a vulnerability

Please report security issues **privately** — do not open a public issue.

- Preferred: GitHub private vulnerability reporting on the
  [`computer-use`](https://github.com/DebajyotiSaikia/computer-use) repository
  (Security → _Report a vulnerability_).
- Alternatively, reach the maintainer through <https://computer.deb0.com>.

Please include reproduction steps, affected platform, and version. You will get
an acknowledgement and a remediation timeline. Coordinated disclosure is
appreciated.

## Security model

### Network egress

- Outbound calls are limited to the Anthropic, OpenAI, and GitHub Copilot HTTPS
  endpoints. TLS verification is always on and no custom CA is configured.
- The local HTTP server and WebSocket hub bind exclusively to `127.0.0.1`.

### Action gating (whitelist)

- Every action is evaluated by the whitelist engine **before** it executes.
- GUI actions (mouse, keyboard, screenshot, wait) are allowed by default; shell
  commands and application launches default to **ask**.
- Non-whitelisted actions pause and wait for explicit user approval, with a
  configurable timeout that defaults to **deny**.

### Shell command execution

- Commands run via `os/exec` with an explicit argv — never through a shell
  interpreter (`sh -c`), preventing shell metacharacter injection.
- The full command and its output are written to the audit log.
- Each command has a 30-second timeout.

### Secrets at rest

- API keys and OAuth tokens are encrypted with **AES-256-GCM**. The key is
  derived from the machine identity plus a per-install random salt.
- The config endpoint returns secrets redacted; the UI never receives raw keys.

### Auditing and screenshots

- Screenshots, decisions, actions, whitelist checks, and LLM calls are written
  to an append-only NDJSON audit log **before** execution.
- Audit entries redact anything resembling an API key or token.
- Only 25%-scale screenshot thumbnails are persisted; full-resolution captures
  exist only in memory during a loop iteration.

### Prompt injection (XPIA)

- Screenshots can contain attacker-controlled content. Recording replay matches
  state by perceptual hash and only consults the LLM at genuine branch points,
  which minimizes exposure on known paths.
- The MCP server exposes only the fixed computer-use tool set; it does not load
  external tool servers.

## Hardening recommendations

- Run Orion as a non-privileged user.
- Keep the default whitelist timeout behaviour (`deny`).
- Pre-authorize only the specific shell commands you need, e.g.
  `orion whitelist add command "git *" --effect allow`.
- Review the audit log (`orion audit --tail`) during unattended runs.
