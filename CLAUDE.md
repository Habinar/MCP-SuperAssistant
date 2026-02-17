# CLAUDE.md — MCP-SuperAssistant

This file provides guidance for AI assistants working with the MCP-SuperAssistant codebase.

## Project Overview

MCP-SuperAssistant is a Chrome/Firefox browser extension that integrates the Model Context Protocol (MCP) with major AI chat platforms (ChatGPT, Gemini, Perplexity, Grok, DeepSeek, Mistral, and 10+ others). It allows users to execute MCP tools directly from AI chat interfaces via an injected sidebar UI.

**Repository:** <https://github.com/srbhptl39/MCP-SuperAssistant.git>
**License:** MIT
**Current version:** 0.5.9

## Tech Stack

- **Runtime:** Node.js >=22.12.0 (see `.nvmrc`)
- **Package manager:** pnpm 9.15.1 (monorepo with workspaces)
- **Monorepo orchestration:** Turborepo
- **Language:** TypeScript 5.8 (strict mode)
- **UI framework:** React 19, rendered inside Shadow DOM
- **State management:** Zustand 5
- **Styling:** Tailwind CSS 3.4 + shadcn/ui + Radix UI
- **Build tool:** Vite 6
- **Linting:** ESLint 9 (flat config) + Prettier
- **Extension manifest:** Manifest V3 (generated from `chrome-extension/manifest.ts`)
- **MCP SDK:** `@modelcontextprotocol/sdk` ^1.25.2

## Repository Structure

```
MCP-SuperAssistant/
├── chrome-extension/          # Extension background script + MCP client
│   ├── src/background/        # Service worker (message routing, remote config)
│   ├── src/mcpclient/         # MCP client core (transports: SSE, WebSocket, HTTP)
│   ├── manifest.ts            # Manifest V3 generator
│   └── vite.config.mts
├── pages/
│   └── content/               # Content script — the main application
│       └── src/
│           ├── core/          # Initialization, error handling, performance
│           ├── plugins/       # Plugin system + 17 website adapters
│           │   └── adapters/  # chatgpt, gemini, grok, perplexity, etc.
│           ├── components/    # React components (sidebar, UI, per-site)
│           ├── stores/        # Zustand stores (connection, adapter, tool, ui, config, app)
│           ├── events/        # Event bus system
│           ├── hooks/         # React hooks
│           ├── render_prescript/ # Function call parsing + rendering
│           ├── services/      # Automation services
│           ├── types/         # TypeScript type definitions
│           └── utils/         # Utility functions
├── packages/                  # Shared workspace packages
│   ├── dev-utils/             # Build utilities, manifest parser
│   ├── env/                   # Environment variable handling
│   ├── hmr/                   # Hot module reload for dev
│   ├── i18n/                  # Internationalization (en, ko)
│   ├── module-manager/        # CLI module management tool
│   ├── shared/                # Shared hooks, HOCs, logger, utils
│   ├── storage/               # Chrome storage abstraction layer
│   ├── tailwind-config/       # Shared Tailwind configuration
│   ├── tsconfig/              # Shared TS configs (base, app, module)
│   ├── ui/                    # React component library (shadcn/ui based)
│   ├── vite-config/           # Shared Vite configuration
│   └── zipper/                # Build artifact compression
├── bash-scripts/              # Shell utilities (env copy, version bump)
├── .github/workflows/         # CI: build-zip, e2e, prettier, auto-assign
├── turbo.json                 # Turborepo task definitions
├── pnpm-workspace.yaml        # Workspace: chrome-extension, pages/*, packages/*
├── eslint.config.ts           # Flat ESLint config
└── .prettierrc                # Prettier settings
```

### Workspace packages use the `@extension/*` namespace

All internal packages are imported as `@extension/<name>` (e.g., `@extension/shared`, `@extension/storage`, `@extension/ui`).

## Common Commands

### Development

```bash
pnpm install                  # Install all dependencies (runs postinstall: build eslint + copy env)
pnpm dev                      # Start dev mode with HMR (Chrome)
pnpm dev:firefox              # Start dev mode with HMR (Firefox)
```

### Building

```bash
pnpm build                    # Production build (Chrome)
pnpm build:firefox            # Production build (Firefox)
pnpm zip                      # Build + zip for distribution
pnpm zip:firefox              # Build + zip for Firefox
```

### Code Quality

```bash
pnpm lint                     # ESLint with --fix across all packages
pnpm prettier                 # Prettier formatting across all packages
pnpm type-check               # TypeScript type checking across all packages
```

### Cleaning

```bash
pnpm clean:bundle             # Remove dist directories
pnpm clean:node_modules       # Remove all node_modules
pnpm clean:turbo              # Clear Turbo cache
pnpm clean                    # Full clean (bundle + turbo + node_modules)
pnpm clean:install            # Clean node_modules + fresh install
```

### Testing

```bash
pnpm e2e                      # Build, zip, and run E2E tests (Chrome)
pnpm e2e:firefox              # Build, zip, and run E2E tests (Firefox)
```

Individual package tests can be run with:
```bash
pnpm -F chrome-extension test # Runs vitest in chrome-extension
```

### Other

```bash
pnpm update-version <0.0.0>   # Bump version in all package.json files
pnpm module-manager            # Interactive CLI for module management
```

## Architecture

### Layer Diagram

```
┌──────────────────────────────────────────────────────┐
│  AI Chat Websites (ChatGPT, Gemini, Grok, etc.)     │
├──────────────────────────────────────────────────────┤
│  Content Script (pages/content)                      │
│  ├─ Plugin Registry (17 site adapters)               │
│  ├─ Zustand Stores (6 stores)                        │
│  ├─ Event Bus                                        │
│  └─ Sidebar UI (React + Tailwind in Shadow DOM)      │
├──────────────────────────────────────────────────────┤
│  chrome.runtime messaging                            │
├──────────────────────────────────────────────────────┤
│  Background Service Worker (chrome-extension)        │
│  ├─ Message routing                                  │
│  ├─ Firebase Remote Config                           │
│  └─ MCP Client                                      │
├──────────────────────────────────────────────────────┤
│  Transport Layer (pluggable)                         │
│  ├─ SSE (default, localhost:3006/sse)                │
│  ├─ WebSocket (localhost:3006/message)               │
│  └─ Streamable HTTP (localhost:3006)                 │
├──────────────────────────────────────────────────────┤
│  External MCP Servers                                │
└──────────────────────────────────────────────────────┘
```

### Key Design Patterns

- **Plugin/Adapter pattern**: Each supported AI platform has an adapter in `pages/content/src/plugins/adapters/`. Adapters extend `BaseAdapterPlugin` and are lazily loaded by hostname match.
- **Zustand stores**: Six stores manage application state (`connection`, `adapter`, `tool`, `ui`, `config`, `app`). Located in `pages/content/src/stores/`.
- **Event bus**: Decoupled communication between components via `pages/content/src/events/event-bus.ts`.
- **Shadow DOM isolation**: The sidebar UI renders inside Shadow DOM to avoid CSS conflicts with host pages.
- **Transport plugins**: MCP client uses a plugin registry supporting SSE, WebSocket, and Streamable HTTP transports in `chrome-extension/src/mcpclient/plugins/`.
- **Circuit breaker**: Fault tolerance for MCP connections in `pages/content/src/core/circuit-breaker.ts`.

### Adding a New Website Adapter

1. Create `pages/content/src/plugins/adapters/<site>.adapter.ts` extending `BaseAdapterPlugin`.
2. Define `name`, `version`, `hostnames` (string[] or RegExp[]), and `capabilities`.
3. Implement lifecycle methods: `initialize()`, `activate()`, `deactivate()`, `cleanup()`.
4. Optionally add per-site components in `pages/content/src/components/websites/<site>/`.
5. Optionally add a default config in `pages/content/src/plugins/adapters/defaultConfigs/`.
6. Register the adapter factory in the plugin registry.
7. Add the host permission in `chrome-extension/manifest.ts`.
8. See `pages/content/src/plugins/adapters/README.md` for detailed guidance.

## Code Conventions

### TypeScript

- Strict mode is enabled (`strict: true`, `noImplicitAny: true`, `strictNullChecks: true`).
- Use `import type { ... }` for type-only imports — enforced by ESLint rule `@typescript-eslint/consistent-type-imports`.
- Target and module are both `ESNext`; module resolution is `bundler`.
- Path aliases: `@src/*` maps to `src/*` in chrome-extension; `@src` maps to `src/` in pages/content.

### Formatting (Prettier)

- Single quotes, trailing commas everywhere, semicolons required.
- Print width: 120 characters.
- Arrow function parens: avoid when possible (`x => x`).
- Bracket same line: true.

### Linting (ESLint)

- Flat config in `eslint.config.ts`.
- React 19 with automatic JSX runtime (no `import React` needed).
- `react/prop-types` is off (TypeScript handles prop validation).
- Accessibility rules via `jsx-a11y`.
- Import ordering via `eslint-plugin-import-x`.
- Tailwind CSS class ordering via `eslint-plugin-tailwindcss`.

### Git Hooks

- Pre-commit: runs `lint-staged` which applies Prettier to `*.{js,jsx,ts,tsx,json}` files.

### Module System

- ESM throughout (`"type": "module"` in root package.json).
- All workspace packages are private.

## Environment Variables

Copy `.example.env` to `.env` for local development (done automatically by `pnpm install` via postinstall).

Key variables:
- `CLI_CEB_DEV=true` — development mode flag
- `CLI_CEB_FIREFOX=false` — Firefox build flag
- `FIREBASE_*` — Firebase config for remote config and analytics (dev/prod variants)

Environment variables prefixed with `CEB_*` and `CLI_CEB_*` are treated as global by Turborepo (see `turbo.json`).

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- **build-zip.yml** — Builds and uploads extension artifacts on push to main/dev and PRs.
- **e2e.yml** — Runs E2E tests for both Chrome and Firefox.
- **prettier.yml** — Checks code formatting.
- **auto-assign.yml** — Auto-assigns PR reviewers.
- **greetings.yml** — Welcomes first-time contributors.

## Supported Platforms

The extension injects content scripts into these AI chat services (see `chrome-extension/manifest.ts` for the full list):
ChatGPT, Gemini, Google AI Studio, Perplexity, Grok, DeepSeek, OpenRouter, Mistral, T3 Chat, Kimi, GitHub Copilot, Qwen Chat, Z Chat.

## Gotchas and Notes

- The build must run `ready` (TypeScript compilation of dependencies) before `dev` or `build` — Turbo handles this automatically via task dependencies.
- The content script bundles as IIFE (`index.iife.js`) since it runs in web page context.
- The sidebar UI uses Shadow DOM, so all styling must be self-contained. Use `useShadowDomStyles` hook for dynamic styles.
- The MCP client lives in the background service worker, not the content script. Communication happens via `chrome.runtime.sendMessage`.
- When adding new permissions or host patterns, update both the manifest and the content script match patterns.
- The `.husky/pre-committ` file (note: typo in filename) runs lint-staged before commits.
- Node 22.12.0+ is required — the project uses modern ES features.
