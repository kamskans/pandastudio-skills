<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Captions, AI metadata, YouTube thumbnails

## Captions

```bash
pandastudio caption.set-template --id=$ID --templateId=neon
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-style --id=$ID --color="#fff" --highlightColor="#34B27B" \
  --strokeWidth=3 --strokeColor="#000" --positionY=85
# Change the caption font (system font or a loaded custom font; unloaded fonts
# fall back to a system default):
pandastudio caption.set-style --id=$ID --fontFamily="Georgia"
# Force ALL CAPS on any template (common for shorts). uppercase=false turns caps
# OFF, including for a template that ships uppercase (editorial):
pandastudio caption.set-style --id=$ID --uppercase=true
```

Static templates: `classic | modern | minimal | bold | spotlight | boxed | neon | colored | coloredWords | editorial | glowStack`. New projects default to `glowStack` (since 1.94); when the user says "add captions" without naming a style, use `glowStack` unless a recipe or destination profile sets another (a recipe's caption setting always wins, e.g. Ali style keeps captions off).

**Animated, transcript-driven styles** (each word animates as it's spoken, identical in preview + export): `kineticSlam` (words slam in), `clipWipe` (wipe reveal per word), `gradientPop` (gradient text, elastic pop), `matrixDecode` (character scramble resolves), `glitchRgb` (RGB chromatic split), `blendDifference` (auto-inverts over any footage). Reach for an animated style for Shorts/TikTok energy; keep `bold` / `editorial` for long-form.

**Moving captions for a span (2.0):** `caption.move --whileRegionId=<overlayId> --positionY=62` lifts the captions clear of a lower third / camera card for exactly its span and eases back (or `--atMs --durationMs`, plus `--offsetX`, `--size`). **Hiding them for a span:** `project.hide-captions --startMs --endMs` (alias of `project.add-caption-region`); `project.show-captions --regionId` removes the hide. See native-motion.md "Captions that move". Captions read words from the project's merged transcript — so you must transcribe first.

- `glowStack` is the short-form headline look: the first word of each caption sits small and white on top, and the rest renders big in heavy Poppins with a glowing yellow-to-orange gradient; each word pops in as it's spoken. Use `--wordsPerLine` 3-4 so each caption reads as lead-in + key phrase. `caption.set-style --highlightColor` swaps the gradient for a solid accent.
- `editorial` is a magazine-emphasis style: the word being spoken RIGHT NOW renders large (and takes an accent color) while the rest of the line shrinks, so one big word sweeps across the line in time with the speech. Best with short `--wordsPerLine` (4-6) so each line reads as a headline. Great for talking-head explainers and punchy hooks.

## AI metadata (uses bundled local LLM)

```bash
pandastudio llm.generate-title --id=$ID --json
# → { title: "..." }

pandastudio llm.generate-description --id=$ID --json
# → { description: "..." }

pandastudio llm.generate-timestamps --id=$ID --maxChapters=8 --json
# → { timestamps: [ { timeMs, label }, ... ] }
```

These are READ-ONLY — they return text; they don't write anything. To actually
SAVE title / description / timestamps onto the export (the fields in the export's
Content tab), use `export.set-details`:

```bash
# Write generated (or user-supplied) metadata back onto the export entry.
pandastudio export.set-details --id=$EXPORT_ID \
  --title="How I Got My First $79 Sale" \
  --description="In this video…"
# timestamps take an array of { timeMs, label }:
pandastudio export.set-details --id=$EXPORT_ID --timestamps='[{"timeMs":0,"label":"Intro"}]'
```

`$EXPORT_ID` is the export-library entry id (`export.list` / `export.get`). When the
user is on the export page, the in-app agent already has it as `exportPage.entryId`
in its editor context — so "update the title to X" is a single `export.set-details`
call. Pass only the fields you want to change.

## YouTube thumbnails (v1.18+)

PandaStudio can generate YouTube thumbnails via Replicate's `openai/gpt-image-2` model on **the user's own Replicate account** — PandaStudio never pays for or proxies these calls. Before any thumbnail verb will work, the user must connect Replicate in **Settings → Integrations → Connectors** (a one-time sign-in, no key to paste).

Requires Replicate connected in **Settings → Integrations → Connectors** (a
sign-in the user does in the app, never via CLI). If a verb returns "Connect
Replicate…", tell the user to connect it rather than looping.

**Verbs:**
- `export.generate-thumbnail --id=$EID` — gpt-image-2 (3:2). Two ways to drive it:
  - **Controlled (preferred):** `--subject="a phone showing a first $79 Stripe sale"`
    `--hook="FIRST SALE"` `--reaction=excitement` `--layout=reaction-split`. The
    prompt is assembled deterministically — no transcript/LLM needed. `reaction` ∈
    {excitement (default), shock, delight, awe, curiosity, pride, determination,
    relief}; `layout` ∈ {reaction-split (default), subject-hero, before-after,
    big-face, product-only, versus}. `product-only`/`versus` ignore the person.
  - **Auto:** omit subject/hook and the local LLM suggests the brief from the
    transcript (needs a transcript).
  - `--prompt="…"` overrides everything with a verbatim prompt. `--referenceImagePath=…`
    is the person's face (passed as input_images; skipped for person-less layouts).
- `export.edit-thumbnail --id=$EID --editPrompt="…"` — chat-style refine; **one
  change per call** (runs at `low` quality). Each edit is reverable.
- `export.set-thumbnail --id=$EID --sourcePath=…` — use a user-supplied file
  (PNG/JPEG/WebP).
- `export.revert-thumbnail --iterationId=…` / `export.clear-thumbnail` — history
  is in `entry.thumbnailIterations[]` (via `export.get`).

**Prompt shape that works** (gpt-image-2 rewards photo-language): one focal
subject in photo terms ("extreme close-up of…"); specify lighting; **quote
on-image text** (`reading "GAME OVER"` — its text rendering is sharp); on edits,
lock what shouldn't change and iterate one thing at a time.

(Full arg schemas: `reference/commands.md`.)

