# Motion Philosophy — the bar we author to

> **What changed (v1.32):** this file used to mix structural rules with strong
> aesthetic prescriptions ("dark-premium black canvas", "chrome-gradient
> headlines", "perspective grid + vignette + grain on every scene"). Those
> prescriptions made every output look the same and read as generic — they were
> the cause of the meh promo videos we kept hitting. The new contract is
> **brand-driven**: read the user's brand from the editor context (set in
> Settings → Integrations → Brand kit), and let the brand decide colors, type,
> mood. The rules here are about correctness and craft, not taste.

**Load this file BEFORE writing any motion graphic** — `motion.generate` with a
template, `motion.render-html` with custom HTML, lower-thirds, intros,
transitions. The rules below decide whether the result feels intentional or
defaulted.

> **Authoritative engine docs.** PandaStudio's motion-graphic pipeline is
> [Hyperframes](https://hyperframes.heygen.com/) (HeyGen's open-source HTML→video
> renderer). The composition / determinism / timeline rules in this file mirror
> their published guides — when in doubt, those win.
>
> - [Examples gallery](https://hyperframes.heygen.com/examples) — the aesthetic
>   minimum every motion graphic must clear
> - [Prompting guide](https://hyperframes.heygen.com/guides/prompting)
> - [Common mistakes](https://hyperframes.heygen.com/guides/common-mistakes)
> - [Hyperframes core SKILL.md](https://github.com/heygen-com/hyperframes/blob/main/skills/hyperframes/SKILL.md)

If you don't know what "good" looks like, the floor is: **static text fading in
on a flat background is the lowest tier of motion graphics, in any style.** If
that's what you're about to author, stop and read this file.

---

## Approach

Every motion-graphic job follows the same five steps. Don't skip any.

### Step 1 — Discovery (for exploratory briefs only)

For specific requests ("add a title card with this exact text", "fix the timing
on scene 3", "regenerate scene 2 with a different stat"), skip this step and go
to Step 2.

For open-ended requests ("make me a product promo", "create something for the
intro"), understand intent before picking pixels:

- **Audience** — who watches this? Developers? Executives? General consumers?
- **Platform** — where does it play? YouTube, Shorts/TikTok/Reels, a hero on the
  landing page, an internal demo loop?
- **Priority** — what matters most: motion quality, content accuracy, brand
  fidelity, speed of turnaround?
- **Variations** — does the user want options to choose from, or a single best
  shot?

Ask two or three of these and wait. Don't ask five at once — the user gets
fatigued and stops engaging. For exploratory briefs you may **offer 2–3
variations** that differ meaningfully (different pacing, different energy levels,
different structural approaches — not just color swaps). Don't mandate this;
it's a tool, not a rule.

### Step 2 — Brand kit

**Read the brand kit from the editor-context snapshot** (the one you receive at
the top of every turn). It carries:

- `name`, `tagline`
- `colors`: `primary`, `accent`, `ink`, `background` (any subset may be set)
- `typography`: `display`, `body` (any subset may be set)
- `voice`: one of `minimal | bold | editorial | casual | corporate | playful`
- `logoPath`: an absolute path; pass it via `motion.render-html --assets` when a
  scene needs the logo on screen

> **Use these values verbatim.** Don't invent hex codes, don't substitute fonts.
> If the brand says `primary: #6B4FFF`, that purple is the dominant accent. If it
> says `display: Bricolage Grotesque`, use Bricolage Grotesque — not Inter.

**When the brand kit is empty** (or missing fields like `colors.accent`), the
editor-context block tells you so. **Do not invent a default look.** Three valid
moves, in priority order:

1. **Ask the user inline** for the missing field (`"Quick — what's the brand
   color? Drop a hex code, or 'use default'."`).
2. **Fall back to the neutral house-style defaults** in [house-style.md](./house-style.md).
   Load that file when at least one brand field is missing AND the user said "just
   pick something."
3. **Refuse** if the user explicitly asked for branded content (e.g., "make the
   outro with my logo") and the brand has no `name`/`logoPath`. Surface the
   blocker before authoring.

`voice` is the most powerful single field. It drives motion energy and copy
tone, not specific recipes. See [house-style.md](./house-style.md) §"Voice → motion
defaults" for what each voice typically implies — those are STARTING points, not
mandates.

### Step 3 — Plan

Before writing HTML, think at a high level:

1. **Story** — what should the viewer experience? Identify the narrative arc, the
   key moments, the emotional beats.
2. **Structure** — how many scenes? Which are full-bleed scenes (cover the frame),
   which are companion graphics (overlay a talking host)? **For anything with 3
   or more distinct beats (typical promos, explainers, anything ≥10s total),
   plan to render each scene as a SEPARATE motion graphic and add each as a
   sequential clip on the timeline** — not one monolithic 30s MP4. See
   "Multi-scene authoring" below for the why and the how.
3. **Rhythm** — declare the scene rhythm before implementing. Which scenes are
   quick hits, which are holds, where does energy peak? Name the pattern:
   `fast-fast-SLOW-fast-shader-hold`. Pacing is *track-aware* — see §"Pacing" below.
4. **Timing** — total duration, where transitions land, which clips drive the
   length.
5. **Layout** — build the end state first (see "Layout Before Animation" below).
6. **Animate** — only now add motion using the rules below.

**Build what was asked.** A request for "a title card" is not a request for
"a title card + 3 supporting scenes + ambient music + captions." Every scene,
every element, every tween earns its place. If something additional would
genuinely improve the piece, **propose** it — don't add it.

For small edits (one color tweak, one timing adjustment, one element added),
skip directly to the rules.

### Multi-scene authoring (the default for promos)

If the brief is multi-scene (intro + problem + demo + CTA, or any explainer
arc with ≥3 distinct beats), **render each scene as its own motion graphic
and add each as a separate clip on the timeline.** Don't build one big 30s
HTML composition.

```bash
# From-scratch promo (no host footage): scenes become MAIN TRACK clips via
# project.add-clip. NOT add-motion-graphic — that's for graphics layered on
# top of existing footage (lower thirds, callouts), and there's no footage
# here. Adding overlays to an empty main track produces a project the
# editor opens to "No video to load" because overlays compose ON the main
# track and there's nothing under them.
for SCENE in intro problem demo cta; do
  JOB=$(pandastudio motion.render-html --htmlPath="/tmp/$SCENE.html" \
    --durationMs=6000 --json | jq -r '.data.jobId')
  RESULT=$(pandastudio job.wait --id="$JOB" --timeoutMs=600000 --json)
  OUTPATH=$(echo "$RESULT" | jq -r '.data.job.result.outputPath')
  # Add as a primary clip on the main track. add-clip ffmpeg-probes the
  # MP4 duration; clips append at the end by default (or pass --atIndex).
  pandastudio project.add-clip --id="$PROJECT" --media="$OUTPATH"
done
```

(When the brief is "add a lower third over this recording," the host
video already sits on the main track — use `project.add-motion-graphic`
to layer the graphic on top instead. The verb choice is the entire
distinction between the two flows.)

Why this is the default for promos:

- **Targeted re-renders.** User says "scene 3 needs to be brighter" → you
  re-author and re-render that one scene (~30-60s) and swap the clip on the
  timeline. Versus a 30s monolith where one tweak forces a full 10-min redo.
- **The editor's primitives apply per-clip.** Each scene gets its own trim,
  speed region, FX, zoom, layout-transform. A single combined MP4 wastes
  all of that.
- **Reordering is a drag.** Literally. The user moves scenes around on the
  timeline; you don't need to rebuild a unified HTML.
- **Render ceiling.** `motion.render-html` for a 30s @ 1080p composition
  pushes the engine hard (memory + capture time). Six 5-second renders are
  cheaper than one 30s render, and each is small enough to render reliably
  on slower machines.
- **Parallel-friendly.** Render-pool supports multiple concurrent jobs on
  Apple Silicon — multiple short scenes can be in flight at once where a
  single long render is fully serial.

When you DO want a single combined MP4 (e.g., user explicitly says "give me
one file I can post directly"), use `motion.concat` to stitch the scene MP4s
together — lossless, sub-second, no re-encode. But the timeline-first path
is the default; concat is the escape hatch.

**Single-scene graphics** (a lower third, a title card, a stat reveal, a
chapter divider) stay a single render → single clip. The multi-scene
pattern is only for compositions with multiple named beats.

### Step 4 — Layout Before Animation

Position every element where it should be at its **most visible moment** — the
frame where it's fully entered, correctly placed, not yet exiting. Write this as
static HTML+CSS first. **No GSAP yet.**

**Why this matters:** if you position elements at their animated *start* state
(offscreen, scaled to 0, opacity 0) and tween them to where you think they
should land, you're guessing the final layout. Overlaps stay invisible until the
video renders. By building the end state first, you can see and fix layout
problems before adding any motion.

The process:

1. **Identify the hero frame** for each scene — the moment when the most
   elements are simultaneously visible. That's the layout you build.
2. **Write static CSS** for that frame. The `.scene-content` container fills the
   full scene using `width: 100%; height: 100%; padding: Npx;` with
   `display: flex; flex-direction: column; gap: Npx; box-sizing: border-box`.
   Padding pushes content inward — never `position: absolute; top: Npx` on a
   content container. Absolute-positioned content containers overflow when the
   content is taller than the remaining space. Reserve `position: absolute` for
   decoratives only.
3. **Add entrances with `gsap.from()`** — animate FROM offscreen/invisible TO the
   CSS position. The CSS position is the ground truth; the tween describes the
   journey to get there.
4. **Add exits with `gsap.to()`** only on the final scene (see §"Scene
   transitions" below). All other scene-to-scene exits are handled by the
   transition.

```css
/* RIGHT — flex + padding fills any scene size deterministically. */
.scene-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 120px 160px;
  gap: 24px;
  box-sizing: border-box;
}

/* WRONG — hardcoded dimensions and absolute positioning. */
.scene-content {
  position: absolute;
  top: 200px;
  left: 160px;
  width: 1920px;
  height: 1080px;
  /* breaks when aspect or size changes; content overflows; not portable */
}
```

```js
// Animate INTO the resting positions.
tl.from(".title",    { y: 60, opacity: 0, duration: 0.6, ease: "power3.out" }, 0);
tl.from(".subtitle", { y: 40, opacity: 0, duration: 0.5, ease: "power3.out" }, 0.2);
tl.from(".logo",     { scale: 0.8, opacity: 0, duration: 0.4, ease: "power2.out" }, 0.3);
```

### Step 5 — Author

Write the HTML. Add the GSAP timeline. Set `data-duration` to the slot length.
Anchor the timeline with the no-op tween (see Rule 1 below). Verify with the
quality checks in §"Quality" before claiming done.

---

## Rules (non-negotiable)

These are about correctness, not taste. Violating any one of them produces a
broken composition — black-frame flashes, time-wobble, dropped scenes, off-by-
frames.

1. **Timelines fill their slot.** HyperFrames hides a sub-composition the moment
   `timeline.duration() < data-duration` → black-frame flash. **Every** GSAP
   timeline ends with a no-op duration anchor:

   ```js
   const SLOT = 6;  // matches data-duration on the <stage>
   const tl = gsap.timeline({ paused: true });
   tl.from(...);  // entrance
   tl.to(...);    // any held mid-beats
   tl.to({}, { duration: SLOT }, 0);  // anchor — DO NOT REMOVE
   window.__timelines["scene-id"] = tl;
   ```

   The anchor goes at position 0 with the full slot duration so the timeline's
   reported `duration()` is exactly `SLOT`, regardless of what other tweens
   exist.

2. **Determinism is absolute.** No `Math.random()`, `Date.now()`,
   `performance.now()`, `setInterval`, `setTimeout`, async/await, or
   Promise-driven side effects in the render body. HyperFrames seeks
   deterministically frame-by-frame — every read must be reproducible.
   Seed any randomness with a pure PRNG (mulberry32 etc.). Drive everything
   off the paused timeline.

3. **No `repeat: -1`.** Infinite loops break the capture engine. Calculate the
   exact count from slot duration:
   `repeat: Math.ceil(SLOT / cycleDuration) - 1`.

4. **`{ paused: true }` on every timeline.** The player controls playback. Never
   call `tl.play()` / `tl.pause()` / `tl.seek()` from your code.

5. **Register every timeline.** Even single-scene compositions must do
   `window.__timelines["<composition-id>"] = tl`. Sub-compositions in
   sub-composition files do the same. Framework auto-nests sub-timelines — do
   NOT add them by hand.

6. **Synchronous construction.** Build the timeline at module top-level, not
   inside `async`, `setTimeout`, or `Promise.then`. The capture engine reads
   `window.__timelines` synchronously after page load. Fonts are embedded by
   the compiler — no need to wait.

7. **GSAP animates visuals only.** `opacity`, `x`, `y`, `scale`, `rotation`,
   `color`, `backgroundColor`, `borderRadius`, transforms. Do NOT animate
   `visibility`, `display`, or call `video.play()`/`audio.play()` from a tween.

8. **One animator per property.** Never animate the same property on the same
   element from multiple timelines simultaneously.

9. **Muted + playsinline on `<video>`.** Audio is always a separate `<audio>`
   element. If you need a video's audio, declare both elements.

10. **Use `tl.set(selector, vars, time)` for clip-scoped initial states**,
    never top-level `gsap.set()` on later-scene elements. Later scenes don't
    exist in the DOM at page load.

11. **No `<br>` inside flowing body text.** Forced breaks don't account for
    rendered font width — text that wraps naturally + a `<br>` produces an
    unwanted extra break and overlap. Use `max-width` and let it wrap.
    Exception: short display titles where each word is deliberately on its own
    line ("THE / IMMORTAL / GAME" at 130px).

---

## Scene Transitions

There are TWO valid transition models. Pick the one that matches your
authoring mode:

#### Multi-clip on the timeline (default for promos)

Each scene is its own MP4 clip on the timeline. Transitions live as editor
regions between them.

1. **Each scene's HTML is reveal + hold only.** Every element animates IN
   via `gsap.from()`. After the reveal completes, the scene HOLDS its final
   composition for the remainder of `data-duration` (use the Law-#1
   anchor `tl.to({}, { duration: SLOT }, 0)` to fix the timeline length).
2. **No baked-in exit tweens.** No `gsap.to(..., { opacity: 0 })`, no
   off-screen y, no scale-to-zero. The held last frame is what the editor's
   crossfade region blends FROM into the next clip's first frame.
3. **Transitions are editor regions.** A crossfade between scene 1 and
   scene 2 is a `project.add-fx` region positioned at the boundary. The
   user can swap, retime, or remove transitions without re-rendering any
   scene.
4. **Closing fade** (if any) — the LAST scene may fade out to black at
   its end via a tween; the editor doesn't need a region for that.

#### Single combined render (single-clip path)

One monolithic HTML composition with multiple GSAP-orchestrated scenes,
rendered to a single MP4 that lands as one clip on the timeline. Use when
the brief is short (≤10s) or when you need a transition technique GSAP
can do that the editor can't (motion-blur whip-streak between scenes, a
WebGL shader transition, etc.).

In this mode:

1. Same entrance-on-every-scene rule.
2. **The transition IS the exit** — implement the whip-streak / wipe /
   shader between scenes IN the GSAP timeline. The outgoing scene's
   content must still be fully visible at the moment the transition
   starts.
3. **No bare exit tweens** on intermediate scenes — they fade out by
   being COVERED by the transition, not by emptying themselves first.
4. Only the final scene may fade elements out.

```js
// RIGHT (single-clip path) — entrance only, transition handles the exit.
tl.from("#s1-title",    { y: 50, opacity: 0, duration: 0.7, ease: "power3.out" }, 0.3);
tl.from("#s1-subtitle", { y: 30, opacity: 0, duration: 0.5, ease: "power2.out" }, 0.6);
// NO exit tweens here.
tl.from("#s2-heading",  { x: -40, opacity: 0, duration: 0.6, ease: "expo.out" }, 8.0);

// WRONG — empties the scene before the transition can use it.
tl.to("#s1-title", { opacity: 0, y: -40, duration: 0.4 }, 6.5);
tl.to("#s1-subtitle", { opacity: 0, duration: 0.3 }, 6.7);
// Transition then fires on an already-empty frame.
```

---

## Animation guardrails

Aesthetic guidance, not law. Break them when the brand or brief justifies it,
but justify it.

- **Offset the first animation 0.1–0.3s** (not t=0). A scene starting with
  motion on frame 0 reads abrupt because the prior transition is still
  resolving in the viewer's eye.
- **Vary eases.** Use at least 3 different eases per scene — `power3.out`,
  `expo.out`, `back.out(1.4)`. Repeating one ease across every element makes
  every reveal feel like the same reveal.
- **Don't repeat an entrance pattern within a scene.** If the headline slid up,
  the subtitle slides from a different direction or scales rather than slides.
- **Avoid full-screen linear gradients on dark backgrounds.** H.264 bands them
  visibly. Use radial gradients or solid + localized glow.
- **Type sizing for rendered video.** Headlines 60px+, body 20px+, data labels
  16px+ at 1920×1080. Smaller reads fine on a monitor and illegible in a
  thumbnail.
- **`font-variant-numeric: tabular-nums`** on number columns and any animating
  count-up so digits don't jitter horizontally.

### Pacing

Different surfaces want different pacing. Pick by what the graphic is *for*:

| Surface | Beat length | Notes |
|---|---|---|
| **Hero / promo scene** (full-bleed, dominant on screen) | ~1.2–1.8s per beat | The viewer is watching the graphic. Move fast. |
| **Talking-head companion** (lower-third, side panel, stat callout) | 3–8s per beat | The viewer is split between voice and graphic. Hold longer. |
| **Outro / CTA / logo hold** | 4–6s held with micro-motion | Earned stillness; never frozen. |
| **Title card / chapter divider** | 2.5–4s | Reveal + breathe + transition. |

The `voice` field in the brand kit also nudges pacing — `bold` and `playful`
sit at the fast end of each range, `minimal` and `corporate` at the slow end.

### Micro-motion on holds

Every "held" beat needs at least one micro-motion layer so the frame doesn't
freeze. Drift, shimmer, breathing scale, a slow background gradient pan. The
viewer's eye reads a frozen pixel as a render bug; a 0.5px drift over 3s reads
as life.

```js
// A 3s held card with one breathing layer.
tl.from(".card", { y: 30, opacity: 0, duration: 0.6, ease: "power3.out" }, 0.3);
tl.to(".card-glow", { scale: 1.03, duration: 2.4, ease: "sine.inOut", yoyo: true, repeat: 1 }, 0.6);
tl.to({}, { duration: 3 }, 0);  // anchor
```

---

## Typography

- **Built-in fonts:** write the `font-family` you want in CSS — the compiler
  embeds supported fonts automatically (Inter, Geist, JetBrains Mono, and the
  common Google Fonts faces).
- **Custom fonts:** if the brand specifies a font that isn't built-in, the user
  must place `.woff2` files in a `fonts/` directory in the project. Warn before
  authoring if missing.
- **OpenType features.** Enable `font-feature-settings: "ss01", "cv02", "tnum"`
  (or whatever the chosen font supports) for an extra production-value bump on
  long-form copy. Restrained but effective.
- **External media:** add `crossorigin="anonymous"`.

---

## Assets

When a scene needs a real image — the brand logo, a product screenshot, a
photo — stage it with `--assets`:

```bash
pandastudio motion.render-html --htmlPath=/tmp/scene.html \
  --durationMs=5000 --assets="logo=/Users/me/brand/logo.svg,shot=/Users/me/shots/app.png"
```

Then reference by the alias in HTML:

```html
<img src="logo" alt="" class="hero-logo">
<img src="shot" alt="" class="app-shot">
```

This is strictly better than base64-inlining images (smaller HTML, cleaner
caching, the renderer can pre-decode).

---

## Quality

Before claiming done, run the verifiable checks below — not vibes-based "looks
good in my head."

### The human is the verifier

PandaStudio's user opens every render in the editor and reviews it
themselves. The agent does **not** auto-screenshot, auto-verify, or
otherwise gate its own work on visual inspection — that's spending an
expensive turn (and a couple of MB of vision-model tokens) on a check
the human does in two seconds for free.

Author the composition with care, walk the **textual** brand checklist
before rendering, then render and hand off. The user catches anything
visual you'd miss anyway.

The brand checklist (run mentally before `motion.render-html`):

- Every hex code in the HTML appears in `brand.colors` (primary /
  accent / ink / background) or was explicitly derived from one.
- Display + body fonts match `brand.typography`. No substitutions.
- Motion energy matches `brand.voice`. Bold → snappy + decisive;
  minimal → restrained + breathing; corporate → calm + smooth, etc.
- Logo on screen comes from `brand.logoPath` via `--assets`, not a
  guessed file.
- Tagline (if used in an outro) is the verbatim `brand.tagline`.

### Tools that exist for explicit user requests

`motion.screenshot` and `motion.verify-frames` are kept available for
when the user asks for a preview frame ("show me what scene 2 looks
like at 3 seconds") or a multi-frame sample ("give me 8 frames across
the timeline"). Both inline a vision-ready preview PNG in the tool
result, so when you call them on the user's behalf the response carries
the image inline — you can describe what you see back to the user.

You just don't reach for them in your own authoring flow.

### 3. Run the timeline-duration diagnostic

If a scene ever flashes black mid-render, `data-duration` and `timeline.duration()`
diverged. Re-check the law-#1 anchor `tl.to({}, { duration: SLOT }, 0)`.

### 4. Walk the brand checklist

- Every hex code in the composition appears in the brand kit (or was explicitly
  derived from one).
- Display + body fonts match the brand's `typography`. No substitutions.
- The motion energy matches the brand's `voice`.
- If the logo is in scene, it's the file from `brand.logoPath` (not a guess).
- If the brand has a tagline, the outro uses it verbatim.

Mismatch on any of these = re-author. The brand is the contract.

---

## Anti-patterns

The shortest list of things that produce generic / "meh" output:

- **Defaulting to a dark canvas because "dark looks premium."** If the brand
  isn't dark, don't go dark. (This was the v1.31 failure.)
- **Adding chrome gradients / halos / vignettes / grain because the previous
  graphic had them.** Those are house-style defaults, not universal moves.
- **Fading flat text in on a flat background.** That's the floor; clear it.
- **Using "Inter Bold 96px" as the only type move on every scene.** Type is a
  character — vary scale, weight, emphasis, treatment.
- **One-second hold timing on a companion graphic over a talking host.** They
  haven't finished reading the first word.
- **`repeat: -1` anywhere.** Capture-engine kill.
- **`Math.random()` for "natural-looking" variation.** Each render samples a
  different value → time-wobble. Seed it.
- **Skipping the static-CSS layout step.** Animation-first hides layout bugs
  until render time.
- **Using PandaStudio's own green/cyan editor UI color in a creator's brand
  graphic.** That's *our* product color, not theirs.

---

## References (load on demand)

These live alongside this file. Load only the ones the current job needs.

- **[house-style.md](./house-style.md)** — neutral defaults when the brand kit
  is partial or absent. Voice-to-motion defaults table. Three named "tracks"
  (creator-bright, dark-premium, product-clean) presented as starting points,
  NOT mandates.
- **[examples.md](./examples.md)** — concrete recipes (faux-cursor click,
  parallax-zoom, grid-pixelate-wipe, three.js scene). Adapt patterns; don't
  copy-paste.
- **[easing.md](./easing.md)** — easing dictionary by purpose (entrances,
  decisive moves, settles, micro-motion).
- Hyperframes upstream — link table at the top of this file.

---

## TL;DR

Read the brand. Pick the look from the brand, not from a default. Build the
hero frame in static CSS, then animate INTO it. Anchor every timeline at the
slot duration. Render a screenshot and check it. Re-author until it matches
the brand contract.

If the brand kit is empty, **ask** — don't invent.
