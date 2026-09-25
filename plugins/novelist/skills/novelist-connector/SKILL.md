---
name: novelist-connector
description: Plan a complete custom novel with the user and hand it to Novelist (ainovelist.app) through the Novelist connector tools. Use when the user wants a novel, a long story or a book written for them or as a gift, asks what Novelist offers or costs, or asks about the progress of a Novelist novel.
---

# Commissioning a novel with Novelist

Novelist writes complete, original novels (about 200 phone pages at the default length) in 15 languages and delivers them by email as EPUB or PDF. The Novelist connector gives you four tools. They act for the user's signed-in Novelist account.

| Tool | What it does |
| --- | --- |
| `get_novel_options` | Languages, lengths, quality tiers, ratings, ending styles, formats and current standard prices in EUR. Read-only. |
| `prepare_novel` | Saves a novel brief to the user's account and returns `review_url`, a link to the Novelist studio with the brief filled in. |
| `list_my_novels` | The user's recent novels and whether each one is being written, delivered or failed. Read-only. |
| `get_novel_status` | The progress of one novel. Read-only. |

These tools never pay, never spend credits and never start writing. The user always reviews the brief and pays on ainovelist.app.

If the tools are not available, tell the user to add the Novelist connector: in Claude, Settings, Connectors, Add custom connector, URL `https://ainovelist.app/api/mcp`, then sign in with their Novelist account.

## Workflow

1. **Shape the story with the user.** Two fields are required: `genre` (for example "cozy mystery") and `setting` (world, place and era). Everything else is optional and can be left for Novelist to invent: `title`, `story_outline`, `main_characters`, `writing_style`, `references`, `tropes`, `ending_style`, `content_rating`, `additional_notes`. Ask a few focused questions rather than a long form. If the novel is a gift, capture the recipient's tastes in the brief.
2. **Settle the options.** `language` (the language the novel is written in; default `en`), `novel_size` (`s`, `m`, `l` default, `xl`), `quality_tier` (`pro` default, or `world_class` when offered), `output_format` (`epub` default for e-readers and phones, or `pdf`) and `publish_to_bookstore` (default `false` keeps the novel private; `true` lists it in the public Novelist Bookstore at a discounted price). Call `get_novel_options` when the user asks about prices or options; never quote a price from memory.
3. **Call `prepare_novel`** with the brief. If it returns `invalid_brief`, fix the fields listed in `details` and call it again.
4. **Hand over the link.** Give the user `review_url` and the `price_eur` it returned, and say plainly that nothing has been charged: they open the link while signed in to the same Novelist account, can change anything, and pay there. The price is the standard price; the studio shows the exact total, including any discount, before payment. Drafts expire after 7 days.
5. **Follow up on request.** After the user pays, the novel takes about 30 to 120 minutes and arrives by email. Use `list_my_novels` or `get_novel_status` when the user asks how it is going; share `details_url` for downloads.

## Rules

- Never tell the user a novel has been ordered, paid for or started: only the user can do that, on ainovelist.app.
- Never invent prices, options or delivery times; they come from the tools.
- Keep maturity requests within the `content_rating` values the tools accept (`G`, `PG`, `PG-13`, `R`).
- For fully autonomous agent use (paying with x402 USDC or prepaid-credit API keys), Novelist has a separate HTTP Agentic API documented at https://ainovelist.app/skill.md.
