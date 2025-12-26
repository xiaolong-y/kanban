# Development Guide

Reflections and notes for maintaining and extending this project.

## Architecture Decisions

### Single HTML File

**Decision**: Everything in one `index.html` - HTML, CSS, JS, no external dependencies.

**Why**:
- Zero build step = zero build breakage
- Works offline, works forever
- User can download and own their tool
- No npm, no webpack, no framework churn

**Trade-off**: Can't use TypeScript, modern bundling, or component frameworks. Acceptable for a tool this size (~2000 lines).

### Vanilla JS Over Frameworks

**Decision**: No React, Vue, or other frameworks.

**Why**:
- DOM operations are simple (render a list of cards)
- State is a single object (`S.cards`)
- No component lifecycle complexity
- Framework would add 100KB+ for no benefit

**Pattern used**: Simple MVC with:
- `S` = state object
- `render()` = rebuild DOM from state
- Event handlers mutate state and call `render()`

### LocalStorage as Primary Store

**Decision**: `localStorage` is the source of truth, sync is secondary.

**Why**:
- Works immediately with zero setup
- Offline-first by default
- Sync failures don't lose data

**Implementation**:
```javascript
const save = () => {
  localStorage.setItem(KEY, JSON.stringify({ v: 3, cards: S.cards }));
  scheduleSyncSave(); // Async, can fail silently
};
```

### Last-Write-Wins Sync

**Decision**: No conflict resolution - most recent save overwrites.

**Why**:
- Single-user tool, conflicts are rare
- Merge logic adds complexity and edge cases
- Users understand "refresh before editing on another device"

**Alternative considered**: Card-level timestamps with merge. Rejected as over-engineering for personal use.

## Sync Layer Details

### Three-Tier Architecture

```
Tier 1: File System API (best privacy, Chrome/Edge only)
    ↓ fallback
Tier 2: GitHub Gist (cross-browser, needs token)
    ↓ fallback
Tier 3: Manual JSON export/import (universal)
```

### Gist Sync Flow

```
On token entry:
1. Validate token (GET /user)
2. Search for existing kanban gist (GET /gists)
3. If found → load from it
4. If not found → create new gist

On save (debounced 5s):
1. If no gist ID → search again (handles new devices)
2. PATCH /gists/{id} with current cards
```

### Critical: Gist Discovery

**Problem solved**: User enters token on new device, app creates duplicate gist.

**Solution**: `findExistingKanbanGist()` searches user's gists for one with:
- Filename: `kanban.json`
- Description: `Kanban Board Data`

```javascript
async function findExistingKanbanGist() {
  const res = await fetch('https://api.github.com/gists?per_page=100', {
    headers: { Authorization: `token ${SYNC.gistToken}` }
  });
  const gists = await res.json();
  for (const gist of gists) {
    if (gist.files['kanban.json'] && gist.description === 'Kanban Board Data') {
      return gist.id;
    }
  }
  return null;
}
```

## Known Gotchas

### 1. GitHub Pages Cache

**Symptom**: Pushed fix but site still broken.

**Cause**: GitHub Pages CDN caches aggressively.

**Fix**: Hard refresh (`Cmd+Shift+R`) or add `?t=timestamp` to URL.

### 2. Shell Aliases

**Symptom**: `gh` command does weird things.

**Cause**: User had `alias gh="cd ~/Documents/GitHub"`.

**Fix**: Use full path `/opt/homebrew/bin/gh` or check with `type gh`.

### 3. Syntax Errors from Refactoring

**Symptom**: App completely inactive, no visible errors.

**Cause**: Extra/missing brace after editing control flow.

**Prevention**: Validate before push:
```bash
node -e "
  const fs = require('fs');
  const html = fs.readFileSync('index.html', 'utf8');
  const js = html.match(/<script>([\s\S]*)<\/script>/)[1];
  new Function(js);
  console.log('Syntax OK');
"
```

### 4. LocalStorage Isolation

**Symptom**: Token not syncing between devices.

**Cause**: localStorage is per-origin. Each device/browser has its own.

**Not a bug**: This is expected. Token must be entered on each device.

## Testing

### Manual Testing Checklist

```
[ ] Add card with 'a'
[ ] Edit card with Enter
[ ] Move card with Space/Shift+Space
[ ] Delete card with Backspace
[ ] Set priority with p+c/h/m/l/s
[ ] Set effort with e+1-6
[ ] Filter with f+c/h/m/l/s
[ ] Navigate with h/j/k/l
[ ] Open settings with s
[ ] Export JSON with Ctrl+E
[ ] Import JSON with Ctrl+I
[ ] Drag and drop cards
[ ] Mobile touch interactions
```

### Playwright Debugging

When app is broken, use Playwright to diagnose:

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();

  page.on('pageerror', err => console.log('ERROR:', err.message));

  await page.goto('https://xiaolong-y.github.io/kanban/?t=' + Date.now());
  await page.waitForTimeout(2000);

  const cards = await page.$$('.card');
  console.log('Cards:', cards.length);

  await page.keyboard.press('a');
  const input = await page.$('.add input');
  console.log('Add works:', !!input);

  await browser.close();
})();
```

## Future Enhancements

### Low Effort, High Value

1. **PWA manifest** - Add to home screen on mobile
2. **Keyboard shortcut overlay** - Show on long-press of `?`
3. **Card timestamps display** - Show "added 3d ago"

### Medium Effort

1. **Card-level sync** - Merge by card ID instead of full overwrite
2. **Undo/redo** - Track state history
3. **Search** - Filter cards by text

### High Effort (Probably Don't)

1. **Multi-user** - Would need real backend
2. **Recurring tasks** - Scope creep into task manager
3. **Time tracking** - Different tool category

## Code Style

### Naming

- `S` = global state
- `SYNC` = sync layer state
- `COLS` = column names array
- `PRIS` = priority definitions
- Functions: `camelCase`
- Constants: `UPPER_SNAKE`

### DOM Patterns

```javascript
// Query
document.querySelector('.col[data-col="now"]')
document.querySelectorAll('.card')

// Create
const el = document.createElement('div');
el.className = 'card';
el.dataset.id = card.id;

// Events
document.addEventListener('keydown', handler);
el.addEventListener('click', handler);
```

### Async Patterns

```javascript
// Sync operations that might fail
async function syncSave() {
  try {
    await saveToGist();
    updateSyncStatus('synced');
  } catch (e) {
    console.error('Sync failed:', e);
    updateSyncStatus('error');
    // Don't throw - local save already succeeded
  }
}
```

## Deployment

### GitHub Pages

```bash
# Already configured - just push to master
git push origin master
# Site updates in ~1 minute at xiaolong-y.github.io/kanban
```

### Self-Hosted

```bash
# Any static file server works
python -m http.server 8000
npx serve .
caddy file-server --listen :8000
```

### Local File

Just open `index.html` in browser. Everything works except:
- File System API (needs HTTPS)
- Gist sync works fine

## Debugging Checklist

When something breaks:

1. **Check browser console** (F12 → Console)
2. **Check network tab** for failed API calls
3. **Check localStorage** (`localStorage.getItem('kanban_v3')`)
4. **Check sync state** (`SYNC` object in console)
5. **Hard refresh** to clear cache
6. **Try incognito** to isolate localStorage issues
