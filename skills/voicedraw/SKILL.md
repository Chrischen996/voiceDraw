---
name: voicedraw
description: Make a narrated whiteboard video (MP4) in which an Excalidraw diagram is drawn step by step while a voice explains it, in English or Chinese. Use when asked for an explainer video, an animated diagram, a narrated walkthrough of an architecture, algorithm or process, or a video version of an Excalidraw sketch.
---

# voiceDraw

voiceDraw turns a small JSON script into an MP4 of a whiteboard video: a voice speaks the narration,
and each drawing appears on an Excalidraw canvas at the moment the narration reaches the words you
tie it to. You write the script; the `voicedraw` command checks it and renders it.

## Running the command

Use `npx -y voicedraw@latest` unless `voicedraw` is already installed on the PATH. It needs Node.js
20 or newer. The first render downloads a speech model (about 365 MB), which can take a few minutes.

## Steps

1. Run `voicedraw guide` and read all of it. It is the reference for the script format, and it is the
   version that matches the installed command. Do not write a script from memory.
2. Decide what the video should show before writing JSON: what goes on the canvas, in what order, and
   what is said as each part appears. Aim for one idea per step.
3. Write the script to `<name>.voicedraw.json`.
4. Run `voicedraw check <file>` and fix every error and warning it reports. The two layout warnings
   worth acting on are a stroke running over an existing label and two open strokes whose ends do
   not meet — `check` names both sides of each, so move the label, the stroke, or the order of the
   two.
5. Run `voicedraw snapshot <file>` and look at the PNG it writes. Fix anything that overlaps, is cut
   off, or is hard to read, then check again. Use `--step <n>` to see the canvas partway through.
6. Run `voicedraw render <file>`. It prints the path of the MP4. Give that path to the user.

## Notes

- **Charts stay straight.** A `line` or `freedraw` is smoothed into a curve by default, which suits
  a signature or a wobble. For a data line — a price that falls and recovers, a trajectory — set
  `"smooth": false` so it turns corners instead of bulging.
- **The pointer lands on the stroke.** A `point` action circles the middle of a line, arrow or
  freedraw, not the empty space beside it, so it reads as pointing at the thing.
- Formulas are supported: a `math` element typesets LaTeX, and Greek letters and maths symbols can be
  typed straight into text and labels. The guide has the details; narration says a formula in words.
- The first frame of the video is a cover for thumbnails: the canvas as step 1 ends. If another step
  makes a better picture of the video, set `"cover"` to its number.
- `voicedraw voices` lists the voices. Leave `voice` unset unless the user asks for one: by default
  the narration goes to Microsoft's online voice for its language, and to the built-in voice if
  `edge-tts` or a connection is missing. If the user wants nothing sent online, set a built-in voice
  (`lin`, `chen`, `maple`, `sol`). Match the voice to the narration's language: a Chinese script read
  by an English voice (or the reverse) comes out accented and misread, and `check` flags a `voice`
  that does not match the narration's language.
- Rendering takes roughly as long as the video is. Speech is cached, so rendering again after
  changing only the drawing is faster.
- If a command fails, read its message: it says what is missing. `voicedraw setup` downloads
  everything rendering needs and reports what it found.
