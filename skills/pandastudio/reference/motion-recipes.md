# Motion recipes — proven, seek-safe patterns

A pattern library for `motion.render-html` / `motion.generate`. PandaStudio renders motion graphics through the HyperFrames engine (HTML + one paused GSAP timeline, frame-seekable), so every animation must be **deterministic and seek-correct** — the producer renders by jumping to arbitrary times, not by playing forward.

Use this as a menu: pick 2-4 atomic recipes per scene, compose them on the single paused timeline, and obey the determinism guardrails. Each entry names the recipe, when to reach for it, and the key seek-safe technique.

> Adapted from the HyperFrames `hyperframes-animation` skill (heygen-com/hyperframes, Apache-2.0). Patterns are CSS/GSAP and engine-version-independent. Full per-recipe source lives in that repo's `skills/hyperframes-animation/rules/<name>.md`.

## How to use

1. Read [motion-philosophy.md](motion-philosophy.md) for *what* to build (brand, beat planning, restraint).
2. Pick atomic recipes here for *how* to animate each beat.
3. Validate against the **Determinism guardrails** below before rendering — these are silent bugs `lint`/`validate` won't catch.

---

## Atomic motion recipes

### Text & typography
- **spring-pop-entrance** — the canonical entrance. Element/group springs `scale: 0→1` with `back.out` overshoot. Use `fromTo` so it's correct at t=0 under seek. Stagger groups, cap ~500ms.
- **kinetic-beat-slam** — punchy/rhythmic taglines. Short phrases slam in on one shared beat array with distinct per-phrase entrances (scale-slam / side-snap / rise-rotate), then a locked finale.
- **hacker-flip-3d** — decryption reveal. Character-level 3D rotation + deterministic per-glyph substitution (`back.out` + per-glyph `onUpdate` flicker hash).
- **discrete-text-sequence** — non-linear typing (typos, holds, bulk add, backspace): replace whole text states at time thresholds via `onUpdate` reverse search.
- **3d-text-depth-layers** — extruded big type: N offset layers at `(i·dx, i·dy)` with decreasing alpha.
- **asr-keyword-glow** — highlight keywords synced to ASR word timestamps; glow+scale+color driven through an attack-decay-rest envelope on a `--glow` CSS var.
- **context-sensitive-cursor** — typing cursor whose color switches at segment boundaries; square-wave blink via `(tl.time() % cycle) < cycle/2`.
- **dynamic-content-sequencing** — content-driven duration: precompute a flat `[{startTime,endTime,…}]` from a script; each phrase window = `chars × charSpeed + hold`. No hand-tuned offsets.

### Data & stats
- **counting-dynamic-scale** — count-up where font size grows with the value. Single tween on a numeric proxy; `Math.round`, `tabular-nums`, seek-safe `onUpdate`.
- **stat-bars-and-fills** — number paired with a graphic: growth bars (`scaleY` stagger), progress fill (`scaleX` or measured SVG ring), star-rating wipe (`clip-path`). Transforms only.

### Camera & viewport
- **viewport-change** — virtual camera (zoom/pan/focus-lock): transform one `.world` wrapper, `translate(x,y) scale(S)`; counter-translate `T = -offset × S`.
- **coordinate-target-zoom** — zoom into a non-centered element: scale outer wrapper + counter-translate inner (`T = -offset`).
- **multi-phase-camera** — cinematic pull-back / focus / push sequence + continuous micro-drift.
- **camera-cursor-tracking** — lock viewport to a moving focal point (e.g. typing cursor); measure with `getBoundingClientRect()` after `document.fonts.ready`.
- **depth-of-field-blur** — rack focus: tween `filter: blur()` (+ slight dim) on off-focus layers via a `--dof` var; focal element stays sharp. Finite, seek-safe.

### Layout & network
- **center-outward-expansion** — elements start clustered at center, expand to final positions (CSS sets targets once; GSAP tweens offsets to 0 in lockstep).
- **depth-scatter-assemble** — N elements scatter into / reassemble from a rotating 3D depth-cloud; deterministic index-derived offsets. `preserve-3d` + perspective.
- **orbit-3d-entry** — elements flip in from 3D then settle into an elliptical orbit. Critical: `gsap.set` the orbital start position *before* phase 1 (flip in place, not at center).
- **split-tilt-cards** — two comparison cards with opposing `rotationY` tilts; floating in phase opposition (`Math.PI` offset).
- **avatar-cloud-network** — avatars on an elliptical ring with SVG connection lines to center; staggered entry. Social-proof shot.
- **3d-page-scroll** — a webpage as a tilted 3D card whose content scrolls to reveal sections. Pair with `asr-keyword-glow` for on-page highlighting. Product-demo.
- **ai-tracking-box** — detection overlay: L-bracket corners + fluctuating confidence label following a target; box recomputed per frame from target position.

### SVG & icons
- **svg-path-draw** — outline draws itself via `stroke-dasharray`/`stroke-dashoffset`; measure `getTotalLength()` at setup, tween offset to 0. Rings: rotate stroke `-90deg`.
- **svg-icon-enrichment** — bring icons alive (rotating hands, pulsing dots, dash-flow). Use SVG `setAttribute('transform','rotate(deg cx cy)')` for explicit center, not CSS `transform-origin`.

### Idle & ambient
- **sine-wave-loop** — breathing/idle motion: GSAP `sine.inOut` yoyo with finite repeats, or `onUpdate` reading `tl.time()` when multiplying onto a live value.
- **ambient-glow-bloom** — untriggered soft radial glow behind a hero, peak opacity ≤ ~0.45; or a single-pass traveling sheen. Finite.

### Transitions & interaction
- **scale-swap-transition** — morph between two elements at the same center: exit shrinks+fades, entrance pops with `back.out(2)`.
- **card-morph-anchor** — container morphs apparent size + radius + surface between two shots (uniform `scale` substitutes for forbidden `width`/`height`; paint-only `borderRadius`/`background`/`boxShadow`).
- **reactive-displacement** — collision transition: an entering element's tween drives the exiting element's displacement (victim duration 40-50% of intruder's).
- **motion-blur-streak** — fake velocity blur on a fast entrance/push: SVG `feGaussianBlur` stdDeviation on the motion axis, or a deterministic echo trail that collapses into the lead.
- **press-release-spring** / **physics-press-reaction** / **cursor-click-ripple** — tactile button/click feedback: compression then spring recovery; cursor moves→depresses→emits an expanding ripple (element in DOM from t=0 at `opacity:0`).

### Effect recipes
- **gsap-effects** — drop-in timeline blocks (typewriter, audio visualizer).
- **css-marker-patterns** — marker-highlight emphasis: highlight sweep, hand-drawn circle, burst, scribble, sketch-outline (CSS + GSAP).

---

## Scene transitions (between scenes)

Whole-scene transitions, families from the HyperFrames transition registry — choose by feel, keep them finite and seek-safe:

`css-push` (slide/cover), `css-scale` (zoom in/out), `css-cover` (wipe), `css-dissolve` (cross-fade), `css-blur`, `css-3d` (flip/cube/rotate), `css-radial` (iris), `css-grid` (tiled reveal), `css-distortion`, `css-light` (flash/glow), `css-mechanical`, `css-destruction` (shatter/break).

Default to the quietest option that reads (dissolve/push/scale); reserve 3d/destruction/distortion for deliberate punctuation.

---

## Visual techniques (deeper toolbox)

SVG path drawing · Canvas 2D procedural art · CSS 3D transforms · per-word kinetic typography · Lottie · video compositing · character-by-character typing · variable-font axis animation · GSAP MotionPathPlugin · velocity-matched transitions · audio-reactive animation · clip-path reveal masks · WebGL fragment-shader art.

---

## Determinism guardrails (silent bugs — must obey)

These won't be caught by lint/validate but **break the rendered frames**:

- **One paused timeline per composition** — exactly one `gsap.timeline({ paused: true })` at `window.__timelines["<id>"]`, built synchronously at load. Render duration = root `data-duration`, not timeline length.
- **No render-time nondeterminism** — no clocks, no unseeded `Math.random`, no network, no input state. No `repeat: -1` (use a finite count).
- **Animate only visual properties** — transforms, opacity, filters, colors. Never `display`/`visibility`. No `gsap.set` on a later scene's clips (it fires at t=0 under seek).
- **`fromTo`, not `to`** for anything that must be correct at t=0 — a bare `to` leaves the element in its final state when seeked to time 0.
- **Sized root** — the standalone root needs explicit px width/height and a resolved height chain; a `height:100%` child of an unsized parent collapses to ~0 (content piles top-left). Not linted.
- **Full-screen fills go on a full-bleed child** (`position:absolute; inset:0`), never the composition root — the producer can drop the root's own `background` and render black.
- **Unique ids across the assembled page** — prefix sub-composition ids with the composition id (`#<id>-hero`). Duplicate `<video>`/`<img>` ids render blank.
- **`<video>`/`<audio>` must be a direct child of the host root** (never inside a sub-comp `<template>`); the framework owns playback.
- **No `<br>` in body text**; transformed elements must be block-level + sized.

Run `motion.verify-frames` (vision check) on midpoints before considering a render done.
