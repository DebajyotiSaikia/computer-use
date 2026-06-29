# Record & replay: the semantic action graph

Most "computer use" tools replay automations as pixel macros: fixed coordinates
fired blindly at the screen. They break the moment a window moves, the theme
changes, or the resolution differs. Orion records an automation as a
**semantic action graph** instead — a structured, self-describing recording that
matches _what the screen means_ rather than _where the pixels were_, and that
falls back to the LLM only at genuine branch points.

This is the core of the project. This document describes the data model, how a
recording is captured, how it is replayed and matched, how drift is detected,
and how replay stays safe.

Implementation: [`internal/recording`](https://github.com/DebajyotiSaikia/computer-use)
— `graph.go` (model + persistence), `semantic.go` (element targeting),
`recorder.go` (capture), `replayer.go` (matching, branching, drift).

## Why a graph, not a macro

Each recorded step carries three independent ways to re-find its target, in
descending order of robustness:

1. **Perceptual screen state** — a 64-bit average hash (aHash) of the screen
   _before_ the action, so replay can confirm it is on the expected screen.
2. **Semantic target** — what the action was aimed at (role, label, context,
   ordinal), derived from lightweight pixel heuristics — no OCR, no DOM.
3. **Fallback coordinates** — the original `(x, y)`, used only as a last resort.

Because steps also carry `branch_id` / `parent_branch_id` and the recording
holds a map of alternate **branches**, a recording is a graph, not a line: it can
take a different path when the live screen matches a known alternate state (a
cookie banner, a "resume session?" dialog, an A/B variant).

## The data model

A recording is a `RecordingGraph` persisted as one JSON file under the agent
directory (`recordings/<id>.json`):

```jsonc
{
  "id": "f3a1…",
  "name": "Export monthly report",
  "description": "Export the current report as a named CSV",
  "created_at": "2026-06-28T12:00:00Z",
  "parameters": { "filename": "report.csv" }, // declared {placeholders}
  "actions": [
    /* ordered main path — see below */
  ],
  "branches": {
    "cookie-banner": [
      /* alternate path */
    ],
  },
  "drift_detected_at": null,
  "last_validated_at": "2026-06-28T12:05:00Z",
}
```

Each entry in `actions` (and in every branch) is a `RecordedAction`:

```jsonc
{
  "sequence_num": 1,
  "action": {
    "type": "mouse_click",
    "coordinate": [1840, 96],
    "button": "left",
    "reason": "recorded click",
  },
  "target": {
    "role": "button", // button | input | link | text | image | unknown
    "label": "Export",
    "context": "top-right", // 3×3 screen quadrant for disambiguation
    "ordinal": 1, // which match, if several look alike
    "fallback_xy": [1840, 96],
    "screen_region": { "x": 1790, "y": 80, "w": 100, "h": 32 },
  },
  "screen_state_before": {
    "perceptual_hash": "c3e1a0…",
    "window_title": "Reports — Acme",
    "active_app": "Chrome",
  },
  "screen_state_after": {
    "perceptual_hash": "9b27f4…",
    "window_title": "Reports — Acme",
    "active_app": "Chrome",
  },
  "parameter_keys": [],
  "branch_id": "",
  "timestamp": 0,
}
```

The full set of `action.type` values is the same `Action` vocabulary the agent
uses: `mouse_move`, `mouse_click`, `mouse_double_click`, `mouse_scroll`,
`key_press`, `type_string`, `wait`, `screenshot`, `shell_command`,
`open_application`.

## Recording

The `Recorder` is a drop-in `gui.GUI` decorator: it wraps the real platform
backend, so _every_ action that flows through the GUI during a session is
captured with no extra instrumentation. Query methods (`Screenshot`,
`GetMousePosition`, …) pass straight through; mutating methods (`MouseClick`,
`TypeString`, …) record a step.

For each action the recorder:

1. captures the screen **before** the action and computes its aHash;
2. performs the action against the underlying GUI;
3. derives a **semantic target** from the before-image around the interaction
   point;
4. appends a `RecordedAction` with the action, target, and before/after state
   hashes.

## Semantic targeting (no OCR)

`extractSemanticTarget` analyses a small window (≈100×32 px) centred on the click
and classifies the element with cheap pixel statistics only:

- **input** — a mostly-empty box with a strong border (high border ratio, low
  interior density).
- **button** — a dense, distinctly coloured block.
- **link** — a short horizontal underline run near the baseline.
- **text** / **image** — by interior pixel density.

It also records a **context** quadrant (`top-left` … `bottom-right`) and an
**ordinal** so replay can disambiguate repeated elements. This is deliberately
approximate: its job is _disambiguation during replay_, not perfect recognition.

## Perceptual state matching

Screens are compared by **average-hash similarity** (Hamming distance over the
64-bit aHash, normalised to `0.0–1.0`). aHash is robust to anti-aliasing, minor
re-layout, scrollbar changes, and resolution scaling — exactly the noise that
breaks coordinate macros — while still dropping sharply when the screen is
genuinely different.

## The replay algorithm

For each step, the replayer screenshots the live screen, hashes it, and compares
it to the step's `screen_state_before`. The similarity decides the path:

| Live similarity      | Path                                                           |
| -------------------- | -------------------------------------------------------------- |
| `≥ 0.85` (high)      | **Deterministic** — execute the recorded action as-is.         |
| `0.60 – 0.85` (warn) | Execute, but flag the step as a low-confidence drift warning.  |
| `< 0.60` (mismatch)  | Try a **branch**; if none matches, ask the **LLM** to recover. |

Thresholds (`HighSimilarity` 0.85, `WarnSimilarity` 0.60, `DriftThreshold` 0.75)
are configurable; see [configuration](configuration.md).

### Branches

On a mismatch the replayer first checks every recorded branch: if the live screen
matches a branch's entry state at high similarity, it runs that branch's actions.
This is how a recording absorbs expected detours (consent dialogs, "continue
where you left off", upsell modals) without involving the model at all.

### LLM-constrained fallback

Only when no branch matches does the model get involved — and this is the **only**
point at which the model sees live screen content during replay. It is given the
screenshot and the single next intended action, with a system prompt that
constrains it to "emit only the minimal action(s) needed to reach the state the
recording expected next" and to "avoid any action not within the original
recording scope." The recovery actions it returns are then executed through the
same gated path as everything else.

## Parameters

Recordings are reusable templates. `type_string` and `shell_command` fields may
contain `{placeholder}` tokens; at replay time they are substituted from
`--param key=value` pairs (and each step records which `parameter_keys` it uses):

```bash
orion replay f3a1… --param filename=june-2026.csv
# Replay finished: 3 steps, drift=false, 0 warnings
```

## Drift detection

Per-step similarities are accumulated across the run. If **more than 20 % of
steps** fall below `DriftThreshold` (0.75), the run is marked as drifted:
`drift_detected_at` is stamped on the graph, a `drift_detected` event is streamed
to the UI, and the recording is saved for review. A clean run instead stamps
`last_validated_at`. This turns silent breakage ("the macro clicked the wrong
thing for a week") into an explicit, surfaced signal.

```
orion replay f3a1…
# Replay finished: 7 steps, drift=true, 3 warnings   ← UI shows which steps drifted
```

## Safety during replay

Replay is not a trust bypass. Every action — whether replayed verbatim, taken
from a branch, or proposed by the LLM fallback — is evaluated by the **whitelist
engine** before it touches the GUI, and is blocked unless the decision is
_allow_. Combined with the scope-constrained fallback prompt, this contains a key
prompt-injection risk: even if on-screen content tries to steer the recovery
model, it cannot escalate beyond the recording's intent or past the user's
whitelist rules. Replay activity is also written to the append-only audit log
like any other action.

## See also

- [CLI reference](cli.md) — `orion record` and `orion replay`.
- [Configuration](configuration.md) — similarity/drift thresholds.
- [Architecture](architecture.md) — how recording fits the wider system.
