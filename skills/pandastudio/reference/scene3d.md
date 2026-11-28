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
Characters: `mascot` (round, eyes, cheeks, waving arm, `hat`; `speech` opens
its mouth with a voice) and `character` (a rigged model that acts and talks,
see "Characters that act and talk"). Primitives:
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
`rug`, plus boxes for desks, risers and shelves (`material: "oak"`).
Furniture, generated from parts (metres, standing on y 0, facing +z with its
back toward -z; parts take `materials` / `colors` like the room):
`sofa` (width 2.0, cushions 3, `lived` 0-1 for sat-in cushions; parts fabric,
cushions, legs), `armchair` (0.88 wide), `cushion` (stands up: lean it on a
sofa's back cushions with `rot: [-15, 10, 0]` and `at` y about seat + 0.13),
`throw` (a blanket over an arm: `at` the arm's middle, `height` the arm's top,
0.62 for a default sofa whose left arm's middle is at x = -(width / 2 - 0.08)
from the sofa, with `outside: "-x"` there so the long end hangs down the
outer face), `coffeeTable` (`round: true` for a disc, `shelf`),
`diningTable` (1.8 x 0.9), `chair` (put it at a table's edge facing the
table), `lamp` (floor, or `type: "table"`; the shade glows and a warm light
sits at the bulb, `light: 0` turns it off), `plant` (`type: "leafy"` with
`leaves`, `stems`, `droop`, or `"snake"`), `pictureFrame` (`texture: { image }`
for the picture; raise it with `at` to hang it, back at z 0 against the wall)
and `curtain` (pleated on a rail at `height`, `fullness`, `wind`; set it just
inside the wall beside a window with `rot` so it faces the room). Fabrics
(linen, velvet) take the object's colour.

**Cloth is simulated.** `curtain`, `throw` and `cloth` are real cloth (a
particle grid with stretch, shear and bending, gravity, friction and
collisions with every surface near it), baked when the scene loads (about a
second each; frames read the result, so stills, renders and seeks agree):
- The curtain hangs from its rail and settles against the floor and the
  window; `wind` (0-1) blows it on screen in gusts.
- The throw falls into its folds over the arm's real shape and stays (it
  grips like wool).
- `cloth` is a sheet: flat (`at` above a table, a bed or a sofa: it settles
  over it, a tablecloth or a blanket), hanging (`hang: true` or `pin: "top"` /
  `"corners"` / `[[u, v]]`: a banner, a flag, a sheet on a line), falling on
  screen (`fall: { at, from }`: a dust sheet landing on furniture, a cloth
  revealing or covering something) or blowing (`wind`, metres a second).
- `fabric`: silk (light, slides, billows), cotton, linen, wool, denim (stiff,
  holds). It changes how it folds, not its look: the look is `materials` /
  `colors`.
- Cloth collides with itself (folds stack, a long hem pools on the floor
  in broad folds); `selfCollide: false` loads faster. Cloth doesn't collide
  with other cloth, and things that drop (`drop`) don't land on it. Each
  simulated cloth adds about 1-3 s to loading (more with wind). `sim: false` on a curtain or throw draws
  the folds instead (instant).
Real
models from the CC0 library: `motion.scene-library` with `query: "desk lamp"`
returns refs and real sizes; use `{ kind: "model", src: "polyhaven:desk_lamp_arm_01", at, rot }`
(it comes in at its real size; `fit` rescales). Vases, cameras, sculptures
and particular designs (a specific armchair, a desk lamp) are there; use the
generated furniture when you need a size or colour of your own. A model's own
picture or display shows a scene screen: `screens: { "<material name>": "<screen id>" }`
with a `textureOnly: true` screen.

**3. Surfaces.** `material` names one of 140 presets (`motion.scene-kit`
lists them all with what each is). Families: woods (oak, walnut, pine,
cherry, maple, bamboo, plywood, planks, bark...), stone, brick, tiles and
ground (marble, terrazzo, slateTiles, cobblestone, brick, whiteBrick,
floorTiles, marbleMosaic, asphalt, sand, gravel, snow, mud, forestFloor...),
metals by their real reflectance (gold, silver, copper, brass, bronze,
titanium, iron, nickel, platinum, roseGold, brushedSteel, anodized,
rustyMetal, corrugatedIron...), paints (carPaint with flakes, pearlPaint,
satinPaint, lacquer), glass, gems and liquids (tintedGlass, crystal,
diamond, ruby, sapphire, emerald, ice, water, honey, coffee, milk), soft
bodies (wax, soap, jade, skin, leaf, porcelain, clay), plastics, fabrics and
leathers (linen, velvet, woolBoucle, knit, tweed, suede, blackLeather...),
and special ones (soapFilm, holographic, oilSlick, neon, led, mirror).
Presets "in the object's colour" take `color`.

Your own material in `materials: { "<name>": { preset?, color, roughness,
metalness, clearcoat, transmission, ior, ... } }`, with:
- `attenuationColor` + `attenuationDistance` (light absorbed inside glass,
  liquid, gems: shorter = deeper colour; keep it near the object's size).
- `iridescence` 0-1 (+ `iridescenceIOR`, `iridescenceThickness` [nm, nm]):
  soap bubbles, oil, beetles. `flakes` 0-1: car paint. `brushed` 0-1 +
  `brushDir`: brushed metal. `translucency` 0-1: leaves, lampshades, paper.
  `subsurface` 0-1 + `subsurfaceColor` + `subsurfaceRadius`: wax, jade,
  soap, skin (an approximation: no true scattering).
- Age: `wear` (edges rubbed bright, `wearColor`), `dirt` (crevices and low
  down, `dirtColor`), `dust` (on what faces up), `scratches`, `streaks`
  (running down), each 0-1. Worn things read real; a new prop reads CG.
- `displacement: { pattern, height, scale }` moves the surface for real
  (silhouettes change, shadows fall in the grooves): noise, bricks, tiles,
  waves, hammered, cobbles, ridges, or texture (the Poly Haven set's own
  height map). height in metres (0.002-0.02 for most), scale the size of
  one cell.
- `animate: { type, color, strength, speed, from, to }`: pulse, blink, sweep
  (a band of light crossing), flow (moving energy, lava), edges (glowing
  edges, Tron-style).
- `opacity` 0-1 for see-through without refraction.

`look.key: 5, keyColor: "#ffd6a8", keyDir: [1, 0.75, 0.55]` (points toward
the sun: put the window on that side and it falls through as a window-shaped
patch on the wall and desk), `look.exposure` 1.4-1.7, `look.contrast` 1.1-1.15.

**5. Dress it densely.** Rooms read real because of clutter: a
monitor on a riser, keyboard and mouse, a notebook and pen, sticky notes, a
pen cup, a mug, books (on a shelf and stacked), plants, a lamp, a framed
print, a rug; in a living room a sofa with cushions and a throw, a coffee
table with a mug, a curtain by the window, a lit floor lamp. An empty room
reads as a render.

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

## Lights

`look.key` is the sun; place every other light in `lights`:

```json
"lights": [
  { "type": "point", "at": [-1.6, 1.45, -2.2], "lumens": 800, "kelvin": 2700 },
  { "type": "spot", "at": [1.5, 2.6, -1.6], "target": "shelf", "lumens": 600, "kelvin": 3000, "angle": 30, "penumbra": 0.5, "radius": 0.03, "ies": "downlight" },
  { "type": "area", "at": [0.6, 2.65, 1.6], "target": [0.6, 0, 1.6], "size": [0.6, 0.6], "lumens": 1500, "kelvin": 3500, "visible": true }
]
```

- Types: point (a bare bulb; hard shadows), spot (a cone: `angle` is half
  of it, `penumbra` its soft edge, `radius` the bulb for soft shadows, `ies`
  its beam: downlight, narrow, wallwash, rings, flood, spot or a real .ies
  file), area (a panel or softbox `size [w, h]`, `circular` for a disc; the
  softest light), sun (`dir`).
- Colour: `kelvin` (candle 1900, bulb 2700, halogen 3200, daylight 5600,
  overcast 6500) or `color`. Strength: `lumens` (a 60 W bulb is 800, a
  ceiling panel 2000-4000), `watts` (incandescent equivalent) or `intensity`.
- `target` aims a spot or panel at a point or an object id. `visible: true`
  draws the panel or bulb. `flicker` 0-1 (candles, fire). `keys`
  `[{ t, intensity (a multiplier), kelvin, at, target }]` dims, warms or
  moves it; `show { from, to }` switches it.
- `highlight: { on: "<id>" or [x,y,z], distance }` puts the light where its
  reflection lands on that spot as the camera sees it: the glint on a phone.
- `studio: { preset, target }` lights a subject in one line: three-point,
  portrait, product (top softbox + two strips: clean product reflections),
  softbox, rim, night (warm downlight + cool fill).
- A night interior: `look.key: 0`, `look.environment.intensity` about 0.03,
  then lamps; the furniture `lamp` has its own bulb light (`light: 0` turns
  it off when you place your own). In the fast look shadows come from the
  first four lights; path traced, every light shadows and bounces.

## Motion beyond keys

Every object can also take:
- `path: { points: [[x,y,z], ...], from, to, ease, loop, orient (true: faces
  along it), bank, closed }`: it travels a smooth curve (a paper plane, a
  drone, a car round a bend).
- `parent: "<id>"`: it rides on that object (its `at` becomes relative to
  it); `parentKeep: true` keeps where it stands when it attaches. A phone on
  a tray that slides in, a ball on a moving cart.
- `expr: { "at.y": "0.25+0.15*abs(sin(t*4))", "rot.y": "t*90", scale:
  "1+0.1*a" }`: a formula per channel, every frame. Names: `t` the object's
  clock, `T` scene time, `v` the channel's own value (from keys), `a` its
  audio level, `i` its index, `d` the duration, `pi`. Functions: sin, cos,
  tan, abs, min, max, floor, ceil, round, sqrt, pow, exp, log, sign, fract,
  clamp, mix, step, smoothstep, noise, saw, tri, square, ease. Checked
  before rendering.
- `react: [{ audio: "/abs/music.wav", band: all|bass|mid|treble, to:
  scale|at.x|at.y|at.z|rot.x|rot.y|rot.z|glow, amount, at }]`: moves with the
  music (a speaker pumping on the kick, a logo glowing on the beat);
  `glowColor` for glow. Lights take `react` too (their strength).
- `clock: { speed, offset, from, loop: [a, b], hold: [a, b] }`: the
  object's own time for its keys, path, spin, float and expressions (a loop
  that repeats, a freeze, a slow-motion beat).

## Modelling your own shapes (`mesh`)

When no kit piece or library model fits, model it: `{ "kind": "mesh",
"material": "ceramic", "ops": [ ... ] }`. Each op makes a solid or changes
the shape so far; solids take their own `at` / `rot` / `scale`; the result
sits on y 0 (`base: false` keeps it where the ops put it).

```json
{ "kind": "mesh", "material": "ceramic", "color": "#e9e2d6", "ops": [
  { "op": "cylinder", "radius": 0.12, "height": 0.26 },
  { "op": "subtract", "with": [{ "op": "cylinder", "radius": 0.105, "height": 0.26, "at": [0, 0.02, 0] }] },
  { "op": "union", "with": [{ "op": "torus", "radius": 0.07, "tube": 0.018, "at": [0.13, 0, 0], "rot": [0, 0, -90], "arc": 180 }] } ],
  "decals": [{ "image": "/abs/logo.png", "at": [0, 0.15, 0.12], "from": [0, 0, 1], "size": 0.12 }] }
```

- Solids: box (`round` bevels every edge), sphere, icosphere, cylinder,
  cone, torus (`arc`), capsule, plane, disc, ring, `extrude` (a 2D shape:
  star, polygon, heart, gear, rect with `round`, circle, or your own
  `[[x, y]...]`, with `holes`, `depth`, `bevel`), `lathe` (a profile
  `[[radius, y]...]` turned round y: vases, bottles, glasses, lamps; smoothed
  into a curve), `sweep` (a tube or profile along a path: cables, handles,
  rails).
- Booleans: `union`, `subtract`, `intersect` `{ with: [ops] }`.
- Copies: `array` (`count`, `offset`, `rotate`, `scale` each step: stairs,
  fence posts, a spiral), `radial` (round an axis: petals, spokes),
  `mirror`, `scatter` (`count` copies of `of` over the surface: rocks, rivets,
  berries).
- Shape: `subdivide` (`levels`, smoothed), `smooth`, `twist`, `bend`,
  `taper`, `noise` (organic wobble), `inflate`, `solidify` (thickness for a
  sheet), `transform`. `smoothShading: true` for organic shapes.
- `decals` on any object print a picture onto its surface: `at` in the
  object's own frame, `from` the direction it projects from, `size`, `rot`.
  Labels, stickers, logos, stains.

## Air, smoke, fire, liquids and sand

- `look.haze: { density 0.1-0.5, color, shafts true }` puts light in the air:
  the sun through each window becomes a visible shaft, every spot light a
  beam, and the room softens with distance. Path traced, haze defaults to 96
  samples a pixel (it is slower: estimate first). Darker scenes show it best
  (`look.exposure` 0.7-0.9, low ambient).
- `smoke` at a point: puffs rising, spreading and drifting with `wind`
  (`rate`, `life`, `rise`, `spread`, `size`, `color`, `opacity`, `start`,
  `stop`). Steam from a mug: size 0.04, rise 0.12, spread 0.06, colour
  #ffffff, opacity 0.25.
- `fire`: flame tongues, a hot core and a flickering warm light with moving
  shadows. A candle: size 0.03, intensity 2 (on a `cylinder` of `wax`). A
  fireplace: size 0.4.
- `pour`: a stream from `from` (relative to its `at`) with `velocity`, from
  `start` to `stop`, thinning as it falls; it spreads a puddle, fills a
  container (`into: { radius, height, base }`) or, with `type: "sand"`,
  falls as grains and builds a pile at the angle of repose. `material`:
  water, honey, milk, coffee, oliveOil, sand. Put a glass (`cylinder` in
  `glass`) where it lands to pour a drink.
- `water`: a pool, lake or sea surface with moving waves (`size`, `waves`
  height, `wavelength`, `speed`).

## Characters that act and talk

A `character` is a rigged .glb (a skeleton with animation clips): the
built-in `"robot"` (a cartoon robot, 1.2 m) or the user's own model
(absolute path; Mixamo, Ready Player Me and VRoid exports work). Read what
it can do first: `motion.scene-character { src }` returns its clips (name,
seconds), shape keys and how it talks. Names are checked when the scene
renders, with the closest match suggested.

```json
{ "kind": "character", "id": "bot", "src": "robot", "at": [0, 0, 0],
  "clips": [ { "clip": "Idle", "at": 0 }, { "clip": "Wave", "at": 0.3, "loop": false },
             { "clip": "Idle", "at": 2 }, { "clip": "ThumbsUp", "at": 5.5, "loop": false } ],
  "look": "camera",
  "speech": { "audio": "/abs/voice.wav", "at": 1.0 },
  "expressions": [ { "name": "Surprised", "at": 4.6, "until": 5.4, "amount": 0.8 } ] }
```

- `clips`: one plays at a time from its `at` until the next one (or `until`),
  crossfading over `fade` (0.3 s). `loop: false` holds the last pose (a wave,
  a thumbs up); Idle between actions keeps it alive. Before the first clip:
  its first frame. Clips play in place: move the character with `keys`
  (Walking while `keys` carry it across), turn it with `rot`. It faces +z.
- `look`: `"camera"` or a point turns the head toward it (up to 70 degrees
  from where the body faces), `{ at, from, to, amount }` for part of the scene.
- `speech`: the mouth follows the voice (measured from the audio, 60 times a
  second): a mouth shape key (jawOpen, mouthOpen, viseme_aa) or a jaw bone;
  the robot has neither, so it nods with the beats of the voice. `at` is when
  the voice starts in the scene. The scene's video has no sound of its own:
  pass the same audio to `motion.render-scene`'s `audioPath` (with silence
  before `at`), or place the voice on the timeline at the graphic's start + `at`.
  Narrate with `media.generate-narration` and use its file.
- `expressions`: the model's shape keys over time (the robot: Angry,
  Surprised, Sad).
- Path-traced shots render posed characters too (soft shadows, motion blur
  on fast arms). A character with a clip playing is always "moving", so its
  frames don't average between frames; use enough `look.path.samples`.
- Credit for the robot: RobotExpressive by Tomás Laulhé (Quaternius),
  modified by Don McCurdy, CC0.

## Camera moves from a reference, and Blender files

- `motion.scene-camera-from-video { video, from, to, mode, name }` measures
  a shot you like (its pan, tilt, roll and push or zoom) and returns a rig
  `{ type: "track", track, mode }`. Use it as a shot's `camera.rig` with
  `target`, `distance`, `azimuth`, `elevation` (and `strength` to exaggerate
  or calm it). It follows how the picture moves: a locked-off shot of people
  walking gives little.
- A `model` or `character` `src` can be a `.blend`: the user's own Blender
  (free, blender.org) converts it to glTF once. Without Blender the scene
  says so; ask the user to install it or to export a .glb.

## Render extras

- Denoising: path-traced shots use the AI denoiser (Intel Open Image Denoise
  on the GPU, about 0.4-1.7 s a 1080p frame on Apple silicon) where WebGPU
  runs, and the classic one elsewhere. `look.path.denoise: "classic"` or
  `false` (then raise samples) to choose.
- `look.path.caustics: true`: light focused through glass and water and off
  mirrors onto surfaces (no firefly clamp, 8 bounces, 128 samples: slow).
  Caustics come only from area lights, the sun and the environment, never
  from point or spot lights (they are points, nothing can be traced back to
  them through glass): light a glass product with a small `area` light.
- Solid glass, gems and liquids path trace clear (a patched path tracer: the
  bend where light leaves the glass is approximate).
- `motion.render-scene` `passes`: extra videos for compositing, `depth`,
  `normal`, `albedo`, `id`, `matte:<id>` (needs the object's id),
  `light:<group>` (`sun`, `env` or a `lights[].group`).
- `look.lightMix: { warm: 1.4, sun: 0.6, env: 0.5 }` rebalances light groups
  without moving a light.
- Light linking (path traced): `lights[].only: [ids]` lights just those
  objects, `exclude: [ids]` everything else; up to 20 linked lights. The fast
  look ignores linking.

## Render farm (the user's own GPUs)

- Setup: the user pastes their Runpod key in Settings → Connectors → Runpod
  (checked with Runpod, stored encrypted; the row shows the balance). Never
  ask for a key in chat: send them there. `motion.render-farm` says whether it
  is set up (`configured`, `setup` steps when not) and `check: true` returns
  the balance; its `apiKey` arg is for scripts and CLI setups only.
- `motion.render-scene { ..., farm: { gpus: 4, gpu: "NVIDIA GeForce RTX 4090" } }`
  returns a quote and rents nothing: the GPU (the asked one if Runpod has one
  free, else the next in-stock render GPU, said in `substitute`), `gpuId`,
  price an hour, estimated minutes and cost, and the account balance (with a
  `warning` when it may not cover the render). Tell the user the price; only
  with their yes call again with the same farm plus
  `gpu: quote.gpuId, maxPricePerHour: quote.pricePerHour, confirm: true`.
  If that GPU sells out meanwhile, pods start on another in-stock render GPU
  priced at or under `maxPricePerHour`, never above. The render splits across
  the GPUs, the pieces are joined, and the pods are always deleted. The price
  is charged to the user's Runpod account; `cloud: "COMMUNITY"` is cheaper
  and less reliable than the default `SECURE`.
- The key comes from the Runpod console: Settings > API Keys (Read/Write), with
  credit added under Billing.
- `farm: { endpoint, token }` renders on a worker the user runs themselves
  (the scripts/render-farm image on any NVIDIA machine): no quote needed.

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
