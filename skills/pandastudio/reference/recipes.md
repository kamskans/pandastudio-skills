# Recipes — reusable edit styles

A recipe is an edit that worked, saved so it can be run again on new footage:

- a **prompt** with fill-in blanks (`{{language}}`, `{{accentColor}}`) for the parts that depend on the video,
- an optional **fixed style** (aspect ratio, color correction, grade, background, camera layout, captions) to apply exactly,
- a **checklist** to verify with rendered frames before saying the edit is done.

Recipes are grouped by **format**: `short` (vertical Shorts / Reels / TikTok) and `long` (YouTube videos, product demos, courses, explainers). Starter recipes ship with the app; users can save their own per workspace. In the app, users find them in three places: the **Start from a recipe** row on the home screen (pick a recipe, then record, import or choose a project; the editor opens with that recipe ready), the suggestions in an empty AI chat, and the **Recipes** tab of the AI agent panel, where they fill the blanks and press **Run on this project**. After an edit, the chat offers **Save this edit as a recipe**, which asks you to run `recipe.save`.

## Which recipes exist, and which to use

The catalog is live and changes without an app update, so this doc doesn't
list it. Ask the app:

- **`recipe.pick [--id=<project>]`**: no style named? The recipes that fit this
  project's format and footage, each with `pickWhen` (when it's the right
  pick), plus `defaultId` for this situation. Choose by what's said in the
  video. This is the normal entry point.
- **`recipe.list [--format=short|long]`**: every recipe (the user's saved ones
  first), with `description` and `pickWhen`, when the user asks what styles
  exist.
- **`recipe.get --id`**: one recipe in full (blanks, fixed style, checklist).

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
#    A `look` field (TV-style explainer, educator chapters, course lesson,
#    punchy talking head) colours the panels, cards and background: "Match my
#    footage" (default: render a frame, light panels + dark text for a bright
#    shot, dark for a dark one, accent from the shot if the recipe's clashes),
#    "Light", "Dark" or "My brand colours". Leave it at the default unless the
#    user named a look. `chapters` = "Auto" scales the count to the length
#    (about one per 1-2 min; under a minute, 2 sections at most).
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

## Learn an editing style from a reference

The Recipes tab has **Learn editing style**: choose a local reference or
download a YouTube/video link, name the recipe, prepare evidence, then start
the agent study. Preparation does not import the reference into the current
project. The agent studies it and saves a personal recipe; running that recipe
on new footage is a separate action. This does not retrain a model.

CLI/MCP: `recipe.prepare-reference --file=<absolute-video-path> --title=<name>`
returns a jobId. Wait for the job. Its result supplies a manifestPath, overview
sheets spanning the reference plus opening/ending frames, dense motion sheets,
scene-change candidates and mixed-audio excerpts. Read the manifest's sampling
metadata before interpreting timestamps. Sampling is not exhaustive. Inspect
additional native-rate windows for ambiguous motion; candidates can miss fades
and can include changes that are not cuts. References are limited to 30 min.

Read the whole-video evidence and record timestamped observations with
confidence levels. Capture typography, palette, pacing, holds, camera framing,
B-roll grammar, motion, transitions AND sound texture, envelope and event
alignment. Include straight cuts and silence. Learn film burns and other effects
when the reference uses them; never add a generic pack of effects by default.
Reference audio is a mix: exact sound assets may be unknown. Map to supported
bundled sounds/transitions/FX, label substitutions and unverified audio honestly.
Do not carry source footage, brands, statistics or absolute timestamps into the
new recipe; make the rules portable and content-dependent. Save through
recipe.save, read it back and validate its defaults with recipe.render. Preserve
the user's current project throughout the study.

Review the recipe against evidence, fix the largest discrepancy and retain a
short findings ledger. Review actual rendered frames and decoded audio on the
next application, not only the plan. This evidence-first review approach is
informed by [Motion Video Kit](https://github.com/echris6/motion-video-kit),
particularly its motion vocabulary and render/critic/verification workflow.
Its launch-film pacing targets are not universal defaults for other genres.
