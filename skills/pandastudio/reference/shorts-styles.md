<!-- Part of the pandastudio skill. Editorial grammar for short-form editing. -->

# Shorts styles: the retention grammar

**What this is.** Evidence-based editing recipes for making talking-head shorts
genuinely engaging — reverse-engineered frame-by-frame from 13 top-performing
shorts across 4 creators (Ali Abdaal, Alex Hormozi, Diary of a CEO, Cleo Abram;
studied July 2026). Every rule below is quantified from real videos, and every
step is an executable verb. `reference/shorts.md` gets you a 9:16 project
(fork-from-shot pipeline); THIS file is how you edit it so it retains.

**When to load:** the user asks to make a short "engaging", "viral-style",
"like <creator>", or hands you a raw talking-head / podcast recording for
Shorts, TikTok, or Reels.

---

## 1. Pick the recipe (decision tree)

```
Is it a multi-person conversation (podcast/interview)?
  → podcast-clip
Is it a solo talking head?
  ├─ Punchy advice/business monologue, dense one-liners → bold-caption
  ├─ Teaching a framework / ranking things / listing tools → kinetic-educator
  └─ Scripted explainer where visuals carry the story → polished-explainer
User named a creator? "like Hormozi" → bold-caption ·
"like Ali Abdaal" → kinetic-educator · "podcast clips / DOAC" → podcast-clip ·
"like Cleo Abram / Vox" → polished-explainer
```

If genuinely ambiguous, ask ONE question offering the two closest recipes with
a one-line description of the look each produces. Never ask when the content
type makes it obvious.

---

## 2. The seven laws (apply to EVERY recipe)

These held across all 13 reference videos. Violating them is what makes a
short feel amateur, regardless of styling.

- **L1 — No preamble.** The final cut must reach speech (or the on-screen
  payoff) within **2.5s**. Cut every "hey guys", "so basically", "in this
  video" cold open. The first surviving sentence must be a verdict, claim,
  question, or promise.
- **L2 — The 5-second state law.** Something visible changes every **3–8s**:
  an overlay mutates, a zoom fires, a graphic swaps, the speaker switches.
  When reviewing your edit, scan for any 8s+ stretch with zero visual state
  change — that's a retention hole; fill it (zoom, graphic, cutaway).
- **L3 — Open a loop early, close it late.** First 5s promises something the
  viewer only gets near the end (the full list, the reveal, the answer).
  Structure the trim so the payoff lands at the **55–85%** mark, never earlier.
- **L4 — Captions always-on, tiny vocabulary.** Captions visible whenever
  anyone speaks. 1–6 words per screen (recipe sets the exact number). ONE
  emphasis mechanism per video — color OR weight OR box — never mixed.
- **L5 — Authenticity survives the edit** *(parameter: `keepFillers`,
  default ON for talking-head recipes)*. Top creators keep "um"s, laughs, and
  room feel; trims land BETWEEN beats, never inside sentences. If the user
  asks for "clean/polished speech", flip it: `transcript.remove-fillers`.
- **L6 — Zoning.** Never cover a face, and never let captions and graphic
  text occupy the same zone at the same time. CRITICAL CONSTRAINT: caption
  position is ONE GLOBAL value for the whole timeline (`caption.set-style
  --positionY`) — you cannot move or hide captions for a time range. So the
  caption band is RESERVED AIRSPACE: pick it FIRST, then design every text
  graphic, hook card, and band composition to stay out of it for the entire
  video. A center-zone text card with captions at 60% WILL collide — put
  the card's text in the top ~40% or size/position it clear of the caption
  band. Caption position is also FOOTAGE-DEPENDENT: the recipe values
  (50–66%) assume the face in the upper third; on center-framed or tight
  selfie footage, drop captions to 75–85% (below the chin). Captions
  position against the FULL canvas — during designed-segment bands a
  mid-frame percent lands on the squeezed camera, so factor bands in when
  choosing the one global value. VERIFY on the render sheet at every
  moment a text graphic is live: if a caption touches a face OR overlaps
  graphic text, re-zone — these are the first things a human reviewer
  notices.
- **L7 — End on the payoff frame.** Last 1.5–2s = the completed state
  (finished list, restated title, the answer on screen). Cut immediately
  after the final payoff word — no outro sentence, no fade.

### House signature: light-sweep + camera-click (all shorts)

The **light-sweep transition + camera-click SFX** is the house punctuation
mark. It fires in two places:

1. **The opener** — every short starts with it at t=0, a signature snap
   that makes the first frame feel deliberate.
2. **Every b-roll / animated motion-graphic ENTRANCE** — creators mark the
   moment a new visual layer enters the frame with a sweep + SFX so the
   cut reads as intentional, not jarring. Place it at the overlay's start
   time, aligned to the entrance.

```bash
# ONE overlay carrying both the sweep and its sound (the sound plays once
# when the overlay appears). Do NOT use project.add-transition here —
# transitions bridge a CUT between two clips and no-op mid-timeline over
# a single clip — and do NOT add the click as a separate audio track.
pandastudio project.add-motion-graphic --id=$ID --file=bundled:transition/light-sweep \
  --atMs=<entranceMs> --durationMs=973 --soundUrl=bundled:sound/camera-click
```

Guardrails: the sweep marks ENTRANCES of new visual layers (b-roll,
animated graphics, designed-segment starts) — NOT caption emphasis cards,
and NOT exits. Cap density at roughly one per 8s; past that the sound
stops being punctuation and becomes noise. If two entrances land closer
together, sweep the more important one.

Related default to watch: `project.add-motion-graphic` attaches a mouse-click
SFX by default — pass `--soundUrl=none` on text/emphasis overlays so three
cards don't click at the viewer.

### B-roll policy (sourcing + the no-blocking rule)

- **Video b-roll is USER-PROVIDED only.** Never generate video, never pull
  from the web. If the user attached clips or pointed at a folder, that IS
  the authorization — use them autonomously, matched to transcript beats.
- **Images and motion graphics may be generated autonomously**
  (`media.generate-image`, hyperframes) for concept b-roll.
- **Never pause the edit to ask for footage.** Identify the beats that
  want b-roll (product mentions, process descriptions, "imagine…" moments),
  and fill EVERY slot with the best automated stand-in — animated motion
  graphic, generated image, designed segment — so the delivered short is
  always complete. Then, in the wrap-up, list the upgrade slots: "beats at
  Xs/Ys would be stronger with real footage of A/B — provide clips and
  I'll swap them in." Upgrading is a follow-up, never a blocker.
- Every b-roll entrance gets the house sweep + click (see above).

## 3. Shared pre-flight (every recipe starts here)

**Speed rule: read `reference/shorts-cheatsheet.md` FIRST.** It has the
exact shape of every command a short needs, the conventions (atMs vs
startMs, soundUrl=none, the render queue, apply-edit-plan for bulk
ops), and a copy-paste hyperframes starter shell. With it loaded you should
never grep source code, dump command schemas, or hunt through other
reference files mid-edit — that spelunking is where slow edits lose 3-4
minutes. Budget: a ≤60s short is fully edited in ~6 minutes plus export.

Transcript discipline: fetch the transcript ONCE and build the complete
beat map in a single pass — cuts, zoom beats, graphic slots, caption plan,
payoff frame — then execute the whole plan. Re-reading words per edit is
the other big time sink.

```bash
ID=<project id>                      # from fork-from-shot or project.locate
pandastudio project.set-aspect-ratio --id=$ID --ratio=9:16   # if not already
pandastudio transcript.transcribe --id=$ID --json            # async → job.wait
pandastudio transcript.get --id=$ID --json > /tmp/words.json # word-level timing
```

From `/tmp/words.json` build the **beat map** — the editorial skeleton every
recipe consumes:
1. Split the transcript into idea-beats (a beat = one claim/step/item/answer,
   usually 1–3 sentences).
2. Note each beat's `startMs`/`endMs` (first/last word).
3. Mark the HOOK beat (must satisfy L1/L3) and the PAYOFF beat (L7).
4. Trim dead air between beats — build ONE `project.apply-edit-plan` call
   containing every inter-beat trim (gaps >400ms) AND every zoom from the
   recipe, instead of sequential add-* calls. One call, atomic, one save.
   (`transcript.remove-fillers` only if `keepFillers=off` — see L5.)

Everything time-based below fires at word timestamps from this map — zooms on
the emphasis word, graphics on the beat boundary. That's what makes the edit
feel intentional instead of rhythmic. **Cuts are semantic, never on a timer.**

---

## 4. Recipe: kinetic-educator (Ali Abdaal style)

Solo educator teaching frameworks, ranking tools, listing rules. The retention
engine is a **mutating top-zone overlay** — the camera almost never cuts.

| Parameter | Value |
|---|---|
| Hook | payoff-first verdict, pain question, or claim-stack; ≤2.5s. Optional 2s word-pop title (music-only, speech starts after) |
| Overlay format | one of: label-swap (per-item name+logo) · progressive-sections · chip-stack. Mutation every 3–8s |
| Captions | template `bold`/`colored`; `wordsPerLine` 3–4; chest height (`positionY` 62 — percent of frame height from top) |
| Camera | NO cuts within beats; jump-cut between beats; ONE punch-in at the ~75% act change (`depth=2` = 1.5×) |
| Audio | music bed from first beat (`asset.list-music` → `project.add-audio`, low volume); `keepFillers=on` |
| Payoff cadence | one item/verdict every 3–8s |

The PROVEN top-band mechanism (validated on a real short): **designed
segments** — `paper-panel` / `vox-side-panel` at 9:16 become a top band with
the camera reflowed to the bottom band. One per feature beat:

```bash
# Per feature beat [startMs..endMs] from the beat map. Renders queue
# automatically (app >= 1.60) — submit every beat's render back-to-back,
# keep editing while they run, job.wait the ids when placing:
JOB=$(pandastudio motion.generate --templateId=paper-panel \
  --slots='{"side":"top","title1":"SPEAK,","title2":"IT TYPES","subtitle":"cursor anywhere — just talk"}' \
  --aspectRatio=9:16 --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB
pandastudio project.add-designed-segment --id=$ID --fromJob=$JOB \
  --atMs=<beat startMs> --durationMs=<beat length> --cameraSide=bottom --cameraRatio=55
# Act-change punch-in (once, at the payoff act):
pandastudio project.add-zoom --id=$ID --atMs=<payoff act startMs> --durationMs=<rest> --depth=2 --soundUrl=none
# Captions:
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-template --id=$ID --templateId=bold
pandastudio caption.set-style --id=$ID --wordsPerLine=3 --positionY=62   # percent from top (0-100)
```

The **rapid-list** variant (10 items / 34s — the easiest stunning short to
automate): no captions at all; a per-item overlay (logo + name + use-case) IS
the caption; swap on each item's first word; title card 0–2s; end on the last
item, no outro.

## 5. Recipe: bold-caption (Hormozi 2024+ style)

Punchy monologue advice. IMPORTANT: encode the CURRENT restrained style — NOT
the 2021 clone-look (no ALL-CAPS walls, no emoji, no meme inserts, no b-roll).

| Parameter | Value |
|---|---|
| Hook | promise/stakes sentence at 0.0s, captions from frame 1, NO title card |
| Captions | mixed-case white, drop shadow, no box, no emoji; `wordsPerLine` 2–3; center-frame (`positionY` 50); template `modern` or `bold` + `caption.set-style` overrides |
| Emphasis | bold WEIGHT on the operative word (needs per-word spans — see §11 fallback) |
| Cuts | semantic only: new beat = cut. Median shot 3.5–5.6s |
| Zooms | alternate tight↔wide punch-ins on beat boundaries (`depth=2`, alternate `focusY` slightly); on any beat >8s add a slow drift: `project.add-zoom --depth=1 --durationMs=<beat length>` so no frame is static |
| Chapter pill (listicle variant) | incrementing "N. CATEGORY" pill per item — payoff every ~4.2s (see §11: stateful-overlay fallback) |
| Audio | faint bed for listicles; live/raw formats keep room audio; `keepFillers=on`, keep [laughter] |

## 6. Recipe: podcast-clip (DOAC style)

Two-person conversation → vertical clip. Our most defensible recipe:
PandaStudio records each participant locally, so every device below is exact,
not cropped-guesswork.

| Parameter | Value |
|---|---|
| Clip selection | `export.generate-shots` scores; pick a segment whose payoff can sit at 55–80% of the clip |
| Hook | enter mid-conversation at 0.0s + title banner naming the payoff, auto-out by ~5.5s (`motion.generate` overlay, white bg / black caps) |
| Switching | camera follows the speaking participant (`project.set-clip-layout` / podcast layout transforms per section — see visual-edits.md §podcast) |
| Reaction cutaways | 1–1.5s of the NON-speaker at reaction moments, speaker's audio continues |
| Long holds | >12s single-speaker → break with alternating punch-ins every 5–6s (`project.add-zoom`, alternate framing) |
| Captions | `wordsPerLine` 1–3, bold + stroke, `positionY` 62; per-speaker color when available (§11) |
| Branding | persistent top corner chip (show logo / guest-name strap) via overlay graphic |
| End card | 1.5–2s restating the title (SKIP on emotional/serious clips — match register, L5 spirit) |
| Audio | dry — NO music bed; keep laughter |

Advanced hook (use when the payoff sentence is quotable): **redacted hook** —
pull the payoff sentence to 0s but mute its operative words (trim the audio
words, overlay the show wordmark for the gap), then let the full sentence land
at its natural late position. Viewer must wait to fill the blank.

## 7. Recipe: polished-explainer (Cleo Abram style)

Scripted explainer where the visuals carry the story. v1 scope: structure,
hooks, captions, stat callouts. (Anchored leader-line annotations and
continuous zoom-journeys are deferred — see gaps G7.)

| Parameter | Value |
|---|---|
| Face time | bookends only: hook on-camera 0.5–2s, then visuals; 1.5–3s face cut-ins at pivots; face closes the CTA |
| Hook | curiosity claim spoken at 0.0s over the face; first visual J-cuts in by 2s; first NUMBER on screen by 4–5s |
| Visuals | B-roll/graphics segments 1.75–2.2s per cut; every segment names ONE new thing |
| Stat callouts | promote key numbers to center-screen pills persisting 2–3s (`motion.generate` stat template, one accent color for the WHOLE video) |
| Captions | small ALL-CAPS, single line, bottom-center (`positionY` 80), on 100% of frames incl. b-roll; NO per-word emphasis |
| End | spoken cliffhanger CTA + subscribe pill ("to find out how, subscribe") |

## 8. Verification pass (run before declaring done)

Render checkpoints and CHECK them against the laws — do not trust the timeline
view:

```bash
# ONE call: a 12-frame contact sheet across the whole short (cell k = frames[k],
# row-major). Read the sheet image; check the laws against it.
pandastudio project.render-sheet --id=$ID --count=12 --cols=4 --json
# Then spot-render ONLY the moments that need a closer look:
pandastudio project.render-frame --id=$ID --atMs=<suspect ms> --json
# And verify audio without exporting (music bed present? dead air? levels sane?):
pandastudio audio.probe --id=$ID --json
```

Checklist (view every frame):
- [ ] t=0–2.5s: speech/payoff already happening? (L1) Captions visible? (L4)
- [ ] No face covered by any graphic; caption band correct for the recipe (L6)
- [ ] Walk the beat map: any 8s+ span with zero visual state change? (L2)
- [ ] Payoff lands at 55–85%; final frame IS the payoff state (L3/L7)
- [ ] Caption size/case/emphasis matches the recipe table exactly
- [ ] One emphasis mechanism only; one accent color only (L4)
- Fix and re-render until all pass. Then `export.start --quality=high`.

## 9. Generated visuals: B-roll + motion graphics (use them deliberately)

When Replicate is connected (media.generate verbs available), the agent can
CREATE its inserts on the fly — this is a core part of what makes an edit feel
produced rather than trimmed. Two families:

- **Generated B-roll stills** — `media.generate-image` (one per abstract
  concept the speaker mentions but the camera can't show). House rule from
  the main skill applies: NEVER drop a flat photo — Ken-Burns + vignette it,
  or place it as a styled PiP inset panel. Fire all generations in parallel
  (async jobs), keep editing, attach when done.
- **Motion graphics** — `motion.generate` (bundled templates: stat callouts,
  side panels, label-swaps, title cards) or `motion.render-html` for custom
  compositions. Overlay=true for top-zone graphics per L6.

### Full-frame title card (the question / chapter card)

Reference device (Ali Abdaal, observed on-platform): the speaker fully
yields the frame to typography for a beat — an editorial serif question or
chapter title on a paper background, ONE word in the accent color ("What is
today's **adventure** going to be?"). Distinct from the hook card at t=0:
this fires MID-VIDEO.

- **WHEN:** a rhetorical question, a chapter pivot, or the hook restated —
  only where the words ARE the visual payoff of that beat. Never as
  decoration over ordinary narration.
- **VO continues underneath.** The card text is (near-)verbatim what's
  being spoken during it — audio continuity is what makes it feel like a
  beat, not a cut to a slide. Never pause speech for a card.
- **Duration 1.5–3s, max 1–2 per short.** Full-frame means the face is
  gone; keep total speaker-less time under ~20% of the video.
- **Style:** editorial serif, single accent word (counts as your ONE
  emphasis mechanism per L4 — match the caption highlight color), paper /
  brand-neutral background. One card style per video.
- **Zoning:** the card's typography must stay clear of the caption band
  (captions are global and keep rendering over the card — L6). Compose the
  type in the upper ~60% with the reserved band left quiet.
- **Entrance:** house sweep + camera-click (it's an animated-graphic
  entrance); the card itself renders with `--soundUrl=none`.

```bash
# 9:16 full-frame question card, then place at the beat (VO keeps talking).
# --background=solid: the template is a transparent overlay by default; a
# full-frame card wants the opaque paper background so the speaker yields
# the frame completely. Slots are sentence + emphasisWord (accentColor =
# your video's single accent).
JOB=$(pandastudio motion.generate --templateId=caption-editorial-emphasis \
  --aspectRatio=9:16 --background=solid \
  --slots='{"sentence":"What is todays adventure going to be?","emphasisWord":"adventure","accentColor":"#FFC400"}' | jq -r .jobId)
pandastudio job.wait --id=$JOB
pandastudio project.add-motion-graphic --id=$ID --fromJob=$JOB \
  --atMs=<beatMs> --durationMs=2400 --soundUrl=none --anchorSourceMs=<wordSourceMs>
```

**Animated process graphics — the HyperFrames move (do NOT skip this).**
When the transcript DESCRIBES a process, mechanism, or before/after — "you
speak and it types", "it removes the fillers", "select it and say rewrite" —
the strongest possible insert is a CUSTOM ANIMATED graphic that SHOWS that
exact process, authored with `motion.render-html` (HTML + GSAP against the
HyperFrames contract; load `reference/motion-philosophy.md` before authoring).
Templates are for labels and titles; PROCESSES deserve purpose-built motion.
Common patterns, each ~60 lines of HTML:
- **Dictation/typing**: mock input field + blinking cursor + words typing in
  (staggered GSAP reveal) — for any "speak/type/write" narration.
- **Clean-up/transform**: messy sentence → filler words strike through and
  fade → clean sentence settles — for "removes/fixes/formats" narration.
- **Command → result**: a chip with the spoken command ("panda, rewrite it")
  → text morphs to the improved version.
Rules: pure CSS/text/SVG animation (bundled images have a decode race — see
gaps); anchor the graphic to the transcript words it illustrates; match the
video's single accent palette; register the timeline in bracket form
(`window.__timelines["id"] = tl`); submit all renders up front (they
queue); verify with
`motion.verify-frames` or a render-sheet after placing. These can also BE the
top band: pass the render job to `project.add-designed-segment` so the band
itself animates instead of holding a static panel.

Per-recipe policy (match the reference grammar, don't spray):

| Recipe | Generated-visual policy |
|---|---|
| kinetic-educator | The mutating overlay IS the visual engine (templates). Add 1–3 B-roll/PiP inserts ONLY on abstract beats (a tool, a book, a place the camera can't show); 2–4s each, styled panel not full-frame |
| bold-caption | NONE by default — modern Hormozi is zero-b-roll by design. Only if the user asks |
| podcast-clip | Rare: one 2–4s topical cutaway when a concrete thing is named ("86 billion neurons" → brain render); definition-card overlay for jargon |
| polished-explainer | The main event: visuals carry 75%+ of runtime — stat pills, annotated stills, generated scene imagery; talking head is the bookend |

Rules that keep it tasteful: every insert must be ANCHORED to the transcript
word that motivates it (fire at that word's startMs); one insert per beat max
outside polished-explainer; inserts count as the beat's L2 state change (don't
stack a zoom on top); match the video's single accent color in any generated
graphic. If media.generate fails (no Replicate key), degrade to template
motion graphics only — never block the edit on image generation.

## 10. Speed discipline (users are waiting)

A standard 60–90s short should take roughly **12–18 tool calls**: pre-flight
(3) → ONE `apply-edit-plan` with all trims+zooms → caption setup (3) → overlay
`motion.generate` jobs fired WITHOUT awaiting each (collect jobIds, attach
later) → ONE `render-sheet` verification → targeted fixes → export. If you are
past ~30 calls, you are debugging, not editing — simplify the plan instead of
iterating. Never render overlays serially when they don't depend on each
other, and never verify with sequential single frames when one sheet answers
the question.

## 11. Capability notes (degrade gracefully, don't fake)

Some reference-grade devices need features that are still landing. Use the
substitute, never a broken approximation:

- **Per-word emphasis inside captions** (color/weight/box): not yet in
  `caption.set-style`. Substitute: for the 2–3 MOST load-bearing lines only,
  a `caption-editorial-emphasis` motion overlay (it supports an emphasis
  word); leave regular captions un-emphasized. Do NOT try to fake per-word
  color with annotations.
- **Stateful overlays** (tier lists, chapter pills, filling checklists):
  substitute sequenced independent overlays — re-render the overlay per state
  with `motion.generate` and place each at its beat. Visually identical for
  label-swap and chapter-pill; skip true accumulating tier-lists until the
  feature lands.
- **Per-speaker caption colors**: single style for now; compensate with
  speaker-follow framing so identity is visual.
- **Redacted hook**: buildable today by hand (trim the words + wordmark
  overlay + `project.add-audio` silence) but fiddly; offer it only when the
  payoff quote is genuinely strong.
