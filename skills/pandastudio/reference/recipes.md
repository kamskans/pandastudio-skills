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
| `rules-listicle-cutaways-short` | short | Fast educator listicle: claim-stack pill badges, numbered rules, a full-screen cutaway per rule, picture changes every 3 to 5 s |
| `rapid-fire-list-short` | short | Cheat-sheet list: title card, one item every ~3 s with a colored name and use line as the caption, no music |
| `tv-style-explainer` | long | Chapter header bar, big number callouts, camera sliding into cards and corners |
| `product-demo-walkthrough` | long | Title card per feature, feature banner while demoing, zooms on what's discussed, takeaways card |
| `punchy-youtube-talking-head` | long | Tight jump cuts, punch-in zooms on strong lines, keyword pops, stat cards, takeaways |
| `course-lesson-chapters` | long | Title + learning goals, chapter cards, highlighted definitions, captions, recap, chapter timestamps |
| `product-promo-from-url` | long | Footage optional: launch video from real product footage or website screenshots, camera flies into what's used, drawn cursor clicks, crisp typed close-ups, logo + line close |
| `whiteboard-explainer` | long | No footage: hand-drawn draw-on scenes on paper, one metaphor, red arrow on each key point, summary board |
| `social-ad-hook-variants` | short | Footage optional: 9:16 ad (hook, problem, product, proof, offer, CTA), three hook versions, 1:1 and 16:9 copies |
| `warm-educator-short` | short | Calm premium educator Short: sh-serif-hook-title in the first 2 s, Boxed pill captions, one sh-serif-quote on the key sentence, sh-icon-badge-band cutaways, sh-annotated-card for walked-through screens, gentle punch-ins, soft music |

Shorts recipes follow the short-form grammar measured across popular educator, business and podcast Shorts: no preamble (speech by 2.5 s), something visible changes every 3 to 8 s, graphics in the top ~40%, captions at chest height, faces never covered, end on the payoff.

## Running a recipe

Every recipe has an **input**: `footage` (the default; it edits a recording), `none` (it builds the whole video from scratch: promo, whiteboard) or `optional` (it uses the project's clips when there are any, otherwise builds from scratch). `recipe.get` returns it. Run a `none` recipe in an empty project: if the current project already has clips, ask the user before creating a new one with `project.new`. From the home screen, picking a no-footage recipe offers **Start a new video**, which creates the empty project and opens it with the recipe ready.

```bash
# 1. Find one (filter by format when you know the destination)
pandastudio recipe.list --format=long --json

# 2. Read its blanks, fixed style and checklist
pandastudio recipe.get --id=product-demo-walkthrough --json

# 3. Fill the blanks. Omitted blanks use their defaults; required ones must be given.
pandastudio recipe.render --id=product-demo-walkthrough \
  --values='{"product":"PandaStudio","featureCount":"3"}' --json
# → { prompt, runContext, values, checklist, missing }
```

Then do the edit on the current project: follow `prompt` for the content-dependent work, apply everything under "Fixed style" in `runContext` exactly (don't redesign it), and before reporting done, verify each checklist item with `project.render-frame` or `project.read`. Report anything you couldn't complete.

When the user runs a recipe from the app, the in-app agent receives the same `runContext` in its editor context for that turn, so the flow is identical.

## Saving and sharing

- **"Save this as a recipe" / "remember this style"** after an edit came out right → `recipe.save --recipe='<json>'`. Turn video-specific values (language, colors, product name, topic) into blanks with sensible defaults; keep the things that define the look fixed in `style`; write 3 to 6 checklist items a frame can confirm. Set `format` to `short` for vertical output and `long` otherwise (it's inferred from `aspectRatio` when omitted).
- `recipe.export --id=<id>` writes a `.pandarecipe` file (no project data or footage). `recipe.import --file=<path>` adds one.
- `recipe.delete --id=<id>` removes a saved recipe; starters can't be deleted.

## Caveats

- Recipes are guidance for an agent, not a macro: content steps (which beats get graphics, what the B-roll shows) are decided from the transcript each run, so results vary with the footage.
- A recipe that asks for things the footage can't support (a Short recipe on landscape footage, chapters on a 40-second clip) should be flagged to the user before running.
