# Recipes

End-to-end flows you can lift verbatim. Each is one shell session.

## 1. Render a 16:9 title card with a chosen style pack

```bash
# Pre-flight
pandastudio system.status --json | jq '.data.license'

# Pick a template + theme
pandastudio motion.list --json | jq '.data.templates[] | {id, name, slots: [.slots[].key]} | select(.id=="title-card-vox")'
pandastudio motion.themes --json | jq '.data.themes'

# Resolve the theme client-side: deep-merge theme.colors into slots
THEME_COLORS=$(pandastudio motion.themes --json | jq -c '.data.themes[] | select(.id=="mr-beast") | .colors')
SLOTS=$(jq -nc --argjson colors "$THEME_COLORS" \
  '$colors + {title:"How I Spent $10,000",subtitle:"in 24 hours"}')

# Render
JOB=$(pandastudio motion.generate \
  --templateId=title-card-vox \
  --aspectRatio=16:9 \
  --outputName=intro-card \
  --slots="$SLOTS" \
  --json | jq -r '.data.jobId')

# Wait + report
pandastudio job.wait --id="$JOB" --json | jq '.data.job | {status, result}'
```

## 2. Render a 9:16 (Shorts) lower-third

```bash
# Filter templates to ones that support 9:16
pandastudio motion.list --json | \
  jq '.data.templates[] | select(.aspectRatios | index("9:16")) | {id, name}'

# Pick one — say chapter-divider for a 9:16 chapter break — and
# inspect its slots
pandastudio motion.list --json | \
  jq '.data.templates[] | select(.id=="chapter-divider") | .slots'

# Render
JOB=$(pandastudio motion.generate \
  --templateId=chapter-divider \
  --aspectRatio=9:16 \
  --slots='{"chapterLabel":"Chapter","chapterNumber":"02","chapterTitle":"The Setup"}' \
  --json | jq -r '.data.jobId')

pandastudio job.wait --id="$JOB" --json | jq '.data.job.result.outputPath'
```

## 3. List recent exports and have the local LLM rewrite weak titles

```bash
# Pull exports needing better titles
pandastudio export.list --json | \
  jq -r '.data.exports[] | select(.generatedTitle | length < 20) | .id' | \
while read -r ID; do
  ENTRY=$(pandastudio export.get --id="$ID" --json)
  TRANSCRIPT=$(echo "$ENTRY" | jq -r '.data.entry.transcript.words[].text' | head -c 4000)

  NEW_TITLE=$(pandastudio llm.infer \
    --prompt="Rewrite this transcript's first 4000 chars as a punchy YouTube title (under 60 chars). Reply ONLY with the title, no quotes:\n\n$TRANSCRIPT" \
    --maxTokens=80 \
    --json | jq -r '.data.text' | head -1)

  pandastudio export.update --id="$ID" --patch="$(jq -nc --arg t "$NEW_TITLE" '{generatedTitle:$t}')"
done
```

## 4. Open a project for the user to edit visually

When the user asks "open my latest project," don't try to load it programmatically — there's no headless project-load verb. Just open the editor and tell them where to find it:

```bash
LATEST=$(pandastudio project.list --json | jq -r '.data.projects[0]')
echo "Opening editor — your latest project is: $(echo "$LATEST" | jq -r '.name') at $(echo "$LATEST" | jq -r '.path')"
pandastudio window.editor
```

## 5. Bulk-generate intros for a content series

```bash
# Read a CSV of episode titles, render a title card per row
while IFS=, read -r EP TITLE SUB; do
  SLOTS=$(jq -nc --arg t "$TITLE" --arg s "$SUB" '{title:$t, subtitle:$s}')
  JOB=$(pandastudio motion.generate \
    --templateId=title-card-vox \
    --aspectRatio=16:9 \
    --outputName="intro-ep$EP" \
    --slots="$SLOTS" \
    --json | jq -r '.data.jobId')

  RESULT=$(pandastudio job.wait --id="$JOB" --timeoutMs=120000 --json)
  echo "ep$EP → $(echo "$RESULT" | jq -r '.data.job.result.outputPath')"
done < episodes.csv
```

## 6. Probe the bundled FX library and pick one matching a vibe

```bash
pandastudio asset.list-fx --json | jq '.data.fx[] | {id, title, blendMode, durationSeconds}'

# Resolve a chosen id to its on-disk path (so you can hand it to ffmpeg
# or include it in a project.save payload).
pandastudio asset.resolve --id=light-leak-warm --json | jq '.data.path'
```

## 7. Compose a full edit from scratch (the v1.9.1 flow)

The end-to-end "I gave you 2 videos, give me an edited project" workflow.

```bash
# Step 1: create a project pre-loaded with the source clips. FFmpeg
# probes durations for you.
P=$(pandastudio project.new \
  --name="My Edit" \
  --withMedia='["/Users/me/Downloads/clip-a.mp4","/Users/me/Downloads/clip-b.mp4"]' \
  --json)
ID=$(echo "$P" | jq -r '.data.id')

# Step 2: render an intro title card via motion.generate
INTRO_JOB=$(pandastudio motion.generate \
  --templateId=title-card-vox \
  --slots='{"title":"My Edit","subtitle":"a recap"}' \
  --aspectRatio=16:9 \
  --outputName=intro \
  --json | jq -r '.data.jobId')

INTRO=$(pandastudio job.wait --id="$INTRO_JOB" --timeoutMs=120000 --json)
INTRO_PATH=$(echo "$INTRO" | jq -r '.data.job.result.outputPath')
INTRO_DUR=$(echo "$INTRO" | jq -r '.data.job.result.durationMs')

# Step 3: drop the intro at t=0
pandastudio project.add-motion-graphic \
  --id="$ID" --file="$INTRO_PATH" --durationMs="$INTRO_DUR" --atMs=0

# Step 4: add a lower-third intro 5 seconds in
pandastudio project.add-lower-third \
  --id="$ID" --content="Kamal" --subtitle="Founder" --atMs=5000

# Step 5: drop a film-burn FX between the two source clips
# (assume clip-a is 30s; transition is at the cut)
pandastudio project.add-fx \
  --id="$ID" --fxId=film-burn --atMs=30000

# Step 6: open the editor for the user to take over
pandastudio window.preview --id="$ID"

echo "Project ready: id=$ID"
echo "User can find it at: $(pandastudio project.show --id=$ID --json | jq -r .data.path)"
```

The user can now see + tweak everything in the editor. The agent and the user are working on the same project file — `expectedRevision` keeps them from clobbering each other.

## 8. Round-trip: agent edits, user takes over, agent continues

```bash
# Agent's first pass
P=$(pandastudio project.read --id="$ID" --json)
REV=$(echo "$P" | jq -r '.data.project.revision')

# User opens project in editor and makes changes (manual). Editor's
# autosave bumps the revision. The agent's $REV is now stale.

# Agent comes back later, tries to add another lower-third
RESULT=$(pandastudio project.add-lower-third \
  --id="$ID" --content="More" --atMs=10000 \
  --expectedRevision="$REV" --json)

# If conflict: re-read and retry
if [ "$(echo "$RESULT" | jq -r '.details.code')" = "revision_conflict" ]; then
  # Re-read with the actual on-disk revision
  P=$(pandastudio project.read --id="$ID" --json)
  REV=$(echo "$P" | jq -r '.data.project.revision')
  pandastudio project.add-lower-third \
    --id="$ID" --content="More" --atMs=10000 \
    --expectedRevision="$REV" --json
fi
```

Always check for `revision_conflict` after any save / add-* / remove-* / split-* call. The `details.expected` and `details.actual` tell you how far ahead the on-disk version is.

## 9. THE big one — full agentic edit + export

This is the v1.9.1 flagship workflow: agent gets a folder of source clips, returns a polished, exported MP4 ready to upload. End-to-end, headless, no human in the editor.

```bash
# Step 1: Create a project pre-loaded with the source clips. FFmpeg
# probes durations automatically.
P=$(pandastudio project.new \
  --name="Q4 Recap" \
  --withMedia='["/Users/me/Downloads/clip-a.mp4","/Users/me/Downloads/clip-b.mp4"]' \
  --json)
ID=$(echo "$P" | jq -r '.data.id')
echo "Project: $ID"

# Step 2: Transcribe every clip (Parakeet TDT, ~real-time)
JOB=$(pandastudio transcript.transcribe --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json | jq '.data.job.status'

# Step 3: Clean up the audio (DeepFilter denoising)
JOB=$(pandastudio audio.clean --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=600000 --json | jq '.data.job.status'

# Step 4: Auto-trim filler words and back-to-back repeats
pandastudio transcript.remove-fillers --id=$ID --json | jq '.data'
# → { fillersDeleted: 14, repeatsDeleted: 3, trimsAdded: 11 }

# Step 5: Surgical phrase deletion — find and drop a flubbed take
WORDS=$(pandastudio transcript.search --id=$ID --query="actually wait let me restart" --json \
  | jq -c '[.data.matches[].wordIds | .[]]')
[ "$WORDS" != "[]" ] && pandastudio transcript.delete-words --id=$ID --wordIds="$WORDS" --json

# Step 6: Find a "click here" moment and drop a zoom on it
ZOOM_AT=$(pandastudio transcript.search --id=$ID --query="click on the menu" --json \
  | jq -r '.data.matches[0].startMs // empty')
[ -n "$ZOOM_AT" ] && pandastudio project.add-zoom --id=$ID \
  --atMs=$ZOOM_AT --durationMs=2000 --depth=4

# Step 7: Render an intro title card and drop it at t=0
INTRO_JOB=$(pandastudio motion.generate \
  --templateId=title-card-vox \
  --slots='{"title":"Q4 in 90 seconds","subtitle":"the highlights"}' \
  --aspectRatio=16:9 --outputName=intro --json | jq -r '.data.jobId')
INTRO=$(pandastudio job.wait --id=$INTRO_JOB --timeoutMs=120000 --json)
pandastudio project.add-motion-graphic --id=$ID \
  --file="$(echo $INTRO | jq -r '.data.job.result.outputPath')" \
  --durationMs="$(echo $INTRO | jq -r '.data.job.result.durationMs')" \
  --atMs=0

# Step 8: Add a film-burn FX between the two source clips
CLIP_A_DUR=$(pandastudio project.read --id=$ID --json | jq '.data.project.mainTrack.clips[0].sourceDurationMs')
pandastudio project.add-fx --id=$ID --fxId=film-burn --atMs=$CLIP_A_DUR

# Step 9: Add captions in the bold-yellow MrBeast style
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-template --id=$ID --templateId=bold
pandastudio caption.set-style --id=$ID --highlightColor="#FFD60A" --positionY=85

# Step 10: Apply a cinematic preset
pandastudio project.set-style --id=$ID --padding=40 --shadowIntensity=30 \
  --borderRadius=20 --motionBlurAmount=15
pandastudio project.set-wallpaper --id=$ID --wallpaper=gradient-night

# Step 11: Show the user what's about to render (live preview window)
pandastudio window.preview --id=$ID
echo "Preview opened — give the user a moment to scrub."

# Step 12: EXPORT. Routes through the same Tier-3 renderer the UI uses
# (reusing an open editor on the project, or spawning a hidden one).
EXPORT_JOB=$(pandastudio export.start --id=$ID --quality=high --json \
  | jq -r '.data.jobId')
RESULT=$(pandastudio job.wait --id=$EXPORT_JOB --timeoutMs=900000 --json)
MP4=$(echo "$RESULT" | jq -r '.data.job.result.outputPath')
echo "✓ Exported: $MP4"

# Step 13: Generate YouTube metadata (local LLM, project-aware)
TITLE=$(pandastudio llm.generate-title --id=$ID --json | jq -r '.data.title')
DESC=$(pandastudio llm.generate-description --id=$ID --json | jq -r '.data.description')
TIMESTAMPS=$(pandastudio llm.generate-timestamps --id=$ID --json | jq '.data.timestamps')

cat <<EOF

🎬 Ready to upload to YouTube
Video:        $MP4
Title:        $TITLE
Description:  $DESC

Chapters:
$(echo "$TIMESTAMPS" | jq -r '.[] | "  \((.timeMs/1000) | floor | (./ 60 | floor) | tostring + ":" + (. % 60 | tostring | if length == 1 then "0" + . else . end))  \(.label)"' 2>/dev/null || echo "$TIMESTAMPS")
EOF
```

13 commands, fully unattended, produces a polished MP4 + ready-to-paste YouTube metadata. The agent and the user can also work the same project in parallel (revision-conflict-detected) — at any point, the user can open it in the editor and tweak.

## Pattern: degrade gracefully when license-gated

```bash
STATUS=$(pandastudio system.status --json)
GATED=$(echo "$STATUS" | jq -r '.data.license.automationGated')
if [ "$GATED" = "true" ]; then
  echo "PandaStudio's CLI is license-gated. Open the app, go to Settings → License, and activate. I can re-try once you're done."
  exit 1
fi
# … real work …
```

---

## Motion-graphics authoring — detailed recipes (moved from SKILL.md)

These long-form authoring recipes were moved out of SKILL.md to keep the
core playbook lean. Read this file on demand when you are authoring a
bespoke motion graphic with `motion.render-html` (see also
reference/motion-philosophy.md for the page contract + canonical shell).

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

# STEP 2: Render scenes — renders are serialized by a mutex, so only
# one motion.render-html runs at a time. Always use job.wait after each
# render before starting the next. RENDER_BUSY means a render is already
# in progress; retry after job.wait completes.
JOB1=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene1.html --durationMs=4000 --json | jq -r '.data.jobId')
JOB2=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene2.html --durationMs=3000 --json | jq -r '.data.jobId')
# Both running in parallel now. Wait on each result.
SCENE1=$(pandastudio job.wait --id="$JOB1" --json | jq -r '.data.job.result.outputPath')
SCENE2=$(pandastudio job.wait --id="$JOB2" --json | jq -r '.data.job.result.outputPath')
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

### Recipe: SaaS product promo / intro video (the "Teamble / Linear / Arc" style)

This is the single most common request ("make me a 60–90s product promo / intro
video"). It's a **learnable template**, not a freeform creative task — follow
the structure below and the output will land every time.

**Canonical length:** 60–90s total (Teamble reference is 98s, Linear/Arc promos
land ~60s). Never longer than 120s for a promo — viewers drop off.

**7-act structure** (timings scale proportionally to total length):

| Act | % of runtime | Purpose | Typical shot |
|---|---|---|---|
| 1. Hook | 0–5% | One-line positioning statement | Gradient typography card |
| 2. Brand | 5–8% | Logo lockup | Logo reveal with glow |
| 3. Problem | 8–20% | 2–3 giant word callouts | `word-pop` cards on gradient bg |
| 4. Product snippets | 20–55% | 6–10 UI cutaways, 2–4s each | Zoomed UI fragment + animated cursor + glow accent |
| 5. Act break | 55–60% | Chapter divider / "and here's how" | `chapter-divider` or pull-quote |
| 6. Benefit montage | 60–88% | Alternate typography callouts + UI snippets | Mix `word-pop` + `pull-quote` + UI zooms |
| 7. CTA / Logo | 88–100% | URL, logo return, tagline | `end-screen` or `channel-intro` |

**Pacing rule:** 2–4 seconds per shot. A 90s promo has **25–35 distinct shots**.
If you're building fewer than 20 shots, it will feel slow and linear. If more
than 40, it will feel frenetic — that's fine for Shorts, wrong for a 16:9 promo.

**Design language (SaaS-promo aesthetic):**
- Dark background (near-black) with **purple → magenta → pink gradients** and
  soft bokeh accents. NOT the default Panda green — pick `mkbhd` theme as the
  closest out-of-the-box match, OR author custom HTML for the full purple/pink
  look (example below).
- Bold sans-serif type (Inter / SF / GT America). White or gradient fill.
- Every UI shot has: a tilted framing, an animated cursor glyph on the
  important control, and an accent glow around the element.
- Sparkle / particle accents on reveal moments.
- **No voiceover**. Music + typography carry the story. If the user insists on
  VO, treat it as a different format (explainer, not promo) — the pacing rules
  change.

**Primitive mapping — which motion template does which job:**

| Shot role | Authoring primitive (from `motion-philosophy.md` §1.4) | Typical duration |
|---|---|---|
| Opening tagline | Kinetic-type opener — word-by-word reveal, chrome gradient, scale 1→8× on the hero word | 2500 ms |
| Logo reveal | Crystallize → wordmark (object shrinks / translates into logo position while wordmark fades up) | 2000 ms |
| Single-word callout ("Help", "10x better") | Camera dolly through type — text grows 1× → 8×, opacity fades at peak | 1500–2000 ms |
| Benefit statement | Chrome-gradient sweep across words, dark bookends to prevent edge-tear | 2500 ms |
| Metric / score reveal | Counter tween (number counts up 0 → final over 0.8s, power2.out) + scale entry | 2000 ms |
| Act break | Whip-streak transition + chapter label appearing from off-frame | 2000 ms |
| Focused UI element spotlight | Highlight ring (`box-shadow: 0 0 0 3px var(--accent), 0 0 20px var(--accent)`) + vignette breath | 2000 ms |
| Burst emphasis | Floating cluster drift + sparkle particles | 1200 ms |
| Outro / CTA | Held hero card 4–6s with shimmer-sweep on logo | 3000 ms |
| UI fragment (from screen recording) | `project.add-clip` + `project.add-zoom` into the relevant region | 2500–3500 ms |

Each row authors as its own HTML composition. Every composition starts
from the canonical shell in `reference/motion-philosophy.md` §7 (grid
+ vignette + grain + chrome-gradient heading styles + Law #11 anchor).
Run all renders in parallel, concat the sequence.

**PandaStudio limitation — honest note:** the editor does NOT render 3D
perspective tilts on screen-recording clips. The Teamble reference achieves
its "tilted UI card floating in space" via AfterEffects. In PandaStudio, you
have three workarounds:

1. **Zoom-in on UI (default):** use `project.add-zoom` to push into the
   relevant UI element. Reads similarly on screen, ships today.
2. **Custom HTML motion graphic with CSS `transform: rotate3d()`:** wrap a
   screenshot of the UI element in an HTML scene, apply 3D transform, render
   as a transparent WebM, overlay with `project.add-motion-graphic`. This
   genuinely matches the reference but requires authoring custom HTML per shot.
3. **Record the UI in an AE-style wrapper externally** and bring in as a clip.

If the user wants the 3D-tilted look and accepts longer build time, go with
(2). Otherwise (1) is the default.

**Default audio + captions for this recipe:**

- **Music:** one of the `kinetic-product-drive-*` tracks (bundled — they were
  generated specifically for this format). Volume `0.5–0.7` because there's no
  VO to duck under. Fade in 500 ms, fade out 1000 ms.
- **Captions:** NONE on a pure promo. Typography motion graphics carry the
  narrative. If the user wants captions (e.g., for LinkedIn autoplay-muted),
  use `panda-neon` positioned at `positionY=0.8` so it never overlaps the
  typography cards.
- **LUT on source UI clips:** `modernVibrant` at intensity 0.5 to match the
  saturated gradient aesthetic of the surrounding motion graphics.

**Shot-list executable template** (scale to your runtime):

Before writing any HTML, load [`reference/motion-philosophy.md`](reference/motion-philosophy.md)
and build from the canonical shell in §7. Every scene inherits the grid
+ vignette + grain background, chrome-gradient heading class, and the
Law #11 timeline anchor. Author each scene's HTML as its own file
(`/tmp/scene-hook.html`, `/tmp/scene-brand.html`, etc.), render them
**all in parallel**, then concat.

```bash
# ── Setup ─────────────────────────────────────────────────────────
P=$(pandastudio project.new --name="$PRODUCT Promo" --aspectRatio=16:9 --json)
ID=$(echo "$P" | jq -r '.data.id')

# Pick the kinetic product-drive track with durationMs closest to your target
MUSIC=$(pandastudio asset.list-music --json \
  | jq -r '.data.tracks[] | select(.intents | index("product_video")) | .absolutePath' \
  | head -1)

# ── Author every scene's HTML upfront (not shown — use the canonical ──
#    shell + the motion vocabulary from reference/motion-philosophy.md):
#
#    /tmp/scene-hook.html     — Act 1 hook (kinetic-type opener, 2.5s)
#    /tmp/scene-brand.html    — Act 2 logo reveal (crystallize→wordmark, 2s)
#    /tmp/scene-p1.html       — Act 3 single-word "Help"      (1.8s)
#    /tmp/scene-p2.html       — Act 3 single-word "10x better" (1.8s)
#    /tmp/scene-p3.html       — Act 3 single-word "faster feedback" (1.8s)
#    /tmp/scene-actbreak.html — Act 5 chapter divider w/ whip streak (2s)
#    /tmp/scene-benefit.html  — Act 6 chrome-sweep pull-quote (2.5s)
#    /tmp/scene-stat.html     — Act 6 stat reveal w/ counter tween (2s)
#    /tmp/scene-outro.html    — Act 7 held hero CTA card (3s, min 4s hold)
#
# Each file starts from the canonical shell in §7 of motion-philosophy.md.
# Load philosophy → pick primitives from §1.4 → compose → save.

# ── Fire every render in parallel ────────────────────────────────
# Since v1.17 each motion.render-html call spawns its own Chromium
# process. 3+ concurrent renders finish in the time of one (~5× speedup).
# Collect ALL jobIds first, THEN job.wait each — no job.wait between fires.
HOOK_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-hook.html     --durationMs=2500 --json | jq -r '.data.jobId')
BRAND_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-brand.html    --durationMs=2000 --json | jq -r '.data.jobId')
P1_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-p1.html       --durationMs=1800 --json | jq -r '.data.jobId')
P2_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-p2.html       --durationMs=1800 --json | jq -r '.data.jobId')
P3_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-p3.html       --durationMs=1800 --json | jq -r '.data.jobId')
ACTBREAK_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-actbreak.html --durationMs=2000 --json | jq -r '.data.jobId')
BENEFIT_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-benefit.html  --durationMs=2500 --json | jq -r '.data.jobId')
STAT_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-stat.html     --durationMs=2000 --json | jq -r '.data.jobId')
OUTRO_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/scene-outro.html    --durationMs=3000 --json | jq -r '.data.jobId')

# All nine renders are now racing in parallel. Collect results.
HOOK=$(pandastudio     job.wait --id=$HOOK_JOB     --json | jq -r '.data.job.result.outputPath')
BRAND=$(pandastudio    job.wait --id=$BRAND_JOB    --json | jq -r '.data.job.result.outputPath')
P1=$(pandastudio       job.wait --id=$P1_JOB       --json | jq -r '.data.job.result.outputPath')
P2=$(pandastudio       job.wait --id=$P2_JOB       --json | jq -r '.data.job.result.outputPath')
P3=$(pandastudio       job.wait --id=$P3_JOB       --json | jq -r '.data.job.result.outputPath')
ACTBREAK=$(pandastudio job.wait --id=$ACTBREAK_JOB --json | jq -r '.data.job.result.outputPath')
BENEFIT1=$(pandastudio job.wait --id=$BENEFIT_JOB  --json | jq -r '.data.job.result.outputPath')
STAT=$(pandastudio     job.wait --id=$STAT_JOB     --json | jq -r '.data.job.result.outputPath')
OUTRO=$(pandastudio    job.wait --id=$OUTRO_JOB    --json | jq -r '.data.job.result.outputPath')

# ── Act 4: Product snippets (screen recordings + zooms) ──────────
# The user provides a full-UI screen recording as $UI_REC.
# Instead of one long clip, split into 6–10 short 2.5–3.5s beats,
# zooming into a different UI element on each beat.
pandastudio project.add-clip --id=$ID --media="$UI_REC" --json
CLIP_ID=$(pandastudio project.read --id=$ID --json \
  | jq -r '.data.project.mainTrack.clips[-1].id')

# Apply LUT for the saturated gradient aesthetic
pandastudio project.set-clip-lut --id=$ID --clipId="$CLIP_ID" \
  --lutPreset=modernVibrant --lutIntensity=0.5 --json

# Stack zooms on each UI beat (durations match the narrative)
pandastudio project.add-zoom --id=$ID --clipId="$CLIP_ID" \
  --startMs=0 --endMs=2500 --targetX=0.45 --targetY=0.50 --zoom=1.8 --json
pandastudio project.add-zoom --id=$ID --clipId="$CLIP_ID" \
  --startMs=2500 --endMs=5500 --targetX=0.72 --targetY=0.35 --zoom=2.2 --json
# ... repeat for 6–10 zooms total, rotating focus across UI regions

# ── Concat the pre-UI segments and the post-UI segments ──────────
# Rule from the concat hard-rule above: motion segments MUST be merged
# into single MP4s before hitting the timeline, else audio/transitions break.
INTRO=$(pandastudio motion.concat \
  --clips="[$HOOK,$BRAND,${P3[0]},${P3[1]},${P3[2]}]" \
  --outputName=promo-intro --json | jq -r '.data.outputPath')

OUTRO_BUNDLE=$(pandastudio motion.concat \
  --clips="[$ACTBREAK,$BENEFIT1,$STAT,$OUTRO]" \
  --outputName=promo-outro --json | jq -r '.data.outputPath')

# ── Timeline assembly ────────────────────────────────────────────
# Order: intro motion → UI clip (with zooms) → outro motion.
# Prepend intro, append outro (project.add-clip in order).
# Note: this needs a project that currently has ONLY the UI clip —
# if you're starting fresh, create the project with media=INTRO
# so it's first on the timeline, then add the UI clip, then the outro.
pandastudio project.add-clip --id=$ID --media="$OUTRO_BUNDLE" --json

# ── Music ────────────────────────────────────────────────────────
pandastudio project.add-audio --id=$ID --audioPath="$MUSIC" \
  --volume=0.6 --fadeIn=500 --fadeOut=1000 --json

# ── Preview, then export ─────────────────────────────────────────
pandastudio preview.show --id=$ID
# Watch the full preview. Adjust zoom targets if UI beats feel off.
pandastudio export.start --id=$ID --quality=high --json
```

**Common mistakes agents make on promos (avoid these):**

- **One long UI clip with no zooms.** Screen recording for 40s straight reads
  as "tutorial," not "promo." You MUST break the UI section into 6–10
  zoom-driven beats of 2–4s each.
- **Typography cards longer than 3 seconds.** Viewers read a 3-word title in
  500 ms; anything over 2500 ms feels dead. Keep motion-graphic shots tight.
- **Voiceover on a promo.** This format is music + typography. If VO is
  required, change format (explainer ≠ promo — different pacing, different
  music volume, different caption strategy).
- **Default Panda green theme on a SaaS product promo.** It reads as tutorial/
  educational, not product-marketing. Use `mkbhd` or author custom HTML with
  the product's actual brand palette.
- **Skipping `motion.concat` between motion-graphic scenes.** Revisit the
  hard-rule at the top of the promo section — always merge before adding to
  timeline.
- **Forgetting the CTA/outro.** A promo without a CTA is a trailer. Always
  include Act 7 with a URL, brand lockup, or "Try it today" line.

**When the user wants the full Teamble look (custom purple-gradient HTML):**

If the default `mkbhd` theme isn't enough and the user wants the exact
gradient-bokeh aesthetic, author a reusable HTML template shell:

```html
<!-- saas-promo-word.html — gradient bg + word-pop, fully themeable via slots -->
<!DOCTYPE html><html><head><style>
  body { margin:0; width:1920px; height:1080px;
    background: radial-gradient(ellipse at 30% 40%, #3b1566 0%, #0a0a14 60%);
    font-family: 'Inter', sans-serif; color: #fff;
    display: flex; align-items: center; justify-content: center; }
  .bokeh { position:absolute; border-radius:50%; filter:blur(80px); opacity:0.6; }
  .b1 { left:-100px; bottom:-100px; width:500px; height:500px; background:#e53fc5; }
  .word { font-size:180px; font-weight:800; letter-spacing:-4px;
    opacity:0; transform:translateY(30px);
    animation: rise 500ms cubic-bezier(0.16, 1, 0.3, 1) 100ms forwards; }
  @keyframes rise { to { opacity:1; transform:translateY(0); } }
</style></head><body>
  <div class="bokeh b1"></div>
  <div class="word">{{WORD}}</div>
</body></html>
```

Render each word-pop variant with `motion.render-html`, passing a different
`{{WORD}}` slot substitution per scene, and concat them. This gives the
authentic SaaS-promo look but costs ~30 min of HTML tuning upfront vs. using
the stock `14-word-pop` template.

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

### Default image presentation — tilted "floating device" look

<HARD-RULE>
Any `<img>` inside a `motion.render-html` scene MUST be wrapped in the
tilted floating-device presentation below by default. Motion-graphic scenes
are stylized contexts by nature — a flat, axis-aligned image reads as
unpolished and out of style against the typography scenes around it.

The only exception: if the user explicitly says "flat image" / "no tilt" /
"axis-aligned," render without the wrapper. Never silently downgrade to flat.
</HARD-RULE>

The presentation has four ingredients, each doing a specific job:

1. **3D perspective tilt** — makes the image feel like a floating device, not a screenshot
2. **Soft drop shadow** — anchors it in the gradient background instead of floating flat
3. **Edge glow** — accent color (pink/purple/brand) around the frame, reinforces the palette
4. **Optional animated cursor** — purple gradient cursor glyph on the important UI element

Drop this CSS block into every HTML scene that contains an image. The
`.stage-tilt` class is the only thing you need to apply to the wrapper —
everything else auto-styles from there.

```html
<!DOCTYPE html>
<html><head><style>
  :root {
    --tilt-accent: #c44eff;      /* edge glow — override per brand */
    --tilt-accent-2: #ff4ea3;    /* cursor fill */
    --tilt-bg-from: #3b1566;     /* radial gradient inner */
    --tilt-bg-to: #0a0a14;       /* radial gradient outer */
  }
  body {
    margin: 0; width: 1920px; height: 1080px;
    background: radial-gradient(ellipse at 30% 40%,
      var(--tilt-bg-from) 0%, var(--tilt-bg-to) 60%);
    display: flex; align-items: center; justify-content: center;
    perspective: 1600px;          /* REQUIRED for rotate3d to render depth */
  }
  /* Wrap any <img> (or a <div> with a background-image) in .stage-tilt */
  .stage-tilt {
    transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg);
    transform-origin: center center;
    border-radius: 18px;
    overflow: hidden;
    box-shadow:
      0 40px 80px -20px rgba(0, 0, 0, 0.55),       /* drop shadow */
      0 0 0 1px rgba(255, 255, 255, 0.06),          /* hairline */
      0 0 60px 8px color-mix(in srgb, var(--tilt-accent) 30%, transparent);
    animation: tilt-rise 700ms cubic-bezier(0.16, 1, 0.3, 1) both;
  }
  .stage-tilt img, .stage-tilt video {
    display: block; width: 100%; height: auto;
  }
  @keyframes tilt-rise {
    from { opacity: 0; transform: rotate3d(1, 0.25, 0, 22deg) rotateZ(-2deg) translateY(40px); }
    to   { opacity: 1; }
  }
  /* Optional: animated cursor glyph — absolute-position it over a UI control */
  .tilt-cursor {
    position: absolute; width: 64px; height: 64px; pointer-events: none;
    filter: drop-shadow(0 6px 18px color-mix(in srgb, var(--tilt-accent-2) 55%, transparent));
    animation: cursor-tap 1400ms ease-in-out infinite;
  }
  @keyframes cursor-tap {
    0%, 100% { transform: translate(0, 0) scale(1); }
    50%      { transform: translate(-4px, -4px) scale(0.92); }
  }
</style></head>
<body>
  <div class="stage-tilt" style="width: 900px;">
    <img src="product-screenshot.png" />
  </div>
  <!-- Optional: drop a cursor glyph SVG into .tilt-cursor at an exact x/y -->
</body></html>
```

**Override knobs** (pass through `--slots` or edit the CSS vars):

| Variable | Default | When to change |
|---|---|---|
| `--tilt-accent` | `#c44eff` (purple) | Match the product's brand color |
| `--tilt-accent-2` | `#ff4ea3` (pink) | Cursor glow; usually accent's sibling hue |
| `--tilt-bg-from` / `--tilt-bg-to` | purple → near-black | For light-theme promos, swap to off-white gradient |
| `rotate3d` angle | `16deg` | Reduce to `8deg` for subtler tilt; increase to `22deg` for more dramatic |
| `rotateZ` | `-1deg` | Flip sign to tilt the other way — rotate between scenes for variety |

**When NOT to use this wrapper** (narrow list):

- User explicitly asked for a flat/axis-aligned image
- The image IS the entire frame (a full-bleed photo with no UI) — tilting a photo looks weird; apply only to UI, product cards, screenshots
- The scene is a `07-chapter-divider` or text-only card — tilt is for images, not type

**Rotation variety across a multi-scene promo:** when you have 3+ image
scenes back-to-back, alternate the Z rotation between `-1deg` and `+1deg`
and vary the X rotation between `12deg`, `16deg`, and `20deg` across
scenes. A uniform tilt angle across every scene reads as templated; small
variation reads as crafted.

### No scene is static — always-on motion rule

<HARD-RULE>
**Every `motion.render-html` scene MUST have motion for its full duration.**
A scene that holds a still image for 3 seconds reads as a slideshow and
breaks the "motion graphics" illusion — viewers perceive it as a freeze.
If the content itself doesn't move (typography card, single product shot,
logo reveal), add ambient motion. No exceptions for promo/intro/outro
formats.
</HARD-RULE>

Three ways to satisfy the rule. Pick the cheapest that fits the shot:

**1. Subtle Ken Burns on any image (preferred for UI/product shots).**
A slow pan + zoom on the `.stage-tilt` wrapper — imperceptible per-frame
but unmistakably alive over 3 seconds. Runs in parallel with `tilt-rise`.

```css
.stage-tilt {
	/* entrance (from the earlier rule) keeps running */
	animation:
		tilt-rise 700ms cubic-bezier(0.16, 1, 0.3, 1) both,
		ken-burns 6s ease-in-out 500ms both;
}
@keyframes ken-burns {
	0%   { transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg) scale(1)   translate(0, 0); }
	100% { transform: rotate3d(1, 0.25, 0, 16deg) rotateZ(-1deg) scale(1.06) translate(-14px, -6px); }
}
```

Keep the motion deliberately tiny — `scale(1.06)` and ±14px over 6s is
invisible as a single step but feels cinematic over the shot. Alternate
direction per scene (left-up → right-down → left-down → right-up) for
variety on multi-shot sequences.

**2. Animated gradient / shine sweep (preferred for typography + logo).**
Typography cards shouldn't pan; they should shimmer. Animate the gradient
behind the text or sweep a highlight across it.

```css
.bg-shimmer {
	background: linear-gradient(120deg, #3b1566 0%, #0a0a14 40%, #3b1566 80%, #0a0a14 100%);
	background-size: 220% 100%;
	animation: shimmer 8s ease-in-out infinite;
}
@keyframes shimmer {
	0%, 100% { background-position: 0% 0%; }
	50%      { background-position: 100% 0%; }
}

/* Or a shine sweep over a word */
.shine {
	background: linear-gradient(90deg, transparent 20%, rgba(255,255,255,0.4) 50%, transparent 80%);
	background-size: 220% 100%;
	-webkit-background-clip: text;
	animation: sweep 2200ms ease-in-out infinite;
}
@keyframes sweep {
	0%   { background-position: 100% 0%; }
	100% { background-position: -100% 0%; }
}
```

**3. Bokeh / particle drift (preferred for ambient pads + CTA outros).**
Give bokeh light-blurs a slow drift. Almost free computationally and
always-on.

```css
.bokeh {
	animation: drift 14s ease-in-out infinite alternate;
}
@keyframes drift {
	0%   { transform: translate(0, 0) scale(1); }
	100% { transform: translate(30px, -20px) scale(1.15); }
}
```

**Rule-of-thumb mapping for the 7-act promo recipe:**

| Act / shot type | Default always-on motion |
|---|---|
| Hook (typography) | Shimmer gradient + bokeh drift |
| Brand / logo reveal | Bokeh drift + logo subtle scale 1 → 1.02 loop |
| Word-pop callouts | Shine sweep across the word; bg shimmer |
| UI / product screenshot | Ken Burns on `.stage-tilt` + bokeh drift |
| Pull-quote / benefit | Shimmer gradient |
| Metric / stat reveal | Counter count-up + bokeh drift |
| Chapter divider | Shine sweep + bokeh drift |
| CTA / end screen | Bokeh drift + CTA button pulse (scale 1 → 1.03) |

### Multi-image scenes — never identical, always varied

When a single scene contains **multiple images** (product gallery,
feature grid, testimonial wall, before-after, 3-up comparison), a uniform
layout with identical tilts reads as a wireframe. The Teamble-style
reference gets variety by:

- Rotating Z-axis tilt direction per item: `-2deg`, `+3deg`, `-1deg`, `+4deg`
- Mixing X-axis tilts: shallower (`8-12deg`) for items in the foreground,
  steeper (`18-24deg`) for items in the back, so depth is implied
- Staggering entrance: each image uses `animation-delay: 0ms, 150ms, 300ms, 450ms`
  — the scene "builds" rather than appearing all at once
- Varying scale: lead image at `scale(1)`, secondary at `scale(0.85)`, tertiary
  at `scale(0.7)` — implies focus + hierarchy

```css
.stage-tilt:nth-child(1) { transform: rotate3d(1, 0.25, 0, 12deg) rotateZ(-2deg) scale(1);    z-index: 3; animation-delay: 0ms; }
.stage-tilt:nth-child(2) { transform: rotate3d(1, 0.25, 0, 18deg) rotateZ( 3deg) scale(0.85); z-index: 2; animation-delay: 150ms; }
.stage-tilt:nth-child(3) { transform: rotate3d(1, 0.25, 0, 22deg) rotateZ(-1deg) scale(0.7);  z-index: 1; animation-delay: 300ms; }
```

**Placement patterns to choose from** (pick one per scene, don't mix):

- **Stacked cascade** — images offset diagonally, overlapping 15–20% each
- **Orbital** — one hero image center, supporting ones arranged around it
  (cf. Teamble's "Onboarding Survey" orbital shot)
- **Scattered** — images at random(ish) angles across the frame with
  generous negative space, each at a slightly different Z tilt
- **Stack-and-fan** — a focused hero + a fan of thumbnails behind it
- **Line-up** — 3 items in a horizontal row with descending Z-axis tilt

**Anti-patterns (do NOT do these):**

- Three images in a neat horizontal grid with identical tilts → looks like
  a product-listing page, not a promo
- Every image entering at the same moment → no rhythm
- Identical rotation signs (`rotateZ(-1deg)` on all) → feels machine-placed

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

### Pre-flight EVERY motion graphic with `motion.screenshot` — mandatory

**This is the single highest-leverage performance habit you can adopt.**

Full renders take 10–60 seconds depending on length and complexity. A
single `motion.screenshot` frame is ~2 seconds. If your composition has
a typo, missing CSS class, broken chrome-gradient, or wrong font weight,
catching it on a 2-second screenshot saves a 30–60 second re-render.
Across a 6-render multi-scene promo, the savings compound.

**Required workflow before EVERY motion.render-html call:**

```bash
# 1. Author /tmp/scene-1.html.
# 2. Screenshot at t=0.5s (just past the entrance) AND at the hero
#    moment (where the chrome-gradient should be lit, where the text
#    should have settled, etc.). Two ~2-second calls.
pandastudio motion.screenshot \
  --htmlPath=/tmp/scene-1.html --aspectRatio=16:9 --atMs=500 \
  --outputName=scene-1-t500 --json
pandastudio motion.screenshot \
  --htmlPath=/tmp/scene-1.html --aspectRatio=16:9 --atMs=2000 \
  --outputName=scene-1-t2000 --json

# 3. READ both PNGs (multimodal). Confirm:
#    - Fonts are loading (not Times New Roman fallback)
#    - Chrome gradient is rendering (not flat white text)
#    - Layout fits the canvas (no overflow / cropping)
#    - Per-word stagger is producing the right spans
#    If anything looks wrong, fix the HTML and re-shoot — at 2s each
#    you can iterate ~10 times in the time of one full render.

# 4. Only AFTER both screenshots verify, call motion.render-html.
pandastudio motion.render-html \
  --htmlPath=/tmp/scene-1.html --durationMs=4000 \
  --aspectRatio=16:9 --outputName=scene-1 --json
```

**Skip pre-flight only when:** you're producing a new variant of a
template you've already verified working. New compositions = always
pre-flight.

`motion.screenshot` returns `{ outputPath }` directly — no `job.wait`
needed.

### Frame-verify before declaring done — `motion.verify-frames`

`motion.screenshot` validates layout on a STILL composition. But a motion
graphic is only "done" when you've watched its rendered MP4 land at hero
moments — has the caption landed on the right word? Did the face transition
interpolate smoothly? Does the color-recolor beat actually flip? Lint
passing and preview snapshots won't catch a face that snapped or a caption
that drifted one word off.

**Contract:** after `motion.render-html` (or `export.start` for a full
project), extract 8–15 frames at hero timestamps and READ each PNG
(multimodal) before you declare the piece shippable. This is the
"lint passing ≠ design working" rule from
[`reference/motion-philosophy.md`](reference/motion-philosophy.md) §4.

```bash
# After a render lands, extract frames at hero moments
pandastudio motion.verify-frames \
  --videoPath=/path/to/output.mp4 \
  --timestamps='[0.5,1.5,3.0,5.0,7.5,10.0,12.5,15.0]' \
  --json

# Or from an export-library entry
pandastudio motion.verify-frames \
  --entryId=$ENTRY_ID \
  --timestamps='[0.5,1.5,3.0,5.0,7.5,10.0,12.5,15.0]' \
  --json
```

Returns `{ frames: [{ timestampSeconds, path }...], nextStep: "..." }`.
**Read each `path` as an image** and verify per the checklist in
[`reference/video-authoring.md`](reference/video-authoring.md) §5:

- No cropped faces / text overflow / blank frames
- Caption lands on the right word at each captured moment
- Face mode transitions (Mode A) look smooth, not snapped
- No MG covers the host face area (Mode C) or the screen zone (Mode B)
- Color palette looks right (no accidental flat-white text)
- Backgrounds have the grid + vignette + grain texture

If any frame fails, iterate the motion graphic and re-run
`motion.verify-frames` before `export.start`. **No shipping un-looked-at
work.**

### Assembling multi-scene motion graphics — always use motion.concat

**This is the mandatory last step whenever you render more than one scene.**
Render every scene **sequentially** (renders are serialized by a mutex —
only one motion.render-html runs at a time; always call job.wait between
renders), then join them into one final MP4 with `motion.concat` before
touching the project timeline. The concat is a lossless stream copy — no
re-encode, completes in < 1 second.

**Renders are serialized by a mutex — only one motion.render-html runs
at a time.** Always use job.wait after each render before starting the
next. RENDER_BUSY means a render is already in progress; retry after
job.wait completes.

```bash
# 1. Fire every scene in parallel. Collect jobIds first, then job.wait
#    each — the second fire doesn't block on the first.
INTRO_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/intro.html --durationMs=3000 \
  --outputName=scene-intro --json | jq -r '.data.jobId')

PRODUCT_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/product.html --durationMs=5000 \
  --assets=/Users/me/product1.png,/Users/me/product2.png \
  --outputName=scene-product --json | jq -r '.data.jobId')

CTA_JOB=$(pandastudio motion.render-html \
  --htmlPath=/tmp/cta.html --durationMs=2000 \
  --outputName=scene-cta --json | jq -r '.data.jobId')

# All three Chromium processes are now racing to finish. Collect results.
INTRO_PATH=$(pandastudio job.wait --id="$INTRO_JOB"   --json | jq -r '.data.job.result.outputPath')
PRODUCT_PATH=$(pandastudio job.wait --id="$PRODUCT_JOB" --json | jq -r '.data.job.result.outputPath')
CTA_PATH=$(pandastudio job.wait --id="$CTA_JOB"     --json | jq -r '.data.job.result.outputPath')

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

---

## Authored graphic recipe — logo / brand-card row (content-specific, not a UI template)

Use for "we use X, Y, Z", tool / partner / integration mentions — N rounded
white cards, each a logo, popping in over the host's lower third. This is NOT a
slot-fill template (the logos depend on what's being said); you author + render
it per use and adapt the cards. Stage the logo PNG/SVGs with `--assets` and
reference them by basename.

```html
<!doctype html><html><head><meta charset="utf-8" />
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/gsap.min.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; }
  /* TRANSPARENT — composites over the host. Cards sit in the lower band. */
  html, body { width: 100%; height: 100%; background: transparent; font-family: Inter, system-ui, sans-serif; }
  .stage { position: absolute; inset: 0; }
  .row { position: absolute; left: 0; right: 0; bottom: 8vh;
         display: flex; justify-content: center; gap: 3vw; }
  .card { width: 22vw; max-width: 360px; aspect-ratio: 16/10;
          background: #fff; border-radius: 1.4vw;
          box-shadow: 0 1.2vw 3vw rgba(0,0,0,.28);
          display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 1vw;
          will-change: transform, opacity; }
  .card img { height: 34%; object-fit: contain; }
  .card .label { font-size: 1.9vw; font-weight: 800; color: #08101f; }
</style></head>
<body>
  <!-- duration in SECONDS. Adapt the cards: one .card per logo. -->
  <div class="stage" data-composition-id="logo-row" data-width="1920" data-height="1080" data-duration="5">
    <div class="row">
      <div class="card"><img src="heygen.png"      alt="" /><span class="label">HeyGen</span></div>
      <div class="card"><img src="claude-code.png" alt="" /><span class="label">Claude Code</span></div>
      <div class="card"><img src="hyperframes.png" alt="" /><span class="label">Hyperframes</span></div>
    </div>
  </div>
  <script>
    const tl = gsap.timeline({ paused: true });
    // Cards pop in with a staggered back-ease; exit before the end.
    tl.from(".card", { y: "4vh", scale: 0.8, opacity: 0, duration: 0.5,
                       ease: "back.out(1.7)", stagger: 0.12 }, 0.1);
    tl.to(".card", { opacity: 0, y: "2vh", duration: 0.4, ease: "power2.in",
                     stagger: 0.06 }, 4.4);
    tl.to({}, { duration: 5 }, 0);           // Law #11 anchor — full duration
    window.__timelines = (window.__timelines || {});
    window.__timelines["logo-row"] = tl;     // key === data-composition-id
  </script>
</body></html>
```

```bash
# Render transparent (alpha), staging the logo images by basename:
JOB=$(pandastudio motion.render-html --htmlPath=/tmp/logo-row.html --transparent \
  --durationMs=5000 \
  --assets='["/path/heygen.png","/path/claude-code.png","/path/hyperframes.png"]' \
  --json | jq -r '.data.jobId')
pandastudio job.wait --id="$JOB" --json
# Add as an overlay at the moment the host names the tools (anchor to that word):
pandastudio project.add-motion-graphic --id=$ID --fromJob="$JOB" --durationMs=5000 \
  --atMs=<wordMs> --anchorSourceMs=<wordMs>
```

Adapt freely: 2–4 cards, swap logos for app icons or avatars, change the card
fill / accent, or move the row to a side rail. Pre-flight with
`motion.screenshot --htmlPath=… --atMs=1500` before the full render. The same
approach (transparent overlay + `--assets`) covers screenshot showcases and
icon callouts — author the HTML, render `--transparent`, add as an overlay.

---

## B-roll Ken-Burns shell (moved from SKILL.md)

Wrap a `media.generate-image` still in this shell so it has motion (Law #4 — camera never sleeps). Drop the absolute image path into `<<IMG_PATH>>`; duration is controlled by `--durationMs` on `motion.render-html`.

Drop the absolute file path of the generated image into `<<IMG_PATH>>`.
The shell handles aspect cropping (`object-fit: cover`), slow Ken-Burns
zoom, side vignette, and a subtle grain layer. Total duration is
controlled by `--durationMs` on `motion.render-html`.

```html
<!doctype html>
<html><head><style>
  html, body { margin: 0; height: 100%; background: #000; overflow: hidden; }
  .stage { position: relative; width: 100vw; height: 100vh; overflow: hidden; }
  .broll {
    position: absolute; inset: 0;
    background: url("file://<<IMG_PATH>>") center/cover no-repeat;
    transform-origin: 50% 50%;
    will-change: transform, filter;
    filter: saturate(1.05) contrast(1.04);
  }
  .vignette {
    position: absolute; inset: 0;
    background: radial-gradient(ellipse at center,
      rgba(0,0,0,0) 55%,
      rgba(0,0,0,0.35) 85%,
      rgba(0,0,0,0.65) 100%);
    pointer-events: none;
  }
  .grain {
    position: absolute; inset: 0;
    opacity: 0.06;
    mix-blend-mode: overlay;
    pointer-events: none;
    background-image:
      radial-gradient(rgba(255,255,255,0.5) 1px, transparent 1px),
      radial-gradient(rgba(255,255,255,0.4) 1px, transparent 1px);
    background-size: 3px 3px, 7px 7px;
    background-position: 0 0, 1px 2px;
  }
</style></head>
<body>
  <div class="stage">
    <div class="broll" id="broll"></div>
    <div class="vignette"></div>
    <div class="grain"></div>
  </div>
  <script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
  <script>
    // HyperFrames sets data-duration on <body>; default 4s if absent.
    const SLOT_DURATION = (document.body.dataset.duration | 0) || 4;
    gsap.registerPlugin();
    const tl = gsap.timeline({ paused: true });
    // Slow linear push-in — cinematic Ken-Burns. Pick ONE direction
    // randomly per render to avoid every B-roll feeling identical.
    const dir = Math.random();
    const startScale = 1.0, endScale = 1.08;
    const tx = dir < 0.5 ? -1.5 : 1.5; // % horizontal drift
    tl.fromTo("#broll",
      { scale: startScale, x: "0%" },
      { scale: endScale, x: tx + "%", duration: SLOT_DURATION, ease: "none" },
      0
    );
    // Subtle final-second darken so the cut out feels intentional.
    tl.to("#broll",
      { filter: "saturate(0.95) contrast(1.02) brightness(0.92)", duration: 0.6, ease: "power2.in" },
      SLOT_DURATION - 0.6
    );
    // No-op duration anchor — Law #11.
    tl.to({}, { duration: SLOT_DURATION }, 0);
    window.__timelines = window.__timelines || {};
    window.__timelines.broll = tl;
  </script>
</body></html>
```

