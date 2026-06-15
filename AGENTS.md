# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

App Store Screenshot Generator - a browser-based tool for creating App Store marketing screenshots. Built with vanilla JavaScript, HTML5 Canvas, Three.js, and CSS. No build process required.

## Agent Instructions

**Development Server:**
- The agent should automatically start the local development server when needed using `python3 -m http.server 8000` (or `npx serve .` as fallback)
- The agent should run the server in the background and inform the user which URL to open (e.g., `http://localhost:8000`)
- The agent should NOT ask the user to start the server manually
- The agent should monitor server logs to detect and report any errors or problems to the user

**Git & Commits:**
- The agent should handle all git operations automatically (add, commit, push)
- Before creating a commit, the agent MUST show the proposed commit message to the user and wait for approval
- Only after user approval should the agent proceed with the commit
- The agent should follow standard git commit message conventions

## Development

To run locally, serve via a web server (required for browser APIs and project folder access):

```bash
python3 -m http.server 8000
# or
npx serve .
```

Open `http://localhost:8000` in browser. Opening `index.html` directly from filesystem will break persistence.

## Architecture

**Main files:**

- `index.html` - UI structure with modals for settings, about, project management, translations, and language selection
- `styles.css` - Dark theme styling, responsive layout with CSS Grid (3-column: left sidebar, canvas, right sidebar)
- `app-state.js` - Shared app state, text normalization helpers, and element/popout selection accessors (~310 lines)
- `app.js` - Main application glue and legacy logic (~8600 lines as of 2026-06; actively being split)
- `three-renderer.js` - Three.js 3D rendering for iPhone mockups (~1160 lines)
- `language-utils.js` - Language detection, localized image management, and translation dialogs (~565 lines)

**Key patterns in app.js:**

- `app-state.js` now owns the top-level `state` object, text normalization helpers, runtime selection state, and popout/element accessors
- `updateCanvas()` is the main render function - call after any state change
- `saveState()` persists `project.json` into the selected project folder, called automatically in `updateCanvas()`
- `syncUIWithState()` updates all UI controls to reflect current state
- Project management uses the File System Access API. Users must create or open a local project folder before adding assets.
- Per-screenshot settings: each screenshot stores its own background, device, and text settings

**Folder-backed persistence:**

- Project folders are Chrome/Edge-only for now because they use `window.showDirectoryPicker()`.
- Each project folder stores `project.json` plus image assets under `assets/screenshots/`, `assets/backgrounds/`, and `assets/elements/`.
- `project.json` stores config and relative asset paths (`assetPath`, `imagePath`) rather than embedding uploaded images as base64.
- Directory handles are kept in memory for the current browser session. Reopen a project with the folder/open button after a page reload.
- Keep app preferences such as theme and AI provider keys in `localStorage`; those are not project data.

## app.js Declutter Plan

Goal: reduce `app.js` in small, reversible steps while keeping the no-build, script-tag architecture working. Prefer extracting one cohesive cluster at a time into plain global scripts loaded before `app.js`; avoid ES modules until the app has a dedicated migration pass.

**Guardrails:**

- Keep behavior unchanged during extraction. Move code first, rename later only when tests/manual checks are green.
- Preserve existing globals that other files or inline Tauri handlers call (`createProject`, `importScreenshotsFromTauri`, `exportCurrent`, `exportAll`, etc.).
- Add each new file to `index.html` immediately before `app.js`, after any dependency it needs.
- After each extraction, serve the app with `python3 -m http.server 8000` and smoke test load, upload/import, canvas render, project switching, and export.
- Do not mix large extraction work with UI redesign or formatting-only churn.

**Easy-win extraction order:**

1. `app-state.js`
   - Move `state`, `baseTextDefaults`, runtime selection variables, laurel image preload, and state accessors such as `getCurrentScreenshot()`, `getBackground()`, `getScreenshotSettings()`, `getText()`, `normalizeTextSettings()`, element/popout accessors, and setters.
   - Keep these as globals because most of the app reads them directly.
   - Status: implemented. `index.html` now loads `app-state.js` immediately before `app.js`.

2. `app-constants.js`
   - Move static lookup data: `languageFlags`, `languageNames`, `googleFonts`, `deviceDimensions`, DB constants, sidebar constants, zoom constants, and similar pure configuration.
   - Keep `llm.js`, `language-utils.js`, `magical-titles.js`, `lucide-icons.js`, and `three-renderer.js` loaded before files that consume their globals.

3. `app-storage.js`
   - Move folder-backed project lifecycle code: File System Access helpers, `loadProjectsMeta()`, `saveProjectsMeta()`, `saveState()`, `loadState()`, `switchProject()`, `createProject()`, `renameProject()`, `deleteProject()`, and migrations/reconstruction helpers.
   - Leave UI refresh calls in place, but document dependencies on `updateProjectSelector()`, `syncUIWithState()`, and `updateCanvas()`.

4. `app-fonts.js`
   - Move Google font loading and picker code: `loadGoogleFont()`, `fetchAllGoogleFonts()`, `fontPickerState`, `initFontPicker()`, `initSingleFontPicker()`, `renderFontList()`, and font preview helpers.
   - This is a low-risk extraction because the functions are mostly self-contained DOM helpers.

5. `app-elements.js` and `app-popouts.js`
   - Move element creation, element list/properties UI, drag handling, crop preview handling, popout UI, and related draw helpers that are not part of the core canvas pipeline.
   - Keep shared drawing helpers (`roundRect`, `wrapText`, `hexToRgba`) available globally until a later utility pass.

6. `app-rendering.js`
   - Move canvas render pipeline once state/storage/UI extractions are stable: `getCanvasDimensions()`, `getPreviewScale()`, `updateCanvas()`, `updateSidePreviews()`, `renderScreenshotToCanvas()`, `drawBackground*`, `drawScreenshot*`, `drawText*`, `drawNoise*`, `drawElements*`, and `drawPopouts*`.
   - Be careful: `updateCanvas()` also persists via `saveState()`, so storage must already be extracted and loaded first.

7. `app-export.js`
   - Move `exportCurrent()`, `exportAll()`, export progress modal helpers, `exportAllForLanguage()`, and `exportAllLanguages()`.
   - Keep these globals for Tauri menu actions and buttons.

8. `app-modals.js` / `app-translation.js`
   - Move app alert/confirm modals, settings modal/theme handling, languages modal, translate modal, provider translation functions, and AI translate orchestration.
   - Translation code depends on `llm.js` globals and browser API keys stored in localStorage.

9. `app-events.js`
   - Move `initSync()`, `setupEventListeners()`, sidebar resize/collapse, zoom/pan handlers, picker open/close handlers, and final bootstrap wiring.
   - This should be one of the last steps because it touches almost every other cluster.

**Suggested script order after the first several extractions:**

```html
<script src="llm.js"></script>
<script src="language-utils.js"></script>
<script src="magical-titles.js"></script>
<script src="lucide-icons.js"></script>
<script src="three-renderer.js"></script>
<script src="app-constants.js"></script>
<script src="app-state.js"></script>
<script src="app-storage.js"></script>
<script src="app-fonts.js"></script>
<script src="app.js"></script>
```

Only add script tags for files that exist. Keep `app.js` last until the final bootstrap has been moved.

**Canvas rendering pipeline (in updateCanvas):**
1. `drawBackground()` - gradient/solid/image with optional blur and overlay
2. `drawScreenshot()` - positioned, scaled, rotated screenshot with shadow and border
3. `drawText()` - headline and subheadline with multi-language support
4. `drawNoise()` - optional noise texture overlay

**3D rendering (in three-renderer.js):**
- Uses Three.js with GLTFLoader for iPhone 15 Pro Max model
- `initThreeJS()` - initializes scene, camera, renderer, and lights
- `loadPhoneModel()` - loads the 3D iPhone model from `models/iphone-15-pro-max.glb`
- `renderThreeJSToCanvas()` - renders 3D scene to export canvas with full resolution
- `renderThreeJSForScreenshot()` - renders 3D for specific screenshot (side previews)
- Drag-to-rotate interaction on the 3D preview canvas

**Multi-language text:**
- `state.text.headlines` and `state.text.subheadlines` are objects keyed by language code
- `getTextSettings()` returns either global or per-screenshot text depending on toggle state
- AI translation calls Claude/OpenAI/Google API directly from browser (requires API key in settings)

**Localized screenshots (in language-utils.js):**
- Each screenshot has `localizedImages` object keyed by language code (e.g., `{ 'en': {...}, 'de': {...} }`)
- `detectLanguageFromFilename()` - parses suffixes like `_de`, `-fr`, `_pt-br` from filenames
- `getScreenshotImage(screenshot)` - returns image for current language with fallback chain
- `findScreenshotByBaseFilename()` - matches uploads to existing screenshots by base name
- Duplicate detection shows dialog with Replace/Create New/Skip options when uploading matching files

**UI Components:**
- Right sidebar has three tabs: Background, Device, Text
- Collapsible toggle sections for Noise, Shadow, Border, Headline, Subheadline
- Device tab has 2D/3D mode selector with different controls for each mode
- Side preview carousel with sliding animation between screenshots
- Project selector dropdown with screenshot counts
- Gradient editor with draggable color stops

## Key Functions

**Project & Screenshots (app.js):**
- `createProject()` / `deleteProject()` / `switchProject()` - async, must await and call `updateProjectSelector()` after
- `handleFiles()` - processes uploaded images, detects language, shows duplicate dialog if needed
- `createNewScreenshot()` - creates screenshot entry with localized image support
- `exportCurrent()` / `exportAll()` - generates PNG downloads from canvas (ZIP for batch export)
- `applyPositionPreset()` - applies preset screenshot positioning (centered, bleed, tilt, perspective, etc.)
- `transferStyle()` - copies style settings from one screenshot to another
- `slideToScreenshot()` - animates carousel transition between screenshots
- `updateSidePreviews()` - renders adjacent screenshots in side preview canvases

**Language Utils (language-utils.js):**
- `detectLanguageFromFilename()` - extracts language code from filename suffixes
- `getBaseFilename()` - strips language suffix and extension for matching
- `findScreenshotByBaseFilename()` - finds existing screenshot with same base name
- `getScreenshotImage()` - returns localized image for current language with fallbacks
- `addLocalizedImage()` / `removeLocalizedImage()` - manage per-language images
- `showDuplicateDialog()` - async dialog for handling duplicate uploads
- `showExportLanguageDialog()` - dialog for choosing export scope (current/all languages)

## External Dependencies

- **Three.js** (r128) - 3D rendering for device mockups
- **GLTFLoader** - loads iPhone 3D model
- **JSZip** - creates ZIP files for batch export
- **Google Fonts API** - font picker with 1500+ fonts
