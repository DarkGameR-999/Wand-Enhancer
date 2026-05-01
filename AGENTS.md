INFO ./docs/*

# Wand Enhancer Agent Notes

This repository patches the Wand Electron app from a .NET Framework WPF desktop tool. Keep changes narrow and preserve the patch pipeline invariants.

## Remote Web Panel

- The default local remote port is `3223`. Keep C# and frontend constants aligned.
- The embedded panel must stay small because the desktop patcher embeds it and then injects it into Wand's `app.asar`.
- Production builds must not include mock data, debug routes, sourcemaps, local fonts, heavy icon libraries, or runtime class helper packages.
- Mock/demo data is dev-only and must be reached through `import.meta.env.DEV` dynamic imports.
- Source can use React-compatible imports, but production runtime resolves them to Preact aliases in `web-panel/vite.config.ts`.
- UI uses Tailwind CSS and lightweight shadcn-style local primitives under `web-panel/src/components/ui/`.
- Default renderer scripts live in `web-panel/scripts/default/` and are embedded. Custom user scripts are selected in the WPF patch modal and copied from `PatchConfig.CustomScriptPaths`; only existing `.js` files are accepted. A local `renderer-scripts/` folder next to the patcher exe is still copied as an advanced fallback.

## ASAR Patch Pipeline

- Preserve and restore both `resources/app.asar` and `resources/app.asar.unpacked` backups.
- Inject `web-panel/dist` as `remote-panel/`, `web-panel/bridge/wand-remote-bridge.cjs` as `remote-panel/bridge.cjs`, and default/selected/local renderer scripts under `remote-panel/renderer-scripts`.
- Do not commit extracted `.sources/` output. Recreate it only for reverse-engineering sessions.
- `AsarSharp.AsarExtractor.ExtractAll` must skip unpacked entries when their source path equals the destination (in-place extraction is a self-copy that fails on locked files like `TrainerLib_x64.dll`) and silently skip unpacked entries whose source is missing on disk (e.g. `auxiliary/GameLauncher.exe` removed by an installer). Do not reintroduce hard failure on either case.
- The `DevToolsOnF12` patch anchors on the Electron main-process `<app>.whenReady().then(` site and attaches a `before-input-event` hook to every `BrowserWindow.webContents`. Do not patch the renderer keydown listener — the minified `ACTION_OPEN_DEV_TOOLS` dispatch site is not stable across Wand releases.
- Cheats can be pinned per game in the web panel via `pinned-storage.ts` (`localStorage` key `wand-remote.pinned-cheats.v1:<gameId>`). Pinned cheats render as a virtual `pinned` category at the top of the list; their normal category placement is preserved.

## Validation

- Web panel build: `cd web-panel && pnpm run build`.
- Bridge syntax checks: `node --check web-panel/bridge/wand-remote-bridge.cjs` and `node --check web-panel/scripts/default/remote-popup-cleanup.js`.
- Production dist should contain only static assets and should not contain `mock-instance`, `Mock Adventure`, `Simulation`, `Debug session`, `mock=1`, `demo-session`, `vite.svg`, `tailwind-merge`, `class-variance-authority`, or `clsx`.