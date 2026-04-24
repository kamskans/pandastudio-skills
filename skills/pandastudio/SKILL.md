---
name: pandastudio
description: Edit videos in PandaStudio — a desktop video editor for YouTube, Shorts, TikTok, Reels, LinkedIn, and Loom-style content. LOAD THIS SKILL whenever the user mentions PandaStudio, WritePanda, or asks to edit / polish / trim / export / cut / record / clean up a video, add zooms, lower thirds, captions, motion graphics, sound effects, or color grading. Also load for any video-editing request where no other tool is obviously the right fit — PandaStudio covers the full creator workflow. Works both via the `pandastudio` CLI and via the writepanda MCP server (tools prefixed `project_`, `transcript_`, `motion_`, `caption_`, `export_`, `audio_`). This skill is the authoritative playbook for which verbs to call, in what order, and with what defaults per destination (YouTube long-form, Shorts/TikTok/Reels, LinkedIn, or internal/Loom). Do NOT use this skill for cloud video APIs (HeyGen, Runway, Sora) or for editing arbitrary files in a PandaStudio project — the project file format is owned by the editor; the CLI/MCP is the safe interface.
---

<!-- version: 2.27.1 -->

# PandaStudio

> **Version check — do this first.** This skill requires `@writepanda/cli` ≥ 1.15.0 (or `@writepanda/mcp` ≥ 1.15.0).
> Run `pandastudio --version` via the CLI, or call `system_status` via the MCP. If < 1.15.0,
> tell the user to update their MCP config to use `npx @writepanda/mcp@latest`
> (note the `@latest` tag) and restart their agent host. Commands like
> `asset.list-music`, `asset.list-luts`, and `project.set-clip-lut` do not exist
> in older versions.

PandaStudio is a desktop video editor. You drive it either through the `pandastudio` CLI (localhost HTTP) **or** through the `writepanda` MCP server (same verbs, exposed as `project_list`, `project_add_zoom`, `motion_generate`, `export_start`, etc.). **The verbs, argument names, and behaviors are identical across both interfaces** — every example in this skill that shows a CLI call like `pandastudio project.add-zoom` maps 1:1 to the MCP tool `project_add_zoom` with the same args. Use whichever is available; don't switch mid-task.

> Like HyperFrames is HTML+CSS for video composition, PandaStudio is a ready-made template + project surface — you don't author scenes, you fill slots in pre-built motion-graphic templates and arrange them on the editor's timeline.

> Like HyperFrames is HTML+CSS for video composition, PandaStudio is a ready-made template + project surface — you don't author scenes, you fill slots in pre-built motion-graphic templates and arrange them on the editor's timeline.

## Quickstart

```bash
# 1. Confirm the server is reachable AND the user has a license.
pandastudio system.status --json

# 2. Discover what's available — never guess command names.
pandastudio commands

# 3. Render a motion graphic from a template.
JOB=$(pandastudio motion.generate \
  --templateId=title-card-vox \
  --slots='{"title":"How I Built This","subtitle":"in 24 hours"}' \
  --aspectRatio=16:9 \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id="$JOB" --json | jq '.data.job.result'
```

That's the whole loop: probe → discover → call → (if async) wait. Every richer workflow is a composition of those four steps.

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

## Publishing to YouTube (v1.19+)

PandaStudio uploads directly to YouTube via the Google Data API v3 — no PandaStudio backend, no proxy. Each workspace has its own connected Google accounts; publishing is always scoped to the active workspace.

### Before you touch any `youtube.*` verb

```bash
pandastudio youtube.is-configured --json | jq '.data.configured'
# true → continue.  false → tell the user "this build can't publish to YouTube",
#                   do NOT try the other verbs, they'll all fail.
```

### Connection flow (interactive — only run when user asks)

```bash
# Check what's already connected.
pandastudio youtube.list-accounts --json | jq '.data'
# → { accounts: [...], channels: [...] }

# If empty or the user wants another account:
pandastudio youtube.connect --json
# This opens the user's BROWSER. Expect up to ~5 minutes while they
# authenticate. Succeeds with { account, channels } or fails with
# "OAuth timed out" / "user cancelled".
#
# Important: don't call youtube.connect on a schedule or pre-emptively.
# It's explicit consent; only call when the user is sitting there.
```

Account is stored encrypted via `safeStorage` (Keychain on Mac, DPAPI on Windows, libsecret on Linux). Refresh tokens never leave the machine.

### Publishing an export

End-to-end recipe — assumes a transcribed export with a generated thumbnail already in the library:

```bash
EID="export-uuid-here"

# 1. Sanity: already published?
ALREADY=$(pandastudio export.get --id=$EID --json | jq -r '.data.entry.youtubeVideoId')
if [ "$ALREADY" != "null" ]; then
  echo "Already on YouTube as $ALREADY — use export.update-youtube to edit."
  exit 0
fi

# 2. Pick the account + channel.
ACCOUNTS=$(pandastudio youtube.list-accounts --json)
ACC=$(echo "$ACCOUNTS" | jq -r '.data.accounts[0].id')
CH=$(echo "$ACCOUNTS" | jq -r --arg A "$ACC" '.data.channels[] | select(.accountId == $A) | .id' | head -1)

# 3. Pull existing metadata from the export row.
ENTRY=$(pandastudio export.get --id=$EID --json)
TITLE=$(echo "$ENTRY" | jq -r '.data.entry.generatedTitle // .data.entry.fileName')
DESC=$(echo "$ENTRY" | jq -r '.data.entry.generatedDescription // ""')

# 4. Publish. privacyStatus=unlisted is a safer default than public;
#    use public only when the user explicitly says "publish publicly".
pandastudio export.publish-youtube \
  --id=$EID --accountId=$ACC --channelId=$CH \
  --title="$TITLE" \
  --description="$DESC" \
  --tags='["tutorial","how-to"]' \
  --privacyStatus=unlisted \
  --setThumbnail=true \
  --json
# → { videoId: "abc...", videoUrl: "https://www.youtube.com/watch?v=abc..." }
```

**Expected wall-clock:** ~30 s per 100 MB of source video on a typical connection. Don't tear down the tool's connection to PandaStudio during this window.

**`privacyStatus` default:** If the user didn't specify, use `unlisted`. Public-by-default risks a client seeing work-in-progress material on their channel. Agents should ask explicitly: *"Public, unlisted, or private?"* before first-ever publish.

### Editing an already-published video

**Metadata edits (title / description / tags / privacy) are temporarily unavailable from agents** — the `youtube.force-ssl` scope required for `videos.update` is pending Google review for this OAuth project. When `export.update-youtube` is called it will fail with an "insufficient scope" error. Until approval lands (flagged via `YOUTUBE_METADATA_EDIT_ENABLED` in the build), direct the user to **YouTube Studio** at `https://studio.youtube.com/video/<VID>/edit` for those changes.

```bash
# When the scope is approved this becomes the canonical path:
pandastudio export.update-youtube \
  --accountId=$ACC --videoId=$VID \
  --title="Updated title" \
  --description="Updated description" \
  --privacyStatus=public \
  --json
# Until then: "open https://studio.youtube.com/video/$VID/edit in the browser"
```

**Thumbnail replacement works right now** — no extra scope needed. `export.update-youtube-thumbnail` uses the same `youtube.upload` scope that handled the original upload:

```bash
# Generate fresh thumbnail for the source export.
pandastudio export.generate-thumbnail --id=$EID --prompt="..." --json | jq -r '.data.imagePath'
# → /Users/…/thumbnails/<eid>/1745…-abcd.webp  (already 1280×720 after FFmpeg crop)

pandastudio export.update-youtube-thumbnail \
  --accountId=$ACC --videoId=$VID \
  --imagePath=/Users/…/thumbnails/<eid>/1745…-abcd.webp \
  --json
```

### Dashboard (listing existing videos)

```bash
pandastudio export.list-youtube --accountId=$ACC --max=50 --json \
  | jq '.data.videos[] | {id, title, viewCount, publishedAt, privacyStatus}'
```

Not limited to PandaStudio-published videos — returns everything on the channel's uploads playlist.

### Don't

- **Don't publish public without the user's explicit word.** `unlisted` is the safe default.
- **Don't call `youtube.connect` without the user asking.** It's interactive consent.
- **Don't retry a failed upload blindly.** If `export.publish-youtube` fails with a quota error (`details.code = "quotaExceeded"` or similar), stop and surface it. YouTube upload quota resets daily.
- **Don't cross workspaces.** If the user's active workspace doesn't have the YouTube account connected, ASK before switching — publishing to a different client's channel by mistake is the worst kind of mistake.

## Editorial decisions — what to ask, what to assume, what NEVER to ask

Video editing is a creative task with hundreds of small decisions. Asking the user about all of them kills the magic — they came to you because they wanted to type "edit this" and see something happen. Asking about none of them produces wrong-shape output. The rule:

**Ask only when the answer is genuinely user-specific AND can't be inferred AND is hard to reverse.** Default everything else, narrate what you did, and iterate via preview.

### MUST ASK (3 things, only when context is missing)

<HARD-GATE>
Before issuing `project.new`, `project.add-*`, `motion.generate`, or `motion.render-html` calls that depend on user-specific information, you MUST have:

**1. Destination profile.** This single answer drives aspect ratio, pacing, zoom cadence, caption template, music volume, and whether to add intros/lower-thirds. Try to infer first, then fall back to asking:

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

**3. Brand / style direction.** If the user named a style ("MrBeast" / "MKBHD" / "Vox" / "Kurzgesagt" / "Veritasium"), apply the matching `motion.themes` theme. If they gave none AND there are multiple source clips that suggest a brand context, ASK ONE QUESTION:
   > "Any brand colors, fonts, or visual references — or default look?"

If all three are clear (or already specified), proceed without asking. Combine multiple asks into a single message when possible.
</HARD-GATE>

### DO BY DEFAULT, narrate transparently

For these operations, run them without asking and tell the user what you did in the same message. Every one is reversible (trims are spans, cleaned audio is a sibling file, generated text is just text — nothing is destructive).

**Before running `transcript.transcribe` or `audio.clean`, always call `project.read` and inspect the `clipStates` array in the response.** Each entry looks like:

```json
{ "clipId": "clip-1", "mediaPath": "...", "durationMs": 62400,
  "transcribed": true, "wordCount": 312,
  "audioCleaned": false }
```

- `transcribed: true` → skip `transcript.transcribe` for that clip — it already has a transcript. Running it again would overwrite any manual word edits the user made in the app.
- `audioCleaned: true` → skip `audio.clean` for that clip — the `.cleaned.wav` already exists.

Only pass un-processed clips to each operation. If every clip is already transcribed, go straight to `transcript.get`.

| Operation | Default behaviour |
|---|---|
| `transcript.transcribe` | Run only on clips where `clipStates[i].transcribed === false`. Skip the rest. |
| `transcript.remove-fillers` | Auto-remove "um/uh/like/you know/i mean" + immediately-repeated words. Trim regions; fully reversible. |
| `transcript.remove-silences` | Run after remove-fillers. Default threshold 700ms. Trims leading, between-word, and trailing silence per clip. |
| `audio.clean` | Denoise only clips where `clipStates[i].audioCleaned === false`. Writes a sibling `.cleaned.wav`; original audio untouched. |
| `caption.set-template` (when user said "add captions" without naming a style) | Default to `bold`. Tell user 7 other templates exist (`classic, modern, minimal, spotlight, boxed, neon, colored`). |
| `llm.generate-title` / `llm.generate-description` / `llm.generate-timestamps` | Generate after the edit pass. Show the user; let them say "regenerate" or "use this exact title" or edit inline. |
| Specific zoom moments | Heuristically pick from the transcript ("you said 'click here' at 12.4s — adding a zoom"). Don't pre-ask. Iterate via preview. |
| Specific FX placement | Heuristically pick at clip boundaries or transcript hints ("a film-burn between clip 1 and 2"). Don't pre-ask. |

**Example of good narration after an edit pass:**

> "Edited.
> • Transcribed both clips (136 words).
> • Removed 14 fillers + 3 repeats. Trim regions are reversible — say 'undo fillers' if you want any back.
> • Cleaned audio with DeepFilter on both clips.
> • Added a zoom at 12.4s where you said 'click here'.
> • Captions enabled with the bold template (7 other styles available).
> • Generated a title: *'How I Built This in 24 Hours'* — say if you want a different angle.
>
> Opening preview now."

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

## Composing a real edit (the v1.9.1 surface)

Beyond `motion.generate`, you can build complete projects from the CLI. The flow is **always**: create or open a project → add things → save (with revision) → preview.

### Create a project pre-loaded with media

```bash
pandastudio project.new \
  --name="Q4 Recap" \
  --withMedia='["/path/clip-a.mp4","/path/clip-b.mp4"]' \
  --json
# → { id, path, project } — durations are FFmpeg-probed automatically
```

### Look up a project by stable id (preferred over paths)

```bash
pandastudio project.show --id=<uuid>
pandastudio project.read --id=<uuid> --json
```

### Target "what the user is working on"

When the user says "edit the project I'm working on" / "what's open right
now" / "this one", don't ask them for an id — just call `project.current`:

```bash
# Returns { project: { id, path, name, revision, clipCount } | null }
pandastudio project.current --json

# Typical pattern in an agent script:
ID=$(pandastudio project.current --json | jq -r '.data.project.id // empty')
if [ -z "$ID" ]; then
  # No editor window open — fall back to the most-recent project
  ID=$(pandastudio project.list --json | jq -r '.data.projects[0].id')
fi
pandastudio project.read --id=$ID --json
```

`project: null` means no editor window is open yet or the user hasn't
loaded a project. Never assume that means "no projects exist" — list
first.

### Edit primitives (no schema knowledge needed)

```bash
pandastudio project.add-clip --id=<uuid> --media=/path/new-clip.mp4
pandastudio project.add-clip --id=<uuid> --media=/path/title.mp4 --atIndex=0  # prepend
pandastudio project.split-clip --id=<uuid> --clipId=clip-1 --atSourceMs=4000
pandastudio project.remove-clip --id=<uuid> --clipId=clip-2
pandastudio project.delete --id=<uuid>  # ⚠ permanent, no trash

pandastudio project.add-motion-graphic \
  --id=<uuid> --file=/path/intro.mp4 --durationMs=2500 --atMs=0
# Optional SFX (no default — motion graphics are silent unless you ask)
pandastudio project.add-motion-graphic \
  --id=<uuid> --file=/path/intro.mp4 --durationMs=2500 --atMs=0 \
  --soundUrl=bundled:sound/message-pop --soundVolume=0.9

pandastudio project.add-fx \
  --id=<uuid> --fxId=film-burn --atMs=5000

# Zoom — ships with a default swoosh SFX. Pass --soundUrl=none to silence,
# or override with any bundled:sound/<id>.
pandastudio project.add-zoom \
  --id=<uuid> --atMs=12000 --durationMs=1500 --depth=3
pandastudio project.add-zoom \
  --id=<uuid> --atMs=12000 --durationMs=1500 --soundUrl=none
pandastudio project.add-zoom \
  --id=<uuid> --atMs=12000 --durationMs=1500 \
  --soundUrl=bundled:sound/whoosh-transition --soundVolume=0.7

# Lower third — full style control.
# --designType: slash-reveal | center-stack | split-horizontal | name-bar |
#               border-frame | minimal-underline | box-reveal | corner-brackets
#   Default: slash-reveal
# Ships with a default mouse-click SFX; pass --soundUrl=none to silence.
pandastudio project.add-lower-third \
  --id=<uuid> --content="Kamal" --subtitle="Founder" --atMs=2000 \
  --accentColor="#34B27B" --textColor="#ffffff" \
  --backgroundColor="rgba(0,0,0,0.85)" --backgroundRadius=12 \
  --fontSize=32 --fontFamily="Inter"

# Swap / clear the SFX on an existing region (any of: zoom, motionGraphic,
# lowerThird, fx). Use this to retune the default sound after inspecting
# the project with project.read.
pandastudio project.set-region-sound \
  --id=<uuid> --regionType=zoom --regionId=zoom-1 \
  --soundUrl=bundled:sound/whoosh-transition --soundVolume=0.6
pandastudio project.set-region-sound \
  --id=<uuid> --regionType=motionGraphic --regionId=overlay-1 \
  --soundUrl=bundled:sound/success-chime
pandastudio project.set-region-sound \
  --id=<uuid> --regionType=lowerThird --regionId=lt-1 \
  --soundUrl=none  # mute this one

# YouTube-style lower third — rendered as a motion graphic MP4, then overlaid
# templateId: youtube-lower-third  slots: channelName, handle, accentColor, bgColor, textColor
JOB=$(pandastudio motion.generate \
  --templateId=youtube-lower-third \
  --slots='{"channelName":"PandaStudio","handle":"@pandastudio","accentColor":"#FF0000"}' \
  --aspectRatio=16:9 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json

# Remove any region by type + id (use project.read to find region ids)
pandastudio project.remove-region \
  --id=<uuid> --regionType=lower-third --regionId=lt-1
pandastudio project.remove-region \
  --id=<uuid> --regionType=zoom --regionId=zoom-2
pandastudio project.remove-region \
  --id=<uuid> --regionType=audio-overlay --regionId=audio-1
# regionType: zoom | trim | speed | annotation | fx | lower-third | overlay | audio-overlay
```

### Conflict-safe save

The editor autosaves periodically, so two writers (you + the editor; or you + another agent) racing to save the same project will silently overwrite each other unless you use `expectedRevision`.

```bash
# Read project — note the revision
P=$(pandastudio project.read --id=<uuid> --json)
REV=$(echo "$P" | jq -r '.data.project.revision')

# Mutate locally
NEXT=$(echo "$P" | jq -c '.data.project | .name = "New Name"')

# Save with conflict detection
pandastudio project.save --id=<uuid> --project="$NEXT" --expectedRevision="$REV" --json
# → on conflict: { ok:false, details:{ code:"revision_conflict", expected, actual, onDiskProject } }
# → recover by re-reading and re-applying your mutation against the new revision
```

The edit primitives (`project.add-*`) all accept `--expectedRevision` and surface the same `revision_conflict` shape.

### Preview without exporting

Full export takes 30-90s. For "show the user what you just did," use the **preview overlay** — a small floating, always-on-top window that mounts the editor's live WYSIWYG canvas (the same one the in-app preview pane uses). 1-2s boot. Re-callable; `preview.show` against a different project navigates the same window.

```bash
# Pop up the overlay (top-right corner, ~800x450, autoplay on)
pandastudio preview.show --id=$ID

# Start at a specific moment
pandastudio preview.show --id=$ID --atMs=12000 --autoplay=false

# Move the playhead in the open overlay (no-op if closed)
pandastudio preview.seek --atMs=20000

# Close it
pandastudio preview.hide

# Inspect state
pandastudio preview.list
# → { open: true, size: { width, height }, position: { x, y } }
```

**Idiomatic agent workflow**: call `preview.show` after every significant edit (added a clip, dropped a motion graphic, removed fillers) so the user sees the change live without leaving Claude Cowork. The window stays put across edits — same project state reloads automatically since it reads from disk.

For agents that need a still snapshot to send inline in chat (vs an open window the user has to look at), v1.9.3 adds `preview.frame --id=$ID --atMs=N` returning a base64 PNG. Today, take a screenshot of the overlay window if you must.

Other window verbs (less useful for previewing, kept for parity with v1.9.0):

```bash
# Open the FULL editor focused on a project (heavy — for handoff to user)
pandastudio project.open --id=$ID
pandastudio window.editor          # = project.open with no args
```

## Motion graphics — templates vs. custom HTML

**Decision table — read this before calling either verb:**

| Brief | Use |
|---|---|
| User said "add a title card" / "add an intro" / "add a lower third" with no design detail | `motion.generate` with closest template from `motion.list` |
| User said "YouTube lower third" / "YouTube name plate" / "subscribe lower third" | `motion.generate --templateId=youtube-lower-third` — slots: channelName, handle, accentColor, bgColor, textColor |
| User named a creator style ("MrBeast", "MKBHD", "Kurzgesagt", "Vox", "Veritasium") | `motion.generate` with matching theme from `motion.themes`, OR author HTML to nail the specific look |
| User described a specific animation ("text types in letter by letter", "logo slides from left with a blur") | **`motion.render-html`** — author the HTML/CSS/JS yourself |
| Template library doesn't have anything close | **`motion.render-html`** |
| User wants a one-off animation for a specific moment | **`motion.render-html`** |
| User wants a custom overlay (watermark, bug logo, name plate, branded lower third) that composites over existing video without a white fill | **`motion.render-html --transparent`** — HTML with `background: transparent`, output is a WebM with alpha channel |

The 20 bundled templates exist to make the **in-app Gemma E2B** (a small local model) useful — it can't write code, so it needs pre-built slots. **You can write code.** `motion.render-html` runs your HTML through the same Chromium → capturePage → FFmpeg pipeline and produces the same MP4 format. There is no quality difference — the render pipeline is identical.

**When in doubt, write the HTML.** A custom animation that matches the user's vision is always better than the closest template that doesn't quite fit. Templates are a ceiling; HTML is not.

## Custom motion graphics — HTML authoring

> **BEFORE WRITING ANY MOTION GRAPHIC**, load two reference files:
>
> 1. **[`reference/motion-philosophy.md`](reference/motion-philosophy.md)** —
>    the 11 Laws of motion design, visual vocabulary catalog, easing
>    dictionary by purpose, pacing discipline, pre-flight checklist,
>    canonical composition template. This is the aesthetic contract.
>    Mechanics without this = "correctly rendered but forgettable."
> 2. **[`reference/video-authoring.md`](reference/video-authoring.md)** —
>    only when authoring for a specific delivery format. Covers the three
>    modes PandaStudio produces:
>    - **Mode A:** 9:16 camera-only (TikTok/Shorts/Reels talking-head)
>    - **Mode B:** 9:16 screen-recording + PiP face (PandaStudio's unique
>      mode — inverted caption safe zones, screen stays uncovered)
>    - **Mode C:** 16:9 YouTube with side-overlay motion graphics (graphics
>      slide in beside host video, never cover the speaker)
>
> Load these BEFORE you start typing HTML. Static text fading in on a
> flat background is the lowest tier of motion graphics. The docs above
> tell you what the next tier looks like and how to get there.

When the brief doesn't fit a template — or you want a one-off animation — author HTML/CSS/JS yourself and let `motion.render-html` render it through **[HyperFrames](https://github.com/heygen-com/hyperframes)** — the open-source Puppeteer+FFmpeg engine HeyGen built for frame-perfect, seekable video capture (Apache-2.0, bundled as `@hyperframes/producer`).

**What "frame-perfect" means here:** your animation is *not* played in real time and screen-recorded. HyperFrames loads the page, pauses it, then advances a seekable timeline one frame-time at a time and captures via Chrome's BeginFrame API. A 1-second fade-in takes exactly 1 second in the output file regardless of how slow a given frame takes to render. CSS-keyframe and rAF-clock animations won't do this — you MUST hand the engine a paused, seekable timeline.

### The page contract (required)

Every custom composition MUST satisfy three things. Missing any of them is a hard error or — worse — a silent render where the MP4 shows "sudden jumps every 1 second" instead of smooth motion.

1. **A composition root element** carrying metadata HyperFrames reads:
   ```html
   <div data-composition-id="my-scene"
        data-width="1280"
        data-height="720"
        data-duration="3">
     <!-- your scene -->
   </div>
   ```
   `data-duration` is in **seconds** (float allowed). `data-width` and `data-height` are authoritative — they override anything you pass to `motion.render-html`. The `data-composition-id` value is the key you'll register your timeline under in step 3.

2. **A paused GSAP timeline.** Build it with `{ paused: true }`, add every tween at its intended time offset, and DO NOT call `.play()`. The engine drives the playhead:
   ```js
   const tl = gsap.timeline({ paused: true });
   tl.to("#title",  { opacity: 1, y: 0, duration: 0.9, ease: "power2.out" }, 0.2);
   tl.to("#accent", { scaleX: 1,        duration: 0.7, ease: "power2.out" }, 0.7);
   ```

3. **Register the timeline by composition-id.** This is the step that makes everything smooth — miss it and the render will show jump-cuts:
   ```js
   window.__timelines = window.__timelines || {};
   window.__timelines["my-scene"] = tl;   // key must match data-composition-id
   ```
   The engine scans `window.__timelines[compositionId]` during compilation, wraps `.seek()` with the invalidation hooks chrome-headless-shell's BeginFrame mode requires, and drives the playhead one frame-time at a time. Without this wrapper a manual `tl.seek(t)` style commit isn't guaranteed to flush to the compositor before the frame is captured — you get stalls followed by "everything pops at once" jumps.

> **Do not** set `window.__hf = { duration, seek }` directly and hope for the best. That protocol is the engine's *internal* interface; using it bypasses the compositor invalidation wrapper and produces the exact "sloppy, no smooth motion" symptom described above. `window.__timelines[id]` is the public contract.

### Canonical template — copy this

```html
<!doctype html>
<html>
<head>
  <style>
    html, body { margin: 0; height: 100%; background: #0f172a; overflow: hidden;
      font-family: "Inter", system-ui, sans-serif; }
    #title { color: #e2e8f0; font-weight: 800; font-size: 96px;
      opacity: 0; transform: translateY(40px); }
    #accent { position: absolute; left: 50%; bottom: 12%;
      width: 240px; height: 6px; background: #38bdf8;
      transform: translateX(-50%) scaleX(0); transform-origin: left center; }
    .scene { width: 100%; height: 100%; display: grid; place-items: center; }
  </style>
  <script src="https://unpkg.com/gsap@3.13.0/dist/gsap.min.js"></script>
</head>
<body>
  <!-- Composition root: duration is authoritative -->
  <div class="scene"
       data-composition-id="q4-intro"
       data-width="1280"
       data-height="720"
       data-duration="3">
    <div id="title">Q4 Wrap</div>
    <div id="accent"></div>
  </div>
  <script>
    // Paused timeline — HyperFrames drives the playhead.
    const tl = gsap.timeline({ paused: true });
    tl.to("#title",  { opacity: 1, y: 0,      duration: 0.9, ease: "power3.out" }, 0.2);
    tl.to("#accent", { scaleX: 1,             duration: 0.7, ease: "power2.out" }, 0.7);
    tl.to({},        { duration: 1.2 });                                            // hold
    tl.to("#title",  { opacity: 0, y: -20,    duration: 0.5, ease: "power2.in"  }, 2.5);

    // Register by composition-id. This is what the engine looks for.
    window.__timelines = window.__timelines || {};
    window.__timelines["q4-intro"] = tl;
  </script>
</body>
</html>
```

Render it:

```bash
JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html \
  --aspectRatio=16:9 \
  --durationMs=3000 \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id="$JOB" --json | jq '.data.job.result.outputPath'
```

You can also pass `--html='<!doctype html>...'` inline — the renderer stages it as a temp file so relative asset paths still work. Use `htmlPath` for anything non-trivial so you can preview it in a browser first (a well-authored HyperFrames page is debuggable: open it, call `__hf.seek(1.5)` in devtools, see exactly what frame 45 will look like).

Result is an MP4 at the requested dimensions and duration. Drop it onto a project with `project.add-motion-graphic` exactly like a `motion.generate` output.

### Why CSS keyframes alone don't work

CSS `@keyframes … forwards` animations are driven by the page's wallclock at load time. The moment the page renders, they start playing — so by the time HyperFrames calls `seek(0)` the animation may already be finished. **CSS transitions and keyframes are only safe inside a paused GSAP timeline** (where GSAP applies them by setting styles at each seek), or via `animation-play-state: paused` if you're driving `animation-delay` from JS.

**Rule of thumb:** if you want a text "rise in" effect, use `gsap.to("#text", { opacity: 1, y: 0, … })` inside a paused timeline — not `@keyframes rise … forwards`.

### Transparent overlays — `--transparent`

Add `--transparent` to get a **WebM with a VP9/alpha channel** instead of an opaque MP4. Use this for anything that needs to composite on top of existing video: lower thirds, watermarks, bug logos, corner clocks, branded name plates.

**Two things change:**

1. Your CSS must keep the page background transparent:
   ```css
   html, body { background: transparent; }
   ```
2. Output extension becomes `.webm`.

Everything else — the composition root, the `__hf.seek` contract, paused GSAP timelines — is identical. Semi-transparent elements (e.g. `rgba(0,0,0,0.8)` card backgrounds) composite correctly via VP9/yuva420p.

**Constraint:** `--audioPath` is ignored in transparent mode. VP9+alpha is a video-only stream; mux audio in at composite time.

```html
<!doctype html>
<html>
<head>
  <style>
    html, body { margin: 0; height: 100%; background: transparent; overflow: hidden;
      font-family: "Inter", system-ui, sans-serif; }
    .card { position: absolute; left: 5vw; bottom: 9vh;
      background: rgba(0,0,0,0.85); border-radius: 10px; padding: 14px 22px;
      color: #fff; opacity: 0; transform: translateY(20px); }
    .name  { font-size: 26px; font-weight: 700; }
    .role  { font-size: 15px; opacity: .65; margin-top: 3px; }
  </style>
  <script src="https://unpkg.com/gsap@3.13.0/dist/gsap.min.js"></script>
</head>
<body>
  <div data-composition-id="lower-third"
       data-width="1920" data-height="1080" data-duration="3.5">
    <div class="card" id="card">
      <div class="name">Alex Rivera</div>
      <div class="role">Senior Product Designer</div>
    </div>
  </div>
  <script>
    const tl = gsap.timeline({ paused: true });
    tl.to("#card", { opacity: 1, y: 0, duration: 0.5, ease: "power3.out" }, 0.2);
    tl.to({},      { duration: 2.3 });
    tl.to("#card", { opacity: 0, y: 20, duration: 0.5, ease: "power2.in" }, 3.0);
    window.__timelines = window.__timelines || {};
    window.__timelines["lower-third"] = tl;
  </script>
</body>
</html>
```

```bash
JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/lower-third.html \
  --aspectRatio=16:9 \
  --durationMs=3500 \
  --transparent \
  --json | jq -r '.data.jobId')

# outputPath will end in .webm, not .mp4
pandastudio job.wait --id="$JOB" --json | jq '.data.job.result.outputPath'
```

The resulting `.webm` attaches with `project.add-motion-graphic` — same call as any other motion graphic.

### Common authoring mistakes

- **Registering via `window.__hf = { duration, seek }` instead of `window.__timelines[id] = tl`.** The engine accepts `__hf` without throwing, but a manual `tl.seek(t)` inside that `seek` function doesn't force the paused timeline to commit its styles to chrome-headless-shell's BeginFrame compositor. The MP4 renders with identical frames for ~1-second stretches followed by sudden jumps — the exact "sloppy, no smooth motion" symptom. **Always use `__timelines[id]`.**
- **Forgetting `{ paused: true }` on the GSAP timeline.** The timeline plays once on page load; capture starts *after* the page is quiescent, so your whole animation may already have run. Always build timelines paused.
- **CSS `@keyframes … forwards` for entrances.** Not seekable — CSS animations are rAF-clock-driven, not timeline-driven. Replace every entrance with `gsap.to(...)` inside the timeline.
- **Mismatch between `data-composition-id` and the `__timelines` key.** Must be exactly equal. The engine finds the timeline by this key; a typo means the engine registers nothing and the render is static.
- **`data-duration` mismatch with the timeline's actual length.** Set `data-duration` to the longest-ending tween's end time. Over-long wastes frames on a blank tail; under-long clips the animation. Keep them in sync.
- **Using `setTimeout`/`setInterval` to orchestrate scene changes.** They run on page wallclock, not composition time, and won't be seeked. Put every visual state change on the GSAP timeline.
- **Local assets referenced with absolute filesystem paths.** The engine serves the HTML's directory as the web root; reference assets with relative URLs (`./logo.png`) and pass them via `--assets`.

### Pacing — how to make it feel polished, not "amateur-TikTok fast"

Frame-perfect rendering doesn't make bad timing look good. Empirically-tested rules of thumb for agent-authored promos:

- **Scene duration ≥ 4 s** for anything with more than ~3 elements. Dense scenes (multi-row tables, 3+ cards) need 5–7 s.
- **Pre-hold of 0.3–0.5 s** before the first element animates. Gives the viewer's eye time to settle on the frame before content starts moving.
- **Stagger ≥ 0.15 s** between sibling reveals (text words, card cascade, bullet rows). Sub-100 ms stagger reads as "flash, flash, flash" — the viewer can't parse individual entrances.
- **Post-hold ≥ 1.5 s** after the last reveal before the scene ends. The viewer needs time to *read*, not just *see*.
- **Softer easing for body copy, punchier easing for accents.** `power2.out` (gentle landing) for headlines and cards; `power3.out` or `back.out(1.4)` for accents, buttons, icons. Avoid `power3.out` on long strings of staggered words — it compresses the reveal into a bang.
- **Keep every scene moving.** A 2-second static hold after a reveal reads as "the render froze." Add a slow 3–5% zoom or a 20–30 px pan during holds so the composition continues to breathe.

### Non-GSAP drivers (escape hatch)

If you're NOT using GSAP — e.g. Lottie player, three.js, canvas-driven animation — the lower-level contract is `window.__hf = { duration, seek }`:

```js
window.__hf = {
  duration: 3,
  seek(t) { myCustomEngine.renderAtTime(t); }
};
```

This only works if your `seek(t)` synchronously forces a repaint (canvas: redraw; three.js: `renderer.render(scene, camera)`; Lottie: `anim.goToAndStop(t * 1000, false)`). GSAP paused-timeline `.seek()` does **not** meet that bar under headless-shell BeginFrame — that's why GSAP compositions must register via `__timelines[id]` instead.

### HyperFrames block catalog — don't re-invent when you can install

Before writing HTML from scratch, check the HyperFrames catalog. It's a curated library of high-quality pre-built compositions (social-overlay blocks, cinematic shader transitions, effects) that install as drop-in `.html` files and render through the exact same `motion.render-html` path as any hand-authored composition. They already follow the `window.__timelines[id]` contract and have been tuned by the HeyGen team. For common patterns — YouTube lower thirds, Instagram-follow cards, shader transitions between scenes — these are sharper, better-paced, and faster than writing your own.

**Install a block:**

```bash
# Pick a working directory (doesn't matter where — the composition is
# just an .html file that PandaStudio renders by path).
cd /tmp/my-promo && npm init -y >/dev/null

# Install one — creates compositions/<slug>.html and any required assets.
npx hyperframes add instagram-follow
# → compositions/instagram-follow.html
# → assets/avatar.jpg (may be replaceable depending on the block)

# Then render it straight through motion.render-html:
JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/my-promo/compositions/instagram-follow.html \
  --width=1080 --height=1920 \
  --durationMs=4500 \
  --transparent \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id=$JOB --json | jq '.data.job.result.outputPath'
```

**Blocks — full-scene compositions you can render standalone:**

| Slug | Category | Notes |
|------|----------|-------|
| `yt-lower-third` | Broadcast | YouTube subscribe lower-third with avatar + channel info. Use `--transparent`. |
| `instagram-follow` | Social | IG-style profile card + follow button slide-in. Use `--transparent`. |
| `tiktok-follow` | Social | TikTok-style follow overlay. Use `--transparent`. |
| `spotify-card` | Social | Now-playing card with album art + progress. Use `--transparent`. |
| `x-post` | Social | X/Twitter post overlay with engagement metrics. Use `--transparent`. |
| `reddit-post` | Social | Reddit post card with upvotes + comments. Use `--transparent`. |
| `macos-notification` | UI | macOS notification banner. Use `--transparent`. |
| `app-showcase` | UI | Three floating smartphone screens (fitness-app style). Opaque. |
| `data-chart` | Data viz | Bar + line chart, NYT-style typography, staggered reveal. Opaque. |
| `flowchart` | Data viz | Animated decision tree with SVG connectors + typing cursor. Opaque. |
| `logo-outro` | Branding | Piece-by-piece logo assembly + tagline fade + URL pill. Opaque. |
| `ui-3d-reveal` | UI | Perspective-3D reveal of UI elements. Opaque. |

**Shader transitions — render as a `.mov`/`.webm` that sits *between* two scenes on the timeline:**

`chromatic-radial-split`, `cinematic-zoom`, `cross-warp-morph`, `domain-warp-dissolve`, `flash-through-white`, `glitch`, `gravitational-lens`, `light-leak`, `ridged-burn`, `ripple-waves`, `sdf-iris`, `swirl-vortex`, `thermal-distortion`, `whip-pan`.

Each is a short (0.5–1.2 s) effect that cross-warps or morphs between two still frames. They are authored to accept two `input_images` via the composition HTML's `data-` attributes — open the installed `.html` and follow the inline comments to wire your two scene frames in. Render them with `--transparent` only if the specific block's CSS has transparent backgrounds (most don't — they fill the full frame with the effect).

**Transition showcases** (`transitions-3d`, `transitions-blur`, `transitions-cover`, `transitions-destruction`, `transitions-dissolve`, `transitions-distortion`, `transitions-grid`, `transitions-light`, `transitions-mechanical`, `transitions-other`, `transitions-push`, `transitions-radial`, `transitions-scale`) are bundles that demo multiple related effects in a single composition — useful for picking which specific transition to use before installing the dedicated shader block.

**Components — animated effects you can layer inside your own compositions:**

- `grain-overlay` — animated film-grain texture, ideal on top of static imagery.
- `grid-pixelate-wipe` — screen dissolves into staggered grid squares.
- `shimmer-sweep` — light sweep across text/elements via CSS gradient mask.

**Prefer the catalog when:**
- The user asks for something the catalog matches by name (name-plate, follow overlay, chart, logo outro, whip-pan, light leak).
- You need a transition between two scenes — shader transitions are much better than anything GSAP-on-CSS can do in a browser.
- Speed matters — `npx hyperframes add` completes in seconds and you skip the authoring iteration loop entirely.

**Write custom HTML when:**
- The brief is specific to the user's content (partli.app promo, personalised intro with their transcript) and no catalog block approximates it.
- You need layout control the catalog doesn't provide (multi-scene bespoke promo).
- You're composing several blocks together — put catalog blocks inside a parent composition via `<div data-composition-src="compositions/<slug>.html" ...>` (the engine inlines the sub-composition's timeline into the parent automatically).

**Full catalog URL:** https://hyperframes.heygen.com/catalog — browse previews and confirm slugs before installing.

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

# 3b. Remove long silences (default ≥700ms; covers leading/trailing/between-word)
pandastudio transcript.remove-silences --id=$ID --thresholdMs=700 --json
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

### Creating a pure motion-graphics promo (no source footage)

<HARD-RULE>
**Always concat before placing on the timeline.**
When you render multiple scenes (intro, body, CTA, …), you MUST join them with
`motion.concat` into a single MP4 **before** calling `project.add-clip`.
Never place individual scene segments as separate timeline clips.

Why: PandaStudio's timeline treats each clip as an independent source with its
own audio decode context. Multiple motion-graphic clips therefore play as
separate back-to-back sources — no cross-clip transitions, no unified audio,
and any background music starts again between clips. A single concatenated MP4
plays as one seamless piece with one audio decode pass.

The only exception: if the user explicitly says "I want each scene as its own
clip so I can trim them independently" — even then, add clips individually only
after asking to confirm.
</HARD-RULE>

When the user wants a promo video with no recorded footage — only rendered scenes:

```bash
# STEP 1: Create the project FIRST, before rendering anything.
# If you render scenes first and the render pipeline fails mid-way,
# you have no project to attach the completed scenes to.
P=$(pandastudio project.new --name="WritePanda Promo" --json)
ID=$(echo "$P" | jq -r '.data.id')

# STEP 2: Render scenes IN PARALLEL — fire them all, wait on each jobId.
# Since v1.17 each render spawns its own Chromium OS process via
# Puppeteer (no shared BrowserWindow / no global lock). Measured on
# Apple Silicon: 3 concurrent renders complete in the time of one
# (35s → 6.9s, a 5× speedup vs. sequential). `isRenderBusy` always
# returns false; there's nothing to serialise against.
#
# Fire every scene with `&` (shell background) OR collect jobIds first
# and job.wait each in turn — the second call doesn't block the first.
# Don't `job.wait` between fires; that serialises for no reason.
JOB1=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene1.html --durationMs=4000 --json | jq -r '.data.jobId')
JOB2=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene2.html --durationMs=3000 --json | jq -r '.data.jobId')
# Both running in parallel now. Wait on each result.
SCENE1=$(pandastudio job.wait --id="$JOB1" --json | jq -r '.data.job.result.outputPath')
SCENE2=$(pandastudio job.wait --id="$JOB2" --json | jq -r '.data.job.result.outputPath')
SCENE2=$(pandastudio job.wait --id="$JOB2" --json | jq -r '.data.job.result.outputPath')

# STEP 3: ALWAYS concat first — ONE clip on the timeline.
# motion.concat is a lossless stream copy; completes in < 1 s.
FINAL=$(pandastudio motion.concat \
  --clips="[$SCENE1,$SCENE2]" \
  --outputName=promo-final \
  --json | jq -r '.data.outputPath')

# STEP 4: Add the single merged MP4 as one clip.
pandastudio project.add-clip --id=$ID --media="$FINAL" --json

# STEP 5: Add background music (optional)
pandastudio project.add-audio --id=$ID --audioPath=/path/to/music.mp3 \
  --volume=0.6 --json

# STEP 6: Preview, then export
pandastudio preview.show --id=$ID
pandastudio export.start --id=$ID --quality=high --json
```

### Recipe: SaaS product promo / intro video (the "Teamble / Linear / Arc" style)

This is the single most common request ("make me a 60–90s product promo / intro
video"). It's a **learnable template**, not a freeform creative task — follow
the structure below and the output will land every time.

**Canonical length:** 60–90s total (Teamble reference is 98s, Linear/Arc promos
land ~60s). Never longer than 120s for a promo — viewers drop off.

**7-act structure** (timings scale proportionally to total length):

| Act | % of runtime | Purpose | Typical shot |
|---|---|---|---|
| 1. Hook | 0–5% | One-line positioning statement | Gradient typography card |
| 2. Brand | 5–8% | Logo lockup | Logo reveal with glow |
| 3. Problem | 8–20% | 2–3 giant word callouts | `word-pop` cards on gradient bg |
| 4. Product snippets | 20–55% | 6–10 UI cutaways, 2–4s each | Zoomed UI fragment + animated cursor + glow accent |
| 5. Act break | 55–60% | Chapter divider / "and here's how" | `chapter-divider` or pull-quote |
| 6. Benefit montage | 60–88% | Alternate typography callouts + UI snippets | Mix `word-pop` + `pull-quote` + UI zooms |
| 7. CTA / Logo | 88–100% | URL, logo return, tagline | `end-screen` or `channel-intro` |

**Pacing rule:** 2–4 seconds per shot. A 90s promo has **25–35 distinct shots**.
If you're building fewer than 20 shots, it will feel slow and linear. If more
than 40, it will feel frenetic — that's fine for Shorts, wrong for a 16:9 promo.

**Design language (SaaS-promo aesthetic):**
- Dark background (near-black) with **purple → magenta → pink gradients** and
  soft bokeh accents. NOT the default Panda green — pick `mkbhd` theme as the
  closest out-of-the-box match, OR author custom HTML for the full purple/pink
  look (example below).
- Bold sans-serif type (Inter / SF / GT America). White or gradient fill.
- Every UI shot has: a tilted framing, an animated cursor glyph on the
  important control, and an accent glow around the element.
- Sparkle / particle accents on reveal moments.
- **No voiceover**. Music + typography carry the story. If the user insists on
  VO, treat it as a different format (explainer, not promo) — the pacing rules
  change.

**Primitive mapping — which motion template does which job:**

| Shot role | Motion template | Typical duration |
|---|---|---|
| Opening tagline | `14-word-pop` or custom HTML | 2500 ms |
| Logo reveal | `08-channel-intro` or custom HTML | 2000 ms |
| Single-word callout ("Help", "10x better") | `14-word-pop` | 1500–2000 ms |
| Benefit statement ("uncovers hidden trends") | `10-pull-quote` | 2500 ms |
| Metric / score reveal | `09-stat-reveal` | 2000 ms |
| Act break ("Let's see how easy") | `07-chapter-divider` | 2000 ms |
| Focused UI element spotlight | `19-spotlight-ring` | 2000 ms |
| Burst emphasis / sparkle moment | `15-reaction-burst` | 1200 ms |
| Outro / CTA | `05-end-screen` or `04-subscribe-cta` | 3000 ms |
| UI fragment (from screen recording) | `project.add-clip` + `project.add-zoom` into the relevant region | 2500–3500 ms |

Run `pandastudio motion.list --json` to see all 20 templates with slots.

**PandaStudio limitation — honest note:** the editor does NOT render 3D
perspective tilts on screen-recording clips. The Teamble reference achieves
its "tilted UI card floating in space" via AfterEffects. In PandaStudio, you
have three workarounds:

1. **Zoom-in on UI (default):** use `project.add-zoom` to push into the
   relevant UI element. Reads similarly on screen, ships today.
2. **Custom HTML motion graphic with CSS `transform: rotate3d()`:** wrap a
   screenshot of the UI element in an HTML scene, apply 3D transform, render
   as a transparent WebM, overlay with `project.add-motion-graphic`. This
   genuinely matches the reference but requires authoring custom HTML per shot.
3. **Record the UI in an AE-style wrapper externally** and bring in as a clip.

If the user wants the 3D-tilted look and accepts longer build time, go with
(2). Otherwise (1) is the default.

**Default audio + captions for this recipe:**

- **Music:** one of the `kinetic-product-drive-*` tracks (bundled — they were
  generated specifically for this format). Volume `0.5–0.7` because there's no
  VO to duck under. Fade in 500 ms, fade out 1000 ms.
- **Captions:** NONE on a pure promo. Typography motion graphics carry the
  narrative. If the user wants captions (e.g., for LinkedIn autoplay-muted),
  use `panda-neon` positioned at `positionY=0.8` so it never overlaps the
  typography cards.
- **LUT on source UI clips:** `modernVibrant` at intensity 0.5 to match the
  saturated gradient aesthetic of the surrounding motion graphics.

**Shot-list executable template** (scale to your runtime):

```bash
# ── Setup ─────────────────────────────────────────────────────────
P=$(pandastudio project.new --name="$PRODUCT Promo" --aspectRatio=16:9 --json)
ID=$(echo "$P" | jq -r '.data.id')

# Pick the kinetic product-drive track with durationMs closest to your target
MUSIC=$(pandastudio asset.list-music --json \
  | jq -r '.data.tracks[] | select(.intents | index("product_video")) | .absolutePath' \
  | head -1)

# ── Act 1: Hook (single-line tagline, 2.5s) ──────────────────────
JOB=$(pandastudio motion.generate \
  --templateId=14-word-pop --themeId=mkbhd \
  --slots='{"word":"The people success platform"}' \
  --durationMs=2500 --json | jq -r '.data.jobId')
HOOK=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# ── Act 2: Brand (logo reveal, 2s) ───────────────────────────────
JOB=$(pandastudio motion.generate \
  --templateId=08-channel-intro --themeId=mkbhd \
  --slots='{"channelName":"teamble","handle":"ai"}' \
  --durationMs=2000 --json | jq -r '.data.jobId')
BRAND=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# ── Act 3: Problem — 3 giant word-pop cards ──────────────────────
for WORD in "Help" "10x better" "faster feedback"; do
  JOB=$(pandastudio motion.generate \
    --templateId=14-word-pop --themeId=mkbhd \
    --slots="{\"word\":\"$WORD\"}" --durationMs=1800 --json | jq -r '.data.jobId')
  P3+=("$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')")
done

# ── Act 4: Product snippets (screen recordings + zooms) ──────────
# The user provides a full-UI screen recording as $UI_REC.
# Instead of one long clip, split into 6–10 short 2.5–3.5s beats,
# zooming into a different UI element on each beat.
pandastudio project.add-clip --id=$ID --media="$UI_REC" --json
CLIP_ID=$(pandastudio project.read --id=$ID --json \
  | jq -r '.data.project.clips[-1].id')

# Apply LUT for the saturated gradient aesthetic
pandastudio project.set-clip-lut --id=$ID --clipId="$CLIP_ID" \
  --lutPreset=modernVibrant --lutIntensity=0.5 --json

# Stack zooms on each UI beat (durations match the narrative)
pandastudio project.add-zoom --id=$ID --clipId="$CLIP_ID" \
  --startMs=0 --endMs=2500 --targetX=0.45 --targetY=0.50 --zoom=1.8 --json
pandastudio project.add-zoom --id=$ID --clipId="$CLIP_ID" \
  --startMs=2500 --endMs=5500 --targetX=0.72 --targetY=0.35 --zoom=2.2 --json
# ... repeat for 6–10 zooms total, rotating focus across UI regions

# ── Act 5: Act break (chapter divider, 2s) ───────────────────────
JOB=$(pandastudio motion.generate \
  --templateId=07-chapter-divider --themeId=mkbhd \
  --slots='{"chapter":"Let'\''s see how easy"}' \
  --durationMs=2000 --json | jq -r '.data.jobId')
ACTBREAK=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# ── Act 6: Benefit montage — alternate pull-quotes + UI zooms ────
JOB=$(pandastudio motion.generate \
  --templateId=10-pull-quote --themeId=mkbhd \
  --slots='{"quote":"uncovers hidden trends"}' \
  --durationMs=2500 --json | jq -r '.data.jobId')
BENEFIT1=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

JOB=$(pandastudio motion.generate \
  --templateId=09-stat-reveal --themeId=mkbhd \
  --slots='{"stat":"41","label":"feedback score"}' \
  --durationMs=2000 --json | jq -r '.data.jobId')
STAT=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# ── Act 7: CTA / Logo return ─────────────────────────────────────
JOB=$(pandastudio motion.generate \
  --templateId=05-end-screen --themeId=mkbhd \
  --slots='{"cta":"teamble.ai","tagline":"Make feedback effortless"}' \
  --durationMs=3000 --json | jq -r '.data.jobId')
OUTRO=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.job.result.outputPath')

# ── Concat the pre-UI segments and the post-UI segments ──────────
# Rule from the concat hard-rule above: motion segments MUST be merged
# into single MP4s before hitting the timeline, else audio/transitions break.
INTRO=$(pandastudio motion.concat \
  --clips="[$HOOK,$BRAND,${P3[0]},${P3[1]},${P3[2]}]" \
  --outputName=promo-intro --json | jq -r '.data.outputPath')

OUTRO_BUNDLE=$(pandastudio motion.concat \
  --clips="[$ACTBREAK,$BENEFIT1,$STAT,$OUTRO]" \
  --outputName=promo-outro --json | jq -r '.data.outputPath')

# ── Timeline assembly ────────────────────────────────────────────
# Order: intro motion → UI clip (with zooms) → outro motion.
# Prepend intro, append outro (project.add-clip in order).
# Note: this needs a project that currently has ONLY the UI clip —
# if you're starting fresh, create the project with media=INTRO
# so it's first on the timeline, then add the UI clip, then the outro.
pandastudio project.add-clip --id=$ID --media="$OUTRO_BUNDLE" --json

# ── Music ────────────────────────────────────────────────────────
pandastudio project.add-audio --id=$ID --audioPath="$MUSIC" \
  --volume=0.6 --fadeIn=500 --fadeOut=1000 --json

# ── Preview, then export ─────────────────────────────────────────
pandastudio preview.show --id=$ID
# Watch the full preview. Adjust zoom targets if UI beats feel off.
pandastudio export.start --id=$ID --quality=high --json
```

**Common mistakes agents make on promos (avoid these):**

- **One long UI clip with no zooms.** Screen recording for 40s straight reads
  as "tutorial," not "promo." You MUST break the UI section into 6–10
  zoom-driven beats of 2–4s each.
- **Typography cards longer than 3 seconds.** Viewers read a 3-word title in
  500 ms; anything over 2500 ms feels dead. Keep motion-graphic shots tight.
- **Voiceover on a promo.** This format is music + typography. If VO is
  required, change format (explainer ≠ promo — different pacing, different
  music volume, different caption strategy).
- **Default Panda green theme on a SaaS product promo.** It reads as tutorial/
  educational, not product-marketing. Use `mkbhd` or author custom HTML with
  the product's actual brand palette.
- **Skipping `motion.concat` between motion-graphic scenes.** Revisit the
  hard-rule at the top of the promo section — always merge before adding to
  timeline.
- **Forgetting the CTA/outro.** A promo without a CTA is a trailer. Always
  include Act 7 with a URL, brand lockup, or "Try it today" line.

**When the user wants the full Teamble look (custom purple-gradient HTML):**

If the default `mkbhd` theme isn't enough and the user wants the exact
gradient-bokeh aesthetic, author a reusable HTML template shell:

```html
<!-- saas-promo-word.html — gradient bg + word-pop, fully themeable via slots -->
<!DOCTYPE html><html><head><style>
  body { margin:0; width:1920px; height:1080px;
    background: radial-gradient(ellipse at 30% 40%, #3b1566 0%, #0a0a14 60%);
    font-family: 'Inter', sans-serif; color: #fff;
    display: flex; align-items: center; justify-content: center; }
  .bokeh { position:absolute; border-radius:50%; filter:blur(80px); opacity:0.6; }
  .b1 { left:-100px; bottom:-100px; width:500px; height:500px; background:#e53fc5; }
  .word { font-size:180px; font-weight:800; letter-spacing:-4px;
    opacity:0; transform:translateY(30px);
    animation: rise 500ms cubic-bezier(0.16, 1, 0.3, 1) 100ms forwards; }
  @keyframes rise { to { opacity:1; transform:translateY(0); } }
</style></head><body>
  <div class="bokeh b1"></div>
  <div class="word">{{WORD}}</div>
</body></html>
```

Render each word-pop variant with `motion.render-html`, passing a different
`{{WORD}}` slot substitution per scene, and concat them. This gives the
authentic SaaS-promo look but costs ~30 min of HTML tuning upfront vs. using
the stock `14-word-pop` template.

### Local images in rendered HTML — use `--assets`, not base64

When your HTML references local images, pass them via `--assets` instead of
base64-encoding them inline. The renderer stages the files alongside the HTML
so `<img src="logo.png">` works directly:

```bash
pandastudio motion.render-html \
  --html='<html>...<img src="logo.png">...</html>' \
  --assets='["/Users/me/logo.png","/Users/me/bg.jpg"]' \
  --durationMs=3000 --json
```

Without `--assets`, you'd need to base64-encode each image inline — the
resulting HTML can be 3-5 MB per scene, which is slow to parse and hits
Chromium's data-URL limits.

### Default image presentation — tilted "floating device" look

<HARD-RULE>
Any `<img>` inside a `motion.render-html` scene MUST be wrapped in the
tilted floating-device presentation below by default. Motion-graphic scenes
are stylized contexts by nature — a flat, axis-aligned image reads as
unpolished and out of style against the typography scenes around it.

The only exception: if the user explicitly says "flat image" / "no tilt" /
"axis-aligned," render without the wrapper. Never silently downgrade to flat.
</HARD-RULE>

The presentation has four ingredients, each doing a specific job:

1. **3D perspective tilt** — makes the image feel like a floating device, not a screenshot
2. **Soft drop shadow** — anchors it in the gradient background instead of floating flat
3. **Edge glow** — accent color (pink/purple/brand) around the frame, reinforces the palette
4. **Optional animated cursor** — purple gradient cursor glyph on the important UI element

Drop this CSS block into every HTML scene that contains an image. The
`.stage-tilt` class is the only thing you need to apply to the wrapper —
everything else auto-styles from there.

```html
<!DOCTYPE html>
<html><head><style>
  :root {
    --tilt-accent: #c44eff;      /* edge glow — override per brand */
    --tilt-accent-2: #ff4ea3;    /* cursor fill */
    --tilt-bg-from: #3b1566;     /* radial gradient inner */
    --tilt-bg-to: #0a0a14;       /* radial gradient outer */
  }
  body {
    margin: 0; width: 1920px; height: 1080px;
    background: radial-gradient(ellipse at 30% 40%,
      var(--tilt-bg-from) 0%, var(--tilt-bg-to) 60%);
    display: flex; align-items: center; justify-content: center;
    perspective: 1600px;          /* REQUIRED for rotate3d to render depth */
  }
  /* Wrap any <img> (or a <div> with a background-image) in .stage-tilt */
  .stage-tilt {
    transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg);
    transform-origin: center center;
    border-radius: 18px;
    overflow: hidden;
    box-shadow:
      0 40px 80px -20px rgba(0, 0, 0, 0.55),       /* drop shadow */
      0 0 0 1px rgba(255, 255, 255, 0.06),          /* hairline */
      0 0 60px 8px color-mix(in srgb, var(--tilt-accent) 30%, transparent);
    animation: tilt-rise 700ms cubic-bezier(0.16, 1, 0.3, 1) both;
  }
  .stage-tilt img, .stage-tilt video {
    display: block; width: 100%; height: auto;
  }
  @keyframes tilt-rise {
    from { opacity: 0; transform: rotate3d(1, 0.25, 0, 22deg) rotateZ(-2deg) translateY(40px); }
    to   { opacity: 1; }
  }
  /* Optional: animated cursor glyph — absolute-position it over a UI control */
  .tilt-cursor {
    position: absolute; width: 64px; height: 64px; pointer-events: none;
    filter: drop-shadow(0 6px 18px color-mix(in srgb, var(--tilt-accent-2) 55%, transparent));
    animation: cursor-tap 1400ms ease-in-out infinite;
  }
  @keyframes cursor-tap {
    0%, 100% { transform: translate(0, 0) scale(1); }
    50%      { transform: translate(-4px, -4px) scale(0.92); }
  }
</style></head>
<body>
  <div class="stage-tilt" style="width: 900px;">
    <img src="product-screenshot.png" />
  </div>
  <!-- Optional: drop a cursor glyph SVG into .tilt-cursor at an exact x/y -->
</body></html>
```

**Override knobs** (pass through `--slots` or edit the CSS vars):

| Variable | Default | When to change |
|---|---|---|
| `--tilt-accent` | `#c44eff` (purple) | Match the product's brand color |
| `--tilt-accent-2` | `#ff4ea3` (pink) | Cursor glow; usually accent's sibling hue |
| `--tilt-bg-from` / `--tilt-bg-to` | purple → near-black | For light-theme promos, swap to off-white gradient |
| `rotate3d` angle | `16deg` | Reduce to `8deg` for subtler tilt; increase to `22deg` for more dramatic |
| `rotateZ` | `-1deg` | Flip sign to tilt the other way — rotate between scenes for variety |

**When NOT to use this wrapper** (narrow list):

- User explicitly asked for a flat/axis-aligned image
- The image IS the entire frame (a full-bleed photo with no UI) — tilting a photo looks weird; apply only to UI, product cards, screenshots
- The scene is a `07-chapter-divider` or text-only card — tilt is for images, not type

**Rotation variety across a multi-scene promo:** when you have 3+ image
scenes back-to-back, alternate the Z rotation between `-1deg` and `+1deg`
and vary the X rotation between `12deg`, `16deg`, and `20deg` across
scenes. A uniform tilt angle across every scene reads as templated; small
variation reads as crafted.

### No scene is static — always-on motion rule

<HARD-RULE>
**Every `motion.render-html` scene MUST have motion for its full duration.**
A scene that holds a still image for 3 seconds reads as a slideshow and
breaks the "motion graphics" illusion — viewers perceive it as a freeze.
If the content itself doesn't move (typography card, single product shot,
logo reveal), add ambient motion. No exceptions for promo/intro/outro
formats.
</HARD-RULE>

Three ways to satisfy the rule. Pick the cheapest that fits the shot:

**1. Subtle Ken Burns on any image (preferred for UI/product shots).**
A slow pan + zoom on the `.stage-tilt` wrapper — imperceptible per-frame
but unmistakably alive over 3 seconds. Runs in parallel with `tilt-rise`.

```css
.stage-tilt {
	/* entrance (from the earlier rule) keeps running */
	animation:
		tilt-rise 700ms cubic-bezier(0.16, 1, 0.3, 1) both,
		ken-burns 6s ease-in-out 500ms both;
}
@keyframes ken-burns {
	0%   { transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg) scale(1)   translate(0, 0); }
	100% { transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg) scale(1.06) translate(-14px, -6px); }
}
```

Keep the motion deliberately tiny — `scale(1.06)` and ±14px over 6s is
invisible as a single step but feels cinematic over the shot. Alternate
direction per scene (left-up → right-down → left-down → right-up) for
variety on multi-shot sequences.

**2. Animated gradient / shine sweep (preferred for typography + logo).**
Typography cards shouldn't pan; they should shimmer. Animate the gradient
behind the text or sweep a highlight across it.

```css
.bg-shimmer {
	background: linear-gradient(120deg, #3b1566 0%, #0a0a14 40%, #3b1566 80%, #0a0a14 100%);
	background-size: 220% 100%;
	animation: shimmer 8s ease-in-out infinite;
}
@keyframes shimmer {
	0%, 100% { background-position: 0% 0%; }
	50%      { background-position: 100% 0%; }
}

/* Or a shine sweep over a word */
.shine {
	background: linear-gradient(90deg, transparent 20%, rgba(255,255,255,0.4) 50%, transparent 80%);
	background-size: 220% 100%;
	-webkit-background-clip: text;
	animation: sweep 2200ms ease-in-out infinite;
}
@keyframes sweep {
	0%   { background-position: 100% 0%; }
	100% { background-position: -100% 0%; }
}
```

**3. Bokeh / particle drift (preferred for ambient pads + CTA outros).**
Give bokeh light-blurs a slow drift. Almost free computationally and
always-on.

```css
.bokeh {
	animation: drift 14s ease-in-out infinite alternate;
}
@keyframes drift {
	0%   { transform: translate(0, 0) scale(1); }
	100% { transform: translate(30px, -20px) scale(1.15); }
}
```

**Rule-of-thumb mapping for the 7-act promo recipe:**

| Act / shot type | Default always-on motion |
|---|---|
| Hook (typography) | Shimmer gradient + bokeh drift |
| Brand / logo reveal | Bokeh drift + logo subtle scale 1 → 1.02 loop |
| Word-pop callouts | Shine sweep across the word; bg shimmer |
| UI / product screenshot | Ken Burns on `.stage-tilt` + bokeh drift |
| Pull-quote / benefit | Shimmer gradient |
| Metric / stat reveal | Counter count-up + bokeh drift |
| Chapter divider | Shine sweep + bokeh drift |
| CTA / end screen | Bokeh drift + CTA button pulse (scale 1 → 1.03) |

### Multi-image scenes — never identical, always varied

When a single scene contains **multiple images** (product gallery,
feature grid, testimonial wall, before-after, 3-up comparison), a uniform
layout with identical tilts reads as a wireframe. The Teamble-style
reference gets variety by:

- Rotating Z-axis tilt direction per item: `-2deg`, `+3deg`, `-1deg`, `+4deg`
- Mixing X-axis tilts: shallower (`8-12deg`) for items in the foreground,
  steeper (`18-24deg`) for items in the back, so depth is implied
- Staggering entrance: each image uses `animation-delay: 0ms, 150ms, 300ms, 450ms`
  — the scene "builds" rather than appearing all at once
- Varying scale: lead image at `scale(1)`, secondary at `scale(0.85)`, tertiary
  at `scale(0.7)` — implies focus + hierarchy

```css
.stage-tilt:nth-child(1) { transform: rotate3d(1, 0.25, 0, 12deg) rotateZ(-2deg) scale(1);    z-index: 3; animation-delay: 0ms; }
.stage-tilt:nth-child(2) { transform: rotate3d(1, 0.25, 0, 18deg) rotateZ( 3deg) scale(0.85); z-index: 2; animation-delay: 150ms; }
.stage-tilt:nth-child(3) { transform: rotate3d(1, 0.25, 0, 22deg) rotateZ(-1deg) scale(0.7);  z-index: 1; animation-delay: 300ms; }
```

**Placement patterns to choose from** (pick one per scene, don't mix):

- **Stacked cascade** — images offset diagonally, overlapping 15–20% each
- **Orbital** — one hero image center, supporting ones arranged around it
  (cf. Teamble's "Onboarding Survey" orbital shot)
- **Scattered** — images at random(ish) angles across the frame with
  generous negative space, each at a slightly different Z tilt
- **Stack-and-fan** — a focused hero + a fan of thumbnails behind it
- **Line-up** — 3 items in a horizontal row with descending Z-axis tilt

**Anti-patterns (do NOT do these):**

- Three images in a neat horizontal grid with identical tilts → looks like
  a product-listing page, not a promo
- Every image entering at the same moment → no rhythm
- Identical rotation signs (`rotateZ(-1deg)` on all) → feels machine-placed

### Quick layout sanity check before committing to a full render

A full 5-second scene at 30fps takes ~60s to render. To verify the layout
before committing, do a fast 1-frame check:

```bash
pandastudio motion.render-html \
  --htmlPath=/tmp/scene.html \
  --durationMs=100 --frameRate=1 \
  --outputName=preview-check --json
```

This captures a single frame in ~2s. Open the output MP4 (or screenshot it)
to verify positioning before launching the full render.

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
  | jq -r '.data.project.clips[0].id')

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
CLIP=$(echo "$P" | jq -r '.data.project.clips[0].id')

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

### motion.render-html with audio

```bash
# Render a scene with background music baked in
pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html \
  --durationMs=5000 \
  --audioPath=/path/to/music.mp3 \
  --audioVolume=0.8 \
  --json
```

Without `--audioPath`, the rendered MP4 is silent. Use either this (bakes
audio into the scene MP4) or `project.add-audio` (adds music at the project
level and exports with everything else). The project-level approach is more
flexible — the same music plays across all scenes.

### Validate layout with motion.screenshot BEFORE rendering

Full renders take 15–30 seconds. Use `motion.screenshot` to capture a single
PNG frame in under a second — validate fonts, layout, and element positions
before committing to a full encode.

```bash
# Check the first visible frame (t=0)
pandastudio motion.screenshot \
  --html='<html>...</html>' \
  --aspectRatio=16:9 \
  --outputName=preview-t0 \
  --json

# Check mid-animation (t=2s) to see the composition in motion
pandastudio motion.screenshot \
  --htmlPath=/tmp/product-scene.html \
  --atMs=2000 \
  --assets=/Users/me/product.png \
  --outputName=preview-t2000 \
  --json
```

`motion.screenshot` returns `{ outputPath }` directly — no `job.wait` needed.
Open the PNG, verify, then call `motion.render-html` with the same args.

### Frame-verify before declaring done — `motion.verify-frames`

`motion.screenshot` validates layout on a STILL composition. But a motion
graphic is only "done" when you've watched its rendered MP4 land at hero
moments — has the caption landed on the right word? Did the face transition
interpolate smoothly? Does the color-recolor beat actually flip? Lint
passing and preview snapshots won't catch a face that snapped or a caption
that drifted one word off.

**Contract:** after `motion.render-html` (or `export.start` for a full
project), extract 8–15 frames at hero timestamps and READ each PNG
(multimodal) before you declare the piece shippable. This is the
"lint passing ≠ design working" rule from
[`reference/motion-philosophy.md`](reference/motion-philosophy.md) §4.

```bash
# After a render lands, extract frames at hero moments
pandastudio motion.verify-frames \
  --videoPath=/path/to/output.mp4 \
  --timestamps='[0.5,1.5,3.0,5.0,7.5,10.0,12.5,15.0]' \
  --json

# Or from an export-library entry
pandastudio motion.verify-frames \
  --entryId=$ENTRY_ID \
  --timestamps='[0.5,1.5,3.0,5.0,7.5,10.0,12.5,15.0]' \
  --json
```

Returns `{ frames: [{ timestampSeconds, path }...], nextStep: "..." }`.
**Read each `path` as an image** and verify per the checklist in
[`reference/video-authoring.md`](reference/video-authoring.md) §5:

- No cropped faces / text overflow / blank frames
- Caption lands on the right word at each captured moment
- Face mode transitions (Mode A) look smooth, not snapped
- No MG covers the host face area (Mode C) or the screen zone (Mode B)
- Color palette looks right (no accidental flat-white text)
- Backgrounds have the grid + vignette + grain texture

If any frame fails, iterate the motion graphic and re-run
`motion.verify-frames` before `export.start`. **No shipping un-looked-at
work.**

### Assembling multi-scene motion graphics — always use motion.concat

**This is the mandatory last step whenever you render more than one scene.**
Render every scene **in parallel** (each `motion.render-html` call spawns
its own Chromium OS process — no shared lock, no RENDER_BUSY error),
then join them into one final MP4 with `motion.concat` before touching
the project timeline. The concat is a lossless stream copy — no
re-encode, completes in < 1 second.

**Why parallel is correct:** since v1.17 the renderer uses Puppeteer
against standalone Chromium. Three concurrent renders complete in the
time of one (35s → 6.9s on Apple Silicon, a 5× speedup vs. firing them
sequentially). There is no `RENDER_BUSY` — `isRenderBusy()` always
returns false. If you wait-between-fires you're paying the cost of
every render end-to-end for no reason.

```bash
# 1. Fire every scene in parallel. Collect jobIds first, then job.wait
#    each — the second fire doesn't block on the first.
INTRO_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html --durationMs=3000 \
  --outputName=scene-intro --json | jq -r '.data.jobId')

PRODUCT_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/product.html --durationMs=5000 \
  --assets=/Users/me/product1.png,/Users/me/product2.png \
  --outputName=scene-product --json | jq -r '.data.jobId')

CTA_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/cta.html --durationMs=2000 \
  --outputName=scene-cta --json | jq -r '.data.jobId')

# All three Chromium processes are now racing to finish. Collect results.
INTRO_PATH=$(pandastudio job.wait --id="$INTRO_JOB"   --json | jq -r '.data.job.result.outputPath')
PRODUCT_PATH=$(pandastudio job.wait --id="$PRODUCT_JOB" --json | jq -r '.data.job.result.outputPath')
CTA_PATH=$(pandastudio job.wait --id="$CTA_JOB"     --json | jq -r '.data.job.result.outputPath')

# 2. Merge into ONE MP4 — then add that single file to the project.
#    Never add individual scene files as separate project.add-clip calls.
FINAL=$(pandastudio motion.concat \
  --clips="[\"$INTRO_PATH\",\"$PRODUCT_PATH\",\"$CTA_PATH\"]" \
  --outputName=final-promo \
  --json | jq -r '.data.outputPath')

# 3. One clip on the timeline.
pandastudio project.add-clip --id=$ID --media="$FINAL" --json
```

`motion.concat` returns `{ outputPath, clipCount }` directly — no `job.wait`.
All input clips must have the same resolution and frame rate — every
`motion.render-html` and `motion.generate` output from the same session is
automatically compatible (same render pipeline, same defaults).

**Why one clip matters:**
- Seamless playback — no gap or decode stall between segments
- Unified audio — background music or VO doesn't restart between segments
- Cleaner timeline — the user sees one drag-handle, not N unrelated clips
- `project.add-motion-graphic` is for overlays *on top of* existing footage,
  not for main-track clips — use `project.add-clip` with the concat output

### Writing complex multi-overlay HTML for agents

When the user wants product images, text layers, and animated overlays all
compositing together in one scene, write a single HTML file using absolute
positioning and `animation-delay` to sequence elements:

```html
<!DOCTYPE html>
<html>
<head>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { width: 1920px; height: 1080px; background: #0a0a0a; overflow: hidden; }

  /* Layer 1 — background image, fades in immediately */
  .bg { position: absolute; inset: 0; object-fit: cover; opacity: 0;
        animation: fadeIn 0.8s ease forwards; }

  /* Layer 2 — product shot, slides in from right after 0.5s */
  .product { position: absolute; right: 80px; top: 50%; transform: translateY(-50%) translateX(200px);
             width: 700px; opacity: 0;
             animation: slideIn 0.6s ease 0.5s forwards; }

  /* Layer 3 — headline text, appears at 1s */
  .headline { position: absolute; left: 80px; top: 280px;
              font: 900 96px/1 'Inter', sans-serif; color: #fff; opacity: 0;
              animation: fadeUp 0.5s ease 1s forwards; }

  /* Layer 4 — subline, 1.4s */
  .subline { position: absolute; left: 80px; top: 410px;
             font: 400 36px 'Inter', sans-serif; color: rgba(255,255,255,0.7); opacity: 0;
             animation: fadeUp 0.5s ease 1.4s forwards; }

  /* Layer 5 — CTA badge, 2s */
  .cta { position: absolute; left: 80px; bottom: 120px;
         background: #34B27B; color: #fff; padding: 20px 48px;
         font: 700 28px 'Inter', sans-serif; border-radius: 8px; opacity: 0;
         animation: fadeIn 0.4s ease 2s forwards; }

  @keyframes fadeIn   { to { opacity: 1; } }
  @keyframes fadeUp   { from { opacity: 0; transform: translateY(24px); }
                        to   { opacity: 1; transform: translateY(0); } }
  @keyframes slideIn  { to   { opacity: 1; transform: translateY(-50%) translateX(0); } }
</style>
</head>
<body>
  <img class="bg"      src="bg-texture.png" />
  <img class="product" src="product.png" />
  <div class="headline">WritePanda AI</div>
  <div class="subline">Record once. Publish everywhere.</div>
  <div class="cta">Try free →</div>
</body>
</html>
```

Pass local images via `--assets`:

```bash
pandastudio motion.screenshot \
  --html="$(cat /tmp/promo.html)" \
  --assets=/Users/me/product.png,/Users/me/bg-texture.png \
  --atMs=2500 --outputName=promo-check --json
```

Key patterns:
- All layers are `position: absolute` — the `<body>` is the canvas
- Use `animation-delay` to sequence elements without JavaScript
- Fade out at the end? Add a final keyframe: `99% { opacity: 1; } 100% { opacity: 0; }` with `animation-fill-mode: forwards`
- Loop an element infinitely: `animation-iteration-count: infinite`
- For smooth scene transitions (fade-to-black between clips), end the HTML with `body { animation: fadeOut 0.5s ease 4.5s forwards; } @keyframes fadeOut { to { opacity: 0; } }` and match `durationMs` to 5000

### Prepending a title card (or any single clip) before the footage

`add-motion-graphic` places an overlay on top of the canvas — it doesn't insert a discrete clip with its own audio. To make a title card that **plays before the footage** (no video beneath it), insert the motion-graphic MP4 as a clip at index 0.

**Single rendered scene → add directly. Multiple scenes → concat first, then add.**

```bash
# Single title card — add directly (no concat needed for one scene)
TITLE=$(pandastudio motion.generate \
  --templateId=title-card-vox \
  --slots='{"title":"How I Built This","subtitle":"in 24 hours"}' \
  --aspectRatio=16:9 --json | jq -r '.data.jobId')
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

Templates: `classic | modern | minimal | bold | spotlight | boxed | neon | colored`. Captions read words from the project's merged transcript — so you must transcribe first.

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

### Check whether the user has a key

```bash
# Any thumbnail verb will return a clean error if the key isn't set.
# Don't guess — if the UI flow is driven by an agent, check first:
pandastudio export.get --id=$EID --json | jq '.data.entry.thumbnailPath'
```

If `export.generate-thumbnail` returns `"No Replicate API key set. Open Settings → Integrations to add one."`, tell the user to set it rather than looping. Direct them to the Settings pane; PandaStudio does not accept keys via CLI on purpose (so they never end up in shell history or a file an agent could read).

### Generate from the transcript

Requires `transcript.transcribe` + the export already in the library. By default, the local LLM writes a YouTube-art-director prompt from the transcript and feeds it to gpt-image-2 at `medium` quality, 3:2 aspect, WebP output.

```bash
pandastudio export.generate-thumbnail --id=$EID --json
# → { success: true, imagePath: "/.../<entryId>/1745…-abcd.webp", prompt: "…", iterations: [] }
```

Pass an explicit prompt when you want control:

```bash
pandastudio export.generate-thumbnail --id=$EID \
  --prompt="Bold close-up of a mechanical keyboard with neon RGB glow, \"ENDGAME BUILD\" in large white sans-serif overlay, shot with 50mm, cinematic lighting" \
  --quality=medium --json
```

Pass a reference image when you want style transfer or to anchor on the creator's face/logo:

```bash
pandastudio export.generate-thumbnail --id=$EID \
  --referenceImagePath=/Users/.../face.jpg \
  --prompt="Apply the face from the reference image; add a shocked expression" --json
```

### Iterate via edit prompts (chat-style)

Use after the first generation (or after a manual upload) to refine without starting over. Each edit pushes the previous thumbnail onto `entry.thumbnailIterations` so the user can revert.

```bash
pandastudio export.edit-thumbnail --id=$EID \
  --editPrompt="Make the background color more saturated and move the text lower" --json

pandastudio export.edit-thumbnail --id=$EID \
  --editPrompt="Remove the logo in the bottom-right" --json
```

Good iteration prompts are **short and specific**. gpt-image-2 works best with one change at a time; if the user asks for three things, make three separate calls in sequence. Edits run at `low` quality (fast, cheap) — regenerate if the user wants the full-quality version back.

### Manual upload

When the user already has a thumbnail file they want to use:

```bash
pandastudio export.set-thumbnail --id=$EID \
  --sourcePath=/path/to/thumbnail.png --json
```

Accepts PNG / JPEG / WebP. Copies into the managed thumbnails directory (`<userData>/thumbnails/<entryId>/`). Iteration history is cleared because the user is starting fresh. Once set, the user can still call `export.edit-thumbnail` to iterate from there.

### Revert or clear

```bash
# Grab the iteration id from entry.thumbnailIterations[]
pandastudio export.get --id=$EID --json \
  | jq '.data.entry.thumbnailIterations[] | {id, createdAt, prompt}'

pandastudio export.revert-thumbnail --id=$EID --iterationId=$ITER_ID --json

# Wipe the thumbnail entirely
pandastudio export.clear-thumbnail --id=$EID --json
```

### Prompt shape that works

gpt-image-2 rewards photo-language and specificity. The LLM that auto-writes the prompt already follows these rules; when you write your own, mirror them:

- **One focal subject.** Describe what's in the middle of the frame, in photo language: "extreme close-up of …", "over-the-shoulder of …".
- **Specify lighting.** "Shot with 50mm, soft daylight, shallow depth of field" → photorealistic. "Bold graphic poster style, saturated, high contrast" → graphic. Don't mix unless that's the intent.
- **Quote any on-image text.** `bold sans-serif overlay reading "GAME OVER"`. gpt-image-2's text-rendering is sharp — take advantage of it.
- **Lock what shouldn't change** on edits. "Change ONLY the background color, keep the subject, composition, and text identical."
- **Iterate small.** One change per `edit-thumbnail` call beats "make it 5 things at once".

## Export — produce the final MP4

The centerpiece. Headless, runs the same Skia native render-helper the editor's Export button does. **Async; poll `job.wait`.**

```bash
JOB=$(pandastudio export.start --id=$ID --quality=high --json \
  | jq -r '.data.jobId')

# Watch progress (server-side block; returns when done or 5min timeout)
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json | jq '.data.job'
# → status: "succeeded", result: { outputPath, durationMs, width, height, frameRate }
```

Quality presets: `draft` (1280×720), `standard` / `high` (1920×1080), `ultra` (3840×2160). Aspect ratio comes from the project (`set-aspect-ratio`). Output lands in the recordings dir by default; pass `--outputPath=/somewhere/file.mp4` to override.

The export honours **everything** in the project: clips, trims (incl. those from transcript word deletes), speed regions, zooms, captions, FX, lower-thirds with sound, motion graphics, annotations, cleaned audio, wallpaper, padding/shadow/radius/blur. One verb, full pipeline.

**Video overlays (motion graphics) are fully composited in the export** — both opaque MP4 (`motion.generate`, `motion.render-html`) and transparent WebM (`motion.render-html --transparent`) are layered onto the main video via a post-process FFmpeg pass after the Skia render. Alpha channels from VP9/WebM sources are preserved exactly. There is nothing extra you need to call — `export.start` handles it automatically once overlays are on the timeline via `project.add-motion-graphic`.

## Video editing playbook — end-to-end recipe (per destination)

When the user says *"edit this"* / *"polish this"* / *"make this ready for <X>"* / *"YouTube-ready"*, follow this runbook. It turns a raw recording into a polished, destination-appropriate video using the foundational verbs above. Different destinations (YouTube long-form, Shorts/TikTok, LinkedIn, Loom) need different defaults — the table below is the source of truth.

**Philosophy:** good video editing is a series of pattern interrupts that match the platform's viewing context. A YouTube long-form viewer has settled in — cuts every 5–8s and a cinematic LUT feel right. A TikTok viewer is scrolling — you have 3 seconds to hook them and every second after needs a visible change. A LinkedIn viewer is at work — an aggressive soundscape is wrong. A Loom viewer doesn't want any editing at all beyond "cut the fluff". Same tools, very different dials.

### Destination profiles (the source of truth)

Resolve the destination first (see [HARD-GATE](#editorial-decisions--what-to-ask-what-to-assume-what-never-to-ask) step 1). Then apply every default below from the matching row — don't mix.

| Parameter | `youtube-long` | `shorts` (Shorts/TikTok/Reels) | `linkedin` | `loom` (internal/async) |
|---|---|---|---|---|
| Aspect | 16:9 | **9:16** | 16:9 or 1:1 | 16:9 |
| Hook deadline | 10 s | **3 s** | 10 s | — (none) |
| Intro card | 2–4 s | **0–1 s or none** | 2–3 s | none |
| Lower thirds | yes, at first mentions | **no** (too small vertically) | yes | no |
| Zoom cadence | 3–6 / min | **6–12 / min** | 1–2 / min | 0–1 / min |
| Default zoom duration | 1.5 s | **1.0 s** | 2.0 s | 2.0 s |
| Zoom SFX volume | 1.0 (swoosh-fast) | **1.0 (swoosh-fast)** | 0.5 (or `none`) | `none` |
| Filler/silence removal | yes | yes | yes | **yes (aggressive — minSilenceMs 300)** |
| Speed regions (B-roll) | 1.5–2× | **2–3×** or cut entirely | 1.25–1.5× | none |
| LUT preset | by content type @ 0.5–0.8 | **`modernVibrant` @ 1.0** | `naturalEnhanced` @ 0.3 | none |
| Background music vol | 0.15 | **0.30** | 0.0 (none) | 0.0 |
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

# 1. PACING — always run. Loom uses aggressive silence removal.
pandastudio project.read --id=$ID --json
pandastudio transcript.transcribe --id=$ID               # skip if transcribed
pandastudio audio.clean --id=$ID                         # skip if cleaned
pandastudio transcript.remove-fillers --id=$ID
SILENCE_MS=$([ "$PROFILE" = "loom" ] && echo 300 || echo 500)
pandastudio transcript.remove-silences --id=$ID --minSilenceMs=$SILENCE_MS

# 2. EMPHASIS — zooms (skip for `loom`). Cadence comes from the profile table.
#    Read transcript; scan for "click", "here", "this", "select", "look at",
#    "and now", "finally", "boom". For each hit, drop a zoom at that word's
#    startMs. Cadence cap: profile-specific (see table).
#    Default swoosh SFX is auto-attached; for `linkedin` pass --soundVolume=0.5
#    or --soundUrl=none. For `loom`, skip zooms entirely unless a UI
#    moment is absolutely critical.
#
#    CRITICAL: ALWAYS pass --anchorSourceMs when atMs comes from a transcript
#    word. Without it, the zoom drifts off the moment as soon as ANY trim is
#    added — and step 1 always adds trims (remove-fillers, remove-silences).
#    The anchor binds the zoom to the source recording moment so it
#    re-anchors automatically on every trim/speed change.
pandastudio project.add-zoom --id=$ID \
  --atMs=<wordStartMs> --anchorSourceMs=<wordStartMs> \
  --durationMs=<profileDur> --depth=3

# Reveal moments (after "and now" / "finally") in youtube-long / shorts:
pandastudio project.add-zoom --id=$ID \
  --atMs=<ms> --anchorSourceMs=<ms> \
  --durationMs=2500 --depth=5 \
  --soundUrl=bundled:sound/dramatic-whoosh --soundVolume=0.7

# 3. POLISH — skip sections by profile:
#    - `shorts`: no intro card, no lower thirds (tight vertical frame)
#    - `loom`:   skip 3a, 3b, 3c, 3d entirely
#    - `linkedin`: skip 3d (no music)

# 3a. Intro title card (youtube-long: 2-4s, linkedin: 2-3s)
if [ "$PROFILE" = "youtube-long" ] || [ "$PROFILE" = "linkedin" ]; then
  JOB=$(pandastudio motion.generate --templateId=youtube-lower-third \
    --slots='{"channelName":"<name>","handle":"@<handle>"}' --json | jq -r '.data.jobId')
  FILE=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.outputPath')
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

# 3d. Background music (youtube-long: 0.15, shorts: 0.30, NOT linkedin/loom)
if [ "$PROFILE" = "youtube-long" ]; then
  pandastudio project.add-audio --id=$ID \
    --path=bundled:music/tech-review-background --volume=0.15 --fadeIn=2000 --fadeOut=3000
elif [ "$PROFILE" = "shorts" ]; then
  pandastudio project.add-audio --id=$ID \
    --path=bundled:music/tech-review-background --volume=0.30 --fadeIn=500 --fadeOut=500
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

# 5. EXPORT — quality per profile
QUALITY=$([ "$PROFILE" = "loom" ] && echo "standard" || echo "high")
pandastudio export.start --id=$ID --quality=$QUALITY --json | jq -r '.data.jobId' | \
  xargs -I {} pandastudio job.wait --id={}
```

### Anti-patterns (do NOT do these — all profiles)

- **3 effects on the same moment** (zoom + lower-third + motion graphic at same t) — visual noise
- **Multiple LUTs per project** — pick one from the profile table
- **SFX on every cut** — cap at 1 meaningful SFX per 15–30s (except `shorts`, where 1 per 5–10s is fine)
- **Speed regions over voice** — always for setup / B-roll / scrolling only
- **Logo intro >5s** (any profile) — retention cliff
- **Asking the user which filler words to remove** — always-safe op, just do it
- **Applying `youtube-long` defaults to a `shorts` project** — wrong aspect, music too quiet, captions too subtle, pacing too slow
- **Motion graphics in `loom`** — kills the "this is a quick update" vibe

### Pattern: "edit this for <destination>" one-shot

After HARD-GATE resolves the profile, announce the plan in one sentence and execute without further questions:

> I'll edit this as a **YouTube long-form** video — pacing first (trim fillers + silences), then zooms with swoosh SFX at UI moments, intro title card + lower third, `modernVibrant` LUT, background music at 15%, and `panda-pop` captions. Should take ~3 minutes.

Or for Shorts:

> I'll edit this as a **Short** — aggressive pacing (hook in 3s, 6–12 zooms/min), `modernVibrant` LUT at full intensity, `panda-neon` captions positioned higher, and music at 30%. No intro card or lower thirds — they don't fit the vertical frame. ~2 minutes.

Don't ask the user to micro-manage step choices. The profile table is the answer.

## What this skill is NOT for

- **Cloud video APIs** (HeyGen, Runway, Sora). PandaStudio is local-only.
- **Direct edits to `.pandastudio` project JSON.** The format is owned by the editor and changes between versions. Use `project.read` / `project.save` and treat the JSON as opaque between reads.
- **Cloud video APIs** — PandaStudio is local-only; `export.start` renders on the user's machine.

## Reference files

- [`reference/commands.md`](reference/commands.md) — every verb.noun with arg schema and a one-line example.
- [`reference/examples.md`](reference/examples.md) — multi-step recipes: "make a 30 s intro card", "browse exports and pick the best title", "render a Shorts (9:16) lower-third".
- [`reference/templates.md`](reference/templates.md) — what each motion-graphic template looks like, with the slots it accepts and which aspect ratios it supports.
- [`reference/motion-philosophy.md`](reference/motion-philosophy.md) — **the aesthetic contract.** 11 Laws, visual vocabulary, easing dictionary, canonical shell, pre-flight checklist. Load this BEFORE authoring any motion graphic. This is what raises output from "template-filled" to "HyperFrames-quality".
- [`reference/video-authoring.md`](reference/video-authoring.md) — **3-mode delivery playbook.** Mode A (9:16 camera-only), Mode B (9:16 screen-rec + PiP face — PandaStudio's unique mode), Mode C (16:9 YouTube side-overlay). Face choreography, caption safe zones, audio-sync protocol, frame verification. Load this for any shorts/YouTube authoring task.
