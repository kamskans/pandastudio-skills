<!-- Part of the pandastudio skill. THE house standard for every promo and motion-graphics video. -->

# Promo and motion-graphics videos: the house standard

Load this for every video where motion graphics ARE the picture: a product
launch, a promo, a feature teaser, a "what's new" reel, an app ad, an
animated explainer, an intro or outro. It is also the bar for any custom
graphic you author over footage (Mode A). There is one standard, not a menu of
styles: the **launch-film grammar**, proven by recreating a top-tier 40 s SaaS
launch film shot for shot in PandaStudio.

The how-to lives in two places:
- [saas-launch-film.md](saas-launch-film.md): the full recipe, timings and a
  snippet for every kit primitive (`lf-kit.js` / `lf-kit.css`).
- [motion-philosophy.md](motion-philosophy.md): the engine contract (seek-safe
  timelines, `data-*` attributes, determinism). Obey it; this doc does not
  repeat it.

Style still comes from the brand (colours, fonts, voice, light or dark). The
grammar below is how every brand moves.

---

## The prime directive

**The product (or the subject) is the hero, shown doing its job, in one
continuous world a camera moves through. Type is punctuation.**

- Show, never tell: a feature beat depicts the feature working (a reply types
  itself, a number rolls, a card lands, a chip flips), it does not headline it.
- Every beat changes state on screen. A frame that could be a slide is wrong.
- The same system everywhere (brand tokens, fonts, motion vocabulary), a
  different composition in every beat.

---

## Beat structure

Default 40 s; scale every time proportionally for 15-60 s. Map the product's
real surfaces onto each beat.

| # | Beat | Share | Job |
|---|---|---|---|
| 1 | Pain hook | 6% | A question or claim, word by word, while the pain piles up as objects at different depths (notifications, tabs, tools, reps). Never names the product. |
| 2 | The miss / the turn | 9% | One object alone, sharp; a serif line with ONE accent word and a hand-drawn underline. |
| 3 | Reveal | 5% | That object morphs into the real UI; the camera pulls back to the whole app, big (80-90% of frame width), with depth. |
| 4 | Logo | 5% | Wordmark resolves above the app. First time the name appears. |
| 5 | Dive in | 10% | A fast dolly into the app; something docks into place (icons into chips, a sidebar assembling). |
| 6-8 | 2-3 demos | 35% | Each a DIFFERENT interaction: type and answer, click and result, drag and sort, a before/after, a count-up. Cursor or caret drives it; callouts name the part. |
| 9 | Montage | 5% | "Every <thing>." over 3-5 fast cuts of other screens (LF.montage). |
| 10 | Proof (optional) | 6% | One real number counting up, a real testimonial, or the real integrations row. Never invented. |
| 11 | Calm / resolution | 5% | The tidy "after" state. |
| 12 | End card | 14% | Mark + wordmark, two-part tagline, CTA pill, URL, brand icons popping in. Mirrors the logo beat. Hold. |

Shorter films drop beats, never the grammar:
- **15 s teaser:** hook (1.5 s), reveal (2 s), one demo (6 s), montage (1.5 s), end card (4 s).
- **30 s promo:** hook, reveal + logo, two demos, montage, end card.
- **Consumer app** (fitness, finance, social): same beats with a phone
  (`LF.appWindow` with `chrome: "none"` in a device frame) and the pain as the
  real-life moment, not software clutter.
- **9:16:** same beats, one idea per frame vertically, type in the top 60%.

---

## The motion grammar (every beat, every film)

1. **Word-by-word blur reveals with ghost text.** Lines arrive one word at a
   time from blur, and the unread words are visible as a soft blur just ahead
   (`LF.reveal`). Letters for wordmarks (`mode: "char"`). No plain fades.
2. **Depth of field and rack focus.** Only what the viewer should read is
   sharp. Far layers are blurred; transitions between focal points are rack
   focus (`LF.rackFocus`, `LF.blurIn`, `LF.depth` layers at depth d blur by
   |ln d|).
3. **A virtual camera that never stops.** A slow linear push (2-5%) on every
   hold; fast expo dollies (`whoosh` eases: smooth start, expo tail) between
   framings, with directional motion blur only while moving (`LF.camera`).
4. **Morph continuity.** The before and after are the same element: the
   notification grows into the card, the card is inside the app window, the
   dock icons fly into the filter chips. Cuts only between chapters, and those
   are rack focus, a whip or a camera move, never a plain crossfade.
5. **Micro-interactions.** Buttons dip and spring (`LF.press`), switches flip
   (`LF.toggle`), a cursor travels on an arc and clicks with a ripple
   (`LF.cursor`), numbers roll (`LF.odometer`, `LF.countUp`), a caret types
   with human jitter (`LF.type`).
6. **Sound synced to actions.** One sound per real event, on its frame: a key
   per typed character (use `LF.type(...).times`), a click per cursor click
   (`LF.cursor(...).clicks`), a tick per montage cut (`LF.montage` returns the
   cut times), a pop per card, a swoosh only on the big camera moves (never
   two within a second), a riser into an impact on each logo. Music bed
   forward; no voiceover unless asked.

Easing: expo/quint out for arrivals, `whoosh` for camera dollies, `linear` for
pushes, a slight back-out overshoot only on small pops (icons, chips, pills).
No bouncy or elastic eases on big elements, no looping "breathing" scale.

---

## The brand-derived visual system

Set it in one call, `LF.theme({...})`, from `workspace.capture-brand
--url=<site> --apply=false` (or the workspace brand kit):

| Role | From the brand | Default |
|---|---|---|
| ground | the site's background | warm off-white (light) or near-black (dark) |
| two blob tints | primary + accent at 12-60% alpha | pale rose + pale sky |
| ink | text colour | near-black / near-white |
| accent | the ONE accent word, underline, rings, CTA | brand primary |
| serif | narrative lines | Newsreader (a Google serif that suits the brand) |
| sans | UI and wordmark | the brand's UI font, else Inter |
| dark | true for dark brands | false |

Rules: never a flat ground (the drifting blobs + grain from `LF.ground`); one
accent colour; the real logo file (check it IS the logo; capture sometimes
picks an icon; `LF.brand("<Product>")` often has the real mark); real
screenshots or a faithful HTML trace of the real UI; never invented claims,
numbers, reviews or logos.

---

## Building blocks (use these, do not hand-roll them)

| Need | Kit primitive |
|---|---|
| Brand tokens, light/dark | `LF.theme`, `LF.ground` |
| The app, big, from a real screenshot or a traced UI | `LF.appWindow` + `LF.tiltIn` + `LF.camera` |
| Floating cards / chips over the app with parallax | `LF.depth` |
| "Look at this part" | `LF.popout` (lift a region), `LF.callout` (ring + label + leader) |
| Actor | `LF.cursor`, `LF.press`, `LF.type` |
| Integrations, dock, platforms | `LF.brand`, `LF.brandRow` (3,300 real brand icons) |
| Proof | `LF.countUp`, `LF.odometer`, `LF.testimonial` |
| Comparison | `LF.beforeAfter` |
| Montage | `LF.montage` |
| Pricing | `LF.pricing` |
| End card | `LF.endCard` |
| Named looks the kit lacks | `motion.catalog` (adapt its timeline into the kit grammar) |

Reference the kit by name and PandaStudio inlines it (renders, screenshots,
render-film, templates):

```html
<link rel="stylesheet" href="_shared/launchfilm/lf-kit.css">
<script src="_shared/gsap.min.js"></script>
<script src="_shared/launchfilm/lf-kit.js"></script>
```

---

## Audio decisions

- Launch films and promos: music bed + sound design, no voiceover by default.
  Ask once if the user has not said; with a voiceover, generate it FIRST and
  time beats to its lines (media-generation.md).
- Beds built for this: `launch-pulse`, `quiet-launch` (`asset.list-music`);
  loop with a second region and a 600 ms crossfade when the film is longer.
- Sounds from `asset.list-sounds` by their current names (`message-pop`,
  `swoosh-fast`, `ui-tick`, `marker-strike`, `keyboard-key-1/2/3`,
  `keyboard-space`, `keyboard-typing-fast`, `mouse-click`, `success-chime`,
  `noise-riser`, `logo-impact`). Place them in one `project.batch`.

---

## Mandatory verification (do all of it before you hand anything over)

1. **Render-sheet review.** For every scene, `motion.screenshot` at each reveal
   and one mid-move; after assembly, `project.render-sheet` (or frames every
   1 s tiled) of the whole film. Look at it. Fix, re-render, look again.
2. **Text inside the frame.** No word clipped by the frame edge or hidden
   behind another layer on any hold; nothing important in the outer 4% (safe
   area); type ≥ 20 px at 1080p after camera scale.
3. **Timing.** Every line holds long enough to read (≈ 0.3 s per word + 0.8 s);
   the reveal lands before the camera leaves; the film length matches the
   brief ±0.5 s; no dead frame (every hold has a push).
4. **Sound sync.** Each sound sits on its event's frame (±1 frame); no two
   swooshes within a second; the bed fades out on the end card.
5. **Loudness.** `export.start`, then `export.verify`: integrated loudness
   about -14 LUFS (export normalises), true peak ≤ -1 dBTP, duration equal to
   the timeline.
6. **Grammar check.** Word reveals with ghost text; depth of field on every
   hold; the camera always moving; one morph linking pain to product; each demo
   a different interaction; real icons, not initials; the end card mirrors the
   logo beat.

Report the checks you ran and what you fixed. "It renders" is not the bar.

---

## Anti-patterns (each one fails the standard)

- The templated slideshow: one shell (background + headline + pill) reused per
  beat with the text swapped.
- A small, flat app: a screenshot sitting at 50% of the frame with no depth,
  no camera and no parallax. The reveal must feel big.
- One demo and no montage in a 30 s+ film.
- Integration icons as initials or grey circles (use `LF.brand`).
- Headlines that state a benefit the UI could have shown.
- Plain opacity fades, bouncy/elastic eases, looping breathing scale.
- Hard cuts or crossfades mid-chapter; a camera that stops.
- Sounds scattered on a grid instead of on events; stacked swooshes.
- Invented numbers, reviews, customer logos or screens.
