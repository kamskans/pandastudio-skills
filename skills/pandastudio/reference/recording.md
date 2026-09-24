# Recording the screen yourself (agent-driven, v1.86+)

You can START and STOP a high-quality screen recording directly — no UI, no
user in the loop. This is the full-quality alternative to a browser's built-in
capture: drive a web app (or anything on screen) yourself, record it into
PandaStudio, then edit and export. macOS/Windows only.

```bash
# 1. (optional) see what you can target — displays + windows
pandastudio recording.list-sources --json
#   → { displays:[{id:"screen:1:0",name:"…",primary:true}], windows:[{id:"window:123:0",name:"Google Chrome — …"}] }

# 2. start (defaults to the primary display; pass --source to pick a window/display)
pandastudio recording.start --json                          # whole primary display
pandastudio recording.start --source="window:123:0" --json  # just that Chrome window
#   → { recordingId, screenPath, freeDiskMb, lowDisk }

# 3. …now do the thing you want to capture (click through the app, etc.)…

# 4. stop — finalizes the MP4 AND creates an editable project by default
pandastudio recording.stop --name="ACME tutorial" --json
#   → { screenPath, durationMs, projectId, projectPath, projectCreated:true }
```

Then edit the returned project like any other: `transcript.transcribe` →
`transcript.remove-fillers` → `project.add-zoom` on the key clicks →
`media.generate-narration` for a voiceover (`project.add-audio`) → `export.start`.

Notes:
- **Permission:** screen capture needs the one-time OS Screen Recording grant.
  It is already granted for anyone who has ever recorded in the app, so this
  runs with zero interaction. On a brand-new install that never recorded, the
  first `recording.start` returns a clear "grant Screen Recording and retry"
  error instead of hanging — surface that to the user; you cannot grant it for
  them.
- **One at a time.** `recording.start` fails if a recording is already active —
  call `recording.stop` first.
- **No mic on this path.** Only screen (and optional `--systemAudio=true`).
  Record clean, then add narration with `media.generate-narration`.
- **Low disk:** if `recording.start` returns `lowDisk: true` (under 10 GB free),
  tell the user BEFORE a long capture. A recording writes continuously, and
  running out of space part-way through loses the take.
- **Encoder can't start (some Windows laptops with two graphics cards):**
  `recording.start` fails with a message saying the screen video encoder
  couldn't start. There is no fallback on this path; ask the user to record
  from the PandaStudio app itself, which can switch to standard screen capture.
- `recording.stop --createProject=false` just finalizes the MP4 and returns
  `screenPath` if you want to compose `project.new --withMedia=…` yourself.
- `recording.stop` results can carry a `warnings` array (capture-helper
  problems such as a stalled video feed padded with the last frame). Surface
  them to the user.
- **Countdown:** `recording.start --countdownSeconds=3` shows a 3-2-1 overlay
  (0-10 s, default 0) before capture starts. Use it when the USER is about to
  present or act on screen, not when you drive the capture yourself. The
  overlay is excluded from the recording. If the user presses Esc, the call
  fails with `code: countdown_cancelled` and nothing is recorded: ask before
  retrying.
- **HUD countdown preference:** recordings the user starts from the recording
  bar count down first (timer button: Off / 3s / 5s / 10s, default 3s; Esc or
  the record button cancels). Read or change it with `recording.get-countdown`
  / `recording.set-countdown --seconds=5` when the user asks.
