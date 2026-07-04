<!-- Part of the pandastudio skill. Editorial grammar for long-form editing. -->

# Long-form styles: the retention grammar (PROVISIONAL)

**What this is.** The retention layer for `youtube-long` edits — what to do
ON TOP of SKILL.md's default edit pipeline (transcribe → fillers → STT fixes
→ bad takes → silences) and the destination-profile defaults, so a 5–20 min
video *holds*, not just plays clean.

**Honesty header:** the shorts grammar (`shorts-styles.md`) is quantified
from a 13-video frame-by-frame creator study. THIS file is not yet — the
devices below transfer from that study and from the shorts validation runs;
the long-form-specific NUMBERS are sane defaults pending a dedicated
long-form study. Where a number is provisional it's marked `(prov.)`.
Don't present provisional numbers to the user as researched fact.

---

## 1. The laws, re-parameterized for long-form

- **LF1 — Cold open, then the promise.** First frame is content, not logo.
  The video's promise (what the viewer gets by the end) must land inside
  **10s** (profile's hook deadline). Cut channel-intro throat-clearing the
  same way shorts cut preamble. Intro cards only if the user asks — and
  AFTER the hook, never before.
- **LF2 — Pattern interrupt cadence.** Something visible changes every
  **20–40s** `(prov.)`: a zoom, a b-roll insert, a graphic, a designed
  segment, a chapter card. The shorts law (3–8s) at long-form pace reads
  as exhausting; the FAILURE mode here is the 90s+ static stretch — scan
  the render sheet for any and fill it.
- **LF3 — Loops at two levels.** The video-level promise opens in the hook
  and closes in the final chapter. Each CHAPTER opens its own small loop
  ("here's what most people get wrong about X") and closes it before the
  next chapter card. Chapter boundaries are where viewers decide to leave —
  close a loop just before, open one just after.
- **LF4 — Captions are seasoning, not law.** Unlike shorts (always-on
  required), long-form default is profile-driven: full captions for
  tutorial content, keyword/emphasis captions or none for cinematic
  content. When captions are on, ALL of L6's zoning rules apply unchanged
  (reserved caption band, footage-dependent position, never over a face or
  graphic text).
- **LF5 — Authenticity survives (unchanged).** keepFillers default ON for
  talking-head; trims between beats. Same as shorts L5.
- **LF6 — Zoning (unchanged).** Import L6 from shorts-styles.md verbatim —
  same canvas, same collision physics, same render-sheet verification.
- **LF7 — End for the algorithm, not the applause.** Long-form does NOT cut
  on the payoff frame like shorts: the last 10–20s is the bridge to the
  next video (verbal handoff + space for an end screen). Cut the "thanks
  for watching, don't forget to like" boilerplate; keep the content bridge.

## 2. Devices that transfer directly (same commands, long-form placement)

- **House signature (sweep + camera-click)** — same rule as shorts: opener
  at t=0 + every b-roll / animated-graphic entrance, BUT respect density:
  at long-form pace cap it at roughly one per 30s `(prov.)` and reserve it
  for chapter-level moments, or it stops reading as punctuation.
- **Full-frame title card** (see shorts-styles.md §9) — MORE at home in
  long-form than shorts: it's the natural **chapter card**. One per chapter
  boundary, editorial serif, one accent word, VO continues underneath,
  1.5–3s. Same `caption-editorial-emphasis --background=solid` command.
- **Designed segments (band/split layouts)** — for process explanations,
  comparisons, stat walkthroughs. In 16:9 use cameraSide left/right (the
  Vox/MKBHD split), not top/bottom bands.
- **Animated process graphics** — when the transcript describes a process,
  depict it (typing, transforms, command→result). Same hyperframes shell,
  16:9 dimensions (1920×1080).
- **B-roll policy** — identical: video b-roll user-provided only, generated
  images/graphics autonomous, never pause the edit, report upgrade slots.
- **Lower thirds** — long-form specific (shorts ban them): introduce the
  speaker/brand at first appearance, re-introduce topics at chapter starts.
- **Verification** — same render-sheet discipline, but sample AT chapter
  boundaries + mid-chapter rather than uniformly: that's where the state
  changes live. Same gates apply (STATIC_RENDER, contract lint).

## 3. Long-form-only structure

- **Chapters.** Derive chapter boundaries from the transcript's topic
  shifts. Each chapter: card (device above) + optional lower-third +
  YouTube chapter timestamp. CAVEAT: `llm.generate-timestamps` currently
  returns SOURCE-time, not edited-time — remap through the trim map before
  publishing chapters, or compute timestamps yourself from the edited
  transcript.
- **Zoom rhythm, not zoom density.** Profile says 3–6/min; distribute them
  ON transcript beats (claims, numbers, punchlines) with `anchorSourceMs`,
  never on a fixed grid. A zoom that lands mid-sentence on nothing reads
  as a mistake.
- **Music.** Only when asked (profile rule). If asked: duck under speech
  (vol ≤ 0.15), swap or lift at chapter boundaries so the bed tracks the
  structure.

## 4. Speed discipline

Budget scales with length: pre-flight + beat map ~3 min, then roughly
**+1 min of agent time per 2 min of video** `(prov.)` for cuts + devices,
plus render/export. A 10-min video should be fully edited in ~10 min of
agent work. Same one-pass beat-map rule as shorts: read the transcript
ONCE, plan every chapter/cut/zoom/graphic, then execute via
`project.apply-edit-plan`. The cheat sheet (`shorts-cheatsheet.md`)
applies verbatim — same verbs, same conventions, same gotchas.

## 5. What still needs the study (do not improvise)

Quantified hook structures per genre, real pattern-interrupt cadence per
audience, chapter length distributions, retention-curve-informed payoff
placement, b-roll density norms. When the long-form creator study lands,
this file gets the same treatment shorts-styles.md got — until then the
`(prov.)` numbers are defaults, not evidence.
