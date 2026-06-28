# CLI Reference

The `perseus` binary is the single entry point. With no arguments it defaults to
`start`.

Project site: <https://computer.deb0.com>

```
perseus <command> [flags]
```

## start

Start the agent and the local web UI.

```
perseus start [--prompt "..." | --prompt-file path]
              [--provider anthropic|openai|copilot]
              [--model <id>] [--reasoning none|low|medium|high|max]
              [--context <tokens>] [--headless] [--no-ui]
              [--agent-dir <path>]
```

| Flag            | Description                                           |
| --------------- | ----------------------------------------------------- |
| `--prompt`      | Task to run immediately.                              |
| `--prompt-file` | Read the task from a file.                            |
| `--provider`    | Provider to use (defaults to the configured default). |
| `--model`       | Model id (e.g. `claude-sonnet-4-6`).                  |
| `--reasoning`   | Reasoning effort / thinking budget.                   |
| `--context`     | Context window size in tokens.                        |
| `--headless`    | Run the web server but do not open a browser.         |
| `--no-ui`       | Terminal-only; no web server (requires a prompt).     |
| `--agent-dir`   | Override the storage directory.                       |

## record

Start in recording mode; actions performed by a session are captured.

```
perseus record [--name "..."] [--description "..."] [--agent-dir <path>]
```

## replay

Replay a saved recording, substituting parameters.

```
perseus replay <recording-id> [--param key=value ...] [--agent-dir <path>]
```

## auth

Authenticate a provider. API-key providers prompt for the key; Copilot runs the
GitHub OAuth device flow on the terminal.

```
perseus auth <anthropic|openai|copilot> [--agent-dir <path>]
```

## audit

Query or follow the audit log.

```
perseus audit [--session <id>] [--tail] [--format pretty|json] [--agent-dir <path>]
```

| Flag        | Description                   |
| ----------- | ----------------------------- |
| `--session` | Filter by session id.         |
| `--tail`    | Follow new entries.           |
| `--format`  | `pretty` (default) or `json`. |

## config

Read or modify configuration. See [configuration.md](configuration.md).

```
perseus config                      # print the config (secrets redacted)
perseus config --get ui.port        # read a value by dotted key
perseus config --set ui.port=7600   # set a value
perseus config --edit               # open in $EDITOR
```

## whitelist

Manage whitelist rules.

```
perseus whitelist list
perseus whitelist add <command|window|url|action_type|app> <pattern> [--effect allow|deny|ask]
perseus whitelist remove <id>
```

Examples:

```sh
perseus whitelist add command "git *" --effect allow
perseus whitelist add action_type "shell_command" --effect deny
```

## mcp

Run as a Model Context Protocol server over stdio. See [mcp.md](mcp.md).

```
perseus mcp [--agent-dir <path>]
```

## version

```
perseus version
```
