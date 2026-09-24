# Edit runbook — per-destination detail

SKILL.md holds the destination-profile table (the source of truth) and the
default pipeline. This file is the detail behind them: creator-style
overrides, anchoring, the ordered runbook as shell, performance, entry
triggers and one-shot plan announcements.

**Philosophy:** good video editing is a series of pattern interrupts that match
the platform's viewing context. A YouTube long-form viewer has settled in —
cuts every 5–8s and a cinematic LUT feel right. A TikTok viewer is scrolling —
you have 3 seconds to hook them and every second after needs a visible change.
A LinkedIn viewer is at work — an aggressive soundscape is wrong. A Loom viewer
doesn't want any editing beyond "cut the fluff". Same tools, very different
dials.

## LUT by content type (`youtube-long` only)

Other profiles use their fixed preset from the profile table.

| Content type | Preset | Intensity |
|---|---|---|
| Tech tutorial / SaaS demo | `modernVibrant` | 0.7 |
| Cinematic vlog | `cinematicTealOrange` | 0.9 |
| Educational / neutral | `naturalEnhanced` | 0.5 |
| Moody storytelling | `moodyDark` | 0.7 |
| Travel / lifestyle | `warmSunset` | 0.7 |

Flat / washed-out camera footage (log or flat picture profile)? Correct it
FIRST with `project.set-clip-color --preset=flat-footage`, then add the LUT on
top. Apply to every clip via `project.set-clip-lut`. Skip for `loom`.

## Creator-style overrides (when the user names a style)

When the user says *"like Ali Abdaal's videos"* / *"MKBHD style"* /
*"MrBeast-style"* / etc., start from the matching base profile, then apply the
overrides below on top. Unlisted styles → base profile. For the Ali Abdaal
look on on-camera footage, the app's **Ali style** recipe
(`educator-talking-head-chapters`) supersedes this row: run the recipe.

| Style | Base profile | Pacing | LUT | Music | Caption template | Motion-graphic cadence + notes |
|---|---|---|---|---|---|---|
| **Ali Abdaal** (productivity / book reviews / tutorial long-form) | `youtube-long` | 1 visual change every **3–5s**; aggressive filler + silence removal | `modernVibrant` @ 0.5 | warm ambient / lofi @ 0.15–0.20 | `bold`, positionY 85 (below lower-third zone) | Intro title card (3s held) · host lower-third at 0:04–0:09 · 3–4 right-rail concept callouts at emphasis claims · 1 stat-reveal full-frame takeover if the video cites a number · outro card 4–6s hold with "Like & Subscribe" + shimmer on handle |
| **MKBHD** (tech reviews / product-focused long-form) | `youtube-long` | 1 change every **4–6s** — contemplative, product breathes on screen | `modernVibrant` @ 0.6 OR `cinematicTealOrange` @ 0.5 | upbeat tech-review bed @ 0.20 | `minimal` @ positionY 85 | Clean intro wordmark (2s) · minimal lower-thirds (1 total, on first product mention) · stat-reveals over product shots use chrome-gradient numbers on dark · outro: product recap card + subscribe |
| **MrBeast** (stunts / challenges / max-retention) | `youtube-long` | 1 change every **2–3s** — very fast, shorts-like cadence | `warmSunset` @ 0.8 (saturated, warm) | dramatic orchestral bed @ 0.30 | `neon`, **huge** (fontSize ~4.5rem, near the 5.0rem max), color-coded by topic, positionY 85 | Big chrome-gradient kinetic-type every ~5s · frequent full-frame stat takeovers with counter tweens · countdown overlays if the video has stakes · outro: "what's next" teaser card, **hold full 6s** |
| **Veritasium / Kurzgesagt-live** (science / education long-form) | `youtube-long` | 1 change every **5–7s** — contemplative, give diagrams time to read | `naturalEnhanced` @ 0.4 | ambient / orchestral @ 0.12 | `minimal` @ positionY 85 | Explanatory diagrams as motion graphics (labeled SVGs with `power2.inOut` reveals, `stagger: 0.15` on labels) · chapter dividers with chrome-gradient section titles · one or two hero stat-reveals with counter tweens · outro: citations card + subscribe |
| **Vox / Johnny Harris** (explainer / essay long-form) | `youtube-long` | 1 change every **4–6s** — narrative-driven | `cinematicTealOrange` @ 0.7 | cinematic bed @ 0.18 | `minimal` @ positionY 85 | Chapter cards at every act break (bold chrome-gradient section titles) · map / timeline / chart motion graphics · pull-quote callouts in right rail · outro: credits card + next video teaser |

**Rule:** any "style X" edit MUST still follow the 11 Laws from
`reference/motion-philosophy.md`. The overrides change palette, cadence, and
the *suggested* music/intro/outro; they do NOT let you ship flat-white text on
a flat-black background.

**Music + intro/outro stay opt-in even in style mode.** The Music and cadence
columns describe what the style *would* include, but background music and
intro/outro cards are still added ONLY when the user asked. If the user named a
style without mentioning them, apply palette / pacing / captions / mid-roll
graphics, and *offer* the music bed + intro/outro.

## Anchoring — every transcript-derived region MUST be anchored

Region positions are stored in **edited time**. When `transcript.remove-fillers`,
`remove-silences`, `delete-words` or `find-replace` add trims, the edited-time
map shifts, and a region authored against the previous edited time drifts (a
"subscribe" lower third silently moves 800ms early because 800ms of "um"s
before it got trimmed).

**The fix is `--anchorSourceMs`.** When you derive `atMs` / `startMs` from a
transcript word's source time, pass the same value as `--anchorSourceMs`. The
region records its anchor in raw recording time and every later trim/speed edit
rebases it back onto the anchor.

v1.35.0+: every region is auto-anchored on creation even without
`--anchorSourceMs` (the primitive back-computes an anchor from the resolved
`atMs`). Still pass it when you have a transcript word in hand: it documents
intent and skips a round trip. Legacy projects (schemaVersion < 4) are
auto-migrated on the first mutation; `type: "free"` anchors are preserved as
opt-outs.

| Verb | Anchor args | When required |
|---|---|---|
| `project.add-zoom` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when atMs comes from a transcript word |
| `project.add-motion-graphic` | `--anchorSourceMs`, `--anchorSourceEndMs` | Same |
| `project.add-lower-third` | `--anchorSourceMs`, `--anchorSourceEndMs` | Same |
| `project.add-annotation` | `--anchorSourceMs`, `--anchorSourceEndMs` | Always when startMs comes from a transcript word |
| `project.add-designed-segment`, `add-motion`, `add-adjustment`, `add-emoji`, `add-background-effect`, `caption.move` | `--anchorSourceMs` | Same |
| `project.add-audio` | `--anchorSourceMs`, `--anchorSourceEndMs` | SFX pinned to a word. **NEVER for background music** (keep it free-floating) |

Free-floating is OK when the user placed a region by edited time (an outro
card in "the last 5 seconds"). `anchorSourceMs` is global source time (ms from
recording start): in multi-clip projects sum the preceding clips'
`sourceDurationMs`. Verify with `timeline.source-to-edited`. Verbs WITHOUT an
anchor (`add-transition`, region edits) need a pre-converted edited time.

## The runbook (ordered — do not rearrange)

```bash
# 0. Resolve the project + destination profile
ID=$(pandastudio project.current --json | jq -r '.data.project.id // empty')
[ -z "$ID" ] && ID=$(pandastudio project.list --json | jq -r '.data.projects[0].id')
# $PROFILE from the MUST-ASK gate: youtube-long | shorts | linkedin | loom
# $ASPECT: youtube-long | linkedin | loom → 16:9, shorts → 9:16
pandastudio project.set-aspect-ratio --id=$ID --ratio=$ASPECT

# 1. PACING — the default cleanup pipeline (SKILL.md "default edit pipeline").
#    audio.clean runs async; wait on it before export. First read pulls the
#    transcript; later reads pass --includeTranscript=false.
pandastudio project.read --id=$ID --json
pandastudio transcript.transcribe --id=$ID               # skip if transcribed
AUDIO_CLEAN_JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId // empty')
pandastudio transcript.remove-fillers --id=$ID
# find-issues is READ-ONLY — delete the discarded wordIds (keep the most recent
# take; keep severity "low" candidates unless clearly abandoned).
ISSUES=$(pandastudio transcript.find-issues --id=$ID --json | jq -c '.data.issues')
DROP=$(echo "$ISSUES" | jq -c '[.[] | select(.severity != "low") | .wordIds[]]')
[ "$DROP" != "[]" ] && pandastudio transcript.delete-words --id=$ID --wordIds="$DROP"
if [ "$PROFILE" = "loom" ]; then
  pandastudio transcript.remove-silences --id=$ID --thresholdMs=300  # aggressive
else
  pandastudio transcript.remove-silences --id=$ID                    # 600ms default
fi

# 2. EMPHASIS — zooms (skip for `loom`). Cadence + duration from the profile table.
#    HARD RULE #1 — no agent zooms on screen-share clips. PandaStudio auto-adds
#    zooms from cursor telemetry; adding more stacks zooms on the same moments.
#    A clip with webcamPath ("both" mode) or a screen-only clip already has
#    telemetry zooms: SKIP. Only author zooms on pure-camera clips.
#    HARD RULE #2 — long-form zooms ride through a thought: emphasis 6–8 s,
#    sustained/held 10–20 s, reveal ≥6 s. A 1.5 s zoom on long-form reads as a
#    twitch. Don't stack zooms within 5 s of each other.
#    Shot selection: punchy claims, specific numbers, opinionated statements
#    ("the best", "most people don't know"), pivots ("but here's the thing").
#    ALWAYS pass --anchorSourceMs when atMs comes from a transcript word.
pandastudio project.add-zoom --id=$ID \
  --atMs=<wordStartMs> --anchorSourceMs=<wordStartMs> --durationMs=7000 --depth=2
# Sustained held zoom for a sub-topic
pandastudio project.add-zoom --id=$ID \
  --atMs=<sectionStartMs> --anchorSourceMs=<sectionStartMs> --durationMs=15000 --depth=2
# Reveal (1-2 per video max)
pandastudio project.add-zoom --id=$ID --atMs=<ms> --anchorSourceMs=<ms> \
  --durationMs=6000 --depth=5 --soundUrl=bundled:sound/dramatic-whoosh --soundVolume=0.7

# 3. POLISH — skip by profile: `shorts` no lower thirds; `loom` skips 3b + 3c.
#    3a (intro/outro) and 3d (music) are OPT-IN: only when the user asked.

# 3a. Intro / outro card — ONLY IF ASKED. Fastest: `creator-card` via motion.generate;
#     custom HTML (motion-philosophy.md §7) only for a bespoke intro.
if [ "$USER_ASKED_FOR_INTRO" = "1" ]; then
  JOB=$(pandastudio motion.generate --templateId=creator-card \
    --slots='{"headline":"<title>"}' --aspectRatio=16:9 --json | jq -r '.data.jobId')
  pandastudio job.wait --id=$JOB --json
  pandastudio project.add-motion-graphic --id=$ID --fromJob=$JOB --durationMs=3000 --atMs=0
fi

# 3b. Lower third at first mention of a person/product (NOT shorts/loom)
if [ "$PROFILE" = "youtube-long" ] || [ "$PROFILE" = "linkedin" ]; then
  JOB=$(pandastudio project.add-lower-third --id=$ID \
    --name="<name>" --title="<role>" --atMs=<ms> --anchorSourceMs=<ms> \
    --json | jq -r '.data.jobId')
  pandastudio job.wait --id=$JOB
  # Captions on? Lift them clear for exactly its span (2.0):
  # pandastudio caption.move --id=$ID --whileRegionId=<its overlay id> --positionY=62
fi

# 3c. LUT per the profile table (youtube-long: content-type table above).

# 3d. Background music — ONLY IF ASKED. Profile volume, ducked under the voice.
if [ "$USER_ASKED_FOR_MUSIC" = "1" ]; then
  VOL=$([ "$PROFILE" = "shorts" ] && echo 0.30 || echo 0.15)
  pandastudio project.add-audio --id=$ID --audioPath=bundled:music/corporate-underscore \
    --volume=$VOL --fadeIn=1000 --fadeOut=2000 --ducking=true
fi

# 4. CAPTIONS per profile (a recipe's caption setting always wins)
case "$PROFILE" in
  shorts)   TEMPLATE=neon ;;
  linkedin) TEMPLATE=minimal ;;
  *)        TEMPLATE="" ;;   # youtube-long: keyword pops, no burn-in (minimal only if the user insists); loom: optional
esac
if [ -n "$TEMPLATE" ]; then
  pandastudio caption.toggle --id=$ID --enabled=true
  pandastudio caption.set-template --id=$ID --templateId=$TEMPLATE
  # ALL CAPS on any template: caption.set-style --uppercase=true (false turns
  # OFF a template that ships caps, e.g. editorial).
  # HIDE captions for part of the video (edited ms; alias project.hide-captions):
  # pandastudio project.add-caption-region --id=$ID --startMs=12000 --endMs=17000
  # Show them again (alias project.show-captions; ids in editor.captionRegions[].id):
  # pandastudio project.remove-caption-region --id=$ID --regionId=caption-hide-1
fi

# MUTE part of the main audio without cutting the picture (music/SFX keep
# playing). Edited ms. For a cough, a name, dead air you want silent.
# pandastudio project.add-mute-region --id=$ID --atMs=12000 --durationMs=3000
# pandastudio project.remove-mute-region --id=$ID --regionId=mute-1

# 4.5 VERIFY FRAMES — MANDATORY. Never export without looking.
# render-frame at each hero moment (one PNG per check keeps the vision call
# light) and motion.verify-frames on every rendered motion-graphic MP4. Check:
# no cropped faces / text overflow / blank frames; captions on the right word;
# no graphic over the host's face or the screen zone; designed text rendering.
# VERIFICATION NEVER "FAILS" THE EDIT: every project.* mutation commits the
# moment it returns ok. If the frame read times out or the model doesn't
# respond, the edit still landed: report "Done — <edit> is on the timeline; I
# couldn't finish the visual check", never "failed".
pandastudio preview.show --id=$ID
# pandastudio motion.verify-frames --videoPath=/tmp/motion-intro.mp4 --timestamps='[0.3,1.0,1.8,2.7]' --json

# 5. EXPORT — only after the verify pass AND the user's go-ahead on the preview.
[ -n "$AUDIO_CLEAN_JOB" ] && pandastudio job.wait --id="$AUDIO_CLEAN_JOB"
QUALITY=$([ "$PROFILE" = "loom" ] && echo "standard" || echo "high")
pandastudio export.start --id=$ID --quality=$QUALITY --json | jq -r '.data.jobId' | \
  xargs -I {} pandastudio job.wait --id={}
```

## Performance — keep wall-clock minimal

Motion-graphic renders dominate (each scene ~20–45s); everything else is
rounding error.
- **`motion.render-html` renders run one at a time.** App >= 1.60 queues them
  (submit several, keep editing, `job.wait` each when placing); older apps
  return `RENDER_BUSY` for a second concurrent render.
- **Run `audio.clean` in the background** (capture its jobId after transcribe;
  `job.wait` only before `export.start`). Safe to overlap with a render.
- **Pre-flight HTML with `motion.screenshot`** before a full render — a ~2s
  screenshot at `--atMs=<mid-scene>` beats a 20–45s wasted render. It inlines
  GSAP and seeks the paused timeline.
- **Re-read sparingly**: `--includeTranscript=false` after the first
  `project.read`; reuse what mutations return.

## Entry triggers — phrases that start the playbook

Don't ask the user to expand any of them: resolve profile + style, announce
the plan, execute.

| User says | Resolve to |
|---|---|
| "edit this" / "polish this" / "make it engaging" / "make it ready" | `youtube-long` (duration band permitting), no style override |
| "YouTube-ready" / "make a YouTube video" / "edit for YouTube" | `youtube-long`, no style override |
| "make it a Short" / "TikTok" / "Reel" / "vertical" / "9:16" | `shorts`, no style override |
| "for LinkedIn" | `linkedin` |
| "Loom" / "internal update" / "just cut the fluff" | `loom` |
| "edit like Ali Abdaal" / "Ali Abdaal style" / "tutorial style" / "productivity video" | `youtube-long` + Ali style recipe |
| "MKBHD style" / "tech review style" / "product review" | `youtube-long` + **MKBHD** override |
| "MrBeast style" / "high-retention" / "challenge video" / "maximum engagement" | `youtube-long` + **MrBeast** override |
| "Veritasium style" / "educational" / "Kurzgesagt vibe" / "explainer" | `youtube-long` + **Veritasium** override |
| "Vox style" / "essay" / "narrative" / "Johnny Harris style" | `youtube-long` + **Vox** override |

## Pattern: one-shot execution

After profile/style resolution, announce the plan in one sentence and execute.
Load `reference/motion-philosophy.md` before the motion-graphics steps. Run the
full runbook including the frame-verification gate. Do NOT ask the user to
approve individual steps; if something genuinely needs input (a missing brand
reference, a lower-third name), collect ALL questions in one message.

> I'll edit this as a **Short** — hook in 3s, 6–12 zooms/min, `modernVibrant`
> LUT at full intensity, `neon` captions positioned higher, no intro card or
> lower thirds (they don't fit the vertical frame). Frame-verify before
> export. ~2 minutes.

> I'll edit this as a **MrBeast-style YouTube video** — 2–3s pacing,
> `warmSunset` LUT at 0.8, huge color-coded `neon` captions, chrome
> kinetic-type every 5s, full-frame stat takeovers. Want the dramatic music bed
> and the 6s outro teaser card too? Frame-verify before export. ~6 minutes.
