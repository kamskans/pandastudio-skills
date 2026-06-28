<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Narration (TTS) + B-roll image generation

## Narration / voiceover (Replicate TTS)

Generate a voiceover for promos and explainers using the user's own Replicate
key — no microphone needed. Three models; ElevenLabs v3 is the default and most
expressive.

### The verb

```bash
RES=$(pandastudio media.generate-narration \
  --text="Meet PandaStudio. [excited] Edit your videos just by talking to Claude." \
  --model=elevenlabs-v3 \
  --json)
AUDIO=$(echo "$RES" | jq -r '.data.audioPath')
DUR=$(echo "$RES" | jq -r '.data.durationMs')

# Place it on the timeline (startMs positions it; pass the returned durationMs)
pandastudio project.add-audio --id="$PID" --path="$AUDIO" --startMs=0 --durationMs=$DUR
```

- **Models:** `elevenlabs-v3` (default — embed inline delivery tags in the text
  like `[excited]`, `[whispers]`, `[sighs]`), `gemini-flash-tts` (30 voices,
  strong multilingual; `--style` sets the tone), `minimax-turbo` (fast;
  `--style` maps to an emotion, `--speed` 0.5-2).
- **Voices:** omit `--voice` for the model default. Examples — gemini:
  `Kore` / `Puck` / `Charon`; minimax: `Friendly_Person` / `Wise_Woman`.
- **Canonical promo loop:** write the script → `media.generate-narration` →
  `project.add-audio` at `startMs` with the returned `durationMs` → time your
  motion graphics / B-roll to the voice.
- **Requires a Replicate API key** (Settings → Integrations — same key as image
  gen). If absent, the verb returns a clear error; surface it, don't loop.

## B-roll generation (Replicate gpt-image-2)

PandaStudio ships with a project-level image-gen verb backed by the
user's own Replicate API key. Use it to author B-roll, concept
stills, mood-board frames, or reference imagery for explainer beats —
without leaving the editor.

### The verb

```bash
pandastudio media.generate-image \
  --prompt="cinematic 35mm photo, sunlit modern desk with vintage typewriter, warm tones, shallow depth of field" \
  --aspectRatio=3:2 \
  --quality=medium
# → { imagePath: "/Users/.../generated-images/<ts>-cinematic-35mm.webp", ... }
```

**Aspect ratios are gpt-image-2 native:** `1:1` / `3:2` / `2:3`. For
a 16:9 video, generate `3:2` and crop in the wrap. For 9:16, generate
`2:3`. Don't ask the model for `16:9` — it doesn't exist in this API.

**Requires a Replicate API key.** If the user hasn't connected one in
Settings → Integrations, the verb returns an error explaining how to
set it up. Don't loop on this; surface to the user.

### ⚠ Don't drop a flat photo straight into the timeline

A still image cut between A-roll always reads as amateur. Per
motion-philosophy Law #4 (*"Camera never sleeps"*), every B-roll
beat needs at least one micro-motion layer. The two-step recipe:

```bash
# 1. Generate the still
RES=$(pandastudio media.generate-image \
  --prompt="<visual prompt>" --aspectRatio=3:2 --json)
IMG=$(echo "$RES" | jq -r '.data.imagePath')

# 2. Wrap it in a Ken-Burns + vignette HTML and render to WebM
#    (use the canonical B-roll shell below)
JOB=$(pandastudio motion.render-html \
  --html="$BROLL_HTML" \
  --aspectRatio=16:9 \
  --durationMs=4000 \
  --json | jq -r '.data.jobId')
WEBM=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# 3. Drop into the project at the matching transcript moment
#    (--file MUST be quoted — the path contains a space, see the
#     "ALWAYS QUOTE THE --file PATH" warning above)
pandastudio project.add-motion-graphic \
  --id="$PID" --file="$WEBM" --atMs=$BEAT_MS --durationMs=4000
```

### Canonical B-roll HTML shell

The Ken-Burns + vignette + grain shell that wraps a generated still (so it
never reads as a flat photo) lives in [`reference/examples.md`](reference/examples.md)
under "B-roll Ken-Burns shell" — copy it, drop the image path into `<<IMG_PATH>>`,
render with `motion.render-html --durationMs=…`.

### When to reach for B-roll

| Beat | B-roll move |
|---|---|
| Host says "imagine X" / "picture this" | Concept still that visualises X — 3-4s, host audio under |
| Host names a product / tool / place | Product still or location photo — 2-3s, single zoom |
| Host says "studies show" / "research shows" | Abstract data-vis aesthetic still (charts, lines on dark bg) — 3s |
| Mid-explainer pause ("...") | Pattern break still — texture, atmosphere, no humans — 1.5-2s |

### Density rules

- **Max 1 generated B-roll per 8 seconds of host runtime.** More than that and the host disappears from their own video. The viewer wants the *person*, B-roll is seasoning.
- **Minimum 1.5s on screen.** Anything shorter feels like a glitch.
- **Pair with a clip-transform-region for camera-only Mode A/C.** B-roll plays in one half, host stays visible in the other half — see `reference/video-authoring.md` §5b. Never let B-roll cover the host's face.
- **Quality `low` for first-pass exploration**, `medium` for the keeper. Don't burn `high` until the prompt is locked.
- **Reuse `referenceImagePath`** to keep visual continuity across multiple B-roll stills in the same video — pass the first generation as the reference for subsequent ones.

### What this verb is NOT for

- **YouTube thumbnails** — use `export.generate-thumbnail` (it's tied to an export entry and tracks iteration history).
- **Logos / brand marks / typography** — image-gen wrecks fine type. Author those as HTML in `motion.render-html`.
- **Anything with text overlays** — gpt-image-2's text rendering is unreliable. Generate a clean photo, add text via the motion-graphic wrap.

