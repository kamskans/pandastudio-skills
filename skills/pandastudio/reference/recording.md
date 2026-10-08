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
- `recording.stop` results can also carry a `health` array: `low-disk` (under
  2 GB free during the take), `disk-critical` (under 0.5 GB: the capture
  stopped and saved itself early, so the recording is shorter than you drove
  it), `capture-stopped-by-system`. Tell the user.
- **Crash-safe.** Recordings are written so a crash, force-quit or power loss
  keeps what was captured (everything up to the last ~2 s on macOS, ~30 s on
  Windows). Find them and rebuild them into a normal recording + project:

  ```bash
  pandastudio recording.unfinished --json
  #   → { recordings:[{ recordingId, bytes, hasScreen, hasWebcam, hasMic, screenBackend, via, health }] }
  pandastudio recording.recover --recordingId=1791125062160 --json
  #   → { screenPath, webcamPath?, durationMs, projectId, projectPath, warnings? }
  ```

  Recovery lines the mic and camera up with the screen from the start times
  recorded during the take. `screenBackend: "legacy"` entries come from older
  builds; the user recovers those from the home screen.
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
- **Clean desktop:** full-screen recordings hide the desktop icons by default
  (macOS: Finder's desktop is switched off and Finder relaunches before the
  countdown; Windows: the shell's Show desktop icons toggle). They always come
  back: on stop, a cancelled or failed start, a crash (a watchdog process) or
  quit, and at the next launch if all else failed. Window recordings are left
  alone. `recording.start --cleanDesktop=false` skips it for one take;
  `recording.get-clean-desktop` / `recording.set-clean-desktop --enabled=false`
  read or change the setting (the desktop button on the recording bar). The app
  can't switch on Do Not Disturb (no public API); `recording.start` returns a
  `tip` asking the user to, so pass it on before they present.

## Record an area (source changes, available with the matching app build)

The app source picker offers **Area**: choose a display, drag a box, move it or
resize its corners, and press Enter or Record. Esc cancels. Aspect choices are
Free, 16:9, 9:16, 1:1 and 4:3. Pixel-size presets lock the box size while allowing
movement. The last box is remembered per display. A thin, capture-excluded border
marks the area during recording, including pauses.

`recording.list-sources` returns each display's `displayId`, `bounds` (global
points), and `scaleFactor`. `area` uses **display-local points**, so x/y start at
0 on that display, even when its global origin is negative. Minimum 32 x 32
points. Rectangles are clamped to the display, scaled for Retina/DPI, and rounded
inward to even pixel dimensions for H.264. The start result reports the effective
`area` and `pixelSize`. Areas cannot target window sources.

```bash
pandastudio recording.list-sources --json
pandastudio recording.start --area='{"displayId":1,"x":100,"y":80,"width":640,"height":360}' --aspectRatio=16:9 --json
# Or fit a portrait area on the chosen display:
pandastudio recording.start --source=screen:1:0 --aspectRatio=9:16 --json
pandastudio recording.stop --name="Area demo" --json
```

macOS crops through ScreenCaptureKit sourceRect; Windows crops the native frame
before encoding. The file itself is the selected area, so crash recovery and
pause/resume keep the same framing. New area projects use the source's native
aspect with zero padding; cursor positions are relative to the area and clicks
outside it are ignored. Native capture must be available for area recording;
an encoder/permission failure returns an error instead of recording the full
screen. Existing whole-screen browser fallback is unchanged.
