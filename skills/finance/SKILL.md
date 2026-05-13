---
name: finance
description: Use when the user wants to log income, log an expense, see what they spent on something, or manage a vendor. Triggers on "log an expense", "I got royalties of $X", "what did I spend on editing", "add a vendor", "list my expenses for [period]", "how much did I make from [title]".
---

# Finance

## What's tracked

- **Vendors** — editors, cover artists, narrators, etc. (`stos_vendors_*`)
- **Transactions** — income or expense rows tied to pen name / series / title / vendor (`stos_finance_transactions_*`)
- **Summaries** — totals by period / category / book / vendor / pen name (`stos_finance_summary`)

## Flow: log an expense

If the user says "log a $400 expense to Sarah Marsh for editing on *Curses and Currents*":

1. Find or create the vendor: `stos_vendors_list` → look for "Sarah Marsh". If not found, `stos_vendors_create({ fields: { vendorName: 'Sarah Marsh', vendorType: 'Editor' } })`.
2. Resolve the title: `stos_titles_list({ penNameId })` → match by name.
3. Create the transaction:

```js
stos_finance_transactions_create({
  fields: {
    transactionName: 'Editing — Sarah Marsh',
    transactionType: 'expense',
    amount: 400,
    transactionDate: '2026-05-13',
    category: 'Editing',
    penNameId,
    bookId: titleId,
    vendorId,
    paymentStatus: 'pending',
  },
})
```

4. Confirm: "Logged a $400 editing expense to Sarah Marsh against *Curses and Currents*."

## Flow: log income

Same pattern, `transactionType: 'income'`. Vendor is optional for royalty/sales income.

## Flow: review

For "how much did I spend on editing this quarter":

```js
stos_finance_summary({ range: 'quarter', groupBy: 'category' })
```

Look at the `Editing` row in `groups[]`. For "by title" use `groupBy: 'book'`; for per-vendor use `groupBy: 'vendor'`.

## Language

Per the project's user-facing-copy rules: never use developer vocabulary. Say "expense," "income," "vendor." Don't say "transaction record," "row," "table," "schema," or name any specific tech (Stripe / Supabase / etc.).

## Anti-patterns

- **Double-creating a vendor.** Always `stos_vendors_list` and search before creating.
- **Forgetting the pen-name scope.** A multi-pen-name author wants per-identity ledgers — don't conflate.
- **Listing every transaction when the user asked for a total.** Use `stos_finance_summary` for aggregates.
