# Motion-graphics launch and promo films

Use this when the video IS the motion graphics: a product launch, a feature reveal, a SaaS promo, a
teaser, an app or company launch, an animated explainer. Screenshots, a logo, a screen recording or a
camera clip can appear inside it, but graphics carry the film. For a promo cut from a screen
recording of someone using the product, see the `product-promo-from-url` recipe instead.

What makes these look produced rather than generated is craft, not effects. PandaStudio bundles
HyperFrames' launch-video craft (Apache-2.0) and its catalog of ~390 motion components. Read the
craft through `motion.craft`, build each beat as its own frame composition that mounts catalog
components, and render the whole film in one pass with `motion.render-film`.

## The three rules that decide quality

Read `motion.craft --id=motion-language` once per film. In short:

1. **Smooth beats bouncy.** Long-tail `power3` (or `expo.out` on a fast arrival) settles. No
   `back.out`, `elastic`, `bounce` as a default. Overshoot only when explicitly playful.
2. **Reveal on the voiceover cue, in the back half.** At t=0 a frame shows only what the voice is
   saying then. Each further line, card or number arrives when the voice names it. Never dump the
   whole frame in the first quarter and then freeze: that is the slideshow look.
3. **No lazy breathing, no slow drifting camera in the back half.** A still, finished frame beats
   a frame kept "alive" by pulsing scale or a creeping pan. Subtle jitter is the only aliveness.

Plus the seek-safe core every frame must obey: one paused GSAP timeline, no `repeat: -1`/`yoyo`,
no `Math.random`/`Date.now`, `fromTo` entrances, no CSS transitions or `@keyframes` for motion.

## Workflow

### 1. Brief

Pin down: product, audience, the one-line promise, proof (numbers, features, logos, quotes), CTA and
URL, length (30 to 60s; 45s is a good default), aspect ratio, voiceover or not, music or not. If the
user gave a URL, run `workspace.capture-brand --url=<url> --apply=false` and use its colours, fonts,
logo and screenshots. Use `apply=false` unless the user asked to change their workspace brand.

### 2. Design system

List presets with `motion.craft --kind=preset`, read `motion.craft --id=design-presets` for when to
pick each, then read the chosen preset (`motion.craft --id=<preset>`): its frontmatter `colors`,
`typography` and `components` are the look. Map the brand onto its roles (canvas, ink, accent) and
keep the preset's type ramp, spacing and component rules. Use the canvas colour as the film's
`groundColor`. One design system for the whole film.

### 3. Story

Read `motion.craft --id=story-design`. Pick ONE arc (PAS, Future Pacing, Demo Loop, BAB,
Feature-Benefit Cascade). Lay out beats, each with one job and a role (hook, pain_point,
product_intro, feature_showcase, benefit_highlight, social_proof, cta, branding). Write each beat's
voiceover in the shape of the matching script-bank pattern, in short phrase-cues (each cue is a
reveal). Tag a candidate blueprint per beat from `motion.craft --id=blueprints`. Vary the shapes.
Pick each beat's `transitionIn` (see step 6). Show the user the beat list with voiceover before
building when they are around.

### 4. Voiceover first (it sets the timing)

With narration, generate each beat's line (`media.generate-narration`, one call per beat) and read
its duration. Each frame's `durationMs` = its narration + about 600 ms of air (the last beat gets
1.5 to 2 s to hold the lockup). Silent films: 2.5 to 5 s per beat by density. Timing drives step 5:
write each frame's reveals at the moments its voiceover says the words.

### 5. Build each frame

Per beat, read `motion.craft --id=visual-design` once, then the beat's blueprint
(`motion.craft --id=<blueprint>`) and the rules it cites (`motion.craft --id=<rule>`). Write a
time-coded shot sequence for yourself (Scene 1 0.0-1.2s: ...), keeping the blueprint's signature move.

Before hand-building any named look, search the catalog: `motion.catalog --query="camera pull back"`,
`"chromatic wipe"`, `"logo sting"`, `"count up stat"`, `"device mockup"`, `"typed prompt"`. Read the
hit with `motion.catalog-item --name=<name>`: its `htmlPath` is the recipe (markup, CSS hooks, GSAP
timeline) and `demoPath` shows it mounted. Two ways to use an item:

- **Mount it** inside the frame when it is a whole shot (a pull-back reveal, a logo sting, a device
  stage): `<div data-composition-id="pull-back-reveal" data-composition-src="catalog:pull-back-reveal"
  data-start="0" data-duration="4.5" data-track-index="2" data-variable-values='{"headline":"..."}'></div>`.
  Only its declared variables are editable this way.
- **Adapt its recipe** into your own markup when you need your copy, colours and layout (most
  components are demo-sized cards; the value is the timeline and the technique). Keep the motion,
  restyle with the design system.

A frame is a full HTML document (or a `<template>` fragment):

```html
<!doctype html><html><head>
<style>
  #root { position: relative; width: 1920px; height: 1080px; overflow: hidden; }
  .bg { position: absolute; inset: 0; background: #F0EBDE; }
  .headline { font-family: "Newsreader"; font-size: 120px; color: #1F2BE0; opacity: 0; }
</style></head><body>
<div id="root" data-composition-id="hook" data-width="1920" data-height="1080" data-duration="4.2">
  <div class="bg clip" data-start="0" data-duration="4.2" data-track-index="0"></div>
  <div class="headline clip" data-start="0" data-duration="4.2" data-track-index="1">Getting traffic is hard.</div>
</div>
<script>
  window.__timelines = window.__timelines || {};
  const tl = gsap.timeline({ paused: true });
  tl.fromTo(".headline", { opacity: 0, y: 40 }, { opacity: 1, y: 0, duration: 0.8, ease: "power3.out" }, 0.2);
  window.__timelines["hook"] = tl;
</script>
</body></html>
```

- Paint the frame's full-bleed background on a `class="clip"` layer, never on `#root`.
- Times in `data-start` / `data-duration` are SECONDS.
- Name fonts by family; PandaStudio embeds Google fonts automatically.
- Logos and screenshots: pass their paths in `assets` on `motion.render-film` and reference them by
  file name (`<img src="logo.png">`). Never redraw a real logo or a real product screen.
- Keep content in the top ~83% when captions will be on.

Check each frame before the film: `motion.screenshot --html=... --atMs=<t>` at its first reveal, its
middle and its end (read `previewPath`). Fix layout, contrast and overflow here, it is cheap.

### 6. Render the film

```bash
pandastudio motion.render-film --frames='[
  {"id":"01-hook","htmlPath":"/tmp/f1.html","durationMs":4200},
  {"id":"02-intro","htmlPath":"/tmp/f2.html","durationMs":5100,"transitionIn":"zoom-through"},
  {"id":"03-feature","htmlPath":"/tmp/f3.html","durationMs":6000,"transitionIn":"push-slide LEFT"},
  {"id":"04-cta","htmlPath":"/tmp/f4.html","durationMs":5500,"transitionIn":"chromatic-wipe RIGHT"}
]' --groundColor="#F0EBDE" --assets='["/path/logo.png"]' --json
```

Transitions: `crossfade` (same visual world), `blur-crossfade` (backgrounds clash), `push-slide DIR`
(a run of feature beats), `zoom-through` (section change), `squeeze`, `chromatic-wipe DIR` and
`whip-pan DIR` (high energy, use once or twice), `iris`, `cut`. Pick a small set and repeat it.
Duration suffix optional (`crossfade 0.4s`). A transition extends the outgoing frame, so frame
starts never move. The result lists `frames[{id, startMs}]`: that is where each voiceover goes.
Read `warnings`: a frame whose timeline wasn't registered renders static.

Verify the film: `motion.verify-frames --videoPath=<outputPath> --timestamps=[...]` at each frame's
middle and through two or three transitions.

### 7. Sound and assembly

Put the film in a project so the user can adjust it: `project.new` with the film's aspect ratio,
`project.add-clip --media=<film>`, then each narration line with `project.add-audio` at its frame's
`startMs`. Sound design (bundled `asset.list-sounds`): `noise-riser` into `logo-impact` on the logo
landing, `swoosh-fast` only on zoom-through / whip / wipe transitions (about one per 8 to 10 s),
`ui-tick` for list items and checks, `mouse-click` for cursor clicks, `keyboard-*` under typed
prompts. Music, when wanted, sits well under the voice (`asset.list-music` or
`media.generate-music`) and fades out on the end card. Captions usually off: the type is on screen.
Export with `export.start`, then `export.verify` before handing it over.

## Craft index

| Need | Read |
|---|---|
| The whole story method, arcs, script bank | `motion.craft --id=story-design` |
| Shot sequences, layout vocabulary | `motion.craft --id=visual-design` |
| Move names, motion doctrine, seek-safe rules | `motion.craft --id=motion-language` |
| Velocity-matched cuts inside a frame | `motion.craft --id=cut-catalog` |
| The 22 shot shapes by beat role | `motion.craft --id=blueprints`, then `--id=<blueprint>` |
| GSAP recipes behind each move | `motion.craft --id=rules`, then `--id=<rule>` |
| Looks | `motion.craft --kind=preset`, `--id=design-presets`, `--id=<preset>` |
| Components and blocks | `motion.catalog --query=...`, `motion.catalog-item --name=...` |

The craft docs were written for HyperFrames' own CLI. Where they say `npx hyperframes`, frame
workers, `STORYBOARD.md` or `index.html`, use the PandaStudio steps above: `motion.catalog` for
catalog search, one frame document per beat, `motion.render-film` for assembly, `motion.screenshot`
and `motion.verify-frames` for checks, the project timeline for audio.
