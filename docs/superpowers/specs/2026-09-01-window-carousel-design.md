# Window Carousel — Research Report & Design

**Status:** Research / design only. No implementation. Deferred project — starts after
`mpk-deck`, `ai-hub`, `content-system`, `trading-system` reach their current milestones.

**Date:** 2026-09-01
**Author:** Claude (research + design), for user review.

---

## 1. Executive Summary

Window Carousel is a Windows-only **Viewer / Detector / Router** for a 3.5" secondary
display. It replaces the Windows taskbar *for a registered set of windows*, shows a
System/AI **Home dashboard** by default, and briefly swaps to a relevant window when an
event fires, then returns Home.

**Core recommendations coming out of the research:**

1. **Do not embed or reparent foreign windows.** Cross-process `SetParent` is fragile
   (per-monitor DPI mismatch, UIPI, broken rendering in the host, multi-HWND Electron/Chrome,
   z-order/input bugs). Instead, **move/resize the real window to fill the 3.5" display**
   and let the OS composite it. Home is a normal always-on-top borderless window on that
   display; it shows when no view is active. This matches spec §13–§14 (manage real windows,
   no continuous capture).

2. **Use DWM live thumbnails (`DwmRegisterThumbnail`) only for previews** — the `+ Add View`
   picker and (optionally) Mini Bar hover previews. GPU-composited, near-zero CPU, always
   live. This is the single sanctioned "capture-like" path; it is not used for the main view.

3. **Claude / Codex usage needs no new integration.** The user's environment already
   maintains the data on disk:
   - `~/.claude/last-rate-limits.json` — Claude 5h / 7d `used_percentage` + `resets_at`
   - `~/.claude/last-statusline-input.json` — full statusline payload (model, context %,
     cost, cache stats, `session_name`, `thinking`, `effort`)
   - `~/.claude/codex-usage-cache.json` — Codex 5h / 7d `used_percent` + `resets_at` + `updated`
   - `~/.claude/ccusage_monthly_cache.json`, `~/.claude/ccusage_codex_monthly_cache.json` — monthly cost

   Window Carousel **reads these files and watches them for changes** (`ReadDirectoryChangesW`
   / `ReadDirectoryChangesW`). Zero API calls, zero risk to Claude/Codex performance, fully
   event-driven. The only missing piece — live *status* (WORKING/REVIEWING/…) and *current
   task* — is filled by a tiny Claude Code hook that writes a status file on
   `UserPromptSubmit` / `Stop` / `Notification` / `SubagentStop`.

4. **One event transport: a localhost WebSocket server** inside Window Carousel, JSON
   messages, mirroring how GlazeWM exposes its own IPC. Language-neutral; MPK Deck, trading,
   content, and Jarvis just open a socket and send `Event` / `Command` frames.

5. **Tech stack: C# / .NET 8 + WPF.** (User decision 2026-09-01 — free choice, leaned C# for
   flexibility + lower runtime weight; workspace-consistency with `mpk-deck`'s PySide6 was the
   only strong counter-argument and the user discounted it.) Window Carousel is
   window-management-heavy — unlike `mpk-deck`, which is a MIDI UI that only focuses windows
   by title. Every serious Windows WM in the research is native (GlazeWM C#, Komorebi Rust).
   C# gives a first-class Win32 message pump (`HwndSource`), clean P/Invoke, native
   `LibreHardwareMonitorLib`, a single self-contained exe, and it **eliminates the main
   Python risk** (WinEvent-hook callback GC-pinning, R9). WPF over WinUI 3 (maturity, Win32
   interop via `HwndHost`/`HwndSource`, lighter, stable for a long-running tray app).

6. **Architecture: a small Provider layer** (from Zebar) feeding an immutable view-model that
   the UI renders, plus a **Router state machine** kept strictly separate from detection
   (spec §24). Four providers: `SystemProvider`, `AiUsageProvider`, `ViewProvider`,
   `EventProvider`.

**What Window Carousel is closest to:** none of the reference projects do a "rotating single
view" — they are status bars (YASB, Zebar) or tiling WMs (GlazeWM, Komorebi). Window Carousel
is a **kiosk/dashboard rotator + a window mover**, not a window manager. It borrows *patterns*
from those projects, not code.

### Resolved since first draft (user, 2026-09-01)

- **R1 — panel type: real 3.5" mini monitor.** A standard display Windows composites. The
  "move real windows onto it" strategy holds; the serial-LCD branch is dropped.
- **Tech stack: C# / .NET 8 + WPF** (see item 5 above).
- **Must also run without the mini monitor** — Carousel opens on the primary display at a
  default size when the mini monitor is absent, same "degrade gracefully" rule as
  `mpk-deck`'s Monitor Manager. It is not hard-bound to one display.
- **MPK Deck control is a *maybe*, not a design driver.** The only plausible use is
  NEXT / PREVIOUS to slide through registered views (+ maybe HOME). `SHOW_<view>` / `LOCK`
  are dropped from the assumed set. The WS command channel stays generic so adding more
  later is cheap, but nothing is designed around it.
- **Taskbar hiding: known-clean allowlist only.** Toggle ex-styles only for window
  classes/exes verified to hide without side effects; everything else stays in the taskbar.
- **New repo:** `window-carousel` under `github.com/DD-PLAY-AI` (public, like `mpk-deck`).
  A `ROADMAP.md` entry gets added when work actually begins — not now (deferred).

---

## 2. Similar Project Research

| Project | Purpose | Language / Stack | Useful parts | Not useful | License | Carousel application |
|---|---|---|---|---|---|---|
| **YASB (Reborn)** `amnweb/yasb` | Configurable Windows status bar (polybar/waybar-style) | Python + **PyQt6**, `cx_Freeze`, Win32 APIs | Widget base class + Pydantic-validated YAML config + `WidgetBuilder` factory; `EventService` central pub/sub bus; per-monitor `BarManager`→`Bar`; CSS-like stylesheet with hot-reload; 50+ widgets incl. CPU/memory/taskbar/active-window | It is a *bar*, never moves/reparents foreign windows; its "taskbar widget" only launches/focuses, no view area; no small-screen dashboard mode | MIT | **Primary reference for Mini Bar + widget/config architecture.** Copy the *pattern* (validated-schema config → widget factory → pub/sub bus → hot-reload styling), not code. The design is GUI-toolkit-agnostic even though the impl is PyQt6. |
| **GlazeWM** `glzr-io/glazewm` | i3-style tiling WM for Windows | **C# / .NET**, Win32 via P/Invoke | Window discovery + **window rules** (match by process / class / title / regex); positioning via `SetWindowPos`; multi-monitor; **localhost WebSocket IPC** with `query windows/workspaces/monitors/focused` + `sub --events`; `glazewm-js` typed client | Full tiling engine, keybind layer, workspaces — all out of scope | GPL-3.0 → **architecture reference only, no code reuse** | **Reference for Window Matcher + move/resize + the event/IPC protocol shape.** Its IPC (query + subscribe over localhost WS) is almost exactly what §23 asks for — adopt that design. |
| **Zebar** `glzr-io/zebar` | Customizable taskbars / desktop widgets / popups | **Tauri** (Rust backend) + native webview (HTML/TS widgets), npm `zebar` client | **Provider architecture**: `ProviderManager` coordinates reactive providers (CPU, memory, battery, network, glazewm, window); `ProviderEmitter`; data flows provider→widget via IPC; widget = `.zebar.json` + HTML | Webview-per-widget rendering model; Tauri toolchain; deep GlazeWM coupling | MIT | **Primary reference for the Provider pattern.** `SystemProvider` / `AiUsageProvider` / `ViewProvider` / `EventProvider` each own a data source and emit normalized state; UI consumes only that. |
| **Komorebi** `LGUG2Z/komorebi` | Tiling WM for Windows | **Rust** (9-crate workspace), extends DWM | Event-driven core (captures window events, repositions); **named-pipe** subscription to every `WindowManagerEvent` (+ optional TCP); JSON-schema'd notifications; "minimal CPU and memory" | Full tiling engine; Rust dev velocity for a UI-heavy dashboard is poor; sponsorware for latest | Complex (commercial-ish) → **idea reference only** | Confirms the **event-driven window-management model** (hook events → reposition) and that a Rust core is the way to get "minimal CPU" *if* runtime cost ever becomes the binding constraint. Named-pipe vs WS is a transport data point. |
| **InfoPanel** `habibrehmansg/infopanel` | Desktop / external-display sensor visualization | **C# / .NET**, WPF-ish, Direct2D rendering | **LibreHardwareMonitor + HWiNFO SHM integration** for CPU/GPU/RAM/temps; drag-and-drop panel editor; **secondary-screen + USB-LCD (BeadaPanel, Turing) support**; Direct2D for "smoother, lower-CPU" overlays; profiles | Its whole UI is a different product; USB-LCD path is moot (panel is a real monitor) | MIT | **Reference for the System Monitor data layer** — how a C# app wires `LibreHardwareMonitorLib` for CPU/GPU/RAM% + temps and keeps the overlay low-CPU (Direct2D, throttled redraw). Read its sensor code; don't take a dependency. |
| **Rainmeter** | Long-lived Windows desktop widgets (CPU/RAM/net/disk/clock) | C++ core + Lua/INI skins | 15+ years of **small-space information hierarchy**: big number + thin bar + tiny history graph + NORMAL/WARN/CRIT color states; low-frequency updates (1s typical) | Skin/plugin ecosystem, INI config model — nothing to reuse structurally | GPL-2.0 → **visual/UX reference only** | **Reference for Home View information density**: number-first hierarchy, 1s refresh, no 60fps graphs, restraint on a small canvas. Matches the user's Mini Display inspiration notes in §5. |
| **ccusage** `ryoppippi/ccusage` | Cost/usage reports from local agent logs | Node / npx, zero deps | Reads `~/.claude/*.jsonl` locally, no account, no telemetry; already wired into the user's `statusline.ps1` for monthly cost | CLI report tool, not a live status feed; monthly granularity | MIT | **Already in use.** Window Carousel reads the cache files `statusline.ps1` produces (`ccusage_monthly_cache.json`, `ccusage_codex_monthly_cache.json`) for the "Mo" cost figure. No direct dependency. |
| **claude-code-statusline** `ohugonnot/claude-code-statusline` | Real-time 5h/weekly quota in the status line | Bash/POSIX | Documents the mechanism: **Claude Code ≥2.1.x passes `rate_limits.five_hour` / `.seven_day` directly on statusline stdin** (no API call); fallback = OAuth token from `~/.claude/.credentials.json` → undocumented `https://api.anthropic.com/api/oauth/usage` with `anthropic-beta: oauth-2025-04-20` | The fallback endpoint is undocumented and can change | MIT | **Confirms Window Carousel should not call any endpoint itself.** Claude Code already delivers `rate_limits` on stdin; the user's `statusline.ps1` already persists it. We read the file. |
| **claude-codex-usage-dashboard** `frankchiu-dev/...` | Local Windows dashboard, Claude + Codex, 5h + weekly | **Node 18+, zero npm deps**, LAN web UI | Validates the exact approach: Claude from `~/.claude/usage-cache.json` (populated by a statusLine integration); Codex from newest `~/.codex/sessions` `rate_limits` snapshot; "dashboard does not poll — depends on when the tools write" | Web-UI delivery; its own cache-file names differ from the user's | (unspecified — treat as reference) | **Direct precedent for the AI monitor design.** Same data-source strategy we recommend. Confirms "not affiliated / best-effort / verify trademark" caveats for any logo use. |
| **codex-cli-usage** `wakamex/codex-cli-usage` | Codex rate-limit CLI | Python | `codex app-server --stdio` JSON-RPC → **`account/rateLimits/read`** → `rateLimits.primary` / `.secondary` with `pct` / `resets_at` / `window_secs`; per-model + code-review limits; fallback `/backend-api/codex/usage`; reads `~/.codex/auth.json`, never writes | Spawning `codex app-server` per poll is heavier than reading a cache file | (unspecified) | **Reference for how the user's `~/.claude/scripts/codex-usage.ps1` gets its numbers.** Window Carousel does not run this — it reads `codex-usage-cache.json`. Good to know the method for debugging. |
| **TaskbarXI**, **Dual Monitor Taskbar**, **PowerToys Command Palette Dock** | Second-monitor taskbars / docks | C# / C++ | Prior art for "a taskbar on the second monitor" and hiding/replacing the primary; PowerToys Command Palette Dock has live system monitors + transparency + any-edge placement | General-purpose taskbars — Window Carousel is scoped to *registered* windows only | MIT (PowerToys, DMT) | **Reference for Mini Bar placement + taskbar-replacement UX** and for the limits of hiding the real taskbar. |

### Reference role map (revised from the user's §43 hypothesis)

```
YASB Reborn      → Mini Bar + widget/config architecture (YAML + validated schema + factory + pub/sub + hot-reload styling)
Zebar            → Provider pattern (data source → normalized state → UI)
GlazeWM          → Window Matcher + move/resize + localhost-WS event/IPC protocol shape
Komorebi         → event-driven reposition model; Rust core only if runtime cost binds
InfoPanel        → System Monitor data layer (how a C# app wires LibreHardwareMonitorLib, low-CPU)
Rainmeter        → small-screen information hierarchy + update-frequency restraint
claude-code-statusline / claude-codex-usage-dashboard → AI usage = read local cache files, never call an endpoint
codex-cli-usage  → how the Codex numbers are produced (for debugging only)
```

The user's hypothesis was essentially correct. The one correction: **GlazeWM's IPC design
(not just its window matching) is a strong reference** — it already implements "query state +
subscribe to events over localhost WS", which is what §23 is asking for.

---

## 3. Additional Projects Found (not in the user's list)

- **`glazewm-js`** — typed TS client for GlazeWM's IPC. If Window Carousel ever wants to know
  real window/workspace state from GlazeWM (in case the user runs one), this is the shape to
  copy for our own client library.
- **Komorebi's named-pipe notification schema** (`komorebic notification-schema`) — a
  concrete, versioned JSON event schema for window events. Good prior art for versioning our
  own `Event` model (§23).
- **`bozdemir/claude-usage-widget`** — cross-platform desktop widget + CLI, `pip install`,
  Claude + opt-in Codex, burn alerts / forecasts / 11 themes. Closest existing thing to the
  Home View's AI panel. Worth reading for the "pacing marker" idea (expected % vs actual %,
  already in the user's `statusline.ps1` concept).
- **`f-is-h/usage4Claude`** (macOS menu bar) and **`cero-t/ccc-usage-dashboard`** — more
  combined Claude+Codex dashboards; all converge on the same "read local files" approach,
  which strengthens confidence in the recommendation.
- **PowerToys `MoveWindowToMonitor` / FancyZones** — FancyZones' snapshot/restore of window
  layouts and its per-monitor DPI handling in C# is a good reference if the move/resize math
  gets hairy.
- **`LibreHardwareMonitorLib`** (NuGet) — the realistic GPU/CPU/temp source on this hardware
  (MX150-class), referenced as a DLL the way InfoPanel does it. Some values may need admin —
  verify in the spike. HWiNFO Shared Memory is the fallback if LHM misreads the GPU.
- Plenty of C# `SetWinEventHook` + `HwndSource` message-pump gists/snippets exist; the
  out-of-context hook pattern is well-trodden in .NET.

---

## 4. Home View UI/UX Recommendation

**Layout = spec §4.** Reinterpreted responsively, not pixel-fixed. Visual detail inspiration
= the (not-provided) Mini Display image, approximated from the user's §5 notes: big numbers,
sparklines, thin progress bars, NORMAL/WARN/CRIT color, clear card borders, dark background,
high contrast. Aesthetic: **modern Windows dashboard that doesn't age**, not cyberpunk.

### Regions (top → bottom)

1. **Status strip** — clock (large), `SYSTEM ●` health dot (green/amber/red = worst of
   CPU/GPU/RAM/view-health).
2. **System row** — CPU / GPU / RAM as `LABEL  NN%` (large %) + thin bar. Sparkline optional,
   shown only when width allows (see §29 density rules). Color by threshold.
3. **AI split (2 columns)** — `CLAUDE ●` / `CODEX ●`, each: status word (IDLE/WORKING/
   WAITING/REVIEWING/COMPLETED/ERROR/LIMIT_NEAR), `5H ▓▓░ NN%`, `7D ▓░░ NN%`, `Reset Xh Ym`
   countdown. A thin **pacing marker** on each bar (expected % given elapsed window time) —
   cheap, already conceptually in `statusline.ps1`.
4. **Event timeline** — last **3** events on the compact layout, up to ~8 when tall. Icon +
   source + message + relative time. History kept in memory capped at **50** (§9, §34).
5. **Mini Bar** — pinned bottom. `⌂ 🎹 📈• ✎ 🤖 +`. Never a dashboard; navigation + current
   view + unread dot only.

### Responsive rules (spec §28–§29)

- Single fluid layout (WPF `Grid` with star sizing + `ViewBox` where proportional scaling is
  wanted), **no per-resolution breakpoints for sizing** — clamp font sizes, bars, and gaps
  to the window size.
- **Density, not just scale**: three tiers keyed off *width* (and height for the timeline):
  - **Compact** (≤ ~1100 logical px wide): §4 layout exactly. No sparklines. 3 events.
  - **Comfortable**: sparklines appear; 5–6 events; AI panel adds `Context NN%` line.
  - **Expanded** (wide external monitor): add Disk / Network / a temperature line; longer
    timeline; AI panel adds cost + cache hit-ratio. Still one screen, still Home.
- Minimum target **960×640 logical** must stay readable — verify with a real 960×640 window
  during Technical Validation. Assume per-monitor DPI v2; a 3.5" high-res panel likely runs
  150–250% scale.

### Detail / Expanded view

Anything beyond the above (token counts, per-model 7d limits, cost breakdown, full event
history) lives in an **Expanded View** reachable from the Home AI card, not on the default
Home. Keeps the 3.5" Home calm (§7, §8).

---

## 5. Mini Bar Recommendation

- **Model after YASB's taskbar widget UX**, scoped to registered views only. Fixed height
  (~1 row), icon-only by default, label on hover if width allows.
- **Order = carousel navigation order.** Left/right (or MPK NEXT/PREVIOUS) walks it. `⌂` Home
  is index 0, undeletable.
- **Per-item state**: current-view highlight (accent underline/border), unread indicator
  (small dot; `•` in §10), optional unread count. States UNREAD → VIEWING → READ; entering a
  view clears UNREAD.
- **`+`** opens Add View (see §7 window mgmt).
- No animation beyond a short (~120ms) highlight move. No glow, no bounce (§11).
- Mini Bar is **always visible**, including while a foreign window fills the view area — it
  stays on top as a thin always-on-top strip, or the foreign window is sized to leave a
  reserved strip at the bottom. Prefer the reserved-strip approach (no z-order fighting).

---

## 6. Window Management Recommendation

### Decision: move real windows, do not embed

Research is unambiguous that cross-process embedding via `SetParent` is unreliable:
- `SetParent` does not fix `WS_CHILD`/`WS_POPUP` styles; you must toggle them manually.
- **Per-monitor DPI-awareness mismatch** between Carousel and the target ⇒ undefined behavior
  / crashes (mixed-mode DPI + null HWND under message-only windows is a known crash).
- **UIPI**: `SetWindowLongPtr` fails if the target runs at higher integrity.
- Host rendering breaks: hover states and animations in the host stop; workaround is
  disabling hardware acceleration (unacceptable for our UI).
- Electron/Chrome/Terminal use **multiple HWNDs**; you reparent the wrong one and get a blank
  or a fragment.

So the view area is **not** a container. When a view activates:

```
resolve Matcher → HWND
  ├─ if minimized: ShowWindow(SW_RESTORE)
  ├─ compute target rect = 3.5" display work area minus Mini Bar strip
  ├─ SetWindowPos(HWND, HWND_TOP, rect, SWP_NOACTIVATE?) — with DPI-correct coords
  ├─ optionally SetForegroundWindow (only if the view needs input; see focus-stealing R6)
  └─ record previous rect + placement (for restore)
on view exit:
  ├─ restore previous rect/placement OR ShowWindow(SW_MINIMIZE) per view config
  └─ show Home (Home window was underneath / hidden, now raised)
```

Home is a borderless, always-on-top, `WS_EX_TOOLWINDOW` window. It targets the **mini
monitor** when present (identified by EDID / device name, not display index — same rule as
`mpk-deck` Phase 2), and **falls back to the primary display** at a default size when the
mini monitor is absent (user decision 2026-09-01 — "must run without the mini monitor").
Display hot-plug (`WM_DISPLAYCHANGE` / `EVENT_...`) re-homes the window. While a foreign
window fills the view area, Home is hidden or covered; on return it is `SetWindowPos`-raised.
When Carousel is on the primary display (no mini monitor), the view area is a sub-rect of
Carousel's own window, not the whole screen.

### Window Matcher (spec §18)

Match spec, evaluated in priority order, first match wins, all fields optional and AND-ed:

```
executable   : basename, case-insensitive       (e.g. "chrome.exe")
process_name : optional, == executable minus ext
window_class : GetClassName                      (e.g. "Chrome_WidgetWin_1")
title        : substring | glob | regex (flag)   (e.g. "* - YouTube - *")
title_regex  : explicit regex form
url_host     : browser only, best-effort via UI Automation (see below)
ancestor     : match only top-level, owned-window rules
```

- **HWND is never persisted.** On startup / reconnect, enumerate top-level windows
  (`EnumWindows`), filter to visible + non-tool + has-title, score against saved matchers.
- **Browser tab problem (§18):** a browser is one process with many windows/tabs. Matching by
  title is brittle (title changes with the active tab). Best-effort: read the address/omnibox
  via **UI Automation** (`AutomationElement`, `ControlType.Edit` / `LegacyIAccessible`) to
  get the URL host. If UIA read fails, fall back to title glob and accept imperfection.
  Document this as a known limitation. Do **not** install browser extensions.
- **DWM thumbnail in the Add View picker**: enumerate candidates, show each as a live
  `DwmRegisterThumbnail` preview + exe/title. User picks → create matcher (offer to
  auto-fill exe+class, let user tighten title/regex) → minimize the window, return Home
  (§7, §15).

### How the user triggers "Add" (user question, 2026-09-01)

| Entry point | Feasible? | Plan |
|---|---|---|
| **`+` button in the Carousel window** | Yes | **Primary.** Opens the picker above. |
| **Carousel's own tray-icon right-click menu** | Yes (we own that menu) | Add two items: **"Add foreground window"** (grab the last non-Carousel foreground HWND, prefill a matcher, confirm) and **"Add window…"** (opens the full `+` picker). Cheap — the tray icon exists anyway. |
| **Global hotkey** (e.g. add window under cursor / foreground) | Yes (`RegisterHotKey`) | Optional, config-gated, off by default. |
| **Right-click the Windows taskbar / an app's taskbar button** | **No** | Windows 11 does not let third-party apps inject items into the taskbar or jump-list context menus. Shell context-menu injection there is closed. Not pursued. |
| Right-click a foreign window's titlebar | Technically possible via a mouse hook | Invasive, rejected (§14 "no continuous hooks into other apps" spirit). |

So: **`+` button + tray-menu "Add foreground window"** are the two supported paths; global
hotkey is an optional extra. The taskbar idea is not possible.

### Persistent vs Temporary (spec §16–§17)

- **Window Carousel does not launch anything.** Persistent apps auto-start themselves via
  Windows; Carousel loads saved matchers and searches. Found ⇒ `CONNECTED`; not found ⇒
  `WAITING` (not `ERROR`); background re-scan on a low-frequency timer **and** on
  `EVENT_SYSTEM_FOREGROUND` / `EVENT_OBJECT_SHOW` hook (a new window appearing triggers a
  match attempt — event-driven, §31).
- **Temporary** views live only for the session; not restored after reboot; auto-removed when
  the window closes (`EVENT_OBJECT_DESTROY`).

### Taskbar hiding (spec §19) — known-clean allowlist only (user decision 2026-09-01)

- Maintain a small allowlist of window classes / exes **verified** to hide without side
  effects (built during the spike and extended as views are added). For a matched window on
  the allowlist: `SW_HIDE` → set `WS_EX_TOOLWINDOW`, clear `WS_EX_APPWINDOW` → `SW_SHOW`
  (or `ITaskbarList::DeleteTab(hwnd)`).
- **Everything not on the allowlist stays in the Windows taskbar.** No "try and revert"
  probing — the user does not want windows flickering or misbehaving on registration.
- The Add View flow can show a "hide from taskbar" checkbox per view, defaulting off,
  enabled only when the target matches an allowlist entry.
- On view removal / shutdown, restore original ex-styles for anything that was changed.

---

## 7. Claude Monitor Recommendation

**Data source: files, already maintained. No endpoint calls from Window Carousel.**

| Field | Source | Notes |
|---|---|---|
| 5h used % + reset | `~/.claude/last-rate-limits.json` → `five_hour.{used_percentage,resets_at}` | Written by `statusline.ps1` from Claude Code's stdin `rate_limits` (Claude Code ≥2.1.x provides it natively). |
| 7d used % + reset | same file → `seven_day` | |
| Model, context %, cost, cache hit-ratio, `session_name`, `thinking`, `effort` | `~/.claude/last-statusline-input.json` | Full statusline payload; `session_name` doubles as a rough "current task" label. |
| Monthly cost | `~/.claude/ccusage_monthly_cache.json` → `totals.totalCost` | ccusage, refreshed by `statusline.ps1` every 5 min. |
| **Status** (IDLE / WORKING / WAITING / REVIEWING / COMPLETED / ERROR / LIMIT_NEAR) | **New: a Claude Code hook** writes `~/.claude/carousel/claude-status.json` | See below. LIMIT_NEAR is derived by Carousel from used % thresholds. |
| **Current task** | same hook (from `UserPromptSubmit` prompt text, truncated) + `session_name` | |
| Last activity | mtime of `last-statusline-input.json` / status file | |

### The status hook (small, additive, already the user's pattern)

A hook script on `UserPromptSubmit` (→ WORKING + task text), `Notification` (→ WAITING),
`Stop` / `SubagentStop` (→ IDLE/COMPLETED), `PreToolUse` for review tools (→ REVIEWING,
optional). It appends a one-line JSON to `~/.claude/carousel/claude-status.json`. Cost:
one file write per event, no polling, no effect on Claude Code performance. This is the
**only** new integration on the Claude side.

### Update mechanism

`FileSystemWatcher` (`ReadDirectoryChangesW`) on `~/.claude/` and `~/.claude/carousel/`.
Debounce 200–500 ms. Pure event-driven; the files only change when Claude is actually used
(§31).

### Reliability

The undocumented `/api/oauth/usage` endpoint and the exact statusline schema **can change** —
but that is *Claude Code's* dependency, not ours. If Claude Code stops providing `rate_limits`
on stdin, `statusline.ps1` writes nothing new and Carousel shows the last-known value greyed
out with a stale indicator. Acceptable degradation.

---

## 8. Codex Monitor Recommendation

Same UI model as Claude, normalized behind one interface.

| Field | Source | Notes |
|---|---|---|
| 5h / 7d used % + reset | `~/.claude/codex-usage-cache.json` → `five_hour` / `seven_day` (`used_percent`, `resets_at`), `updated` | Written by `~/.claude/scripts/codex-usage.ps1`, which calls `codex app-server --stdio` → `account/rateLimits/read` (`rateLimits.primary`/`.secondary`). Refreshed by `statusline.ps1` when older than 5 min. |
| Monthly cost | `~/.claude/ccusage_codex_monthly_cache.json` → `totals.costUSD` | |
| Status / current task | **Codex writes its own native memory per run**; there is no live status file yet. Options: (a) reuse the Codex plugin's session artifacts under `~/.codex/sessions`; (b) a Codex wrapper hook (the user's `codex-run.ps1` already emits structured SUMMARY/FILES_CHANGED/TESTS) writes `~/.claude/carousel/codex-status.json` at start/end of a delegation. | Recommend (b) — the wrapper is already the integration point per `project_codex_integration`. |
| Last activity | `updated` field + status file mtime | |

### Normalization — `AiUsageProvider`

```
interface AiAgent:
    id            : "claude" | "codex"
    display_name  : str
    status        : AgentStatus enum
    current_task  : str | None
    primary_usage : Usage   { used_pct: float, resets_at: datetime, label: "5H" }
    weekly_usage  : Usage   { used_pct: float, resets_at: datetime, label: "7D" }
    monthly_cost  : Money | None
    last_activity : datetime
    stale         : bool          # source file older than freshness threshold
```

`ClaudeSource` and `CodexSource` implement a common `read() -> AiAgent`; the provider watches
their files and emits `AiAgent` snapshots. UI never knows which file anything came from (§8).

### Caveat

`account/rateLimits/read` via `codex app-server` on Windows: the user's memory notes
`codex exec resume` is broken, but `codex-usage.ps1` evidently works (the cache file is
populated). Window Carousel does not spawn `codex app-server` itself — it only reads the
cache — so this risk is contained to the existing script.

---

## 9. Detector / Event Recommendation

### Strict separation (spec §24): Detect → Event → Router → DisplayAction

```
Detectors (observe only, emit Event)          Router (decide)                 DisplayAction (act on UI)
─────────────────────────────────────         ──────────────────             ────────────────────────
SystemDetector      CPU/GPU/RAM thresholds  →  classify severity          →   HomeStatusUpdate
AiDetector          status/usage transitions   compute attention score        TimelineAppend
ViewDetector        connected/waiting/closed/  apply per-source policy         MiniBarHighlight
                    not-responding (hooks)     dedupe + rate-limit             AutoView(view, duration)
ExternalDetector    WS frames from other apps  priority queue                  CriticalInterrupt(ack)
```

- **Not every event switches the view.** Most events → `HomeStatusUpdate` + `TimelineAppend`
  only. `AutoView` requires `severity ≥ notice` **and** a `suggested_view` **and** the user
  not currently interacting with another view.
- **Router is a state machine** with states `HOME`, `AUTO_VIEW(view, timer)`,
  `MANUAL_VIEW(view)`, `CRITICAL(view, awaiting_ack)`. Transitions are the only place view
  switches happen.

### Event model (spec §23)

```json
{
  "id": "uuid",
  "source": "trading_sys | content_sys | jarvis | mpk_deck | system | claude | codex | view",
  "type": "order_failed",
  "severity": "info | notice | warning | critical",
  "suggested_view": "trading" | null,
  "message": "Order rejected: insufficient margin",
  "data": { },
  "timestamp": "2026-09-01T12:00:00Z",
  "schema_version": 1
}
```

Commands (external → Carousel, spec §21) share the transport, different frame type. MPK Deck
control is a *maybe* (user, 2026-09-01) — the realistic use is sliding through views:

```json
{ "kind": "command", "action": "HOME|NEXT|PREVIOUS", "view": null }
```

The parser accepts an open-ended `action` string and `view` field so `SHOW_<view>` / `LOCK`
etc. can be added later without a schema change, but v0.3 only implements `HOME/NEXT/PREVIOUS`.

### Transport (spec §23)

**localhost WebSocket** (`ws://127.0.0.1:<port>`), bind loopback only, single JSON frame per
message, same `query` + `subscribe` + `send` model as GlazeWM's IPC.

- Chosen over Named Pipe: trivially cross-language (Python, C#, Rust, Node, browser all have
  WS), easy to test, GlazeWM/Zebar prove the pattern. Named pipes are marginally lower
  overhead but a worse polyglot story.
- Chosen over local HTTP: WS gives server-push (events) without polling.
- Auth: loopback-only + a token in a user-readable file (`~/.claude/carousel/ipc-token`);
  reject frames without it. No network exposure.
- Detectors that need Win32 signals use **`SetWinEventHook` (WINEVENT_OUTOFCONTEXT)** on a
  thread with a running Win32 message pump (`HwndSource` / a dedicated STA loop):
  - `EVENT_SYSTEM_FOREGROUND`, `EVENT_SYSTEM_MINIMIZEEND` (foreground/restore)
  - `EVENT_OBJECT_LOCATIONCHANGE` (a tracked window moved/resized — for "user grabbed it")
  - `EVENT_OBJECT_DESTROY`, `EVENT_OBJECT_SHOW` (view closed / candidate appeared)
  - `EVENT_OBJECT_NAMECHANGE` on tracked HWNDs (title-based re-match, e.g. browser tab)
  - The `WinEventProc` delegate must be **held as a field** on the hook class (not a
    transient lambda) or the GC collects it and the callback crashes.

---

## 10. Performance Recommendation (spec §30–§34)

| Concern | Approach |
|---|---|
| Idle CPU ~0–1% | No polling loops. WinEvent hooks + `FileSystemWatcher` + WS server (async, idle = 0). The only timer is the 1s system-sensor read and a low-frequency (30–60s) view re-scan / reconnect. |
| System sensors | 1s refresh (§32). CPU/RAM via `PerformanceCounter` / PDH. GPU + temps via `LibreHardwareMonitorLib` (single shared `Computer` instance, `Update()` once/sec). No history buffer beyond the sparkline's ~60 samples (ring buffer). |
| GPU activity | No GPU-accelerated effects. Static or CSS-transition UI. DWM thumbnails are GPU-composited by *DWM*, negligible, and only alive while the Add View picker is open. |
| No continuous capture | Enforced by design — the view area shows the *real* window, not a copy (§14, §33). DWM thumbnail is the only exception and is preview-scoped. |
| Disk I/O | Read cache files only on change (watcher). Event history in memory, capped at 50 (§9). Log to a size-capped rotating file (1 MB × 3). |
| Memory / handle leaks | Unhook every `SetWinEventHook` on teardown; single reusable hook range where possible. Cache HWNDs weakly, invalidate on `EVENT_OBJECT_DESTROY`. One WS server, bounded client list. No unbounded lists anywhere. |
| Long-run stability (§34) | Multi-day soak test in the Test Plan: monitor Carousel's own RSS, handle count, thread count, GDI objects over 72h; assert flat. |
| Animation | ≤120ms transitions, no infinite/idle animation, no 60fps graphs (§32, §52). |

Rendering: WPF retained-mode; bind the view-model, avoid rebuilding the visual tree each
tick. Sparklines as a cached `StreamGeometry` / `DrawingVisual` redrawn at the 1 Hz data
rate, not on `CompositionTarget.Rendering`. No `BitmapEffect` / blur.

---

## 11. Technology Stack Comparison

**Decided: C# / .NET 8 + WPF** (user, 2026-09-01). Table kept for the record.

| Criterion | **C# / .NET 8 + WPF** (chosen) | Python + PySide6 + pywin32/ctypes | Rust (Tauri or native) |
|---|---|---|---|
| Workspace consistency | New stack in the workspace | Matches `mpk-deck` (user discounted this) | New stack |
| Window move/resize/focus | Proven (GlazeWM) | Proven in `mpk-deck` (but only focus-by-title, trivial) | Proven (Komorebi) |
| WinEvent hooks + message loop | **Cleanest** — native pump via `HwndSource`, delegate held by the class | Works (`win32_window_monitor`, Qt loop) but callback must be GC-pinned or it crashes (R9) | Clean |
| DWM thumbnails | Native P/Invoke | ctypes P/Invoke, ~40 lines | Native |
| GPU / temp sensors | **`LibreHardwareMonitorLib` native** (InfoPanel does exactly this) | NVML pure-Python OK; LHM needs `pythonnet` friction | crates exist, less mature |
| Per-monitor DPI v2 | First-class | `SetProcessDpiAwarenessContext` via ctypes; Qt mostly handles it | Manual |
| Idle CPU / RAM | ~40–80 MB | ~60–120 MB | ~15–40 MB, best idle |
| Packaging | **single self-contained exe** | `cx_Freeze` / PyInstaller | single exe |
| Dev velocity (Claude + user) | Good | Highest (both fluent) | Low for UI-heavy work |
| Long-run stability | Good | Good with discipline | Best |

### Why C# won here

- Window Carousel is **window-management-heavy**. `mpk-deck`'s Python window code is a
  one-liner (`SetForegroundWindow` by title); nothing about it transfers to continuous
  cross-process move/resize + hooks + DWM + per-monitor DPI. The consistency argument is
  therefore weaker than it looks.
- C# removes **R9** (the WinEvent-hook GC-pinning failure mode) structurally, gives native
  `LibreHardwareMonitorLib` (the realistic GPU/temp source on an MX150-class part), a single
  self-contained exe, and a proper Win32 message pump for free via `HwndSource`.
- Every window-manager in the research is native for these reasons.
- No rewrite risk, no fallback plan to maintain, no `pythonnet` sidecar.

### Still a small spike first (§15 Phase 0) — but not a language go/no-go

The spike now only de-risks the genuinely uncertain Win32 behavior (cross-process move/resize
at 200% DPI, taskbar-hide per app type, DWM thumbnail cost), not "does the language work".

---

## 12. Final Architecture

```
┌──────────────────────────── Window Carousel (single process) ────────────────────────────┐
│                                                                                          │
│  Providers (own a data source, emit normalized state)                                     │
│  ┌────────────────┐ ┌────────────────┐ ┌───────────────┐ ┌────────────────┐                │
│  │ SystemProvider │ │ AiUsageProvider│ │ ViewProvider  │ │ EventProvider  │                │
│  │ 1s sensor read │ │ file-watch on  │ │ EnumWindows + │ │ WS server +    │                │
│  │ CPU/GPU/RAM    │ │ ~/.claude/*    │ │ WinEventHook  │ │ Win32 detectors│                │
│  └───────┬────────┘ └───────┬────────┘ └──────┬────────┘ └───────┬────────┘                │
│          └──────────────────┴─────────────────┴──────────────────┘                        │
│                                   │  (immutable snapshots)                                 │
│                          ┌────────▼─────────┐                                              │
│                          │   AppState       │  single source of truth, immutable updates  │
│                          └────────┬─────────┘                                              │
│              ┌────────────────────┼────────────────────┐                                   │
│      ┌───────▼────────┐   ┌───────▼────────┐   ┌────────▼────────┐                          │
│      │ Router (FSM)   │   │ Home View (UI) │   │ Mini Bar (UI)   │                          │
│      │ Event→Attention│   │ System / AI /  │   │ nav + unread    │                          │
│      │ →DisplayAction │   │ Timeline cards │   │                 │                          │
│      └───────┬────────┘   └────────────────┘   └─────────────────┘                          │
│              │ DisplayAction                                                               │
│      ┌───────▼──────────────┐                                                              │
│      │ WindowController     │  SetWindowPos / ShowWindow / ex-style toggle / DWM thumbs     │
│      │ (the only code that  │                                                              │
│      │  touches foreign HWNDs)                                                             │
│      └──────────────────────┘                                                              │
│                                                                                          │
│  Persistence: views.json (matchers, persistent/temp, order, per-view durations)           │
│               config.json (thresholds, layout prefs, ipc port)                            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
        ▲ WS (ws://127.0.0.1)                         ▲ files (~/.claude/*, hooks)
        │                                             │
  MPK Deck, trading-system, content-system, Jarvis    Claude Code hook, codex-run.ps1 hook
```

### Data flow

1. Providers read their source (event-driven wherever possible) and push **immutable
   snapshots** into `AppState`.
2. `AppState` change notifies the Router and the UI.
3. Router runs its FSM: incoming `Event` → severity/attention → maybe a `DisplayAction`.
4. `DisplayAction` is the *only* path to `WindowController`; the UI never moves windows.
5. `WindowController` is the *only* code that calls Win32 on foreign HWNDs — single choke
   point for the risky calls, easy to test and reason about (deep-module boundary).

### View lifecycle

`REGISTERED → (search) → WAITING | CONNECTED → VIEWING → CONNECTED → … → CLOSED/REMOVED`

### Event lifecycle

`received → validated (schema_version) → deduped/rate-limited → scored → queued →
DisplayAction dispatched → timeline-recorded → (if AutoView) auto-return timer → HOME`

### Provider interface (uniform)

```
class Provider:
    def start(self) -> None          # subscribe / open watchers
    def stop(self)  -> None          # tear down, no leaks
    signal state_changed(snapshot)   # immutable value object
```

### Persistence model

Two JSON files under `~/.claude/carousel/` (or `%APPDATA%\WindowCarousel\`):
`views.json` (matchers + order + type + per-view auto-return duration override) and
`config.json` (thresholds, layout tier overrides, IPC port/token, **auto-return default
= 12 s**). No database. Atomic write (temp +
rename). Corrupt file ⇒ back up + start from defaults + Home (§3 "errors keep Home").

---

## 13. Risk List

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| ~~R1~~ | **RESOLVED (user, 2026-09-01): real 3.5" mini monitor**, a standard display Windows composites. | — | The "move real windows onto it" strategy holds. Serial-LCD branch dropped. |
| **R1b** | With no mini monitor, Carousel runs on the primary display and the view area is a sub-rect of its own window — foreign window vs. Carousel chrome overlap / z-order on one screen. | Medium — a real layout question for the "no mini monitor" mode | Decide in v0.1: frame-around-a-hole (Carousel chrome surrounds the positioned foreign window, kept 1 z-level below it) vs. Carousel shrinks to just the Mini Bar while a view is active. Lean toward the latter (simpler, no z-order fighting). |
| R2 | Undocumented `/api/oauth/usage` + statusline `rate_limits` schema change | AI usage bars go stale | Not our dependency (Claude Code's). Show last-known + stale flag. `AiUsageProvider` isolates the parse. |
| R3 | `codex app-server` `account/rateLimits/read` unstable on Windows | Codex usage stale | We read the cache file, never spawn the server. Contained to existing `codex-usage.ps1`. |
| R4 | Live status/task detection needs hooks that don't exist yet | No WORKING/REVIEWING until built | v0.2 scope; degrade to "usage-only" cards until then. Hook is small and additive. |
| R5 | Fullscreen / maximized / exclusive-fullscreen apps (games, video) resist move/resize | View shows wrong size or steals the whole screen | Detect `WS_MAXIMIZE` / fullscreen; un-maximize before positioning; for exclusive fullscreen, show a "can't embed" Home card instead of fighting it. |
| R6 | Focus-stealing: `SetForegroundWindow` is restricted; moving a window may or may not focus it | View appears but doesn't take keyboard, or unexpectedly grabs focus | Only foreground a view when its config says it needs input. Use `SWP_NOACTIVATE` otherwise. Accept Windows' foreground-lock rules; document. |
| R7 | Per-monitor DPI v2: 3.5" high-res panel at 200%+, coordinate math wrong | Windows land half-off-screen or wrong size | `SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)`; do all rect math in physical pixels; test on the real panel. |
| R8 | Taskbar hide fails for UWP/Electron | Some views still show in the real taskbar | **Accepted** — known-clean allowlist only (user decision). Non-allowlisted windows keep their taskbar button. No probing. |
| ~~R9~~ | WinEvent hook / callback GC | Crash after some runtime | **Largely mitigated by the C# decision** — the delegate is held as a field on the hook class; native message pump via `HwndSource`. Still covered by the 72h soak test. |
| R10 | Browser tab matching unreliable | Wrong tab/window shown | UIA omnibox read best-effort + title glob fallback; document limitation; let the user pin a specific window. |
| R11 | Scope creep — Carousel absorbs other systems' logic (§52) | The thing it's designed not to become | Router "receives and displays, never acts" is enforced by architecture: no business logic past `EventProvider`; `WindowController` is the only side-effecting module and it only moves windows. |
| R12 | `mpk-deck` Phase 2 (Monitor Manager / Workspace Profiles) overlap | Duplicated window-placement logic | **Confirmed coexist** (user, 2026-09-01): `mpk-deck` develops independently and places its *own* Deck window; Window Carousel merely *registers the Deck window as a View*. No shared code; boundary is "who owns the placement of window X" — mpk-deck owns its own, Carousel owns the view-area slot. |

---

## 14. MVP v0.1

**Goal:** the shell works — Home renders responsively on the 3.5" display, system stats are
live, the Mini Bar navigates, and registered windows can be shown in the view area and
returned Home. **No** AI monitor, **no** detector/router, **no** external event API yet.

### In scope

- Window Carousel shell: borderless always-on-top window, PMv2 DPI. Targets the mini monitor
  by EDID/device name when present, **primary display otherwise**; re-homes on display
  hot-plug. In no-mini-monitor mode, Carousel shrinks to the Mini Bar while a view is active
  (R1b).
- Home View: Status strip + System row (CPU/GPU/RAM, 1s) + empty AI placeholders + empty
  timeline + Mini Bar. Responsive (compact/comfortable tiers). Verified at 960×640 and on the
  real panel.
- `SystemProvider` (CPU/RAM via PDH/`PerformanceCounter`, GPU + temps via
  `LibreHardwareMonitorLib`, graceful "n/a" when a sensor is missing).
- Mini Bar: nav order, current-view highlight, `+`.
- Add View: `EnumWindows` list with DWM thumbnails → create matcher → minimize → Home.
- `ViewProvider` + Window Matcher (exe / class / title / regex; no URL yet).
- `WindowController`: restore, `SetWindowPos` to view rect, restore-on-exit, ex-style
  toggle (best-effort taskbar hide).
- View registry with persistent/temporary flag; `views.json` persistence; reconnect scan
  (timer + `EVENT_OBJECT_SHOW` hook).
- Multi-monitor aware; picks the smallest / user-designated display for itself.
- Home-return on: manual nav, view window not found, any error.
- Self-resource soak check (24h minimum).

### Explicitly out (later versions)

- v0.2: `AiUsageProvider` (Claude + Codex from cache files), the status hook, AI cards
  populated, event timeline fed by AI transitions, Expanded View.
- v0.3: `EventProvider` + WS server, `Detector`s, `Router` FSM, Notification Overlay,
  AutoView + auto-return, unread, priority queue, Critical/ack, MPK Deck commands, external
  system events.

---

## 15. Implementation Plan (once approved)

**Prerequisites resolved:** real mini monitor (R1), C# / .NET 8 + WPF (§11).

### Phase 0 — Technical Validation spike (throwaway C#, ~1–2 days)

De-risk the uncertain Win32 behavior before committing to v0.1:

1. Move a Chrome window, an Electron app (VS Code), and a Win32 app (Notepad) to fill a
   secondary display precisely, at 200% DPI, then restore them. Measure glitches, focus
   behavior, fullscreen/maximized cases.
2. `SetWinEventHook` (`LOCATIONCHANGE` + `FOREGROUND` + `MINIMIZEEND` + `DESTROY` + `SHOW`)
   via `HwndSource` message pump; run 24h; confirm CPU ~0 and no handle growth.
3. `DwmRegisterThumbnail` preview of 6 windows in a grid; measure CPU/GPU.
4. `LibreHardwareMonitorLib` for CPU/GPU/RAM % + temps on the actual hardware; confirm it
   reads the MX150-class GPU; check admin-rights requirement.
5. `WS_EX_TOOLWINDOW` taskbar-hide on Chrome, VS Code, a UWP app, an Electron app — record
   which hide cleanly → seeds the known-clean allowlist (§6).
6. No-mini-monitor mode: Carousel on primary, shrink-to-Mini-Bar while a view is active (R1b).

**Output:** the known-clean allowlist seed, confirmed sensor access, and a green light on the
move/resize + hook model. No language decision — that is settled.

### Phase 1 — v0.1 (brainstorm each sub-piece per project rules, then subagent-driven-development)

1. Shell + display binding + DPI + config load.
2. `AppState` + `Provider` base + `SystemProvider` + Home System row.
3. Home layout (responsive tiers), Mini Bar (nav only).
4. `WindowController` (the deep module — most tests live here).
5. Window Matcher + `ViewProvider` + reconnect scan.
6. Add View (enum + DWM thumbnail + matcher creation flow).
7. Persistence (`views.json`, atomic write, corrupt-file recovery).
8. Home-return paths + error handling.
9. Soak test harness.

### Testing strategy

- **Unit**: Window Matcher scoring (table-driven), Router FSM transitions (v0.3),
  `AiUsageProvider` parsing against captured real cache files (v0.2), rect/DPI math.
- **Integration**: `WindowController` against real spawned windows (Notepad, a throwaway WPF
  test window) — move, verify geometry via `GetWindowRect`, restore, assert.
- **Manual / live**: the 3.5" panel at its real resolution + 960×640; focus behavior;
  fullscreen apps; browser tab matching.
- **Performance**: idle CPU sampled over 1h (assert < 1% avg); paint profiling on sparklines.
- **Long-running stability**: 72h soak — Carousel RSS, handle count, GDI objects, thread
  count logged every 5 min, asserted flat (±10%). Repeated view switches (scripted, 1000×)
  with no HWND/handle growth.

### Verification before "done" (per workspace rules)

Each phase: tests green + the specific manual check for that piece run and its output shown.
No "should work".

---

## 16. Decisions

### Resolved (user, 2026-09-01)

| # | Decision |
|---|---|
| Panel | Real 3.5" mini monitor (standard display). |
| Stack | C# / .NET 8 + WPF. |
| No-mini-monitor mode | Required — Carousel runs on the primary display, shrinks to the Mini Bar while a view is active. |
| Home layout | §4 is the development baseline, not final. Refine against the reference image later. |
| MPK Deck control | Not a design driver. NEXT / PREVIOUS view sliding at most; WS command channel stays generic. |
| Taskbar hiding | Known-clean allowlist only. |
| Repo | New `window-carousel` under `github.com/DD-PLAY-AI` (public). |

### Still open — not blocking, decide during/after Phase 0

1. **`session_name` vs. status hook for "current task"** — Claude Code already puts
   `session_name` in the statusline payload (e.g. this session is *"Window Carousel micro
   display workspace"*). Window Carousel could show that as a free "what's Claude doing"
   label with **zero new code**. The richer option is a small Claude Code hook (fires on
   `UserPromptSubmit` / `Stop` / etc.) that writes a `carousel/claude-status.json` with
   `WORKING` / `REVIEWING` / `IDLE` + the prompt text. **This is a v0.2 component of Window
   Carousel, built months from now — nothing to do in this session.** Recommendation: ship
   v0.2 with `session_name` only; add the hook in v0.3 if the status word turns out to
   matter. *(This item is just a note — no answer needed now.)*
2. **Add View picker display** — cosmetic, decide in v0.1. (Lean: on whichever display the
   Carousel window is currently on.)

### Resolved since (user, 2026-09-01)

| Item | Decision |
|---|---|
| Add trigger | `+` button (primary) + Carousel tray-menu "Add foreground window". Optional global hotkey. **Not** the Windows taskbar (not extensible). |
| Auto-return default | **12 s** global default, per-view override. |
| Repo | **Created now**: `github.com/DD-PLAY-AI/window-carousel` (public, MIT), cloned to `C:\DC\DD\window-carousel\`, seeded with README + this design doc. No app code. |

---

## Appendix A — Confirmed facts about the local environment (2026-09-01)

- Claude Code **v2.1.252** emits `rate_limits.five_hour` / `.seven_day`
  (`used_percentage`, `resets_at`) on statusline stdin. `context_window` includes
  `context_window_size: 1000000` (Sonnet 5, 1M), `used_percentage`, `current_usage`,
  `prompt_cache` stats, `session_name`, `thinking.enabled`, `effort.level`.
- `~/.claude/statusline.ps1` already persists `last-rate-limits.json` +
  `last-statusline-input.json`, refreshes `ccusage_monthly_cache.json` and
  `ccusage_codex_monthly_cache.json` (5-min cache), and triggers
  `~/.claude/scripts/codex-usage.ps1` → `codex-usage-cache.json` (5-min cache).
- `codex-usage-cache.json` shape: `{ updated, five_hour:{used_percent,resets_at},
  seven_day:{used_percent,resets_at} }`.
- ⇒ Window Carousel's AI monitor is a **file reader + watcher**, not an API client.

## Appendix B — Sources

- YASB Reborn — https://github.com/amnweb/yasb , https://deepwiki.com/amnweb/yasb , https://yasb.dev/
- GlazeWM — https://github.com/glzr-io/glazewm , https://deepwiki.com/glzr-io/glazewm , https://github.com/glzr-io/glazewm-js
- Zebar — https://github.com/glzr-io/zebar , https://deepwiki.com/glzr-io/zebar
- Komorebi — https://github.com/LGUG2Z/komorebi , https://deepwiki.com/LGUG2Z/komorebi
- InfoPanel — https://github.com/habibrehmansg/infopanel , https://infopanel.net/
- LibreHardwareMonitor — https://librehardwaremonitor.com/
- ccusage — https://github.com/ryoppippi/ccusage (via user's statusline.ps1)
- claude-code-statusline — https://github.com/ohugonnot/claude-code-statusline
- Claude Code statusline gotchas — https://gist.github.com/jtbr/4f99671d1cee06b44106456958caba8b
- Claude Code usage feature requests — https://github.com/anthropics/claude-code/issues/48660 , https://github.com/anthropics/claude-code/issues/38249
- claude-codex-usage-dashboard — https://github.com/frankchiu-dev/claude-codex-usage-dashboard
- codex-cli-usage — https://github.com/wakamex/codex-cli-usage
- Codex rate-limit detail request — https://github.com/openai/codex/issues/29618
- SetParent docs — https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setparent
- Embedding/reparenting problems — https://learn.microsoft.com/en-us/answers/questions/5867687/ , https://learn.microsoft.com/en-us/answers/questions/54675/
- DWM Thumbnail Overview — https://learn.microsoft.com/en-us/windows/win32/dwm/thumbnail-ovw
- Hide from taskbar — https://learn.microsoft.com/en-us/archive/blogs/gsmyth/hidden-vs-minimized (WS_EX_TOOLWINDOW / ITaskbarList)
- Monitoring another window (SetWinEventHook) — https://devblogs.microsoft.com/oldnewthing/20210104-00/?p=104656
- WINEVENTPROC / hook docs — https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wineventproc
- Second-monitor taskbars — TaskbarXI, Dual Monitor Taskbar, PowerToys Command Palette Dock (https://learn.microsoft.com/windows/powertoys/)
- GlazeWM IPC (query + subscribe over localhost WS) — https://deepwiki.com/glzr-io/glazewm , https://www.npmjs.com/package/glazewm
