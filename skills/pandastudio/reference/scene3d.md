# 3D scenes (`motion.render-scene`)

A 3D scene is a three.js world described as JSON: clay or toon set pieces on a
stage (an open book, a desk, a plinth), worlds that pop in and out, the user's
own product screens standing in the world with a cursor that clicks, highlights
and ticks, a camera path per shot, big headlines and an HTML end card. Use it
for **product promos, launch reels, education/app ads and explainers in the
pop-up 3D look** (the "clay diorama + app card" reels). It renders through the
same engine as `motion.render-html`, so the result places, edits and
re-renders like any graphic.

When NOT to use it: a talking-head edit (use styles/recipes), a flat 2D
graphic or lower third (templates / `motion.render-html`), a real screen
recording inside a window (the launch-film kit, `reference/saas-launch-film.md`).

## Workflow

1. **Get the screens first.** The product's UI comes from images: the user's
   screenshots, or ones you capture (`style.capture` for a page,
   `motion.screenshot` of an HTML mock-up of a card in the product's look).
   One image per state the viewer should see (an empty form, then overlays add
   the typing). Use absolute paths. Never invent a product screen the user
   doesn't have: ask for screenshots when there are none.
2. `motion.scene-kit` (once): the kinds, their params, the fields, the units.
   `motion.scene-kit --starter=popup-book-promo` returns a complete scene to
   copy; `device-hero` is a single phone on a plinth.
3. Write the scene: shots (background + camera keys), a persistent stage
   (`objects`), one `set` per world, one `screen` that carries every UI state
   (`content` items switch by time), its `overlays` for the interaction,
   `texts` for headlines, `overlays` for the end card.
4. `motion.scene-stills` with the times that matter (each shot's middle, each
   click). Read `sheetPath`. Fix framing, crowding and overlay positions.
5. Sound: `media.compose-soundtrack` with cues on the shot cuts (whoosh),
   the set pops (pop, repeat), clicks, typing and ticks (shimmer); pass the
   audio as `audioPath`.
6. `motion.render-scene` → `job.wait` → place it (`project.add-motion-graphic`
   with the jobId) or hand over the MP4. Keep `scenePath`: to change anything,
   edit that file and render again with `--scenePath`.

## Units and conventions

- World units. The `book` is 12.2 wide and 6.8 deep with its pages' top at
  **y 0.86**: put a world's pieces at `y 0.9` to stand on the pages.
  Backdrops go behind (z -2.5 to -5), props in front (z 1 to 3).
- The default camera looks from about `(0, 12, 24)` at `(0, 4, 0)` with fov
  28. Closer, more dramatic: `(0, 11, 22)`. A wide grid: `(0, 60, 70)` with
  `fog: false` on that shot.
- **Times are seconds on the scene clock** (absolute), for shots, sets, keys,
  overlays and texts alike. Angles are degrees.
- Screen overlay `x`, `y`, `rect` are **0-1 fractions of the screen** (x right,
  y down). Measure them on the screenshot: `x = px / imageWidth`.
- Colours are CSS hex. Materials: clay (default, soft sheen), toon, soft,
  gloss, glass, metal, paper, emissive.

## The pieces

Stage: `book` (opens with `open: { at, dur }`), `desk`, `plinth`,
`closedBook` (stack them with titles). Backdrops: `wall` (bricks + crenels),
`building` (windows), `tower`, `mountain`, `hill`, `leaf` rows, `cloud`, `sun`.
Props: `house`, `tree`, `bush`, `tufts`, `flower`, `pot`, `cubes`, `arch`
(with `ticks`: a protractor), `ruler`, `balloon`, `scatter` (confetti of small
shapes; `float: true` makes them spin in the air), `label` (text on a plate).
Paper craft: `origami` with `shape` sheet | boat | plane | whale | owl | house |
rocket (flat facets, two-sided paper), `waves` (layered paper sea), `ship`
(paper sailing ship), `disc` (a flat sun behind a world). The paper-craft
story: one `sheet` that leaves with `exit: { style: "unfold" }` while a
`boat` arrives with `enter: { style: "unfold" }`, a paper `plane` flying
across between worlds (`keys` from one side to the other with a little roll),
each shape carrying its own screen (`parent: "<shape id>"`).
Characters: `mascot` (round, eyes, cheeks, waving arm, `hat`). Primitives:
`box`, `sphere`, `capsule`, `cylinder`, `cone`, `torus`, `star`, `plane`.
Build anything else from primitives in a `group` with `children` (a fox: a
capsule body, a 4-sided cone head and ears, a cone tail).

Every object takes `at`, `rot`, `scale`, `color`, `material`, `keys`
(`[{ t, at?, rot?, scale?, ease? }]`), `show: { from, to }`,
`enter` / `exit: { at, dur, style, from }`, `float`, `spin`, `children`.
Styles: `pop` (scale, the default), `unfold` (opens up from flat, folds
away), `spin` (turns in), `fly` (arrives from / leaves toward the `from`
offset with a roll). Nothing is drawn before an entrance starts or after an
exit ends.

## Sets: worlds that pop

```json
{ "id": "english", "from": 7.4, "to": 12.2, "pop": { "in": 0.35, "out": 0.35, "stagger": 0.025 },
  "objects": [ { "kind": "wall", "at": [0, 0.9, -3.6], "color": "#8d5ce6" }, … ] }
```

Every object scales up with a springy ease at `from` (each one `stagger` s
after the previous, or its own `delay`) and shrinks away at `to`. Make
consecutive sets touch (`to` of one = `from` of the next) so the world swaps
while the stage stays. A set that spans two shots stays up across the cut.
`pop: false` = plain cuts.

Grid of mini worlds ("all your subjects"): a set of `instance`s, each
`{ kind: "instance", of: "<set id>", at, screens: [{ id: "<screen>", frameAt: <s> }], children: [{ kind: "book" }] }`
(a frozen copy of that world and the screen as it looked at `frameAt`).

## Screens: the product in the world

```json
{ "id": "app", "kind": "card", "at": [0, 4.7, 0.2], "rot": [-12, 0, 0], "scale": 0.9, "size": [7.4, 4.6],
  "enter": { "at": 0.35, "dur": 0.65 },
  "content": [ { "image": "/abs/maths.png", "from": 0, "to": 7.4, "fade": 0 },
               { "image": "/abs/english.png", "from": 7.4, "to": 12.2, "fade": 0 } ],
  "overlays": [ … ] }
```

Kinds: `card` (white rounded slab, the app-card look), `phone`, `tablet`,
`laptop`, `plain` (just the picture: a label on a paper shape). `parent: "<object id>"`
puts the screen on that object: `at`/`rot` become relative to it and the
screen moves, pops and leaves with it (a strip on the front of a boat). `size` should match the screenshot's aspect (`fit: cover`
crops the rest). One screen carrying every state keeps the card in place while
the worlds change around it.

Overlays (drawn on the screen in order, so a `box` can cover part of the
screenshot before `text` is drawn on it):

- `cursor { path: [{ t, x, y, click? }], tag?, from?, to? }`: glides between
  points (0.6 s before each), dips and ripples on `click`. `tag` is a name
  pill (a collaborator's cursor). Start and end off-screen (`x 1.07, y 1.05`).
- `text { from, to, x, y, text, size, align, weight, color, typing: { from, to }, move: { from, to, x, y } }`:
  typed answers, labels, chips sliding into place. `size` is a fraction of
  the screen height (0.05 ≈ body text).
- `box { from, to, rect, fill, stroke, radius, move: { from, to, rect } }`:
  a filled answer field, a label chip, a card that slides into a column, a
  white patch that hides something in the screenshot.
- `highlight { from, to, rect, color }`: a marker under words (multiply).
- `tick { t, to?, x, y }`: the green check that pops in. Give it a `to` (the
  end of that screen state) or it stays.
- `image { from, to, image, rect, move }`: a sticker, an icon, a photo.

## Shots and camera

```json
{ "id": "maths", "from": 0, "to": 5, "background": "#7d43e0",
  "camera": { "keys": [ { "t": 0, "pos": [0, 13, 24.5], "look": [0, 4, 0] }, { "t": 5, "pos": [1, 12, 22.5], "look": [0.2, 4.2, 0] } ] } }
```

A shot can float its 3D on a designed background instead of a ground:
`backdrop: "linear-gradient(180deg,#e7faf0,#c9efdc 52%,#f5fffa)"` (no ground,
no fog): the launch-film look of app windows (`card` screens with
`bodyColor: "#17171c"`) turning slowly in space with a big word above them.
`ground: false` drops just the ground.

The camera cuts between shots and eases between a shot's keys: a slow drift
of 1-2 units sideways and a little closer per shot reads as alive. The ground
takes the shot's background unless `ground` is set. A first shot on a strong
brand colour, then a light studio grey (#efeeee), is the house look.

## The film finish (look.post)

Launch films need it: `look: { post: { motionBlur: 0.6, samples: 5, bloom: { strength: 0.7 }, grain: 0.035, vignette: 0.3 } }`.
- **Motion blur**: several moments inside each frame's shutter, only when something
  moves (the camera, an object arriving or leaving, keys), so holds stay fast.
  Fast camera moves and things flying in (`enter: { style: "fly", from, roll }`) are
  what make it feel filmed.
- **Depth of field**: a shot's `dof: { aperture: 0.15-0.3, focus }` (focus: a point,
  an object id or a distance; default the camera's look point; a camera key's
  `focus` moves it). Keep it subtle on screens people must read.
- **Bloom**: only what is brighter than white glows: 3D type with `glow` 2+,
  emissive props. Screens stay under it.
- **Grain and vignette**: a film finish; 0 turns them off.

## A camera that never cuts

`camera.smooth: true` (or a shot's `camera.smooth`) passes the camera through
every key without stopping: one flight through the set, banking with `roll`,
focus following `focus`. Plan one smooth flight per world (the studio opening),
then cut to the next world. Push the camera into a screen at the end of a
world (keys ending close to the screen) and cut to the next shot on the same
picture: a transition through the screen.

## Real product footage on screens

`content: [{ video: "/abs/recording.mp4", mediaStart: 12, from, to }]` plays a
recording on a screen, frame-accurate in the render. Use PandaStudio's own
recordings of the product doing the work (the agent editing, a result
appearing): a working UI beats a screenshot every time. Stills
(`motion.scene-stills`) show the frame of each moment automatically.
`popouts: [{ rect, from, depth, scale }]` lifts parts of the screen (a panel, a
button) toward the viewer. A screen key with `size` reshapes it (a prompt box
growing into the window that shows the result) without squashing the picture.

## Type that lives in the scene

`kind: "text3d"`: crisp type standing in the world, so objects pass in front of
and behind it; `glow: 2.4` blooms it, `animate: rise | type | fade | roll`
(`items` + `every` for a rolling list), keys move it (a word sliding between a
window and its pop-outs). Headlines in `texts` add `animate: mask` (words rise
out of their slot), `scale` (a punch-in from blur), `type`, `roll` with
`items`, `slide: { from, to }` (travels across the frame), and
`layer: "behind"` (under the 3D).

## Moving light

A backdrop shot takes `lights: [{ color, x, y, size, opacity, keys }]`: soft
glows (in % of the frame) that drift over the gradient, so a flat background
breathes.

## Texts and the end card

`texts: [{ text: "Any subject*.*", from: 0, to: 5, position: "top", color: "#fff", size: 112 }]`:
words rise in one by one; `*…*` takes the accent colour (default #ff5a1f, the
full stop is the classic use). `position: bottom` when the screen sits high.
The end card is `overlays: [{ html, from, to, fade }]` over everything: the
brand's real logo (an `<img>` with an absolute path) and CTA. No scripts.

## Checks before the render

- `motion.scene-stills` at every shot's middle and every click: the screen
  readable (at least 55% of the frame wide), headlines clear of the screen,
  nothing important cut by the frame edge, the cursor on the control it clicks.
- Each tick has a `to`; each content item's `from`/`to` matches the shots.
- Images exist (the render lists missing ones in `details.errors`).
- Pacing: a world every 4-5 s, one interaction per world, the card never
  empty.
