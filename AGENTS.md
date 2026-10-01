# AGENTS.md

## What is this

ZenX Bridge connects AI agents to a user's real Chromium browser via a local `bsk` CLI + daemon + browser extension. The repo is a **dual workspace**: Cargo (Rust) + pnpm (JS/TS).

## Structure

| Path | What it is |
|------|------------|
| `crates/bsk-cli` | `bsk` CLI binary and local daemon (Rust). Published crate name: `bsk`. |
| `crates/bsk-protocol` | Shared wire types, JSON-RPC frames, JSON schemas. `dump-schema` binary auto-generates `crates/bsk-protocol/schema/`. |
| `apps/extension` | Chromium extension (WXT + React + Tailwind v4). MV3, talks to daemon over WebSocket on `ws://127.0.0.1:52800`. |
| `packages/ui` | Shared extension UI components (shadcn-style, tailwind-merge + CVA). Consumed via source imports, not built. |
| `packages/i18n` | i18next-based i18n. Also consumed via source imports. |
| `skill/SKILL.md` | The agent skill file — copied into `crates/bsk-cli/skill/` by `build.rs` at cargo build time. Edit only the root copy. |
| `scripts/` | Node scripts for release artifacts (e.g. `render-version-json.mjs`). Tests use `node:test`. |

## Commands

### Full CI-equivalent check (what passes in CI)

**Rust:**
```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo test --workspace --locked
```

**Frontend:**
```bash
pnpm install --frozen-lockfile
pnpm --filter @browser-skill/extension exec wxt prepare   # generates .wxt/ types — required before compile/test
pnpm lint                                                  # biome check + stylelint
pnpm --filter @browser-skill/extension compile             # tsc --noEmit
pnpm ext:test                                              # vitest run (extension only)
pnpm ext:build                                             # wxt build
```

**Node scripts:**
```bash
node --test scripts/*.test.mjs
```

### Quick iteration

```bash
# Rust: build the CLI only (skips schema dump since it's already generated)
cargo build -p bsk

# Extension dev server (hot reload)
pnpm ext:dev

# Extension tests only
pnpm ext:test

# Lint only
pnpm lint
```

### Schema regeneration

The `cli:build` script regenerates protocol schemas then builds JS:
```bash
pnpm cli:build   # runs: cargo run -p bsk-protocol --bin dump-schema --locked && pnpm -r build
```

Schemas land in `crates/bsk-protocol/schema/`. These are committed and should match what `dump-schema` produces.

## Design invariants

These are the rules a change must not break. A "quirk" below is a fact about
the toolchain; an invariant is a fact about the **architecture** — breaking one
silently degrades the product, and reviewers should reject the change.

1. **The daemon is the only control plane.** CLI and any other consumer reach
   the browser through the daemon's IPC/WebSocket surface. Do not add a second
   path that talks to the extension directly.
2. **Loopback is the supported mode; upstream remote/server mode is carried but
   unsupported.** The daemon's default — and the only mode this fork supports —
   binds WS to `127.0.0.1` (`start.rs`) and IPC to a local socket/named pipe.
   Never bind a public interface and never open a listening port for "remote"
   convenience. `crates/bsk-cli/src/daemon/remote/**` and the extension's
   `src/transport/remote-*` modules are inherited from upstream and are
   **frozen**: do not extend them, do not present them as a fork capability, and
   do not adopt their future upstream changes (`docs/UPSTREAM_SYNC.md` §4). If a
   remote capability is ever needed, design it on our own terms instead of
   inheriting upstream's device-pairing and public-listener model.
3. **No telemetry, analytics, or crash reporting.** `PRIVACY.md` states it
   outright. Do not add SDKs, beacons, or usage counters.
4. **User tabs are protected by default.** Automation runs in an isolated
   Agent Window; a user-window tab is only touched after an explicit
   `tab_borrow`, and borrowed tabs must be returned. Cross-session isolation
   holds: one session never sees another session's Agent Window tabs.
5. **Borrow never happens silently.** Missing confirmation wiring must fail
   closed, not approve (see `approveBorrow` in `tools/tabs.ts`).
6. **Sessions are bounded.** `session stop` is the cleanup path; the 5-minute
   session idle reaper and the 60s browser liveness reaper are safety nets, not
   the contract. An agent must not rely on them.
7. **Reconnects are generation-guarded.** A stale socket must not clobber the
   registration of the newer connection under the same `instance_id`.
8. **Protocol compatibility is negotiated, not assumed.** Handshake compares
   `protocol_version`, and minor drift is allowed but surfaced as
   `version_skew`. Do not silently drop unknown-version peers.
9. **The generated schemas are committed truth.** `crates/bsk-protocol/schema/`
   must match `dump-schema` output; a protocol change that skips regeneration
   breaks the extension's contract.
10. **`skill/SKILL.md` is the single source of the agent instructions.** It is
    copied into the crate by `build.rs`; never edit the copy.
11. **Command availability is tiered.** Baseline / `0.2.4+` / fork-only. A new
    command must declare its tier in the docs, never imply the released binary
    has it.

## Upstream relationship

This repository is a **downstream distribution (soft fork)** of
[`Tencent/BrowserSkill`](https://github.com/Tencent/BrowserSkill). The full
procedure, the ported/not-ported lists and the current divergence numbers live
in **`docs/UPSTREAM_SYNC.md`** — read it before syncing or before assuming a
change came from upstream. The short version:

- **Keep the wire protocol compatible.** It is what makes upstream fixes cheap
  to cherry-pick. Do not diverge in `bsk-protocol` handshake semantics.
- **Take bug fixes, skip remote/server mode.** Sync by directed cherry-pick, not
  by whole-tree merge.
- **Identity is ours.** Version numbers, install URLs, `repository` fields and
  the published name are ours; a merge conflict on any of them resolves to our
  side. Version bumps are a separate release action, never part of a sync
  commit.
- **Fork-only surface stays documented.** `invoke`, `templates`, `completion`,
  `browsers close`, `browser-tabs`, `tab observe`, profile account id, the
  `since` cursor and smart labels exist only here; `zenxbrowser` and
  `browserskill-pro` depend on them and cannot run on an upstream build.

## Companion repositories

This repo is one of three. The others are not optional context — a change here
usually has a counterpart in another repo, so link them instead of duplicating.

| Repo | Local checkout | Contains |
|------|----------------|----------|
| [`916938/zenx-bridge`](https://github.com/916938/zenx-bridge) | this one (`zenx-bridge-main`) | CLI, daemon, extension, DSH plugin, the bundled `zenx-bridge` skill, and the sync record `docs/UPSTREAM_SYNC.md` |
| [`916938/zenx-bridge-skill`](https://github.com/916938/zenx-bridge-skill) | sibling dir `../zenx-bridge-skill` | The Pro skill package: helper scripts, examples, layered references, `command-registry.json`, plus the reusable sync procedure `skill/references/upstream-sync.md` and the read-only helper `scripts/upstream_sync.py` |
| [`Tencent/BrowserSkill`](https://github.com/Tencent/BrowserSkill) | `Tencent` git remote | Upstream; see the section above |

Note the naming trap: the GitHub slug is `zenx-bridge`, but local checkouts here
are named `zenx-bridge-main` (fork) and `zenx-bridge-skill` (skill package).
Do not assume a directory name matches a repo name.

Rules that keep the two distributable skills from drifting apart:

1. **Two different skills, not two copies of one file.** `skill/SKILL.md` here is
   `name: zenx-bridge` — embedded by `build.rs` (`crates/bsk-cli/src/skill_install/mod.rs`)
   and installed by `bsk install-skill`. The one in the skill repo is
   `name: zenx-bridge-skill` (Pro), installed by that repo's installer. Never
   overwrite one with the other: they differ in depth and audience, and the
   Pro package's tables are generated from its own `command-registry.json`.
2. **Fork-only surface lands in both.** Adding or changing `invoke`, `templates`,
   `completion`, `browsers close`, `--browser-id` tabs, `tab observe`, the
   `since` cursor, profile account id or smart labels requires updating
   `skill/SKILL.md` here **and** the Pro repo's `skill/SKILL.md` +
   `skill/references/command-registry.json` (then regenerate with its
   `generate_command_docs.py`).
3. **A sync batch touches both repos.** After updating the ported/skipped tables
   and the verification numbers in `docs/UPSTREAM_SYNC.md`, check whether the
   procedure, the unportable-feature-lines table or the verification matrix in
   `../zenx-bridge-skill/skill/references/upstream-sync.md` needs the same edit.
4. **Verify flags against source, never memory.** Before documenting a command,
   read `crates/bsk-cli/src/cli/*.rs` and `crates/bsk-protocol/schema/` here.

## Important quirks

- **`wxt prepare` is required** before `tsc --noEmit` or `vitest` — it generates `.wxt/` type stubs. CI always runs it; you must too.
- **`--locked` on cargo commands** — the CI enforces lockfile integrity. Always pass `--locked` to `cargo build`, `cargo test`, `cargo clippy`, etc.
- **pnpm 10.17.0** — pinned in `packageManager` field. Do not upgrade without updating the lockfile.
- **Cargo edition 2024, rust-version 1.85** — edition 2024 is new; some patterns differ from 2021 (e.g. `gen` keyword reserved, `unsafe_op_in_unsafe_fn` warn-by-default). Rust stable toolchain with `rustfmt` + `clippy` components.
- **Biome formatter only, linter disabled** — `biome.json` has `"linter": { "enabled": false }`. Formatting is enforced; linting is not.
- **Stylelint for CSS** — uses `stylelint-config-standard`. The `.stylelintrc.json` allows Tailwind directives (`@apply`, `@theme`, `@source`, `@custom-variant`).
- **Extension path aliases** — `@/` and `~/` map to `./src/`. `@browser-skill/i18n` resolves to the local package source, not a built artifact.
- **Version sources are separate** — CLI version lives in `Cargo.toml` workspace version; extension version in `apps/extension/package.json`. Release workflows verify they match the git tag. When bumping, update both independently.
- **Release tags** — CLI releases use `cli-v*` tags; extension releases use `ext-v*` tags.
- **`skill/SKILL.md` is the single source of truth** — the `build.rs` in `bsk-cli` copies it into the crate during cargo build. Do not edit `crates/bsk-cli/skill/SKILL.md` directly.
- **Extension tests use `happy-dom`** (not jsdom). Test files live alongside source as `*.test.ts` / `*.test.tsx`.
- **`AGENT_INSTALL.md`** is an agent-facing install guide. It says "never use sudo" and defines "done = `bsk doctor` passes."

## Style conventions

- Rust: 4-space indent, 100 char max width (`rustfmt.toml`), clippy cognitive complexity threshold 30.
- JS/TS/CSS/MD: 2-space indent, 100 char line width (Biome). Double quotes, always semicolons, trailing commas. JSON: no trailing commas.
- LF line endings everywhere (`.editorconfig`).
- Extensions `.gitignore`d: `target/`, `node_modules/`, `dist/`, `.output/`, `.wxt/`.
