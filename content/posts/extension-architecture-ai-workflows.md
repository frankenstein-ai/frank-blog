+++
date = '2026-01-26T11:30:00-03:00'
draft = false
title = 'Chrome Extension Architecture for AI Workflows'
+++

## The Challenge

Building AI-powered browser extensions is different from building web apps. You need to carefully decide which components run where, how to bundle dependencies, and how to manage state across different execution contexts.

Get it wrong, and you'll have slow performance, data loss, or broken functionality. Get it right, and you can run sophisticated AI workflows entirely client-side.

Here's what we learned building Frank Bookmark.

## Extension Components: Know Your Environments

Chrome extensions have three main execution contexts, each with different capabilities and constraints.

### Popup (popup.html + popup.js)

**What it's good for:**
- User interface for immediate actions
- Save buttons
- Search input
- Quick feedback
- Result display

**Limitations:**
- Limited execution time (closes when user clicks outside)
- Cannot run long-running AI inference
- Cannot access all Chrome APIs
- State doesn't persist when closed

**Key insight:** Use popup for UI only. Forward all heavy operations to the background.

### Background Service Worker (background.js)

**What it's good for:**
- Persistent execution context
- Long-running AI inference
- Database connections
- Cross-origin fetching
- State management

**Capabilities:**
- Access to all Chrome APIs
- Can fetch across origins (with permissions)
- Maintains database connections
- Persists between popup open/close
- No direct UI rendering

**Key insight:** This is where your AI lives. Load models here, run inference here, manage data here.

### Content Scripts (Optional)

**What they're good for:**
- Injecting into web pages
- Accessing page DOM
- Page-specific functionality

**Limitations:**
- Limited Chrome API access
- Page-specific (not global)
- Security restrictions

**Key insight:** Not needed for most AI workflows. Use background worker instead.

## The Bundling Problem

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

## Architecture Pattern: Self-Contained Extension

The winning architecture emerged from our experiments:

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

### Why This Works

**Separation of concerns:**
- UI in popup (fast, responsive)
- Heavy lifting in background (persistent, powerful)
- Clean message passing between them

**No external dependencies:**
- No localhost web app required
- No server needed
- Works offline after initial install
- Self-contained and portable

**Performance:**
- Model loads once in background
- Stays loaded across popup open/close
- Inference happens in persistent context
- Database connection maintained

## Data Flow

User interaction flows through the architecture:

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

## Storage Architecture

### Extensions Cannot Share Storage with Web Apps

Critical limitation: Different origins = separate storage:
- IndexedDB not shared
- OPFS not shared
- LocalStorage not shared
- chrome.storage is extension-only

**Implication:** Build self-contained extensions, not extensions that depend on web apps.

### Recommended Pattern

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

## AI Model Loading

Transformers.js works reliably in Service Workers. Key patterns:

### Load Model Early

```javascript
// background.js
let model = null;

chrome.runtime.onInstalled.addListener(async () => {
  // Load model in background on install
  model = await loadModel();
  console.log('Model loaded and ready');
});
```

Loading on install means:
- 3-5 second wait happens once
- Subsequent operations are fast
- Better user experience

### Handle Loading State

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

### Reuse Model Instance

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

## Module Organization

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

Modules let you organize code, Vite bundles them for the extension.

## Build Pipeline

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

Vite handles:
- Bundling ES6 modules
- Copying WASM files
- Tree shaking
- Minification (optional)

## Performance Considerations

### Model Loading

Load once, reuse forever:
- Initial load: 3-5 seconds
- Subsequent inference: < 200ms
- Show loading state to users
- Cache in background worker

### Database Operations

sqlite-vec performance:
- Fast enough for 1,000+ bookmarks
- Use indexes on frequently queried fields
- Batch writes when possible
- Save after critical operations

### Search Performance

Real-world measurements:
- Keyword: < 100ms
- Semantic: < 200ms (after model load)
- Hybrid: < 300ms (after model load)

All fast enough for responsive UI.

## Common Pitfalls

### 1. Long Operations in Popup

**Problem:** Popup closes, operation killed.

**Solution:** Forward to background worker.

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

### 2. Trying to Share Storage

**Problem:** Extension and web app can't share databases.

**Solution:** Build self-contained extension.

```javascript
// DON'T: Try to share database
const sharedDb = await openSharedDatabase(); // Won't work

// DO: Extension has its own database
const extensionDb = await openDatabase(); // Works
```

### 3. Direct Module Loading

**Problem:** Extensions don't support ES6 modules.

**Solution:** Bundle with Vite.

```javascript
// DON'T: Direct import in extension
import { model } from 'transformers'; // Won't work

// DO: Bundle with Vite
// vite.config.js handles bundling
```

### 4. Not Handling Service Worker Lifecycle

**Problem:** Service Worker can terminate.

**Solution:** Persist state to OPFS.

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

## Testing Strategy

### Test in Extension Context

Don't assume web app testing is enough:
- Load extension in Chrome
- Test with popup open/close
- Test with Service Worker restart
- Test with OPFS persistence

### Test at Scale

Use realistic data:
- 1,000+ bookmarks
- Large text content
- Multiple searches
- Extended sessions

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

Or distribute directly:
- Users download ZIP
- Extract to folder
- Load unpacked extension in Chrome
- Extension works immediately

## Lessons Learned

### 1. Background Worker is Key

This is where AI lives. Design your architecture around it.

### 2. Bundling is Required

ES6 modules don't work in extensions. Use Vite or Rollup.

### 3. Self-Contained Wins

Don't depend on localhost web apps. Build everything into the extension.

### 4. Model Preloading Matters

Load AI models on install, not on first use. Better UX.

### 5. OPFS is Production-Ready

Persistent storage works well. Use it.

## Recommendations

For building AI-powered Chrome extensions:

**Architecture:**
- Popup for UI only
- Background Service Worker for AI
- Self-contained (no external dependencies)

**Build System:**
- Vite for bundling
- Multiple entry points (background, popup)
- WASM file copying

**AI Integration:**
- Load models in background on install
- Reuse model instances
- Handle loading states gracefully

**Storage:**
- sqlite-vec for database + vectors
- OPFS for persistence
- Don't try to share with web apps

**Testing:**
- Test in extension context
- Test at scale (1,000+ items)
- Test Service Worker lifecycle

## Conclusion

Building AI-powered Chrome extensions requires understanding the unique constraints of the extension environment. You can't treat it like a web app.

The key insights:
- Background Service Worker is where AI lives
- Bundling is required for modern JavaScript
- Self-contained extensions work best
- OPFS provides reliable persistence
- Model preloading improves UX

Get the architecture right, and you can build sophisticated AI workflows that run entirely in the browser, with excellent performance and complete privacy.

Frank Bookmark proves this architecture works at scale with production-ready performance.
