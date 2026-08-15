---
name: ploxs-presentations
description: Create and edit designed Google Slides decks through the production Ploxs MCP server (ploxs.com), including authoring the slide HTML yourself. Use when the user asks to make or edit a presentation or slide deck, apply a brand style to slides, design slides as HTML frames, or list/inspect their Ploxs decks.
---

# Ploxs Presentations

Production server `ploxs` at `https://ploxs.com/mcp`.

## Deck checklist

Every new deck runs these in order. Never skip one, never claim a step you did not run.
Tool results name the next step by number — trust that over your memory of this page.

1. **`get_account_status`** — obey `mcp.initialCreationMode` (see below).
2. Chat image attachments — **`prepare_presentation_image_upload`** once, hand the user
   the `uploadUrl`, then **`get_presentation_image_upload_status`** until `ready`.
3. Style — one saved style the user picked, or one complete inline `style_config`.
4. Create **once**, passing `creator_choice` — **`create_presentation`** (ploxs), or
   **`get_html_frame_spec`** then **`create_presentation_from_html`** (native). Keep the
   returned `jobId` and `statusUrl`.
5. **`wait_for_presentation`** with that `jobId`; if the client exposes only the compatible
   **`get_presentation_status`** name, use it instead. Each call waits ~45s; `timedOut: true`
   means **still building, not failed** — call the same tool again immediately, as many
   times as it takes.
6. Finish with the full Google Slides edit and view URLs on separate lines. If you never
   got them, give the user the `statusUrl`. Never ask the user for a link.

## Start

1. Call **`get_account_status`** before every initial deck. If
   `google.driveConnected` is false, give the user `links.googleDrive` / `links.settings`
   and wait. Credit or entitlement errors block Ploxs generation and AI edits, but
   never block credit-free native HTML-frame creation.
2. Obey `mcp.initialCreationMode`:

   | Mode | Required behavior |
   | --- | --- |
   | `ask` (default) | Always ask: “Should Ploxs create and design it, or should I create it natively and use Ploxs only to convert it?” Ask even when the request appears to choose a path, then pass the answer as `creator_choice`. Both creation tools **refuse** an ask-mode call without it, so nothing is saved by skipping the question. |
   | `ploxs` | Use `create_presentation`; never author initial HTML frames. |
   | `native` | Author the initial slides and use `create_presentation_from_html`; never call `create_presentation`. |

   These modes apply only to initial creation. Existing-deck edits always use the edit
   tools.
3. If the topic itself is missing, ask one short topic question before spending a job.

Creation returns a `jobId` and a `statusUrl`; edits return a `task`. Wait for completion
before a dependent action or a completion claim. Keep the `jobId`: later outline/edit
tools accept it directly as `deck_ref`, so a same-chat edit never requires the user to
paste the Slides URL.

The presentation is the deliverable. Keep chat to required questions/actions, terse
status updates, and final links unless the user asks for explanation. Do not narrate
reasoning, write a prose slide plan, explain design choices, or print frame HTML before
submitting it.

## Chat image attachments

For chat image attachments, call **`prepare_presentation_image_upload`** with their exact
unique filenames, give its single `uploadUrl` to the user, and wait until
**`get_presentation_image_upload_status`** returns `ready`; the user may upload them in
several selections from different folders. Pass the `sessionId` as
`asset_session_id` to either creation tool. Native frames reference returned ids with
`<img data-ploxs-image-id="presentation_image_N">`. Do not add descriptions or mapping.

## Choose a style

Use exactly one style source per call.

- Call **`list_style_configs`** only when the user refers to a saved, uploaded, or recent
  deck style. Read the full config; a matching name does not prove its palette or design.
  Ask if several match or the config conflicts with the request.
- Otherwise pass a complete inline **`style_config`**: `style`, all seven
  `colorPalette` roles, and `typographyConfig`. Put persistent rules in
  `brandGuidelines` and personality in `customStyleDirective`.
- Let the destination tool validate the style before queueing. Use
  **`validate_style_config`** only to diagnose a reported config problem or when the user
  explicitly requests validation; do not duplicate routine config payloads.
- Use **`auto_style: true`** only when the user gives no usable direction. Never combine
  style sources.

## Ploxs creation

Call **`create_presentation`** once with the source material (`markdown`, `urls`,
`file_texts`, `csv_sources`), optional `asset_session_id`, `instructions` / `slide_count`,
and one style source. It creates a new Google Slides file.

Then use the completion tool from checklist step 5 with the `jobId`. If `timedOut` is
true, call the same tool again with the same `jobId`. Keep the completed `deckRef`.

## Native creation

1. Call **`get_html_frame_spec`** once with the chosen style. Leave `include_chart_spec`
   at its default (true) — the chart plumbing is unguessable, so a deck that discovers
   mid-author that a slide compares quantities cannot add one without it. Pass false only
   for a deck you know carries no quantitative data. **Keep the returned `styleRef`.**
2. Treat HTML/CSS as the planning medium. Read the structured stage, palette, typography,
   briefs, contract, overlays, `patterns.catalog`, `techniques`, the two worked examples,
   and any requested chart protocol, then compose every
   **final** frame directly. Do not first translate the deck into prose; do not create a
   prototype, sample, validation slide, outline, or design memo; do not spend one turn per
   slide. The two examples bracket the density range rather than sampling it — a bare
   typographic statement at the floor, a full command dashboard at the ceiling. Build
   between them from `patterns.catalog`: never reuse example copy or numbers, and never
   repeat one layout throughout the deck.
3. Put the finished HTML directly in the complete `frames` array and send it to
   **`create_presentation_from_html`** once, passing `style_ref` (the `styleRef` from
   step 1), any ready `asset_session_id`, and a plain-text title. Never retype the style config on this call: a single
   missing palette key rejects the entire authored deck, which is the most expensive way
   this flow can fail.
   Creation performs converter validation before queueing, and invalid frames consume no
   job. Do not emit the HTML in chat or an intermediate document, and never repeat the
   large frame payload through a separate validation call.
4. If creation returns `invalid_html_frames`, repair only the named final frames and
   resubmit. Otherwise use the completion tool from checklist step 5 with its `jobId`;
   repeat only if `timedOut` is true, then keep the `deckRef`.

Frames convert exactly as authored. Follow the returned contract literally, especially:

- one fixed 1280×720 frame per slide with one top-level element and inline CSS; the root
  explicitly owns that full canvas and authors every inset/alignment itself because Ploxs
  adds no margins, centering, scaling, or repositioning to native frames
- no `font-family` declarations except deliberate monospace; deck fonts already apply
- use the whole style as one design system while varying composition by message
- use only supplied numbers and label estimates, projections, and dates on-slide
- **use the icon library** — `icons.mandate` resolves `__ICON_<keywords>__` placeholders to
  professional inline SVG from a 200,000+ icon set. Give icons one consistent role across
  the deck (metric tiles, capability rows, step markers). A deck with zero icons has left
  the cheapest source of visual quality unused; a restrained style uses fewer and larger
  icons, not none
- **put a real chart on any slide whose point is a comparison, trend, distribution, or
  part-of-whole** — copy the returned Chart.js plumbing exactly (unique canvas id, loader,
  `dataset.initialized`, `animation: false`). CSS-drawn bars are decoration, not data, and
  restating the numbers in prose wastes the slide
- when data is not comparative, use HTML markup so it converts to editable Slides shapes
  instead of a flat chart image
- `techniques.degrades` lists editability trades, not quality warnings — gradients,
  pseudo-element decoration, and clipped shapes all render as authored, so do not strip
  decoration to avoid them

## Existing decks

- Use **`list_presentations`** to find known decks.
- Use **`connect_presentation`** for another Slides URL/id. If it returns an `actionUrl`,
  give it to the user and retry only after approval.
- Fetch **`get_presentation_outline`** before targeting a slide number.
- Use `edit_slide`, `add_slides`, `add_image_to_slide`, or
  `add_infographic_to_slide` for one change; prefer **`update_presentation`** for an
  ordered compound change. Wait with **`wait_for_presentation_edit`** before dependent
  edits, and keep no more than three edit tasks active per key.
- Creation tools never update a deck. Calling either again creates a duplicate Drive
  file, so edit the live `deckRef` instead.

When the user wants to change, refresh, or swap an existing image/chart, confirm that it
should be replaced and set `replace_existing_asset: true` (`redesign_slide` defaults to
true); otherwise the asset tools add another one. In a batch, slide numbers resolve
against the deck state at that operation.

## Errors and handoff

- `native_mode_preference` → use native HTML creation.
- `ploxs_mode_preference` → use Ploxs creation.
- `google_not_linked` / `google_scope_upgrade_required` → relay the action link and wait.
- `invalid_html_frames` → repair the reported frames; never retry unchanged.
- `style_config_required` / `style_choice_conflict` / `invalid_style_config` → correct
  the single style input.
- `presentation_not_connected` → call `connect_presentation`.
- `slide_not_found` → fetch the live outline again.
- `active_job_limit` / `rate_limited` → wait, then retry.
- Entitlement or credit errors from Ploxs generation or edit tools: report the billing
  link and wait. Native HTML-frame creation remains credit-free. After the user
  recharges or usage becomes available, continue in this same chat and retry the
  original billable tool call with the existing job or deck context.

Never invent a `deckRef` or slide number. On completion, label the Google Slides edit and
view URLs and put each full URL on its own line.
