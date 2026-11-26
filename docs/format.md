# Writing a voiceDraw script

A voiceDraw script is a JSON file that says two things: what to draw on an Excalidraw canvas, and
what to say while drawing it. `voicedraw render` turns the file into an MP4 — the narration is spoken
by a voice, and each drawing appears at the moment the narration reaches the words you tie it to.

```json
{
  "voicedraw": 1,
  "steps": [
    {
      "say": "Take a client and a server. Every page load begins with a request from one to the other.",
      "do": [
        { "at": "client", "add": { "id": "client", "type": "rectangle", "x": 140, "y": 280, "width": 220, "height": 110, "label": "Client" } },
        { "at": "server", "add": { "id": "server", "type": "rectangle", "x": 900, "y": 280, "width": 220, "height": 110, "label": "Server" } },
        { "at": "request", "add": { "id": "request", "type": "arrow", "from": "client", "to": "server", "label": "GET /" } }
      ]
    }
  ]
}
```

The workflow is short and repeatable:

1. Write the script, saved as something like `dns.voicedraw.json`.
2. Run `voicedraw check file`. It lists errors and layout warnings in plain text. Fix every one.
3. Run `voicedraw snapshot file` to save the finished canvas as a PNG (`--step 2` shows the canvas as
   step 2 ends). Look at it — you cannot judge a layout from coordinates alone.
4. Run `voicedraw render file` to speak the narration and write the MP4.

## The script

| Property | |
| --- | --- |
| `voicedraw` | `1`. Required. |
| `steps` | The list of steps, played in order. Required. |
| `title` | A name for the video. Not drawn. |
| `canvas` | `{ "width": 1280, "height": 720, "background": "#ffffff" }`, which is the default. |
| `voice` | A name from `voicedraw voices`. Default: the online voice for the narration's language, or the first built-in one if that is unavailable. A name starting with `edge:` is an online voice (see Narration). |
| `speed` | Speaking rate, 0.5 to 2. Default 1. |
| `captions` | `false` leaves the narration out of the picture. Default `true`. |
| `cover` | The number of the step whose finished canvas is the cover. Default `1`. `false` for no cover. |

The canvas is 1280 x 720 units and the video is 1920 x 1080 pixels, so everything is drawn 1.5 times
its size. Excalidraw's own defaults (20 px text, 2 px lines) are the right size at this scale. `x` grows
to the right and `y` grows down from the top left corner.

## Steps

A step is one beat of the video:

| Property | |
| --- | --- |
| `say` | The narration, one or more sentences. |
| `do` | A list of actions, played while the step is spoken. |
| `pause` | Seconds of silence after the step. |
| `duration` | For a step with no `say`: how long it lasts (default: as long as its actions take). |

A step usually has one idea. What is said is also what the captions show.

## Actions

Each action does one thing, and says when it happens:

| Property | |
| --- | --- |
| `at` | When the action starts: a phrase copied from the step's `say` (it starts as that phrase is spoken), or a number of seconds from the start of the step. Left out, it follows the previous action. |
| `duration` | Seconds the animation takes. Each action has a sensible default. |
| `enter` | How added elements appear: `draw` (stroke by stroke, the default), `fade`, or `none`. |

Then exactly one verb: `add`, `update`, `remove`, `camera`, or `point`.

### add

```json
{ "at": "box", "add": { "id": "box", "type": "rectangle", "x": 100, "y": 100, "width": 200, "height": 100, "label": "Cache" } }
```

`add` is an element spec, or a list of them drawn one after another.

### update

```json
{ "at": "dim", "update": { "id": "box", "opacity": 35 } }
{ "at": "move", "update": { "id": "box", "dx": 40, "dy": 0 } }
```

`update` changes an element already on the canvas: style, position (`x`/`y` absolute, `dx`/`dy`
relative), size, a text element's content, or a formula's LaTeX. `id` is one element or a list.

### remove

```json
{ "at": "clear", "remove": "box" }
```

Fades an element out.

### camera

```json
{ "camera": { "fit": ["cache", "server"], "padding": 80 } }
{ "camera": { "x": 0, "y": 720, "width": 1280 } }
{ "camera": "reset" }
```

`fit` takes a list of ids or `"all"`. With `x`, `y`, `width`, the height follows from the canvas
proportions. `"reset"` returns to the whole canvas.

### point

A laser pointer circles an element (by id) or a point `[x, y]` for a moment and leaves no mark. It is
used to direct the eye to something already drawn. For a line, arrow or freedraw, the orbit is centred
on the middle of the stroke; for a shape, on its own box.

```json
{ "at": "this one", "point": "cache" }
```

## Elements

Every element needs an `id`, unique in the script, and a `type`. Property names are Excalidraw's.

| Type | Needs | Also takes |
| --- | --- | --- |
| `rectangle`, `ellipse`, `diamond` | `x`, `y`, `width`, `height` | `label`, `rounded` (rectangle and diamond; default true) |
| `text` | `x`, `y`, `text` (`\n` breaks lines) | `fontSize`, `fontFamily`, `textAlign` |
| `math` | `x`, `y`, `tex` (a formula in LaTeX; see Math) | `fontSize`, `textAlign` |
| `arrow` | `from`, `to`: each an element id or a point `[x, y]` | `via`, `rounded`, `label`, `startArrowhead`, `endArrowhead` |
| `line` | `points`: `[[x, y], ...]` | `closed` (joins the ends, so `backgroundColor` fills it), `rounded`, `smooth` |
| `freedraw` | `points`: `[[x, y], ...]` | `smooth` |
| `image` | `x`, `y`, `src` (PNG, JPEG, GIF, WebP or SVG; path relative to the script) | `width`, `height` (default: the file's size and proportions) |

`x`, `y` is the top left corner. Points are in canvas coordinates. On a `text` or `math` element,
`"textAlign": "center"` makes `x` its middle and `"right"` its right edge, which places it without
knowing how wide it will be.

**`smooth`** controls how a path is drawn through its points. On by default, a `line` or `freedraw` is
a smooth curve through its points — right for a signature, a hand-drawn wobble, or an arrow that
should sweep. Set `"smooth": false` to keep the segments straight, which is what a data line wants:
a price that falls and recovers should turn a corner, not bulge into an S.

```json
{ "id": "price", "type": "freedraw", "smooth": false,
  "points": [[140, 80], [300, 200], [420, 250], [500, 330]], "strokeColor": "#e03131", "strokeWidth": 4 }
```

An arrow whose `from` or `to` is an id attaches to that element: it starts at the element's outline,
facing the other end, and follows when the element moves. `via` lists points the arrow passes through,
and `"rounded": true` turns the corners at those points into a curve. Arrowheads are `"arrow"`,
`"triangle"`, `"bar"`, `"dot"` or `null`; the default is an `"arrow"` at the end and nothing at the
start.

A `label` is typed in the middle of a shape and wraps to its width, or sits on the middle of an arrow.
`fontSize` (default 20; 16 on arrows) and `fontFamily` apply to labels too. Fonts are `"hand"` (the
default handwriting, which covers Chinese), `"normal"` and `"code"`.

### Math

A `math` element is a formula written in LaTeX, typeset the way a textbook prints it:

```json
{ "at": "sum", "add": { "id": "sum", "type": "math", "x": 640, "y": 200, "textAlign": "center", "fontSize": 32, "tex": "\\sum_{i=1}^{n} i = \\frac{n(n+1)}{2}" } }
```

- Write every backslash twice. JSON takes a single one as an escape, so `"\frac"` and `"\theta"`
  are not what they look like. `voicedraw check` names a formula it cannot typeset and says why.
- `tex` is the formula alone, with no `$` around it, set as a displayed equation. LaTeX's maths and
  the AMS additions are understood: fractions, roots, sums and integrals with limits, Greek letters,
  `\mathbb`, `\mathbf`, `\text{words}`, matrices, `cases`, and `aligned` or `\\` for several lines.
  `\textcolor{#e03131}{...}` colours one part. There are no equation numbers.
- `fontSize` is the size (default 20, which matches text; 28 to 40 suits a formula that is the
  subject of a step) and `strokeColor` the colour.
- `update` moves and recolours a formula like anything else, and `"tex"` replaces it, fading one
  formula into the next: a derivation is one element updated line by line.

```json
{ "at": "so", "update": { "id": "sum", "tex": "1 + 2 + \\dots + 100 = 5050", "strokeColor": "#2f9e44" } }
```

Text and labels are not LaTeX: `"label": "$\alpha$"` is drawn as typed, and `check` warns about it.
Type the symbol itself instead, which works in any text, label and font: `α β θ λ π σ Δ Σ Ω`,
`→ ← ↔ ⇒`, `≤ ≥ ≠ ≈ ± × ÷ √ ∞ ∈ ∑ ∫ ∂ ∇`, `x² xₙ ½`. Use a `math` element when the notation
needs layout: a fraction, limits on a sum, a root over an expression, a matrix.

### Style

| Property | Values | Default |
| --- | --- | --- |
| `strokeColor` | hex colour | `"#1e1e1e"` |
| `backgroundColor` | hex colour or `"transparent"` | `"transparent"` |
| `fillStyle` | `"solid"`, `"hachure"`, `"cross-hatch"`, `"zigzag"` | `"solid"` |
| `strokeWidth` | 1 thin, 2 normal, 4 bold | 2 |
| `strokeStyle` | `"solid"`, `"dashed"`, `"dotted"` | `"solid"` |
| `roughness` | 0 clean, 1 hand-drawn, 2 sketchy | 1 |
| `opacity` | 0 to 100 | 100 |
| `angle` | radians, clockwise | 0 |

Excalidraw's palette, which looks right together:

| | Stroke | Fill |
| --- | --- | --- |
| red | `#e03131` | `#ffc9c9` |
| green | `#2f9e44` | `#b2f2bb` |
| blue | `#1971c2` | `#a5d8ff` |
| yellow | `#f08c00` | `#ffec99` |
| violet | `#6741d9` | `#d0bfff` |
| gray | `#868e96` | `#e9ecef` |

## Layout

`voicedraw check` warns about the layout it can measure; `voicedraw snapshot` shows the rest.

- Leave at least 40 units between elements that are not inside one another. Two that half overlap
  get a warning.
- An element may sit fully inside another (parts inside a container).
- A label needs room: about 12 units per Latin letter and 20 per Chinese character at the default
  size, plus 20 of padding. Ellipses and diamonds hold less than their width suggests.
- A shape that contains other elements should not have a label, because the label sits in its middle.
  Put a `text` element near its top instead.
- A formula takes more room than its LaTeX suggests. At the default size a symbol is about 12 units
  wide; a line is about 20 tall, and 40 to 55 with a fraction or a sum in it. `check` measures the
  real thing, so its warnings about a formula are exact.
- Text is for names, values and short phrases. The narration is in the captions already.

Two warnings are worth learning because they catch the things that most quietly ruin a picture:

- **A stroke over a label.** A `line` or `freedraw` whose points pass through an existing label covers
  the text with its stroke. `check` names both (`its points run over the label of "box"`), so the
  author — who usually cannot see the canvas — knows to move the label, the stroke, or the order of
  the two.
- **Strokes that almost join.** Two open strokes whose ends sit within the reach of their round caps
  but not where one should continue the other read as a broken line. `check` prints both ends of
  each (`its ends are near "down"'s, but the two do not join`) and says which to move. Ends that
  genuinely meet are left alone.

## Narration

- Write for the ear: short sentences, plain words. What is said is also what the captions show.
- The voice does not read LaTeX, or symbols such as `=` and `²`. Say a formula in words ("x
  squared", "the sum from one to n") and show it in a `math` element.
- Numbers, percentages and years are read out. A short word in capitals (`API`, `DNS`, `JSON`) is
  spelled letter by letter; write it the way it sounds (`jason`) to have it read as a word. An English
  word the voice does not know is read by its spelling, which is usually close and sometimes wrong.
- English inside a Chinese sentence is less reliable than either language alone: about one such word
  in four comes out unclear. Keep to common terms, or say the term in Chinese and write the English
  on the canvas.
- A word's time within its sentence is estimated, so an `at` lands within about a quarter of a second
  of the word. The start of each sentence is exact.
- Left unset, the voice is one of Microsoft's online voices (`edge:zh-CN-XiaoxiaoNeural` for
  Chinese, `edge:en-US-EmmaMultilingualNeural` for English). They sound natural and say English and
  Chinese in one sentence without trouble, but the narration is sent to Microsoft and the `edge-tts`
  program must be installed. Without it, or without a connection, the built-in voice (`lin` or
  `maple`) says the whole script instead. Name a built-in voice to keep the narration on this
  machine.
- The built-in Chinese voices are the model's own Mandarin speakers, left alone. An English speaker
  mixed into one of them would make the English words a little better but bends the vowels toward an
  English accent and flattens the tones; the Chinese comes out less natural for it, so the mix is
  not applied. A Chinese script that names an English voice (or vice versa) is what the built-in
  voices are worst at: it says the language of the other with an accent. `check` flags a `voice`
  whose language does not match the narration's.

## A longer example

```json
{
  "voicedraw": 1,
  "title": "Why a cache helps",
  "steps": [
    {
      "say": "Every time the app needs a user, it goes to the database and asks. And the database is slow.",
      "do": [
        { "at": "app", "add": { "id": "app", "type": "rectangle", "x": 120, "y": 260, "width": 200, "height": 100, "label": "App", "backgroundColor": "#a5d8ff" } },
        { "at": "database", "add": { "id": "db", "type": "rectangle", "x": 940, "y": 260, "width": 220, "height": 100, "label": "Database", "backgroundColor": "#e9ecef" } },
        { "at": "asks", "add": { "id": "query", "type": "arrow", "from": "app", "to": "db", "label": "80 ms" } },
        { "at": "slow", "point": "query" }
      ]
    },
    {
      "say": "So we put a cache in between. It remembers the answers it has already seen.",
      "do": [
        { "at": "cache", "remove": "query" },
        { "add": { "id": "cache", "type": "ellipse", "x": 530, "y": 250, "width": 200, "height": 120, "label": "Cache", "backgroundColor": "#ffec99" } },
        { "at": "remembers", "add": [
          { "id": "ask", "type": "arrow", "from": "app", "to": "cache", "label": "1 ms" },
          { "id": "miss", "type": "arrow", "from": "cache", "to": "db", "strokeStyle": "dashed", "label": "only on a miss" }
        ] }
      ],
      "pause": 0.5
    },
    {
      "say": "Most requests never reach the database at all.",
      "do": [
        { "at": "never", "update": { "id": ["db", "miss"], "opacity": 35 } },
        { "at": "database", "camera": { "fit": ["app", "cache"], "padding": 120 } }
      ]
    }
  ]
}
```
