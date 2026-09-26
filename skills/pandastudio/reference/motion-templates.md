<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Motion graphics: backgrounds, designed segments, template catalog

## The template workflow

```bash
# 1. ALWAYS discover first — templates + editable slots (runtime source of truth).
#    Narrow it: --family=stats-data, --query="subscribe", --tags=quote,
#    --aspect=9:16 (see "Families, search and retired templates" below).
#    Also returns registryBlocks (slot-less standalone compositions: effects,
#    overlays, flowcharts, code snippet, beat-driven cuts): render a block's
#    htmlPath with motion.render-html (templates.md "Hyperframes registry blocks").
pandastudio motion.list --query="bar chart" --aspect=16:9 --json   # MCP: motion_list

# 2. Pick the template that fits THIS beat, render it with your text/colors +
#    a background mode. Don't hardcode one template for every insert.
JOB=$(pandastudio motion.generate --templateId=<TEMPLATE_ID> \
  --slots='{ ...the chosen template's slots from motion.list... }' \
  --aspectRatio=16:9 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json

# 3. Add the rendered clip (at the playhead by default; atMs to place it).
pandastudio project.add-motion-graphic --id="$PROJECT" --fromJob="$JOB" --durationMs=4000
```

- **Editable everything** — text, colors, list items and images are `slots`;
  pass only the ones you change. `motion.list` returns each slot's key, type
  (`string` / `color` / `list` / `image`) and default.
- **Image slots** take an **absolute file path** (project media or a
  `media.generate-image` output); the renderer stages it. `image-showcase` is
  the dedicated one (a screenshot/photo on a 3D-tilted card, 16:9 + 9:16);
  `vox-side-panel` has a small photo slot. Example:
  `--slots='{"image":"/abs/screenshot.png","headline":"Ship faster.","eyebrow":"SEE IT IN ACTION"}'`.
- **`--fromJob`, NOT `--file`, for render outputs** — pass the render `jobId`
  (after `job.wait`); the server resolves the path. `--file` is only for
  external uploads and **must be quoted**: render outputs live under
  `…/Application Support/…` (a space); an unquoted path truncates and silently
  produces a dead overlay (on the timeline, never renders).
- **Placement** — `atMs` re-times it; anchor to a transcript word with
  `--anchorSourceMs`.
- **Default SFX** — every primitive that places an animated callout attaches a
  stinger by default. Don't pass `soundUrl` to "set the default"; omit it.
  Override only when the user asks, or `--soundUrl=none` (MCP: `null`) for a
  silent callout (text/emphasis overlays: three cards shouldn't click at the
  viewer).

  | Verb | Default | Notes |
  |---|---|---|
  | `project.add-motion-graphic` | `bundled:sound/mouse-click` | Since v1.36.0; every motion graphic, custom MP4/WebM, designed-segment panels. |
  | `project.add-designed-segment` | `bundled:sound/mouse-click` | Inherits from add-motion-graphic. |
  | `project.add-zoom` | `bundled:sound/swoosh-fast` | |
  | `project.add-lower-third` | `bundled:sound/mouse-click` | Inherits the motion-graphic default. |
  | `project.add-fx` | none | FX often carry their own audio. |

  `asset.list-sounds` lists other bundled sound ids.

## Lower thirds: one call

`project.add-lower-third --name="…" --title="…" --atMs=<ms> [--templateId=lt-*]
[--enter --exit]` — ONE async call that renders the nameplate (default
`lt-vox-marker`) AND places it as a transparent overlay, laid out for the
project's aspect (each design has a real 9:16 and 1:1 layout; pass
`--aspectRatio` only to override). Returns `{ jobId }`; `job.wait` resolves
once the region is placed. Pick the design by tone: `lt-vox-marker` bold
(default), `lt-glass-card` modern, `lt-minimal-line` elegant, `lt-bold-bar`
loud / broadcast, `lt-logo-name` with a company or channel logo
(`--slots='{"logo":"/abs/logo.png"}'`), `lt-duo` two speakers at once
(`--slots='{"name2":"…","title2":"…"}'`). Keep one design per video. Pass
`--anchorSourceMs` when atMs comes from a transcript word. lt-* designs animate their own entrance:
give them only an `--exit`. With captions on, lift them clear for the lower
third's span: `caption.move --whileRegionId=<its overlay id> --positionY=62`.
(The pre-2.95 CSS designs + `--designType` are gone.)

## Editing a graphic that's already placed

Every overlay placed from a `motion.generate` job, an inline-`--html`
`motion.render-html` job, `project.add-lower-third`, or the editor's Graphics
panel carries `generatedFrom` in `project.read` (`kind: "template"` with
`templateId` + `slots` + `background`, or `kind: "html"` with the markup). To
fix a typo, change a title, swap a color or switch to glass, **re-render in
place instead of deleting and regenerating**:

```bash
JOB=$(pandastudio project.update-motion-graphic --id="$PROJECT" --overlayId=overlay-3 \
  --slots='{"headline":"Record, edit, publish"}' --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json | jq '.data.job.result'
```

- Template graphics: pass only the slots that change (merged over
  `generatedFrom.slots`), and/or `--background=solid|transparent|glass`.
- HTML graphics: pass `--html` with the full new markup (start from
  `generatedFrom.html`).
- Timing, position, size, SFX, anchor and link group are untouched. The old
  render stays on disk.
- No `generatedFrom` (an imported file, or an older graphic): the verb fails
  clearly; render a new one and replace it.
- An `--htmlPath` render isn't editable (its relative assets can't be
  replayed); use inline `--html` + `--assets` when edits may follow.
- The user can do the same: select the graphic → Settings → **Edit graphic** →
  **Update graphic**.

## GIFs, animated emoji and looping overlays

- **GIF / animated WebP / APNG:** `project.add-motion-graphic
  --file=/path/reaction.gif` converts it to a looping transparent WebM, so it
  animates in preview AND export. Still images place as image overlays.
- **Animated emoji:** `project.add-emoji --id=$ID --emoji=🔥 --atMs=4200
  --durationMs=2500 --x=78 --y=30 --size=22 [--enter=pop --exit=fade]` places a
  looping Google Noto animated emoji (downloaded once, cached). Find one with
  `asset.list-emoji --query=laugh`. Sparingly, for reactions and emphasis in
  Shorts, never over the speaker's face.
- **Loop any video overlay:** `project.update-region --regionType=overlay
  --regionId=<id> --loop=true` repeats it for the whole window (default: play
  once, hold the last frame). Looping overlays play without their own sound.

## Video templates (storyboards): a whole video, not one overlay

When the user wants a **finished short video** (a channel intro, an episode
open, a lesson opener) rather than a single overlay, use a **storyboard** — a
multi-scene video template you fill in once:

1. `motion.list-storyboards` → every storyboard: its id, `audience`
   (youtube | podcaster | course) and its `params` (the brief — text + color
   fields; color fields default to the workspace brand kit).
2. `motion.generate-storyboard --storyboardId=<id> --params='{…}'` → renders
   each scene and concatenates them into ONE MP4. Returns a `jobId`. Omitted
   color params fall back to the brand kit (`workspace.set-brand`); omitted
   **required** text params fail fast, so read the brief first.
3. `job.wait` → then `project.add-motion-graphic --fromJob`.

Storyboards render N scenes sequentially, so set a generous `job.wait`
timeout. Scene joins are hard cuts (transitions live inside each scene's own
animation). Prefer a storyboard over hand-composing several `motion.generate`
calls when one fits the brief.


## Backgrounds, designed segments and the template catalog

### Background modes (`--background`)

| Mode | Output | Use |
|---|---|---|
| `solid` | hard full-frame card | opaque templates (cards, stat, checklist, comparison) — the default for those |
| `transparent` | alpha; camera shows through un-painted pixels | overlay templates — the default for those |
| `glass` | alpha + frosted blur of the camera behind the content | overlay templates when you want the modern frosted look; also pass `--backdropBlurStrength=24` on `add-motion-graphic` |

The frost follows the overlay's mask: on a graphic behind the presenter
(`text-behind`, `behindPerson`) the backdrop around the person is frosted
and the presenter stays sharp in front; a shape-masked graphic frosts only
inside its shape.

Omit `--background` to use the template's natural mode. **Never force
`solid` on an overlay template** — its see-through region renders black.
Overlay templates are flagged `overlay: true` in `motion.list`.

### Designed segments (host on one half, panel on the other)

`paper-panel` (torn paper, hand underline) and `vox-side-panel` (graph-paper
spec sheet) are "designed segments": an opaque panel fills one half, the other
half is transparent for the host. Add it with
**`project.add-designed-segment`** instead of `add-motion-graphic` — that
one atomic call also repositions the camera into the open half (a
`cam-{side}-50` clip-transform) over the same span, so host + panel can't
drift. Set the panel's `side` slot; the camera takes the opposite side, with
the panel's own `cameraRatio` (`paper-panel` 55, `vox-side-panel` 50).

**`--durationMs` is how long the panel stays** — set it to the length of the
topic it covers, not a fixed few seconds. The panel **reveals once and holds**:
it has no exit animation, and the overlay holds its last frame for the whole
region (it does NOT loop), so a 30-second explainer beat is just
`--durationMs=30000`. The region end is what removes it. (Same for any overlay:
play-once + hold; size the region/durationMs to the on-screen time you want.)

```bash
JOB=$(pandastudio motion.generate --templateId=paper-panel \
  --slots='{"side":"left","title1":"What MyAgentMail","title2":"handles for you","subtitle":"Inboxes, replies and webhooks"}' \
  --background=transparent --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json
# durationMs = full length of the topic the panel explains (here 28s):
pandastudio project.add-designed-segment --id="$PROJECT" --fromJob="$JOB" \
  --durationMs=28000 --cameraSide=right --cameraRatio=55
```

**9:16 Shorts variant — top/bottom split.** For vertical, the split runs
horizontally. **Use the same premium panels** — `paper-panel` or
`vox-side-panel` — rendered at `--aspectRatio=9:16`: they're aspect-aware and
reflow into a top/bottom band. Use `cameraSide: top`/`bottom` instead of
`left`/`right`; the panel's `side` slot and `cameraSide` are opposites (panel
`top` ⇒ host `bottom`). Keep each panel's own `cameraRatio` (`paper-panel` → 55,
`vox-side-panel` → 50). See "Editing a Short — the vertical playbook" for the
full example and when to reach for it.

### Families, search and retired templates

Every template has a `family` and intent `tags` in `motion.list`. The Graphics
tab groups by family (chips with counts, a search box, Recently used and
Featured rows), and the Lower 3rds tab shows only the `lower-thirds` family.

| Family | Jobs |
|---|---|
| `titles` | bold title, statement / quote, hook, chapter card |
| `text-behind` | words behind the presenter (`project.add-title-behind`) |
| `captions` | caption emphasis line |
| `callouts` | keyword pills, marker highlight, hand-drawn circle, pointer |
| `lists-steps` | numbered / pill / check lists, steps, flowchart, pyramid |
| `stats-data` | stat count-up, bar / line chart, progress bar |
| `comparisons` | versus, before / after, twin cards |
| `product` | product or app showcase, image showcase, agent chat |
| `panels` | side panel / paper panel (designed segments) |
| `social-proof` | testimonial, comments, pricing |
| `social` | subscribe, social follow |
| `intro-outro` | logo intro / outro, countdown |
| `end-cards` | end card / CTA |
| `lower-thirds` | name plates (`project.add-lower-third`) |

Find the right one instead of scanning the whole list:
`motion.list --family=lists-steps`, `--query="subscribe"` (ranked search over
name, description, family and tags; words like vs, cta, quote, chart, timer
work), `--tags=quote,testimonial`, `--aspect=9:16`.

**Retired templates.** Cut templates are marked `retired: true` with a
`replacement`. `motion.list` hides them (pass `--includeRetired` to see
them) and so do both editor tabs. They still render and re-render, so older
projects keep their look. A render of one returns a `warnings` entry
("<id> is retired; use <replacement> instead.") — tell the user, and use
the replacement for anything new. To move an existing graphic onto its
replacement, run `project.update-motion-graphic --overlayId=… --templateId=<replacement>`:
text, list items and same-named colours carry over, and the result lists
`switched.carried` / `switched.dropped`. The editor's inspector offers the
same thing as a "Switch to <name>" button. Never switch without being asked.

Retired in 2.0.2 (check `motion.list --includeRetired` for the live list):

| Retired | Use instead |
|---|---|
| `grain-overlay`, `transitions-destruction`, `caption-parallax-layers`, `yt-prism-title`, `morph-text`, `mk-emphasis-type`, `shader-dissolve`, `vfx-text-cursor` | `transitions-3d` (Bold title) |
| `sh-serif-quote`, `vox-quote`, `hw-title`, `hw-path-text` | `serif-statement` |
| `ring-title`, `parallax-zoom` | `creator-card` |
| `yt-logo-intro` | `logo-intro` |
| `mk-callout-highlight` | `vox-marker` |
| `hw-text-cloud`, `sh-icon-badge-band` | `keyword-pills` |
| `numbered-slide`, `pill-list`, `key-takeaways`, `mk-specs-list`, `parallax-unzoom` | `list` |
| `flowchart-vertical` | `flowchart` |
| `hw-pipeline` | `glow-steps` |
| `apple-money-count`, `mk-progress-stat` | `count-up` |
| `data-chart`, `mk-line-graph` | `bar-chart` / `line-chart` |
| `calm-proof-card`, `sh-annotated-card`, `sketched-frame`, `yt-vertical-fill` | `image-showcase` |
| `split-panel` | `paper-panel` |
| `lt-accent-slab` | `lt-vox-marker` |
| `lt-broadcast-bar`, `lt-kinetic-stack` | `lt-bold-bar` |
| `lt-corner-plate`, `lt-typewriter-tag`, `lt-gradient-sweep` | `lt-minimal-line` |
| `lt-avatar-pill` | `lt-logo-name` |
| `instagram-follow`, `tiktok-follow` | `social-follow` (platform option) |
| `camcorder-hud`, `mk-background`, `mk-clone-wall-transition` | none (use `motion.render-film` transitions or the kit's `LF.ground`) |

### Template catalog

Pick by brief. Slots in **bold** are required; the rest are optional. **Any
slot you omit falls back to the template's own designer-picked default —
including every color** (see each template's `defaults` in `motion.list`). So you
pass only the slots you want to change and the rest look correct out of the box;
you never need to spell out every color. (A workspace brand kit, when set,
overrides the template's default colors; with no brand kit you get the
template's own palette.) All are 16:9 / 9:16 / 1:1 unless noted. `O` = overlay
(transparent-capable).

**Titles & chapter cards**
- `creator-card` (4.5s) — full-bleed brand title / chapter card with a live dot. Bold YouTube-creator look. Slots: **headline**, eyebrow, brandColor, inkColor, dotColor.
- `text-behind` `O` (3s, 1.5–8s) — Text behind you: 1–3 huge words behind the presenter's head. Add it with `project.add-title-behind` (one call: render + head-height placement + behindPerson), not motion.generate. Slots: **text**, style (`3d`\|`bold`\|`serif`), accentColor, animation (`in-out`\|`in`\|`none`), y (set from the face).
- `transitions-3d` `O` (4.5s, 2–10s) — **Bold title**, the loud title: small lead / HUGE word or phrase / small trail over the footage. Styles: `3d` (swings up out of depth, default), `bold` (letters rise out of blur), `serif` (italic word + hand-drawn underline), `kinetic` (each word punches in). Lead and trail resolve word by word; soft shade for legibility; slow push. Slots: lead, **emphasis** (1–4 words), trail, style, position (`center`\|`upper`\|`lower`), animation (`in-out`\|`in`), shade (`on`\|`off`), textColor, accentColor.
- `logo-intro` (4s, 2.5–8s) — channel / brand opener on a soft brand-tinted ground: the logo resolves with a sheen, the name writes itself, the tagline follows, then it racks out into the video. No logo file = a monogram tile (the Graphics panel pre-fills the brand-kit logo). Slots: logo (image), **wordmark**, tagline, animation (`in-out`\|`in` = hold), style (`light`\|`dark`\|`brand`), brandColor, accentColor.
- `logo-outro` (5s, 3–10s) — the sign-off that mirrors `logo-intro`: logo, name, tagline, then a CTA pill (with a press) and the URL / handle; holds to the end. Slots: logo, **wordmark**, tagline, cta, url, style, brandColor, accentColor.

**Educator Short overlays** (all `O`, built vertical-first for 9:16 talking heads; they sit in the top third/half so the speaker and the Boxed captions stay clear below. Pair with `caption.set-template boxed`.)
- `sh-serif-hook-title` `O` (5s, 2–10s) — the Shorts opener: a cream serif setup line resolves word by word, then a big gold-gradient serif payoff racks into focus with a light sweep and two small sparkles, over a soft dark fade at the top; slow push. Place at 0 ms. Slots: line1, **line2**, sparkles (on/off), inkColor, accentColor, accent2Color.

**Calm explainer beats** (16:9, cream page, serif display; the "Floating camera explainer" look. Full-frame ones hide the camera: place them as normal motion graphics over the footage.)
- `calm-statement` (4.5s, 2.5–10s) — full-frame beat on a warm page: one big serif line builds word by word out of blur, the phrase in `*asterisks*` turns italic in the accent with a hand-drawn underline, two soft tints drift while the camera pushes in. Real 16:9 / 9:16 / 1:1 layouts; with the Transparent background it becomes a page-coloured card over the footage. Use on the sentence a section hinges on. Slots: **text**, bgColor, waveColor (background tint), inkColor, accentColor.
- `calm-tier-stack` (6s, 16:9 / 9:16 / 1:1) — a 2 to 5 tier pyramid that builds bottom up out of blur, each tier a lighter shade of the accent, labels revealing word by word; the top tier can be shown as the goal (pale fill + drawn outline). For ladders, levels, price tiers. Slots: title, tiers[{label}] (bottom first), fadeTop (on/off), bgColor, accentColor, inkColor.
- `calm-twin-cards` `O` (5s, 16:9 / 9:16 / 1:1) — two rounded pastel cards beside a full-frame talking head, each with a short serif question or claim: each lifts out of blur and its words resolve, the pair drifts at different depths, then racks out. 9:16 stacks them below the face. Slots: left, right, leftColor, leftInk, rightColor, rightInk.

**Agent / AI chat** (use this instead of hand-building a chat or agent mock-up)
- `agent-chat` (21.4s, 9:16 and 16:9) — a Claude-style app on a phone: the prompt types in on the keyboard, a thinking line and one tool step show (collapsed to "2 steps"), then the reply streams line by line. `**asterisks**` make words bold; answer4 to answer8 render as bullets; empty answer slots are skipped. 9:16: the phone fills the frame. 16:9: the phone tilts up out of depth onto a drifting page at `side` right (default), left or center, with a slow push; add it with `--layer=background` under a `cam-left-portrait` card (side right) for the camera-card look. Keep copy close to the default lengths: the timeline is fixed at 21.4 s and longer text is cut off. Slots: **prompt**, thinking, lead, toolStep, answer1 to answer10, model, side, pageColor.
  ```bash
  pandastudio motion.generate --templateId=agent-chat --aspectRatio=16:9 --slots='{"prompt":"Build me a landing page for my course","toolStep":"write index.html","answer1":"Done. The page has a **hero**, three benefits and a signup form.","answer2":"","answer3":"","answer4":"","answer5":"","answer6":"","answer10":"Want me to deploy it?"}'
  ```

**Keyword pills around the speaker** (transparent, over the FULL-FRAME shot, never over a camera-card slide)
- `keyword-pills` `O` (8s, 16:9 / 9:16 / 1:1) — 1 to 4 off-white serif pills (~80px) that rack in out of blur around the speaker (beside the head, at the shoulders, above; stacked under the face in 9:16), each on the word it names (`at`), drift a little while held and blur out after `hold` seconds (default 3.2). They avoid the `face` box, the frame edges and each other, shrinking if a spot is tight. ALWAYS pass the real face box: run `project.detect-face` over the same span first. Slots: items[{text, at}], face ("x,y,width,height" 0-1), hold, pillColor, inkColor.
  ```bash
  FACE=$(pandastudio project.detect-face --id=$P --fromMs=$S --toMs=$E --json | jq -r '.data.face | "\(.x),\(.y),\(.width),\(.height)"')
  pandastudio motion.generate --templateId=keyword-pills --background=transparent --slots="{\"items\":[{\"text\":\"Consistency\",\"at\":\"0.4\"},{\"text\":\"Patience\",\"at\":\"1.6\"}],\"face\":\"$FACE\"}"
  # then project.add-motion-graphic --fromJob=... --atMs=$S --durationMs=$((E-S))  (normal layer, not background)
  ```
  If `found` is false the camera isn't visible there: pick another moment.

**Camera-card slides** (16:9 long-form, the Ali style look: a full-frame cream slide under the camera as a portrait card. Not for Shorts.)
- `serif-statement` `O` (3.6s, 2–10s, all aspects) — NOT a camera-card slide: the one line that states a section's point, or a quote, in large white serif over the full-frame shot, lower part by default. Rises in (or builds word by word), holds, fades out; `*asterisks*` = italic accent with a hand-drawn underline. Add an `attribution` ("Naval Ravikant") to make it a **quote card for a named person**. Real portrait and square layouts (up to 5 rows in 9:16), so it is also the Shorts quote. Add as a normal (foreground) motion graphic on the sentence itself, at most once every 30 to 60 s, never over a camera-card slide. Slots: **text**, attribution, position (`lower`\|`center`\|`upper`; `top`/`bottom` also accepted; keep it off the face), reveal (`line`\|`words`), shade (on/off), fadeOut (on/off; off to hold for a longer region), inkColor, accentColor.
- `list` (8s, 4-30 s, 16:9 / 9:16 / 1:1) — the one list engine: `style` numbers (steps, questions, a framework: numbered circles + a serif line), pills (a short list of THINGS: rounded pills with a coloured dot, optional illustration `image` at `imageAt`) or checks (takeaways: ticks draw on). Each item blurs in word by word on its own `at` (seconds into the graphic, lands on the spoken word); with `focus` on, earlier items soften as the next lands. `cameraSide` none (fills the frame, default) or leaves room for a camera card: left/right in 16:9, top/bottom in 9:16. Type steps down to fit (two lines per item, three in 9:16). Slots: style, items[{text, at}], title, eyebrow, cameraSide, focus (on/off), image, imageAt, color1-3, pillColor, eyebrowColor, bgColor, inkColor. Ali style camera card: `--layer=background` under a `cam-left-portrait` region with `cameraSide=left`, started 320 ms early and ended 320 ms late (add 0.32 to every `at`).
- `kit-product-slide` (6s, 16:9 / 9:16) — one product or tool per beat: small eyebrow with optional logo, the name in serif, a one-line description, a screenshot on a white card (bundled default screenshot), and an optional dark "Replaces" strip naming the tools it replaces (lands at `replacesAt` seconds, so it hits the word). At 16:9 the content fills the right side (set `cameraSide` right to mirror); at 9:16 it is a full-frame vertical page. Slots: **name**, description, image, eyebrow ("In the kit"), logo, replaces[{tool}] (up to 4), replacesLabel, replacesAt, cameraSide (left/right), accentColor, bgColor, inkColor, mutedColor. At 16:9 use it BEHIND the camera, exactly like glow-steps with `backdrop=panel`:
  ```bash
  JOB=$(pandastudio motion.generate --templateId=kit-product-slide --slots='{"name":"WritePanda","description":"Delete words to cut. Captions, zooms, and publish straight to YouTube.","image":"/path/screenshot.png","replaces":[{"tool":"Descript"},{"tool":"Opus Clip"}]}' --json | jq -r .data.jobId)
  pandastudio project.add-clip-transform-region --id=$P --startMs=$S --endMs=$E --preset=cam-left-portrait --transitionMs=320
  pandastudio project.add-motion-graphic --id=$P --fromJob=$JOB --atMs=$((S-320)) --durationMs=$((E-S+640)) --layer=background
  ```
  Grab the screenshot first (`motion.screenshot` of the product's site, or a real app capture). Don't run more than two of these back to back: return to the full-frame camera between pairs.

**Hand-drawn diagram**
- `glow-steps` `O` (5s, 16:9 / 9:16 / 1:1) — 2 to 4 steps drawn in glowing hand-drawn ink: number and label rack in from blur, a circle scribbles around them, arrows draw between; slow camera push. In 9:16 they stack down the middle (lower part over footage). `layout` stack (down one side) or row; `side` left/center/right; `backdrop` panel (solid, full frame: use as a BACKGROUND graphic under a camera card on the other side), scrim (soft shade over footage) or none. Each step takes an optional `at` (seconds into the graphic) so it lands on the spoken word; steps can reveal out of order (raw, edit, then recipe in the middle). Slots: steps[{label, at}], layout, side, backdrop, inkColor, panelColor. The split look, speaker on the right with the steps on the left:
  ```bash
  pandastudio motion.generate --templateId=glow-steps --slots='{"steps":[{"label":"RAW","at":"1.6"},{"label":"RECIPE","at":"3.0"},{"label":"EDIT","at":"2.1"}],"layout":"stack","side":"left","backdrop":"panel"}' --background=transparent
  pandastudio project.add-clip-transform-region --id=$P --startMs=$S --endMs=$E --preset=cam-right-portrait --transitionMs=320
  # The card animates in over the 320 ms BEFORE $S and out over the 320 ms AFTER $E;
  # the background must cover both or the wallpaper flashes behind the card.
  # So start it 320 ms early (add 0.32 to every step's `at`) and end it 320 ms late.
  pandastudio project.add-motion-graphic --id=$P --fromJob=$JOB --atMs=$((S-320)) --durationMs=$((E-S+640)) --layer=background
  ```
  **Keep the background's content clear of the card.** `cam-left-portrait` covers roughly x=80..750 of a 1920-wide frame (about 35% width), so put text, lists and screenshots in the x=820..1860 zone (mirror it for `cam-right-portrait`). Content laid out from x=560 gets cut off by the card. Check with `project.render-frame` before exporting.
  Leave the sentence before the split full frame when the previous beat was a card on the OTHER side; two card segments back to back make the camera swell and shrink.

**Lower thirds** (all `O`, 5s default, 3-12s via `--durationMs`, transparent overlays with real 16:9 / 9:16 / 1:1 layouts — add in ONE call with `project.add-lower-third --name --title --atMs [--templateId]`; slots: **name**, title + per-template colors. Also in the editor's Lower 3rds tab.)
- `lt-vox-marker` — **the default.** A highlighter stroke wipes across and the name resolves on it word by word; mono role on an accent rule. Long names wrap onto a second stroke. Bold, editorial.
- `lt-bold-bar` — solid bar wipes on from an accent notch with the name in heavy caps; an accent tab with the role drops out underneath. Loud, broadcast / podcast. Slots add barColor, titleColor.
- `lt-logo-name` — a logo tile pops in and a clean light card unfolds out of it with name + role. Slot **logo** (image path, optional; empty = built-in mark in the accent colour), cardColor, inkColor. For guests from a company, a channel, a brand.
- `lt-duo` — two people named at once (interviews, podcasts, panels): bottom-left + bottom-right in 16:9 / 1:1, stacked for a top/bottom split in 9:16. Slots add **name2**, title2, accent2Color.
- `lt-glass-card` — frosted glass card racks in from blur, an accent bar grows, a light sweep crosses the glass. Modern, calm.
- `lt-minimal-line` — name resolves letter by letter, a thin line draws, a letter-spaced role follows; a soft shade keeps it readable on bright footage. Elegant, documentary.

**Subscribe & follow** (transparent overlays, 16:9 / 9:16 / 1:1, place with `motion.generate` + `project.add-motion-graphic`)
- `yt-lower-third` "Subscribe / Like" `O` (4.5s) — channel card; a cursor clicks through the CTA. Slot **action**: `subscribe` (default) \| `subscribe-bell` \| `like-subscribe` \| `like-subscribe-bell` \| `like`. Slots: **name** (channel), title, avatar (optional image), accentColor, cardColor, inkColor.
- `social-follow` `O` (4.5s) — follow card for any platform with its real mark and its own button, clicked into Following / Subscribed. Slot **platform**: `instagram` (default) \| `tiktok` \| `youtube` \| `x` \| `linkedin` \| `threads` \| `facebook` \| `twitch` \| `bluesky` \| `snapchat` \| `pinterest` \| `substack` \| `spotify` \| `github` \| `discord`. Slots: **displayName**, handle, followerCount, avatar (optional photo; the platform mark becomes its badge), verified (`no`/`yes`). Replaces instagram-follow / tiktok-follow.

**Data & explainer**
- `stat-reveal` (4s, 16:9 / 9:16 / 1:1) — a huge number that counts up on a full-bleed brand card (digits blur while they spin), eyebrow + label reveal word by word, a drawn accent line, real portrait type. "10,000+ subscribers", "3× faster". Slots: eyebrow, prefix, **value**, suffix, **label**, brandColor, inkColor, accentColor.
- `count-up` `O` (4s, 2.5-12 s, all aspects) — a number counting up OVER your footage with a soft shade: prefix/suffix pop on landing, small label above, a line below with a drawn accent stroke. `position` center/left/right/top/bottom (default bottom, off the face). Slots: **value** (e.g. 10,000 or 3.5), prefix, suffix, label, eyebrow, from, position, shade (on/off), accentColor, inkColor.
- `bar-chart` (6s, 3.5-15 s, all aspects) — 2 to 8 label/value rows: axis and gridlines draw, bars rise one after another while their numbers count up, the biggest (or `highlight` = 1-based index, or `none`) lands in the accent. Columns in 16:9/1:1, horizontal bars in 9:16. Full-frame page by default; with `--background=transparent` (or glass; the editor's Background toggle) it becomes a frosted panel beside the speaker (right side in 16:9, lower part in 9:16). Slots: title, subtitle, **items[{label, value}]**, prefix, suffix, highlight, barColor, bgColor, inkColor.
- `line-chart` (6s, 3.5-15 s, all aspects) — 3 to 12 points, oldest first: the line draws left to right with a glowing head, the area fills behind it, each point pops as it is reached, and the last value counts up big beside the title. Same page / over-video panel switch as bar-chart. Slots: title, subtitle, **items[{label, value}]**, prefix, suffix, showValues (on/off), lineColor, bgColor, inkColor.
- `progress-bar` `O` (5s, 2.5-12 s, all aspects) — a goal meter on a frosted card: label, a number counting up and a bar (or `style` ring) filling toward `goal` with a glowing head and a sheen on landing. `showAs` percent or value ("$6,200", with "Goal $10,000" below). Slots: label, kicker, **value**, goal (default 100), showAs, prefix, suffix, style (bar/ring), position (bottom/center/top), accentColor, cardColor, inkColor.
- `countdown` (4s, 1.5-60 s, all aspects) — `style` numbers (3-2-1: each number racks in from blur, the ring sweeps once per count) or timer (m:ss ticking one real second at a time, depleting ring), ending on an optional `finalText` ("Go"). Set the duration to fit: 3-2-1 + a final word reads well at 4 s; a timer needs `from` + ~1.5 s. Full-frame card, or over footage with a soft shade with `--background=transparent`. Slots: style, from (3, or seconds / m:ss), label, finalText, accentColor, bgColor, inkColor.
- `callout` `O` (4s, 2-12 s, all aspects) — point at a spot in the footage: a circle, box or arrow draws at `x`/`y` (0-1 fractions; `width` = circle diameter or box width, `height` = box height), optional dim of the rest, and a leader line out to a frosted label card placed where there is room (always in frame). Slots: label, kicker, shape (circle/box/arrow), x, y, width, height, side (auto/left/right/top/bottom), dim (on/off), accentColor, cardColor, inkColor.
- `comparison` (5s, 16:9 / 9:16 / 1:1) — old way vs new way, this vs that: two cards arrive from opposite sides, headings resolve word by word, a VS badge pops, then the verdict (left dims with a cross, right lifts with a check and accent ring). Side by side at 16:9 and 1:1, stacked at 9:16. Slots: leftTag, **leftHeading**, leftCaption, rightTag, **rightHeading**, rightCaption, bgColor, leftColor, rightColor, inkColor.
- `before-after` (6s, 16:9 / 9:16 / 1:1) — slider wipe between two pictures of the same thing: the card rises out of depth, the handle wipes the after image over the before, overshoots, settles mid-frame; Before/After chips. Empty images turn into a words-only before/after (each label large in its half). Slots: before (image), after (image), beforeLabel, afterLabel, headline, bgColor, inkColor, accentColor.
- `flowchart` (6s, 16:9 / 9:16 / 1:1) — numbered glass step-cards joined by connectors a travelling light draws; the camera follows each step, then pulls back to the whole flow. One row in 16:9 (a snake for 5-6), a column in 9:16, a 2-column snake in 1:1. Slots: nodes[label] (2 to 6), brandColor, cardColor, inkColor, accentColor.

**Host + panel split**
- Use `paper-panel` or `vox-side-panel` (both below and in "Designed segments" above). They **reveal once and hold** (no exit): set `--durationMs` to the topic length. Add via `project.add-designed-segment`.
  > **Designed segments are coupled.** As of v1.35.0, the camera layout transform and the panel motion graphic created by `project.add-designed-segment` share a `linkGroupId`. Moving one on the timeline shifts the other by the same delta; deleting one deletes its peer. The agent's mental model can stay simple ("this is one beat"), and direct-manipulation in the UI never produces an orphan transform with no panel to balance it.
  >
  > **Removing / updating a clip-transform (standalone or paired).** Pass `regionType=clip-transform` to the generic verbs — there is NO separate `remove-clip-transform-region` / `update-clip-transform-region` verb. Examples:
  > ```bash
  > # Remove (deleting the paired panel graphic too if part of a designed segment)
  > pandastudio project.remove-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 --json
  >
  > # Change the preset and bump the slide-in transition
  > pandastudio project.update-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 \
  >   --preset=cam-left-50 --transitionMs=480 --json
  >
  > # Shift it later by 2s (paired peer in the link group shifts by the same delta)
  > pandastudio project.update-region --id=$PROJECT \
  >   --regionType=clip-transform --regionId=ctr-1 \
  >   --startMs=6000 --endMs=12000 --json
  > ```
  > Valid presets: `cam-bottom-half`, `cam-top-half`, `cam-right-portrait`, `cam-right-portrait-sm`, `cam-left-portrait`, `cam-bottom-right-quarter`, `cam-bottom-left-quarter`, `cam-right-55`, `cam-left-55`, `cam-right-50`, `cam-left-50`, plus the podcast layout-over-time presets `layout-side-by-side`, `layout-podcast`, `layout-host-full`, `layout-guest-full` (see "Podcast: change layout over time" below). `transitionMs` clamps to `0–2000`.

**Background graphic + camera card** — to put a full-frame motion graphic BEHIND a talking-head (camera as a side card): add the graphic with **`project.add-motion-graphic --layer=background`** (renders it behind the camera), then add a camera clip-transform with a small CARD preset (`cam-right-portrait-sm` recommended, or `cam-bottom-right-quarter` for a corner) over the SAME span. The camera card composites on top automatically, in preview AND export. **Always pass `--layer=background`** — it's what makes the live EDITOR PREVIEW correct (without it the graphic still ends up behind the card in the exported video via the camera punch-through, but the preview shows it on top, which confuses the user). In the UI this is the one-click **"Background"** placement in the Graphics panel.

**Vox explainer style** (bold, motion-forward; the Vox templates default to Vox yellow `#F7C600` but every color is brand-aware)
- `vox-marker` (4.5s, all aspects) — heavy headline reveals word by word from blur, then a marker wipes across the emphasis word (or phrase) and it flips to the marker ink along the wipe. The iconic Vox headline. Full-frame card, or over your video with a soft shade with `--background=transparent`. Slots: eyebrow, **headline**, **emphasisWord**, bgColor, inkColor, accentColor, markInk.
- `vox-stat` (4.5s, all aspects) — a big figure racks in while its digits roll, an accent rule draws, the label reveals word by word and a "SOURCE:" credit fades in. Vox data callout on a dark card (real numbers; the dark alternative to `stat-reveal`; for a number over footage use `count-up`). Slots: eyebrow, **value**, **label**, source, bgColor, inkColor, accentColor.
- `image-showcase` (6s, 16:9 / **9:16**) — **the dedicated image/screenshot template.** A screenshot or photo floats in on a 3D-tilted card with a soft shadow + glass sheen, beside (16:9) or above (9:16) an optional headline, then turns to face the camera. Optional `highlight` ("x,y,w,h" in % of the image) rings one area and dims the rest once the card faces the camera (proof beats). Ships a real default screenshot. Pass `--background=transparent` to float the card straight on the host footage. **Featured.** Slots: **image** (abs path), headline, eyebrow, highlight, bgColor, inkColor, accentColor.
- `app-showcase` (7s, 16:9 / 9:16 / 1:1) — your app, big: a screenshot inside a `frame` of `browser` (default, with `url` in the address bar), `mac` or `phone` rises out of depth with a 3D tilt while the camera pushes and dives onto it, with foreground parallax and a word-by-word headline. 9:16 dives in and pans across a wide screen; the phone frame stands beside the headline at 16:9. Slots: **image**, frame, headline, eyebrow, url, bgColor, inkColor, accentColor.
- `vox-annotation` (4.5s, all aspects) — a hand-drawn loop draws around a subject word, a curved arrow draws out to a note that reveals word by word. "this is what matters" callout. Full-frame card, or with `--background=transparent` it circles a spot in your footage at `x`/`y` (0-1; leave `subject` empty to just circle something in the shot). Slots: subject, note, x, y, bgColor, inkColor, accentColor.
- `vox-side-panel` `O` (16:9 / **9:16**) — Vox designed segment: a graph-paper half-panel with a two-line marker title (words resolve, then the marker sweeps), a taped specimen card with a photo (bundled default; subject text if the photo is removed) that drops in and gets taped, and a monospace spec list that types itself; the other half is transparent for the host. **Aspect-aware** — at 16:9 it's a left/right side panel; render at `--aspectRatio=9:16` and it reflows into a top/bottom **band** for Shorts. Add via `project.add-designed-segment` (16:9: `side` left/right, cameraRatio 50; 9:16: `side` top/bottom, `cameraSide` opposite, cameraRatio 50). Slots: side, **title1**, **title2**, photo, **subject**, spec1, spec2, spec3, paperColor, inkColor, accentColor, accent2Color.
- `paper-panel` `O` (16:9 / **9:16**) — designed segment: a torn-paper sheet whose two-line title resolves word by word (the second line accented, with a hand-drawn underline drawn under it) + a short subtitle; the paper drifts while the type pushes in; the other half is transparent for the host. The cleanest, most editorial of the panels; titles fit on one row. **Aspect-aware** — at 16:9 it's a left/right side panel (camera 55%, panel 45%); render at `--aspectRatio=9:16` and the sheet becomes a top/bottom **band** for Shorts. Add via `project.add-designed-segment` (16:9: `side` left/right; 9:16: `side` top/bottom, `cameraSide` opposite; cameraRatio 55 either way). Slots: side (16:9 `left`/`right`, 9:16 `top`/`bottom`), **title1**, **title2** (accented line), subtitle, paperColor, inkColor, accentColor.
- `testimonial` (7s, 16:9 / 9:16 / 1:1) — one real customer quote as a large card: stars pop, the quote resolves word by word in serif, then photo, name and role settle in. `background` `scene` (drifting brand ground) or `none` (a smaller card low over your footage). Use real quotes only. Slots: **quote**, name, role, avatar (image), stars (5/4/0), background, bgColor, inkColor, accentColor.
- `pricing` (7s, 16:9 / 9:16 / 1:1) — one to three plans rise out of blur; the `highlight`ed plan lifts while the others rack out of focus, its price rolls in, a `badge` pops and its button presses. Row at 16:9/1:1, stacked (highlighted first, others compact) at 9:16. Slots: headline, tiers[{name, price, period, features ("a; b; c", up to 5), cta}] (1-3), highlight (1-3), badge, bgColor, inkColor, accentColor.
- `yt-comment-card` `O` (8s, 16:9 / 9:16 / 1:1) — up to three YouTube-style comment cards land lower-left (lower-middle at 9:16): each rises out of blur, its words resolve, its like button is tapped (thumb fills, count ticks up). Real avatar photos (bundled defaults), light or dark cards. Slots: **comments** ("@user: text | @user: text"), avatar1-3 (images), cardStyle (light/dark).
- `end-card` (6s, 16:9 / 9:16 / 1:1) — the closing frame: logo + name resolve letter by letter, a two-part serif tagline, then an optional YouTube Subscribe button a cursor clicks into Subscribed, a pressing CTA pill, the URL/handle and real platform icons (`socials`: "YouTube, Instagram, TikTok") popping in. `layout` `left` keeps the right side clear for YouTube end-screen videos. Slots: logo (image), name, tagline, cta, url, subscribe (on/off), socials, layout (center/left), bgColor, inkColor, accentColor.

#### Import a podcast from separate files (host + guest recorded apart)

When the two (or more) speakers were recorded SEPARATELY — e.g. local tracks
exported from Zoom/Riverside, or two phone cameras — assemble them into a single
two-speaker podcast composite with **`project.add-podcast-clip`**. This produces
the exact same representation as an in-app/remote podcast recording (host =
mediaPath, guests = `participants[]`, `kind: "podcast"`), so every podcast verb
below (layout-over-time, per-speaker transcript editing) works on it.

```bash
# Host + one guest (2-party). Auto-syncs the guest to the host by default.
pandastudio project.add-podcast-clip --id=$PROJECT \
  --media="/path/host.mp4" --guests='["/path/guest.mp4"]' --json

# 3-4 participants: pass guests in order (guest, guest-2, guest-3).
pandastudio project.add-podcast-clip --id=$PROJECT \
  --media="/path/host.mp4" \
  --guests='["/path/guest1.mp4","/path/guest2.mp4"]' \
  --labels='["Alex (host)","Sam","Jordan"]' --json
```

- **Auto-sync**: each guest is audio-aligned to the host (energy-envelope cross-
  correlation) and the offset stored on the participant. Pass `--autoSync=false`
  to assemble with zero offsets. Re-run later with **`project.auto-sync-podcast
  --clipId=…`**. The response includes a per-guest `{ offsetMs, confidence,
  rawOffsetMs }` report. **It only applies a shift when confident** (correlation
  ≥ 0.35): a low-confidence result leaves `offsetMs` at 0 (so tracks recorded in
  alignment, or isolated per-mic remote tracks that don't cross-correlate, are
  never shifted by a spurious peak) — `rawOffsetMs` shows the rejected guess.
  Nudge manually with `project.set-participant-offset` when needed.
- **Manual nudge**: **`project.set-participant-offset --clipId=… --speaker=guest|guest-2|guest-3 --offsetMs=…`**
  shifts one guest's video AND audio vs the host (±10000; positive delays; 0
  clears). The N-party analogue of `project.set-webcam-offset` (guest-1 only).
  The host is the reference clock and can't be offset.
- **Quick 2-party shortcut**: `project.add-clip --media=host --webcam=guest`
  also makes a podcast composite (no auto-sync; use `add-podcast-clip` for that
  or 3+ speakers).
- **Per-speaker transcription**: run `transcript.transcribe` after assembling.
  For a podcast composite it transcribes EACH participant source separately and
  merges into one speaker-attributed transcript (words tagged
  `host`/`guest`/`guest-2`…). `transcript.get` returns those `speaker` tags, and
  `transcript.delete-words` / `remove-fillers` / `remove-silences` operate across
  the merged conversation — cutting a moment removes it from every speaker's
  video at once, exactly like a recorded podcast.
- **Layout**: the project's webcam layout is set to `podcast` automatically;
  3-4 participants auto-grid. Change it over time with the `layout-*` /
  `podcast-*` clip-transform presets (below).

> UI parity: the Home screen's **Import video** card runs this same flow — pick
> ONE file for a single clip, or SEVERAL (host first, then guests) to assemble a
> podcast composite; auto-sync runs in the background.

#### Podcast: change layout over time (within ONE recording)

To switch a podcast's layout PART-WAY THROUGH a single recording — e.g. side-by-side for the intro, cut to the host full-frame while they make a point, then back — add **clip-transform regions with `layout-*` presets**. These are the SAME timeline item as a camera transform ("Layout transform" in the UI), but instead of repositioning one clip they swap the whole composite for the window. Both the host and guest tiles interpolate at the region edges (default 320ms), so the layout animates.

```bash
# Host full-frame from 30s–45s, animating in/out from the base side-by-side layout
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=30000 --endMs=45000 --preset=layout-host-full --json
# Guest full-frame for a reaction shot
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=61000 --endMs=66000 --preset=layout-guest-full --json
```

`layout-*` presets: `layout-side-by-side` (both, side by side / stacked), `layout-podcast` (two co-equal tiles), `layout-host-full` (speaker 1 / mediaPath only), `layout-guest-full` (speaker 2 / webcamPath only). Outside any region the clip uses its natural layout (the project preset or per-clip override from `project.set-clip-layout`). Edit/remove via `project.update-region` / `project.remove-region` with `regionType=clip-transform` (above).

**Multi-party (host + up to 3 guests) — pick WHICH participants with the participant-aware presets.** The `layout-*` presets above are 2-person only (host=speaker 1, guest=speaker 2). For 3-4 participants, use the participant-aware family + a `--participants` list of speaker ids (`host`, `guest`, `guest-2`, `guest-3` — read them from `transcript.get` / `project.read` clip participants):

```bash
# Solo — feature just one chosen participant full-frame (e.g. the 2nd guest)
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=30000 --endMs=45000 --preset=podcast-solo --participants='["guest-2"]' --json
# Pair — two participants side by side (ordered left → right)
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=46000 --endMs=60000 --preset=podcast-pair --participants='["host","guest-3"]' --json
# Grid — everyone (omit --participants) or a chosen subset
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=0 --endMs=30000 --preset=podcast-grid --json
# Screen share — the shared screen + a strip of participants
pandastudio project.add-clip-transform-region --id=$PROJECT \
  --startMs=61000 --endMs=120000 --preset=podcast-screen-share --json
```

Presets: `podcast-solo` (participants[0] full-frame), `podcast-pair` (participants[0]=left, [1]=right), `podcast-grid` (auto grid — 2 side-by-side / 3 three-up / 4 2×2; the chosen subset or everyone), `podcast-screen-share` (shared screen + participant strip). Selections are validated against the clip's real participants; a stale id (a guest who wasn't in the call) is dropped. Speaker-driven flow: read `transcript.get` speaker tags, then drop a `podcast-solo --participants=[whoever is talking]` region over each span.

> **`layout-*` is PODCAST-ONLY.** These presets only do anything on a clip with `kind === "podcast"` (a clip carrying BOTH a host and a guest source). On a normal screen/camera/upload clip there is no second speaker, so a `layout-*` region is a no-op — use the `cam-*` presets there. Confirm `kind` via `project.read` before placing a `layout-*` region.

**Pull-quote / emphasized line**
- `caption-editorial-emphasis` `O` (4s, 2–10s, all aspects) — one-sentence pull-quote, ONE word (or short phrase) blown up in huge Playfair italic. The lead words resolve out of blur one by one (ghost text ahead), the emphasis racks in with a short glide and a hand-drawn underline, the rest follows; slow push; holds the final frame. Transparent overlay by default; `background: dark` makes it a quiet dark card (storyboards use that). Slots: **sentence**, **emphasisWord**, underline (on/off), background (`transparent`\|`dark`), inkColor, accentColor. Trailing punctuation after the emphasis auto-merges onto the emphasis (so `"…starts with a single frame."` renders `single frame.` as one unit). If `emphasisWord` is NOT a substring of `sentence`, it's appended to the end as a punchline — useful when you want the sentence to LEAD with normal text and END with the dramatic pull-out. **TALKING-HEAD OPENER (default, do this on every talking-head edit): place ONE within the first 10–30s that introduces the TOPIC** — a short line naming what the video is about, derived from the speaker's opening sentences (e.g. host opens "today I want to talk about how we cut our render times in half" → emphasis card `"Cutting render times in half."`). It orients the viewer at the exact moment a talking-head otherwise loses them and reads as deliberate editorial framing. This is a strong default for ANY `kind === "camera"` footage whenever you're adding graphics, not just "make it engaging" briefs. **After the opener, use sparingly: at most 2–3 total, reserving one for the climax / "money line."** The opening topic intro and the chapter-closing payoff are the strongest slots; sprinkling it every minute burns the size contrast. Not a replacement for the running caption track (`caption.set-template editorial` is the style for that). Add via `project.add-motion-graphic` (it's an overlay, not a main-track clip).
  ```bash
  JOB=$(pandastudio motion.generate --templateId=caption-editorial-emphasis \
      --slots='{"sentence":"Every great video starts with a single frame.","emphasisWord":"single frame"}' \
      --aspectRatio=16:9 --json | jq -r '.jobId')
  pandastudio job.wait --id=$JOB
  pandastudio project.add-motion-graphic --fromJob=$JOB --atMs=3000 --durationMs=4000
  ```

> List slots (`items`, `nodes`) take an array of objects, e.g.
> `"items":[{"label":"first"},{"label":"second"}]`. `motion.screenshot`
> renders a single frame of any template+slots combo if you want to preview
> before committing to a full render.

