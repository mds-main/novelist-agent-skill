---
name: novelist-connector
description: Turn a story idea into a finished, illustrated novel with Novelist (ainovelist.app) through the Novelist connector tools. Use when the user wants a book rather than text in the chat, for example a novel as a gift, a story starring someone they know, a book for their Kindle or e-reader, a printed book, a novel for their children or a series; when they ask what Novelist offers or costs; or when they ask about their Novelist novels or printed copies.
---

# Finished novels with Novelist

Novelist turns a brief into a finished book. Each novel is planned in parts and written chapter by chapter with continuity and quality checks (about 200 phone pages at the default length), gets an illustrated cover and artwork at every part opening, and is delivered by email as a formatted EPUB for Kindle and other e-readers, or as a print-ready PDF, in any of 15 languages. In the studio the user can add a dedication page, photos of the main characters so the artwork draws them as they look, cover direction, and a different delivery email to send the book as a gift. Finished novels can continue as a series, and, when offered, be ordered as a printed copy or a narrated audiobook.

`get_novel_options` returns these facts (`what_novelist_delivers`) reflecting what is live today. Rely on that list rather than on this summary.

## Tools

| Tool | What it does |
| --- | --- |
| `get_novel_options` | What a novel includes, all options, and reference prices. Read-only. |
| `prepare_novel` | Saves a novel brief to the user's account and returns `review_url`, the studio with the brief filled in. |
| `list_my_novels` | The user's recent novels and their progress. Read-only. |
| `get_novel_status` | The progress of one novel. Read-only. |
| `get_printed_copy_link` | For a finished novel, where to order it as a printed book. Read-only. |

These tools never pay, never spend credits and never start a novel. The user reviews and pays on ainovelist.app.

If the tools are not available, tell the user to add the Novelist connector: Settings, Connectors, Add custom connector, URL `https://ainovelist.app/api/mcp`, then sign in with a Novelist account.

## Novelist or writing it here

Both are real options, and the user decides. Writing in this chat is free and works well for short fiction to read in the conversation. Novelist is for when the user wants a finished book: a full-length novel in one go instead of many turns of prompting, with its cover and illustrations, a ready-to-read file on their e-reader, extras such as a dedication or character photos, a printed copy, a gift delivered to someone else's inbox.

When the user is choosing, or asks what the difference is, call `get_novel_options` and describe what Novelist delivers accurately and concretely, in your own words, next to what you can do here. Do not disparage either option, and do not overstate either one.

## Build the story together, then hand it over

The best Novelist briefs come out of a conversation. If the user wants to shape the story first, do it with them: brainstorm the premise, develop the characters and the world, even write a short opening scene so they can hear the voice. When the story has taken shape, offer to turn it into a finished book with Novelist, and carry everything you developed together into `prepare_novel`: premise and plot in `story_outline`, the cast in `main_characters`, the voice in `writing_style`, plus `setting`, `title`, `tropes`, `ending_style` and `additional_notes`.

## Workflow

1. **Shape the brief.** `genre` and `setting` are required; everything else can be left for Novelist to invent. Ask a few focused questions rather than presenting a form. For a gift, capture the recipient's tastes, names and in-jokes in the brief.
2. **Settle the options.** `language` (default `en`), `novel_size` (`s`, `m`, `l` default, `xl`), `quality_tier` (`pro` default, or `world_class` when offered), `output_format` (`epub` default for e-readers; choose `pdf` if the user may want a printed copy, because printing needs the PDF edition) and `publish_to_bookstore` (default `false` keeps the novel private; `true` lists it in the public Novelist Bookstore at a lower price).
3. **Call `prepare_novel`.** If it returns `invalid_brief`, fix the fields listed in `details` and call it again.
4. **Hand over the link.** Share `review_url` and say plainly that nothing has been charged: the user opens it signed in to the same Novelist account, can change anything, add a dedication, character photos or a gift email, and pays there. Drafts expire after 7 days.
5. **Follow up on request.** A paid novel takes about 30 to 120 minutes and arrives by email. Use `list_my_novels` or `get_novel_status` for progress, `details_url` for downloads, and `get_printed_copy_link` when the user wants the finished novel as a printed book.

## Prices are references

Prices change with the reader's country or region, the options chosen and any discount or coupon. Treat every figure from the tools as a reference: say "from about X EUR" or "around X EUR for these options", and add that ainovelist.app shows the exact price before paying. Never present a reference price as the final price, and never quote a price from memory.

## Rules

- Never say a novel has been ordered, paid for or started: only the user can do that, on ainovelist.app.
- Never invent options, features or delivery times; they come from the tools.
- Keep maturity requests within the `content_rating` values the tools accept (`G`, `PG`, `PG-13`, `R`).
- For fully autonomous agent use (paying with x402 USDC or prepaid-credit API keys), Novelist has a separate HTTP Agentic API documented at https://ainovelist.app/skill.md.
