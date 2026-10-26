---
name: ads-studio
description: Use when the user wants to plan, create, measure, or fix advertising — Amazon, Meta, Google, TikTok, LinkedIn, Pinterest, X or ChatGPT ads. Triggers on "how are my ads doing", "what's my ROAS", "which market should I cut", "start an ad for [title]", "why is my ad spend not showing sales", "check my tracking links", "work out an ad strategy", "which books should I advertise".
---

# Ads Studio

Roughly 95 `stos_ads_*` tools, including eight for Amazon, all on the StorytellerOS
connector (no second connector is needed for ads). This exists because
without it an agent will reach for whichever one sounds closest and get an answer
that is confidently wrong in one of five specific ways. Those five are the whole
point of this document.

## The five rules, before anything else

**1. Never add money across currencies.** Ad accounts report in their own
currency — the UK profile in GBP, Germany in EUR, Australia in AUD. `spend`,
`sales`, `orders` and every derived figure belong to one currency, named on the
row. There is no exchange rate anywhere in this system, so a total across
currencies is not a number, and a ROAS across them is worse: it looks like a
verdict on a campaign.

This is not hypothetical. An analysis in September 2026 divided a USD royalty by a
GBP spend and reported the UK at ROAS 0.99 — then the recommendation was to cut
the UK. Computed inside one currency:

| Market | Spend | ROAS |
|---|---|---|
| US | $2,262 | 1.98 |
| Germany | €1,349 | **0.58** |
| UK | £493 | **3.82** |
| Australia | A$360 | 4.85 |
| Canada | C$101 | 16.25 |

The UK was one of the best markets. The one losing money was Germany, the
second-largest spend on the account. Acting on the cross-currency figure meant
switching off the winner and leaving the loser running.

**2. Ad data NEVER comes from the sales platform.** The connected retailer
analytics reports ad columns for accounts that happen to have ads wired into it.
Those are absent for every user onboarded under the partner contract, so a tool
built on them works on one demo account and is empty for customers. Ad spend, ad
sales, orders, clicks and ROAS come from `stos_ads_*` and `stos_ads_amazon_*`
only. `stos_sales_retailer_performance` deliberately returns no ad columns — that
is not a gap to work around.

**3. `null` is not `0`.** A measure a platform does not report comes back null.
"No data" and "none sold" are opposite findings, and substituting 0 turns the
first into the second. Never fill a null with zero when summarising, and never
divide by a null-or-zero spend — say "no spend" instead of reporting 0.00×.

**4. Ads start PAUSED and are started explicitly, in the app.** Every create path
produces paused ads. Starting them spends real money, so it is a separate,
deliberate action the author takes — never a step you fold into creation, and
never something you do because a plan said to. If asked to "launch" ads, create
them paused and say clearly that the author starts them.

**5. A small spend is not a signal.** A market that spent €0.34 and returned
nothing is at 0.00× and means nothing. Below about 25 units of currency, report
the figure without a verdict; drawing conclusions from it buries the findings that
matter.

## Which tool answers which question

| The question | Start here |
|---|---|
| How are my ads doing overall? | `stos_ads_timeline` then `stos_ads_analytics` |
| Which market should I cut or scale? | `stos_ads_amazon_profitability` (per profile, so per currency) |
| Is this campaign making money? | `stos_ads_groups_dashboard` — campaign groups roll up across networks |
| What is running right now? | `stos_ads_campaigns_unified` (all networks) or `stos_ads_amazon_campaigns` |
| Which books should I advertise? | `stos_sales_selling_platforms` + `stos_sales_report` — earn first, then advertise |
| Why does spend show no sales? | Tracking, almost always. See "When the numbers look wrong" |
| What should I even run? | `stos_ads_plans_list`, then the strategy flow below |
| Is my Amazon data current? | `stos_ads_amazon_status`, then `stos_ads_amazon_sync` |

## Flow: work out a strategy before spending

The house playbook is deterministic — it is not the model guessing. Give it the
situation and it returns platform choice, campaign structure, targeting, bidding,
what to test first, and the numbers at which to stop or scale.

1. `stos_ads_strategy_plans_list` — is there already an adopted plan for this book?
   (`stos_ads_capabilities` first if you don't know what is connected.)
2. `stos_ads_strategy_plan_create` with the situation brief: work type, genre, where
   the book sits in the catalogue (first in series / mid / standalone / box set), how
   it is sold (KU / wide / direct), and budget posture.
3. `stos_ads_strategy_plan_generate`, then read it back with
   `stos_ads_strategy_plan_get`. **Gaps are meaningful**: an area listed
   in `gaps` is one where the playbook has NO opinion, and presenting a default
   there as a recommendation is the failure mode to avoid. Say the author must
   decide it.
4. `stos_ads_strategy_plan_adopt` to make a version the one in use. **An adopted plan is immutable** —
   changing your mind writes a new version rather than editing it, so the record
   of what was decided stays true. Never describe adoption as "updating" a plan.

Four research reads support this, and all four cost nothing: keyword ideas
(`stos_ads_strategy_keyword_ideas`), reach estimate
(`stos_ads_strategy_reach_estimate`), ad-library scan
(`stos_ads_strategy_ad_library_scan`, what comparable books are running, in their
own words), and review audit (`stos_ads_strategy_review_audit`). The review audit is the best source of ad copy there is, because
it is the words real readers already used.

## Flow: create an ad for a title

1. Resolve the title and its ASIN — the ASIN lives on the **title variation**, not
   the title, so `stos_title_variations_list({ titleId })` and read `asin`.
2. Check the destination resolves before spending on sending people to it. See
   tracking below.
3. Create: `stos_ads_amazon_create` for Amazon, `stos_ads_create` for the brokered
   networks. Both produce PAUSED ads.
4. Tell the author the ads exist and are paused, and that starting them is theirs
   to do in the app. Do not offer to start them.

## Flow: click tests

`stos_ads_creative_hooks` and `stos_ads_creative_copy_per_ad` write the ideas;
`stos_ads_test_runs_list` reads a running test; `stos_ads_test_run_judge` without
`confirm` shows the results, and only after the author picks the winners do you call
it again with `confirm: true`. Never choose winners yourself.

## Flow: boost a published post

`stos_ads_create` with `postId` (from `stos_social_posts_list`) makes a boost.
Created paused like every other ad.

## When the numbers look wrong

Work down this list. The first four are far more common than "the ads are bad",
and each one looks exactly like bad ads from the outside.

1. **Spend but no sales.** Check tracking first:
   `stos_ads_tracking_links_list`, then `stos_ads_tracking_bridge_check` per link.
   A link that has never been checked is a link nobody has confirmed goes
   anywhere. The whole setup is in `stos_ads_tracking_setup_get`; Amazon
   Attribution in `stos_ads_tracking_amazon`. Then `stos_ads_tracking_verify` for pixels — a pixel that exists is
   not a pixel that has fired.
2. **Sales but no attributed spend.** The ASIN is probably unmapped.
   `stos_ads_title_mappings_list` and `stos_sales_mapping_overrides_list`. Ad
   facts join to titles by ASIN only — there is no variation id on the ad rows —
   so an unmapped ASIN means real spend attached to no book.
3. **A market reports nothing at all.** Its retailer may have no platform key, in
   which case per-store reporting is silently empty rather than wrong. Run
   `stos_sales_map_retailer_platforms`, which fills those keys from the payer
   names that account's own rows use, and check `stillUnmapped`.
4. **Stale figures.** `stos_ads_amazon_status` shows freshness;
   `stos_ads_amazon_sync` refreshes. Amazon's own reports lag by a day or two, so
   "today shows nothing" is usually normal.
5. **Only then** consider that the ads are underperforming — and check rule 5
   before saying so, because a tiny spend proves nothing either way.

## Anti-patterns

- **Summing or dividing across currencies.** Rule 1. The single most damaging
  mistake available here.
- **Reading ad figures from `stos_sales_*`.** Rule 2. Works on one account, empty
  for customers.
- **Starting ads.** Rule 4. Create paused; the author starts them.
- **Filling a null with 0 to make a table tidy.** Rule 3. It inverts the finding.
- **Recommending a cut from a two-week window on a £5 spend.** Rule 5.
- **Treating a `gaps` entry as a recommendation.** The playbook is saying it does
  not know.
- **Describing adoption as an update.** Adopted plans are immutable by design.
- **Looking up an ASIN on the title.** It is on the variation.
- **Concluding "the ads do not work" before checking tracking.** Broken
  measurement and failed advertising are indistinguishable from the report, and
  the first is both more common and fixable.
