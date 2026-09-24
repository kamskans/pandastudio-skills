# Transcript-based editing — PandaStudio's signature feature

The reason humans pick PandaStudio over Premiere is that you edit by
**deleting words from the transcript**, not by scrubbing the timeline. The CLI
exposes the same model. Every deletion becomes a **trim region** the export
skips — identical to deleting the word in the editor's transcript pane.

## Read clip state first

Before `transcript.transcribe` or `audio.clean`, call `project.read` and read
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
- `audioCleaned: true` → skip `audio.clean`. `echoReduced: true` means
  room-echo reduction is on too (`roomRt60Ms` = the reverb time it measured).
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
# → { removedCount, fillersRemoved, repeatsRemoved, trimsAdded }

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
words. Each word has `startMs` (source) and `editedStartMs` (output; `null` =
inside a trim).

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

## Silences

`transcript.remove-silences` runs the SAME two passes as the UI Remove Silences
button and unions them: (1) transcript word-gaps (leading, between-word,
trailing) and (2) ffmpeg audio-level `silencedetect` (noise -30 dB) on each
clip's media, which catches real dead air the transcript misses when
speech-to-text invents phantom words over quiet stretches. Default threshold
600ms; don't hand-pick a higher value "to be safe". `--paddingMs=100` sets the
margin kept around every word, so it never cuts into words. Run it AFTER the
content cleanup. If the user already removed silences in the UI, a fresh
`project.read` shows the new `trimCount` / `editedDurationMs` /
`totalTrimmedMs`: treat that as done.

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
