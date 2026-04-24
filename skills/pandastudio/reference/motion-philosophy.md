# Motion Philosophy — the bar we author to

**Load this file BEFORE writing any motion graphic.** If you're about to call
`motion.generate` with an HTML template, `motion.render-html` with custom
HTML, or build a lower-third / intro / transition, the rules in this doc
decide whether the result looks premium or looks like default AI output.

The bar we're aiming for: the HeyGen HyperFrames launch aesthetic and the
Infinite Global Payments spot (`x-XTeSJ-DKx63uB4.mp4`, 1920×1080, 30fps,
900 frames). Black canvas, chrome-gradient type, motion-blurred whips,
object callbacks, held hero moments. One idea per beat, kinetic not still.

If you don't know what "good" looks like, the short answer is: **static text
fading in on a flat background is the lowest tier of motion graphics.** If
that's what you're about to author, stop and read this file.

---

## 0 · The 11 Laws (memorize)

1. **One idea per beat. Cut fast.** Average scene length ≈ 1.5 seconds in
   a reference-quality piece. Each visual lands ONE word or concept and
   moves on. If a scene is saying two things, split it.
2. **Black is the canvas.** ~90% of every frame is black or near-black.
   Negative space is the design. Color earns its place by carrying a
   message — decorative color cheapens.
3. **Light is the brand, not color.** Chrome gradients on type, soft halos,
   vignettes, light beams. The piece is *lit*, not *colored*. Light reads
   as premium.
4. **Camera never sleeps.** Even on "still" frames, the grid recedes,
   particles drift, the vignette breathes, the logo shimmers. Static =
   death. Every hold beat needs at least one micro-motion layer.
5. **Motion blur is a feature.** Every transition uses a streak / blur
   trail — to convey energy AND to mask the cut. Hard cuts and crossfades
   feel cheap; whip-streaks feel expensive.
6. **Object metaphors carry meaning.** A prop that appears early returns
   later — the same coin, the same card, the same icon. Visual continuity
   = brand. One callback minimum per piece.
7. **Palette is symbolic, not decorative.** ≤ 5 active hues across the
   whole piece, each one owning a concept. Don't add a color because it
   "looks nice" — assign it a meaning.
8. **Type is a character.** Words SCALE (8× for hero moments), MORPH,
   COMPRESS, GLOW. Typography drives ~60% of the storytelling. A
   text-only beat can be the strongest beat. Never fade flat text in and
   call it done.
9. **Hold the hero shot.** Logo reveal = 1.5–2s of stillness (with
   micro-shimmer). Outro card = 4–6s. Speed earns space for stillness to
   land. Kinetic chaos → calm = catharsis.
10. **One unifying texture across everything.** Reference pieces use a
    perspective grid + crosshair markers + vignette + grain on every
    scene. Even when invisible, that grid is the spine of the whole
    piece. Without a unifying texture you don't have a piece — you have
    clips.
11. **Timelines must fill their slots.** HyperFrames hides a
    sub-composition the moment `timeline.duration() < data-duration` →
    black-frame flash. Every GSAP timeline ends with
    `tl.to({}, { duration: SLOT_DURATION }, 0)` as a no-op duration
    anchor. Non-negotiable. Recipe + diagnostic in §4.

---

## 1 · The Visual Vocabulary (reusable moves)

Every premium motion piece assembles from a small catalog of primitives.
Don't invent — pick from the list, tune the numbers.

### 1.1 Backgrounds (always layered in this order, bottom → top)

| Layer | What it does | Recipe |
|---|---|---|
| **1. Base canvas** | Pure black or near-black stage | `body { background: #000 }` |
| **2. Perspective grid** | Hints at depth, establishes "lit stage" | `<div>` with `transform: perspective(900px) rotateX(60deg)` + two `repeating-linear-gradient`s (`rgba(255,255,255,.05) 0 1px, transparent 1px 80px` at 0° and 90°). GSAP animates `background-position-y` for parallax. |
| **3. Crosshair markers** | `+` registration marks at grid intersections — the house signature | 16 absolute-positioned `<div>` `+` shapes, or an SVG |
| **4. Scene content** | Your actual motion graphics | Tracks 2–8 in the HF data-attr model |
| **5. Vignette** | Darkens edges, focuses eye center, "breathes" | `<div>` `position: absolute; inset: 0;` with `background: radial-gradient(ellipse at center, transparent 30%, #000 95%); pointer-events: none;`. Animate opacity `0.7 ↔ 0.9` on a 4s `sine.inOut` yoyo loop. |
| **6. Grain** | Subtle film texture, unifies across scenes | `npx hyperframes add grain-overlay` if available, or three radial-gradients at tile sizes 3/5/7px with alphas `.03/.02/.015`. Deterministic, headless-safe. |

**Rule:** grid + vignette + grain appear on ≥ 60% of scenes. They are not
decoration — they are the *unifying texture*. Strip them and the piece
disintegrates into unrelated clips.

### 1.2 Type System

Typography does more storytelling than any other element. Get this right
and mediocre animation still reads premium. Get it wrong and the best
animation looks cheap.

- **Single sans-serif throughout a piece.** Geometric, generous tracking.
  `Inter`, `Suisse Int'l`, `SF Pro`, `Geist`, `Space Grotesk`. Per-beat
  font switching is an advanced technique — don't use it on v1.
- **All headline text uses chrome gradient:**
  ```css
  background: linear-gradient(180deg, #ffffff 0%, #999999 60%, #cccccc 100%);
  -webkit-background-clip: text;
  color: transparent;
  ```
  This alone takes flat-white text and makes it read 10× more expensive.
- **Halo glow on emphasis words:**
  ```css
  text-shadow:
    0 0 20px rgba(255,255,255,0.6),
    0 0 40px rgba(255,255,255,0.3);
  ```
- **Word-by-word kinetic reveal** (NOT character-by-character — words land harder):
  ```html
  <span class="word" data-start="0.0">Global</span>
  <span class="word" data-start="0.4">payments</span>
  <span class="word" data-start="0.8">are</span>
  <span class="word" data-start="1.0">a</span>
  <span class="word" data-start="1.2">pain</span>
  ```
  GSAP:
  ```js
  tl.from('.word', {
    y: 30, opacity: 0, scale: 0.85,
    duration: 0.6, ease: 'power3.out',
    stagger: 0.35
  });
  ```
- **Type SCALES dramatically.** Section labels at 48px. Body reveal at
  96–120px. Hero kinetic-type at 480px+ (literal screen-filling). Use
  `font-size: clamp()` and animate the `scale` transform.
- **No sentence case for emphasis.** Section words Title Case
  ("Sign Up In Seconds", "Built For Businesses").
- **Chrome-gradient sweep** (the hero-word move): 8-stop gradient with
  dark bookends, animate `background-position` from `100% 0` → `0% 0`
  over 0.6s `power2.out` with `stagger: 0.04` across words. Dark bookends
  prevent edge-tear when the sweep finishes.
  ```css
  background: linear-gradient(90deg,
    #14110a 0%, #14110a 15%,
    #5a3215 25%, #c84f1c 40%, #e2b53f 55%, #2a8a7c 70%,
    #14110a 85%, #14110a 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-size: 300% 100%;
  background-position: 100% 0;  /* resting state */
  ```

### 1.3 Color Story

5 hues maximum per piece, each with a meaning. A working palette:

| Role | Example hex | When |
|---|---|---|
| Canvas | `#000` / `#0a0a0a` | Always, every frame |
| Brand primary (chrome) | `#fff → #999` gradient | All headline type |
| Accent 1 (problem / alert) | `#e10b1f` red | Thing we're solving |
| Accent 2 (solution / brand core) | `#33d4c8` teal | The answer / product |
| Accent 3 (energy / speed) | `#a155ff` magenta | High-velocity beats, API feel |

Rule: when adding a new scene, ask "what's the one color carrying this
beat?" If you can't name it, you haven't earned it. Kill the color.

### 1.4 Motion Vocabulary (named moves with GSAP recipes)

These are the reusable primitives. Master ~5 of them and you can build
most of the reference pieces.

| Move | What it does | GSAP recipe |
|---|---|---|
| **Camera dolly through type** | Text grows 1× → 8× and fades to 0 at peak — feels like flying through a 3D word | `tl.fromTo(text, { scale: 1, opacity: 1 }, { scale: 8, opacity: 0, duration: 1.5, ease: 'power2.in' })` |
| **Light-streak whip** | Glowing bar zips across frame in 0.3–0.4s, blurred, hides the cut | `gsap.fromTo('.streak', { xPercent: -100, scaleX: 0.5 }, { xPercent: 250, scaleX: 1.5, duration: 0.4, ease: 'power3.in' })` — fire AT the cut, not before |
| **Word ghost reveal** | Word #2 starts entering while word #1 is still leaving (0.15s overlap) | `stagger: 0.35` on reveal + each word exits at `+= 0.5` → overlap by 0.15s |
| **Chrome-gradient sweep** | Hero-word rolls chrome left-to-right | See §1.2 recipe |
| **Object morph drift** | A shrinks + drifts one way, B grows + drifts from same vector, light streak hides the swap | Two divs, GSAP `to(A, {x:200, scale:0.5, opacity:0})`, `from(B, {x:-200, scale:0.5, opacity:0})`, both 0.5s |
| **Crystallize → wordmark** | Element shrinks / translates into logo position while wordmark fades up | Two aligned timelines: coin scales 1→0.3 + moves to logo X/Y; wordmark fades 0→1 |
| **Energy pulse along path** | Glow travels along a network edge, target node lights up at arrival | SVG path with `stroke-dasharray` + `stroke-dashoffset` tween 1→0; node `boxShadow: '0 0 30px var(--glow)'` at path end |
| **Color recolor (no cut)** | Same scene, palette shifts via CSS custom properties — reads as a new beat without a new cut. *Highest ROI move in the vocabulary.* | `tl.to(':root', { '--accent': '#ff9430', '--edge': '#ffd84a', duration: 0.6 }, T)` |
| **Slide-up phone reveal** | Device mockup enters from bottom edge with headline above | `tl.from(phone, { y:'100%', duration:1, ease:'power3.out' }, 0).from(headline, { y:30, opacity:0 }, 0.3)` |
| **Wheel + side-panel** | Central UI rotates while labeled text panels slide in L/R | `tl.to(wheel, { rotation:120, duration:1.5 }).from(panel, { x:-100, opacity:0 }, '<0.3')` |
| **Floating cluster drift** | 3+ objects gently bob continuously — keeps "still" frames alive | `gsap.to(coins, { y:'-=15', duration:2, repeat:-1, yoyo:true, ease:'sine.inOut', stagger:{each:0.4, from:'random'} })` |
| **Vignette breath** | Vignette opacity wobbles to prevent stillness | `gsap.to(vignette, { opacity:0.9, duration:4, repeat:-1, yoyo:true, ease:'sine.inOut' })` |
| **Cut-the-curve vertical whip** | Default adjacent-beat transition: exit rides up w/ blur, entry rises from below w/ matching blur, same direction, velocity matched | **Exit:** `tl.to(wrap, { y:-150, filter:"blur(30px)", duration:0.33, ease:"power2.in" })` **Entry:** `gsap.set(wrap, { y:150, filter:"blur(30px)" })` then `tl.to(wrap, { y:0, filter:"blur(0px)", duration:1.0, ease:"power2.out" }, 0)` |
| **Faux-cursor click event** | 7-tween sequence selling a UI interaction (click, ripple, overshoot, settle) | See §3 of this doc for full recipe |

### 1.5 Pacing Discipline (numeric)

- **Default scene length:** 1.0–2.0s. Longer only for hero moments + outro.
- **Reveal cadence inside a scene:** new visual element every 0.3–0.6s.
  No dead air > 1s mid-piece.
- **Word-reveal stagger:** 0.3–0.4s per word for narrative reads.
  0.5–0.6s for dramatic single-word emphasis.
- **Whip transition duration:** 0.3–0.4s. Faster = glitchy. Slower = loses energy.
- **Hold durations:**
  - Logo crystallization: 1.5–2s
  - Final CTA card: 4–6s (the longest single shot in the piece is the outro)
  - Section headlines: 1–1.5s of read time after fully revealed
- **Breathing rule:** every ~7–8s of kinetic density, give the viewer a 1s
  rest beat (brand reveal, coin spin, outro hold).

---

## 2 · Easing Dictionary (by purpose)

Don't guess eases. These are the exact values from reference-quality work.

| Purpose | Ease | Typical duration |
|---|---|---|
| Word reveal (slide-in) | `expo.out` | 0.20–0.33s |
| Generic element enter | `power2.out` / `power3.out` | 0.2–0.5s |
| Generic element exit | `power2.in` | 0.2–0.33s |
| Beat-to-beat whip EXIT | `expo.in` / `power2.in` | 0.2–0.33s |
| Beat-to-beat whip ENTRY | `expo.out` / `power2.out` | 0.5–1.0s |
| Camera pan between stops | `power2.inOut` | 1.2–2.3s |
| Linear hold (after entry) | `"none"` | 0.4–0.65s |
| Bouncy card settle | `back.out(1.2)` to `back.out(1.5)` | 0.3–0.5s |
| Click compress | `power4.in` | **0.07s (70ms)** |
| Click release (overshoot) | `back.out(3)` | 0.30s |
| UI overshoot (settle) | `elastic.out(1, 0.3)` to `elastic.out(1, 0.4)` | 0.20s |
| Cursor blink | `steps(1)` yoyo | repeat, 0.4s on / 0.4s off |
| Continuous rotation | `"none"` | full beat |
| Breathe / drift | `sine.inOut` yoyo | 2–4s, `repeat: -1` |

**Stagger values (reference roster):**

- Chrome-gradient sweep across words: `stagger: 0.04`
- Dot-grid ripple per-column: `0.019`; within-column: `0.004`
- Code-stream lines: `stagger: 0.06`
- Scaffold output lines: `0.08–0.12s` explicit delays (not stagger —
  lines need individual eases)
- Harmonic bar spikes: `(i % 12) * 0.008`

**Defaults pattern:** don't use `gsap.defaults()`. Every tween declares its
ease and duration explicitly. Inheritance bugs are harder to diagnose than
verbose tweens.

**Rule of thumb for scene pairing:** at least 3 different eases appear in
any non-trivial scene. All-`power2.out` feels robotic.

---

## 3 · Concrete Recipes You'll Reach For

### 3.1 Timeline skeleton (the law-#11 anchor is REQUIRED)

```js
(() => {
  const tl = gsap.timeline({ paused: true });
  // … all your tweens …
  tl.to({}, { duration: SLOT_DURATION }, 0);  // Law #11 anchor
  window.__timelines['<data-composition-id>'] = tl;
})();
```

Always at the end of the IIFE. Key must match `data-composition-id`
exactly. The anchor is a no-op tween whose only purpose is to keep
`timeline.duration() >= data-duration`, preventing HF from hiding the
sub-composition with a black-frame flash.

### 3.2 Faux-cursor click event (7 tweens)

Sells a UI interaction as a deliberate, physical click. Use for product
demos, tool intros, "watch me click this button" beats.

```js
// 1. Cursor compress
tl.to(cursor, { scale: 0.82, duration: 0.07, ease: 'power4.in' }, T);
// 2. Target compress
tl.to(target, { scale: 0.96, duration: 0.07, ease: 'power4.in' }, T);
// 3. Ripple expand
tl.set(ripple, { scale: 0, opacity: 0.9 }, T);
tl.to(ripple, { scale: 2.5, opacity: 0, duration: 0.2, ease: 'power2.out' }, T);
// 4. Cursor release (overshoot)
tl.to(cursor, { scale: 1, duration: 0.3, ease: 'back.out(3)' }, T + 0.07);
// 5. Target overshoot
tl.to(target, { scale: 1.02, duration: 0.2, ease: 'elastic.out(1, 0.4)' }, T + 0.07);
// 6. Target settle
tl.to(target, { scale: 1, duration: 0.2, ease: 'power2.out' }, T + 0.27);
// 7. Micro-delay before subsequent effect fires
```

**Critical:** SVG cursor's `transform-origin` must sit at the click-tip
pixel (e.g. `transform-origin: 4.3px 2.6px`). If it's at the SVG centre,
the click looks like it clipped above or below the target.

### 3.3 Color recolor (no cut) — highest-ROI move

```html
<div class="flowchart" style="--edge: #5db4ff; --node-glow: rgba(91,180,255,.6);">
  <!-- nodes use var(--edge) for borders, var(--node-glow) for box-shadow -->
</div>
```

```js
// At t=2.5s, recolor the entire scene in one tween
tl.to('.flowchart', {
  '--edge': '#ffd84a',
  '--node-glow': 'rgba(255,148,48,0.7)',
  duration: 0.6,
  ease: 'power2.inOut'
}, 2.5);
```

Same DOM, same composition, but the meaning shifts (Global → Affordable,
Before → After, Idle → Active). Cheap to build, expensive to look at.

### 3.4 Velocity-matched seams (advanced)

When an entry tween hands off to a linear hold (or to another tween),
derive a custom ease so end-velocity matches the next tween's
start-velocity. The eye reads any velocity discontinuity as a stall.

```js
// Entry 0→1 over 0.9s handing off to a linear hold at v=−0.123/s
// Match at seam: p'(1) = 0.194.  p(t) = a·t² + b·t → a=−0.806, b=1.806.
const entryEase = (t) => -0.806 * t * t + 1.806 * t;

tl.to(card, { z: -50, duration: 0.9, ease: entryEase }, 0);
tl.to(card, { z: -80, duration: 0.65, ease: 'none' }, 0.9);
```

Worth it for hero beats. Not worth it for every tween.

### 3.5 Tween-comment convention

Every entry/exit tween's comment explicitly names the matching tween in
the adjacent beat. Discoverable seamlessness months later.

```js
// ENTRY — blur de-ramps from 22px to match the outgoing "circle" beat's 22px exit.
// Leftward motion carried by "Anything" sliding in from x: 360.
gsap.set(wrap, { filter: 'blur(22px)' });
tl.to(wrap, { filter: 'blur(0px)', duration: 0.33, ease: 'power2.out' }, 0);

// EXIT (at local 1.89) — matches incoming "engine" beat's 150px y + 30px blur entry.
tl.to(wrap, { y: -150, filter: 'blur(30px)', duration: 0.33, ease: 'power2.in' }, 1.89);
```

**Rule:** if you can't name the matching tween in a comment, you haven't
designed the seam. Go back and design it.

---

## 4 · Pre-flight checklist — run BEFORE claiming done

No motion graphic ships without every one of these passing. If you can't
tick a box, fix it before calling `export.start`.

- [ ] **Average scene length ≤ 2s** in mid-section (intro/outro may hold longer)
- [ ] **No dead air > 1s** anywhere except deliberate hold moments
- [ ] **Every transition uses motion** (whip, morph, slide, recolor — no hard fades)
- [ ] **Color palette ≤ 5 active hues**, each with a meaning
- [ ] **All headline text uses chrome gradient + halo** — no flat white
- [ ] **Background grid + crosshairs present in ≥ 60% of scenes**
- [ ] **Vignette layer on top of every scene**
- [ ] **Grain overlay on every scene** (subtle but there)
- [ ] **≥ 1 callback** — a visual element that returns later
- [ ] **Outro holds ≥ 4 seconds** with micro-motion (shimmer, breath)
- [ ] **Every sub-composition timeline ends with the Law #11 anchor**
- [ ] **All tween end-times snap to multiples of `1/fps`** (at 30fps:
      0.0333, 0.0667, 0.1, 0.1333… — steep-tail eases like `expo.in`
      alias visibly at sub-frame boundaries)
- [ ] **Ran the timeline-duration diagnostic** (see below)
- [ ] **Extracted frames and actually looked at them** — see
      `motion.verify-frames` verb. Lint passing ≠ design working. No
      ship without frame inspection.

### Timeline-duration diagnostic

Open preview, devtools console, and verify every timeline fills its slot:

```js
const p = document.querySelector('hyperframes-player');
const iw = p.shadowRoot.querySelector('iframe').contentWindow;
Object.fromEntries(Object.entries(iw.__timelines).map(([k, v]) =>
  [k, +v.duration().toFixed(4)]));
```

Any gap where `timeline.duration() < data-duration` is a black-frame risk
and must be fixed before export.

---

## 5 · The "What Would Reference Work Do?" sanity check

Before you ship any motion graphic, ask:

1. **Could I cut this scene in half?** If yes, do it.
2. **Is this color carrying a meaning?** If no, kill it.
3. **Does the camera move during this beat?** If no, add drift / shimmer / vignette breath.
4. **Where's the motion blur?** If there's none on the transition, add a whip streak.
5. **Will the viewer see this same visual element again later?** If no, can I make a callback?
6. **Does the type SCALE during its reveal, or just fade in?** Fading in is the lowest-effort move. Scale.
7. **What's the negative space doing?** If 80%+ of the frame isn't black/dark, you're over-designing.
8. **Is this beat ONE idea?** If you can't summarize it in 2 words, you've packed too much in.
9. **What's the unifying texture across all scenes?** If you don't have one, you don't have a piece — you have clips.
10. **Where does the viewer rest?** Identify the breathing moments. If there are none, build them in.

---

## 6 · Anti-patterns (what NOT to do)

- ❌ **Centered, axis-aligned, motionless text fades.** This is the lowest
  motion-graphics tier. Always add scale, blur, or directional energy.
- ❌ **Hard cuts between scenes.** Every cut needs a transition element
  bridging it (streak, morph, wipe, shader).
- ❌ **Flat white text on flat black background.** Use chrome gradient +
  halo glow. Costs nothing. Looks 10× more expensive.
- ❌ **6+ colors across the piece.** You're decorating, not communicating.
- ❌ **No callbacks.** A visual element that appears once and never
  returns wastes its setup.
- ❌ **Outro that ends on the last beat.** Always hold the CTA for 4+ seconds.
- ❌ **Scenes that try to say two things.** Split into two scenes.
- ❌ **Static backgrounds.** Every background has at least one slow-moving
  element (parallax grid, drifting particles, breathing vignette).
- ❌ **Decorative grain or vignette.** They're not decoration — they're
  the unifying *texture*. Every scene, every time.
- ❌ **Forgetting to render a draft and look at it.** Lint passing ≠
  design working. **VIEW THE FRAMES.**
- ❌ **`Math.random()` / `Date.now()` / unseeded PRNGs inside a render
  loop.** Renders must be deterministic frame-to-frame. Use harmonic
  hashes instead:
  `80 + 220 * Math.abs(Math.sin(i*0.7 + 0.3) * Math.cos(i*1.3 + 0.7))`
- ❌ **Animating `<video>` dimensions directly.** Wrap the `<video>` in a
  `<div>` and animate the wrapper. Direct `<video>` animation freezes
  the decoder and the video pauses while audio continues.
- ❌ **Using `window.__hf.seek` / `window.__hf.duration` directly.** That
  bypasses the compositor invalidation wrapper — animations render with
  1-second stalls. ALWAYS register through `window.__timelines[id]` and
  let the paused-timeline + HF tick loop drive seek.
- ❌ **Leaning on registry blocks for hero moments.** The HyperFrames
  launch reference installs ZERO registry blocks — every scene is
  hand-built. Registry = production velocity. Bespoke = reference bar.

---

## 7 · How PandaStudio's agent authors motion graphics

**`motion.render-html` is the only verb on your surface.** Do not call
`motion.generate` / `motion.list` / `motion.themes` — those are for the
in-app Gemma E2B local model (which can't write code). The bundled
templates are v0 quality and do not meet the bar in this document.

For every motion graphic, regardless of type or complexity:

1. Load this doc (if not already loaded).
2. If authoring for a specific delivery format (9:16 shorts, 16:9
   YouTube side-overlay), also load `reference/video-authoring.md`.
3. Start from the canonical shell below — grid + vignette + grain + Law
   #11 anchor are mandatory, not optional.
4. Add scene content on tracks 2–8. Use vocabulary from §1.4. Pick
   eases from §2. Respect pacing floors from §1.5.
5. After rendering, run `motion.verify-frames` and READ each frame.
6. Run the pre-flight checklist (§4) before declaring done.

Same applies for any scene: lower thirds, intro cards, stat reveals,
sponsor reads, chapter dividers, subscribe CTAs, outro cards, spotlight
rings, reaction bursts. Every one gets the 11 Laws treatment. No
"this is just a small lower third, I'll skip the grid" — the grid is
cheap and it's what separates template-filled from reference-quality.

### Canonical composition template (use verbatim)

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Composition</title>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
<style>
  /* brand tokens — override per project */
  :root {
    --canvas: #000;
    --primary: #ffffff;
    --accent: #33d4c8;
    --energy: #a155ff;
    --alert: #e10b1f;
    --chrome: linear-gradient(180deg, #fff 0%, #999 60%, #ccc 100%);
  }
  html, body { width: 100%; height: 100%; margin: 0; background: var(--canvas); overflow: hidden; }
  .stage { position: relative; width: 100%; height: 100%; font-family: Inter, system-ui, sans-serif; }

  /* unifying texture — mandatory */
  .grid-floor {
    position: absolute; inset: 0;
    transform: perspective(900px) rotateX(60deg) translateY(20%);
    background:
      repeating-linear-gradient(0deg,  rgba(255,255,255,.05) 0 1px, transparent 1px 80px),
      repeating-linear-gradient(90deg, rgba(255,255,255,.05) 0 1px, transparent 1px 80px);
    opacity: 0.6;
  }
  .vignette {
    position: absolute; inset: 0; pointer-events: none;
    background: radial-gradient(ellipse at center, transparent 30%, #000 95%);
  }
  .grain {
    position: absolute; inset: 0; pointer-events: none; opacity: 0.04;
    background:
      radial-gradient(#fff 1px, transparent 1px) 0 0 / 3px 3px,
      radial-gradient(#fff 1px, transparent 1px) 1px 1px / 5px 5px,
      radial-gradient(#fff 1px, transparent 1px) 2px 2px / 7px 7px;
  }

  /* chrome-gradient headline */
  .h-chrome {
    font-weight: 700;
    background: linear-gradient(180deg, #fff 0%, #999 60%, #ccc 100%);
    -webkit-background-clip: text;
    color: transparent;
    text-shadow: 0 0 20px rgba(255,255,255,0.3), 0 0 40px rgba(255,255,255,0.15);
  }
</style>
</head>
<body>
<div class="stage" data-composition-id="root" data-width="1920" data-height="1080" data-duration="6">
  <div class="grid-floor clip" data-start="0" data-duration="6" data-track-index="0"></div>
  <div class="grain      clip" data-start="0" data-duration="6" data-track-index="1"></div>
  <!-- your content here, tracks 2–8 -->
  <div class="vignette   clip" data-start="0" data-duration="6" data-track-index="9"></div>
</div>

<script>
(() => {
  const SLOT_DURATION = 6; // must match data-duration
  const tl = gsap.timeline({ paused: true });

  // ambient life — vignette breath (law #4)
  gsap.to('.vignette', { opacity: 0.85, duration: 4, repeat: -1, yoyo: true, ease: 'sine.inOut' });

  // … your tweens here …

  // LAW #11 anchor — do not remove
  tl.to({}, { duration: SLOT_DURATION }, 0);
  window.__timelines['root'] = tl;
})();
</script>
</body>
</html>
```

Start every bespoke composition from this template. The grid, vignette,
and grain are mandatory. The anchor tween is mandatory. Add content on
tracks 2–8.

---

## 8 · TL;DR — the philosophy in one sentence

> **One idea per beat, lit not colored, kinetic not still, callbacks not
> novelty, hold the hero, breathe the outro — the grid is always under
> everything, every timeline fills its slot, every exit snaps to a frame
> boundary, and every cut hides inside a motion-blurred whip.**

Load this when you load motion-philosophy.md. Then go build.
