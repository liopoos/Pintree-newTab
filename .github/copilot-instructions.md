# PintreeNewTab Copilot Instructions

## Project Overview

PintreeNewTab is a browser extension built with **WXT** (Web Extension Tools) that transforms browser bookmarks into a beautiful, navigable new-tab page. It's a cross-browser extension (Chrome/Edge/Firefox) using Manifest V3.

**Tech Stack:** WXT, Tailwind CSS, DaisyUI, Sortable.js, IndexedDB, vanilla JavaScript (no framework)

## Architecture

### Core Structure
- **`src/entrypoints/`**: Extension entry points
  - `background/index.ts`: Service worker that opens the extension page on icon click
  - `content/index.ts`: Empty content script (placeholder)
  - `page/`: Main application (new tab page)
    - `index.html`: Single-page application
    - `js/index.js`: ~1900 lines, handles all bookmark operations, UI rendering, search
    - `utils/`: IndexedDB wrapper and utility functions
    - `config/index.js`: Constants for database table names

### Data Architecture
- **Chrome Bookmarks API**: Primary data source (`chrome.bookmarks.getTree()`, `create()`, `update()`, `remove()`, `move()`)
- **IndexedDB**: Caches favicon images and settings (tables: `Icons`, `SetUp`)
- **Data Flow**: Chrome bookmarks → Structured tree → UI rendering → IndexedDB for assets

### Key Design Patterns
1. **Tree Structure**: Bookmarks converted to nested objects with `type: "folder"|"link"`, `id`, `parentId`, `children[]`
2. **Global State**: `firstLayer` (bookmark tree), `BookmarkFolderActiveId` (active folder), `BreadcrumbsList` (navigation path)
3. **Debounced Search**: Search input uses debounce utility from `utils.js`
4. **Sortable.js Integration**: Drag-and-drop for bookmark reordering, syncs with Chrome API via `chrome.bookmarks.move()`

## Development Workflows

### Setup & Running
```bash
npm install           # Install dependencies
npm run dev           # Launch dev mode (auto-opens browser with extension)
npm run dev:firefox   # Firefox-specific development
npm run build         # Production build → .output/ directory
npm run zip           # Create distributable ZIP
```

**Critical**: WXT auto-reloads the extension on code changes. No manual browser reloading needed.

### Build Output
- Built files go to `.output/` (not tracked in git)
- Separate builds for Chrome (`-chrome.zip`) and Firefox (`-firefox.zip`)

### TypeScript Configuration
- `tsconfig.json` extends `.wxt/tsconfig.json` (auto-generated)
- Mix of TypeScript (`.ts` entrypoints) and vanilla JS (`.js` page logic)

## Internationalization (i18n)

**Pattern**: Use `browser.i18n.getMessage("key")` for all UI text
- Message files: `public/_locales/{locale}/messages.json`
- Supported: zh_CN (default), zh_TW, en, ja, ru, ko, de
- HTML elements with class `*_i18n` get auto-populated (see `index.js` line ~681-707)
- Example: `browser.i18n.getMessage("searchResults")`

## Critical Code Conventions

### Bookmark Operations
```javascript
// Always structure bookmarks like this:
const structuredNode = {
  type: children ? "folder" : "link",
  addDate: dateAdded,
  title: title,
  id: id,
  parentId: parentId,
  index: index,
  url: url // only for links
};
```

### IndexedDB Usage
```javascript
import db from "@/entrypoints/page/utils/IndexedDB.js";
// Always open DB before operations:
db.openDB(dbNames).then(() => { /* operations */ });
// Don't forget to close on page unload: window.onbeforeunload
```

### Favicon Handling
- Favicons fetched via `fetchFaviconAsBase64()` and cached in IndexedDB `Icons` table
- Fallback chain: cached → fetch from URL → default SVG
- Function: `getFaviconURL()` in `utils.js`

### Search Implementation
- **Bookmark search**: `searchBookmarks()` filters tree recursively, respects active folder
- **Web search**: Opens new tab to Google/Baidu/Bing
- **AI search**: Opens ChatGPT/Perplexity/Secret Tower with query
- Search clears on empty input, restores previous folder state

## WXT-Specific Details

### Manifest Configuration
- Edit `wxt.config.ts` for permissions, icons, action config
- **Key permissions**: `storage`, `bookmarks`, `favicon`
- **Version**: Update in both `wxt.config.ts` (line 9) and `package.json`

### Path Aliases
- `@/` maps to `src/` (WXT default)
- `~/` maps to project root for assets (`~/assets/svg/google.svg`)

### Multi-Browser Support
- Use `browser.*` API (polyfilled by WXT), not `chrome.*`
- Firefox-specific builds handle API differences automatically

## Common Tasks

### Adding a New UI String
1. Add to all `public/_locales/*/messages.json` files
2. Use `browser.i18n.getMessage("yourKey")` in JS
3. Or add class `yourKey_i18n` to HTML element

### Adding a Search Engine
1. Add SVG to `src/assets/svg/` or `public/images/`
2. Import in `js/index.js`
3. Add case to `searchWeb()` or `searchAI()` function
4. Update search tab UI rendering

### Modifying Bookmark Structure
- **Never** modify bookmarks without syncing to Chrome API
- Order: UI change → `chrome.bookmarks.*` call → update `firstLayer` tree
- Use `findInTree()` and `deleteFromTree()` helpers from `utils.js`

## Testing Notes
- No automated test suite currently
- Manual testing: Load extension in dev mode via `npm run dev`
- Check browser console for errors (IndexedDB, bookmark API failures)
- Test both Chrome and Firefox builds separately

## Gotchas & Quirks
- **Sortable.js**: Must set `data-id` attribute on draggable elements for Chrome API sync
- **IndexedDB versioning**: Increment version in `IndexedDBHelper` constructor to trigger schema updates
- **Tailwind JIT**: Only scans `src/entrypoints/page/index.html` and `js/*.js` (see `tailwind.config.ts`)
- **Favicon fetching**: Fails for some sites (CORS, no favicon); always provide default fallback
- **Global variables**: `firstLayer`, `BookmarkFolderActiveId` are critical—avoid overwriting

## Key Files Reference
- [wxt.config.ts](wxt.config.ts): Extension manifest config
- [src/entrypoints/page/js/index.js](src/entrypoints/page/js/index.js): Core application logic
- [src/entrypoints/page/utils/utils.js](src/entrypoints/page/utils/utils.js): Helper functions for tree traversal, favicon fetching
- [src/entrypoints/page/utils/IndexedDB.js](src/entrypoints/page/utils/IndexedDB.js): IndexedDB wrapper class
- [tailwind.config.ts](tailwind.config.ts): Tailwind + DaisyUI configuration
