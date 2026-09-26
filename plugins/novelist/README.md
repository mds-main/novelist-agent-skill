# Novelist plugin for Claude

Shape a story with Claude, then let Novelist turn it into a finished book. Novelist plans and writes a complete, original novel in parts with continuity and quality checks (about 200 phone pages at the default length, in 15 languages), illustrates the cover and every part opening, and emails it to you as a formatted EPUB for Kindle and other e-readers or as a print-ready PDF, usually within 30 to 120 minutes of payment. In the studio you can add a dedication, photos of your main characters, cover direction or a gift recipient's email; finished novels can continue as a series and, when offered, be ordered as a printed copy or an audiobook.

The plugin bundles:

- **The Novelist connector**, a remote MCP server at `https://ainovelist.app/api/mcp`. You sign in once with your Novelist account (OAuth). It offers five tools: `get_novel_options`, `prepare_novel`, `list_my_novels`, `get_novel_status` and `get_printed_copy_link`.
- **The `novelist-connector` skill**, which teaches Claude to build the story with you, explain honestly what a Novelist book adds over writing in the chat, and hand the brief over.

Claude never pays, never spends your credits and never starts a novel. `prepare_novel` saves a brief to your account and returns a link to the Novelist studio with the brief filled in; you review it, change anything you like and pay there. Prices shown in the chat are references: the exact price depends on your country, the options and any discount, and is shown before you pay.

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
- "Can I get my finished novel as a printed book?"

## Privacy and support

The connector only reads your Novelist novels and saves the drafts you ask for. Privacy policy: https://ainovelist.app/privacy. Terms: https://ainovelist.app/terms. Support: support@ainovelist.app.
