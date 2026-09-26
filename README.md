# Novelist Agent Skill

[![skills.sh](https://skills.sh/b/mds-main/novelist-agent-skill)](https://skills.sh/mds-main/novelist-agent-skill)

Public skill distribution for the Novelist Agentic API: browse and buy novels, generate custom novels and saga sequels (Pro or World Class quality, S/M/L/XL length, artwork styles, dedications, character photos, cover direction, EPUB/PDF), buy AI-narrated audiobooks, order printed copies shipped worldwide, and review books.

Paid API operations support x402 USDC payments or API keys backed by prepaid credits. API-key usage requires signing up in the app first at [ainovelist.app](https://ainovelist.app), creating an API key in the dashboard, and adding prepaid credits.

Live options and prices: `GET https://ainovelist.app/api/agent/v1/catalog`.

x402 wallets prove they control the paying wallet with a short signed message (`X-Wallet-Signature` / `X-Wallet-Timestamp`) before the API returns their downloads, status, history or orders, and when they review a book with a public `tx_hash`; `PAYMENT.md` explains the flow.

Install with:

```bash
npx skills add mds-main/novelist-agent-skill --skill novelist
```

Use directly without installing:

```bash
npx skills use mds-main/novelist-agent-skill --skill novelist
```

## Claude plugin and connector

This repository is also a Claude plugin marketplace. The `novelist` plugin (`plugins/novelist`) bundles the Novelist connector, a remote MCP server at `https://ainovelist.app/api/mcp` that users sign in to with their Novelist account, and a `novelist-connector` skill. Claude builds the story with the user and hands it to Novelist as a brief; the user reviews and pays on ainovelist.app. The connector never pays, spends credits or starts a novel, and its prices are references (the exact price depends on country, options and discounts).

```bash
claude plugin marketplace add mds-main/novelist-agent-skill
claude plugin install novelist@novelist
```

In Claude on the web, desktop or mobile, add the connector under Settings, Connectors, Add custom connector, with the URL `https://ainovelist.app/api/mcp`. See `plugins/novelist/README.md`.

The canonical skill files live in `skills/novelist` (`SKILL.md`, `GENERATION.md`, `PRINT.md`, `AUDIOBOOK.md`, `PAYMENT.md`, `package.json`). The Novelist website's `/skill.md` page and `/api/skill/*` downloads serve these same public files.

These files must stay aligned with the API. When a core feature ships in the Novelist app, update this skill in the same change set (see `AGENTS.md` in the Novelist repositories).
