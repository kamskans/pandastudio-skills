<!-- Part of the pandastudio skill. The method behind the "SaaS launch film" recipe. -->

# SaaS launch film: the Twitter-launch look, for any product

The recipe that runs this method is `motion-graphics-launch` ("Product launch
film", app 2.0.2; its old id `saas-launch-film` still resolves). With a screen
recording of the product, `product-promo-from-url` puts the real recording
inside the same kit (see "The user's screen recording inside the window").

Use this for "make a professional launch video for <product>", "a launch film
like the ones on Twitter/X", "a product teaser with motion graphics", for the
user's own product or a well-known SaaS, and as the build manual for EVERY
promo and motion-graphics video: the house standard
([promo-and-mg-videos.md](promo-and-mg-videos.md)) is this grammar. It is the
method reverse-engineered from a 40 s reference film (a fictional unified
inbox, "UnifiedChat") and proven by recreating that film shot for shot in
PandaStudio.

This doc is the specific, repeatable recipe: story beats with timings, the
motion grammar, the visual system, how to build the product UI, sound, and
verification, plus a shared motion kit so every primitive is one call.
[launch-video.md](launch-video.md) adds the HyperFrames catalog and
`motion.render-film` for named looks the kit lacks.

## What makes this look

1. **The product UI is the hero.** Every beat shows the real interface doing
   something: a card arrives, a reply types itself, a number rolls, a chip
   lands. Nothing is a slide about a feature.
2. **One product world, continuous morphs.** A notification card grows into a
   full card, which becomes one card in a grid, which is revealed to be inside
   the app window. Icons in a dock fly into filter chips. Cuts happen only
   between chapters.
3. **Depth of field everywhere.** Whatever the viewer should read is sharp;
   everything else is blurred. Transitions are rack focus, not fades.
4. **A camera that never stops.** A slow push on every hold; fast whoosh
   dollies (smooth start, expo tail) with motion blur between framings.
5. **Editorial type as punctuation.** Two or three serif lines, revealed word
   by word with the unread words visible as a blur ahead of the reader, one
   italic accent word in the brand colour with a hand-drawn underline.
6. **A calm, bright ground.** Warm off-white with two huge soft colour blobs
   that drift. Never flat, never dark.
7. **Sound design carries it.** Music bed forward, one sound per real event.
   No voiceover.

## Story structure (40 s default; scale proportionally for 20-60 s)

| # | Beat | Time | What happens | Archetype |
|---|---|---|---|---|
| 1 | Pain hook | 0-2.6 | Serif question word by word ("How many apps did you check today?") while notification cards fly in at different depths and app-icon counters tick up. Ends by rushing past the camera. | kinetic type + chaos |
| 2 | The miss | 2.6-6.0 | One card drifts in from the distance, sharp and alone. A status tag ticks ("2h", "Yesterday", "No reply · 3 days"). Line: "And you still missed *this one.*" with underline. | single object |
| 3 | Reveal | 6.0-8.0 | That card expands, siblings slide in, rows assemble, the camera pulls back and the window chrome materialises around them. | morph + pull-back |
| 4 | Logo | 8.0-10.0 | Wordmark resolves letter by letter above the window; the dock sits below. | lockup |
| 5 | Dive in | 10.0-14.0 | Dock icons fly into the window as filter chips while the camera dollies to the top-left; the header text writes itself (rack focus onto it). | camera dolly |
| 6-8 | 2-3 feature demos | 14-28 | Each a different interaction: ask-and-answer (typing, button press, answer card with odometer + underline); a thread where a reply writes itself and sends (whoosh up, typing dots, answer). | UI demo |
| 9 | Montage | 28-30 | "Every conversation." over 4-5 very fast cuts of other screens, each a short push. | montage |
| 10 | Proactive moment | 30-32.5 | A macOS notification from the product slides in over the blurred app. | notification |
| 11 | Calm | 32.5-34.5 | Blur clears onto a tidy, "all done" version of the app. | resolution |
| 12 | End card | 34.5-40 | Wordmark, tagline in two parts ("Every conversation." then "One calm inbox." in grey), small CTA line, app/integration icons popping in one by one. Hold. | lockup |

Rules: the pain beat never names the product; the product name first appears
at the logo beat; each demo is a DIFFERENT interaction; the end card mirrors
the logo beat.

## Picking the product and its real assets

- **The user's own product:** `workspace.capture-brand --url=<site> --apply=false`
  for colours, fonts, logo and viewport screenshots of the site (1920x1080,
  1x; marketing pages usually show the real UI). Better still: ask for 2x
  screenshots of the app itself (a Retina screenshot is 2x) or a screen
  recording to pull stills from.
- **A well-known SaaS** (Linear, Notion, Figma...): same, from its public site.
  Crop the real UI out of the capture-brand screenshots for montage panels and
  pop-outs; for the hero and the close-ups, trace the UI in HTML (below) so
  text stays sharp at 2x.
- **The logo:** capture sometimes picks a UI icon. Check it; `LF.brand("<Product>")`
  draws the real mark for ~3,300 brands (Simple Icons).
- Never invent claims or numbers about a real product; demo copy is scenario
  content (names, messages), not performance claims.

## Visual system from the brand

Map the brand onto these roles and set them as CSS tokens (`:root` of
`lf-kit.css`, override per film):

| Token | Role | Reference film |
|---|---|---|
| `--lf-ground` | canvas | warm off-white #fcfbf9 |
| blob colours | two soft brand tints on the ground | pale rose + pale sky |
| `--lf-ink` | text | #17171a |
| `--lf-accent` | the one accent word + underline | crimson #c0283f |
| `--lf-serif` | narrative lines | Newsreader |
| `--lf-sans` | UI | Inter |
| wordmark | product logo / name | Inter 700, tight tracking |

A dark brand (Linear) keeps the same grammar on a near-black ground with
lighter blobs; the accent is the brand colour. Fonts: name Google fonts in CSS
and they embed automatically.

## Building the UI mockup

Two sources, often mixed in one film:
- **A real screenshot** as a large app window: `LF.appWindow(world, { src,
  size, scale })` (see "App from screenshots" below). Crisp up to the camera
  zoom the capture allows: 1x screenshots at `scale: 1` stay sharp to about
  1.25x; 2x screenshots at `scale: 0.5` to 2x.
- **A traced HTML version** of the real screens, when the UI has to change
  state (typing, cards arriving) or be seen close. Build it as below and pass
  it as `html` to `LF.appWindow` so it gets the same chrome, depth and moves.

- Build the app window once at native size (the reference used 1312x902) as
  an HTML builder function, and reuse it in every scene so cuts and morphs line
  up. `lf-kit.css` ships window chrome (`.lf-win`, `.lf-winbar`,
  `.lf-lights`), bubbles, pills, buttons, toasts and a dock.
- Put the window in a **world** element and move a **camera** over it. Measure
  positions in window coordinates; frame shots with `LF.frame(cx, cy, zoom)`.
- Trace real screens for layout (sidebar widths, card sizes, font sizes), then
  fill with scenario content. Avatars: `media.generate-image --aspectRatio=1:1`
  headshots, or initials.
- Keep UI text 13-16 px at native size; the camera zooms 1.4-2x for close-ups,
  so type stays crisp.

## The motion kit (`lf-kit.js` / `lf-kit.css`)

Reference the kit by name and PandaStudio inlines the bundled copy at render,
screenshot and render-film time (like GSAP), so it works with inline `html`:

```html
<link rel="stylesheet" href="_shared/launchfilm/lf-kit.css">
<script src="_shared/gsap.min.js"></script>
<script src="_shared/launchfilm/lf-kit.js"></script>
```

(Any path ending in `lf-kit.js` / `lf-kit.css` is inlined; bundled templates
use these exact paths.)

Everything is seek-safe: plain GSAP tweens on the paused timeline, plus
`LF.clock(tl, duration)` which drives functions of time registered with
`LF.onFrame(fn)` (camera, typing, odometer, counters). Call `LF.clock` last; it
is also the full-length anchor tween.

### Scene skeleton

```html
<div id="root" class="lf-root" data-composition-id="demo" data-width="1920" data-height="1080" data-duration="6">
  <div class="lf-lens" id="lens"><div class="lf-world" id="world"><!-- the product UI --></div></div>
  <!-- screen-space layers (titles, dock, notification) go here -->
</div>
<script>
  const tl = gsap.timeline({ paused: true });
  // brand -> tokens in one call; returns two drifting blob tints for the ground
  const blobs = LF.theme({ accent: "#c0283f", serif: "Newsreader", sans: "Inter",
    blobs: ["rgba(247,208,211,0.6)", "rgba(206,226,245,0.7)"] }, { dur: 6 });
  LF.ground("#root", blobs); // dark brands: LF.theme({ dark: true, ... })
  // ... primitives below ...
  LF.clock(tl, 6);
  window.__timelines = { demo: tl };
</script>
```

### Word reveal with ghost text

```js
LF.reveal(tl, "#headline", 0.1, { stagger: 0.11, blur: 13, y: 8, dur: 0.5, ghost: 0.42, ghostLead: 0.14 });
// explicit per-word times: { times: [0, 0.1, 0.22, 0.39] }; letters: { mode: "char", stagger: 0.028 }
```

Accent word: `<em class="lf-accent">this one.</em>` inside the line.

### Hand-drawn underline

```js
const ul = LF.underline("#headline .lf-accent", { host: "#world", width: 1.8, wobble: 2.2, color: "#d45a6c" });
LF.draw(tl, ul, 1.3, 0.42);
```

### Rack focus and depth of field

```js
LF.rackFocus(tl, ["#input", "#heading"], "#answer", 2.0, { blur: 5.5, dur: 0.45 });
gsap.set(".far-card", { filter: "blur(3px)" }); // static depth
LF.blurIn(tl, "#answer", 1.9, { blur: 16, y: 40, dur: 0.4 });
```

### Virtual camera with motion blur

```js
const cam = LF.camera("#lens", "#world", [
  { t: 0,    ...LF.frame(240, 380, 1.83) },                  // close on a card
  { t: 3.4,  ...LF.frame(240, 366, 1.93), ease: "linear" },  // slow push
  { t: 3.9,  ...LF.frame(657, 413, 1.36), ease: "whooshLate" }, // fast pull back
  { t: 5.0,  ...LF.frame(659, 444, 0.68), ease: "whoosh" },
], { mb: 1.0 });
// cam.toScreen({x, y}, t) -> where a world point is on screen at time t
```

Eases: `linear` for pushes, `whoosh`/`whooshLate`/`whooshLater` for dollies
(start from rest, expo tail; later = slower start), `quintInOut` for whips.
Never start an expo-out move straight from a hold: the velocity jump reads as
a glitch. Motion blur is directional and scales with screen speed.

### Morph continuity

Make the "before" and "after" the same element. The notification card is the
grid card with a compact layer on top:

```js
tl.to(card, { height: 188, duration: 0.36, ease: "power3.inOut" }, 3.35);
tl.to(compactLayer, { opacity: 0, filter: "blur(4px)", duration: 0.16 }, 3.35);
tl.fromTo(fullLayer, { opacity: 0 }, { opacity: 1, duration: 0.26, immediateRender: false }, 3.45);
tl.fromTo(siblings, { x: 160, opacity: 0, filter: "blur(10px)" }, { x: 0, opacity: 1, filter: "blur(0px)", duration: 0.45, ease: "expo.out", stagger: 0.06 }, 3.85);
tl.fromTo(windowChrome, { opacity: 0 }, { opacity: 1, duration: 0.28 }, 4.8); // the grid was inside the app all along
```

Icons flying into chips: compute each target once from layout offsets, then
fly in screen space along an arc to `cam.toScreen(target, landTime)`, shrinking
to the chip icon size, and open the chip (`width: 0` to `auto`) as it lands.

### Typing with caret (and "AI writes" word mode)

```js
LF.type("#input", "What price did we agree with Karan?", 0.1, { dur: 0.8, placeholder: "Ask anything", caretUntil: 1.05 });
const t = LF.type("#composer", reply, 2.3, { words: true, dur: 2.5, ghostNext: true });
// t.times[] = when each character lands -> place key-click sounds
tl.fromTo("#send", { backgroundColor: "#c4c1bc" }, { backgroundColor: "#17171a", duration: 0.12 }, t.end); // button state
```

### Odometer number

```js
LF.odometer("#price", "$4,800", 1.6, { dur: 0.42, stagger: 0.05 });
```

### Counters, pops, notification

```js
LF.counter(el, [[0, 9], [1.0, 10], [1.8, 14]]);          // ticking badge with a pop
LF.pop(tl, ".icon", 6.0, { stagger: 0.1, from: 0.5 });   // staggered icon pops, slight overshoot
LF.notify(tl, "#notif", 0.05, { fromY: -36, out: 1.95 }); // macOS notification in, hold, out
```

UI glyphs: `LF.icon("search"|"play"|"arrow"|"back"|"checks"|"inbox", px)`.
Avatar with platform badge: `LF.avatar(src, "linkedin", 44, initials)`.

### Real brand icons (integrations, dock, platforms)

```js
LF.brand("Notion", 56);                       // app-icon tile in the brand colour
LF.brand("GitHub", 40, { variant: "color" }); // the glyph alone ("mono" = currentColor)
document.querySelector("#row").appendChild(LF.brandRow(["Slack", "GitHub", "Figma", "Jira"], 64));
```

~3,300 brands from Simple Icons (CC0; per-icon licences checked at import),
matched by name or slug ("Google Drive" = "googledrive"); Slack, LinkedIn,
WhatsApp, Instagram, Gmail and Google Chat use the kit's own drawn marks.
PandaStudio inlines only the brands your HTML names, so write the names as
string literals. An unknown name renders a monogram tile and logs a warning;
near-black marks are lifted to grey on dark grounds. Never initials for a
brand that exists.

### App from screenshots (the big reveal)

```js
const app = LF.appWindow("#world", {
  src: "app@2x.png", size: [2880, 1800], scale: 0.5,   // 1440x900 CSS, sharp to 2x zoom
  chrome: "browser", url: "linear.app",                 // or "mac" | "phone" | "none"
  x: 240, y: 90,                                        // world position
});                                                    // or { html: tracedUI, width, height }
LF.tiltIn(tl, app.el, 0.3);                             // swings up out of depth, settles flat
const cam = LF.camera("#lens", "#world", [
  { t: 0,   ...LF.frame(960, 560, 0.95) },
  { t: 2.4, ...LF.frame(960, 540, 1.02), ease: "linear" },           // slow push on the hold
  { t: 3.0, ...LF.frame(app.centre(300, 200).x, app.centre(300, 200).y, 1.6), ease: "whoosh" },
]);
// floating layers over the app, with parallax and depth blur (host is a screen layer outside the lens)
LF.depth(cam, "#fx", [{ el: chipEl, x: 1500, y: 260, d: 1.3 }, { el: iconsEl, x: 400, y: 900, d: 1.45 }], { dof: 10 });
// lift a region (screenshot px) toward the camera while the window dims
LF.popout(tl, app, [96, 64, 640, 150], 3.2, { scale: 1.4, dy: -30, back: 5.2 });
```

Make it big: at the reveal the window spans 80-90% of the frame width, and the
first close-up is 1.4-2x. `app.point(px, py)`, `app.rect(...)` and
`app.centre(...)` convert screenshot pixels to world coordinates for the
camera, cursor and callouts.

### The user's screen recording inside the window

A `<video>` inside `LF.appWindow` plays in sync with the film (video time =
composition time from 0), so cut one file per scene that starts where the
scene starts, speed up waiting with ffmpeg (`setpts`), and mount it at the
recording's CSS size:

```js
const app = LF.appWindow("#world", { html: '<video src="beat1.mp4" muted playsinline style="width:1600px;height:1000px;display:block"></video>',
  width: 1600, height: 1000, x: 160, y: 60 });   // a 3200x2000 (2x) recording: sharp to a 2x camera
LF.tiltIn(tl, app.el, 2.75);
```

Map every click in the recording through your cuts and speed changes to place
its `mouse-click`, and push the camera to the click's world position
(`x + css_x, y + barH + css_y`).

One pointer, and it must not move when it clicks. Don't add `LF.cursor` over a
recording that already shows a pointer. Before building the film, step through
each click in the recording frame by frame: the pointer stays on the click point
while it presses. If you draw your own pointer into a CDP capture, position it
with the CSS `translate` property (or `left`/`top`) and press it with `scale`
about its tip (`transform-origin` at the tip). Positioning it with `transform`
while animating the separate `scale` property scales about the page's top-left
corner, so the pointer slides toward the corner and back on every click.

### Cursor, press, toggle (micro-interactions)

```js
const cur = LF.cursor("#world", [
  { t: 1.0, x: 1300, y: 900 },
  { t: 1.8, x: 820, y: 480, click: true },   // arcs there, presses, ripples
  { t: 2.6, x: 1040, y: 690 },
]);                                          // cur.clicks -> mouse-click sounds
// (x, y) is the arrow's tip. On a click the tip stays on the point for the whole
// press (a 0.9 dip about the tip), then leaves; give the key after a drag
// `drag: true` so the pointer moves off at once with the dragged item.
LF.press(tl, "#send", 1.8, { to: { backgroundColor: "#17171a" } });
LF.toggle(tl, "#notify-switch", 2.2);        // <span class="lf-toggle"><i></i></span>
```

### Feature callout

```js
LF.callout(tl, "#world", app.rect(700, 60, 600, 140), 3.5,
  { label: "Agents draft the fix", kicker: "New", side: "right", dim: 0.45 });
```

Ring (drawn in), optional dim around it, a leader line drawn to a label card
whose text reveals word by word.

### Before / after

```js
// #ba holds two same-size panels; #after is the second
LF.beforeAfter(tl, "#ba", null, "#after", 1.0, { width: 1400, to: 0.5, dur: 0.9, labels: ["Before", "After"] });
```

### Stat count-up

```js
LF.countUp("#num", 12480, 1.2, { dur: 1.0, suffix: "+", bar: "#bar", frac: 0.82 });
// ring: { ring: "#ringCircle" } draws an SVG circle to frac; decimals, prefix, commas
```

Only real numbers (the user's, or public facts about the product).

### Montage (3-5 fast cuts with a line over them)

```js
// each panel: a full-frame element inside #mont (an app window crop, a screenshot, a traced screen)
const cuts = LF.montage(tl, "#mont", ["#p1", "#p2", "#p3", "#p4"], 0.1,
  { each: 0.5, cuts: ["push", "whip", "zoom", "whip"], line: "#every" });
// cuts -> a ui-tick (or one swoosh-fast on a whip) at each time
```

### Testimonial, pricing

```js
const card = LF.testimonial({ quote, name, role, avatar: "av.jpg", brand: "Vercel", stars: 5 });
host.appendChild(card); LF.testimonialIn(tl, card, 0.4);
const pr = LF.pricing({ tiers: [{ name: "Free", price: "$0" }, { name: "Pro", price: "$16", period: "/mo", features: [...], highlight: true }] });
host.appendChild(pr); LF.pricingIn(tl, pr, 0.3);   // lifts + rack-focuses the highlight, rolls its price
```

Only real quotes and real prices.

### End card

```js
const t = LF.endCard(tl, "#end", {
  logo: LF.brand("Linear", 96, { variant: "color" }),  // or "logo.svg"
  wordmark: "Linear",
  tagline: ["The product development system", "for teams and agents."],
  cta: "Get started", url: "linear.app",
  icons: ["GitHub", "Slack", "Figma", "Sentry", "Zendesk"],
  at: 0.3,
});  // t.logo, t.tagline, t.cta, t.icons[] -> logo-impact, ticks, pops
```

### Gotchas (all hit while building the reference)

- A second `fromTo` on the same element needs `immediateRender: false`, or it
  overwrites the first tween's starting state at t=0.
- Transforms do nothing on inline spans: make slid/scaled spans
  `display: inline-block`.
- `.lf-world` has a large fixed size so absolutely placed text doesn't wrap;
  keep your own containers sized.
- Put the motion blur on the lens (screen space), depth blur on elements
  (world space, so it scales with the camera).
- No `Math.random`: use `LF.rng(seed)`.
- A traced UI passed to `LF.appWindow({ html })` still paints its own
  background. If the window must stay invisible until a morph lands in it,
  hide the window element (`opacity: 0` until the landing frame) and fade its
  ground in, or it covers anything behind it in the world.
- Web fonts are embedded only when the scene HTML names them in a CSS
  `font-family:` rule. A font that appears only in a JS string (injected CSS)
  or a `font:` shorthand renders in a serif fallback: add
  `<style>.fonts { font-family: "Bricolage Grotesque"; }</style>` for each.
- `message-pop` is a 5.7 s file: cut it with `endMs` (about 0.45 s after the
  start) or it rings under the next beat.
- Screen-space type over a busy UI needs a scrim (a bottom gradient) to stay
  readable; UI text inside `LF.appWindow` may be cropped by the camera, the
  captions and titles may not.

## Building and rendering

- One HTML composition per chapter; a single longer composition where a morph
  or camera move must stay continuous (the reference: hook 2.65 s,
  miss-to-brief 11.3 s, ask 4 s, reply 10 s, montage 2 s, finale 10 s).
- Pre-flight with `motion.screenshot --htmlPath=... --atMs=...` at 3-8 moments
  per scene; tile them and fix layout before rendering.
- Render with `motion.render-html --htmlPath=... --frameRate=60 --width=1920
  --height=1080` (60 fps is cheap: ~10 captured frames/s at 1080p). Or
  `motion.render-film` when you want built-in transitions.
- Assemble: `project.new`, one `project.add-clip` per scene in order, then
  audio with `project.batch`. A timeline of 60 fps renders exports at 60 fps
  on its own (frame rate `auto`); `export.start`'s result `frameRate.fps`
  confirms it. Pass `--frameRate=60` if you mix in a screen recording.

## Sound

- Bed: `launch-pulse` (or `quiet-launch`) from `asset.list-music`, forward in
  the mix; loop it with a second region and a 600 ms crossfade for films over
  33 s; fade out over the end card.
- Sound design is half this style. Follow the sound map in
  [`audio-color-music.md`](audio-color-music.md#sound-design-the-sound-map)
  (which action gets which cue, volumes, restraint, syncing to keyframes).
  For this film: a small pop (`notif-pop-small-*`, vol ~0.35) as cards land,
  `motion-whoosh-medium-*` only on the big camera moves (hook collapse,
  pull-back, dolly, send), never two within a second; `ui-tap-*` for chips and
  montage cuts; `outcome-marker-underline` under each underline; one
  `type-key-soft-*` per typed character (the `times` from `LF.type`, rotate
  variants, vol ~0.3), `type-enter-*` on send; `ui-click-soft-*` on button
  presses; `notif-message-in-*` / `notif-chime-*` for the product's
  notification; `impact-riser-short-*` ending on each logo frame into
  `impact-logo-sting-*`.
- Build the cue list from the scene timelines (scene start on the project
  timeline + the tween time in the scene) and place it in ONE call:
  `project.add-sound-cues --group=sfx --cues=@cues.json`; retime and re-run
  the same group. Dip the bed under dense clusters (typing, montage) with
  `project.set-volume-keyframes --target=audio`.
- Export normalises to -14 LUFS / -1 dBTP; no captions (the type is on screen).

## Verification

Run the mandatory checklist in [promo-and-mg-videos.md](promo-and-mg-videos.md)
(render sheet, text inside the frame, timing, sound sync, loudness, grammar).
For this recipe specifically:

1. Per scene: screenshots at every reveal and one mid-move (motion blur should
   be visible only while moving).
2. Whole film: extract frames every 1 s and tile them; if you are matching a
   reference, pair reference and yours at the same timestamps
   (`ffmpeg -ss t -i ... -frames:v 1`, hstack) and fix anything off by more
   than ~0.2 s or ~20 px.
3. `export.start`, then check the export's `loudness` and that the duration
   matches the timeline.
4. Checklist: the UI changes state in every beat; a morph links pain to
   product; each demo is a different interaction; depth of field on every
   hold; camera always moving; accent word + underline used at most twice; end
   card mirrors the logo beat; no invented claims about a real product.
