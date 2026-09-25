# Novelist plugin for Claude

Plan a complete custom novel in a conversation with Claude, then review and pay for it on [ainovelist.app](https://ainovelist.app). Novelist writes an original novel (about 200 phone pages at the default length, in 15 languages) and emails it to you as an EPUB or PDF, usually within 30 to 120 minutes of payment.

The plugin bundles:

- **The Novelist connector**, a remote MCP server at `https://ainovelist.app/api/mcp`. You sign in once with your Novelist account (OAuth). It offers four tools: `get_novel_options`, `prepare_novel`, `list_my_novels` and `get_novel_status`.
- **The `novelist-connector` skill**, which teaches Claude how to shape a good novel brief with you and hand it over.

Claude never pays, never spends your credits and never starts a novel. `prepare_novel` saves a brief to your account and returns a link to the Novelist studio with the brief filled in; you review it, change anything you like and pay there.

## Install

Claude Code:

```bash
claude plugin marketplace add mds-main/novelist-agent-skill
claude plugin install novelist@novelist
```

Claude (web, desktop, mobile): Settings, Connectors, Add custom connector, and enter `https://ainovelist.app/api/mcp`.

## Example prompts

- "Write me a cozy mystery novel set in 1950s Lisbon, with a retired tram driver as the detective."
- "What does a Novelist novel cost, and which languages can it be written in?"
- "How is my Novelist novel coming along?"

## Privacy and support

The connector only reads your Novelist novels and saves the drafts you ask for. Privacy policy: https://ainovelist.app/privacy. Terms: https://ainovelist.app/terms. Support: support@ainovelist.app.
