---
name: wordpress-llm-optimization
description: Use when the user wants their WordPress content optimized for AI search and assistants — being cited by ChatGPT/Claude/Perplexity/AI Overviews, "LLM SEO", "GEO", "AEO", answer engine optimization, or "make my site show up in AI answers". Triggers on "optimize for AI search", "LLM-friendly", "get cited by AI", "answer engine".
---

# WordPress LLM optimization

Restructure posts and pages so AI assistants can retrieve, quote, and cite them. Complements wordpress-seo (classic search); run that skill's audit too when the user wants both.

## Tools

Same WordPress tools as wordpress-seo — content-level changes only:

```js
stos_wp_posts_get({ id, penNameId? })
stos_wp_posts_update({ id, fields: { title, content, excerpt }, penNameId? })
stos_wp_pages_get / stos_wp_pages_update
```

Multiple sites: pass `penNameId` (from `stos_pen_names_list`) to target a specific pen name's site; omit for the account default. Ask which site when the user has several pen names.

## What LLM-retrievable content looks like

1. **Answer-first structure** — the direct answer to the page's core question appears in the first ~2 sentences, then the elaboration. AI retrieval quotes openings.
2. **Self-contained sections** — each H2/H3 section makes sense quoted alone: restate the subject in the section ("Amazon KDP royalties are…" not "They are…"). No "as mentioned above".
3. **Extractable Q&A** — a short FAQ block near the end with real questions as H3s and 1–3 sentence answers. Add matching FAQPage JSON-LD in a `<script type="application/ld+json">` block.
4. **Clear entity naming** — full names on first use (book titles, pen names, series, tools); pronouns only after the entity is established in the same section.
5. **Facts in stable shapes** — prices, dates, steps as lists or tables, not buried in prose.
6. **Freshness signals** — a visible "Last updated" line when the user revises evergreen content.

## Workflow

1. Fetch the post, map its structure (headings, opening, FAQ presence).
2. Score against the six points above; report the gaps briefly.
3. Rewrite with `stos_wp_posts_update` after the user approves the plan — preserve voice; restructure, don't flatten.

## Cautions

- Keep the author's voice — restructure paragraphs, don't sand the prose down to listicle mush.
- Never invent facts to fill an FAQ; pull answers from what the post already establishes.
- For published posts, keep the slug and title intent stable (see wordpress-seo's redirect caution).
