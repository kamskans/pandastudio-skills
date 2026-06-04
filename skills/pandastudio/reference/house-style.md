# House style — neutral defaults when the brand kit is partial or absent

> **When to load this file.** Open it only when at least one field in the
> brand kit (from the editor-context snapshot) is missing AND the user has
> said "just pick something" or "use a default." If the brand kit is complete,
> motion-philosophy + the brand kit alone are enough.
>
> **What this file is NOT.** It is not a single house aesthetic you apply to
> every job. It's three named starting points to choose between. Picking the
> wrong one is the single most common failure (a YouTube tutorial getting a
> moody black title card when it wanted bright and bold).

## Voice → motion defaults

The brand kit's `voice` field is the highest-leverage signal. Use this table
when you have the voice but not the full visual identity:

| Voice | Energy | Typical canvas | Typical type treatment | Pacing |
|---|---|---|---|---|
| **minimal** | Low / spacious | Off-white or very dark, generous negative space | Display sans, restrained scale ramps | Slow end of each range |
| **bold** | High / decisive | Saturated single brand color or near-black | Heavy display sans, oversized, high-contrast | Fast end of each range |
| **editorial** | Medium / magazine-like | Off-white or paper-cream tones | Mix display serif + body sans, asymmetric layouts | Medium, with emphasis sweeps |
| **casual** | Medium / friendly | Soft pastels or muted brand color, rounded shapes | Geometric sans, slight rounded corners | Medium, slightly bouncy easings |
| **corporate** | Low / steady | Calm tinted solid (deep blue, slate, off-white) | Clean grotesque, conservative weights | Slow, smooth power-eases |
| **playful** | High / kinetic | Multi-color or vibrant gradient bg | Display sans + accent color emphasis | Fast, lots of micro-motion |

These are STARTING points. If the brand says `colors.background: #FFF8E7` but
the voice is `bold`, use `#FFF8E7` — the brand wins.

## Named tracks (pick one if the brief is open)

When the brief is open ("make me something for the intro") AND no brand colors
are set, offer the user a choice between these three named tracks. Don't
default silently to any of them — pick the one that matches the content type.

### Track A — Creator-bright (YouTube creator / tutorial)

The look: bold, bright, instantly legible. Made for fast YouTube content.

- **Canvas:** a full-bleed saturated brand hue (or a bright off-white if no
  brand color is set, fallback `#FFF8E7`). Edge to edge — minimal background
  texture. Negative space is generous but the canvas itself is loud.
- **Type:** heavy display sans (Inter ExtraBold, Geist Black, Plus Jakarta
  Sans Bold), oversized. One dominant brand hue + a near-black ink. Headlines
  120px+ at 1920×1080. **No** chrome gradients. **No** halo glow. Color
  carries the energy; type doesn't need a "premium" finish.
- **Transitions:** snappy pops, slides, grid/pixelate wipes. **No** motion-blur
  whip-streaks (those are Track B). Frames want to feel decisive, not cinematic.
- **Unifying spine:** consistent type system + color-block language + a
  repeated motion signature. **No** perspective grid, vignette, or grain
  overlay — those darken the frame and fight the canvas.

Fallback palette when the user has no brand color and picks Track A:

```
primary    #7C3AED  (or #16BDF0, #FF6B35, #34B27B — pick one and STICK)
ink        #0A101F
background #FFF8E7  (or use primary as full-bleed bg, with ink for type)
```

### Track B — Dark-premium (branded promo / launch / hero)

The look: cinematic, restrained, light-not-color. The aesthetic HyperFrames
itself ships in its launch demos.

- **Canvas:** black or near-black (`#0A0A0A`–`#111111`), ~90% negative space.
- **Type:** chrome-gradient headlines, halo glow on emphasis words, word-by-
  word kinetic reveals that scale rather than fade.
- **Transitions:** motion-blur whip-streaks, light-streak wipes, mask reveals.
- **Unifying spine:** perspective grid + crosshair markers + soft vignette
  + subtle grain on every scene. The grid is THE recognizable thread.
- **Backgrounds layered, bottom up:** dark gradient base → perspective grid at
  ~6% opacity → vignette → grain at ~3% opacity.

Fallback palette when the user picks Track B without a brand color:

```
primary    #FFFFFF + #B8C5D6 chrome gradient
accent     a single saturated hue derived from content (cyan, magenta, gold)
ink        #FFFFFF (with halo on emphasis words)
background #0A0A0A
```

### Track C — Product-clean (calm announcement / feature explainer)

The look: spacious, precise, understated — Linear / Vercel / Apple keynote
aesthetic.

- **Canvas:** calm tinted solid (deep blue, slate) or a soft off-white
  (`#F5F5F0`). Generous breathing room.
- **Type:** clean restrained sans (Inter, Geist), conservative weights
  (Regular / Medium / Semibold — no Black). No gradients. No glow.
- **Transitions:** clean directional slides, content-aware masks, soft cross-
  fades with motion. Quick, never showy.
- **Unifying spine:** a fixed spatial grid + consistent easing + restraint.
  Every scene shares the same margins, the same type scale, the same eases.

Fallback palette when the user picks Track C without a brand color:

```
primary    #0B5FFF  (or muted brand hue if available)
accent     a complementary muted hue
ink        #0D1218
background #F5F5F0  (light track) or #0A1422 (dark track)
```

## Asking the user to pick

When you need to offer a choice, the prompt looks like:

> Three quick options — which fits the vibe? Reply 1, 2, or 3:
>
> 1. **Bright creator** (full-bleed brand color, bold type, snappy cuts) — for
>    YouTube tutorials and explainer content
> 2. **Dark premium** (black canvas, chrome type, motion-blurred reveals) — for
>    branded promos and product launches
> 3. **Product clean** (calm tinted bg, restrained type, soft slides) — for
>    feature announcements and keynote-style content

Pick the closest one based on the brief; only ask if it's genuinely ambiguous.

## Voice-only authoring (no brand colors, no track explicitly picked)

If the user said "just pick something" or you're authoring under time pressure
with only the voice known, use this fallback decision tree:

- `voice: bold` or `playful` → Track A (creator-bright)
- `voice: minimal` or `corporate` → Track C (product-clean)
- `voice: editorial` → Track C with serif display + asymmetric layouts
- `voice: casual` → Track A with friendlier color (warm orange, muted pink)
- No voice set → ASK before authoring. Don't silently pick.

## Fonts when none is specified

Fallback display fonts (in this order — first one available wins):

```
Bricolage Grotesque, Plus Jakarta Sans, Geist, Inter, system-ui, sans-serif
```

Fallback body fonts:

```
Inter, Geist, IBM Plex Sans, system-ui, sans-serif
```

## Colors to actively avoid as defaults

- **PandaStudio's editor green / cyan** (`#34B27B`, `#16BDF0`) — these are OUR
  product UI colors, not the user's brand. Never pick them as the agent's
  default.
- **Generic Tailwind blue / Bootstrap blue** (`#3B82F6`, `#0D6EFD`) — read as
  "default web app," not "designed."
- **Pure red on pure black** — H.264 compresses red badly on dark backgrounds.
- **Linear gradients across a full dark frame** — banding artifacts. Use radial
  gradients or a solid + localized glow instead.

## Lazy defaults to question

If you find yourself reaching for any of these, stop and confirm:

- `#000000` background — almost always Track B speaking; if the brief isn't
  branded promo, you're probably on the wrong track.
- `Inter Bold 96px` everywhere — type as a single move; you've stopped
  designing.
- `power2.out` on every tween — use 3+ eases per scene.
- One-color palette with no accent — Track A wants a brand hue but also an ink
  contrast; pure monochrome reads as "unfinished."
- `data-duration="3"` for every scene — pacing should respond to the brief, not
  default.

---

## Reminder

Everything in this file is a **fallback**. The moment the user gives you a
brand color, font, or voice — switch to those. The brand wins. Always.
