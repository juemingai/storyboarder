# Repository Guidelines

## Project Structure & Module Organization
Storyboarder is an Electron application with multiple webpack-built renderer targets. Core application code lives in `src/js`, with Electron main-process entry points under `src/js/main`, UI windows under `src/js/window` and `src/js/windows`, shared state/utilities under `src/js/shared`, domain models in `src/js/models`, import/export logic in `src/js/importers` and `src/js/exporters`, and Shot Generator/XR/AR features in their matching folders. Static app assets and bundled data live in `src/data`, packaging resources in `build`, webpack configs in `configs`, automation in `scripts`, the auxiliary server in `server`, and tests in `test`.

## Build, Test, and Development Commands
- `npm install`: install dependencies and Electron native app dependencies via `postinstall`.
- `npm start`: run all development watchers plus Electron in parallel.
- `npm run start:electron`: launch Electron only; useful after a watcher is already running.
- `npm run start:shot-generator`: watch-build only the Shot Generator renderer.
- `npm run build`: production-build all renderer/server targets.
- `npm test`: run Mocha, Electron renderer tests, and Electron main-process tests.
- `npm run dist:mac`, `npm run dist:win`, `npm run dist:linux`: create platform packages after building.

## Coding Style & Naming Conventions
Use JavaScript/React patterns already present in the codebase and keep modules focused. Prefer `const`/`let`, CommonJS or existing local import style as appropriate for the touched file, and 2-space indentation. Name tests and modules after the feature they cover, for example `src/js/exporters/pdf/...` with `test/exporters/pdf/...`. Keep assets descriptive and colocated with their feature when possible.

## Testing Guidelines
Tests use Mocha plus `electron-mocha`. Place unit-style tests near the matching domain area inside `test`, and use suffixes that match the runtime: `*.renderer.test.js` for renderer tests, `*.main.test.js` for Electron main tests, and `*.test.js` for plain Node tests. Before committing, run `npm test`; for fixture-heavy work, restore fixtures with `npm run clean:fixtures`.

## Commit & Pull Request Guidelines
Recent history favors short, imperative commit subjects such as `print-project: align footer text to bottom` or `show audio duration in msecs on audio control`. Keep commits scoped to one change and include the subsystem prefix when helpful. Pull requests should include a concise description, linked issue if applicable, test results, and screenshots or recordings for UI-visible changes. Contributors may need to sign the project CLA before merge.

## Security & Configuration Tips
Do not commit secrets, signing credentials, generated release artifacts, or local preference files. Use existing scripts for packaging and notarization paths, and keep platform-specific configuration under `build` or `configs`.
