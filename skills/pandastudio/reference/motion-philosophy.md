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
   which are companion graphics (overlay a talking host)?
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

Multi-scene pieces have ONE rule, no exceptions: **the transition is the exit.**

1. **Always use entrance animations on every scene.** Every element animates IN
   via `gsap.from()`. No element appears fully formed. A scene with 5 elements
   needs 5 entrance tweens.
2. **Never use exit animations** on a scene before the final one. No
   `gsap.to(..., { opacity: 0 })`, no off-screen y, no scale-to-zero. The
   outgoing scene's content must be fully visible at the moment the transition
   starts.
3. **Always use a transition between scenes.** No jump cuts. Crossfade, slide,
   wipe, shader — any of those. The transition itself owns the exit.
4. **Only the final scene** may fade elements out (e.g., fade to black). This
   is the only place `gsap.to(..., { opacity: 0 })` is allowed.

```js
// RIGHT — entrance only, transition handles the exit.
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
