<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Motion graphics: backgrounds, designed segments, template catalog

### Background modes (`--background`)

| Mode | Output | Use |
|---|---|---|
| `solid` | hard full-frame card | opaque templates (cards, stat, checklist, comparison) — the default for those |
| `transparent` | alpha; camera shows through un-painted pixels | overlay templates — the default for those |
| `glass` | alpha + frosted blur of the camera behind the content | overlay templates when you want the modern frosted look; also pass `--backdropBlurStrength=24` on `add-motion-graphic` |

Omit `--background` to use the template's natural mode. **Never force
`solid` on an overlay template** — its see-through region renders black.
Overlay templates are flagged `overlay: true` in `motion.list`.

### Designed segments (host on one half, panel on the other)

`split-panel` is a "designed segment": an opaque brand panel fills one half,
the other half is transparent for the host. Add it with
**`project.add-designed-segment`** instead of `add-motion-graphic` — that
one atomic call also repositions the camera into the open half (a
`cam-{side}-50` clip-transform) over the same span, so host + panel can't
drift. Set the panel's `side` slot; the camera takes the opposite side.

**`--durationMs` is how long the panel stays** — set it to the length of the
topic it covers, not a fixed few seconds. The panel **reveals once and holds**:
it has no exit animation, and the overlay holds its last frame for the whole
region (it does NOT loop), so a 30-second explainer beat is just
`--durationMs=30000`. The region end is what removes it. (Same for any overlay:
play-once + hold; size the region/durationMs to the on-screen time you want.)

```bash
JOB=$(pandastudio motion.generate --templateId=split-panel \
  --slots='{"side":"left","headline":"What MyAgentMail handles:","items":[{"label":"inbox creation"},{"label":"sending & replies"},{"label":"webhooks"}]}' \
  --background=transparent --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json
# durationMs = full length of the topic the panel explains (here 28s):
pandastudio project.add-designed-segment --id="$PROJECT" --fromJob="$JOB" \
  --durationMs=28000 --cameraSide=right
```

**9:16 Shorts variant — top/bottom split.** For vertical, the split runs
horizontally. **Use the same premium panels** — `paper-panel` or
`vox-side-panel` — rendered at `--aspectRatio=9:16`: they're aspect-aware and
reflow into a top/bottom band. Use `cameraSide: top`/`bottom` instead of
`left`/`right`; the panel's `side` slot and `cameraSide` are opposites (panel
`top` ⇒ host `bottom`). Keep each panel's own `cameraRatio` (`paper-panel` → 55,
`vox-side-panel` → 50). See "Editing a Short — the vertical playbook" for the
full example and when to reach for it.

### Template catalog

Pick by brief. Slots in **bold** are required; the rest are optional. **Any
slot you omit falls back to the template's own designer-picked default —
including every color** (see each template's `defaults` in `motion.list`). So you
pass only the slots you want to change and the rest look correct out of the box;
you never need to spell out every color. (A workspace brand kit, when set,
overrides the template's default colors; with no brand kit you get the
template's own palette.) All are 16:9 / 9:16 / 1:1 unless noted. `O` = overlay
(transparent-capable).

**Titles & chapter cards**
- `creator-card` (4.5s) — full-bleed brand title / chapter card with a live dot. Bold YouTube-creator look. Slots: **headline**, eyebrow, brandColor, inkColor, dotColor.
- `transitions-3d` `O` (4.5s) — editorial title that flips in with a 3D rotate: small lead / HUGE word / small trail. Slots: lead, **emphasis**, trail, textColor, accentColor.
- `grain-overlay` `O` (5s) — bold uppercase headline + kicker with a flickering film-grain texture over the footage. Slots: kicker, **headline**, textColor.
- `transitions-destruction` `O` (5s) — headline assembles from slabs, holds, then shatters apart. Dramatic reveal/transition. Slots: kicker, **headline**, accentColor, textColor.
- `caption-parallax-layers` `O` (5s) — one word stacked in 5 depth layers (solid front + receding ghosts) that enters with a vertical stretch and parallaxes. Slots: eyebrow, **headline**, frontColor, ghostColor.

**Lower thirds** (all `O`, 5s, transparent overlays — add in ONE call with `project.add-lower-third --name --title --atMs [--templateId]`; slots: **name**, title + per-template colors. Also in the editor's Lower 3rds tab.)
- `yt-lower-third` `O` (4.5s) — subscribe lower-third: avatar + name + title + red Subscribe pill, slides in bottom-left. Slots: **name**, title, accentColor, cardColor, inkColor.
- `lt-vox-marker` — the name lands on a highlighter swipe; mono role on an accent rule. The Vox look.
- `lt-broadcast-bar` — news chyron: accent tab + dark name bar wipe open, accent role strip below.
- `lt-glass-card` — frosted translucent pill with an accent bar. Modern, premium.
- `lt-kinetic-stack` — oversized name snaps in word-by-word over an accent rule. Loud, energetic.
- `lt-minimal-line` — thin line draws between the name and a letter-spaced role. Editorial, quiet.
- `lt-corner-plate` — documentary L-bracket draws on around a mono kicker + caps name. Extra slot: kicker.
- `lt-accent-slab` — bold accent block with a knockout name. Punchy.
- `lt-avatar-pill` — creator pill: circular initial avatar + name + handle + live dot. YouTube/stream.
- `lt-typewriter-tag` — name types in mono with a blinking caret; accent tag chip pops under it.
- `lt-gradient-sweep` — name settles as a gradient bar sweeps in over a soft glow.

**Data & explainer**
- `stat-reveal` (4s) — a huge number that counts up, with eyebrow + optional prefix/suffix + label. "10,000+ subscribers", "3× faster". Slots: eyebrow, prefix, **value**, suffix, **label**, brandColor, inkColor, accentColor.
- `key-takeaways` (6s) — titled checklist; points reveal one-by-one with drawn ticks. The "here's what you need to know" recap. Slots: **title**, items[label], bgColor, accentColor, inkColor.
- `comparison` (5s) — two labeled panels slide in with a VS badge; the right (winning) panel pulses. Before/after, old-way/new-way. Slots: leftTag, **leftHeading**, leftCaption, rightTag, **rightHeading**, rightCaption, bgColor, leftColor, rightColor, inkColor.
- `flowchart` (6s) — horizontal node cards joined by drawn arrows, revealed step by step. Process / workflow. Slots: nodes[label], brandColor, cardColor, inkColor, accentColor.

**Reveals & recaps**
- `parallax-zoom` (5s) — a center hero card zooms to fill while four corner satellites parallax out. Hero reveal / intro. Slots: **headline**, items[label], brandColor, cardColor, inkColor.
- `parallax-unzoom` (4.5s) — a focus card unzooms to reveal a 2×2 grid as the others parallax in. "ways to use it" / features recap. Slots: items[label], brandColor, cardColor, inkColor.

**Host + panel split**
- `split-panel` `O` (16:9 / 1:1) — opaque brand panel on one half (headline + bullet list), transparent on the other for the host. **Reveals once and holds** (no exit) — set `--durationMs` to the topic length; it holds for exactly that long. Add via `project.add-designed-segment` (see above). Slots: side (`left`/`right`), **headline**, items[label], brandColor, inkColor.
  > **Designed segments are coupled.** As of v1.35.0, the camera layout transform and the panel motion graphic created by `project.add-designed-segment` share a `linkGroupId`. Moving one on the timeline shifts the other by the same delta; deleting one deletes its peer. The agent's mental model can stay simple ("this is one beat"), and direct-manipulation in the UI never produces an orphan transform with no panel to balance it.
  >
  > **Removing / updating a clip-transform (standalone or paired).** Pass `regionType=clip-transform` to the generic verbs — there is NO separate `remove-clip-transform-region` / `update-clip-transform-region` verb. Examples:
  > ```bash
  > # Remove (deleting the paired panel graphic too if part of a designed segment)
  > pandastudio project.remove-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 --json
  >
  > # Change the preset and bump the slide-in transition
  > pandastudio project.update-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 \
  >   --preset=cam-left-50 --transitionMs=480 --json
  >
  > # Shift it later by 2s (paired peer in the link group shifts by the same delta)
  > pandastudio project.update-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 \
  >   --startMs=6000 --endMs=12000 --json
  > ```
  > Valid presets: `cam-bottom-half`, `cam-top-half`, `cam-right-portrait`, `cam-right-portrait-sm`, `cam-left-portrait`, `cam-bottom-right-quarter`, `cam-bottom-left-quarter`, `cam-right-55`, `cam-left-55`, `cam-right-50`, `cam-left-50`, plus the podcast layout-over-time presets `layout-side-by-side`, `layout-podcast`, `layout-host-full`, `layout-guest-full` (see "Podcast: change layout over time" below). `transitionMs` clamps to `0–2000`.

**Background graphic + camera card** — to put a full-frame motion graphic BEHIND a talking-head (camera as a side card): add the graphic with **`project.add-motion-graphic --layer=background`** (renders it behind the camera), then add a camera clip-transform with a small CARD preset (`cam-right-portrait-sm` recommended, or `cam-bottom-right-quarter` for a corner) over the SAME span. The camera card composites on top automatically, in preview AND export. **Always pass `--layer=background`** — it's what makes the live EDITOR PREVIEW correct (without it the graphic still ends up behind the card in the exported video via the camera punch-through, but the preview shows it on top, which confuses the user). In the UI this is the one-click **"Background"** placement in the Graphics panel.

**Imported pack — HeyGen ports & Vox explainer style** (bold, motion-forward; the Vox templates default to Vox yellow `#F7C600` but every color is brand-aware)
- `shader-dissolve` (4s, 16:9) — a real WebGL domain-warp dissolve between two full-frame title cards with an iridescent edge glow. Premium transition between two words. Slots: **titleA**, **titleB**, colorA, inkA, colorB, inkB.
- `ring-title` (4.5s, 16:9) — a rotating 3D ring of accent-tinted segments over a centered title + eyebrow. Premium 3D intro. Slots: **title**, eyebrow, bgColor, inkColor, accentColor.
- `vox-marker` (4.5s) — heavy headline reveals word by word, then a highlighter marker sweeps behind one emphasis word and the word flips to read on it. The iconic Vox headline. Slots: eyebrow, **headline**, **emphasisWord**, bgColor, inkColor, accentColor, markInk.
- `vox-stat` (4.5s) — a big figure pops in with an accent rule, a label, and a "SOURCE:" credit. Vox data callout (real numbers; an alternative look to `stat-reveal`). Slots: eyebrow, **value**, **label**, source, bgColor, inkColor, accentColor.
- `image-showcase` (6s, 16:9 / **9:16**) — **the dedicated image/screenshot template.** A screenshot or photo floats in on a 3D-tilted card with a soft shadow + glass sheen, beside (16:9) or above (9:16) an optional headline. The go-to for highlighting a product page, app screen, or any image in a demo — tilt-and-present, the common product-video pattern. **Aspect-aware** (render at `--aspectRatio=9:16` and it reflows to caption-on-top, big tilted card below). Pass `--background=transparent` to float the card straight on the host footage. **Featured.** Slots: **image** (abs path), headline, eyebrow, bgColor, inkColor, accentColor.
- `vox-quote` (5s) — oversized accent quotation mark + quote + attribution (name / role) under an accent rule. Pull-quote / testimonial. Slots: **quote**, name, role, bgColor, inkColor, accentColor.
- `vox-annotation` (4.5s, 16:9) — a hand-drawn marker circle scribbles around a subject word with a handwritten note + curved arrow. "this is what matters" callout. Slots: **subject**, note, bgColor, inkColor, accentColor.
- `vox-side-panel` `O` (16:9 / **9:16**) — Vox designed segment: a graph-paper half-panel with a two-line marker title, a taped specimen card, and a monospace spec list; the other half is transparent for the host. **Aspect-aware** — at 16:9 it's a left/right side panel; render at `--aspectRatio=9:16` and it reflows into a top/bottom **band** (marker titles + specs on the left, taped card on the right) for Shorts. Add via `project.add-designed-segment` (16:9: `side` left/right, cameraRatio 50; 9:16: `side` top/bottom, `cameraSide` opposite, cameraRatio 50). Slots: side, **title1**, **title2**, **subject**, spec1, spec2, spec3, paperColor, inkColor, accentColor, accent2Color.
- `paper-panel` `O` (16:9 / **9:16**, v2.99.0) — designed segment: a torn-paper sheet carrying a two-line title (the second line accented with a hand-drawn underline) + a short subtitle; the other half is transparent for the host. The cleanest, most editorial of the panels. **Aspect-aware** — at 16:9 it's a left/right side panel (camera 55%, panel 45%); render at `--aspectRatio=9:16` and the sheet becomes a top/bottom **band** with a horizontal torn edge for Shorts. Add via `project.add-designed-segment` (16:9: `side` left/right; 9:16: `side` top/bottom, `cameraSide` opposite; cameraRatio 55 either way). Slots: side (16:9 `left`/`right`, 9:16 `top`/`bottom`), **title1**, **title2** (accented line), subtitle, paperColor, inkColor, accentColor.

#### Import a podcast from separate files (host + guest recorded apart)

When the two (or more) speakers were recorded SEPARATELY — e.g. local tracks
exported from Zoom/Riverside, or two phone cameras — assemble them into a single
two-speaker podcast composite with **`project.add-podcast-clip`**. This produces
the exact same representation as an in-app/remote podcast recording (host =
mediaPath, guests = `participants[]`, `kind: "podcast"`), so every podcast verb
below (layout-over-time, per-speaker transcript editing) works on it.

```bash
# Host + one guest (2-party). Auto-syncs the guest to the host by default.
pandastudio project.add-podcast-clip --id=$PROJECT \
  --media="/path/host.mp4" --guests='["/path/guest.mp4"]' --json

# 3-4 participants: pass guests in order (guest, guest-2, guest-3).
pandastudio project.add-podcast-clip --id=$PROJECT \
  --media="/path/host.mp4" \
  --guests='["/path/guest1.mp4","/path/guest2.mp4"]' \
  --labels='["Alex (host)","Sam","Jordan"]' --json
```

- **Auto-sync**: each guest is audio-aligned to the host (energy-envelope cross-
  correlation) and the offset stored on the participant. Pass `--autoSync=false`
  to assemble with zero offsets. Re-run later with **`project.auto-sync-podcast
  --clipId=…`**. The response includes a per-guest `{ offsetMs, confidence,
  rawOffsetMs }` report. **It only applies a shift when confident** (correlation
  ≥ 0.35): a low-confidence result leaves `offsetMs` at 0 (so tracks recorded in
  alignment, or isolated per-mic remote tracks that don't cross-correlate, are
  never shifted by a spurious peak) — `rawOffsetMs` shows the rejected guess.
  Nudge manually with `project.set-participant-offset` when needed.
- **Manual nudge**: **`project.set-participant-offset --clipId=… --speaker=guest|guest-2|guest-3 --offsetMs=…`**
  shifts one guest's video AND audio vs the host (±10000; positive delays; 0
  clears). The N-party analogue of `project.set-webcam-offset` (guest-1 only).
  The host is the reference clock and can't be offset.
- **Quick 2-party shortcut**: `project.add-clip --media=host --webcam=guest`
  also makes a podcast composite (no auto-sync; use `add-podcast-clip` for that
  or 3+ speakers).
- **Per-speaker transcription**: run `transcript.transcribe` after assembling.
  For a podcast composite it transcribes EACH participant source separately and
  merges into one speaker-attributed transcript (words tagged
  `host`/`guest`/`guest-2`…). `transcript.get` returns those `speaker` tags, and
  `transcript.delete-words` / `remove-fillers` / `remove-silences` operate across
  the merged conversation — cutting a moment removes it from every speaker's
  video at once, exactly like a recorded podcast.
- **Layout**: the project's webcam layout is set to `podcast` automatically;
  3-4 participants auto-grid. Change it over time with the `layout-*` /
  `podcast-*` clip-transform presets (below).

> UI parity: the Home screen's **Import video** card runs this same flow — pick
> ONE file for a single clip, or SEVERAL (host first, then guests) to assemble a
> podcast composite; auto-sync runs in the background.

#### Podcast: change layout over time (within ONE recording)

To switch a podcast's layout PART-WAY THROUGH a single recording — e.g. side-by-side for the intro, cut to the host full-frame while they make a point, then back — add **clip-transform regions with `layout-*` presets**. These are the SAME timeline item as a camera transform ("Layout transform" in the UI), but instead of repositioning one clip they swap the whole composite for the window. Both the host and guest tiles interpolate at the region edges (default 320ms), so the layout animates.

```bash
# Host full-frame from 30s–45s, animating in/out from the base side-by-side layout
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=30000 --endMs=45000 --preset=layout-host-full --json
# Guest full-frame for a reaction shot
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=61000 --endMs=66000 --preset=layout-guest-full --json
```

`layout-*` presets: `layout-side-by-side` (both, side by side / stacked), `layout-podcast` (two co-equal tiles), `layout-host-full` (speaker 1 / mediaPath only), `layout-guest-full` (speaker 2 / webcamPath only). Outside any region the clip uses its natural layout (the project preset or per-clip override from `project.set-clip-layout`). Edit/remove via `project.update-region` / `project.remove-region` with `regionType=clip-transform` (above).

**Multi-party (host + up to 3 guests) — pick WHICH participants with the participant-aware presets.** The `layout-*` presets above are 2-person only (host=speaker 1, guest=speaker 2). For 3-4 participants, use the participant-aware family + a `--participants` list of speaker ids (`host`, `guest`, `guest-2`, `guest-3` — read them from `transcript.get` / `project.read` clip participants):

```bash
# Solo — feature just one chosen participant full-frame (e.g. the 2nd guest)
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=30000 --endMs=45000 --preset=podcast-solo --participants='["guest-2"]' --json
# Pair — two participants side by side (ordered left → right)
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=46000 --endMs=60000 --preset=podcast-pair --participants='["host","guest-3"]' --json
# Grid — everyone (omit --participants) or a chosen subset
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=0 --endMs=30000 --preset=podcast-grid --json
# Screen share — the shared screen + a strip of participants
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=61000 --endMs=120000 --preset=podcast-screen-share --json
```

Presets: `podcast-solo` (participants[0] full-frame), `podcast-pair` (participants[0]=left, [1]=right), `podcast-grid` (auto grid — 2 side-by-side / 3 three-up / 4 2×2; the chosen subset or everyone), `podcast-screen-share` (shared screen + participant strip). Selections are validated against the clip's real participants; a stale id (a guest who wasn't in the call) is dropped. Speaker-driven flow: read `transcript.get` speaker tags, then drop a `podcast-solo --participants=[whoever is talking]` region over each span.

> **`layout-*` is PODCAST-ONLY.** These presets only do anything on a clip with `kind === "podcast"` (a clip carrying BOTH a host and a guest source). On a normal screen/camera/upload clip there is no second speaker, so a `layout-*` region is a no-op — use the `cam-*` presets there. Confirm `kind` via `project.read` before placing a `layout-*` region.

**Pull-quote / emphasized line**
- `caption-editorial-emphasis` `O` (4s, all aspects) — one-sentence pull-quote, ONE word (or short phrase) blown up in huge Playfair italic that slides in from the left. Regular words pop in word-by-word in Inter; emphasis slides in below them; holds the final frame. Transparent overlay — drops on any clip. Slots: **sentence**, **emphasisWord**, inkColor, accentColor. Trailing punctuation after the emphasis auto-merges onto the emphasis (so `"…starts with a single frame."` renders `single frame.` as one unit). If `emphasisWord` is NOT a substring of `sentence`, it's appended to the end as a punchline — useful when you want the sentence to LEAD with normal text and END with the dramatic pull-out. **TALKING-HEAD OPENER (default, do this on every talking-head edit): place ONE within the first 10–30s that introduces the TOPIC** — a short line naming what the video is about, derived from the speaker's opening sentences (e.g. host opens "today I want to talk about how we cut our render times in half" → emphasis card `"Cutting render times in half."`). It orients the viewer at the exact moment a talking-head otherwise loses them and reads as deliberate editorial framing. This is a strong default for ANY `kind === "camera"` footage whenever you're adding graphics, not just "make it engaging" briefs. **After the opener, use sparingly: at most 2–3 total, reserving one for the climax / "money line."** The opening topic intro and the chapter-closing payoff are the strongest slots; sprinkling it every minute burns the size contrast. Not a replacement for the running caption track (`caption.set-template editorial` is the style for that). Add via `project.add-motion-graphic` (it's an overlay, not a main-track clip).
  ```bash
  JOB=$(pandastudio motion.generate --templateId=caption-editorial-emphasis \
      --slots='{"sentence":"Every great video starts with a single frame.","emphasisWord":"single frame"}' \
      --aspectRatio=16:9 --json | jq -r '.jobId')
  pandastudio job.wait --id=$JOB
  pandastudio project.add-motion-graphic --fromJob=$JOB --startMs=3000 --durationMs=4000
  ```

> List slots (`items`, `nodes`) take an array of objects, e.g.
> `"items":[{"label":"first"},{"label":"second"}]`. `motion.screenshot`
> renders a single frame of any template+slots combo if you want to preview
> before committing to a full render.

