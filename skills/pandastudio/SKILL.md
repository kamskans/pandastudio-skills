---
name: pandastudio
description: Drive PandaStudio — a desktop video editor for YouTube creators — from the command line. Use when the user wants to list / read / create / save PandaStudio projects, generate motion-graphic title cards, lower thirds, or FX intros from templates, browse the bundled sound + FX libraries, query the export library, run inference through PandaStudio's local LLM, or open the editor / exports / home windows. Talks to a localhost-only HTTP API the user must enable in Settings → Local automation. Do NOT use this skill for unrelated video tools, cloud video APIs, or for editing arbitrary files in a PandaStudio project (the project file format is owned by the editor; the CLI is the safe interface).
---

<!-- version: 1.5.0 -->

# PandaStudio

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

| Operation | Default behaviour |
|---|---|
| `transcript.transcribe` | Run on every clip without an existing transcript. |
| `transcript.remove-fillers` | Auto-remove "um/uh/like/you know/i mean" + immediately-repeated words. Trim regions; fully reversible. |
| `audio.clean` | Denoise every clip via DeepFilter. Writes a sibling `.cleaned.wav`; original audio untouched. |
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

### Edit primitives (no schema knowledge needed)

```bash
pandastudio project.add-clip --id=<uuid> --media=/path/new-clip.mp4
pandastudio project.split-clip --id=<uuid> --clipId=clip-1 --atSourceMs=4000
pandastudio project.remove-clip --id=<uuid> --clipId=clip-2

pandastudio project.add-motion-graphic \
  --id=<uuid> --file=/path/intro.mp4 --durationMs=2500 --atMs=0

pandastudio project.add-fx \
  --id=<uuid> --fxId=film-burn --atMs=5000

pandastudio project.add-lower-third \
  --id=<uuid> --content="Kamal" --subtitle="Founder" --atMs=2000
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

## Custom motion graphics — beyond the templates

The 19 bundled `motion.list` templates exist to make small models (like the in-app Gemma 4 E2B) usable for motion-graphic generation: pick a `templateId`, fill some slots, ship. **You're not a small model.** When the brief doesn't fit a template — or you want a one-off animation that's nothing like the bundled set — author the HTML/CSS/JS yourself and let `motion.render-html` render it through the same Chromium → capturePage → FFmpeg pipeline.

The contract is loose:

- Animations should auto-start on `DOMContentLoaded` (CSS keyframes, GSAP, Lottie, anything Chromium renders). No `body.go` ceremony — capture begins after first paint.
- The window honours the dimensions you ask for (`aspectRatio` 16:9 / 9:16 / 1:1 OR explicit `width`+`height`).
- Capture runs for `durationMs` (default 2500, range 100–60000) at `frameRate` FPS (default 30, range 1–60).
- The HTML loads with `webSecurity: true` and `sandbox: true`. Inline external CDNs (Google Fonts, GSAP, Lottie via unpkg) work; local file fetches outside the temp HTML don't.

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

## Transcript-based editing — PandaStudio's signature feature

The reason humans pick PandaStudio over Premiere is that you edit by **deleting words from the transcript**, not by scrubbing the timeline. The CLI exposes the same model.

### The full edit loop

```bash
# 1. Transcribe every clip (Parakeet TDT, async, runs locally)
JOB=$(pandastudio transcript.transcribe --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=300000 --json

# 2. Pull the merged transcript — every word with edited-time start/end
pandastudio transcript.get --id=$ID --json | jq '.data.words[0:20]'

# 3a. AUTO: drop every "um" / "uh" / "you know" + immediate repeats
pandastudio transcript.remove-fillers --id=$ID --json

# 3b. SURGICAL: delete specific words by ID
pandastudio transcript.delete-words --id=$ID --wordIds='["clip-1:w-42","clip-1:w-43"]' --json

# 3c. PHRASE search → bulk delete
WORDS=$(pandastudio transcript.search --id=$ID --query="this is a test" --json \
  | jq -c '[.data.matches[].wordIds | .[]]')
pandastudio transcript.delete-words --id=$ID --wordIds="$WORDS" --json
```

Every deletion translates internally into a **trim region** the export pipeline skips. It's identical to clicking the word in the editor's transcript pane and hitting delete.

### Audio cleanup (DeepFilter)

```bash
JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json
# → each clip gets a sibling .cleaned.wav file; export auto-uses it
```

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

## What this skill is NOT for

- **Cloud video APIs** (HeyGen, Runway, Sora). PandaStudio is local-only.
- **Direct edits to `.pandastudio` project JSON.** The format is owned by the editor and changes between versions. Use `project.read` / `project.save` and treat the JSON as opaque between reads.
- **Triggering exports today.** There is no `export.start` verb yet — the export pipeline is renderer-driven. You can browse and patch existing exports (`export.list`, `export.get`, `export.update`, `export.delete`) but to create one, open the editor (`window.editor`) and let the user click Export.

## Reference files

- [`reference/commands.md`](reference/commands.md) — every verb.noun with arg schema and a one-line example.
- [`reference/examples.md`](reference/examples.md) — multi-step recipes: "make a 30 s intro card", "browse exports and pick the best title", "render a Shorts (9:16) lower-third".
- [`reference/templates.md`](reference/templates.md) — what each motion-graphic template looks like, with the slots it accepts and which aspect ratios it supports.
