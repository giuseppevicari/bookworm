# CLAUDE.md — Bookworm Project

## What This File Is
This is your persistent project memory. Read it at the start of every session.
Never deviate from the decisions recorded here unless explicitly told to by the user.
Update this file when new decisions are made.

---

## Project Overview

**Name:** Bookworm
**Tagline:** Narrative Frequency Visualizer
**Deliverable:** A single self-contained `index.html` file. No build step. No backend. No external files. Runs in any modern browser by double-clicking.

**What it does:** Users upload a book or document, enter one or more words to track (e.g. character names, plot elements), and the app plots a smooth frequency curve for each word across the timeline of the text. Chapter markers are overlaid on the chart for navigation. A semantic network view and export tools are also available.

**Primary use case:** Showcase application for vibe coding demos. Must be impressive, immediately usable, and require zero setup from the audience.

---

## Tech Stack — Pinned Versions (Do Not Change)

| Library | Version | Source |
|---|---|---|
| Chart.js | 4.4.3 | jsDelivr (cdnjs blocked by ORB due to `Chart.js` path) |
| chartjs-plugin-annotation | 3.0.1 | jsDelivr |
| PDF.js | 3.11.174 | cdnjs |
| D3.js | 7 (latest) | jsDelivr (`d3@7/dist/d3.min.js`) |

Load all libraries from CDN in the `<head>`. Do not use npm, bundlers, or local files.
Always verify that chartjs-plugin-annotation is compatible with the pinned Chart.js version before writing chart code.

---

## Jira
Always use Jira project key `BW` unless I specify otherwise.

---

## Architecture

Everything lives in `index.html`. Organise the code in this order inside the file:

1. `<head>` — CDN script/style tags, CSS variables, global styles
2. `<body>` — HTML structure only, no inline styles
3. `<script>` block at end of body, divided into clearly commented sections:
   - **State** — single global state object
   - **Parser** — file reading, PDF extraction, text normalisation
   - **Analyser** — sliding window, chapter detection, co-occurrence matrix
   - **Chart** — Chart.js initialisation, rendering, annotation layer
   - **UI** — event listeners, panel logic, tab switching, animation loop
   - **Export** — PNG export, URL encoding/decoding

Keep functions small and single-purpose. Use descriptive names. Add a one-line comment above every function explaining what it does.

---

## Global State Object

Maintain a single `state` object. Never use scattered global variables. Shape:

```js
const state = {
  rawText: '',            // full extracted text string
  words: [],              // array of { word, color, visible }
  windows: [],            // array of { offset, counts: { normForMatch(word): n } }
  chapters: [],           // array of { title, wordOffset, windowIdx }
  wordCounts: {},         // total occurrence count per word; keyed by normForMatch(w.word)
  title: '',              // extracted book title — set by showFileInfo(); not rendered on screen
  wordCharMap: [],        // char offsets of every whitespace-delimited word (for excerpt lookup)
  showChapterLabels: true,// toggle for chapter label visibility on chart
  windowSize: 500,        // words per window (user-adjustable)
  stepSize: 100,          // words per step (fixed — slider removed in BW-14)
  exactMatch: true,       // true = \b word boundary; false = substring match
  chart: null,            // Chart.js instance
  activeTab: 'frequency', // 'frequency' | 'network'
  zoomLevel: 1,           // x-axis zoom factor (1 = full view)
  zoomOffset: 0,          // pan position as % (0–100); used when zoomLevel > 1
};
```

`chapters[n].windowIdx` is computed once in `runAnalysis` after `state.windows` is built, and read by both `chapterLabelPlugin.afterDraw` and `buildChapterAnnotations`. Never recompute it inside `afterDraw` — that runs on every render frame.

---

## Build Phases

Complete and test each phase before starting the next. Do not skip ahead.

### Phase 1 — UI Shell & File Upload
- Dark-themed layout with header, upload zone, word input, and chart placeholder
- Drag-and-drop + click-to-browse file upload
- Accept `.txt` and `.pdf` only
- Extract full text from .txt via `parseTxt()`: read as `ArrayBuffer`, attempt strict UTF-8 decode (`TextDecoder('utf-8', { fatal: true })`), fall back to `windows-1252` if that throws — covers the vast majority of real-world `.txt` files
- Extract full text from .pdf via PDF.js (handle async page-by-page extraction)
- Show loading spinner/progress bar during extraction
- Store extracted text in `state.rawText`
- On success: show word count and file name; enable word input

### Phase 2 — Sliding Window Analysis & Frequency Chart
- Parse comma-separated word input; assign a distinct color to each word
- Run sliding window across `state.rawText` using `state.windowSize` and `state.stepSize`
- Store results in `state.windows`
- Render Chart.js line chart: x-axis = window index, y-axis = frequency count
- Each word = one dataset with its assigned color
- Smooth curves (tension: 0.4)
- Add a window size slider; re-run analysis on change without re-parsing text (step size is fixed at 100 — no slider)
- Words can be added/removed/toggled without re-parsing text

### Phase 3 — Chapter Detection & Overlay
- Scan `state.rawText` for headings matching these patterns (case-insensitive):
  - `Chapter [number]` (digits or written: one, two … twenty)
  - `Chapter [Roman numeral]`
  - `CHAPTER [anything]`
  - `Part [number or Roman]`
  - `Book [number or Roman]`
  - `Prologue`, `Epilogue`, `Afterword`, `Introduction`
- Record word offset of each match; store in `state.chapters`
- If zero chapters detected: silently skip overlay, show notice "No chapters detected"
- If chapters found: render vertical dashed annotation lines on chart using chartjs-plugin-annotation
- Show chapter title as rotated label at top of each line
- Display detected chapter list as a scrollable "04 — Chapter Navigator" panel below the chart; clicking an item calls `zoomToChapter(i)` to zoom the chart to that section

### Phase 4 — Reading Mode Side Panel
- Clicking any data point on the chart opens a side panel (slides in from right on desktop; overlays on mobile)
- Panel shows the raw text excerpt for that window (~500 words centred on the clicked position)
- All tracked words are highlighted in their chart colors within the excerpt (use `<mark>` styled with the word's color)
- Panel has a visible close (×) button
- Panel does not obscure the chart on screens wider than 1200px (use side-by-side layout)
- Panel title shows the window position and chapter name if within a chapter

### Phase 5 — Chart Animation ("Replay the Story")
- Play/Pause button in the toolbar
- Animation progressively reveals the chart from left to right (increase visible data points on each frame)
- Speed slider: Slow (~30s for full book), Medium (~15s), Fast (~5s)
- Animation is pausable and resumable mid-way
- Loops back to start when complete if loop toggle is on
- During animation, a vertical "playhead" line moves across the chart
- Pressing Play resets to start if already at end

### Phase 6 — Relationship Heatmap Tab
~~Removed. The heatmap tab has been deleted from the app.~~

### Phase 7 — Export & Shareable URL
- "Export PNG" button behaviour depends on the active tab:
  - **Frequency tab:** composites the chart canvas with a legend row (colored dot + word label + total occurrence count, e.g. `Frodo (142)`) onto an offscreen canvas; downloads as `bookworm-export.png`. Canvas background filled via the `bgFill` `beforeDraw` plugin registered inline on the Chart instance.
  - **Network tab:** calls `exportNetworkPng()` — clones the D3 SVG, resolves all inline CSS variables to literal values (they don't resolve in a blob context), draws onto a canvas with `devicePixelRatio` scaling, downloads as `bookworm-network.png`.
- ~~"Copy Link" button~~ — removed. The `copyShareLink` function and URL param encoding still exist in code but are not exposed in the UI.
- On page load: parse URL parameters (`?words=Darcy,Elizabeth&window=500&exact=1`) and pre-populate word input and window slider if present
- Shareable URL does not include book text — user must re-upload

### Post-Phase Additions

**Word count badges:** After analysis runs, total occurrence counts appear in two places: (1) each word chip in the input section (`.chip-count` badge, styled monospace); (2) each clickable legend pill under the chart (`.legend-count` badge). Both read from `state.wordCounts` keyed by `normForMatch(w.word)`. The PNG export legend also includes the count (format `word (142)`). `renderLegend()` is called at the end of `renderChart()` and `updateChart()`, so counts appear as soon as the chart renders.

**Exact match toggle:** Checkbox in the "02 — Track Words" card, positioned directly below the word input field. When on (default), uses `\b` word-boundary regex so "book" won't match "bookworm". When off, uses substring matching. Re-runs analysis on toggle if results already exist.

**Word stats bar:** A bar rendered above the frequency chart (below the animation toolbar) after analysis runs. Shows one pill per tracked word with: color swatch, word label, total count, and percentage of total words. Hidden until analysis completes. Element ID: `wordStatsBar`, CSS class: `word-stats-bar` / `word-stats-bar visible`.

**Chapter labels on chart (canvas plugin):** Chapter labels are rendered above the chart area using a custom Chart.js `afterDraw` plugin (`chapterLabelPlugin`), NOT via chartjs-plugin-annotation labels. The plugin draws rotated text directly onto the canvas above `chartArea.top`. Chart options must include `layout.padding.top: 90` to reserve space for the labels. A toggle checkbox (`#chapterLabelsToggle`) sets `state.showChapterLabels` and calls `chart.update('none')`. See pitfall #9.

**Zoom and pan controls (BW-8):** A `#zoomControls` div lives inside `#frequencyView` immediately below `#chartWrapper`. It contains a Zoom slider (`#zoomSlider`, 1×–20×) and a Position pan slider (`#panRow` / `#panSlider`, 0–100%). `applyZoom()` sets `chart.options.scales.x.min/max` and calls `chart.update('none')`. The pan row is hidden (`display:none`) when zoom is 1×. `resetZoom()` restores both sliders to defaults and clears `min/max`. `renderChart()` calls `resetZoom()` on every new analysis so the view always starts full-width.

**Semantic Network tab (BW-6):** A second tab `#tabNetwork` alongside Frequency. Renders a D3 v7 force-directed SVG graph in `#networkSvg`. The `buildCoOccurrences(term, contextRadius=10)` function scans all positions of `term` in the tokenized text and counts words within ±10 tokens, excluding stop words (constant `STOP_WORDS` Set) and the search term itself. Returns the top 20 by count. `renderNetwork(term, termCount, related)` builds the D3 simulation: central node (radius 22 px), related nodes sized by `networkNodeRadius()`, edge width by co-occurrence strength. Fixed-position tooltip appended to `<body>` on mouseenter, removed on re-render. Clicking a related node calls `generateNetwork(word)` to recenter. Export PNG on the Network tab calls `exportNetworkPng()`: clones the SVG, resolves all inline CSS variables (`var(--text-primary)` etc.) to literal hex values via a `varMap` lookup, serializes to a Blob URL, draws onto a canvas with `devicePixelRatio` scaling, then downloads as `bookworm-network.png`. **CSS vars must be replaced before serialization** — they do not resolve inside a blob context. The search input placeholder is "Enter a word…" — phrase search is not supported (BW-12). Node labels use `style('fill', ...)` with CSS variables so they adapt to the active theme in the live view.

**Sample text (BW-17):** Alice's Adventures in Wonderland is embedded as a JS template literal constant `SAMPLE_TEXT` (with metadata in `SAMPLE_TEXT_META`). A "or try with sample text" link (`#sampleTextBtn` / `.link-btn`) sits below the upload zone badges inside the upload card, inside `#sampleTextRow`. Clicking it calls `loadSampleText()`, which populates `state.rawText`, runs `detectChapters` and `buildWordCharMap`, then calls `showFileInfo("Sample Text: Alice's Adventures in Wonderland", wordCount)`. `showFileInfo` hides `#sampleTextRow`; the "Change" button handler restores it. If `SAMPLE_TEXT` is empty, a friendly error is shown. The sample text flows through the same analysis pipeline as uploaded files.

**In-app tutorial (BW-15):** A welcome modal (`#tutorialOverlay` / `.tutorial-overlay`) appears on first visit, gated by `localStorage` key `bw-tutorial-seen`. Shows 3 numbered step cards: Load a text → Enter words → Explore the chart. Two CTAs: "Try Sample Text" (`#tutorialTrySample`) calls `loadSampleText()` then hides the modal; "Upload My Own" (`#tutorialUploadOwn`) programmatically clicks `#fileInput` then hides the modal. Close button (`#tutorialClose`) and clicking the backdrop also dismiss it. A "How it works" pill button (`#tutorialBtn`) in `.header-actions` (alongside `#themeToggle`) re-opens the modal at any time. Both header buttons are wrapped in `.header-actions` (flex row, `margin-left: auto`) — the old `margin-left: auto` that was on `.theme-toggle` has been moved to `.header-actions`.

**Multi-word phrase search (BW-25):** The frequency analyser supports phrases like "white rabbit" or "red queen" in addition to single words. In `computeWindows` and `runAnalysis`, words are pre-split into `singles` (no space) and `phrases` (contains space). Singles use Set lookup; phrases are pre-tokenized once via `tokenize(w.word)` and matched by scanning consecutive token sequences. All counts maps are keyed by `normForMatch(w.word)` so accented and plain spellings resolve to the same bucket. The Network tab search input does not support phrases (single word only).

**`normForMatch()` helper and Unicode normalization pipeline:** A `normForMatch(s)` function in the ANALYSER section strips combining diacritical marks (U+0300–U+036F) and lowercases, producing a canonical ASCII-safe key for all comparison and counting operations. It is used in: `tokenize()` (to normalize text tokens before storing), all counts map keys (`computeWindows`, `runAnalysis`, chart datasets, `renderChips`, `renderWordStats`), and `highlightExcerpt()` (to match accented text with accented or plain search terms). The original `w.word` string is **never modified** — it is preserved for display only. This separation (normalize at comparison time, preserve at display time) is the correct pattern for accent-insensitive search.

**Light/dark mode toggle (BW-13):** A pill button in the header (`#themeToggle`) switches between dark (default) and light mode. Theme is applied by setting `data-theme="light"` on `<html>`; dark mode has no attribute. Light mode variables are declared under `[data-theme="light"]` in the CSS. Preference is persisted in `localStorage` under key `bw-theme`. `applyTheme(theme)` sets the attribute, updates the button icon/label, and calls `renderChart()` if a chart exists. Chart colours (grid lines, tick labels, tooltip) are resolved at render time by `getThemeColors()`, which reads `document.documentElement.dataset.theme`. The `chapterLabelPlugin.afterDraw` also reads the theme to pick label colour. Light mode variables:
```css
--bg-primary: #f5f4f8; --bg-secondary: #eeecf6; --bg-card: #ffffff;
--border: #d4d0ea; --text-primary: #1a1825; --text-secondary: #6b6785;
--success: #16a34a; --danger: #dc2626;
```

---

## Design System

**Theme:** Dark by default (refined, editorial dark — data journalism meets literary analysis), with a user-toggleable light mode. See the Light/dark mode toggle entry in Post-Phase Additions for full details.

**Dark mode colors (CSS variables on `:root`):**
```css
--bg-primary: #0e0e12;
--bg-secondary: #16161d;
--bg-card: #1c1c26;
--border: #2a2a38;
--text-primary: #e8e6f0;
--text-secondary: #8884a0;
--accent: #7c6af7;
--accent-hover: #9d8fff;
--success: #4ade80;
--danger: #f87171;
```

Accent and font variables are shared between themes and are not overridden in light mode.

**Word colors** (assign in this order, cycling if more than 8 words):
```
#7c6af7, #f472b6, #34d399, #fb923c, #60a5fa, #facc15, #a78bfa, #f43f5e
```

**Typography:**
- Headings: `'DM Serif Display'` from Google Fonts
- Body/UI: `'DM Sans'` from Google Fonts
- Monospace (text excerpts): `'JetBrains Mono'` from Google Fonts

**Component rules:**
- All cards: `border-radius: 12px`, `border: 1px solid var(--border)`
- Buttons: pill-shaped (`border-radius: 999px`), smooth hover transitions (150ms)
- Inputs/sliders: styled to match dark theme, never use browser defaults
- Scrollbars: thin, dark, styled with `scrollbar-width: thin`
- Loading states: animated gradient shimmer, not a spinning circle

**Chart style:**
- Background: `var(--bg-card)`
- Grid lines, tick colours, and tooltip colours are resolved at render time by `getThemeColors()` — never hardcode these values
- Chart title: not rendered. `state.title` is extracted (TXT: "Title: …" header; PDF: metadata; fallback: cleaned filename) and stored but not displayed anywhere.
- Tooltip: card styled to match the active theme, with colored word label and frequency value
- Chapter annotation lines: dashed, `rgba(255,255,255,0.25)`, label colour picked in `chapterLabelPlugin.afterDraw` based on theme

---

## Known Pitfalls — Read Before Writing Code

1. **PDF.js async:** `getTextContent()` is async per page. Always await all pages before concatenating. Never assume text is available synchronously after PDF load.

2. **chartjs-plugin-annotation registration:** Must be explicitly registered with `Chart.register(ChartAnnotation)` before any chart is created. Forgetting this causes silent annotation failure.

3. **Large texts and performance:** Books can be 200k+ words. Run the sliding window analysis in a `setTimeout` or `requestAnimationFrame` loop to avoid blocking the UI thread. Show progress during this.

4. **Chapter regex edge cases:** Roman numerals are ambiguous (e.g. "I" matches too broadly). Anchor patterns to line starts: `/^(chapter|part|book)\s+/im`. Test with both "Chapter 1" and "CHAPTER ONE".

5. **ROMAN regex can match empty string via backtracking (critical):** A ROMAN pattern with all-optional groups (`M{0,4}`, `C{0,3}`, `X{0,3}`, `I{0,3}`) can match an empty string. When the `(?=[MDCLXVI])` lookahead passes (e.g. for "muffled" because `m` is in the `i`-flag set), the engine consumes one char, then `\b` fails mid-word, and backtracks to a zero-length match. The `\b` then fires at the preceding space→word boundary, so the match succeeds as an empty string. Fix: add `(?!\w)` immediately after the number alternation group `(?:[0-9]+|${ROMAN}|${WRITTEN})`. This rejects the empty match when the next character is a word character. **Do NOT add `\b` inside the ROMAN string itself** — the outer `(?!\w)` already covers this, and a redundant inner `\b` creates inconsistency.

6. **Chapter deduplication threshold must be small:** The deduplication pass (removing matches within N chars of each other) guards against the same heading being caught by two patterns. Since all three patterns match different keywords, true duplicates are impossible — the threshold only needs to cover off-by-one positions. Use `> 5` chars, NOT `> 50`. A 50-char threshold silently drops a Chapter heading that appears within 50 chars of a Part heading (e.g. "PART II.\n_Subtitle._\n\nCHAPTER I." is only ~43 chars between the two).

7. **Word matching accuracy:** Use word-boundary regex (`\b`) for matching, not simple `includes()`. "art" should not match inside "eart" or "артефакт".

8. **Export on mobile:** `chart.toBase64Image()` may return a blank image if the canvas was not rendered with `devicePixelRatio` set correctly. Always set `devicePixelRatio: window.devicePixelRatio` in Chart.js options.

9. **Canvas rotation math for vertical text:** After `ctx.rotate(-Math.PI/2)`, the axes are remapped: `+x` in the rotated frame moves **upward** on the canvas, `+y` moves **left**. To draw text above `chartArea.top`, `ctx.translate(xPx, chartArea.top - 2)` then `ctx.fillText(label, positiveOffset, 0)` draws the label upward from that anchor. Using a negative x offset (or `textAlign: 'right'`) draws it downward into the chart area — a common mistake.

10. **URL length limits:** Keep shareable URLs short. Encode word list as comma-separated plain text, not JSON. Do not attempt to encode book text in the URL.

11. **`afterDraw` runs on every chart event — keep it O(chapters) not O(chapters×windows):** The `chapterLabelPlugin.afterDraw` hook fires on every tooltip hover, resize, and animation frame. Any inner loop over `state.windows` inside it is a hot-path O(n×m) scan. Always pre-compute chapter-to-window mappings (stored as `ch.windowIdx`) in `runAnalysis` once, after `state.windows` is populated, and just read the cached value in `afterDraw`.

12. **Export PNG background defaults to black without a fill plugin:** `chart.toBase64Image()` composites the canvas onto a transparent PNG, which renders black. Always keep the `bgFill` `beforeDraw` plugin registered inline on the Chart instance so it fills the correct theme color before every draw. If `renderChart()` is rewritten, do not remove this plugin.

13. **Text file encoding — never use `FileReader.readAsText`:** Many real-world `.txt` files (especially Windows-created ones) are encoded in Windows-1252, not UTF-8. `FileReader.readAsText(file, 'UTF-8')` silently garbles non-ASCII bytes (e.g. `û` U+00FB → replacement chars or wrong glyphs) rather than throwing an error, making the bug invisible. Always read as `ArrayBuffer`, then attempt `new TextDecoder('utf-8', { fatal: true }).decode(bytes)`, and fall back to `new TextDecoder('windows-1252').decode(bytes)` if that throws. This covers UTF-8, UTF-8 BOM, and Windows-1252 without user intervention.

14. **Unicode normalization form mismatches in JS object key lookups:** JavaScript strings that look identical visually may be stored in different Unicode normalization forms (NFC vs NFD) depending on the input source (keyboard, clipboard, file). Using `w.word` directly as a JS object key causes silent lookup failures: `win.counts[w.word]` returns `undefined`, and `undefined || 0` silently reports zero matches. Fix: always normalize to a canonical form with `normForMatch(w.word)` before using as a key. Use this canonical key everywhere counts are written or read; use `w.word` only for display.

15. **Diacritics in search terms strip silently if normalized too early:** If you call `normalize('NFD').replace(/combining/g, '')` inside `addWord()`, the chip label permanently loses its accent (e.g. "Nazgûl" becomes "Nazgul"). Normalization must happen at comparison time, not at storage time. Store `w.word` as typed; apply `normForMatch()` only when building regex patterns, count keys, or token comparisons. For `highlightExcerpt()`, NFD-normalize the excerpt text and build the regex with `[\\u0300-\\u036f]*` interleaved between each letter so accented characters in the source text are matched by plain search terms and vice versa.

16. **Multi-word phrase search requires consecutive token sequence matching:** Phrases cannot be matched by a single Set lookup against tokenized text. Detect phrases by `w.word.includes(' ')`, pre-tokenize them once with `tokenize(w.word)`, then scan the token array for consecutive matches. Handle the window boundary check (`j + pt.length <= start + windowSize`) to avoid counting phrases that span across window edges. Single words and phrases must be processed separately in both `computeWindows` and `runAnalysis`.

---

## File Structure

```
bookworm/
├── CLAUDE.md          ← this file
├── index.html         ← the entire application
├── .github/
│   └── workflows/
│       ├── claude.yml                  ← Claude PR Assistant
│       └── claude-code-review.yml      ← Claude Code Review on PRs
└── test/
    ├── alice.txt            ← Alice's Adventures in Wonderland (from Project Gutenberg)
    ├── nazgul_test.txt      ← UTF-8 test file with mixed Nazgûl/nazgul spellings
    └── nazgul_latin1.txt    ← Windows-1252 encoded test file with "Nazgûl" (byte 0xFB)
```

Keep test files in the `test/` folder. They are not part of the deliverable.

---

## Tools Available
- **Atlassian MCP** (`mcp__atlassian__*`): read/transition Jira issues in project `BW`
- **GitHub MCP** (`mcp__github__*`): create PRs, read issues
- **Playwright** (via `npx playwright` / temp node script): browser-based verification and screenshots
- **`/verify` skill**: run the app and screenshot to confirm a change works before reporting done

---

## Testing Checklist (Run After Each Phase)

- [ ] .txt file uploads and text is extracted correctly
- [ ] .pdf file uploads and all pages are extracted (test with a multi-page PDF)
- [ ] Word input accepts comma-separated words, assigns colors correctly
- [ ] Frequency chart renders with correct curves
- [ ] Adding/removing words updates chart without re-parsing
- [ ] Window size slider recalculates analysis correctly
- [ ] Chapter markers appear at correct positions
- [ ] Clicking a chart point opens the reading panel with correct excerpt
- [ ] Tracked words are highlighted correctly in the excerpt
- [ ] Animation plays, pauses, and resets correctly
- [ ] Frequency PNG export downloads with correct theme background and legend row (word + count)
- [ ] Network PNG export downloads correctly with resolved colours (no black or empty canvas)
- [ ] URL params (`?words=…&window=…`) pre-populate the UI on load

---

## Session Discipline

- **Always read this file at the start of a new session.**
- **Complete and test one phase before starting the next.**
- **Never change pinned CDN versions.**
- **Never introduce external files or a build step.**
- **After each phase: commit to git with a message like `feat: phase 2 — sliding window + chart`**
- **If something doesn't work after two attempts: stop, explain the problem, and ask the user how to proceed.**
- **Update this file when a significant decision is made or a pitfall is discovered.**
