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

# Fast-forward a setup step
pandastudio project.add-speed --id=$ID --startMs=8000 --endMs=20000 --speed=2

# Drop a text annotation
pandastudio project.add-annotation --id=$ID --startMs=2000 --endMs=4000 \
  --type=text --text="Look here →" --x=50 --y=30

# Switch aspect ratio (incl. 9:16 for Shorts)
# A camera-only project whose source aspect differs from the new ratio is auto
# cover-cropped (centered) to FILL the frame, so a 16:9 webcam in a 9:16 project
# no longer letterboxes. (Editor adds face-aware centering on open; the headless
# crop is centered. Screen recordings keep their letterbox + padding.)
pandastudio project.set-aspect-ratio --id=$ID --ratio=9:16

# Apply cinematic style
pandastudio project.set-style --id=$ID --padding=40 --shadowIntensity=30 \
  --borderRadius=20 --motionBlurAmount=15

# Enlarged custom cursor (screen recordings only — draws a bigger cursor that
# tracks the captured cursor telemetry). 0 = off, ~1.5 = noticeably bigger.
pandastudio project.set-style --id=$ID --cursorScale=1.5

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
pandastudio project.set-webcam-style --id=$ID --reset=true            # back to preset defaults
# shape: auto (preset radius, default) | rectangle | rounded | circle.
# Px values are at a 1080p reference and scale with the export resolution.
# Applies to the camera tile in pip / side-by-side / vertical-stack; podcast
# participant grids keep their co-equal tile design.

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
#   1) pandastudio transcript.get --id=$ID            # words carry speaker
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

# Export defaults (pre-fills the Export dialog; CLI export.start uses its own --quality)
# PandaStudio is a video-only exporter; format is always mp4.
pandastudio project.set-export-settings --id=$ID --quality=source --format=mp4
```

