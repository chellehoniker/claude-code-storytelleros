---
name: getting-started
description: Use when a user is new to StorytellerOS, asks "where do I start?", "what should I do first?", "help me get set up", or when you notice their account is nearly empty (no pen names, no books). Walks the top-level first-run tasks one at a time — add a book, connect socials, set up email sending — without firehosing.
---

# StorytellerOS Getting Started

## When to Use
- User just connected StorytellerOS and asks what to do
- User says "getting started", "onboard me", "set up my account", "what now?"
- A tool call reveals an empty account (0 pen names or 0 books) and the user seems unsure where to begin
- User asks how to "unlock" a studio (Project, Social, Marketing)

## The one rule: one step at a time

New authors churn when they get a wall of setup instructions. Never list every
remaining task. Find the SINGLE most valuable next step, help them finish it,
celebrate it, then ask if they want the next one. Building the relationship
beats completing the checklist.

## Step 0 — Read the account state

Call `stos_getting_started` (no arguments). It returns:

- `readiness` — live flags: pen name/book counts, AI + image key presence,
  per-pen-name email sender status, social profile link
- `nextSteps` — ordered list of ONLY the missing steps
- `guides` — canonical web-app + MCP walkthroughs for each top task

Take the FIRST item in `nextSteps` and offer that. Mention at most one other
thing exists ("after this there are a couple of optional connections — we'll
get to them whenever you like").

If `stos_getting_started` isn't available (older server), fall back to
`stos_account_manifest` for pen name/book counts and infer from there.

## The first-run arc (reference)

The natural order, matching the in-app onboarding wizard:

### 1. Pen name (the foundation)
Everything hangs off a pen name — series, books, email audience, social voice.
- MCP: `stos_pen_names_create` with just a name is enough to start.
- Ask conversationally: "What name do you publish under?" Don't ask for
  genres/tagline/heat level up front — those can be enriched later
  (see the `pen-names` skill).

### 2. First book — the 30-second win
If the user is already published on Amazon, the web app's ASIN import is the
fastest path and genuinely delightful — recommend it over manual entry:

> Open **Project Studio → Titles → Quick Create** at storytelleros.com, paste
> your book's ASIN (the `B0…` code in its Amazon URL), and hit search. Title,
> synopsis, covers, and every published format (ebook, paperback, hardcover,
> audiobook) import automatically as variations. A whole backlist? **Titles →
> Import Published** takes your full KDP report CSV in one shot.

Not published yet, or the user prefers staying in chat: `stos_titles_create`
(needs `penNameId`), then `stos_title_variations_bulk` for formats.

Terminology guard: a **variation** is a FORMAT (ebook/paperback/hardcover/
audiobook). An **edition** is a VERSION (1st, revised). Never say "edition"
for a format.

### 3. Story bible — the "wow" moment
Once a book exists and a manuscript is attached, offer the story bible: you
can extract characters, locations, lore, and timeline from the manuscript in
conversation and write them into StorytellerOS. See the `story-bible` skill.
This is the moment new users fall in love — offer it early.

### 4. Connect socials (unlocks Social Studio)
Social OAuth happens in the browser — it cannot be completed over MCP.

> Open **Settings → Connections** at storytelleros.com and connect Facebook,
> Instagram, Threads, or Pinterest. Each takes one approval click.

After connecting, scheduling works from Social Studio, and social posting from
Claude routes through the Author Automations Social plugin (see the
`social-handoff` skill — StorytellerOS itself never wraps social posts).

### 5. Email sending (unlocks Marketing Studio campaigns)
StorytellerOS ships its own email engine — BYO SMTP, unlimited contacts, no
per-subscriber fees. Senders are **per pen name**:

> Open **Marketing Studio → Email → Settings → Sender**, pick your SMTP
> provider (or enter host/port/username/password), set the from-address, and
> send the verification email.

`readiness.emailSending.perPenName` shows configured/verified per pen name.
Once verified, campaigns, automations, contacts, lists, and signup forms all
work over MCP (`stos_email_*`; orient with `stos_email_overview`).

### 6. Optional polish
- BYOK AI key (Settings → Integrations) for in-app AI features
- Freepik key for AI covers and marketing imagery
- Claude connector already works — they're talking to you through it

## Suggested opening move

For a brand-new account, after reading readiness, a strong opener is:

> "Let's get your first book in. Are you already published on Amazon? If so,
> grab any of your books' Amazon pages and paste me the ASIN — or tell me the
> pen name you write under and we'll start from scratch."

Then do the work with them, one exchange at a time.

## Handoffs
- Connector/auth problems → `stos-setup` skill
- Multi-pen-name questions → `pen-names` skill
- Manuscript → story bible → `story-bible` skill
- Social posting → `social-handoff` skill
- Email campaigns in depth → `marketing-studio` skill
