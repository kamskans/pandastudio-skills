# Easing dictionary — by purpose

> Load when you're picking eases for a scene and want more than `power2.out` on
> everything. Easing is the *adverb* of motion: the same move with a different
> ease reads as a different intention. The cardinal rule:
> **`*.out` for entrances, `*.in` for exits, `*.inOut` for moves between two
> on-screen positions.** Getting in/out backwards is the most common tell that
> motion was defaulted, not designed.

These are GSAP ease strings — drop them straight into `ease:` on a tween. Set
`defaults: { duration: 0.6, ease: "power2.out" }` at the timeline level to
declare the composition's motion language once, then override per-tween.

## Entrances (decelerate INTO rest — `.out`)

| Ease | Feel | Use for |
|---|---|---|
| `power1.out` | gentle | secondary text, subtle fades |
| `power2.out` | standard | the default UI entrance; safe everywhere |
| `power3.out` | punchy | hero lines, title cards, anything that should land with intent |
| `power4.out` | hard stop | big bold reveals, decisive product shots |
| `expo.out` | premium / dramatic | luxury reveals, slow-then-snap; steeper deceleration than power4 |

## Playful / decisive pops (overshoot then settle)

| Ease | Feel | Use for |
|---|---|---|
| `back.out(1.2)` | elegant overshoot | refined pops, badges |
| `back.out(1.7)` | standard pop | icons, chips, call-outs (the go-to "boing") |
| `back.out(2)`–`back.out(2.5)` | playful / bouncy | `casual`/`playful` brands; bigger arg = more overshoot |
| `elastic.out(1, 0.3)` | springy scatter | drops, scatter-in, settling on a surface (amp, period) |
| `bounce.out` | ball drop | score counters, an icon landing hard |

## Moves between two on-screen positions / camera (`.inOut`)

| Ease | Feel | Use for |
|---|---|---|
| `power2.inOut` | smooth reposition | element gliding to a new spot |
| `power3.inOut` | weighty reposition | larger objects, layout shifts |
| `expo.inOut` | snappy + dramatic | fast camera-style pushes, whip moves |
| `circ.inOut` | orbital / mechanical | camera arcs, orbital motion, anything circular |

## Settles, holds, micro-motion (the "breathe")

| Ease | Feel | Use for |
|---|---|---|
| `sine.inOut` | continuous, organic | breathing scale, Ken Burns drift, ambient gradient pan, any `yoyo` loop |
| `none` (linear) | constant velocity | camera dollies, conveyor/marquee motion, mechanical drifts |

A held card with one breathing layer (see motion-philosophy §"Micro-motion"):

```js
tl.to(".card-glow", { scale: 1.03, duration: 2.4, ease: "sine.inOut",
                      yoyo: true, repeat: 1 }, 0.6);
```

## Exits (accelerate AWAY — `.in`)

| Ease | Feel | Use for |
|---|---|---|
| `power2.in` | standard exit | the default leave |
| `power3.in` | fast clear | snappy dismissals, getting out of the way for the next beat |

Remember: exits are shorter than the matching entrance (~0.4s in / 0.25s out).

## Mechanical / stepped

| Ease | Feel | Use for |
|---|---|---|
| `steps(N)` | discrete ticks | typing effects, cursor blink, odometer/counter ticks, frame-by-frame sprites |

## Velocity-matched cuts (one continuous move across a transition)

When two scenes should read as one camera move, match velocities at the cut:
the outgoing element exits on `power2.in`/`power3.in` (accelerating, often with
motion blur), and the incoming element enters on the matching
`power2.out`/`power3.out` (decelerating). The eye reads the seam as a single
push rather than two separate animations. See examples.md §"Velocity-matched
transition".

---

## Quick rules

- 3+ distinct eases per scene. Repeating one ease across every element makes
  every reveal feel identical.
- `.out` in, `.in` out, `.inOut` between. Always.
- `back`/`elastic`/`bounce` carry personality — use them for `playful`/`casual`
  voices, sparingly for `corporate`/`minimal`.
- `sine.inOut` is the workhorse for everything that loops or breathes.
