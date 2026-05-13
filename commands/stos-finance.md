---
description: Log a one-line income or expense to StorytellerOS finance
argument-hint: <income|expense> <amount> <note>
allowed-tools: [Bash, Read, Write]
---

# Finance Quick-Capture

Log this finance entry: $ARGUMENTS

Follow the `finance` skill. Parse the type (`income` / `expense`), amount, and note from the arguments. If a vendor or pen name is mentioned, look it up first with `stos_vendors_list` / `stos_pen_names_list`. Then call `stos_finance_transactions_create` with the appropriate fields. Confirm the saved transaction's id and amount back to the user in one line.

Never expose tech-stack vocabulary back to the user — call it an "expense" or "income," not a "transaction record."
