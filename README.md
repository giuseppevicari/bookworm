# Bookworm — Narrative Frequency Visualizer

Track how characters, themes, and plot elements rise and fall across the timeline of any book or document.
Created as a vibe coding experiment and learning experience.

---

## What it does

Upload a `.txt` or `.pdf` file, type one or more words to track (e.g. character names, recurring motifs), and Bookworm plots a smooth frequency curve for each word across the full timeline of the text. Chapter markers are detected automatically and overlaid on the chart for navigation.


---

## Features

- **Frequency chart** — sliding-window analysis with adjustable window size; smooth curves per tracked word; up to 8 words or phrases at once
- **Word count display** — total occurrences shown in the legend and on each word chip after analysis
- **Exact / substring match** — toggle word-boundary matching on or off
- **Chapter detection** — automatically finds `Chapter`, `Part`, `Book`, `Prologue`, `Epilogue`, and more; overlays dashed markers on the chart with rotated labels
- **Chapter Navigator** — scrollable panel listing all detected chapters; click any entry to zoom the chart to that section
- **Reading panel** — click any data point to read the raw excerpt at that position, with tracked words highlighted in their chart colours
- **Replay animation** — play button progressively reveals the chart left-to-right with adjustable speed; pausable and loopable
- **Semantic network** — force-directed graph showing co-occurrence relationships for any word in the text; click nodes to re-centre
- **Zoom and pan** — zoom into any region of the frequency chart with a slider; pan with a second slider
- **Export PNG** — download the frequency chart (with labelled legend) or the semantic network as a PNG image
- **Light / dark theme** — toggleable; preference saved across sessions
- **Sample text** — Alice's Adventures in Wonderland is built in; no upload needed to try the app

---

## Usage

### Option 1 — Open locally

```
git clone https://github.com/giuseppevicari/bookworm.git
cd bookworm
open index.html          # macOS
start index.html         # Windows
xdg-open index.html      # Linux
```

It runs entirely in the browser.

### Option 2 — Try the sample

Open the app and click **"or try with sample text"** beneath the upload zone to load Alice's Adventures in Wonderland instantly.

### Tracking words

1. Upload a `.txt` or `.pdf` file (or use the sample text).
2. Type a word or phrase in the **Track Words** box and press Enter or comma. Add up to 8 terms.
3. Click **Analyse**.
4. Use the **Window Size** slider to adjust smoothing (larger = smoother, smaller = more detail).
5. Switch to the **Network** tab and search a word to explore its co-occurrence relationships.

---


## Requirements

- Any modern browser (Chrome, Firefox, Safari, Edge — released 2020 or later)
- No internet connection required once the page has loaded (all CDN assets are loaded on first visit)
- PDF text extraction requires a text-based PDF; scanned image PDFs are not supported

---

## Deployment

```
vercel deploy
```

The included `vercel.json` configures the project as a static site with no framework detection.

**Any static host** (GitHub Pages, Netlify, Cloudflare Pages): upload or point the host at the repository root.

---

## Troubleshooting

**PDF shows no text / empty analysis**
The PDF is likely a scanned image. Bookworm uses PDF.js text extraction, which requires a text layer. Try a different PDF or paste the text into a `.txt` file.

**No chapters detected**
Chapter detection looks for headings at the start of a line (`Chapter 1`, `CHAPTER ONE`, `Part II`, `Prologue`, etc.). Plain prose without structural headings will not produce markers — this is expected.

**Chart looks flat for a rare word**
Try reducing the window size so each window covers fewer words. The default (500 words) smooths over short bursts of usage.

**Export PNG is black**
This can happen if the browser blocks canvas operations on a `file://` URL in strict security mode. Use a local server (`npx serve .`) or deploy to a host.


---

## Reporting issues

Open an issue at [github.com/giuseppevicari/bookworm/issues](https://github.com/giuseppevicari/bookworm/issues). Include:

- Browser and OS
- File type (`.txt` / `.pdf`) and approximate word count
- Steps to reproduce
- What you expected vs. what happened

---

## License

[MIT](LICENSE) © 2026 giuseppevicari
