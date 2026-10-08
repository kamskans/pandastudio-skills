# 3D camera captions

`project.add-3d-captions` turns one talking-head moment into a short designed
segment where a camera flies between the speaker and the words: it whips in,
trucks between caption groups set at different depths (they slide apart by
parallax), tucks a hero word behind the speaker's head, can wrap a clause in a
ring that turns in front of them, and ends on a staircase of words on a push.
Text steps at 15 fps with motion blur and leads the voice by 0.2 s. It is the
After Effects "3D text camera" look, built from the project's own footage,
person matte and word timings, rendered by the HyperFrames engine and placed
over the range.

## When to use it

- The HOOK of a talking-head video (the first 5-15 s of a Short or a
  long-form intro), or one punchline / thesis line later on. One per Short,
  at most one per 2-3 minutes of long-form. It is a showpiece, not captions.
- The speaker is on camera, full frame, in a locked-off medium shot (head
  about a quarter to a third of the frame height, room beside the head).
- The project is 16:9 (vertical is not supported yet).
- The words are Latin-script (English, Spanish, French, ...).

## When not to

- Plain subtitles: use the caption templates.
- A moving or handheld camera, a walk-and-talk, a zoom inside the range: the
  plate and matte transforms assume a locked shot. The verb measures the
  background and refuses (force=true overrides; the result will look wrong).
- An extreme close-up (the face fills the frame): no room behind or around the
  head. The verb refuses.
- Screen recordings, slides, B-roll: there is no speaker to fly around.
- Over a range that already has zooms, camera cards or graphics: the segment
  replaces the picture there (it is a full-frame render), so those are hidden.
  Place it before adding other moves in that range, or pick a clean range.

## How to call it

1. Transcribe and clean up first (fillers, bad takes, silences): the segment
   is rendered from the edited timeline, so do it AFTER the cuts.
2. Pick the range on the transcript: start and end on a pause between
   sentences, 5-20 s (8-14 s is the sweet spot), 3-5 short clauses.
3. Call:

```
pandastudio project.add-3d-captions --id=<projectId> --startMs=<edited ms> --endMs=<edited ms> \
  [--style=bold|editorial] [--heroWords='["agent","finished cut"]'] [--accent=#F7471B]
pandastudio job.wait --id=<jobId>
```

MCP: `project_add_3d_captions` then `job_wait`.

- `style`: `bold` (default; heavy sans heroes in one saturated accent, light
  sans captions) or `editorial` (italic display-serif heroes, text-serif
  captions, cream with a gold accent). Match the video's look: bold for
  energetic creator / tech content, editorial for calm, premium, essay-style.
- `heroWords`: the words to feature big (one per beat). Default: the most
  stressed content word of each beat from the speech map. Pass them when the
  user named the point of the line or the brand / product name.
- `accent`: the hero colour, e.g. the brand colour (workspace.get-brand).
- `grade` (default true): a film finish over the segment (curves, skin-safe
  saturation, grain). Pass false when the rest of the video is ungraded and
  the jump would show.
- `hideCaptions` (default true): the editor's captions are hidden over the
  range; the segment carries its own words.

It takes a while: a matte bake if the clip has none yet, then about 1.5
minutes of rendering for 10 s on an Apple Silicon Mac. Tell the user it is
rendering. The rendered segment is a high-bitrate intermediate (about 5 MB per
second); the export re-encodes it.

## What comes back

`job.wait` returns `{ overlayId, linkGroupId, outputPath, startMs, endMs,
beats[{ kind, startMs, endMs, restMs, text, hero, heroBehind, still }], qc,
footage, warnings }`.

- `beats`: the shot plan. kinds: `whip` (opening whip-in), `truck` (a new
  caption layout arriving by parallax), `ring` (a clause on a turning ring in
  front of the speaker), `staircase` (the finale on a push). `still` is a PNG
  of that beat at rest: LOOK at every still before moving on (heads cover
  the hero word but it still reads, nothing cut off at the frame edge,
  captions off the face).
- `qc`: the automatic layout check (words leaving the frame while they should
  be read, words touching). `qc.ok=false` items are also in `warnings`.
- `footage`: the head box, the camera-motion measurement and the darker side
  the captions favour.
- `warnings`: say them to the user in plain words.

Times in the result are relative to the segment start; add `startMs` for
timeline time (`project.render-frame --atMs=<startMs + restMs>` shows the same
frame inside the edit).

## Changing it

- Re-run with other `heroWords`, `style` or `accent`: a new render over an
  overlapping range replaces the old one (the result lists `replaced`).
- Remove it: `project.remove-region --regionType=overlay --regionId=<overlayId>`
  (the caption hide over the range goes with it).
- After later cuts INSIDE the range, re-run it: the render is a picture of the
  footage at the time it was made.

## The rules it follows (for judging the stills)

- Captions lead the voice by 0.2 s on the 15-fps grid; each clause is its own
  group at its own depth (near groups bigger, with bigger shadows); the
  camera keeps creeping, it never fully lands.
- One hero per segment goes behind the head (masthead style) only when at
  least 60 % of it stays readable; otherwise heroes sit in front, at depth,
  in the accent colour.
- Captions favour the darker / roomier side of the frame and stay clear of the
  speaker; a line that cannot clear them gets a dark halo.
- No trailing periods on screen.

## Limits

- 16:9 only; Latin-script text only; one clip per segment (no range across
  two clips); no speed changes inside the range.
- Matte edges can flicker on fine hair; the mirrored padding can show for a
  frame during the fastest whip.
- The grain re-rolls every frame, so judge grain on the render, not a still.

Credit: the camera, depth, posterize and QC kit is ported from the Apache-2.0
HyperFrames community skill camera-3d-captions
(`motion-templates/_shared/cam3d/ATTRIBUTION.md`).
