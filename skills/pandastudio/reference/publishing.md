<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Publishing to YouTube + Instagram

## Publishing to YouTube (v1.19+)

PandaStudio uploads directly to YouTube via the Google Data API v3 — no PandaStudio backend, no proxy. Each workspace has its own connected Google accounts; publishing is always scoped to the active workspace.

**Flow** (only when the user asks to publish — never pre-emptively):

1. **Gate:** `youtube.is-configured` → if `false`, tell the user this build can't publish and stop (the other verbs will all fail).
2. **Connect if needed:** `youtube.list-accounts` → if empty, `youtube.connect` (opens the browser for OAuth, up to ~5 min; explicit consent — never on a schedule). Tokens are stored encrypted via `safeStorage`; they never leave the machine.
3. **Publish an export:** `export.publish-youtube --id=$EID --accountId=… --channelId=… --title=… --description=… --tags='[…]' --privacyStatus=unlisted --setThumbnail=true`. Pull `--title`/`--description` from the export row (`generatedTitle`/`generatedDescription`). Returns `{ videoId, videoUrl }`. Check `export.get` first — if `youtubeVideoId` is already set, it's published. `<` and `>` are stripped from title/description automatically (YouTube rejects them) and the description is clamped to 5000 chars, so you don't need to pre-sanitize.
4. **List a channel's videos:** `export.list-youtube --accountId=… --max=50`.

**Hard caveats:**
- **`privacyStatus` defaults to `unlisted`** — never publish public without the user's explicit word; ask "Public, unlisted, or private?" before a first publish.
- **Metadata edits are currently unavailable to agents** — `export.update-youtube` needs the `youtube.force-ssl` scope (pending Google review); it fails with "insufficient scope". Direct the user to YouTube Studio (`https://studio.youtube.com/video/<VID>/edit`). **Thumbnail replacement works now** via `export.update-youtube-thumbnail` (same `youtube.upload` scope).
- **Quota:** on a `quotaExceeded` error, stop and surface it (resets daily) — don't retry blindly.
- **Don't cross workspaces:** if the active workspace lacks the connected account, ASK before switching — publishing to the wrong client's channel is the worst mistake.

(Full arg schemas for every `youtube.*` / `export.*-youtube` verb: `reference/commands.md`.)

## Publishing to Instagram (Reels)

Instagram Reel publishing goes through PandaStudio's license-server broker (which holds a Composio key), not direct OAuth. The rendered video uploads **straight to Composio storage via a presigned URL — the bytes never pass through our server**, so there's no per-publish cost. Requires an activated license.

**Key constraint:** Instagram's API only publishes from **Business or Creator accounts** (a Meta restriction — Personal accounts cannot, via any tool). Always check `instagram.account` first.

**Flow** (only when the user asks to publish):

1. **Connect if needed:** `instagram.connect` → opens the browser for consent, returns `{ connectionId, redirectUrl }`. Then poll `instagram.status --connectionId=…` until `active: true`.
2. **Verify the account can publish:** `instagram.account` → if `publishable` is false (Personal account), tell the user to switch to a Business/Creator account in the Instagram app and reconnect; do not attempt to publish.
3. **Caption (optional):** `llm.generate-caption --id=$EID` writes a Reel caption (punchy hook + a few hashtags) from the transcript on the local model — no API key. Use its output as `--caption` below, or let the user write their own.
4. **Publish an export:** `export.publish-instagram --id=$EID --caption='…' --shareToFeed=true`. Returns `{ mediaId, permalink }`. Long-running: Instagram processes the Reel for ~30-120 s before it's live.

**Hard caveats:**
- **Short-form only.** Instagram takes Reels (9:16, up to ~90 s). Export a Reel-appropriate clip — don't push a long-form 16:9 video.
- **Rate limit:** 25 published posts per 24 h per account; surface a quota error rather than retrying.
- **Hashtags go in the caption** (max 2200 chars). There's no separate tags field.

(Full arg schemas for every `instagram.*` / `export.publish-instagram` verb: `reference/commands.md`.)

