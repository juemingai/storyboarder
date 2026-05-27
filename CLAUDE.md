# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Storyboarder is an **Electron desktop app** for creating storyboards. It uses Node.js + Electron 18, React 16, Redux, and Three.js (via react-three-fiber) for a 3D Shot Generator feature.

## Commands

```bash
# Install dependencies (both root and server)
cd server && npm install && cd .. && npm install

# Start development (runs all webpack watchers + electron in parallel)
npm start

# Start with a specific storyboard file
npm start ./test/fixtures/example.storyboarder

# Run only the Electron process (when watchers are already running)
npm run start:electron

# Watch-build a single renderer bundle
npm run start:shot-generator   # or start:xr, start:ar, start:shot-explorer, etc.

# Run tests
npm test
# mocha (Node/main process tests): test/*[!renderer].test.js
# electron-mocha renderer: test/*.renderer.test.js, test/**/*.renderer.test.js
# electron-mocha main: test/**/*.main.test.js

# Run a single mocha test file
npx mocha test/models/board.test.js

# Restore test fixtures after a test run modifies them
npm run clean:fixtures

# Build all webpack bundles (production)
npm run build

# Build distributable
npm run dist:mac
npm run dist:win
npm run dist:linux

# Quick build without code signing (for local testing)
CSC_IDENTITY_AUTO_DISCOVERY=false electron-builder build -m --arm64 --x64 --dir
```

## Architecture

### Process Structure

Electron splits code into a **main process** and multiple **renderer processes** (each a BrowserWindow):

- `src/js/main.js` — Electron main process entry. Creates/manages all windows, handles IPC, file system, menu, auto-updater.
- `src/js/window/` — UI views loaded directly into the main storyboard BrowserWindow (non-webpack, CommonJS `require`). Contains the core drawing canvas, toolbar, timeline, scene panel etc.
- `src/js/windows/` — One subdirectory per feature window (shot-generator, print-project, preferences, etc.). Each has a `main.js` (main-process controller) that creates a `BrowserWindow`.
- `src/*.html` — Each BrowserWindow loads one of these HTML entry points (e.g. `shot-generator.html`, `main-window.html`).

### IPC Communication

Windows communicate via Electron IPC. The pattern is:
- `ipcMain.on(channel, handler)` in main process
- `ipcRenderer.send(channel, data)` / `ipcRenderer.invoke(channel, data)` in renderer
- Feature windows like Shot Generator also have a `service.js` alongside their `window.js` that acts as an IPC proxy to the main window

### Webpack Bundles

Each renderer with complex UI is bundled independently via webpack. Configs live in `configs/`:
- `shot-generator` — 3D scene editor (Three.js/React Three Fiber)
- `shot-explorer` — shot comparison view
- `xr` / `ar` — VR/AR views
- `print-project` — print layout
- `language-preferences` — language settings UI

### State Management

Redux is shared across Electron processes using **electron-redux** (`composeWithStateSync`). The store in `src/js/shared/store/configureStore.js` applies this in both main and renderer contexts, so dispatching an action in a renderer automatically syncs to main and all other renderers.

Reducers are in `src/js/shared/reducers/`:
- `shot-generator.js` — scene objects, camera, lights, characters, etc. (~1500 lines, includes all selectors)
- `toolbar.js` — active tool state
- `preferences.js` — user preferences
- `remoteDevice.js` — mobile companion state

### Shot Generator

The Shot Generator is the most complex feature — a 3D scene editor using React Three Fiber:
- `src/js/shot-generator/SceneManagerR3fLarge.js` — main perspective camera view
- `src/js/shot-generator/SceneManagerR3fSmall.js` — top-down orthographic view
- `src/js/shot-generator/components/Editor/` — root React component, layout, screenshot trigger
- `src/js/shared/IK/SGIkHelper.js` — IK system singleton managing character rag-doll poses

### Shared Code

`src/js/shared/` is used by both main and renderer processes:
- `actions/` — Redux action creators
- `helpers/` — key maps, store observers, utilities
- `IK/` — inverse kinematics for character posing (Three.js)
- `network/` — P2P networking for mobile companion

### Domain Models

`src/js/models/` — pure JS models for `board.js`, `scene.js`, `shot-list.js`, `watermark.js`.

### Server

`server/` is a separate Node.js Express/WebSocket server that enables a mobile companion app. It has its own `package.json` and webpack config. Run with `npm run start:server`.

### Exporters / Importers

`src/js/exporters/` contains export logic (Final Cut Pro X, PDF, PSD, archive, FFmpeg video).  
`src/js/importers/` contains import logic (Final Draft FDX, Fountain screenplay format).

## Testing

Test files follow runtime-specific suffix conventions:
- `*.test.js` — plain Node (mocha)
- `*.renderer.test.js` — Electron renderer process (electron-mocha)
- `*.main.test.js` — Electron main process (electron-mocha)

Tests are colocated under `test/` mirroring the source path (e.g. `src/js/exporters/pdf/` → `test/exporters/pdf/`). Fixture files live in `test/fixtures/`; always restore them after modification with `npm run clean:fixtures`.

## Key Conventions

- **CommonJS throughout**: The project uses `require`/`module.exports`, not ES modules. The only exception is some Three.js utilities that export ES module format.
- **No TypeScript**: Plain JavaScript everywhere.
- **Commit style**: Short imperative subjects with subsystem prefix, e.g. `print-project: align footer text` or `shot-generator: fix camera reset`.
- **Vendored libraries**: `src/js/vendor/` holds patched forks of `tether-drop` and `tether-tooltip` — do not replace with npm versions.
- **File format**: `.storyboarder` files are JSON with associated `images/` folders containing PNG boards per scene.
- **Preferences**: Stored at `~/Library/Application Support/Storyboarder/pref.json` (Mac). Use `npm run prefs` to print the path.
