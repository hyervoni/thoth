<p align="center">
  <img src="thoth.png" alt="Thoth" width="180">
</p>

<h1 align="center">Thoth</h1>

<p align="center">
  A minimalist, browser-based speed reader with RSVP playback, EPUB import, chapter navigation, and offline persistence.
</p>

## What it is

Thoth is a single-file speed-reader built around **Rapid Serial Visual Presentation (RSVP)**. Words flash one at a time on a focused stage at a user-controlled speed (150–800 WPM), removing the saccadic overhead of moving your eyes across a page.

## Features

- **EPUB import** with automatic chapter and table-of-contents extraction
- **Plain-text mode** for quick paste-and-read
- **Chapter navigation** with a live progress indicator
- **Bookmarks** to save and restore reading positions
- **Adjustable WPM** (150–800) with live speed control
- **Offline-first**: library and progress persist locally via IndexedDB
- **Editorial typography** (Fraunces, Newsreader, JetBrains Mono) on a dark, paper-tone palette
- **Zero backend**: a single static HTML file, deployable anywhere

## Stack

- Vanilla JavaScript — no framework, no build step
- IndexedDB for client-side persistence
- [JSZip](https://stuk.github.io/jszip/) (loaded on demand from jsDelivr) for EPUB unpacking
- Google Fonts for typography

## Usage

1. Click **LOAD** to open the input panel.
2. Paste text or click **Import EPUB** to load a book.
3. Click **Load and read** → press **Start**.
4. Adjust **Speed** with the slider; use **‹‹** to rewind, **🔖** to manage bookmarks, and the chapter bar to jump between chapters.

Your last book and reading position are restored automatically on next visit.

## Browser support

Modern evergreen browsers (Chrome, Firefox, Safari, Edge). Requires `IndexedDB` and ES2020+ JS.

## License

MIT
