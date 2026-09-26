# Timing an edit to the words

> **Shortcut:** for a talking head or a Short, `project.style-edit` runs this
> whole loop in one call (import checks → speech map → native motion elements
> + zooms on the cues → score) in a house style. See
> [motion-elements.md](motion-elements.md). The loop below is what it does,
> and what to do by hand when you want more control.

The best editors time everything to the speaker: the zoom lands on the word
they lean on, the keyword graphic appears as they say it, the hit lands on the
punchline, the music builds toward the payoff and gets out of the way while
they talk. PandaStudio gives you the timing as data, so any model can do this
on any video: Shorts, long-form talking heads, tutorials, podcasts, promos.

## The loop

1. **Transcribe** (`transcript.transcribe`). It uses the engine the user's
   settings pick. For Tamil, Telugu, Hindi and other languages the local
   English model gets wrong, check `system.get-transcription-language` /
   `-provider` first; with a cloud provider set up (Deepgram / ElevenLabs) or
   the language set, it transcribes in that language.
2. **Tighten** as the pipeline says (fillers, bad takes, silences). Cuts go
   between words, never into one.
3. **Map the speech**: `project.speech-map` returns the stressed words with
   their edited-time position (`atMs`), a nearby `phrase`, a `strength`
   (1 light, 2 strong, 3 hero), the `reasons` and a `suggest`ion. It is
   measured from the audio (louder than the speaker's usual level, a sharp
   attack, a pause before, a drawn-out word) plus numbers, currency and
   English key terms inside non-English speech, so it works in any language.
   Set the density with `everyMs`: Shorts 3000-4000, long-form 8000-15000,
   tutorials 10000+.
4. **Write the cue sheet.** Take the moments you'll use (and the structure you
   see in the transcript: the hook, each point, the payoff, the ending) and
   write ONE table of named times. Everything below reads it.
5. **Place the picture on the cues.**
   - Zooms: `project.add-zoom` on strength 2-3 moments. Alternate 1.25x and
     1.5x on consecutive beats so jump cuts read as intentional; save 2x for
     the one hero line. Pass `soundUrl: null` if you'll score the edit (the
     score adds its own whoosh, and compose-soundtrack clears defaults anyway).
   - Keyword graphics: **native motion elements are the default**
     ([motion-elements.md](motion-elements.md)). One
     `project.add-motion-elements` call places them all, each anchored to its
     cue word (`wordId`), so later cuts carry them along: `keyword` on the
     phrase (1-4 words, condensed or translated to English, not verbatim),
     `count` on numbers, `chip` on key terms, `stamp` / `slam` on the hero
     line, `steps` for a spoken list, `endCard` at the end, `layer: "behind"`
     for a word behind the speaker's head. `zone: "auto"` keeps them off the
     face. Latin script only (the engine fonts).
   - HTML compositions are for bespoke extras: a chart, a diagram, a UI mock,
     a logo moment, non-Latin text. Build those as ONE continuous composition
     (a `CUES` object at the top drives the GSAP timeline), render it
     transparent and place it once (`project.add-motion-graphic`, full frame);
     on `layer: background` it becomes a designed background behind a keyed
     speaker.
6. **Score it last**: `project.compose-soundtrack` builds music + sound
   design from the timeline you just made (zooms, graphics, pops, cuts, where
   the person talks, the hero words) and places it under the voice with
   ducking. Native motion elements are read directly (each one's sound role
   at its start, a pop per step, a hero hit on a slam); pass an HTML
   composition's cue times as `cues` (they're invisible to the timeline
   otherwise). Style by format: `short`, `talking-head`,
   `calm`, `promo` (no voice). For full control, `dryRun: true` returns the
   score; edit it and render with `media.compose-soundtrack`
   ([soundtrack.md](soundtrack.md)).
7. **Check the moments**: `project.render-frame` at each cue (the graphic is
   on screen, clear of the face, the zoom has landed), then export and
   `export.verify` (about -14 LUFS, true peak under -1 dBTP).

## Density by format

| Format | Moments | Zooms | Graphics | Score |
|---|---|---|---|---|
| Short (15-90 s) | one per 3-4 s | every other sentence, 1.25x/1.5x, 2x on the kicker | one keyword per beat, hook title in the first second, end card | `short` (or `talking-head` for a calm speaker) |
| Long-form talking head | one per 8-15 s | on real emphasis only, 1.25x mostly | chapter titles + the key number or term, sparse | `talking-head`, music low, hits only at chapter turns |
| Tutorial / walkthrough | one per 10 s+ | on the thing being shown | labels and steps | `calm` |
| Promo / launch (no voice) | the edit's own beats | on reveals | the story itself | `promo` |

## Music under speech

- Harmony and texture while they talk; drums low; no busy melody.
- The groove builds by chapter or act and resolves on the ending.
- Sound effects only on graphic moments and the hero line: a whoosh 0.1 s
  before a move (never two within about a second), a riser that ENDS on the
  hero word, pops on small chips, a sub-drop on a "go deep" / reveal moment.
- Jump cuts stay dry: a sound on every cut reads as a gimmick.

## Keyword graphics vs captions

- Word-by-word captions are right when the language renders well in caption
  fonts and the transcript timing is word-accurate.
- Keyword graphics are better for code-switched speech (Tanglish, Hinglish),
  for languages the caption fonts don't cover, and for a premium look: they
  carry the meaning (often in English) without covering the speaker.

## Import checks (do these before styling)

Run `project.inspect-footage`: it says per clip whether the footage is flat /
log, whether the backdrop is a flat green or blue, and whether it was shot
rotated, with the exact verb to run for each (`suggestions`). What it checks:

- **Flat / log colour** (washed-out, low contrast, grey blacks): apply
  `project.set-clip-color --preset=flat-footage` first.
- **Flat green or blue backdrop**: `project.set-clip-chroma-key`, then a
  designed background (a `layer: background` composition or
  `project.set-wallpaper`); check the edges with `project.render-frame`.
- **Vertical camera footage** (a phone or a camera turned on its side) is
  stored landscape with a rotation flag; PandaStudio reads the flag, so it
  appears upright. If a frame ever renders sideways, report it.

## Worked example: a 73 s Tamil talking-head Short

With native elements this whole example is one call plus English keywords for
the Tamil hero moments:
`project.style-edit --style=bold-short --applyFixes=true --texts='{"<wordId>":"6 months", ...}'`.
The hand-built version below (HTML compositions) remains the reference for
bespoke looks.

Speech map + transcript gave these cues (edited seconds): hook 0.05, "launch
a brand in six months" 14.3 / 16.3, "it's not the logo" 20.0, "20-30 years"
25.5 (hero), "no money can create it" 30.5, "generation" x3 35.8 / 37.3 /
38.3, "eyes closed" 40.1, "that grip" 49.3, "one category, one region, go
deep" 53.4 / 56.3 / 58.3, "depth is more important than reach" 61.9 / 64.1
(hero), end card 69.3.

- Picture: flat-footage grade, green backdrop keyed, a warm dark background
  composition with outline words ("BRAND", "TRUST", "DEPTH") behind her head,
  a foreground composition with a headline or chip on every cue (hook title,
  "6 MONTHS", a 20-30 count-up, a "₹ CAN'T BUY IT" stamp, generation steps,
  the DEPTH > REACH slam, an end card), zooms 1.5x / 1.25x alternating, 2x on
  the hero line.
- Sound: `project.compose-soundtrack --style=talking-head` with the
  composition's cues passed as `cues`.
