# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

YouTube Row Fixer is a Manifest V3 browser extension for Chrome and Firefox. It changes how many videos, posts and shorts YouTube shows per row, and adds a few cosmetic options (hide shorts, hide channel avatars, full titles, wide channel layout). There is no test suite.

## Commands

- `npm run dev`: Vite + `@crxjs/vite-plugin` dev server using `extension/manifest-dev.json`. Load `dist/` as an unpacked extension. This mainly serves popup UI work (see the dev-mode caveat below).
- `npm run test`: **not tests.** It runs webpack, which bundles the background/content/inject scripts into `build/`.
- `npm run build`: `next lint`, then `next build` and `next export -o build` (the popup), then webpack, then `node utils/build.js`. CONTRIBUTING says to run `npm run test && npm run build`. Load `build/` unpacked to test the real extension.
- `npm run zip`: writes `zip/Chrome v<version>.zip` and `zip/Firefox v<version>.zip` from `build/`. It swaps in the matching manifest for each and reads the version from `extension/manifest-chrome.json`.
- `npm run prettier`: formats `src/`.
- Lint only: `npx next lint` (uses `next/core-web-vitals`).

## Build pipeline

Two separate bundlers write into `build/`:

1. **Next.js 12 static export** builds the popup (`src/pages/index.jsx` → `src/App.jsx`). `utils/build.js` then post-processes the output so MV3 accepts it:
   - It copies `extension/manifest-chrome.json` to `build/manifest.json`.
   - It renames `_next/` to `next/` and rewrites every `/_next/` reference, because extensions reject paths that start with `_`.
   - It pulls inline `<script>` tags out of `index.html` into `inline_script*.js` files, because MV3's CSP blocks inline scripts.
2. **Webpack** (`webpack.config.js`) bundles the non-UI entries. The output paths are load-bearing: `background.js`, `content_script.js`, `inject/bridge.js`, `inject/inject_script.js` and `inject/lib/ytZara.js`. Both `background.js` and the manifests refer to these exact paths.

The `@` alias points to `src/`. In dev mode, `index.html` + `src/index.jsx` render the same `App` that Next renders in the production build.

**Manifests:** `extension/manifest-chrome.json` and `extension/manifest-firefox.json` must be kept in sync. Version bumps go in both. Firefox uses `background.scripts` instead of `service_worker`, and it adds `browser_specific_settings.gecko`. `manifest-dev.json` is only used by Vite and is not kept up to date.

## Runtime architecture

Four execution contexts talk to each other through `chrome.storage.local`:

- **Popup** (`src/components/**`): each control is a `SwitchControl` or `SliderControl` bound to a storage key through `useStorageState` (`src/hooks/useStorage.js`). The popup writes to storage and nothing else.
- **Background** (`src/background/background.js`): on startup and on every storage change, it uses `chrome.scripting.registerContentScripts` to register the content scripts dynamically when `extensionStatus` is on, and unregisters them when it is off. It registers two scripts:
  - `inject/bridge.js` in the ISOLATED world.
  - `inject/lib/ytZara.js` + `inject/inject_script.js` in the **MAIN** world.
  Because these registrations point to webpack output filenames, the extension's main feature only works from a `build/` load.
- **Static content script** (`src/content-scripts/main.js`, declared in the manifest, runs at `document_start`): handles the CSS-only options. Each option module in `src/content-scripts/modules/options/` calls `addStyle(id, css)`, which inserts a `<style id=storageKey class="RFYT">`. Turning an option off calls `removeElementById(storageKey)`. `optionsChanges.js#injectAllChanges` re-applies all of them whenever storage changes.
- **MAIN-world inject script** (`src/content-scripts/inject/main.js`): the core rows-per-row logic. It uses the vendored `ytZara.ytProtoAsync` library (by cyfung1031, MIT) to get YouTube's Polymer prototypes and monkey-patches `ytd-rich-grid-renderer`:
  - It overrides `calcElementsPerRow`, `calcMaxSlimElementsPerRow` and `refreshGridLayout`. The magic widths `194` (shorts/slim items) and `310` (breaking news) identify which kind of shelf is being calculated.
  - It patches `ytd-rich-shelf-renderer.refreshGridLayoutNew` to a no-op.
  - `reflowLayout` also sets the `--ytd-rich-grid-*-per-row` CSS variables directly on existing grids.
  - "Dynamic" mode replaces the user's values with fixed counts at the sm/md/lg breakpoints (640/768/1000px).

  The MAIN world can't use `chrome.*`, so `bridge.js` relays settings over `window` CustomEvents (`src/content-scripts/modules/utils/port.js`, event names in `src/data/event.js`). The inject script dispatches `getRowFixerData`. The bridge replies with `sendRowFixerData`, whose payload is JSON-stringified as `{data}`, and it pushes the same event again on every storage change.

## Settings keys

`src/data/storage-key.js` is the single source of truth. When you add a setting, you need to:

- add a `Key*` constant,
- add it to `settingKey` (the list that gets fetched),
- add a default to `defaultSetting`,
- add a popup control in `Main.jsx`,
- either add a CSS option module and wire it into `injectAllChanges`, or map it in `inject/main.js#handleDataEvent`.

Storage reads fall back to `defaultSetting`. There are two near-identical storage helpers: `src/utils/browserStorage.js` (popup/background) and `src/content-scripts/modules/utils/storage.js` (content scripts).

## Fixing YouTube layout breakage

Most fixes add CSS selectors to the option modules (for example, commits for #90 and #91). YouTube ships several class-name generations of the same element side by side: `ytLockupMetadataViewModel*` (camelCase), `yt-lockup-metadata-view-model--*` and `yt-lockup-metadata-view-model-wiz--*`. Add the new selector next to the existing ones and keep the old ones. Pages are scoped with `[page-subtype="home"|"subscriptions"|"channels"|...]`. `hideShort.js` lists test URLs.
