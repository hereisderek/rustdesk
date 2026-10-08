# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Commands

### Rust build

Requires vcpkg with `libvpx libyuv opus aom` installed and `VCPKG_ROOT` set (see README for per-OS package prerequisites).

```sh
cargo run                        # debug build + run (Sciter UI, deprecated)
cargo build --release
cargo check -p hbb_common        # check a single workspace crate (members: libs/scrap, libs/hbb_common, libs/enigo, libs/clipboard, libs/virtual_display, libs/virtual_display/dylib, libs/portable, libs/remote_printer)
cargo test -p hbb_common         # run tests for one crate
cargo test <test_name>           # run a single test by name (matches across the workspace)
cargo clippy
```

### Flutter build (current UI)

`build.py` wraps the cargo build and the Flutter build/packaging into one step; `--print-features` shows the cargo feature set without building.

```sh
python3 build.py --flutter --hwcodec               # full Rust + Flutter build
python3 build.py --flutter --skip-cargo             # Flutter only, reuse existing Rust lib
```

Inside `flutter/`, standard Flutter commands apply: `flutter pub get`, `flutter run`, `flutter build <platform>`, `flutter test`, `flutter test test/<file>_test.dart` for a single test.

### Localization

`res/lang.py` diffs `src/lang/*.rs` translation files against each other (e.g. to find missing/stale keys). See the Localization section in AGENTS.md for the editing workflow.

## Architecture

RustDesk is a Rust core (network protocol, screen capture, input, codecs) driven by one of two UI layers, communicating over an FFI/bridge boundary rather than in-process function calls:

- **Flutter UI** (`flutter/lib/`) talks to the Rust core via `flutter_rust_bridge`. `flutter/lib/models/` mirrors Rust-side state into Dart; `desktop/`, `mobile/`, and `web/` hold platform-specific screens on top of the shared `common/` widgets.
- **Sciter UI** (`src/ui/`) is deprecated but still present; avoid extending it for new features.

Core Rust process roles, spread across `src/`:
- `src/client.rs` initiates outbound peer connections; `src/server/` (`connection.rs`, `video_service.rs`, `audio_service.rs`, `input_service.rs`, `clipboard_service.rs`, `display_service.rs`) implements the services a controlled side exposes to a controlling peer.
- `src/rendezvous_mediator.rs` handles signaling with the rendezvous/relay server (NAT traversal / hole punching, falling back to relay).
- `src/platform/` isolates OS-specific implementations (input injection, permissions, service install) behind a common interface used by `src/server/` — this is the pattern to follow when adding platform-specific behavior (see AGENTS.md's "Be minimally invasive" section).
- `src/ipc.rs` is the local IPC channel between the main app process and the elevated/background service process (`src/service.rs`, `naming.rs` binaries).

Workspace library crates under `libs/` are independently versioned/tested Cargo crates, not just modules:
- `hbb_common` — protobuf messages, config, TCP/UDP framing; nearly every other crate depends on it.
- `scrap` — screen capture + video codec backends, gated by Cargo features (`hwcodec`, `vram`, `mediacodec`, `drm`).
- `enigo` — cross-platform input simulation.
- `clipboard` — cross-platform clipboard, including file copy/paste (`unix-file-copy-paste` feature).

Cargo features (`Cargo.toml`) select platform capture/codec backends at compile time (e.g. `drm`/`drm-wake` for Linux DRM/KMS capture, `hwcodec`/`vram` for hardware codecs) — check which features a build enables before assuming a code path is compiled in.
