# Kanban

A single HTML file. No build step. No dependencies. No server.
Just you and your tasks, synchronized across devices.

## Philosophy

> "Stop starting, start finishing."

Kanban emerged from Toyota's manufacturing system in the 1940s. The core insight: **limit work in progress**. When you cap how much you're doing at once, you finish faster, see blockers sooner, and reduce context-switching overhead.

This board enforces that discipline. The **Now** column has a WIP limit. When you exceed it, the header turns red. Not as punishment, but as information. The system is telling you: something needs to finish before something new can start.

### The Two Gates

```
┌───────┐      ┌───────┐      ┌───────┐      ┌───────┐      ┌───────┐
│ Inbox │  ›   │  Now  │  ‹   │ Next  │  →   │ Later │  →   │ Done  │
│capture│ PUSH │ focus │ PULL │planned│      │someday│      │archive│
└───────┘      └───────┘      └───────┘      └───────┘      └───────┘
```

**Push Gate (Inbox → Now):** Before pushing a task into active work, pause. Is this truly the next most important thing? Friction here prevents impulse-driven work.

**Pull Gate (Next → Now):** When capacity frees up, pull work from the planned queue. This is proactive, intentional selection—not reactive task-switching.

## Use

### Priority System

| Priority | Color | Use Case |
|----------|-------|----------|
| `CRITICAL` | Red | Deadlines today, blocking issues |
| `HIGH` | Orange | Important, time-sensitive |
| `MEDIUM` | Yellow | Standard work |
| `LOW` | Blue | Nice to have |
| `SOMEDAY` | Gray | Maybe later |

Cards are sorted by priority automatically. Critical items float to the top. Someday items fade into the background.

### Effort Sizing

Each card can be sized: `15m`, `1h`, `4h`, `1d`, `3d`, `1w`. Larger efforts get physically larger cards. A week-long project should feel heavy when you look at it.

### Keyboard Shortcuts

| Category | Keys | Action |
|----------|------|--------|
| **Navigate** | `h` `j` `k` `l` | Vim-style movement |
| | `gg` | Go to first card |
| | `G` | Go to last card |
| | `g` + `i/n/x/l/d` | Jump to column |
| **Act** | `a` | Add card |
| | `Enter` | Edit card |
| | `Space` | Move card right |
| | `Shift+Space` | Move card left |
| | `Backspace` | Delete card |
| **Assign** | `p` + `c/h/m/l/s` | Set priority |
| | `e` + `1-6` | Set effort (15m→1w) |
| **Filter** | `f` + `c/h/m/l/s` | Filter by priority |
| | `ff` | Clear filters |
| **Sync** | `s` | Open settings |
| | `Ctrl+E` | Export JSON |
| | `Ctrl+I` | Import JSON |

## Design

### Flexoki Theme

Colors from [Steph Ango's Flexoki](https://stephango.com/flexoki)—an inky palette designed for long reading sessions. Low contrast where it matters, high contrast where it counts. Automatic dark mode.

### Typography

IBM Plex Mono throughout. Monospace isn't just aesthetic—it's functional. Characters align. Columns align. Your eye can scan vertically without snagging.

### Information Density

Everything visible at once. No modals (except help and settings). No pagination. No infinite scroll. The entire state of your work fits on one screen.

## Infrastructure

| Layer | Implementation |
|-------|----------------|
| **Frontend** | Single HTML file, vanilla JS, no build step |
| **Hosting** | GitHub Pages, free, global CDN |
| **Storage** | localStorage (local) + GitHub Gist (sync) |
| **Auth** | Your GitHub token, stored in your browser only |

### Sync Architecture

Three tiers, gracefully degrading:

1. **File System API** — Save to Dropbox/iCloud folder. Chrome/Edge only.
2. **GitHub Gist** — Private gist in your account. Works everywhere.
3. **Manual Export** — Download/upload JSON. Universal fallback.

Sync is debounced (5 seconds after last change) and uses last-write-wins conflict resolution. Simple, predictable, sufficient for single-user workflows.

### Privacy Model

There is no server. Your data lives in:
- Your browser's localStorage
- Your GitHub account (if you enable Gist sync)
- Your cloud folder (if you use File System API)

No analytics. No tracking. No accounts. The HTML file doesn't phone home.

## Setup

### Quick Start

1. Open [xiaolong-y.github.io/kanban](https://xiaolong-y.github.io/kanban/)
2. Start using (data saves to localStorage)

### Cross-Device Sync

1. Press `s` to open settings
2. Select "GitHub Gist" as sync method
3. [Create a token](https://github.com/settings/tokens/new?scopes=gist&description=Kanban%20Sync) with `gist` scope
4. Paste token and wait for "Gist created"
5. Use same token on other devices

### Self-Host

```bash
# Clone and serve
git clone https://github.com/xiaolong-y/kanban.git
cd kanban
python -m http.server 8000
# Open http://localhost:8000
```

Or just download `index.html` and open it directly.

## License

MIT. Single file. Fork it, own it.
