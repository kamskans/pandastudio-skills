<!-- Part of the pandastudio skill. Editorial grammar for long-form editing. -->

# Long-form styles: the retention grammar

**What this is.** The retention layer for `youtube-long` edits — what to do ON
TOP of SKILL.md's default edit pipeline (transcribe → fillers → STT fixes →
bad takes → silences) and the destination-profile defaults, so a 5–20 min
video *holds*, not just plays clean.

**Evidence.** Quantified from a 9-video measured study (July 2026): 3 videos
each from Ali Abdaal (educator), MKBHD (product review), Fireship
(dev explainer) — ffmpeg scene detection (963 hard cuts), 502
vision-classified frames, hook/chapter/ending anatomy per video. Numbers
below are measured, not vibes. `shorts-styles.md` is the sibling file for
9:16; this file assumes 16:9.

---

## 1. The laws (all three creators, no exceptions found)

- **LF1 — Thesis inside 10s, cold open always.** All 9 videos speak the
  thesis/promise within 10s. No logo-first opens; a branded sting (~3s) is
  acceptable only AFTER a spoken tease (MKBHD). Intro ends and chapter 1
  starts by 24–72s (≤6% of runtime). Hook cut rate runs 0.6–1.9× body rate —
  faster is common but not law; the spoken promise is the law.
- **LF2 — The two-level rhythm.** Micro: median shot 3–6.5s, 7–13 visual
  changes/min (creator-dependent, see recipes). Macro: the LAYOUT alternates
  every 20–40s (talking head ↔ b-roll/graphic/insert). A stretch can pass
  micro (jump cuts) and still fail macro — check both on the render sheet.
- **LF3 — Static is a spent resource.** Nothing exceeds 60s without a
  visual change, and every measured 30–60s hold was DELIBERATE: the verdict
  monologue, the closing argument, a live terminal carrying its own motion.
  Budget at most one long hold per video and place it at 72%+ (the
  reasoning/payoff moment). An accidental 45s static stretch mid-video is
  the #1 amateur tell.
- **LF4 — No burned-in speech captions. Keyword pops instead.** 0/502
  frames across all three creators had subtitle-style captions (platform
  captions handle accessibility). On-screen text = 1–4 word KEYWORD pops
  timed to the spoken word (serif pops for educator, angled color stickers
  for dev content, credit tags/spec cards for review), plus full-frame
  kinetic quote cards for the tweetable line (2–4 per video).
- **LF5 — Authenticity survives (unchanged from shorts L5).** keepFillers
  default ON for talking-head; trims between beats.
- **LF6 — Zoning (unchanged from shorts L6).** Same collision physics.
  With captions off by default the reserved band constraint mostly moves to
  keyword pops + lower thirds: same rule, never over a face or other text.
- **LF7 — Speech runs to the final seconds.** No endscreen slates, no
  subscribe animations, no outro montages anywhere in the corpus. Formula:
  verdict/recap → verbal bridge (next video / question to comments) → fixed
  sign-off → hard stop, speech ending within ~6s of the last frame. Cut the
  "thanks for watching, don't forget to like" boilerplate; keep the bridge.
- **LF8 — Segmentation is IN-EDIT, chapter markers optional.** All 9 videos
  segment visually; only Ali also populates YouTube chapters. Boundary
  treatments (pick per recipe): full-frame numbered title card (2–4s), a
  scorecard/checklist pivot graphic, or a persistent incrementing corner
  tag. If you publish YouTube chapters, remember `llm.generate-timestamps`
  returns SOURCE-time — remap through the trim map first.

## 2. Recipes (pick by content, mirror of the shorts recipes)

### educator-pip (Ali Abdaal) — frameworks, habits, how-to
- **Workhorse layout (28–54% of frames): full-frame graphic with small PiP
  talking head** (rounded corner card, ~1/6 frame width). Face stays on
  screen ~85–90% of runtime while the graphic carries the content. In
  PandaStudio: `project.add-motion-graphic --layer=background` + a CARD
  camera clip-transform over the same span (or `add-designed-segment`).
- 8–9 changes/min, median shot ~5s; if nothing cuts, animate a diagram
  element in.
- Keyword pops: 1–4 word serif overlays at the spoken keyword. Kinetic
  quote cards (cream bg, serif, word-by-word, one accent word — the
  `caption-editorial-emphasis` card) 2–4× per video.
- Chapters: few long chapters (6–10) → full-frame numbered title cards
  2–4s; many micro-chapters (20+) → persistent corner counter tag instead.
  Populate YouTube chapter markers.
- Diagrams over raw screenshots: rebuild UIs as clean mock cards.
- Ending: verbal bridge, then the video's longest hold (30–60s single-take)
  is allowed HERE.

### product-review (MKBHD) — reviews, comparisons, "should you buy"
- **Alternation engine: talking head ↔ product b-roll on a 20–40s period**
  (~50/50 split of frames). 7–8 changes/min, median shot 5.5–6.5s.
- Spec-card ritual: ONE white rounded spec card (≤4 bullets) over slow hero
  b-roll ~20s in; never repeated.
- The pivot is a scorecard/checklist graphic at the "here's the catch"
  moment — not a chapter card. Full-frame data charts for comparisons.
- Credit-tag every borrowed clip (small diagonal source tag).
- Verdict = the longest hold (32–55s), placed 72–92%. Ending liturgy:
  recap → question to comments → sign-off, speech to within 6s of end.

### dev-explainer (Fireship) — code, tools, technical deep-dives
- **No talking head (0% of frames). Voiceover over: additive flat diagrams
  on black (elements pop in per narrated clause), live-typed terminals with
  real output, document scans with highlighter + hand-drawn arrows, and
  meme/stock punctuation (~35% of frames, 1–3s each) after each
  explanatory beat.**
- Fastest rhythm: ~13 changes/min, median shot ~3s, hook ~1.4× faster.
- Text = angled colored sticker keywords (1–3 words, rotated 3–10°); the
  flag/option under discussion gets a floating label beside the terminal.
- Structure: hidden numbered listicle (~1 boundary/min), each boundary a
  2–4s black card with numbered badge + logo. No YouTube chapters needed.
- Terminals are the legitimate long shots (20–47s) — keystroke motion
  carries them; don't cut away mid-command.
- Ending: hard stop, fixed sign-off, final frame is a gag beat. Zero
  endscreen.

## 3. Devices that transfer from shorts (same commands)

House sweep + camera-click on graphic/b-roll entrances (long-form cap:
reserve for chapter-level moments, ~1/30s max); full-frame title cards
(`caption-editorial-emphasis --background=solid` + dark inkColor) as
chapter cards; designed segments (16:9: cameraSide left/right); animated
process graphics (hyperframes, 1920×1080); b-roll policy verbatim (user
video only, generated stills/graphics autonomous, never block, report
upgrade slots); lower thirds at first mentions (long-form only — shorts ban
them); zooms 3–6/min ON transcript beats with `anchorSourceMs`, never on a
grid; sponsor reads keep the house grammar (mid-roll ~50% mark is the norm,
and it may be the video's longest unbroken shot).

## 4. Verification + speed

Render-sheet at chapter boundaries + mid-chapter (that's where state lives),
plus a dedicated LF2/LF3 scan: flag any 60s+ window with no visual change
and any 30–60s hold that isn't the designated payoff moment. Same gates
apply (STATIC_RENDER, contract lint). The cheat sheet
(`shorts-cheatsheet.md`) applies verbatim — same verbs, conventions,
gotchas. Budget: pre-flight + beat map ~3 min, then ~1 min of agent time
per 2 min of video, plus renders/export; a 10-min video ≈ 10 min of agent
work via one `apply-edit-plan` batch.
