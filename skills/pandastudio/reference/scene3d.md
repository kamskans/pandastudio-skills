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

## Camera rigs: moves instead of keys

A shot's `camera.rig` describes the move and the subject; the rig measures
the subject at every frame (so it follows things that move):

```json
{ "id": "hero", "from": 0, "to": 3, "camera": { "rig": { "preset": "hero-orbit", "target": "lap" } } }
```

Presets (distances in units of the target's size): `hero-orbit` (a 45 degree
sweep round it), `slow-push` (a slow push in), `crane-up` (rises over it),
`dolly-left` / `dolly-right` (tracks past it), `reveal` (starts close on
`target`, ends wide on `wide`), `screen-dive` (flies into a screen and ends
where the screen exactly fills the frame: cut there to the full-screen
recording). Or write one: `{ type: orbit | pushIn | dolly | crane | reveal | into,
target, from, to, radius, height, distance, travel, azimuth (0 = from the
front), elevation, ease, fov: [from, to], roll: [from, to] }`. A preset's own
fields can be overridden (`"azimuth": 25`). Save a move you like with
`motion.scene-camera` (`action: "save"`, a name, the rig without its target)
and use it by name in any scene. Keys still work for anything a rig can't do.

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

## Photoreal: path-traced rooms with real models

For the "real room" look (a sunlit desk, a laptop flying in, glowing type in
the air), a shot takes `render: "path"`: real light transport (soft shadows,
light bouncing off the walls, glass you see through) instead of the clay
raster. Work in metres (a desk top at y 0.76).

**1. The room.** One object builds it from its floor plan:

```json
{ "kind": "room", "plan": [[-1.6,-1.2],[2.6,-1.2],[2.6,2.8],[-1.6,2.8]], "height": 2.6, "open": [2],
  "openings": [{ "wall": 1, "at": 1.0, "width": 1.3, "height": 1.5, "sill": 0.85, "mullions": [2, 3] }],
  "colors": { "walls": "#e9b9ae", "ceiling": "#f3e6e0" } }
```

Wall `i` runs from corner `i` to `i + 1`; `at` is metres from its first
corner to the opening's middle. Windows get a frame, bars, glass and a sill;
`kind: "door"` cuts a doorway. Leave the wall the camera looks through out
with `open`. Parts take `materials` / `colors`: walls (plaster), floor
(woodFloor), ceiling, frames, skirting, glass, sills.

**2. The things in it.** Generated (any size, any colour): `shelf` (a
bookcase full of books), `books` (a row, or `flat: true` a stack), `monitor`
(or a screen of kind `monitor` to show the product on it), `keyboard`,
`mouse`, `mug` (with coffee), `notebook`, `pen`, `penCup`, `stickyNotes`,
`rug`, plus boxes for desks, risers and shelves (`material: "oak"`). Real
models from the CC0 library: `motion.scene-library` with `query: "desk lamp"`
returns refs and real sizes; use `{ kind: "model", src: "polyhaven:desk_lamp_arm_01", at, rot }`
(it comes in at its real size; `fit` rescales). Plants, lamps, vases, frames,
cameras and furniture are there; computer gear is generated. A model's own
picture or display shows a scene screen: `screens: { "<material name>": "<screen id>" }`
with a `textureOnly: true` screen.

**3. Surfaces.** `material` names a preset: oak, walnut, tableWood,
woodFloor, parquet, laminate, plaster (painted: takes the object's colour),
concrete, paintedBrick, marble, terrazzo, granite, linen, denim, velvet,
leather, aluminium, steel, chrome, gold, brass, copper, blackMetal, plastic,
glossPlastic, rubber, ceramic, glass, frostedGlass, paperMatte, screenGlass.
Textured ones use a real CC0 texture set at its real size. Your own:
`materials: { "deskTop": { "preset": "oak", "roughness": 0.4 } }` or
`{ "textures": "polyhaven:<texture id>", "tile": 1.2 }` (find ids with
`motion.scene-library` `type: "textures"`).

**Things falling into place.** Any object takes `drop`: it starts above where
it rests and falls, bounces and settles on whatever is in place by then
(real physics, baked when the scene loads, so every frame is the same on
every render):

```json
{ "kind": "notebook", "id": "notebook", "at": [-0.36, 0.76, 0.1], "drop": { "at": 2.2, "from": 0.35, "spin": [25, 40, 10], "bounce": 0.2 } }
```

`from` is the height above its `at` (or an `[x, y, z]` offset), `spin` turns
it at the start (degrees), `push` / `twist` throw it (m/s, degrees/s),
`bounce` and `friction` 0-1. It lands near its `at`, not exactly on it:
check with stills. Stagger drops (a few tenths of a second apart) and don't
drop two things onto the same spot. Whatever it lands on must be in place by
the time the last thing drops.

**Particles.** `{ kind: "particles", type: "dust", at, area: [w, h, d], count }`:
`dust` drifting in the light (by a window), `bokeh` (out-of-focus lights;
with `dof`), `sparks` and `confetti` bursting from `at` at `burst` seconds
(`floor` is where confetti lands), `snow`. Sizes are real (confetti 1.2 cm,
dust 1.5 mm): set `size` larger for a wide shot.

**4. Light like an interior photographer.** `look.environment: { hdri: "polyhaven:art_studio", intensity }`
(about 1.3 in a closed room, 0.5 in an open set), a warm sun
`look.key: 5, keyColor: "#ffd6a8", keyDir: [1, 0.75, 0.55]` (points toward
the sun: put the window on that side and it falls through as a window-shaped
patch on the wall and desk), `look.exposure` 1.4-1.7, `look.contrast` 1.1-1.15.

**5. Dress it densely.** Rooms read real because of clutter: a
monitor on a riser, keyboard and mouse, a notebook and pen, sticky notes, a
pen cup, a mug, books (on a shelf and stacked), plants, a lamp, a framed
print, a rug. An empty desk reads as a render.

- Quality: `look.path: { samples: 16, moments: 4, bounces: 4, denoise: true }`
  are the defaults (the denoiser smooths the light and keeps textures and
  screen text sharp). Raise samples only for glass close-ups.
- 3D type (`text3d`) glows as a light in the room; keep it off bright screens
  (green type over a white UI is unreadable) and clear of props in front.
- Cost: measure it. `motion.render-scene` with `estimate: true` times two real
  frames on this computer and returns `estimatedMinutes` without rendering;
  tell the user before anything over a few minutes. Measured: about 7 s a
  1080p frame on Apple silicon (8 s of video, about 25 minutes), about 45 s
  on Intel laptop graphics under Windows (8 s, about 3 hours: there, render
  at 1280x720 and `look.path.samples` 8, or keep path tracing to a short
  opening). Check framing with `motion.scene-stills` first (they path-trace
  too), render once. Keep path-traced shots to the moments that need them
  (the opening) and raster the rest.
- Holds are cheap: when nothing changes from one frame to the next, the
  render averages them (`look.path.temporal`, on), so a still moment gets
  cleaner and steadier the longer it lasts.
- Library assets download on first use (a few MB each) and are cached; a
  render offline works with what is already cached.
- A path-traced render uses the computer's GPU; it is slow on a machine
  without one.

## Texts and the end card

`texts: [{ text: "Any subject*.*", from: 0, to: 5, position: "top", color: "#fff", size: 112 }]`:
words rise in one by one; `*…*` takes the accent colour (default #ff5a1f, the
full stop is the classic use). `position: bottom` when the screen sits high.
The end card is `overlays: [{ html, from, to, fade }]` over everything: the
brand's real logo (an `<img>` with an absolute path) and CTA. No scripts.

## The live 3D view in the editor

When the user selects a 3D scene graphic on the timeline, Edit graphic shows
the scene running live in draft (raster, fast) at the playhead, the same
checks the stills return for that moment, and a button that renders the
final (path-traced) look of that frame. Point users there to scrub and look;
edits still go through the scene file and motion.render-scene.

## Checks before the render

- `motion.scene-stills` at every shot's middle and every click: the screen
  readable (at least 55% of the frame wide), headlines clear of the screen,
  nothing important cut by the frame edge, the cursor on the control it clicks.
- Fix every entry in the stills' `hints` before rendering. They are measured,
  not guessed: a screen or 3D type hidden behind something nearer (and what
  hides it), screens and type out of shot or cut by the frame edge, props
  floating above or sunk into what they stand on (with the height to fix),
  a key light from behind the camera (flat light), and the picture itself:
  clipped white, crushed black, too dark or washed out, fireflies. Give the
  things that matter an `id` so the hints name them. `critique[].objects`
  has each object's share of the frame (`visible`) and how much of it is
  hidden.
- Each tick has a `to`; each content item's `from`/`to` matches the shots.
- Images exist (the render lists missing ones in `details.errors`).
- Pacing: a world every 4-5 s, one interaction per world, the card never
  empty.
