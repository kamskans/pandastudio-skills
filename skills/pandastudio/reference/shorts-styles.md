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
- **L6 — Zoning.** Never cover a face. Graphics live in the top ~40% of the
  frame; captions in the 48–66% vertical band; faces below/behind.
- **L7 — End on the payoff frame.** Last 1.5–2s = the completed state
  (finished list, restated title, the answer on screen). Cut immediately
  after the final payoff word — no outro sentence, no fade.

## 3. Shared pre-flight (every recipe starts here)

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
4. Trim dead air between beats: `project.add-trim` on inter-beat gaps >400ms.
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

```bash
# Per beat i with [startMs..endMs] from the beat map:
pandastudio motion.generate --templateId=<overlay template> \
  --slots='{"title":"<item name>","subtitle":"for <use case>"}' \
  --aspectRatio=9:16 --overlay=true --json      # → add at beat startMs, duration = beat length
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
| Emphasis | bold WEIGHT on the operative word (needs per-word spans — see §9 fallback) |
| Cuts | semantic only: new beat = cut. Median shot 3.5–5.6s |
| Zooms | alternate tight↔wide punch-ins on beat boundaries (`depth=2`, alternate `focusY` slightly); on any beat >8s add a slow drift: `project.add-zoom --depth=1 --durationMs=<beat length>` so no frame is static |
| Chapter pill (listicle variant) | incrementing "N. CATEGORY" pill per item — payoff every ~4.2s (see §9: stateful-overlay fallback) |
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
| Captions | `wordsPerLine` 1–3, bold + stroke, `positionY` 62; per-speaker color when available (§9) |
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
for T in 0 1500 2500 <hook end> <mid> <payoff start> <last frame - 500>; do
  pandastudio project.render-frame --id=$ID --atMs=$T --out=/tmp/verify-$T.png
done
```

Checklist (view every frame):
- [ ] t=0–2.5s: speech/payoff already happening? (L1) Captions visible? (L4)
- [ ] No face covered by any graphic; caption band correct for the recipe (L6)
- [ ] Walk the beat map: any 8s+ span with zero visual state change? (L2)
- [ ] Payoff lands at 55–85%; final frame IS the payoff state (L3/L7)
- [ ] Caption size/case/emphasis matches the recipe table exactly
- [ ] One emphasis mechanism only; one accent color only (L4)
- Fix and re-render until all pass. Then `export.start --quality=high`.

## 9. Capability notes (degrade gracefully, don't fake)

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
