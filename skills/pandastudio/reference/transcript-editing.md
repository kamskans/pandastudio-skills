# Transcript-based editing — PandaStudio's signature feature

The reason humans pick PandaStudio over Premiere is that you edit by
**deleting words from the transcript**, not by scrubbing the timeline. The CLI
exposes the same model. Every deletion becomes a **trim region** the export
skips — identical to deleting the word in the editor's transcript pane.

## Read clip state first

Before `transcript.transcribe` or `audio.enhance`, call `project.read` and read
`clipStates[]`:

```json
{ "clipId": "clip-1", "mediaPath": "...", "durationMs": 62400,
  "transcribed": true, "wordCount": 312,
  "audioCleaned": false, "kind": "camera" }
```

- `transcribed: true` → skip `transcript.transcribe` for that clip. Re-run it
  only with `--clipId` when the transcript is genuinely wrong (new audio, a bad
  run): word fixes survive it (below), but it costs time and can drop fixes
  whose words the new run hears differently.
- `voiceEnhanced` present → skip `audio.enhance` for that clip (each entry has
  `strength` and a `summary`; `legacy: "clean"` is a Clean audio result from
  before 2.2, which plays as Light with only noise removal; run `audio.enhance`
  on it only if the user wants the full chain). `audioCleaned: true` just means
  the clip plays a processed track.
- `wordFixes: n` → the clip carries n stored text fixes.
- `contentIssues` (top level, only when something is transcribed) →
  `{ total, duplicateTakes, falseStarts, adjacentRepeats }`. If `total > 0`,
  run `transcript.find-issues` during the polish.

Only pass un-processed clips to each operation. If every clip is already
transcribed, go straight to `transcript.get`.

## The full edit loop

```bash
# 0. Check which clips still need processing (avoids clobbering in-app edits)
STATE=$(pandastudio project.read --id=$ID --json | jq '.data.clipStates')

# 1. Transcribe only clips that don't already have a transcript
JOB=$(pandastudio transcript.transcribe --id=$ID --json | jq -r '.data.jobId')
pandastudio job.wait --id=$JOB --timeoutMs=300000 --json

# 2. Pull the merged transcript — every word with source + edited times
pandastudio transcript.get --id=$ID --format=words --json | jq '.data.words[0:20]'
# To READ it (planning, finding beats): --format=text, or a window with
# --fromMs --toMs (edited ms); the default compact format is also fine.

# 3a. AUTO: drop every "um" / "uh" + immediate repeats
pandastudio transcript.remove-fillers --id=$ID --json
# → { removedCount, fillersRemoved, repeatsRemoved, trimsAdded, faded,
#     kept: [{ wordIds, text, reason: "joined to the next word", code: "fused", side }] }
# Every filler is cut by default; one with no dip in level between it and the
# word it runs into is cut at the word boundary (faded in export).
# --force=false keeps those instead and lists them in kept[].

# 3b. Remove silences (default 600ms = the UI button; leading/trailing/between-word)
#     Omit --thresholdMs to use the default; only pass it to override.
pandastudio transcript.remove-silences --id=$ID --json
# → { removedCount, totalTrimmedMs, audioDetectionFailed }

# 3c. Fix STT errors — NEVER use project.read → JSON mutation → project.save for this.
pandastudio transcript.find-replace --id=$ID --find="RightPanda" --replace="WritePanda" --json
# → { replacedCount, wordsPatched, wordsMerged }

# 3d. SURGICAL: delete specific words by ID
pandastudio transcript.delete-words --id=$ID --wordIds='["clip-1:w-42","clip-1:w-43"]' --json

# 3e. PHRASE search → bulk delete
WORDS=$(pandastudio transcript.search --id=$ID --query="this is a test" --json \
  | jq -c '[.data.matches[].wordIds | .[]]')
pandastudio transcript.delete-words --id=$ID --wordIds="$WORDS" --json

# 3f. RESTORE previously deleted words (undo a delete / filler / repeat removal).
#     Removes only the restored words' time from the cuts; silence trims are left
#     untouched. Mirrors the editor's right-click → Restore on struck-through words.
pandastudio transcript.restore-words --id=$ID --wordIds='["clip-1:w-42","clip-1:w-43"]' --json
```

**`transcript.get` shows ALL words, including deleted ones.** Deleted words
become trims (gone from the export) but stay in the raw word list. To confirm a
deletion, check `trimsAdded` in the response instead of looking for missing
words. A compact word is `[id, text, editedStartMs, editedEndMs, sourceStartMs]`:
edited times first (what plays; `null` = inside a trim), the source start last.

**STT coherence:** fix transcript errors with `transcript.find-replace` BEFORE
`motion.generate`, `llm.generate-title` or captions. Titles, slot values and
captions are derived from the transcript text, so a "RightPanda" propagates.

## Fillers

`transcript.remove-fillers` default is the SAFE tier only: vocalised pauses
(um, uh, uhm, umm, hmm, hm) AND immediate repeated words. These are sounds,
never lexical, so removing every match is correct. `--aggressive=true` also
removes `like / you know / i mean / sort of / kind of`; these are real words
too, so it sometimes cuts legitimate uses ("I like this template" loses
"like"). Opt in only when the user asks for a thorough cleanup and is willing
to skim for false positives. Trims are reversible.

## Bad takes: `transcript.find-issues` → `transcript.delete-words`

`find-issues` is READ-ONLY. It surfaces re-takes (`duplicate-take`),
abandoned restarts (`false-start`) and stutters (`adjacent-repeat`), each with
the `wordIds` of the discarded attempt. Default: **keep the most recent (last,
cleaner) take and delete the earlier attempt** by feeding the candidate's
`wordIds` into `transcript.delete-words`. Running find-issues without deleting
is the same as doing nothing.

**`severity: "low"` candidates are REVIEW-class: keep them by default.** A low
`false-start` means the restart diverges from the fragment, which is often
deliberate parallel structure ("one for transcription, one for outreach"), not
a flub; deleting it destroys the sentence. Only delete a low candidate when the
context clearly shows an abandoned take. The detector already skips
comma-terminated parallel list items and lone stopword "repeats" across pause
tokens. If a repeat might be intentional emphasis, or you can't tell which take
is cleaner, ask the user which to keep.

## Edit for clarity: `transcript.edit-for-clarity` → `transcript.apply-clarity`

The Transcript tab's **Edit for clarity** (like Descript Underlord's): it
PROPOSES cuts of words that don't change what the speaker means, and nothing
is cut until a proposal is accepted. Use it for long talking-head videos and
podcasts when the user wants the talk tighter ("edit for clarity", "tighten
it up", "cut the rambling"). Run it after `remove-fillers` and `find-issues`;
it skips words already cut.

```
pandastudio transcript.edit-for-clarity --id <projectId>            # rules + the connected AI, else the local model (a job when a model runs)
pandastudio transcript.edit-for-clarity --id <projectId> --engine=rules   # instant, no model
pandastudio transcript.edit-for-clarity --id <projectId> --kinds=filler-phrase,repeat
pandastudio transcript.edit-for-clarity --id <projectId> --cuts='[{"wordIds":["w12","w13","w14"],"kind":"restatement","reason":"repeats the line before"}]'
pandastudio transcript.apply-clarity --id <projectId> --proposalIds='["cl_ab12","cl_cd34"]'   # or '["all"]'
```

- Kinds: `filler-phrase` ("you know what I mean", "basically what I'm saying
  is", "if that makes sense"; English lexicon), `repeat` (a phrase said twice
  in a row; the second is kept), `false-start`, `restatement` (a sentence or
  clause that only says again what was just said), `retake`
  (find-issues' duplicate-take).
- **The model is the user's own.** The verb reads with the local model the
  `llm.*` verbs use. When it is downloaded the verb returns `{ jobId }`;
  `job.wait` for `{ proposals, sections, sectionsRead, refused }` (about
  2 minutes for a 40-minute talk). Without it the verb answers at once with the
  rules' proposals and `model: "unavailable"` (or `"off"` for
  `--engine=rules`). The app's Edit for clarity button reads with the model
  picked for the AI chat when one is connected (the user's own sign-in or
  key), else the local model. Never send the transcript to another cloud
  model for this.
- The local model is small: it mostly confirms fillers; restated sentences
  and false starts the rules miss are best found by YOU (next point).
- **You are the user's AI too**: read `transcript.get` and pass your own picks
  as `--cuts` (word ids, kind, reason). They go through the same checks as the
  model's and come back as proposals (`source: "agent"`), refused ones with
  `why`. Use this for restatements and false starts (and for languages the
  lexicon doesn't cover). The checks decide the kind; yours is a hint.
- Every model / agent pick is checked: a verbatim run of ONE sentence, no
  negation ("not", "never", "n't") and no number, at a clause boundary unless
  it is a real repeat or pure filler, at least 3 words of the sentence left,
  and a `restatement` must really repeat the sentence before (its meaning
  words). A whole sentence goes only as a restatement or an unfinished start.
- Each proposal: `{ id, kind, source rules|ai|agent, wordIds, text, before,
  after, cutAt (where text starts in before), reason, startMs, endMs (source),
  editedStartMs }`. `after` is `""`
  when the whole sentence goes.
- **Show the user the before/after and let them choose** (accept all, some, or
  none), then `transcript.apply-clarity --proposalIds`. Nobody to ask (a
  routine): apply only `filler-phrase`, `repeat` and `retake`; leave
  `restatement` and `false-start` for the user.
- `apply-clarity` cuts like `delete-words` (snapped to measured pauses, one
  revision, undoable, `restore-words` brings words back). Proposal ids are
  stable for the same words and kept in memory per project until the app
  restarts; one whose words changed since is skipped as `stale`.

## Silences

`transcript.remove-silences` runs the SAME two passes as the UI Remove Silences
button and unions them: (1) transcript word-gaps (leading, between-word,
trailing) and (2) ffmpeg audio-level `silencedetect` (noise -30 dB) on each
clip's media, which catches real dead air the transcript misses when
speech-to-text invents phantom words over quiet stretches. Default threshold
600ms; don't hand-pick a higher value "to be safe". `--paddingMs=100` sets the
margin kept around every word, so it never cuts into words. When the
transcript times words back to back (each word "ends" as the next starts), a
pause the audio measures as silent is still cut: a word is guarded only on its
voiced part, and a quiet word the audio reads as silence stays guarded whole.
That holds even when the word timings leave no gap at all: every silencedetect
pause of at least `thresholdMs` is a cut candidate where the full-band level
also hears it as quiet. The result's `audioPauses` = `{found, quiet,
clearOfWords, cut}` says what the audio held; `removedCount: 0` always comes
with a warning saying why (no pause that long, sound in each, words' voiced
sound around each, or already cut). Trust it: don't run your own silencedetect
and cut with `project.apply-edit-plan`. Run it AFTER the content cleanup. If the user already removed silences in the UI, a fresh
`project.read` shows the new `trimCount` / `editedDurationMs` /
`totalTrimmedMs`: treat that as done.

## Adding pauses (pacing)

The opposite of removing silences: give speech room to breathe. Ideal for
TTS / ElevenLabs narration (writing pauses into the script makes the voice
speed up instead), so for generated narration run `space-out` right after
transcribing it.

```bash
# Plan first: how many pauses, how much longer (writes nothing)
pandastudio transcript.space-out --id "$ID" --preset=natural --dryRun=true
# → { pauses: 34, addedMs: 18200, description: "Adds 34 pauses, +18.2 s", items: [...], skipped }
pandastudio transcript.space-out --id "$ID" --preset=natural   # one revision
# One pause after a word (250 / 500 / 1000 / 2000 typical, 30..60000 ms)
pandastudio transcript.insert-pause --id "$ID" --afterWordId=w123 --ms=500
pandastudio project.remove-pause --id "$ID" --afterWordId=w123   # or --regionId, --all=true
```

- Targets (natural / relaxed / tight): after a comma 0.15 / 0.2 / 0.12 s,
  after a sentence 0.42 / 0.5 / 0.35 s, at a paragraph or speaker change
  0.7 / 0.8 / 0.6 s. The gap heard NOW (after cuts and earlier pauses) is
  topped up; a longer gap is never shortened, and rerunning adds nothing (an
  existing pause is lengthened, not duplicated). `--fromMs --toMs` (edited ms)
  limits it to a stretch.
- Pauses go where the word really ends in the audio, not at the transcript's
  word end (speech-to-text ends words early, most of all on TTS): the start of
  the measured silence after the word, else just before the next word. So a
  pause never splits a word; insert-pause reports it as `snappedTo`.
- A pause is extra timeline time filled with the take's own room tone (the
  quietest nearby gap between words, looped); it is silent only when the take
  has no pause anywhere. On video the frame holds. Everything after it
  (captions, anchored zooms / overlays / graphics) moves later with its words.
- Stored as a speed region with `freezeMs` (the length) and `pause`; change the
  length with `project.update-region --regionType=speed --regionId --freezeMs`.
  A pause can't go inside a speed change (those joints are listed in `skipped`).
- Run it AFTER cuts and remove-silences, and BEFORE graphics / B-roll timing.

## Fixing words

**`transcript.find-replace`** patches word text in place and keeps caption
timing continuous. Matcher caveats:
- Case-insensitive; punctuation is ignored on both sides (`--find="graph, crew"`
  matches "graph crew").
- DIGITS are significant when the find phrase contains them: `--find="try30"`
  matches only the merged STT token "try30", never a plain "try". A letters-only
  find still ignores transcript-side digits (`--find="than"` also matches
  "than60").
- A multi-word find ALSO matches a single merged STT token: `--find="Wispr
  Flow"` hits the one token "Wispr Flow", and `--find="of $499"` hits the
  merged "of$499".
- A find that exactly equals a token's raw text always matches, so
  space/digit/punctuation-bearing tokens (even "30%") are targetable verbatim.
- A multi-word `--find` collapses to the FIRST word's slot: that word's text
  becomes `--replace`, the other matched words are blanked (one word spanning
  the whole match, so captions have no gap; e.g. STT's "P a n d a S t u d i o"
  becomes one word). Include any punctuation you want kept in `--replace`.
- It splits a match that crosses a deleted part (the replacement goes on the
  kept words), verifies trims are unchanged, and supports `--preview=true`.
  Text edits never move cuts.

**`transcript.insert-words`** — when STT DROPPED a spoken word entirely
(find-replace only rewrites words that exist). Anchor with `--afterWordId` (or
`--beforeWordId` to add at the very start) from `transcript.get`, and pass
`--text`. It computes plausible timing (fills the gap the drop left, sized to
the local speaking rate) and is non-destructive: existing word ids, and any
trims/zooms/captions anchored to them, are untouched. It does NOT
re-transcribe.

## Word fixes survive re-transcription

Every text fix (`transcript.find-replace`, `transcript.insert-words`, a
double-click edit in the app) is stored on the clip keyed by time + the
transcriber's original text, not by word id. When the clip is transcribed again
(`transcript.transcribe --clipId`, a camera-audio swap, the app's retry), each
fix is re-applied where the new run heard the same original words at the same
moment (case/punctuation ignored, ~0.4 s tolerance); a fix the new run already
got right counts as re-applied. The job result reports `wordEditsReapplied`,
`wordEditsDropped` and `droppedWordEdits[]` (`{ clipId, description, reason }`:
`no-match` = the original words aren't heard there any more, `conflict` = a
different word is now heard where one was inserted, `crosses-cut` = a merged
fix would straddle a deletion). Surface dropped fixes and redo them with
find-replace if they still apply. Voiceover words merged into the clip are
kept. Deletions are trims (time-based) and never touched by re-transcription.

Bulk cleanup (`remove-fillers`, `remove-silences`, `find-issues`) only counts
and acts on the visible source range of a shot fork; hidden source words
and pauses outside the fork are excluded.
