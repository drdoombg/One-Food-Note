# One Food Note — project notes for an AI assistant

A single self-contained `index.html` that holds a personal study library of poultry / food-safety
material (class notes, interactive trainers, Word notes, anatomy figures). It is published to GitHub
Pages. **Only `index.html` is served** — it embeds everything and loads nothing from the network.

## Layout of this folder

```
index.html                  <- THE DELIVERABLE. Generated. Never hand-edit.
PROJECT.md                  <- this file
_build/build.py             <- generator: reads the sources below, writes index.html
_build/shell.html           <- the page itself (HTML/CSS/JS) with a /*__PAYLOAD__*/ placeholder
<source files>              <- .html .docx .png .jpg .pdf, loose in the folder
Anatomy and Physio/         <- anatomy sources
```

Rebuild after changing anything: `python _build/build.py`

## How the single file is put together

`build.py` compresses each source and inlines it; `shell.html` is the UI and revives the payload at
click time. Three payload objects are injected in place of `/*__PAYLOAD__*/`:

| object  | holds | revived by |
|---|---|---|
| `DOCS`  | whole HTML documents, gzip + base64, shown in an iframe via `srcdoc` | `DecompressionStream('gzip')` |
| `NOTES` | HTML *fragments* (Word notes, the anatomy PDF reflowed), gzip + base64 | same |
| `A`     | images, WebP + base64, keyed by short id | `data:image/webp;base64,...` |

Sources ≈ 16 MB → `index.html` ≈ 4.3 MB, because:

- **PDF** (`Gallus gallus.pdf`, 27 pages) is **not** embedded as a PDF. Its text is extracted with
  PyMuPDF and reflowed to HTML; its 33 figures are pulled out and recompressed. Labels are baked
  into those figures, so nothing is lost. 2.4 MB → ~0.6 MB, and it reflows on a phone.
- **Word notes** are converted by `docx_html()` (python-docx): headings, lists, tables, inline
  images. They render as native HTML panes, not iframes, so they follow dark mode.
- **Images keep their own resolution** (WebP q76–78, no downscaling). An earlier build capped them
  at 1700px and the dense charts became unreadable — readability beats load time here, by explicit
  instruction. `NATIVE` in `build.py` is the "do not shrink" cap.
- **`Shelf Life.png`** is a 10902x5230 sheet holding seven separate panels. `SL_PANELS` crops it at
  full size. Four of those panels already existed inside `Shelf Life Notes.docx` at half resolution,
  so `SUBS` (keyed by the Word image's md5) swaps the sharp crop in; the three panels the notes never
  had (`sl5`, `sl6`, `sl7`) are appended to the end of that note. There is deliberately **no separate
  chart card** — it would duplicate the notes. `_build/split.py` is the recursive XY-cut that found
  the panel boxes; re-run it if the sheet changes.
- **HTML notes** are embedded byte-for-byte (plus the anchor fix below) so their own CSS and
  scripts keep working — `Shelf Life Organism and Conditions.html` is a carefully tuned `100vh`
  grid app and must get the full viewport. Do not restyle or re-flow these.

## The UI contract

- **Home is a card menu.** No persistent top bars — the user objected to bars eating ~20% of the
  screen. An item opens full-bleed; the only chrome is a small translucent pill bottom-left
  (`☰ Menu`, or `← Back` one level inside a group). Esc does the same.
- **Cards carry no subtitles.** Name + icon only. Groups show a count chip.
- **Colour taxonomy** (one row per colour, set by `c:` in `SECTIONS`):
  - `gold` — **Poultry Bible only. Leave this row alone.**
  - `blue` — labs and micro
  - `green` — process, packaging and NPD
  - `red` — anatomy, science and quality
- **Similar notes group behind one card** (`kids: [...]`) and open a selectable list first. Nesting
  is two levels only: row → card → list. Do not add a third.
- **Nothing is embedded twice.** The build prints `unreferenced:` — any asset not shown anywhere is a
  mistake, as is the same artwork living in two cards.
- **Images**: fit on open, wheel/pinch zooms at the pointer, drag pans, double-click toggles
  fit ↔ 2.5×, clicking off the picture closes.
- **Long notes carry their own index**: `with_index()` builds a contents list from `<h2>`s and puts
  an `↑ Index` link at the end of every section.

## Adding material

1. Put the file in the folder (or `Anatomy and Physio/`).
2. `.docx` → add to the loop that fills `NOTES`; image → add to the `IMG` loop; whole HTML page →
   add to the `DOCS` loop.
3. Add a card in `SECTIONS` in `_build/shell.html` under the right colour, as
   `{n:"Name", type:"doc|rich|img", src:"key"}`, or inside a group's `kids`.
4. `python _build/build.py`.

## Traps already hit — do not reintroduce

- **`<\/script>` is not an HTML closing tag.** It is only valid inside a JS string. Writing it into
  an injected `<script>` leaves the tag open and swallows the rest of the document. `inject_anchor_fix()`
  asserts the tags balance.
- **In a `srcdoc` iframe a plain `#anchor` link does not scroll.** Every embedded document with
  in-page links gets `ANCHOR_FIX`, a delegated click handler calling `scrollIntoView`.
- **`scroll-behavior:smooth` broke jumps** in the 67,000-pixel bible — the animation never landed.
  Jumps are instant on purpose.
- **Never measure a hidden pane.** The zoom view read a `display:none` pane, got width 0, scaled the
  image to nothing and showed a blank screen. It now re-fits via `ResizeObserver` once the pane has
  a real size.
- The bible's contents rows were plain `<div>`s, and its appendices were missing from them;
  `patch_bible()` turns the rows into links, ids the chapters, appends the appendices, and swaps the
  Save-as-PDF button for `↑ Index`.
- Browser storage is used only for theme and last-opened item; it may throw, so every access is
  wrapped.

## Verifying a change

There is no test suite. Headless Chrome is the harness:

```
chrome --headless=new --disable-gpu --virtual-time-budget=30000 --dump-dom file:///...  (grep the <title>)
```

The pattern used so far: copy `index.html`, inject a probe script before `</body>` that clicks
through the UI and writes its findings into `document.title`. This caught every bug listed above.
Decoding checks are also worth running: gunzip each `DOCS`/`NOTES` entry, confirm every `href="#x"`
has a matching `id`, and confirm each `A` entry starts with `RIFF....WEBP`.

Requires `DecompressionStream`: Chrome/Edge 80+, Firefox 113+, Safari 16.4+.
