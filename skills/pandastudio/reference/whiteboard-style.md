<!-- Part of the pandastudio skill. Load when the brief calls for a whiteboard / hand-drawn / sketch explainer. -->

# Whiteboard style — hand-drawn explainer videos

A named, proven design system for **from-scratch explainers where the concept
is the star and there is no footage of it**: paper background, SVG strokes that
draw themselves on, captions that write in left-to-right like handwriting,
marker typography, and a 4-color marker palette. The RSA-Animate /
Minute-Physics genre, built entirely with `motion.render-html`.

Verified end-to-end in production (July 2026, app 1.60): a 20s black-hole
explainer — 4 scenes, all draw-on effects, zero static-render failures.

## When to use it

| Fits | Why |
|---|---|
| Science / education explainers (black holes, compound interest, how X works) | The concept is invisible; a drawing that assembles IS the explanation |
| "How it works" process videos (request flow, supply chain, onboarding flow) | Boxes + arrows drawing on in narration sync |
| Startup / product CONCEPT pitches (not UI demos) | Napkin-sketch feel reads honest, approachable |
| Training / internal onboarding | Friendly tone lowers the stakes |
| Metaphor-driven storytelling (leaky bucket, flywheel) | You literally draw the metaphor |
| Kids / beginner content | The aesthetic itself signals "this will be simple" |

**Wrong tool for:** product UI walkthroughs (screen recording + zooms), premium
brand promos (use the custom Mode-B scene system with real brand type),
precise data-viz, talking-head enhancement. (A future variant — hand-drawn
annotations as transparent overlays doodled over real footage — is plausible
but not this recipe.)

**Trigger phrases:** "whiteboard animation", "hand-drawn", "handwritten
effect", "sketch style", "doodle video", "draw it out", "explain the concept
of X" (when there's no footage and the tone wants friendly, not premium).

## The design system

One system across every scene (this is what makes it cohesive), varied
composition per scene (this is what keeps it from feeling templated):

- **Canvas:** warm paper `#f4f1e8`, subtle 64px graph grid, soft vignette.
  NOT white — pure white reads as an empty browser page on video.
- **Ink palette (markers):** ink `#24211b` (primary strokes + captions),
  blue `#2f6fed` (underlines, structural rings), red `#e0483b` (annotations,
  arrows, the "point being made"), amber `#f2a626` (energy: stars, light,
  highlights). Black fill `#141414` only for solid shapes (the black hole).
- **Type:** `'Caveat','Comic Sans MS',cursive` for big top captions (~82px,
  weight 700); `'Permanent Marker','Caveat',cursive` for side labels
  (~46–58px). Red labels in Permanent Marker read as "scribbled emphasis".
- **Scene grammar (repeat per scene):** top caption writes in → main drawing
  draws on → red annotation arrow + label lands on the payoff. Caption gets a
  wavy hand underline in blue.
- **SFX:** a quiet paper/pen tick at each scene start —
  `project.add-audio --audioPath="bundled:sound/crumpled-paper" --durationMs=700 --volume=0.3`.

## The three core techniques

### 1. Draw-on strokes (the whiteboard illusion)

Every drawn element is an SVG path with `pathLength="1"` and CSS
`stroke-dasharray:1; stroke-dashoffset:1` (fully hidden), revealed by
tweening `strokeDashoffset` to 0:

```css
.draw { fill:none; stroke:#24211b; stroke-width:7; stroke-linecap:round;
  stroke-linejoin:round; stroke-dasharray:1; stroke-dashoffset:1; }
```

```js
tl.to('.star',  { strokeDashoffset:0, duration:1.7, ease:'power1.inOut' }, 0.4);
tl.to('.ticks', { strokeDashoffset:0, duration:0.4, ease:'power2.out', stagger:0.05 }, 2.0);
```

`pathLength="1"` normalizes every path so dasharray/dashoffset are always 0..1
regardless of real path length — one CSS rule covers every stroke. Stagger
multi-stroke figures slightly so they feel drawn in sequence, not printed.

### 2. Handwriting text reveal

Wrap the text in a span and animate `clip-path` left-to-right with
`ease:'none'` (a hand writes at roughly constant speed — an eased reveal
reads as a wipe, not writing):

```html
<div class="caption cap-top"><span class="write">Gravity crushes its mass</span></div>
```
```css
.write { display:inline-block; clip-path:inset(0 100% 0 0); }
```
```js
tl.to('.cap-top .write', { clipPath:'inset(0 0% 0 0)', duration:1.2, ease:'none' }, 0.15);
```

Budget ~0.12–0.16s per word-character-cluster; a ~6-word caption ≈ 1.2s.
Reveal the wavy underline right AFTER the caption finishes.

### 3. Hand-drawn accents

- **Wavy underline:** a quadratic path dipping ~14px mid-span:
  `M x1,y Q mid,y+14 x2,y-4` — never a straight line.
- **Annotation arrows:** a curved Q-path from the label toward the subject,
  plus two short barb strokes at the tip, all class `.draw`, drawn right
  before their label writes in.
- **Sparkle ticks / starbursts:** short radial strokes around a point,
  staggered ~0.03–0.05s.

## Canonical scene skeleton

Each scene is its own 5s composition → its own clip (multi-scene rule from
`motion-philosophy.md` applies — do NOT build one monolithic file):

```html
<!doctype html><html><head><meta charset="utf-8"><style>
* { margin:0; padding:0; box-sizing:border-box; }
html,body { width:1920px; height:1080px; overflow:hidden; }
.comp { position:relative; width:1920px; height:1080px; background:#f4f1e8; }
.paper { position:absolute; inset:0; }
.paper::before { content:""; position:absolute; inset:0;
  background-image:
    linear-gradient(rgba(40,38,33,.045) 1px, transparent 1px),
    linear-gradient(90deg, rgba(40,38,33,.045) 1px, transparent 1px);
  background-size:64px 64px; }
.paper::after { content:""; position:absolute; inset:0;
  background:radial-gradient(125% 120% at 50% 44%, transparent 52%, rgba(70,58,34,.12)); }
.draw { fill:none; stroke:#24211b; stroke-width:7; stroke-linecap:round;
  stroke-linejoin:round; stroke-dasharray:1; stroke-dashoffset:1; }
.pop { opacity:0; }
.caption { position:absolute; left:0; right:0; top:78px; text-align:center;
  color:#24211b; font-family:'Caveat','Comic Sans MS',cursive;
  font-size:82px; font-weight:700; }
.write { display:inline-block; clip-path:inset(0 100% 0 0); }
</style></head><body>
<div id="sc1" class="comp" data-composition-id="sc1"
     data-width="1920" data-height="1080" data-duration="5">
  <div class="paper"></div>
  <svg style="position:absolute;inset:0" viewBox="0 0 1920 1080" xmlns="http://www.w3.org/2000/svg">
    <!-- .draw paths (pathLength="1") + .pop solids here -->
  </svg>
  <div class="caption"><span class="write">Caption text…</span></div>
</div>
<script src="_shared/gsap.min.js"></script>
<script>
(function(){
  const tl = gsap.timeline({ paused:true });
  // caption write-on → strokes draw → annotation → micro-motion breathe
  tl.to({}, { duration:5 }, 0);            // slot anchor (Rule 1)
  window.__timelines["sc1"] = tl;
})();
</script>
</body></html>
```

Micro-motion for the hold (every scene needs one): a slow ±10–15° ring
rotation, a 1.1x breathe on the hero shape (`sine.inOut`, `yoyo:true,
repeat:1`), or a gentle group wobble. Solids "pop" in with
`back.out(1.5)` from `scale:0.4–0.5` + opacity.

## ⚠ Gotchas (both cost a re-render — obey them)

1. **SVG transforms: use `svgOrigin`, NEVER a px `transformOrigin`.**
   `transformOrigin:'760px 560px'` on an SVG element makes GSAP resolve the
   origin against the element's own bounding box (transform-box ambiguity) —
   scaled/rotated shapes visibly drift off-center (a "centered" dot renders on
   the ring's edge). `svgOrigin:'760 560'` pins the origin in SVG user-space
   (= your cx/cy) and is always correct:
   ```js
   tl.to('.hole', { scale:1.12, svgOrigin:'760 560', duration:0.7,
                    ease:'sine.inOut', yoyo:true, repeat:1 }, 4.5);
   ```
2. **Don't trust `motion.screenshot` for this style.** Compositions whose
   every element starts hidden (dashoffset 1 / clip-path 100% / opacity 0)
   can screenshot as an empty paper frame even with `--atMs` mid-scene.
   That does NOT mean the animation is broken. The real verification is the
   render pipeline itself: the pre-render lint + post-render STATIC_RENDER
   gate catch a dead timeline, and `motion.verify-frames` on the rendered MP4
   shows the true animated frames. Render → verify-frames → read the PNGs.

Plus the standard contract (motion-philosophy.md): GSAP via
`_shared/gsap.min.js`, paused timeline, slot anchor, `window.__timelines`
registration, no randomness.

## Assembly & pacing

- 5s per scene is the sweet spot; a 20s explainer = 4 scenes, 30s = 6.
- Scene beat budget: caption 0.15–1.4s, main drawing 0.4–2.5s, annotation
  2.5–3.5s, breathe/hold 3.5–5s. Hard cut between scenes is fine (the fresh
  paper reads as "new page"); `fade-white` transition optional.
- Scenes are MAIN-TRACK clips (`project.add-clip`, from-scratch flow), NOT
  overlays. Renders are serial: one `motion.render-html` + `job.wait` at a time.
- Narration: if the user wants VO, generate it FIRST
  (`media.generate-narration`) and time each scene's caption + drawing to its
  line. Marker SFX at each scene start either way.
- 16:9 default; the style ports to 9:16 by stacking caption-over-drawing.
