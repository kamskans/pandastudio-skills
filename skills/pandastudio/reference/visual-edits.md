<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Visual edits: zooms, trims, speed, crop, webcam/podcast layouts


## Visual edits — zooms, trims, speed, annotations, style

```bash
# Zoom into a UI element from t=5s for 1.5s
pandastudio project.add-zoom --id=$ID --atMs=5000 --durationMs=1500 \
  --depth=4 --focusX=0.3 --focusY=0.5

# Cursor-follow zoom: the camera pans to track the mouse for the whole
# region (Screen-Studio style). Screen recordings only — needs cursor
# telemetry. focusX/focusY are ignored. Great for walkthroughs where the
# action moves around the screen.
# TIP: give follow-cursor zooms at least ~2500ms. There's a ~0.5s zoom-in
# ramp and a ~1s zoom-out ramp, so anything shorter holds for almost no time
# and reads as "zoom in then immediately out" — the follow never shows. 2.5-4.5s
# is the sweet spot.
pandastudio project.add-zoom --id=$ID --atMs=5000 --durationMs=8000 \
  --depth=3 --followCursor=true
# Toggle follow on an existing zoom:
pandastudio project.update-region --id=$ID --regionType=zoom \
  --regionId=zoom-1 --followCursor=true

# Cut a section directly (without going through transcript)
pandastudio project.add-trim --id=$ID --startMs=12000 --endMs=15000

# Fast-forward a setup step (startMs/endMs are SOURCE ms, like trims)
pandastudio project.add-speed --id=$ID --startMs=8000 --endMs=20000 --speed=2
# Timelapse a long boring stretch (an install, a render, a loading screen)
pandastudio project.add-speed --id=$ID --startMs=60000 --endMs=240000 --speed=40
# --speed takes any value from 0.25 to 100 (0.01 steps). Regions faster than 4
# play SILENT in preview and export (sped-up speech is noise); keep narration
# spans at 4 or below. Retime or change speed later with project.update-region
# --regionType=speed --regionId=speed-1 --speed=8.

# Drop a text annotation. x/y are the box's TOP-LEFT corner in % of the
# frame, NOT its centre (width/height are % too; defaults w=30 h=20; omit
# both x and y to centre the box). To centre a box of width w: x = 50 - w/2,
# or pass --anchor=center and give the centre (add-annotation only;
# update-region x/y stay top-left). Keep x + width <= 100 and y + height <= 100;
# the result returns the resolved top-left `position`, `size` and `warnings`
# when the box runs past an edge: fix those, then check with render-frame.
pandastudio project.add-annotation --id=$ID --startMs=2000 --endMs=4000 \
  --type=text --text="Look here →" --x=35 --y=30 --width=30 --height=12
# Same box, by its centre
pandastudio project.add-annotation --id=$ID --startMs=2000 --endMs=4000 \
  --type=text --text="Look here →" --anchor=center --x=50 --y=36 --width=30 --height=12
# Annotations always draw over the presenter: a title that goes BEHIND the
# person is a motion graphic + project.set-overlay-mask --behindPerson=true.

# Switch aspect ratio (incl. 9:16 for Shorts)
# A camera-only project whose source aspect differs from the new ratio is auto
# cover-cropped (centered) to FILL the frame, so a 16:9 webcam in a 9:16 project
# no longer letterboxes. (Editor adds face-aware centering on open; the headless
# crop is centered. Screen recordings keep their letterbox + padding.)
pandastudio project.set-aspect-ratio --id=$ID --ratio=9:16

# Apply cinematic style. shadowIntensity is the slider's percentage (0-100);
# a 0-1 fraction works too. App versions before 1.94 stored the number as given
# and rendered 30 as a full-strength black wash over the wallpaper — pass 0.3
# there, or upgrade (1.94+ normalizes, and heals projects written earlier).
pandastudio project.set-style --id=$ID --padding=40 --shadowIntensity=30 \
  --borderRadius=20 --motionBlurAmount=15

# Enlarged custom cursor (screen recordings only — draws a bigger cursor that
# tracks the captured cursor telemetry). 0 = off, ~1.5 = noticeably bigger.
pandastudio project.set-style --id=$ID --cursorScale=1.5
# Smart hide for that cursor: fade out after 2 s still, back in before it moves;
# optionally also while a zoom is in. Both default off.
pandastudio project.set-style --id=$ID --cursorHideIdle=true --cursorIdleSeconds=2
pandastudio project.set-style --id=$ID --cursorHideDuringZoom=true

# Main-video FRAME: border ring + circle (PROJECT-LEVEL, Video panel -> Frame).
# In a CAMERA-ONLY recording the camera IS the main video, so this (not
# set-webcam-style) is how it gets a border or a circle, including inside
# cam-left-portrait / cam-right-portrait card sections, where a ring looks best.
pandastudio project.set-style --id=$ID --borderWidth=6 --borderColor="#ffffff"
pandastudio project.set-style --id=$ID --shape=circle        # true circle, centred in the card
pandastudio project.set-style --id=$ID --resetFrame=true     # clear shape + border
# shape: rounded (borderRadius corners, default) | circle. borderWidth 0-40 px
# at a 1080p reference (scales with export size); the ring sits INSIDE the edge
# and the shadow follows it. Draws only while the video is a card (padding > 0,
# a cam-* card, a corner layout); never when the video fills the canvas, so a
# camera-only video cutting between full-frame and card sections shows the ring
# on the cards only. Podcast guest tiles (3-4 person grids) are not framed.

# Pick a wallpaper
pandastudio project.set-wallpaper --id=$ID --wallpaper=gradient-night

# Reframe the main recording (crop, all values normalized 0-1)
pandastudio project.set-crop --id=$ID --x=0.1 --y=0.05 --width=0.8 --height=0.9

# Move / scale the main video INSIDE the frame (drag-resize equivalent).
# Different from set-crop: crop trims source pixels; this repositions and
# zooms the WHOLE video within the frame. scale>1 zooms in (overflow clipped
# to the canvas), scale<1 shrinks it so the wallpaper shows around it. x/y
# shift the center as a fraction of the canvas (0 = centered). Global; preview
# and export match. Pass --reset (or no x/y/scale) to clear it.
pandastudio project.set-screen-transform --id=$ID --scale=1.4 --y=-0.05
pandastudio project.set-screen-transform --id=$ID --reset

# Face centering (auto-reframe). When a clip is cover-cropped — a 9:16 fill or a
# top/bottom designed-segment band — the crop centers geometrically by default,
# which can slice a high-framed face. A FOCAL POINT (source-normalized 0-1) biases
# the cover-crop toward the face so it stays in frame. The EDITOR auto-detects this
# for vertical talking-head clips on open, so usually you don't touch it. To set it
# explicitly (e.g. building a short fully headless), or to override:
pandastudio project.set-focal-point --id=$ID --x=0.5 --y=0.32   # all clips
pandastudio project.set-focal-point --id=$ID --clipId=clip-2 --x=0.6 --y=0.3
pandastudio project.set-focal-point --id=$ID --clear            # back to centered

# Webcam overlay — preset or manual position (PROJECT-LEVEL)
pandastudio project.set-webcam-layout --id=$ID --preset=picture-in-picture
# presets: none | picture-in-picture | vertical-stack | side-by-side | podcast
#          | podcast-host-full | podcast-guest-full
#   podcast = two co-equal speaker tiles (host=mediaPath, guest=webcamPath).
#   podcast-host-full  = ONLY speaker 1 (host) full-frame.
#   podcast-guest-full = ONLY speaker 2 (guest) full-frame.
#   Auto-selected ("podcast") when ingesting a podcast recording; set it manually
#   to convert a screen+camera clip into the two-speaker composite.
pandastudio project.set-webcam-layout --id=$ID --cx=0.85 --cy=0.85 --scale=0.35
pandastudio project.set-webcam-layout --id=$ID \
  --cropX=0 --cropY=0.1 --cropWidth=1 --cropHeight=0.8  # remove letterbox bars

# Camera APPEARANCE — shape / border ring / shadow (PROJECT-LEVEL). Complements
# set-webcam-layout (which does position/size/crop). Fields merge; reset clears.
pandastudio project.set-webcam-style --id=$ID --shape=circle          # round camera bubble
pandastudio project.set-webcam-style --id=$ID --shape=rounded --cornerRadius=24
pandastudio project.set-webcam-style --id=$ID --borderWidth=4 --borderColor="#ffffff"
pandastudio project.set-webcam-style --id=$ID --shadow=0.6            # 0=none, omit=preset default
pandastudio project.set-webcam-style --id=$ID --mirror=true           # flip the camera (selfie view)
pandastudio project.set-webcam-style --id=$ID --reset=true            # back to preset defaults (clears mirror too)
# shape: auto (preset radius, default) | rectangle | rounded | circle.
# Px values are at a 1080p reference and scale with the export resolution.
# Applies to the camera tile in pip / side-by-side / vertical-stack; podcast
# participant grids keep their co-equal tile design.
# --mirror: the live camera bubble while recording is mirrored but the recorded
# camera is not, so users sometimes say "my camera looks flipped / wrong way
# round" after recording. --mirror=true flips the camera tile in preview,
# render-frame and export (--mirror=false undoes it). Default off. Never applied
# to podcast layouts (those tiles are remote guests) or to a camera-only
# video's main track. Don't mirror when the user holds up text or a product to
# the camera: it would read backwards.

# Attach a CAMERA VIDEO recorded or generated elsewhere (v1.89.3+) to a clip, so
# it plays as the camera layer of a Screen + camera clip. Same length as the clip
# is ideal (the result warns when they differ). webcam='' removes it.
# The camera layer plays MUTED. --audio=auto (default) muxes the camera's sound
# into a copy of the main video when that video is silent (camera = always,
# keep = never); re-run transcript.transcribe when usedCameraAudio is true.
pandastudio project.set-clip-webcam --id=$ID --clipIndex=0 --webcam="/abs/path/presenter.mp4"
# Then style it with set-webcam-layout / set-webcam-style below. In the editor:
# Video tab -> Camera video -> Add camera video.

# Camera CARD shapes (v1.89.3+). A picture-in-picture tile is square unless the
# camera is cropped: then the tile takes the crop's shape (clamped 9:16..16:9),
# so a tall crop is a portrait card, never a squashed square.
pandastudio project.set-webcam-layout --id=$ID --preset=picture-in-picture \
  --cropX=0.3 --cropY=0 --cropWidth=0.4 --cropHeight=1 --cx=0.17 --cy=0.5 --scale=1.8
#   -> tall camera card on the left third over a full-bleed screen
# Side-by-side is adjustable too: cx < 0.5 = camera on the LEFT, scale 0.4-1 =
# card height as a fraction of the screen panel, cy = where that card sits.
pandastudio project.set-webcam-layout --id=$ID --preset=side-by-side --cx=0.2 --cy=0.5 --scale=0.7

# Move or resize the camera and/or screen for ONE SECTION only (custom
# clip-transform). Outside the window the project layout applies, with a smooth
# blend at the edges. Works on screen + camera recordings.
pandastudio project.add-clip-transform-region --id=$ID --startMs=12000 --endMs=20000 \
  --preset=custom --webcamCx=0.83 --webcamCy=0.5 --webcamScale=2.2      # camera card jumps right and grows
pandastudio project.add-clip-transform-region --id=$ID --startMs=30000 --endMs=36000 \
  --preset=custom --screenScale=0.8 --screenX=0.08                        # shrink the screen for a beat
# Any section can also crop the camera differently. With a portrait card crop on
# the project, a full-frame talking beat needs the whole camera frame back:
pandastudio project.add-clip-transform-region --id=$ID --startMs=40000 --endMs=48000 \
  --preset=layout-guest-full --webcamCropX=0 --webcamCropY=0 --webcamCropWidth=1 --webcamCropHeight=1

# FULL-FRAME CAMERA beats on a screen + camera clip (v1.89.4+). layout-guest-full
# takes cameraFit: fill (default) = the camera fills the frame with no card
# border, shadow or rounding; centered = a large centred camera over a flat
# colour with the screen hidden. Change an existing section with update-region.
pandastudio project.add-clip-transform-region --id=$ID --startMs=52000 --endMs=58000 \
  --preset=layout-guest-full --cameraFit=centered --backgroundColor="#F4F0E8"
pandastudio project.update-region --id=$ID --regionType=clip-transform --regionId=ctr-2 --cameraFit=fill
# In the editor: select the section, Camera full -> Fill the frame / Centred on a colour.

# KEEP THE FACE CENTRED in the camera card (v1.89.4+). Detects the presenter's
# face through the camera video and stores a smoothed track that preview,
# render-frame and export all follow, so a tight card crop never shows an empty
# wall when they lean. ASYNC: returns { jobId }; job.wait. About 4s per 30s of
# camera. Re-run after replacing the camera video. clear=true turns it off.
JOB=$(pandastudio project.center-camera-on-face --id=$ID --json | jq -r .data.jobId)
pandastudio job.wait --id=$JOB --json
pandastudio project.center-camera-on-face --id=$ID --clear=true
# In the editor: Video tab -> Layout -> Keep face centred.

# PER-SECTION podcast layout: a different layout for each clip (section). Use
# this to cut to whoever is talking. Split first, then set each section.
pandastudio project.set-clip-layout --id=$ID --clipId=clip-2 --preset=podcast-host-full
# preset: podcast | podcast-host-full | podcast-guest-full | picture-in-picture
#         | side-by-side | vertical-stack | none (none clears → inherit project)

# PER-CLIP style: override padding / roundness / shadow (each 0–100) for ONE
# clip, so clips in a multi-clip project can be framed differently — e.g. a
# full-bleed camera clip (padding/borderRadius/shadow 0) next to a padded,
# rounded podcast clip. Project-level equivalents are project.set-style.
# Omit a field to leave it; pass null to clear an override (inherit project).
pandastudio project.set-clip-style --id=$ID --clipId=clip-1 \
  --padding=0 --borderRadius=0 --shadowIntensity=0
pandastudio project.set-clip-style --id=$ID --clipId=clip-2 --padding=18 --borderRadius=24

# Speaker-driven editing (podcast): transcript.get tags every word with a
# `speaker` field — "host" (speaker 1 / mediaPath) or "guest" (speaker 2 /
# webcamPath). Read those spans, split the clip where the active speaker
# changes, then set each section's layout to that speaker's full-frame:
#   1) pandastudio transcript.get --id=$ID --format=words   # words carry speaker
#   2) pandastudio project.split-clip ... at each speaker-change boundary
#   3) pandastudio project.set-clip-layout --clipId=<section> --preset=podcast-host-full
#      (or podcast-guest-full when the guest is talking)

# Podcast guest/host sync nudge — when the two speakers are slightly out of
# sync. offsetMs shifts the guest vs the host (positive delays guest, 0 clears).
# Honored in preview AND export (guest video + audio move together).
pandastudio project.set-webcam-offset --id=$ID --clipId=clip-1 --offsetMs=120

# Update any placed region in-place (patch only what changes)
pandastudio project.update-region --id=$ID \
  --regionType=zoom --regionId=zoom-1 --depth=2 --focusX=0.6
pandastudio project.update-region --id=$ID \
  --regionType=annotation --regionId=ann-1 --text="Updated text" --y=20
pandastudio project.update-region --id=$ID \
  --regionType=fx --regionId=fx-1 --opacity=0.5 --endMs=5000
pandastudio project.update-region --id=$ID \
  --regionType=audio-overlay --regionId=audio-1 \
  --startMs=2000 --endMs=15000 --sourceStartMs=4000 --volume=0.55
# regionType: zoom | trim | speed | annotation | fx | overlay | audio-overlay

# Duplicate a placed region (all settings, new id), right after the original
pandastudio project.duplicate-region --id=$ID --regionType=annotation --regionId=ann-1
# ...or at a given start (EDITED ms; SOURCE ms for --regionType=speed)
pandastudio project.duplicate-region --id=$ID --regionType=zoom --regionId=zoom-1 --atMs=42000
# Zooms and speed regions never overlap their own kind: the copy moves to the
# next free gap (result.shiftedMs) or the call fails if none fits. Not for trims.

# Export defaults (pre-fills the Export dialog; CLI export.start uses its own --quality)
# PandaStudio is a video-only exporter; format is always mp4.
pandastudio project.set-export-settings --id=$ID --quality=source --format=mp4
```


## Clips: add, move, split, insert, remove, delete

- `project.new --withMedia='["/a.mp4","/b.mp4"]'` creates a project pre-loaded;
  clip durations are FFmpeg-probed.
- `project.duplicate --id` (or `--path`) makes an EXACT copy (every edit,
  transcript, aspect ratio) with a fresh id, a `<name> (copy)` name and a new
  `.pandastudio` file; media is shared, the source untouched. Returns
  `{ id, path, name }`. Use to try a variant edit.
- `add-clip` (`--atIndex=0` prepends), `move-clip`, `split-clip`, `remove-clip`
  all carry every region (trims, speeds, zooms, overlays, captions, anchors)
  with the clip it sits on; nothing is dropped by a move.
- **Insert mid-recording = split, then add:** `timeline.edited-to-source
  --editedMs=<playhead>` returns `clipId` + `clipSourceMs`; `project.split-clip
  --clipId=<clipId> --atSourceMs=<clipSourceMs>` returns `rightClipIndex`;
  `project.add-clip --media=<file> --atIndex=<rightClipIndex>`. `split-clip`
  never changes the output: the left half ends at the split point, the right
  half covers the clip's full media with a head trim over the part the left
  half plays (clips play media from 0; in-points are head trims). Don't delete
  that head trim unless you want the right half to replay the start.
- **`project.delete` is permanent — no trash** and needs the user's
  confirmation in PandaStudio (ask in chat first). By default it removes only
  the project file and KEEPS the source recording; `--deleteRecording=true`
  also deletes the original recording file(s) (irreversible, only when the user
  explicitly asks to delete the footage). Returns `deletedRecordings`.
- `project.read` shapes: clips at `mainTrack.clips[]` (each has
  `sourceDurationMs`, no per-clip `durationMs`; use `clipStates[]` for a
  normalized view), motion-graphic / transition overlays at
  `editor.mediaOverlayRegions[]`, audio overlays at TOP-LEVEL
  `project.audioOverlays[]` (not under `editor`), plus `aspectRatio`,
  `editedDurationMs` (post-trim), `sourceDurationMs`, `totalTrimmedMs`,
  `trimCount`.

## Smart cursor hide (drawn cursor, screen recordings)

`project.set-style --cursorHideIdle=true [--cursorIdleSeconds=2]` fades the
enlarged cursor (`cursorScale > 0`) out after it has been still for N seconds
(0.5-10, default 2) and back in just before it moves; clicks count as activity.
`--cursorHideDuringZoom=true` also fades it out while a zoom is in (it follows
the zoom's ease). Both default OFF, including on new projects. Idle time is
measured on the EDITED timeline: a trimmed pause doesn't count, a speed-up
shortens it, and a cut where the cursor jumps shows it again. Preview,
render-frame and export fade on the same frames. Use idle-hide for
talking-over-a-screen videos where the cursor sits parked; leave
hide-during-zoom off when a zoom follows the cursor and the viewer needs to see
what it points at. UI: Video tab > Cursor size box.

## Focus regions: spotlight, blur, pixelate (v1.50.0+)

`project.add-spotlight --atMs=<ms> --durationMs=<ms> [--kind=spotlight|blur]` —
a focus rect over the video. `kind=spotlight` (default) DIMS everything outside
the rect; `kind=blur` BLURS everything inside it (hide an email, username or
other sensitive detail). Rect: `--x --y --width --height` as 0..1 fractions of
the video (default a centred half-size box, `x=y=0.25 width=height=0.5`). `--roundness` (px corner radius,
default 16), `--feathering` (px soft edge, default 12; alias `--feather`).
Spotlight: `--maskOpacity` 0..1 (surround darkness, default 0.6). Blur:
`--blurAmount` px (default 12). The rect tracks content through zooms. `atMs`
is EDITED time (no anchor arg).

- **Edit / delete (v1.85.0):** `project.update-spotlight --regionId=<id>
  [--startMs --endMs --kind --x --y --width --height --roundness --feathering
  --maskOpacity --blurAmount --style --pixelSize --shape --source --keyframes]`
  patches only what you pass; `project.remove-spotlight --regionId=<id>`
  deletes it. Ids: `editor.spotlightRegions[].id`.
- **Pixelate + oval:** `--style=blur|pixelate` (blur kind only; default gaussian),
  `--pixelSize=<px>` (mosaic block at a 1080p reference, 4..120, default 16),
  `--shape=rectangle|ellipse` (ellipse = the oval inscribed in the rect,
  roundness ignored; works for spotlights too). `--style=pixelate` with no
  `--kind` implies `kind=blur`. Reach for PIXELATE for privacy (emails, account
  IDs, API keys, phone numbers, license plates): a gaussian blur on large text
  can stay half-legible; a 16+ px mosaic can't be read back. Use
  `--pixelSize=24`+ on big text, `--shape=ellipse` for faces. Example:
  `project.add-spotlight --atMs=4000 --durationMs=6000 --style=pixelate
  --pixelSize=20 --x=0.1 --y=0.08 --width=0.3 --height=0.05`. The mosaic grid
  is anchored to the region's corner and grows with zooms, so it covers the
  same content in preview, render-frame and export. `project.apply-edit-plan`
  add-blur / add-spotlight ops accept `style`, `pixelSize` and `shape` too.
- **Moving subject (2.0):** `project.track-focus-face --regionId=<spotlightId>`
  makes a blur follow a face; `update-spotlight --source=person` masks a
  spotlight / blur to the presenter. See native-motion.md "Masks".
- **Privacy-blur flow:** `render-frame` → read the PNG → locate the text →
  `add-spotlight --kind=blur` with converted coords (see "Seeing frames"
  below) → `render-frame` again to verify → export.

## Speaker background: blur, remove, studio image, outline (v3.69.0+)

`project.add-background-effect --mode=blur|remove|image [--atMs --durationMs |
--startMs --endMs] [--strength=<px>] [--backgroundImage=<studioId|path>
--backgroundFit=cover|contain] [--outline --outlineWidth --outlineColor
--outlineShadow] [--matteContract --matteFeather] [--anchorSourceMs]` — AI
PERSON SEGMENTATION on the camera video for the region's span, a timeline
region like a zoom (draggable, trimmable, source-anchored, rebases on
trims/speeds). CAMERA / TALKING-HEAD footage only. Bundled on-device model (no
network); preview and export match. Duration defaults to 5000ms.

- `mode=blur` keeps the speaker sharp and blurs everything behind (video-call
  style; `--strength` px sigma at 1080p, default 18).
- `mode=remove` cuts the background away so the project wallpaper shows
  through — pair with `set-wallpaper`, OR a **background-layer media overlay**
  to put an IMAGE/VIDEO behind the speaker (the Shorts look):
  `add-motion-graphic --file=<img/video> --layer=background` (or flip an
  existing overlay with `update-region --regionType=overlay
  --layer=background`; UI: right-click → "Send behind video").
- `mode=image` is the **virtual studio**: it composites the speaker over a
  STUDIO PLATE. Bundled ids: `warm-creator` (default, soft warm key),
  `tech-rgb` (cool RGB rim), `neutral-grey`, `podcast-warm` (warm tungsten),
  `daylight-airy`, `gradient-gel` (magenta/teal), `cinematic-dark` (dramatic
  side) — or an absolute/`file://`/`data:` image. `--backgroundFit=cover`
  (default) or `contain`. Pick a plate lit like the footage. No outline by
  default.
- **Matte tuning:** `--matteContract=<px>` > 0 pulls the person edge INWARD
  (kills a background fringe), < 0 pushes it out (−60..60); `--matteFeather`
  softens it (0..60). Apply to blur AND remove.
- **`--outline`** is a colored keyline + drop shadow hugging the person (the
  VOX magazine-cutout look). **ON by default for `mode=remove`** (v3.80.0); pass
  `--outline=false` for a bare cutout. Off for `mode=blur` unless passed.
  `--outlineWidth` px@1080p (default 36), `--outlineColor` (default `#ffffff`),
  `--outlineShadow` (default true). Pair remove + outline with a cream
  wallpaper for the reference look.
- Regions: `editor.backgroundEffectRegions[]` (`.outline =
  {enabled,width,color,shadow}`); retime/restyle with
  `update-region --regionType=background-effect [--startMs --endMs --mode
  --strength --backgroundImage --backgroundFit --outline --outlineWidth
  --outlineColor --outlineShadow --matteContract --matteFeather]`, delete with
  `remove-region --regionType=background-effect`. Batchable in
  `apply-edit-plan` as `{op:'add-background-effect',atMs,durationMs,mode,...}`.
  2.0: `amount` keyframes fade it in/out (`set-keyframes
  --target=background-effect`).

## Green screen (chroma key)

**On an overlay (v3.153.0, app 1.94+):** `project.set-overlay-chroma-key
--regionId=<overlay-id> [--color=auto|#RRGGBB] [--similarity=0-1]
[--smoothness=0-1] [--spill=0-1] [--enabled=false]` removes a flat-colour
backdrop from an IMAGE or VIDEO media overlay (a presenter on green, stock
footage on green, a UI element on a flat colour). Add the footage first with
`add-motion-graphic --file=<video>`, then key it.

**On the MAIN video or the CAMERA (v3.183.0, app 2.0+):**
`project.set-clip-chroma-key --clipId=<clip-id> [--target=screen|camera]
[--color=auto|#RRGGBB] [--similarity] [--smoothness] [--spill]
[--keepCard=true] [--enabled=false]` keys a main-track clip with the same
keyer. `--target=screen` (default) keys the main video (a camera-only video
filmed on green, a screen recording, a still): the keyed area shows the
project wallpaper / gradient / backdrop and background-layer overlays
(`set-wallpaper` / `add-motion-graphic --layer=background`). `--target=camera`
keys the clip's camera layer (PiP card, side-by-side tile, camera-full
section): the presenter stands directly on the screen recording. A keyed
camera drops its card box (black underlay, shadow, border ring, rounded
corners); `--keepCard=true` keeps the frame with the keyed person inside.

Shared rules:
- `--color` defaults to `auto` on first enable: read from the edges of the
  first shown frame, which keys a real (duller than #00FF00) screen without
  tuning; `detectedColor` says what it found, a `warning` means detection
  failed and the standard chroma green `#00B140` was used.
- Defaults: similarity 0.45 (1 = as far from the key as grey is), smoothness
  0.25 (soft edge), spill 0.5 (removes the green tint on edges). Omitted
  settings keep their current value, so nudge one at a time.
- ALWAYS check with `render-frame`: backdrop patches or a green halo left →
  raise `--similarity` by 0.05; edges, hair or green-ish clothing eaten →
  lower it. `--enabled=false` removes the key.
- Main/camera: key first, then the clip's grade (`set-clip-lut` /
  `set-clip-color`, same `--target`) grades the person only. Works with zooms,
  reframe, layouts, masks, behind-the-person graphics (the person mask is keyed
  too) and speaker background. With the `blur-self` backdrop a keyed main video
  stands on the wallpaper instead. Fade the key with `keyStrength` keyframes:
  `add-motion --target=frame` (main video) or `--target=webcam` (camera), and
  `set-keyframes --target=overlay` for overlays. Not for multi-party podcast
  grids (3+ tiles).
- Stored as `mediaOverlayRegions[].chromaKey`, `mainTrack.clips[].chromaKey`
  and `clips[].webcamChromaKey` (`{color,similarity,smoothness,spill,keepCard?}`).
  Preview, render-frame and export share one keyer.
- NOT `add-background-effect`: that finds a PERSON with AI and can't cut out a
  flat-colour backdrop or a non-person subject.
- UI: select the overlay → Green screen → On; for clips, Video tab → Green
  Screen (Screen / Camera toggle; On detects the colour, Detect re-reads it).

## Seeing frames: render-frame, render-sheet, detect-face, export.verify

- **`project.render-frame --atMs=<ms> [--width=<px>] [--outPath=<png>]
  [--detectFaces=true]`** composites the preview frame at that edited time and
  returns `{ path, width, height, timeMs, maskRect }`. The PNG size does NOT
  depend on the editor window: by default the export resolution capped to a
  1920 long edge (1080x1920 for 9:16, 1920x1080 for 16:9); `--width`
  overrides (64-3840 per edge; pass ~540 to just eyeball). Layout, overlays,
  captions, grade, background effect, the camera at the captured moment
  (custom sections, any transition length) and blur/spotlight regions are
  drawn identically to the preview. `clampedFrom` appears when `atMs` was
  outside the video. A vision model should `read` the returned `path`.
- **Placing a focus region from an image:** `maskRect` is the video content
  rect as 0..1 fractions of the image, the SAME space as spotlight x/y/w/h.
  Convert an image box (ix,iy,iw,ih): `x=(ix-maskRect.x)/maskRect.width`,
  `y=(iy-maskRect.y)/maskRect.height`, `width=iw/maskRect.width`,
  `height=ih/maskRect.height`. Prefer an un-zoomed moment for placement.
- **`project.render-sheet --fromMs --toMs --count --cols`** returns one contact
  sheet (cell k = frame k, row-major): the cheap way to check a whole span or
  an animation.
- **`project.detect-face --fromMs --toMs`** samples the FINAL composited frames
  (layout, crop, zoom applied) and returns `face` {x,y,width,height} as 0..1
  fractions of the frame (the union across samples, so head movement is
  covered) plus `median`. A small picture-in-picture card is searched on the
  camera's own frame and mapped into the output (`via: "camera"`). A range past
  the end is cut to the last frame (`clampedToMs`); samples outside are
  `outOfRange: true`. `found:false` = no visible face. Use it before placing
  anything that must not cover the speaker (keyword pills, side labels).
- **Missing media is reported, not hidden.** If an overlay, motion graphic,
  wallpaper image or screen-share file can't be read, `render-frame` and
  `export.start` still render but return `warnings` naming the files. Tell the
  user; re-generate or re-import before exporting.
- **`export.verify --exportId=<id>`** (async, job.wait; alias `export.check`)
  compares the finished MP4 with the editor preview: picture length vs the
  edit, sound present and in step, and editor-vs-export frame pairs at even
  moments plus inside every layout section. Read `summary` and look at
  `sheetPath` (editor left, export right, red outline = difference) before
  telling the user the export is good. Frames are judged perceptually against
  what H.264 alone does to the same frame; a flagged frame's `reasons` name
  what differs and `worstBlock` where. Same as "Check against editor" on the
  export page.

## Reset and duplicate

- **`project.clear-edits`** — one atomic call wipes every region + audio
  overlays + turns captions off (keeps clips, transcript, aspect ratio;
  `--full=true` also resets LUT/crop/webcam/wallpaper). Use it for "start
  over"; do NOT loop `project.remove-region` (that removes ONE region by
  `--regionType` + `--regionId`).
- **`project.duplicate-region --regionType=<type> --regionId=<id> [--atMs=<ms>]`**
  copies a placed region with every setting under a new id. Types: `zoom |
  speed | annotation | fx | overlay | clip-transform | background-effect |
  spotlight | audio-overlay | motion | adjustment` (not `trim`). Without
  `--atMs` the copy lands right after the original; `--atMs` is EDITED ms,
  except for `speed` (SOURCE ms like trims). A region in a link group (a
  designed segment's panel + camera transform) is duplicated with its peers
  into a NEW group. Zooms and speed regions can't overlap their own kind, so
  the copy moves to the first gap that fits (`shiftedMs`) or fails. Returns
  `{ regionId, startMs, endMs, shiftedMs, created[] }`. Same as Cmd/Ctrl+D
  (Cmd/Ctrl+C then V pastes at the playhead).
