# Book Tracker — Design Document

Describes the application as it exists in `book-tracker.html` (commit `d13ecb7` plus the fixes, usability changes and new features listed in §10). It documents current behaviour; it is not a proposal.

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
| `<style>` | 13–293 | Design tokens, light/dark themes, all component CSS |
| Markup | 295–922 | Header, selection bar, tab bar, 8 view panels (one of them holding two sub-panels), 5 modal dialogs, toast, chart tooltip |
| `<script>` | 923–4723 | Whole app in one strict-mode IIFE |

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
- **Event handling:** delegated. Listeners sit on `document.body` or on stable containers and dispatch on `data-*` attributes (`data-detail-id`, `data-action`, `data-id`, `data-tier-move`, `data-move` / `data-move-list`, `data-send-id`, `data-log-add` / `data-log-save` / `data-log-remove`, `data-listen-add` / `data-listen-save` / `data-listen-remove`, `data-audio-save` / `data-pages-save`, `data-log-kind`, `data-grab-from`, `data-chart-toggle`, `data-tip`, `data-select-id`, `data-challenge`, `data-prof-*`). Re-rendering therefore never orphans a listener.
- **View switching:** each tab is a `.view` div, and the active one has `.active` (CSS `display:block`). Switching tabs moves the class and calls `renderAll()`. The Logs tab has a second level of the same idea: two `.log-panel` divs, one `.active` at a time (§4.3).
- **Module-level UI state** (not persisted): `selectionMode`, `selectedIds` (Set), `editingId`, `titleHits`, `bulkEntries`, `bulkRawText`, `bulkStopRequested`, `dragSrcId`, `dragDropped`, `chartViewMode`, `librarySort`, `sendBookId`, `coverProbeCache`, `saveCount` and `pendingUndo` (undo, §5.8), `editCoverData` (upload in the open Edit dialog, §5.13), `detailBookId` (§5.14). Search boxes and the series sort are read straight from their inputs at render time.

### 2.3 Utilities

| Name | Purpose |
|---|---|
| `$(id)` | `document.getElementById` shorthand |
| `splitList(str)` | `"a, b,,c"` → `["a","b","c"]` |
| `uid()` | base-36 timestamp + random suffix |
| `esc(s)` | HTML-escapes `& < > " '` |
| `toast(msg, isError, opts)` | Bottom-centre notification: 2.2 s normally, resets its timer on repeat calls. Error toasts last 6 s and can't be replaced by a normal toast while showing. `opts.action = {label, run}` adds a clickable button (Undo), and `opts.ms` sets the duration. |
| `matchesSearch(book, query)` | The one search rule for every tab: case-insensitive substring of title or author. An empty query matches everything. |
| `showEmpty(emptyId, isEmpty, filtering)` | Shows a tab's empty-state message. While a search or filter is active it reads "No books match the current search or filters." instead of the default text. |
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
| `lastBackupAt` | ISO timestamp \| null | set by Export (this profile) and Backup All (every profile); null = never |
| `lastBackupBookCount` | int ≥ 0 | book count at that backup |
| `readingGoals` | `{ "YYYY": int }` | yearly book goals (§5.12); normalized to 4-digit year keys with targets 1–9999 |
| `data` | Library | `{ books: Book[], challenges: Challenge[] }` |

`normalizeStore()` fills in missing profile fields, only accepts `#rrggbb` colours and parseable `lastBackupAt` timestamps, and repairs an invalid `activeId`. It runs on load and on backup restore. Every library passes through `normalizeLibrary()` → `normalizeBook()` / `normalizeChallenge()` on load, legacy migration, backup restore, and single-library import. These drop non-object entries, restrict enums to known values, coerce numbers (clamping progress to 0–100 and rating to 0–5), validate dates, turn lists into string arrays, default a missing title to "Untitled", and pad or trim bingo cards to 25 squares.

### 3.4 Book

| Field | Type | Default | Set by / used for |
|---|---|---|---|
| `id` | string | `uid()` | |
| `isbn` | string | `""` | lookup key; dedupe key |
| `title` | string | `""` | required in form |
| `author` | string | `""` | comma-joined if several |
| `cover` | URL string | `""` | only verified URLs are stored by lookups |
| `coverData` | image data URL | `""` | uploaded cover (§5.13); shown instead of `cover` when present. Normalized to `data:image/(png\|jpeg\|gif\|webp);base64,…` of at most 200,000 chars, else `""` |
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
| `ownership` | string[] | `[]` | how the book is held (§5.15). Any of `owned-physical`, `owned-ebook`, `owned-audiobook`, `borrowed`, `loaned`, `wishlist`, several at once. Stored in `OWNERSHIP` order; unknown values are dropped. Replaces the older `owned` boolean, which `normalizeOwnership` migrates |
| `sortIndex` | int | *(absent)* | added by TBR drag-and-drop or ↑/↓ |
| `topReadRank` | int | *(absent)* | added by Top 5 drag-and-drop or ↑/↓ |
| `readingLog` | `{date, pages}[]` | `[]` | daily page log (§5.16). One entry per day, oldest first. `normalizeReadingLog` drops bad dates and non-positive counts, and adds same-day entries together (capped at `READING_LOG_MAX` = 5000) |
| `audioLength` | int (minutes) \| null | `null` | audiobook running time (§5.17). Clamped to `AUDIO_LENGTH_MAX` = 12000 (200 h); anything non-positive or unparseable becomes `null` |
| `listenLog` | `{date, minutes}[]` | `[]` | daily listening log (§5.17). The exact twin of `readingLog` with minutes in place of pages, normalized by `normalizeListenLog` the same way (capped at `LISTEN_LOG_MAX` = 1440, one day) |

Both audio fields are kept on *every* book, not only audiobooks, so changing a book's format away from audiobook and back never throws its times away.

A **reread** is modelled as a *separate* book record: a copy with a new `id`, `isReread: true`, `status: "reading"`, reset progress, dates, notes, reading log and listening log, and copied quotes. `audioLength` is a property of the recording rather than of one pass through it, so it is carried over — as it is by `copyForProfile`.

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

Profile-level settings (`readingGoals`, `lastBackupAt`) travel only in the Backup All file. A single-profile Export carries just the library. Uploaded covers are inside book records, so both formats include them.

**Import** checks the file's contents: a `profiles` array means a full restore, which replaces every profile after a confirm. A `books` array means a single library, which replaces the active profile's library (with a confirm if it isn't empty).

---

## 4. User Interface

### 4.1 Global chrome

- **Header:** app title, a profile chip (emoji + `<select>` switcher), **Manage Profiles**, and a backup reminder (§5.9) with a **Back up now** link, shown only when a backup is due. Action buttons: **+ Add Book**, **Select Books**, **Export**, **Import**, **Backup All Profiles**, **Bulk Import (CSV/ISBNs)**. The two file inputs are hidden and triggered by those buttons.
- **Selection bar** (sticky, shown only in select mode): count, Select all visible, Clear, Check for Covers, Clear Covers, Delete Selected, Done. Bulk edits: a comma-separated text box, an "as genre/mood tags / as shelves" select, and **Add to Selected**; a "Move selected to…" status select and **Apply** (§5.6).
- **Tab bar:** pill buttons with `data-view`.
- **Toast** and **chart tooltip:** single shared floating elements.

### 4.2 Book card (`bookCardHTML`)

Used in the TBR, Reading, Read, DNF, Series, Mood Picks, and Recommendations lists.

- Clicking the **cover** or the **title** opens the read-only detail view (§5.14). The title is a `<button class="title-link">`, so it's keyboard reachable.

- 56×80 cover or a text placeholder (first 20 chars of the title).
- Title, author ("Unknown author" if blank).
- Badges: series + number, format, *Next Up* (gold), *★ Top Read* (gold), *Reread*, *Wishlist* (dashed accent outline; books in the `wishlist` category that are either on the TBR, wherever the card appears, or shown on the Read tab), each shelf, each tag.
- Cover: the uploaded `coverData` if present, otherwise `cover`, otherwise the placeholder.
- Status extras: progress bar and the **Log pages** row (reading, §5.16), stars and notes excerpt (read), "Dropped: reason" (dnf).
- Actions by status:

| Status | Buttons |
|---|---|
| all | Edit |
| tbr | Start Reading, Move to Up Next / Remove from Up Next |
| reading | Mark Finished, DNF |
| read | Log Reread |
| all (only if >1 profile) | Send to… |
| tbr, in the Up Next / Someday tiers only | ↑ / ↓ (disabled at the ends of the tier) |

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

The order runs from browsing what you own, through what you are reading and how fast, to
what you might read next and what you thought of it all. **Library** is the tab the app
opens on.

| Tab | Contents |
|---|---|
| **Library** | Cover-grid browse view of every book (2:3 covers, title, author, no action buttons; cover or title opens the detail view). Search (title/author), an ownership-category filter, and sort by title, by author (authorless last) or by category (§5.15). Sorting by category also prints each book's categories under its author, so the order has a visible reason. Shows a book count, or "N of M books" while searching. |
| **Read** | Search, sort (newest/oldest finished, highest rated, title A–Z), "Rereads only" checkbox, then cards with status `read`. |
| **Logs** | The day-to-day tracker, split into two **sub-tabs** — 🎧 *Audiobooks* and 📖 *Books* — because the two kinds of book are counted in different units. Each side shows only books with status `reading`: five stat tiles over what is in progress, search, one card per book (editable total, amount done and left, a log row, the day's entry) and its own per-day line chart with a per-book picker and a "View as table" toggle. Both sides are generated from the same code (§5.17). |
| **Pacing** | "‹year› reading goal" number box + **Save goal** (Enter also saves), the stat tiles (led by the goal tile when a goal is set, then the page-log tiles once anything is logged), and the **Currently Reading Pace** table (§5.16). No charts — the ones that describe the library live on Rankings. |
| **TBR & Discovery** | *Next Up Suggestions* panel (shown only if any). Search, shelf filter, ownership-category filter, genre/mood filter, 🎲 Pick for me (result under the toolbar). Two draggable tiers, **Up Next** and **Someday**, ordered by `sortIndex`. Then 🌙 Mood-Based Picks, 👥 Compare With Another Profile, ✨ Recommended For You and 🎯 Reading Challenges. |
| **DNF** | Search (title/author), then cards with status `dnf`. |
| **Series %** | Sort select: Name A–Z (default), Closest to complete, Next Up available first (§5.10). One card per series: read count / total, % bar, Next Up badge, member books sorted by number. |
| **Rankings** | Two-column grid: 🏆 Top 5 Reads (drag, or ↑/↓ on each row), 🔁 Reread Candidates, ✍️ Top Authors (top 10), 📖 Top Series (top 10). Then 💬 Favorite Quotes, and the four charts that describe the library rather than the pace: Fiction vs Nonfiction, Page Length Distribution, Books by Genre/Tag and Pages Read per Month (§5.4). |

**Sub-tabs.** Only the Logs tab has them. They work exactly as the tab bar does — one delegated
listener on `#logsSubtabs`, `.active` moved between the buttons and between the `.log-panel`
divs — and the chosen side is remembered in `activeLogKind` for as long as the page is open.


### 4.4 Dialogs

All dialogs use `.modal-overlay` / `.modal`. Clicking the backdrop closes a dialog, except the bulk importer while it is running.

**Add / Edit Book**
1. *ISBN lookup:* ISBN field + Lookup. Fills title, author, pages, and cover, and merges tag suggestions.
2. *Search by title:* up to 8 candidate buttons (cover, title, author · year · pages). Picking one fills the form, resolves a working cover, and merges tags.
3. Core fields: Title*, Author, Cover URL, **Upload cover…** (preview thumbnail, Replace / Remove upload, status line; §5.13), Series name / #, Format, Pages, Status.
4. **Pages** and **Audiobook length** depend on the *format* rather than the status, and swap places: `updateFormatDependentFields` shows whichever measure the chosen format is counted in and hides the other, so an audiobook's dialog has no page box. Audiobook length is an hours box plus a minutes box; leaving both empty, or a total of zero, stores `null`, and a total over `AUDIO_LENGTH_MAX` is refused rather than clamped, so a mistyped running time is never silently turned into a different one. A hidden measure is only hidden, never cleared — a page count survives a trip through *Audiobook* and back, exactly as the audio fields survive the reverse.
5. Fields that depend on status:

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

6. Always shown: **Ownership** (a checkbox per category, built from `OWNERSHIP`; a book can be in several at once), Shelves (comma-separated), Genre/mood tags (comma-separated).
7. Actions: Delete (edit only, confirm, then an Undo toast; §5.8), Cancel, Save. Save lays the form values over the existing record (or `{}` for a new book) and passes the result through `normalizeBook`. Fields the form doesn't show survive the edit: `dateAdded`, `sortIndex`, `topReadRank`, `readingLog`, `listenLog`. `coverData` comes from the upload state, and `ownership` from the checkboxes.

**Bulk Import:** two stages.
- *Preview:* "Found N new book(s)". An optional **"Title and ISBN columns are separate lists"** checkbox appears only when some rows contain both, and its hint gives the import count for each reading. Then "Add as" priority, Cancel, and Start Import.
- *Progress:* bar, "Looking up i of N" label, scrolling ✓/⚠ log, and Stop → Done.

**Profiles:** one row per profile with an editable emoji input, a name input (saves on every keystroke, no re-render so the caret isn't lost), 8 colour swatches, Active badge or Switch to, Duplicate (deep JSON copy), Delete (blocked for the last profile, with confirm), and "N books · added date". Below: Add a profile (Enter submits). New profiles get colours in turn from the palette.

**Send to another profile:** one button per other profile.

**Book detail** (read-only): opened from a card's or Library item's cover/title. Shows the cover (110×165), title, author, a definition list of every field that has a value, then the editable reading and listening logs and full sections for the DNF reason, notes (line breaks kept), and every quote. Buttons: **Edit** (closes this and opens Add/Edit for the book) and **Close**; clicking the backdrop also closes it.

### 4.5 Native prompts

`confirm()` guards bulk delete, clear covers, single delete, restore/replace on import, profile delete, and challenge delete. **New Challenge** uses `prompt()` for the name, `confirm()` to choose bingo vs checklist, and `prompt()` for comma-separated goals or bingo squares (up to 25; blank leaves "Square N"). A challenge's **Edit** button re-prompts for the name and the comma-separated labels. Bingo ticks stay with their square position, and checklist ticks stay with goals whose wording is unchanged.

---

## 5. Feature Logic

### 5.1 Next Up suggestions (series)
For each series, find the highest-numbered **read** book. The suggestion is the lowest-numbered **tbr** book above it. Books with no series number are ignored. Suggestions appear in the TBR panel, as a badge on the matching TBR cards, and on the Series tab.

### 5.2 Rankings
- **Top 5 Reads:** read books with `isTopRead`. Books with a `topReadRank` come first in rank order, then the rest by rating descending, cut to 5. Dragging and the ↑/↓ buttons both call `applyTopReadOrder`, which sets `topReadRank` on the visible rows.
- **Reread Candidates:** any book with `isRereadCandidate`, whatever its status.
- **Top Authors:** first-read (non-reread) books with an author, grouped on the author with case, punctuation and spaces removed ("R. J. Barker" = "RJ Barker"; non-Latin names fall back to a case-insensitive match). Displays the first spelling seen. Sorted by count, then by average rating of rated books. Top 10.
- **Top Series:** series with at least 1 read book, sorted by average rating of rated read books, then read count. Top 10.
- **Quotes:** every quote from every book, attributed "— Title, p. N".

### 5.3 Stats tiles
Built from `chartsReadBooks()` = status `read` and **not** a reread. When the current year has a goal, a **‹year› Goal** tile comes first (§5.12):

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
All charts are hand-built HTML/SVG, use colours from the palette tokens, share a hover tooltip (`data-tip`), and can be shown as a table. Each has an explanatory empty state. The first four live on **Rankings** and describe the library; the last lives on **Logs**, once per sub-tab, and describes the last 30 days. The "View as table" buttons share one delegated handler, which redraws just the one daily chart or all four library charts.

| Chart | Form | Data rules |
|---|---|---|
| Fiction vs Nonfiction | 100% segmented bar + legend | Tag `fiction` vs `nonfiction`/`non-fiction` (case-insensitive). Untagged books are counted and noted as excluded. Inline % label only if the segment is ≥12%. |
| Page Length Distribution | Donut (200×200 viewBox) | Bins <200, 200–349, 350–499, 500–699, 700+, using the sequential ordinal ramp from light to dark. Label if the slice is ≥6%. Unknown length is excluded and noted. |
| Books by Genre / Tag | Horizontal bars, single hue | Tags other than fiction/nonfiction. More than 8 tags → top 7 + "Other". Count sits inside the bar if the bar is ≥22% of max, otherwise to its right. |
| Logged per Day | Line + 10% area wash, last 30 days, one on each side of the **Logs** tab | `renderDailyLogChart(kind)`, drawn once and used twice. Sums that side's log across its books, or one book on its own when the picker names it. Y max and tick labels come from the kind (`niceTimeMax` + "1h 30m" for minutes, `niceMax` + a plain number for pages). A dot marks each day with something logged, so an isolated day is visible where the line alone would look flat. End point marked and labelled; 9px invisible hit circles for tooltips; a footnote gives the 30-day total and daily average. Table view lists only days with something logged, newest first. Unlike the charts above it counts as things are read rather than when a book is finished, so it includes books still in progress and ones later dropped. |
| Pages Read per Month | Line + 10% area wash, horizontal scroll | Books with both a finish date and pages. A continuous month series runs from the first month to the current month, with zero-filled gaps. Y max rounded up by `niceMax`. 5 grid lines. At most ~10 x labels. End point marked and labelled. 10px invisible hit circles for tooltips. |

### 5.5 Discovery
- **Pick For Me** (the 🎲 button in the TBR toolbar): weighted random pick from the TBR. Up Next weighs 3, Someday weighs 1. The result (cover + Start Reading) is shown under the toolbar. `pickForMe(resultId)` still takes the container to draw into, from when there was a second copy of this button on a separate Discovery tab.
- **Mood-Based Picks:** a `<select>` of every tag in the library. Shows TBR books with the chosen tag. The selection is kept across re-renders.
- **Recommended For You:** from read books rated ≥4, add the rating to a score for each of their tags and for their author. Score each TBR book by the sum of its matching tag and author scores. Top 8 with score > 0.
- **Compare With Another Profile:** pick another profile.
  - *Both read:* your read books that match (`sameBook`) one of their read books, with both ratings.
  - *Their 4★+ you don't have:* their read books rated ≥4 with no match anywhere in your library. Capped at 50, each with **Add to my TBR**.
- **Reading Challenges:** bingo (5×5 grid, click toggles green) or checklist (checkboxes). Shows "done / total", an Edit button, and a ✕ delete button (with confirm).

### 5.6 Multi-select operations
- **Select all visible:** every `.book-card[data-id]` in the active view.
- **Delete Selected:** confirm, then remove with an Undo toast (§5.8).
- **Add to Selected (tags/shelves):** splits the text box on commas and appends each label to the chosen field of every selected book, skipping labels the book already has (case-insensitive). Books that gain nothing aren't counted.
- **Move selected to status:** does what that status's card button does. *reading* sets `dateStarted` if unset. *read* sets progress 100 and `dateFinished` = today. *dnf* sets `dateFinished` = today. *tbr* only changes the status. Books already in the target status are skipped, and no dialog opens.
- Both bulk edits go through `bulkUpdate`: each changed book is copied, edited, and rebuilt with `normalizeBook` before one save and redraw. The selection stays in place afterwards.
- **Check for Covers:** runs through the selected books one at a time, and the button shows "Checking i/N". For each book:
  0. Books with an uploaded cover (`coverData`) are skipped entirely.
  1. Probe the current cover. If it is OK, skip. If it timed out, count it as *stalled* and **leave it alone**.
  2. If it is missing, run `fetchIsbnData(isbn)`. If that gives no cover, run `fetchByTitleAuthor(title, author)`.
  3. Replace the cover only if a different, verified cover was found.
  The toast reports how many were updated, and how many stalled if any did.
- **Clear Covers:** blanks the cover **URLs** (with a confirm) so a wrong-but-loadable cover can be resolved again. Uploaded covers are left alone; they can only be removed in the Edit dialog.

### 5.7 Profiles
- **Switch:** save → set `activeId` → repoint `data` → leave select mode → save → apply theme → re-render. The toast names the new profile.
- **Theme:** the profile colour becomes `--accent`, and `--accent-ink` is set to dark or white based on Rec. 601 luma (> 0.6 → dark). The document title becomes "‹name› — Book Tracker".
- **Send to…** and **Add to my TBR** both use `copyForProfile`. It copies bibliographic fields (including an uploaded `coverData`), series, format, pages, audiobook length, shelves, and tags. The copy starts with no ownership category — how the sender holds their copy says nothing about the recipient's. Status is set to tbr/someday, notes to "From ‹sender›", and reading history is dropped. Both refuse duplicates via `sameBook`.
- **`sameBook(a,b)`:** if both books have ISBNs (digits/X only), compare ISBNs. Otherwise compare trimmed, lower-cased title **and** author.

### 5.8 Undo for book deletes
- Deleting books (single delete in the Edit dialog, or Delete Selected) goes through `deleteBooksWithUndo(ids, message)`. It records each removed book with its index in `data.books`, removes them, saves, and shows a toast with an **Undo** button for 8 seconds (`UNDO_MS`).
- **Undo** puts the books back at their original indexes (inserted in ascending order) and shows "Restored N books".
- Deleted books are held in memory only (`pendingUndo`). The offer lapses when the 8 seconds run out, or on **any** later `saveData()`: every save increments `saveCount`, and an undo recorded at an older count is dropped. Switching profile and importing both save, so an undo can never restore books into a different library. Clicking a stale Undo shows "Nothing to undo".
- Challenge and profile deletes remain confirm-only.

### 5.9 Backup reminder
- **Export** sets `lastBackupAt` / `lastBackupBookCount` on the active profile. **Backup All Profiles** sets them on every profile *before* building the file, so a restored backup knows when it was made.
- `backupReminder(profile)` returns a message when the profile has books and either **≥ 14 days** (`BACKUP_STALE_DAYS`) have passed since the last backup, or **≥ 20 books** (`BACKUP_STALE_GROWTH`) have been added since. A never-backed-up profile is measured from its `created` date with a baseline of 0 books, so a brand-new library isn't flagged on its first book.
- Messages: "⚠ Never backed up", "⚠ Last backup N days ago", "⚠ N books added since last backup". Hovering shows the exact last-backup time. **Back up now** runs Backup All Profiles. It's rendered by `renderBackupNudge()` as part of `renderAll()`. Nothing blocks.

### 5.10 Series sort
`SERIES_SORTERS`: **name** (`localeCompare`); **complete** (read ÷ total descending, then more books read, then name); **nextup** (series with a Next Up suggestion first, then name).

### 5.11 Reordering without drag
TBR tier cards and Top 5 rows have ↑/↓ buttons (`moveButtonsHTML`). `moveInList(list, id, dir)` reads the list's current on-screen order from the DOM, just as a drop does. It then swaps the book with its neighbour and passes the new order to the same function the drag uses: `applyTierOrder(tier, ids)` for TBR (including the filtered-view slot logic) or `applyTopReadOrder(ids)`. Finally it saves, redraws, and puts focus back on the same button so repeated presses keep moving the book.

### 5.12 Reading goal
- Stored per profile in `readingGoals`, keyed by calendar year, so past years' goals stay put.
- **Save goal** (or Enter) validates a whole number from 1 to 9999. Saving an empty box removes the current year's goal. It saves and re-renders Stats.
- Progress counts `chartsReadBooks()` (status `read`, not rereads, the same definition as **Books Read**) whose `dateFinished` falls in the current local calendar year. Read books without a finish date don't count toward the goal.
- The tile shows `read / target`, "‹year› Goal · N%", then either "· N to go" or "· reached!", plus a progress bar capped at 100%. There's no tile when no goal is set.
- The input shows the saved goal on each render, except while it has focus, so typing isn't overwritten.

### 5.13 Uploaded covers
- **Upload cover…** in the Add/Edit dialog opens an `image/*` file picker. `imageFileToCoverData(file)` then:
  1. rejects anything that isn't an image, or is over 25 MB;
  2. decodes it and scales it down (never up) to fit **300 × 450** (`COVER_UPLOAD_MAX_W/H`);
  3. draws it onto a canvas with a white background (so transparent PNGs don't turn black);
  4. re-encodes as JPEG at quality 0.85, stepping down through 0.7, 0.55 and 0.4 until the data URL is ≤ **200,000 characters** (`COVER_DATA_MAX`, ~150 KB), or refuses the image.
- The result is held in `editCoverData` with a preview and a size readout, and is written to the book only on Save. **Remove upload** clears it.
- Display priority everywhere `coverImg` is used: `coverData` → `cover` → text placeholder.
- `normalizeBook` keeps `coverData` only if it matches `COVER_DATA_RE` and is within the size cap, so imports can't bring in oversized or non-image data.
- **Storage cost:** a typical upload is 10–60 KB, and browsers give each origin roughly 5 MB of `localStorage`. Around a hundred uploaded covers can fill it, at which point `saveData` shows its "Couldn't save" error (§10.1 #7).

### 5.14 Book detail view
`openDetail(book)` builds a `<dl>` of the fields that have values. Which rows appear depends on status: Priority for tbr, Progress for reading, Rating for read. The rest are Ownership, Series, Format, Pages, ISBN, Started, Finished and Added (dates via `toLocaleDateString`), Flags, Shelves, Tags, and whether the cover is uploaded or from a URL. Audiobook length, Time listened and Time remaining appear for any book that has either audio field, so they stay visible on a book whose format was later changed away from audiobook. Below that come the DNF reason, the full notes (`white-space: pre-wrap`), and every quote with its page. Everything goes through `esc()`. Delegation: a click on any `[data-detail-id]` is handled first in the body click listener. Focus moves to **Close** when the dialog opens.

### 5.15 Ownership categories
- **`OWNERSHIP`** is the single list of categories, in the order they are offered, listed
  and sorted in: *Owned — physical*, *Owned — ebook*, *Owned — audiobook*, *Borrowed*,
  *Loaned out*, *Wishlist*. Adding one there is the only edit needed — the dialog's
  checkboxes and both filters are built from it at load.
- A book carries a **list**, not a single choice, because a real shelf does not work that
  way: the hardback and the audiobook can both be yours, and a copy you own can be lent
  out. `ownership` is stored in `OWNERSHIP` order however the boxes were ticked, so two
  books in the same categories always compare and read the same way.
- **Migration.** The older `owned` boolean said *whether* a book was owned but not in what
  form, so `normalizeOwnership` reads the format as the best evidence for that:
  `owned: true` becomes the matching owned category, `owned: false` becomes `["wishlist"]`,
  and a book with neither field stays uncategorised rather than being claimed as owned on
  no evidence. New books start with nothing ticked for the same reason.
- **Sorting** by category (Library) uses `ownershipRank`: the position of the first
  category a book is in, so a book that is owned-physical *and* loaned sorts with the
  owned ones. A book in no category sorts last rather than leading with a blank. Title
  breaks ties.
- **Filtering** (Library and TBR) shares one `matchesOwnership` and one option list built
  by `fillOwnershipFilter`: *All categories*, *Owned (any form)* — which covers all three
  owned categories — each category on its own, and *No category yet*.
- The **Wishlist badge** still shows on any TBR book in the wishlist category, and on a
  not-owned read book only where the Read tab asks for it (`renderRead` passes
  `{wishlistBadge:true}` to `bookCardHTML`). The other categories are deliberately not
  badged: the browsing views stay bare, and the categories are readable from the detail
  view, the Library's category sort, and the filters.
- The detail view lists every category a book is in, and omits the row entirely for a book
  in none.


### 5.16 Daily page log and pace
- Each card on the Logs tab's 📖 *Books* side has a pages box, a date box (defaults to today, can't be set to a future day) and **Log pages** (Enter anywhere in the row does the same). `addLogEntry` accepts 1–5000 pages. A second entry for the same day is added to the first.
- When `pages` is known, logging raises `progress` to logged ÷ pages (capped at 100%). It never lowers it, because a reader who starts logging partway through has read more than the log shows. Logging a day before `dateStarted` (or with none set) moves `dateStarted` back to that day.
- The card shows "Today: N pages · M days logged · Edit log" once anything is logged. **Edit log** opens the detail view, which lists the log newest first. Each day has a pages box with **Save** (Enter also saves) and **Remove**, for fixing typos. Saving 0 removes the day, and values outside 0–5000 are refused.
- Corrections go through `setLogEntry`. Normally `progress` only moves forward, but if it sits exactly where the log put it (logged ÷ pages, before the correction), it follows the corrected log up or down. That way a typo that pushed a book to 100% is undone. Progress set some other way, such as the slider, is left alone.
- **Stat tiles** (only once some book has a log entry), summed across all books: Pages Today; Pages / Day over the last 7 and 30 days (divided by every day in the window, not only reading days); Reading Streak (consecutive days with pages, counting back from today, or from yesterday if nothing is logged today yet).
- **Currently Reading Pace** table on the Pacing tab, one row per reading book, headed *Read / listened*, *Per day*, *Est. finish*. One code path for both units, via `LOG_KINDS` (§5.17): done = max(logged, progress% × total); per day = logged ÷ days from the first entry to today inclusive; est. finish = today + ⌈remaining ÷ per day⌉, or "Any page now" / "Any minute now" when nothing is left. The h/m in an audiobook's cells is what tells the two kinds of row apart. Either kind shows "—" without a log or a total, and both hints ask about whichever measure each book is counted in.
- Dates are stepped by calendar day (`daysAgoStr`), so a daylight-saving change can't skip or repeat a day.

### 5.17 The two logs
The Logs tab tracks the same thing twice over: an audiobook against its running time in
minutes, everything else against its page count in pages. Rather than write the cards,
tiles and chart once per unit and watch the two drift apart, everything that actually
differs is gathered in one table and the rest is written once against it.

- **`LOG_KINDS`** has two entries, `audio` and `pages`. Each supplies which books belong
  (`belongs`), where the log lives (`entries`) and what it is measured against (`total`),
  two formatters (`fmt` = "2h 30m" / "150 pages", `bare` = the same number where the unit
  is already established, as on the right of "2h 30m of 10h"), the wording for every label
  and button, and the chart's axis rules. `id` is "Audio" or "Pages", which is also how
  every element on the page is named: `log<Id>Cards`, `log<Id>StatGrid`, and so on, so
  `el(kind, "Cards")` finds the right one.
- Built on that: `logDone`, `logPace`, `syncLogProgress`, `logFiguresHTML`,
  `logTotalRowHTML`, `logEntryRowHTML`, `logSummaryHTML`, `logCardHTML`,
  `renderLogSection`, `logTotalsByDate` and `renderDailyLogChart` — each written once and
  called twice. The pace table on Pacing (§5.16) reads through the same table.
- The one thing that cannot be shared is the inner controls of the two editable rows: a
  time needs an hours and a minutes box, a page count needs one number box. Both rows emit
  the `data-*` attributes the existing click and keydown handlers already look for, so the
  handlers themselves never had to learn about the split.
- **Minutes and pages are the only stored units.** Hours exist only on screen
  (`formatMinutes`), which keeps every sum a plain addition.
- **Times are typed as an hours box plus a minutes box** (`hmInputsHTML` / `readHmPair` /
  `setHmPair`), used by the card, the detail log editor and the Add/Edit dialog alike.
  Two boxes mean there is no format to parse and none to get wrong — "1h 20", "1:20" and
  "80" cannot be confused because they cannot be typed. `readHmPair` returns `null` when
  both boxes are empty, so "left blank" is distinguishable from a deliberate zero. Each box
  and its unit are one `.hm-pair`, so "h" is never stranded on the next line in a narrow
  card; `#logAudioCards` and `#logPagesCards` use a 300px minimum column instead of 220px.
- **The total is editable from the card**: `saveAudioLength` and its paged twin
  `savePageCount`. Clearing the box (or zero) removes it, which is how one entered by
  mistake is taken off again. Out-of-range values are refused rather than clamped, so a
  mistyped total is never silently turned into a different one.
- **Logging** (`addLogEntry` / `addListenEntry`): several entries for a day add together,
  and logging a day before `dateStarted` moves the start back. **Progress** follows the log
  through `syncLogProgress` — listened or read ÷ total, capped at 100%, and only ever
  upward. `setLogEntry` / `setListenEntry` apply the one exception: a correction may pull
  progress back down, but only when it is sitting exactly where the log put it.
- **Corrections** live in the detail view, which shows a *Reading log* or *Listening log*
  section (or both): newest day first, each with its boxes, **Save** (Enter works) and
  **Remove**. Saving zero removes the day.
- **The figures line** shows done, remaining and a percentage. Done takes the higher of the
  log and the progress slider turned back into an amount — a slider moved by hand is
  evidence of reading the log never saw — and a book marked *Read* always reads 100%.
  Without a total the bar falls back to the slider and the line says so, so the bar is never
  showing a figure the text contradicts.
- **Only books in progress get a card** (`logInProgress`). The tab answers "what have I got
  on", so a finished, abandoned or not-yet-started book is not something left to track and
  does not take up space. Every card therefore has the same status, which is why there is no
  status badge, no status filter, and no *Start* or *Log Reread* action — only **Mark
  Finished** and **DNF**, the two ways out of the tab. Cards sort by title.
- **Nothing is lost when a book leaves.** Its log stays on the record: still counted in the
  chart below, still in the Pacing figures, and still editable from the book's detail view,
  reachable from Library, Read, DNF or anywhere else the book appears.
- **Stat tiles** sum everything in progress, never the filtered list — they would be
  confusing if a search changed them: In Progress, total, done, remaining (only over books
  that have a total), and per day over the last 7 days.
- **The chart is the exception**: it covers every book of its kind, finished ones included,
  because a day's reading does not stop having happened when the book ends. Marking a book
  finished must not retroactively flatten last week's chart. Its picker lists them all to
  match.
- Audiobook and paged figures never mix. A book changing format simply moves from one
  sub-tab to the other; both its fields are kept, so nothing is lost either way (§3.4).

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
- Book titles on cards and Library items are real buttons, so the detail view can be opened from the keyboard.
- Reordering doesn't require drag-and-drop: ↑/↓ buttons (with `aria-label`s) work by touch and keyboard, and keep focus between presses.
- Cover images use `alt=""`. The title is shown next to them, or used as placeholder text.
- Tabs are plain buttons without ARIA tab roles. Modals don't trap focus and don't close on Escape.
- Form labels in the book dialog are not linked to inputs with `for`. The profile dialog's "Add a profile" label is.

---

## 9. Security & Privacy

- **Data locality:** libraries never leave the browser. Outbound requests carry only ISBNs, titles, and authors to the public APIs above.
- **Output escaping:** `esc()` is applied to user and API text inserted via `innerHTML`, including attribute values such as cover `src`, `data-title`, and every book/challenge id in `data-*` attributes. The chart tooltip is filled with `textContent`, so decoded `data-tip` text can never become markup.
- **JSONP:** the iTunes response runs as a script with page privileges. Apple is implicitly trusted.
- **Uploaded covers:** stored only as `data:image/(png|jpeg|gif|webp);base64,…` URLs of bounded size (checked on upload and in `normalizeBook`), so a `coverData` value can't carry a script URL or arbitrary markup into an `<img src>`.
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
| 13 | Saving a book from the Edit dialog rebuilt it from blank fields, so its `sortIndex` (TBR position) and `topReadRank` (Top 5 position) were lost on every edit | Save merges the form over the existing record and runs `normalizeBook` (§4.4) |

### 10.2 Usability changes

Made from `usability-fixes.md` and checked in headless Edge (49 checks).

| # | Friction | Change |
|---|---|---|
| U1 | Search existed only on TBR and Read | Search on Currently Reading, DNF Pile and Library too. All tabs use `matchesSearch`, and empty states say when a search or filter is hiding everything. |
| U2 | Selection mode could only delete or fix covers | Bulk add tags/shelves and bulk status change through `normalizeBook` (§5.6) |
| U3 | Book deletes were permanent | 8-second Undo after single and bulk book deletes (§5.8). Challenge/profile deletes stay confirm-only. |
| U4 | Series tab was alphabetical only | Sort by closest to complete, or Next Up available first (§5.10) |
| U5 | No sign a backup was overdue | Per-profile last-backup tracking and a header reminder (§5.9) |
| U6 | Reordering needed drag-and-drop (unusable on touch) | ↑/↓ buttons on TBR tier cards and Top 5 rows, sharing the drag's ordering code (§5.11) |

### 10.3 New features

Made from `new-features.md` and checked in headless Edge (51 checks, plus the earlier suites re-run).

| # | Feature | Summary |
|---|---|---|
| F1 | Numeric reading goal | Per-profile, per-year target with a progress tile on Stats & Pace (§5.12) |
| F2 | Manual cover upload | Shrunk, size-capped JPEG data URL in `coverData`; wins over the URL (§5.13) |
| F3 | Book detail view | Read-only dialog from a cover or title click, with full notes and quotes (§5.14) |
| F4 | Owned vs. wishlist | `owned` field, dialog select for TBR, Wishlist badge and TBR filter (§5.15) |
| F5 | Daily page log | Pages read per day on Currently Reading books, feeding pace tiles, a per-book pace/finish-date table and a daily chart on Stats & Pace (§5.16) |
| F6 | Audiobooks tab | Every book flagged *Audiobook*, with an editable running time, a minutes-per-day listening log, time listened/remaining per book, collection tiles and a daily line chart (§5.17) |
| F7 | Audiobooks counted in time throughout | Currently Reading logs minutes for an audiobook instead of pages, the pace table reports either unit, and the Add/Edit dialog swaps the page box for the running time (§5.17, §4.4) |
| F8 | Tabs reorganised into eight | Library first; Currently Reading and Audiobooks merged into a two-sided **Logs** tab generated from one description of the two units; Stats & Pace narrowed to **Pacing** with its library charts moved to Rankings; Discovery folded into TBR (§4.3, §5.17) |
| F9 | Logs scoped to books in progress | The Logs tab lists only `reading` books, dropping the status filter, the status badge and the start/reread actions; finished logs stay in the chart, the Pacing figures and the detail view (§5.17) |
| F10 | Ownership categories | `ownership` list replacing the `owned` boolean: owned physical/ebook/audiobook, borrowed, loaned out, wishlist, several at once, with a category sort and filter on Library and a category filter on TBR (§5.15) |

### 10.4 Still open

1. **Full re-render on every change.** Every mutation and tab switch rebuilds every view, both sides of the Logs tab included. This is simple and correct but scales linearly with library size. It's a deliberate trade-off rather than a bug.
2. **Legacy data key** `bookTrackerData_v1` is left in place after migration, on purpose. Deleting it would break an older copy of the file opened in the same browser.
3. **Touch devices.** HTML5 drag-and-drop still doesn't work on most touch devices (use the ↑/↓ buttons instead), and chart tooltips need a mouse (every chart has a table view).
4. **Drag position in grids.** The drop position is based only on vertical position, so ordering within a single row of a multi-column card grid is approximate.
5. **Comma-separated labels.** Challenge goals and bingo squares can't contain commas.
6. **Uploaded covers use shared storage.** Each upload counts against the ~5 MB `localStorage` quota shared by every profile in the browser (§5.13).
7. **Dialogs and keyboard.** Dialogs, including the new detail view, don't close on Escape or trap focus (§8.4).
