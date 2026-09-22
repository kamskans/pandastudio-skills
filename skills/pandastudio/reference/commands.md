# Command reference

The canonical list lives at `pandastudio commands --json` (run it — the registry grows). This file is a snapshot for offline reading.

Every command takes args as `--key=value` flags. Object/array values must be JSON: `--slots='{"a":1}'`.

## system.*

| Command | Args | Purpose |
|---|---|---|
| `system.status` | — | App version + license block. Always run this first. |
| `system.list` | — | List every available `verb.noun` with summaries. Same as `pandastudio commands`. |
| `skill.read` | `section` (id or title words), `file` (`shorts`, `reference/shorts.md`, `SKILL.md`) | This skill, in pieces, for agents that can't install skills (MCP-only clients: `skill_read`). No args: version, opening, section outline with sizes, reference docs. Works without a license. |
| `system.ping` | — | `{ pong: true }`. Heartbeat. |
| `system.echo` | `payload` (any) | Echo for transport debugging. |

## project.*

Project files are JSON on disk under the user's recordings dir. All paths are validated to live inside that dir — out-of-tree absolute paths are rejected. Most commands accept either `--id=<uuid>` (preferred — stable across renames) or `--path=...`.

### Lookup + lifecycle

| Command | Args | Purpose |
|---|---|---|
| `project.list` | `allWorkspaces` (bool, default false), `sortBy` (modifiedAt\|createdAt\|name), `order`, `limit`, `query` | Projects in the active workspace (or all, with `allWorkspaces=true`), newest-first: `{ id, revision, path, name, clipCount, modifiedAt, createdAt, sizeBytes, workspaceId }`. Top-level response also returns `currentWorkspaceId`. |
| `project.locate` | `id` (req) | Look up a project's owning workspace WITHOUT reading the body. Returns `{ id, filePath, workspaceId, workspaceName, isInActiveWorkspace }`. Pre-flight check before any edit/export/publish on a bare project id — prevents publishing to the wrong client's YouTube channel. See SKILL.md "Workspaces" §"When given a project id with no other context". |
| `project.read` | `id` \| `path` | Full JSON. **Pass back `project.revision` as `expectedRevision` on save.** |
| `project.show` | `id` \| `path` (or no args) | Resolve to path + summary. With no args, returns recordingsDir + userDataDir. |
| `project.new` | `name` (req), `withMedia` (string\[\] or comma-list of paths) | Create v3 project; if `withMedia` set, FFmpeg-probes each video and adds it as a clip. |
| `project.save` | `id` \| `path`, `project` (req), `expectedRevision` (optional) | Overwrite. Returns `{ ok:false, details:{ code:"revision_conflict" } }` if expectedRevision is stale. |
| `project.delete` | `id` \| `path` | Permanent delete (no trash). |
| `project.open` | `id` \| `path` (optional) | Open editor focused on this project. |

### Edit primitives (no schema knowledge required)

All accept `id` or `path`, plus optional `expectedRevision` for conflict-safe writes. Returns the updated `project` (with bumped `revision`).

| Command | Args | Purpose |
|---|---|---|
| `project.add-clip` | `media` (path), `atIndex` (optional, 0 = prepend) | Insert a video clip on the main track. Probes duration. Default: append at end. Later clips' trims/regions/anchors move with them. Mid-recording insert: `split-clip` first, then `atIndex=<rightClipIndex>`. |
| `project.remove-clip` | `clipId` | Drop a clip by id. |
| `project.batch` | `commands` (array of `{command, args}`), `stopOnError` | Run up to 200 project/transcript/caption/timeline/audio commands on one project in one call; per-step `createdIds`/`removedIds`. Not atomic. |
| `project.set-clip-webcam` | `clipId` \| `clipIndex`, `webcam` (path, `''` removes), `audio` (auto\|camera\|keep) | Attach, replace or remove a clip's camera video (makes it a Screen + camera clip that follows the project camera layout). For camera footage made elsewhere, e.g. a Seedance AI presenter. The camera layer plays muted; `audio=auto` muxes the camera's sound into a copy of a silent main video (then re-transcribe). Warns when lengths differ. Not for podcast clips. |
| `project.move-clip` | `clipId`, `toIndex` | Reorder a clip on the main track. Every region moves with the clip it sits on (a trim spanning two clips is cut at the boundary, each piece follows its clip), so zooms/trims/etc. stay attached to the same content. Never drops a region. |
| `project.split-clip` | `clipId`, `atSourceMs` (clip's own source time; `timeline.edited-to-source` → `clipSourceMs`) | Split a clip in two without changing the output. Left keeps the id and ends at the split; right (`rightClipId`, `rightClipIndex`) covers the full media with a head trim over the left's part. Anchors/trims after the split move with the right half; transcript words divide at the split. |
| `project.add-motion-graphic` | `file` or `fromJob`, `durationMs`, `atMs` (optional, defaults to end-of-timeline), `muted` (default true), `volume` (0–1), `loop` | Drop an MP4/WebM (typically from `motion.generate`) as a media-overlay region; a `fromJob` render keeps its source so it stays editable. Animated GIF / WebP / APNG files are converted to a looping transparent WebM. An overlay VIDEO's own audio is muted by default; pass `muted=false` to hear it in preview and export. Timeline mute regions still silence it. |
| `project.set-overlay-chroma-key` | `regionId` (req), `color` (`auto` default \| `#RRGGBB`), `similarity`, `smoothness`, `spill` (0–1), `enabled` (false removes) | Green screen on an image/video overlay: removes a flat-colour backdrop. `auto` detects the colour from the frame edges. Check with `render-frame`. |
| `project.set-overlay-crop` | `regionId` (req), `x`, `y`, `width`, `height` (0–1 of the overlay SOURCE; omit or pass 0,0,1,1 to clear) | Crop an image/video/graphic overlay's source pixels: cut black bars off B-roll, take the centre of a wide clip for a vertical short. |
| `project.set-overlay-backdrop-blur` | `regionId` (req), `strength`, `tint` | Frosted-glass blur of whatever is under an overlay, shaped by the overlay's alpha (use a transparent overlay; opaque ones show nothing). |
| `project.set-region-sound` | `regionId` (req), `regionType` (req: `zoom` \| `motionGraphic` \| `fx`), `soundUrl` (req; `none` removes), `soundVolume` | Swap, add or mute the sound effect on a placed zoom, motion graphic or FX. |
| `project.update-motion-graphic` | `overlayId` (req), `slots` (template), `background` (`solid`\|`transparent`\|`glass`), `html` (HTML graphics) | **Async.** Re-render a placed generated graphic with edits, in place (timing, position, sound kept). Needs `generatedFrom` on the overlay. |
| `project.add-emoji` | `emoji` (req: 🔥, `1f525` or `fire`), `atMs`, `durationMs` (default 3000), `x`/`y` (center %, default 50), `size` (height %, default 25), `soundUrl` | **Async.** Place a looping Google Noto animated emoji overlay. Discover with `asset.list-emoji`. |
| `project.update-region` | `regionType`, `regionId`, then only the fields to change | Patch a placed region in place. For `regionType=overlay` this includes `x`/`y`/`width`/`height`, `layer`, `muted` (the overlay video's own audio) and `volume` (0–1). |
| `project.duplicate-region` | `regionType` (zoom/speed/annotation/fx/overlay/clip-transform/background-effect/spotlight/audio-overlay), `regionId`, `atMs?` | Copy a region with all its settings and a new id, right after the original or at `atMs` (EDITED ms; SOURCE ms for speed). Link groups copy as a new group. Zooms/speed shift to the next free gap (`shiftedMs`) or fail when none fits. |
| `project.center-camera-on-face` | `clipId?` (default: every clip with a camera), `stepMs?` (default 500), `clear?` | **Async.** Keep the presenter's face centred in the camera card: detects the face through the camera video and stores a smoothed face track that preview, render-frame and export follow. `clear=true` removes it. |
| `project.set-clip-color` | `clipId` (req), `target` (`screen` default \| `camera`), `preset` (`flat-footage` \| `none`), `brightness`, `contrast`, `saturation`, `warmth` (each −1..1, 0 = unchanged), `reset` | Color-correct a clip BEFORE any LUT look. `flat-footage` fixes flat, washed-out camera footage (log / flat picture profiles). Merges with the current correction; `reset=true` or `preset=none` clears. Same result in preview and export. |
| `project.add-lower-third` | `name` (req), `title`, `atMs` (req), `templateId` (default `lt-vox-marker`), `slots`, `anchorSourceMs` | **Async.** Renders an `lt-*` nameplate template and places it as a transparent overlay in one call. Returns `{ jobId }`; `job.wait` resolves once placed. |
| `project.add-transition` | `transitionId` (bundled) OR `file`, `atMs` (req, the cut time), `durationMs` (default 1000) | Place a scene-change transition centered on a cut. Discover ids via `asset.list-transitions`. |
| `project.add-fx` | `fxId` (bundled) OR `src` (custom URL/path), `atMs`, `durationMs`, `speed` (0.25–4, default 1) | Drop an FX overlay. Bundled FX inherits blend mode + opacity from the manifest. `speed` sets the loop playback rate. |
| `project.add-zoom` | `atMs`, `durationMs`, `depth` (1-6, default **2** = 1.5× soft modern), `focusX/Y` (0-1), `soundUrl` (default `bundled:sound/swoosh-fast`) | Highlight a UI moment with a zoom region. Ships with a default swoosh SFX; pass `soundUrl=none` to silence. |
| `project.add-clip-transform-region` | `startMs`, `endMs`, `preset` (`custom` with `screenX`/`screenY`/`screenScale` and/or `webcamCx`/`webcamCy`/`webcamScale` moves the video or camera for just that window, any recording / `cam-bottom-half` / `cam-top-half` / `cam-right-portrait` / `cam-left-portrait` / `cam-bottom-right-quarter` / `cam-bottom-left-quarter` / 16:9 splits `cam-{left,right}-{50,55}` / 9:16 Shorts splits `cam-{top,bottom}-{50,55}`), `transitionMs` (default 320), `cameraFit` (`layout-guest-full` on screen + camera: `fill` default, no card chrome \| `centered` over `backgroundColor`, default `#F4F0E8`) | Time-bounded layout transform on the main video clip — shrink camera to make room for a motion graphic during an explainer beat. **Camera-only / user-uploaded recordings only — never on screen recordings.** See video-authoring §5b. |
| `project.add-trim` | `startMs`, `endMs` | Cut a section the exporter skips. |
| `project.apply-edit-plan` | `plan` (JSON array string), `expectedRevision?` | BATCH: apply many timeline ops in ONE call, atomically (all-or-nothing). Ops: `add-trim`, `add-zoom`, `add-speed`. ALWAYS prefer this over sequential add-* calls when executing a beat map. |
| `project.render-sheet` | `fromMs?`, `toMs?`, `count?` (default 12, max 24), `cols?` (default 4), `outPath?` | ONE call → tiled contact-sheet PNG of N verified preview frames across a range (row-major; cell k = `frames[k]`). THE pacing/motion verification tool — replaces N render-frame calls. |
| `project.add-speed` | `startMs`, `endMs` (source ms), `speed` (0.25 to 100, 0.01 steps) | Speed up or slow down a span. 8 to 100 timelapses installs/renders/loading. Regions faster than 4 play silent in preview and export. |
| `project.add-annotation` | `startMs`, `endMs`, `type` (text/figure), `text`, `x/y/width/height` (%) | Drop text or figure annotation on canvas. |
| `project.update-spotlight` | `regionId` (req), then only what changes: `startMs`, `endMs`, `kind` (spotlight\|blur), `x`/`y`/`width`/`height`, `shape` (rectangle\|ellipse), `style` (gaussian blur\|pixelate), `blurAmount`, `pixelSize`, `maskOpacity`, `roundness`, `feathering` | Edit a focus region placed with `project.add-spotlight`. |
| `project.remove-spotlight` | `regionId` (req) | Remove a spotlight/blur region (ids under `editor.spotlightRegions[]`). |
| `project.set-aspect-ratio` | `ratio` (16:9/9:16/1:1/4:3/3:4) | Switch project aspect ratio. |
| `project.set-vertical-screen-layout` | `fill` (`follow` default \| `fit`), `corner` (`top-left` \| `top-right` \| `bottom-left` \| `bottom-right`) | Vertical layout for SCREEN RECORDINGS: screen fills the frame panning with the mouse (`follow`) or shows whole over a blur (`fit`); camera as a corner square. Set 9:16 first. Returns `{ followed, centered, skipped }`. |
| `project.set-wallpaper` | `wallpaper` | Set project background wallpaper id or 'none'. |
| `project.set-style` | `padding/shadowIntensity` (0-100, the slider percentage; 0-1 also accepted)`/borderRadius/motionBlurAmount/showBlur`, main-video frame `shape` (rounded\|circle) / `borderWidth` (0-40) / `borderColor` / `resetFrame` | Bulk-set cinematic style preset fields + the main-video frame (border ring, circle). Camera-only videos get their border here. |

## asset.*

Bundled sound effects + FX overlays that ship inside the app installer.

| Command | Args | Purpose |
|---|---|---|
| `asset.list-emoji` | `query` (optional) | Animated emoji available to `project.add-emoji` (char, Noto code, name, keywords). |
| `asset.list-sounds` | — | Every bundled sound: `{ id, name, category, absolutePath }`. |
| `asset.list-fx` | — | Every FX overlay: `{ id, title, blendMode, defaultOpacity, durationSeconds, defaultSoundId, absolutePath }`. |
| `asset.list-transitions` | — | Every bundled transition: `{ id, title, category, durationSeconds, defaultSoundId, absolutePath }`. Use the id with `project.add-transition`. |
| `asset.resolve` | `id` (string, required) | Resolve a bundled-asset id to its on-disk path. Returns `{ kind: "sound" \| "fx", path }`. |

## motion.*

Motion-graphic templates (title cards, lower thirds, end screens, etc.) with optional style packs.

| Command | Args | Purpose |
|---|---|---|
| `motion.list` | — | Returns `{ templates, registryBlocks }`. `templates`: slot-parameterized templates `{ id, name, slots, defaults, aspectRatios, durationMs, fileUrl }` (render with `motion.generate`). `registryBlocks`: standalone Hyperframes blocks `{ name, kind, title, tags, durationMs, dimensions, htmlPath }` (render the `htmlPath` with `motion.render-html`; no slots). |
| `motion.themes` | — | Every style pack: `{ id, name, swatch, colors }`. |
| `motion.generate` | `templateId` (string, required), `slots` (object, required), `aspectRatio` (`16:9` \| `9:16` \| `1:1`), `outputName` (string) | **Async.** Returns `{ jobId, outputPath }`. Poll `job.get` or block on `job.wait`. |
| `motion.render-film` | `frames` (req: `[{id, html|htmlPath, durationMs, transitionIn?}]`), `aspectRatio` or `width`+`height`, `groundColor`, `assets` (paths referenced by file name), `frameRate`, `outputName` | **Async.** Render a graphics-led film from per-beat frame compositions with between-frame transitions (`crossfade`, `blur-crossfade`, `push-slide DIR`, `zoom-through`, `squeeze`, `chromatic-wipe DIR`, `whip-pan DIR`, `iris`, `cut`; optional `0.4s`). Transitions extend the outgoing frame, so frame starts never move. Up to 180s. Result `{ outputPath, durationMs, frames[{id, startMs, durationMs}], transitions[], warnings[] }`. See reference/launch-video.md. |
| `motion.catalog` | `query`, `kind` (`component`\|`block`), `tag`, `limit` | Search the ~390-item HyperFrames catalog (camera moves, transitions, kinetic type, stats, device mockups, logo stings, CTAs). Returns `{ total, items[{name, title, description, tags, variables}] }`. |
| `motion.catalog-item` | `name` (req) | One item: `variables`, `htmlPath` (the recipe), `demoPath` (a working mount), `mount` snippet. Mount in any composition with `data-composition-src="catalog:<name>"`; renders stage it offline and hold it for the mount's `data-duration`. |
| `motion.craft` | `id`, `kind` (`guide`\|`blueprint`\|`rule`\|`preset`) | Launch-film craft docs. No id lists topics; with id returns the doc (`story-design`, `visual-design`, `motion-language`, `cut-catalog`, `blueprints`, `rules`, `transitions`, `design-presets`, any blueprint, rule or preset). |
| `motion.render-html` | `html` OR `htmlPath` (one required), `aspectRatio` (`16:9`/`9:16`/`1:1`) or explicit `width`+`height`, `durationMs` (default 2500), `frameRate` (default 30), `outputName` | **Async.** Render arbitrary HTML/CSS/JS to MP4 — for custom scenes, and for rendering a `registryBlocks` block by its `htmlPath` (from `motion.list`). Returns `{ jobId, outputPath }`. |

Theme application: pull `motion.themes`, find the theme the user wants, merge `theme.colors` into `slots` before sending. The backend doesn't know about themes — they're a pre-render layer.

## media.*

Project-agnostic media generation. Currently a single verb wrapping Replicate gpt-image-2 for B-roll, concept stills, reference frames.

| Command | Args | Purpose |
|---|---|---|
| `media.generate-image` | `prompt` (required), `aspectRatio` (`1:1` \| `3:2` (default) \| `2:3`), `quality` (`low` \| `medium` (default) \| `high`), `referenceImagePath` (path/URL, optional), `outputName` (slug, optional) | Generate one image. Returns `{ imagePath, prompt, aspectRatio, predictionId }`. Requires Replicate to be connected (Settings → Integrations → Connectors). The canonical B-roll workflow is `media.generate-image` → `motion.render-html` (Ken-Burns + vignette wrap) → `project.add-motion-graphic` — see SKILL.md "B-roll generation". For 16:9 video, generate `3:2` and crop in the wrap; for 9:16, generate `2:3`. |
| `media.generate-sound-effect` | `prompt` (required), `durationMs` (optional), `provider` (`auto` (default) \| `elevenlabs` \| `replicate`), `loop` (ElevenLabs only), `promptInfluence` (0-1, ElevenLabs only), `seed` (Replicate only) | Generate one sound effect (key press, typing burst, click, whoosh, riser, impact, UI pop, ambience). Returns `{ audioPath, durationMs, provider, model }`. `auto` uses ElevenLabs Sound Effects when ElevenLabs is connected, otherwise Stable Audio 2.5 via Replicate. Short effects are generated longer, then leading silence is removed and the clip is cut to `durationMs`. Place with `project.add-audio` at the frame of the action. Check `asset.list-sounds` first. |
| `media.import` | `url` (required, https), `name` (optional) | Download a video, image or audio file from a link into PandaStudio's storage. Returns `{ path, kind, bytes }`. Use for anything a generation service returns as a URL (Higgsfield outputs expire after about a week), then place it with `project.add-clip` / `project.add-motion-graphic` / `project.add-audio`. Refuses non-https and private-network links; 2 GB max. |
| `connector.list` | `state` (optional) | Every MCP connector for this workspace and whether it is connected. Read-only; only the user can connect one. |
| `connector.tools` | `connector`, `tool`, `search`, `sizes` (all optional) | Browse a connected service's tools: one line per tool, or one tool's full input schema (`tool`). `sizes` reports schema cost in tokens. |
| `connector.call` | `connector`, `tool` (required), `args`, `timeoutMs`, `async` | Run one tool on a connected service. Returns `data`/`text`/`structured`, `links` (media.import them) and `files` (inline images/audio saved to disk). `async` returns a jobId for job.wait. |

## export.*

The export library (MyExports view). Read + patch only — there's no `export.start` yet; new exports require the editor.

| Command | Args | Purpose |
|---|---|---|
| `export.list` | — | Every export entry, newest-first. |
| `export.get` | `id` (string, required) | Single entry by id. |
| `export.update` | `id` (string, required), `patch` (object, required) | Patch entry fields (e.g. `generatedTitle`, `generatedDescription`). |
| `export.delete` | `id` (string, required) | Delete the library row (does NOT delete the underlying MP4 on disk). |
| `export.publish-instagram` | `id` (required), `caption`, `shareToFeed` (default true) | Publish an export as an Instagram Reel via the broker. Requires a connected Business/Creator account. Returns `{ mediaId, permalink }`. Long-running (~30-120s). |

## instagram.*

Reel publishing via the license-server Composio broker. Requires an activated license + a connected Business/Creator account (Personal accounts can't publish — a Meta restriction). The exported video uploads straight to Composio via a presigned URL; bytes never transit our server.

| Command | Args | Purpose |
|---|---|---|
| `instagram.connect` | — | Open the browser to connect an account. Returns `{ connectionId, redirectUrl }`; then poll `instagram.status`. |
| `instagram.status` | `connectionId` (optional) | `{ status, active }`. Poll after connect until `active`. |
| `instagram.account` | — | `{ connected, account, error }`. `account.publishable` is true only for Business/Creator. |
| `instagram.disconnect` | — | Remove the connection. Published Reels stay on Instagram. |

## transcript.* (v1.9.1)

The editorial primitive that makes PandaStudio PandaStudio. Every operation that "deletes words" actually adds trim regions the export pipeline skips.

| Command | Args | Purpose |
|---|---|---|
| `transcript.transcribe` | `id` \| `path`, `clipId` (optional) | **Async.** Run Parakeet TDT 0.6B on each clip's audio. Returns `{ jobId }`. Re-transcribing a clip keeps its word fixes (find-replace / insert-words / app edits): the job result reports `wordEditsReapplied`, `wordEditsDropped`, `droppedWordEdits[]`. |
| `transcript.get` | `id` \| `path` | Merged edited-time transcript: every word with `id`, `text`, `startMs`, `endMs`. Use `id`s as input to `delete-words`. |
| `transcript.delete-words` | `id` \| `path`, `wordIds` (string[]) | Translate word IDs into trim regions. Coalesces adjacent deletions. |
| `transcript.remove-fillers` | `id` \| `path`, `includeRepeats` (bool, default true) | Auto-detect filler words ('um','uh','you know',…) plus back-to-back repeats. Bulk-trims them. |
| `transcript.search` | `id` \| `path`, `query` | Find a phrase across the merged transcript. Returns matches with their word IDs. |

## audio.* (v1.9.1)

| Command | Args | Purpose |
|---|---|---|
| `audio.clean` | `id` \| `path`, `clipId` (optional), `echo` (optional bool) | **Async.** Run DeepFilter denoising on each clip; writes a sibling `.cleaned.wav` and points `clip.cleanedAudioPath` at it. `--echo=true` also reduces room reverb (writes `.cleaned.dereverb.wav`, keeps the DeepFilter WAV); `--echo=false` switches back. Job result reports `rt60Ms`, `decayBeforeMs`, `decayAfterMs` per clip. |
| `audio.probe` | `id` \| `path`, `clipId?`, `noiseDb?` (-30), `minSilenceSec?` (0.5) | EARS without export: per-clip hasAudio, mean/max dB, silence spans (clip-source time), + the project's music-bed overlays. Synchronous. |

## caption.* (v1.9.1)

| Command | Args | Purpose |
|---|---|---|
| `caption.toggle` | `id` \| `path`, `enabled` (bool) | Show/hide captions for the whole project. |
| `caption.set-template` | `id` \| `path`, `templateId` | Default `glowStack`. Static: `classic, modern, minimal, bold, spotlight, boxed, neon, colored, coloredWords, editorial, glowStack`. Animated (word by word as spoken): `kineticSlam, clipWipe, gradientPop, matrixDecode, glitchRgb, blendDifference`. |
| `caption.set-style` | `id` \| `path`, color/font/stroke/positionY/wordsPerLine overrides | Per-template style overrides. `positionY` is PERCENT of frame height from the top (0-100, default 85) — not a 0-1 fraction (fractions are auto-converted, but write percent). |
| `project.add-caption-region` (alias `project.hide-captions`) | `atMs` + `durationMs` or `startMs` + `endMs` (edited ms) | HIDE captions for a stretch, e.g. under a full-frame statement card or a UI demo. Returns `regionId`. |
| `project.remove-caption-region` (alias `project.show-captions`) | `regionId` (req; ids under `editor.captionRegions[]`) | Show captions there again. |

## export.* (v1.9.1 — the centerpiece)

| Command | Args | Purpose |
|---|---|---|
| `export.verify` | `exportId` \| (`exportPath` + project `id`/`path`) \| project only (newest export), `samples?` (default 8, max 24), `meanThreshold?` (0.05), `blockThreshold?` (0.2) | **Async.** Check an export matches the editor preview: picture length vs the edit (2-frame tolerance), sound present / not silent / same length as the picture, and editor-vs-export frame pairs at even moments plus inside every layout section's edges and near the end. Result `{ ok, summary[], durationOk, mismatches, frames[{atMs, reason, meanDiff, worstBlockDiff, match}], audio{ok, lengthDiffMs, silent}, sheetPath }`. Run it after every export you hand over. |
| `export.start` | `id` \| `path`, `outputPath` (optional), `quality` (`draft \| standard \| high \| ultra`), `normalizeLoudness` (optional: `streaming` \| `podcast` \| `off`, or `true`/`false`, `-14`/`-16`; default = the project setting, streaming) | **Async.** Render the project to MP4 via the same Tier-3 PixiJS pipeline the editor's Export Video button uses (v1.24+). Reuses an open editor if it's already on the target project, otherwise spawns a hidden editor window for the duration of the render. Returns `{ jobId, outputPath }`. Honours every region/style/caption/FX/motion-graphic in the project. The final mix is loudness-normalised (two-pass, -14 LUFS / -1 dBTP by default); the job result's `loudness` reports input/output LUFS or why it was skipped. |
| `export.list` | — | Every entry in the export library, newest-first. |
| `export.get` | `id` | Read a single library entry. |
| `export.update` | `id`, `patch` | Patch fields like generatedTitle. |
| `export.delete` | `id` | Delete library row (NOT the underlying MP4). |

## llm.*

PandaStudio bundles Gemma 4 E2B (~2B params). Good for summarisation / classification / short structured output. Bad for long prose or visuals.

| Command | Args | Purpose |
|---|---|---|
| `llm.status` | — | `{ downloaded, downloading, path, size }`. |
| `llm.infer` | `prompt` (string, required), `maxTokens` (number, default 256) | One-shot inference. Returns `{ text }`. |
| `llm.generate-title` | `id` \| `path`, `maxChars` (default 70) | **Project-aware.** Reads the merged transcript, returns a YouTube-ready title. |
| `llm.generate-description` | `id` \| `path`, `maxChars` (default 400) | **Project-aware.** Returns a 3-5 sentence description. |
| `llm.generate-timestamps` | `id` \| `path`, `maxChapters` (default 8) | **Project-aware.** Returns `[{ timeMs, label }, …]` chapter markers. |
| `llm.generate-caption` | `id` \| `path`, `maxChars` (default 2200) | **Project-aware.** Returns an Instagram Reel caption (punchy hook + hashtags) for `export.publish-instagram`. |

## job.*

| Command | Args | Purpose |
|---|---|---|
| `job.get` | `id` (string, required) | Snapshot of one job's status + progress + result. |
| `job.list` | — | Every job in memory (last hour after completion). |
| `job.wait` | `id` (string, required), `timeoutMs` (number, default 300_000 = 5 min, hard cap 30 min) | Block server-side until terminal state. Returns `{ job, timedOut? }`; `timedOut: true` is NOT a failure, call again with the same id. **Prefer this over client-side polling.** |
| `job.cancel` | `id` (string, required) | Cancel a running or queued job. Idempotent. |

## preview.* (v1.9.2)

Floating, always-on-top overlay window that mounts the editor's WYSIWYG canvas. ~1-2s boot, same render code path as the in-app preview pane. Singleton — second `preview.show` reuses the existing window.

| Command | Args | Purpose |
|---|---|---|
| `preview.show` | `id` \| `path`, `atMs` (optional), `autoplay` (bool, default true), `width` (default 800), `height` (default 450) | Open or refocus the preview overlay on a project. |
| `preview.seek` | `atMs` | Move the playhead in the open overlay. |
| `preview.hide` | — | Close the overlay. |
| `preview.list` | — | `{ open, size?, position?, url? }` — what's visible right now. |

## workspace.* / youtube.* (the rest: SKILL.md "Workspaces", reference/publishing.md)

| Command | Args | Purpose |
|---|---|---|
| `workspace.get-brand` | — | The active workspace's brand kit (colors, fonts, logoPath, voice). Read it before authoring custom graphics and use its values. |
| `workspace.rename` | `id` (req), `name` (req) | Rename a workspace. |
| `youtube.list-channels` | `accountId` (req) | Re-pull a connected account's channels from YouTube (a just-created channel missing from `youtube.list-accounts`). |
| `youtube.disconnect` | `accountId` (req) | Remove a YouTube connection from the active workspace. Confirm with the user first. |

## window.*

| Command | Args | Purpose |
|---|---|---|
| `window.editor` | — | Open (or focus) the editor window. |
| `window.home` | — | Open (or focus) the home/dashboard window. |
| `window.exports` | — | Open (or focus) the MyExports library window. |
| `window.preview` | `id` \| `path` (optional) | **v1.9.1** — open the editor focused on a project so the user can SEE what an agent is doing visually. No rendering cost beyond the existing live preview pane. Chromeless overlay variant ships in v1.9.2. |
| `window.focus` | — | Bring the front-most app window to the foreground. (Available even when license-gated.) |
| `window.list` | — | Every open window: `{ id, title, url, visible, focused }`. |

## Details and edge cases

The MCP tool descriptions are kept short to save context. These are the details they leave out.

**Workspaces and projects**
- `workspace.set-brand`: empty-string fields are dropped; an unknown `voice` value is ignored.
- `workspace.capture-brand`: the first run downloads the HyperFrames capture CLI, so give it a long timeout.
- `project.duplicate`: media files are shared with the source, not copied.
- `project.rename`: the file on disk keeps its name (files are keyed by project id), so renaming is safe while the project is open.

**Podcasts**
- `project.add-podcast-clip`: guests auto-sync by audio cross-correlation; preview and export draw every participant through the N-party podcast layout.
- `project.auto-sync-podcast --id --clipId`: cross-correlates each guest's audio against the host, stores a per-participant offset, and returns the offsets with a confidence. Use it after `add-podcast-clip --autoSync=false`, or to redo a sync.
- `project.set-participant-offset --id --clipId --speaker=guest|guest-2|guest-3 --offsetMs`: manual nudge, ±10000 ms. Positive delays that participant; 0 clears it. The host is the reference clock. Applied in preview and export. (`project.set-webcam-offset` only reaches the first guest.)

**Clips, regions and overlays**
- `project.move-clip`: a move never drops a region. A region anchored inside a cut starts where kept content resumes. Out-of-range `toIndex` is clamped.
- `project.split-clip`: clips always play their media from 0; an in-point is a head trim (the same as dragging a clip's left edge).
- `project.add-audio`: `fadeOut` only applies to bounded overlays (ignored on an uncapped full-length one). Useful to crossfade the seam of a looped music bed.
- `project.auto-reframe`: without `zoom`, each speaker's face is sized to a consistent fraction of the frame. Letterbox bars are detected once per clip and stay excluded even after `project.set-screen-transform`.
- `project.add-motion-graphic` with `file=bundled:transition/<id>`: stamps the transition id, so it cover-fits any canvas (a 16:9 sweep fills 9:16) and carries its own sound.
- `project.set-overlay-crop`: out-of-range values are clamped; the worst case is the whole source.
- `project.set-overlay-chroma-key`: first enable uses similarity 0.45, smoothness 0.25, spill 0.5; `color: auto` is read from the edges of the overlay's first visible frame. Preview, render-frame and export share one keyer.
- `project.add-mute-region` / `project.hide-captions`: regions may overlap.
- `project.add-emoji`: each emoji asset is downloaded once, then cached.

**Transcript**
- `transcript.find-replace`: a match that crosses a deleted stretch is split at the cut: the replacement goes on the kept words and the deleted words stay as they were (listed in `splitMatches`). Result: `{ replacedCount, wordsPatched, wordsMerged, trimsChanged: false, splitMatches }`.
- `transcript.restore-words`: a cut over a whole deleted sentence is split so the other words stay deleted; silence-removal trims are left alone. Returns `trimsRemoved` (cuts changed).
- `transcript.insert-words`: timing fills the gap the dropped word left, sized to the local speaking rate, borrowing a few ms from the previous word when the neighbours touch.
- `transcript.transcribe`: the job result includes `wordEditsReapplied`; each `droppedWordEdits[]` entry is `{ clipId, description, reason }` with reason `no-match`, `conflict` or `crosses-cut`.
- `caption.set-style`: a px `fontSize` is converted to rem before the 1.0-5.0rem clamp.

**Media, recording, motion**
- `recording.start`: on a brand-new install that never recorded, the first call returns a "grant Screen Recording and retry" error rather than hanging.
- `media.generate-image`: `referenceImagePath` also accepts an https URL. `outputName` is slugified with a timestamp appended (same for `media.image-to-video`).
- `motion.render-html`: seek through the timeline, never `window.__hf.seek`: it skips the compositor invalidation and renders with 1-second stalls.
- `motion.verify-frames`: frames are written to `<recordings dir>/<outputName stem>/frame-<ms>.png`. The full-res `path` is ~2 MB as base64; read `previewPath`.
- `motion.screenshot`: `atMs` snaps to the 30 fps frame grid. `outputPath` is 1920x1080; `previewPath` is a 1280-wide copy.
- `asset.list-luts` categories: natural, cinematic, dramatic, vintage, modern. `asset.list-music` `recommendedFor`: youtube-long, shorts, linkedin, loom.
- `preview.show`: a single window, takes 1-2 s to boot.

**Export and publishing**
- `export.start`: reuses an editor window already open on the project, otherwise renders in a hidden one that closes afterwards. Loudness: -14 LUFS integrated / -1 dBTP by default, two-pass, video untouched; the result's `loudness` carries `preset`, `targetLufs`, `mode` (linear|dynamic) and, when skipped, `reason` (no-audio|silent|failed|unavailable).
- `export.verify` result: `durationMs`, `expectedDurationMs`, `durationDiffMs`, `compared`, `frames[{ atMs, reason, meanDiff, worstBlockDiff, worstBlock{x,y}, match, previewPath, exportPath }]`, `skipped[]`, `audio{ expected, present, maxVolumeDb, silent, ok }`.
- `export.set-thumbnail`: copies the file into the managed thumbnails folder; prefer `export.generate-thumbnail` when you want iteration history.
- `export.publish-youtube`: expect roughly 30 s per 100 MB to upload. To replace a thumbnail, `export.generate-thumbnail` output is already 1280x720.
- Preview proxies are made for 10-bit, 4:2:2/4:4:4, HDR, ProRes/DNxHD, or over-150 Mbps sources.
