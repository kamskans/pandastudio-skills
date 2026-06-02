# Motion Philosophy — the bar we author to

**Load this file BEFORE writing any motion graphic.** If you're about to call
`motion.generate` with an HTML template, `motion.render-html` with custom
HTML, or build a lower-third / intro / transition, the rules in this doc
decide whether the result looks premium or looks like default AI output.

> **Authoritative engine docs.** PandaStudio's motion-graphic pipeline is
> [Hyperframes](https://hyperframes.heygen.com/) (HeyGen's open-source HTML→video
> renderer). The composition / determinism / timeline rules in this file
> mirror their published guides — when in doubt, those win:
>
> - **[Examples gallery](https://hyperframes.heygen.com/examples)** —
>   the **aesthetic minimum** every motion graphic you ship must clear.
>   Walk this page mentally before authoring. Kinetic-type, chrome-
>   gradient sweep, perspective grid, light-streak whip, object morph,
>   energy pulse, slide-up reveal. Whatever you build sits at or above
>   that level, never below.
> - [Prompting guide](https://hyperframes.heygen.com/guides/prompting) — the
>   authoritative LLM-targeted authoring playbook
> - [Common mistakes](https://hyperframes.heygen.com/guides/common-mistakes)
> - [Hyperframes core SKILL.md](https://github.com/heygen-com/hyperframes/blob/main/skills/hyperframes/SKILL.md)
> - [Animation-library adapters (gsap, three, anime, lottie, waapi, css)](https://github.com/heygen-com/hyperframes/tree/main/skills)
>
> If a motion-graphic render comes out static or off-by-frames, 9 times out of
> 10 you violated one of the rules in §0 (the laws) or §6 (anti-patterns).
> Re-read those before debugging deeper.

The bar we're aiming for is reference-grade work in **whatever aesthetic the
content calls for** — the bright, bold YouTube-creator look for a tutorial; the
HeyGen HyperFrames launch aesthetic (black canvas, chrome type, motion-blurred
whips) for a branded promo; the restrained Linear/Vercel keynote look for a
product announcement. These are three different *looks* (the **tracks** in §0.5),
not three quality bars. The quality bar is constant: kinetic, intentional,
frame-perfect.

If you don't know what "good" looks like, the short answer is: **static text
fading in on a flat background is the lowest tier of motion graphics — in any
track.** If that's what you're about to author, stop and read this file.

---

## 0 · The Laws (memorize)

> **Read this honestly.** This list was originally written ("the 11 Laws") when
> PandaStudio shipped only the dark-premium launch look, so several "laws" were
> really that look's *styling* dressed up as universal truth: *black is the
> canvas*, *light not color*, *perspective grid on every scene*. They are NOT
> universal — apply them to a bright YouTube-creator graphic and you get exactly
> the failure we keep hitting (a moody black chrome-gradient card where the
> creator wanted bold and bright). So each law below now states the **universal
> principle first**, then — where the original was track-specific — names how
> each track (A · YouTube-creator, B · dark-premium, C · product-launch; see
> §0.5) expresses it. The numbering is preserved because other docs reference
> "Law #4" and "Law #11" by number.

1. **One idea per beat.** Each visual lands ONE word or concept, then moves on.
   If a scene is saying two things, split it. *Pacing differs by context:* a
   hero/promo beat averages ≈1.5s (Track B); a companion graphic over a talking
   host holds far longer (§1.6) because the viewer is split between voice and
   graphic. Fast-cut energy is Track B's default, not a universal speed.
2. **Commit to a canvas and protect its negative space.** Pick the frame's
   ground and defend it — don't crowd it. *Track-specific:* **A** = a full-bleed
   saturated brand color (or bright off-white) edge to edge; **B** = black /
   near-black, ~90% negative space; **C** = a calm tinted solid or soft off-white
   with generous breathing room. (The old "black is the canvas" is Track B only.)
3. **Production value is the brand — the right kind per track.** The frame should
   read as *made on purpose*, not defaulted. *Track-specific:* **A** = bold,
   confident, high-contrast, instantly legible; **B** = *lit* not colored — chrome
   gradients, halos, light beams; **C** = precise, spacious, understated. (The old
   "light is the brand, not color" is Track B's finish, not a universal ban on
   color — Track A is deliberately color-forward.)
4. **Camera never sleeps.** Even on "still" frames something moves — a drift, a
   shimmer, a breathing vignette, a slow scale. Static = death. Every hold beat
   needs at least one micro-motion layer. *(Universal.)*
5. **No naked cuts — every transition carries motion.** Hard cuts and plain
   crossfades feel cheap. *The technique is track-specific:* **B** = motion-blur
   whip-streaks; **A** = snappy pops, slides, grid/pixelate wipes (§3.8); **C** =
   clean directional slides and masks. Motion blur is ONE option, not the law.
6. **For multi-scene pieces, callbacks beat novelty.** A prop/icon/shape that
   appears early returns later — visual continuity reads as intentional. *N/A for
   a single lower third or one-shot card;* it's a law for pieces with ≥3 scenes.
7. **Intentional, limited, brand-derived palette.** ≤5 active hues, each owning a
   concept; usually fewer (Track A often runs one dominant brand hue + a near-black
   ink). Never add a color because it "looks nice," and **never default to the
   PandaStudio editor's green/cyan** — derive the palette from the *creator* (§0.5).
   *(Universal.)*
8. **Type is a character.** Words SCALE, MOVE, COMPRESS, GLOW — typography drives
   most of the storytelling and a text-only beat can be the strongest beat. Never
   fade flat text in and call it done. *(Universal; the treatment — chrome+halo vs
   solid heavy fill vs clean restrained — is per track.)*
9. **Hold the hero shot.** A reveal or outro earns stillness (with micro-motion):
   logo/hero ≈1.5–2s, outro/CTA ≈4–6s. Speed earns space for stillness to land.
   *(Universal.)*
10. **One unifying system across scenes.** A multi-scene piece needs a spine that
    ties scenes together. *The spine is track-specific:* **B** = perspective grid +
    crosshair markers + vignette + grain on every scene; **A** = a consistent type
    system + color-block language + a repeated motion signature; **C** = a fixed
    spatial grid + consistent easing + restraint. (The old "perspective grid on
    everything" is Track B's spine — do NOT put it under a bright Track A graphic.)
11. **Timelines must fill their slots.** HyperFrames hides a sub-composition the
    moment `timeline.duration() < data-duration` → black-frame flash. Every GSAP
    timeline ends with `tl.to({}, { duration: SLOT_DURATION }, 0)` as a no-op
    duration anchor. Non-negotiable, every track. Recipe + diagnostic in §4.
12. **Determinism is law.** No `Math.random` / `Date.now` / `performance.now`, no
    `repeat: -1`, no `setInterval`/`setTimeout`/async in the render body. Renders
    must be bit-identical frame-to-frame because HyperFrames seeks deterministically.
    Seed any randomness; drive everything off the paused timeline. *(Universal — was
    buried in §6 anti-patterns; it's a law.)*

---

## 0.5 · Aesthetic tracks — choose the look BEFORE you author

**Read this before §1.** §0's laws fold in *where* they're track-specific (canvas,
finish, transition technique, unifying spine). This section is where you actually
**pick the track** — do it first, before you write a line of HTML, because it
decides canvas color, type treatment, and motion language. Picking wrong is the
single most common motion-graphic failure: a YouTube tutorial gets a moody black
chrome-gradient title card when it wanted bold, bright, and full-bleed.

The universal laws (#1, #4, #6–#9, #11, #12) hold no matter which track you pick.
The track decides the rest:

**Choose ONE track per piece and commit to it:**

| Track | Canvas | Type | Color | When |
|---|---|---|---|---|
| **A · YouTube-creator** | FULL-BLEED saturated brand color, or bright off-white. Edge-to-edge, no letterboxed card floating in black. | Huge, heavy (800–900 weight), often lowercase, tight tracking. Solid fills, not chrome. | One dominant brand hue carrying the whole frame + a near-black ink for text. Bold and legible at a glance. | The default for tutorials, vlogs, talking-head explainers, "how I built X", reaction/commentary. **Most PandaStudio edits.** |
| **B · Dark-premium** | Black / near-black, ~90% negative space. The §0 laws apply verbatim. | Chrome-gradient, haloed, scales to 8×. | ≤5 hues, light-as-brand, perspective grid + crosshairs + vignette + grain. | Branded promos, launch films, sizzle reels, anything cinematic with no host talking. |
| **C · Product-launch / keynote** | Deep-tinted solid or soft off-white. Calm, lots of breathing room. Linear/Vercel/Apple-keynote energy. | Clean sans, medium-heavy, restrained scale. Subtle gradient emphasis on ONE word max. | 2–3 hues, low saturation, generous neutrals. Precision over energy. | Feature announcements, changelog reels, B2B/SaaS, "introducing X". |

### Profile → default track

| Destination profile | Default track | Notes |
|---|---|---|
| `youtube-long`, `shorts`, `tiktok`, `reels` | **A · YouTube-creator** | Unless the creator's own brand is explicitly dark/cinematic. |
| `linkedin` | **C · Product-launch** | Professional, restrained. A also fine for personal-brand creators. |
| `loom` / internal | minimal — captions + zooms; graphics only if asked | Don't over-produce a screen-share walkthrough. |
| Branded promo / sizzle / no host on screen | **B · Dark-premium** | The §0 laws are the bar here. |

The profile picks the *default*; the **creator's brand always overrides it**. If
you can see the channel's palette (thumbnail, logo, set, lighting, prior graphics),
match THAT. A creator whose brand is hot-pink-on-cream gets hot-pink-on-cream
graphics regardless of profile.

### Brand-first color — and the editor-green trap

**Derive the palette from the CREATOR, never from the PandaStudio editor UI.**
PandaStudio's interface brand color is a green (`#34B27B`) and a cyan
(`#0BC0F0`). Those belong to the *tool*, not to the user's *content*. Defaulting
a user's graphic to editor-green is the equivalent of Final Cut stamping its
icon color into every export — it's an anti-pattern (see §6). Sources for the
real palette, in priority order:
1. An explicit brand color the user gave you.
2. The channel's thumbnails / logo / recurring graphics.
3. The set + lighting in the recording itself.
4. A neutral, track-appropriate default (Track A: a confident saturated hue you
   pick *with a reason* — not green-because-it-was-on-screen; Track B: black +
   chrome; Track C: off-white + one restrained accent).

When you don't know the brand, say so and pick a defensible neutral — don't
reach for the editor's chrome.

### Scene graphic vs companion graphic (opaque vs transparent)

Two structurally different things, often confused:

- **Scene graphic (opaque, full-frame):** the host steps away and the graphic
  OWNS the frame — a title card, a chapter divider, a stat reveal, an outro. Render
  OPAQUE (mp4, or a webm with a solid background). It fills 1920×1080 edge to
  edge. **It is not a small ribbon floating over the camera.** Use it when the
  voiceover pauses or the script hits a section break. Tracks A/B/C all do scene
  graphics.
- **Companion graphic (transparent, alongside host):** the host keeps talking and
  the graphic shares the frame — a lower third, a side panel of bullet points, a
  concept callout, a designed split segment. Render TRANSPARENT (`motion.render-html
  --transparent` → VP9-alpha webm) so the camera shows through. Pace it slower
  (§1.6) because the viewer is split between voice and graphic.

A layered scene graphic (Track A or B) is built like the canonical shell: a
full-frame background plate, then content on top, then ambient texture. A
companion graphic has a transparent background and only paints its panel.

**You do NOT need transparency for a scene graphic.** When the host is silent and
the graphic owns the frame, an opaque full-frame plate is correct and simpler —
start the graphic as the voiceover pauses. Reserve transparent renders for
companion graphics that genuinely share the frame with live footage. (If the user
explicitly wants a transparent overlay, honor that.)

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

- **Single geometric sans-serif throughout a piece.** Generous tracking.
  Pick ONE and commit. Per-beat font switching is advanced — don't use it on v1.
  - **CRITICAL — only use fonts the renderer can actually load.** HyperFrames
    renders headless and resolves type by fetching the declared family from
    Google Fonts, then caching it deterministically. A family that is NOT on
    Google Fonts — `SF Pro`, `Suisse Int'l`, `-apple-system`, `system-ui`, or a
    bare `sans-serif` — fails that fetch and SILENTLY falls back to whatever the
    host OS happens to have. That is the single most common reason a graphic
    "looks cheap": the premium typeface you specified never actually rendered.
    Never use those as the primary family.
  - **Guaranteed set (bundled with the engine, offline-safe):** `Inter`,
    `Montserrat`, `Outfit`, `Oswald`, `Archivo Black`, `League Gothic`,
    `Nunito`. Any other Google Fonts family also works (fetched on first use) —
    e.g. `Space Grotesk`, `Sora`, `Manrope`, `Inter Tight`, `Archivo`. When in
    doubt, use `Inter`.
  - **Declare it explicitly.** Put a Google Fonts `<link>` for the exact family
    and weights in `<head>`, then set `font-family: "<Family>", sans-serif`. The
    trailing generic is a last-ditch fallback, never part of the design.
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

≤5 hues per piece, each with a meaning, **derived from the creator's brand — not
the PandaStudio editor's green/cyan** (Law #7, §0.5). The palette shape depends on
the track.

**Track A (YouTube-creator)** — usually just TWO that do the heavy lifting:

| Role | Example | When |
|---|---|---|
| Brand fill | one confident saturated hue (the creator's) | The full-bleed canvas + accents |
| Ink | near-black (`#06141a`) or near-white depending on fill | All heavy headline type |
| (optional) live/alert | `#ff2d2d` | A pulsing "live"/"new" dot, sparingly |

**Track B (dark-premium)** — the lit, symbolic palette:

| Role | Example hex | When |
|---|---|---|
| Canvas | `#000` / `#0a0a0a` | Every frame |
| Brand primary (chrome) | `#fff → #999` gradient | All headline type |
| Accent 1 (problem / alert) | `#e10b1f` red | Thing we're solving |
| Accent 2 (solution / brand core) | `#33d4c8` teal | The answer / product |
| Accent 3 (energy / speed) | `#a155ff` magenta | High-velocity beats |

**Track C (product-launch)** — 2–3 low-saturation hues over generous neutrals
(off-white/deep-tint canvas, one restrained accent on a single emphasized word).

Rule: when adding a new scene, ask "what's the one color carrying this beat, and
does it come from the creator's brand?" If you can't name it, or it's
green-because-it-was-on-screen, you haven't earned it. Kill it.

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
| **Floating cluster drift** | 3+ objects gently bob throughout the scene — keeps "still" frames alive. Use a finite repeat count derived from composition duration; `repeat: -1` is forbidden because Hyperframes seeks deterministically and infinite repeats run on GSAP's internal time. | `gsap.to(coins, { y:'-=15', duration:2, repeat: Math.ceil(SLOT_DURATION/4) - 1, yoyo:true, ease:'sine.inOut', stagger:{each:0.4} })` (seeded stagger — drop `from:'random'`; use a deterministic order or seeded PRNG instead) |
| **Vignette breath** | Vignette opacity wobbles continuously across the scene. Same finite-repeat rule. | `tl.to(vignette, { opacity:0.85, duration:1.6, repeat: Math.ceil(SLOT_DURATION/3.2) - 1, yoyo:true, ease:'sine.inOut' }, 0)` (attach to `tl`, not floating `gsap.to` — keeps it inside the seek-driven timeline) |
| **Cut-the-curve vertical whip** | Default adjacent-beat transition: exit rides up w/ blur, entry rises from below w/ matching blur, same direction, velocity matched | **Exit:** `tl.to(wrap, { y:-150, filter:"blur(30px)", duration:0.33, ease:"power2.in" })` **Entry:** `gsap.set(wrap, { y:150, filter:"blur(30px)" })` then `tl.to(wrap, { y:0, filter:"blur(0px)", duration:1.0, ease:"power2.out" }, 0)` |
| **Faux-cursor click event** | 7-tween sequence selling a UI interaction (click, ripple, overshoot, settle) | See §3 of this doc for full recipe |

### 1.5 Pacing Discipline (numeric)

**The numbers below are for HERO / standalone motion-graphic pieces
(30s branded promos, launch reels) where the entire screen is the
motion graphic and there's no host speaking. For MOTION-GRAPHIC
OVERLAYS on top of a YouTube host (Mode C from video-authoring.md),
see §1.6 below — those timings are slower because the viewer is
split-attention between the host's voice and the graphic.**

- **Default scene length:** 1.0–2.0s. Longer only for hero moments + outro.
- **Reveal cadence inside a scene:** new visual element every 0.3–0.6s.
  No dead air > 1s mid-piece.
- **Word-reveal stagger:** 0.3–0.4s per word for narrative reads.
  0.5–0.6s for dramatic single-word emphasis.
- **Whip transition duration:** 0.3–0.4s. Faster = glitchy. Slower = loses energy.
- **Hold durations:**
  - Logo crystallization: 1.5–2s
  - Final CTA card: 4–6s (the longest single shot in the piece is the outro)

### 1.6 Pacing for OVERLAY motion graphics (YouTube side-panel, lower thirds, concept callouts)

When the motion graphic overlays existing host footage (Mode C from
`video-authoring.md`), **the viewer can't absorb it as fast as a
standalone piece** — they're also listening to the speaker. Slower
pacing is correct, not a bug.

| Overlay type | Total duration on timeline | Animation completes at | Hold ratio |
|---|---|---|---|
| **Concept callout** (3–5 word right-rail card) | **5–7 s** | 1.2 s | animation 20% / hold 80% |
| **Lower third** (name + title) | **5–8 s** | 1.0 s | animation 15% / hold 85% |
| **Stat reveal** (number + label) | **4–5 s** | 1.5 s (counter tween 0→final) | animation 35% / hold 65% |
| **Intro title card** (full-frame takeover) | **3–5 s** | 1.8 s | animation 45% / hold 55% |
| **Chapter marker** (small top-left badge) | **2–3 s** | 0.4 s | animation 20% / hold 80% |
| **Outro CTA card** (full-frame takeover) | **5–7 s** | 1.5 s + shimmer loop | animation 25% / hold 75% (with continuous shimmer) |

**The hold-ratio rule is the critical one.** An overlay where the
animation runs the full duration leaves the viewer nothing to read —
the text is moving the whole time and they can't parse it. Structure
your GSAP timeline so everything lands within the first 20–45% of the
scene, then the static frame HOLDS for the remaining 55–80%.

**Canonical timing pattern for a concept callout (total 6s):**

```js
// Scene total: 6s. Animation lands at 1.2s. Hold 4.8s. Exit at 5.7s.
const SCENE = 6;
const IN_END = 1.2;   // reveal complete by 1.2s
const OUT_START = 5.7; // exit begins at 5.7s
const OUT_END = 6.0;

const tl = gsap.timeline({ paused: true });

// Entry: 0 → 1.2s (card slides in from right + chrome-sweep)
tl.from('.callout-card', { x: '100%', duration: 0.8, ease: 'expo.out' }, 0);
tl.from('.callout-text', { y: 20, opacity: 0, duration: 0.5, ease: 'power2.out' }, 0.6);
// Chrome sweep on the headline word:
tl.to('.headline', { backgroundPosition: '0% 0', duration: 0.6, ease: 'power2.out' }, 1.0);

// HOLD 1.2s → 5.7s (4.5s of stillness). Viewer reads the card.
// Add one micro-motion loop so it's not frozen — vignette breath or a
// slow drift on the background grid.
gsap.to('.callout-bg', { backgroundPositionY: '+=40', duration: 4, ease: 'none' });

// Exit: 5.7s → 6.0s (card slides out to right with blur)
tl.to('.callout-card', { x: '100%', filter: 'blur(10px)', duration: 0.3, ease: 'power2.in' }, 5.7);

tl.to({}, { duration: SCENE }, 0);  // Law #11 anchor
window.__timelines['concept-callout'] = tl;
```

**Reading-time formula** for copy-heavy overlays (pull quotes, benefit
callouts):

```
minimum_duration_seconds = max(5, words × 0.35 + 2)
```

A 3-word callout ("uncovers hidden trends") needs at least 5s. A 6-word
callout ("this is the best part") needs at least 4.1s → rounds to 5s.
A 10-word pull quote needs 5.5s → use 6s. **Never ship an overlay
under 4 seconds total** — the viewer can't catch it AND listen to the
host at the same time.

**On the timeline:** `project.add-motion-graphic --durationMs=<total>`
where `<total>` matches the scene duration you authored. Don't author
a 6000ms scene and then place it on the timeline for 3000ms —
that truncates half the animation + all the hold.
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
| Breathe / drift | `sine.inOut` yoyo | 2–4s cycle, `repeat: ⌈SLOT_DURATION/cycle⌉ - 1` (NEVER `-1` — see anti-patterns) |

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

### 3.6 Deterministic randomness (seeded PRNG)

Hyperframes' frame-perfect capture means `Math.random()` produces a
DIFFERENT random sequence on every render — particle bursts, scattered
stars, jittered offsets all change between exports. The fix is a
seeded mulberry32 PRNG: same seed → same sequence → same frames.

```js
// Drop this at the top of your <script> body. ~10 LOC, no deps.
function mulberry32(seed) {
  let a = seed >>> 0;
  return () => {
    a = (a + 0x6D2B79F5) >>> 0;
    let t = a;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const rand = mulberry32(20251101);  // any constant — change to reroll

// Use anywhere you'd have used Math.random():
const stars = Array.from({ length: 80 }, (_, i) => ({
  x: rand() * 100,        // % of canvas
  y: rand() * 100,
  size: 1 + rand() * 2.5,
}));
```

For one-off harmonic offsets (no PRNG state needed), the
sin-cosine-hash trick works too:

```js
// 80 → 300 range, varies per index, deterministic, no state.
const offset = i =>
  80 + 220 * Math.abs(Math.sin(i * 0.7 + 0.3) * Math.cos(i * 1.3 + 0.7));
```

**Never** use `Math.random()`, `Date.now()`, `performance.now()`,
`crypto.getRandomValues()`, or any GSAP `from: 'random'` stagger. All
of those produce non-deterministic frames.

### 3.7 three.js — when (and how) to go 3D

The Hyperframes examples gallery (kinetic-type, chrome logo reveals,
particle-burst stat reveals) is overwhelmingly built on top of three.js.
Flat 2D motion graphics top out at "polished" — three.js is what gets
you to "highly professional." Default to 3D unless one of the explicit
exclusions below applies.

**Use three.js when:**

- The brief is a logo reveal, hero title, or product reveal (3D extrusion + chrome material >> any 2D approximation)
- Kinetic-type — words flying through 3D space, camera dollying through type, depth of field on receding lines
- Stat reveals — particle clouds assembling into the number, glow halo, perspective falloff
- "Premium" / "professional" / "cinematic" / "Apple-keynote-style" briefs
- Any environment scene — perspective grid floor, sky parallax, infinite hallway, network-of-nodes
- Object morph between two hero objects (vertex morph targets + DOF beat 2D dissolves)

**Stay 2D when:**

- The motion graphic is a thin overlay on a host (lower-third chyron, name plate, bug logo)
- Total duration < 2s — 3D needs hold time to read
- Vertical 9:16 with the 3D scene wouldn't compose (most product reveals work in portrait, but talking-head overlays do not)
- Battery / perf is critical (rare; the renderer is offline so this almost never applies)

**The page contract for three.js (deterministic seek)**

Hyperframes' frame-perfect capture extends to three.js via a global
clock variable. You do NOT use `THREE.Clock` (it's wall-clock and
non-deterministic). Instead:

```js
// 1. Load three.js from CDN inside the composition <head>:
//    <script src="https://unpkg.com/three@0.160.0/build/three.min.js"></script>

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, w / h, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(w, h);
document.body.appendChild(renderer.domElement);

// 2. Drive every animatable property from a single time variable.
//    Hyperframes writes the current frame's time into __hfThreeTime
//    (seconds, float). Read it inside your render loop:
function tick() {
  const t = window.__hfThreeTime ?? 0;  // Hyperframes-driven
  cube.rotation.y = t * 0.5;
  particles.position.z = -t * 2;
  renderer.render(scene, camera);
  requestAnimationFrame(tick);
}
tick();

// 3. Register the timeline by composition-id like any GSAP scene.
//    Mix freely: GSAP can drive DOM overlays, three.js drives 3D.
window.__timelines = window.__timelines || {};
window.__timelines["my-3d-scene"] = tl;
```

The `__hfThreeTime` variable is updated by Hyperframes between frame
captures — same seek mechanism that drives GSAP's `tl.seek(t)`. Using
it means:

- 30fps render → frame `n` reads `__hfThreeTime = n / 30`
- Same input scene + same time = same pixels (deterministic)
- No `Math.random()` in render loops — use the seeded mulberry32 PRNG
  from §3.6 instead

**Three.js patterns that map to Hyperframes examples**

| Hyperframes example | three.js implementation |
|---|---|
| Kinetic-type (words flying through space) | `TextGeometry` with extruded depth, perspective camera dolly via `camera.position.z = lerp(start, end, t)` |
| Chrome logo reveal | Extruded geometry + `MeshBasicMaterial` with chrome gradient baked into a CanvasTexture (avoid `MeshPhysicalMaterial` clearcoat — it has WebGL-context issues with `preserveDrawingBuffer`) |
| Particle stat reveal | `Points` geometry with custom vertex shader, particles assemble from random positions to target positions over `t` |
| Energy pulse along path | `THREE.CatmullRomCurve3` + glowing sprite that interpolates `curve.getPointAt(t)` |
| Perspective grid floor | `GridHelper` with `material.opacity = 0.3`, camera looks at horizon, fog for distance falloff |
| Object morph A → B | Two meshes with `morphTargetInfluences`, GSAP tweens influence from 0 to 1 |

**Material choice — IMPORTANT.** Hyperframes runs in headless Chrome
with `preserveDrawingBuffer: true`. `MeshPhysicalMaterial` with
clearcoat or transmission has been observed to throw `glCopySub-
TextureCHROMIUM` errors in this configuration. Stick to
`MeshBasicMaterial` (with chrome gradient as a CanvasTexture for the
"physical" look) or `MeshStandardMaterial` for lit objects. Avoid
clearcoat/transmission until verified working in headless.

**Performance budget**

- Particles: ≤ 5000 (more starts dropping frames during capture)
- Geometry: keep under ~50k triangles total
- Post-processing: only `EffectComposer` with `BloomPass` for hero
  shots; everything else is too slow for offline render to justify

### 3.8 Track-A scene graphic — full-bleed creator title/chapter card

The YouTube-creator workhorse: the host pauses, an opaque full-frame card owns
the screen for 2–3s, then the host returns. **Opaque, edge-to-edge, brand color,
heavy type — no grid, no chrome, no black.** Render opaque (mp4) since it owns
the frame. Pick the brand hue from the creator (§0.5), not the editor.

```html
<div id="root" data-composition-id="chapter" data-width="1920" data-height="1080" data-duration="3.0"
     style="position:relative;width:1920px;height:1080px;background:#0BC0F0;overflow:hidden;
            font-family:Inter,sans-serif">
  <!-- BRAND fills the frame. Swap #0BC0F0 for the creator's hue. -->
  <div class="scene" style="position:absolute;inset:0;display:flex;flex-direction:column;
       align-items:center;justify-content:center;gap:18px">
    <div class="eyebrow" style="display:inline-flex;align-items:center;gap:18px;font-size:30px;
         font-weight:700;letter-spacing:0.35em;text-transform:uppercase;color:#062a33">
      <span class="livedot" style="width:20px;height:20px;border-radius:50%;background:#ff2d2d;
            box-shadow:0 0 22px rgba(255,45,45,.6)"></span><span>Part 2 · Live</span></div>
    <!-- HEAVY, near-black ink on the brand color. Lowercase reads modern. -->
    <div class="headline" style="font-size:200px;font-weight:900;color:#06141a;
         letter-spacing:-0.04em;line-height:0.9;text-transform:lowercase">live demo</div>
  </div>
  <div id="grid" style="position:absolute;inset:0;display:grid;
       grid-template-columns:repeat(16,1fr);grid-template-rows:repeat(9,1fr)"></div>
</div>
<script>
  const grid=document.getElementById("grid");
  for(let i=0;i<16*9;i++){const c=document.createElement("div");c.className="cell";
    c.style.cssText="background:#06141a;transform:scale(0);transform-origin:center";grid.appendChild(c);}
  const tl=gsap.timeline({paused:true});
  // grid-pixelate-wipe IN (cover from center) then OUT (reveal to edges)
  tl.to("#grid .cell",{scale:1,duration:0.5,stagger:{amount:0.4,from:"center"},ease:"power2.inOut"},0);
  tl.to("#grid .cell",{scale:0,duration:0.5,stagger:{amount:0.4,from:"edges"},ease:"power2.inOut"},0.55);
  tl.from(".headline",{y:30,opacity:0,duration:0.5,ease:"power3.out"},0.7);
  tl.from(".eyebrow",{opacity:0,duration:0.4},0.85);
  tl.fromTo(".livedot",{scale:0.8},{scale:1.2,duration:0.5,ease:"sine.inOut",yoyo:true,repeat:3},1.0);
  tl.to({},{duration:3.0},0);                 // Law #11 anchor
  window.__timelines=window.__timelines||{};window.__timelines["chapter"]=tl;
</script>
```

### 3.9 Catalog plate moves (parallax-zoom / parallax-unzoom / grid-pixelate-wipe)

Three entrance/exit techniques lifted from the HyperFrames catalog. They work in
ANY track — they're *motion*, not *styling* — so layer text on top of whatever
canvas the track calls for. Each is a self-contained block plus its GSAP tween.

**grid-pixelate-wipe** — the frame dissolves into a grid of cells that scale in
from the center (cover) and out to the edges (reveal). The cleanest way to mask a
cut or stamp content onto the frame. Full recipe inline in §3.8 above.

**parallax-zoom (push-in reveal)** — a background plate scales up slowly while
foreground content rises and settles. Reads as a confident "lean in." Use for a
title landing or a stat reveal.

```js
// .plate = full-frame bg image/color; .fg = the headline/stat on top.
const tl = gsap.timeline({ paused: true });
tl.fromTo(".plate", { scale: 1.0 }, { scale: 1.12, duration: SLOT, ease: "power1.inOut" }, 0); // slow drift the WHOLE slot
tl.from(".fg",      { y: 60, opacity: 0, duration: 0.7, ease: "power3.out" }, 0.25);
tl.from(".fg .sub", { y: 24, opacity: 0, duration: 0.5, ease: "power2.out" }, 0.5);
tl.to({}, { duration: SLOT }, 0);            // Law #11
```

**parallax-unzoom (pull-back reveal)** — the inverse: the plate starts pushed in
and eases back out to 1.0 while content settles. Reads as "stepping back to show
the whole picture" — good for an outro or a "here's everything" summary card.

```js
const tl = gsap.timeline({ paused: true });
tl.fromTo(".plate", { scale: 1.14 }, { scale: 1.0, duration: SLOT, ease: "power1.out" }, 0);
tl.from(".fg",      { scale: 0.92, opacity: 0, duration: 0.7, ease: "power3.out" }, 0.2);
tl.to({}, { duration: SLOT }, 0);
```

Determinism note: a continuous plate drift over the whole slot needs NO repeat —
it's a single tween spanning `SLOT`, so it's already seek-safe. Don't loop it.

---

## 4 · Pre-flight checklist — run BEFORE claiming done

No motion graphic ships without every one of these passing. If you can't
tick a box, fix it before calling `export.start`.

**Universal (every track):**
- [ ] **Picked a track (§0.5) and the look matches it** — bright/full-bleed for A, black/chrome for B, calm/tinted for C. The single most important check.
- [ ] **Palette is brand-derived, ≤5 hues, NOT the editor's green/cyan** unless that's genuinely the creator's brand
- [ ] **No dead air** beyond deliberate holds; pacing fits the mode (fast for hero/promo, slower for companion-over-host §1.6)
- [ ] **Every transition carries motion** (whip / wipe / slide / recolor — no hard fades), in the track's technique
- [ ] **Type scales/moves on reveal** — never a flat fade-in
- [ ] **Scene graphics fill the frame; companion graphics are transparent** (§0.5) — a scene graphic is NOT a small ribbon floating in empty space
- [ ] **Outro/hero holds** with micro-motion (shimmer / breath / slow scale)
- [ ] **Every sub-composition timeline ends with the Law #11 anchor**
- [ ] **Determinism (Law #12):** no `Math.random`/`Date.now`, no `repeat:-1`
- [ ] **All tween end-times snap to multiples of `1/fps`** (at 30fps: 0.0333, 0.0667, 0.1… — steep-tail eases like `expo.in` alias at sub-frame boundaries)
- [ ] **Ran the timeline-duration diagnostic** (see below)
- [ ] **Extracted frames and actually looked at them** — see `motion.verify-frames`. Lint passing ≠ design working. No ship without frame inspection.

**Track B (dark-premium) — additionally:**
- [ ] **Average scene length ≤ 2s** in mid-section (intro/outro may hold longer)
- [ ] **Headlines use chrome gradient + halo** — no flat white
- [ ] **Perspective grid + crosshairs in ≥60% of scenes**, vignette + grain on every scene
- [ ] **≥1 callback** — a visual element that returns later (multi-scene pieces)

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

**The track / color failures (the ones we actually keep hitting):**

- ❌ **Defaulting a user's graphic to the PandaStudio editor's green (`#34B27B`)
  or cyan (`#0BC0F0`).** Those are the *tool's* brand, not the *creator's*. Seeing
  green on screen in the editor is not a reason to put green in the content — it's
  the equivalent of Final Cut stamping its logo color into every export. Derive the
  palette from the creator (§0.5, Law #7). When you don't know it, pick a defensible
  neutral *with a stated reason* — never green-by-accident.
- ❌ **Shipping the dark-premium look (black canvas, chrome type, perspective grid)
  on YouTube-creator content.** That's Track B applied where Track A belongs. A
  tutorial/vlog/explainer wants bright, bold, full-bleed brand color and heavy solid
  type — not a moody black card. Pick the track FIRST (§0.5).
- ❌ **Shrinking a scene graphic into a small ribbon floating in empty space.** When
  the host steps away and the graphic owns the frame, it must FILL the frame
  (1920×1080, edge to edge). A tiny centered pill on a vast empty background reads as
  unfinished. (A small ribbon is only correct as a *companion* lower-third over live
  footage — different thing, §0.5.)
- ❌ **Reaching for a transparent render when the graphic owns the frame.** Scene
  graphics are opaque and full-bleed; transparency is for companion graphics that
  share the frame with live footage. Don't add alpha you don't need.
- ❌ **One-aesthetic-fits-all.** Authoring every piece in the same look regardless of
  destination and creator brand. The track is a per-piece decision.

**Process trap — verifying a transparent render:**

- ❌ **Trusting `ffprobe`'s `pix_fmt` or an `ffmpeg overlay` composite to judge
  whether a transparent render "has alpha".** VP9-alpha webm stores alpha as a
  separate side-stream: `ffprobe` reports the main plane as `yuv420p` *even when
  alpha is present*, and ffmpeg's default decoder / `overlay` filter silently drop
  the alpha (so a transparent overlay composites as **black**). Neither is a bug.
  **A `yuv420p` reading is EXPECTED and not evidence of a problem.** To actually
  verify alpha: check the `ALPHA_MODE=1` stream tag (`ffprobe -show_entries
  stream_tags`), decode with an explicit `-c:v libvpx-vp9`, or just composite in
  Chromium (which is what PandaStudio's renderer uses and it decodes VP9 alpha
  correctly). This exact trap once produced a false "critical transparency
  regression" alarm — the alpha was fine the whole time.

**The craft failures:**

- ❌ **Centered, axis-aligned, motionless text fades.** This is the lowest
  motion-graphics tier. Always add scale, blur, or directional energy.
- ❌ **Hard cuts between scenes.** Every cut needs a transition element
  bridging it (streak, morph, wipe, shader).
- ❌ **Untreated default type.** Flat white on black with no treatment is the
  Track-B tell (use chrome gradient + halo there). Track A wants heavy solid fills
  with real weight contrast; Track C wants a clean restrained face with at most one
  gradient-emphasized word. The anti-pattern is *defaulting*, not the specific fill.
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
- ❌ **`Math.random()` / `Date.now()` / `performance.now()` / unseeded
  PRNGs.** Renders must be bit-deterministic frame-to-frame. Use a
  seeded mulberry32 PRNG, OR a harmonic hash:
  `80 + 220 * Math.abs(Math.sin(i*0.7 + 0.3) * Math.cos(i*1.3 + 0.7))`
- ❌ **`repeat: -1` on GSAP tweens.** Infinite repeats run on GSAP's
  internal time, not seek time, so frames render at non-deterministic
  phases. Use a finite count that fills the composition: e.g. for a 8s
  composition with a 2s breath cycle, `repeat: 4`. Pad with
  `tl.set({}, {}, COMPOSITION_DURATION)` so the timeline length
  remains exact (Law #11). Same for ALL "live" effects — vignette
  breath, particle drift, ambient bobs, chromatic shimmer.
- ❌ **`setInterval` / `setTimeout` / async-await inside the render
  body or while building timelines.** Synchronous-only. Hyperframes'
  seek is also synchronous — async work mid-build produces
  partially-initialised state when the renderer captures.
- ❌ **Animating `visibility` / `display` / `width` / `height` for
  reveals.** Animate `opacity` (or GSAP's `autoAlpha`), `transform`,
  and color. The omitted properties either don't tween, layout-thrash
  on every frame, or require a full reflow that breaks Hyperframes'
  frame-perfect capture.
- ❌ **`<br>` tags for line breaks in headings or body copy.** Use
  `max-width` and let CSS wrap naturally — `<br>` will land in the
  wrong place when text length changes between renders.
- ❌ **SVG-filter grain via `<filter>` data URI.** Breaks Safari /
  html2canvas (Hyperframes uses Chromium for video render but the same
  HTML often gets thumbnail-captured elsewhere). Use the CSS radial-
  gradient grain pattern from §1.1.
- ❌ **Exit animations on non-final scenes.** When a shader transition
  fires between scenes, the transition IS the exit — adding a fade-out
  before it produces a double-fade that looks broken.
- ❌ **Animating `<video>` dimensions directly.** Wrap the `<video>` in a
  `<div>` and animate the wrapper. Direct `<video>` animation freezes
  the decoder and the video pauses while audio continues.
- ❌ **Composition timeline shorter than embedded `<video>`.** The video
  freezes on its last decoded frame for the trailing duration. Pad the
  GSAP timeline to ≥ video duration via `tl.set({}, {}, videoDuration)`.
- ❌ **Using `window.__hf.seek` / `window.__hf.duration` directly.** That
  bypasses the compositor invalidation wrapper — animations render with
  1-second stalls. ALWAYS register through `window.__timelines[id]` and
  let the paused-timeline + HF tick loop drive seek.
- ❌ **Calling `video.play()` / `audio.play()` / setting `currentTime`
  manually.** Hyperframes owns media playback. Author your `<video>` as
  `muted playsinline` with `data-start` + `data-duration` attributes
  and let the framework drive it. Audio goes in separate `<audio>`
  elements with the same data-attributes.
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
2. **Pick the track (§0.5)** from the destination + the creator's brand. This
   decides canvas, type, color, and motion language before you write any HTML.
3. If authoring for a specific delivery format (9:16 shorts, 16:9
   YouTube side-overlay), also load `reference/video-authoring.md`.
4. Start from the right shell for the track: **Track A** → the full-bleed creator
   shell in §3.8; **Track B** → the dark-premium canonical shell below (grid +
   vignette + grain); **Track C** → a calm tinted/off-white frame with restrained
   type. The **Law #11 anchor and Law #12 determinism are mandatory in every
   track**; the grid/vignette/grain are Track B's spine, NOT universal.
5. Add scene content. Use motion vocabulary from §1.4 + plate moves from §3.8–3.9.
   Pick eases from §2. Respect the track's pacing (§1.5 hero/promo, §1.6 companion).
6. After rendering, run `motion.verify-frames` and READ each frame.
7. Run the pre-flight checklist (§4) before declaring done.

Same applies for any scene: lower thirds, intro cards, stat reveals, sponsor
reads, chapter dividers, subscribe CTAs, outro cards, spotlight rings, reaction
bursts. Every one gets the Laws treatment **in its track** — no "this is just a
small lower third, I'll skip the craft." But "gets the treatment" does NOT mean
"gets the perspective grid": a bright Track-A card with the Track-B grid jammed
under it is exactly the failure §0.5 warns about.

### Canonical composition template — the Track-B (dark-premium) shell

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

  // Ambient life — vignette breath (law #4). Attached to the paused
  // timeline (NOT a floating gsap.to) so Hyperframes' deterministic
  // seek drives every frame at the right phase. `repeat: -1` would
  // run on GSAP's internal time and produce non-deterministic frames.
  // For a 1.6s cycle yoyo over a 6s composition we need ⌈6/3.2⌉-1 = 1
  // repeat (1.6s × 2 cycle direction × (1+1 plays) = 6.4s ≈ slot).
  const BREATH_CYCLE = 1.6;
  tl.to('.vignette', {
    opacity: 0.85,
    duration: BREATH_CYCLE,
    repeat: Math.max(0, Math.ceil(SLOT_DURATION / (BREATH_CYCLE * 2)) - 1),
    yoyo: true,
    ease: 'sine.inOut',
  }, 0);

  // … your tweens here …

  // LAW #11 anchor — do not remove. Guarantees the timeline's
  // cumulative duration === SLOT_DURATION so Hyperframes captures the
  // full composition without trailing black frames.
  tl.to({}, { duration: SLOT_DURATION }, 0);
  window.__timelines['root'] = tl;
})();
</script>
</body>
</html>
```

Use this shell for **Track-B (dark-premium)** pieces — the grid, vignette, and
grain are mandatory *there*. For **Track A** start from §3.8 instead (full-bleed
brand color, heavy type, NO grid/vignette/grain). For **Track C** use a calm
tinted or off-white frame with restrained type. The anchor tween (Law #11) and
determinism (Law #12) are mandatory in every track.

---

## 8 · TL;DR — the philosophy in one sentence

> **Pick the track for the content (bright creator A / dark-premium B / keynote
> C), derive color from the creator not the editor, then: one idea per beat,
> kinetic not still, type that moves, hold the hero, breathe the outro — every
> timeline fills its slot, every render is deterministic, every exit snaps to a
> frame boundary, and no cut is ever naked.**

Load this when you load motion-philosophy.md. Then go build.
