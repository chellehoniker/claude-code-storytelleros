---
name: wordpress-seo
description: Use when the user wants to improve search rankings for their WordPress content — audit or optimize titles, slugs, excerpts, headings, image alt text, internal links, or schema markup. Triggers on "SEO my post", "optimize this post for search", "audit my blog's SEO", "why isn't my post ranking", "add schema to my post", "fix my slugs".
---

# WordPress SEO

Audit and improve on-page SEO for posts and pages on the user's connected WordPress site(s), through StorytellerOS.

## Tools

Works entirely with the standard WordPress tools — no separate SEO tool:

```js
stos_wp_posts_list({ status: "publish", penNameId? })
stos_wp_posts_get({ id, penNameId? })
stos_wp_posts_update({ id, fields: { title, slug, excerpt, content }, penNameId? })
stos_wp_pages_list / stos_wp_pages_get / stos_wp_pages_update   // same shapes
stos_wp_media_list({ penNameId? })                              // find images missing alt
stos_wp_media_update({ id, fields: { alt }, penNameId? })
```

Multiple sites: users can connect one WordPress site per pen name plus an account default. Pass `penNameId` (from `stos_pen_names_list`) to work on a specific pen name's site; omit it for the account default. If the user has several pen names, ASK which one before auditing.

## Audit workflow

For each post/page the user wants audited, check in this order:

1. **Title** — under ~60 characters, leads with the topic keyword, written for a human click.
2. **Slug** — short, hyphenated, keyword-bearing. ⚠️ NEVER change the slug of a PUBLISHED post without telling the user it changes the URL and old links will break unless they add a redirect. Draft posts: change freely.
3. **Excerpt** — this feeds the meta description on most themes. 120–155 characters, states the payoff, includes the keyword naturally.
4. **Heading structure** — exactly one H1 (the title; content should start at H2), no skipped levels, headings describe their section.
5. **Image alt text** — every `<img>` in content and every attached media item has descriptive alt text (not "image1.png").
6. **Internal links** — link related posts on the same site to each other. Use `stos_wp_posts_list({ search })` to find candidates.
7. **Schema markup** — JSON-LD in a `<script type="application/ld+json">` block inside the content (Article, Book, Review, FAQPage as fits). WordPress preserves it.

Report findings as a short list (worst first), then fix what the user approves via `stos_wp_posts_update` — send only the fields being changed.

## Example: add Book schema to a book-launch post

```js
stos_wp_posts_update({
  id: 42,
  fields: {
    content: existingContent + `\n<script type="application/ld+json">
{"@context":"https://schema.org","@type":"Book","name":"Storms and Secrets",
"author":{"@type":"Person","name":"A. Author"},"inLanguage":"en",
"workExample":{"@type":"Book","bookFormat":"https://schema.org/EBook"}}
</script>`,
  },
});
```

## Cautions

- Never bulk-rewrite published content without showing the user the plan first.
- Don't stuff keywords — one natural use in title, first paragraph, and one heading is plenty.
- Yoast/RankMath meta fields (meta title, focus keyword, noindex) are NOT reachable through these tools yet — if the user asks, say those still need editing in WordPress directly.
