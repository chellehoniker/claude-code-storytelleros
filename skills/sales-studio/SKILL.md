---
name: sales-studio
description: Use when the user wants to manage retailers, retailer links, title variations (ebook/paperback/audiobook), marketing assets, or reviews. Triggers on "add Amazon link for [title]", "list my retailers", "create a paperback variation", "add an audiobook ASIN", "list reviews for [title]", "log a marketing asset".
---

# Sales Studio

## What this covers

| Entity | Tool prefix | Notes |
|---|---|---|
| Retailer | `stos_retailers_*` | The platform itself (Amazon, Kobo, Apple Books, Google Play, Smashwords, Direct2Readers) |
| Title variation | `stos_title_variations_*` | A format of a title (ebook / paperback / hardback / audiobook / large-print / other) |
| Retailer link | `stos_retailer_links_*` | Per-store URL + sales metadata, tied to a (title variation, retailer) pair |
| Marketing asset | `stos_title_marketing_assets_*` | Per-title promo material (blurbs, copy, file URLs, platform tags) |
| Review | `stos_reviews_*` | Editorial or reader-feedback records |

There is no separate "products" entity — a sellable product in StorytellerOS *is* a title variation. Use `stos_title_variations_*` for anything a reader can buy.

## Flow: add a retailer link

If the user says "add the Amazon ebook link for *Curses and Currents*":

1. Find the title: `stos_titles_list({ penNameId })` → match by name.
2. Find or create the ebook variation: `stos_title_variations_list({ titleId })` → match `variation_type: 'ebook'`. If missing, `stos_title_variations_create({ fields: { book_id: titleId, variation_name: 'Ebook', variation_type: 'ebook', asin: '...' } })`.
3. Find the retailer: `stos_retailers_list` → match "Amazon".
4. Create the link:

```js
stos_retailer_links_create({
  fields: {
    title_variation_id: variationId,
    retailer_id: amazonId,
    link_name: 'Curses and Currents — Kindle',
    product_url: 'https://amazon.com/dp/...',
    status: 'live',
    go_live_date: '2026-05-13',
  },
})
```

## Flow: add a marketing asset

```js
stos_title_marketing_assets_create({
  fields: {
    book_id: titleId,
    asset_name: 'Launch blurb (Instagram)',
    asset_type: 'blurb',
    platform: ['instagram'],
    content: 'Three sisters. One coven. ...',
    status: 'approved',
    character_limit: 2200,
  },
})
```

## Flow: log a review

```js
stos_reviews_create({
  fields: {
    book_id: titleId,
    source: 'NetGalley',
    rating: 5,
    reviewer_name: '...',
    excerpt: '...',
  },
})
```

## Bulk operations on variations + retailer links

When you have many variations or retailer links to manage in one pass (e.g. importing a backlog, syncing translations, batch-updating ISBNs), use the bulk endpoints instead of looping per-record:

- **`stos_title_variations_bulk({ items: [...] , on_conflict: 'merge' | 'skip' | 'error' | 'ask' })`** — import or upsert many variations at once. Use `on_conflict: 'ask'` for safety when you're not sure if records exist; the API returns candidates and you re-submit with `force_action` after the user confirms.
- **`stos_title_variations_duplicates({ penNameId?, titleId? })`** — server-side duplicate finder (matches on ASIN if present, else on book_id + variation_name + type). Returns slim records (id / name / type / language); fetch the full record only for the ones you actually need to merge.
- **`stos_retailer_links_bulk({ items: [...] })`** — same pattern for retailer links. Matches on `(title_variation_id, retailer_id, product_url)`.

These tools exist because looping `_create` or `_update` per record forces every MCP round-trip to replay the full tool schema (~50KB per turn). A 30-variation import via bulk is roughly one round-trip; the same import via a loop is 30. Same principle as the `character-merge` skill — let the server do the per-record work and keep Claude's context small.

## Anti-patterns

- **Creating a retailer link without a title variation.** Variations are the join point; a link without one is orphaned.
- **Storing a Kindle URL on the paperback variation.** Match the variation to the format.
- **Looping `stos_title_variations_create` for an import.** Use `stos_title_variations_bulk` instead.
- **Reviewing variations pairwise to find duplicates.** Use `stos_title_variations_duplicates` first — server returns the groups; fetch full records only for the candidates you confirm.
