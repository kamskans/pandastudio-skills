<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Narration (TTS) + B-roll image generation

## Narration / voiceover

Generate a voiceover for promos and explainers — no microphone needed. Two
engines, and **local is the default so it works out of the box**:

- **Local Kokoro (default).** Kokoro-82M runs fully **on-device** (English, US +
  UK accents) via the bundled `kokoro-helper` sidecar. No API key, no cloud, no
  per-use cost. The ~330 MB model auto-downloads on first use (like the Whisper
  model does); pre-warm it with `system.download-kokoro-model` or check
  `system.is-kokoro-model-downloaded`.
- **Cloud Replicate (opt-in).** Set `--model` to a Replicate model when you want
  more voices/languages or the most expressive delivery. Needs Replicate
  connected (Settings → Integrations → Connectors).

### The verb

```bash
# Local (default) — omit --model, or pass --model=kokoro-local. Voices:
# af_heart (default) / af_bella / am_michael / bf_emma / bm_george / ...
# (a*=American, b*=British; f=female, m=male). --speed 0.5-2.
RES=$(pandastudio media.generate-narration \
  --text="Meet PandaStudio. Edit your videos just by talking to Claude." \
  --voice=af_heart \
  --json)
AUDIO=$(echo "$RES" | jq -r '.data.audioPath')
DUR=$(echo "$RES" | jq -r '.data.durationMs')

# Place it on the timeline (startMs positions it; pass the returned durationMs)
pandastudio project.add-audio --id="$PID" --audioPath="$AUDIO" --startMs=0 --durationMs=$DUR --transcribe=true

# Cloud — pass a Replicate model explicitly:
pandastudio media.generate-narration \
  --text="Meet PandaStudio. [excited] Edit by talking to Claude." \
  --model=elevenlabs-v3 --json
```

- **Engine selection:** omit `--model` to use the workspace default (local
  Kokoro). `--model=kokoro-local` forces on-device; a Replicate id
  (`elevenlabs-v3` | `gemini-flash-tts` | `minimax-turbo`) forces cloud. Set the
  workspace default with `system.set-narration-engine` (`local-kokoro` |
  `replicate`); read it with `system.get-narration-engine`.
- **Cloud models:** `elevenlabs-v3` (embed inline tags like `[excited]`,
  `[whispers]`), `gemini-flash-tts` (30 voices, multilingual; `--style` sets the
  tone), `minimax-turbo` (fast; `--style` maps to an emotion, `--speed` 0.5-2).
  Local Kokoro is English-only and ignores `--language`/inline tags.
- **Names and brands in narration.** The local Kokoro voice spells any word it doesn't know letter by letter. It now says joined words made of words it knows as separate words (`PandaStudio` -> "Panda Studio", `writepanda` -> "write panda"), and leaves short acronyms (AI, URL) spelled. For a name or word that can't be split, pass `--pronunciations='{"Kamal":"Ka mal"}'` (whole-word, case-insensitive, applied for every engine). Listen back to the first line before generating a long script.
- **Canonical promo loop:** write the script → `media.generate-narration` →
  `project.add-audio` at `startMs` with the returned `durationMs` → time your
  motion graphics / B-roll to the voice.
- **Licensing note:** the local path is fully permissive — Kokoro weights are
  Apache-2.0 and the sidecar links misaki-rs built without its espeak fallback,
  so no GPL `espeak-ng` is compiled in.

### The user's OWN voice, and the user's cloned voice

- **Their own voice, recorded:** when the user wants to narrate themselves
  ("I'll talk over it", a silent screen recording), don't generate TTS: point
  them to the editor's **Audio tab → Record voiceover**. The preview plays from
  the playhead while their mic records; pausing cuts the gap; the take lands as
  a normal audio overlay (`voiceover-*.wav`, in `project.read` under
  `audioOverlays[]`). With "Transcribe for captions" on (default), its words
  merge into the transcript like `project.add-audio --transcribe`. You can't
  record their voice yourself.
- **Their cloned voice:** `--model=elevenlabs-direct` uses the user's OWN
  ElevenLabs account (pass the voice name or voice_id; needs ElevenLabs
  connected in Settings → Integrations).

### Always `--transcribe=true` when placing a VOICEOVER

Audio overlays are never transcribed otherwise (`transcript.transcribe` only
walks main-track clips), so a narration-driven video ends up with NO
transcript: no captions and no way to read back what was said. The flag
transcribes the narration and merges its words into the project transcript at
the overlay's position. It returns a `transcribeJobId`; `job.wait` on it before
`caption.toggle` or any transcript verb, and CHECK the job status: it fails
loudly (instead of merging nothing) when the audio has no recognisable speech.
Other edits while it runs are safe (the merge re-reads and retries). The words
belong to that overlay: `project.update-region` moving it moves them,
`project.remove-audio` removes them. `audioPath` must be an existing absolute
file (a missing/empty path → `audio_file_invalid`; check the variable isn't
empty). Do NOT pass it for music or ambience.

```bash
NARR=$(pandastudio media.generate-narration --text="..." --json | jq -r '.data.audioPath')
OUT=$(pandastudio project.add-audio --id=$ID --audioPath="$NARR" --startMs=0 --transcribe=true --json)
pandastudio job.wait --id=$(echo "$OUT" | jq -r '.data.transcribeJobId') --json
pandastudio caption.toggle --id=$ID --enabled=true   # now has words to render
```

For a from-scratch promo with narration, generate the VO FIRST and time each
scene to its line (TTS runs longer than you'd guess; see promo-and-mg-videos.md
"Audio: decide voiceover & music FIRST").

## B-roll generation (Replicate gpt-image-2)

PandaStudio ships with a project-level image-gen verb that runs on the
user's own connected Replicate account. Use it to author B-roll, concept
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

**Requires Replicate connected.** If the user hasn't connected it in
Settings → Integrations → Connectors, the verb returns an error saying
so. Don't loop on this; surface it to the user.

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



## AI presenter with Seedance (camera footage without a shoot)

Seedance 2.5 (Replicate `bytedance/seedance-2.5`) generates a realistic talking
presenter WITH her own voice and lip sync in one pass. No separate TTS track is
needed. Reach for it when a recipe or edit needs a camera layer and there is no
real footage (examples, demos, faceless-with-a-face explainers).

One verb runs it: `media.generate-presenter`. It is ASYNC: it returns `{ jobId }` at
once, and `job.wait` (give it `--timeoutMs=900000`; a take takes 5 to 10 minutes)
returns `{ videoPath, durationMs, width, height }`, the take in
<userData>/generated-avatars/.

```bash
pandastudio media.generate-presenter --durationSec=18 --aspectRatio=9:16 \
  --prompt='A woman in her early thirties, cream knit sweater, sitting at a desk by a
window in a bright home office, speaking straight to camera: "Most people think good
lighting needs expensive gear. It does not. Turn your desk to face a window and switch
off the ceiling light." Voice: warm, mid-pitch, unhurried, close microphone, quiet room.
No music, no sound effects. Single continuous locked-off medium close-up, tripod, no
camera movement, no cuts, same framing and lighting throughout.' --json
```

Do not add `media.generate-narration` on top: the voice is already in the file.

**Real people.** `referenceVideos` / `referenceImages` keep a person's look and
`referenceAudios` drives the lip sync ([Video1], [Image1], [Audio1] in the prompt). But
Seedance refuses a REAL person's face as a reference, photo or video, with "flagged as
sensitive (E005)" (confirmed Sep 18 2026 on the user's own footage). Don't retry or try to
disguise the face. To put the user on camera, use their own recording.

**Speaking in the user's own voice** (no ElevenLabs plan needed):
`media.generate-narration --model=voice-clone --voiceSample=<their recording> --voiceSampleStartMs=… --voiceSampleDurationMs=15000`
clones the voice from 10 to 20 s of their clean speech (Chatterbox). Keep each call to one
or two sentences (it drops words from long scripts), transcribe each take and redo any
that miss a word, then join them. `speed` below 1 slows a fast speaker (0.55 is its
slowest), and an ffmpeg `atempo` stretch fine-tunes the length. Only the user's own voice,
or one they have permission to use.

Rules that make it work:

- **One take, at most 30 seconds.** Each generation invents its own voice, so
  keep the script under ~70 words and make it ONE take. Two takes can sound
  like two different people.
- **Describe the person in words.** Uploading a photo of a realistic face as
  `image` or `reference_images` is rejected as sensitive (E005). Describe age,
  look, clothing and setting in the prompt instead.
- **Dialogue in double quotes**, with `generate_audio: true`. Add a voice line
  (age, accent, tone, pace, "close microphone, quiet room") and "no music, no
  sound effects".
- **Lock the camera**: "single continuous locked-off medium close-up, tripod,
  no camera movement, no cuts, same framing and lighting".
- Inputs: `durationSec` 4-30, `resolution` 480p or 720p, `aspectRatio` 16:9 or 9:16,
  `seed`. A script runs at about 2.4 words a second, so size the take to the words.

```bash
# 1. Generate the take (the verb downloads it for you; no media.import step)
JOB=$(pandastudio media.generate-presenter --durationSec=30 --prompt='...' --json | jq -r '.data.jobId')
PRESENTER=$(pandastudio job.wait --id="$JOB" --timeoutMs=900000 --json | jq -r '.data.job.result.videoPath')
# 2. Check the words: new project from it, transcribe, compare with the script
# 3a. Camera-only video: project.new --withMedia=<path>, then edit as usual
# 3b. Presenter over a screen track: put the screen video on the main track and
#     attach the presenter as its camera (make the screen track the same length).
#     A silent screen track takes the presenter's voice automatically
#     (audio=auto); then transcribe so transcript edits cut both.
pandastudio project.set-clip-webcam --id=$ID --clipIndex=0 --webcam="$PRESENTER" --json
pandastudio transcript.transcribe --id=$ID --json   # then job.wait
pandastudio project.set-webcam-layout --id=$ID --preset=picture-in-picture \
  --cropX=0.3 --cropY=0 --cropWidth=0.4 --cropHeight=1 --cx=0.17 --cy=0.5 --scale=2.9
```

Tell the user the presenter is AI-generated and that platforms may require a
disclosure label.

## Faceless videos — the full pipeline

A "faceless" video is **narration carrying the story over AI-generated IMAGES
that DEPICT each beat**, with slow Ken-Burns motion. No face, no camera (the
format behind history/mystery/educational channels). Each scene MUST be a real
image that SHOWS the beat ("the cyclops" → *a one-eyed giant in a torch-lit
cave*, NOT a card with the word "Cyclops"). The words belong in the voiceover.
Motion-graphic text scenes are the WRONG tool here; use them only for an
optional title card or a lower-third stat.

1. **Break the topic into beats.** One clear VISUAL idea per beat (~8–20s of
   narration). A 3–5 min video is ~12–20 beats.
2. **Write the narration line** for the beat.
3. **Generate the narration** → `media.generate-narration` (local Kokoro by
   default). ONE beat per call (~40–60 words): Kokoro caps a single call around
   ~25s, so long scripts get truncated. Grab its `durationMs`.
4. **Generate the IMAGE** → `media.generate-image` with a vivid, LITERAL visual
   prompt (subject, setting, lighting, mood, no on-screen words). 16:9 → `3:2`,
   9:16 → `2:3`. **Keep one art style across every image** (state it in every
   prompt, e.g. "cinematic oil-painting, warm dramatic light").
5. **Ken-Burns it as the next beat** → `media.image-to-video --id=$PID
   --imagePath=<img> --durationMs=<beat narration + ~400ms> --aspectRatio=9:16
   --zoom=in` (or `--zoom=out`, optional `--pan=left|right|up|down`). With
   `--id` the still becomes the next main-track IMAGE CLIP (kind `image`,
   filling the frame) and the zoom / pan a keyframed motion region over it: the
   engine draws the full-resolution image every frame (nothing baked) and both
   stay editable (the clip: `project.set-clip-duration`, move / split / remove;
   the move: Motion keyframes or `project.set-keyframes --target=main
   --regionId=<regionId>`). Returns `clipId`, `regionId` and `startMs`.
   `--aspectRatio` sets the project frame on the first beat (16:9, 9:16, 1:1,
   4:5); `--atIndex` inserts instead of appending. Alternate `in`/`out` across
   beats. (Only reach for a `motion.render-html` Ken-Burns shell for a bespoke
   CSS treatment — grain, parallax layers, vignette animation.)
   Without `--id` the verb BAKES an MP4 and returns `videoPath` (feed it to
   `project.add-clip --media=<videoPath>`); `--bake=true` with `--id` when the
   user wants the file itself. A still WITHOUT a move is
   `project.add-clip --media=<img> --durationMs=<ms>` (fills the frame;
   `--fill=false` frames it like a video).
6. **Lay the narration under it** → `project.add-audio --audioPath=…
   --startMs=<the beat's startMs>`.
7. **Polish (expected):** a quiet music bed (`asset.list-music` →
   `project.add-audio --ducking=true` at low volume, e.g. 0.15), burned captions
   (faceless viewers often watch muted; transcribe the narration with
   `--transcribe=true`), maybe ONE title card at the top.
8. **Export** (16:9 for YouTube, 9:16 for a faceless short).

**Timing rule:** each scene's length = its narration length; place each beat's
narration at the `startMs` the verb returned (beats carry the +400ms tail, so
never sum raw narration lengths). If Replicate isn't connected, image
generation is unavailable: say so and offer bundled templates as a lesser
fallback, or ask them to connect Replicate (Settings → Integrations →
Connectors).

```bash
# One beat (9:16) — repeat per beat.
IMG=$(pandastudio media.generate-image --prompt="a one-eyed giant in a torch-lit cave, cinematic oil-painting, warm dramatic light" --aspectRatio=2:3 --json | jq -r '.data.imagePath')
NARR=$(pandastudio media.generate-narration --text="In the cave of the cyclops, Odysseus faced a giant who ate men whole." --voice=am_michael --json)
DUR=$(echo "$NARR" | jq -r '.data.durationMs'); WAV=$(echo "$NARR" | jq -r '.data.audioPath')
START=$(pandastudio media.image-to-video --id="$PID" --imagePath="$IMG" --durationMs=$((DUR + 400)) --aspectRatio=9:16 --zoom=in --json | jq -r '.data.startMs')
pandastudio project.add-audio --id="$PID" --audioPath="$WAV" --startMs=$START --endMs=$((START + DUR)) --volume=1
```
