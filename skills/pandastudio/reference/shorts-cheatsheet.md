# Shorts command cheat sheet — zero spelunking

Every command a standard short edit needs, with exact shapes, auto-generated
from the live CLI arg schemas. If you are editing a short, this file plus
`shorts-styles.md` is ALL you need — do NOT grep source code or dump full
command schemas; that costs minutes.

Setup (zsh-safe — unquoted $VAR does not word-split in zsh):

```bash
cd <openscreen repo>   # or wherever the CLI lives
ps_() { node electron/cli/pandastudio.mjs --no-launch "$@"; }
ID=<project-id>        # pass --id=$ID on every project/transcript/caption verb
```

Conventions that save round-trips:

- Overlay placement time is `--atMs` (NOT startMs). `atMs`/`startMs`/`endMs`
  are EDITED-timeline ms. Pass `--anchorSourceMs` (source-time ms, e.g. a
  transcript word's source time) whenever placing from a transcript beat so
  the region survives later trims.
- Bundled asset URLs: `bundled:transition/light-sweep`,
  `bundled:sound/camera-click`, `bundled:sound/mouse-click`. Full catalogs:
  `asset.list-transitions/sounds/music`.
- `project.add-motion-graphic` attaches a mouse-click SFX by default — pass
  `--soundUrl=none` on text/emphasis overlays.
- Bulk timeline ops (trims/zooms/speeds) go through ONE
  `project.apply-edit-plan` call, not 20 sequential add-* calls.
- `motion.render-html` / `motion.generate` submissions must run SERIALLY —
  submit one, `job.wait` it, then submit the next. Parallel submissions race.
- Zoom depth→scale: 1=1.25x, 2=1.5x (house default), 3=2.0x… Prefer depth 2.
- Transcript pacing: fetch the transcript ONCE, build your full beat map in
  one pass (cuts + zoom beats + graphic slots + payoff), THEN execute. Do not
  re-derive timings per edit.

## Hyperframes starter shell (9:16 band / full-frame graphics)

Copy this shell verbatim and edit the content + timeline only. The contract
lines (composition attrs, gsap src, paused timeline, bracket-form
registration) must survive as-is. Pure CSS/text/SVG animation only — no
images (image decode is not awaited; frames can render empty). No
Date.now()/Math.random() — renders must be deterministic.

```html
<div id="comp" data-composition-id="comp" data-width="1080" data-height="1920"
     style="width:1080px;height:1920px;position:relative;overflow:hidden;background:transparent;">
  <!-- content here — absolute-positioned children -->
</div>
<script src="_shared/gsap.min.js"></script>
<script>
  const tl = gsap.timeline({ paused: true });
  // tl.to("#el", { ... , duration: 0.6, ease: "power3.out" }, 0.2);
  window.__timelines["comp"] = tl;   // bracket form — required by the lint
</script>
```

For a designed-segment band panel: render FULL-FRAME 1080×1920 with
`--transparent=true`, paint only the band the camera does NOT occupy, leave
the camera's band fully transparent (`cameraSide=bottom --cameraRatio=55`
means paint the top ~45%).

## Command shapes

### `project.read`
Read a project's full JSON (look up by id or path). Returns { path, project, clipStates }. Pass `--includeTranscript=false` to strip per-clip transcripts from the returned project object — they're typically the bulk of the payload (often 600+ KB on a 5-min recording) and most agent flows don't need the words after pacing is done. clipStates always includes a transcribed/wordCount summary so you still know whether to call transcript.transcribe. Pass project.revision back as expectedRevision on save.

```
ps_ project.read 
```
Optional: `--id=<string>` `--path=<string>` `--includeTranscript=<boolean>`

### `project.render-sheet`
Render N preview frames sampled across an edited-time range and tile them into ONE contact-sheet PNG (row-major, left→right then top→bottom). THE verification tool: one call + one image read shows pacing, caption changes, zoom ramps, and overlay mutations — use this instead of N sequential render-frame calls. Returns { sheetPath, cols, rows, frames: [{index,row,col,atMs,timeMs,path}] } — per-cell times come from the frames array, nothing is burned into the image. Each frame's seek is verified the same way render-frame verifies (no stale frames). Requires the project to be open (or openable) in the editor.

```
ps_ project.render-sheet 
```
Optional: `--id=<string>` `--path=<string>` `--fromMs=<number>` `--toMs=<number>` `--count=<number>` `--cols=<number>` `--outPath=<string>`

### `project.render-frame`
Render the composited preview frame at a given edited-time to a PNG and return its path so a vision agent can READ it. Use to locate on-screen text/UI (e.g. an email to blur) before placing a focus region. Returns { path, width, height, timeMs, maskRect }. `maskRect` is the video content rect as 0..1 fractions of the image — the SAME space spotlight/blur x/y/width/height use. To cover something you see at image-fractions (ix,iy,iw,ih): regionX=(ix-maskRect.x)/maskRect.width, regionY=(iy-maskRect.y)/maskRect.height, regionW=iw/maskRect.width, regionH=ih/maskRect.height. Note: existing focus regions are NOT drawn in this frame (so you see the content clearly), and the frame reflects any active zoom at that time.

```
ps_ project.render-frame --atMs=<number>
```
Optional: `--id=<string>` `--path=<string>` `--outPath=<string>`

### `transcript.get`
Return the merged edited-time transcript: every word, with id, text, startMs, endMs in EDITED-TIMELINE coordinates (sum of preceding clip durations applied). Use the word IDs as input to transcript.delete-words.

```
ps_ transcript.get 
```
Optional: `--id=<string>` `--path=<string>`

### `transcript.search`
Find every occurrence of a phrase in the merged transcript. Returns matches with their word IDs (so you can delete them or jump to them).

```
ps_ transcript.search --query=<string>
```
Optional: `--id=<string>` `--path=<string>`

### `transcript.delete-words`
Delete words from the project by ID — internally translates each word's [startMs, endMs] into a trim region the exporter skips. This IS the editor's 'click word, hit delete' workflow, callable from the CLI.

```
ps_ transcript.delete-words --wordIds=<array>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `transcript.remove-silences`
Detect and trim silent sections longer than `thresholdMs`. Runs the SAME two passes as the UI Remove Silences button and unions them: (1) transcript word-gaps — leading silence before the first word, inter-word pauses, trailing silence after the last word; (2) ffmpeg audio-level `silencedetect` (noise=-30dB) on each clip's media, which catches real dead air the transcript misses when speech-to-text invents phantom words over quiet stretches. Bulk-creates trim regions. Default threshold: 500ms. Returns { removedCount, totalTrimmedMs, audioDetectionFailed }.

```
ps_ transcript.remove-silences 
```
Optional: `--id=<string>` `--path=<string>` `--thresholdMs=<number>` `--expectedRevision=<number>`

### `transcript.remove-fillers`
Auto-detect filler words across the merged transcript and translate them into trim regions. By DEFAULT removes only vocalised pauses (um, uh, uhm, umm, hmm, hm) — sounds with no lexical meaning, so removing every occurrence can never break a sentence. Pass `aggressive=true` to ALSO catch lexical-word fillers (like, you know, i mean, sort of, kind of); these can false-positive on legitimate uses ('I like this template' loses 'like'), so make it explicit. Always also strips immediately-repeated words ('the the') unless includeRepeats=false. Call after transcribe.

```
ps_ transcript.remove-fillers 
```
Optional: `--id=<string>` `--path=<string>` `--aggressive=<boolean>` `--includeRepeats=<boolean>` `--expectedRevision=<number>`

### `project.apply-edit-plan`
Apply a whole editing plan in ONE call: a JSON array of timeline ops applied atomically (single read → all ops → single save; any invalid op aborts the batch with nothing written). Supported ops: {op:'add-trim',startMs,endMs} · {op:'add-zoom',atMs,durationMs,depth?,focusX?,focusY?,soundUrl?,anchorSourceMs?} · {op:'add-speed',startMs,endMs,speed}. This replaces 20+ sequential add-* round-trips when executing a beat map — ALWAYS prefer it for bulk edits.

```
ps_ project.apply-edit-plan --plan=<string>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `project.add-trim`
Mark a span the exporter SKIPS. Use directly to delete sections by time range; transcript.delete-words and transcript.remove-fillers also create trims under the hood.

```
ps_ project.add-trim --startMs=<number> --endMs=<number>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `project.add-zoom`
Add a zoom region at a given timeline time. Used to highlight UI moments — typical pattern: agent finds 'click X' in the transcript, drops a zoom on that beat. PASS anchorSourceMs WHEN PLACING FROM A TRANSCRIPT WORD (or any source-time value): the zoom will then re-anchor automatically when the user/agent adds trims or speed regions, instead of drifting off the click.

```
ps_ project.add-zoom --atMs=<number> --durationMs=<number>
```
Optional: `--id=<string>` `--path=<string>` `--depth=<number>` `--focusX=<number>` `--focusY=<number>` `--followCursor=<boolean>` `--soundUrl=<string>` `--soundVolume=<number>` `--anchorSourceMs=<number>` `--anchorSourceEndMs=<number>` `--expectedRevision=<number>`

### `project.add-speed`
Mark a span to play back at a different speed. Useful for fast-forwarding setup steps.

```
ps_ project.add-speed --startMs=<number> --endMs=<number> --speed=<number>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `project.add-motion-graphic`
Drop a motion-graphic MP4 (typically the output of motion.generate) onto the timeline as a media overlay.

```
ps_ project.add-motion-graphic --durationMs=<number>
```
Optional: `--id=<string>` `--path=<string>` `--fromJob=<string>` `--file=<string>` `--atMs=<number>` `--soundUrl=<string>` `--soundVolume=<number>` `--anchorSourceMs=<number>` `--anchorSourceEndMs=<number>` `--backdropBlurStrength=<number>` `--backdropBlurTint=<string>` `--layer=<string>` `--expectedRevision=<number>`

### `project.add-designed-segment`
Create a 'designed segment' — the YouTube split look where the host stays live on one half of the frame and a motion-graphic panel fills the other. 16:9: host right / panel left, like a Vox or MKBHD explainer beat. 9:16 SHORTS: host in a top/bottom band, panel in the other band (cameraSide 'top'|'bottom', paired with a 9:16-rendered premium panel like `paper-panel` or `vox-side-panel`, which reflow into a band). One atomic call: the camera is repositioned + cover-cropped into its half (a full-bleed split clip-transform) AND a full-frame transparent panel graphic is composited on top, both over the SAME time range with the same transition and source anchor, so they can never drift or mismatch. Everything stays in the live compositor — the camera is full-quality and scrubbable, nothing is baked. AUTHOR THE PANEL GRAPHIC AS A FULL-FRAME (1920×1080) TRANSPARENT render (motion.render-html --transparent): paint the opaque panel + content on the graphic's panel side, leave the camera's half fully transparent. For 9:16 render the panel at 1080×1920 — the premium `paper-panel` and `vox-side-panel` are aspect-aware and reflow into a top/bottom band. See SKILL.md 'Designed segments' and 'Editing a Short'.

```
ps_ project.add-designed-segment --durationMs=<number> --cameraSide=<string>
```
Optional: `--id=<string>` `--path=<string>` `--fromJob=<string>` `--file=<string>` `--atMs=<number>` `--cameraRatio=<number>` `--transitionMs=<number>` `--soundUrl=<string>` `--soundVolume=<number>` `--anchorSourceMs=<number>` `--anchorSourceEndMs=<number>` `--expectedRevision=<number>`

### `project.update-region`
Patch an existing region in-place. Only the fields you pass are changed. Use project.read to find region ids (editor.*Regions for visual regions, audioOverlays[] for audio). Common patches by type — zoom: startMs/endMs/depth/focusX/focusY; trim: startMs/endMs; speed: startMs/endMs/speed; annotation: startMs/endMs/text/x/y/width/height; fx: startMs/endMs/opacity/blendMode/speed; overlay: startMs/endMs/x/y/width/height; clip-transform: startMs/endMs/preset/transitionMs; audio-overlay: startMs/endMs/sourceStartMs/volume. Time-shifting one half of a link group (e.g. a designed-segment pair) shifts every peer by the same delta automatically.

```
ps_ project.update-region --regionType=<string> --regionId=<string>
```
Optional: `--id=<string>` `--path=<string>` `--startMs=<number>` `--endMs=<number>` `--sourceStartMs=<number>` `--volume=<number>` `--depth=<number>` `--focusX=<number>` `--focusY=<number>` `--followCursor=<boolean>` `--speed=<number>` `--x=<number>` `--y=<number>` `--width=<number>` `--height=<number>` `--text=<string>` `--accentColor=<string>` `--textColor=<string>` `--backgroundColor=<string>` `--backgroundRadius=<number>` `--fontSize=<number>` `--fontFamily=<string>` `--opacity=<number>` `--blendMode=<string>` `--preset=<string>` `--transitionMs=<number>` `--expectedRevision=<number>`

### `project.remove-region`
Remove a placed region by type + id. Use project.read to find region ids (editor.*Regions for visual regions; audioOverlays[] for audio). regionType: zoom | trim | speed | annotation | fx | overlay | clip-transform | audio-overlay. Deleting one half of a link group (e.g. a designed-segment's clip-transform or its panel motion graphic) cascades to its peer so the pair stays coherent — see addDesignedSegment.

```
ps_ project.remove-region --regionType=<string> --regionId=<string>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `motion.generate`
Render a bundled motion-graphic template by id, with your own text/color slot values and a background mode. Returns { jobId }; poll job.get/job.wait for the output path, then add it to the timeline with project.add-motion-graphic (or project.add-designed-segment for split-panel). Discover templates + their editable slots with motion.list.

```
ps_ motion.generate --templateId=<string> --slots=<object>
```
Optional: `--aspectRatio=<string>` `--background=<string>` `--outputName=<string>` `--transparent=<boolean>` `--glassMode=<boolean>`

### `motion.render-html`
Render arbitrary HTML/CSS/JS to MP4. Pass `html` (inline) or `htmlPath`. Returns { jobId, outputPath }; poll job.wait. No template/slot machinery.

```
ps_ motion.render-html 
```
Optional: `--html=<string>` `--htmlPath=<string>` `--aspectRatio=<string>` `--width=<number>` `--height=<number>` `--durationMs=<number>` `--frameRate=<number>` `--outputName=<string>` `--audioPath=<string>` `--audioVolume=<number>` `--assets=<array>` `--transparent=<boolean>`

### `motion.verify-frames`
Extract PNG frames at specified timestamps from a rendered video for visual verification. Each frame in the response carries a full-res `path` AND a downscaled `previewPath` (1280-wide, ~600KB base64). For vision-capable models: `read` the `previewPath` of each frame to inspect the composition — fast and the model can actually see it. For non-vision models: skip the `read` and surface `outDir` to the user. Pass either entryId (export-library entry) or videoPath (arbitrary MP4). Timestamps are in seconds.

```
ps_ motion.verify-frames --timestamps=<array>
```
Optional: `--entryId=<string>` `--videoPath=<string>` `--outputName=<string>`

### `asset.list-transitions`
List every bundled transition (id, title, category, durationSeconds). Use the id with project.add-transition.

```
ps_ asset.list-transitions 
```
Optional: none

### `asset.list-sounds`
List every bundled sound effect (id, name, category, absolute path).

```
ps_ asset.list-sounds 
```
Optional: none

### `asset.list-music`
List every bundled background music track (id, title, category, mood, durationMs, absolute path). Use absolutePath with project.add-audio to attach a track to a project.

```
ps_ asset.list-music 
```
Optional: none

### `caption.toggle`
Enable or disable captions for the whole project.

```
ps_ caption.toggle --enabled=<boolean>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `caption.set-template`
Pick a caption template. One of: classic, modern, minimal, bold, spotlight, boxed, neon, colored, editorial.

```
ps_ caption.set-template --templateId=<string>
```
Optional: `--id=<string>` `--path=<string>` `--expectedRevision=<number>`

### `caption.set-style`
Override caption style fields: wordsPerLine, positionY (percent of frame height from the top, 0-100; e.g. 85 = lower area. Fractions ≤ 1.0 are treated as a 0-1 fraction and converted, so 0.85 also means 85%), fontFamily (font name, e.g. 'Inter', 'Georgia', or a loaded custom font — unloaded fonts fall back to a system default), color, backgroundColor, highlightColor, highlightBackgroundColor, strokeColor, strokeWidth, fontSize (in REM, 1.0–5.0 — matches the editor's Font size slider; values are clamped to that range, and a px value is converted to rem).

```
ps_ caption.set-style 
```
Optional: `--id=<string>` `--path=<string>` `--wordsPerLine=<number>` `--positionY=<number>` `--fontFamily=<string>` `--color=<string>` `--backgroundColor=<string>` `--highlightColor=<string>` `--highlightBackgroundColor=<string>` `--strokeColor=<string>` `--strokeWidth=<number>` `--fontSize=<string>` `--expectedRevision=<number>`

### `project.add-audio`
Add a background audio track (music, VO, ambient) to the project. The audio plays from startMs to endMs on the edited timeline; sourceStartMs selects an in-point into the source file. Use project.update-region with regionType=audio-overlay to trim/reposition later. Use project.remove-audio (or remove-region) to remove. The export pipeline auto-mixes at the specified volume.

```
ps_ project.add-audio --audioPath=<string>
```
Optional: `--id=<string>` `--path=<string>` `--startMs=<number>` `--endMs=<number>` `--sourceStartMs=<number>` `--durationMs=<number>` `--volume=<number>` `--fadeIn=<number>` `--fadeOut=<number>` `--maxDurationMs=<number>` `--anchorSourceMs=<number>` `--anchorSourceEndMs=<number>` `--expectedRevision=<number>`

### `audio.probe`
Give the agent EARS without exporting: per-clip audio stats from the export-relevant track (cleanedAudioPath when present, else source media). Returns hasAudio, mean/max volume (dB), detected silence spans (source-time ms), and the project's audio overlays (music beds) so an agent can verify "is there dead air / is the level sane / did my music land" before export. Synchronous — a 1-2 min clip probes in a few seconds.

```
ps_ audio.probe 
```
Optional: `--id=<string>` `--path=<string>` `--clipId=<string>` `--noiseDb=<number>` `--minSilenceSec=<number>`

### `export.start`
Render the project to MP4 via the same Tier-3 PixiJS pipeline the editor's Export Video button uses. Agent exports route through an editor BrowserWindow (the existing one if it's already on this project, otherwise a hidden one spawned for the duration of the render). Async — returns { jobId, outputPath }; poll job.wait. Output lands at outputPath; defaults to <project-name>.mp4 in the recordings dir.

```
ps_ export.start 
```
Optional: `--id=<string>` `--path=<string>` `--outputPath=<string>` `--quality=<string>`

### `job.wait`
Block (up to timeoutMs, default 5 minutes) until the job reaches a terminal state. Hard-capped at 30 minutes — enough to cover realistic motion-graphic renders. Returns `{ job, timedOut: true }` when the deadline elapses while the job is still running; this is NOT a failure — the agent should re-call job.wait with the same id to keep polling. The job continues regardless of whether anyone is waiting on it.

```
ps_ job.wait --id=<string>
```
Optional: `--timeoutMs=<number>`

## The 6-minute budget

A standard ≤60s short should spend roughly: pre-flight + transcript beat map
2 min · apply-edit-plan + captions + music 1 min · author + render graphics
2 min (render time dominates — author ALL HTML files first, then submit
serially) · render-sheet verify + fixes 1 min · then export (export runs
async; job.wait it). If you are past 8 minutes before export, you are
spelunking — come back to this file.
