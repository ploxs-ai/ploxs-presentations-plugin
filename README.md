# Ploxs Presentations

A Claude Code plugin that connects Claude to the hosted Ploxs Streamable HTTP MCP server at `https://ploxs.com/mcp`.

## What it provides

- Create presentations from notes, URLs, file text, and tabular data.
- Reuse saved Ploxs brand styles or validate a custom style.
- List and inspect connected Google Slides presentations.
- Edit slides, add slides, and add generated images or infographics.

## Authentication

The plugin uses OAuth. When Claude Code prompts you to authenticate, sign in to Ploxs and approve access. Ploxs also requires Google Drive to be connected before it can create or edit Google Slides.

## Installation

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install ploxs-presentations@claude-community
```

Start a new session after installation so Claude Code loads the plugin and its MCP tools.

## Links

- Website: https://ploxs.com
- Setup: https://ploxs.com/mcp/setup
- Privacy: https://ploxs.com/privacy
- Terms: https://ploxs.com/terms
