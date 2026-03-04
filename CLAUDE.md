# CLAUDE.md — MCP-SuperAssistant

> Instructions for AI assistants working with this codebase.
> Treat this as a day-one onboarding doc for a senior engineer.

## Quick Reference — Read First

1. Use `import type { ... }` for type-only imports — enforced by ESLint.
2. Never use `any` — use `unknown` and narrow with type guards. Existing `any` in `stores.ts` types is legacy; do not spread further.
3. Run `pnpm lint && pnpm type-check` before suggesting any commit.
4. Prefer editing existing files over creating new ones.
5. All UI renders inside Shadow DOM — styles must be self-contained.
6. The MCP client lives in the background service worker, NOT the content script. Communication uses `chrome.runtime.sendMessage`.

## Project Overview

MCP-SuperAssistant is a Chrome/Firefox browser extension that integrates the Model Context Protocol (MCP) with 13+ AI chat platforms (ChatGPT, Gemini, Perplexity, Grok, DeepSeek, Mistral, etc.). Users execute MCP tools directly from AI chat interfaces via an injected sidebar UI.

- **Repository:** <https://github.com/srbhptl39/MCP-SuperAssistant.git>
- **License:** MIT
- **Current version:** 0.5.9

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js >=22.12.0 (`.nvmrc`) |
| Package manager | pnpm 9.15.1 (monorepo + workspaces) |
| Monorepo | Turborepo |
| Language | TypeScript 5.8 (strict mode) |
| UI | React 19, Shadow DOM isolation |
| State | Zustand 5 with devtools middleware |
| Styling | Tailwind CSS 3.4 + shadcn/ui + Radix UI |
| Build | Vite 6 |
| Linting | ESLint 9 (flat config) + Prettier |
| Manifest | Manifest V3 (generated from `chrome-extension/manifest.ts`) |
| MCP SDK | `@modelcontextprotocol/sdk` ^1.25.2 |

## Repository Structure

```
MCP-SuperAssistant/
├── chrome-extension/          # Background service worker + MCP client
│   ├── src/background/        # Service worker (message routing, remote config)
│   ├── src/mcpclient/         # MCP client core
│   │   ├── core/              #   McpClient, PluginRegistry, EventEmitter
│   │   ├── plugins/           #   Transport plugins (SSE, WebSocket, Streamable HTTP)
│   │   ├── types/             #   Transport types (plugin, config, events, primitives)
│   │   └── config/            #   Default configuration
│   ├── manifest.ts            # Manifest V3 generator
│   └── vite.config.mts
├── pages/content/             # Content script — the main application
│   └── src/
│       ├── core/              # Initialization, circuit breaker, error handling, performance
│       ├── plugins/           # Plugin system + adapter registry
│       │   └── adapters/      # 15 website adapters (chatgpt, gemini, grok, etc.)
│       ├── components/        # React components (sidebar, UI, per-site overrides)
│       ├── stores/            # Zustand stores (connection, adapter, tool, ui, config, app)
│       ├── events/            # Typed event bus system
│       ├── hooks/             # React hooks (useAdapter, useEventBus, useShadowDomStyles, etc.)
│       ├── render_prescript/  # Function call parsing + rendering
│       ├── services/          # Automation services
│       ├── types/             # TypeScript type definitions (stores.ts is the source of truth)
│       └── utils/             # Utility functions
├── packages/                  # Shared workspace packages (@extension/* namespace)
│   ├── dev-utils/             # Build utilities, manifest parser
│   ├── env/                   # Environment variable handling
│   ├── hmr/                   # Hot module reload for dev
│   ├── i18n/                  # Internationalization (en, ko)
│   ├── module-manager/        # CLI module management tool
│   ├── shared/                # Shared hooks, HOCs, logger (createLogger), utils
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

### Namespace Convention

All internal packages use `@extension/<name>` (e.g., `@extension/shared`, `@extension/storage`, `@extension/ui`).

### Path Aliases

- `@src/*` maps to `src/*` in `chrome-extension`
- `@src` maps to `src/` in `pages/content`

## Commands

### Development

```bash
pnpm install          # Install deps (postinstall: builds eslint config + copies .env)
pnpm dev              # Dev mode with HMR (Chrome)
pnpm dev:firefox      # Dev mode with HMR (Firefox)
```

### Build & Ship

```bash
pnpm build            # Production build (Chrome)
pnpm build:firefox    # Production build (Firefox)
pnpm zip              # Build + zip for distribution
pnpm zip:firefox      # Build + zip for Firefox
```

### Quality Gates

```bash
pnpm lint             # ESLint --fix across all packages
pnpm prettier         # Prettier formatting across all packages
pnpm type-check       # TypeScript type checking across all packages
```

### Testing

```bash
pnpm e2e              # Build, zip, run E2E tests (Chrome)
pnpm e2e:firefox      # Build, zip, run E2E tests (Firefox)
pnpm -F chrome-extension test  # Vitest in chrome-extension package
```

### Maintenance

```bash
pnpm clean            # Full clean (dist + turbo cache + node_modules)
pnpm clean:install    # Clean node_modules + fresh install
pnpm update-version <0.0.0>  # Bump version across all package.json files
pnpm module-manager   # Interactive CLI for module management
```

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────────────────┐
│  AI Chat Websites (ChatGPT, Gemini, Grok, etc.)        │
├─────────────────────────────────────────────────────────┤
│  Content Script (pages/content) — bundles as IIFE      │
│  ├─ Plugin Registry → hostname-matched adapter loading │
│  ├─ Zustand Stores (6) → single source of truth       │
│  ├─ Typed Event Bus → decoupled component comms        │
│  └─ Sidebar UI (React 19 in Shadow DOM)                │
├─────────────────────────────────────────────────────────┤
│  chrome.runtime.sendMessage (async message passing)    │
├─────────────────────────────────────────────────────────┤
│  Background Service Worker (chrome-extension)          │
│  ├─ Message routing + Firebase Remote Config           │
│  └─ MCP Client (PluginRegistry → transport selection)  │
├─────────────────────────────────────────────────────────┤
│  Transport Layer (pluggable via ITransportPlugin)      │
│  ├─ SSE (default, localhost:3006/sse)                  │
│  ├─ WebSocket (localhost:3006/message)                 │
│  └─ Streamable HTTP (localhost:3006)                   │
├─────────────────────────────────────────────────────────┤
│  External MCP Servers                                  │
└─────────────────────────────────────────────────────────┘
```

### Key Design Patterns

- **Plugin/Adapter pattern:** Each AI platform has an adapter extending `BaseAdapterPlugin` in `pages/content/src/plugins/adapters/`. Adapters declare `name`, `version`, `hostnames`, and `capabilities`. Loaded lazily by hostname match.
- **Zustand stores with devtools:** Six stores (`connection`, `adapter`, `tool`, `ui`, `config`, `app`) each co-locate state + actions in a single `create<State>()(devtools(...))` call. Actions emit events via the event bus for cross-store coordination.
- **Typed event bus:** Singleton `TypedEventBus` class with namespaced events (e.g., `connection:status-changed`, `error:unhandled`). Supports once-listeners, wildcards, and event history. Located in `pages/content/src/events/`.
- **Shadow DOM isolation:** Sidebar UI renders inside Shadow DOM. Use `useShadowDomStyles` hook for dynamic styles. All Tailwind must be scoped to the shadow root.
- **Transport plugins:** MCP client uses `ITransportPlugin` interface with `PluginRegistry` for transport selection (SSE, WebSocket, Streamable HTTP) in `chrome-extension/src/mcpclient/plugins/`.
- **Circuit breaker:** Fault tolerance for MCP connections. Tracks failure/success counts, transitions through `closed → open → half-open` states. Located in `pages/content/src/core/circuit-breaker.ts`.
- **Logger pattern:** Every module creates a scoped logger via `createLogger('ModuleName')` from `@extension/shared`. Use this consistently — never use raw `console.log`.

### Data Flow: Tool Execution

```
User clicks tool in sidebar
  → React component dispatches via useStores/useMcpCommunication hook
    → chrome.runtime.sendMessage to background
      → MCP Client selects transport plugin
        → Transport sends to external MCP server
          → Response flows back through the same chain
            → Zustand store updates → React re-renders
```

### Initialization Sequence

The app initializes in strict order via `core/main-initializer.ts`:

```
1. Environment setup
2. Event bus initialization
3. Core services (error handler, circuit breaker, performance monitor)
4. Global event handlers
5. Store initialization (all 6 stores — order matters)
6. Plugin system registry
7. Plugin context creation
8. UI rendering
```

All steps are wrapped in try-catch with `globalErrorHandler.handleError(error, { component, operation })`.

### Store Patterns

Each store follows this structure:

```typescript
// 1. Interface co-locates state + actions
export interface FooState {
  data: string;
  setData: (value: string) => void;
}

// 2. Separate initial state (Omit actions)
const initialState: Omit<FooState, 'setData'> = { data: '' };

// 3. Create with devtools middleware
export const useFooStore = create<FooState>()(
  devtools((set, get) => ({
    ...initialState,
    setData: (value) => {
      set({ data: value });
      eventBus.emit('foo:data-changed', { value });
    },
  }), { name: 'FooStore' })
);
```

Stores that persist use `persist()` middleware wrapping `devtools()`. The `ui`, `app`, and `config` stores persist to `localStorage` via `createJSONStorage()` with `partialize()` to select which fields to persist.

Cross-store communication: use the event bus, not direct store imports. Direct subscriptions between stores can create circular dependencies.

### Event Naming Convention

Events use `{category}:{action}` dot-notation:

| Category | Examples |
|----------|---------|
| `app` | `app:initialized`, `app:cleanup` |
| `connection` | `connection:status-changed`, `connection:error` |
| `tool` | `tool:execution-started`, `tool:execution-completed` |
| `adapter` | `adapter:activated`, `adapter:deactivated`, `adapter:error` |
| `ui` | `ui:sidebar-toggled`, `ui:theme-changed` |
| `error` | `error:unhandled` |

All events are typed in `EventMap` interface (`pages/content/src/events/event-types.ts`). The event bus keeps a history of the last 100 events for debugging.

### Hook Usage Patterns

Access stores via granular hooks with `useShallow()` to prevent unnecessary re-renders:

```typescript
// Good — granular selector
const { isVisible, toggle } = useAppStore(useShallow(state => ({
  isVisible: state.isVisible,
  toggle: state.toggle,
})));

// Bad — reads entire store (causes re-render on any change)
const store = useAppStore();
```

Event bus hooks auto-unsubscribe on unmount:

```typescript
useEventListener('connection:status-changed', (data) => {
  // Handles event, auto-cleans up
}, [dependencies]);
```

### Chrome Storage Abstraction

Use `@extension/storage` — never access `chrome.storage` directly:

```typescript
import { createStorage, StorageEnum } from '@extension/storage';

const myStorage = createStorage<MyType>('key', defaultValue, {
  storageEnum: StorageEnum.Local,  // Local, Session, Sync, or Managed
  liveUpdate: true,                // Keeps all contexts in sync
});

const value = await myStorage.get();
await myStorage.set(newValue);
const snapshot = myStorage.getSnapshot(); // Sync access for React
```

## Code Conventions

### TypeScript

- Strict mode: `strict: true`, `noImplicitAny: true`, `strictNullChecks: true`.
- Use `import type { X }` for type-only imports — enforced by `@typescript-eslint/consistent-type-imports`.
- Target and module: `ESNext`. Module resolution: `bundler`.
- Prefer `interface` for object shapes, `type` for unions/intersections/utilities.
- Use `unknown` over `any`. Narrow with type guards or `satisfies`.
- Avoid `enum` — use `as const` objects or union literal types.
- Use early returns over nested conditionals.
- Destructure function parameters when 3+ args.

### React

- Functional components only. Automatic JSX runtime (no `import React`).
- `react/prop-types` is off — TypeScript handles prop validation.
- Extract custom hooks when component logic exceeds ~30 lines.
- Event handlers: `handleX` internally, `onX` in props.
- Components rendering in Shadow DOM must use `useShadowDomStyles`.

### Formatting (Prettier)

- Single quotes, trailing commas everywhere, semicolons required.
- Print width: 120 characters.
- Arrow parens: avoid when possible (`x => x`).
- Bracket same line: true.

### Linting (ESLint)

- Flat config in `eslint.config.ts`.
- Import ordering via `eslint-plugin-import-x`.
- Tailwind class ordering via `eslint-plugin-tailwindcss`.
- Accessibility rules via `jsx-a11y`.

### Logging

- Always use `createLogger('ModuleName')` from `@extension/shared/lib/logger`.
- Never use `console.log`, `console.warn`, or `console.error` directly.
- Log levels: `debug` for internal state changes, `warn` for unimplemented capabilities, `error` for failures.

### Error Handling

- Use `unknown` in catch blocks, then narrow: `if (error instanceof Error)`.
- Include context in error messages: what operation was being performed, what failed.
- Circuit breaker wraps MCP connection operations — don't add redundant retry logic.
- Event bus emits `error:unhandled` for uncaught listener errors with recursion guard.

### File Organization

- Group imports: external packages → `@extension/*` packages → relative imports. Separate with blank lines.
- Adapter files: `<site>.adapter.ts` in `pages/content/src/plugins/adapters/`.
- Store files: `<domain>.store.ts` in `pages/content/src/stores/`.
- Hook files: `use<Name>.ts` in `pages/content/src/hooks/`.
- Type definitions: centralized in `pages/content/src/types/stores.ts`.
- Per-site UI overrides: `pages/content/src/components/websites/<site>/`.
- Index files re-export only — no logic.

### Git

- Pre-commit hook: `lint-staged` applies Prettier to `*.{js,jsx,ts,tsx,json}`. (Note: hook file has a typo: `.husky/pre-committ`.)
- Use conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`.
- Keep commits atomic — one logical change per commit.
- Never commit `.env` files, secrets, or large binaries.

### Module System

- ESM throughout (`"type": "module"` in root package.json).
- All workspace packages are private.
- Use `.js` extension in re-export paths within `chrome-extension` (TypeScript with ESM resolution).

## Adding a New Website Adapter

1. Create `pages/content/src/plugins/adapters/<site>.adapter.ts` extending `BaseAdapterPlugin`.
2. Define `name`, `version`, `hostnames` (string[] or RegExp[]), and `capabilities` (from `AdapterCapability` type).
3. Implement lifecycle: `initialize(context)` → `activate()` → `deactivate()` → `cleanup()`. Call `super` methods.
4. Define CSS selectors as a private readonly object (follow `ChatGPTAdapter.selectors` pattern).
5. Implement capability methods: `insertText()`, `submitForm()`, `attachFile()` as needed.
6. Optionally add per-site UI components in `pages/content/src/components/websites/<site>/`.
7. Optionally add default config in `pages/content/src/plugins/adapters/defaultConfigs/`.
8. Register the adapter factory in the plugin registry index.
9. Add host permission in `chrome-extension/manifest.ts`.
10. See `pages/content/src/plugins/adapters/README.md` for the complete guide.

### Adapter Checklist

- [ ] Extends `BaseAdapterPlugin`
- [ ] All lifecycle methods call `super` and update `currentStatus`
- [ ] Uses `createLogger('<Name>Adapter')` for logging
- [ ] CSS selectors have fallback alternatives (AI chat UIs change frequently)
- [ ] MutationObserver cleanup in `deactivate()`/`cleanup()` to prevent memory leaks
- [ ] Host permission added to manifest

## Adding a New MCP Transport

1. Create a directory in `chrome-extension/src/mcpclient/plugins/<transport>/`.
2. Implement `ITransportPlugin` interface from `chrome-extension/src/mcpclient/types/plugin.ts`.
3. Register in the `PluginRegistry` via `chrome-extension/src/mcpclient/core/PluginRegistry.ts`.
4. Add the transport type to `ConnectionType` union in `pages/content/src/types/stores.ts`.

## Environment Variables

Copy `.example.env` to `.env` for local development (automated by `pnpm install` postinstall).

| Variable | Purpose |
|----------|---------|
| `CLI_CEB_DEV=true` | Development mode flag |
| `CLI_CEB_FIREFOX=false` | Firefox build flag |
| `FIREBASE_*` | Firebase remote config + analytics (dev/prod variants) |

Variables prefixed with `CEB_*` and `CLI_CEB_*` are Turborepo globals (see `turbo.json`).

## CI/CD

GitHub Actions in `.github/workflows/`:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `build-zip.yml` | Push to main/dev, PRs | Build + upload extension artifacts |
| `e2e.yml` | PRs | E2E tests for Chrome and Firefox |
| `prettier.yml` | PRs | Formatting check |
| `auto-assign.yml` | PR opened | Auto-assign reviewers |
| `greetings.yml` | First-time contributors | Welcome message |

## Supported Platforms

ChatGPT, Gemini, Google AI Studio, Perplexity, Grok, DeepSeek, OpenRouter, Mistral, T3 Chat, Kimi, GitHub Copilot, Qwen Chat, Z Chat.

See `chrome-extension/manifest.ts` for the full host permission list.

## Critical Gotchas

### Build & Dev

- Turbo runs `ready` (TS compilation of dependencies) before `dev`/`build` automatically via task dependencies. Never skip it.
- Content script bundles as IIFE (`index.iife.js`) — it runs in web page context, not extension context.
- Node 22.12.0+ is required. The project uses modern ES features unavailable in older versions.

### Shadow DOM

- The sidebar UI uses Shadow DOM. All styling must be self-contained within the shadow root.
- Use `useShadowDomStyles` hook for dynamic style injection.
- Global CSS from host pages does NOT leak into the shadow root (this is intentional).
- Tailwind classes must be compiled and injected into the shadow root's `<style>` element.

### Extension Architecture

- The MCP client lives in the background service worker, NOT the content script.
- Content script ↔ background communication is async via `chrome.runtime.sendMessage`.
- The background service worker can be terminated by the browser at any time (Manifest V3 constraint). Design for statelessness.

### Adapter Development

- AI chat platform UIs change frequently. Always use multiple fallback CSS selectors.
- Clean up MutationObservers, intervals, and event listeners in `deactivate()`/`cleanup()` to prevent memory leaks.
- Test adapters on the actual platform — DOM structure varies between logged-in/logged-out states.
- The `insertText()` method must handle both `contenteditable` divs and standard `<textarea>` inputs.

### Permissions

- When adding new host patterns, update BOTH `chrome-extension/manifest.ts` AND the content script match patterns.
- Manifest V3 requires explicit host permissions — wildcard patterns need justification for Chrome Web Store review.

## Do NOT

- Do not use `console.log` — use `createLogger()`.
- Do not add `any` types — use `unknown` and narrow.
- Do not import React (automatic JSX runtime is configured).
- Do not put logic in `index.ts` files — re-exports only.
- Do not add inline styles to Shadow DOM components without `useShadowDomStyles`.
- Do not retry MCP operations manually — the circuit breaker handles fault tolerance.
- Do not store sensitive data in `chrome.storage.local` without the storage abstraction layer (`@extension/storage`).
- Do not create new Zustand stores without corresponding types in `pages/content/src/types/stores.ts`.
- Do not skip `super.initialize(context)` / `super.activate()` calls in adapter lifecycle methods.
- Do not read the full store in React components — use `useShallow()` with granular selectors.
- Do not assume extension context is always valid — handle `"Extension context invalidated"` errors gracefully.
- Do not subscribe to stores directly from other stores — use the event bus for cross-store coordination.
- Do not add new event types without adding them to `EventMap` in `pages/content/src/events/event-types.ts`.
