<p align="center">
  <img src="thoth.png" alt="Thoth" width="180">
</p>

<h1 align="center">Thoth</h1>

<p align="center">
  A minimalist, browser-based speed reader with RSVP playback, EPUB import, chapter navigation, and offline persistence.
</p>

<p align="center">
  <a href="https://hyervoni.github.io/thoth/"><strong>Open Thoth →</strong></a>
</p>

<p align="center">
  No install, no account. Press <em>Sample text</em> to try it without an EPUB.
</p>

## What it is

Thoth is a single-file speed-reader built around **Rapid Serial Visual Presentation (RSVP)**. Words flash one at a time on a focused stage at a user-controlled speed (150–800 WPM), removing the saccadic overhead of moving your eyes across a page.

## Features

- **EPUB import** with automatic chapter and table-of-contents extraction
- **Chapter navigation** with a live progress indicator
- **Bookmarks** to save and restore reading positions
- **Adjustable WPM** (150–800) with live speed control
- **Screen stays awake** while playing, so the phone does not dim mid-chapter
- **Daily goal**: reading pauses once a day with a congratulations card when you pass 10 pages (2,500 words, counted only from words actually played)
- **Fully offline**: works without network after first load, via a service worker that caches the app shell. Library and progress stay on your device (IndexedDB) — no account, no cloud, no third-party requests
- **Editorial typography** (Fraunces, Newsreader, JetBrains Mono) on a dark, paper-tone palette
- **Zero backend**: a single static HTML file, deployable anywhere

## Stack

- Vanilla JavaScript — no framework, no build step
- IndexedDB for client-side persistence
- Service worker for offline caching of the app shell
- [JSZip](https://stuk.github.io/jszip/) (self-hosted) for EPUB unpacking
- Self-hosted variable fonts (Latin): Fraunces, Newsreader, JetBrains Mono

## Usage

1. Open the **library** (the stacked-books icon, bottom right) — or go straight to **Sample text** to try the reader without an EPUB.
2. **Import EPUB** to add a book; pick it from the list and press **Continue**.
3. Set the pace with the **Speed** slider (150–800 WPM).
4. The **rewind** icon steps back, the **bookmark** icon saves and restores positions, and the chapter line above the controls opens the table of contents.
5. Tap the middle of the word to enter **focus mode** — everything but the word disappears. Tap again, or press Escape, to leave.

Your library and reading positions are restored automatically on next visit.

## Browser support

Modern evergreen browsers (Chrome, Firefox, Safari, Edge). Requires `IndexedDB` and ES2020+ JS.

## License

MIT
