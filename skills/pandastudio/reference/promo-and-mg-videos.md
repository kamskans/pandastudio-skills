# Promo / all-motion-graphics videos — the design bar

Load this whenever the deliverable is a video **built from scratch where the
motion graphics ARE the video** — a promo, a "what's new" reel, a product
teaser, an explainer with no source footage, an intro/outro piece. (For graphics
layered OVER a recording — lower thirds, callouts on a talking head — this doc
does NOT apply; use the bundled-template path in SKILL.md.)

This is the most visible, brand-defining asset a creator makes. The bar is high,
and it is NOT cleared by "it animates." Read this before authoring scene one.

---

## The prime directive

**Every scene is a DISTINCT composition that SHOWS its point. One design system,
never one layout.**

Three rules, in priority order:

1. **SHOW, don't tell.** A feature beat must *depict* the feature, not state it in
   a headline. "Publish to Reels" → a phone mockup with a clip posting to a Reels
   UI. "One-click captions" → captions burning in word-by-word on a video frame.
   "Spotlight & blur" → the effect actually dimming/blurring a mock screen. A
   giant headline that says the feature is the lowest tier of communication.
2. **Vary the composition every scene.** Layout, dominant visual, type scale, and
   motion choreography must change beat to beat. Hold the *system* constant
   (palette, type family, motion vocabulary) so it feels cohesive — but if scene
   3 and scene 5 could be swapped by changing the text, they are the same scene.
3. **Compose 2–4 motion techniques per scene** (kinetic type + camera move +
   reveal + ambient), and **put a real transition between scenes**.

---

## Audio: decide voiceover & music FIRST (they drive timing)

Narration and a music bed are core capabilities, not afterthoughts. For a promo,
**ASK the user whether to include them before building** (`media.generate-narration`
— 3 TTS models; `media.generate-music` / `asset.list-music`) — don't silently ship
a silent video. Once decided, the order of operations matters:

- **If there's narration, generate the VO BEFORE timing any visuals.** TTS length
  is unpredictable and usually runs LONGER than your read-aloud estimate (a ~30s
  script came back 42s). Generate each line with `media.generate-narration`, read
  the returned `durationMs`, then set each scene's `data-duration` to its line
  length **+ ~0.8s** of breathing room. Author visuals first and you guarantee a
  full re-time + re-render pass once the real VO lengths land.
- **Music shorter than the timeline.** `media.generate-music` (Lyria-2) returns a
  fixed ~30s instrumental. For a longer video, in order of preference: (1) **LOOP
  it** — add the same track as back-to-back `project.add-audio` regions, each with a
  short `--fadeIn`/`--fadeOut` (~500ms) at the seam so the join is inaudible; (2)
  generate two different ~30s tracks and sequence them; (3) trim the timeline to the
  music. Bundled `asset.list-music` tracks list their real `durationMs` — pick one
  long enough and you skip looping entirely.
- `project.add-audio` honors `--fadeIn`/`--fadeOut` (ms); the export mixer applies
  them. Put a fade-out on the final music region so the bed doesn't cut off hard.

---

## ❌ The #1 promo failure — the templated slideshow

**Building one scene shell (background + eyebrow + big headline + a pill below)
and reusing it for every scene with the text swapped.** The result animates, has
a nice palette, passes the motion check — and is a flat, monotonous slideshow.
This is the single most common way an agent ships a bad promo. It happened in a
real build: seven scenes, identical layout and motion, only the words changed.

If your scenes share a layout, you have failed this doc. There is no "consistency"
defense — consistency lives in the *system* (color/type/motion), not in repeating
a composition. Do not reuse a scene template across beats.

---

## Show-don't-tell map (depict the feature, don't headline it)

| The beat is about… | DEPICT it as… |
|---|---|
| Publishing to a platform (Reels, YouTube, TikTok) | a phone/app mockup with the clip actually posting into that platform's UI |
| Captions | a video frame with captions animating on word-by-word (the karaoke highlight) |
| Spotlight / blur / focus | a mock screenshot where the effect literally dims the surround / blurs a field |
| AI voiceover / voices | waveforms that pulse like speech; a mic; one "speaking" at a time |
| AI music | a reactive equalizer / waveform moving to an implied beat + a prompt typing in |
| Languages / translation | scripts morphing between languages; a globe; text re-typing in another script |
| Faceless / shorts / multi-format | actual device frames (16:9, 9:16, a motion-graphic card) assembling |
| Speed / batch / automation | many items flowing through a pipeline, a counter racing, cards dealing out |
| A number / metric | a counter tweening up with a supporting graphic (bar/ring), not a static figure |

If a beat genuinely has no visual (a pure tagline / hook), THAT is when a bold
type-only moment is right — and it should be a *distinct*, full-bleed kinetic
treatment, not the same headline layout the other scenes use.

---

## Scene-variety matrix — rotate layout archetypes

Pick a DIFFERENT archetype for adjacent scenes. Never run the same one twice in a row:

- **Full-bleed kinetic type** — the hook / a money line. Type fills the frame, scales, slams.
- **Device-frame showcase** — a phone/laptop mockup demoing the feature (the show-don't-tell workhorse).
- **Split-screen** — concept on one side, visual on the other.
- **Centered lockup** — logo / outro / a single statement, breathing.
- **Animated diagram / flow** — how something connects or moves through steps.
- **Data-viz** — a counter, bars, a ring, a chart building.
- **Card grid / assembling cards** — multiple items (tools, formats, options) popping in with depth.
- **UI demo** — a real (mock) app surface with a cursor performing the action.

A 6–8 scene promo should use 5+ distinct archetypes. Two type-headline scenes
total is the ceiling; the rest must show something.

---

## Worked example — a distinct "show-don't-tell" scene

A "Publish to Reels" beat, depicted (phone posting), NOT a headline. Adapt the
*approach*, not the pixels. (Authoring contract — GSAP load, `data-width`,
determinism — is in [motion-recipes.md](motion-recipes.md) and
[motion-philosophy.md](motion-philosophy.md). GSAP is auto-injected as of
v1.56.0, but always load `<script src="gsap.min.js">` to be safe on older builds.)

```html
<div id="root" data-composition-id="reels" data-width="1920" data-height="1080" data-duration="6">
  <div class="bg"><!-- shared system bg --></div>
  <!-- DEPICTION: an actual phone mockup, a clip inside it, and the Reels UI -->
  <div class="phone" id="phone">
    <div class="clip" id="clip"><!-- a frame of the user's video --></div>
    <div class="reels-ui" id="ui"><!-- IG Reels chrome: avatar, caption, like/share --></div>
    <div class="post-toast" id="toast">Posted to Reels</div>
  </div>
  <div class="label" id="label">Publish in one click</div>
</div>
<script src="gsap.min.js"></script>
<script>
  const tl = gsap.timeline({ paused: true });
  tl.fromTo("#phone", {y:80,opacity:0,rotationY:-18}, {y:0,opacity:1,rotationY:0,duration:0.8,ease:"power3.out"}, 0.2);
  tl.fromTo("#clip",  {scale:1.15}, {scale:1,duration:1.2,ease:"sine.out"}, 0.4);      // clip "plays"
  tl.fromTo("#ui",    {x:40,opacity:0}, {x:0,opacity:1,duration:0.5,ease:"power2.out"}, 1.0);
  tl.fromTo("#toast", {y:20,opacity:0,scale:0.8}, {y:0,opacity:1,scale:1,duration:0.5,ease:"back.out(2)"}, 2.4); // the "post" moment
  tl.fromTo("#label", {opacity:0}, {opacity:1,duration:0.5}, 1.4);
  tl.to({}, {duration:6}, 0);
  window.__timelines = (window.__timelines||{}); window.__timelines["reels"] = tl;
</script>
```

The next scene (e.g. captions) uses a DIFFERENT archetype (a clip with karaoke
captions animating), the one after that a different one again. The phone is never
reused as a generic container for unrelated beats.

---

## Quality gate — before you call a promo done

Layout-correctness is not motion quality, and motion is not design variety.
A render can pass every technical check and still be a monotonous slideshow.
So verify all three:

1. **Motion** — AUTOMATIC on app >= 1.60: the renderer fails any clip frozen
   for >=90% of its duration (STATIC_RENDER), so a frozen scene can't reach
   you as success. Per-scene holds inside a longer video can still hide — for
   multi-scene single renders, extract 3+ frames *within each scene* and
   confirm they visibly differ.
2. **Depiction** — for each feature scene, is the feature SHOWN, or only
   headlined? Headlined → re-author to depict it.
3. **Variety** — line the scenes up. Do adjacent scenes share a layout / dominant
   visual / motion? If yes → it's the templated slideshow. Re-author to distinct
   compositions.

Only when all three pass is it shippable. "It animates" is not the bar.
