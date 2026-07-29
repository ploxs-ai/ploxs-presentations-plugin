---
name: ploxs-presentations
description: Create and edit designed Google Slides decks through the production Ploxs MCP server (ploxs.com), including authoring the slide HTML yourself. Use when the user asks to make or edit a presentation or slide deck, apply a brand style to slides, design slides as HTML frames, or list/inspect their Ploxs decks.
---

# Ploxs Presentations

Production server `ploxs` at `https://ploxs.com/mcp`.

Everything is async: creation returns a `jobId`, edits return a `task`. Always wait for
completion before a dependent step or before telling the user it's done.

## Start

1. **`get_account_status`** — `google.driveConnected` must be true before creating or
   editing Google Slides. If it isn't, give the user the `links.googleDrive` /
   `links.settings` URL and wait. It also returns `billing` and `modelTiers`; on
   insufficient credits, surface the message and stop rather than retrying. Read
   `mcp.initialCreationMode` before choosing the initial creation path:
   `conversion_only` overrides every other routing signal, so author the slides yourself
   and use conversion without asking which path to take.
2. **Pick a style** (see below).
3. If the topic is genuinely unspecified, ask one short question and stop. Don't guess a
   topic and burn a deck job on it.

Before beginning a creation or conversion, tell the user which path you are taking. Keep
them oriented with short progress updates while you author frames and while a queued job
is running.

## Styles

Exactly one style source per call — combining them errors (`style_choice_conflict`).

- **`list_style_configs`** — every style available: saved (`manual`), uploaded from brand
  files (`upload`), and recent decks (`deck:` prefixed). Optional `query` (≤120 chars),
  `limit` (1–50), `include_deck_styles`. Each entry includes the full config JSON, so you
  can copy one, tweak it, and pass it inline.
- **`style_config_id`** — use when the user identifies a specific saved/uploaded/deck
  style and the match is unambiguous. The existence of a saved style is not permission to
  choose it.
- **`style_config`** — the normal default when the user did not identify a saved style.
  Build it from their request, brand context, audience and subject, then validate it.
  Complete means: `style`
  (prose description), `colorPalette` with all 7 hex colors (primary, secondary, accent,
  background, surface, text, textMuted), and `typographyConfig` (headingFont, bodyFont,
  googleFontsImport). Put durable rules in `brandGuidelines` (colorUsage, typographyUsage,
  layoutUsage, imageryUsage, chartUsage, dos, donts) and personality in
  `customStyleDirective` — those persist into later edits and generated images, unlike
  top-level `instructions`.
- **`validate_style_config`** — validates/normalizes a candidate before you spend a job on
  it. Optional `strict`.
- **`auto_style: true`** — last resort, only when the user gives no direction and you
  cannot construct a reasonable style config. The presence of unrelated library styles
  does not change this. It must be explicit; omitting every style source returns
  `style_config_required`. Say that Ploxs will invent the style. Never use it for authored
  frames.

A style's name is only a reference label. It may be a brand name, a deck title, or
arbitrary text; it does **not** prove what colors or visual language the config contains.
Always read the actual `colorPalette` hexes and the full config before considering it.
Use a name match when the user mentions that name. If multiple styles match, ask which
one. If the user requests something like "Amazon colors" but an Amazon-named saved style
has a conflicting palette, show the mismatch and ask for confirmation before using it.
Never silently substitute a name match for the user's stated visual direction.

## First work out which job this is

Ploxs supports two initial creation paths. Route from what the user explicitly asks Ploxs
and their AI assistant to do, not from how complete the supplied material is.

| | |
|---|---|
| **Generation** — Ploxs creates and designs the initial deck | `create_presentation` |
| **Conversion** — you create the initial slides; Ploxs converts them | `create_presentation_from_html` |

- **Account override:** if `mcp.initialCreationMode` is `conversion_only`, always use
  conversion. Do not ask which path and do not call `create_presentation`. Existing-deck
  edits still go through Ploxs.
- Use **generation** when the user explicitly asks Ploxs to create, write, or design the
  presentation. Notes, URLs, files, tables, drafted copy, or exact figures can all still
  use generation when that is what the user asks Ploxs to do.
- Use **conversion** when the user explicitly asks you/their AI assistant to create or
  design the slides and asks Ploxs only to convert/export them to Google Slides, or when
  they explicitly ask to use Ploxs for conversion.

If the request does not say who should create the initial slides, **ask one short
question** before doing either: "Would you like Ploxs to create and design the
presentation, or should I create the slides and use Ploxs only to convert them to Google
Slides?" Do not infer the answer from raw material, complete content, repo files, exact
layouts, or figures. Once decided, tell the user which path you are starting.

## Generation — Ploxs writes and designs the deck

**`create_presentation`**:

- `markdown` — notes or outline text; `urls` (≤10) — pages to extract;
  `file_texts` (≤20) — pasted document text; `csv_sources` (≤20) — tabular data
- `instructions` (≤4000) — tone, audience, emphasis
- `slide_count` (1–60) — omit to let the content decide
- one style source, as above
- `idempotency_key` — optional; retries are already idempotent for ~10 minutes

Every successful creation is uploaded to Google Drive; there is no local-only or
Ploxs-only output option.

Then **`wait_for_presentation`** with the `job_id` (`timeout_seconds` 5–420, default 420 —
seven minutes, enough for a slow build in one call). If it returns `timedOut: true` the
job is still running: wait again, don't assume failure. **`get_presentation_status`** gives
the same job state as a one-shot check. Tell the user when the job is queued and give a
brief update if another wait is needed.

On completion, **keep the `deckRef`** — every edit tool needs it. In the final response,
make the destination unmistakable: label the edit and view links and put each full URL on
its own line. Do not hide either URL behind a few linked words inside a sentence.

## Conversion — you author the slides as HTML frames

Use this when the routing above selects conversion, including the account-level
`conversion_only` override. Frames convert exactly as authored — no regeneration, no
restyling.

**Call `get_html_frame_spec` with your style, then start writing frames immediately.**

That one response is the whole brief: stage size and safe area, palette, fonts, type
scale, HTML-first creative direction, the binding export contract, the Chart.js protocol
with a copy-ready widget, and any overlay exclusion zones. Follow it literally; don't
re-derive tokens or invent colors, fonts and sizes.

Create **real HTML/CSS compositions, not conventional slide templates.** Treat each fixed
1280×720 frame as a polished static web canvas. When the content and style support it,
design landing-page heroes, product or dashboard surfaces, editorial spreads, command
centres, metric walls, bento/data grids, timelines, comparison interfaces, and layered
information systems. Use semantic nesting, CSS Grid/Flexbox, borders, radii, shadows,
gradients, pseudo-elements, real images, icon placeholders, and charts within the returned
contract.

Treat the **entire resolved style config as a binding design system**: palette roles,
typography, type scale, alignment, spacing rhythm, density, border/radius/shadow language,
personality, brand rules, imagery, iconography, charts, and decorative grammar. Commit to
the strongest coherent interpretation it supports. A palette-and-font swap over repeated
generic cards is not style implementation. Do not dilute a distinctive direction into
safe corporate slides. Fidelity can also be deliberately quiet—minimal and luxury styles
should commit through typography, scale, whitespace, and precision rather than arbitrary
decoration.

Keep the design system consistent while varying the composition and focal structure to
fit each slide's message. Use the safe canvas intentionally: strong negative space may be
part of the art direction, but tiny centred content and accidental underfilling are not.
Use UI as a static visual language, never as fake interaction; do not add controls, empty
media placeholders, or unsupported behaviours.

Write **every frame in one pass, back-to-back in a single turn.** No outline document, no
design memo, no turn-per-slide. Each frame is the inner HTML of one slide; frame 1 becomes
slide 1. With subagents, give each worker one frame plus the spec's tokens — send the full
contract and `charts` block only to a worker whose slide has a chart.

Hold these constant as you write:

- sizes copied from `briefs.layout`, not a fresh scale per slide
- the deck-wide visual language derived from `creativeDirection` and every returned brief
- a small semantic class vocabulary (`.stage`, `.eyebrow`, `.panel`, `.metric`, `.rail`,
  `.note`, `.icon`) without forcing every slide into the same structure. Class names
  auto-suffix per frame so they cannot collide; per-frame prefixes like `.f1-panel` only
  bloat the CSS
- **no `font-family` anywhere** — the deck already applies the heading font to `h1`–`h6`
  and the body font to everything else
- numbers only from the material you were given. Anything projected, guided, estimated or
  dated gets labelled on the slide itself (`2026 (guided)`, `as of 2021`) — a caveat in
  chat doesn't travel with the deck

**Charts:** follow `spec.charts` and copy its widget's plumbing verbatim — unique
`<canvas id>`, its loader, `dataset.initialized`, `animation: false`. Anything less
exports as an empty box. If the data doesn't need a chart, build it as markup; that
converts to editable Slides shapes rather than a flat image.

**`validate_html_frames`** — free, no job, all frames in one call. Run it when any frame
has a chart: those failures are silent and cost a whole job. Pure-markup decks can go
straight to submit, which reports the same warnings, and errors queue nothing. Warnings
quote what they matched — if you can't find that thing in your frame, it isn't about your
frame.

**`create_presentation_from_html`** — `frames` (`{name, html}` in slide order), `title`
(plain text, not HTML-escaped), the same style you gave the spec, optional `instructions`
(stored for later edits). It always uploads the result to Google Drive. Then
`wait_for_presentation` exactly as for generation, keep the `deckRef`, and present the
final edit/view URLs clearly on their own labeled lines.

## Existing decks

- **`list_presentations`** — decks Ploxs knows, each with `deckRef`, title and URLs.
  Optional `limit` (1–100).
- **`connect_presentation`** — bind a deck Ploxs hasn't seen: `presentation` is a Slides
  URL or raw id. A deck new to Ploxs also needs a style (`style_config_id` /
  `style_config`) — `style_config_required` means you skipped it. If it returns
  `ready: false` with an `actionUrl`, give the user that link (a Google file-picker
  authorization), wait for approval, then call it again.
- **`get_presentation_outline`** — `deck_ref` → numbered slides with titles and text.
  **Always fetch this before targeting a slide number**; numbers are 1-based against the
  live deck.

## Editing a deck

Generation and conversion each happen **once**, for the initial deck.
`create_presentation` and `create_presentation_from_html` cannot update a deck — calling
either after a deck exists builds a *second* deck and a second Drive file, leaving the
user's original link stale. Every later change goes through these tools on the `deckRef`,
which run Ploxs' AI against the live file and edit it in place.

Each takes `deck_ref` plus an optional `idempotency_key`:

- **`edit_slide`** — `slide_number`, `prompt` (≤4000). Optional `layout_archetype`
  (≤120); `use_slide_context` defaults true.
- **`add_slides`** — `content` (≤30000). Optional `slide_count` (1–20),
  `insertion_placement` (`before` / `after` / `end`, default `end`; `before`/`after`
  require `anchor_slide_number`), `instructions` (≤4000).
- **`add_image_to_slide`** — `slide_number`. Optional `prompt` (≤2000), `art_style`
  (≤240), `image_type` (≤120), `aspect_ratio` (`16:9|4:3|1:1|9:16|3:4`), `placement`
  (≤120). Give a `prompt` or leave `use_slide_context` on.
- **`add_infographic_to_slide`** — `slide_number`. Optional `data_description` (≤4000),
  `chart_type` (≤120), `chart_title` (≤240), `placement`. Same rule: a description or
  slide context.
- **`update_presentation`** — `operations`, 1–20 of `edit_slide` / `add_image` /
  `add_infographic` / `add_slides`. They run sequentially and each `slide_number` resolves
  against the deck **as it exists at that step** (an earlier `add_slides` shifts later
  numbers). Stops at the first failure and reports what completed. Prefer this over many
  single calls.
- **`wait_for_presentation_edit`** — `task_id`, `timeout_seconds` (5–420, default 420).
  **`get_presentation_edit_status`** for a one-shot check. Poll before a dependent edit;
  at most 3 edit tasks active per key.

**Adding vs. replacing:** the two asset tools **add** by default and keep whatever is
already on the slide (`replace_existing_asset: false`), so a slide can quietly accumulate
two charts. `redesign_slide` defaults true. When the user wants to change, refresh or swap
the existing image/chart, confirm and pass `replace_existing_asset: true` with
`redesign_slide: true`. Same choice inside an `update_presentation` batch.

## Errors

Returned as text with `(code: …, retryable: …)`.

- `google_not_linked` / `google_scope_upgrade_required` — relay the action link, wait
- `invalid_html_frames` / `html_frames_invalid` — names the frames and rules; fix and
  resubmit, never retry unchanged
- `style_config_required` — pass the style the frames were authored against
- `style_choice_conflict` — you sent more than one style source
- `invalid_style_config` — the message lists the missing fields; fix and revalidate
- `conversion_only_preference` — the account requires agent-authored conversion; use the
  HTML frame flow, not generation
- `presentation_not_connected` — run `connect_presentation` first
- `slide_not_found` — the message has the real slide count; re-fetch the outline
- `active_job_limit` — wait for running work to finish, then retry
- `rate_limited` — pause briefly, then retry
- entitlement / credit errors — report to the user, don't retry

Never invent a `deckRef` or a slide number — source them from `create_presentation`,
`create_presentation_from_html`, `connect_presentation`, `list_presentations` and
`get_presentation_outline`.
