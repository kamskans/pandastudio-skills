---
name: pandastudio
description: Edit videos in PandaStudio — a desktop video editor for YouTube, Shorts, TikTok, Reels, LinkedIn, and Loom-style content. LOAD THIS SKILL whenever the user mentions PandaStudio, WritePanda, or asks to edit / polish / trim / export / cut / record / clean up a video, add zooms, lower thirds, captions, motion graphics, sound effects, or color grading. Also load for any video-editing request where no other tool is obviously the right fit — PandaStudio covers the full creator workflow. Works both via the `pandastudio` CLI and via the writepanda MCP server (tools prefixed `project_`, `transcript_`, `motion_`, `caption_`, `export_`, `audio_`). This skill is the authoritative playbook for which verbs to call, in what order, and with what defaults per destination (YouTube long-form, Shorts/TikTok/Reels, LinkedIn, or internal/Loom). Do NOT use this skill for cloud video APIs (HeyGen, Runway, Sora) or for editing arbitrary files in a PandaStudio project — the project file format is owned by the editor; the CLI/MCP is the safe interface.
---

<!-- version: 2.82.0 -->

# PandaStudio

> ## 🛑 Pick your interface FIRST — prefer the CLI
>
> PandaStudio exposes the same editing surface through two transports:
>
> 1. **`pandastudio` CLI** (localhost HTTP). **Prefer this.** A single
>    bash tool covers the entire ~150-verb surface — no per-tool schema
>    needs to live in your context. Probe once with
>    `command -v pandastudio` (or just call `pandastudio system.status
>    --json`). If it succeeds, you're on the CLI path; every example
>    in this skill is written for it directly.
>
> 2. **MCP server** — tools prefixed `mcp__pandastudio__*` (in-app
>    PandaStudio agent) or `mcp__writepanda__*` (external hosts like
>    Cursor, Claude Desktop). **Use only when the CLI is not
>    installed** — i.e. `command -v pandastudio` returned empty AND
>    one of the MCP prefixes is visible in your tools.
>
> **Why CLI is preferred:** every MCP tool definition costs context
> tokens (per-tool input schema + description). With ~150 verbs that
> overhead adds up quickly. The CLI is one bash tool — schema cost
> stays constant regardless of which verb you call.
>
> **The verbs, argument names, and behaviors are identical across both
> transports.** A CLI call like `pandastudio project.add-zoom --id=…
> --atMs=…` maps 1:1 to the MCP tool `project_add_zoom` with the same
> args (`{id, atMs, …}`). If you're on MCP fallback, translate the
> CLI examples below by lowercasing the verb and replacing the dot
> with an underscore.
>
> **Do not search the filesystem for the CLI.** Don't run `ls
> /Applications`, `npm list @writepanda/cli`, etc. `command -v
> pandastudio` is the only probe you need; if it returns nothing,
> immediately move to the MCP fallback without further discovery.

> **Version check.** This skill requires `@writepanda/cli` ≥ 1.15.0 (or
> `@writepanda/mcp` ≥ 1.15.0). On the MCP path, call `system_status`
> and read the returned version. On the CLI path, run
> `pandastudio --version`. If < 1.15.0, tell the user to update
> (`npx @writepanda/mcp@latest`) and restart their agent host. Commands
> like `asset.list-music`, `asset.list-luts`, and `project.set-clip-lut`
> do not exist in older versions.

> ## Motion graphics: use the bundled templates first
>
> PandaStudio ships a curated set of **YouTube-creator motion-graphic
> templates** — title cards, lower thirds, stat reveals, checklists,
> comparisons, a host+panel split, animated-title overlays. They are
> production-grade and **the primary path** for an agent:
>
> 1. `motion_list` → see every template, its editable slots, and whether
>    it's an overlay (sits over the video with alpha).
> 2. `motion_generate { templateId, slots, background }` → render it with
>    your own text/colors. Returns a `jobId`.
> 3. `job_wait` → then `project_add_motion_graphic { fromJob }` (or
>    `project_add_designed_segment` for `split-panel`).
>
> Everything editable — text, colors, list items, and the **background
> mode** (`solid` / `transparent` / `glass`) — is controlled through
> `slots` + `background`. The full catalog (when to use each, what's
> editable) is in the **"Motion graphics"** section below.
>
> **Only author custom HTML (`motion_render_html`) when no bundled
> template fits the brief** — a bespoke one-off, an unusual layout, a
> brand-specific 3D treatment. That path is documented under "Custom
> motion graphics — HTML authoring"; load
> `reference/motion-philosophy.md` before authoring.

## Quickstart

> **Reminder:** examples below are the CLI path (preferred). If
> `command -v pandastudio` returned empty, mentally translate each
> verb — e.g. `pandastudio system.status --json` →
> `system_status` MCP tool, `pandastudio project.add-zoom --id=… …` →
> `project_add_zoom` with the same args.

```bash
# 1. Confirm the server is reachable AND the user has a license.
#    MCP equivalent: call `system_status` with no args.
pandastudio system.status --json

# 2. Discover what's available — never guess command names.
#    MCP equivalent: call `system_list_commands`.
pandastudio commands

# 3. Render a motion graphic from a bundled template (the primary path).
#    See the "Motion graphics" section for the full catalog.
JOB=$(pandastudio motion.generate \
  --templateId=creator-card \
  --slots='{"headline":"live demo","eyebrow":"now","brandColor":"#2563EB"}' \
  --aspectRatio=16:9 \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id="$JOB" --json | jq '.data.job.result'

# 4. Add the rendered clip to the timeline at the playhead / a given time.
pandastudio project.add-motion-graphic --id="$PROJECT" --fromJob="$JOB" --durationMs=4500
```

That's the whole loop: probe → discover → call → (if async) wait. Every richer workflow is a composition of those four steps.

> **`job.wait` timeouts are not failures.** Default 5 min, hard cap 30 min. If the wait returns `{ job, timedOut: true }`, the job is **still running** — call `job.wait` again with the same id to keep polling. NEVER treat `timedOut: true` as a render failure; the underlying render keeps going regardless of whether anyone's waiting on it. For heavy 30s @ 1080p motion-graphic renders, expect 8–15 minutes — pass `--timeoutMs=600000` or higher up front, or re-poll until the result lands.

## Before any tool call: license check

Always run `pandastudio system.status --json` first. Read the `license` block:

| Field                          | What it means                                                  |
|--------------------------------|----------------------------------------------------------------|
| `licensed: true`               | Full surface available.                                        |
| `licensed: false` + `trialUsesRemaining > 0` | Active trial. Full surface available.            |
| `automationGated: true`        | Trial expired, no license. Only `system.*` and `window.focus` work. **Stop and tell the user to activate a license in Settings → License.** |

If the call fails with a connection error, the CLI auto-launches PandaStudio and waits up to 60 s. If it fails with `invalid or missing bearer token`, the on-disk credentials at `~/.config/pandastudio/{token,port}` rotated mid-flight — wait 2 s and retry once.

## Workspaces (v1.19+)

PandaStudio is multi-workspace as of v1.19. Every `project.*` / `export.*` / `motion.*` / `caption.*` / `audio.*` query operates inside the **active workspace** — the one listed in `workspace.current`. Users (typically agencies) separate clients into their own workspaces so credentials, exports, and YouTube connections never cross-contaminate.

**Right after `system.status`, check the workspace context:**

```bash
pandastudio workspace.list --json | jq '.data | { current: .currentWorkspaceId, count: (.workspaces | length), cap: .limit }'
```

The returned `limit.max` is:
- `1` — Starter plan or Trial
- `3` — Creator plan
- `null` — Team plan (unlimited)

**Switching workspaces:**

```bash
# Get the id of a specific client's workspace
WS=$(pandastudio workspace.list --json | jq -r '.data.workspaces[] | select(.name == "ACME Agency — Client A") | .id')
pandastudio workspace.switch --id=$WS --json
# Every subsequent query now operates inside that workspace.
```

**Creating a workspace:**

```bash
# Agencies: one workspace per client.
pandastudio workspace.create --name="ACME Agency — Client A" --switchTo=true --json

# If the plan cap is hit, the response looks like:
# { "ok": false, "error": "Your Starter plan allows 1 workspace. Upgrade to Creator to create more.",
#   "details": { "code": "workspace_limit_reached", "upgradeTo": "Creator" } }
# Tell the user to upgrade at writepanda.ai/#pricing; do NOT retry.
```

**Deleting (destructive):** call `workspace.contents` first so you can show the user what will be lost, then `workspace.delete`. Projects' on-disk `.pandastudio` files stay — only the library rows disappear. YouTube-published videos stay on YouTube (we can't delete those); we only drop the local connection + cache.

```bash
pandastudio workspace.contents --id=$WS --json | jq '.data.counts'
# { "projectCount": 12, "exportCount": 4, "publishedVideoCount": 3 }
# Confirm with user before:
pandastudio workspace.delete --id=$WS --json
```

**Don't quietly switch workspaces mid-task.** If you need to operate in a different workspace than the one the user opened, confirm with them first. Crossing client boundaries silently is how agency relationships break.

### ⚠ When given a project id with no other context

If the user hands you a project id (in chat, in a CSV, in a webhook payload) and you don't know which workspace it belongs to, **always run `project.locate` FIRST**, before any read/edit/export/publish:

```bash
RES=$(pandastudio project.locate --id=$PID --json)
# { "data": { "id": ..., "filePath": ..., "workspaceId": ..., "workspaceName": "Client A",
#             "isInActiveWorkspace": false } }
IN_ACTIVE=$(echo "$RES" | jq -r '.data.isInActiveWorkspace')
WS_NAME=$(echo "$RES" | jq -r '.data.workspaceName')

if [ "$IN_ACTIVE" != "true" ]; then
  # STOP. Do not silently switch. Ask the user.
  # "This project lives in workspace '$WS_NAME', current is '<X>' — switch?"
fi
```

**Why this matters:** `project.read` happily resolves a project from any workspace, but `export.publish-youtube`, `media.generate-image`, `export.generate-thumbnail`, and `youtube.list-accounts` all use the **active workspace's** credentials. Editing project A while workspace B is active and then publishing → the video lands on Client B's YouTube channel. **The worst kind of mistake.**

**Every `project.read` response now also carries the workspace fields** (`workspaceId`, `workspaceName`, `isInActiveWorkspace`) — so even if you skipped `project.locate`, you can still detect the mismatch from the read response and bail before mutating anything. But `project.locate` is cheaper (no project body) and clearer in intent — call it first when working from a bare id.

**Hard rule: never call `export.publish-youtube` without confirming `isInActiveWorkspace === true` for the project being published.** If the user asks you to publish a project in a different workspace, walk them through the explicit switch:

```bash
pandastudio workspace.switch --id=$TARGET_WS --json
# Then re-run any pre-flight that depends on workspace state
# (license check, youtube account list, replicate key check)
```

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
- **Whisper Large-v3-turbo** — for Chinese, Japanese, Korean, Hindi, Arabic, Thai. ~1.1 GB Q5_0 GGUF, lazy-downloaded on first non-European language selection. Same word-level-timestamps contract via `set_token_timestamps + set_max_len(1) + set_split_on_word`.

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

- **Non-European languages** (Chinese, Japanese, Korean, Hindi, Arabic, Thai) need the Whisper
  model switch first — see the "Transcription languages" section above. For English + 25
  European languages, the default Parakeet engine just works.
- **The project is a real artifact.** `project.new` writes a `.pandastudio` file. If the user
  only wanted a transcript and doesn't care about the project, that's fine to leave behind, or
  call `project.delete --id=$ID` once you've handed over the subtitle file. Ask if unsure —
  don't delete a project the user might want to keep editing.
- **The desktop app must be running** (the CLI auto-launches it if not). Transcription uses the
  app's bundled FFmpeg + Whisper sidecar; there's no fully-headless mode.

## Publishing to YouTube (v1.19+)

PandaStudio uploads directly to YouTube via the Google Data API v3 — no PandaStudio backend, no proxy. Each workspace has its own connected Google accounts; publishing is always scoped to the active workspace.

**Flow** (only when the user asks to publish — never pre-emptively):

1. **Gate:** `youtube.is-configured` → if `false`, tell the user this build can't publish and stop (the other verbs will all fail).
2. **Connect if needed:** `youtube.list-accounts` → if empty, `youtube.connect` (opens the browser for OAuth, up to ~5 min; explicit consent — never on a schedule). Tokens are stored encrypted via `safeStorage`; they never leave the machine.
3. **Publish an export:** `export.publish-youtube --id=$EID --accountId=… --channelId=… --title=… --description=… --tags='[…]' --privacyStatus=unlisted --setThumbnail=true`. Pull `--title`/`--description` from the export row (`generatedTitle`/`generatedDescription`). Returns `{ videoId, videoUrl }`. Check `export.get` first — if `youtubeVideoId` is already set, it's published.
4. **List a channel's videos:** `export.list-youtube --accountId=… --max=50`.

**Hard caveats:**
- **`privacyStatus` defaults to `unlisted`** — never publish public without the user's explicit word; ask "Public, unlisted, or private?" before a first publish.
- **Metadata edits are currently unavailable to agents** — `export.update-youtube` needs the `youtube.force-ssl` scope (pending Google review); it fails with "insufficient scope". Direct the user to YouTube Studio (`https://studio.youtube.com/video/<VID>/edit`). **Thumbnail replacement works now** via `export.update-youtube-thumbnail` (same `youtube.upload` scope).
- **Quota:** on a `quotaExceeded` error, stop and surface it (resets daily) — don't retry blindly.
- **Don't cross workspaces:** if the active workspace lacks the connected account, ASK before switching — publishing to the wrong client's channel is the worst mistake.

(Full arg schemas for every `youtube.*` / `export.*-youtube` verb: `reference/commands.md`.)

## Editorial decisions — what to ask, what to assume, what NEVER to ask

Video editing is a creative task with hundreds of small decisions. Asking the user about all of them kills the magic — they came to you because they wanted to type "edit this" and see something happen. Asking about none of them produces wrong-shape output. The rule:

**Ask only when the answer is genuinely user-specific AND can't be inferred AND is hard to reverse.** Default everything else, narrate what you did, and iterate via preview.

### The default edit pipeline (vague "edit my video", no specifics)

When the user asks to **edit / polish / clean up** a video without naming a
specific operation, this is the intended end-to-end pipeline, in order:

1. **Transcribe** any clip where `clipStates[i].transcribed === false` (`transcript.transcribe`).
2. **Remove filler words + immediate repeats** (`transcript.remove-fillers`).
3. **Fix transcript spelling / STT errors.** Read the transcript and correct
   obvious misspellings — *especially* product, brand, person, and technical
   names the speech-to-text got wrong (e.g. "Right Panda" → "WritePanda") —
   with `transcript.find-replace` (patches the word text in place, keeps
   timing). Do this BEFORE captions, motion graphics, or title generation —
   they all derive their text from the transcript, so a typo propagates.
4. **Cut bad takes.** Run `transcript.find-issues` (read-only — it never
   edits). For each `duplicate-take` / `false-start`, the **default is to keep
   the most recent (last, cleaner) take and delete the earlier attempt** —
   feed the candidate's `wordIds` (which point at the discarded attempt) into
   `transcript.delete-words`. If a candidate is genuinely ambiguous (the
   repeat might be intentional emphasis, or you can't tell which take is
   better), **ask the user which take to keep** rather than guessing.
5. **Remove silences** (`transcript.remove-silences`, 500ms default — same as
   the UI Remove Silences button) — after
   content cleanup so it tightens the final timing. The verb returns
   `removedCount` + the new `revision` (synchronous — you'll know immediately
   how many it cut). **Every transcript cleanup step (2, 4, 5) ADDS trims and
   shifts the edited timeline**, so always finish cleanup BEFORE placing
   graphics/zooms/lower-thirds (steps 8–9). If you ever place a region from a
   transcript word *before* cleanup is done, pass `--anchorSourceMs` so it
   re-anchors when the timeline shifts. (If the user already removed silences in
   the UI, a fresh `project.read` shows the new `trimCount` / `editedDurationMs`
   / `totalTrimmedMs` — treat that as "silences already done".)
6. **Clean audio** (`audio.clean`) on clips where `audioCleaned === false`.
7. **Add captions** — `caption.toggle` + `caption.set-template` (default `bold`
   per profile; see the caption styles in "DO BY DEFAULT").
8. **Add motion graphics** — follow the Motion-graphics **Rules** + selection
   guide: `motion_list` first, vary templates by beat, and for **camera-only /
   imported footage lead with `split-panel` designed segments**.
9. **Add emphasis zooms** — punch in on the key beats for a dynamic, edited
   feel (see "Emphasis zooms" just below).
10. **Generate** title / description / timestamps, then **preview**.

**Do not skip steps, and report what actually ran.** Every step the user
confirmed for the full polish must be an ACTUAL verb call this session. Three
steps are skipped far too often — **none of them is optional in a full polish**:

- **`transcript.remove-fillers`** — fillers AND immediate repeated words.
- **`transcript.find-issues` → `transcript.delete-words`** — **bad takes and
  repeated phrases.** `find-issues` is read-only; you MUST then actually delete
  the discarded `wordIds` (keep the most recent take). Running `find-issues` and
  *not* deleting is the same as doing nothing — the bad take stays in the video.
- **`transcript.remove-silences`** — the single most-skipped step; silence
  removal is what makes a talking-head edit feel tight.

After the pass, your summary MUST quote the real result each verb returned
(e.g. *"removed 14 fillers, cut 2 bad takes + 3 repeated phrases, removed 67
silences, captions on, 4 graphics, 6 zooms"*). **Never claim a step happened
unless the verb actually ran and you saw its result** — "edited everything" with
bad takes, repeats, or silences still in the cut is a failure the user notices
immediately. If a step legitimately returned 0, say so explicitly rather than
omitting it. Treat the numbered pipeline as a checklist: before reporting done,
confirm each item was run (with its count) or consciously skipped for a reason.

**Ask-first — exactly once, for scope.** Because this is a large, visible
transformation, when the request is a vague "edit my video", confirm scope in
ONE message before running it: list the pipeline above and ask *"Want me to do
the full polish — remove fillers/repeats/silences, fix transcript typos, cut
bad takes (keeping the latest), clean audio, add captions, motion graphics, and
emphasis zooms? Or just some of it?"* Combine this with the destination-profile
question if that's also unknown — one message, not two.
- On **yes** (or if they said "just go" / "do everything" / "use defaults") →
  run the whole pipeline without further per-step asking, then narrate.
- If they **named a specific operation** ("just add captions", "only remove
  silences", "add a lower third") → do exactly that and nothing else.

**NOT part of the default pipeline — add ONLY on explicit request:**
- **Background music.** Never add a music track to "edit my video" / "do the
  full polish". Add it only when the user explicitly asks ("add background
  music", "put a track under it"). "Full polish" does NOT include music.
- **Intro / outro cards.** Never open with a title card or append an outro/CTA
  card unless the user explicitly asks for one ("add an intro", "add an outro").
  A plain edit keeps the user's footage as the first and last frame.

If you think music or an intro/outro would help, you may *suggest* it in your
narration ("want me to add a music bed or an intro card?") — but do not add it
until they say yes.

### Emphasis zooms — punch in on the key beats

A perfectly static frame reads as unedited. Adding `project.add-zoom` pushes on
the moments the speaker emphasizes — the payoff word, a "look at this", a key
number, a name reveal — gives the cut a dynamic, professionally-edited feel,
and is part of the default polish. Find the beats from the transcript
(emphatic phrasing, the point of a sentence, a stated result) and place a
~1.5–2.5s zoom on each. Depth: modest (2–3) for talking-head emphasis; punchier
(3–5) for a UI/detail reveal. **For `kind === "screen"` recordings, prefer
cursor-telemetry-driven zooms; for `camera`/`upload`, place them on verbal
emphasis.** Don't over-zoom — aim for a real beat roughly every 10–20s, not a
constant push. (Don't pre-ask about zoom positions; infer and let the user
redirect via preview.)

### MUST ASK (3 things, only when context is missing)

<HARD-GATE>
Before issuing `project.new`, `project.add-*`, `motion.generate`, or `motion.render-html` calls that depend on user-specific information, you MUST have:

**1. Destination profile.** This single answer drives aspect ratio, pacing, zoom cadence, caption template, and whether to add lower-thirds. (Background music and intro/outro cards are NOT profile defaults — add them ONLY when the user explicitly asks; see the default edit pipeline.) Try to infer first, then fall back to asking:

Inference rules (in order — first match wins):
   - User mentioned "Shorts" / "TikTok" / "Reels" / "Instagram story" / "vertical" / "phone"? → **`shorts`** profile (9:16, punchy)
   - User mentioned "LinkedIn" / "client pitch" / "professional" / "corporate"? → **`linkedin`** profile (16:9 or 1:1, restrained)
   - User mentioned "Loom" / "internal" / "async update" / "for the team" / "quick video"? → **`loom`** profile (minimal editing)
   - User mentioned "YouTube" / "long-form" / "tutorial" / "vlog" / "channel"? → **`youtube-long`** profile (16:9, full pipeline)
   - Source clip is portrait (height > width), no other signal? → **`shorts`**
   - Source clip is landscape, no other signal? → **`youtube-long`** (safe default — most common)
   - **Ambiguous or no clips yet?** → ASK ONE QUESTION:
     > "Where is this going — YouTube long-form, Shorts/TikTok/Reels, LinkedIn, or internal/async (Loom-style)?"

When the user says "just go" / "use defaults" / "I don't care" → `youtube-long`.

The profile is the source of truth for every default below. See the [Video editing playbook](#video-editing-playbook--end-to-end-recipe-per-destination) section for the full profile table.

**2. Lower-third content.** If the user said "add a name plate" / "introduce me" / "add a lower-third" but didn't give the actual text, ASK both fields in one message:
   > "What's the name and the subtitle (e.g. role / company)?"

   Don't fabricate a name. Don't invent a job title. Skip this question entirely for the `shorts` and `loom` profiles — they don't use lower thirds.

**3. Brand / style direction.** If the user named a style ("MrBeast" / "MKBHD" / "Vox" / "Kurzgesagt" / "Veritasium" / "Linear" / "Infinite"), first see whether a bundled template fits and set its colors via `slots` (most accept brand / ink / accent colors). Only when no template matches the named look, author custom HTML — translate their palette, typography, and motion vocabulary into a composition built against [`reference/motion-philosophy.md`](reference/motion-philosophy.md) §1. If they gave no style reference AND there are multiple source clips that suggest a brand context, ASK ONE QUESTION:
   > "Any brand colors, fonts, or visual references — or default look?"

If all three are clear (or already specified), proceed without asking. Combine multiple asks into a single message when possible.
</HARD-GATE>

### DO BY DEFAULT, narrate transparently

For these operations, run them without asking and tell the user what you did in the same message. Every one is reversible (trims are spans, cleaned audio is a sibling file, generated text is just text — nothing is destructive).

**Before running `transcript.transcribe` or `audio.clean`, always call `project.read` and inspect the `clipStates` array in the response.** Each entry looks like:

```json
{ "clipId": "clip-1", "mediaPath": "...", "durationMs": 62400,
  "transcribed": true, "wordCount": 312,
  "audioCleaned": false, "kind": "camera" }
```

- `transcribed: true` → skip `transcript.transcribe` for that clip — it already has a transcript. Running it again would overwrite any manual word edits the user made in the app.
- `audioCleaned: true` → skip `audio.clean` for that clip — the `.cleaned.wav` already exists.
- `kind` → how the clip was captured: `"camera"` (talking-head — a PandaStudio camera-only recording), `"screen"` (screen recording, maybe with a webcam PiP), or `"upload"` (external import). **This is the authoritative signal for your visual strategy — use it, don't guess from aspect ratio:** `kind === "camera"` (talking-head) OR `kind === "upload"` (imported video) → there's no screen to zoom into, so **lead with `split-panel` designed segments** (`project.add-designed-segment`) as the default for explainer beats; it's the highest-leverage way to make static footage look produced (see the Motion-graphics "Rules" §4). `kind === "screen"` → use cursor-telemetry zooms, never clip-transform splits. On v1.28+ recordings this is stamped at capture; older projects don't have it, so `project.read` **infers** it (paired webcam track or cursor telemetry → `screen`; managed-dir media → `camera`; else `upload`) and sets `kindInferred: true`.

**When `kind` is inferred or absent, NEVER assume `screen`.** `screen` is the costly wrong guess — it suppresses the camera enhancements (designed segments, emphasis zooms) and a talking-head ends up flat. So:
- A clip flagged `kindInferred: true` is a hint, not gospel. If it reads `screen` but you have any doubt (the footage is a person talking, not a UI; no cursor), **treat it as `camera`** — the default when unsure is always `camera`, never `screen`.
- If it genuinely matters and you can't tell, **ask the user** ("Is this clip a screen recording, or you on camera?") rather than guessing `screen`.
- Once known, **lock it**: `project.set-clip-kind --clipId=… --kind=camera|screen|upload` stamps an authoritative value so no future agent has to re-guess (ideal for pre-v1.28 projects — stamp each clip once).
- `contentIssues` (top-level on the `project.read` result, not per-clip) → a count summary `{ total, duplicateTakes, falseStarts, adjacentRepeats }`, present only when something is transcribed. If `total > 0`, run `transcript.find-issues` during the polish pass to see the actual candidates.

Only pass un-processed clips to each operation. If every clip is already transcribed, go straight to `transcript.get`.

| Operation | Default behaviour |
|---|---|
| `transcript.transcribe` | Run only on clips where `clipStates[i].transcribed === false`. Skip the rest. |
| `transcript.remove-fillers` | Auto-remove "um/uh/like/you know/i mean" + immediately-repeated words. Trim regions; fully reversible. |
| `transcript.find-issues` | Run after remove-fillers. Surfaces re-takes (`duplicate-take`), abandoned restarts (`false-start`), and stutters (`adjacent-repeat`) as candidates — each with the `wordIds` of the discarded attempt. **Read-only — it never edits.** **Default: keep the most recent (last) take and delete the earlier attempt** by feeding the candidate's `wordIds` into `transcript.delete-words`. Review against context first — if a repeat looks intentional (emphasis) or you can't tell which take is cleaner, ask the user which to keep rather than blind-applying. |
| `transcript.remove-silences` | Run after the content cleanup. Default threshold 500ms — the SAME threshold as the UI Remove Silences button, so your pass removes the same silences a manual click would. Don't hand-pick a higher value (e.g. 700) "to be safe" — that leaves dead air the user expects gone. Trims leading, between-word, and trailing silence per clip. |
| `audio.clean` | Denoise only clips where `clipStates[i].audioCleaned === false`. Writes a sibling `.cleaned.wav`; original audio untouched. |
| `caption.set-template` (when user said "add captions" without naming a style) | Default to `bold`. Tell user other templates exist (`classic, modern, minimal, spotlight, boxed, neon, colored, texture, editorial`). `texture` fills large uppercase words with a flowing texture mask — pick the texture (lava/marble/metal/wood/concrete/rock, default lava) via `caption.set-style --texture=marble`. `editorial` is magazine-style emphasis — the currently-spoken word renders big + accent while the rest of the line shrinks, so emphasis sweeps the line. |
| `llm.generate-title` / `llm.generate-description` / `llm.generate-timestamps` | Generate after the edit pass. Show the user; let them say "regenerate" or "use this exact title" or edit inline. |
| Specific zoom moments | Heuristically pick from the transcript ("you said 'click here' at 12.4s — adding a zoom"). Don't pre-ask. Iterate via preview. |
| Specific FX placement | Heuristically pick at clip boundaries or transcript hints ("a film-burn between clip 1 and 2"). Don't pre-ask. |

**Example of good narration after an edit pass:**

> "Edited.
> • Transcribed both clips (136 words).
> • Removed 14 fillers + 3 repeats. Trim regions are reversible — say 'undo fillers' if you want any back.
> • Cut 2 bad takes (kept the cleaner second attempt each time).
> • Removed 67 silences (>500ms), tightening ~48s of dead air.
> • Cleaned audio with DeepFilter on both clips.
> • Added a zoom at 12.4s where you said 'click here'.
> • Captions enabled with the bold template (8 other styles available).
> • Generated a title: *'How I Built This in 24 Hours'* — say if you want a different angle.
>
> Opening preview now."

Note how the summary names the **actual count for every step that ran** —
including silences. That's the bar: a summary line backed by a real verb call
and its returned result, never a vague "cleaned it up".

### NEVER ASK about

These produce friction with no upside. Just decide.

- Whether to remove filler words (it's reversible — do it)
- Whether to remove repeated words (same)
- Whether to clean audio (sibling file, original untouched)
- Specific zoom positions or focus points (infer; user redirects via preview)
- Specific caption colors / font sizes (template defaults are good)
- Whether to generate a title / description / timestamps (cheap, useful)
- Whether to enable captions when user said "add captions" (yes — they said yes)
- Specific lower-third design/colour (use defaults; user can swap)

### Preview, then export. Never export, then preview.

After a meaningful edit pass:

1. Call `preview.show --id=<uuid>` (opens the editor focused on the project — same single-step UX, ~2-3s).
2. Tell the user what you did (the narration block above).
3. Ask: *"Does this look right? Anything to tweak before I export?"*
4. **Only after explicit user confirmation** call `export.start`.

Export is 30-90s and produces a multi-MB MP4. Wasting an export because you skipped the preview is the worst UX failure in this surface. The editor's preview pane shows every effect / caption / FX / lower-third / motion graphic / zoom region exactly as the export will render them.

## Output modes

- **Default**: pretty one-line summaries to stdout. Show these to the user.
- **`--json`**: raw `{ ok, data, error }` envelope. **Always use `--json` when you intend to parse the response or chain commands** — pipe through `jq`.

## Discovery (the most important habit)

Don't memorise the command surface — the registry is the source of truth and it grows. Every PandaStudio launch self-describes:

```bash
pandastudio commands --json   # full schema with arg hints per command
```

Pattern-match `summary` against the user's intent. If you can't find a verb that fits, **say so** rather than fabricating one.

## Async jobs

`motion.generate` and any future `export.start` return a `jobId` immediately — the `data` block does **not** carry the result. Wait server-side:

```bash
pandastudio job.wait --id="$JOB" --timeoutMs=120000 --json
```

Terminal `job.status` is `succeeded | failed | canceled`. Read `result.outputPath` for the rendered MP4.

## Argument shape

Flags are either **scalars** (`--name=value`) or **JSON** (`--slots='{"title":"x"}'`). Anything starting with `{` or `[` is parsed as JSON. Strings stay strings; `true` / `false` / numbers auto-coerce.

## Error model

Every response: `{ ok: boolean, data?: ..., error?: string, details?: ... }`.

- HTTP 4xx/5xx → transport problem. CLI exits ≥ 1 to stderr.
- HTTP 200 + `ok: false` → handler-level error. CLI exits 1 with `error: <msg>` to stderr. The most useful machine-readable codes:
  - `license_required` / `trial_expired` — show the user the license activation flow
  - `unknown command` — typo; run `pandastudio commands` to recover
  - `invalid or out-of-tree project path` — project paths must live under the user's recordings dir; never pass arbitrary absolute paths

## Composing a real edit

> **HARD INVARIANT — every video the agent creates lives in a project.**
>
> Whatever the brief — promo, explainer, transcript-driven edit, PDF-to-video,
> a single motion graphic, B-roll over a recording, a one-shot title card —
> the **editor project is the deliverable**, not the raw MP4. Loose files in
> `~/Library/Application Support/pandastudio/recordings/` are *intermediates*
> the editor consumes; they are never the agent's hand-off.
>
> The two valid starting states:
>
> 1. A project is **already open** in the editor (`project.current` returns
>    a non-null id) → use it. Add your work to that timeline. Save. Preview.
> 2. No project is open (`project.current` returns null, or the chat opened
>    from the home screen) → **`project.new` is your FIRST tool call**, before
>    any motion render / transcription / b-roll generation. Name the project
>    something the user will recognise ("PandaScribe Promo", "Q4 Recap",
>    "How to install — explainer"). Pick the aspect ratio from the destination
>    profile (16:9 YouTube, 9:16 Shorts, 1:1 LinkedIn square).
>
> After the work lands in the project, `preview.show --id=<project-id>` to
> open it in the editor for the user. THAT is the hand-off. The chat message
> reports the project name/id, not a file path on disk.
>
> Exceptions are vanishingly rare — only when the user explicitly says
> "just give me the MP4, no project" (e.g. they want to upload it elsewhere
> right away). When in doubt, default to project.

Flow is always: **create or open a project → add things → save (conflict-safe)
→ preview.** Every verb's arg schema is discoverable at runtime — call
`system_list_commands` (MCP) or `pandastudio commands` (CLI), or see
`reference/commands.md`. So this section is the *judgment* the schema can't give
you: which verb, in what order, and the non-obvious gotchas.

### Target the right project

- **`project.new --withMedia='["/a.mp4","/b.mp4"]'`** — create pre-loaded; clip
  durations are FFmpeg-probed automatically.
- **`project.current`** → the open editor project (`{id,path,name,revision,clipCount}`
  or `null`). Use it when the user says "this one" / "what's open" — don't ask
  for an id. `null` ≠ "no projects exist"; fall back to `project.list`.
- **`project.read --id`** → full state. Key fields the schema won't spell out:
  clips at `mainTrack.clips[]`, overlays at `editor.mediaOverlayRegions[]`;
  `editedDurationMs` (post-trim — use for cadence planning), `sourceDurationMs`,
  `trimCount`, and per-clip `clipStates[].kind` (`camera` / `screen` / `upload`,
  your visual-strategy signal). Pass `--includeTranscript=false` after the first
  read.

### Adding things — the gotchas (call discovery for the arg schemas)

- **Clips:** `add-clip` (`--atIndex=0` prepends), `move-clip`, `split-clip`,
  `remove-clip`. **`project.delete` is permanent — no trash.**
- **Motion graphics — use `--fromJob`, NOT `--file`, for render outputs.** Pass
  the render `jobId` (after `job.wait`); the server resolves the path itself:
  ```bash
  JOB=$(pandastudio motion.render-html --htmlPath=/tmp/card.html --durationMs=4000 --json | jq -r '.data.jobId')
  pandastudio job.wait --id="$JOB" --json
  pandastudio project.add-motion-graphic --id=$ID --fromJob="$JOB" --durationMs=4000
  ```
  `--file` is only for external uploads and **must be quoted** — render outputs
  live under `…/Application Support/…` (a space); an unquoted path truncates and
  silently produces a dead overlay (shows on the timeline, never renders).
- **Placing on a spoken word in a trimmed timeline:** trims make source-time ≠
  output-time. From `transcript.get`, each word has `startMs` (source) and
  `editedStartMs` (output; `null` = inside a trim → skip it). Pass the **source**
  `startMs` as the position **and** `--anchorSourceMs=<same>` so the region
  re-anchors when later cleanup trims shift the timeline. `timeline.source-to-edited
  --sourceMs=N` returns the output time (or `null` if trimmed).
- **Mid-video graphic on camera / upload footage:** don't cover the host — use
  **`project.add-designed-segment`** (the split-panel beat; see Motion-graphics
  Rules §5). Screen recordings use `project.add-zoom` instead, never a split.
- **Zoom / lower-third / fx / SFX:** `add-zoom` (default depth 2 + swoosh SFX;
  `--soundUrl=none` to silence), `add-lower-third` (8 `--designType`s), `add-fx`,
  `set-region-sound` (retune/clear a placed region's SFX). Arg values: discovery.
- **Reset:** **`project.clear-edits`** — one atomic call wipes every region +
  audio overlays + turns captions off (keeps clips, transcript, aspect ratio;
  `--full=true` also resets LUT/crop/webcam/wallpaper). Use it for "start over";
  do NOT loop `project.remove-region` (that's for removing ONE region by
  `--regionType` + `--regionId`).

### Conflict-safe save

The editor autosaves, so two writers (you + the editor, or two agents) overwrite
each other silently unless you pass **`--expectedRevision`** (from
`project.read`'s `revision`). On conflict you get
`{ code:"revision_conflict", expected, actual, onDiskProject }` — re-read,
re-apply your change, retry. All `project.add-*` verbs accept it too.

### Preview without exporting

`preview.show --id` pops the live WYSIWYG overlay (~1–2s boot; `--atMs`,
`--autoplay`). `preview.seek --atMs` moves the playhead; `preview.hide` closes;
`preview.list` inspects it. **Call `preview.show` after every significant edit**
so the user sees the change live without leaving the chat. (`project.open` opens
the full editor — heavy, for handing off to the user.)

## Motion graphics

PandaStudio ships a curated set of **YouTube-creator templates**. They are
the primary way to add motion graphics — production-grade, editable, and
faster than authoring HTML. Custom HTML (`motion_render_html`) is the
fallback for briefs no template fits (see the next section).

### Rules — recommendations, not rigid law (bias toward DOING)

These are strong defaults, not handcuffs — use your judgment. The worst outcome
is a near-empty timeline because you were being cautious. **A video full of
points should be full of graphics.**

1. **`motion_list` FIRST, every time.** Discover the catalog + each template's
   editable slots at runtime; don't generate from memory or copy a templateId
   out of an example.
2. **Add a graphic on most meaningful beats — don't be shy.** Name-drops,
   claims, numbers, lists, comparisons, tool/product mentions, section changes
   — each is a candidate. If a 5-minute video has ten clear points, ~ten
   graphics is reasonable. **Under-graphicking is as much a failure as the
   wrong graphic** — when a beat clearly wants a visual, add one.
3. **Repetition is good — it creates consistency.** Reusing one title style for
   every chapter, or `split-panel` across many explainer beats, makes a video
   feel *designed*, not random. Never skip a template just because you already
   used it. (The earlier "don't repeat" guidance was wrong — ignore it.)
4. **The ONE hard line: never misuse a purpose-specific template.** A few
   templates *mean* something — use them only when the content matches:
   `stat-reveal` → a real number (never a generic title) · `comparison` →
   exactly two things contrasted · `flowchart` → an actual sequence of steps ·
   `key-takeaways` → a list of points · `yt-lower-third` → introducing a
   person/channel. Everything else (the title family, `split-panel`, the
   parallax reveals) is **generic — reuse freely.** See the class column below.
5. **Camera-only / imported footage → lead with `split-panel`.** When the
   clip's `kind` is `"camera"` (talking-head) or `"upload"` (imported), there's
   no screen to zoom into, so a static frame is what makes it look flat. The
   `split-panel` designed segment (host one half, brand panel the other, via
   `project.add-designed-segment`) is the highest-leverage fix and your
   **default workhorse** for explainer beats on this footage — use it liberally
   (alternate side + content), with the other templates sprinkled around it.
   For `kind === "screen"`, prefer cursor-telemetry zooms; don't split the frame.
6. **Text isn't your only option.** When a beat wants a *visual* — logos for the
   tools being named, a product screenshot, an animated diagram — author a
   custom graphic (see "Authored graphics" below). Don't force every beat into a
   text template.

### Selection guide (beat → template)

"Class" tells you how freely to reuse it: **Generic** = reuse anytime · **Purpose**
= only when the content matches · **Semi-generic** = reusable, has a natural fit.

| What's happening in the video | Reach for | Class |
|---|---|---|
| Open / chapter / section title | `creator-card`, `transitions-3d`, `grain-overlay`, `transitions-destruction`, `caption-parallax-layers` — pick a look, reuse it per section | **Generic** — repeat freely |
| Explainer beat, host on camera/imported footage | `split-panel` | **Generic workhorse** — repeat freely |
| Hero reveal / intro / "ways to use it" recap | `parallax-zoom`, `parallax-unzoom` | Semi-generic |
| Introduce a person / channel / "subscribe" | `yt-lower-third` | Purpose |
| A real number / metric / result | `stat-reveal` | **Purpose — numbers only** |
| "Here are the N things…" / key points / recap | `key-takeaways` | Purpose |
| This vs that / before vs after / old vs new | `comparison` | Purpose |
| A process / steps / workflow | `flowchart` | Purpose |
| ONE thesis sentence / pull-quote / "money line" (hook or climax) | `caption-editorial-emphasis` | **Purpose — at most 2–3 per video** |
| Logos / tools / partners · a screenshot · a diagram (a VISUAL, not text) | **Author a custom graphic** — see "Authored graphics" below | Authored (not a UI template) |

### Authored graphics — your repertoire is bigger than the gallery

The 13 templates above are the **UI gallery**: fixed-structure, slot-fill. But
your repertoire is larger — you can also **author content-specific graphics**
that *can't* be slot-templated because what they show depends on what's being
discussed. Build these as transparent overlays with `motion.render-html`; they
composite over the host exactly like an overlay template. Reach for one whenever
a beat needs a *visual* rather than words:

- **Logo / brand-card row** — N rounded white cards, each a logo, popped in over
  the lower third. For "we use X, Y, Z", tool / partner / integration mentions
  (e.g. HeyGen · Claude Code · Hyperframes).
- **Image / screenshot showcase** — real images via `--assets` (product shots,
  UI grabs) in a framed or tilted card.
- **Animated diagram / chart** — data-driven SVG (a flow, a metric building, a
  bar) when the point is structural, not a single number.
- **Icon / emoji concept callout** — a glyph + short label punched on a concept.
- **Reuse a template's shell, swap text → graphics** — take the *look* of an
  overlay template (the lower-band card, the side panel, the depth stack) and
  put logos / images / animated SVG where the text would go.

These never appear in the UI gallery (they're not in the manifest) — they're
yours to consider whenever a slot-fill template doesn't capture the beat.
Copy-able recipes (logo-card row, etc.) live in
[`reference/examples.md`](reference/examples.md); the authoring contract (page
shell, deterministic seek, transparent overlays) is in
[`reference/motion-philosophy.md`](reference/motion-philosophy.md).

### The workflow

```bash
# 1. ALWAYS discover first — templates + editable slots (runtime source of truth).
pandastudio motion.list --json          # MCP: motion_list

# 2. Pick the template that fits THIS beat (see the selection guide), then
#    render it with your own text/colors + a background mode.
#    Replace <TEMPLATE_ID> + slots with the chosen template's — do NOT
#    hardcode one template for every insert.
JOB=$(pandastudio motion.generate \
  --templateId=<TEMPLATE_ID> \
  --slots='{ ...the chosen template's slots from motion.list... }' \
  --aspectRatio=16:9 \
  --json | jq -r '.data.jobId')          # MCP: motion_generate
pandastudio job.wait --id="$JOB" --json

# 3. Add the rendered clip to the timeline (at the playhead by default).
pandastudio project.add-motion-graphic --id="$PROJECT" --fromJob="$JOB" --durationMs=4000
```

- **Editable everything** — every template's text, colors, and list items
  are `slots`; pass only the ones you want to change, the rest use defaults.
  `motion.list` returns each template's slot keys, types (`string` / `color`
  / `list`), and defaults.
- **`--fromJob` not `--file`** — pass the render `jobId` to the add tool; it
  resolves the path internally (hand-built paths truncate at the space in
  "Application Support" and silently fail).
- **Placement** — `add-motion-graphic` drops it at the playhead/end as an
  overlay. To re-time, pass `atMs`. Anchor to a transcript word with
  `anchorSourceMs` so it survives later transcript edits.

### Background modes (`--background`)

| Mode | Output | Use |
|---|---|---|
| `solid` | hard full-frame card | opaque templates (cards, stat, checklist, comparison) — the default for those |
| `transparent` | alpha; camera shows through un-painted pixels | overlay templates — the default for those |
| `glass` | alpha + frosted blur of the camera behind the content | overlay templates when you want the modern frosted look; also pass `--backdropBlurStrength=24` on `add-motion-graphic` |

Omit `--background` to use the template's natural mode. **Never force
`solid` on an overlay template** — its see-through region renders black.
Overlay templates are flagged `overlay: true` in `motion.list`.

### Designed segments (host on one half, panel on the other)

`split-panel` is a "designed segment": an opaque brand panel fills one half,
the other half is transparent for the host. Add it with
**`project.add-designed-segment`** instead of `add-motion-graphic` — that
one atomic call also repositions the camera into the open half (a
`cam-{side}-50` clip-transform) over the same span, so host + panel can't
drift. Set the panel's `side` slot; the camera takes the opposite side.

**`--durationMs` is how long the panel stays** — set it to the length of the
topic it covers, not a fixed few seconds. The panel **reveals once and holds**:
it has no exit animation, and the overlay holds its last frame for the whole
region (it does NOT loop), so a 30-second explainer beat is just
`--durationMs=30000`. The region end is what removes it. (Same for any overlay:
play-once + hold; size the region/durationMs to the on-screen time you want.)

```bash
JOB=$(pandastudio motion.generate --templateId=split-panel \
  --slots='{"side":"left","headline":"What MyAgentMail handles:","items":[{"label":"inbox creation"},{"label":"sending & replies"},{"label":"webhooks"}]}' \
  --background=transparent --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json
# durationMs = full length of the topic the panel explains (here 28s):
pandastudio project.add-designed-segment --id="$PROJECT" --fromJob="$JOB" \
  --durationMs=28000 --cameraSide=right
```

### Template catalog

Pick by brief. Slots in **bold** are required; the rest are optional
(colors default to PandaStudio's electric-blue brand `#2563EB` where unset).
All are 16:9 / 9:16 / 1:1 unless noted. `O` = overlay (transparent-capable).

**Titles & chapter cards**
- `creator-card` (4.5s) — full-bleed brand title / chapter card with a live dot. Bold YouTube-creator look. Slots: **headline**, eyebrow, brandColor, inkColor, dotColor.
- `transitions-3d` `O` (4.5s) — editorial title that flips in with a 3D rotate: small lead / HUGE word / small trail. Slots: lead, **emphasis**, trail, textColor, accentColor.
- `grain-overlay` `O` (5s) — bold uppercase headline + kicker with a flickering film-grain texture over the footage. Slots: kicker, **headline**, textColor.
- `transitions-destruction` `O` (5s) — headline assembles from slabs, holds, then shatters apart. Dramatic reveal/transition. Slots: kicker, **headline**, accentColor, textColor.
- `caption-parallax-layers` `O` (5s) — one word stacked in 5 depth layers (solid front + receding ghosts) that enters with a vertical stretch and parallaxes. Slots: eyebrow, **headline**, frontColor, ghostColor.

**Lower third**
- `yt-lower-third` `O` (4.5s) — subscribe lower-third: avatar + name + title + red Subscribe pill, slides in bottom-left. Slots: **name**, title, accentColor, cardColor, inkColor.

**Data & explainer**
- `stat-reveal` (4s) — a huge number that counts up, with eyebrow + optional prefix/suffix + label. "10,000+ subscribers", "3× faster". Slots: eyebrow, prefix, **value**, suffix, **label**, brandColor, inkColor, accentColor.
- `key-takeaways` (6s) — titled checklist; points reveal one-by-one with drawn ticks. The "here's what you need to know" recap. Slots: **title**, items[label], bgColor, accentColor, inkColor.
- `comparison` (5s) — two labeled panels slide in with a VS badge; the right (winning) panel pulses. Before/after, old-way/new-way. Slots: leftTag, **leftHeading**, leftCaption, rightTag, **rightHeading**, rightCaption, bgColor, leftColor, rightColor, inkColor.
- `flowchart` (6s) — horizontal node cards joined by drawn arrows, revealed step by step. Process / workflow. Slots: nodes[label], brandColor, cardColor, inkColor, accentColor.

**Reveals & recaps**
- `parallax-zoom` (5s) — a center hero card zooms to fill while four corner satellites parallax out. Hero reveal / intro. Slots: **headline**, items[label], brandColor, cardColor, inkColor.
- `parallax-unzoom` (4.5s) — a focus card unzooms to reveal a 2×2 grid as the others parallax in. "ways to use it" / features recap. Slots: items[label], brandColor, cardColor, inkColor.

**Host + panel split**
- `split-panel` `O` (16:9 / 1:1) — opaque brand panel on one half (headline + bullet list), transparent on the other for the host. **Reveals once and holds** (no exit) — set `--durationMs` to the topic length; it holds for exactly that long. Add via `project.add-designed-segment` (see above). Slots: side (`left`/`right`), **headline**, items[label], brandColor, inkColor.
  > **Designed segments are coupled.** As of v1.35.0, the camera layout transform and the panel motion graphic created by `project.add-designed-segment` share a `linkGroupId`. Moving one on the timeline shifts the other by the same delta; deleting one deletes its peer. The agent's mental model can stay simple ("this is one beat"), and direct-manipulation in the UI never produces an orphan transform with no panel to balance it.

**Pull-quote / emphasized line**
- `caption-editorial-emphasis` `O` (4s, all aspects) — one-sentence pull-quote, ONE word (or short phrase) blown up in huge Playfair italic that slides in from the left. Regular words pop in word-by-word in Inter; emphasis slides in below them; holds the final frame. Transparent overlay — drops on any clip. Slots: **sentence**, **emphasisWord**, inkColor, accentColor. Trailing punctuation after the emphasis auto-merges onto the emphasis (so `"…starts with a single frame."` renders `single frame.` as one unit). If `emphasisWord` is NOT a substring of `sentence`, it's appended to the end as a punchline — useful when you want the sentence to LEAD with normal text and END with the dramatic pull-out. **Use sparingly: at most 2–3 times per video, and reserve one of those for the climax / "money line" of the piece.** The opening hook and the chapter-closing payoff are the strongest slots; sprinkling it every minute burns the size contrast. Not a replacement for the running caption track (`caption.set-template editorial` is the style for that). Add via `project.add-motion-graphic` (it's an overlay, not a main-track clip).
  ```bash
  JOB=$(pandastudio motion.generate --templateId=caption-editorial-emphasis \
      --slots='{"sentence":"Every great video starts with a single frame.","emphasisWord":"single frame"}' \
      --aspectRatio=16:9 --json | jq -r '.jobId')
  pandastudio job.wait --id=$JOB
  pandastudio project.add-motion-graphic --fromJob=$JOB --startMs=3000 --durationMs=4000
  ```

> List slots (`items`, `nodes`) take an array of objects, e.g.
> `"items":[{"label":"first"},{"label":"second"}]`. `motion.screenshot`
> renders a single frame of any template+slots combo if you want to preview
> before committing to a full render.

## Custom motion graphics — HTML authoring (fallback, only when no template fits)

Reach for this ONLY when no bundled template fits the brief (a bespoke one-off,
unusual layout, brand-specific 3D). For the common cases, use the templates
above — faster and already designed.

> **CRITICAL: render scene-by-scene, NOT one big 30s MP4.** If the brief is
> multi-scene (a promo / explainer with ≥3 distinct beats, anything ≥10s
> total), author **each scene as its own ~5-8s HTML composition** and add
> each as a sequential clip on the timeline (via `project.add-clip` for
> from-scratch content, or via `project.add-motion-graphic` only when the
> graphics are layered on top of host footage — see "Two distinct flows"
> below for the disambiguation). Do NOT build one monolithic 30s
> composition. Three reasons:
>
> 1. **Targeted re-renders.** When the user says "scene 3 needs the brand
>    color brighter," you redo that one scene (~30-60s), not the whole
>    promo (~10 min).
> 2. **Editor primitives apply per-clip.** Each scene gets its own trim,
>    speed, zoom, FX, layout transform — wasted on a single mega-clip.
> 3. **Render reliability.** `motion.render-html` for a 30s @ 1080p
>    composition pushes memory + capture-time hard. Five 6-second renders
>    are cheaper individually AND parallelizable through the render pool.
>
> Each scene's HTML is **reveal + hold** only — no baked-in exit tweens.
> The editor handles transitions between clips via `project.add-fx`
> crossfades (or just hard cuts). The full pattern + worked example lives
> in `reference/motion-philosophy.md` → "Multi-scene authoring".
>
> Single-scene graphics (one lower-third, one title card, one stat reveal)
> stay a single render → single clip. The rule above is for multi-scene
> compositions only.
>
> The escape hatch — `motion.concat` — exists for when the user explicitly
> asks for one combined MP4. It's not the default for promos.

> **Two distinct flows. Pick the right one BEFORE adding clips.**
>
> Motion graphics live in PandaStudio in two places, and which one you use
> depends on whether there's host footage:
>
> 1. **Generated video from scratch** — promo / explainer / PDF-to-video /
>    any brief where YOU are producing the primary content (no host on
>    camera, no uploaded recording). The rendered scene MP4s **ARE the
>    video**. They go on the **main track** via `project.add-clip`, one
>    clip per scene. The timeline IS your scene sequence.
>
> 2. **Layered on existing footage** — user has a recording open (host
>    talking on camera, screen capture, etc.) and wants motion graphics
>    composited ON TOP (lower thirds, stat callouts, brand watermarks,
>    side-panel explainers). The recording's clips are already on the
>    main track. Motion graphics go in `mediaOverlayRegions` via
>    `project.add-motion-graphic`, sitting above the host video at
>    specific time windows.
>
> Picking the wrong verb produces a broken project: `project.add-motion-graphic`
> on an EMPTY main track produces an editor that opens to "No video to load"
> because overlays compose ON the main track, and there's nothing there.
>
> **Chain for a brand-new promo / explainer / video-from-scratch**
> (the from-scratch case — use `project.add-clip`):
>
> ```bash
> # 1. Create a fresh project. Set aspectRatio to match the destination
> #    profile (16:9 YouTube, 9:16 Shorts/Reels, 1:1 LinkedIn square).
> P=$(pandastudio project.new --name="PandaScribe Promo" --aspectRatio=16:9 --json | jq -r '.data.id')
>
> # 2. Render each scene as its own motion graphic, then add the rendered
> #    MP4 to the MAIN TRACK with project.add-clip. NOT add-motion-graphic
> #    — that's for overlays on existing footage, and there's no footage
> #    here. add-clip ffmpeg-probes the MP4 duration automatically.
> for SCENE in intro problem demo testimonial cta; do
>   # Author the HTML for $SCENE somewhere (inline or tmp file)…
>   JOB=$(pandastudio motion.render-html --htmlPath="/tmp/$SCENE.html" --durationMs=6000 --json | jq -r '.data.jobId')
>   RESULT=$(pandastudio job.wait --id="$JOB" --timeoutMs=600000 --json)
>   OUTPATH=$(echo "$RESULT" | jq -r '.data.job.result.outputPath')
>   pandastudio project.add-clip --id="$P" --media="$OUTPATH"
> done
>
> # 3. Open the project in the editor so the user can preview, scrub,
> #    tweak individual scenes, export. THIS is the hand-off, not the
> #    bare MP4 paths.
> pandastudio preview.show --id="$P"
>
> # 4. Report to the user in chat with the project id (NOT the
> #    intermediate MP4 paths). Example:
> #    "Created PandaScribe Promo with 5 scenes on the timeline. Open in
> #     the editor — tell me which scene to tweak."
> ```
>
> Same flow applies when a project IS already open with no main-track
> content yet: skip step 1, reuse the existing `$P` from `project.current`.
>
> **Chain for layered graphics on existing footage** (host video already
> on the main track — use `project.add-motion-graphic`):
>
> ```bash
> # The user is editing a recording — clips are already on the main
> # track. Add a lower-third at 3.5s.
> JOB=$(pandastudio motion.generate --templateId=lower-third-clean \
>   --slots='{"name":"Kamal Kannan","title":"Founder, PandaStudio"}' --json | jq -r '.data.jobId')
> pandastudio job.wait --id="$JOB" --json
> pandastudio project.add-motion-graphic --id="$P" --fromJob="$JOB" \
>   --atMs=3500 --durationMs=4000
> ```

You author HTML/CSS/JS and render it with `motion.render-html` (the HyperFrames
engine — frame-perfect, seekable capture). The page MUST satisfy three things
(the full contract, canonical shell, easing dictionary, and three.js seek live
in `reference/motion-philosophy.md` — read it before authoring):

1. A composition root with `data-composition-id`, `data-width`, `data-height`,
   `data-duration` (seconds, float ok).
2. A **paused** GSAP timeline (`gsap.timeline({ paused: true })`) with a Law-#11
   anchor tween spanning the full duration. NO CSS keyframes, NO setTimeout/rAF,
   NO `repeat:-1` (Hyperframes seeks deterministically; infinite repeats break).
3. Register it: `window.__timelines["<data-composition-id>"] = tl;`

### Render verbs

```bash
# Pre-flight ONE frame before a full render (sub-second; catches layout bugs).
pandastudio motion.screenshot --htmlPath=/tmp/scene.html --atMs=1500 --json

# Render to MP4 (opaque). Renders are SEQUENTIAL — job.wait before firing the
# next or you get RENDER_BUSY. For many scenes, fire in parallel + job.wait each.
JOB=$(pandastudio motion.render-html --htmlPath=/tmp/scene.html --durationMs=4000 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json

# Transparent overlay (lower third / watermark / name plate): author with
# `background: transparent`, add --transparent → WebM + VP9 alpha.
# Frosted glass: render --transparent, then on add pass --backdropBlurStrength=24
# (12 subtle · 36+ heavy) + --backdropBlurTint='rgba(10,10,10,0.18)'.

# Add by jobId — never hand-build the output path (truncates at the space in
# "Application Support" and silently fails).
pandastudio project.add-motion-graphic --id=$ID --fromJob="$JOB" --durationMs=4000
# Multiple scenes → motion.concat the rendered MP4s into one before adding.
```

**References.** Authoring is **brand-driven** as of v1.32 — the editor-context
snapshot includes the user's brand kit (colors, type, voice, logo) on every
turn. Read those values verbatim and let them decide aesthetics; don't reach
for a hard-coded default look. Authoring contract:

- [`reference/motion-philosophy.md`](reference/motion-philosophy.md) — the
  authoritative correctness + craft contract (timeline anchor, determinism,
  layout-before-animation, scene transitions, quality checks). **Always read
  before authoring custom HTML.** Recently rewritten to be brand-agnostic; old
  prescriptions (dark canvas / chrome gradients / vignette + grid + grain as
  defaults) are gone.
- [`reference/house-style.md`](reference/house-style.md) — neutral fallback
  defaults when the brand kit is partial or absent. Three named tracks
  (creator-bright, dark-premium, product-clean) presented as STARTING points
  the user picks between, not as a default to silently apply. Voice → motion
  defaults table.
- [`reference/examples.md`](reference/examples.md) — concrete recipes
  (faux-cursor click, parallax-zoom, grid-pixelate-wipe, three.js setup).

**Quality gate.** The **user is the final reviewer** of every render.
Do NOT auto-call `motion.screenshot` or `motion.verify-frames` as part
of your normal authoring flow — the human will open the MP4 in the
editor and catch any issue in two seconds, faster and more reliably
than the agent can. Your job is to author the composition with care,
walk the brand checklist textually before rendering (colors derived
from `brand.colors`, fonts from `brand.typography`, voice matches
`brand.voice`, logo from `brand.logoPath` when on screen), then render
and hand off. If the user comes back saying "scene 3 is broken" or
"the color's wrong," fix it and re-render — don't pre-emptively burn
turns inspecting frames.

`motion.screenshot` and `motion.verify-frames` are still available as
tools for when the user explicitly asks ("preview scene 2 at the 3-second
mark," "show me 8 frames across the timeline"). Just don't reach for
them on your own.

Upstream engine docs — canonical for engine internals: <https://hyperframes.heygen.com>.

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

## Transcript-based editing — PandaStudio's signature feature

The reason humans pick PandaStudio over Premiere is that you edit by **deleting words from the transcript**, not by scrubbing the timeline. The CLI exposes the same model.

### The full edit loop

```bash
# 0. Check which clips still need processing (avoids clobbering in-app edits)
STATE=$(pandastudio project.read --id=$ID --json | jq '.data.clipStates')
# clipStates: [{ clipId, transcribed, wordCount, audioCleaned }, ...]

# 1. Transcribe only clips that don't already have a transcript
#    (if all are transcribed, skip this step entirely)
JOB=$(pandastudio transcript.transcribe --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=300000 --json

# 2. Pull the merged transcript — every word with edited-time start/end
pandastudio transcript.get --id=$ID --json | jq '.data.words[0:20]'

# 3a. AUTO: drop every "um" / "uh" / "you know" + immediate repeats
pandastudio transcript.remove-fillers --id=$ID --json
# → returns { removedCount, fillersRemoved, repeatsRemoved, trimsAdded }

# 3b. Remove silences (default ≥500ms = the UI button; covers leading/trailing/between-word)
#     Omit --thresholdMs to use the 500ms default; only pass it to override.
pandastudio transcript.remove-silences --id=$ID --json
# → returns { removedCount, totalTrimmedMs }

# 3c. Fix STT errors — NEVER use project.read → JSON mutation → project.save for this.
#     find-replace patches the word text in-place and preserves timing.
pandastudio transcript.find-replace --id=$ID --find="RightPanda" --replace="WritePanda" --json
# → returns { replacedCount, wordsPatched }

# 3d. SURGICAL: delete specific words by ID
pandastudio transcript.delete-words --id=$ID --wordIds='["clip-1:w-42","clip-1:w-43"]' --json

# 3e. PHRASE search → bulk delete
WORDS=$(pandastudio transcript.search --id=$ID --query="this is a test" --json \
  | jq -c '[.data.matches[].wordIds | .[]]')
pandastudio transcript.delete-words --id=$ID --wordIds="$WORDS" --json
```

Every deletion translates internally into a **trim region** the export pipeline skips. It's identical to clicking the word in the editor's transcript pane and hitting delete.

**`transcript.get` shows ALL words, including ones you've deleted.** Deleted words become trim regions — they're gone from the audio export — but they still appear in the raw word list. If you need to verify a deletion happened, check `trimsAdded` in the response rather than calling `transcript.get` afterwards and looking for missing words.

**STT coherence with motion graphics**: Fix all transcript errors with `transcript.find-replace` BEFORE calling `motion.generate` or `llm.generate-title`. The local LLM and motion-graphic slot values are derived from the transcript text — a "RightPanda" in the transcript will propagate into the title card if you generate it first.

### Audio cleanup (DeepFilter)

```bash
# Check clipStates first — skip if all clips are already cleaned
JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json
# → only un-cleaned clips are processed; already-cleaned clips are skipped automatically
# → each processed clip gets a sibling .cleaned.wav file; export auto-uses it
```

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

# 5) Remove
pandastudio project.remove-audio --id=$ID --overlayId=audio-1 --json
# or: pandastudio project.remove-region --regionType=audio-overlay --regionId=audio-1
```

**Arg precedence for duration** (matches the primitive):

1. Explicit `--endMs` — always wins
2. Legacy `--maxDurationMs` — `endMs = startMs + maxDurationMs`
3. `--durationMs` fallback
4. Probed real file duration (when nothing else is specified)

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

**Current library (v2):**

| id | category | mood | intents | recommendedFor |
|---|---|---|---|---|
| `tech-review-background` | corporate | energetic | tech_review, tutorial, explainer, saas_walkthrough | youtube-long, linkedin |
| `chill-vlog-ambience` | lofi | calm | vlog, day_in_life, lifestyle, ambient_underscore | youtube-long, shorts |
| `kinetic-product-drive-a` | electronic | energetic | product_video, kinetic_text, promo, intro, outro, product_reveal, motion_graphics | youtube-long, shorts, linkedin |
| `kinetic-product-drive-b` | electronic | energetic | product_video, kinetic_text, promo, intro, outro, product_reveal, motion_graphics | youtube-long, shorts, linkedin |
| `generic-underscore-a` | generic | neutral | generic, background, under_voiceover, default | youtube-long, linkedin, loom |
| `generic-underscore-b` | generic | neutral | generic, background, under_voiceover, default | youtube-long, linkedin, loom |

**Intent → track selection (use unless the user specifies a track):**

- Product video / product demo / product reveal → `kinetic-product-drive-a` or `-b`
- Kinetic text / motion graphics / promo → `kinetic-product-drive-a` or `-b`
- YouTube **intro** or **outro** → `kinetic-product-drive-a` or `-b` (energetic hook; trim to length)
- Tech review / tutorial / explainer / SaaS walkthrough → `tech-review-background`
- Vlog / day-in-life / lifestyle → `chill-vlog-ambience`
- Anything else / don't-know / "just add music" → one of the `generic-underscore-*` tracks
- **LinkedIn / Loom:** prefer `generic-underscore-*` (neutral, won't distract from message) — never use `kinetic-product-drive-*` on LinkedIn unless the user's brief is explicitly promo/reveal

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

**Full cinematic workflow:**

```bash
# Create project, add clip, grade it, add music, preview
P=$(pandastudio project.new --name="Cinematic Short" \
  --withMedia='["/path/footage.mp4"]' --json)
ID=$(echo "$P" | jq -r '.data.id')
CLIP=$(echo "$P" | jq -r '.data.project.mainTrack.clips[0].id')

# Apply cinematic teal-orange grade
pandastudio project.set-clip-lut \
  --id=$ID --clipId="$CLIP" \
  --lutPreset=cinematicTealOrange --lutIntensity=0.85 --json

# Add bundled lofi music
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

## Visual edits — zooms, trims, speed, annotations, style

```bash
# Zoom into a UI element from t=5s for 1.5s
pandastudio project.add-zoom --id=$ID --atMs=5000 --durationMs=1500 \
  --depth=4 --focusX=0.3 --focusY=0.5

# Cut a section directly (without going through transcript)
pandastudio project.add-trim --id=$ID --startMs=12000 --endMs=15000

# Fast-forward a setup step
pandastudio project.add-speed --id=$ID --startMs=8000 --endMs=20000 --speed=2

# Drop a text annotation
pandastudio project.add-annotation --id=$ID --startMs=2000 --endMs=4000 \
  --type=text --text="Look here →" --x=50 --y=30

# Switch aspect ratio (incl. 9:16 for Shorts)
pandastudio project.set-aspect-ratio --id=$ID --ratio=9:16

# Apply cinematic style
pandastudio project.set-style --id=$ID --padding=40 --shadowIntensity=30 \
  --borderRadius=20 --motionBlurAmount=15

# Pick a wallpaper
pandastudio project.set-wallpaper --id=$ID --wallpaper=gradient-night

# Reframe the main recording (crop, all values normalized 0-1)
pandastudio project.set-crop --id=$ID --x=0.1 --y=0.05 --width=0.8 --height=0.9

# Webcam overlay — preset or manual position
pandastudio project.set-webcam-layout --id=$ID --preset=picture-in-picture
# presets: none | picture-in-picture | vertical-stack | side-by-side
pandastudio project.set-webcam-layout --id=$ID --cx=0.85 --cy=0.85 --scale=0.35
pandastudio project.set-webcam-layout --id=$ID \
  --cropX=0 --cropY=0.1 --cropWidth=1 --cropHeight=0.8  # remove letterbox bars

# Update any placed region in-place (patch only what changes)
pandastudio project.update-region --id=$ID \
  --regionType=zoom --regionId=zoom-1 --depth=2 --focusX=0.6
pandastudio project.update-region --id=$ID \
  --regionType=lower-third --regionId=lt-1 \
  --content="Kamal Kannan" --accentColor="#00ff88"
pandastudio project.update-region --id=$ID \
  --regionType=annotation --regionId=ann-1 --text="Updated text" --y=20
pandastudio project.update-region --id=$ID \
  --regionType=fx --regionId=fx-1 --opacity=0.5 --endMs=5000
pandastudio project.update-region --id=$ID \
  --regionType=audio-overlay --regionId=audio-1 \
  --startMs=2000 --endMs=15000 --sourceStartMs=4000 --volume=0.55
# regionType: zoom | trim | speed | annotation | fx | lower-third | overlay | audio-overlay

# Export defaults (pre-fills the Export dialog; CLI export.start uses its own --quality)
pandastudio project.set-export-settings --id=$ID --quality=source --format=mp4
pandastudio project.set-export-settings --id=$ID --format=gif \
  --gifFrameRate=30 --gifLoop=true --gifSizePreset=large
```

## Captions

```bash
pandastudio caption.set-template --id=$ID --templateId=neon
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-style --id=$ID --color="#fff" --highlightColor="#34B27B" \
  --strokeWidth=3 --strokeColor="#000" --positionY=85
```

Templates: `classic | modern | minimal | bold | spotlight | boxed | neon | colored | texture | editorial`. Captions read words from the project's merged transcript — so you must transcribe first.

- `texture` fills large uppercase words with a flowing texture mask (lava/marble/metal/wood/concrete/rock) — pick the texture via `caption.set-style --texture=marble` (default lava).
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

These are READ-ONLY — they return text for you to display or stash on the export-library entry; they don't mutate the project. Perfect for auto-populating the YouTube upload box after `export.start` finishes.

## YouTube thumbnails (v1.18+)

PandaStudio can generate YouTube thumbnails via Replicate's `openai/gpt-image-2` model using **the user's own Replicate API key** — PandaStudio never pays for or proxies these calls. Before any thumbnail verb will work, the user must open **Settings → Integrations** and paste a key from https://replicate.com/account/api-tokens. The key is stored encrypted via the OS keychain (Keychain on macOS, DPAPI on Windows, libsecret on Linux).

Requires a Replicate key set in **Settings → Integrations** (not via CLI — by
design, so it never lands in shell history). If a verb returns "No Replicate API
key set", tell the user to add one rather than looping.

**Verbs:**
- `export.generate-thumbnail --id=$EID` — local LLM writes a prompt from the
  transcript → gpt-image-2 (3:2 WebP). Add `--prompt="…"` for control, or
  `--referenceImagePath=…` to anchor on a face/logo. (Needs the export in the
  library + a transcript.)
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

## Export — produce the final MP4

The centerpiece. Routes through the same Tier-3 PixiJS renderer the editor's Export Video button uses (v1.24+ — was a separate Skia native pipeline before that, see release notes for the convergence). When an editor window is already open on the project you're exporting, the agent reuses it. Otherwise the agent spawns a hidden editor window for the duration of the render and closes it after. **Async; poll `job.wait`.**

```bash
JOB=$(pandastudio export.start --id=$ID --quality=high --json \
  | jq -r '.data.jobId')

# Watch progress (server-side block; returns when done or 5min timeout)
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json | jq '.data.job'
# → status: "succeeded", result: { outputPath, durationMs, width, height, frameRate }
```

Quality presets: `draft` (1280×720), `standard` / `high` (1920×1080), `ultra` (3840×2160). Aspect ratio comes from the project (`set-aspect-ratio`). Output lands in the recordings dir by default; pass `--outputPath=/somewhere/file.mp4` to override.

The export honours **everything** in the project: clips, trims (incl. those from transcript word deletes), speed regions, zooms, captions, FX, lower-thirds with sound, motion graphics, annotations, cleaned audio, wallpaper, padding/shadow/radius/blur. One verb, full pipeline.

**Video overlays (motion graphics) are fully composited in the export** — both opaque MP4 (`motion.generate`, `motion.render-html`) and transparent WebM (`motion.render-html --transparent`) are composited inline by the Tier-3 pipeline. Alpha channels from VP9/WebM sources are preserved exactly. There is nothing extra you need to call — `export.start` handles it automatically once overlays are on the timeline via `project.add-motion-graphic`.

## Video editing playbook — end-to-end recipe (per destination)

When the user says *"edit this"* / *"polish this"* / *"make this ready for <X>"* / *"YouTube-ready"*, follow this runbook. It turns a raw recording into a polished, destination-appropriate video using the foundational verbs above. Different destinations (YouTube long-form, Shorts/TikTok, LinkedIn, Loom) need different defaults — the table below is the source of truth.

**Philosophy:** good video editing is a series of pattern interrupts that match the platform's viewing context. A YouTube long-form viewer has settled in — cuts every 5–8s and a cinematic LUT feel right. A TikTok viewer is scrolling — you have 3 seconds to hook them and every second after needs a visible change. A LinkedIn viewer is at work — an aggressive soundscape is wrong. A Loom viewer doesn't want any editing at all beyond "cut the fluff". Same tools, very different dials.

### Destination profiles (the source of truth)

Resolve the destination first (see [HARD-GATE](#editorial-decisions--what-to-ask-what-to-assume-what-never-to-ask) step 1). Then apply every default below from the matching row — don't mix.

| Parameter | `youtube-long` | `shorts` (Shorts/TikTok/Reels) | `linkedin` | `loom` (internal/async) |
|---|---|---|---|---|
| Aspect | 16:9 | **9:16** | 16:9 or 1:1 | 16:9 |
| Hook deadline | 10 s | **3 s** | 10 s | — (none) |
| Intro / outro card | **only if the user asks** (then 2–4 s) | **only if the user asks** | **only if the user asks** (then 2–3 s) | none |
| Lower thirds | yes, at first mentions | **no** (too small vertically) | yes | no |
| Zoom cadence | 3–6 / min | **6–12 / min** | 1–2 / min | 0–1 / min |
| Default emphasis zoom duration | **7 s** | 3 s | 4 s | 2 s |
| Sustained held-zoom duration (section reframe) | **15 s** | — | 8 s | — |
| Agent zooms on screen-share clips | **NEVER** (telemetry handles it) | **NEVER** | **NEVER** | **NEVER** |
| Zoom SFX volume | 1.0 (swoosh-fast) | **1.0 (swoosh-fast)** | 0.5 (or `none`) | `none` |
| Filler/silence removal | yes | yes | yes | **yes (aggressive — minSilenceMs 300)** |
| Speed regions (B-roll) | 1.5–2× | **2–3×** or cut entirely | 1.25–1.5× | none |
| LUT preset | by content type @ 0.5–0.8 | **`modernVibrant` @ 1.0** | `naturalEnhanced` @ 0.3 | none |
| Background music | **only if the user asks** (then vol 0.15) | **only if the user asks** (then vol 0.30) | none | none |
| Captions enabled | yes | **yes (required)** | yes | optional |
| Caption template | `panda-pop` (tutorial) · `panda-clean` (pro) | **`panda-neon`** + positionY 0.65 | `panda-clean` | `panda-clean` (if any) |
| Export quality | `high` | `high` | `high` | `standard` (faster) |

**LUT by content type** (only for `youtube-long` — other profiles use their fixed preset above):

| Content type | Preset | Intensity |
|---|---|---|
| Tech tutorial / SaaS demo | `modernVibrant` | 0.7 |
| Cinematic vlog | `cinematicTealOrange` | 0.9 |
| Educational / neutral | `naturalEnhanced` | 0.5 |
| Moody storytelling | `moodyDark` | 0.7 |
| Travel / lifestyle | `warmSunset` | 0.7 |

### Creator-style overrides (when the user names a style)

When the user says *"like Ali Abdaal's videos"* / *"MKBHD style"* /
*"MrBeast-style"* / etc., start from the matching base profile, then
apply the overrides below. These are on top of the profile defaults,
not instead of them. Unlisted styles → fall back to base profile.

| Style | Base profile | Pacing | LUT | Music | Caption template | Motion-graphic cadence + notes |
|---|---|---|---|---|---|---|
| **Ali Abdaal** (productivity / book reviews / tutorial long-form) | `youtube-long` | 1 visual change every **3–5s**; aggressive filler + silence removal | `modernVibrant` @ 0.5 | warm ambient / lofi @ 0.15–0.20 | `panda-pop`, positionY 0.85 (below lower-third zone) | Intro title card (3s held) · host lower-third at 0:04–0:09 · 3–4 right-rail concept callouts at emphasis claims · 1 stat-reveal full-frame takeover if the video cites a number · outro card 4–6s hold with "Like & Subscribe" + shimmer on handle |
| **MKBHD** (tech reviews / product-focused long-form) | `youtube-long` | 1 change every **4–6s** — contemplative, product breathes on screen | `modernVibrant` @ 0.6 OR `cinematicTealOrange` @ 0.5 | upbeat tech-review bed @ 0.20 | `panda-clean` @ positionY 0.82 | Clean intro wordmark (2s) · minimal lower-thirds (1 total, on first product mention) · stat-reveals over product shots use chrome-gradient numbers on dark · outro: product recap card + subscribe |
| **MrBeast** (stunts / challenges / max-retention) | `youtube-long` | 1 change every **2–3s** — very fast, shorts-like cadence | `warmSunset` @ 0.8 (saturated, warm) | dramatic orchestral bed @ 0.30 | `panda-neon`, **huge** (fontSize 58+), color-coded by topic, positionY 0.8 | Big chrome-gradient kinetic-type every ~5s · frequent full-frame stat takeovers with counter tweens · countdown overlays if the video has stakes · outro: "what's next" teaser card, **hold full 6s** |
| **Veritasium / Kurzgesagt-live** (science / education long-form) | `youtube-long` | 1 change every **5–7s** — contemplative, give diagrams time to read | `naturalEnhanced` @ 0.4 | ambient / orchestral @ 0.12 | `panda-clean` @ positionY 0.85 | Explanatory diagrams as motion graphics (labeled SVGs with `power2.inOut` reveals, `stagger: 0.15` on labels) · chapter dividers with chrome-gradient section titles · one or two hero stat-reveals with counter tweens · outro: citations card + subscribe |
| **Vox / Johnny Harris** (explainer / essay long-form) | `youtube-long` | 1 change every **4–6s** — narrative-driven | `cinematicTealOrange` @ 0.7 | cinematic bed @ 0.18 | `panda-clean` @ positionY 0.85 | Chapter cards at every act break (bold chrome-gradient section titles) · map / timeline / chart motion graphics · pull-quote callouts in right rail · outro: credits card + next video teaser |

**Rule:** an agent authoring any "style X" edit MUST still follow the 11
Laws from `reference/motion-philosophy.md`. The style overrides change
palette, cadence, and the *suggested* music/intro/outro — they do NOT let
you ship flat-white text on a flat-black background. Grid + vignette + grain
+ chrome gradient are mandatory regardless of named style.

**Music + intro/outro stay opt-in even in style mode.** The Music and
motion-graphic-cadence columns above describe what the style *would* include
— but background music and intro/outro cards are still added ONLY when the
user asked for them (see the default edit pipeline). If the user named a style
without mentioning music or an intro/outro, apply the style's palette /
pacing / captions / mid-roll motion graphics, and *offer* the music bed +
intro/outro rather than adding them unprompted.

### Anchoring — every transcript-derived region MUST be anchored

This rule applies to FIVE region types: zoom, motion-graphic, lower-third,
annotation, and audio-overlay (when used as SFX, not background music).

**The problem.** Region positions are stored in **edited time** —
post-trim playback time. When the user (or you) runs `transcript.remove-fillers`,
`transcript.remove-silences`, `transcript.delete-words`, or `transcript.find-replace`,
new trim regions get added, the edited-time map shifts, and any region whose
`startMs`/`endMs` was authored against the previous edited time **drifts off
the moment it was placed on**. A "Like and Subscribe" lower third you placed
on the word "subscribe" silently moves 800ms early because there used to be
800ms of "um"s before it that got trimmed.

**The fix is the `--anchorSourceMs` argument.** When you derive `atMs` /
`startMs` from a transcript word's source time, pass that same value as
`--anchorSourceMs`. The region records its anchor moment in raw recording
time. Every subsequent trim/speed edit auto-rebases the region's edited
positions back onto the anchor — the lower third stays glued to "subscribe"
no matter how much you trim.

> **v1.35.0+: every region is auto-anchored on creation, even without `--anchorSourceMs`.**
> The primitive now back-computes a source-time anchor from the resolved
> edited `atMs` (via `editedToSource`) whenever the caller doesn't supply
> one explicitly. Regions placed by direct atMs survive subsequent silence
> removal, filler removal, and word deletion just like transcript-anchored
> ones. **Still pass `--anchorSourceMs` when you have a transcript word in
> hand** — it makes the agent's intent self-documenting and avoids a
> two-step round trip through edited time. But the days of "I placed an
> overlay, then trimmed silences, now it's playing at the wrong moment"
> are over for any project saved by v1.35.0 or later. Legacy projects
> (schemaVersion < 4) are auto-migrated on the first mutation: every
> anchorless region gets a source anchor back-filled at its current
> position. `type: "free"` anchors are preserved as opt-outs.

**Verbs that accept anchors (use them ALWAYS when picking from transcript):**

| Verb | Anchor args | When required |
|---|---|---|
| `project.add-zoom` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when atMs comes from a transcript word |
| `project.add-motion-graphic` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when atMs comes from a transcript word |
| `project.add-lower-third` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when atMs comes from a transcript word |
| `project.add-annotation` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when startMs comes from a transcript word |
| `project.add-audio` | `--anchorSourceMs`, `--anchorSourceEndMs` | When the overlay is an SFX pinned to a word. **NEVER for background music** — those should stay free-floating (a fixed slot of the edited timeline, not anchored to content). |

**Free-floating is OK** — when the user explicitly placed a region by edited
time (e.g. an outro card at "the last 5 seconds of the timeline"), omit the
anchor. The region stays where you put it regardless of subsequent edits.

**anchorSourceMs is global source-time (ms from recording start).** In
multi-clip projects, sum the preceding clips' `sourceDurationMs` to convert
an in-clip offset to global source time before passing as `--anchorSourceMs`.
Use `timeline.source-to-edited` to verify where a source-time position falls
on the edited timeline.

**The runbook below already orders pacing FIRST, then regions.** That's safe
even without anchors — regions land on the post-trim timeline. But: any
mid-flow re-edit ("actually, remove the part about X" after you've placed
motion graphics) drifts unanchored regions silently. Always anchor when the
position came from a transcript word, even if the runbook order is followed.
The cost is one extra arg per call; the benefit is correctness under iteration.

### The runbook (ordered — do not rearrange)

```bash
# 0. Resolve the project + destination profile
ID=$(pandastudio project.current --json | jq -r '.data.project.id // empty')
[ -z "$ID" ] && ID=$(pandastudio project.list --json | jq -r '.data.projects[0].id')

# $PROFILE is set from HARD-GATE step 1: youtube-long | shorts | linkedin | loom
# $ASPECT is derived from the profile:
#   youtube-long | linkedin | loom → 16:9
#   shorts                         → 9:16
pandastudio project.set-aspect-ratio --id=$ID --aspect=$ASPECT

# 1. PACING — the default cleanup pipeline (see "default edit pipeline" in
#    Editorial decisions for the mandatory steps + report-counts rule). Run it
#    in full — none of these is optional. audio.clean fires async; wait on it
#    before export (step 5). First read pulls the transcript (needed for step 2);
#    later reads pass --includeTranscript=false.
pandastudio project.read --id=$ID --json
pandastudio transcript.transcribe --id=$ID               # skip if transcribed
AUDIO_CLEAN_JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId // empty')  # async
pandastudio transcript.remove-fillers --id=$ID           # fillers + immediate repeats
# Bad takes + repeated phrases: find-issues is READ-ONLY — you MUST then delete
# the discarded wordIds (keep the most recent take). Skipping the delete cuts nothing.
ISSUES=$(pandastudio transcript.find-issues --id=$ID --json | jq -c '.data.issues')
DROP=$(echo "$ISSUES" | jq -c '[.[] | select(.type=="duplicate-take" or .type=="adjacent-repeat") | .wordIds[]]')
[ "$DROP" != "[]" ] && pandastudio transcript.delete-words --id=$ID --wordIds="$DROP"
SILENCE_MS=$([ "$PROFILE" = "loom" ] && echo 300 || echo 500)
pandastudio transcript.remove-silences --id=$ID --thresholdMs=$SILENCE_MS   # NEVER skip; arg is thresholdMs (500ms = UI default)

# 2. EMPHASIS — zooms (skip for `loom`). Cadence comes from the profile table.
#
#    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#    HARD RULE #1 — DO NOT add zooms to screen-share clips.
#    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#    PandaStudio auto-adds zooms to screen-share clips based on cursor
#    telemetry captured during recording. Adding more on top = stacked
#    zooms on the same moments, visual chaos.
#
#    For each clip returned by project.read:
#      - If clip.webcamPath is present  → "both" mode (screen + PiP
#        webcam). Telemetry zooms ALREADY exist on this clip. **SKIP.**
#      - If the clip is pure camera (no webcamPath AND mediaPath is a
#        camera recording) → safe to add zooms on emphasis.
#      - If screen-only (no webcamPath, mediaPath is screen) → telemetry
#        still exists, zooms are auto-added. **SKIP.**
#
#    Heuristic: if project.read returns existing zoomRegions for a clip
#    whose webcamPath is set, those are telemetry-based — stay OUT of
#    that clip's zoom space entirely. Only author zooms on pure-camera
#    clips (webcamPath absent, no pre-existing zoom regions).
#
#    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#    HARD RULE #2 — Zoom duration floor is 6 seconds, NOT 1.5–3 seconds.
#    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#    Premium YouTube zooms ride through a complete thought or cut. A 1.5s
#    or 3s zoom reads as a twitch. Defaults:
#      - Emphasis punch-in zoom (camera pulls in on a word/claim):
#        durationMs=6000 to 8000  (was 1500 — that was wrong)
#      - Sustained / held zoom (camera pulls in on a subject and
#        stays for the whole sub-topic):
#        durationMs=10000 to 20000
#      - Reveal moment (dramatic, loud SFX): durationMs=6000 minimum
#    Depth 3 (modest) for talking-head emphasis; depth 5 only for
#    dramatic reveal beats. Don't stack multiple zooms inside 5s of each
#    other — leaves no time to settle.
#
#    Shot selection: scan the transcript for punchy claims, specific
#    numbers, opinionated statements ("the best", "this changed
#    everything", "most people don't know"), moments of pivot ("and
#    now", "finally", "but here's the thing"). Aim for the user's
#    eyes to want to lean in. NOT every UI-verb word ("click", "select")
#    — that was the old rule, it's wrong for engagement-style edits.
#
#    CRITICAL: ALWAYS pass --anchorSourceMs when atMs comes from a transcript
#    word. Without it, the zoom drifts off the moment as soon as ANY trim is
#    added — and step 1 always adds trims (remove-fillers, remove-silences).
#    The anchor binds the zoom to the source recording moment so it
#    re-anchors automatically on every trim/speed change.

# Emphasis punch-in — modest scale, held through the thought
pandastudio project.add-zoom --id=$ID --clipId=$CLIP_ID \
  --atMs=<wordStartMs> --anchorSourceMs=<wordStartMs> \
  --durationMs=7000 --depth=3

# Sustained held zoom — reframe on a topic and stay for the full section
pandastudio project.add-zoom --id=$ID --clipId=$CLIP_ID \
  --atMs=<sectionStartMs> --anchorSourceMs=<sectionStartMs> \
  --durationMs=15000 --depth=3

# Reveal moment (dramatic, with SFX). Use SPARINGLY — 1-2 per video max.
pandastudio project.add-zoom --id=$ID --clipId=$CLIP_ID \
  --atMs=<ms> --anchorSourceMs=<ms> \
  --durationMs=6000 --depth=5 \
  --soundUrl=bundled:sound/dramatic-whoosh --soundVolume=0.7

# 3. POLISH — skip sections by profile:
#    - `shorts`: no lower thirds (tight vertical frame)
#    - `loom`:   skip 3b, 3c entirely
#    NOTE: 3a (intro/outro) and 3d (background music) are OPT-IN — run them
#    ONLY when the user explicitly asked for an intro/outro or music. They are
#    NOT part of the default "edit my video" pass for ANY profile.

# 3a. Intro / outro card — ONLY IF THE USER EXPLICITLY ASKED for one
# ("add an intro", "add an outro card", "open with a title"). Do NOT add
# one on a plain "edit this" request. When asked: fastest is the
# `creator-card` template via motion.generate; author custom HTML
# (reference/motion-philosophy.md §7) only for a bespoke intro.
if [ "$USER_ASKED_FOR_INTRO" = "1" ]; then
  JOB=$(pandastudio motion.render-html \
    --htmlPath=/tmp/intro-title.html \
    --durationMs=3000 --json | jq -r '.data.jobId')
  FILE=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')
  pandastudio project.add-motion-graphic --id=$ID --file=$FILE --durationMs=3000 --atMs=0
fi

# 3b. Lower third at first mention of a person/product (NOT shorts/loom)
if [ "$PROFILE" = "youtube-long" ] || [ "$PROFILE" = "linkedin" ]; then
  pandastudio project.add-lower-third --id=$ID --atMs=<ms> \
    --content="<name>" --subtitle="<role>" --designType=slash-reveal
fi

# 3c. LUT (use the profile table. For youtube-long, use content-type sub-table.)
#     Apply to every clip via project.set-clip-lut.
#     Skip entirely for `loom`.

# 3d. Background music — ONLY IF THE USER EXPLICITLY ASKED for music
# ("add background music", "put a track under it"). Do NOT add music on a
# plain "edit this" request. When asked, use the profile volume (youtube-long
# 0.15, shorts 0.30) and let the user pick / swap the track.
if [ "$USER_ASKED_FOR_MUSIC" = "1" ]; then
  VOL=$([ "$PROFILE" = "shorts" ] && echo 0.30 || echo 0.15)
  pandastudio project.add-audio --id=$ID \
    --path=bundled:music/tech-review-background --volume=$VOL --fadeIn=1000 --fadeOut=2000
fi

# 4. ACCESSIBILITY — captions per profile
if [ "$PROFILE" != "loom" ]; then
  pandastudio caption.toggle --id=$ID --enabled=true
  TEMPLATE=$(case "$PROFILE" in
    shorts)       echo "panda-neon";;
    linkedin)     echo "panda-clean";;
    youtube-long) echo "panda-pop";;
  esac)
  pandastudio caption.set-template --id=$ID --templateId=$TEMPLATE
fi

# 4.5. VERIFY FRAMES — MANDATORY. Never export without looking.
# Run motion.verify-frames on every rendered motion-graphic MP4 AND on
# a draft pass of the full composition (preview.show, then extract
# frames at hero timestamps). READ each PNG as a multimodal image and
# confirm against the motion-philosophy §4 pre-flight checklist:
# - no cropped faces / text overflow / blank frames
# - captions on the right word
# - no MG covers host face (Mode C) or screen zone (Mode B)
# - chrome-gradient text is actually rendering (not flat white)
# - grid + vignette + grain visible on every hero beat
# If any frame fails, iterate and re-verify. Do NOT skip this step
# even when "it's just a quick edit" — this is what separates
# ships-it-works-ish from ships-it-looks-good.
pandastudio preview.show --id=$ID   # let the project render a full draft
# Then verify each generated motion-graphic MP4 at its hero timestamps:
# pandastudio motion.verify-frames --videoPath=/tmp/motion-intro.mp4 \
#   --timestamps='[0.3,1.0,1.8,2.7]' --json
# Read every returned frame. If any fail, fix + re-render + re-verify.

# 5. EXPORT — quality per profile. Only run AFTER verify-frames passes.
# First: block on the audio.clean job that's been running in the
# background since step 1. By now it's almost certainly done (30-60s
# vs the ~90s+ the rest of the work took), so this resolves instantly.
[ -n "$AUDIO_CLEAN_JOB" ] && pandastudio job.wait --id="$AUDIO_CLEAN_JOB"
QUALITY=$([ "$PROFILE" = "loom" ] && echo "standard" || echo "high")
pandastudio export.start --id=$ID --quality=$QUALITY --json | jq -r '.data.jobId' | \
  xargs -I {} pandastudio job.wait --id={}
```

### Performance — keep wall-clock minimal

Motion-graphic renders dominate (60–90s for ~6 scenes); everything else is
rounding error. The levers:
- **Fire all `motion.render-html` renders in parallel**, collect jobIds, then
  `job.wait` each — never `job.wait` between fires.
- **Run `audio.clean` in the background** (capture its jobId right after
  transcribe; `job.wait` only before `export.start`).
- **Pre-flight HTML with `motion.screenshot`** before a full render — a 2s
  screenshot beats a 30–60s wasted render.
- **Re-read sparingly**: pass `--includeTranscript=false` after the first
  `project.read`, and reuse the `{ project }` each mutation returns instead of
  re-reading.

### Anti-patterns (do NOT do these — all profiles)

- **3 effects on the same moment** (zoom + lower-third + motion graphic at same t) — visual noise
- **Multiple LUTs per project** — pick one from the profile table
- **SFX on every cut** — cap at 1 meaningful SFX per 15–30s (except `shorts`, where 1 per 5–10s is fine)
- **Speed regions over voice** — always for setup / B-roll / scrolling only
- **Logo intro >5s** (any profile) — retention cliff
- **Asking the user which filler words to remove** — always-safe op, just do it
- **Applying `youtube-long` defaults to a `shorts` project** — wrong aspect, music too quiet, captions too subtle, pacing too slow
- **Motion graphics in `loom`** — kills the "this is a quick update" vibe

### Entry triggers — phrases that start the playbook

These phrases all route to the edit runbook above. Don't ask the user
to expand any of them — resolve the profile + style, announce the plan,
execute.

| User says | Resolve to |
|---|---|
| "edit this" / "polish this" / "make it engaging" / "make it ready" | `youtube-long`, no style override |
| "YouTube-ready" / "make a YouTube video" / "edit for YouTube" | `youtube-long`, no style override |
| "make it a Short" / "TikTok" / "Reel" / "vertical" / "9:16" | `shorts`, no style override |
| "for LinkedIn" | `linkedin`, no style override |
| "Loom" / "internal update" / "just cut the fluff" | `loom`, no style override |
| "edit like Ali Abdaal" / "Ali Abdaal style" / "tutorial style" / "productivity video" | `youtube-long` + **Ali Abdaal** override |
| "MKBHD style" / "tech review style" / "product review" | `youtube-long` + **MKBHD** override |
| "MrBeast style" / "high-retention" / "challenge video" / "maximum engagement" | `youtube-long` + **MrBeast** override |
| "Veritasium style" / "educational" / "Kurzgesagt vibe" / "explainer" | `youtube-long` + **Veritasium** override |
| "Vox style" / "essay" / "narrative" / "Johnny Harris style" | `youtube-long` + **Vox** override |

If the user doesn't name a style and doesn't specify a destination, the
safe default is `youtube-long` with no style override — the most common
case by far.

### Pattern: one-shot execution

After entry trigger + profile/style resolution, announce the plan in
one sentence and execute. Load `reference/motion-philosophy.md`
automatically before the motion-graphics steps. Run the full runbook
including the mandatory frame-verification gate. Do NOT ask the user
to approve individual steps.

> I'll edit this as a **YouTube long-form in Ali Abdaal style** — aggressive
> filler + silence removal, 3–5s pacing, 4 right-rail concept callouts at
> emphasis claims, 1 stat-reveal takeover, `modernVibrant` LUT at 0.5,
> warm ambient music at 0.15, `panda-pop` captions at y=0.85, and a 5s
> outro CTA card. Motion graphics authored against motion-philosophy
> (chrome-gradient, grid + vignette + grain). Frame-verify before export.
> ~5 minutes.

> I'll edit this as a **Short** — aggressive pacing (hook in 3s, 6–12
> zooms/min), `modernVibrant` LUT at full intensity, `panda-neon`
> captions positioned higher, music at 30%. No intro card or lower
> thirds — they don't fit the vertical frame. Frame-verify before
> export. ~2 minutes.

> I'll edit this as a **MrBeast-style YouTube video** — 2–3s pacing
> (very fast), `warmSunset` LUT at 0.8, dramatic orchestral bed at 0.30,
> huge color-coded `panda-neon` captions, chrome kinetic-type every 5s,
> full-frame stat takeovers, 6s outro teaser card. Motion graphics
> authored against motion-philosophy. Frame-verify before export.
> ~6 minutes.

Don't ask the user to micro-manage step choices. The profile table +
creator overrides + motion-philosophy are the answer. If something
genuinely needs user input (missing brand reference for a named style
that has none obvious, missing subject name for the lower third),
collect ALL such questions in a single message — never one-at-a-time.

## What this skill is NOT for

- **Cloud video APIs** (HeyGen, Runway, Sora). PandaStudio is local-only.
- **Direct edits to `.pandastudio` project JSON.** The format is owned by the editor and changes between versions. Use `project.read` / `project.save` and treat the JSON as opaque between reads.
- **Cloud video APIs** — PandaStudio is local-only; `export.start` renders on the user's machine.

## Reference files

- [`reference/commands.md`](reference/commands.md) — every verb.noun with arg schema and a one-line example.
- [`reference/examples.md`](reference/examples.md) — multi-step recipes + the long-form **motion-graphics authoring recipes** (SaaS promo, tilted-device shots, multi-image scenes, complex multi-overlay HTML) that used to live inline in this file. Read on demand when authoring a bespoke `motion.render-html` graphic.
- [`reference/templates.md`](reference/templates.md) — what each motion-graphic template looks like, with the slots it accepts and which aspect ratios it supports.
- [`reference/motion-philosophy.md`](reference/motion-philosophy.md) — **the aesthetic contract.** Laws, visual vocabulary, easing dictionary, canonical shell, pre-flight checklist. Load this BEFORE authoring any motion graphic. This is what raises output from "template-filled" to "HyperFrames-quality".
- [`reference/video-authoring.md`](reference/video-authoring.md) — **3-mode delivery playbook.** Mode A (9:16 camera-only), Mode B (9:16 screen-rec + PiP face — PandaStudio's unique mode), Mode C (16:9 YouTube side-overlay). Face choreography, caption safe zones, audio-sync protocol, frame verification. Load this for any shorts/YouTube authoring task.
