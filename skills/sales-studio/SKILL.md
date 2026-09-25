---
name: sales-studio
description: Use when the user wants to read sales and royalties, or manage retailers, retailer links, title variations (ebook/paperback/audiobook), marketing assets or reviews. Triggers on "how are my books selling", "what did I earn last month", "which store sells best", "where should I publish next", "why are my sales missing", "sync my sales", "set my KENP rate", "add Amazon link for [title]", "list my retailers", "create a paperback variation", "add an audiobook ASIN", "list reviews for [title]", "log a marketing asset".
---

# Sales Studio

Two halves: **reading** what sold, and **maintaining** the catalogue it is reported
against. The reading half is what users actually ask about and was missing from
this skill entirely — a CRUD-only guide meant an agent asked "how are my books
selling" would start creating retailer links.

## Read this before reporting any figure

**Currencies are never converted or cross-summed.** Royalties arrive in the
currency each store paid in and stay there. `stos_sales_report` returns money
keyed by currency for that reason. Report them separately; there is no exchange
rate in this system.

**`null` is not `0`.** A store that does not report a measure returns null. "No
data" and "none sold" are opposite findings, and a store with no reporting must be
OMITTED rather than shown as $0.00 — Apple Books, Google Play and Barnes & Noble
are commonly reached through Draft2Digital, which pays ONE collective royalty
rather than a figure per store, and BookBub and Goodreads are not stores at all.
Showing those as zero beside real ad spend reads as "this did not sell", which is
the false negative that makes someone stop doing something that works.

**KENP reads are not sales, and estimated KENP revenue is not added to royalties.**
Page reads are reported as reads. The USD estimate uses the author's own
per-page rate (`stos_sales_kenp_rates_list`) and is an estimate, labelled as one,
kept separate from reported royalty. Never add the two into one "total earnings".

**Ad figures do not come from here.** `stos_sales_retailer_performance` returns no
ad columns, deliberately. The sales platform reports ad data only for accounts
that wired ads into it, absent for everyone onboarded under the partner contract.
Ads live in `stos_ads_*` — see the `ads-studio` skill.

**Two royalty figures exist and they do not reconcile.** `stos_sales_report` is
canonical for "what did I earn". `stos_sales_retailer_performance` is the
store/country cross-section for "where did it come from". Prefer the first for
totals and the second for structure, and say which one a figure came from.

## Flow: how are my books selling?

1. `stos_sales_report({ penNameId?, from?, to? })` — **call this first.** One call
   returns paid units, free downloads, page reads, KENP estimates, royalties by
   native currency, net, per-title performance, per-source performance, the daily
   trend, mapping completeness, and deterministic recommendations. It
   canonicalises overlapping rows, so figures are not double-counted.
2. `stos_sales_overview` for the cheap version — catalogue counts and a finance
   rollup, head counts rather than row bodies. Use it for "how much do I have",
   not "how much did I earn".
3. `stos_sales_records_list` only when you genuinely need the individual rows.

## Flow: where does this catalogue earn, and where could it earn more?

`stos_sales_retailer_performance({ groupBy })`, and the choice of `groupBy` is the
whole question:

| groupBy | Answers |
|---|---|
| `retailer` | Store totals — is this buy link earning? Only stores WITH a link. |
| `payer` | Who actually pays the royalty (Amazon, ACX, Kobo, an aggregator). |
| `marketplace` | Country level. **Use this for headroom** — it covers every country with sales, link or not. |
| `title` | Which book to promote. |
| `variation` | Per format; audiobook and ebook economics differ. |
| `link` | One buy link, for auditing. |

`retailer` and `link` are questions about a LINK, so they can only include stores
that have one. Everything else is a question about the catalogue. Using `retailer`
to answer a country question once reduced eighteen markets to one and concealed
£212k of United Kingdom royalty.

## Flow: sales are missing or attached to the wrong book

The commonest real problem, and it has three distinct causes. Check in order:

1. **An identifier nothing is mapped to.** `stos_sales_unmapped_list` returns each
   ASIN / ISBN / SKU / store title with sales and no title variation, heaviest
   first. Fix with `stos_sales_mapping_attach`, which also re-attributes history.
2. **An identifier mapped to the WRONG variation.**
   `stos_sales_mapping_overrides_list` shows what has already been decided,
   resolved to the variation and title it points at. `resolves: false` means it
   points at a variation that no longer exists — those are the rows to correct. Fix
   by calling `stos_sales_mapping_attach` again with the same kind and value (it
   upserts), or `stos_sales_mapping_remove` to drop it, which also re-attributes.
   On one real account, 100 of 104 mappings were wrong.
3. **A store reporting nothing at all.** Its retailer probably has no platform key,
   so per-store reporting is silently EMPTY rather than wrong — which reads as "no
   sales" instead of "not mapped yet". Run `stos_sales_map_retailer_platforms`; it
   fills those keys from the payer names that account's own rows actually use, only
   fills blanks, and refuses any retailer matching more than one payer rather than
   guessing. Check `stillUnmapped` and resolve those with `stos_retailers_update`.

Then `stos_sales_sync` to pull fresh data, and `stos_sales_connections_list` to
check a connection is healthy before blaming the data.

## Flow: which buy links should exist?

`stos_sales_selling_platforms` lists every (variation, payer, marketplace) that has
actually EARNED — the set of links that ought to exist, independent of the links
that do. Call it before `stos_retailer_links_create` or `stos_retailer_links_bulk`
instead of creating links one at a time from guesswork. It is fully paged; a real
account returns thousands of rows.

## KENP rates

`stos_sales_kenp_rates_list` and `stos_sales_kenp_rate_update`. The rate is the
author's own estimate of USD per page, per pen name — Amazon does not publish it in
advance. Every KENP revenue figure is therefore an estimate and must be labelled
one, and never folded into reported royalty.

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

- **Answering "how are my books selling" with CRUD calls.** `stos_sales_report`
  first. It is one call and it is canonical.
- **Adding royalties across currencies, or converting them.** There is no rate.
- **Showing a non-reporting store as $0.00.** Omit it and say why.
- **Adding estimated KENP revenue to reported royalties.** Two different things,
  one of them an estimate.
- **Reading ad spend or ROAS from any `stos_sales_*` tool.** Use `stos_ads_*`.
- **Using `groupBy: 'retailer'` for a country question.** It hides every market
  without a buy link.
- **Attaching a mapping without first listing the existing ones.** The identifier
  may already be mapped, to the wrong variation.
- **Presenting the two royalty figures as if they reconcile.** They do not; say
  which tool a number came from.
- **Creating a retailer link without a title variation.** Variations are the join point; a link without one is orphaned.
- **Storing a Kindle URL on the paperback variation.** Match the variation to the format.
- **Looping `stos_title_variations_create` for an import.** Use `stos_title_variations_bulk` instead.
- **Reviewing variations pairwise to find duplicates.** Use `stos_title_variations_duplicates` first — server returns the groups; fetch full records only for the candidates you confirm.
