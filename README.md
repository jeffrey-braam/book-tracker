# Book Tracker

A personal reading log in a single HTML file. It tracks what you own, what you're reading and how fast, what to read next, and what you thought of it all. There's no install, no server and no account, and your data never leaves your browser.

## Getting started

1. Download `book-tracker.html`.
2. Open it in any modern browser.

That's it. Your library is saved automatically in the browser's `localStorage`. Use **Export** or **Backup All Profiles** to keep a copy as a JSON file, since clearing site data will erase it.

## Features

### Adding books
- **ISBN lookup** fills in the title, author, page count, cover and genre tags.
- **Search by title** shows up to 8 matching editions to pick from.
- **Manual entry** of any field, including a cover URL or an **uploaded cover image**. Uploads are shrunk to a small JPEG and stored with the book.
- **Bulk import** from a CSV or a plain list of ISBNs. It detects the header and ISBN column, skips books you already have, and looks each book up one at a time. You can stop partway and keep what's been added.

### Tracking a book
Each book moves through **To-be-read → Reading → Read** or **Did Not Finish**, and records:
- format (physical, ebook, audiobook), pages or audiobook length, series and number
- start and finish dates, progress, rating (0–5 stars), notes, quotes with page numbers, and the reason you dropped it
- shelves and genre/mood tags
- **ownership categories**, and a book can have several: owned (physical, ebook or audiobook), borrowed, loaned out, or wishlist
- flags for Top 5, reread candidate, and rereads (logged as a separate record so the original read stays intact)

### Tabs
| Tab | What it's for |
|---|---|
| **Library** | A cover grid of every book, with search, a category filter, and sorting by title, author or category. |
| **Read** | Finished books, sortable by date, rating or title, with a rereads-only filter. |
| **Logs** | A daily tracker for books in progress, with **📖 Books** (pages) and **🎧 Audiobooks** (hours and minutes) side by side. Log what you read each day. Progress updates itself, and a 30-day chart shows your reading. |
| **Pacing** | A yearly book goal, reading stats (books per month, pages, streaks, pages per day), and a table estimating when each current book will be finished. |
| **TBR & Discovery** | Your to-be-read list in two tiers, **Up Next** and **Someday**, reordered by drag or ↑/↓. It also has series "next up" suggestions, 🎲 Pick for me, mood-based picks, recommendations based on what you rated highly, comparing with another profile, and reading challenges (bingo cards or checklists). |
| **DNF** | Books you gave up on, and why. |
| **Series %** | How far through each series you are, and which book is next. |
| **Rankings** | Top 5 reads, reread candidates, top authors and series, favourite quotes, and charts: fiction vs nonfiction, page lengths, genres, and pages read per month. |

Clicking any cover or title opens a detail view of the book. From there you can also correct its reading log.

### More
- **Profiles:** separate libraries on one machine, each with its own name, emoji and accent colour. Books can be sent between profiles, and duplicates are refused.
- **Select mode:** bulk delete, add tags or shelves, change status, or find missing covers for many books at once.
- **Undo** for 8 seconds after deleting books.
- **Backup reminder** once 14 days have passed or 20 books have been added since your last backup.
- **Light and dark mode** follow your system setting. Every chart can also be shown as a table.

## How it works

**One file, no dependencies.** `book-tracker.html` has the styles, the markup and one self-contained script. It uses no framework, no build step and no charting library, and every chart is drawn as plain HTML/SVG.

**Data.** Everything lives in a single `localStorage` entry (`bookTrackerProfiles_v1`) holding all profiles and their books and challenges. Every loaded, imported or restored record is first checked and repaired: unknown values are dropped, numbers clamped, dates validated. Bad data can't break the page.

**Rendering.** Each change updates the data in memory, saves it, and redraws every view from scratch. This keeps the screen always in line with the data. Clicks are handled by a few listeners on the page, which read `data-*` attributes on the buttons, so redrawing never breaks a button.

**Book metadata and covers** come from free public APIs that need no key: Open Library first, then Google Books, then Apple's iTunes search for cover art only. Each source is tried in turn if the previous one fails. A cover is only saved after it has actually loaded, so broken or blank images are never stored. A slow network is never mistaken for "no cover".

**Privacy.** The only things sent anywhere are ISBNs, titles and authors, when you look a book up. Your library, ratings, notes and logs stay in your browser.

**Pages and minutes.** Pages and audiobook minutes are counted separately and never mixed. The Logs tab builds both sides from one shared description of the two units, so they always work the same way.

## Backups and moving data

| Action | File | Contents |
|---|---|---|
| **Export** | `book-tracker-<profile>.json` | The current profile's books and challenges |
| **Backup All Profiles** | `book-tracker-all-profiles-<date>.json` | Every profile, including goals and settings |
| **Import** | either file | A full backup replaces all profiles, and a single export replaces the current profile's library. Both ask first. |

The browser's `localStorage` holds about 5 MB, which is enough for thousands of books. Uploaded covers take the most space, roughly 10–60 KB each.

## Known limitations
- There's no sync between devices or browsers. Move data with Export and Import.
- Drag-and-drop reordering doesn't work on most touch screens, so use the ↑/↓ buttons. Chart tooltips need a mouse, but the table view works everywhere.
- Dialogs don't close with Escape.

## More detail
[`DESIGN.md`](DESIGN.md) is the full design document. It covers the data model, every feature's rules, the lookup and import pipelines, the visual design system, and the security notes.

`book-test-data - Test Data(2).csv` is a small sample file for trying out Bulk Import.
