# Recipes — reusable edit styles

A recipe is an edit that worked, saved so it can be run again on new footage:

- a **prompt** with fill-in blanks (`{{language}}`, `{{accentColor}}`) for the parts that depend on the video,
- an optional **fixed style** (aspect ratio, color correction, grade, background, camera layout, captions) to apply exactly,
- a **checklist** to verify with rendered frames before saying the edit is done.

Recipes are grouped by **format**: `short` (vertical Shorts / Reels / TikTok) and `long` (YouTube videos, product demos, courses, explainers). Starter recipes ship with the app; users can save their own per workspace. In the app, users find them in three places: the **Start from a recipe** row on the home screen (pick a recipe, then record, import or choose a project; the editor opens with that recipe ready), the suggestions in an empty AI chat, and the **Recipes** tab of the AI agent panel, where they fill the blanks and press **Run on this project**. After an edit, the chat offers **Save this edit as a recipe**, which asks you to run `recipe.save`.

## Starter recipes

| id | format | what it makes |
|---|---|---|
| `split-screen-short-broll` | short | Speaker in a studio on the bottom half, B-roll on top, keyword pills, click on every image change |
| `framework-explainer-short` | short | Educator framework: pain-question hook echoed as typography, section headers building in the top third, highlighted captions, payoff punch-in |
| `hard-truth-one-liner-short` | short | Blunt one-idea Short: restarts and pauses cut hard, huge 2-word Bold captions with the spoken word in the accent color, punch-in on every other sentence, strongest on the kicker, no music or graphics |
| `rules-listicle-cutaways-short` | short | Fast educator listicle: claim-stack pill badges, numbered rules, a full-screen cutaway per rule, picture changes every 3 to 5 s |
| `rapid-fire-list-short` | short | Cheat-sheet list: title card, one item every ~3 s with a colored name and use line as the caption, no music |
| `storytime-turn-short` | short | Story Short: hook card pinned at the top for the whole video over a speaker card, jump cuts, Colored captions, thud + 1.5x push-in on the turn, no music |
| `tv-style-explainer` | long | Chapter header bar, big number callouts, camera sliding into cards and corners |
| `product-demo-walkthrough` | long | Calm feature tour from a screen recording (app 2.0.2): kit title card per feature over a held frame, numbered feature banner, callouts on clicks, 1.5x zooms, narration if you don't speak, soft clicks under a ducked bed, takeaways card |
| `punchy-youtube-talking-head` | long | Tight jump cuts, punch-in zooms on strong lines, keyword pops, stat cards, takeaways |
| `educator-talking-head-chapters` | long | The Ali style (default long-form on-camera edit, needs app 1.98.0): you full frame at least half the time; `list` slides (style numbers or pills) / kit-product-slide behind a cam-left-portrait card (max two in a row), serif-statement over the shot, glow-steps diagrams, keyword-pills placed with project.detect-face; three or more moves per video over 3 min, a zoom every 40 s of full frame, soft music, no captions; on 2.0 also chapter titles behind the presenter and one adjustment look for asides |
| `business-lesson-rule-slams` | long | Blunt business lesson: restarts and read-out notes cut hard, beat-by-beat punch-ins as a second angle, full-screen Rule N text slams with a thud, bold boxless captions, no music |
| `course-lesson-chapters` | long | Title + learning goals, chapter cards, highlighted definitions, captions, recap, chapter timestamps |
| `product-promo-from-url` | long | Product launch video from the user's screen recording (app 2.0.2): the recording plays inside LF.appWindow, word-reveal hook, camera push-ins on what's used, a click sound on every click, callouts, montage, end card. No recording: use `motion-graphics-launch` |
| `motion-graphics-launch` | long | Product launch film, no footage (app 2.0.2): built from the website with capture-brand, real 2x screenshots and traced UI in the lf-kit launch-film grammar: pain hook, the miss, morph reveal, logo, 2-3 different demos, montage, end card, sound on every event. Former id `saas-launch-film` resolves here |
| `whiteboard-explainer` | long | No footage: hand-drawn draw-on scenes on paper, one metaphor, red arrow on each key point, summary board |
| `social-ad-hook-variants` | short | Footage optional: 9:16 ad (hook, problem, product, proof, offer, CTA), three hook versions, 1:1 and 16:9 copies |
| `warm-educator-short` | short | Calm premium educator Short: sh-serif-hook-title in the first 2 s, Boxed pill captions, one serif-statement (9:16, upper) on the key sentence, emoji and keyword-pills cutaways, image-showcase for walked-through screens, gentle punch-ins, soft music |
| `faceless-short` | short | No footage: hook-first 30 to 50 s story, one AI image per beat with slow zooms, cloud or local AI narrator, big Bold captions, quiet cinematic music |
| `faceless-youtube-explainer` | long | No footage: narrated story in 10 to 20 s beats over one AI image each with slow zooms, 2.5 s title card, Modern captions, music bed |

Starter prompts name the 2.0 native moves where they fit (behind-the-presenter titles, caption moves, keyframed punch-ins, speed ramps, adjustment layers, audio ducking) and say "skip if the command doesn't exist", because the same catalog serves older apps. On 2.0, do them; on an older app, skip them silently.

Shorts recipes follow the short-form grammar measured across popular educator, business and podcast Shorts: no preamble (speech by 2.5 s), something visible changes every 3 to 8 s, graphics in the top ~40%, captions at chest height, faces never covered, end on the payoff.

A recipe's former ids (`aliases`, e.g. `saas-launch-film` after it merged into `motion-graphics-launch`) still work in `recipe.get`, `recipe.render` and `recipe.apply-style`; use the id `recipe.list` shows.

## Running a recipe

Every recipe has an **input**: `footage` (the default; it edits a recording), `none` (it builds the whole video from scratch: promo, whiteboard) or `optional` (it uses the project's clips when there are any, otherwise builds from scratch). `recipe.get` returns it. Run a `none` recipe in an empty project: if the current project already has clips, ask the user before creating a new one with `project.new`. From the home screen, picking a no-footage recipe offers **Start a new video**, which creates the empty project and opens it with the recipe ready.

```bash
# 1. Find one (filter by format when you know the destination)
pandastudio recipe.list --format=long --json

# 2. Read its blanks, fixed style and checklist
pandastudio recipe.get --id=product-demo-walkthrough --json

# 3. Fill the blanks. Omitted blanks use their defaults, and anything still
#    empty comes back as "(you decide this from the video: <label>)" with its
#    label in `agentFilled` — read the transcript or render a frame and choose
#    it yourself.
#    EXCEPTION: fields with `fromUser: true` (the idea behind a faceless or
#    whiteboard video, a product name, website or offer) must come from the
#    user. recipe.render FAILS until they're given ("needs your input first"):
#    ask the user, never invent them. Fields with `allowScript: true` take an
#    idea OR the user's finished script: for a script pass
#    values.<key> = the script and values.<key>Kind = "script"; it's appended
#    to the prompt and must be narrated word for word.
#    Fields with `type: "images"` take the user's OWN pictures: pass
#    values.<key> as a JSON array (or newline list) of absolute image paths.
#    Empty renders as "none": generate the images with media.generate-image
#    (user's Replicate / Higgsfield connector), or, when neither is connected,
#    ask the user for images. Unless the field is `optional`, render fails
#    until `minCount` paths are given. At most `maxCount` are used.
pandastudio recipe.render --id=product-demo-walkthrough \
  --values='{"product":"PandaStudio","featureCount":"3"}' --json
# → { prompt, runContext, values, checklist, agentFilled }
```

```bash
# 4. Apply the fixed style (aspect ratio, captions + template, wallpaper, camera
#    layout, backdrop, grade). Deterministic — do this BEFORE the content work.
pandastudio recipe.apply-style --id=product-demo-walkthrough \
  --values='{"product":"PandaStudio"}' --json
# → { applied: ["Aspect ratio 16:9", "Captions off"], failed: [], notes: [], warnings: [] }
```

A recipe's `style` can carry the whole screen + camera layout, and apply-style
sets all of it: `screen` (`padding`, `borderRadius`, `shadow`, `fill`: `fit` |
`cover`) and `camera` (`layout`, `crop`, `cx`/`cy`/`scale`, `shape`,
`cornerRadius`, `borderWidth`/`borderColor`, `shadow`, `faceCentred`). Camera
steps are skipped when the project has no camera video. `warnings` flags
recordings whose shape doesn't match the canvas (e.g. a 16:10 Mac recording in a
16:9 project); with `fill: "cover"` the screen is scaled to fill the frame
instead of leaving a strip of wallpaper. Pass each warning on to the user.

Example, faceless Short from the user's own script:

```bash
pandastudio recipe.render --id=faceless-short \
  --values='{"topic":"Every summer the Eiffel Tower grows...","topicKind":"script"}' --json
```

Then do the edit on the current project: follow `prompt` for the content-dependent work, do anything listed in the apply-style `notes` yourself, and before reporting done, verify each checklist item with `project.render-frame` or `project.read`. When the user pressed Run in the app, the style was already applied for you — verify it rather than redoing it. Report anything you couldn't complete.

When the user runs a recipe from the app, the in-app agent receives the same `runContext` in its editor context for that turn, so the flow is identical.

## Saving and sharing

- **"Save this as a recipe" / "remember this style"** after an edit came out right → `recipe.save --recipe='<json>'`. A recipe that builds on pictures can declare an images blank: `{ "key": "images", "label": "Your own images", "type": "images", "optional": true, "maxCount": 12 }` and use `{{images}}` in the prompt (the app shows a file picker and which image connector would generate them otherwise). Turn video-specific values (language, colors, product name, topic) into blanks with sensible defaults; keep the things that define the look fixed in `style`; write 3 to 6 checklist items a frame can confirm. Set `format` to `short` for vertical output and `long` otherwise (it's inferred from `aspectRatio` when omitted).
- `recipe.export --id=<id>` writes a `.pandarecipe` file (no project data or footage). `recipe.import --file=<path>` adds one. Users can do both from the app: Import sits at the top of the Recipes tab, and Export (plus Delete, for their own) sits in a recipe's detail view.
- `recipe.delete --id=<id>` removes a saved recipe; starters can't be deleted.

## Caveats

- Recipes are guidance for an agent, not a macro: content steps (which beats get graphics, what the B-roll shows) are decided from the transcript each run, so results vary with the footage.
- A recipe that asks for things the footage can't support (a Short recipe on landscape footage, chapters on a 40-second clip) should be flagged to the user before running.
