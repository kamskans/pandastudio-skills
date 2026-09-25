# Motion-graphic templates: finding the right one

PandaStudio ships slot templates (text, colours, lists, images you fill in)
grouped into families. The runtime list is the source of truth; this page is
how to search it. The per-template catalog with slots and placement notes is in
[motion-templates.md](./motion-templates.md).

```bash
pandastudio motion.list --json            # everything live (MCP: motion_list)
```

## Search and filter

| Want | Call |
|---|---|
| A template for a job | `motion.list --query="subscribe"` (ranked over name, description, family and tags; intent words like `vs`, `cta`, `quote`, `chart`, `timer`, `name` work) |
| One family | `motion.list --family=stats-data` (comma-separate for several) |
| Tagged templates | `motion.list --tags=quote,testimonial` (any of) |
| Only one aspect | `motion.list --aspect=9:16` |
| The family counts | the `families` array in any `motion.list` result |
| Retired ones too | `motion.list --includeRetired` (only to inspect an old project's graphic) |
| Skip registry blocks | `motion.list --includeBlocks=false` |

Families: `titles`, `text-behind`, `captions`, `callouts`, `lists-steps`,
`stats-data`, `comparisons`, `product`, `panels`, `social-proof`, `social`,
`intro-outro`, `end-cards`, `lower-thirds`. The editor's Graphics tab shows the
same families as chips, with search, Recently used and Featured rows; the Lower
3rds tab shows the `lower-thirds` family.

```bash
# Inspect the chosen template's slots and defaults before rendering
pandastudio motion.list --query="stat" --json | \
  jq '.data.templates[0] | {id, family, slots, defaults, aspectRatios, durationMs}'
```

Pass only the slots you change; every omitted slot (including colours) falls
back to the template's default, and a workspace brand kit fills colour slots
tagged with a brand role.

## Aspect ratios

Each template lists the layouts it is authored for in `aspectRatios`. A 9:16
layout is a real portrait layout, not a shrunk 16:9 one. Render at the
project's aspect (`--aspectRatio=9:16`); filter with `--aspect=9:16` to see
what fits a vertical project.

## Retired templates

Retired templates (`retired: true`) have a `replacement`. They're hidden from
`motion.list` and the editor, but they still render and re-render, so older
projects keep their look. Rendering one returns a `warnings` entry naming the
replacement: tell the user and use the replacement for anything new. Move a
placed graphic onto its replacement only when asked:

```bash
pandastudio project.update-motion-graphic --id=$P --overlayId=$OVERLAY --templateId=<replacement>
```

Text, list items and same-named colours carry over; the job result lists
`switched.carried` and `switched.dropped`. The retired → replacement table is
in [motion-templates.md](./motion-templates.md#families-search-and-retired-templates).

## When no template fits

Use `motion.render-html` to render your own HTML/CSS/JS on the same engine
(no slots), or `motion.catalog` to search the ~390 HyperFrames components and
blocks for a named look. See [motion-philosophy.md](./motion-philosophy.md)
and [custom-html.md](./custom-html.md).

## Registry blocks

`motion.list` also returns `registryBlocks`: standalone HyperFrames
compositions (media-treatment overlays, handwritten annotations, decision
trees, a code snippet, beat-freeze cut). They have no slots: render the
block's `htmlPath` with `motion.render-html` and edit the HTML to change text.
Retired blocks are left out unless `--includeRetired`.

```bash
HP=$(pandastudio motion.list --json | jq -r '.data.registryBlocks[] | select(.name=="caption-kinetic-slam") | .htmlPath')
JOB=$(pandastudio motion.render-html --htmlPath="$HP" --durationMs=4000 --json | jq -r '.data.jobId')
```
