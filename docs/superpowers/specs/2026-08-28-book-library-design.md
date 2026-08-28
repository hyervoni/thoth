# Book Library — Design

**Date:** 2026-08-28
**Status:** Approved for planning

## Problem

Thoth persists exactly one book. `index.html` opens IndexedDB `thoth` v1 with a
single object store `books` and writes every save to the fixed key `'current'`
(`index.html:1089-1149`). Importing a second EPUB overwrites the first,
destroying its reading position and bookmarks. `annotations.md:82` already
tracks this as the main gap against Reedy and Spritz.

A second problem falls out of the same design: the saved record holds the full
book text, and `persistProgress()` rewrites that whole record on a 600 ms
debounce while you read. Every progress tick serializes the entire book.

## Goals

- Keep any number of books, each with its own reading position and bookmarks.
- Switch between them from the existing LOAD panel.
- Delete a book explicitly.
- Migrate the existing v1 record without losing progress.
- Stop rewriting the full book text on every progress save.

## Non-goals

Cover art, search, tags/collections, per-book stats, sorting controls, and
export/import of the library. A book-count cap and automatic eviction are also
out: storage is bounded by manual deletion only.

## Scope change: plain-text input is removed

With a real library, the paste flow stops earning its place — every pasted
article would become a permanent row sorted above the books by recency,
burying them. Rather than special-case paste permanence, the input goes away.

Removed: the `<textarea>` (`index.html:657`), the `import-row` and
`Load and read` button (`index.html:660-663`), `loadText()`
(`index.html:851-853`), and the `loadBtn` / `sampleBtn` handlers
(`index.html:921-931`). The `.sample-btn` CSS class stays — `epubBtn` uses it.

EPUB import becomes the only way text enters the library.

### The sample text survives as an ephemeral demo

"Sample text" remains as the one-tap way to try Thoth without owning an EPUB —
otherwise a first-time visitor on a phone faces an app they cannot even look
at. It now loads the excerpt straight into the reader and plays. It writes
nothing: no library row, no IndexedDB access at all. Reloading leaves no trace.

Consequences, which the implementation must honor:

- Entering the demo flushes the pending progress save of the book you were
  reading, exactly like switching books.
- `settings.currentBookId` is left untouched, so a reload during the demo
  returns you to your real book.
- The bookmark button is inert while the demo is loaded — there is no row to
  save bookmarks into.
- `beforeunload` (`index.html:1171`) must not save while the demo is loaded;
  its `words.length` guard is no longer sufficient on its own.

## Data model (IndexedDB `thoth`, version 2)

Three object stores.

**`library`** — one lightweight record per book, `keyPath: 'id'`. This is the
only store read to render the list.

```js
{
  id,            // string — see "Book identity"
  title,         // string
  author,        // string — '' when the EPUB has no dc:creator
  wordCount,     // number — total words, for the progress percentage
  cursor,        // number — current word index
  bookmarks,     // array — same shape as today (index.html:689)
  addedAt,       // number — Date.now() at import
  lastOpenedAt,  // number — Date.now() at open; list sort key
}
```

There is deliberately no `source` field and no `preview` field. Every row is an
EPUB or a migrated legacy record, both drawn identically, so nothing would ever
read them. A field that is written and never read drifts out of sync silently.

**`books`** — the existing store, reused unchanged for the heavy text. Still
out-of-line keys, so the current `put(value, key)` calls keep working. Key is
the book `id`, value is `{ chapters: [{ title, text }] }`.

**`settings`** — one record under key `'app'`: `{ wpm, currentBookId }`.

### WPM becomes global

`wpm` moves out of the per-book record into `settings`. Today it rides along
with the single book; with several books, a per-book WPM would drop you back to
300 when you open a different title after settling at 500. Reading speed
belongs to the reader, not the book. This is an intentional behavior change.

## Book identity

`id` is derived once, at import, by the first rule that yields a usable value.
Ids are opaque strings with no prefix or namespace.

1. `dc:identifier` from the OPF, normalized: trimmed, lowercased, with a
   leading `urn:isbn:` or `urn:uuid:` prefix stripped. Rejected when shorter
   than 4 characters after normalization.
2. `title|author`, each part trimmed, lowercased, and whitespace-collapsed.
3. The filename with the `.epub` extension removed, lowercased.

`parseEpub()` currently returns `{ title, chapters }` and reads only
`dc:title` (`index.html:1221`). It gains `dc:creator` and `dc:identifier` from
the same `opfDoc`, and returns `{ id, title, author, chapters }`.

### Identifier collisions are validated against the title

`dc:identifier` is unreliable in the wild — Calibre and other generators reuse
one UUID across every book in an export. Trusting it blindly means importing
book B silently opens book A, with no error to explain it.

So a dedupe hit is confirmed before it is honored: if `library[id]` exists and
its stored title matches the incoming title (normalized as in rule 2), it is
the same book and it opens. If the titles differ, it is not: the import
proceeds as a new book under `id + '#' + n`, using the lowest free `n >= 2`.

Re-importing the same file still deduplicates, because there the titles agree.

## Migration (v1 → v2)

`onupgradeneeded` for version 2 only creates the `library` and `settings`
stores. The data migration runs at startup, outside the versionchange
transaction — the fragile part of migrating IndexedDB — inside a `try/catch`
that lets the app boot empty rather than fail if the old record is malformed.

`migrateV1()`:

1. Read `books['current']`. Absent → return; the migration is done.
2. Determine the title. The v1 record has one only if it came from an EPUB; a
   titleless record was a paste, and is titled from the first ~6 words of its
   first chapter. Its reading position is preserved either way — an app update
   must not silently discard someone's book, even one the app can no longer
   create.
3. Derive the id by applying rule 2 to `title|` (v1 stored no author and no
   identifier).
4. Write `books[id] = { chapters }` (text first).
5. Write `library[id]` with `cursor` from `saved.index`, `bookmarks` from
   `saved.bookmarks`, `wordCount` summed from the chapters, and
   `addedAt` / `lastOpenedAt` from `saved.savedAt`.
6. Write `settings.app = { wpm: saved.wpm || 350, currentBookId: id }`.
7. Delete `books['current']`.

The delete comes last, so a failure at any earlier step leaves `'current'`
in place and the next boot retries. Re-running after a successful migration is
a no-op.

**Known one-time edge case.** The migrated book gets a title-derived id because
v1 never stored `dc:identifier`. Re-importing that same EPUB later takes rule 1
and produces a different id, so it lands as a second row. This affects at most
the one book carried over from v1, and the duplicate can be deleted. Not worth
special-casing.

## Deployment and upgrade safety

**The service worker cache must be bumped**, `thoth-v13` → `thoth-v14`
(`sw.js:1`). The fetch handler is cache-first with no revalidation for
same-origin GETs (`sw.js:38-51`), so without the bump every returning visitor
keeps running the old `index.html` indefinitely and the library never ships.

**`openDB()` must handle `blocked`.** The deployed v1 code opens the database
and never closes the connection, and registers no `onversionchange`
(`index.html:1094-1104`). A tab left open on the old code therefore blocks the
v2 upgrade — and `blocked` fires neither `onsuccess` nor `onerror`, so the
promise never settles and the new tab hangs mute on an empty stage. `openDB()`
adds an `onblocked` handler that surfaces "Thoth is open in another tab — close
it and reload".

**The new code registers `db.onversionchange = () => db.close()`**, so the next
schema change never reproduces this. It cannot help the v1 → v2 step, whose
blocking side is already deployed.

**Multiple tabs on the new code** are last-write-wins on `settings` and on the
shared `library` row. Two tabs reading the same book will fight over `cursor`.
Accepted: this is a single-user local reader, and coordination would cost more
than the problem.

## Storage helpers

`dbGet(key)` and `dbSet(key, value)` (`index.html:1105-1122`) are hardcoded to
the `books` store. They become store-aware — `dbGet(store, key)`,
`dbSet(store, value, key)` — plus `dbGetAll(store)` and `dbDelete(store, key)`.
Records in `library` use its `keyPath`, so their writes pass no explicit key.

## UI

The `#panel` element (`index.html:652-666`) is retitled **Library**. It holds
the book list where the textarea used to be, with `Import EPUB` and
`Sample text` pinned below.

No layout CSS is needed: `.toc-list` already carries `flex: 1; min-height: 0;
overflow-y: auto` (`index.html:209-217`) — the same flex properties the
textarea has today (`index.html:513-515`) — so the list drops into the same
slot, scrolls on its own, and leaves the actions anchored at the bottom.

Rows reuse the `.toc-list` container and the bookmark row pattern
(`index.html:1048-1080`): a primary button on the left, a `✕` delete button on
the right. The only new CSS is the metadata line.

Each row shows the title, and below it `author · 42% · 3 May`. The open book is
marked. Sort is `lastOpenedAt` descending, so the book you are reading is
always first. An empty library renders no list at all.

The list is rendered when the panel opens and re-rendered after any mutation.

The toolbar button keeps its `LOAD` text but its `aria-label` changes from
`Load text` to `Library` (`index.html:648`) — with the textarea gone, the old
label is simply false. The empty-stage hint `paste text to begin`
(`index.html:616`) is reworded to point at importing.

## Flows

**Open a book.** Flush the current book's pending progress save, read
`books[id]`, then call `loadBook({ title, chapters }, { restoredIndex: rec.cursor,
fromStorage: true, restoredBookmarks: rec.bookmarks })`. Update `lastOpenedAt`
and `settings.currentBookId`, close the panel. **No autoplay** — the reader
stops on the word where you left off and you press Continue. Import and the
demo both autoplay; choosing from a list is a different intent than having just
brought something in.

**Import an EPUB.** Parse, derive the id, then check `library[id]` and confirm
any hit against the stored title. A confirmed hit opens the stored book with its
progress and bookmarks, writing only `lastOpenedAt`. Otherwise both stores are
written and playback starts, as today.

**Sample text.** Loads the excerpt into the reader and plays, touching no
store. See "The sample text survives as an ephemeral demo".

**Delete.** `confirm()` first, since it is irreversible and a mis-tap on a
crowded mobile list costs a whole book. Then delete from both stores. Deleting
the open book clears the stage: stop playback, reset `words`, `chapters`,
`bookmarks`, `cursor` and `bookTitle`, hide the chapter bar, restore the
`empty-hint` placeholder, and null `settings.currentBookId`.

**Startup.** Open the DB, run `migrateV1()`, read `settings` and apply `wpm` to
the slider, then open `currentBookId` if set. Otherwise the existing empty
state.

**Progress saves.** `persistProgress()` writes only the `library` row, from an
in-memory reference to the current record. The `books` store is written just
once per book, at import. WPM changes write `settings` on the same debounce.

`loadBook()` (`index.html:855-878`) no longer persists as a side effect; its
callers decide what to write.

## Error handling

**Write order at import: text first, then the library row.** If the text write
fails there is never a phantom row pointing at nothing, and the failure is
surfaced with `alert()` — consistent with how import already reports an
oversized or unreadable EPUB. Today `doSave()` swallows everything with a
`console.warn` (`index.html:1140`); for import that is not enough, because the
user would believe a book was saved that is not there.

**Orphan rows.** If opening a row finds no text in `books`, show a clear message
and offer to delete the row, instead of leaving an entry that never opens.

**Progress saves stay silent** on failure. An alert every 600 ms while reading
would be worse than the lost tick.

**Quota.** No book-count cap. `MAX_EPUB_MB = 100` (`index.html:1337`) is
unchanged. A `QuotaExceededError` on import surfaces as "no space left — delete
a book".

## Compatibility

iOS 12 / Safari 12 support is a standing constraint (`annotations.md:39-43`):
no optional chaining, no `clamp()` in new code. `IDBObjectStore.getAll()` has
been available since Safari 10.1, so rendering the list is covered.

## Documentation

`README.md` advertises a "Plain-text mode for quick paste-and-read" (line 19)
and describes the LOAD flow around pasting (lines 36-41). Both are updated as
part of this work.

`annotations.md` is explicitly marked human-only (`do not automate`) and is not
touched.

## Verification

The repo has no test suite and no build step, so verification is a manual
Playwright MCP pass against a local server, in both mobile and desktop
viewports:

1. With a v1 `'current'` record present, load the app — the book appears in the
   library with its title, progress and bookmarks intact, and `'current'` is
   gone afterwards.
2. Same, with a titleless v1 record — it migrates titled by its first words.
3. Import two EPUBs, alternate between them, confirm each returns to its own
   position.
4. Re-import an already-stored EPUB — no duplicate row; it opens the stored copy.
5. Import two EPUBs that share a `dc:identifier` but differ in title — both land
   as separate rows.
6. Delete the open book — the stage returns to the empty hint.
7. Reload — the last book reopens at its saved position.
8. Change WPM in one book, open another — the WPM carries over.
9. Open a book from the list — it does not autoplay; import and the demo do.
10. Tap Sample text, then reload — no library row was created and the previously
    open book comes back.
11. With the app open in a second tab running the old code, load the new one —
    the "open in another tab" message appears instead of a silent hang.
