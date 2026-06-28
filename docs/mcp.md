# MCP Server

Perseus Pisces can run as a [Model Context Protocol](https://modelcontextprotocol.io)
server, exposing its computer-use primitives as MCP **tools** over the stdio
transport. It is JSON-RPC 2.0 implemented with the Go standard library only — no
SDK, no dependencies. Any MCP client (Claude Desktop, IDEs, custom agents) can
then see the real screen and drive the mouse and keyboard, with every call still
passing the same whitelist and audit engine.

Project site: <https://computer.deb0.com>

## Running

```sh
perseus mcp            # serve over stdio (diagnostics go to stderr)
```

stdout carries only protocol messages; logs go to stderr.

## Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "perseus": {
      "command": "perseus",
      "args": ["mcp"]
    }
  }
}
```

## Tools

| Tool                  | Arguments                | Result           |
| --------------------- | ------------------------ | ---------------- |
| `screenshot`          | —                        | PNG image        |
| `get_screen_size`     | —                        | `{width,height}` |
| `get_cursor_position` | —                        | `{x,y}`          |
| `get_active_window`   | —                        | title + bounds   |
| `mouse_move`          | `x, y`                   | ok               |
| `mouse_click`         | `x, y, button?`          | ok               |
| `mouse_double_click`  | `x, y`                   | ok               |
| `mouse_scroll`        | `x, y, delta_x, delta_y` | ok               |
| `key_press`           | `keys: string[]`         | ok               |
| `type_string`         | `text`                   | ok               |
| `wait`                | `ms`                     | ok               |
| `shell_command`       | `command`                | command output   |
| `open_application`    | `app_name`               | ok               |

Coordinates are screen pixels with the origin at the top-left. Call
`screenshot` first to see the screen, then act.

## Whitelist gating

GUI tools (mouse, keyboard, screenshot) are allowed by default. `shell_command`
and `open_application` default to **ask** — with no Perseus UI connected to
approve them, they time out and are denied. Pre-authorize what you need:

```sh
perseus whitelist add command "git *" --effect allow
```

## Protocol notes

- Lifecycle: `initialize` → `notifications/initialized` → `tools/list` /
  `tools/call`; `ping` is supported.
- Protocol version is negotiated; `2024-11-05`, `2025-03-26`, and `2025-06-18`
  are accepted.
- MCP is a tools layer, not an LLM transport: the host's model still calls its
  provider directly and invokes these tools via native tool-calling.
