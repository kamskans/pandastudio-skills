<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# FX & transitions

## Effects (FX) & transitions — texture, mood, scene changes

Two different tools, and they have DIFFERENT rules about when you may add them.
**FX** are looping texture overlays composited over a clip (film grain, light
leaks, embers). **Transitions** are short overlays placed ON a cut to bridge two
clips (dip-to-black, flash, glitch). Discover both at runtime: `asset.list-fx`
and `asset.list-transitions`.

**The split that matters:**
- **Transitions** are an engaging-tier flourish: add them at real section
  boundaries when the brief is "make it engaging / cinematic / dynamic", and
  never on a plain "clean it up".
- **FX are EXPLICIT-REQUEST-ONLY.** Never add an FX overlay on your own — not on
  a plain edit, and **not even for "make it engaging".** An effect is a
  deliberate creative choice the user makes; you add one only when they ask for
  it by name. If you think one would help, *suggest* it in narration ("want a
  film burn between these two?") and wait for a yes. (See the "NOT part of the
  default pipeline" list in the editorial section.)

### The golden rule: restraint

FX and transitions are seasoning, not the meal. The failure mode here is the
*opposite* of motion graphics — where under-graphicking is the risk, here
**over-doing it is the risk.** A glitch on every cut and grain on every clip
reads as amateur, not produced. Rules:

- **Transitions: only at real scene/topic boundaries**, never on every cut. A
  tight talking-head edit has many silence/filler cuts a second apart — do NOT
  transition those. Place a transition where the *subject* changes (new chapter,
  location, a hard B-roll cut, intro→content, content→outro). Rule of thumb:
  **0 for a short clean clip; ~1 per major section** (a 6-section video → ~3–5).
  Only when the brief asks for an engaging/cinematic feel.
- **FX: only when the user explicitly asks for an effect** — then place exactly
  what they asked for, where they asked for it. Even on an explicit request,
  keep it restrained (1–3 per video, not a constant wash; if you can't name why
  a clip needs it, don't add it). You never decide on your own that a video
  "needs" an effect.
- When the user says **"clean it up" / "edit my video"** OR **"make it engaging
  / cinematic"** → add **no FX**. Transitions are fine on the "engaging" brief
  (section breaks only). Narrate whatever you added so they can dial it back.

### Transitions — `project.add-transition`

Places a full-frame transition overlay **centered on a cut** (the overlay goes
opaque at its midpoint so it masks the join). Pass `atMs` = the cut time between
two clips — read clip boundaries from `project.read`. `--durationMs` defaults to
1000 (the native length). Discover ids with `asset.list-transitions`.

```bash
pandastudio project.add-transition --id=$PROJECT \
  --transitionId=fade-black --atMs=42000 --json   # MCP: project_add_transition
```

| Id | Look | When to reach for it |
|---|---|---|
| `fade-black` | dip to black | The safe, classic scene break — a beat of black between two sections. Time-passing, chapter change. |
| `fade-white` | dip to white | Brighter, optimistic version of the dip — reveals, upbeat pivots, product shots. |
| `flash` | quick warm-white pop | A snappy hit on a hard beat / energetic cut — montage, fast pivots. Punchy, brief. |
| `light-sweep` | bright bar wipes across | A clean directional wipe — moving to a new location/topic with momentum. |
| `film-burn` | organic fire bloom | A warm, filmic, vintage scene change — storytelling/cinematic pieces. |
| `glitch` | digital RGB tearing | Tech/edgy/energetic content — a deliberately abrupt, modern cut. |

> Picking by vibe: clean/corporate → `fade-black`/`fade-white`; energetic/social
> → `flash`/`glitch`; cinematic/story → `film-burn`/`light-sweep`. Pick ONE
> transition style and reuse it across the video's section breaks — consistency
> reads as designed; a different transition on every cut reads as a demo reel.

### FX overlays — `project.add-fx`

A looping texture overlay over a clip span (screen/lighten/add/normal blend).
`--speed=0.25–4` scales the loop speed (default 1). Discover ids + defaults with
`asset.list-fx`. The 13 bundled effects, grouped by what they're FOR:

- **Film / vintage texture** — `film-grain` (subtle moving grain — the all-purpose "filmic" wash), `dust-scratches` (old-film dust + scratches), `film-burn` (warm organic burn — also a transition), `vhs-static` (retro VHS noise/tracking).
- **Light & lens** — `light-leak` (soft drifting light leak), `light-flare` (warm flare bloom), `lens-flare-sweep` (anamorphic horizontal flare), `light-streaks` (streaking light), `prism-leak` (chromatic prism leak). Set a warm/cinematic or dreamy tone.
- **Atmosphere / particles** — `bokeh-drift` (soft drifting bokeh orbs), `embers` (rising warm embers — fire/energy/hero), `snow-drift` (falling snow — seasonal/calm).
- **Punctuation** — `film-flash` (a 2–4 frame camera-shutter pop; use it ON a hard beat or to accent a reveal, NOT as a continuous wash — set its span short, ~0.3s).

```bash
pandastudio project.add-fx --id=$PROJECT \
  --fxId=film-grain --startMs=0 --endMs=15000 --speed=1 --json   # MCP: project_add_fx
```

> FX defaults (blend mode, opacity, native speed, default SFX) come from the
> manifest — `asset.list-fx` returns them; pass only what you want to override.
> `--speed` is most useful on `film-burn` (bump to 1.5–2 for a faster churn) and
> the particle FX. Keep opacity restrained — a texture you *notice* is usually
> too strong.

