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

**Ducking music under the voice** — `project.set-audio-ducking` lowers a music
overlay automatically while someone speaks and brings it back up in the
pauses. The voice is the main track's transcript words plus any transcribed
voiceover (default), or detected speech (`--source=energy`, for footage
without a transcript). Speech is measured once per clip and stored on it; with
a transcript it also trims each word to where it is actually voiced
(speech-to-text stretches a sentence's last word over the pause after it). The
ramp down (`attackMs`) finishes as the voice starts, so the first syllable is
already clear; pauses shorter than attack + release + 400 ms stay ducked, so
the music only surfaces in real breaks (no pumping between phrases).
Deleted words and mute regions don't duck (nothing is heard there).

```bash
# Music bed ducked 12 dB under the voice (defaults: attack 150, release 400)
pandastudio project.set-audio-ducking --id=$ID --regionId=audio-1 --json
# Deeper, slower duck
pandastudio project.set-audio-ducking --id=$ID --regionId=audio-1 \
  --amountDb=18 --attackMs=250 --releaseMs=800 --json
# No transcript? Duck under detected speech instead
pandastudio project.set-audio-ducking --id=$ID --regionId=audio-1 --source=energy --json
# Off (keeps the settings) / remove entirely
pandastudio project.set-audio-ducking --id=$ID --regionId=audio-1 --enabled=false --json
pandastudio project.set-audio-ducking --id=$ID --regionId=audio-1 --remove=true --json
# Or duck from the start when adding the music
pandastudio project.add-audio --id=$ID --audioPath=$MUSIC --volume=0.3 --ducking=true --json
```

Set the music `volume` for the pauses (e.g. 0.25-0.35); ducking takes it
`amountDb` lower under speech. Stored as `audioOverlays[].ducking`
(`{enabled, amountDb, attackMs, releaseMs, source}`).

### Volume keyframes (automation over time)

Any main-track clip (`--target=clip`) or audio overlay (`--target=audio`:
music, voiceover, SFX) can carry a VOLUME curve: keyframes of absolute volume
0-2 (1 = original level) with the shared easing library between them. While a
track has keyframes they replace its static volume; fades, mute regions and
ducking still multiply on top. Preview and export play the same curve.

- **Clip keyframe times are the clip's SOURCE ms** (the transcript's clock), so
  a keyframe stays on its sound when words are deleted or a speed region is
  added. Map an edited time first with `timeline.edited-to-source`.
- **Audio keyframe times are ms from the overlay's start**, so moving the
  overlay moves its curve.

```bash
# Music: swell in over the intro, sit under the talk, rise for the outro
pandastudio project.set-volume-keyframes --id=$ID --target=audio --regionId=audio-1 \
  --keyframes='[{"timeMs":0,"volume":0.6},{"timeMs":4000,"volume":0.15,"easing":"linear"},{"timeMs":52000,"volume":0.15},{"timeMs":56000,"volume":0.6}]' --json

# One keyframe at a time (returns keyframeId); same time = change it
pandastudio project.add-volume-keyframe --id=$ID --target=audio --regionId=audio-1 \
  --timeMs=30000 --volume=0.05 --easing=ease-in-out --json
pandastudio project.remove-volume-keyframe --id=$ID --target=audio --regionId=audio-1 \
  --keyframeId=vk-3 --json

# Clip audio: dip a loud laugh (clip SOURCE ms 61200-62400) to 30%
pandastudio project.set-volume-keyframes --id=$ID --target=clip --clipIndex=0 \
  --keyframes='[{"timeMs":61000,"volume":1,"easing":"linear"},{"timeMs":61200,"volume":0.3,"easing":"hold"},{"timeMs":62400,"volume":1}]' --json

# Clear the curve (back to the static volume)
pandastudio project.set-volume-keyframes --id=$ID --target=audio --regionId=audio-1 --keyframes='[]' --json
```

**Media overlay videos' own sound** (a b-roll or screen clip placed as an
overlay, unmuted with `project.update-region --regionType=overlay --regionId=<id> --muted=false`) takes the same
automation with `--target=overlay --regionId=<mediaOverlayRegions id>`:
volume keyframes (ms from the overlay start) and
`project.set-audio-ducking --target=overlay --regionId=...` to sit it under
the narration. Mute regions still silence it.

Fields: `timeMs` (required), `volume` (required, 0-2), `easing` (linear | hold
| ease-in | ease-out | ease-in-out | back | elastic | bezier |
ease-in-out-cubic | smooth-out | smooth-pan; default ease-in-out), `bezier`
(with easing bezier). Unknown fields, out-of-range values, two keyframes at
one time, and times past the track's end are rejected. Stored as
`volumeKeyframes` on the clip / audio overlay. For a simple fade at the
overlay's start/end, `fadeIn` / `fadeOut` are still the shortest path; for a
whole-clip level, `project.set-clip-volume`.

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

### Composed soundtracks: music + SFX locked to the edit

For promos, launch films, intros and motion pieces, compose the soundtrack
instead of choosing one: `media.compose-soundtrack` renders music, beats and
effects from a score written on the edit's own cues, so every hit lands on its
moment and cuts fall on beats. Local, no key, ~1 s per 15 s. Full method,
format and recipes: [soundtrack.md](soundtrack.md). The library and the
generators below remain the right tools for long beds under speech and for
specific real-world sounds.

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

### Sound design: the sound map

Product demos, launch films and UI walkthroughs are carried by sound effects
(plus a voiceover or a music bed): a click on every press, a pop as a card
lands, a key per typed character, a whoosh on a scene change. ~190 bundled
sounds (`asset.list-sounds`, use as `bundled:sound/<id>` or the bare id):

```bash
pandastudio asset.list-sounds --summary=true --json          # categories + variant groups
pandastudio asset.list-sounds --category=notification --json # ui notification motion digital impact typing outcome ambience
pandastudio asset.list-sounds --tag=whoosh --json            # or --group=ui-click-soft, --mood=positive, --query=pop
```

Ids are `category-name-variant`; a `variantGroup` (e.g. `ui-click-soft` = 1, 2,
3) holds interchangeable takes. Every new sound starts at its onset (0 ms, so a
cue at the action frame is in sync) and is loudness-normalised per category
with true peak at or below -1 dBTP, so one volume means the same thing across a
category. The 25 older ids (`mouse-click`, `swoosh-fast`, `keyboard-key-*`,
`logo-impact`, `room-tone`...) still work at their original, generally hotter,
levels; prefer the new ones.

**Which action gets which cue** (volumes assume a music bed or VO; go 0.1-0.2
higher with neither):

| On screen | Cue (id or group) | Volume | How often |
|---|---|---|---|
| Cursor clicks a button / link | `ui-click-soft-*`, `ui-mouse-click-*` | 0.4-0.6 | every visible click |
| Primary CTA press, hard confirm | `ui-button-press-*`, `ui-click-hard-*` | 0.5-0.7 | per press |
| Deliberate hover | `ui-hover-*` | 0.15-0.25 | at most one per 500 ms |
| Toggle / switch | `ui-toggle-on-*`, `ui-toggle-off-*` | 0.4-0.5 | per toggle |
| Chip, tag, checkbox, list item checks off | `ui-tap-*`, `ui-select-*` | 0.3-0.5 | per item, rotate variants |
| Panel, menu, modal opens / closes | `ui-open`, `ui-close`, `ui-maximize`, `ui-minimize` | 0.4 | per open |
| List scrolls | `ui-scroll-*` (cut with `durationMs`) | 0.2-0.3 | per gesture |
| Drag lands, item snaps into place | `ui-drop`, `impact-wood-knock` | 0.4 | per drop |
| A character is typed | `type-key-soft-*` (laptop), `type-key-mech-*` (mechanical); `type-space-*`, `type-enter-*` on send | 0.25-0.4 | one per character at its real time |
| A line appears typed all at once | `type-burst-*` cut to the typing span | 0.3-0.4 | per line |
| Chat message arrives / is sent | `notif-message-in-*` / `notif-message-out-*` | 0.5 | per message |
| Card, bubble, tooltip pops in | `notif-pop-small-*` (UI), `-medium-*`, `notif-pop-large` (hero) | 0.3-0.5 | per card; in a 4+ stagger only first, last or every other |
| Notification / toast | `notif-chime-*`, `notif-ding` | 0.5-0.6 | at most ~one per 3 s |
| Badge count goes up | `notif-badge-ping-*` | 0.3-0.4 | per increment |
| Card / panel slides in or out | `motion-slide-in-*`, `motion-slide-out`, `motion-swipe-*` | 0.4-0.6 | per slide |
| Card flips to reveal | `motion-card-flip-*` | 0.5 | per flip |
| Scene change, camera flies to a new area | `motion-whoosh-medium-*` (quick cut `-short-*`, big move `-long-*`, `motion-whip-pan`) | 0.5-0.7 | ~one per 6-10 s, never two within 1 s |
| Build into a hit or reveal | `motion-whoosh-reverse-*` ending on the hit | 0.5 | a few per film |
| Element rises / drops away | `motion-swoosh-up`, `motion-swoosh-down` | 0.4-0.5 | sparingly |
| Big camera push-in / pull-out | `motion-zoom-in-swell`, `motion-zoom-out-swell` | 0.3-0.5 | big zooms only |
| Loading, AI thinking, data moving | `digital-processing`, `digital-data-transfer`, `digital-scan-*` (cut to the spinner) | 0.3-0.4 | per wait |
| Number ticks, data point, tech accent | `digital-blip-*`, `digital-chirp-*`, `digital-beep-*` | 0.3-0.5 | light |
| Glitch transition | `digital-glitch-*` | 0.4-0.6 | two per film at most |
| Success, checkmark | `outcome-success-*`, `digital-success-beep` | 0.5-0.6 | per success |
| Error, rejected | `outcome-error-*`, `digital-error-beep-*` | 0.5 | per error |
| Payment, revenue, a sale | `outcome-cash-*`, `outcome-coins` | 0.5-0.6 | per moment |
| Screenshot, photo | `outcome-camera-shutter-*` | 0.5 | per shot |
| Celebration, achievement | `outcome-confetti-pop` + `outcome-sparkle-*` | 0.5 | once or twice |
| Underline, circle, scribble drawn | `outcome-marker-underline`, `outcome-marker-circle`, `outcome-pencil-scribble-*` (cut to the stroke) | 0.4-0.5 | per stroke |
| Document, page, approval | `outcome-paper-page-turn-*`, `outcome-paper-slide`, `outcome-stamp` | 0.4-0.5 | per page |
| A title or statement lands | `impact-soft-*` (subtle), `impact-deep-*` | 0.4-0.6 | a few per film |
| Logo reveal / end card | `impact-riser-short-*` / `-long-*` ending on the logo frame, then `impact-logo-sting-*` or `impact-boom-*` / `impact-sub-drop`; `impact-downlifter-*` after | riser 0.4-0.5, hit 0.6-0.8 | open and close |
| Under everything | `ambience-room-tone-*` or `ambience-air-1` (normalised to -38 LUFS), looped | 0.6-1.0 | one bed, so there's no dead silence |

**Restraint:**
- Sound what the viewer is watching, not every moving pixel: no cue for
  parallax drift, easing overshoot, blur, a floating camera or background loops.
  A card landing with its shadow and chip animating gets ONE cue.
- Density: about 3-4 cues a second at most in a busy stretch (typing is the
  exception: keys at their real cadence, quieter).
- Vary: rotate a variant group (`ui-click-soft-1`, `-2`, `-3`...) and nudge
  repeats by ±0.05-0.1 volume. `project.add-sound-cues` warns when one sound
  plays 4+ times in a row.
- Whooshes are the most common mistake: about one per 6-10 s, never two
  within a second, not on every camera move or blur cut.
- Duck the music under dense clusters (a typing run, a montage): dip the bed
  ~4-6 dB (e.g. 0.30 → 0.18) with `project.set-volume-keyframes --target=audio
  --regionId=<music>` and bring it back after. Keep effects under a voice: no
  cue on a VO line's key words unless it's the action being named.
- Give the big moment contrast: 300-600 ms with no cues (and the music dipped)
  before a logo sting or a reveal.

**Syncing cues to the motion:**
- The motion timeline is the truth. For HTML/GSAP scenes use the tween times
  you authored plus the scene clip's start on the project timeline
  (`atMs = sceneStartMs + tweenMs`; cue times are EDITED-timeline ms). For
  footage, check frames around each action (`motion.verify-frames`,
  `project.render-frame`).
- Transients (click, tap, key, pop, hit) start ON the contact frame: the press,
  or the end of a landing tween.
- Moves (whoosh, slide, swoosh) peak mid-move: start 50-150 ms before the move
  begins, or `atMs = moveMidMs - durationMs / 2`.
- Reverse whooshes and risers END on the hit: `atMs = hitMs - durationMs`
  (`durationMs` from `asset.list-sounds`).
- Typing: one key per character at `LF.type`'s `times`; space and enter on the
  matching characters. Staggers: the tween's stagger interval.

Place them all in one call (write the list to a file for long ones):

```bash
cat > /tmp/cues.json <<'JSON'
[
  {"sound":"motion-whoosh-medium-1","atMs":2350,"volume":0.6},
  {"sound":"notif-pop-small-1","atMs":2900,"volume":0.35},
  {"sound":"ui-click-soft-2","atMs":4120,"volume":0.5},
  {"sound":"type-key-soft-1","atMs":4600,"volume":0.3},
  {"sound":"type-key-soft-2","atMs":4680,"volume":0.3},
  {"sound":"type-enter-1","atMs":5400,"volume":0.4},
  {"sound":"impact-riser-short-1","atMs":9283,"volume":0.45},
  {"sound":"impact-logo-sting-1","atMs":12000,"volume":0.7}
]
JSON
pandastudio project.add-sound-cues --id=$ID --group=sfx --cues=@/tmp/cues.json --json
# → { overlayIds, count, cues:[{overlayId,sound,startMs,endMs}], replaced, warnings? }
# Retimed? Edit the file and run the same command: group "sfx" is replaced, not doubled.
# Clear it: --group=sfx --cues='[]'
```

Each cue is an ordinary audio overlay (the lane `project.add-audio` uses for
SFX), so `project.update-region --regionType=audio-overlay`, `remove-audio`,
ducking and volume keyframes all work on it. Pass `anchorSourceMs` only for a
cue pinned to a transcript word (a hit on a punchline); cues timed to graphics
stay free.

Nothing fitting in the bundle: `media.generate-sound-effect --prompt="…"
--durationMs=…` (one sound per call), then use its path as the cue's `sound`.
Don't generate whooshes (they often come back musical). Before calling it done,
export and check the levels: the loudest hit, a typing stretch and a voice line
(`ffmpeg -i out.mp4 -af ebur128=peak=true -f null -`).

### Loudness normalisation on export (on by default)

Every export's FINAL mix (voice + music + SFX + overlay audio) is normalised to
a standard loudness with a two-pass EBU R128 measurement; the video stream is
copied untouched.

| Setting | Target | Use for |
|---|---|---|
| `streaming` (default, `true`, `-14`) | -14 LUFS integrated, -1 dBTP true peak | YouTube, Shorts, TikTok, Instagram, Spotify video, LinkedIn |
| `podcast` (`-16`) | -16 LUFS, -1 dBTP | Apple Podcasts / podcast hosts, audio-first uploads |
| `off` (`false`) | mix left as is | only when the user mastered the audio elsewhere |

```bash
# Per project (saved; the Export dialog shows it too):
pandastudio project.set-export-settings --id=$ID --normalizeLoudness=podcast
# Per export (overrides the project setting for this one render):
pandastudio export.start --id=$ID --quality=high --normalizeLoudness=streaming --json
```

The `job.wait` result carries `loudness`: `{ preset, targetLufs, applied,
inputLufs, outputLufs, outputTruePeakDbtp, mode, reason }`. `mode: "linear"` =
one clean gain change; `"dynamic"` = gentle limiting was needed to reach the
target without clipping. When `applied` is false, `reason` says why: `no-audio`
(video-only export), `silent` (boosting would only amplify hiss), `failed`
(measurement or apply pass errored: the export still completed with the audio
as mixed; surface it), `unavailable` (media engine not installed). Tell the
user the before/after ("normalised from -23.4 to -14.0 LUFS"). Don't pre-boost
clip volumes (`project.set-clip-volume`) to make a quiet recording loud;
normalisation does that on the whole mix. Relative balance still matters: keep
music under the voice (ducking or overlay volume), then let normalisation set
the overall level.
