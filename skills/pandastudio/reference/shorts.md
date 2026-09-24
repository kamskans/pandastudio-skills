<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Shorts: export → vertical clips

## Shorts: turning an exported video into vertical clips (v1.40+)

The shorts pipeline is two-step: **discover** the moments worth clipping, then **fork** the source project into a 9:16 short you can refine — or hand back to the user for review. Use this whenever the user says "make a vertical short of the X moment", "give me 3 shorts from yesterday's tutorial", "cut this for TikTok", etc.

### Step 1 — Discover shots in an exported video

The AI Shorts generator suggests 3–5 self-contained `[startMs, endMs]` segments scored 1–10, the same as the Exports library's **Generate Shorts** button. Pass the export id: it writes the shots onto that entry under `generatedShots` (replacing any previous set) and also returns them, best score first. MCP: `export_generate_shots { id }`.

```bash
# Pick an export to mine for shorts
EXPORT=$(pandastudio export.list --json | jq -r '.data.exports[0].id')

# Run the generator (local LLM picks engaging self-contained spans; up to a minute or two)
pandastudio export.generate-shots --id="$EXPORT" --json
# → .data = { id, shots: [{ id, startMs, endMs, title, description, score }, ...] }
#   and entry.generatedShots holds the same list
```

**Needs:**
- **A transcript on the export.** Exports copy the source project's transcript at export time, so run `transcript.transcribe` on the project BEFORE exporting. An export without one fails with `code: no_transcript`.
- **The local AI model.** If it isn't downloaded the verb fails with `code: model_missing`. There is no verb to download it: ask the user to click **Settings → AI Model → Download model** in PandaStudio (or the "Download AI model" banner in the Exports library), then retry. `llm.status` reports `downloaded: true` once it's ready.

If the model reply is unusable, it falls back to pause-based ~45 s segments (title = the opening words, score 5). Shots under 10 s are dropped.

### Step 2 — Fork the source project for ONE shot (the right path almost always)

`project.fork-from-shot` makes a **new 9:16 project** from the original source project — NOT from the flat exported MP4. Use this whenever the source project still exists and you want the short fully re-editable:

```bash
# Pick the highest-scoring shot from the export (read back from the entry,
# or take .data.shots[0].id straight from the generate-shots result)
SHOT=$(pandastudio export.list --json | jq -r --arg e "$EXPORT" '
  .data.exports[] | select(.id == $e) | .generatedShots
  | sort_by(.score) | reverse | .[0].id
')

# Fork — creates a new project, returns its id + path
pandastudio project.fork-from-shot --exportId="$EXPORT" --shotId="$SHOT" --json
# → { id, path, name: "Original — Short: ...", aspectRatio: "9:16", project: {...} }
```

**Screen recordings:** the fork's automatic fill-crop is skipped when the clip has a camera attached, so a Short forked from a screen recording starts shrunk-to-fit. Run `project.set-vertical-screen-layout --id=$NEW_PROJECT --fill=follow --corner=bottom-right` right after forking (see "Vertical screen recording" below).

**What the fork does to the project:**
- **Keeps** the source's first-row timing edits: clips, trim regions (silences/fillers/bad takes), speed regions, per-clip transcribed/audioCleaned status, per-clip transcript, root transcript, cleaned-audio paths, LUT/color grade.
- **Strips** every overlay/region row (they were sized for 16:9): zooms, FX, motion graphics, captions, transitions, lower thirds, annotations, clip-transforms (podcast/designed-segment layouts), background music, wallpaper, padding/shadow framing.
- **Switches** aspectRatio to 9:16 and adds two trim regions cropping everything outside the shot's edited-time `[startMs, endMs]` range.
- **Fills the vertical frame** — applies a centered cover-crop so a landscape (16:9) source fills the 9:16 canvas instead of being letterboxed into a band. (Probed from the source video's real dimensions; skipped for podcast composites and already-vertical sources.) **Face centering is automatic:** when the short opens in the editor, vertical talking-head clips are face-detected and a focal point is set so the cover-crop (and any top/bottom designed-segment band) keeps the face in frame rather than slicing the geometric middle. Override with the "Center on face" control or `project.set-focal-point` (see Settings below).
- **Names** the new project `"{Source Name} — Short: {shot title}"` and gives it a fresh UUID + revision 0.

The result is a **clean 9:16 canvas with the timing already done**. Now make it
retain: **load [`shorts-styles.md`](shorts-styles.md)** — the evidence-based recipe
playbook (kinetic-educator / bold-caption / podcast-clip / polished-explainer)
with the seven retention laws, per-recipe caption/zoom/overlay parameters, and
the verification pass. The snippet below is minimal generic polish only:

```bash
NEW_PROJECT=$(pandastudio project.fork-from-shot --exportId="$EXPORT" --shotId="$SHOT" --json | jq -r '.data.id')

# A bold caption template for vertical content
pandastudio caption.toggle --id="$NEW_PROJECT" --enabled=true
pandastudio caption.set-template --id="$NEW_PROJECT" --templateId=neon

# A topic-introducing pull-quote in the opening 3 seconds (see "Editorial emphasis" rules)
pandastudio motion.generate --templateId=caption-editorial-emphasis \
  --slots='{"sentence":"The whole stack, in 60 seconds.","emphasisWord":"60 seconds"}' \
  --aspectRatio=9:16 --json

# Emphasis zooms on the key beats
# ... drive the rest of the polish pipeline at 9:16 ...
```

### Drift detection — when the source project changed after export

If the user has edited the source project AFTER the export (removed a filler, added a graphic), the shot's `[startMs, endMs]` may no longer point at the same moment in the timeline. The verb detects this and returns:

```json
{ "ok": false,
  "error": "Source project has changed since export (export was at revision 4, project is now at 7) …",
  "details": { "code": "revision_mismatch", "expectedRevision": 4, "liveRevision": 7 } }
```

**Default behavior:** surface this to the user verbatim and ask whether to fork anyway. If they confirm, retry with `--force=true`. Don't silently force.

### When the fork isn't available

`project.fork-from-shot` is the ONLY shot-editing path an agent has — and the
only one the in-app **Edit Short** button leads with too (it silently falls
back to clipping the flat exported MP4 only for the cases below; there is no
separate agent verb for that fallback). The fork is unavailable when:
- The source project has been **deleted/moved** (the verb returns
  `code: "source_missing"`).
- The export pre-dates **v1.40** (`sourceProjectPath` is empty — the verb
  returns "Forking is only available for exports created in v1.40+").

In those cases tell the user the source project for this export isn't available,
so the short can't inherit the original edits — they can still make the short
from the **Edit Short** button in My Exports (which clips the rendered video),
or re-export from the source project to get a fork-capable export. Everywhere
else, just fork.

### Batch: make N shorts from one export

Combine the two verbs to fan out:

```bash
EXPORT=$(pandastudio export.list --json | jq -r '.data.exports[0].id')

# Generate, then fork the top 3 shots in parallel (shots come back best-first)
pandastudio export.generate-shots --id="$EXPORT" --json | jq -r '.data.shots[0:3] | .[].id' \
| while read SHOT; do
  pandastudio project.fork-from-shot --exportId="$EXPORT" --shotId="$SHOT" --json &
done
wait
```

Each fork is a separate 9:16 project the agent (or user) can then polish independently — different topic, different graphics, different music.

### Editing a Short — the vertical playbook (v1.43+)

The fork hands you a **clean 9:16 canvas with the cut already timed**. A bare
clip is not a Short — a Short lives or dies on **completion rate** (the ~70%
watch-through is where the algorithm starts pushing it).

> **Authoritative grammar: [`shorts-styles.md`](shorts-styles.md).** Pick a
> recipe there FIRST (kinetic-educator / bold-caption / podcast-clip /
> polished-explainer) and follow its parameter table; the layers below are the
> generic mechanics those recipes drive. One correction from the July-2026
> frame-by-frame study of top creators, superseding older guidance in this
> section: modern top shorts are NOT busy or transition-heavy. Every cut is
> semantic (new idea = new cut, every 3.5–8s), emphasis uses ONE mechanism,
> and between-beat transitions are essentially absent. Visual state must
> change every 3–8s — but via overlay/zoom/speaker changes, not FX spam.

**The five layers, in the order you add them:**

1. **Captions — always on, big, word-by-word.** This is non-negotiable for
   vertical: most Shorts are watched muted, and animated captions add roughly
   15–25% retention on their own. Toggle them on and pick a punchy template —
   `neon`, `bold`, or `hormozi`-style word-pop. Keep them large and
   high-contrast; the reading rhythm should match your pacing.
   ```bash
   pandastudio caption.toggle --id="$P" --enabled=true
   pandastudio caption.set-template --id="$P" --templateId=neon
   ```

2. **Editorial emphasis — used liberally.** Where a normal edit is sparing
   with pull-quotes, a Short leans in: stamp an emphasis title on the key
   *verb, number, or outcome* of almost every beat (the hook line, the payoff,
   any "3x", "$0", "in 60 seconds"). Animate it ON as the word is spoken.
   ```bash
   pandastudio motion.generate --templateId=caption-editorial-emphasis \
     --slots='{"sentence":"It writes the whole thing for you.","emphasisWord":"whole thing"}' \
     --aspectRatio=9:16 --json
   ```

3. **A transition on every text beat.** A clean transition each time a new
   on-screen text or section lands gives the edit its pulse — it reads as
   "produced" and resets attention. Place one at each beat boundary where a new
   emphasis title / graphic appears (a fast `whip`, `zoom`, or `glitch` — keep
   them short, ~200–300ms). Pair with an emphasis **zoom** on the punchline.

4. **Motion graphics — two vertical-native modes.** Both fill the 9:16 frame
   properly (the engine reflows to `data-width`/`data-height`, so always render
   at `--aspectRatio=9:16`; a 16:9 graphic dropped into a Short letterboxes):
   - **Full-screen graphic** — a `--aspectRatio=9:16 --background=solid` card
     between beats (a stat reveal, a big title, a list) that takes the whole
     frame for 1–2s, then cuts back to the host. Great as the hook frame and
     for the pattern interrupt.
   - **Top/bottom split (designed segment)** — the vertical equivalent of the
     desktop left/right split: the host fills one horizontal band while a
     motion-graphic panel fills the other. This is the canonical "react /
     explain over a graphic" Shorts look. **Use the same premium panels as
     16:9 — `paper-panel` (torn-paper title) or `vox-side-panel` (graph-paper +
     taped specimen card).** They are aspect-aware: render at `--aspectRatio=9:16`
     and they reflow into a top/bottom band automatically. Set the panel `side`
     to `top`/`bottom`, and call `project.add-designed-segment` with the matching
     **`cameraSide`** (opposite the panel) and the panel's `cameraRatio`
     (`paper-panel` → 55, `vox-side-panel` → 50):
     ```bash
     # Render the premium panel at 9:16 (reflows into a top band; transparent
     # bottom reveals the host). Same template as the 16:9 explainer.
     JOB=$(pandastudio motion.generate --templateId=paper-panel \
       --slots='{"side":"top","title1":"THE ONE TRICK","title2":"STOP TYPING","subtitle":"Speak it. The app writes it."}' \
       --aspectRatio=9:16 --background=transparent --json | jq -r '.data.jobId')
     pandastudio job.wait --id="$JOB" --json

     # Couple panel + host in ONE atomic call. Panel on top → host fills the
     # bottom band. paper-panel pairs with cameraRatio 55 (vox-side-panel → 50).
     pandastudio project.add-designed-segment --id="$P" --fromJob="$JOB" \
       --durationMs=4000 --cameraSide=bottom --cameraRatio=55 \
       --atMs=2000 --json
     ```
     The panel `side` (top/bottom) and `cameraSide` are **opposites** — panel
     `top` ⇒ `cameraSide bottom`. Prefer the premium panels over the plainer
     `split-panel`, exactly as you do for 16:9. **The host band keeps the face
     centered automatically** (the editor face-detects vertical talking-head
     clips and sets a focal point the cover-crop biases toward); override with
     `project.set-focal-point` if needed.

5. **A pattern interrupt around 25–35s.** If the short runs past ~25s, change
   *something* at that mark — a full-screen graphic, a hard zoom, a layout flip
   from full-host to split — right where viewers typically drift. One deliberate
   jolt buys the back half of the video.

**The shape of a good 45–60s Short:** hook frame (full-screen graphic or a hard
emphasis title in the first ~1s) → host talking with captions + an emphasis
title on each beat + a transition at each → one `paper-panel`/`vox-side-panel`
top/bottom split or a full-screen graphic for the main explanation → pattern
interrupt at ~30s → payoff line as a
big emphasis title → end on the result. Cut anything that doesn't earn its
second; aim for a visible change every 2–4 seconds.


## 9:16 layouts: which one for which footage

| Footage in a 9:16 project | Verb |
|---|---|
| Camera-only, one person | `project.set-shorts-layout --layout=full` (or `camera-corner` over a blurred copy of itself) |
| Screen recording (with or without camera) | `project.set-vertical-screen-layout` |
| Landscape, more than one person (talk show, interview, podcast panel) | `project.auto-reframe` |
| Single stationary talking head, fine-tune the crop | `project.set-focal-point` |

### Shorts layout: full-frame vs camera-corner-over-blur

For a **camera-only** clip in a 9:16 project, `project.set-shorts-layout` is the
one-click layout picker:

```bash
# Camera shrinks to a draggable bottom-right tile over a blurred copy of itself
pandastudio project.set-shorts-layout --id=$PID --layout=camera-corner
# Camera fills the frame (clears the transform + backdrop)
pandastudio project.set-shorts-layout --id=$PID --layout=full
```

`camera-corner` sets the main-clip transform AND a `blur-self` backdrop
together; reposition the tile afterward with `project.set-screen-transform`
(`x`/`y` are canvas-fraction center offsets, `scale` the tile size). The two
pieces are also independently settable: `project.set-backdrop
--mode=blur-self|wallpaper` controls only the fill behind a scaled-down video
(invisible while the video fills the frame). For a **screen recording** don't
use these; use the vertical screen layout below. The blurred self-fill renders
identically in preview and export.

### Vertical screen recording: screen fills the frame, camera in a corner

For a **screen recording** (with or without a camera) in a 9:16 project,
`project.set-vertical-screen-layout` is the layout: the screen is the main thing
and the camera is a small square tile in a corner (the tutorial/demo Short
look). Set the aspect first; the crop is tied to it.

```bash
pandastudio project.set-aspect-ratio --id=$PID --ratio=9:16
# Screen fills the frame, cropped around the mouse and panning with it; camera bottom-right
pandastudio project.set-vertical-screen-layout --id=$PID --fill=follow --corner=bottom-right
# Whole screen visible, full width, over a blurred copy of itself
pandastudio project.set-vertical-screen-layout --id=$PID --fill=fit
# Move just the camera tile (any picture-in-picture layout, 16:9 too)
pandastudio project.set-webcam-layout --id=$PID --corner=top-left
```

- **Pick the fill:** `follow` (default) for demos where the action is in one
  area at a time; it pans with the recording's cursor data (centered when there
  is none) and keeps the mouse in view, gliding rather than snapping. `fit` when
  everything on screen matters at once (a full dashboard, a side-by-side
  comparison), at the cost of a smaller screen.
- **Pick the corner** so the tile doesn't cover what the video is about:
  bottom-right by default; move it if the UI's key controls sit there.
  `render-frame` a few moments to check.
- It sets picture-in-picture, removes padding, and enlarges the tile for phones
  unless the user already sized it. Returns `{ followed, centered, skipped }`; a
  `skipped` clip wasn't cropped (read the reason).
- Cursor-follow zooms still land on the right spot inside the crop.

### Active-speaker auto-reframe: landscape multi-person → vertical (v1.70+)

When you crop a **landscape source with more than one person** (a talk show,
interview, podcast panel, any director-cut footage) into 9:16, a single static
cover-crop lands on the gap between people in wide shots and off-face in
close-ups of whoever isn't centered. `project.auto-reframe` is a **tracked
virtual camera** (the Opus Clip / Vizard approach):

1. **Shot detection** (ffmpeg scene cuts) segments the source.
2. **Dense face tracking** — MediaPipe FaceLandmarker (bundled, offline)
   samples ~7fps, with adaptive tiling so small/far faces in wide shots are
   still found. Detections are associated into per-person tracks.
3. **Audio active-speaker** — on multi-person shots it frames **whoever is
   talking** (mouth-open × speech-energy), not the biggest face.
4. **Smoothed camera** — the crop *pans* to follow the subject within a shot
   (dead-band hold + safe-zone clamp so the face never leaves frame) and
   *cuts* at shot boundaries.

```bash
# Reframe every landscape clip — tracks + follows the active speaker.
pandastudio project.auto-reframe --id=$PID --json
# → { reframed: [{clipId, shots, shotsWithFace}], skipped: [...], activePicture: [{clipId, rect}] }
# One clip only, or tune shot sensitivity / punch-in:
pandastudio project.auto-reframe --id=$PID --clipId=clip-1 --threshold=0.3 --minShotMs=500 --zoom=1.3 --json
# Revert to the plain static cover-crop:
pandastudio project.auto-reframe --id=$PID --clear=true --json
```

- **The right verb (NOT `project.set-focal-point`)** whenever a landscape source
  with multiple/alternating speakers is cut to 9:16. `set-focal-point` sets ONE
  static point for the whole clip, correct only for a single, stationary
  talking head.
- **Async + needs a renderer** (bundled offline detection). It opens a hidden
  editor for the pass; allow a couple of minutes for a few-minute source. Skips
  clips already matching the canvas aspect (reported under `skipped`).
- **`--zoom`**: omit for the default ADAPTIVE punch-in (each speaker's face
  sized to a consistent fraction of frame); a fixed value (e.g. `1.3`) forces a
  uniform punch-in on every shot.
- **Letterboxed sources are handled automatically.** On its first run per clip
  it detects the real picture area and stores it as `activePictureRect` (0–1
  source fractions). Every crop stays inside it, in preview and export,
  including after you move the video with `project.set-screen-transform`; a
  plain `project.set-crop` on a clip with the rect is kept inside it too
  (`rect: null` = no bars). Expect a slight extra punch-in on letterboxed clips.
- **Set the 9:16 aspect FIRST**, then auto-reframe: the track is computed for
  the canvas aspect and ignored if the aspect later changes (recompute after an
  aspect switch).
- **Preview and export render the crop identically, per frame.** Verify from
  the EXPORTED mp4 across a close-up, a pan, AND a wide two-shot (render-frame
  is fine too, but the export is the source of truth near shot cuts).
- Also in the editor as **"Track speakers"** (Video → Layout).
- v1 limitation: one subject per shot — a *held* two-shot where two people
  banter frames the dominant talker.
