# Repository Guidelines

## Project Structure & Module Organization
Storyboarder is an Electron app with multiple webpack entry points. Core app code lives in `src/js`; renderer HTML in `src/*.html`; styles, images, fonts, sounds, and data in `src/css`, `src/img`, `src/fonts`, `src/snd`, and `src/data`. Webpack configs are grouped by surface in `configs/`. The companion server lives in `server/src`. Tests are under `test/`, organized by feature. Build and release assets are in `build/`; automation scripts are in `scripts/`.

## Build, Test, and Development Commands
- `npm install`: install app dependencies and Electron native app dependencies via `postinstall`.
- `npm start`: run all development watchers plus Electron in parallel.
- `npm run start:electron`: launch only the Electron shell in development mode.
- `npm run start:shot-generator`: watch the shot-generator renderer bundle.
- `npm run start:server`: run the server package from `server/`.
- `npm run build`: production-build all webpack targets and the server.
- `npm test`: run Mocha, Electron renderer tests, and Electron main-process tests.
- `npm run dist:mac|dist:win|dist:linux`: build platform installers after `npm run build`.

## Coding Style & Naming Conventions
Use JavaScript/React patterns already present in `src/js`; prefer CommonJS where surrounding files use it. Keep modules focused, place feature files near their existing directory, and use lowercase hyphenated filenames for new scripts or assets. Maintain existing indentation and semicolon style. Use `classnames` for conditional CSS classes.

## Testing Guidelines
Tests use Mocha plus `electron-mocha`. Name tests by runtime: `*.renderer.test.js` for renderer specs, `*.main.test.js` or `*.test.main.js` for main-process specs, and `*.test.js` for plain Node tests. Reset fixture experiments with `npm run clean:fixtures`.

## Commit & Pull Request Guidelines
Recent history uses short imperative summaries, sometimes scoped, such as `print-project: only send READY on first render`. Prefer concise Conventional Commit style when possible: `feat: add board export option`, `fix: prevent missing audio duration`. PRs should include a clear problem statement, implementation notes, test results (`npm test`, targeted commands), linked issues, and screenshots or screen recordings for UI changes.

## Security & Configuration Tips
Do not commit secrets, signing certificates, generated installers, or local preference files. Treat Electron permissions, file-system access, auto-update behavior, and `server/` routes as security-sensitive; document any changed trust boundary in the PR.
