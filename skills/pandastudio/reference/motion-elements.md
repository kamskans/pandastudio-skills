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

Text: the engine fonts are Inter, Poppins and JetBrains Mono, **Latin script
only**. Element text is the English words the speaker used, condensed to a
headline (trimmed of "the / is / to..."). A moment with no Latin words gets a
zoom only. For non-Latin speech, pass English keywords per word:

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

## The 12 types

| type | content | default length | sound (auto) | use it for |
|---|---|---|---|---|
| `keyword` | `{ text }` (or a bare string) | 2.2 s | whoosh | the headline of a beat: 1-4 words rise from a mask in turn, the last in the emphasis colour |
| `chip` | `{ text }` | 1.8 s | pop | a small label: a term, a tool name, a tag |
| `stamp` | `{ text }` | 1.8 s | hit | a verdict: "FREE", "NO", "DONE"; slams in rotated, shakes |
| `count` | `{ value, to?, prefix?, suffix?, label?, decimals? }`; `value: "20-30"` is a range | 2.5 s | tick roll | a number that matters: rolls up to it, label under it |
| `steps` | `{ items[] (1-8), cues?, images? }` or `cueWordIds` | 5 s | pop per item | a list said out loud: each item on its word, latest highlighted. With `images` (one absolute path or null per item) it becomes a card carousel: each image lands as a tilted card on its cue, earlier cards slide aside and dim, the item's label under the card (paper family: polaroids with tape). Give it a roomy zone (`center` or `{x,y,w}`) |
| `slam` | `{ text }` | 1.5 s | hit (hero) | THE line: huge type lands with a camera kick. Once or twice per video |
| `behind` | `{ text }` | 3 s | none | a big outline/solid word behind the speaker (layer behind) |
| `lowerThird` | `{ title, subtitle? }` | 4 s | whoosh (soft) | name + title, a chapter, a source |
| `highlight` | `{ text }` | 2.5 s | none | the phrase with a marker drawn under it as it is said |
| `endCard` | `{ lines: [a, b] }` | 5 s | bell | the two-line takeaway at the end, with a timer bar |
| `progress` | `{ label?, fromMs?, toMs? }` (edited ms; default its own span) | 10 s | none | a chapter / countdown bar |
| `iconPop` | `{ emoji }` or `{ icon }` | 1.5 s | pop | an icon burst: check, cross, plus, arrow-up, arrow-down, clock, star, heart, fire, bolt, bulb, warning, dollar, percent, question, exclamation (emoji map onto these) |
| `frame` | `{ look: viewfinder \| selection \| corners, label? }` | 4 s | none | a frame around the speaker: `viewfinder` = camera-viewfinder overlay on the whole frame (corner brackets, blinking REC or `label`, running timecode, exposure readouts, battery); `selection` = a design-tool selection box around the speaker (handles, `label` name tag, W × H size pill; accent defaults to design-tool blue); `corners` = accent corner brackets around the speaker |
| `sticker` | `{ image: transparent PNG path, label? }` | 2.5 s | pop | a cut-out image as a die-cut sticker: a white border that follows the image's own outline, a soft shadow, a tilted pop and a gentle float; `label` adds a strip of black label tape under it. Make the image with `media.generate-image --transparent=true` (Replicate or Higgsfield connector), or use the user's own PNG; without a connector ask the user for images |

Text limits: 160 characters per field; keep headlines to 1-4 words, a slam to
one short statement. Latin script only (the engine fonts).

## Style

`style: { family, accent, highlight, text, font, size, outline }`, every field
optional. Defaults come from the workspace brand kit (`workspace.get-brand`):
accent = brand primary, highlight = brand accent (else gold), fonts from the
brand when the engine has them (else the family's bundled face).

| family | look |
|---|---|
| `bold` | Poppins, heavy, accent plates, big motion (Shorts) |
| `editorial` | Inter, restrained, shadowed type, thin rules (long-form) |
| `clean` | Inter, light cards, gentle motion (tutorials) |
| `playful` | Poppins, pills, bouncier motion |
| `paper` | Vox paper-cut: Playfair Display in ink on torn newsprint strips that slap on one by one (slightly crooked, soft shadow, grain), a yellow highlighter swipe on the emphasis word, JetBrains Mono label strips (chips, subtitles, count labels). Made for a paper backdrop + keyed-out speaker (`style-edit --style=paper-cut` sets that up) |

Paper strip layout: headlines of 7 words or fewer stay on two strips with
balanced breaks (never a lone word on the last strip); longer ones use three.

`size`: s | m | l | xl. `outline: true` gives slam / behind an outline look.
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

## Layers: in front of or behind the speaker

`layer: "front"` (default) draws over the video. `layer: "behind"` (the
`behind` type's default) draws behind the person: with a chroma key or a
"remove" speaker background it sits under the transparent video; otherwise
PandaStudio cuts the person out with the person matte and draws
[video] → [element] → [person], so the head passes in front. Check with
`project.render-frame` that the word still reads around the head. Any type can
go behind (a count behind the speaker, a keyword), not just `behind`.

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
type expresses, or text in a non-Latin script (HTML has the system fonts).
Both can share a video: native elements for the words, HTML for the one
illustrated moment.
