# Book Tracker — Design Document

Describes the application as it exists in `book-tracker.html` (commit `d13ecb7` plus the fixes listed in §10). It documents current behaviour; it is not a proposal.

---

## 1. Overview

Book Tracker is a personal reading log that runs entirely in the browser. It tracks books through a reading lifecycle (to-be-read → reading → read / did-not-finish), groups them into series, ranks favourites, draws reading statistics, suggests what to read next, and supports several independent "profiles" (libraries) on one machine.

### Goals evident in the code

- **Zero install.** One HTML file: open it and it runs. No server, build step, framework, or charting library.
- **Local-first and private.** All data lives in `localStorage`. The only network traffic is anonymous metadata/cover lookups to public book APIs.
- **Readable by a non-expert.** The source is heavily commented in a teaching tone. It explains CSS variables, event delegation, JSONP, SVG paths, and so on.
- **Distrust remote data.** Cover URLs are verified by actually loading them. Genre tags are filtered through an allow-list. A slow network is never treated as "no cover".

### Non-goals / out of scope today

- Syncing across devices or browsers (only manual JSON export/import).
- Accounts, authentication, or a backend.
- Offline caching of cover images (covers are remote URLs).

---

## 2. Architecture

### 2.1 File layout

| Section | Lines (approx.) | Contents |
|---|---|---|
| `<style>` | 13–225 | Design tokens, light/dark themes, all component CSS |
| Markup | 227–702 | Header, selection bar, tab bar, 9 view panels, 4 modal dialogs, toast, chart tooltip |
| `<script>` | 704–3305 | Whole app in one strict-mode IIFE |

### 2.2 Runtime model

```
localStorage["bookTrackerProfiles_v1"]
          │  loadStore() / saveData()
          ▼
   store = { activeId, profiles: [ {id, name, emoji, color, created, data} ... ] }
                                                                   │
                                          data ◄── direct reference to active profile's data
                                                                   │
                         mutations (handlers) ──► saveData() ──► renderAll()
                                                                   │
                                     every render* fn rebuilds its container's innerHTML
```

- **Single source of truth:** the in-memory `store`. `data` points into the active profile, so almost all feature code reads and writes `data.books` / `data.challenges` and never needs to know profiles exist.
- **Render strategy:** "rebuild everything". After any change, `renderAll()` calls 13 render functions. Each one filters `data.books`, builds an HTML string, and assigns it to `innerHTML`. This is simple, and the screen always matches the data. The trade-off is that all interpolated text must go through `esc()`.
- **Event handling:** delegated. Listeners sit on `document.body` or on stable containers and dispatch on `data-*` attributes (`data-action`, `data-id`, `data-tier-move`, `data-send-id`, `data-grab-from`, `data-chart-toggle`, `data-tip`, `data-select-id`, `data-challenge`, `data-prof-*`). Re-rendering therefore never orphans a listener.
- **View switching:** each tab is a `.view` div, and the active one has `.active` (CSS `display:block`). Switching tabs moves the class and calls `renderAll()`.
- **Module-level UI state** (not persisted): `selectionMode`, `selectedIds` (Set), `editingId`, `titleHits`, `bulkEntries`, `bulkRawText`, `bulkStopRequested`, `dragSrcId`, `chartViewMode`, `librarySort`, `sendBookId`, `coverProbeCache`.

### 2.3 Utilities

| Name | Purpose |
|---|---|
| `$(id)` | `document.getElementById` shorthand |
| `splitList(str)` | `"a, b,,c"` → `["a","b","c"]` |
| `uid()` | base-36 timestamp + random suffix |
| `esc(s)` | HTML-escapes `& < > " '` |
| `toast(msg, isError)` | Bottom-centre notification: 2.2 s normally, resets its timer on repeat calls. Error toasts last 6 s and can't be replaced by a normal toast while showing. |
| `todayStr()` / `parseYmd(s)` | Local-calendar `YYYY-MM-DD` ↔ `Date`. Used for every stored date, so nothing depends on UTC. |
| `makeBook(overrides)` / `makeProfile(name, overrides)` / `blankLibrary()` | Canonical record constructors, so every creation path produces the same fields |

---

## 3. Data Model

### 3.1 Storage keys

| Key | Status |
|---|---|
| `bookTrackerProfiles_v1` | Current. Holds the whole `store`. |
| `bookTrackerData_v1` | Legacy single-library format. Read once and migrated into a profile called "My Library". Never written. |

`saveData()` runs once on load, so a fresh or migrated store is persisted right away and migration never repeats.

### 3.2 Store

```js
{
  activeId: "p_…",
  profiles: [Profile, ...]          // always at least one
}
```

### 3.3 Profile

| Field | Type | Notes |
|---|---|---|
| `id` | string | `"p_" + base36 time + random` |
| `name` | string | max 40 chars in UI; blank → "Unnamed" |
| `emoji` | string | max 4 chars; blank → 📚 |
| `color` | hex string | one of 8 `PROFILE_COLORS`; becomes the app `--accent` |
| `created` | `YYYY-MM-DD` | |
| `data` | Library | `{ books: Book[], challenges: Challenge[] }` |

`normalizeStore()` fills in missing profile fields, only accepts `#rrggbb` colours, and repairs an invalid `activeId`. It runs on load and on backup restore. Every library passes through `normalizeLibrary()` → `normalizeBook()` / `normalizeChallenge()` on load, legacy migration, backup restore, and single-library import. These drop non-object entries, restrict enums to known values, coerce numbers (clamping progress to 0–100 and rating to 0–5), validate dates, turn lists into string arrays, default a missing title to "Untitled", and pad or trim bingo cards to 25 squares.

### 3.4 Book

| Field | Type | Default | Set by / used for |
|---|---|---|---|
| `id` | string | `uid()` | |
| `isbn` | string | `""` | lookup key; dedupe key |
| `title` | string | `""` | required in form |
| `author` | string | `""` | comma-joined if several |
| `cover` | URL string | `""` | only verified URLs are stored by lookups |
| `series` | `{name, num}` \| null | `null` | `num` may be fractional (step 0.5) or null |
| `format` | `"physical"\|"ebook"\|"audiobook"` | `"physical"` | badge only |
| `pages` | int \| null | `null` | stats, charts |
| `status` | `"tbr"\|"reading"\|"read"\|"dnf"` | `"tbr"` | drives which tab and fields apply |
| `priority` | `"next"\|"someday"` | `"someday"` (form defaults new books to `"next"`) | TBR tier |
| `progress` | 0–100 | `0` | reading progress bar |
| `dateStarted` / `dateFinished` | `YYYY-MM-DD` \| null | `null` | pace stats, monthly chart |
| `rating` | 0–5 | `0` | 0 = unrated |
| `notes` | string | `""` | first 120 chars shown on card |
| `dnfReason` | string | `""` | |
| `isTopRead` | bool | `false` | Top 5 list |
| `isRereadCandidate` | bool | `false` | Reread Candidates list |
| `isReread` | bool | `false` | excluded from charts and Books Read |
| `quotes` | `{text, page}[]` | `[]` | Quotes panel |
| `shelves` | string[] | `[]` | TBR shelf filter, badges |
| `tags` | string[] | `[]` | genre/mood; charts, mood picks, recs |
| `dateAdded` | `YYYY-MM-DD` | today | kept across edits |
| `sortIndex` | int | *(absent)* | added by TBR drag-and-drop |
| `topReadRank` | int | *(absent)* | added by Top 5 drag-and-drop |

A **reread** is modelled as a *separate* book record: a copy with a new `id`, `isReread: true`, `status: "reading"`, reset progress, dates, and notes, and copied quotes.

### 3.5 Challenge

```js
{ id, name, type: "bingo" | "checklist", items: [{ label, done }] }
```

Bingo challenges always have exactly 25 items. Squares the user hasn't named are labelled "Square N".

### 3.6 Export formats

| File | Shape | Produced by |
|---|---|---|
| `book-tracker-<profile-slug>.json` | Library `{books, challenges}` | **Export** |
| `book-tracker-all-profiles-<date>.json` | `{kind:"bookTrackerBackup", version:1, exported, activeId, profiles}` | **Backup All Profiles** |

**Import** checks the file's contents: a `profiles` array means a full restore, which replaces every profile after a confirm. A `books` array means a single library, which replaces the active profile's library (with a confirm if it isn't empty).

---

## 4. User Interface

### 4.1 Global chrome

- **Header:** app title, a profile chip (emoji + `<select>` switcher), and **Manage Profiles**. Action buttons: **+ Add Book**, **Select Books**, **Export**, **Import**, **Backup All Profiles**, **Bulk Import (CSV/ISBNs)**. The two file inputs are hidden and triggered by those buttons.
- **Selection bar** (sticky, shown only in select mode): count, Select all visible, Clear, Check for Covers, Clear Covers, Delete Selected, Done.
- **Tab bar:** pill buttons with `data-view`.
- **Toast** and **chart tooltip:** single shared floating elements.

### 4.2 Book card (`bookCardHTML`)

Used in the TBR, Reading, Read, DNF, Series, Mood Picks, and Recommendations lists.

- 56×80 cover or a text placeholder (first 20 chars of the title).
- Title, author ("Unknown author" if blank).
- Badges: series + number, format, *Next Up* (gold), *★ Top Read* (gold), *Reread*, each shelf, each tag.
- Status extras: progress bar (reading), stars and notes excerpt (read), "Dropped: reason" (dnf).
- Actions by status:

| Status | Buttons |
|---|---|
| all | Edit |
| tbr | Start Reading, Move to Up Next / Remove from Up Next |
| reading | Mark Finished, DNF |
| read | Log Reread |
| all (only if >1 profile) | Send to… |

- In select mode, a checkbox appears top-left and a selected card gets an accent outline.

**Card action behaviour**

| Action | Effect |
|---|---|
| Start Reading | status → reading; `dateStarted` = today if unset |
| Mark Finished | status → read, progress 100, `dateFinished` = today, **opens Edit** so the user can rate |
| DNF | status → dnf, `dateFinished` = today, **opens Edit** for a reason |
| Log Reread | pushes a new reread copy (see §3.4) |
| Tier move | sets `priority` |

### 4.3 Tabs

| Tab | Contents |
|---|---|
| **TBR** | *Next Up Suggestions* panel (shown only if any). Search (title/author), shelf filter, genre/mood filter (options rebuilt on every render), 🎲 Pick for me (result shown under the toolbar). Two draggable tiers: **Up Next** and **Someday**, ordered by `sortIndex`. |
| **Currently Reading** | Cards with status `reading`. |
| **Read** | Search, sort (newest/oldest finished, highest rated, title A–Z), "Rereads only" checkbox. |
| **DNF Pile** | Cards with status `dnf`. |
| **Series** | One card per series name (alphabetical): read count / total, % bar, Next Up badge, member books sorted by number. |
| **Rankings & Favorites** | Two-column grid: 🏆 Top 5 Reads (draggable), 🔁 Reread Candidates, ✍️ Top Authors (top 10), 📖 Top Series (top 10), then 💬 Favorite Quotes. |
| **Stats & Pace** | Stat tiles, then four charts, each with a "View as table" toggle. |
| **Discovery** | 🎲 Pick For Me, 🌙 Mood-Based Picks, 👥 Compare With Another Profile, ✨ Recommended For You, 🎯 Reading Challenges. |
| **Library** | Cover-grid browse view of every book (2:3 covers, title, author, no actions). Sort by title or by author, with authorless books last. Shows a book count. |

### 4.4 Dialogs

All dialogs use `.modal-overlay` / `.modal`. Clicking the backdrop closes a dialog, except the bulk importer while it is running.

**Add / Edit Book**
1. *ISBN lookup:* ISBN field + Lookup. Fills title, author, pages, and cover, and merges tag suggestions.
2. *Search by title:* up to 8 candidate buttons (cover, title, author · year · pages). Picking one fills the form, resolves a working cover, and merges tags.
3. Core fields: Title*, Author, Cover URL, Series name / #, Format, Pages, Status.
4. Fields that depend on status:

| Field | tbr | reading | read | dnf |
|---|:-:|:-:|:-:|:-:|
| TBR Priority | ✓ | | | |
| Progress slider | | ✓ | | |
| Dates started/finished | | | ✓ | ✓ |
| Rating | | | ✓ | |
| Notes | | | ✓ | ✓ |
| DNF reason | | | | ✓ |
| Flags (Top 5, Reread candidate, Is reread) | | | ✓ | |
| Quotes (`text \|\| page` per line) | | | ✓ | |

5. Always shown: Shelves (comma-separated), Genre/mood tags (comma-separated).
6. Actions: Delete (edit only, confirm), Cancel, Save. Save rebuilds the record with `makeBook` and keeps `dateAdded`.

**Bulk Import:** two stages.
- *Preview:* "Found N new book(s)". An optional **"Title and ISBN columns are separate lists"** checkbox appears only when some rows contain both, and its hint gives the import count for each reading. Then "Add as" priority, Cancel, and Start Import.
- *Progress:* bar, "Looking up i of N" label, scrolling ✓/⚠ log, and Stop → Done.

**Profiles:** one row per profile with an editable emoji input, a name input (saves on every keystroke, no re-render so the caret isn't lost), 8 colour swatches, Active badge or Switch to, Duplicate (deep JSON copy), Delete (blocked for the last profile, with confirm), and "N books · added date". Below: Add a profile (Enter submits). New profiles get colours in turn from the palette.

**Send to another profile:** one button per other profile.

### 4.5 Native prompts

`confirm()` guards bulk delete, clear covers, single delete, restore/replace on import, profile delete, and challenge delete. **New Challenge** uses `prompt()` for the name, `confirm()` to choose bingo vs checklist, and `prompt()` for comma-separated goals or bingo squares (up to 25; blank leaves "Square N"). A challenge's **Edit** button re-prompts for the name and the comma-separated labels. Bingo ticks stay with their square position, and checklist ticks stay with goals whose wording is unchanged.

---

## 5. Feature Logic

### 5.1 Next Up suggestions (series)
For each series, find the highest-numbered **read** book. The suggestion is the lowest-numbered **tbr** book above it. Books with no series number are ignored. Suggestions appear in the TBR panel, as a badge on the matching TBR cards, and on the Series tab.

### 5.2 Rankings
- **Top 5 Reads:** read books with `isTopRead`. Books with a `topReadRank` come first in rank order, then the rest by rating descending, cut to 5. Dragging sets `topReadRank` on the visible rows.
- **Reread Candidates:** any book with `isRereadCandidate`, whatever its status.
- **Top Authors:** first-read (non-reread) books with an author, grouped on the author with case, punctuation and spaces removed ("R. J. Barker" = "RJ Barker"; non-Latin names fall back to a case-insensitive match). Displays the first spelling seen. Sorted by count, then by average rating of rated books. Top 10.
- **Top Series:** series with at least 1 read book, sorted by average rating of rated read books, then read count. Top 10.
- **Quotes:** every quote from every book, attributed "— Title, p. N".

### 5.3 Stats tiles
Built from `chartsReadBooks()` = status `read` and **not** a reread:

| Tile | Formula |
|---|---|
| Books Read | count |
| Books / Month (avg) | count ÷ calendar months from earliest `dateFinished` to now (inclusive, min 1) |
| Pages Read | Σ pages |
| Longest Book | max pages (title shown, truncated to 24) |
| Fastest Read | min (finished − started) days ≥ 0 |
| Rereads | read books with `isReread` |
| DNF Count | status dnf |
| TBR Size | status tbr |

### 5.4 Charts
All charts are hand-built HTML/SVG, use colours from the palette tokens, share a hover tooltip (`data-tip`), and can be shown as a table. Each has an explanatory empty state.

| Chart | Form | Data rules |
|---|---|---|
| Fiction vs Nonfiction | 100% segmented bar + legend | Tag `fiction` vs `nonfiction`/`non-fiction` (case-insensitive). Untagged books are counted and noted as excluded. Inline % label only if the segment is ≥12%. |
| Page Length Distribution | Donut (200×200 viewBox) | Bins <200, 200–349, 350–499, 500–699, 700+, using the sequential ordinal ramp from light to dark. Label if the slice is ≥6%. Unknown length is excluded and noted. |
| Books by Genre / Tag | Horizontal bars, single hue | Tags other than fiction/nonfiction. More than 8 tags → top 7 + "Other". Count sits inside the bar if the bar is ≥22% of max, otherwise to its right. |
| Pages Read per Month | Line + 10% area wash, horizontal scroll | Books with both a finish date and pages. A continuous month series runs from the first month to the current month, with zero-filled gaps. Y max rounded up by `niceMax`. 5 grid lines. At most ~10 x labels. End point marked and labelled. 10px invisible hit circles for tooltips. |

### 5.5 Discovery
- **Pick For Me** (Discovery tab and TBR toolbar): weighted random pick from the TBR. Up Next weighs 3, Someday weighs 1. The result (cover + Start Reading) is shown in the panel it was requested from: under the TBR toolbar, or in the Discovery panel.
- **Mood-Based Picks:** a `<select>` of every tag in the library. Shows TBR books with the chosen tag. The selection is kept across re-renders.
- **Recommended For You:** from read books rated ≥4, add the rating to a score for each of their tags and for their author. Score each TBR book by the sum of its matching tag and author scores. Top 8 with score > 0.
- **Compare With Another Profile:** pick another profile.
  - *Both read:* your read books that match (`sameBook`) one of their read books, with both ratings.
  - *Their 4★+ you don't have:* their read books rated ≥4 with no match anywhere in your library. Capped at 50, each with **Add to my TBR**.
- **Reading Challenges:** bingo (5×5 grid, click toggles green) or checklist (checkboxes). Shows "done / total", an Edit button, and a ✕ delete button (with confirm).

### 5.6 Multi-select operations
- **Select all visible:** every `.book-card[data-id]` in the active view.
- **Delete Selected:** confirm, then remove.
- **Check for Covers:** runs through the selected books one at a time, and the button shows "Checking i/N". For each book:
  1. Probe the current cover. If it is OK, skip. If it timed out, count it as *stalled* and **leave it alone**.
  2. If it is missing, run `fetchIsbnData(isbn)`. If that gives no cover, run `fetchByTitleAuthor(title, author)`.
  3. Replace the cover only if a different, verified cover was found.
  The toast reports how many were updated, and how many stalled if any did.
- **Clear Covers:** blanks the covers (with a confirm) so a wrong-but-loadable cover can be resolved again.

### 5.7 Profiles
- **Switch:** save → set `activeId` → repoint `data` → leave select mode → save → apply theme → re-render. The toast names the new profile.
- **Theme:** the profile colour becomes `--accent`, and `--accent-ink` is set to dark or white based on Rec. 601 luma (> 0.6 → dark). The document title becomes "‹name› — Book Tracker".
- **Send to…** and **Add to my TBR** both use `copyForProfile`. It copies bibliographic fields, series, format, pages, shelves, and tags. Status is set to tbr/someday, notes to "From ‹sender›", and reading history is dropped. Both refuse duplicates via `sameBook`.
- **`sameBook(a,b)`:** if both books have ISBNs (digits/X only), compare ISBNs. Otherwise compare trimmed, lower-cased title **and** author.

---

## 6. External Integrations (Metadata & Covers)

No API keys. Every call is wrapped so a failure falls through to the next source.

| Service | Endpoint | Used for |
|---|---|---|
| Open Library Books API | `openlibrary.org/api/books?bibkeys=ISBN:…&jscmd=data` | ISBN metadata + cover (the code notes it currently 404s) |
| Open Library Search | `openlibrary.org/search.json` (`q=isbn:…` or `title=` / `author=`) with an explicit `fields` list | ISBN fallback, title search, `cover_i`, subjects |
| Open Library Covers | `covers.openlibrary.org/b/{id\|isbn}/…-L.jpg?default=false` | covers (`default=false` → 404 instead of a blank 1×1 gif) |
| Google Books | `googleapis.com/books/v1/volumes?q=…` | metadata/cover fallback (often returns HTTP 429 to anonymous callers) |
| Apple iTunes Search | `itunes.apple.com/search?entity=ebook` via **JSONP** | last-resort cover art only |

### 6.1 `fetchIsbnData(isbn)`
1. OL Books API → metadata + cover candidates (large/medium/small).
2. OL search by ISBN → `cover_i` candidate. Also metadata if step 1 found nothing.
3. Add the OL cover-by-ISBN candidate. Take the first candidate that loads.
4. If metadata **or** cover is still missing → Google Books (first volume).
5. If the cover is still missing and metadata exists → iTunes covers.
6. Returns `{title, author, pages, isFiction, genres, source, cover}` or `null`.

### 6.2 `fetchByTitleAuthor(title, author)`
OL search (with author, then retried without it) → cover candidates from `cover_i` and ISBN → Google Books `intitle:/inauthor:` → iTunes. Also returns `isbn`.

### 6.3 `searchTitleCandidates(title, author)`
Up to 8 OL docs, or Google Books if OL finds none. Covers are **not** probed here: a broken `<img>` in the picker falls back to its placeholder by itself. The chosen candidate's cover goes through the full resolver.

### 6.4 Cover verification
- `probeCover(url)` loads an `Image` and returns one of three states: **ok** (`naturalWidth > 1`), **missing** (error or 1×1), **timeout** (20 s).
- Results are cached per URL, **except timeouts**. A timeout reflects the network, not the URL.
- `resolveCover(candidates)` returns the first OK URL plus an `inconclusive` flag if any candidate timed out.
- Rendered `<img>` tags use inline `onload="checkCoverLoaded(this)"` / `onerror="coverToPlaceholder(this)"`, both exposed on `window`, to swap in a text placeholder at display time.

### 6.5 iTunes matching safeguards
- JSONP with a random callback name and an 8 s timeout. This is needed because Apple sends no CORS header for `file://` pages.
- Title match: normalized equality, or one title is the other plus a whole word or more (no loose substring match).
- Author match: at least one shared significant word (length > 2) after normalization. Passes if either side is empty.
- Tries `title author` first, then title only. Prefers the 600×600 artwork URL, with 100×100 as fallback.

### 6.6 Genre extraction (`extractGenreAndFiction`)
- Fiction guess: `non-fiction` found anywhere → false; otherwise the word `fiction` → true; otherwise null.
- Subjects are split on `/` and `,`. Parts are dropped if they contain non-ASCII characters, contain parentheses, match `SUBJECT_NOISE_RE`, are exactly fiction/nonfiction/general, or **don't** match the `GENRE_KEYWORDS_RE` allow-list.
- Kept parts are lower-cased and de-duplicated, max 4.
- `mergeTagSuggestions` appends `fiction`/`nonfiction` plus the genres to the user's existing tags without duplicates, and reports what it added.

---

## 7. Bulk Import Pipeline

### 7.1 Parsing (`parseBulkFile(text, splitColumns)`)
1. Split into non-blank lines. `splitCsvLine` handles quoted fields and `""` escapes.
2. **Header detection** (only if the first row has more than one column): exact lower-case matches against
   `isbn: isbn, isbn13, isbn10, isbn-13, isbn-10`, `title: title, book, book title, name`, `author: author, authors, by, writer`.
3. **Column mapping:**
   - Header found, no ISBN column named → guess the ISBN column from unclaimed columns.
   - Single column → ISBN list.
   - No header → guess the ISBN column. The remaining columns are title, then author.
4. **ISBN column guess (`findIsbnColumn`):** uses the first 50 data rows. A column qualifies if at least 50% of its *non-empty* cells look like ISBNs. The column with the most hits wins. Measuring against non-empty cells means a column that is mostly blank can still be detected.
5. **ISBN check (`looksLikeIsbn`):** strip everything except digits and X. Length must be 10 or 13. No checksum validation.
6. **De-duplication** against the library and against earlier rows:
   - with ISBN → by ISBN;
   - without → by `title|author`, or by title alone when the row has no author (a lookup will fill in the author later).
7. **Ambiguous rows** (both ISBN and title): the default counts them as one book. With `splitColumns`, the row becomes two entries (ISBN-only, then title/author). `mixedRows` is counted either way.

The included `book-test-data - Test Data(2).csv` exercises step 4. It has a `Title,ISBN` header, 10 rows with only a title, then 10 rows with only an ISBN.

### 7.2 Execution
- Runs one book at a time, pausing **300 ms** between lookups to avoid rate limits.
- Each entry is looked up with `fetchIsbnData` or `fetchByTitleAuthor`.
- The book is **always added**, even with no match. Values from the file take priority over lookup values. Lookup supplies cover, pages, auto-tags, and ISBN if missing. Status is tbr at the chosen priority.
- `saveData()` runs after every book, so a stop or crash keeps the progress so far.
- Stop takes effect before the next entry. The summary shows added / matched / unmatched.

---

## 8. Visual Design System

### 8.1 Tokens (`:root`, redefined under `prefers-color-scheme: dark`)

| Group | Tokens |
|---|---|
| Surfaces & text | `--bg`, `--panel`, `--border`, `--ink`, `--muted`, `--chip` |
| Brand / state | `--accent`, `--accent-ink` (overridden by profile), `--gold`, `--green`, `--red` |
| Categorical chart | `--series-1` … `--series-8` |
| Chart chrome | `--chart-surface`, `--chart-grid`, `--chart-axis`, `--chart-muted` |
| Sequential ramp | `--ordinal-1` … `--ordinal-5` |

Dark mode follows the operating system only. There is no manual theme toggle.

### 8.2 Typography & shape
- Font: `"Segoe UI", Arial, sans-serif`. h1 1.5rem, h2 1.1rem, h3 0.98rem. Body controls about 0.86rem, badges 0.68rem.
- Radii: buttons 7px (small 6px), panels and cards 10px, modals 12px, tabs and profile chip are pills (20px).
- Buttons: primary is filled accent. Modifiers are `.secondary` (chip + border), `.danger` (red), `.small`. Hover brightens by 8%.

### 8.3 Layout & responsiveness
- Card grid: `repeat(auto-fill, minmax(220px, 1fr))`. Library grid: `minmax(118px, 1fr)` with 2:3 covers.
- Stat tiles `minmax(160px)`, compare columns `minmax(240px)`.
- A single breakpoint at 700px: `.two-col` and inline form rows collapse to one column.
- The monthly chart scrolls horizontally inside `.chart-scroll`. Other charts stretch to fit.
- Layering: sticky selection bar z 50, modals z 100, toast z 200, tooltip z 300.

### 8.4 Accessibility notes (current state)
- Every chart has a table view.
- Tooltips appear on mouse hover only (no keyboard or touch equivalent).
- Cover images use `alt=""`. The title is shown next to them, or used as placeholder text.
- Tabs are plain buttons without ARIA tab roles. Modals don't trap focus and don't close on Escape.
- Form labels in the book dialog are not linked to inputs with `for`. The profile dialog's "Add a profile" label is.

---

## 9. Security & Privacy

- **Data locality:** libraries never leave the browser. Outbound requests carry only ISBNs, titles, and authors to the public APIs above.
- **Output escaping:** `esc()` is applied to user and API text inserted via `innerHTML`, including attribute values such as cover `src`, `data-title`, and every book/challenge id in `data-*` attributes. The chart tooltip is filled with `textContent`, so decoded `data-tip` text can never become markup.
- **JSONP:** the iTunes response runs as a script with page privileges. Apple is implicitly trusted.
- **Import:** JSON is parsed with `JSON.parse`, then every profile, book and challenge is normalized (§3.3), so imported values reach the HTML templates only in their expected types.

---

## 10. Known Limitations & Resolved Issues

### 10.1 Fixed

These were found while reviewing the code and then fixed. Each fix was checked in headless Edge against deliberately malformed data.

| # | Issue | Fix |
|---|---|---|
| 1 | Chart tooltip turned decoded `data-tip` text back into HTML (tag names could inject markup) | Tooltip uses `textContent` |
| 2 | Book and challenge ids went into attributes unescaped | Ids pass through `esc()`, and normalization turns them into strings |
| 3 | An empty bulk-import file threw a `TypeError` | `parseBulkFile` always returns `{entries, mixedRows}` |
| 4 | TBR shelf/genre dropdowns were filled only once (stale after edits or profile switch) | Rebuilt on every render by `fillFilterSelect`, keeping the current choice if it still exists |
| 5 | "Today" was the UTC date, and `YYYY-MM-DD` was parsed as UTC (off-by-one day or month west of UTC) | `todayStr()` / `parseYmd()` use the local calendar throughout. The fastest-read day count is rounded (DST). The monthly chart extends to a future-dated finish instead of failing. |
| 6 | Imported and stored libraries weren't normalized (e.g. a missing title crashed rendering) | `normalizeLibrary` / `normalizeBook` / `normalizeChallenge` run on every entry path. Selection mode ends when the library is replaced. |
| 7 | `localStorage` write errors escaped `saveData()` | Caught. The user sees a 6-second error toast that later "Saved" toasts can't overwrite. |
| 8 | Bingo squares couldn't be named | Squares can be named at creation and later with the Edit button |
| 9 | Deleting a challenge had no confirmation | A confirm was added |
| 10 | Dragging between tiers had no preview, a cancelled drag left the DOM out of order, and reordering in a filtered view caused `sortIndex` collisions | Cross-container preview, re-render on a cancelled drag, and filtered reorders fill the visible books' original slots so hidden books keep their positions |
| 11 | Pick for me on the TBR tab drew its result on the Discovery tab | The result renders under the TBR toolbar |
| 12 | Top Authors split name variants and counted rereads | Normalized author key; rereads excluded |

### 10.2 Still open

1. **Full re-render on every change.** Every mutation and tab switch rebuilds all 13 views. This is simple and correct but scales linearly with library size. It's a deliberate trade-off rather than a bug.
2. **Legacy data key** `bookTrackerData_v1` is left in place after migration, on purpose. Deleting it would break an older copy of the file opened in the same browser.
3. **Touch devices.** HTML5 drag-and-drop doesn't work on most touch devices, and chart tooltips need a mouse (every chart has a table view).
4. **Drag position in grids.** The drop position is based only on vertical position, so ordering within a single row of a multi-column card grid is approximate.
5. **Comma-separated labels.** Challenge goals and bingo squares can't contain commas.
