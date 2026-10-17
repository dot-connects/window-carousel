# Window Carousel

Windows-only **Viewer / Detector / Router** for a 3.5" secondary "mini monitor".

Window Carousel turns a small secondary display into a self-contained micro workspace. It
replaces the Windows taskbar *for a registered set of windows*, shows a System / AI **Home
dashboard** by default, and briefly swaps to a relevant window when an event fires — then
returns Home.

- **View** — show a registered program / browser window in a fixed view area on the mini
  monitor (by moving the real window, not embedding or screen-capturing it).
- **Detect** — observe registered apps, AI (Claude Code / Codex) status and usage, system
  state, and events from other local systems.
- **Route** — on an event, show the relevant view for a while, then auto-return Home.

It does **not** absorb other systems' functionality. It receives and displays; it never runs
their business logic.

## Status

**Design phase. No application code yet.** This project is deferred until the sibling repos
(`mpk-deck`, `ai-hub`, `content-system`, `trading-system`) reach their current milestones.

The full research report and design — reference-project survey, technical validation,
architecture, UI/UX, technology decision, MVP plan — is in
[`docs/superpowers/specs/2026-09-01-window-carousel-design.md`](docs/superpowers/specs/2026-09-01-window-carousel-design.md).

## Planned stack

C# / .NET 8 + WPF. Provider layer (`SystemProvider` / `AiUsageProvider` / `ViewProvider` /
`EventProvider`) → immutable `AppState` → Router state machine → `WindowController` (the only
component that touches foreign windows). Event transport: a localhost WebSocket server.

Part of [D:OT](https://github.com/dot-connects).
