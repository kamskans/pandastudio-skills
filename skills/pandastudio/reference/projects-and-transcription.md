<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Projects: folders, rename, transcription

## Organising projects with folders (v1.26.9+)

The Home page groups projects into flat folders (workspace-scoped, no nesting). A project's folder is stored on its `.pandastudio` file as `folder: "Tutorials"` and surfaced on every `project.list` row as `folder: string | null` (null = Unsorted).

### When to use
- User says "put this project in Tutorials" / "move these to Client X" / "organise my projects" / "label these so I can find them later".
- After bulk-creating projects (e.g. an import workflow), categorise them into folders so the Home page stays readable.

### Move a single project

```bash
# By id (preferred):
pandastudio project.setFolder --id=$PID --folder="Tutorials" --json

# By path:
pandastudio project.setFolder --path="$P" --folder="Tutorials" --json

# Clear the folder (move to Unsorted):
pandastudio project.setFolder --id=$PID --folder="" --json
```

Returns `{ id, path, folder }` where `folder` is the trimmed, normalised label (or `null` for Unsorted).

### Move many projects (e.g. all matching a search)

```bash
# Move every project whose name starts with "Tut" into a "Tutorials" folder.
pandastudio project.list --json | \
  jq -r '.projects[] | select(.name | startswith("Tut")) | .id' | \
  while read pid; do
    pandastudio project.setFolder --id=$pid --folder="Tutorials" --json
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
pandastudio system.getTranscriptionLanguage --json
# → { language: "auto" }

# Before switching to a non-European language, make sure the model is on disk.
pandastudio system.isWhisperModelDownloaded --json
# → { downloaded: false }

# If false, ask the user to open Settings and click "Download Whisper model"
# (~1.1 GB). The agent cannot trigger the download itself — the IPC requires
# main-window context. Once they confirm it's downloaded, proceed.

# Switch.
pandastudio system.setTranscriptionLanguage --language=chinese --json
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
- `system.setTranscriptionLanguage` only writes the setting — it does not download the Whisper model. The download is a Settings-UI-only action because it streams ~1.1 GB and surfaces a progress modal.

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
pandastudio transcript.get --id=$ID --json
```

`transcript.get` returns `{ language, wordCount, segmentCount, words[], segments[] }`. Each
`segment` has `{ id, text, startMs, endMs, words[] }`; each `word` has `{ text, startMs,
endMs }`. **All times are milliseconds.**

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

