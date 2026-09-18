<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Audio cleanup, background audio, music, color (LUT)

### Audio cleanup (DeepFilter)

```bash
# Check clipStates first — skip if all clips are already cleaned
JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json
# → only un-cleaned clips are processed; already-cleaned clips are skipped automatically
# → each processed clip gets a sibling .cleaned.wav file; export auto-uses it
```

### Room echo / reverb (`--echo`)

DeepFilter removes steady noise (fans, hum, hiss) but NOT room reverb: in a
bare or hard-walled room every phrase still trails off. `--echo=true` adds a
second stage on the cleaned track that shortens that tail (statistical
late-reverb suppression with the room's reverb time measured from the
recording itself). Soft speech is not gated, and the speech level is kept the
same, so switching it on and off is a clean A/B.

```bash
# Clean + reduce echo in one job
JOB=$(pandastudio audio.clean --id=$ID --echo=true --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json
# → data.result.results[i]: { clipId, cleanedPath, denoisedPath, echoReducedPath,
#     rt60Ms, decayBeforeMs, decayAfterMs }

# Already cleaned? Only the fast echo stage runs (well under a second per minute of audio)
pandastudio audio.clean --id=$ID --clipId=clip-1 --echo=true --json
# Back to noise removal only (instant, the DeepFilter WAV is kept)
pandastudio audio.clean --id=$ID --clipId=clip-1 --echo=false --json
```

- Opt-in. Turn it on when the user mentions echo, reverb, a boomy/hollow/
  "bathroom" sound, or asks for a studio sound. Don't add it to every clean.
- Files: `x.cleaned.wav` (DeepFilter only, kept) and `x.cleaned.dereverb.wav`
  (+ echo reduction). `clip.cleanedAudioPath` points at whichever is active;
  `clip.cleanedAudioDenoisedPath` holds the DeepFilter one while echo is on.
- Verify from the job result: `decayAfterMs` well below `decayBeforeMs`
  (typically ~50% shorter). `rt60Ms` above ~600 means a very live room: tell
  the user it helps but can't fully fix it, and suggest soft furnishings or a
  closer mic for the next take.
- `project.read` → `clipStates[i].echoReduced` / `roomRt60Ms` show the state.
- In the app: Audio panel → Clean Audio → **Reduce echo** switch.

### Per-clip volume (balance loudness across clips)

Each main-track clip carries a linear audio **gain**: `1` = original level (the
default), `0` = silent, `2` = +6 dB. Use it to balance recordings that were
captured at different levels — e.g. a quiet second take next to a loud first
one — without touching the audio files. Applied in preview AND both export
paths, so what you hear is what exports.

```bash
# Boost clip 2 and clip 3 (0-based indices → clipIndex 1 and 2) to +6 dB.
pandastudio project.set-clip-volume --id=$ID --clipIndex=1 --volume=1.5 --json
pandastudio project.set-clip-volume --id=$ID --clipIndex=2 --volume=1.5 --json

# Or address a clip by id (from project.read → clipStates[].clipId):
pandastudio project.set-clip-volume --id=$ID --clipId=clip-2 --volume=0.6 --json

# Silence a clip's audio entirely (keeps the video):
pandastudio project.set-clip-volume --id=$ID --clipIndex=0 --volume=0 --json

# Reset a clip back to its original level:
pandastudio project.set-clip-volume --id=$ID --clipIndex=1 --volume=1 --json
```

- `volume` is clamped to `[0, 2]`. `volume=1` clears the override.
- Identify the clip by `clipIndex` (0-based, matching `project.read`'s
  `clipStates` order) OR `clipId`. `project.read` surfaces the current gain as
  `clipStates[i].volume` — but only when it's off the default `1`.
- This is continuous per-clip gain, distinct from muting an overlay. There is
  no shell/ffmpeg workaround needed — never write a re-leveled WAV to disk for
  this; the verb does it non-destructively.

### Adding background audio to any project

Audio overlays are **first-class timeline regions** — they can be dragged,
trimmed, and have an in-point into the source file, just like video clips.
Waveform peaks are extracted automatically when you add an overlay so the
timeline UI can render a real waveform.

```bash
# 1) Simple: play the full file from t=0 at 60% volume
pandastudio project.add-audio --id=$ID \
  --audioPath=/path/to/music.mp3 --volume=0.6 --json
# → { overlayId: "audio-1" } — durations are probed automatically

# 2) Position on the timeline: play from 2s–17s of the edited timeline
pandastudio project.add-audio --id=$ID \
  --audioPath=/path/to/music.mp3 \
  --startMs=2000 --endMs=17000 --volume=0.55 --json

# 3) Trim into the source: start playback at the 4s mark of the source file,
#    play 13 seconds of it, positioned at 2s on the edited timeline.
pandastudio project.add-audio --id=$ID \
  --audioPath=/path/to/music.mp3 \
  --startMs=2000 --endMs=15000 --sourceStartMs=4000 \
  --volume=0.55 --json

# 4) Change timing later (drag + trim without removing)
pandastudio project.update-region --id=$ID \
  --regionType=audio-overlay --regionId=audio-1 \
  --startMs=5000 --endMs=20000 --sourceStartMs=2000 --volume=0.7 --json

# 5) Remove (a transcribed voiceover's words go with it → { transcriptWordsRemoved })
pandastudio project.remove-audio --id=$ID --overlayId=audio-1 --json
# or: pandastudio project.remove-region --regionType=audio-overlay --regionId=audio-1
```

**Arg precedence for duration** (matches the primitive):

1. Explicit `--endMs` — always wins
2. Legacy `--maxDurationMs` — `endMs = startMs + maxDurationMs`
3. `--durationMs` fallback
4. Probed real file duration (when nothing else is specified)

**Fades** — `--fadeIn` / `--fadeOut` (ms) add a ramp at the overlay's audible
start / end; the export mixer applies them (`afade`). Fade-out needs a bounded
overlay (`endMs`/`maxDurationMs` set) so the end time is known; it's ignored on an
uncapped full-length overlay. Use a fade-out on the final music region so the bed
doesn't cut off hard, and short (~500ms) fades at a loop seam to hide the join.

**Ducking music under voiceover** — use two overlays: the VO at `volume=1.0`
and the music at `volume=0.2`. Both export through the same FFmpeg `amix`, so
an explicit ducking automation isn't needed for simple cases.

Audio overlays are exported automatically — you don't need to do anything
extra in `export.start`.

### Bundled background music — browse and add in one step

PandaStudio ships with royalty-free background music tracks you can drop into
any project without sourcing external files. Every track carries **`intents`**
(agent-routing hints) and **`recommendedFor`** (destinations) — match on
`intents` first when picking a track; use `mood` and `category` as tiebreakers.

```bash
# 1. List all bundled tracks — each has id, title, category, mood, durationMs,
#    intents[], recommendedFor[], absolutePath
pandastudio asset.list-music --json | jq '.data.tracks'
```

**Current library (v4):**

| id | category | mood | intents | recommendedFor |
|---|---|---|---|---|
| `driving-promo` | electronic | energetic | product_video, promo, intro, outro, product_reveal, kinetic_text, motion_graphics | youtube-long, shorts, linkedin |
| `corporate-underscore` | corporate | neutral | generic, background, under_voiceover, explainer, tutorial, saas_walkthrough, default | youtube-long, linkedin, loom |
| `uplifting-inspirational` | cinematic | uplifting | promo, intro, brand_story, testimonial, explainer, motion_graphics | youtube-long, linkedin, shorts |
| `chill-lofi` | lofi | calm | vlog, lifestyle, day_in_life, ambient_underscore, background | youtube-long, shorts |
| `cinematic-build` | cinematic | dramatic | intro, cinematic, reveal, product_reveal, ambient_underscore, motion_graphics | youtube-long, shorts |
| `bright-playful` | pop | happy | promo, vlog, kinetic_text, social, intro | shorts, youtube-long, linkedin |
| `launch-pulse` | electronic | confident | product_video, promo, product_launch, motion_graphics, product_reveal, saas_walkthrough, no_voiceover | youtube-long, linkedin, shorts |
| `quiet-launch` | electronic | calm | product_video, promo, product_launch, motion_graphics, explainer, under_voiceover, no_voiceover | youtube-long, linkedin, loom |

**Intent → track selection (use unless the user specifies a track):**

- Product video / product demo / product reveal / promo → `driving-promo`
- Kinetic text / motion graphics / energetic YouTube **intro** or **outro** → `driving-promo` (trim to length)
- Big cinematic reveal / problem framing / dramatic intro that builds → `cinematic-build`
- Brand story / testimonial / motivational feature highlight → `uplifting-inspirational`
- Tech review / tutorial / explainer / SaaS walkthrough / under-voiceover bed → `corporate-underscore`
- Vlog / day-in-life / lifestyle / behind-the-scenes → `chill-lofi`
- Fun / lighthearted promo / social clip → `bright-playful`
- **Motion-graphics product launch film with NO voiceover** → `launch-pulse` (the bed carries the cut) or `quiet-launch` when the film is quieter and more considered. Both run ~33s, so loop or repeat them for a longer film, and mix so the finished export lands near -18 LUFS with peaks at or under -1 dBFS.
- Anything else / don't-know / "just add music" → `corporate-underscore` (neutral default)
- **LinkedIn / Loom:** prefer `corporate-underscore` (neutral, won't distract from message) — only use `driving-promo` or `bright-playful` when the brief is explicitly promo/reveal/fun

```bash
# 2. Pick by intent and add to project (agent-friendly filter)
MUSIC=$(pandastudio asset.list-music --json \
  | jq -r '.data.tracks[] | select(.intents | index("product_video")) | .absolutePath' \
  | head -1)

pandastudio project.add-audio --id=$ID \
  --audioPath="$MUSIC" --volume=0.3 --fadeIn=500 --fadeOut=500 --json
```

Among tracks that match an intent, rotate between variants (`-a` and `-b`) or
pick by `durationMs` closest to what the project needs. Never pick by filename
— always query `asset.list-music` so new tracks get picked up automatically.

### Custom music — generate an original track (Lyria-2 or MusicGen)

When the bundled library doesn't cover what the user wants (a specific genre,
mood, or instrument combination), generate an original instrumental track with
`media.generate-music` (Replicate / Google Lyria-2). The bundled library is the
faster default for common moods — reach for generation only when the user asks
for something specific/custom. Requires Replicate to be connected (Settings →
Integrations → Connectors). Output is instrumental only, ~30s, 48kHz stereo — loop it for
longer videos.

```bash
# Describe genre, mood, instruments, tempo, use-case. Returns { audioPath, durationMs }.
RES=$(pandastudio media.generate-music \
  --prompt="upbeat lo-fi hip hop with mellow piano and soft vinyl crackle, relaxed, for a coding montage" \
  --negativePrompt="vocals, harsh, distortion" --json)
MUSIC=$(echo "$RES" | jq -r '.data.audioPath')
DUR=$(echo "$RES" | jq -r '.data.durationMs')

# Place it as background music, sized to the clip (loops/repeats for longer videos).
pandastudio project.add-audio --id=$ID \
  --audioPath="$MUSIC" --volume=0.3 --fadeIn=500 --fadeOut=500 --json
```

Canonical custom-music loop: `media.generate-music` → `project.add-audio`. Pass a
`seed` for reproducible results.

**Matching a track you already have (MusicGen).** `model=musicgen` takes an exact
length and, with `reference`, follows the melody, tempo and feel of a track you
point it at — the way to match a brand track, the bed from an earlier cut, or temp
music the user supplied, instead of describing it in words:

```bash
pandastudio media.generate-music --model=musicgen \
  --prompt="restrained modern tech underscore, soft pulsing synth, minimal percussion, no vocals" \
  --reference="/path/to/their-brand-track.mp3" --referenceStartMs=12000 \
  --referenceDurationMs=15000 --durationSec=45 --json
```

- `reference` can be audio OR a video; the audio is trimmed
  (`referenceStartMs` / `referenceDurationMs`, default 15s, max 30s) and sent with the
  request. Point it only at audio the user has the rights to — their own track, a
  bundled one, or something they licensed. To chase the feel of a video you admire,
  describe its character in `prompt` instead (tempo, instruments, how busy it is)
  rather than feeding in its soundtrack.
- `continuation=true` continues the reference from where it was read instead of
  re-interpreting it — useful to extend a bed you already like.
- `durationSec` means no looping: ask for the film's length.
- MusicGen's weights are published under CC-BY-NC-4.0, so treat its output as
  non-commercial: fine for a draft, an internal cut or a temp bed, not for a track the
  user sells, ships inside a product or runs as an ad. For anything commercial use
  Lyria (the default) or the bundled library, and say why when it comes up.

### Color correction (fix the footage first)

A LUT preset is a creative *look*. It can't rescue footage shot with a flat or log
picture profile (mirrorless / cinema cameras), which looks grey and washed out:
correct that first, then grade.

```bash
# One-click correction for flat camera footage (contrast +0.5, saturation +0.55,
# brightness +0.06, warmth +0.2)
pandastudio project.set-clip-color --id=$ID --clipId="$CLIP_ID" --preset=flat-footage --json
# → { path, revision, clipId, colorCorrection }

# Fine-tune single controls (each -1..1, 0 = unchanged). Merges with what's set.
pandastudio project.set-clip-color --id=$ID --clipId="$CLIP_ID" --warmth=0.35 --json

# Then add the look on top, usually at a lower intensity
pandastudio project.set-clip-lut --id=$ID --clipId="$CLIP_ID" --lutPreset=warmSunset --lutIntensity=0.6 --json

# Clear the correction
pandastudio project.set-clip-color --id=$ID --clipId="$CLIP_ID" --reset=true --json
```

**Camera and screen are graded separately (v1.89.4+).** `set-clip-color` and
`set-clip-lut` take `--target=screen` (default) or `--target=camera`. A camera
grade only touches the camera layer (card, side panel, full-frame beats), so a
warm or black-and-white presenter never tints the screen recording:

```bash
pandastudio project.set-clip-color --id=$ID --clipId="$CLIP_ID" --target=camera --preset=flat-footage --json
pandastudio project.set-clip-lut --id=$ID --clipId="$CLIP_ID" --target=camera --lutPreset=warmSunset --lutIntensity=0.5 --json
```

In the editor the grade panel has a Screen / Camera switch.

Order is fixed: correction, then LUT, in both preview and export. When footage
looks grey or flat in a frame check, reach for `--preset=flat-footage` before
picking a LUT.

### Color grading clips (LUT presets)

Every clip can have a non-destructive cinematic color grade applied. Grades are
applied both in the preview (CSS filter) and baked into the final export (FFmpeg).

```bash
# 1. List available LUT presets
pandastudio asset.list-luts --json | jq '.data.presets'

# Available presets:
# none | cinematicTealOrange | cinematicShadowBlue | filmNoir | vintageKodak
# modernVibrant | moodyDark | warmSunset | coolNordic | bleachBypass
# vintagePolaroid | naturalEnhanced

# 2. Read the project to get clip IDs
CLIP_ID=$(pandastudio project.read --id=$ID --json \
  | jq -r '.data.project.mainTrack.clips[0].id')

# 3. Apply a LUT to a clip
pandastudio project.set-clip-lut \
  --id=$ID \
  --clipId="$CLIP_ID" \
  --lutPreset=cinematicTealOrange \
  --json
# → { path, revision, clipId, lutPreset, lutIntensity }

# 4. Optional: dial in intensity (0.0 = no grade, 1.0 = full, default 1.0)
pandastudio project.set-clip-lut \
  --id=$ID \
  --clipId="$CLIP_ID" \
  --lutPreset=filmNoir \
  --lutIntensity=0.7 \
  --json

# 5. Remove the grade (reset to none)
pandastudio project.set-clip-lut \
  --id=$ID \
  --clipId="$CLIP_ID" \
  --lutPreset=none \
  --json
```

**LUT preset → style heuristic** (apply by default for relevant briefs):

| Style brief | Preset |
|---|---|
| "Cinematic" / "YouTube cinematic look" | `cinematicTealOrange` |
| "Dark / moody" | `moodyDark` or `cinematicShadowBlue` |
| "Vintage / retro / film" | `vintageKodak` or `vintagePolaroid` |
| "Black and white / noir" | `filmNoir` |
| "Vibrant / punchy" | `modernVibrant` |
| "Warm / golden hour" | `warmSunset` |
| "Cool / Nordic / clean" | `coolNordic` |
| "Faded / film" | `bleachBypass` |
| "Natural / subtle enhancement" | `naturalEnhanced` |

LUT is applied **per clip** — multi-clip projects can have different grades per
clip. The grade is non-destructive: `lutPreset=none` removes it instantly with
no re-encode needed.

**Full cinematic workflow** (this recipe is for a brief that EXPLICITLY asked
for a graded, music-backed cinematic piece — e.g. "make a cinematic short with
a calm music bed". The grade + music steps below are part of *that* request.
For a plain "edit my video", do NOT copy the music step — see the editorial
rule "**Background music** … Never add a music track to 'edit my video'"):

```bash
# Create project, add clip, grade it, preview. (Music step is OPT-IN — only
# because this brief asked for it; omit it for a default edit.)
P=$(pandastudio project.new --name="Cinematic Short" \
  --withMedia='["/path/footage.mp4"]' --json)
ID=$(echo "$P" | jq -r '.data.id')
CLIP=$(echo "$P" | jq -r '.data.project.mainTrack.clips[0].id')

# Apply cinematic teal-orange grade
pandastudio project.set-clip-lut \
  --id=$ID --clipId="$CLIP" \
  --lutPreset=cinematicTealOrange --lutIntensity=0.85 --json

# Add bundled lofi music — ONLY because the user asked for a music bed.
# Skip this entirely for an "edit my video" / "polish" request.
MUSIC=$(pandastudio asset.list-music --json \
  | jq -r '.data.tracks[] | select(.mood == "calm") | .absolutePath' | head -1)
pandastudio project.add-audio --id=$ID --audioPath="$MUSIC" --volume=0.5 --json

# Preview before export
pandastudio preview.show --id=$ID
```

### Prepending a title card (or any single clip) before the footage

`add-motion-graphic` places an overlay on top of the canvas — it doesn't insert a discrete clip with its own audio. To make a title card that **plays before the footage** (no video beneath it), insert the motion-graphic MP4 as a clip at index 0.

**Single rendered scene → add directly. Multiple scenes → concat first, then add.**

```bash
# Single title card — fastest path is the `creator-card` template
# (motion.generate). This example shows the custom-HTML fallback for a
# bespoke card; see reference/motion-philosophy.md §7.
TITLE=$(pandastudio motion.render-html \
  --htmlPath=/tmp/title-card.html \
  --durationMs=3000 --aspectRatio=16:9 --json | jq -r '.data.jobId')
TITLE_PATH=$(pandastudio job.wait --id="$TITLE" --json | jq -r '.data.job.result.outputPath')

pandastudio project.add-clip --id=$ID --media="$TITLE_PATH" --atIndex=0 --json

# Multiple motion-graphic scenes (intro + outro) → concat first
OUTRO_JOB=$(pandastudio motion.render-html --htmlPath=/tmp/outro.html --durationMs=2000 \
  --outputName=scene-outro --json | jq -r '.data.jobId')
OUTRO_PATH=$(pandastudio job.wait --id="$OUTRO_JOB" --json | jq -r '.data.job.result.outputPath')

MERGED=$(pandastudio motion.concat \
  --clips="[\"$TITLE_PATH\",\"$OUTRO_PATH\"]" \
  --outputName=bookend-merged --json | jq -r '.data.outputPath')

pandastudio project.add-clip --id=$ID --media="$MERGED" --atIndex=0 --json
```

Omit `--atIndex` (or use a high number) to append at the end instead.
