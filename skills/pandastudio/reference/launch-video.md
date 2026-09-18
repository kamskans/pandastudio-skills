# Motion-graphics launch and promo films

Use this when the video IS the motion graphics: a product launch, a feature reveal, a SaaS promo, a
teaser, an app or company launch, an animated explainer. Screenshots, a logo, a screen recording or a
camera clip can appear inside it, but graphics carry the film. For a promo cut from a screen
recording of someone using the product, see the `product-promo-from-url` recipe instead.

What makes these look produced rather than generated is craft, not effects. PandaStudio bundles
HyperFrames' launch-video craft (Apache-2.0) and its catalog of ~390 motion components. Read the
craft through `motion.craft`, build each beat as its own frame composition that mounts catalog
components, and render the whole film in one pass with `motion.render-film`.

## The shot grammar: one world, one camera

The difference between a launch film and a slideshow is not the components, it is
the shape of the whole piece. Good launch films (HeyGen's Astra film, Linear, Raycast,
Notion) are **one continuous product world the camera flies across**, not a run of
centred cards.

Build it that way:

- **One canvas, laid out in space.** Put the beats at real coordinates on a world
  several times larger than the frame (a 4000×2500 world for a 1920×1080 frame) and
  animate the camera — a wrapper's `x`/`y`/`scale` — from beat to beat. A frame in
  `motion.render-film` is a LEG OF THE FLIGHT, not a slide; consecutive frames should
  continue the same camera path so the cut reads as a move.
- **The product is the subject.** Reconstruct the real interface at large scale (a
  transcript panel, a composer, a result list, a file card) or place a real screenshot,
  and then let it CHANGE: text types, a word strikes through, a reply streams in,
  cards land, a chip flips to Ready. The demo IS the argument. Never a card that
  *describes* a feature in words.
- **An actor drives it.** An oversized cursor (or a touch dot) travels, clicks with a
  ripple, and the interface answers in the same beat.
- **Compose past the edges.** Let panels run off frame. A frame where everything sits
  centred with even margins reads as a slide; a frame cropped by its content reads as a
  window into a bigger place.
- **Depth and air.** A soft colour wash (two or three blurred brand-tinted blobs) under a
  light ground, thin connectors between stations, cards with real shadow. Not flat white.
- **Blur only while flying.** Add a few px of blur during fast camera legs and take it
  off as the camera lands; holds are perfectly sharp.
- **Type is punctuation, not the backbone.** One or two full-bleed lines between
  sections, and the end card. If the film is mostly typography, it is a slideshow.
- **The end card earns its glow.** Dark ground, the mark and wordmark together, the URL
  in a lit pill.

Anti-patterns that make it read as generated: a grid of feature cards; every beat
centred; one idea per frame with a transition between; text that states benefits the
product could have shown; a camera that never moves.

## Agent and chat beats

When a beat is "ask the AI and watch it work", use the catalog's **`claude-exchange`**
block rather than drawing a chat yourself: a full Claude conversation on a phone —
composer, typing, thinking line, tool rows, then the answer streaming in with real
scrolling. `motion.catalog-item --name=claude-exchange` lists its editable fields
(`prompt`, `thinking`, `lead`, `search`, `answer1`…`answer10`).

Its editing contract (shipped as TEMPLATE.md beside it) is binding:

- Change ONLY those declared fields. The Claude name, model name, usage notice,
  disclaimer, icons, layout, palette, timing and typing cadence stay as they are, and
  the brand's colours and fonts are never applied to the Claude shell.
- Keep each replacement within about 20% of the original's length, or the typing and
  streaming timing stops matching the copy.
- Write what your product genuinely does. The answer is an ad for the brand, so it must
  be true: no invented benchmarks, rankings, review quotes or numbers.

Mount it in a device frame inside the world:

```html
<div class="phone">
  <div class="screen clip" data-start="8.6" data-duration="12" data-track-index="2"
       data-composition-id="claude-exchange" data-composition-src="catalog:claude-exchange"
       data-variable-values='{"prompt":"Can you tighten this recording?", …}'></div>
</div>
```

For a long conversation it is often easier to render the block once on its own
(`motion.render-html` at 1080×1920) and play that clip inside the phone with a
`<video>`; the clip holds its last frame, so the answer stays on screen while the camera
moves on. Other chat shapes: `chat-thread`, `notes-typing`, `typed-prompt`,
`streaming-text`, `agent-progress-theater`.

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

### 4. Timing

Default (no voiceover): pace by what is happening on screen — 3 to 5 s for a camera leg that
reveals one thing, 8 to 12 s for a chat or demo beat that has to play out, 4 to 6 s for the end
card. Write the section lines (the on-screen type) at the same time, one short line per section,
and hold each one for 2.5 to 3.5 s.

Only when the user asks for narration: generate each beat's line first
(`media.generate-narration`, one call per beat), read its duration, and set that frame's
`durationMs` to the line plus about 600 ms of air, so reveals can land on the words.

### 5. Build each frame (a camera leg)

Lay the world out first: where each station sits, and the camera path that visits them.
The blueprints that fit this grammar are `camera-journey`, `cursor-ui-demo`,
`prompt-type-submit-generate`, `agent-progress-theater`, `device-surface-showcase`,
`zoom-out-workspace-reveal` and `spatial-pan-stations`. `kinetic-type-beats` and
`grid-card-assemble` are punctuation only — reaching for them every beat is what
produces a deck.

Per beat, read `motion.craft --id=visual-design` once, then the beat's blueprint
(`motion.craft --id=<blueprint>`) and the rules it cites (`motion.craft --id=<rule>`). Write a
time-coded shot sequence for yourself (Scene 1 0.0-1.2s: ...), keeping the blueprint's signature move.

Before hand-building any named look, search the catalog: `motion.catalog --query="camera pull back"`,
`"chromatic wipe"`, `"logo sting"`, `"count up stat"`, `"device mockup"`, `"typed prompt"`. Read the
hit with `motion.catalog-item --name=<name>`: its `htmlPath` is the recipe (markup, CSS hooks, GSAP
timeline) and `demoPath` shows it mounted. Two ways to use an item:

The catalog is yours, not the user's: it is not in the editor UI, because most
items carry fixed demo copy. Use it to build scenes; hand the user the result.

Some blocks depict other companies' products (claude-exchange, chatgpt-exchange,
slack-notification-ad, x-post, spotify-card, the apple-terminal snippets). Use one
only when the video is truthfully about that product being used, never to suggest
that company endorses the user's product, and never as a skin for the user's own
app: for their product, build its real interface instead.

- **Mount it** inside the frame when it is a whole shot (a pull-back reveal, a logo sting, a device
  stage): `<div data-composition-id="pull-back-reveal" data-composition-src="catalog:pull-back-reveal"
  data-start="0" data-duration="4.5" data-track-index="2" data-variable-values='{"headline":"..."}'></div>`.
  Only its declared variables are editable this way. Keep `data-composition-id` equal to the item
  name. About a third of the components are paste-in primitives (blur-in, animated-bar-chart,
  streaming-text…) that ship their animation as a "Timeline integration" snippet instead of a timeline
  of their own; a mount runs that snippet for you from the mount's start, so they move like the rest.
  When you need a different start or pacing, adapt the snippet into your frame's own timeline instead.
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

### 7. Sound and assembly (no voiceover by default)

A launch film carries itself on picture, music and sound design — the reference films
in this space have no narration at all, and neither should these unless the user asks
for one. The on-screen type does the talking: one short line per section.

Put the film in a project so the user can adjust it: `project.new` with the film's aspect ratio,
`project.add-clip --media=<film>`, then the music bed. Match the bed to the film rather than
grabbing one: the launch films that work sit on a restrained, mid-tempo (around 90-100 BPM) bed
with little percussion. Two bundled beds are built for exactly this film and
need no generation: `launch-pulse` (confident, carries the cut) and `quiet-launch` (quieter and
more considered), both ~33s, so repeat them for a longer film. `media.generate-music` writes a
custom one when neither fits ("restrained modern tech underscore, 96 bpm, soft pulsing synth,
minimal percussion, no vocals"), and `--model=musicgen --reference=` matches a track the user
already has. Never lift the audio from a reference video; match its character instead. Sound design (bundled `asset.list-sounds`): `noise-riser` into `logo-impact` on the logo
landing, `swoosh-fast` only on zoom-through / whip / wipe transitions (about one per 8 to 10 s),
`ui-tick` for list items and checks, `mouse-click` for cursor clicks, `keyboard-*` under typed
prompts. With no voiceover the bed sits forward (it is the floor of the mix, not a whisper) and the
effects sit under it. Aim for about -18 LUFS with true peak at -1 dBFS or lower; check with the
export and raise the bed, not the effects, if it comes out quiet. Captions stay off: the type is
already on screen.
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
