---
name: pandastudio
description: Drive PandaStudio — a desktop video editor for YouTube creators — from the command line. Use when the user wants to list / read / create / save PandaStudio projects, generate motion-graphic title cards, lower thirds, or FX intros from templates, browse the bundled sound + FX libraries, query the export library, run inference through PandaStudio's local LLM, or open the editor / exports / home windows. Talks to a localhost-only HTTP API the user must enable in Settings → Local automation. Do NOT use this skill for unrelated video tools, cloud video APIs, or for editing arbitrary files in a PandaStudio project (the project file format is owned by the editor; the CLI is the safe interface).
---

<!-- version: 2.11.0 -->

# PandaStudio

> **Version check — do this first.** This skill requires `@writepanda/cli` ≥ 1.15.0.
> Run `pandastudio --version` before starting any task. If it reports < 1.15.0,
> tell the user to update their MCP config to use `npx @writepanda/mcp@latest`
> (note the `@latest` tag) and restart Claude Desktop. Commands like
> `asset.list-music`, `asset.list-luts`, and `project.set-clip-lut` do not exist
> in older versions.

PandaStudio is a desktop video editor. You drive it through `pandastudio`, a CLI that talks to a localhost HTTP server living inside the running app. Every command is shaped `verb.noun` and accepts JSON-shaped flags.

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

## Editorial decisions — what to ask, what to assume, what NEVER to ask

Video editing is a creative task with hundreds of small decisions. Asking the user about all of them kills the magic — they came to you because they wanted to type "edit this" and see something happen. Asking about none of them produces wrong-shape output. The rule:

**Ask only when the answer is genuinely user-specific AND can't be inferred AND is hard to reverse.** Default everything else, narrate what you did, and iterate via preview.

### MUST ASK (3 things, only when context is missing)

<HARD-GATE>
Before issuing `project.new`, `project.add-*`, `motion.generate`, or `motion.render-html` calls that depend on user-specific information, you MUST have:

**1. Aspect ratio.** Try to infer first:
   - Source clip is portrait (height > width)? → 9:16 (Shorts/Reels/TikTok)
   - Source clip is landscape? → 16:9 (YouTube)
   - Source clip is square? → 1:1
   - User mentioned "Shorts" / "TikTok" / "Reels" / "vertical" / "phone"? → 9:16
   - User mentioned "YouTube" / "long-form" / "horizontal" / "presentation"? → 16:9
   - **Mixed orientations OR no clips yet OR truly ambiguous?** → ASK ONE QUESTION:
     > "Which aspect — 16:9 (YouTube), 9:16 (Shorts/Reels), or 1:1?"

**2. Lower-third content.** If the user said "add a name plate" / "introduce me" / "add a lower-third" but didn't give the actual text, ASK both fields in one message:
   > "What's the name and the subtitle (e.g. role / company)?"

   Don't fabricate a name. Don't invent a job title.

**3. Brand / style direction.** If the user named a style ("MrBeast" / "MKBHD" / "Vox" / "Kurzgesagt" / "Veritasium"), apply the matching `motion.themes` theme. If they gave none AND there are multiple source clips that suggest a brand context, ASK ONE QUESTION:
   > "Any brand colors, fonts, or visual references — or default look?"

If all three are clear (or already specified), proceed without asking.
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
# regionType: zoom | trim | speed | annotation | fx | lower-third | overlay
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

When the brief doesn't fit a template — or you want a one-off animation — author the HTML/CSS/JS yourself and let `motion.render-html` render it through the Chromium → capturePage → FFmpeg pipeline.

The contract is loose:

- Animations should auto-start on `DOMContentLoaded` (CSS keyframes, GSAP, Lottie, anything Chromium renders). No `body.go` ceremony — capture begins after first paint.
- The window honours the dimensions you ask for (`aspectRatio` 16:9 / 9:16 / 1:1 OR explicit `width`+`height`).
- Capture runs for `durationMs` (default 2500, range 100–60000) at `frameRate` FPS (default 30, range 1–60).
- `requestAnimationFrame` fires correctly at the configured frame rate — `backgroundThrottling` is disabled and `setFrameRate(fps)` is applied to the offscreen window. JS-driven animation libraries (GSAP, Anime.js, etc.), canvas animations, and WebGL all tick at the right cadence.
- External resources load freely: Google Fonts, CDN scripts, `https://` images. Pass local files via `--assets` instead of embedding base64.

```bash
cat > /tmp/intro.html <<'HTML'
<!doctype html>
<html><head><style>
  html,body{margin:0;background:#0f172a;color:#e2e8f0;font:700 96px/1 system-ui;
    height:100%;display:grid;place-items:center;overflow:hidden}
  h1{opacity:0;transform:translateY(40px);
    animation:rise .9s cubic-bezier(.2,.7,.2,1) .3s forwards}
  @keyframes rise{to{opacity:1;transform:none}}
</style></head><body><h1>Q4 Wrap</h1></body></html>
HTML

JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html \
  --aspectRatio=16:9 \
  --durationMs=2500 \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id="$JOB" --json | jq '.data.job.result.outputPath'
```

You can also pass `--html='<!doctype html>...'` inline — the renderer stages it as a temp file under `os.tmpdir()` so file:// loads work normally. Use `htmlPath` when the markup is large or you want to debug it in a browser first.

Result is an MP4 in the recordings dir. Drop it onto a project with `project.add-motion-graphic` exactly like a `motion.generate` output.

### Transparent overlays — `--transparent`

Add `--transparent` to get a **WebM with a VP9/alpha channel** instead of an opaque MP4. Use this for anything that needs to composite on top of existing video: lower thirds, watermarks, bug logos, corner clocks, branded name plates.

**HTML contract for transparent renders:**

```css
/* Make the page background transparent — the BrowserWindow is already
   compositing against ARGB, but you must not override it in CSS. */
html, body {
  background: transparent;
}
```

Everything else is the same as a normal `motion.render-html`. Semi-transparent elements (e.g. `rgba(0,0,0,0.8)` card backgrounds) render correctly — the alpha value is preserved through capturePage() → PNG → VP9/yuva420p.

**Constraint:** `--audioPath` is ignored in transparent mode. VP9+alpha is a video-only stream; mux audio in at composite time.

```bash
cat > /tmp/lower-third.html <<'HTML'
<!doctype html>
<html><head><style>
  html,body { margin:0; width:100%; height:100%; background:transparent; overflow:hidden;
    font-family:"Inter",system-ui,sans-serif; }
  .card {
    position:absolute; left:5vw; bottom:9vh;
    background:rgba(0,0,0,0.85); border-radius:10px; padding:14px 22px;
    color:#fff; opacity:0; transform:translateY(20px);
    animation:slide-in .5s cubic-bezier(.16,1,.3,1) .2s forwards;
  }
  .name  { font-size:26px; font-weight:700; }
  .title { font-size:15px; opacity:.65; margin-top:3px; }
  @keyframes slide-in { to { opacity:1; transform:none; } }
</style></head><body>
  <div class="card">
    <div class="name">Alex Rivera</div>
    <div class="title">Senior Product Designer</div>
  </div>
</body></html>
HTML

JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/lower-third.html \
  --aspectRatio=16:9 \
  --durationMs=3500 \
  --transparent \
  --json | jq -r '.data.jobId')

# outputPath will end in .webm, not .mp4
pandastudio job.wait --id="$JOB" --json | jq '.data.job.result.outputPath'
```

The resulting `.webm` can be added to a project with `project.add-motion-graphic` — the same call as any other motion graphic.

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

# STEP 2: Render scenes SEQUENTIALLY — one at a time.
# The render pipeline uses a single Chromium BrowserWindow.
# Parallel renders will FAIL with RENDER_BUSY — the server rejects
# concurrent calls loudly. Always job.wait before the next render.
JOB1=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene1.html --durationMs=4000 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB1" --json   # WAIT — do not fire scene2 yet
SCENE1=$(pandastudio job.wait --id="$JOB1" --json | jq -r '.data.job.result.outputPath')

JOB2=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene2.html --durationMs=3000 --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB2" --json
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

```bash
# Add music (plays from the start, volume 60%)
pandastudio project.add-audio --id=$ID \
  --audioPath=/path/to/music.mp3 --volume=0.6 --json
# → returns { overlayId: "audio-1" }

# Add a VO track starting at 2s
pandastudio project.add-audio --id=$ID \
  --audioPath=/path/to/vo.wav --startMs=2000 --volume=1.0 --json

# Remove it later
pandastudio project.remove-audio --id=$ID --overlayId=audio-1 --json
```

Audio overlays are exported automatically — you don't need to do anything
extra in `export.start`.

### Bundled background music — browse and add in one step

PandaStudio ships with royalty-free background music tracks you can drop into
any project without sourcing external files.

```bash
# 1. List all bundled tracks (id, title, category, mood, durationMs, absolutePath)
pandastudio asset.list-music --json | jq '.data.tracks'

# Example output:
# [
#   { "id": "chill-vlog-ambience", "title": "Chill Vlog Ambience",
#     "category": "lofi", "mood": "calm", "durationMs": 30000,
#     "absolutePath": "/Applications/PandaStudio.app/.../music/chill-vlog-ambience.wav" },
#   { "id": "tech-review-background", "title": "Tech Review Background",
#     "category": "corporate", "mood": "energetic", "durationMs": 30000,
#     "absolutePath": "/Applications/PandaStudio.app/.../music/tech-review-background.wav" }
# ]

# 2. Pick a track and add it to the project (use absolutePath directly)
MUSIC=$(pandastudio asset.list-music --json \
  | jq -r '.data.tracks[] | select(.mood == "calm") | .absolutePath' | head -1)

pandastudio project.add-audio --id=$ID \
  --audioPath="$MUSIC" --volume=0.5 --json
```

**Mood → track selection heuristic** (use unless user specifies):
- Tutorial / explainer → `energetic` (corporate)
- Vlog / day-in-life → `calm` (lofi)
- Product showcase → pick by category (`corporate` for tech, `lofi` for lifestyle)

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

### Assembling multi-scene motion graphics — always use motion.concat

**This is the mandatory last step whenever you render more than one scene.**
Render each scene independently (sequential renders — the Chromium pipeline is
single-flight), then join them into one final MP4 with `motion.concat` before
touching the project timeline. The concat is a lossless stream copy — no
re-encode, completes in < 1 second.

```bash
# 1. Render each scene independently (sequential — job.wait between each).
#    Capture the outputPath from each job.wait result.
INTRO_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html --durationMs=3000 \
  --outputName=scene-intro --json | jq -r '.data.jobId')
INTRO_PATH=$(pandastudio job.wait --id="$INTRO_JOB" --json \
  | jq -r '.data.job.result.outputPath')

PRODUCT_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/product.html --durationMs=5000 \
  --assets=/Users/me/product1.png,/Users/me/product2.png \
  --outputName=scene-product --json | jq -r '.data.jobId')
PRODUCT_PATH=$(pandastudio job.wait --id="$PRODUCT_JOB" --json \
  | jq -r '.data.job.result.outputPath')

CTA_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/cta.html --durationMs=2000 \
  --outputName=scene-cta --json | jq -r '.data.jobId')
CTA_PATH=$(pandastudio job.wait --id="$CTA_JOB" --json \
  | jq -r '.data.job.result.outputPath')

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
# regionType: zoom | trim | speed | annotation | fx | lower-third | overlay

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

## YouTube editing playbook — end-to-end recipe

When the user says *"edit this for YouTube"* / *"make this YouTube-ready"* / *"polish this video"*, follow this runbook. It turns a raw recording into a high-retention polished video using the foundational verbs above. Every step is either always-safe or uses conservative defaults — no content judgment beyond what the transcript tells you.

**Philosophy:** a high-retention YouTube video is a series of pattern interrupts. Our job is to add one every 7 seconds or so — cuts, zooms, sounds, captions, LUT — so the viewer never gets bored enough to click away. PandaStudio's features map to four levers: **pacing**, **emphasis**, **polish**, **accessibility**.

### The runbook (ordered — do not rearrange)

```bash
# 0. Always start with the current project (or ask which one)
ID=$(pandastudio project.current --json | jq -r '.data.project.id // empty')
[ -z "$ID" ] && ID=$(pandastudio project.list --json | jq -r '.data.projects[0].id')

# 1. PACING — cut dead weight (10-40% length reduction typical)
pandastudio project.read --id=$ID --json           # inspect clipStates
pandastudio transcript.transcribe --id=$ID          # skip if already transcribed
pandastudio audio.clean --id=$ID                    # skip if already cleaned
pandastudio transcript.remove-fillers --id=$ID
pandastudio transcript.remove-silences --id=$ID --minSilenceMs=500

# 2. EMPHASIS — zoom at every UI/reveal moment (default swoosh SFX attached)
#    Read transcript, scan for phrases: "click", "here", "this", "select",
#    "look at", "and now", "finally", "boom". For each hit, drop a zoom
#    at that word's startMs. 3-6 zooms per minute for tutorials; 1-2 for
#    vlogs. Never stack within 2s of each other.
pandastudio project.add-zoom --id=$ID --atMs=<wordStartMs> --durationMs=1500 --depth=3
# Big reveal moments (after "and now" / "finally") → depth 5, dramatic whoosh
pandastudio project.add-zoom --id=$ID --atMs=<ms> --durationMs=2500 --depth=5 \
  --soundUrl=bundled:sound/dramatic-whoosh --soundVolume=0.7

# 3. POLISH — intro hook + lower thirds + color + music
# 3a. Intro title card (2-4s, never >10s)
JOB=$(pandastudio motion.generate --templateId=youtube-lower-third \
  --slots='{"channelName":"<name>","handle":"@<handle>"}' --json | jq -r '.data.jobId')
FILE=$(pandastudio job.wait --id=$JOB --json | jq -r '.data.outputPath')
pandastudio project.add-motion-graphic --id=$ID --file=$FILE --durationMs=3000 --atMs=0

# 3b. Lower third at first mention of a person or product
pandastudio project.add-lower-third --id=$ID --atMs=<ms> \
  --content="<name>" --subtitle="<role>" --designType=slash-reveal

# 3c. LUT (one preset, applied to every clip). Pick by content type:
#     tech tutorial / SaaS demo   → modernVibrant @ 0.7
#     cinematic vlog              → cinematicTealOrange @ 0.9
#     educational / neutral       → naturalEnhanced @ 0.5
#     moody storytelling          → moodyDark @ 0.7
#     travel / lifestyle          → warmSunset @ 0.7
#     Read the project's clips and set-clip-lut on each.
#     (Use project_read → project_save round-trip if the CLI verb lacks a direct setter.)

# 3d. Background music at 15%
pandastudio project.add-audio --id=$ID \
  --path=bundled:music/tech-review-background --volume=0.15 --fadeIn=2000 --fadeOut=3000

# 4. ACCESSIBILITY — animated captions (85% of viewers start muted)
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-template --id=$ID --templateId=panda-pop
#   panda-pop  = tutorials (bright, per-word highlight)
#   panda-clean = professional / corporate
#   panda-neon = high-impact shorts (tech / gaming)

# 5. EXPORT — one call, everything composited (zooms+SFX, LUT, captions, overlays)
pandastudio export.start --id=$ID --quality=high --json | jq -r '.data.jobId' | \
  xargs -I {} pandastudio job.wait --id={}
```

### Defaults this runbook encodes

| Lever | Feature | Default |
|---|---|---|
| Pacing | Filler + silence removal | Always run |
| Pacing | Speed region | Only over setup/B-roll (not voice), 1.5–2× |
| Emphasis | Zoom SFX | `swoosh-fast` (already attached by add-zoom) |
| Emphasis | Zoom cadence | 3–6/min tutorials, 1–2/min vlogs |
| Emphasis | Zoom depth | 3 for clicks, 4–5 for reveals |
| Polish | Intro duration | 2–4s (never >10s — retention cliff) |
| Polish | Lower third SFX | Default mouse-click at 0.8 |
| Polish | Music volume | 0.15 |
| Accessibility | Caption template | `panda-pop` (tutorials), `panda-clean` (pro) |

### Anti-patterns (do NOT do these)

- **3 effects on the same moment** (zoom + lower-third + motion graphic at the same t) — visual noise
- **Multiple LUTs per project** — pick one
- **SFX on every cut** — 1 meaningful SFX per 15–30s is the cap
- **Speed regions over voice** — only for setup / B-roll / scrolling
- **Logo intro >10s** — retention graph always shows a cliff there
- **Asking the user which filler words to remove** — always-safe op, just do it

### Pattern: "edit this for YouTube" one-shot

When the user gives a prompt like *"make this video YouTube-ready"*, run the full runbook in order, reporting progress after each phase:

> I'll edit this for YouTube — pacing first (trim fillers + silences), then emphasis (zooms + sounds at each click), then polish (intro card, lower thirds, color grade, music), then captions. Should take ~3 minutes.

Don't ask the user to micro-manage step choices — the defaults above are what every good YouTube editor does by hand. Do ask once, up front, for the three hard-gated editorial decisions (title card text, lower-third names, brand style) if they weren't supplied in the initial prompt.

## What this skill is NOT for

- **Cloud video APIs** (HeyGen, Runway, Sora). PandaStudio is local-only.
- **Direct edits to `.pandastudio` project JSON.** The format is owned by the editor and changes between versions. Use `project.read` / `project.save` and treat the JSON as opaque between reads.
- **Cloud video APIs** — PandaStudio is local-only; `export.start` renders on the user's machine.

## Reference files

- [`reference/commands.md`](reference/commands.md) — every verb.noun with arg schema and a one-line example.
- [`reference/examples.md`](reference/examples.md) — multi-step recipes: "make a 30 s intro card", "browse exports and pick the best title", "render a Shorts (9:16) lower-third".
- [`reference/templates.md`](reference/templates.md) — what each motion-graphic template looks like, with the slots it accepts and which aspect ratios it supports.
