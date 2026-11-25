# Native motion elements

Editable, word-timed graphics the render engine draws itself: keyword
headlines, chips, stamps, counters, step lists, slams, words behind the
speaker, lower thirds, highlights, end cards, progress bars and icon pops.
The preview and the export run the same code, so what you see is what
exports, and nothing is rendered to a video file: change the text, the
colour or the timing and it updates instantly. They are the default for
keyword graphics on talking heads and Shorts. Reach for an HTML composition
only for bespoke work (see "When to use HTML instead").

Needs the app release after 2.0.2 and CLI / MCP 1.186.0+. Stored at
`editor.motionElements`.

## One call: `project.style-edit`

For a talking-head video or a Short, start here. One call styles the edit in
a house style:

```bash
pandastudio project.style-edit --id=<id> --dryRun=true          # read the plan first
pandastudio project.style-edit --id=<id> --applyFixes=true      # then run it
```

In the editor the same pass is **Graphics → Style this edit** (style picker, footage-fix and
music toggles). Moments with no English words come back in that card as "needs a keyword"
fields: when a user asks how to do it themselves, point them there.

It runs `project.inspect-footage` (with `applyFixes=true` it applies the
flat-footage grade / chroma key it suggests; when it keys out a green or
blue backdrop and the project has no background it also sets one, `--backdrop`
or the style's studio gradient, so the speaker never lands on flat black), `project.speech-map` (density
by `format`), places elements and zooms on the stressed moments, then
`project.compose-soundtrack` in the matching score style. Run it AFTER the
cuts (fillers, silences, retakes), because it anchors to words that must
already be in the edit.

| style | for | family | score |
|---|---|---|---|
| `bold-short` (default for 9:16) | Shorts / Reels / TikTok | bold | `short` |
| `editorial-long` (default for 16:9) | YouTube long-form talking heads | editorial | `talking-head` |
| `clean-tutorial` | tutorials, walkthroughs, courses | clean | `calm` |
| `paper-cut` | the Vox paper-cut look (Shorts or long-form) | paper | `calm` |

`paper-cut` also sets the LOOK (skip with `--look=false`): the bundled
newsprint backdrop `/wallpapers/paper-cream.jpg` (or `--backdrop`), the
speaker cut out with a white keyline for the whole edit (a `remove`
background effect with `outline`), and in a vertical frame the cut-out
bottom-aligned in the lower half so the strips own the top band. No zooms
(a punch-in would blow up the paper and keyline). Headlines go in the `top`
band, chips just under it (`{x:50,y:34}`). Its score gets paper foley
instead of synth whooshes: a page flick on every strip as it lands, a
marker squeak on each highlighter, a page turn on the end card.

Rules (first match wins, per moment):

| moment | bold-short | editorial-long | clean-tutorial |
|---|---|---|---|
| the first moment | keyword (hook) | keyword (hook) | keyword (hook) |
| a topic turn (pause + strength 2-3, 45 s apart, long format) | none | lowerThird | lowerThird |
| a number / range / currency | count | count | count |
| strength 3, an English statement inside non-English speech | slam | slam | keyword |
| other strength 3 | stamp, keyword (alternating) | keyword | keyword |
| code-switch key term (an English word in Tamil/Hindi...) | chip | chip | chip |
| other strength 2 | keyword | zoom only | zoom only |
| last 4 s | endCard (hook + hero line) | endCard | endCard |

Zooms on every strength 2-3 moment: 1.5x and 1.25x alternating, 1.8x (depth
3, the closest to 2x) on the single hero moment, placed silent (the score
carries the sound). Graphics never overlap each other or anything you placed.

Text: the engine draws every script, symbols (≤ → ✓ ★) and colour emoji, but
style-edit's automatic headlines are the English words the speaker used,
condensed to a headline (trimmed of "the / is / to..."): headline layouts are
tuned for Latin words. A moment with no Latin words gets a zoom only. For
non-Latin speech, pass keywords per word (any script works when you add
elements yourself):

```bash
pandastudio project.style-edit --id=<id> --texts='{"clip-1:w12":"One big thing","clip-1:w40":"Trust takes years"}' \
  --endCard='["Depth beats reach","Follow for part 2"]'
```

`--accent` / `--highlight` colour every element (default: the brand kit).
Word ids come from the dryRun plan (`plan.skipped` lists moments that got no
graphic, with the reason) or from `project.speech-map` (`moments[].wordId`).

Re-running replaces only what the previous style-edit placed (elements and
zooms carry `origin: "style-edit"`). An element you edit (in the editor or with
`project.update-motion-element`) becomes yours and a re-run leaves it alone.

After it runs: `project.render-sheet --fromMs --toMs --count=8` across the
video and look. Adjust single elements with `project.update-motion-element`.

## The verbs

```bash
pandastudio project.add-motion-element --id=<id> --type=keyword --content="Ship it today" --wordId=clip-1:w42
pandastudio project.add-motion-element --id=<id> --type=count --content='{"value":"20-30","label":"minutes"}' --atMs=12400
pandastudio project.add-motion-elements --id=<id> --elements=@elements.json    # all or nothing
pandastudio project.update-motion-element --id=<id> --elementId=el-3 --style='{"accent":"#ff5a36"}'
pandastudio project.remove-motion-element --id=<id> --elementId=el-3
pandastudio project.list-motion-elements --id=<id> --catalog=true
```

Timing: `wordId` (a transcript word id; the element starts as the word is said
and follows it through later cuts) or `atMs` (edited ms), plus `offsetMs`
(may be negative). Length: `durationMs`, else `endWordId` (runs to the end of
that word), else the type's default. A word cut from the edit removes its
element, like zooms. `anchor: "free"` pins an element to the timeline instead.

Update: pass only what changes. `style` merges (a field set to `null` goes
back to the brand default); `zone` / `layer` / `sound` / `zIndex` `null` reset.
`atMs` / `wordId` move it and keep its length. `project.remove-region
--regionType=motion-element` and `project.duplicate-region
--regionType=motion-element` work too.

Errors name the field and what it accepts, e.g. `element 1: endCard content is
{ lines: [string, string] }: lines must be exactly two strings`.

## The 29 types

| type | content | default length | sound (auto) | use it for |
|---|---|---|---|---|
| `keyword` | `{ text }` (or a bare string) | 2.2 s | whoosh | the headline of a beat: 1-4 words rise from a mask in turn, the last in the emphasis colour |
| `chip` | `{ text }` | 1.8 s | pop | a small label: a term, a tool name, a tag |
| `stamp` | `{ text }` | 1.8 s | hit | a verdict: "FREE", "NO", "DONE"; slams in rotated, shakes |
| `count` | `{ value, to?, prefix?, suffix?, label?, decimals? }`; `value: "20-30"` is a range | 2.5 s | tick roll | a number that matters: rolls up to it, label under it |
| `steps` | `{ items[] (1-8), cues?, images? }` or `cueWordIds` | 5 s | pop per item | a list said out loud: each item on its word, latest highlighted. Without cues every item, the last one too, gets the same time on screen before the exit; end the element at least 1.5 s after the last cue word (a later cue is pulled in so the item still holds). With `images` (one absolute path or null per item) it becomes a card carousel: each image lands as a tilted card on its cue, earlier cards slide aside and dim, the item's label under the card (paper family: polaroids with tape). Give it a roomy zone (`center` or `{x,y,w}`) |
| `slam` | `{ text }` | 1.5 s | hit (hero) | THE line: huge type lands with a camera kick. Once or twice per video |
| `behind` | `{ text }` | 3 s | none | a big outline/solid word behind the speaker (layer behind) |
| `lowerThird` | `{ title, subtitle?, look?, typeSpeed? }`; `look`: `plate` (default) \| `plain` \| `typewriter`; `typeSpeed` = the typewriter's pace in characters per second (5-80, default about 22) | 4 s | whoosh (soft) | name + title, a chapter, a source. `plate` = card + accent bar wiping in; `plain` = the text alone; `typewriter` = typed letter by letter in JetBrains Mono (unless `style.font`) with a blinking cursor; without `typeSpeed` typing fits in the first 55% of the element, with it the pace is kept up to the exit (a slow pace on a short element finishes just before it fades). The subtitle can be a sentence or two (up to 320 characters, wraps to 3 lines). With `style.size` set the title keeps that size on every card and wraps onto a second line when long (shrinking only past two lines), so a set of cards matches; without it the title shrinks to fit one line |
| `highlight` | `{ text }` | 2.5 s | none | the phrase with a marker drawn under it as it is said |
| `endCard` | `{ lines: [a, b] }` | 5 s | bell | the two-line takeaway at the end, with a timer bar |
| `progress` | `{ label?, fromMs?, toMs? }` (edited ms; default its own span) | 10 s | none | a chapter / countdown bar |
| `iconPop` | `{ emoji }` or `{ icon }` | 1.5 s | pop | an icon burst: check, cross, plus, arrow-up, arrow-down, clock, star, heart, fire, bolt, bulb, warning, dollar, percent, question, exclamation (emoji map onto these). `look: "prop"` draws it as a hero object instead: large (a custom zone's `w` sets its width, default 60% of a vertical frame), no disc or rays, complete on its first frame with a soft floor shadow and a slow push-in, no exit (end it on a hard cut); any colour emoji makes a drawn prop (✉️ 🧊 🔒 🚪 🗑️ 🖨️), bundled icons draw in the accent |
| `frame` | `{ look: viewfinder \| selection \| corners \| pointer, label? }` | 4 s | none | a frame around the speaker: `viewfinder` = camera-viewfinder overlay on the whole frame (corner brackets, blinking REC or `label`, running timecode, exposure readouts, battery); `selection` = a design-tool selection box around the speaker (handles, `label` name tag, W × H size pill; accent defaults to design-tool blue); `corners` = accent corner brackets around the speaker; `pointer` = a small box on content (a phrase on a screen capture: thin accent outline, faint tint, corner handles, drawn on left to right), placed with a `{ x, y, w, h }` zone (`h` sets the box height) |
| `sticker` | `{ image: transparent PNG path or "@speaker", label?, twin? }` | 2.5 s | pop | `image: "@speaker"` = the speaker cut out of her own live shot (head and shoulders, needs the person matte); `twin: true` = a mirrored pair in the two lower corners (twin stickers). A cut-out image as a die-cut sticker: a white border that follows the image's own outline, a soft shadow, a tilted pop and a gentle float; `label` adds a strip of black label tape under it. Make the image with `media.generate-image --transparent=true` (Replicate or Higgsfield connector), or use the user's own PNG; without a connector ask the user for images |
| `ticker` | `{ text, label?, speed? }`: one story per line (up to 2000 characters), `label` = the box on the left ("BREAKING", "LIVE"), `speed` px/s at 1080p (40-600, default 160) | 12 s | none | a news ticker: a band at the bottom (or `zone: "top"`) with the stories scrolling across in a loop. Band height follows `style.size`; `zone: { x, y, w }` sets its length and position |
| `credits` | `{ title?, entries }`: `entries` = `[{ role?, name }]` or strings `"Role: Name"` (up to 200) | 15 s | none | rolling credits: the title, then each role (small, the secondary line) above its name, rolling up through the frame over the element's length (lengthen it for a slower roll) |
| `crawl` | `{ title?, text, tilt?, turn? }`: paragraphs up to 6000 characters, `tilt` 0-80 degrees (lean back, default 60; 0 = a flat roll), `turn` -60..60 degrees (twist left / right) | 20 s | none | an opening crawl: the text scrolls up and away on a plane in true perspective, shrinking toward the horizon and fading in the distance. Colour = `style.text`, else the highlight colour. Over footage with captions, `project.hide-captions` over its span (same for credits) |

### Creator effects (PandaStudio 2.1.47 and later, CLI / MCP 1.264.0+)

Short-form effects built around the speaker. Every picture slot (`images`,
`image`, a list row's `image`) takes an image path or `"@speaker"`: the
speaker's own shot, live, as the viewer sees it framed (inside any baked-in
letterbox bars). Picture types with no `images` show `"@speaker"` in every
slot; fewer images than slots repeat in turn.

| type | content | default length | layer | use it for |
|---|---|---|---|---|
| `reelWall` | `{ images?, count? (4-24, default 12) }` | 3 s | behind | reel wall: the wall behind her turns into a grid of tiles of her own shot, popping in from the centre out; she stands in front |
| `cardFan` | `{ images?, count? (3-9, default 5) }` | 3 s | behind | card reveal: photo cards rise from behind her shoulders and fan out in an arc over her head |
| `orbit` | `{ images?, count? (2-6, default 3) }` | 4 s | around | cards circling her on a tilted ring, behind her on the far side, in front on the near side |
| `polaroids` | `{ images?, count? (2-8, default 5), captions? }` | 3.5 s | around | polaroid pile: snapshots drop in one by one, scattered around her, every other one tucked behind; `captions` = handwritten words on the bottom border |
| `wordWall` | `{ text, rows? (3-14), background? }` | 3.5 s | behind | word wall: one word or phrase repeated in rows ("• word wall •") scrolling in alternate directions on a dark wall behind her. A persistent background, not a transition |
| `neon` | `{ text, flicker? (default true) }` | 3 s | front | neon sign: thin glowing tube letters (colour `style.accent`, default hot pink) that stutter on and hum |
| `bloom` | `{ text }` | 2.5 s | front | colour bloom: italic serif letters blooming in one after another, each in a soft rainbow colour, out of a glow (`style.accent` + `highlight` replace the rainbow) |
| `scrapbook` | `{ text }` | 2.5 s | front | scrapbook text: ransom-note letters, each on its own scrap of paper (colour, typeface, tilt, ragged edges), popping on in turn |
| `keycaps` | `{ text (up to 24 letters) }` | 3.5 s | front | keys falling: a keycap per letter rains down, tumbles, bounces and settles into a row along the bottom that spells the word |
| `productCard` | `{ title, subtitle?, price?, image? (default "@speaker"), button? ("Add to cart"), done? ("Added") }` | 4 s | front | add to cart: a shop card slides up below the face; its button is tapped at 45% (press, ripple) and flips to "Added ✓". Only for a product or offer the speaker names |
| `scrollList` | `{ title?, items: (string \| { title, subtitle?, image? })[] (up to 30), look?: playlist \| feed }` | 4 s | front | scroll overlay: a phone app list on a frosted dark panel down the side away from her face, scrolling steadily up (square artwork for a playlist, round avatars for a feed) |
| `splitScreen` | `{ panels?: 2 \| 3, images?, zooms? (1-4 each, default 1, 1.35, 1.75) }` | 3 s | front | three scenes: the frame splits into stacked panels (side by side on 16:9), each her shot at its own zoom (wide, medium, close) or a given picture; panels slide in from alternating sides |
| `field` | `{ pattern: grid \| dots \| wave \| floor, color?, glow?, ink?, inkOpacity?, spacing?, size?, driftX?, driftY?, major?, rulers?, waveY?, waveAmp?, wavePeriodS?, vignette?, image? }` | 6 s | front | moving background drawn by the engine (preview = export): `grid` a drafting sheet drifting by (driftX, driftY) px/s (-15 = up; every `major`th line heavier, `rulers` adds ruler ticks and construction circles), `dots` a drifting dot grid, `wave` a still dot lattice whose dots exist only below a travelling wave line (faint just under it, full again deeper), `floor` a perspective dot floor running toward you (`image` underneath: a lit ground). Box = its `{ x, y, w, h }` zone, else the whole frame; phase follows the edit's clock so it never jumps at a cut; no fade (it cuts with its scene). `layer: "background"` puts it under the speaker over the canvas (a ground she stands on wherever she is placed). Draws under everything on its pass; put `zIndex: -3500` to sit under pictures too |
| `link` | `{ nodes: [{ label?, icon?, image?, x?, y?, atMs? }] (1-8), lines?: [{ from, to, atMs?, style?: solid \| dotted \| dashed, curve? }], sends?: [{ line, atMs, back? }], flow?, layout?: row \| column \| chain \| hub \| free, drawMs? }` | 4 s | front | things and the lines between them: tiles (picture, icon or initials) with names, each line drawing from its first node to its second in 0.4 s with an accent dot on the tip, sends running a dot along a line and lighting the node it reaches, `flow` keeping dots travelling. `chain` = a process (each line draws as the next step lands), `hub` = many sources in a row feeding node 0 below. x / y are fractions of its box |
| `cursor` | `{ look?: arrow \| hand \| dot, moves: [{ x, y, atMs, action?: move \| click \| drag \| hover }], from?, carry?: { w, h, image?, label? }, on? }` | 3 s | front | a pointer acting on what is on screen: glides to each point on its time (ms from the start), presses with an accent ripple on `click`, carries `carry` (a selection-framed box) from the previous point on `drag`, settles on `hover`. `dot` = a translucent tap dot. With `on` (a picture overlay id or a sticker element id) the points are fractions of that picture and follow it; else of the frame |
| `mark` | `{ look: highlight \| outline \| box \| underline \| strike \| circle, box: { x, y, w, h }, on?, drawMs? }` | 2 s | front | a mark on part of a picture: a marker sweep, a rounded outline, a selection box with handles growing left to right, an underline, a strike, a hand-drawn circle. With `on` the box is in fractions of that picture (as in the file, top-left origin) and follows its crop, position, enter animation, tilt and fade; it hides while the picture is off screen |
| `ribbon` | `{ form?: spiral \| sweep \| halo \| arc \| beam \| none, veil? (0-0.9), spheres? }` | 4 s | behind | glass ribbon: a luminous 3D glass band (translucent colour `style.accent`, bright rims `style.highlight`) drawn on with a glowing leading edge, then turning slowly: `spiral` coils around her (a hook), `sweep` is one diagonal beam, `halo` a tilted ring round the head, `arc` a thin arch overhead, `beam` a hairline arc of light (put it `front` over a full-frame field for an empty stage), `none` no band (the room grade and spheres alone, for speaker scenes between ribbon moments). `veil` darkens the room behind her toward a deep tint with a warm glow behind the head (her cut-out stays bright: it needs the person matte); `spheres` floats lit glass spheres around her. It draws under the other elements on its layer, so cards and words behind her sit on top of it. Spiral and sweep fade out below the chest, clear of hands and mics |
| `prop` | `{ name \| svg \| src, size?: s \| m \| l, tone?: light \| dark \| field, flat?, push?, beats?: [{ at, do, part, ... }] }` | 3 s | front | a drawn hero object whose named parts act the sentence out (the Prop Room pack's props): one of 22 bundled props (`envelope`, `printer`, `tank`, `cloche`, `cassette`, `train`, `gate`, `clock`, `iceberg`, `door`, `bulb`, `battery`, `folder`, `funnel`, `magnet`, `trophy`, `rocket`, `laptop`, `phone`, `padlock`, `shredder`, `gauge`; parts and the beats that suit each in electron/assets/props/README.md) or your own SVG drawn to electron/assets/props/STANDARD.md (`src`: an .svg path, read into the project; it is checked: viewBox from 0 0, parts as `<g id>` with no transform). On screen whole on its first frame (arrives from 0.85, then a slow push-in on 2 of 3), a soft warm shadow (`tone` dark: black, field: faint), centred in its box at 0.68 of the box width. Beats at ms from the start, about 12 frames each, no overshoot: `show` (how pop \| slide \| grow \| land \| fade), `hide` (fade \| pop \| away), `move` (dx, dy, rotate, scale, sx, sy about the part's data-pivot; ease out \| inOut \| slow), `fill` (to 0-1: a level rises or drains), `draw` (strokes draw on; ease sweep for a wedge), `spin` (until a `stop: true` spin), `swap` (to another part), `recolor` (color), `pulse` (a ring), `badge` (kind check \| cross \| text, text, big, on a part's corner). In a style pack the beats land on words |

### Data, comparison and device graphics

Every list here lands item by item on the words that say it: give
`cueWordIds` (one transcript word id per item, spoken order; up to one per
item, the rest are paced) or `content.cues` (ms offsets from the start).
`panel` (chart, table, compare, diagram) sets the surface: `none` (straight
on the frame), `solid` (a graphite card in `style.surface`) or `glass`
(frosted over the picture). Text and numbers come from the speech or the
user's own pictures; never invent a figure, a result row or a screenshot.

| type | content | default length | use it for |
|---|---|---|---|
| `chart` | `{ kind?: line \| bars \| tiers \| meter, title?, points: ({ label?, value, display? } \| number)[] (2-12), prefix?, suffix?, max?, limit?, highlight?, xLabel?, yLabel?, panel? }` | 4.5 s | numbers that move: `line` draws a smooth curve point by point (soft area, dots popping, the lit point glowing); `bars` grow from the axis, the lit one in the accent, values counting up; `tiers` are benchmark bars of falling width (white, grey, accent) with a small label over each and `display` typed in ("Above 1.5%"); `meter` is value bars on a card filling to value / max as the number counts, red at `limit` |
| `table` | `{ look?: grid \| prices \| ranks, title?, columns?, rows: ({ cells, mine? } \| string[] \| "a \| b")[] (1-8), total?, climb?, panel? }` | 5 s | `grid`: a card with its title on an accent bar, rules, heavy row headers and each row's cells typed in on its word; `prices`: one strip per row, the price right, `"~$49~ $19"` strikes the old price with a red marker, `total` lands last in an accent tile; `ranks`: numbered rows, the `mine` row lit and, with `climb`, rising to the top on the last cue while the others step down. `~struck~` works in any cell |
| `compare` | `{ look?: panels \| faceoff \| wipe \| swap, title?, left, right (each { label?, text?, image?, icon?, value?, color? }), winner?, flow?, token?: people \| dots \| coins, full?, panel? }`, cues `[left, right, verdict]` | 4.5 s | two sides of a claim: `panels` slide in from either side under label tabs (`value` fills a bar; on glass they tilt toward each other); `faceoff` flies two discs in from beyond the edges to meet with a "vs", then on the verdict a stream of people (dots, coins) pours from the `flow` side into the other, which swells, or the `winner` grows with a ring and a check; `wipe` fills the card with the before side and a handle wipes the after side over it to the middle (`full`: all the way on the verdict; else the before side greys with a cross); `swap` strikes the old pill and raises the new one in the accent |
| `device` | `{ look?: phone \| laptop \| browser \| window \| code \| terminal \| player, image?, title?, lines?, tracks?, scroll?, spot?: { x, y, r? }, zoom? }` | 4.5 s | a capture inside the thing it came from: a phone (metal rim, island), a laptop, a browser (`title` = the url) or a window; a tall capture `scroll`s; `spot` dims the screen around a circle from its cue and `zoom` pushes the camera into it. `code` types `lines` in an editor (line numbers, colouring, caret), `terminal` types `"$ "` commands and shows the output lines; `player` plays the picture with a progress bar, or with no picture shows waveform `tracks` lighting as the playhead crosses |
| `document` | `{ look?: doc \| pages, title?, sections?: ({ label, lines?, text? } \| string)[] (1-5), pages?, images?, glow? }` | 4.5 s | `doc`: a script card (italic serif heading, sections on a dotted connector, placeholder lines); each section lights in the accent on its cue and its `text` types in; `pages`: a grid of pages (the user's page pictures, else plain placeholder pages) rising in, on a frosted panel with `glow` |
| `diagram` | `{ look?: hub \| satellites \| flow \| link \| route, title?, center?, nodes: ({ label, icon?, image? } \| string)[] (1-8; link 2), panel? }` | 5 s | how things connect: `hub` tiles round a hub disc with connectors drawing out; `satellites` pills slowly orbiting a glowing hub (far ones behind it); `flow` a chain where every step waits greyed and lights on its word, packets running along drawn links; `link` two tiles, a line drawing between them and a check; `route` stops along a dashed route over a map grid with a token travelling to each |
| `card` | `{ look?: glass \| solid \| light, title, accent?, text?, icon?, rows?: ({ text, icon? } \| string)[] (up to 5), glint? }` | 4.5 s | an information card: a title with its `accent` words in heavy italic, an icon tile popping on the corner, rows sliding in and checking off (the latest lit in the accent); the card grows to fit each row as it lands; `glint` runs a spark round the glass rim |
| `assistant` | `{ look?: answer \| search \| chat, app?, prompt?, rows?: ({ text?, sub?, mark?: check \| strike \| lit \| ad } \| string)[] (up to 8), messages?: ({ text, from?: me \| them } \| string)[], light? }` | 5 s | someone asking an app: `answer` types the prompt, rings the field, pulses thinking dots, then the rows land (checked, struck, lit, sponsored; a row with no text is a grey placeholder); `search` is a search bar and result rows; `chat` bubbles land on their words (typing dots before theirs) and the thread scrolls |

Layers: `around` is for `orbit` and `polaroids` (their default); pin either
to `front` or `behind` to draw every card on one side. `neon`, `bloom`,
`scrapbook` (and `slam`, `keyword`) read best `behind` only when the shot has
headroom (the top of the head below about 25% of the frame); on a tight crop
leave them in front, where `zone: auto` puts them above the face. Behind and
around elements and speaker stickers use the person matte: the preview shows
"Preparing background" while it computes, the export runs it itself. Run
`project.detect-face` (or `project.set-focal-point`) first so the cards,
walls and panels centre on the face.

### The creator-effects reel, one call per effect

| effect (as in the "everything my editor can do" reels) | one call |
|---|---|
| wipe text | captions with `caption.set-template --templateId=clipWipe` |
| reel wall | `reelWall {}` |
| text slam above the head | `slam { text }` (`layer: behind` with headroom) |
| color bloom | `bloom { text }` |
| card reveal | `cardFan {}` |
| twin stickers | `sticker { image: "@speaker", twin: true }` |
| orbit | `orbit {}` |
| carousel (3 cards at the bottom) | `steps { items: [a, b, c], images: ["@speaker", "@speaker", "@speaker"] }` |
| kinetic title | `keyword { text }` |
| word wall | `wordWall { text }` |
| call to action | `endCard { lines: [ask, line] }` |
| green screen | `project.set-clip-chroma-key` on green-screen footage; otherwise `project.add-background-effect --mode=image --backgroundImage=<studio>` |
| neon sign | `neon { text }` |
| polaroid pile | `polaroids {}` |
| add to cart | `productCard { title, price? }` |
| text behind you | `behind { text }` |
| scrapbook text | `scrapbook { text }` |
| background blur | whole video: `project.set-speaker-background --mode=blur` (replaces any earlier whole-video background); one stretch: `project.add-background-effect --mode=blur` |
| three scenes | `splitScreen {}` |
| keys falling | `keycaps { text }` |
| scroll overlays | `scrollList { title, items }` |

The `creator-effects-short` recipe puts them together with variety rules
(never the same effect twice in a row, picture / word / scene effects in
rotation, one full-frame takeover per 6 s at most).

Text limits: 160 characters per field (a lower-third subtitle 320; a ticker
2000, a crawl 6000, credits 200 entries); keep headlines to 1-4 words, a slam
to one short statement. Any script, symbols (≤ ≥ → ✓ ★) and colour emoji draw
in the preview and the export.

Timing: `durationMs` / `endWordId` on add; `project.update-motion-element
--elementId --atMs --durationMs` to retime one (annotations and arrows:
`project.update-region --startMs --endMs`). The editor has
typed Start / Length fields (0.1 s steps) for elements and annotations, so a
short arrow can show for exactly 0.5 s.

## Style

`style: { family, accent, highlight, text, surface, font, size, outline, opacity, secondary, faces, lineFit }`,
every field optional. Defaults come from the workspace brand kit (`workspace.get-brand`):
accent = brand primary, highlight = brand accent (else gold), fonts from the
brand when the engine has them (else the family's bundled face).

`surface` is the colour of the element's own plate: the lowerThird plate, the
chip pill, the endCard band, the ticker band. Want a dark lower third with
white text? `style: { surface: "#0B1220", text: "#FFFFFF" }` (highlight is the
emphasis colour, not the plate). Without `surface` the family picks it
(`clean` = the brand's light card, `bold` = accent); text left unset picks a
readable ink for the plate, and a `text` colour the family plate can't carry
(white on the clean light card) swaps the plate for a dark one instead of
drawing invisible words. `look: "plain"` on a lowerThird drops the plate
altogether (the text alone, with its shadow).

| family | look |
|---|---|
| `bold` | Poppins, heavy, accent plates, big motion (Shorts) |
| `editorial` | Inter, restrained, shadowed type, thin rules (long-form) |
| `clean` | Inter, light cards, gentle motion (tutorials) |
| `playful` | Poppins, pills, bouncier motion |
| `paper` | Vox paper-cut: Playfair Display in ink on torn newsprint strips that slap on one by one (slightly crooked, soft shadow, grain), a yellow highlighter swipe on the emphasis word, JetBrains Mono label strips (chips, subtitles, count labels). Made for a paper backdrop + keyed-out speaker (`style-edit --style=paper-cut` sets that up) |

Paper strip layout: headlines of 7 words or fewer stay on two strips with
balanced breaks (never a lone word on the last strip); longer ones use three.

`size`: xs | s | m | l | xl. `outline: true` gives slam / behind an outline
look. `font`: any bundled face (Inter, Poppins, JetBrains Mono, Playfair
Display, Bebas Neue, Anton, Great Vibes, Archivo, Archivo Expanded, DM Sans,
Monocraft, Instrument Serif, EB Garamond, PT Serif, Caveat Brush) or a font the
user imported.

### Rich text: one line, several faces (2.1.82+)

The text of `keyword`, `chip`, `highlight` and `lowerThird` (title and
subtitle) can set single words in their own face, so a heavy sans headline
carries an italic serif accent word ("the RISK of *trust*"):

- `*words*`: the "em" face, by default the italic serif (Instrument Serif,
  a touch larger).
- `{name:words}`: a named face. Built-ins: `serif`, `garamond` (EB Garamond
  italic 700), `playfair` (Playfair Display italic), `script` (Caveat Brush),
  `mono`, `light` (300), `regular` (400), `heavy` (900), `italic`, `accent`
  (accent colour), `highlight`.
- `style.faces` defines or overrides faces: `{ "em": { "font": "EB Garamond",
  "italic": true, "weight": 700, "scale": 1.13, "color": "accent" },
  "of": { "font": "Instrument Serif", "italic": true, "scale": 0.6 } }`.
  Fields: `font`, `weight` (100-900), `italic`, `color` (a colour or
  `accent` / `highlight` / `text`), `scale` (0.3-3, shares the baseline),
  `letterSpacing` (em), `baselineShift` (em, + = down).
- A face covers whole words; `\*` and `\{` are literal. The engine shapes each
  word in its face, so wrapping, centring and plates use the real widths and
  preview = export.

`style.lineFit: "width"` (keyword) sets every line of a stacked headline to
the same width, short lines bigger (lines balanced, or break them with `\n`);
`"line"` sets the whole text on one line fitted to the width (a big word). `content.motion:
"slide"` (keyword) slides each line in from off frame, alternating sides
(`slideFrom`: the first line's side, default `left`), decelerating hard, and
on out the way it came, instead of the word-by-word rise. `style.shadow:
false` sets the text flat (no drop shadow); `style.weight` (100-900) sets the
main text's weight over the family's; `style.stack: true` (keyword) breaks
each paragraph into a balanced stack of lines (two for 2-4 words).
`text`: the main text colour (it also wins over a lower third's plate ink).
`opacity`: the main text (0-1).

`secondary: { size, opacity, color }` styles the element's smaller line on its
own: a count's label, a lower third's subtitle, a ticker's label box, credits
roles. `size` is relative to the default (0.4-2.5), so a count with a big
number and a small, faint label is
`style: { secondary: { size: 0.8, opacity: 0.6 } }`, and a louder label is
`secondary: { size: 1.6, color: "#FFD23F" }`.
Mix families only on purpose: one family per video reads as designed.

## Where it sits: zones

`zone: "auto"` (default) puts each type in its home and keeps front elements
off the face (face tracking / the clip focal point): 9:16 = a band above the
face, else the top 8-30%; 16:9 = beside an off-centre face, else the top band,
else the lower band; lower thirds and progress bars have their own homes.
Named bands: `top`, `upper`, `center`, `lower`, `bottom` (still nudged off
the face). Exact: `{ x, y, w? }` in PERCENT of the frame (0-100, not 0-1:
`{x:50,y:34}` = centred, a third down), (x, y) = the element's CENTRE, w = its
width. Dragging an element on the canvas sets this.

Elements on screen at the same time never cover each other (same layer;
`frame` and `behind` excepted). The one added first keeps its spot; a later
one whose spot is taken stacks right under or over it, else moves to the next
free band, else steps down in size, still off the face and the caption band.
An exact `{ x, y }` element never moves (the others dodge it). When no free
spot exists (two exact elements on one spot, or too many tall blocks at once)
`add-motion-element` / `add-motion-elements` / `update-motion-element` return
`warnings` naming the overlapping elements: tell the user, then shorten, move
in time or remove one.

## Layers: in front of or behind the speaker

`layer: "front"` (default) draws over the video. `layer: "behind"` (the
default of `behind`, `reelWall`, `cardFan`, `wordWall`) draws behind the person: with a chroma key or a
"remove" speaker background it sits under the transparent video; otherwise
PandaStudio cuts the person out with the person matte and draws
[video] → [element] → [person], so the head passes in front. Check with
`project.render-frame` that the word still reads around the head. Any type can
go behind (a count behind the speaker, a keyword), not just `behind`.
`layer: "around"` (`orbit`, `polaroids` only, their default) splits the
element: the cards on the far side of the speaker draw behind her, the rest in
front.

Front elements are part of the foreground stack with media overlays and
annotations: position = group base + `zIndex` (B-roll 0, media overlays 1000,
annotations 2000, front elements 3000), higher on top, so by default an
element draws over every picture and title on screen with it. `zIndex` on an
element (default 0) orders it among elements and, past 1000, across groups:
`-2500` puts it under the default overlays. Behind elements stay under the
person whatever their `zIndex` (it orders them among themselves).

## Sound roles

`sound`: auto (the type's role, table above) | none | pop | whoosh | hit | tick
| bell. Elements make no sound by themselves: `project.compose-soundtrack`
reads each role and scores it at the element's start (steps: every item's cue,
slam: a hero hit), so call it LAST. Paper-family elements on `auto` get
paper foley instead (library sound cues in the group `paper-sfx`, replaced on
re-run), timed to each strip's impact. Set `sound: "none"` on an element that
should stay quiet.

## Recipes

**Short (9:16, 15-90 s)**: `project.style-edit --style=bold-short` covers it.
By hand: a keyword hook in the first second, one element per 3-4 s beat
(count for numbers, chip for terms, stamp for verdicts, one slam on the hero
line), zooms between them, an end card, `compose-soundtrack --style=short`.
Captions can stay on (`caption.move` them clear of the top band) or go when the
keywords carry the meaning.

**Long-form talking head (16:9)**: `project.style-edit --style=editorial-long`.
By hand: a keyword on the claim of each section, a lower third per chapter,
count for the key numbers, at most one slam, sparse (one element per 10-20 s),
`compose-soundtrack --style=talking-head`.

**Tutorial**: `clean-tutorial`: chips for tool / menu names, steps for
sequences (`cueWordIds` on each item's word), progress for a countdown, no
slams or stamps, `compose-soundtrack --style=calm`.

**Code-switched speech (Tanglish, Hinglish)**: the speech map already flags
English statements and key terms. Pass `texts` with an English keyword for
the hero moments in the other language; English phrases are used as spoken.

## When to use HTML instead

Native elements cover keyword graphics. Use an HTML composition
(`motion.render-html`, custom-html.md) when the graphic is bespoke: a chart,
a diagram, a UI mock, a logo animation, a scene that isn't text, a layout no
type expresses.
Both can share a video: native elements for the words, HTML for the one
illustrated moment.
