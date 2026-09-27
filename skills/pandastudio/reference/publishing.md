<!-- Part of the pandastudio skill. Detail relocated from SKILL.md for progressive disclosure. -->

# Publishing to YouTube + social (Instagram, TikTok, Facebook, LinkedIn, X)

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

## Publishing to social: Instagram, TikTok, Facebook, LinkedIn, X

Social publishing runs through PandaQueue. Each PandaStudio workspace has its OWN set of connected accounts (an isolated PandaQueue tenant, set up behind the scenes), so accounts connected in one workspace are invisible to every other workspace and every other user. You never handle a key. The video uploads straight to PandaQueue, never through a PandaStudio server. Requires an activated license.

**Flow** (only when the user asks to publish):

0. **Check the plan:** `system.status` → `license.socialPublishing.allowed`. Social publishing is part of Creator, Team and Pro (not Starter). Not allowed: stop, tell the user it comes with those plans, and give them the exported file.
1. **See what's connected:** `social.channels` → `{ channels[{ id, platform, name, status, limits, settings }], networks[{ id, label, connected }] }` for the ACTIVE workspace.
2. **Connect if needed:** `social.connect --network=instagram` (or tiktok, facebook, linkedin, x) opens the network's own sign-in in the user's browser and returns `{ sessionId }` at once. Tell the user to finish in the browser, then poll `social.connect-status --sessionId=…` every few seconds until `state` is `connected` or `failed`. Facebook and LinkedIn page pickers are part of that browser step.
3. **Caption:** `llm.generate-caption --id=$EID` writes a short caption (hook + a few hashtags) on the local model; or use the user's own text. Respect the smallest `limits.maxChars` of the accounts you post to (X is 280).
4. **Confirm with the user:** which accounts, the caption, and now or when. Posting is public and irreversible.
5. **Publish:** `export.publish-social --id=$EID --networks=instagram,tiktok --caption='…'` (or `--channelIds=…` for specific accounts; `--when=schedule --scheduledAt=2026-10-01T18:30:00+01:00` to schedule). It returns `{ jobId }`; `job.wait --id=…` gives `{ postId, state, channels[{ platform, name, status, permalink, error }], summary }`. Tell the user `summary` and the permalinks.
6. **Later:** `social.post-status --postId=… --exportId=$EID` for a scheduled post or one still `publishing`.

**Hard caveats:**
- **Instagram** only publishes from **Business or Creator accounts** (a Meta rule). A video posts as a Reel. It suits 9:16, up to about 90 s.
- **TikTok** wants 9:16 vertical video. **LinkedIn** and **Facebook** take 16:9 or square too. **X** captions are 280 characters.
- **Never cross workspaces:** if the account isn't in the active workspace, ASK before `workspace.switch`. An export from another workspace is refused.
- A `failed` channel carries the network's reason in `error`; report it rather than retrying blindly.

The older `instagram.*` verbs and `export.publish-instagram` still work (they're the same thing with `--networks=instagram`); prefer the social verbs.

(Full arg schemas for every `social.*` / `export.publish-social` verb: `reference/commands.md`.)

