# make-herdr-cbc

An agent skill that walks through building **herdr-cbc**: a patched [Herdr](https://github.com/herdrdev/herdr) binary that recognizes Tencent **CodeBuddy Code** (`cbc`) as a first-class agent kind.

Upstream Herdr ships a fixed `Agent` enum with no CodeBuddy variant. As a result a pane running `cbc` is invisible to `herdr agent list`, and `herdr agent prompt` refuses to drive it (`agent_not_ready` / *"is no longer the pane foreground process"*). Hook-based state reporting only fixes half of it: the agent's state shows up, but input still cannot be delivered, so cross-pane automation degrades to raw `pane send-text` + `send-keys enter`.

This skill closes that gap at the source level.

## What it produces

| Artifact | Location (example run) |
|---|---|
| Patched source tree | `~/pxy/tools/herdr/src`, branch `feat/codebuddy-kind` |
| Standalone binary | `~/.local/bin/herdr-0.9.5-cbc` |
| Patch file | `0001-feat-detect-recognize-CodeBuddy-Code-cbc-as-a-built-.patch` |
| Published fork | <https://github.com/aragornpangit/herdr-cbc> |

The system binary `~/.local/bin/herdr` is never touched, so a live Herdr session keeps running undisturbed.

## What the patch changes

Four files, ~90 lines:

1. `src/detect/mod.rs` — adds `Agent::Codebuddy` to the enum and to `ALL` / `SCREEN_MANIFEST_AGENTS`; maps label `codebuddy`, executable `cbc`, aliases `codebuddy` / `codebuddy-code` / `cbc`; teaches the node-wrapper branch to unwrap `node .../bin/cbc`.
2. `src/detect/manifests/codebuddy.toml` — new screen manifest with `blocked` / `working` / `idle` rules, captured from real CodeBuddy Code v2.172.1 screens.
3. `src/detect/manifest.rs` — registers the manifest in `BUNDLED_MANIFESTS`.
4. `src/config/sound.rs` — routes the new kind to the default sound override.

## Requirements

- Rust toolchain matching the cloned repo's `rust-toolchain.toml` (1.96.1 for Herdr 0.9.5)
- `cbc` installed and on `PATH`
- Network access to `github.com`
- A running Herdr session to capture screen samples from (optional, but the manifest regexes are derived from real screens)

## Quick start

```bash
git clone --depth 1 https://github.com/herdrdev/herdr.git ~/pxy/tools/herdr/src
cd ~/pxy/tools/herdr/src

# apply the changes described in SKILL.md (or cherry-pick the patch in patches/)
git am ~/pxy/tools/herdr/patches/0001-*.patch

cargo check --offline
cargo test --offline --bin herdr detect::
cargo build --offline --release --bin herdr
install -m 0755 target/release/herdr ~/.local/bin/herdr-0.9.5-cbc
```

## Verifying without touching the live session

Start a nested Herdr session with hook reporting disabled, so detection can only come from the patched code:

```bash
herdr pane split --current --direction right --cwd "$PWD" --no-focus   # -> w1:p8
herdr pane run w1:p8 "HERDR_ENV=0 herdr-0.9.5-cbc --session cbctest"
sleep 10

S=$HOME/.config/herdr/sessions/cbctest/herdr.sock
HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc pane run w1:p1 "HERDR_ENV=0 cbc"
sleep 14

HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc agent list            # expect agent=codebuddy, idle
HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc agent prompt w1:p1 "reply PONG" --wait --timeout 60000
```

Expected: `agent_prompted`, and `PONG` appears in the pane. A second cbc pane behaves the same way, so pane-to-pane automation works in both directions.

Manifests can also be checked offline, no session needed:

```bash
./target/release/herdr agent explain --file /tmp/cbc_idle.txt --agent codebuddy
```

## Gotchas

- **TOML regex escaping** — use literal strings `'^\\s*>'`; a doubled backslash silently matches nothing.
- **Shallow clones cannot be pushed** — `git push` fails with *"did not receive expected object"*; run `git fetch --unshallow` first.
- **Nested Herdr is blocked when `HERDR_ENV=1`** — always verify with `HERDR_ENV=0`.
- **`cargo test --lib` tests the wrong crate** — the main crate is a binary; use `--bin herdr`.
- **A bundled manifest alone is not enough** — older builds contain a `codebuddy` manifest string but still cannot detect the process; `detect/mod.rs` must change too.

## Repository layout

```
make-herdr-cbc/
├── SKILL.md                       # the skill itself (Chinese), step-by-step runbook
├── README.md                      # this file
└── patches/
    └── 0001-*.patch               # the reference patch, applies to Herdr 0.9.5
```

## Status

Verified against Herdr **0.9.5** and CodeBuddy Code **v2.172.1** on Linux, October 2026: `cargo test --bin herdr detect::` green, native detection and `agent prompt` confirmed in an isolated nested session with hooks disabled.

Herdr is distributed under its own license; this repository only carries the patch and the procedure, not upstream sources.
