# Book Library Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Thoth's single-book slot with a real multi-book library, so importing a second EPUB no longer destroys the first one's reading position and bookmarks.

**Architecture:** IndexedDB moves from one store holding one record at the fixed key `'current'` to three stores: `library` (one lightweight row per book — metadata, cursor, bookmarks), `books` (the heavy chapter text, keyed by book id), and `settings` (global WPM and the currently open book). The book list lives inside the panel the LOAD button already opens. Plain-text input is removed; EPUB import becomes the only way text enters the library, with the sample excerpt surviving as an ephemeral demo that never touches storage.

**Tech Stack:** Vanilla JavaScript in a single `index.html`, no build step, no framework, no new dependencies. IndexedDB for persistence, JSZip (self-hosted) for EPUB unpacking, a cache-first service worker for offline use.

**Spec:** `docs/superpowers/specs/2026-08-28-book-library-design.md`

## Global Constraints

- **Single file.** All application code lives in `index.html`. This is deliberate — the app deploys as static files with no build step. Do not split it into modules, do not add a bundler, do not add dependencies.
- **iOS 12 / Safari 12 support is required** (`annotations.md:19-46`). In new code: **no** optional chaining (`?.`), **no** nullish coalescing (`??`), **no** CSS `clamp()`. Use `obj && obj.prop` and `min(c, max(a, b))`. Everything else — arrow functions, template literals, `async`/`await`, destructuring, spread, `let`/`const` — works on Safari 12.
- **Database:** name `thoth`, version `2`. Stores: `books` (out-of-line keys), `library` (`keyPath: 'id'`), `settings` (out-of-line keys). The settings record lives under key `'app'`.
- **Service worker cache:** `thoth-v14`, bumped from `thoth-v13` in Task 1.
- **`MAX_EPUB_MB = 100`** stays unchanged.
- **Never modify `annotations.md`.** Its frontmatter says `do not automate` and that is binding.
- **Commit messages:** imperative mood, no `Co-Authored-By` trailer, no "Generated with" lines.
- **Work on a branch**, not `main`. Task 1 Step 1 creates it.

### Local verification setup

Every task is verified with Playwright MCP against a local server. Start it once and leave it running:

```bash
cd /Users/hernan/Work/thoth && python3 -m http.server 8000
```

The app is then at `http://localhost:8000`.

**The service worker will serve you a stale `index.html` and waste your time.** Its fetch handler is cache-first with no revalidation (`sw.js:38-51`). At the start of every verification session, navigate to the page and run this with `browser_evaluate`, then reload:

```js
() => navigator.serviceWorker.getRegistrations()
  .then(rs => Promise.all(rs.map(r => r.unregister())))
  .then(() => 'unregistered')
```

Only Task 8 verifies service worker behavior itself. Every other task runs with it unregistered.

To start a task from a clean database, run this and reload:

```js
() => new Promise(resolve => {
  const req = indexedDB.deleteDatabase('thoth');
  req.onsuccess = () => resolve('deleted');
  req.onerror = () => resolve('error');
  req.onblocked = () => resolve('blocked — close other tabs');
})
```

### How this plan tests without a test framework

The repo has no test runner and no build step, and adding one would break the "open the file and edit it" property the project is built on. Two substitutes are used, both real:

1. **Unit checks via `browser_evaluate`.** The `<script>` in `index.html` is a classic top-level script with no IIFE wrapper — `index.html:685` opens straight into `let words = []`. Every `function foo()` declared there is therefore reachable by name from page-context evaluation. Playwright can call `deriveBookId(...)` with crafted inputs and assert on the return value. These are genuine unit tests against the shipping code.
2. **End-to-end checks** driving the real UI with clicks.

A step that says "Expected: FAIL" means you must actually run it and see that failure before implementing. A check that never failed proves nothing.

### Test fixtures

Task 2 builds three EPUB fixtures in `/tmp/thoth-fixtures/`. Later tasks reuse them. Do not commit them.

| Fixture | Title | Author | `dc:identifier` | Purpose |
|---|---|---|---|---|
| `alpha.epub` | Alpha Book | Ada Lovelace | `urn:uuid:1111...1111` | Normal import |
| `beta.epub` | Beta Book | Bob Barker | `urn:uuid:2222...2222` | Second book, switching |
| `clash.epub` | Clash Book | Cleo Clash | `urn:uuid:1111...1111` | Same identifier as alpha, different title |

---

## File Structure

| File | Change | Responsibility |
|---|---|---|
| `index.html` | Modified by every task | The whole app: markup, styles and logic. |
| `sw.js` | Modify line 1 | Cache version bump, so the update actually reaches returning visitors. |
| `README.md` | Modify lines 19, 36-41 | Stop advertising the removed paste flow. |
| `/tmp/thoth-fixtures/*.epub` | Created, never committed | Test fixtures. |

The single-file structure is a project constraint, not an oversight. Follow it.

---

### Task 1: Storage layer — v2 schema, store-aware helpers, blocked-upgrade handling

Behavior does not change in this task. The app still reads and writes the single `'current'` record; only the plumbing underneath changes. That is deliberate — the schema and helper rewrite is worth verifying on its own before any semantics move on top of it.

**Files:**
- Modify: `index.html:1089-1122` (the persistence block), plus the two call sites at `index.html:1129` and `index.html:1153`
- Modify: `sw.js:1`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `DB_NAME = 'thoth'`, `DB_VERSION = 2`
  - `STORE_BOOKS = 'books'`, `STORE_LIBRARY = 'library'`, `STORE_SETTINGS = 'settings'`, `SETTINGS_KEY = 'app'`, `LEGACY_KEY = 'current'`
  - `openDB(): Promise<IDBDatabase>`
  - `dbGet(store: string, key: string): Promise<any>`
  - `dbGetAll(store: string): Promise<Array<any>>`
  - `dbSet(store: string, value: any, key?: string): Promise<void>` — `key` is omitted for `library`, which has a `keyPath`
  - `dbDelete(store: string, key: string): Promise<void>`

- [ ] **Step 1: Create the branch**

```bash
cd /Users/hernan/Work/thoth && git checkout -b book-library
```

- [ ] **Step 2: Write the failing check**

Navigate to `http://localhost:8000`, unregister the service worker, reload, then run with `browser_evaluate`:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth');
  req.onsuccess = () => {
    const db = req.result;
    const stores = Array.prototype.slice.call(db.objectStoreNames);
    const version = db.version;
    db.close();
    resolve({ version: version, stores: stores });
  };
})
```

- [ ] **Step 3: Run it to verify it fails**

Expected: `{ version: 1, stores: ["books"] }`. The v2 stores do not exist yet.

- [ ] **Step 4: Replace the DB constants and `openDB`**

In `index.html`, replace lines 1090-1104 — from `const DB_NAME = 'thoth';` through the closing brace of `openDB` — with:

```js
  const DB_NAME = 'thoth';
  const DB_VERSION = 2;
  const STORE_BOOKS = 'books';
  const STORE_LIBRARY = 'library';
  const STORE_SETTINGS = 'settings';
  const SETTINGS_KEY = 'app';
  const LEGACY_KEY = 'current';
  let dbPromise;
  let dbBlockedWarned = false;

  function openDB() {
    if (!dbPromise) {
      dbPromise = new Promise((resolve, reject) => {
        const req = indexedDB.open(DB_NAME, DB_VERSION);
        req.onupgradeneeded = () => {
          const db = req.result;
          if (!db.objectStoreNames.contains(STORE_BOOKS)) {
            db.createObjectStore(STORE_BOOKS);
          }
          if (!db.objectStoreNames.contains(STORE_LIBRARY)) {
            db.createObjectStore(STORE_LIBRARY, { keyPath: 'id' });
          }
          if (!db.objectStoreNames.contains(STORE_SETTINGS)) {
            db.createObjectStore(STORE_SETTINGS);
          }
        };
        // Fires when another tab holds an older connection open. It fires
        // neither onsuccess nor onerror, so without this the promise never
        // settles and the app hangs on an empty stage with no explanation.
        req.onblocked = () => {
          if (dbBlockedWarned) return;
          dbBlockedWarned = true;
          alert('Thoth is open in another tab. Close it and reload this page.');
        };
        req.onsuccess = () => {
          const db = req.result;
          // Let a future version upgrade proceed instead of blocking it the
          // way the v1 code blocks this one.
          db.onversionchange = () => db.close();
          resolve(db);
        };
        req.onerror = () => reject(req.error);
      });
    }
    return dbPromise;
  }
```

- [ ] **Step 5: Replace the helpers with store-aware versions**

Replace lines 1105-1122 — the whole of `dbGet` and `dbSet` — with:

```js
  async function dbGet(store, key) {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(store, 'readonly');
      const req = tx.objectStore(store).get(key);
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });
  }
  async function dbGetAll(store) {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(store, 'readonly');
      const req = tx.objectStore(store).getAll();
      req.onsuccess = () => resolve(req.result || []);
      req.onerror = () => reject(req.error);
    });
  }
  async function dbSet(store, value, key) {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(store, 'readwrite');
      if (key === undefined) tx.objectStore(store).put(value);
      else tx.objectStore(store).put(value, key);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }
  async function dbDelete(store, key) {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(store, 'readwrite');
      tx.objectStore(store).delete(key);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }
```

- [ ] **Step 6: Update the two existing call sites**

In `doSave` (was `index.html:1129`), the write becomes:

```js
      await dbSet(STORE_BOOKS, {
        title: bookTitle,
        chapters: chapters.map(c => ({
          title: c.title,
          text: words.slice(c.start, c.end).join(' '),
        })),
        index: cursor,
        wpm,
        bookmarks,
        savedAt: Date.now(),
      }, LEGACY_KEY);
```

In `restoreSaved` (was `index.html:1153`):

```js
      const saved = await dbGet(STORE_BOOKS, LEGACY_KEY);
```

- [ ] **Step 7: Bump the service worker cache**

In `sw.js:1`: `const CACHE = 'thoth-v13';` → `const CACHE = 'thoth-v14';`

This is bumped now rather than at the end so that every later deploy picks the work up. Without it, the cache-first fetch handler serves the old `index.html` forever.

- [ ] **Step 8: Re-run the check from Step 2**

Reload the page first so the new code runs.

Expected: `{ version: 2, stores: ["books", "library", "settings"] }`

- [ ] **Step 9: Verify nothing regressed**

Import `/tmp/thoth-fixtures/alpha.epub` if it exists, or any EPUB, through the existing UI. Let it play a few seconds, then reload the page. Expected: the book returns at the position you left it. The app must behave exactly as it did before this task.

- [ ] **Step 10: Commit**

```bash
git add index.html sw.js
git commit -m "Add v2 stores and store-aware IndexedDB helpers"
```

---

### Task 2: EPUB metadata and book identity

Pure functions plus a widened `parseEpub` return value. Nothing consumes the id yet, so behavior is again unchanged.

**Files:**
- Modify: `index.html:1189-1334` (`parseEpub` and the helpers above it)
- Create: `/tmp/thoth-fixtures/*.epub` (never committed)

**Interfaces:**
- Consumes: nothing from Task 1.
- Produces:
  - `normalizePart(s: string): string` — trimmed, lowercased, whitespace-collapsed
  - `normalizeIdentifier(s: string): string` — `normalizePart` plus a stripped leading `urn:isbn:` / `urn:uuid:`
  - `titleFromText(text: string): string` — the first 6 words, capped at 60 characters
  - `deriveBookId(meta: {identifier: string, title: string, author: string, fileName: string}): string`
  - `parseEpub(file: File): Promise<{id: string, title: string, author: string, chapters: Array<{title: string, text: string}>}>`

- [ ] **Step 1: Build the test fixtures**

Write this to `/tmp/thoth-fixtures/build.py` and run it with `python3`:

```python
import zipfile, os

def build(name, title, author, ident):
    body = " ".join(["word%d" % i for i in range(300)])
    container = ('<?xml version="1.0"?><container version="1.0" '
        'xmlns="urn:oasis:names:tc:opendocument:xmlns:container"><rootfiles>'
        '<rootfile full-path="OEBPS/content.opf" '
        'media-type="application/oebps-package+xml"/></rootfiles></container>')
    opf = ('<?xml version="1.0"?><package xmlns="http://www.idpf.org/2007/opf" '
        'version="2.0" unique-identifier="bookid">'
        '<metadata xmlns:dc="http://purl.org/dc/elements/1.1/">'
        '<dc:title>%s</dc:title><dc:creator>%s</dc:creator>'
        '<dc:identifier id="bookid">%s</dc:identifier></metadata>'
        '<manifest><item id="c1" href="c1.xhtml" media-type="application/xhtml+xml"/>'
        '<item id="c2" href="c2.xhtml" media-type="application/xhtml+xml"/></manifest>'
        '<spine><itemref idref="c1"/><itemref idref="c2"/></spine></package>'
        ) % (title, author, ident)
    def chap(h):
        return ('<?xml version="1.0"?><html xmlns="http://www.w3.org/1999/xhtml">'
                '<head><title>%s</title></head><body><h1>%s</h1><p>%s</p></body>'
                '</html>') % (h, h, body)
    with zipfile.ZipFile(name, 'w') as z:
        z.writestr('mimetype', 'application/epub+zip')
        z.writestr('META-INF/container.xml', container)
        z.writestr('OEBPS/content.opf', opf)
        z.writestr('OEBPS/c1.xhtml', chap('Chapter One'))
        z.writestr('OEBPS/c2.xhtml', chap('Chapter Two'))

os.chdir('/tmp/thoth-fixtures')
build('alpha.epub', 'Alpha Book', 'Ada Lovelace',
      'urn:uuid:11111111-1111-1111-1111-111111111111')
build('beta.epub', 'Beta Book', 'Bob Barker',
      'urn:uuid:22222222-2222-2222-2222-222222222222')
build('clash.epub', 'Clash Book', 'Cleo Clash',
      'urn:uuid:11111111-1111-1111-1111-111111111111')
print(sorted(os.listdir('.')))
```

Expected output includes `alpha.epub`, `beta.epub`, `clash.epub`.

- [ ] **Step 2: Write the failing check**

Reload `http://localhost:8000` and run with `browser_evaluate`:

```js
() => {
  const cases = [
    [{identifier: 'urn:uuid:ABC-123', title: 'T', author: 'A', fileName: 'x.epub'},
     'abc-123'],
    [{identifier: 'urn:isbn:9780000000001', title: 'T', author: 'A', fileName: 'x.epub'},
     '9780000000001'],
    [{identifier: '  ', title: 'Alpha Book', author: 'Ada Lovelace', fileName: 'x.epub'},
     'alpha book|ada lovelace'],
    [{identifier: 'ab', title: 'Alpha  Book', author: '', fileName: 'x.epub'},
     'alpha book|'],
    [{identifier: '', title: '', author: '', fileName: 'My Book.EPUB'},
     'my book'],
  ];
  return cases.map(c => {
    const got = deriveBookId(c[0]);
    return { got: got, want: c[1], pass: got === c[1] };
  });
}
```

- [ ] **Step 3: Run it to verify it fails**

Expected: an error along the lines of `deriveBookId is not defined`.

- [ ] **Step 4: Add the identity helpers**

Insert these immediately above `async function parseEpub(file) {` (was `index.html:1189`):

```js
  // --- Book identity ---
  function normalizePart(s) {
    if (!s) return '';
    return String(s).trim().toLowerCase().replace(/\s+/g, ' ');
  }
  function normalizeIdentifier(s) {
    return normalizePart(s).replace(/^urn:(isbn|uuid):/, '');
  }
  function titleFromText(text) {
    const words = String(text || '').trim().split(/\s+/).filter(Boolean).slice(0, 6);
    const joined = words.join(' ');
    return joined.length > 60 ? joined.slice(0, 60).trim() : joined;
  }
  // First rule that yields a usable value wins. Ids are opaque strings:
  // no prefix, no namespace.
  function deriveBookId(meta) {
    const ident = normalizeIdentifier(meta.identifier || '');
    if (ident.length >= 4) return ident;
    const t = normalizePart(meta.title || '');
    if (t) return t + '|' + normalizePart(meta.author || '');
    const fromFile = normalizePart(String(meta.fileName || '').replace(/\.epub$/i, ''));
    if (fromFile) return fromFile;
    return 'book-' + Date.now().toString(36);
  }
```

- [ ] **Step 5: Run the check to verify it passes**

Reload, re-run the Step 2 evaluation.

Expected: all five entries report `pass: true`.

- [ ] **Step 6: Read author and identifier in `parseEpub`**

After the `titleEl` / `bookTitle` block (was `index.html:1221-1224`), add:

```js
    const creatorEl = opfDoc.querySelector('metadata > [*|creator], metadata > creator');
    const author = creatorEl && creatorEl.textContent ? creatorEl.textContent.trim() : '';
    const identEl = opfDoc.querySelector('metadata > [*|identifier], metadata > identifier');
    const identifier = identEl && identEl.textContent ? identEl.textContent.trim() : '';
```

- [ ] **Step 7: Widen the return value**

Replace `return { title: bookTitle, chapters };` (was `index.html:1333`) with:

```js
    return {
      id: deriveBookId({
        identifier: identifier,
        title: bookTitle,
        author: author,
        fileName: file.name,
      }),
      title: bookTitle,
      author: author,
      chapters: chapters,
    };
```

- [ ] **Step 8: Verify against a real fixture**

Reload, click **LOAD**, then **Import EPUB**, and use `browser_file_upload` with `/tmp/thoth-fixtures/alpha.epub`. Once the book loads, run:

```js
() => ({ title: bookTitle, words: words.length, chapters: chapters.length })
```

Expected: `title` is `"Alpha Book"`, `chapters` is 2, `words` is greater than 500.

The id itself is checked in Task 7, once import consumes it.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Extract author and identifier from EPUB metadata"
```

---

### Task 3: Library rows, settings, and progress writes

This is where the single-slot model dies. After this task the app stores every imported book as its own row and reopens the last one on boot — there is still no UI to switch between them, which Task 6 adds.

**Files:**
- Modify: `index.html:855-878` (`loadBook`), `index.html:1016-1046` (bookmark functions), `index.html:1124-1171` (the save/restore block), `index.html:1340-1364` (the import handler)

**Interfaces:**
- Consumes: `dbGet`, `dbGetAll`, `dbSet`, `dbDelete`, `STORE_*`, `SETTINGS_KEY` (Task 1); `parseEpub` returning `{id, title, author, chapters}` (Task 2)
- Produces:
  - `currentBookId: string | null` and `currentRow: object | null` — module-level state. `currentRow === null` while a book is loaded means the demo is playing (Task 5); every persistence path checks it.
  - `countWords(chapters: Array<{text: string}>): number`
  - `makeRow(book: {id, title, author, chapters}): object` — a fresh `library` record
  - `saveNewBook(book): Promise<object>` — writes text then row, returns the row
  - `loadSettings(): Promise<{wpm: number, currentBookId: string|null}>`
  - `saveSettings(): Promise<void>`
  - `writeRowNow(): Promise<void>` — replaces `doSave`
  - `flushProgress(): void` — replaces `persistBook`
  - `persistProgress(): void` — same name, now debounces `writeRowNow`
  - `openBookById(id: string, opts?: {autoplay?: boolean}): Promise<boolean>`
  - `initApp(): Promise<void>` — replaces `restoreSaved`

- [ ] **Step 1: Write the failing check**

Clear the database, reload, then run:

```js
() => typeof saveNewBook === 'function' && typeof openBookById === 'function'
```

- [ ] **Step 2: Run it to verify it fails**

Expected: `false`.

- [ ] **Step 3: Drop the persistence side effect from `loadBook`**

In `loadBook` (was `index.html:855-878`), delete the line `if (!fromStorage) persistBook();` and remove `fromStorage` from the options destructuring, since nothing else reads it. The first line of the function becomes:

```js
    const { restoredIndex = 0, restoredBookmarks = null } = opts;
```

Callers decide what to write from now on.

- [ ] **Step 4: Replace the save/restore block**

Replace everything from `let saveTimer = null;` through `restoreSaved();` and the `beforeunload` listener (was `index.html:1124-1171`) with:

```js
  // --- Library records ---
  let currentBookId = null;
  let currentRow = null;   // null while the ephemeral sample demo is loaded

  function countWords(chs) {
    let n = 0;
    for (const ch of chs) {
      n += String(ch.text || '').trim().split(/\s+/).filter(Boolean).length;
    }
    return n;
  }

  function makeRow(book) {
    const now = Date.now();
    return {
      id: book.id,
      title: book.title || '',
      author: book.author || '',
      wordCount: countWords(book.chapters),
      cursor: 0,
      bookmarks: [],
      addedAt: now,
      lastOpenedAt: now,
    };
  }

  // Text first, row second: if the text write fails there is never a library
  // row pointing at nothing.
  async function saveNewBook(book) {
    await dbSet(STORE_BOOKS, { chapters: book.chapters }, book.id);
    const row = makeRow(book);
    await dbSet(STORE_LIBRARY, row);
    return row;
  }

  async function loadSettings() {
    const s = await dbGet(STORE_SETTINGS, SETTINGS_KEY);
    if (!s) return { wpm: 350, currentBookId: null };
    return { wpm: s.wpm || 350, currentBookId: s.currentBookId || null };
  }

  async function saveSettings() {
    await dbSet(STORE_SETTINGS, { wpm: wpm, currentBookId: currentBookId }, SETTINGS_KEY);
  }

  let saveTimer = null;
  async function writeRowNow() {
    saveTimer = null;
    if (!currentRow) return;
    currentRow.cursor = cursor;
    currentRow.bookmarks = bookmarks;
    try {
      await dbSet(STORE_LIBRARY, currentRow);
      await saveSettings();
    } catch (e) { console.warn('save failed', e); }
  }
  function flushProgress() {
    if (saveTimer) { clearTimeout(saveTimer); saveTimer = null; }
    writeRowNow();
  }
  function persistProgress() {
    if (saveTimer) clearTimeout(saveTimer);
    saveTimer = setTimeout(writeRowNow, 600);
  }

  async function openBookById(id, opts) {
    const autoplay = !!(opts && opts.autoplay);
    flushProgress();
    const row = await dbGet(STORE_LIBRARY, id);
    if (!row) return false;
    const rec = await dbGet(STORE_BOOKS, id);
    if (!rec || !rec.chapters || !rec.chapters.length) return false;
    currentBookId = id;
    currentRow = row;
    row.lastOpenedAt = Date.now();
    loadBook({ title: row.title, chapters: rec.chapters }, {
      restoredIndex: row.cursor || 0,
      restoredBookmarks: Array.isArray(row.bookmarks) ? row.bookmarks : [],
    });
    await dbSet(STORE_LIBRARY, row);
    await saveSettings();
    if (autoplay) play();
    return true;
  }

  async function initApp() {
    try {
      const s = await loadSettings();
      wpm = s.wpm;
      speedSlider.value = wpm;
      speedNum.textContent = wpm;
      if (s.currentBookId) await openBookById(s.currentBookId, { autoplay: false });
    } catch (e) { /* boot empty rather than fail */ }
  }
  initApp();
  window.addEventListener('beforeunload', () => { if (currentRow) writeRowNow(); });
```

- [ ] **Step 5: Point the bookmark functions at the new writer**

In `addBookmark` (was `index.html:1016`), add a guard on the second line and swap the write at the end:

```js
  function addBookmark() {
    if (!words.length) return;
    if (!currentRow) return;   // the sample demo owns no row
    if (bookmarks.some(b => b.index === cursor)) return; // skip exact duplicate
```

and replace the trailing `persistBook();` with `flushProgress();`.

In `removeBookmark` (was `index.html:1032`), replace `persistBook();` with `flushProgress();`.

- [ ] **Step 6: Make import write a library row**

In the `epubInput` change handler (was `index.html:1352-1356`), replace the body of the `try` block with:

```js
      const book = await parseEpub(file);
      const row = await saveNewBook(book);
      currentBookId = row.id;
      currentRow = row;
      loadBook({ title: book.title, chapters: book.chapters });
      await saveSettings();
      panel.classList.remove('open');
      play();
```

Deduplication and quota errors arrive in Task 7.

- [ ] **Step 7: Verify the rows are written**

Reload the page, import `/tmp/thoth-fixtures/alpha.epub`, let it play about five seconds, then run:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth');
  req.onsuccess = () => {
    const db = req.result;
    const tx = db.transaction(['library', 'settings'], 'readonly');
    const rows = tx.objectStore('library').getAll();
    const set = tx.objectStore('settings').get('app');
    tx.oncomplete = () => {
      db.close();
      resolve({ rows: rows.result, settings: set.result });
    };
  };
})
```

Expected: one row with `title: "Alpha Book"`, `author: "Ada Lovelace"`, a `wordCount` above 500 and a `cursor` above 0; and settings holding that row's `id` as `currentBookId`.

- [ ] **Step 8: Verify a second import no longer destroys the first**

Import `/tmp/thoth-fixtures/beta.epub`, then re-run the Step 7 evaluation.

Expected: **two** rows, alpha's `cursor` preserved from where you stopped. This is the whole point of the feature — do not proceed if there is only one row.

- [ ] **Step 9: Verify the boot path**

Reload the page. Expected: Beta Book loads at its saved position, paused, and the play button reads `Continue`.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "Store each book as its own library row"
```

---

### Task 4: Migrate the v1 record

**Files:**
- Modify: `index.html` — add `migrateV1` beside the library functions and call it from `initApp`

**Interfaces:**
- Consumes: `dbGet`, `dbSet`, `dbDelete`, `STORE_*`, `LEGACY_KEY`, `SETTINGS_KEY` (Task 1); `deriveBookId`, `titleFromText` (Task 2); `countWords`, `initApp` (Task 3)
- Produces: `migrateV1(): Promise<void>` — idempotent; a no-op once `books['current']` is gone

- [ ] **Step 1: Seed a v1 database**

Clear the database, then run this to write a realistic v1 record — note it opens at **version 1**, exactly as the old code did:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth', 1);
  req.onupgradeneeded = () => req.result.createObjectStore('books');
  req.onsuccess = () => {
    const db = req.result;
    const tx = db.transaction('books', 'readwrite');
    tx.objectStore('books').put({
      title: 'Legacy Book',
      chapters: [{ title: 'One', text: 'alpha beta gamma delta epsilon zeta eta theta' }],
      index: 3,
      wpm: 525,
      bookmarks: [{ index: 2, chapterIdx: 0, chapterTitle: 'One', percent: 25, savedAt: 1 }],
      savedAt: 1735689600000,
    }, 'current');
    tx.oncomplete = () => { db.close(); resolve('seeded'); };
  };
})
```

- [ ] **Step 2: Write the failing check**

Reload the page, then run:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth');
  req.onsuccess = () => {
    const db = req.result;
    const tx = db.transaction(['books', 'library'], 'readonly');
    const legacy = tx.objectStore('books').get('current');
    const rows = tx.objectStore('library').getAll();
    tx.oncomplete = () => {
      db.close();
      resolve({ legacyStillThere: !!legacy.result, rows: rows.result });
    };
  };
})
```

- [ ] **Step 3: Run it to verify it fails**

Expected: `{ legacyStillThere: true, rows: [] }`. The old record is stranded and the app shows an empty stage.

- [ ] **Step 4: Add `migrateV1`**

Insert immediately above `async function initApp() {`:

```js
  // Runs at startup rather than inside onupgradeneeded: the versionchange
  // transaction is the fragile place to do this, and a failure here must not
  // take the app down with it.
  async function migrateV1() {
    const saved = await dbGet(STORE_BOOKS, LEGACY_KEY);
    if (!saved || !saved.chapters || !saved.chapters.length) return;
    // A v1 record without a title came from the removed paste flow. Title it
    // from its own first words rather than discarding someone's reading.
    let title = saved.title || '';
    if (!title) title = titleFromText(saved.chapters[0].text || '');
    const id = deriveBookId({ identifier: '', title: title, author: '', fileName: '' });
    await dbSet(STORE_BOOKS, { chapters: saved.chapters }, id);
    const when = saved.savedAt || Date.now();
    await dbSet(STORE_LIBRARY, {
      id: id,
      title: title,
      author: '',
      wordCount: countWords(saved.chapters),
      cursor: saved.index || 0,
      bookmarks: Array.isArray(saved.bookmarks) ? saved.bookmarks : [],
      addedAt: when,
      lastOpenedAt: when,
    });
    await dbSet(STORE_SETTINGS, {
      wpm: saved.wpm || 350,
      currentBookId: id,
    }, SETTINGS_KEY);
    // Last, so any earlier failure leaves 'current' in place for a retry.
    await dbDelete(STORE_BOOKS, LEGACY_KEY);
  }
```

- [ ] **Step 5: Call it from `initApp`**

In `initApp`, add the call as the first statement inside the `try`:

```js
    try {
      await migrateV1();
      const s = await loadSettings();
```

- [ ] **Step 6: Run the check to verify it passes**

Reload the page, re-run the Step 2 evaluation.

Expected: `legacyStillThere: false`, and one row with `title: "Legacy Book"`, `cursor: 3`, and one bookmark. The stage should show the book paused, with the speed slider reading **525**.

- [ ] **Step 7: Verify idempotency**

Reload the page twice more and re-run the check.

Expected: still exactly one row. Nothing duplicates.

- [ ] **Step 8: Verify a titleless legacy record**

Clear the database and re-run the Step 1 seed with `title: ''`. Reload, then re-run the Step 2 check.

Expected: one row titled `"alpha beta gamma delta epsilon zeta"` — the first six words.

- [ ] **Step 9: Verify a corrupt record cannot brick the boot**

Clear the database, seed with `chapters: 'not-an-array'` instead of the array, reload.

Expected: the app boots to the empty stage, no uncaught exception in the console.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "Migrate the v1 single-book record into the library"
```

---

### Task 5: Remove plain-text input, keep the sample as an ephemeral demo

**Files:**
- Modify: `index.html:616` (empty hint), `index.html:648` (aria-label), `index.html:652-666` (panel markup), `index.html:707-709` (element refs), `index.html:851-853` (`loadText`), `index.html:911-931` (handlers)
- Modify: `README.md:19` and `README.md:36-41`

**Interfaces:**
- Consumes: `flushProgress`, `currentRow`, `loadBook`, `play` (Task 3)
- Produces:
  - `SAMPLE_TEXT: string`
  - `loadSampleDemo(): void` — loads the excerpt and plays it, touching no store
  - `libraryList: HTMLElement` — the `#libraryList` container Task 6 renders into

- [ ] **Step 1: Write the failing check**

Reload and run:

```js
() => ({
  textarea: !!document.getElementById('textInput'),
  loadBtn: !!document.getElementById('loadBtn'),
  libraryList: !!document.getElementById('libraryList'),
  panelTitle: document.querySelector('#panel .panel-title').textContent,
  ariaLabel: document.getElementById('textBtn').getAttribute('aria-label'),
})
```

- [ ] **Step 2: Run it to verify it fails**

Expected: `{ textarea: true, loadBtn: true, libraryList: false, panelTitle: "Your text", ariaLabel: "Load text" }`.

- [ ] **Step 3: Replace the panel markup**

Replace `index.html:652-666` in full with:

```html
<div class="panel" id="panel">
  <div class="panel-header">
    <div class="panel-title">Library</div>
    <button class="close-btn" id="closePanelBtn">Close ✕</button>
  </div>
  <div class="toc-list" id="libraryList"></div>
  <div class="panel-actions">
    <div class="import-row">
      <button class="sample-btn" id="epubBtn">Import EPUB</button>
      <button class="sample-btn" id="sampleBtn">Sample text</button>
    </div>
  </div>
  <input type="file" accept=".epub,application/epub+zip" id="epubInput" hidden>
</div>
```

No layout CSS is needed: `.toc-list` already carries `flex: 1; min-height: 0; overflow-y: auto` (`index.html:209-217`), the same flex behavior the textarea had, so the list fills the same slot and the actions stay anchored at the bottom.

- [ ] **Step 4: Fix the hint and the aria-label**

`index.html:616`:

```html
      <span class="empty-hint">import an epub<br>to begin</span>
```

`index.html:648`:

```html
    <button class="icon-btn" id="textBtn" aria-label="Library">LOAD</button>
```

The visible `LOAD` text stays. With the textarea gone, `Load text` was simply false for screen readers.

- [ ] **Step 5: Update the element references**

Replace the three refs at `index.html:707-709` with:

```js
  const libraryList = document.getElementById('libraryList');
  const sampleBtn = document.getElementById('sampleBtn');
```

`textInput` and `loadBtn` no longer exist.

- [ ] **Step 6: Replace `loadText` with the demo loader**

Replace `loadText` (was `index.html:851-853`) with:

```js
  const SAMPLE_TEXT = "It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife. However little known the feelings or views of such a man may be on his first entering a neighbourhood, this truth is so well fixed in the minds of the surrounding families, that he is considered as the rightful property of some one or other of their daughters. \"My dear Mr. Bennet,\" said his lady to him one day, \"have you heard that Netherfield Park is let at last?\" Mr. Bennet replied that he had not. \"But it is,\" returned she; \"for Mrs. Long has just been here, and she told me all about it.\" Mr. Bennet made no answer. \"Do not you want to know who has taken it?\" cried his wife impatiently. \"You want to tell me, and I have no objection to hearing it.\"";

  // Ephemeral: no library row, no IndexedDB write at all. currentBookId is
  // left alone on purpose, so a reload during the demo returns to the real
  // book. currentRow === null is what makes every persistence path a no-op.
  function loadSampleDemo() {
    flushProgress();
    currentRow = null;
    loadBook({ title: 'Sample text', chapters: [{ title: '', text: SAMPLE_TEXT }] });
    panel.classList.remove('open');
    play();
  }
```

- [ ] **Step 7: Replace the handlers**

Replace the `textBtn` handler (was `index.html:911-917`) with:

```js
  textBtn.addEventListener('click', () => {
    stop();
    panel.classList.add('open');
  });
```

Delete the `loadBtn` handler entirely (was `index.html:921-927`) and replace the `sampleBtn` handler (was `index.html:929-931`) with:

```js
  sampleBtn.addEventListener('click', loadSampleDemo);
```

- [ ] **Step 8: Run the check to verify it passes**

Reload, re-run the Step 1 evaluation.

Expected: `{ textarea: false, loadBtn: false, libraryList: true, panelTitle: "Library", ariaLabel: "Library" }`.

- [ ] **Step 9: Verify the demo writes nothing**

With a book already imported and playing, note its position. Then open **LOAD** → **Sample text**. Confirm the excerpt plays. Reload the page and run the Step 7 evaluation from Task 3.

Expected: the row count is unchanged, no row is titled `Sample text`, and the app comes back to your real book at its saved position.

- [ ] **Step 10: Verify bookmarks are inert during the demo**

Load the demo, let it play, press **🔖** and then **Save current position**.

Expected: no bookmark is added, and no error appears in the console.

- [ ] **Step 11: Update the README**

`README.md:19` — delete the `**Plain-text mode** for quick paste-and-read` bullet.

`README.md:36-41` — replace the Usage steps with:

```markdown
1. Click **LOAD** to open your library.
2. Click **Import EPUB** to add a book, or **Sample text** to try the reader without one.
3. Pick a book from the list → press **Continue**.
4. Adjust **Speed** with the slider; use **‹‹** to rewind, **🔖** to manage bookmarks, and the chapter bar to jump between chapters.
```

Also update `README.md:41`'s closing line if it still claims a single book is restored: it should say your library and reading positions are restored automatically.

- [ ] **Step 12: Commit**

```bash
git add index.html README.md
git commit -m "Remove plain-text input, keep sample as an ephemeral demo"
```

---

### Task 6: The library list

**Files:**
- Modify: `index.html` — add three CSS rules near the bookmark styles (after `index.html:334`), add render/delete/clear functions near the bookmark functions, wire the `textBtn` handler

**Interfaces:**
- Consumes: `dbGetAll`, `dbDelete`, `dbSet`, `STORE_*` (Task 1); `currentBookId`, `currentRow`, `openBookById`, `saveSettings`, `flushProgress` (Task 3); `libraryList` (Task 5)
- Produces:
  - `renderLibrary(): Promise<void>`
  - `deleteBook(id: string): Promise<void>`
  - `clearStage(): void`
  - `formatRowMeta(row: object): string` — the `author · 42% · 3 May` line

- [ ] **Step 1: Write the failing check**

With two books imported, reload, click **LOAD**, then run:

```js
() => document.querySelectorAll('#libraryList .bookmark-row').length
```

- [ ] **Step 2: Run it to verify it fails**

Expected: `0` — the container is empty.

- [ ] **Step 3: Add the three CSS rules**

Insert after the `.bookmark-delete:hover` rule (was `index.html:334`):

```css
  .book-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 4px;
  }
  .book-item .sub {
    font-family: 'JetBrains Mono', monospace;
    font-size: 10px;
    color: var(--ink-soft);
    letter-spacing: 0.05em;
  }
  .book-item.current .label { color: var(--accent); font-style: italic; }
```

Everything else reuses `.bookmark-row`, `.bookmark-item` and `.bookmark-delete` verbatim.

- [ ] **Step 4: Add the render and delete functions**

Insert immediately above the `bookmarkBtn` click listener (was `index.html:1082`):

```js
  // --- Library ---
  function formatRowMeta(row) {
    const parts = [];
    if (row.author) parts.push(row.author);
    const total = row.wordCount || 0;
    const pct = total ? Math.round((row.cursor || 0) / total * 100) : 0;
    parts.push(pct + '%');
    const when = new Date(row.lastOpenedAt || row.addedAt || Date.now());
    parts.push(when.toLocaleDateString(undefined, { day: 'numeric', month: 'short' }));
    return parts.join(' · ');
  }

  function clearStage() {
    stop();
    words = [];
    chapters = [];
    bookmarks = [];
    bookTitle = '';
    cursor = 0;
    currentBookId = null;
    currentRow = null;
    chapterBar.classList.remove('visible');
    wordEl.innerHTML = '<span class="empty-hint">import an epub<br>to begin</span>';
    playBtn.textContent = 'Start';
    updateUI();
  }

  async function deleteBook(id) {
    await dbDelete(STORE_LIBRARY, id);
    await dbDelete(STORE_BOOKS, id);
    if (id === currentBookId) {
      clearStage();
      await saveSettings();
    }
    renderLibrary();
  }

  async function renderLibrary() {
    let rows = [];
    try { rows = await dbGetAll(STORE_LIBRARY); } catch (e) { rows = []; }
    rows.sort((a, b) => (b.lastOpenedAt || 0) - (a.lastOpenedAt || 0));
    libraryList.innerHTML = '';
    rows.forEach(r => {
      const row = document.createElement('div');
      row.className = 'bookmark-row';
      row.innerHTML =
        '<button class="bookmark-item book-item">' +
          '<span class="label"></span>' +
          '<span class="sub"></span>' +
        '</button>' +
        '<button class="bookmark-delete" aria-label="Delete">✕</button>';
      const item = row.querySelector('.bookmark-item');
      if (r.id === currentBookId) item.classList.add('current');
      row.querySelector('.label').textContent = r.title || 'Untitled';
      row.querySelector('.sub').textContent = formatRowMeta(r);
      item.addEventListener('click', async () => {
        panel.classList.remove('open');
        const ok = await openBookById(r.id);
        if (!ok) {
          alert('This book\'s text is missing. Removing it from the library.');
          await deleteBook(r.id);
          panel.classList.add('open');
        }
      });
      row.querySelector('.bookmark-delete').addEventListener('click', async (e) => {
        e.stopPropagation();
        const name = r.title || 'this book';
        if (!confirm('Delete ' + name + '? This cannot be undone.')) return;
        await deleteBook(r.id);
      });
      libraryList.appendChild(row);
    });
  }
```

Opening from the list passes no `autoplay`, so the reader stops on the word you left off at — import and the demo are the flows that start playing on their own.

- [ ] **Step 5: Render the list when the panel opens**

Change the `textBtn` handler from Task 5 to:

```js
  textBtn.addEventListener('click', () => {
    stop();
    renderLibrary();
    panel.classList.add('open');
  });
```

- [ ] **Step 6: Re-render after an import**

At the end of the `try` block in the `epubInput` handler, after `play();`, add:

```js
      renderLibrary();
```

- [ ] **Step 7: Run the check to verify it passes**

Reload, click **LOAD**, re-run the Step 1 evaluation.

Expected: `2` with two books imported. Each row shows a title and an `author · N% · date` line, and the open book's title is in the accent color.

- [ ] **Step 8: Verify switching preserves both positions**

Read Alpha for a few seconds, open **LOAD**, pick Beta, read a few seconds, open **LOAD**, pick Alpha again.

Expected: Alpha resumes where you left it, paused, and does not autoplay. Then run the Task 3 Step 7 evaluation and confirm both rows carry distinct non-zero cursors.

- [ ] **Step 9: Verify deletion**

Delete the book that is **not** open — confirm the dialog appears and the row disappears while the stage keeps playing. Then delete the open one.

Expected: the stage clears to `import an epub to begin`, and after a reload the app stays empty rather than reopening a deleted book.

- [ ] **Step 10: Verify the orphan path**

Import Alpha, then delete only its text and reload:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth');
  req.onsuccess = () => {
    const db = req.result;
    const tx = db.transaction('library', 'readonly');
    const rows = tx.objectStore('library').getAll();
    tx.oncomplete = () => {
      const id = rows.result[0].id;
      const tx2 = db.transaction('books', 'readwrite');
      tx2.objectStore('books').delete(id);
      tx2.oncomplete = () => { db.close(); resolve('text removed for ' + id); };
    };
  };
})
```

Open **LOAD** and click the row.

Expected: the "text is missing" alert appears, the row is removed, and the panel stays open. Nothing hangs.

- [ ] **Step 11: Verify the narrow viewport**

Resize to 375×667 and open the panel with three or more books.

Expected: rows stack without overflowing horizontally, long titles wrap, and the list scrolls with `Import EPUB` still visible at the bottom.

- [ ] **Step 12: Commit**

```bash
git add index.html
git commit -m "Add the library list with open and delete"
```

---

### Task 7: Import deduplication and error handling

**Files:**
- Modify: `index.html` — the `epubInput` change handler and a new `resolveImportId` beside it

**Interfaces:**
- Consumes: `dbGet`, `STORE_LIBRARY` (Task 1); `normalizePart` (Task 2); `saveNewBook`, `openBookById`, `saveSettings`, `currentBookId`, `currentRow` (Task 3); `renderLibrary` (Task 6)
- Produces: `resolveImportId(book: {id, title}): Promise<{id: string, existing: object|null}>`

- [ ] **Step 1: Write the failing check**

Clear the database, reload, import `alpha.epub`, then import it a second time, then import `clash.epub` (same identifier, different title). Run:

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth');
  req.onsuccess = () => {
    const db = req.result;
    const tx = db.transaction('library', 'readonly');
    const rows = tx.objectStore('library').getAll();
    tx.oncomplete = () => {
      db.close();
      resolve(rows.result.map(r => ({ id: r.id, title: r.title })));
    };
  };
})
```

- [ ] **Step 2: Run it to verify it fails**

Expected: **one** row — `clash.epub` overwrote `alpha.epub`, because they share an identifier and nothing checks the title. Two distinct books collapsed into one. That is the bug this task fixes.

- [ ] **Step 3: Add `resolveImportId`**

Insert immediately above the `epubInput` change listener:

```js
  // dc:identifier is unreliable — Calibre and friends reuse one UUID across
  // every book in an export. A dedupe hit is only honored when the stored
  // title agrees; otherwise the import gets a disambiguated id.
  async function resolveImportId(book) {
    const wanted = normalizePart(book.title);
    let n = 1;
    for (;;) {
      const candidate = n === 1 ? book.id : book.id + '#' + n;
      const row = await dbGet(STORE_LIBRARY, candidate);
      if (!row) return { id: candidate, existing: null };
      if (normalizePart(row.title) === wanted) return { id: candidate, existing: row };
      n++;
    }
  }
```

- [ ] **Step 4: Rewrite the import handler body**

Replace the contents of the `try` block in the `epubInput` change listener with:

```js
      const book = await parseEpub(file);
      const resolved = await resolveImportId(book);
      if (resolved.existing) {
        panel.classList.remove('open');
        await openBookById(resolved.id, { autoplay: true });
        renderLibrary();
        return;
      }
      book.id = resolved.id;
      let row;
      try {
        row = await saveNewBook(book);
      } catch (saveErr) {
        const quota = saveErr && saveErr.name === 'QuotaExceededError';
        alert(quota
          ? 'No space left on this device. Delete a book and try again.'
          : 'Could not save the book: ' + (saveErr && saveErr.message ? saveErr.message : saveErr));
        return;
      }
      currentBookId = row.id;
      currentRow = row;
      loadBook({ title: book.title, chapters: book.chapters });
      await saveSettings();
      panel.classList.remove('open');
      play();
      renderLibrary();
```

The `finally` block that restores the button label already runs on these early returns — leave it as it is.

- [ ] **Step 5: Run the check to verify it passes**

Clear the database, reload, then repeat Step 1's sequence: alpha, alpha again, clash.

Expected: **two** rows. One `Alpha Book` at the bare identifier id, one `Clash Book` at that same id suffixed with `#2`. Re-importing alpha produced no third row.

- [ ] **Step 6: Verify the re-import kept its position**

Read Alpha for several seconds, note the progress counter, then import `alpha.epub` again.

Expected: the stored copy opens and plays from where you were — not from zero — and no duplicate row appears.

- [ ] **Step 7: Verify the oversized-file guard still works**

```bash
head -c 105000000 /dev/urandom > /tmp/thoth-fixtures/huge.epub
```

Import it.

Expected: the `File too large (100.1 MB)` alert, and no row is created.

- [ ] **Step 8: Verify a malformed EPUB is reported**

```bash
echo "not an epub" > /tmp/thoth-fixtures/broken.epub
```

Import it.

Expected: a `Could not read EPUB:` alert, no row created, and the Import button returns from `Processing…` to its normal label.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Validate dedupe hits against the title and report save failures"
```

---

### Task 8: Full-system verification

No new behavior. This task runs the spec's acceptance checklist end to end, including the two paths every earlier task deliberately switched off: the service worker and the blocked upgrade.

**Files:**
- Modify: `index.html` or `sw.js` only if a check fails

**Interfaces:**
- Consumes: everything.
- Produces: a verified branch ready for review.

- [ ] **Step 1: Confirm the service worker cache was bumped**

```bash
grep -n "const CACHE" /Users/hernan/Work/thoth/sw.js
```

Expected: `const CACHE = 'thoth-v14';`. If it still reads `v13`, fix it now — without it the work never reaches anyone who has opened Thoth before.

- [ ] **Step 2: Verify the service worker serves the new shell**

Clear site data, reload twice so the worker installs and claims, then run:

```js
() => caches.keys()
```

Expected: `["thoth-v14"]` and nothing else — the activate handler deletes older caches.

- [ ] **Step 3: Verify offline still works**

With the worker active, use `browser_evaluate` to confirm the app shell loads from cache after a reload with the network served entirely by the worker:

```js
() => caches.open('thoth-v14').then(c => c.match('index.html')).then(r => !!r)
```

Expected: `true`.

- [ ] **Step 4: Verify the blocked-upgrade message**

This needs the deployed v1 code holding a v1 connection. Simulate it: clear the database, then in tab A run

```js
() => new Promise(resolve => {
  const req = indexedDB.open('thoth', 1);
  req.onupgradeneeded = () => req.result.createObjectStore('books');
  req.onsuccess = () => { window.__held = req.result; resolve('holding v1 open'); };
})
```

Then open `http://localhost:8000` in a second tab.

Expected: the second tab shows the `Thoth is open in another tab` alert instead of hanging silently on an empty stage. Close tab A afterwards.

- [ ] **Step 5: Run the spec's acceptance checklist**

Work through all eleven checks from the spec's Verification section, in a mobile viewport (375×667) and then a desktop one (1440×900):

1. A v1 record with a title migrates with its title, progress and bookmarks, and `'current'` is gone afterwards.
2. A titleless v1 record migrates titled by its first words.
3. Two EPUBs alternate, each returning to its own position.
4. Re-importing a stored EPUB opens the stored copy with no duplicate row.
5. Two EPUBs sharing a `dc:identifier` but differing in title land as separate rows.
6. Deleting the open book returns the stage to the empty hint.
7. Reloading reopens the last book at its saved position.
8. WPM set in one book carries over to another.
9. Opening from the list does not autoplay; import and the demo do.
10. Tapping Sample text creates no row, and a reload returns to the real book.
11. The blocked-upgrade message appears instead of a silent hang.

- [ ] **Step 6: Check the console is clean**

Exercise import, switch, delete and the demo in one session, then read the log with `browser_console_messages`.

Expected: no uncaught errors. A `save failed` warning is only acceptable if you deliberately provoked a quota failure.

- [ ] **Step 7: Confirm no iOS 12 regressions**

```bash
grep -nE '\?\.|\?\?|clamp\(' /Users/hernan/Work/thoth/index.html
```

Expected: no matches. If any appear in code this branch added, rewrite them as `obj && obj.prop` and `min(c, max(a, b))`.

- [ ] **Step 8: Confirm the untouched file stayed untouched**

```bash
git diff main --stat
```

Expected: `index.html`, `sw.js`, `README.md`, and the two `docs/superpowers/` documents. **`annotations.md` must not appear.**

- [ ] **Step 9: Commit any fixes**

```bash
git add -A
git commit -m "Fix issues found in full-system verification"
```

Skip this step if nothing needed fixing.

---

## Done

When all eight tasks are checked off, the branch is ready for review. Use `superpowers:requesting-code-review` before merging, and `superpowers:finishing-a-development-branch` to integrate.
