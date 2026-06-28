<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Captions, AI metadata, YouTube thumbnails

## Captions

```bash
pandastudio caption.set-template --id=$ID --templateId=neon
pandastudio caption.toggle --id=$ID --enabled=true
pandastudio caption.set-style --id=$ID --color="#fff" --highlightColor="#34B27B" \
  --strokeWidth=3 --strokeColor="#000" --positionY=85
# Change the caption font (system font or a loaded custom font; unloaded fonts
# fall back to a system default):
pandastudio caption.set-style --id=$ID --fontFamily="Georgia"
```

Templates: `classic | modern | minimal | bold | spotlight | boxed | neon | colored | editorial`. Captions read words from the project's merged transcript — so you must transcribe first.

- `editorial` is a magazine-emphasis style: the word being spoken RIGHT NOW renders large (and takes an accent color) while the rest of the line shrinks, so one big word sweeps across the line in time with the speech. Best with short `--wordsPerLine` (4-6) so each line reads as a headline. Great for talking-head explainers and punchy hooks.

## AI metadata (uses bundled local LLM)

```bash
pandastudio llm.generate-title --id=$ID --json
# → { title: "..." }

pandastudio llm.generate-description --id=$ID --json
# → { description: "..." }

pandastudio llm.generate-timestamps --id=$ID --maxChapters=8 --json
# → { timestamps: [ { timeMs, label }, ... ] }
```

These are READ-ONLY — they return text for you to display or stash on the export-library entry; they don't mutate the project. Perfect for auto-populating the YouTube upload box after `export.start` finishes.

## YouTube thumbnails (v1.18+)

PandaStudio can generate YouTube thumbnails via Replicate's `openai/gpt-image-2` model using **the user's own Replicate API key** — PandaStudio never pays for or proxies these calls. Before any thumbnail verb will work, the user must open **Settings → Integrations** and paste a key from https://replicate.com/account/api-tokens. The key is stored encrypted via the OS keychain (Keychain on macOS, DPAPI on Windows, libsecret on Linux).

Requires a Replicate key set in **Settings → Integrations** (not via CLI — by
design, so it never lands in shell history). If a verb returns "No Replicate API
key set", tell the user to add one rather than looping.

**Verbs:**
- `export.generate-thumbnail --id=$EID` — gpt-image-2 (3:2). Two ways to drive it:
  - **Controlled (preferred):** `--subject="a phone showing a first $79 Stripe sale"`
    `--hook="FIRST SALE"` `--reaction=excitement` `--layout=reaction-split`. The
    prompt is assembled deterministically — no transcript/LLM needed. `reaction` ∈
    {excitement (default), shock, delight, awe, curiosity, pride, determination,
    relief}; `layout` ∈ {reaction-split (default), subject-hero, before-after,
    big-face, product-only, versus}. `product-only`/`versus` ignore the person.
  - **Auto:** omit subject/hook and the local LLM suggests the brief from the
    transcript (needs a transcript).
  - `--prompt="…"` overrides everything with a verbatim prompt. `--referenceImagePath=…`
    is the person's face (passed as input_images; skipped for person-less layouts).
- `export.edit-thumbnail --id=$EID --editPrompt="…"` — chat-style refine; **one
  change per call** (runs at `low` quality). Each edit is reverable.
- `export.set-thumbnail --id=$EID --sourcePath=…` — use a user-supplied file
  (PNG/JPEG/WebP).
- `export.revert-thumbnail --iterationId=…` / `export.clear-thumbnail` — history
  is in `entry.thumbnailIterations[]` (via `export.get`).

**Prompt shape that works** (gpt-image-2 rewards photo-language): one focal
subject in photo terms ("extreme close-up of…"); specify lighting; **quote
on-image text** (`reading "GAME OVER"` — its text rendering is sharp); on edits,
lock what shouldn't change and iterate one thing at a time.

(Full arg schemas: `reference/commands.md`.)

