# voiceDraw

Narrated whiteboard videos from a text file. You describe what to draw on an
[Excalidraw](https://excalidraw.com) canvas and what to say; voiceDraw speaks the narration, draws
each element as its words are spoken, and writes an MP4.

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

```sh
npx voicedraw render hello.voicedraw.json     # writes hello.mp4
```

The format is meant to be written by a coding agent as much as by a person. The goal is to make an
agent's explanations quicker to take in: a model writes faster than anyone can read, so let it draw
and talk instead.

## Install

You do not run voiceDraw yourself; your coding agent does. So the way to install it is to ask the
agent. Paste this into Claude Code, Codex, Cursor, Gemini CLI or any other agent that can run a shell
command:

```text
Install the voicedraw skill with `npx skills add Chrischen996/voiceDraw -g -y`, then make me a video
explaining how DNS works.
```

It needs Node.js 20 or newer.

To try it once without installing anything:

```text
Run `npx -y voicedraw@latest guide`, then make me a video explaining how DNS works.
```

### By hand

The command also works on its own:

```sh
npm install -g voicedraw
voicedraw setup        # optional: download the speech model now instead of on the first render
```

Rendering uses three things, and `voicedraw setup` reports on each:

- **A speech model**, about 365 MB, downloaded on first use to a cache folder.
- **A Chromium-family browser** to draw the frames and encode the video. An installed Chrome,
  Chromium or Edge is used; if there is none, a headless Chrome is downloaded. A browser built
  without an H.264 encoder still works, several times more slowly.
- **ffmpeg**, to put picture and sound in one file. It comes with the `ffmpeg-static` dependency. If
  your npm is set to skip install scripts, install ffmpeg yourself or point `VOICEDRAW_FFMPEG` at one.

Drawing and encoding happen locally. Speech does too, except for one thing: when
[`edge-tts`](#online-voices) is installed and a script does not name a voice, the narration goes to
Microsoft's online voice. Name a built-in voice (`--voice lin`) to keep it all on your machine.

## Use

```
voicedraw guide                 the script format, with examples
voicedraw check <script>        errors and layout warnings, without rendering
voicedraw snapshot <script>     one moment of the canvas as a PNG (--step 2, --at 12.5)
voicedraw render <script>       the MP4 (-o out.mp4, --voice lin, --speed 1.1, --no-captions)
voicedraw voices                the voices
```

`voicedraw guide` prints the full reference, which is also in [`docs/format.md`](docs/format.md). In
short: a script is a list of steps, each with narration (`say`) and actions (`do`). An action adds,
updates or removes an Excalidraw element, moves the camera, or points at something, and its `at` names
the phrase of the narration it starts on. The renderer finds when the voice reaches that phrase: the
start of a sentence exactly, a word inside a sentence to within about a quarter of a second.

The first frame of the video is a cover, the canvas as step 1 ends, so a feed that takes its
thumbnail from that frame does not show an empty canvas. `"cover": 3` in the script picks another
step and `"cover": false` leaves the cover out.

Besides Excalidraw's shapes, text, arrows and images there is a `math` element, which typesets a
formula from LaTeX: `{ "type": "math", "tex": "\\frac{n(n+1)}{2}" }`. It is written out as it
appears, and an update can replace it with the next line of a derivation.

**Chart lines stay straight.** A `freedraw` element is a pen stroke: by default its points are
smoothed into a curve, which is right for handwriting and signatures. When the points are data — a
price line, a trajectory, anything with corners — set `"smooth": false` and the segments stay
straight:

```json
{ "id": "price", "type": "freedraw", "smooth": false,
  "points": [[140, 80], [300, 200], [420, 250], [500, 330]] }
```

`check` exists because whoever writes a script usually cannot see it. It catches references to
elements that are not there, phrases the narration never says, shapes that half overlap, labels that
will not fit and drawings hidden under the captions. Two of its warnings catch the things that most
often ruin a picture quietly:

- **A stroke that crosses a label.** A `line` or `freedraw` whose points run over an existing label
  hides the text; `check` names both sides of it, so the author knows to move the label, the
  stroke, or the order of the two.
- **Strokes that almost join.** Two open strokes whose ends sit close together, but not where one
  should continue the other, read as a broken line. `check` prints the ends of both and says which
  one to move; ends that actually meet are left alone.

`snapshot` renders a still in a second or two, with no speech, to look at the layout before
rendering the video.

## With an agent

voiceDraw is a command-line program, and its skill is one Markdown file of instructions:
[`skills/voicedraw/SKILL.md`](skills/voicedraw/SKILL.md). Nothing in either is specific to one
agent. Any agent that can run a shell command can make a video:

- **An agent that reads skills:** `npx skills add Chrischen996/voiceDraw` installs the skill for the
  agents it finds, in this project, or for every project with `-g`. `-a <agent>` picks the agents.
  This is the command the prompt under [Install](#install) has the agent run for you.
- **Claude Code, as a plugin:** this repository is also a plugin, so its `skills` folder is picked
  up when the repository is installed as one.
- **Any other agent:** tell it to run `npx -y voicedraw@latest guide` and make a video of what you
  want explained.

The skill only says what to run: read `voicedraw guide`, write a script, `check`, `snapshot`,
`render`. The format reference lives in the command, so the skill does not go out of date with it.

## What it costs in tokens

A script says what to draw and leaves how to the renderer, so there is little of it to write.

| Script | Video | The script |
| --- | --- | --- |
| [`examples/cache.voicedraw.json`](examples/cache.voicedraw.json) | 16 s | 549 tokens |
| [`examples/dns.voicedraw.json`](examples/dns.voicedraw.json) | 23 s | 716 tokens |
| [`examples/binary-search.voicedraw.json`](examples/binary-search.voicedraw.json) | 25 s | 685 tokens |

What an agent reads first does not grow with the video: the skill is a few hundred tokens and
`voicedraw guide` about 4,100. Layout problems come back from `check` as text, so finding them costs
no screenshots, and one `snapshot` is usually enough to look at before rendering.

## Voices

Left unset, a script is narrated by an [online voice](#online-voices) for the language of its
narration: `edge:zh-CN-XiaoxiaoNeural` for Chinese and `edge:en-US-EmmaMultilingualNeural` for
English. If that cannot be had (no `edge-tts`, no connection), the built-in voice says the whole
script instead, so one video never has two voices in it. A voice the script or `--voice` names is
used as it is, with no fallback.

The built-in voice is a small Kokoro speech model run on the CPU by sherpa-onnx. It speaks Mandarin
and English several times faster than real time on a laptop, and the model is Apache-2.0 licensed.

`voicedraw voices` lists four built-in voices: `lin` (female) and `chen` (male) for Chinese, `maple`
and `sol` for English. A script can also ask for any speaker by the model's own name for it, such as
`"voice": "zf_001"`.

### Online voices

The built-in voice is good and not great. Microsoft's online neural voices, the ones behind Read
Aloud in Edge, sound closer to a person, so they are the default. A script can also name one:

```json
{ "voicedraw": 1, "voice": "edge:en-US-AvaMultilingualNeural", "steps": [] }
```

or `voicedraw render talk.voicedraw.json --voice edge:zh-CN-YunxiNeural`. `voicedraw voices` lists a
few, and `edge-tts --list-voices` has them all. They work differently from the built-in voice:

- They need the [`edge-tts`](https://github.com/rany2/edge-tts) program (`pipx install edge-tts` or
  `uv tool install edge-tts`) and a connection. Without them, a script that names no voice falls back
  to the built-in one.
- The narration is sent to Microsoft. The built-in voice sends nothing anywhere.
- Microsoft runs this service for its browser and does not offer it as an API, so it can change or
  stop without notice, and it comes with no licence to use the voices elsewhere. For something you
  depend on, use the built-in voice or `--tts-command` with a service you have an agreement with.

### Another engine

To use another speech engine altogether, pass `--tts-command`. The command is run once per sentence,
gets the sentence on stdin, and must write a WAV file to the path in `$VOICEDRAW_OUT`:

```sh
voicedraw render talk.voicedraw.json --tts-command "python my_tts.py"
```

## Limits

- Chinese means Mandarin. Other languages are not supported by the built-in voice.
- The built-in voice's English dictionary has about 178,000 words. An unknown word is assembled from
  known ones or read by spelling rules, which is sometimes wrong for names. Respell a name the way it
  sounds if it comes out badly.
- English words inside a Chinese sentence are less reliable than either language alone.
- A word's time inside its sentence is an estimate, good to about a quarter of a second. Sentence
  starts are exact.
- There is no audio besides the narration.

## Credits and licenses

voiceDraw is MIT licensed. How it works inside, and how to build and test it, is in
[CONTRIBUTING.md](CONTRIBUTING.md). It builds on:

- [Excalidraw](https://github.com/excalidraw/excalidraw) (MIT), bundled into the player with its fonts.
- [MathJax](https://github.com/mathjax/MathJax-src) (Apache-2.0), bundled into the command with its
  TeX font, which typesets the formulas.
- [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (Apache-2.0), which runs the speech model.
- [Kokoro v1.1](https://huggingface.co/hexgrad/Kokoro-82M-v1.1-zh) (Apache-2.0), the speech model,
  downloaded at run time in sherpa-onnx's packaging of it. The pronunciation dictionaries that come
  with it are from misaki (Apache-2.0).
- ffmpeg, through [`ffmpeg-static`](https://github.com/eugeneware/ffmpeg-static), which downloads a
  GPL build and runs it as a separate program.
