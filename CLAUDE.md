# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## What this is

Orca is an Electron desktop IDE for running multiple CLI coding agents (Claude Code, Codex, OpenCode, etc.) in parallel, each isolated in its own git worktree, with a terminal-first UI, an in-app browser, GitHub/Linear/GitLab review integration, and a mobile companion app for remote monitoring. `mobile/` is a separate pnpm workspace (own lockfile) and is excluded from the root workspace via `pnpm-workspace.yaml` (`packages: []`) — never run root commands expecting them to touch `mobile/`.

## Commands

Package manager is **pnpm**. Common commands:

- `pnpm dev` — run the Electron app in dev mode (runs `ensure:electron-runtime` first).
- `pnpm build` — full desktop + native build (`build:desktop` + `build:native`).
- `pnpm test` — run the Vitest unit suite (`config/vitest.config.ts`). Runs `ensure-native-runtime.mjs --runtime=node` first.
  - Single test file: `pnpm exec vitest run --config config/vitest.config.ts path/to/file.test.ts`
  - Single test by name: add `-t "test name"`.
- `pnpm lint` — oxlint plus a chain of quality/consistency gates (code-quality audits, max-lines ratchet, skill-guide/manifest verification, localization checks). Run the specific sub-check when iterating instead of the whole chain (e.g. `pnpm run audit:code-quality:native`).
- `pnpm typecheck` (`pnpm tc`) — three separate `tsc --noEmit` passes: `tc:node` (main/relay/preload), `tc:cli`, `tc:web` (renderer). Run the one matching the code you touched for faster feedback.
- `pnpm test:e2e` — Playwright e2e against a built Electron app (`electron-headless` project). There are many narrower `test:e2e:*` scripts for specific areas (terminal rendering, SSH/Docker, perf) — prefer the narrow one relevant to your change over the full suite.
- `pnpm format` — `oxfmt --write .`.

CI/local gates worth knowing about before large changes: `check:max-lines-ratchet` (no growing already-long files — see the max-lines rule in AGENTS.md), `check:reliability-gates`, and the localization catalog/extraction/coverage checks (any new user-facing string needs matching i18n catalog entries).

## Architecture

Standard Electron three-process split, plus a fourth **relay** layer that does the actual host I/O:

- **`src/main/`** — Electron main process: window/app lifecycle, per-agent integrations (one subfolder per supported agent/provider: `claude/`, `codex/`, `gemini/`, `github/`, `gitlab/`, `bitbucket/`, `azure-devops/`, `automations/`, etc.), IPC handler registration (~90+ `ipcMain.handle` call sites), auth/accounts, native module bridging (computer-use, notification-status).
- **`src/relay/`** — A subprocess-facing service layer used by main (and reachable from remote/SSH hosts) that does the heavy lifting: PTY management (`pty-handler*`, `pty-source-*`), git operations (`git-handler-*`, one file per concern: staging, worktrees, branch cleanup, submodules...), filesystem watching/streaming (`fs-handler-*`, `relay-watcher-*`), and a custom framed protocol (`protocol.ts`, `relay-frame-decoder.ts`) with backpressure/credit-based flow control for PTY and git output streams (`pty-source-credit-*`, `dispatcher-writer-*`). This is what lets Orca drive agents/git over SSH or WSL, not just locally — see `wsl-*` and `ssh-*` files and the SSH/WSL guidance in AGENTS.md.
- **`src/preload/`** — the narrow, typed IPC bridge (`api-types.ts`, `index.ts`) exposed to the renderer as `window.electronAPI`-style APIs; keep new IPC surface here typed and minimal.
- **`src/renderer/`** — the React UI (Vite + Tailwind + shadcn primitives). `src/renderer/src/store/` is Zustand-based, split into `slices/`; watch for selector fan-out (`check:zustand-selector-fanout`) when adding selectors. `src/renderer/src/components/ui/` holds shadcn primitives — reuse them per `docs/STYLEGUIDE.md` before writing custom CSS. `src/renderer/src/assets/main.css` is the canonical source for design tokens.
- **`src/cli/`** — the `orca` CLI (`orca worktree create`, `snapshot`, `click`, `fill`, etc.) that lets agents drive Orca itself; has its own runtime/dispatch/handler-group structure (`handlers/`, `dispatch.ts`, `runtime/`) and is built independently (`build:cli`) with its own tsconfig (`tsconfig.cli.json`) and bin verification (`verify:cli-bin.mjs`).
- **`src/shared/`** — types and pure logic shared across main/relay/renderer/cli (agent detection/kind, hook types, prompt-injection guards, etc.) — put cross-process logic here rather than duplicating it.
- **`native/`** — platform-specific native modules built per OS (`computer-use-{macos,linux,windows}`, `notification-status-macos`, `windows-cli-launcher`); see `build:native` and the Linux glibc-floor constraint in AGENTS.md.
- **`config/`** — nearly all build/check tooling lives here as standalone scripts (`config/scripts/*.mjs`) rather than inline npm-script bodies, plus the split tsconfigs (`tsconfig.node.json`, `tsconfig.cli.json`, `tsconfig.web.json`) and `vitest.config.ts`.
- **`mobile/`** — separate Expo/React Native workspace (own `package.json`, `pnpm-workspace.yaml`, lockfile) for the companion mobile app; treat it as a distinct project when working there.

### Agent integrations pattern

Each supported CLI agent/provider gets its own directory under `src/main/` (and often `src/shared/`) named after the agent (`claude/`, `codex/`, `gemini/`, `amp/`, `droid/`, `cursor/`, etc.) containing detection, launch, hook, and account-management logic specific to that tool. When adding support for a new agent or provider, follow the shape of an existing sibling directory rather than inventing a new pattern.

### Testing conventions

Tests are colocated as `*.test.ts` next to the source file they cover (not in a separate `__tests__` tree), across `src/`, `config/scripts/`, and `tests/tools/`. E2E specs live under `tests/e2e/` and run via Playwright against a built, headless Electron app.

## Fork rebrand: "Dark Factory" (this fork only)

This fork (`eccosai2018/orca`) is being rebranded from **Orca** to **Dark Factory** in stages. The rename is deliberately split into "safe" (cosmetic/pointer) and "not done" (functional/native) categories — don't casually extend either without re-reading the reasoning below, since several of these are one-way or break existing user state if done carelessly.

**Renamed — UI text** (i18n catalog, window/tab/HTML titles, tray, app menu, notifications, crash/update/GPU-fallback dialogs, About panel via `BASE_APP_NAME` in `src/main/startup/dev-instance-identity.ts`): pure display strings, no functional coupling, safe to keep extending.

**Renamed — packaging identity**: `productName` in `config/electron-builder.config.cjs` is `'DarkFactory'` — **no space, deliberately**. It's used as a literal path token by several build/smoke/CI scripts (`config/scripts/smoke-packaged-cli.mjs`, `smoke-packaged-hang-watchdog-worker.mjs`, `src/main/local-builds/local-build-candidate.ts`'s `DarkFactory.app/Contents/Resources/...` extraction path) that are not all shell-quoting-safe; a spaced product name would silently break the ones that aren't. This is independent from `BASE_APP_NAME` (`'Dark Factory'`, with the space) — that constant drives `app.setName()` (About panel, userData folder, Keychain label text) via safe Node path APIs, not string-concatenated shell tokens, so it can carry the nicer spaced form. **If you add any new script that references the packaged `.app`/`.exe` name as a bare string, use `DarkFactory`, not `Dark Factory`.**

**Renamed — GitHub pointers**: `MAIN_RELEASE_REPO` (`src/shared/release-channel.ts`, drives the in-app auto-updater's release check), `ORCA_SKILLS_REPOSITORY_URL` (`src/shared/agent-feature-install-commands.ts`, the `npx skills add <url>` install command), and `package.json`'s `homepage` now point at `eccosai2018/orca` instead of `stablyai/orca`. **This means auto-update only works if this fork actually publishes signed GitHub Releases** — if it doesn't, the updater will find nothing. `HOURLY_RELEASE_REPO`/`ADHOC_RELEASE_REPO` (dev-channel infra, irrelevant to a personal fork) were deliberately left pointing at `stablyai/*`.

**Renamed — git attribution** (`src/shared/orca-attribution.ts`, `src/main/attribution/terminal-attribution.ts`): the `Co-authored-by:` trailer and `Made with [...]` PR/issue footers that get written into commits/PRs when the attribution toggle is on now read `Co-authored-by: Echos 2018 AI Labs` and link to `eccosai2018/orca`. The on-disk shim scripts (`~/.../orca-terminal-attribution/`) are versioned via `ATTRIBUTION_SHIM_VERSION` — bump it (it's currently `'7'`) whenever you change the generated wrapper script text, or existing users keep the stale on-disk copy.

**Renamed — macOS-safe internal identifiers**: `TERM_PROGRAM` env value set on spawned PTYs (`src/main/providers/local-pty-provider.ts`, `src/main/daemon/pty-subprocess.ts`) and the Keychain service name for managed Claude credentials (`ORCA_CLAUDE_SERVICE` in `src/main/claude-accounts/keychain.ts`) are both single-source-of-truth constants with no other code branching on their exact value, so renaming them is functionally safe. Note: renaming the Keychain service name orphans anything already stored under the old name — affected users see a one-time re-login, not a crash.

**Deliberately NOT renamed** — each of these requires coordinated native-build or storage-migration work, not a text edit:
- **Native bundle paths**: `Orca Computer Use.app` (a *separately built* Xcode target under `native/computer-use-macos`, unrelated to electron-builder's `productName`) and the Windows `Orca.exe`/`taskkill /IM Orca.exe` NSIS-updater identity in `src/main/daemon/daemon-host-relocation.ts`. Renaming these means renaming the actual native build target output, in lockstep with every script that locates it.
- **Protocol constants**: the `X-Orca-Agent-Hook-Token` header name, sent by every agent-hook script (Claude/Codex/Amp/Gemini/Cursor/etc.) and read on the relay side. It's wire format shared by both ends — renaming it requires updating every hook script and the relay reader together, atomically.
- **Windows Keychain/firewall storage keys**: `FIREWALL_RULE_NAME = 'Orca.MobilePairing'` (`src/main/runtime/windows-mobile-firewall.ts`) — same orphaning risk as the macOS Keychain rename above, not yet done.
- **Comments and internal identifiers** (`OrcaRuntimeService`, `orca-profiles/`, env vars like `ORCA_AGENT_HOOK_TOKEN`) — out of scope by design; this is a UI/product rebrand, not a full codebase search-and-replace.
