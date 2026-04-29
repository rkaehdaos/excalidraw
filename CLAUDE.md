# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Structure

Excalidraw is a **monorepo** with a clear separation between the core library and the application:

- **`packages/excalidraw/`** - Main React component library published to npm as `@excalidraw/excalidraw`
- **`excalidraw-app/`** - Full-featured web application (excalidraw.com) that uses the library
- **`packages/common/`** - Shared utilities and constants (no internal deps)
- **`packages/math/`** - Geometry/vector math (depends on `@excalidraw/common`)
- **`packages/element/`** - Element types and rendering logic (depends on `common`, `math`)
- **`packages/utils/`** - Font subsetting and export utilities
- **`examples/`** - Integration examples (NextJS, browser script)

**Package dependency order** (important for builds): `common → math → element → excalidraw → excalidraw-app`

## Development Commands

```bash
# Dev server (port 3001)
yarn start

# Build
yarn build              # Full app build
yarn build:packages    # Build all packages in dependency order

# Testing
yarn test              # Run vitest once
yarn test:update       # Run tests with snapshot updates (use before committing)
yarn test:ui           # Launch Vitest UI
yarn test:typecheck    # TypeScript type checking only
yarn test:all          # typecheck + eslint + prettier + tests

# Run a single test file
yarn vitest packages/excalidraw/tests/charts.test.ts
yarn vitest packages/element/tests/bounds.test.ts

# Code quality
yarn fix               # Auto-fix prettier + eslint (run before committing)
yarn fix:code          # eslint --fix only
```

## Architecture

### State Management

Uses **Jotai** (atomic state) with two separate scopes — never mix them:
- **`editor-jotai`** (`packages/excalidraw/editor-jotai.ts`) — atoms for the library/component
- **`app-jotai`** (`excalidraw-app/app-jotai.ts`) — atoms for the full web app

ESLint enforces this: direct `jotai` imports are banned; use `editor-jotai` or `app-jotai` instead.

### Rendering Pipeline

Canvas-based rendering with clear layer separation:
- `packages/excalidraw/renderer/` — canvas drawing logic
- `packages/excalidraw/scene/` — scene graph and element management
- Element geometry math lives in `@excalidraw/element` and `@excalidraw/math`

### Import Restrictions

ESLint enforces import order: `builtin → external → @excalidraw external → internal`

Inside `packages/excalidraw/**`: **no imports from barrel `index.tsx` files** — always use relative imports to specific files.

Type imports must use `import type` syntax (separate-type-imports rule).

### Build System

- **Packages** (`common`, `math`, `element`, `excalidraw`): esbuild via `scripts/buildBase.js` and `scripts/buildPackage.js`
  - Outputs to `dist/dev/` (sourcemaps) and `dist/prod/` (minified)
  - Package.json `exports` field selects dev/prod build via `development`/`production` conditions
- **App** (`excalidraw-app/`): Vite with TypeScript checking, SVG-as-React-components, PWA support, and manual chunk splitting for locales/CodeMirror/Mermaid

### Key Files

- `packages/excalidraw/index.tsx` — public API entry point for the npm package
- `excalidraw-app/App.tsx` — main app orchestration (~1000 lines)
- `packages/excalidraw/appState.ts` — `AppState` type definition (~400 lines)
- `setupTests.ts` — global Vitest setup (Canvas mock, IndexedDB, FontFace polyfill)
- `vitest.config.mts` — test config with path aliases resolving `@excalidraw/*` to `src/` in dev

### Pre-commit Hooks (lint-staged)

- `*.{js,ts,tsx}` → `eslint --fix`
- `*.{css,scss,json,md,html,yml}` → `prettier --write`

Run `yarn fix` manually if hooks don't catch everything.
