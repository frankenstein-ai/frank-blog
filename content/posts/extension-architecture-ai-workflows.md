+++
date = '2026-01-26T11:30:00-03:00'
draft = false
title = 'Chrome Extension Architecture for AI Workflows'
+++

## The challenge

Building AI-powered browser extensions is different from building web apps. You need to decide which components run where, how to bundle dependencies, and how to manage state across different execution contexts.

Get it wrong and you end up with slow performance, data loss, or broken functionality. Get it right and you can run sophisticated AI workflows entirely client-side.

Here's what I learned building Frank Bookmark.

## Extension components: know your environments

Chrome extensions have three main execution contexts, each with different capabilities and constraints.

### Popup (popup.html + popup.js)

The popup handles the user interface for immediate actions: save buttons, search input, quick feedback, and result display.

It has real limitations though. Execution time is limited because the popup closes when users click outside. It cannot run long-running AI inference, cannot access all Chrome APIs, and state doesn't persist when it closes. I use the popup for UI only and forward all heavy operations to the background.

### Background Service Worker (background.js)

This is where the AI lives. The background Service Worker provides a persistent execution context for long-running AI inference, database connections, cross-origin fetching, and state management. It has access to all Chrome APIs, can fetch across origins (with permissions), maintains database connections, and persists between popup open/close cycles. There's no direct UI rendering, but that's fine since the popup handles display.

I load models here, run inference here, and manage data here.

### Content scripts (optional)

Content scripts can inject into web pages and access the page DOM for page-specific functionality. They have limited Chrome API access and page-specific scope. For most AI workflows, I didn't need them. The background worker handles everything.

## The bundling problem

Extensions cannot use ES6 modules directly. No `import/export`, no Node.js package resolution, no native module loading.

This breaks most modern JavaScript libraries, including:
- Transformers.js
- sqlite-vec
- Readability
- Any npm package

### Solution: Vite + Rollup

Bundle all modules into single files for extension contexts:

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      input: {
        background: 'src/background.js',
        popup: 'src/popup.js'
      },
      output: {
        entryFileNames: '[name].js',
        dir: 'extension/dist'
      }
    }
  },
  plugins: [
    // Copy WASM files to dist/
    {
      name: 'copy-wasm',
      writeBundle() {
        // Copy *.wasm files
      }
    }
  ]
}
```

This bundles ES6 modules into extension-compatible files and handles WASM file copying.

## Architecture pattern: self-contained extension

The architecture that worked emerged from trial and error:

```
Extension
├── Popup (UI Layer)
│   ├── Show current URL
│   ├── Save button
│   ├── Search input
│   └── Results display
│
└── Background Service Worker (Logic Layer)
    ├── AI Model (Transformers.js)
    ├── Database (sqlite-vec + OPFS)
    ├── Content Extractor (Readability)
    └── Search Engine
        ├── Keyword search
        ├── Semantic search
        └── Hybrid search
```

### Why this works

The UI lives in the popup where it stays fast and responsive. Heavy lifting happens in the background where execution is persistent and capable. Clean message passing connects the two.

There are no external dependencies. No localhost web app required, no server needed. It works offline after initial install and is fully self-contained and portable.

For performance, the model loads once in the background and stays loaded across popup open/close cycles. Inference happens in a persistent context and the database connection is maintained.

## Data flow

Here's how a user interaction flows through the architecture:

```
User clicks "Save" in popup
  ↓
chrome.runtime.sendMessage({ action: 'save', url })
  ↓
Background worker receives message
  ↓
Fetch URL content (cross-origin allowed)
  ↓
Extract text content (Readability)
  ↓
Generate embedding (Transformers.js)
  ↓
Save to database (sqlite-vec + OPFS)
  ↓
Persist database file (OPFS)
  ↓
chrome.runtime.sendMessage(response)
  ↓
Popup shows "Saved!" confirmation
```

All heavy operations happen in the background. The popup just shows feedback.

## Storage architecture

### Extensions cannot share storage with web apps

This is a critical limitation. Different origins mean separate storage: IndexedDB is not shared, OPFS is not shared, LocalStorage is not shared, and chrome.storage is extension-only.

The implication: build self-contained extensions, not extensions that depend on web apps.

### Recommended pattern

```
Extension has its own database
  ↓
All processing in extension
  ↓
Web app optional (for advanced management)
  ↓
Export/import for sync if needed
```

Don't try to share databases. Build the extension to stand alone.

## AI model loading

Transformers.js works reliably in Service Workers. Here are the patterns I use.

### Load model early

```javascript
// background.js
let model = null;

chrome.runtime.onInstalled.addListener(async () => {
  // Load model in background on install
  model = await loadModel();
  console.log('Model loaded and ready');
});
```

Loading on install means the 3-5 second wait happens once. Every operation after that is fast, and the user experience is much better.

### Handle loading state

```javascript
async function handleSearch(query) {
  if (!model) {
    // Model not loaded yet
    return { status: 'loading', message: 'AI model loading...' };
  }

  // Model ready, perform search
  return await search(query);
}
```

Always check model state before using it.

### Reuse model instance

```javascript
// DON'T: Load model for each request
async function search(query) {
  const model = await loadModel(); // 3-5 seconds every time
  return await model.embed(query);
}

// DO: Reuse loaded model
let model = await loadModel(); // Once

async function search(query) {
  return await model.embed(query); // < 200ms
}
```

Model loading is expensive. Do it once.

## Module organization

Clean structure for extension code:

```
extension/src/
├── background.js          # Service worker entry point
├── popup.js               # Popup UI logic
├── popup.html            # Popup HTML
└── modules/
    ├── storage-vec.js     # Database operations
    ├── extractor.js       # Content extraction
    ├── embeddings.js      # AI model loading/inference
    └── search.js          # Search logic

extension/dist/           # Build output
├── background.js         # Bundled
├── popup.js             # Bundled
├── popup.html           # Copied
└── *.wasm              # WASM files (~34MB)
```

Modules let you organize code. Vite bundles them for the extension.

## Build pipeline

Complete build process:

```bash
# Install dependencies
npm install

# Build extension
npm run build

# Output: extension/dist/
# - background.js (bundled)
# - popup.js (bundled)
# - popup.html
# - *.wasm files
# - manifest.json
```

Vite handles bundling ES6 modules, copying WASM files, tree shaking, and optional minification.

## Performance considerations

### Model loading

Load once, reuse from there. Initial load takes 3-5 seconds. Subsequent inference runs under 200ms. Show a loading state to users and cache the model in the background worker.

### Database operations

sqlite-vec is fast enough for 1,000+ bookmarks. Use indexes on frequently queried fields, batch writes when possible, and save after critical operations.

### Search performance

Real-world measurements with Frank Bookmark:
- Keyword: < 100ms
- Semantic: < 200ms (after model load)
- Hybrid: < 300ms (after model load)

All fast enough for a responsive UI.

## Common pitfalls

### Long operations in popup

The popup can close at any time, killing whatever was running. Forward heavy work to the background worker instead.

```javascript
// DON'T: Long operation in popup
async function save() {
  await longRunningAITask(); // Popup might close
}

// DO: Forward to background
async function save() {
  chrome.runtime.sendMessage({ action: 'save' });
}
```

### Trying to share storage

Extensions and web apps can't share databases. Build a self-contained extension instead.

```javascript
// DON'T: Try to share database
const sharedDb = await openSharedDatabase(); // Won't work

// DO: Extension has its own database
const extensionDb = await openDatabase(); // Works
```

### Direct module loading

Extensions don't support ES6 modules. Bundle with Vite.

```javascript
// DON'T: Direct import in extension
import { model } from 'transformers'; // Won't work

// DO: Bundle with Vite
// vite.config.js handles bundling
```

### Not handling Service Worker lifecycle

The Service Worker can terminate. Persist state to OPFS.

```javascript
// Save state before termination
chrome.runtime.onSuspend.addListener(() => {
  saveDatabase();
});

// Restore state on restart
chrome.runtime.onStartup.addListener(() => {
  loadDatabase();
});
```

## Testing strategy

### Test in extension context

Don't assume web app testing is enough. Load the extension in Chrome and test with the popup opening and closing, with Service Worker restarts, and with OPFS persistence.

### Test at scale

Use realistic data: 1,000+ bookmarks, large text content, multiple searches, and extended sessions.

## Deployment

Package for Chrome Web Store:

```bash
# Build extension
npm run build

# Create ZIP
cd extension
zip -r frank-bookmark.zip dist/*

# Upload to Chrome Web Store
```

Or distribute directly. Users download the ZIP, extract to a folder, load it as an unpacked extension in Chrome, and it works immediately.

## What I learned

The background Service Worker is where AI lives. I designed the entire architecture around it, and that turned out to be the right call. Bundling is not optional since ES6 modules don't work in extensions, so Vite or Rollup is a requirement. Self-contained extensions that don't depend on localhost web apps work better than hybrid setups. Loading AI models on install rather than on first use makes a noticeable difference in how the extension feels. And OPFS persistence works well in practice.

## Recommendations

For building AI-powered Chrome extensions, here is what I'd suggest:

Keep the popup for UI only. Put all AI work in the Background Service Worker. Make the extension self-contained with no external dependencies.

For the build system, use Vite for bundling with multiple entry points (background, popup) and WASM file copying.

For AI integration, load models in the background on install, reuse model instances, and handle loading states gracefully.

For storage, use sqlite-vec for the database and vectors, OPFS for persistence, and don't try to share storage with web apps.

For testing, test in the extension context, test at scale with 1,000+ items, and test Service Worker lifecycle behavior.

## Wrapping up

Building AI-powered Chrome extensions requires understanding the unique constraints of the extension environment. You can't treat it like a web app.

The background Service Worker is where AI lives. Bundling is required for modern JavaScript. Self-contained extensions work best. OPFS provides reliable persistence. Model preloading improves the user experience.

Get the architecture right and you can build sophisticated AI workflows that run entirely in the browser, with good performance and complete privacy. Frank Bookmark is proof that this architecture works at scale.
