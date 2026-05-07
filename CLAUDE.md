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

**What it does:** Users upload a book or document, enter one or more words to track (e.g. character names, plot elements), and the app plots a smooth frequency curve for each word across the timeline of the text. Chapter markers are overlaid on the chart for navigation. Additional views and tools help users explore co-occurrence and export results.

**Primary use case:** Showcase application for vibe coding demos. Must be impressive, immediately usable, and require zero setup from the audience.

---

## Tech Stack — Pinned Versions (Do Not Change)

| Library | Version | Source |
|---|---|---|
| Chart.js | 4.4.3 | jsDelivr (cdnjs blocked by ORB due to `Chart.js` path) |
| chartjs-plugin-annotation | 3.0.1 | jsDelivr |
| PDF.js | 3.11.174 | cdnjs |

Load all libraries from CDN in the `<head>`. Do not use npm, bundlers, or local files.
Always verify that chartjs-plugin-annotation is compatible with the pinned Chart.js version before writing chart code.

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
  rawText: '',          // full extracted text string
  words: [],            // array of { word, color, visible }
  windows: [],          // array of { offset, counts: { word: n } }
  chapters: [],         // array of { title, wordOffset }
  windowSize: 500,      // words per window (user-adjustable)
  stepSize: 100,        // words per step (user-adjustable)
  chart: null,          // Chart.js instance
  isAnimating: false,   // animation play state
  animationSpeed: 1,    // 1=slow, 2=medium, 3=fast
};
```

---

## Build Phases

Complete and test each phase before starting the next. Do not skip ahead.

### Phase 1 — UI Shell & File Upload
- Dark-themed layout with header, upload zone, word input, and chart placeholder
- Drag-and-drop + click-to-browse file upload
- Accept `.txt` and `.pdf` only
- Extract full text from .txt via FileReader
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
- Add window size and step size sliders; re-run analysis on change without re-parsing text
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
- Display detected chapter list as a scrollable panel below the chart

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
- Two tabs above the chart: "Frequency" and "Heatmap"
- Heatmap tab renders a square matrix of all tracked word pairs
- Cell color intensity = number of windows where both words appear together (co-occurrence count)
- Use a single-hue gradient (e.g. dark navy → vivid accent color) for cell intensity
- Show numeric co-occurrence count inside each cell if space allows
- Axes labelled with word names; diagonal cells (word vs itself) shown in neutral gray
- Heatmap re-renders automatically when words are added/removed

### Phase 7 — Export & Shareable URL
- "Export PNG" button: uses `chart.toBase64Image()` to download the active view (frequency chart or heatmap canvas) as `bookworm-export.png`
- "Copy Link" button: encodes the current word list, window size, step size as URL query parameters (`?words=Darcy,Elizabeth&window=500&step=100`)
- On page load: parse URL parameters and pre-populate word input and sliders if present
- Show a toast notification ("Link copied!") on successful copy
- Shareable URL does not include book text — user must re-upload

---

## Design System

**Theme:** Dark. Not generic dark — refined, editorial dark. Think data journalism meets literary analysis.

**Colors (CSS variables):**
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
- Grid lines: `rgba(255,255,255,0.05)`
- Axis labels: `var(--text-secondary)`
- Tooltip: dark card with colored word label and frequency value
- Chapter annotation lines: dashed, `rgba(255,255,255,0.25)`, label in `var(--text-secondary)`

---

## Known Pitfalls — Read Before Writing Code

1. **PDF.js async:** `getTextContent()` is async per page. Always await all pages before concatenating. Never assume text is available synchronously after PDF load.

2. **chartjs-plugin-annotation registration:** Must be explicitly registered with `Chart.register(ChartAnnotation)` before any chart is created. Forgetting this causes silent annotation failure.

3. **Large texts and performance:** Books can be 200k+ words. Run the sliding window analysis in a `setTimeout` or `requestAnimationFrame` loop to avoid blocking the UI thread. Show progress during this.

4. **Chapter regex edge cases:** Roman numerals are ambiguous (e.g. "I" matches too broadly). Anchor patterns to line starts: `/^(chapter|part|book)\s+/im`. Test with both "Chapter 1" and "CHAPTER ONE".

5. **Word matching accuracy:** Use word-boundary regex (`\b`) for matching, not simple `includes()`. "art" should not match inside "eart" or "артефакт".

6. **Heatmap canvas:** The heatmap is a custom `<canvas>` drawn with the 2D API, not a Chart.js chart. Keep it separate from the main chart instance.

7. **Export on mobile:** `chart.toBase64Image()` may return a blank image if the canvas was not rendered with `devicePixelRatio` set correctly. Always set `devicePixelRatio: window.devicePixelRatio` in Chart.js options.

8. **URL length limits:** Keep shareable URLs short. Encode word list as comma-separated plain text, not JSON. Do not attempt to encode book text in the URL.

---

## File Structure

```
bookworm/
├── CLAUDE.md          ← this file
├── index.html         ← the entire application
└── test/
    ├── pride.txt      ← Pride and Prejudice (from Project Gutenberg)
    └── moby.txt       ← Moby Dick (from Project Gutenberg)
```

Keep test files in the `test/` folder. They are not part of the deliverable.

---

## Tools Available in This Session
- Playwright MCP: use after each phase to screenshot and verify rendering
- GitHub MCP: use to commit after each passing phase
- feature-dev Plugin: use to develop features
- code-review Plugin: use to review code 
- code-simplifier Plugin: use to simplify code
- context7 Plugin
- frontend-design Plugin: use to design frontend     

---

## Testing Checklist (Run After Each Phase)

- [ ] .txt file uploads and text is extracted correctly
- [ ] .pdf file uploads and all pages are extracted (test with a multi-page PDF)
- [ ] Word input accepts comma-separated words, assigns colors correctly
- [ ] Frequency chart renders with correct curves
- [ ] Adding/removing words updates chart without re-parsing
- [ ] Window and step sliders recalculate analysis correctly
- [ ] Chapter markers appear at correct positions
- [ ] Clicking a chart point opens the reading panel with correct excerpt
- [ ] Tracked words are highlighted correctly in the excerpt
- [ ] Animation plays, pauses, and resets correctly
- [ ] Heatmap renders correct co-occurrence values
- [ ] PNG export downloads a non-blank image
- [ ] Shareable URL encodes and decodes word list correctly

---

## Session Discipline

- **Always read this file at the start of a new session.**
- **Complete and test one phase before starting the next.**
- **Never change pinned CDN versions.**
- **Never introduce external files or a build step.**
- **After each phase: commit to git with a message like `feat: phase 2 — sliding window + chart`**
- **If something doesn't work after two attempts: stop, explain the problem, and ask the user how to proceed.**
- **Update this file when a significant decision is made or a pitfall is discovered.**
