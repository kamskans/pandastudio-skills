# Soundtracks: `media.compose-soundtrack`

Compose the music, beats and sound effects for an edit from a **score** you
write, on the **same timeline as the picture**. PandaStudio synthesizes every
sound on the user's machine (no service, no key, no licence questions), masters
it to -14 LUFS with true peak under -1 dBTP, and writes a WAV. About 1 s to
render 15 s.

Use it for **promos, launch films, product spots, intros/outros, Shorts hooks,
UI demos and motion-graphics pieces**: anything where hits must land on
moments. For a 10-minute bed under a talking head, the bundled music library
(`asset.list-music`) is still the better choice; a synthesized bed gets
repetitive over that length.

> Scoring an existing edit (talking head, Short, tutorial)? Use
> `project.compose-soundtrack`: it builds the score below from the project's
> own timeline and stressed words. This page is the manual, full-control path.

## The method: one timeline, not "sync music to video"

Do not pick a track and then try to cut to it. Write one timeline and put both
the picture and the sound on it:

1. **Write the timeline table first.** Every scene and key moment with an exact
   time: `wipe 2.4`, `slam 3.0`, `price 3.52`, `features 4.8` (then every
   1.3 s), `logo 12.95`. These become the score's `cues`. In the HTML
   composition, define the same numbers once (`const CUES = {...}`) and drive
   the GSAP timeline from them, so picture and sound read the same values.
2. **Pick the tempo to fit the edit.** Where there's a steady groove, choose
   the BPM so every cut lands on a beat: `beat = cut spacing / 2` (or `/ 4`).
   Features every 1.3 s means beat = 0.65 s = **92.3 BPM** with
   `chordBeats: 2`, so each feature gets one chord and every cut is a downbeat.
   BPM = 60 / beat.
3. **Place sounds on cues, never on a grid.** The slam gets `boom + kick +
   snare`; each cut gets a `whoosh` 0.12 s before it; typed letters get
   `tick`s at the letter times; popping elements get `pop`s; logo letters get
   `tick`s as they land.
4. **Render, place, check.** `media.compose-soundtrack`, then
   `project.add-audio --startMs=0` with the returned `durationMs`. Moving a
   moment later? Change its cue and call again: every sound on that cue moves
   with it.

## Score format

```json
{
  "duration": 15,
  "seed": 3,
  "cues": { "slam": 3.0, "features": 4.8, "logo": "slam+9.95" },
  "events": [
    { "at": "slam", "sound": "boom", "dur": 2.2, "gain": 1.1 },
    { "at": "slam", "sound": "kick", "gain": 1.2 },
    { "at": "features-0.12", "sound": "whoosh", "dur": 0.3, "pan": 0.5 },
    { "at": "logo+0.17", "sound": "tick", "gain": 0.5, "repeat": { "count": 10, "every": 0.03 } }
  ],
  "sections": [
    { "from": "features", "to": 12.6, "bpm": 92.3077, "chords": ["Am", "F", "C", "G"],
      "chordBeats": 2, "drums": "four-on-floor", "hats": "16ths", "bass": "octave",
      "arp": "updown", "whooshOnChange": true }
  ],
  "master": { "lufs": -14, "fadeOut": 0.45 }
}
```

- **Times** (`at`, `from`, `to`, cue values): seconds, a cue name, or cue
  arithmetic (`"slam+0.2"`, `"logo - 0.12"`). Cues may reference other cues.
- **events[]**: `sound` (list below), `gain` (0-4, default 1), `pan` (-1 left
  to 1 right), `dur` (s), `note` / `notes` (`"A3"`, `"C#4"`, MIDI numbers) or
  `chord` (`"Am"`, `"Fmaj7"`, `"Gsus4"`, `"Dm7"`), `from` / `to` (Hz, for
  sweeps and pitched blips), `bright` (0 dark to 1 bright, default 0.5),
  `reverb` (0-1 send), and `repeat: { count, every, gainStep, alternatePan,
  jitter }` for ticks, typing and rolls.
- **sections[]**: a groove over a stretch: `bpm` (required), `chords` cycled
  every `chordBeats` beats (default 4), plus `drums`, `hats`, `bass`, `arp`
  (`arpRate` 4ths / 8ths / 16ths), `pad` (true/false/gain), `whooshOnChange`,
  and `melody: { notes: ["E5", null, "G5", ...], rate, sound, gain }` where
  `null` is a rest (a note holds through the rests after it). Balance the
  parts with `mix: { drums, hats, bass, arp, pad, melody }` (0-2, 1 =
  default). Bars start at the section's `from`.
- **master**: `lufs` (default -14), `ceiling` (true-peak dBFS, default -1),
  `fadeIn`, `fadeOut` (default 0.45 s), `drive` (1 clean to 4 crushed, default
  1.3), `room` (reverb size 0-1, default 0.6).
- Same score + same `seed` = byte-identical audio.
- Errors name the exact field (`events[3].sound "cowbell" is unknown`); fix
  that field and call again. `warnings` flags events past the end.

## Sounds

| Sound | What it is | Use it for | Useful params |
|---|---|---|---|
| `kick` | Sine with a fast pitch drop | Beats, the hit under every slam | `from`/`to` (155→45 Hz), `dur` |
| `snare` | Noise burst + short tone | Backbeat, slam crack | `bright` |
| `clap` | Three quick smacks + tail | Backbeat alternative, reveals | `bright` |
| `hat` / `openhat` | Bright noise ticks | Grooves (sections do this for you) | |
| `tick` | Tiny bright click | Typed letters, logo letters landing, count-ups, UI ticks | `bright`, `repeat` |
| `click` | Dry mouse click | Cursor clicks on buttons | |
| `blip` | Pitched chirp | UI confirmations, price tags, badges | `from` (Hz) or `note`, `dur` |
| `pop` | Bubbly rising pop | Cards, chips, icons, ears popping in | `from`/`to`, `bright` |
| `typing` | A human burst of keys | A whole typed line in one event | `dur` = typing time |
| `whoosh` | Airy swell and fade | Just before each cut or camera move (never two within a second) | `dur` 0.25-0.5, `pan` |
| `riser` | Noise opening + rising tone | Into a drop, a slam, a logo | `dur` = lead-in, `from`/`to` |
| `downlifter` | Falling sweep | Right after a hit, transitions out | `dur` |
| `impact` | Kick + boom + crack | The single biggest moment | `dur`, `bright` |
| `boom` | Distorted falling sub | Hero slams, logo lands | `from`/`to` (70→32 Hz), `dur` 1.5-2.5 |
| `subdrop` | Clean sub sweep down | Drops, dramatic pauses | `from`/`to`, `dur` |
| `shimmer` | Twinkling high partials | Sparkles, "magic" moments, logo glints | `note`, `dur` |
| `bass` | Filtered saw + sub | Bass lines (sections do this) | `note`, `dur`, `bright` |
| `pluck` | Soft plucked tone | Arps, melodies, logo chimes | `note`/`notes`, `dur` |
| `pad` | Detuned sine chord | Beds, intros, the chord under a slam | `chord` or `notes`, `dur` |
| `lead` | Detuned saw with vibrato | Hooks and melodies | `note`, `dur`, `bright` |
| `bell` | FM bell | End cards, success moments | `note`, `dur` |
| `keys` | Electric-piano tine | Warm chords, lo-fi, calm tutorials | `chord`/`notes`, `dur` |

## Recipes

**15-30 s product promo (the PandaCrawl pattern).** Ticks + soft pad under the
opening; three kicks building; a riser into the turn; the slam = `boom + kick
+ snare + pad`; a riser into the groove; a section at the cut tempo with
`four-on-floor`, `16ths` hats, `octave` bass, `updown` arp,
`whooshOnChange: true`; typing ticks on the agent/typing moment; a riser into
the logo; logo = `kick + boom + pad`, ticks on the letters, a rising pluck
arpeggio, a final `blip`. The complete score is at the end of this page.

**Launch film (45-90 s).** Several sections, each with its own chords and a
denser groove as energy builds (intro `pad` only, then `half-time`, then
`four-on-floor`); `impact` on each chapter title; `whoosh` on camera moves;
`click` on every cursor click; `typing` on typed lines; `bell` + `shimmer` on
the end card; `fadeOut: 1.5`.

**Shorts hook (0-3 s).** A `subdrop` or `impact` exactly on the first word,
`pop`s on each caption chip, then a `backbeat` section at 100-120 BPM under the
rest at `gain: 0.7` so speech stays clear.

**Calm tutorial / lo-fi bed.** `bpm: 80`, chords like `["Fmaj7","Em7","Dm7",
"Cmaj7"]`, `drums: "half-time"`, `hats: "8ths"`, `bass: "root"`, `pad` on,
`melody` with `sound: "keys"`, `mix: { drums: 0.6 }`, `master.lufs: -18` when
it sits under a voice (the export mixer ducks it further with
`project.set-audio-ducking`).

**Cinematic trailer.** `pad` chords with long `dur` and `reverb: 0.7`, `boom`
on every title, `riser` of 2-4 s into the climax, `impact` on the logo,
`room: 0.9`.

## Tips

- Minor keys (`Am F C G`) read driven and modern; major (`C G Am F`) reads
  bright and friendly; `sus` and `maj7` chords read airy.
- Keep hits sparse: one big moment every 3-5 s. Stacked booms turn to mud.
- A `whoosh` goes 0.1-0.15 s **before** the cut, a `riser` ends **on** the hit
  (`at: "slam-0.5"`, `dur: 0.5`).
- Pan small repeated sounds alternately (`repeat.alternatePan: true`) for
  width; keep kicks, booms and bass centred.
- With a voiceover, generate the voice first, take its line times as cues, and
  keep the music `gain` around 0.6-0.8 under it.

## Worked example: the PandaCrawl 15 s spot

Cues from the edit: wipe 2.4, slam 3.0, price 3.52, features 4.8 (six
features, 1.3 s each), agent typing 11.32, logo 12.95. Groove tempo:
60 / 0.65 = 92.3 BPM, two beats per feature.

```json
{
  "duration": 15, "seed": 3,
  "cues": { "wipe": 2.4, "slam": 3.0, "price": 3.52, "features": 4.8, "agent": 11.32, "logo": 12.95 },
  "events": [
    { "at": 0, "sound": "pad", "notes": ["A2", "E3", "A3", "C4"], "dur": 2.45, "gain": 0.83 },
    { "at": 0.12, "sound": "tick", "gain": 0.72, "repeat": { "count": 7, "every": 0.085, "alternatePan": true } },
    { "at": 0.92, "sound": "kick", "gain": 0.55, "repeat": { "count": 3, "every": 0.24 } },
    { "at": "wipe-0.75", "sound": "riser", "from": 180, "to": 1400, "dur": 0.75, "gain": 0.9 },
    { "at": "wipe-0.45", "sound": "whoosh", "dur": 0.45, "pan": -0.3 },
    { "at": "wipe", "sound": "kick" },
    { "at": "slam-0.45", "sound": "riser", "from": 300, "to": 700, "dur": 0.45, "gain": 0.7 },
    { "at": "slam", "sound": "boom", "dur": 2.2, "gain": 1.1 },
    { "at": "slam", "sound": "snare", "gain": 1.2 },
    { "at": "slam", "sound": "kick", "gain": 1.2 },
    { "at": "slam", "sound": "pad", "notes": ["A3", "E4", "A4", "C5", "E5"], "dur": 1.6, "gain": 1.17 },
    { "at": "price", "sound": "blip", "from": 900, "dur": 0.15 },
    { "at": "price+0.1", "sound": "blip", "from": 1350, "dur": 0.15, "gain": 0.83 },
    { "at": "features-0.5", "sound": "riser", "from": 300, "to": 2400, "dur": 0.5, "gain": 0.9 },
    { "at": "features-0.12", "sound": "whoosh", "dur": 0.3, "gain": 0.8, "pan": 0.5 },
    { "at": "agent", "sound": "tick", "gain": 0.48, "pan": 0.2, "repeat": { "count": 18, "every": 0.021 } },
    { "at": "logo-0.4", "sound": "riser", "from": 400, "to": 3000, "dur": 0.4, "gain": 0.7 },
    { "at": "logo", "sound": "kick", "gain": 1.2 },
    { "at": "logo", "sound": "boom", "from": 60, "to": 30, "dur": 2.0, "gain": 0.8 },
    { "at": "logo", "sound": "pad", "notes": ["A2", "A3", "E4", "A4", "B4", "E5"], "dur": 2.05, "gain": 1.33 },
    { "at": "logo+0.17", "sound": "tick", "gain": 0.48, "repeat": { "count": 10, "every": 0.03 } },
    { "at": "logo+0.55", "sound": "pluck", "note": "A5", "dur": 0.9, "gain": 0.41, "pan": -0.3 },
    { "at": "logo+0.64", "sound": "pluck", "note": "C6", "dur": 0.9, "gain": 0.41, "pan": -0.1 },
    { "at": "logo+0.73", "sound": "pluck", "note": "E6", "dur": 0.9, "gain": 0.41, "pan": 0.1 },
    { "at": "logo+0.82", "sound": "pluck", "note": "A6", "dur": 0.9, "gain": 0.41, "pan": 0.3 },
    { "at": "logo+0.93", "sound": "blip", "from": 1760, "dur": 0.6, "gain": 0.83 }
  ],
  "sections": [
    { "from": "features", "to": 12.6, "bpm": 92.3077, "chords": ["Am", "F", "C", "G", "Am", "F"],
      "chordBeats": 2, "drums": "four-on-floor", "hats": "16ths", "bass": "octave",
      "arp": "updown", "arpRate": "16ths", "whooshOnChange": true }
  ],
  "master": { "fadeOut": 0.45, "drive": 1.3 }
}
```
