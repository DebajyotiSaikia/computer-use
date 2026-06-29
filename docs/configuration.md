# Configuration

Configuration is a single JSON document. API keys and tokens are encrypted at
rest; everything else is plain JSON you can edit by hand or via the CLI/UI.

Project site: <https://computer.deb0.com>

## Location

| Platform | Path                                   |
| -------- | -------------------------------------- |
| Windows  | `C:\orion\config\config.json`  |
| macOS    | `~/.orion/config/config.json` |
| Linux    | `~/.orion/config/config.json` |

Override the whole agent directory with `--agent-dir` or the
`ORION_AGENT_DIR` environment variable. The directory also holds
`sessions/`, `recordings/`, `audit/`, and `memory/`.

## At-rest encryption

`providers.*.api_key`, `providers.copilot.access_token`, and the refresh token
are encrypted with **AES-256-GCM**. The key is derived from the machine identity
(Windows MachineGuid / macOS IOPlatformUUID / Linux `/etc/machine-id`) plus a
random salt stored in `config/.salt`. Encrypted values are written with an
`enc:` prefix; the API and `config --get` return them masked.

## Editing

```sh
orion config                      # print (secrets redacted)
orion config --get whitelist.timeout_behavior
orion config --set ui.port=7600
orion config --edit               # open in $EDITOR
```

The web UI's **Config** view edits the same document; masked secrets are
preserved on save.

## Schema

```jsonc
{
  "version": "0.1.0",
  "agent": {
    "loop_delay_min_ms": 300, // min pause between iterations
    "loop_delay_max_ms": 800, // max pause between iterations
    "max_actions_per_loop": 5, // cap actions executed per turn
    "screenshot_scale": 1.0, // 0–1 downscale sent to the model
    "max_conversation_turns": 50, // history kept for context
    "default_prompt_file": "",
    "agent_dir": "", // optional storage override
  },
  "whitelist": {
    "user_response_timeout_seconds": 30,
    "timeout_behavior": "deny", // "deny" | "allow"
    "notify_on_allow": true,
    "notify_on_deny": true,
  },
  "providers": {
    "default": "anthropic",
    "anthropic": { "api_key": "", "enabled": true },
    "openai": { "api_key": "", "enabled": true },
    "copilot": {
      "access_token": "",
      "refresh_token": "",
      "token_expiry": "",
      "enabled": true,
    },
  },
  "ui": {
    "port": 7523,
    "auto_open": true,
    "theme": "dark", // "dark" | "light"
    "screenshot_fps": 2,
  },
  "recording": {
    "drift_threshold": 0.75,
    "state_similarity_warning": 0.85,
    "auto_validate_interval_h": 24,
  },
  "audit": {
    "include_screenshots": true,
    "screenshot_thumbnail_pct": 25,
    "max_audit_file_size_mb": 500,
  },
}
```

Missing fields fall back to these defaults, and a corrupt file is replaced with
a clean default config on load.

## Credentials

Set provider credentials with `orion auth <provider>` (recommended) or through
the UI's provider wizard. Copilot uses the GitHub OAuth device flow; the
long-lived token is stored encrypted and the short-lived Copilot token is held
in memory and refreshed automatically.
