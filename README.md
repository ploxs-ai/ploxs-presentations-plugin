# Ploxs Presentations

Ploxs connects Claude to designed Google Slides through the production Streamable HTTP
MCP server at `https://ploxs.com/mcp`.

## Two ways to build a deck

- **Ploxs generates the slides** — give it notes, URLs, document text, or CSV data and a
  saved or custom brand style.
- **Claude authors the slides** — Claude reads the selected Ploxs style as concrete
  design tokens, writes every final slide once, and asks Ploxs to validate and convert
  them exactly as authored into Google Slides.

Choose the default behavior on the Ploxs MCP setup page: **Ask every time** (the default),
**Ploxs creates**, or **Assistant creates**. Ask mode always confirms the creator before
a new deck; the other modes route directly. Later edits still go through Ploxs.

Both paths create a normal Ploxs deck, so the editing tools—rewrite a slide, add slides,
or add generated images and infographics—work on either.

### How the authored path behaves

- **Your design system carries through.** Claude receives the stage geometry, palette,
  type scale, fonts, brand rules, imagery direction, and other style guidance before
  authoring.
- **Frames are authored as real HTML/CSS compositions.** The fixed slide canvas can use
  editorial layouts, dashboards, product surfaces, timelines, metric walls, and other
  compositions that fit the content and brand.
- **Charts use Chart.js**, built only from figures in your material and captured during
  conversion. The create call validates their export contract before queueing.
- **The initial deck is created once, then edited in place.** Later changes use Ploxs'
  editing tools on the same Google Slides file, preserving its link.
- **Figures stay honest on the slide.** Projected, guided, estimated, or dated values are
  labeled in the deck rather than only in chat.

## Install

In Claude Desktop or claude.ai:

1. Open **Customize** → **Plugins**.
2. Choose **Add marketplace** → **Add from a repository** and enter:

   ```txt
   https://github.com/vipinsanthosh/ploxs-presentations-plugin.git
   ```

3. Install **Ploxs Presentations**.
4. Open the plugin, click **Connect**, and approve the Ploxs OAuth sign-in.
5. Start a new chat so the current tools and skill load.

In Claude Code, install directly from this repository:

```txt
/plugin marketplace add vipinsanthosh/ploxs-presentations-plugin
/plugin install ploxs-presentations@ploxs
```

If you use the Claude community marketplace instead:

```txt
/plugin marketplace add anthropics/claude-plugins-community
/plugin install ploxs-presentations@claude-community
```

## Long-running deck builds

Building a deck typically takes 1–4 minutes, occasionally longer. Claude waits for it in
a single tool call, and the Ploxs server keeps that call open for up to seven minutes.

Claude Code users can give slow builds additional client-side headroom:

```sh
MCP_TOOL_TIMEOUT=450000 claude
```

If the client times out, the job continues on Ploxs and Claude can resume by checking its
status.

## Before creating decks

Sign in at [ploxs.com](https://ploxs.com), connect Google Drive in Settings, and make sure
the account has usage available. Ploxs creates the Slides file in your own Drive, so
Drive linking requires your approval.

## Contents

| Path | Purpose |
| --- | --- |
| `.claude-plugin/marketplace.json` | Marketplace entry used when this repository is added |
| `.claude-plugin/plugin.json` | Plugin manifest |
| `.mcp.json` | Production MCP server configuration (`https://ploxs.com/mcp`) |
| `skills/ploxs-presentations/SKILL.md` | Generation, HTML conversion, editing, charts, and error handling |

Reinstall or update the marketplace entry to pick up skill changes. The skill ships in
this repository while the MCP tools it drives run on Ploxs.

## Links

- Website: https://ploxs.com
- Setup: https://ploxs.com/mcp/setup
- Privacy: https://ploxs.com/privacy
- Terms: https://ploxs.com/terms

## License

MIT — see [LICENSE](./LICENSE).

## Contributing

This repository is generated from the Ploxs monorepo. Direct edits are overwritten on the
next release; please report issues at privacy@ploxs.com.
