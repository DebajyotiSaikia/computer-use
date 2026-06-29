# CLI Reference

The `orion` binary is the single entry point. With no arguments it defaults to
`start`.

Project site: <https://computer.deb0.com>

```
orion <command> [flags]
```

## start

Start the agent and the local web UI.

```
orion start [--prompt "..." | --prompt-file path]
              [--provider anthropic|openai|copilot]
              [--model <id>] [--reasoning none|low|medium|high|max]
              [--context <tokens>] [--headless] [--no-ui]
              [--unattended | --approve-all] [--agent-dir <path>]
```

| Flag            | Description                                                     |
| --------------- | --------------------------------------------------------------- |
| `--prompt`      | Task to run immediately.                                        |
| `--prompt-file` | Read the task from a file.                                      |
| `--provider`    | Provider to use (defaults to the configured default).           |
| `--model`       | Model id (e.g. `claude-sonnet-4-6`).                            |
| `--reasoning`   | Reasoning effort / thinking budget.                             |
| `--context`     | Context window size in tokens.                                  |
| `--headless`    | Run the web server but do not open a browser.                   |
| `--no-ui`       | Terminal-only; no web server (requires a prompt).               |
| `--unattended`  | Auto-deny actions needing approval (no prompts).                |
| `--approve-all` | Auto-approve actions needing approval (deny rules still apply). |
| `--agent-dir`   | Override the storage directory.                                 |

### Approval modes

Actions not already settled by an allow/deny rule (mainly `shell_command` and
`open_application`) need a decision. How that decision is made depends on the
launch mode:

| Mode                    | Responder                                                        |
| ----------------------- | ---------------------------------------------------------------- |
| `orion start` (default) | confirm dialog in the **web UI**                                 |
| `orion start --no-ui`   | **native OS dialog** (zenity/osascript/MessageBox), with timeout |
| `--unattended`          | auto-deny immediately, no prompt                                 |
| `--approve-all`         | auto-approve (explicit `deny` rules still win)                   |

`--unattended` and `--approve-all` are mutually exclusive. When no responder is
available (e.g. headless with no display), the action is denied rather than
blocking. Everything is audited regardless of mode.

## record

Start in recording mode; actions performed by a session are captured.

```
orion record [--name "..."] [--description "..."] [--agent-dir <path>]
```

## replay

Replay a saved recording, substituting parameters.

```
orion replay <recording-id> [--param key=value ...] [--agent-dir <path>]
```

## auth

Authenticate a provider. API-key providers prompt for the key; Copilot runs the
GitHub OAuth device flow on the terminal.

```
orion auth <anthropic|openai|copilot> [--agent-dir <path>]
```

## audit

Query or follow the audit log.

```
orion audit [--session <id>] [--tail] [--format pretty|json] [--agent-dir <path>]
```

| Flag        | Description                   |
| ----------- | ----------------------------- |
| `--session` | Filter by session id.         |
| `--tail`    | Follow new entries.           |
| `--format`  | `pretty` (default) or `json`. |

## config

Read or modify configuration. See [configuration.md](configuration.md).

```
orion config                      # print the config (secrets redacted)
orion config --get ui.port        # read a value by dotted key
orion config --set ui.port=7600   # set a value
orion config --edit               # open in $EDITOR
```

## whitelist

Manage whitelist rules.

```
orion whitelist list
orion whitelist add <command|window|url|action_type|app> <pattern> [--effect allow|deny|ask]
orion whitelist remove <id>
```

Examples:

```sh
orion whitelist add command "git *" --effect allow
orion whitelist add action_type "shell_command" --effect deny
```

## mcp

Run as a Model Context Protocol server over stdio. See [mcp.md](mcp.md).

```
orion mcp [--agent-dir <path>]
```

## version

```
orion version
```
