<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Captions, AI metadata, YouTube thumbnails

## Captions

```bash
pandastudio caption.set-template --id=$ID --templateId=neon
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-style --id=$ID --color="#fff" --highlightColor="#34B27B" \
  --strokeWidth=3 --strokeColor="#000" --positionY=85
# Change the caption font. The render engine draws its built-in fonts:
# Inter, Poppins, Playfair Display, Bebas Neue (condensed caps), Anton,
# Great Vibes (script), JetBrains Mono. Other names fall back to Inter.
pandastudio caption.set-style --id=$ID --fontFamily="Bebas Neue"
# Force ALL CAPS on any template (common for shorts). uppercase=false turns caps
# OFF, including for a template that ships uppercase (editorial):
pandastudio caption.set-style --id=$ID --uppercase=true
```

`caption.set-template` starts from the template's own look: it clears earlier style overrides (as picking a style in the Captions tab does) and takes the template's designed words per group when it has one; position is kept. So pick the template FIRST, then tune with `caption.set-style`.

Static templates: `classic | modern | minimal | bold | spotlight | boxed | neon | colored | coloredWords | editorial | glowStack`. Emphasis templates: `hormoziEmphasis | tiltedBox | serifItalic | condensedCaps | scriptKeyword` (below). New projects default to `glowStack` (since 1.94); when the user says "add captions" without naming a style, use `glowStack` unless a recipe or destination profile sets another (a recipe's caption setting always wins, e.g. Ali style keeps captions off).

**Animated, transcript-driven styles** (each word animates as it's spoken, identical in preview + export): `kineticSlam` (words slam in), `clipWipe` (wipe reveal per word), `gradientPop` (gradient text, elastic pop), `matrixDecode` (character scramble resolves), `glitchRgb` (RGB chromatic split), `blendDifference` (auto-inverts over any footage). Reach for an animated style for Shorts/TikTok energy; keep `bold` / `editorial` for long-form.

**Moving captions for a span (2.0):** `caption.move --whileRegionId=<overlayId> --positionY=62` lifts the captions clear of a lower third / camera card for exactly its span and eases back (or `--atMs --durationMs`, plus `--offsetX`, `--size`). **Hiding them for a span:** `project.hide-captions --startMs --endMs` (alias of `project.add-caption-region`); `project.show-captions --regionId` removes the hide. See native-motion.md "Captions that move". Captions read words from the project's merged transcript — so you must transcribe first.

### Emphasis: highlight the IMPORTANT word, not the spoken one

Most short-form caption presets (Captions.ai, Submagic, Hormozi-style edits)
color, box or restyle the KEY word of each phrase and keep it styled for the
whole time its group is on screen. That is `highlightMode`:

- `spoken` (default): the word being spoken gets the highlight (karaoke).
- `emphasis`: the key words (transcript `word.emphasis`) get the emphasis
  style for their group's whole time on screen; no spoken highlight.
- `both`: key words get the emphasis style AND the spoken word gets the
  highlight; a word that is both takes the emphasis style.

Emphasis templates (all `highlightMode: emphasis`):

| Template | Look | Words/group |
|---|---|---|
| `hormoziEmphasis` (`hormozi-emphasis`) | Inter 900 uppercase white, 12px black outline, key word `#7CFC00` at 1.08x | 3 |
| `tiltedBox` (`tilted-box`) | Poppins 800 uppercase white; key word on a `#7C3AED` purple box tilted ~4° (alternating) | 3 |
| `serifItalic` (`serif-italic`) | Playfair Display 700 white, no outline; key word swaps to italic, 1.06x | 4 |
| `condensedCaps` (`condensed-caps`) | Bebas Neue uppercase pale cyan `#8FD3FF`; key word white at 1.3x | (kept) |
| `scriptKeyword` (`script-keyword`) | Inter 700 white; key word in Great Vibes script, cyan `#22D3EE`, 1.8x (the script face runs small) | (kept) |

More looks (spoken highlight unless noted):

| id | look | use for |
|---|---|---|
| `mutedBox` | dim words on a slate box, the spoken word lights up white | calm talking heads, podcasts |
| `blackBar` | white caps on a black bar, spoken word yellow | news / explainer energy |
| `oneWordPop` | one huge yellow word at a time, outlined | fast hype Shorts |
| `stickerTilt` | black caps on a tilted yellow sticker box | playful, UGC |
| `spokenTilt` | white caps, the spoken word on a tilted purple box | Shorts, creator style |
| `wordBoxes` | one word per white box; key words (emphasis) 1.4x bigger | punchy statements |
| `typewriterMono` | JetBrains Mono caps, spoken word inverted (white block) | docs, tech, archival |
| `goldSerif` | warm gold Playfair; key word italic white (emphasis) | cinematic, stories |
| `keywordBox` | white caps, key word on an orange box (emphasis) | frameworks, tips |
| `limeItalic` | white caps; key word lime italic 1.15x, spoken word pops (both) | hype, sports, money |
| `softPill` | white words, spoken word on a mint pill | friendly, lifestyle |


Applying an emphasis template (or `caption.set-style --highlightMode=emphasis|both`)
detects the key words first when the transcript has none; the result carries
`emphasis: { detected, emphasizedWords, warnings? }`. Any template works in
emphasis mode: without its own emphasis look, key words take its highlight look.

```bash
pandastudio caption.set-template --id=$ID --templateId=tiltedBox
# Tune the emphasis layer (any template):
pandastudio caption.set-style --id=$ID --highlightMode=both --emphasisColor="#FFE000" \
  --emphasisBackgroundColor=transparent --emphasisScale=1.2 --emphasisItalic=true \
  --emphasisFontFamily="Great Vibes" --boxRotation=5
# Re-detect the key words (after re-transcribing, or for denser emphasis):
pandastudio caption.mark-emphasis --id=$ID --everyMs=1800
# Mark / unmark words by hand (ids from transcript.get; hand marks survive re-detection):
pandastudio caption.mark-emphasis --id=$ID --wordIds='["w12","w40"]'
pandastudio caption.mark-emphasis --id=$ID --wordIds='["w7"]' --emphasis=false
```

How key words are picked: `project.speech-map` measured from each clip's audio
(loudness, attack, a pause before, a drawn-out word, strong words; works in any
language), at caption density (`minGapMs` 1200, `everyMs` 2500); the words of
strength-2 and strength-3 moments become emphasis, plus every number /
currency / percent word. `reset=true` also replaces the words marked by hand.
In the editor: Captions tab → Highlight = Key words; right-click words in the
Transcript tab → "Emphasize in captions".

`boxRotation` (0-15°) tilts highlight boxes, emphasis boxes and the group
background box; each box alternates direction with a small per-word jitter
seeded by the word id (repeatable renders). Any caption line wider than 90% of
the frame (a long merged word, a big emphasis scale, a tilted box) shrinks to
fit instead of running off the edges. Stacked (`glowStack`) captions ignore
`highlightMode`.

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

PandaStudio generates YouTube thumbnails on **the user's own image connector**: Replicate's `openai/gpt-image-2` when Replicate is connected, otherwise Higgsfield's GPT Image 2.5 — PandaStudio never pays for or proxies these calls.

Requires Replicate or Higgsfield connected in **Settings → Integrations →
Connectors** (a sign-in the user does in the app, never via CLI). If a verb
returns "No image connector is connected…", tell the user rather than looping,
and offer the fallback: they give you an image of their own and you set it with
`export.set-thumbnail`.

**Verbs:**
- `export.generate-thumbnail --id=$EID` — GPT Image at 3:2, cropped to 1280×720. Two ways to drive it:
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

