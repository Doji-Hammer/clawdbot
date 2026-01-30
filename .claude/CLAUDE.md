# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Moltbot is a personal AI assistant that runs on your own devices, answering through messaging channels you already use (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Google Chat, Microsoft Teams, Matrix, Zalo, etc.). The Gateway service is the control plane that orchestrates agents, channels, and LLM providers.

**Repo:** https://github.com/moltbot/moltbot

## Build, Test, and Development Commands

**Runtime:** Node 22+ (keep Node + Bun paths working)

```bash
# Install dependencies
pnpm install

# Build (TypeScript compilation)
pnpm build

# Lint and format
pnpm lint          # oxlint
pnpm format        # oxfmt (check)
pnpm format:fix    # oxfmt (write)

# Tests
pnpm test          # vitest (unit/integration)
pnpm test:coverage # with V8 coverage (70% threshold)
pnpm test:e2e      # end-to-end tests
pnpm test:live     # live tests with real API keys (CLAWDBOT_LIVE_TEST=1)

# Run single test file
pnpm test src/path/to/file.test.ts

# Run CLI in dev
pnpm moltbot <command>    # runs via bun/tsx
pnpm dev                  # alias

# Gateway development
pnpm gateway:watch        # auto-reload on changes
pnpm gateway:dev          # dev mode (skip channels)

# Pre-commit hooks
prek install              # runs same checks as CI

# Full gate (run before pushing)
pnpm lint && pnpm build && pnpm test
```

## Architecture

### Core Layers

```
src/
├── cli/              # CLI commands + program builder (entry: entry.ts)
├── gateway/          # Gateway service - WebSocket + HTTP control plane
│   ├── server/       # WS connection management, plugin HTTP, control UI
│   └── protocol/     # OpenAI-compatible API layer
├── agents/           # Pi agent runtime (from @mariozechner/pi-* packages)
│   └── pi-embedded-runner/  # In-process agent with auth, model selection, tools
├── channels/         # Channel abstraction layer
│   └── plugins/      # Adapter interfaces (messaging, auth, outbound, security)
├── routing/          # Agent <-> channel account bindings
├── plugins/          # Plugin runtime + registry
├── plugin-sdk/       # Public SDK for extensions (moltbot/plugin-sdk)
├── config/           # Config loading + validation
└── infra/            # Daemon management (launchd/systemd), bonjour, ports
```

### Built-in Channels (src/)

`discord/`, `telegram/`, `slack/`, `signal/`, `imessage/`, `web/` (WhatsApp), `line/`

### Extension Channels (extensions/)

`matrix/`, `msteams/`, `zalo/`, `zalouser/`, `bluebubbles/`, `googlechat/`, `tlon/`, `twitch/`, `voice-call/`, etc.

### Native Apps (apps/)

- `macos/` - SwiftUI menubar app (uses Observation framework, not ObservableObject)
- `ios/` - SwiftUI iOS app
- `android/` - Kotlin Android app

### Message Flow

```
Channel receives message
  → Gateway: ChannelMessagingAdapter.onInboundMessage()
  → Resolve routing bindings → find agent
  → Pi agent processes (tools, memory, LLM)
  → ChannelOutboundAdapter.send() → message delivered
```

### Key Patterns

- **Dependency Injection:** `createDefaultDeps()` provides logger, config loader, channel registry
- **Plugin Registry:** Dynamic loading via `PluginRuntime`; plugins self-register adapters
- **Configuration-Driven:** Bindings, policies, agents, hooks all declared in config
- **Session Isolation:** Each agent has isolated workspace (`~/.clawdbot/agents/{agentId}`)

## Project Structure

- **Source:** `src/` (CLI in `src/cli`, commands in `src/commands`)
- **Tests:** colocated `*.test.ts` (e2e in `*.e2e.test.ts`, live in `*.live.test.ts`)
- **Docs:** `docs/` (Mintlify, hosted at docs.molt.bot)
- **Extensions:** `extensions/*` (workspace packages)
- **Built output:** `dist/`

## Coding Conventions

- **Language:** TypeScript (ESM), strict typing, avoid `any`
- **Formatting:** Oxlint + Oxfmt; run `pnpm lint` before commits
- **Naming:** "Moltbot" for product/docs headings; `moltbot` for CLI/package/paths
- **File size:** Aim for ~500-700 LOC; split/refactor for clarity
- **Comments:** Brief comments for tricky/non-obvious logic only

## Commit & PR Guidelines

- Create commits with `scripts/committer "<msg>" <file...>` (keeps staging scoped)
- Concise, action-oriented messages (e.g., `CLI: add verbose flag to send`)
- PR merge flow: create temp branch from `main`, squash/rebase, run full gate locally, merge back
- When merging PRs: add changelog entry with PR # and thanks, add co-contributor if squashing
- After merging: run `bun scripts/update-clawtributors.ts` for new contributors

## Testing Guidelines

- **Framework:** Vitest with V8 coverage (70% threshold)
- **Live tests:** `CLAWDBOT_LIVE_TEST=1 pnpm test:live` (requires real API keys)
- **Docker tests:** `pnpm test:docker:live-models`, `pnpm test:docker:onboard`, etc.
- Test additions generally don't need changelog entries unless they alter user-facing behavior

## Docs (Mintlify)

- Hosted at docs.molt.bot
- Internal links: root-relative, no `.md`/`.mdx` (e.g., `[Config](/configuration)`)
- Anchors: `[Hooks](/configuration#hooks)`
- Avoid em dashes and apostrophes in headings (breaks anchors)
- README keeps absolute URLs (`https://docs.molt.bot/...`) for GitHub

## Plugin Development

Plugins live in `extensions/` as workspace packages:

```json
{
  "name": "@moltbot/discord",
  "moltbot": {
    "extensions": ["./index.ts"]
  }
}
```

- Import from `moltbot/plugin-sdk`
- Runtime deps in `dependencies`; avoid `workspace:*` in dependencies
- Put `moltbot` in `devDependencies` or `peerDependencies`

## Key Files & Locations

- **Config:** `~/.clawdbot/config.json` (or `moltbot.json`)
- **Credentials:** `~/.clawdbot/credentials/`
- **Sessions:** `~/.clawdbot/sessions/`
- **Agent workspaces:** `~/.clawdbot/agents/{agentId}/`
- **Version locations:** `package.json`, `apps/*/Info.plist` or `build.gradle.kts`

## Important Constraints

- Never edit `node_modules`
- Never update the Carbon dependency
- Patched dependencies (`pnpm.patchedDependencies`) must use exact versions (no `^`/`~`)
- Patching requires explicit approval
- CLI progress: use `src/cli/progress.ts` (not hand-rolled spinners)
- Terminal tables: use `src/terminal/table.ts`
- Colors: use shared palette in `src/terminal/palette.ts` (no hardcoded colors)
- Tool schemas: avoid `Type.Union`, use `stringEnum`/`optionalStringEnum`; no raw `format` property names

## Multi-Agent Safety

- Do not create/apply/drop git stash unless explicitly requested
- Do not create/modify git worktrees unless explicitly requested
- Do not switch branches unless explicitly requested
- When pushing: `git pull --rebase` is OK (never discard other agents' work)
- When committing: scope to your changes only

## macOS Development

- Gateway runs as menubar app (no separate LaunchAgent)
- Restart via Moltbot Mac app or `scripts/restart-mac.sh`
- Logs: `./scripts/clawlog.sh` for unified log queries
- Don't rebuild macOS app over SSH; must run directly on Mac
- SwiftUI: prefer Observation framework (`@Observable`, `@Bindable`) over `ObservableObject`

## Release Flow

- Read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` before release work
- Do not change version numbers without explicit consent
- Ask permission before running npm publish/release steps

## Troubleshooting

- Run `moltbot doctor` for migration issues and config warnings
