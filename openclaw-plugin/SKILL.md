# termux-x11 — openclaw skill

Gives openclaw agents direct control over the X11 desktop running in Termux (Xfce4 + TigerVNC).

## Tools

| Tool | Description |
|------|-------------|
| `x11_screenshot` | Capture the current X11 display as a base64 image (or file path) |
| `x11_click` | Mouse click at pixel coordinates |
| `x11_type` | Type text into the focused window |
| `x11_key` | Send a key or combination (xdotool syntax: `ctrl+l`, `Return`, `Escape`) |
| `x11_windows` | List open windows; optionally focus one by ID or name |

## Usage examples

```
# Navigate Chromium to a URL
x11_key key="ctrl+l"
x11_type text="https://example.com"
x11_key key="Return"
x11_screenshot
```

## Config

Config file: `~/.config/termux-x11/config.yaml`

```yaml
core:
  display: ":1"
  bin_dir: "~/.config/termux-x11/bin"
  screenshot_dir: "/tmp/x11-shots"

openclaw:
  screenshot_return: "base64"   # "base64" | "path"
```

`screenshot_return: "base64"` — image data inline in tool result (recommended for agents).  
`screenshot_return: "path"` — agent receives the file path; it must read or serve the file separately.

## Relationship with the `browser` tool

OpenClaw's own `browser` tool and these `x11_*` tools operate at genuinely different levels, and knowing which one answers a given question is most of what goes wrong in practice:

- **`browser`** acts on the DOM via element refs (snapshot → ref → click/type). Reliable, survives layout changes, but only sees what its active `profile` is actually attached to.
- **`x11_*`** acts on the whole X11 desktop directly — pixels and windows, no DOM awareness at all. It can see things `browser` cannot: whether Chromium is even running, what window is focused, the literal screen state, anything outside a browser tab entirely.

**When the `browser` tool seems broken or confusing, check `x11_windows` and/or `x11_screenshot` before asking the operator.** This is real, available ground truth — "is there actually a Chromium window?", "what does the screen actually show right now?" — and it resolves most `browser`-tool confusion directly instead of escalating it. Don't report "the browser isn't working" without having looked.

### `browser` tool profiles on this kind of setup, concretely

`browser`'s own `browser-automation` skill documents `profile="user"` (an "existing-session"/chrome-mcp attach) as *the* way to reach a real, logged-in browser. On a Termux+X11 setup like this one, that is usually **not** the profile that is actually paired and working — `profile="chrome"` (extension-relay transport) is. Concretely:

- `profile` omitted → the managed/CDP profile. On Termux, this usually fails with a generic `"No supported browser found"` error, because its auto-detection logic doesn't know about Termux's `chromium` package path. This is normal and expected, not a sign anything is broken — don't retry it, switch to `profile="chrome"`.
- `profile="chrome"` → extension-relay. Requires a real Chromium running on `DISPLAY=:1` (visible via `x11_windows` as a window named like `"... - Chromium"`) with the OpenClaw Chrome extension loaded and paired (`openclaw browser extension pair`, a one-time human step).

### The tab-sharing gotcha — the single most important thing in this file

**Pairing the extension and sharing a tab are two separate steps, and only one of them is visible in status output.** `openclaw browser doctor`/`status` can report everything green — plugin enabled, profile correct, "OpenClaw Chrome extension is connected" — while `browser` with `action="tabs"` still returns an empty list. That is not a broken pairing. It means **no tab has been shared yet.**

Sharing a tab is a manual, per-tab action the operator takes *in Chromium itself*: click the OpenClaw toolbar icon on the tab to share, or drag it into the OpenClaw tab group. Nothing automates this, and nothing in the doctor/status output distinguishes "not paired" from "paired but no tab shared" — both can look identical from the tool's side (connected, zero tabs). If `tabCount` is `0` after confirming the extension-relay check passes, the next step is telling the operator to share a tab, not re-pairing, not re-checking the plugin config, not concluding the setup is broken.

### Pixel actions vs. ref actions — a real limitation, not yet solved

`x11_click`/`x11_type` work on raw screen coordinates. Vision models do not reliably reason about exact pixel geometry from a screenshot, so coordinate-based clicks inside a web page are meaningfully less reliable than `browser`'s ref-based `action="act"`. Prefer `browser` for anything *inside* page content once a tab is reachable; reserve `x11_*` for whatever is outside any page (desktop chrome, other apps, dialogs) or for diagnosing `browser`/Chromium state itself, not for driving page interactions as a substitute for `browser`. A dedicated heuristic image-analysis tool for pixel-accurate interaction is a real, separate future need, not something this skill solves.

### Site-level interface memory

Before spending several turns rediscovering how a specific site's UI behaves, check whether that work is already recorded: `memory_search` the site's name or domain. `memory_search`/`memory_get` are plain semantic search over workspace Markdown files — there is no separate API to populate, just write a note.

After resolving something non-obvious about a specific site (an unusual login flow, a selector that needed `refs="aria"`, a workflow that only works via `x11_*` because the page blocks CDP automation, anything that cost real back-and-forth to figure out), write it down — a short `memory/sites/<domain>.md` note is enough. The goal is that the *next* session hitting the same site starts from that note instead of from zero. This directly reduces the "I keep having to explain this" problem without any new tool.

## Installation

Run the termux-x11 `install.sh --openclaw` from the repo, or copy the `openclaw-plugin/plugin/` directory to:

```
~/.openclaw/workspace/skills/termux-x11/plugin/
```

Then in `~/.openclaw/workspace/skills/termux-x11/` also place a `SKILL.md` (this file) so openclaw loads it into the agent's system prompt.

Run `npm install` in the plugin directory before first use.
