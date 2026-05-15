# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Products

Two products in one repo:

### ClifPad (`clif-pad-ide/`)
Desktop AI-native code editor — Tauri 2 (Rust) + SolidJS + Monaco Editor.

```
cd clif-pad-ide && npm install && npm run tauri dev
```

- `src/` — SolidJS frontend (components, stores, lib/tauri.ts IPC wrappers)
- `src-tauri/src/` — Rust backend (commands/, services/, state.rs)
- `src-tauri/www/` — Landing page (clifcode.io, Vercel)
- `src-tauri/scripts/` — Version bump script (bump-version.js)

**SolidJS conventions**: `class=` not `className=`, stores in `src/stores/`, Tauri commands registered in `lib.rs`.

### ClifCode (`clif-code-tui/`)
TUI terminal agent — pure Rust binary.

```
cd clif-code-tui && cargo run --release
```

- `src/main.rs` — CLI entry, TUI loop, agent orchestration
- `src/backend.rs` — OpenRouter/OpenAI/Ollama API client
- `src/tools.rs` — Tool definitions and execution (read_file, write_file, edit_file, search, etc.)
- `src/ui.rs` — Terminal UI rendering (crossterm)
- `src/session.rs` — Session persistence (auto-save conversations)
- `src/config.rs` — API keys, provider setup
- `src/git.rs` — Git integration (commit on task completion)
- `src/repomap.rs` — Workspace structure analysis
- `npm/clifcode/` — Main npm wrapper package (`npm i -g clifcode`)
- `npm/@clifcode/cli-*/` — Platform-specific binary packages
- `scripts/bump-version.mjs` — Syncs version across Cargo.toml + npm packages

## Commands

```bash
# ClifPad
cd clif-pad-ide
npm run dev              # Vite dev server (frontend only)
npm run tauri dev        # Full Tauri dev (Rust + frontend)
npm run build            # TypeScript compile + Vite production build
npm run test             # Vitest unit tests
npm run test:watch       # Vitest watch mode

# ClifCode
cd clif-code-tui
cargo run                # Debug build
cargo run --release      # Optimized release build (LTO, opt-level=3)
```

Version bump: `node scripts/bump-version.js <version>` (ClifPad) / `node scripts/bump-version.mjs <version>` (ClifCode).

## Architecture

```
ClifPad:   Tauri 2 (Rust) ──IPC──> SolidJS + Monaco + xterm.js + @xterm/*
ClifCode:  Pure Rust binary ──ureq streaming──> crossterm terminal UI
```

**ClifPad Rust backend** — modular commands under `src-tauri/src/commands/`: `fs`, `git`, `ai`, `ai_provider`, `pty`, `agent`, `security`, `indexer`, `lsp`, `search`, `gh`, `review`, `sync`, `window`. Services under `services/`: `file_watcher`. State managed via Tauri's `manage()` with `PtyState`, `LspState`, `WatcherState`. AI communication uses Tauri events (`ai_stream`, `claude-code-output`).

**ClifCode AI flow**: user prompt → tool-calling loop (read/write/edit/search/run) → streaming LLM responses → crossterm UI. 3-tier context compaction for long tasks. Auto-commits on task completion with undo via `/undo`.

## Versioning & Release

Conventional commits: `feat:` bumps minor, `fix:` bumps patch. Semantic release in `clif-pad-ide/` triggers on push to main, publishing both products. `release.yml` builds ClifPad (macOS Apple Silicon + Intel, Windows, Linux) and ClifCode (5 platforms), then publishes to crates.io. `clifcode-release.yml` is a separate workflow triggered by `clifcode-v*` tags for independent ClifCode releases + npm publish.