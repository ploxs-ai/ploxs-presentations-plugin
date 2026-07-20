---
name: ploxs-presentations
description: Create, inspect, and edit designed Google Slides presentations through the Ploxs MCP server. Use when the user asks to make a presentation or slide deck, apply a brand style to slides, list or inspect their Ploxs/Google Slides decks, or edit slides (rewrite a slide, add slides, add images or infographics/charts).
---

# Ploxs Presentations

Ploxs turns notes, URLs, document text, and tabular data into designed Google Slides
presentations, and can then edit those decks slide-by-slide. All work happens through
the Ploxs MCP tools (server `ploxs-presentations` at `https://ploxs.com/mcp`).

## Before anything else

1. Call `get_account_status` first. Check:
   - `google.driveConnected` — must be true before creating or editing Google Slides.
     If not, send the user to the `links.settings` / `links.googleDrive` URL from the
     response and wait for them to connect.
   - `billing` / `modelTiers` — if credits are insufficient, tool calls will fail with
     an entitlement error; surface the message and stop.
2. Operations are asynchronous. Creation returns a `jobId`; edits return a `task`.
   Always wait for completion with `wait_for_presentation` (jobs) or
   `wait_for_presentation_edit` (tasks) before making a dependent change or telling
   the user it's done.

## Choosing a style (do this before creating)

Exactly one style source per create/connect call — combining them errors
(`style_choice_conflict`):

- `style_config_id` — preferred. Call `list_style_configs` (optionally with `query`)
  and pick a saved (`manual`), uploaded (`upload`), or recent-deck (`deck:` prefixed)
  style. Reuse the user's saved brand style whenever one exists.
- `style_config` — inline JSON when the user supplies custom branding. A complete
  config needs: `style` (prose description), `colorPalette` with all 7 hex colors
  (primary, secondary, accent, background, surface, text, textMuted), and
  `typographyConfig` (headingFont, bodyFont, googleFontsImport). Check it with
  `validate_style_config` before creating; fix any listed missing fields.
  Put durable brand rules in `brandGuidelines` (dos/donts, colorUsage,
  typographyUsage, imageryUsage, chartUsage) and personality in
  `customStyleDirective` — those persist and shape later edits and images.
- `auto_style: true` — last resort only, when the user has no style preference and no
  saved styles exist. Say explicitly that Ploxs will invent the visual style.

## Creating a presentation

Call `create_presentation` with the content sources you have:

- `markdown` for notes/outline text, `urls` (max 10) for pages to extract,
  `file_texts` (max 20) for pasted document text, `csv_sources` for tabular data.
- Optional: `instructions` (tone/audience/emphasis, max 4000 chars),
  `slide_count` (1–60), `export_to_google: false` to skip Google Drive and keep the
  deck on Ploxs only.

Then `wait_for_presentation` with the returned `job_id` (default timeout 240s; if it
returns `timedOut: true`, wait again rather than assuming failure). On completion,
give the user the `editUrl`/`viewUrl` and **keep the `deckRef`** — every edit tool
needs it. Creation is idempotent for ~10 minutes, so retrying a failed call won't
double-charge.

Daily creation limits exist (25/day wallet accounts, 100/day subscribers); a
`daily_job_limit` error is retryable tomorrow, not now.

## Working with existing decks

- `list_presentations` — decks already known to Ploxs, each with `deckRef`, title,
  and URLs.
- `connect_presentation` — bind a deck Ploxs hasn't seen, from a Slides URL or ID.
  If it returns `ready: false` with an `actionUrl`, give the user that link (a Google
  file-picker authorization), wait for them to approve, then call it again. A deck
  new to Ploxs also requires a style (`style_config_id` or `style_config`) —
  `style_config_required` means you skipped that.
- `get_presentation_outline` — slide numbers, titles, and text for a bound deck.
  **Always fetch this before targeting a slide number**; numbers are 1-based and
  resolved against the live deck.

## Editing

Each mutation queues a task; poll with `wait_for_presentation_edit(task_id)` before
the next dependent edit. At most 3 edit tasks can be active at once.

- `edit_slide` — rewrite/redesign one slide from a `prompt` (≤4000 chars). Optional
  `layout_archetype`; `use_slide_context` defaults true.
- `add_slides` — insert new slides from `content` (≤30k chars). `insertion_placement`
  `before`/`after` requires `anchor_slide_number`; default is `end`.
- `add_image_to_slide` — generated image; optional `prompt`, `art_style`,
  `aspect_ratio` (`16:9|4:3|1:1|9:16|3:4`), `placement`. Either give a `prompt` or
  leave `use_slide_context` on. Also honours `redesign_slide` (default true) and
  `replace_existing_asset` (default false) — see **Adding vs. replacing** below.
- `add_infographic_to_slide` — chart/infographic from `data_description`; optional
  `chart_type`, `chart_title`. Same rule: description or slide context. Also honours
  `redesign_slide` (default true) and `replace_existing_asset` (default false) — see
  **Adding vs. replacing** below.

### Adding vs. replacing (infographics and images)

By default these two tools **add** a new chart/picture and keep any that are already
on the slide (`replace_existing_asset: false`), so a slide can quietly accumulate two
infographics or two images. Redesign of the slide composition is on by default
(`redesign_slide: true`).

When the user's intent is to **change, update, refresh, or swap** the infographic/image
on a slide — not to place a second one next to the existing one — the default should be
to remake the slide: confirm with the user, then pass `replace_existing_asset: true`
(with `redesign_slide: true`) so the old chart/picture is removed and the slide is
redesigned around the new one. A good check-in is: *"This slide already has an
infographic — do you want me to remake the slide with the new one (replacing the old),
or add it alongside?"* — and default to replace-and-remake unless they ask for both.

Only leave `replace_existing_asset` at its default (keep both) when the user explicitly
wants an additional infographic/image kept alongside the existing one. The same choice
applies to these ops inside an `update_presentation` batch.
- `update_presentation` — batch of 1–20 of the above operations in one task. Ops run
  sequentially and each `slide_number` is resolved against the deck **as it exists at
  that step** (an earlier `add_slides` shifts later numbers). The batch stops at the
  first failure and reports what completed. Prefer this over many single calls when
  making several related changes.

## Error handling

Errors come back as text with `(code: <code>, retryable: <bool>)`.

- `google_not_linked` / `google_scope_upgrade_required` — relay the action link,
  wait for the user.
- `presentation_not_connected` — run `connect_presentation` first.
- `slide_not_found` — the message includes the real slide count; re-fetch the
  outline and re-target.
- `active_job_limit` / `rate_limited` — wait for running tasks, then retry.
- `invalid_style_config` — the message lists missing fields; fix and revalidate.
- Non-retryable entitlement/credit errors — report to the user; don't retry.

Never invent a `deckRef` or slide number: always source them from
`create_presentation`/`connect_presentation`/`list_presentations` and
`get_presentation_outline`.
