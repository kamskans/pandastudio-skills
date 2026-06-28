<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Custom motion graphics: HTML authoring + render verbs

## Custom motion graphics — HTML authoring (when the content needs a visual no template captures)

Use the bundled templates for what they cover (titles, lower-thirds, stat
reveals, comparisons, side panels) — they're faster and already designed. But
**author your own when the beat needs a content-specific visual the gallery
can't express** — most often an **explainer diagram, flowchart, architecture, or
chart** (see "Authored graphics" above), and also bespoke one-offs, unusual
layouts, or brand-specific 3D. Authoring is a core skill here, not an admission
of defeat: an explainer that draws the system it's describing beats one that
lists it. Prefer a template when one genuinely fits; author confidently when
none does.

> **CRITICAL: render scene-by-scene, NOT one big 30s MP4.** If the brief is
> multi-scene (a promo / explainer with ≥3 distinct beats, anything ≥10s
> total), author **each scene as its own ~5-8s HTML composition** and add
> each as a sequential clip on the timeline (via `project.add-clip` for
> from-scratch content, or via `project.add-motion-graphic` only when the
> graphics are layered on top of host footage — see "Two distinct flows"
> below for the disambiguation). Do NOT build one monolithic 30s
> composition. Three reasons:
>
> 1. **Targeted re-renders.** When the user says "scene 3 needs the brand
>    color brighter," you redo that one scene (~30-60s), not the whole
>    promo (~10 min).
> 2. **Editor primitives apply per-clip.** Each scene gets its own trim,
>    speed, zoom, FX, layout transform — wasted on a single mega-clip.
> 3. **Render reliability.** `motion.render-html` for a 30s @ 1080p
>    composition pushes memory + capture-time hard. Five 6-second renders
>    are cheaper individually AND parallelizable through the render pool.
>
> Each scene's HTML is **reveal + hold** only — no baked-in exit tweens.
> Scene-to-scene transitions are placed on the timeline with
> `project.add-transition` (see the "Effects (FX) & transitions" section),
> not baked into the HTML. The full pattern + worked example lives
> in `reference/motion-philosophy.md` → "Multi-scene authoring".
>
> Single-scene graphics (one lower-third, one title card, one stat reveal)
> stay a single render → single clip. The rule above is for multi-scene
> compositions only.
>
> The escape hatch — `motion.concat` — exists for when the user explicitly
> asks for one combined MP4. It's not the default for promos.

> **Two distinct flows. Pick the right one BEFORE adding clips.**
>
> Motion graphics live in PandaStudio in two places, and which one you use
> depends on whether there's host footage:
>
> 1. **Generated video from scratch** — promo / explainer / PDF-to-video /
>    any brief where YOU are producing the primary content (no host on
>    camera, no uploaded recording). The rendered scene MP4s **ARE the
>    video**. They go on the **main track** via `project.add-clip`, one
>    clip per scene. The timeline IS your scene sequence.
>
> 2. **Layered on existing footage** — user has a recording open (host
>    talking on camera, screen capture, etc.) and wants motion graphics
>    composited ON TOP (lower thirds, stat callouts, brand watermarks,
>    side-panel explainers). The recording's clips are already on the
>    main track. Motion graphics go in `mediaOverlayRegions` via
>    `project.add-motion-graphic`, sitting above the host video at
>    specific time windows.
>
> Picking the wrong verb produces a broken project: `project.add-motion-graphic`
> on an EMPTY main track produces an editor that opens to "No video to load"
> because overlays compose ON the main track, and there's nothing there.
>
> **Chain for a brand-new promo / explainer / video-from-scratch**
> (the from-scratch case — use `project.add-clip`):
>
> ```bash
> # 1. Create a fresh project. Set aspectRatio to match the destination
> #    profile (16:9 YouTube, 9:16 Shorts/Reels, 1:1 LinkedIn square).
> P=$(pandastudio project.new --name="PandaScribe Promo" --aspectRatio=16:9 --json | jq -r '.data.id')
>
> # 2. Render each scene as its own motion graphic, then add the rendered
> #    MP4 to the MAIN TRACK with project.add-clip. NOT add-motion-graphic
> #    — that's for overlays on existing footage, and there's no footage
> #    here. add-clip ffmpeg-probes the MP4 duration automatically.
> for SCENE in intro problem demo testimonial cta; do
>   # Author the HTML for $SCENE somewhere (inline or tmp file)…
>   JOB=$(pandastudio motion.render-html --htmlPath="/tmp/$SCENE.html" --durationMs=6000 --json | jq -r '.data.jobId')
>   RESULT=$(pandastudio job.wait --id="$JOB" --timeoutMs=600000 --json)
>   OUTPATH=$(echo "$RESULT" | jq -r '.data.job.result.outputPath')
>   pandastudio project.add-clip --id="$P" --media="$OUTPATH"
> done
>
> # 3. Open the project in the editor so the user can preview, scrub,
> #    tweak individual scenes, export. THIS is the hand-off, not the
> #    bare MP4 paths.
> pandastudio preview.show --id="$P"
>
> # 4. Report to the user in chat with the project id (NOT the
> #    intermediate MP4 paths). Example:
> #    "Created PandaScribe Promo with 5 scenes on the timeline. Open in
> #     the editor — tell me which scene to tweak."
> ```
>
> Same flow applies when a project IS already open with no main-track
> content yet: skip step 1, reuse the existing `$P` from `project.current`.
>
> **Chain for layered graphics on existing footage** (host video already
> on the main track — use `project.add-motion-graphic`):
>
> ```bash
> # The user is editing a recording — clips are already on the main
> # track. Add a lower-third at 3.5s.
> JOB=$(pandastudio motion.generate --templateId=lower-third-clean \
>   --slots='{"name":"Kamal Kannan","title":"Founder, PandaStudio"}' --json | jq -r '.data.jobId')
> pandastudio job.wait --id="$JOB" --json
> pandastudio project.add-motion-graphic --id="$P" --fromJob="$JOB" \
>   --atMs=3500 --durationMs=4000
> ```

You author HTML/CSS/JS and render it with `motion.render-html` (the HyperFrames
engine — frame-perfect, seekable capture). The page MUST satisfy three things
(the full contract, canonical shell, easing dictionary, and three.js seek live
in `reference/motion-philosophy.md` — read it before authoring):

1. A composition root with `data-composition-id`, `data-width`, `data-height`,
   `data-duration` (seconds, float ok).
2. A **paused** GSAP timeline (`gsap.timeline({ paused: true })`) with a Law-#11
   anchor tween spanning the full duration. NO CSS keyframes, NO setTimeout/rAF,
   NO `repeat:-1` (Hyperframes seeks deterministically; infinite repeats break).
3. Register it: `window.__timelines["<data-composition-id>"] = tl;`

### Render verbs

```bash
# Pre-flight ONE frame before a full render (sub-second; catches layout bugs).
pandastudio motion.screenshot --htmlPath=/tmp/scene.html --atMs=1500 --json

# Render to MP4 (opaque). Renders are SEQUENTIAL — job.wait before firing the
# next or you get RENDER_BUSY. For many scenes, fire in parallel + job.wait each.
JOB=$(pandastudio motion.render-html --htmlPath=/tmp/scene.html --durationMs=4000 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json

# Transparent overlay (lower third / watermark / name plate): author with
# `background: transparent`, add --transparent → WebM + VP9 alpha.
# Frosted glass: render --transparent, then on add pass --backdropBlurStrength=24
# (12 subtle · 36+ heavy) + --backdropBlurTint='rgba(10,10,10,0.18)'.

# Add by jobId — never hand-build the output path (truncates at the space in
# "Application Support" and silently fails).
pandastudio project.add-motion-graphic --id=$ID --fromJob="$JOB" --durationMs=4000
# Multiple scenes → motion.concat the rendered MP4s into one before adding.
```

**References.** Authoring is **brand-driven** as of v1.32 — the editor-context
snapshot includes the user's brand kit (colors, type, voice, logo) on every
turn. Read those values verbatim and let them decide aesthetics; don't reach
for a hard-coded default look. Authoring contract:

- [`reference/motion-philosophy.md`](reference/motion-philosophy.md) — the
  authoritative correctness + craft contract (timeline anchor, determinism,
  layout-before-animation, scene transitions, quality checks). **Always read
  before authoring custom HTML.** Recently rewritten to be brand-agnostic; old
  prescriptions (dark canvas / chrome gradients / vignette + grid + grain as
  defaults) are gone.
- [`reference/house-style.md`](reference/house-style.md) — neutral fallback
  defaults when the brand kit is partial or absent. Three named tracks
  (creator-bright, dark-premium, product-clean) presented as STARTING points
  the user picks between, not as a default to silently apply. Voice → motion
  defaults table.
- [`reference/examples.md`](reference/examples.md) — concrete recipes
  (faux-cursor click, parallax-zoom, grid-pixelate-wipe, three.js setup).
- [`reference/motion-recipes.md`](reference/motion-recipes.md) — a menu of ~30
  named, seek-safe atomic motion patterns (spring-pop-entrance, kinetic-beat-slam,
  stat-bars-and-fills, depth-of-field-blur, multi-phase-camera, svg-path-draw,
  css-marker-patterns…), scene-transition families, and the determinism
  guardrails. Pick 2-4 per scene and compose them. Read when you need a proven
  pattern for a specific beat rather than inventing motion from scratch.

**Quality gate.** The **user is the final reviewer** of every render.
Do NOT auto-call `motion.screenshot` or `motion.verify-frames` as part
of your normal authoring flow — the human will open the MP4 in the
editor and catch any issue in two seconds, faster and more reliably
than the agent can. Your job is to author the composition with care,
walk the brand checklist textually before rendering (colors derived
from `brand.colors`, fonts from `brand.typography`, voice matches
`brand.voice`, logo from `brand.logoPath` when on screen), then render
and hand off. If the user comes back saying "scene 3 is broken" or
"the color's wrong," fix it and re-render — don't pre-emptively burn
turns inspecting frames.

`motion.screenshot` and `motion.verify-frames` are still available as
tools for when the user explicitly asks ("preview scene 2 at the 3-second
mark," "show me 8 frames across the timeline"). Just don't reach for
them on your own.

Upstream engine docs — canonical for engine internals: <https://hyperframes.heygen.com>.

