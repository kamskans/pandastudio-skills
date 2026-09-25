<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Projects: workspaces, brand kit, folders, rename, transcription

## Workspaces (v1.19+)

Every `project.*` / `export.*` / `motion.*` / `caption.*` / `audio.*` query
operates inside the **active workspace** (`workspace.current`). Agencies keep
clients in separate workspaces so credentials, exports and YouTube connections
never cross-contaminate.

```bash
# Right after system.status: the workspace context
pandastudio workspace.list --json | jq '.data | { current: .currentWorkspaceId, count: (.workspaces | length), cap: .limit }'
# limit.max: 1 = Starter plan or Trial, 3 = Creator, null = Team (unlimited)

# Switch (confirm with the user first if it's not the workspace they opened)
WS=$(pandastudio workspace.list --json | jq -r '.data.workspaces[] | select(.name == "ACME Agency — Client A") | .id')
pandastudio workspace.switch --id=$WS --json

# Create (agencies: one workspace per client)
pandastudio workspace.create --name="ACME Agency — Client A" --switchTo=true --json
# Plan cap hit → { ok:false, error:"Your Starter plan allows 1 workspace…",
#   details:{ code:"workspace_limit_reached", upgradeTo:"Creator" } }
# Tell the user to upgrade at writepanda.ai/#pricing; do NOT retry.

# Rename
pandastudio workspace.rename --id=$WS --name="Client A (2027)" --json
```

**Deleting (destructive, needs the user's confirmation in PandaStudio):** call
`workspace.contents` first so you can show what will be lost, ask in chat, then
`workspace.delete`. Projects' on-disk `.pandastudio` files stay, only the
library rows disappear. YouTube-published videos stay on YouTube; only the
local connection + cache go.

```bash
pandastudio workspace.contents --id=$WS --json | jq '.data.counts'
# { "projectCount": 12, "exportCount": 4, "publishedVideoCount": 3 }
pandastudio workspace.delete --id=$WS --json
```

### When given a project id with no other context

Run `project.locate` FIRST, before any read/edit/export/publish:

```bash
RES=$(pandastudio project.locate --id=$PID --json)
# { "data": { "id", "filePath", "workspaceId", "workspaceName": "Client A", "isInActiveWorkspace": false } }
if [ "$(echo "$RES" | jq -r '.data.isInActiveWorkspace')" != "true" ]; then
  # STOP. Ask: "This project lives in workspace 'Client A', current is '<X>' — switch?"
  :
fi
```

`project.read` resolves a project from any workspace, but
`export.publish-youtube`, `media.generate-image`, `export.generate-thumbnail`
and `youtube.list-accounts` use the **active workspace's** credentials: editing
project A while workspace B is active and then publishing puts the video on
Client B's channel. Every `project.read` also carries `workspaceId`,
`workspaceName`, `isInActiveWorkspace`, but `project.locate` is cheaper (no
body). After an explicit `workspace.switch`, re-run any pre-flight that depends
on workspace state (license check, YouTube account list, connector check).

## Project-look defaults (v1.49.1+)

Save a workspace's preferred **look** once so every NEW project and fresh
recording starts from it. Covers background (`wallpaper`), `captionSettings`
and `editorDefaults` (padding, shadow, corner radius, blur). The editor exposes
it as "Save as default for new projects".

```bash
pandastudio workspace.get-project-defaults --json          # null = none set
pandastudio workspace.set-project-defaults \
  --defaults='{"wallpaper":"/wallpapers/wallpaper5.jpg","captionSettings":{"enabled":true,"templateId":"editorial"},"editorDefaults":{"padding":18,"borderRadius":8}}' \
  --json                                                    # any subset; unknown fields dropped
pandastudio workspace.set-project-defaults --defaults=null --json   # clear
```

Use for "use this background for all my videos" / "always start new projects
with these captions". It doesn't retro-edit existing projects.

## Brand kit — set it, or auto-capture it from a URL (v1.84+)

The workspace brand kit (colors primary/accent/ink/background, display/body
fonts, logo, voice) feeds brand-aware captions, motion graphics, lower thirds
and thumbnails. Read it with `workspace.get-brand`.

```bash
# Manual: set any subset.
pandastudio workspace.set-brand --brand='{"name":"Acme","colors":{"primary":"#2563EB","ink":"#111827","background":"#FFFFFF"},"typography":{"display":"Inter"}}' --json

# Auto: pull the real brand off a website (HyperFrames capture, classifies the
# site's colors/fonts/logo, MERGES into the kit; hand-set fields survive).
# ASYNC; the first run downloads the capture CLI, so use a long timeout.
JOB=$(pandastudio workspace.capture-brand --url=https://acme.com --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --timeoutMs=300000 --json | jq '.data.job.result.brand'

# Capture WITHOUT touching the workspace kit (a client's or competitor's site,
# or a promo for a product that isn't the user's brand):
JOB=$(pandastudio workspace.capture-brand --url=https://acme.com --apply=false --maxScreenshots=4 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --timeoutMs=300000 --json \
  | jq '.data.job.result | {brand, logo: .captured.logoPath, og: .captured.ogImagePath, shots: [.captured.screenshots[].path]}'
```

Reach for `workspace.capture-brand` when the user says "use my brand", "make it
match my site", or you're onboarding a client and only have a URL. Needs
network. The classified brand is a starting point; apply corrections with
`workspace.set-brand`.
- **`--apply=false`** returns `{ applied:false, brand }` and leaves the active
  kit alone (default `true` = merge). Don't overwrite the user's brand with
  someone else's site.
- **Logo**: `captured.logoPath` is the site's real mark, picked by scoring
  every image / inline SVG on the page: the header / nav logo (in the home
  link, "logo" in its class / id / aria-label / alt, named after the site,
  first in the header) wins, else the page's declared logo (JSON-LD
  Organization / og:logo), else a footer logo named after the site, else a
  capture-flagged SVG that is a real mark, else the apple-touch-icon, the SVG
  favicon, the og:image (`captured.logoSource`: header-logo | declared-logo |
  page-logo | logo-svg | touch-icon | favicon-svg | og-image | small-icon).
  UI glyphs, small status icons in page content and customer logos are never
  chosen. SVG wordmarks drawn in `currentColor` come out black: recolour them
  for a dark ground. A text-only wordmark site gets its app icon; set the
  wordmark font via typography instead.
- **Screenshots**: `captured.screenshots[]` are real 1920×1080 viewport PNGs of
  the live page (hero first, then down the page; `--maxScreenshots` 0–12,
  default 4). Use them in promos (`motion.render-html --assets=<path>` /
  `<img src="screenshot-1.png">`) instead of drawing a fake UI.
  `captured.ogImagePath` is the site's social card.

## Organising projects with folders (v1.26.9+)

The Home page groups projects into flat folders (workspace-scoped, no nesting). A project's folder is stored on its `.pandastudio` file as `folder: "Tutorials"` and surfaced on every `project.list` row as `folder: string | null` (null = Unsorted).

### When to use
- User says "put this project in Tutorials" / "move these to Client X" / "organise my projects" / "label these so I can find them later".
- After bulk-creating projects (e.g. an import workflow), categorise them into folders so the Home page stays readable.

### Move a single project

```bash
# By id (preferred):
pandastudio project.set-folder --id=$PID --folder="Tutorials" --json

# By path:
pandastudio project.set-folder --path="$P" --folder="Tutorials" --json

# Clear the folder (move to Unsorted):
pandastudio project.set-folder --id=$PID --folder="" --json
```

Returns `{ id, path, folder }` where `folder` is the trimmed, normalised label (or `null` for Unsorted).

### Move many projects (e.g. all matching a search)

```bash
# Move every project whose name starts with "Tut" into a "Tutorials" folder.
pandastudio project.list --json | \
  jq -r '.projects[] | select(.name | startswith("Tut")) | .id' | \
  while read pid; do
    pandastudio project.set-folder --id=$pid --folder="Tutorials" --json
  done
```

### Read folder during listing

`project.list` rows now carry `folder`. Group locally:

```bash
pandastudio project.list --json | jq '
  .projects
  | group_by(.folder // "Unsorted")
  | map({folder: .[0].folder, count: length, ids: map(.id)})
'
```

### Caveats
- Folders are case-sensitive: "Tutorials" and "tutorials" are different folders. Use consistent casing.
- No nesting. `folder: "A/B"` is a single folder literally named `"A/B"` — it does NOT create a nested hierarchy.
- Folders are workspace-scoped: switching workspace shows a different set of folders (because the underlying project files differ).
- Renaming a folder is "move every project from old name → new name". There's no single rename verb yet; loop over `project.list` filtered by the old folder.

## Renaming a project (v1.40.0+)

`project.rename` sets the display name shown in the editor title bar and the Home grid. Identify by `id` (preferred) or `path`; pass the new `name` (trimmed, must be non-empty). The on-disk **filename does not change** (project files are keyed by id), so this is safe to call even while the project is open in the editor.

```bash
pandastudio project.rename --id=$PID --name="How I Built MyAgentMail" --json
# Returns { id, path, name }.
```

Use when the user says "rename this project", "call it X", or after `llm.generate-title` if they want the project itself (not just the export) to carry the new title. This is separate from the YouTube video title set at export time.

## Transcription languages (v1.27.0+)

PandaStudio ships two transcription engines under the hood:

- **Parakeet TDT v3** (default, "auto") — auto-detects English + 25 European languages, word-level timestamps native, ~473 MB model, ~30× realtime on CPU. Preserves filler words ("um", "uh", repeats) which is critical for the transcript-based editor. Bundled download on first use.
- **Whisper Large-v3-turbo** — for Chinese, Japanese, Korean, Hindi, Arabic, Thai, Tamil, Telugu, Kannada, Malayalam, Bengali, Marathi, Gujarati, Punjabi. ~1.1 GB Q5_0 GGUF, lazy-downloaded on first non-European language selection. Same word-level-timestamps contract via `set_token_timestamps + set_max_len(1) + set_split_on_word`.

The setting is **workspace-scoped**: a user can run an English channel in one workspace and a Mandarin channel in another without crosstalk.

### Check + switch from agent

```bash
# Read the active engine.
pandastudio system.get-transcription-language --json
# → { language: "auto" }

# Before switching to a non-European language, make sure the model is on disk.
pandastudio system.is-whisper-model-downloaded --json
# → { downloaded: false }

# If false, ask the user to open Settings and click "Download Whisper model"
# (~1.1 GB). The agent cannot trigger the download itself — the IPC requires
# main-window context. Once they confirm it's downloaded, proceed.

# Switch.
pandastudio system.set-transcription-language --language=chinese --json
# → { language: "chinese" }

# Now `transcript.transcribe` (or the editor's Transcribe button) will use
# Whisper instead of Parakeet for every call until the language is changed
# back.
```

### When to call this

- **User asks for a non-European transcription**: "transcribe this Chinese video", "give me the Japanese transcript", etc. → check + switch.
- **Project content suggests a language mismatch**: clip filenames or metadata indicate a non-European language but the current setting is "auto". Surface to the user before flipping the setting yourself.
- **Restoring "auto" after a one-off job**: if you switched for a specific clip, switch back when you're done so subsequent English/European transcriptions get Parakeet's faster + filler-preserving path.

### Caveats

- Whisper's seq2seq decoder smooths over fillers. That's fine for Chinese/Japanese/Korean where fillers behave differently anyway, but DO NOT switch to Whisper for English projects — the editor's "Remove Filler Words" / "Remove Silences" features depend on Parakeet's CTC honesty.
- Language hint is locked, not auto-detected, when Whisper is active. If the user picks "chinese" and then transcribes a Japanese file, the output is garbage. Match the setting to the actual source language.
- `system.set-transcription-language` only writes the setting — it does not download the Whisper model. The download is a Settings-UI-only action because it streams ~1.1 GB and surfaces a progress modal.

## Transcription provider: local vs cloud

**When the language is one the on-device models are weak at, say so before
you edit.** Parakeet (English + 25 European languages) and Whisper are good
enough to cut against. Whisper is NOT good at Tamil, Telugu, Kannada or
Malayalam, and the transcript is the foundation everything else stands on:
captions, `transcript.remove-fillers`, `transcript.remove-silences`, shorts
detection and any edit-by-text all inherit its mistakes.

`system.get-transcription-provider` reports who transcribes: `local` (default),
`deepgram` (Nova-3) or `elevenlabs` (Scribe), plus `ready` saying which cloud
providers have a key. Both cloud providers transcribe those languages properly
for roughly a penny a minute.

```bash
pandastudio system.get-transcription-provider --no-launch --json
```

If the user works in one of those languages on `local`, tell them the
transcript will be rough and that Settings → Transcription can switch to a
cloud provider. **Do not switch it yourself without asking**:
`system.set-transcription-provider` sends their audio to a third party and
bills their account there. Ask, then switch if they say yes. If a cloud
provider is set but has no key, transcription silently falls back to local, so
check `ready` before assuming the good path ran.

## Smooth preview for heavy camera footage

Some camera footage can't be decoded in real time, so the editor preview stutters however light the edit is: 10-bit or 4:2:2/4:4:4 video, HDR, ProRes / DNxHD / CineForm / MPEG-2, or anything above 150 Mbps (for example 4K 50fps H.264 4:2:2 10-bit intra from Sony cameras, ~480 Mbps).

With the setting on (`auto`, the default), opening a project makes a lighter copy of those sources in the background: same frame size and timing, 8-bit H.264 (HEVC above 4096x2160), hardware-encoded where possible, stored under the app's `preview-proxies` folder. The preview switches to it at the next pause. **Exports, transcription and every edit keep reading the original file.**

```bash
# Is a copy being built for this source?
pandastudio system.preview-proxy-status --path=/Users/me/Footage/C0042.MP4 --json
# → { status: { state: "generating", progress: 0.42, reasons: ["high-bit-depth", "chroma", "bitrate"] } }

# All sources this session + cache size
pandastudio system.preview-proxy-status --json

# Read / change the setting
pandastudio system.get-preview-proxy-mode --json
pandastudio system.set-preview-proxy-mode --mode=off --json
```

States: `not-needed` (plays fine as is), `queued`, `generating`, `ready`, `failed` (original keeps playing), `skipped` (less than 15 GB free), `unsupported` (media engine missing, odd frame size, or a very large frame this machine can't play as HEVC), `disabled` (setting is off).

### When to use it

- **User says the preview is choppy on camera footage**: check `system.preview-proxy-status`. If `generating`, tell them playback smooths out when it finishes. If `disabled`, suggest turning it back on.
- **Don't wait for it** before editing, `project.render-frame`, or `export.start`. Render-frame reads whatever the preview is showing; export always reads the original.
- **Disk space**: copies are capped at 30 GB (oldest removed first). The user can remove them in Settings → Playback.

## Transcribing a standalone audio / video file → text, SRT, or VTT

A very common ask: the user has a loose `interview.mp3` / `lecture.mov` on disk and just
wants a transcript or subtitle file. PandaStudio is project-based, so there is **no bare
`transcribe <file>` verb** — but wrapping the file in a project is one command, and YOU
(the agent) do any SRT/VTT formatting. **Do not look for a `transcript.export` verb; it does
not exist and is not needed — the timestamps are already in the `transcript.get` JSON, so
formatting subtitles is a pure text transform you perform yourself.**

**The minimum flow** (works for audio OR video — both carry an audio track Whisper reads):

```bash
# 1. Wrap the file in a project. --withMedia adds it as the first clip.
ID=$(pandastudio project.new --name="Interview" --withMedia="/abs/path/interview.mp3" --json | jq -r '.data.id')

# 2. Transcribe. The Parakeet model (~473 MB) auto-downloads on first ever call;
#    later calls are instant. Returns a jobId.
JOB=$(pandastudio transcript.transcribe --id=$ID --json | jq -r '.data.jobId')

# 3. Wait for the job, then pull the transcript with word-level timestamps.
pandastudio job.wait --id=$JOB --json
pandastudio transcript.get --id=$ID --format=words --json
```

`transcript.get --format=words` returns a flat `words[]`; each word has `{ id, text,
startMs, endMs, editedStartMs, editedEndMs, speaker? }`. `--format=full` gives the old
`{ language, wordCount, segmentCount, words[], segments[] }` shape; the default `compact`
format and `--format=text` (edited-time `[m:ss.s]` lines) are for reading, and
`--fromMs/--toMs` (edited ms) limit any format to a window. **All times are milliseconds.**

### ⚠ Which time base to use — this decides whether the subtitles are in sync

Every word carries TWO time bases:

- `startMs` / `endMs` — **SOURCE** time (the raw recording, before any edits).
- `editedStartMs` / `editedEndMs` — **EDITED-timeline** time, with trims + speed regions
  applied. `null` means the word sits inside a trim (it was cut and is NOT in the export).

**Rule: subtitles must match the file the user will actually upload.**

- The project has **no edits** (a loose file you just wrapped in a project to transcribe):
  either base works — they're identical. Use `startMs`/`endMs`.
- The project **has any edits** (filler removal, remove-silences, deleted words, speed
  regions — i.e. the normal PandaStudio flow): you MUST use `editedStartMs`/`editedEndMs`
  and **skip every word whose `editedStartMs` is `null`**. Those words were trimmed out of
  the export; including them, or using source times, produces an SRT that drifts further
  out of sync with every cut.

Do NOT build cues from `segments[].text` on an edited project — a segment can straddle a
trim. Build cues from `words[]`, filtered to non-null `editedStartMs`, then group them into
lines (a new cue on a gap of ~700ms+, or every ~7-10 words / ~42 chars per line).

### Captions as SRT for a YouTube upload (the common ask)

YouTube uses an uploaded `.srt` to power auto-translation into other languages, so the
timing and the spelling both matter. There are two valid routes — pick by whether the
project's transcript has human corrections in it.

**Route A (default): reuse the project transcript.** Free, and it keeps any transcript
fixes the user made (brand/product names via `transcript.find-replace`). Requires using the
edited time base:

```bash
pandastudio transcript.get --id=$ID --format=words --json > /tmp/t.json
# Format cues from words[] using editedStartMs/editedEndMs (skip null = trimmed).
```

**Route B: transcribe the exported MP4 fresh.** Wrap the export in a throwaway project and
transcribe it (same flow as any standalone file). The new transcript's `startMs`/`endMs` are
already relative to the exported video, so the timing is correct by construction — no
edited/source reasoning at all:

```bash
EID=$(pandastudio project.new --name="srt-tmp" --withMedia="/abs/path/export.mp4" --json | jq -r '.data.id')
JOB=$(pandastudio transcript.transcribe --id=$EID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --json
pandastudio transcript.get --id=$EID --format=words --json    # startMs/endMs are already export-relative
```

**Which to use:** prefer Route A when the source project is available — it costs no second
pass and, crucially, preserves corrected spellings. Re-transcribing (Route B) re-runs STT
and will reintroduce the original misspellings, which YouTube then translates into every
language. Use Route B when the source project isn't at hand, the export came from elsewhere,
or you want timing that cannot be gotten wrong.

Either way the cue timestamps are `HH:MM:SS,mmm` measured from the start of the exported
video (both `editedStartMs` and a fresh transcript's `startMs` are already zero-based on it,
so there is no offset math).

**Producing the output the user asked for — you format it, no verb:**

- **Plain text** → join `segments[].text` with newlines (or spaces for a single paragraph).
- **SRT** → number each segment from 1, timestamp format `HH:MM:SS,mmm` (comma before ms),
  blank line between cues:
  ```
  1
  00:00:00,000 --> 00:00:03,480
  Every great video starts with a single frame.

  2
  00:00:03,480 --> 00:00:06,900
  And the right tools make all the difference.
  ```
- **VTT** → first line `WEBVTT` then a blank line; timestamp format `HH:MM:SS.mmm` (DOT before
  ms, not comma); no cue numbers required:
  ```
  WEBVTT

  00:00:00.000 --> 00:00:03.480
  Every great video starts with a single frame.

  00:00:03.480 --> 00:00:06.900
  And the right tools make all the difference.
  ```

Convert `startMs`/`endMs` to `HH:MM:SS` by integer-dividing: `h = ms/3600000`, `m =
(ms/60000)%60`, `s = (ms/1000)%60`, `ms3 = ms%1000` zero-padded to 3 digits. Write the
result to a file next to the source (`interview.srt` / `interview.vtt`) unless the user
names a path. Prefer **segment-level cues** for readability; only emit word-level cues if the
user explicitly wants karaoke-style timing.

**Caveats for this flow:**

- **Non-European languages** (Chinese, Japanese, Korean, Hindi, Arabic, Thai, Tamil, Telugu, Kannada, Malayalam, Bengali, Marathi, Gujarati, Punjabi) need the Whisper
  model switch first — see the "Transcription languages" section above. For English + 25
  European languages, the default Parakeet engine just works.
- **The project is a real artifact.** `project.new` writes a `.pandastudio` file. If the user
  only wanted a transcript and doesn't care about the project, that's fine to leave behind, or
  call `project.delete --id=$ID` once you've handed over the subtitle file. Ask if unsure —
  don't delete a project the user might want to keep editing.
- **The desktop app must be running** (the CLI auto-launches it if not). Transcription uses the
  app's bundled FFmpeg + Whisper sidecar; there's no fully-headless mode.

