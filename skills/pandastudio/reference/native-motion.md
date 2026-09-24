# Native motion: keyframes on footage

Control over the FOOTAGE itself (the main video and media overlays), rendered
identically in the editor preview, `project.render-frame` and the export.
Use this for camera-style moves on real footage. For animated text and
vector graphics keep using motion graphics (`motion.*`, HTML/GSAP).

Always verify after adding motion: `project.render-sheet --fromMs --toMs
--count=8` across the span (or `project.render-frame` at 2 to 3 times inside
it) and look at the frames.

WHEN to reach for each of these is in SKILL.md "Which tool for which moment".

## Names that look alike, and time bases

- In `project.add-motion` and `set-keyframes`, `--target=camera` is the ZOOM
  camera (zoom, focus) and `--target=webcam` is the person's camera card (PiP
  bubble). Everywhere else "camera" means the person's camera layer:
  `set-clip-lut` / `set-clip-color` / `set-clip-chroma-key --target=camera`,
  `add-adjustment --layer=camera`.
- Overlay-scoped verbs take the overlay as `--regionId` (`set-overlay-mask`,
  `set-overlay-chroma-key`, `set-audio-ducking`, the volume-keyframe verbs; the
  older `--overlayId` still works).
- Aliases accepted: `feather` / `feathering`, `rect` / `rectangle`, `ellipse` /
  `oval`, `flat` / `flat-footage`, `holdMs` / `freezeMs`, `audio` /
  `reverseAudio`.
- **SOURCE ms** (a clip's own recording clock, = transcript word times): trims,
  speeds / ramps / reverse `startMs`-`endMs`, `sourceMs`, `anchorSourceMs`, clip
  volume keyframes, `split-clip --atSourceMs`.
- **EDITED ms** (the output timeline): `atMs` / `durationMs` on add-* verbs,
  zooms, overlays, caption moves, motion tracks, adjustment layers,
  `render-frame --atMs`, `preview.seek`.
- Keyframe `timeMs` is ms from its region / overlay start. Convert with
  `timeline.source-to-edited` / `timeline.edited-to-source`.

## Keyframes

A keyframe is `{ timeMs, x?, y?, scale?, rotation?, opacity?, easing?, bezier? }`.

| Field | Unit / range | Meaning |
|---|---|---|
| `timeMs` | ms, >= 0 | Time from the START of the motion region or overlay (not project time). |
| `x` | fraction of canvas WIDTH, -5..5 | Horizontal offset of the centre. `0.1` = 10% of the width to the right. |
| `y` | fraction of canvas HEIGHT, -5..5 | Vertical offset of the centre. `0.1` = 10% of the height down. |
| `scale` | multiplier, 0.05..20 | About the centre. `1` = unchanged, `1.3` = 30% bigger. |
| `rotation` | degrees clockwise, -3600..3600 | About the centre. |
| `opacity` | 0..1 | Multiplies the element's own opacity. |
| `easing` | `linear` \| `hold` \| `ease-in` \| `ease-out` \| `ease-in-out` (default) \| `back` \| `elastic` \| `bezier` | Curve from THIS keyframe to the next keyframe that sets the same property. `hold` jumps at the next keyframe. `back` and `elastic` overshoot. |
| `bezier` | `[x1, y1, x2, y2]`, x in 0..1 | CSS cubic-bezier points, only with `easing: "bezier"`. |

Rules:
- Each keyframe sets only the properties it names; a property keeps its own
  keyframe track (so a fade can finish before a move ends).
- Before the first keyframe of a property its first value holds; after the
  last, the last value holds. A property no keyframe sets stays neutral.
- Unknown fields, out-of-range values, unknown easings and two keyframes at
  the same `timeMs` are REJECTED with the allowed list; nothing is clamped
  silently.
- **Entrance / exit = `project.set-animation` or `--enter` / `--exit`, never
  keyframes.** Keyframes are only for motion `set-animation` can't express: a
  move in the middle of the span, a path, a dodge, a pull-back.

## Main video: motion regions

```bash
# Slow push-in over a 4 s beat (Ken Burns on footage)
pandastudio project.add-motion --id=$ID --atMs=12000 --durationMs=4000 --preset=push-in --json

# Custom: punch in with a small tilt, then settle back
pandastudio project.add-motion --id=$ID --atMs=30500 --durationMs=1500 --keyframes='[
  {"timeMs":0,"scale":1,"rotation":0,"easing":"back"},
  {"timeMs":250,"scale":1.35,"rotation":-2,"easing":"ease-in-out"},
  {"timeMs":1500,"scale":1,"rotation":0}
]' --json
# → { regionId: "motion-1" }
```

- `atMs` / `durationMs` are EDITED (output) ms, like zooms. Returns `regionId`.
- Presets: `push-in`, `pull-out` (whole duration), `slide-in-left|right|up|down`,
  `fade-in`, `fade-out`, `spin-in`, `pop-in`, `shake` (entrances take the first
  600 ms, then hold).
- Pin to a word with `--anchorSourceMs=<word.startMs>` (source ms), same as zooms.
- Timing is OUTPUT time: keyframe offsets and the region length never follow
  the source clock. A trim or speed change before the region moves the region
  with its content; its length and keyframe offsets stay the same.
- Outside every motion region the video is untouched. Overlapping motion
  regions combine (offsets and rotations add, scales and opacities multiply).
- Stacks with everything that already moves the video: zooms scale the result,
  layout transforms and the screen transform are the resting position the
  keyframes move from.
- Opacity < 1 shows the wallpaper / background behind the video.
- The camera tile (webcam) is not affected; the cursor overlay moves with the video.
- Crop is not keyframable.

Edit afterwards:
- `project.set-keyframes --target=main --regionId=motion-1 --keyframes='[...]'` (replace the track)
- `project.add-keyframe --target=main --regionId=motion-1 --timeMs=800 --scale=1.2` (same time merges)
- `project.remove-keyframe --target=main --regionId=motion-1 --keyframeId=kf-2`
- Retime: `project.update-region --regionType=motion --regionId=motion-1 --patch='{"startMs":…,"endMs":…}'`
- Remove / duplicate / clear: `project.remove-region|duplicate-region --regionType=motion`, `project.clear-edits`.

## Media overlays (images, videos, motion graphics)

```bash
# Mid-span move (set-animation can't do this): at 2 s the card slides left to
# clear a graphic, at 5 s it slides back. Its entrance and exit stay on
# project.set-animation (below).
pandastudio project.set-keyframes --id=$ID --target=overlay --regionId=overlay-3 --keyframes='[
  {"timeMs":2000,"x":0,"easing":"back"},
  {"timeMs":2400,"x":-0.25},
  {"timeMs":5000,"x":-0.25,"easing":"ease-in-out"},
  {"timeMs":5400,"x":0}
]' --json
```

- `timeMs` is ms from the overlay's `startMs`, so moving the overlay moves its animation.
- x/y move the overlay's box by a fraction of the canvas; scale and rotation
  are about the box centre; opacity multiplies the overlay's `opacity`.
- A keyframed overlay can still be dragged in the editor (it moves the resting
  position) but not resized; `project.set-keyframes ... --keyframes='[]'` clears it.

## Enter / exit transitions (annotations, emoji, lower thirds, overlays)

Every annotation (text, image, arrow) and every media overlay (image, video,
motion graphic, animated emoji, lower third) can have an ENTER and an EXIT
transition next to its keyframed move:

```bash
# An arrow that pops in and fades out (x/y = the box's top-left corner in %,
# or its centre with --anchor=center)
pandastudio project.add-annotation --id=$ID --type=figure --arrowDirection=down-left \
  --startMs=4000 --endMs=7000 --x=60 --y=20 --width=12 --height=15 --enter=pop --exit=fade --json
# -> { annotationId: "ann-2" }
# Change them later (omit one to keep it, none clears it)
pandastudio project.set-animation --id=$ID --regionType=annotation --regionId=ann-2 \
  --enter=slide-left --enterMs=500 --exit=none --json
# Emoji: pop in, fade out
pandastudio project.add-emoji --id=$ID --emoji=fire --atMs=4200 --durationMs=2500 --enter=pop --exit=fade
# A lower third that slides away at its end (lt-* designs animate their own entrance)
pandastudio project.set-animation --id=$ID --regionType=overlay --regionId=overlay-4 \
  --exit='{"preset":"slide-left","durationMs":450,"easing":"ease-in"}' --json
# Plus a move in between: keyframes on an annotation
pandastudio project.set-keyframes --id=$ID --target=annotation --regionId=ann-2 \
  --keyframes='[{"timeMs":0,"rotation":0},{"timeMs":3000,"rotation":12,"scale":1.15}]' --json
```

| Preset | Enter | Exit |
|---|---|---|
| `fade` | fades in | fades out |
| `pop` | grows from 50% with a small overshoot, fading in | shrinks to 50%, fading out |
| `rise` | moves 4% of the frame height up into place, fading in | sinks 4% down, fading out |
| `zoom` | shrinks from 140% into place, fading in | grows to 140%, fading out |
| `slide-left` / `slide-right` | comes in from the left / right EDGE of the frame | leaves to that edge |
| `slide-up` / `slide-down` | comes up from the BOTTOM / down from the TOP edge | leaves to that edge |

- Shape: a preset name, or `{"preset","durationMs","easing","bezier"}`.
  `durationMs` 50..10000 (default 400). Default curve: ease-out for an enter
  (back, i.e. overshoot, for pop), ease-in for an exit. Any easing name works.
- The enter plays from the layer's start and the exit ends at its end, so
  moving, trimming or resizing the layer keeps them on its edges (keyframes are
  relative to the start only). When the layer is shorter than enter + exit,
  both shrink in proportion.
- Transitions and keyframes combine (offsets and rotations add, scales and
  opacities multiply). Text and arrows stay sharp at any scale.
- In the editor: select the annotation or overlay; "Enter and exit" and the
  keyframe list are in its inspector.

## Captions that move

Captions are their own text system (template, words, highlight). Only their
placement box is keyframable: a caption placement track moves and resizes
them over a span of the timeline, e.g. up and out of the way while a lower
third or a camera card is on screen.

```bash
# Lift the captions clear of a lower third for exactly its span, then back
pandastudio caption.move --id=$ID --whileRegionId=overlay-4 --positionY=62 --json
# -> { regionId: "motion-3", startMs, endMs }
# Or a span in edited ms, smaller and shifted right
pandastudio caption.move --id=$ID --atMs=12000 --durationMs=5000 --positionY=20 --size=0.8 --offsetX=0.15 --json
# Full control: a captions track with your own keyframes
pandastudio project.add-motion --id=$ID --target=captions --atMs=3000 --durationMs=4000 --keyframes='[
  {"timeMs":0,"positionY":85,"easing":"ease-in-out"},{"timeMs":400,"positionY":60},
  {"timeMs":3600,"positionY":60,"easing":"ease-in-out"},{"timeMs":4000,"positionY":85}]' --json
```

- `positionY` is % of the frame height from the top (the `caption.set-style
  positionY` unit), `offsetX` a fraction of the width, `size` a multiplier
  about the caption's anchor. A field no keyframe sets keeps the caption
  setting; outside the span the caption settings apply.
- `caption.move` eases over `transitionMs` (default 300) at each end; default
  target height is 20 points above the current setting.
- Edit with `project.set-keyframes --target=main --regionId=<id>`, retime with
  `project.update-region --regionType=motion`, remove with
  `project.remove-region --regionType=motion`. In the editor: "+" menu →
  "Move captions (keyframes)", then the Caption placement inspector.

## Still image clips and Ken Burns

A still can sit on the main track as a clip of its own (kind `image`): it is
shown for its duration, drawn by the engine from the full-resolution image,
silent (it contributes silence to the mix) and without a transcript. Trims,
speed regions and freeze frames keep their usual meaning (they change how long
it shows); it can't be reversed (the verbs refuse). Remove-silences never cuts
a still. Split, move and remove work like on any clip.

```bash
# A still as a clip: fills the frame (cover crop, no padding / corners / shadow) for 4 s
pandastudio project.add-clip --id=$ID --media=/abs/title.png --durationMs=4000 --json
# -> { clipId: "clip-3", kind: "image", startMs, endMs }
pandastudio project.set-clip-duration --id=$ID --clipId=clip-3 --durationMs=6000 --json

# Ken Burns beat: an image clip + a keyframed move over it (a screen motion region)
pandastudio media.image-to-video --id=$ID --imagePath=/abs/beat3.png --durationMs=6400 --zoom=out --pan=left --json
# -> { mode: "native", placement: "clip", clipId: "clip-4", regionId: "motion-4", startMs: 18400, endMs: 24800 }
# Cutaway over existing footage at 30 s (a full-frame image overlay with the move)
pandastudio media.image-to-video --id=$ID --imagePath=/abs/diagram.png --atMs=30000 --durationMs=4000 --json
# The old behaviour, a baked MP4 (no project, or --bake=true)
pandastudio media.image-to-video --imagePath=/abs/beat3.png --durationMs=6400 --json   # -> { mode: "baked", videoPath }
```

- `zoom` in (default) / out / none, `zoomAmount` 0.05..0.4 (default 0.12),
  `pan` none / left / right / up / down (the direction the picture drifts;
  with a zoom the small end starts at 1.1x so there is room to pan, and the
  whole move stays inside the image), `easing` default linear (back /
  elastic are refused: they overshoot and would show the image edge),
  `atIndex` to insert instead of append.
- The move is an ordinary motion region (`scale`, `x`, `y` keyframes) anchored
  to the image clip: it moves with the clip, is removed with it, and
  `project.set-clip-duration` stretches it with the still.
- In the editor: "Append clip ▾ → Add still image…", right-click a still on
  the timeline → "Show for N s".

## Speed ramps

A speed region can EASE into and out of its speed instead of jumping.
Speed regions are in SOURCE ms (like trims); ramp lengths are source ms too.

```bash
# Classic ramp: 1x, ease up to 4x over 0.8 s, cruise, ease back to 1x over 0.8 s
pandastudio project.add-speed --id=$ID --startMs=20000 --endMs=32000 --speed=4 \
  --rampIn='{"durationMs":800}' --rampOut='{"durationMs":800}' --json

# A to B: start at 1x and accelerate into an 8x timelapse over the whole span
pandastudio project.add-speed-ramp --id=$ID --startMs=40000 --endMs=46000 --fromSpeed=1 --toSpeed=8 --json

# Add / change / remove a ramp on an existing region
pandastudio project.update-region --id=$ID --regionType=speed --regionId=speed-2 \
  --patch='{"rampOut":{"durationMs":600,"speed":1,"easing":"ease-out"}}' --json
pandastudio project.update-region --id=$ID --regionType=speed --regionId=speed-2 --patch='{"rampIn":null}' --json
```

| Ramp field | Unit / range | Meaning |
|---|---|---|
| `durationMs` | SOURCE ms, > 0 | How long the ease takes. rampIn + rampOut must fit inside the region. |
| `speed` | 0.25..100, default 1 | Speed at the region edge: where rampIn starts, where rampOut ends. |
| `easing` | `ease-in-out` (default) \| `ease-in` \| `ease-out` \| `linear` \| `hold` \| `back` \| `elastic` | Shape of the change. Overshooting curves are clamped to 0.25..100. |

- `preservePitch` (default true): voices keep their pitch at any speed, same
  as constant speed regions. `false` makes the pitch follow the speed like
  tape (deeper slow, higher fast), in the preview and the export.
- Everything that follows the video retimes automatically: transcript edits,
  captions, anchored zooms / overlays / motion regions, audio, the export.
  The video never goes black at the end: the output simply gets shorter or
  longer by exactly the time the ramp saves or adds.
- Audio above 4x is muted (existing rule), also inside ramps.
- Speed regions may not overlap each other; the verbs reject an overlap.
- Motion keyframes are in OUTPUT time: a ramp under a motion region does not
  change when its keyframes happen.

## Motion tracks: camera, bubble, frame

`project.add-motion` animates more than the main video. `target` picks what
the keyframes drive; each target has its own keyframe fields (unknown fields
are rejected). All use the same easing names.

| target | fields | what it does |
|---|---|---|
| `screen` (default) | `x` `y` `scale` `rotation` `opacity` | moves the main video (presets allowed) |
| `camera` | `zoom` (1..5), `zoomAmount` (0..1), `focusX` `focusY` (0..1) | the zoom camera; replaces zoom regions while it runs |
| `webcam` | `cx` `cy` (0..1 centre), `size` (0.3..3), `keyStrength` (0..1) | the picture-in-picture bubble; `keyStrength` fades the camera's green-screen key |
| `frame` | `padding` `radius` (0..100), `shadow` (0..1), `keyStrength` (0..1) | the frame style of the main video; `keyStrength` fades the main video's green-screen key |
| `captions` | `positionY` (0..100 % from the top), `offsetX` (-0.5..0.5 of the width), `size` (0.3..3) | where the captions sit and how big they are (see "Captions that move") |

A field no keyframe sets keeps the project's own value.

```bash
# Move the camera bubble out of the way while a graphic shows, then back
pandastudio project.add-motion --id=$ID --target=webcam --atMs=12000 --durationMs=6000 \
  --keyframes='[{"timeMs":0,"cx":0.85,"cy":0.8},{"timeMs":600,"cx":0.15,"cy":0.8},{"timeMs":5400,"cx":0.15,"cy":0.8},{"timeMs":6000,"cx":0.85,"cy":0.8}]' --json
# Pull back to reveal the background: padding 0 -> 40 and rounder corners
pandastudio project.add-motion --id=$ID --target=frame --atMs=0 --durationMs=2000 \
  --keyframes='[{"timeMs":0,"padding":0,"radius":0,"easing":"ease-in-out"},{"timeMs":1500,"padding":40,"radius":30}]' --json
# Bake a zoom (or a cursor-follow zoom) into editable camera keyframes
pandastudio project.convert-to-keyframes --id=$ID --regionId=zoom-3 --json
# -> { regionId: "motion-4", keyframes: 4, removedZoomIds: ["zoom-3"] }
```

- Convert to keyframes grows the span to whole zooms (with their 1 s zoom
  out), chains of connected zooms and existing camera tracks, and renders
  the same frames. A plain zoom becomes 4 keyframes on its own curves
  (`smooth-out` in and out); cursor-follow becomes one keyframe per change.
- Easing names: linear, hold, ease-in, ease-out, ease-in-out, back, elastic,
  bezier, ease-in-out-cubic (layout transitions), smooth-out (the zoom
  ramp), smooth-pan (connected zoom pans).
- Layout sections (`add-clip-transform-region`, `update-region
  --regionType=clip-transform`) take `easing` (+ `bezier`) for their
  transition in and out; default ease-in-out-cubic.

## Masks

One mask system: a SOURCE (shape, person, layer) plus `invert`, `feather`
(px at 1080p) and `expand` (-1..1). The engine renders it the same in the
preview and the export.

```bash
# Title text BEHIND the presenter (the person matte cuts it out). The title
# must be a motion graphic (annotations can't go behind): render it, add it,
# mask it, look at it. If the render fails, skip the move; no annotation.
pandastudio motion.generate --templateId=transitions-3d \
  --slots='{"lead":"","emphasis":"FOCUS","trail":""}' --json        # -> { jobId }
pandastudio job.wait --id=<jobId> --json
pandastudio project.add-motion-graphic --id=$ID --fromJob=<jobId> --atMs=62000 --durationMs=3000 --json
pandastudio project.set-overlay-mask --id=$ID --regionId=overlay-3 --behindPerson=true --json
pandastudio project.render-frame --id=$ID --atMs=63500 --json      # the head covers part of it; it still reads
# Wipe reveal: a rectangle mask that grows from the left over 0.8 s
pandastudio project.set-overlay-mask --id=$ID --regionId=overlay-4 --source=shape --x=0 --y=0 --width=1 --height=1 \
  --keyframes='[{"timeMs":0,"width":0,"easing":"ease-out"},{"timeMs":800,"width":1}]' --json
# Track matte: show a video only through a title's letters (luma or alpha)
pandastudio project.set-overlay-mask --id=$ID --regionId=overlay-5 --source=layer --matteOverlayId=overlay-6 --mode=alpha --json
# Privacy blur that follows a face
pandastudio project.add-spotlight --id=$ID --kind=blur --atMs=4000 --durationMs=6000 --json   # -> spotlight id
pandastudio project.track-focus-face --id=$ID --regionId=spotlight-2 --json
# Spotlight on the presenter (dim everything else), no shape needed
pandastudio project.update-spotlight --id=$ID --regionId=spotlight-3 --source=person --json
# Fade a background blur in over 0.6 s
pandastudio project.set-keyframes --id=$ID --target=background-effect --regionId=bgfx-1 \
  --keyframes='[{"timeMs":0,"amount":0},{"timeMs":600,"amount":1}]' --json
```

| Keyframe target | Fields |
|---|---|
| `focus` | x, y, width, height (0..1 of the video), roundness, feathering, blurAmount, pixelSize, maskOpacity |
| `background-effect` | amount (0..1, fade), strength (blur px) |
| `mask` (regionId = overlay id) | x, y, width, height (0..1 of the frame), roundness, feather, expand |
| `overlay` | x, y, scale, rotation, opacity, backdropBlur (glass), keyStrength (chroma key 0..1) |
| `annotation` | x, y, scale, rotation, opacity (text, image and arrow annotations) |

- `project.track-focus-face --regionId=<spotlightId> [--smoothing=0.1..1] [--stepMs] [--padding]`
  makes a focus region follow a face; lower `--smoothing` is steadier
  (default 0.5).
- `person` masks need the person matte (the same one speaker background
  effects use); on a screen recording without a person they hide nothing.
- A track matte hides the overlay while its matte overlay isn't showing.
- `update-region --regionType=overlay|motion --motionBlur=0..1` adds motion
  blur while that layer moves (per layer; the zoom camera keeps the
  project's motion blur setting).

## Blend modes

Media overlays (video, image, motion graphics), FX overlays and adjustment
layers take a `blendMode`: `normal` `multiply` `screen` `overlay` `softLight` `hardLight` `darken` `lighten` `colorDodge` `colorBurn` `difference` `exclusion` `add` `hue` `saturation` `color` `luminosity` (`add` = linear dodge; kebab-case like
`soft-light` is accepted and stored as `softLight`). The math is the W3C
Compositing and Blending spec (premultiplied alpha, sRGB, like CSS
`mix-blend-mode`) and is the same in the preview, `project.render-frame` and
the export.

```bash
# A light leak / flare shot on black: screen drops the black
pandastudio project.add-motion-graphic --id=$ID --file="$LEAK" --atMs=4000 --durationMs=3000 --blendMode=screen --json
# Paper texture over the video: multiply keeps the dark fibres, drops white
pandastudio project.update-region --id=$ID --regionType=overlay --regionId=overlay-3 --blendMode=multiply --json
# Remove the blend again
pandastudio project.update-region --id=$ID --regionType=overlay --regionId=overlay-3 --blendMode=normal --json
# FX in a different mode
pandastudio project.add-fx --id=$ID --fxId=light-leak --atMs=0 --durationMs=5000 --blendMode=add --opacity=0.6 --json
```

| Want | Mode |
|---|---|
| drop a black background (leaks, flares, particles, glows) | `screen` (softer) or `add` (hotter) |
| drop a white background (ink, paper, line art) | `multiply` |
| lay a texture or colour wash INTO the footage | `overlay`, `softLight` (gentler), `hardLight` (stronger) |
| tint the footage with a graphic's colour, keep its light | `color` (hue + saturation) or `hue` |
| invert-style graphic that reads on any footage | `difference`, `exclusion` |

- Opacity, and opacity keyframes, apply to the overlay BEFORE it blends, so a
  fade in / out of a blended overlay works as expected.
- An overlay used as another overlay's track matte keeps shaping the matte; its
  own blend mode does not apply to the matte.
- FX default to `screen`; overlays and adjustment layers to `normal` (stored
  as no field). `normal` or `null` in `update-region` clears an overlay's or
  adjustment layer's mode; an FX always has one.
- Adjustment layer + blend mode = the adjusted picture blended back over the
  unadjusted one (After Effects' adjustment-layer blend): `--blur=20
  --blendMode=screen` is a soft glow, `--contrast=0.5 --blendMode=overlay`
  punch, `--saturation=-1 --blendMode=luminosity` takes only the new
  brightness. The fade and `amount` become the blend opacity instead of
  scaling the effects. A mode with no effects blends the picture over itself
  (`multiply` = deeper, `screen` = brighter).
- Verify with `project.render-frame` inside the span.

## Adjustment layers

An adjustment region applies ONE effect stack to everything beneath it over an
edited-time span: wallpaper, main video, camera, overlays, motion graphics,
annotations, FX. Captions, blur/spotlight regions and the watermark stay on
top and are not changed. Overlapping layers stack in start order.

```bash
# Moody flashback: desaturated, cool, vignette, grain, fading in and out
pandastudio project.add-adjustment --id=$ID --atMs=30000 --durationMs=6000 \
  --saturation=-0.6 --temperature=-0.4 --vignette=0.6 --grain=0.4 --fadeInMs=400 --fadeOutMs=400 --json
# → { regionId: "adj-1" }

# Punch it up, then remove the grain
pandastudio project.update-adjustment --id=$ID --regionId=adj-1 --contrast=0.3 --grain=null --json
```

| Effect | Unit / range | Neutral |
|---|---|---|
| `exposure` | photographic stops, -1..1 (+1 = twice as bright) | 0 |
| `contrast` | -1..1 (-1 = flat grey, +1 = double) | 0 |
| `saturation` | -1..1 (-1 = black and white, +1 = double) | 0 |
| `temperature` | -1 cool (blue) .. 1 warm (orange) | 0 |
| `tint` | -1 green .. 1 magenta | 0 |
| `blur` | gaussian blur, px at 1080p, 0..60 | 0 |
| `vignette` | edge darkening, 0..1 | 0 |
| `grain` | animated film grain, 0..1 | 0 |
| `glow` | bloom around bright areas, 0..1 | 0 |
| `chromaticAberration` | red / blue split, px at 1080p, 0..20 | 0 |

- At least one effect must be set; unknown fields and out-of-range values are
  rejected with this list. 0 or null in `update-adjustment` removes an effect.
- `fadeInMs` / `fadeOutMs` scale every effect in and out.
- px values are authored at 1080p and scale with the export size.
- Retime with `project.update-region --regionType=adjustment`, remove with
  `project.remove-region --regionType=adjustment`; duplicate / clear-edits work too.
- One effect library: a clip's grade (`project.set-clip-color` /
  `project.set-clip-lut`, the Color panel) and an adjustment layer are the same
  effects rendered the same way. Keep using `set-clip-color` / `set-clip-lut`
  for a clip's static base grade; use an adjustment layer when the look must
  change over time or cover only part of the timeline.

### Looks, correction, one clip only, keyframes

An adjustment layer can also carry a LUT **look** (`look` = any
`asset.list-luts` id, `lookIntensity` 0..1) and a clip-style **correction**
(`correction='{"brightness":0.1,"contrast":0.3,"saturation":0.2,"warmth":0.1}'`,
each -1..1, or `correctionPreset=flat`).

**Scope.** Without `clipId` the layer changes the whole frame (under captions).
With `clipId` (from `project.read` mainTrack.clips[].id) it changes only that
clip's screen layer, or its camera tile with `layer=camera`, under overlays,
graphics and FX. With `clipId`, `atMs` / `durationMs` default to the whole clip.

**Keyframes** animate any value: an effect (`blur`, `saturation`, ...),
`lookIntensity`, or `amount` (0..1, the whole stack's strength). Times are ms
from the layer start, with the same easing names as motion keyframes. A
keyframed value overrides the layer's static one.

```bash
# Fade a teal-orange look in over one clip's first 2 s, then hold it
pandastudio project.add-adjustment --id=$ID --clipId="$CLIP_ID" --look=cinematicTealOrange \
  --keyframes='[{"timeMs":0,"lookIntensity":0,"easing":"ease-in-out"},{"timeMs":2000,"lookIntensity":1}]' --json
# Grade only the camera: fix flat footage
pandastudio project.add-adjustment --id=$ID --clipId="$CLIP_ID" --layer=camera --correctionPreset=flat --json
# One more keyframe (same timeMs merges); clear the animation with keyframes=[]
pandastudio project.add-keyframe --id=$ID --target=adjustment --regionId=adj-2 --timeMs=4000 --amount=0.3 --json
pandastudio project.update-adjustment --id=$ID --regionId=adj-2 --keyframes='[]' --json
```

**Stacking order** inside one layer: correction, look, colour (exposure,
temperature, tint, saturation, contrast), blur, glow, colour fringe, grain,
then the vignette on top. Across layers: the clip's own grade first, then layers
scoped to that clip (start order), then whole-frame layers (start order).

## Freeze frame

```bash
# Hold the frame under the playhead at 12.4 s for 2 s, then carry on
pandastudio project.add-freeze-frame --id=$ID --atMs=12400 --holdMs=2000 --json
# → { regionId: "speed-3", sourceMs: 15120 }

# Freeze on a word (transcript / source time)
pandastudio project.add-freeze-frame --id=$ID --sourceMs=<word.startMs> --holdMs=1500 --json

# Longer hold / remove
pandastudio project.update-region --id=$ID --regionType=speed --regionId=speed-3 --patch='{"freezeMs":3000}' --json
pandastudio project.remove-region --id=$ID --regionType=speed --regionId=speed-3 --json
```

- The hold ADDS `holdMs` to the output. Everything after it that follows the
  video (captions, anchored zooms / overlays / motion / adjustments) moves
  later with its content; music and free audio overlays stay where they are.
- The recording's own audio is silent during the hold (music and SFX overlays
  keep playing). Captions hold on the frozen word.
- Stored as a speed region spanning 1 ms of source with `freezeMs` (so trims,
  remove / duplicate and the time math treat it like any speed region). It
  can't sit inside another speed region or a cut.
- Pair it with an adjustment layer (desaturate, vignette) and a motion region
  (slow push-in) over the hold for the classic "record scratch" beat.

## Reverse playback

`project.add-reverse` plays a span backwards (a rewind beat) in the same
output time it would take forwards.

```bash
# Rewind the last 3 seconds of a demo, silently, at 2x
pandastudio project.add-reverse --startMs=41000 --endMs=44000 --speed=2
# Same span given in edited (output) time, with the audio played backwards
pandastudio project.add-reverse --atMs=30500 --durationMs=3000 --audio=reverse
```

- Span in SOURCE ms (`startMs` + `endMs`, transcript time) or EDITED ms
  (`atMs` + `durationMs`). At least 100 ms of source.
- Constant speed only (`speed` 0.25 to 100, default 1): a reversed span can't
  have ramps, can't be a freeze and can't overlap another speed region.
- Audio: `mute` (default; reversed speech is rarely wanted) or `reverse`.
  Music and SFX overlays are unaffected.
- Stored as a speed region with `reverse: true` (and `reverseAudio`), so
  trims, anchors, remove / duplicate and the timeline treat it like any speed
  region. Toggle later with `project.update-region --regionType=speed` and
  `{ reverse: false }`, change the sound with `{ reverseAudio: "reverse" }`.
- The first preview or export of a reversed span builds a cached reversed
  copy of it (a second or so per 10 s of 1080p); needs the media engine
  (ffmpeg). Captions keep forward order (they follow the words).
- Verify with `project.render-sheet` across the span: the frames run
  backwards.
