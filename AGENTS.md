# OpenCode Agent Instructions — Zed Editor

> **Project:** Zed — high-performance, multiplayer code editor
> **Language:** Rust (edition 2024, toolchain 1.97.1)
> **License:** GPL-3.0-or-later
> **Repository:** https://github.com/zed-industries/zed

---

## Table of Contents

1. [Coding Guidelines](#coding-guidelines)
2. [Project Architecture](#project-architecture)
3. [Key Crates Map](#key-crates-map)
4. [GPUI Framework](#gpui-framework)
5. [Build System](#build-system)
6. [Testing](#testing)
7. [Linting & Code Quality](#linting--code-quality)
8. [CI/CD](#cicd)
9. [Agent Skills](#agent-skills)
10. [Crash Investigation](#crash-investigation)
11. [Pull Request Hygiene](#pull-request-hygiene)
12. [Important Files & Paths](#important-files--paths)

---

## Coding Guidelines

- Prioritize code correctness and clarity. Speed and efficiency are secondary unless specified.
- Do not write comments that summarize code. Comments should only explain "why" for non-obvious reasons.
- Prefer implementing in existing files unless it's a new logical component. Avoid many small files.
- Avoid `unwrap()` — use `?` to propagate errors.
- Be careful with indexing — may panic if out of bounds.
- Never silently discard errors with `let _ =` on fallible operations:
  - Propagate with `?` when the caller should handle them
  - Use `.log_err()` when ignoring but wanting visibility
  - Use `match` or `if let Err(...)` for custom logic
- When implementing async operations that may fail, ensure errors propagate to the UI layer.
- Never create files with `mod.rs` paths — use `src/some_module.rs`.
- When creating new crates, specify `[lib] path = "..."` in Cargo.toml instead of default `lib.rs`.
- Use full words for variable names (no abbreviations).
- Use variable shadowing to scope clones in async contexts.
- Prefer GPUI executor timers in tests over `smol::Timer::after(...)`.

---

## Project Architecture

### Top-Level Structure

```
zed-main/
  Cargo.toml          — Workspace root (244+ crates)
  crates/             — All Zed source code
  extensions/         — Built-in extensions (glsl, html, proto, test-extension)
  docs/               — User-facing documentation (mdBook)
  tooling/            — Developer tooling (lints, compliance, perf, xtask)
  script/             — Build/CI scripts (111+)
  assets/             — Static assets (icons, fonts, themes, settings, keymaps)
  .agents/            — Agent skill definitions
  .rules              — Coding guidelines for AI agents
```

### Workspace Layout

The workspace has **244+ member crates** under `crates/`, plus 4 extension crates and 3 tooling crates. Default member: `crates/zed`.

### Architecture Overview

1. **GPUI Framework** — Custom GPU-accelerated UI framework (immediate + retained mode)
2. **Text/Buffer Layer** — CRDT-like text editing on rope/sum-tree data structures
3. **Language Layer** — Tree-sitter syntax highlighting, LSP client
4. **Editor Core** — Main text editor component
5. **Project/Workspace** — File management, workspace layout
6. **AI/Agent System** — Multi-provider LLM integration (Anthropic, OpenAI, Google, Ollama, etc.)
7. **Edit Prediction** — Ghost-text/inline completion system
8. **Git** — Git integration and UI
9. **Debugger** — DAP-based debugging
10. **Extensions** — WASM-based extension system
11. **Collaboration** — Real-time collaboration via WebSocket + LiveKit
12. **Remote Editing** — SSH-based remote development

---

## Key Crates Map

| Crate | Path | Role |
|-------|------|------|
| `zed` | `crates/zed/` | Main entry point, wires everything together |
| `gpui` | `crates/gpui/` | GPU-accelerated UI framework (the foundation) |
| `editor` | `crates/editor/` | Core `Editor` type for code editing |
| `project` | `crates/project/` | File management, navigation, LSP communication |
| `workspace` | `crates/workspace/` | Local state, project grouping |
| `vim` | `crates/vim/` | Vim workflow over `editor` |
| `lsp` | `crates/lsp/` | LSP client implementation |
| `language` | `crates/language/` | Language understanding (symbols, syntax maps) |
| `collab` | `crates/collab/` | Collaboration server |
| `rpc` | `crates/rpc/` | Client-server protocol (protobuf) |
| `theme` | `crates/theme/` | Theme system and default themes |
| `ui` | `crates/ui/` | Reusable UI components |
| `cli` | `crates/cli/` | CLI binary |
| `settings` | `crates/settings/` | Settings system |
| `fs` | `crates/fs/` | Filesystem abstraction |
| `worktree` | `crates/worktree/` | File tree representation |
| `multi_buffer` | `crates/multi_buffer/` | Multi-buffer editing |
| `agent` | `crates/agent/` | Core AI agent logic |
| `git` | `crates/git/` | Git data model and operations |
| `dap` | `crates/dap/` | Debug Adapter Protocol client |
| `extension` | `crates/extension/` | Core extension types |
| `extension_api` | `crates/extension_api/` | Public WASM API for extensions |

---

## GPUI Framework

GPUI is the foundation. All UI and entity management goes through it.

### Context Types

- `App` — Root context, holds all state. Not `Send` (main thread only).
- `Context<T>` — Wraps `App` for `Entity<T>`. Derefs to `App`.
- `AsyncApp` / `AsyncWindowContext` — Owned async contexts for across `.await` points.
- `Window` — Application window. Passed as `window` before `cx`. Used for focus, actions, drawing, input.

### Entities

`Entity<T>` is a handle to state. With `thing: Entity<T>`:
- `thing.read(cx)` — Read state
- `thing.update(cx, |state, cx| ...)` — Mutate state with context
- `thing.update_in(cx, |state, window, cx| ...)` — Same with window access
- `thing.downgrade()` — `WeakEntity<T>` (returns `Result` on all methods)

### Concurrency

- All entity access and UI rendering on **single foreground thread**.
- `cx.spawn(async |cx| ...)` — Foreground async task
- `cx.spawn(async |this, cx| ...)` — When cx is `Context<T>` (this: `WeakEntity<T>`)
- `cx.background_spawn(async { ... })` — Background thread work
- `Task<R>` — Future that starts immediately. Dropping cancels it. Must `.await`, `.detach()`, or store it.

### Rendering

- `Render` trait: `fn render(&mut self, window, cx) -> impl IntoElement`
- `RenderOnce` trait: for components constructed just to become elements
- Tailwind-like style API: `div()`, `.border_1()`, `.child()`, etc.
- `SharedString` — `&'static str` or `Arc<str>` (avoids copying)
- Conditional rendering: `.when(condition, |this| ...)`, `.when_some(option, |this, val| ...)`

### Actions

- Actions dispatched via keyboard or `window.dispatch_action(action.boxed_clone(), cx)`
- Unit struct actions: `actions!(namespace, [Action1, Action2])`
- Complex actions: `#[derive(Action)]`
- Handlers: `.on_action(|action, window, cx| ...)`
- Entity-aware: `.on_action(cx.listener(|this, action, window, cx| ...))`

### Notification & Events

- `cx.notify()` — Re-render view, trigger observers
- `cx.emit(event)` — Typed event emission (requires `impl EventEmitter<T>`)
- `cx.subscribe(entity, |this, entity, event, cx| ...)` — Handle emitted events
- `cx.observe(entity, |this, entity, cx| ...)` — React to entity changes
- Returns `Subscription` (auto-deregisters on drop)

---

## Build System

### Rust Toolchain

- **Channel:** 1.97.1 (stable, minimal profile)
- **Components:** rustfmt, clippy, rust-analyzer, rust-src
- **Targets:** `wasm32-wasip2`, `wasm32-unknown-unknown`, `x86_64-unknown-linux-musl`
- **Config:** `rust-toolchain.toml`

### Build Profiles

| Profile | Features |
|---------|----------|
| `dev` | incremental, codegen-units=16, limited debug, opt-level=3 for proc-macros |
| `dbg` | Inherits dev, debug=full |
| `release` | lto=thin, codegen-units=1, limited debug |
| `release-fast` | Inherits release, no LTO, full debug, codegen-units=16 |

### Key Build Commands

```bash
cargo build                                    # Default (crates/zed)
cargo build --release                          # Release build
cargo build --release --package collab         # Collab server
cargo build -p zed --profile release-fast      # Fast release
```

### Cargo Aliases

```bash
cargo xtask <cmd>                              # Custom xtask commands
cargo perf-test                                # Release-fast test with perf
cargo perf-compare                             # Release-fast perf compare
```

### Corgi Build Sandbox

Build scripts and proc-macros run in a sandbox with no network and restricted filesystem. Pinned tools: CMake 4.1.2, prebuilt libwebrtc.

---

## Testing

### Test Runner

Uses **cargo-nextest** as primary test runner.

```bash
cargo nextest run --workspace --no-fail-fast                    # All tests
cargo nextest run --package collab                             # Specific package
cargo nextest run -E 'test(test_name)'                         # Specific test
```

### GPUI Tests

Use `#[gpui::test]` for tests needing `TestAppContext`, deterministic executors, or scheduler coverage.

**Supported arguments:** `&TestAppContext`, `&mut TestAppContext`, `StdRng`, `BackgroundExecutor`, `&App`, `&mut App`

**Attribute arguments:**
- `iterations = N` — Run sequential seeds starting at 0
- `seed = N` / `seeds(...)` — Explicit seeds
- `retries = N` — Retry failing runs
- `on_failure = "path::to::function"` — Call on final failure

### Reproducing GPUI Test Failures

```bash
# Basic run
cargo -q test -p <crate> <test_name> -- --nocapture

# With failing seed
SEED=<seed> cargo -q test -p <crate> <test_name> -- --nocapture

# Sweep seeds for flaky tests
ITERATIONS=100 cargo -q test -p <crate> <test_name> -- --nocapture

# Debug parking failures
PENDING_TRACES=1 cargo -q test -p <crate> <test_name> -- --nocapture
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `SEED=<u64>` | Scheduler seed for reproducibility |
| `ITERATIONS=<usize>` | Override iterations count |
| `PENDING_TRACES=1` | Print pending task traces on panic |
| `DEBUG_SCHEDULER=1` | Print scheduler clock/timer debug |
| `GPUI_RUN_UNTIL_PARKED_LOG=1` | Log parking when `allow_parking()` enabled |
| `RUST_BACKTRACE=1` | Show panic backtraces |
| `ZED_HEADLESS=1` | Force headless platform mode |

### Test Contexts

- `TestAppContext` (gpui) — Base test context
- `VisualTestContext` — Window-aware test context
- `EditorTestContext` — Editor-specific setup with FakeFs, Project, Buffer
- `EditorLspTestContext` — Editor tests with LSP mock
- `VimTestContext` — Vim-specific test setup

### Visual Regression Tests (macOS only)

```bash
cargo build -p zed --bin zed_visual_test_runner --features visual-tests
cargo run -p zed --bin zed_visual_test_runner --features visual-tests
UPDATE_BASELINES=1 cargo run -p zed --bin zed_visual_test_runner --features visual-tests
```

### Doc Tests

```bash
cargo test --workspace --doc --no-fail-fast
```

### Randomized/Chaos Testing

```bash
# CI: 50,000 iterations, 200 operations
script/randomized-test-ci
```

---

## Linting & Code Quality

### Clippy

```bash
./script/clippy                                  # Full workspace clippy + extra checks
script\clippy.ps1                                # Windows
cargo xtask clippy                               # Via xtask
cargo xtask clippy --fix                         # Auto-fix mode
cargo xtask clippy --package <crate>             # Single package
```

**Workspace clippy rules (deny):** `dbg_macro`, `todo`, `declare_interior_mutable_const`, `redundant_clone`, `disallowed_methods`

**Disallowed methods:**
- `std::process::Command::spawn/output/status` → use `smol::process::Command`
- `smol::Timer::after` in tests → use `gpui::BackgroundExecutor::timer`
- `serde_json::from_reader` → use `from_slice`
- `cocoa::foundation::NSString::alloc` → use `ns_string()`

### Formatting

```bash
cargo fmt --all -- --check                       # Check formatting
cargo fmt --all                                 # Auto-format
```

**Config:** `rustfmt.toml` — edition 2024, style_edition 2024

### Custom Dylint Lints

```bash
cargo dylint --all -- --workspace                # All custom lints
tooling/lints/single-lint <lint_name> -p <crate> # Single lint
tooling/lints/single-lint <lint_name>            # All packages
```

**Current lints:**
- `SHARED_STRING_FROM_STR_LITERAL` — Use `SharedString::from_static`
- `ASYNC_BLOCK_WITHOUT_AWAIT` — Async blocks with no `.await`
- `BLOCKING_IO_ON_FOREGROUND` — Blocking IO on main thread
- `ENTITY_UPDATE_IN_RENDER` — Mutating entity inside `Render::render`
- `MAP_LOOKUP_THEN_INSERT` — Redundant map lookups
- `NOTIFY_IN_RENDER` — `cx.notify()` inside render
- `OWNED_STRING_INTO_SHARED` — String literal to SharedString patterns

**Lint crate:** `tooling/lints/` (pins nightly-2026-03-21, separate from main workspace)

### Other Quality Checks

```bash
typos --config typos.toml                        # Spell checking
script/shellcheck-scripts                        # Shell script linting
script/check-todos                               # Check for TODO markers
script/check-keymaps                             # Validate keymaps
script/check-licenses                            # License compliance
script/check-links                               # Documentation links
script/prettier                                  # Prettier on settings + docs
```

---

## CI/CD

### GitHub Actions Workflows (47 total)

All workflows are **auto-generated** via `cargo xtask workflows` from Rust source code.

**Key workflows:**
- `run_tests.yml` — Smart filtered testing (only affected packages), full on main/stable/preview
- `release.yml` — Triggered by `v*` tags, builds for all platforms
- `release_nightly.yml` — Every 4 hours, bundles for Linux/macOS/Windows
- `run_bundling.yml` — On PR labeled `run-bundling`
- `deploy_collab.yml` — Collab server deployment to Kubernetes
- `nix_build.yml` — Nix flake builds
- `extension_tests.yml` — Extension-specific testing (WASM targets)
- `autofix_pr.yml` — Auto-fix formatting/linting

### CI Environment

- `CARGO_TERM_COLOR=always`
- `RUST_BACKTRACE=1`
- `CARGO_INCREMENTAL=0`

### Xtask Commands

```bash
cargo xtask clippy                              # Clippy check
cargo xtask compliance version <tag>            # PR compliance
cargo xtask licenses                             # License check
cargo xtask package-conformity                   # Package standards
cargo xtask publish-gpui                         # Publish GPUI to crates.io
cargo xtask sandbox-tests                        # Linux Bubblewrap tests
cargo xtask wsl-sandbox-tests                    # Windows WSL sandbox tests
cargo xtask setup-webrtc                         # Download WebRTC deps
cargo xtask web-examples                         # Build/serve WASM examples
cargo xtask workflows                            # Regenerate CI workflows
cargo xtask check-workflows                      # Validate workflows
```

### Docker

```bash
docker build -f Dockerfile-collab .              # Collab server image
docker compose up                                # Local dev (MinIO + LiveKit)
```

---

## Agent Skills

Skills are defined in `.agents/skills/`. Available skills:

### gpui-bench
**Path:** `.agents/skills/gpui-bench/SKILL.md`
Design, write, review, run, and interpret GPUI Criterion benchmarks. Covers `#[gpui::bench]`, `BenchAppContext`, headless rendering, responsiveness benchmarks, profiling with Instruments/xctrace, and before/after methodology.

**Key rules:**
- Never enable `test-support` feature in benchmarks
- Use `bench-support` feature for benchmark infrastructure
- Every benchmark invocation must have a 5-minute timeout
- Run smoke modes before measured runs
- Do not run measured benchmarks concurrently

### gpui-test
**Path:** `.agents/skills/gpui-test/SKILL.md`
Write, debug, and reproduce GPUI tests. Covers `#[gpui::test]` arguments, `TestAppContext`, scheduler seeds, `ITERATIONS/SEED` reproduction, parking failures, and pending task traces.

### zed-cherry-pick
**Path:** `.agents/skills/zed-cherry-pick/SKILL.md`
Cherry-pick merged PRs to `preview` or `stable` release branches. Covers branch discovery, conflict resolution, validation, and PR creation.

### lint-creator
**Path:** `.agents/skills/lint-creator/SKILL.md`
Add, modify, or test custom dylint lints in `tooling/lints`. Covers module layout, UI test conventions, running tests with pinned nightly, and validating against the real codebase.

---

## Crash Investigation

### Sentry Integration

- Crash investigation prompts: `.factory/prompts/crash/investigate.md`
- Crash fix prompts: `.factory/prompts/crash/fix.md`
- Fetch crash reports: `script/sentry-fetch <issue-id>`
- Generate investigation prompt: `script/crash-to-prompt <issue-id>`

---

## Pull Request Hygiene

When opening or updating a PR:

- Use clear, correctly capitalized, imperative title (e.g., `Fix crash in project panel`)
- Avoid conventional commit prefixes (`fix:`, `feat:`, `docs:`, etc.)
- Avoid trailing punctuation
- Optionally prefix with crate name when one crate is the clear scope (e.g., `git_ui: Add history view`)
- Include `Release Notes:` as the final section in the PR body
- Use one bullet under `Release Notes:`:
  - `- Added ...`, `- Fixed ...`, or `- Improved ...` for user-facing changes
  - `- N/A` for docs-only and non-user-facing changes
- Format exactly with a blank line after heading:

```
Release Notes:

- N/A
```

---

## Important Files & Paths

### Configuration
| File | Purpose |
|------|---------|
| `Cargo.toml` | Workspace manifest (244+ crates, profiles, lints) |
| `rust-toolchain.toml` | Rust 1.97.1, components, cross targets |
| `rustfmt.toml` | Formatting config (edition 2024) |
| `clippy.toml` | Clippy config (disallowed methods) |
| `.cargo/config.toml` | Main cargo config (aliases, flags) |
| `.cargo/ci-config.toml` | CI overlay (-D warnings) |
| `.config/nextest.toml` | Test runner config (timeouts, ordering) |

### Source Code
| Path | Purpose |
|------|---------|
| `crates/zed/src/main.rs` | Application entry point |
| `crates/zed/src/zed.rs` | Core Zed module (8300+ lines) |
| `crates/gpui/src/app.rs` | App, Context types, entity management |
| `crates/gpui/src/window.rs` | Window management |
| `crates/editor/src/editor.rs` | Core Editor type |
| `crates/project/src/project.rs` | Project abstraction |
| `crates/language/src/language_registry.rs` | Language/LSP registry |
| `crates/lsp/src/lsp.rs` | LSP client implementation |
| `crates/extension_api/src/extension_api.rs` | Extension WASM API |

### Assets
| Path | Purpose |
|------|---------|
| `assets/settings/default.json` | Default settings (~2420 lines) |
| `assets/keymaps/` | Platform-specific keybindings |
| `assets/themes/` | Built-in themes |
| `assets/icons/` | 300+ SVG icons |
| `assets/fonts/` | Bundled fonts (IBM Plex Sans, Lilex) |
| `assets/prompts/` | Handlebars prompt templates |

### Scripts
| Path | Purpose |
|------|---------|
| `script/clippy` | Full lint pipeline |
| `script/bundle-linux` | Linux release bundle |
| `script/bundle-mac` | macOS DMG bundle |
| `script/bundle-windows.ps1` | Windows installer |
| `script/bootstrap` | Dev environment setup |
| `script/linux` | Linux dependency installation |

### Testing
| Path | Purpose |
|------|---------|
| `crates/gpui/src/app/test_context.rs` | GPUI test context |
| `crates/editor/src/test/` | Editor test utilities |
| `crates/collab/tests/integration/` | Collaboration integration tests |
| `tooling/lints/` | Custom Dylint lints |
| `tooling/lints/ui/` | Lint UI test fixtures |

### Documentation
| Path | Purpose |
|------|---------|
| `docs/` | mdBook documentation |
| `CONTRIBUTING.md` | Contribution guidelines |
| `.rules` | AI agent coding guidelines |

---

## Rules Hygiene

The `.rules` file is read by every agent session. Keep it high-signal.

### After any agentic session
If you discover a non-obvious pattern, include a **"Suggested .rules additions"** heading in your PR description with proposed text. Do **not** edit `.rules` inline during normal work.

### High bar for new rules
New rules must meet **all three** criteria:
1. **Non-obvious** — someone familiar with the codebase would still get it wrong
2. **Repeatedly encountered** — came up more than once
3. **Specific enough to act on** — concrete instruction, not vague principle

Rules for a single crate belong in that crate's own `.rules` file.

### What NOT to put in `.rules`
Avoid architectural descriptions (module layout, data flow, key types). These go stale fast. Rules should be **traps to avoid**, not **maps to follow**.

---

*This file is maintained by OpenCode agents. Last updated: 2026-09-08*
