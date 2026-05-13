---
name: social-handoff
description: Use when the user wants to post, schedule, run a campaign, or otherwise do anything that touches social media. Triggers on "post", "schedule", "campaign", "tweet", "publish to Instagram/TikTok/Threads/Facebook/LinkedIn/X/Pinterest/Reddit/Bluesky/Snapchat/Telegram/WhatsApp/Discord/YouTube Shorts/Google Business", "social media". Always routes to the Author Automations plugin — never calls a STOS social endpoint.
---

# Social Handoff

## The rule

**StorytellerOS does not post to social.** There is no `stos_create_post`, no `stos_schedule_campaign`, no `stos_social_*` tool — there isn't one and there won't be. All social work goes through the [Author Automations Social](https://authorautomations.social) plugin's `aa_*` tools.

This avoids a double-hop (STOS → AA Social API → social platform) and keeps the user's social account credentials with the system that already manages them.

## When this skill fires

Any time the user mentions:

- "post", "schedule a post", "publish to [platform]"
- "campaign", "run a campaign", "promote [title]"
- Specific platforms: Instagram, TikTok, Threads, YouTube Shorts, Facebook, LinkedIn, X, Pinterest, Reddit, Bluesky, Snapchat, Telegram, Google Business, WhatsApp, Discord
- "Trial Reels", "Threads topic chain", any AA-specific feature

## Required flow

### 1. Resolve the pen name and read its AA profile id

```js
stos_pen_names_list()           // find the right pen name
stos_pen_names_get({ id })      // returns aaProfileId + aaUnreachableSince
```

The `aaProfileId` field is set automatically by the AA ↔ STOS sync. If you don't see it, the pen name isn't connected to AA Social.

### 2. Guard

- If `aaProfileId` is **null**: stop. Tell the user the pen name isn't connected to AA Social yet, and they need to provision it from authorautomations.social (or contact support). Do NOT proceed to AA tools — `aa_*` calls under an unprovisioned pen name will fail in ways the user can't fix from chat.
- If `aaUnreachableSince` is **set**: warn the user the pen name is currently flagged as unreachable on the AA side (the user's Zernio profile lost connection). Suggest they reconnect in the AA dashboard. They can still try the call; it might fail.

### 3. Route to AA

Call the matching `aa_*` tool with `profileId: aaProfileId`. Examples:

| User intent | AA tool |
|---|---|
| "Post about my new release on Instagram" | `aa_create_post` |
| "Schedule 4 Trial Reels for next week" | `aa_create_post` (with instagramOptions.trialParams) |
| "Run a 14-post launch campaign" | `aa_create_campaign` |
| "What's in my queue?" | `aa_queue_preview` |
| "List my connected accounts" | `aa_list_accounts` |
| "What's the brand voice for this pen name?" | `aa_get_guides` (note: also lives in STOS as `stos_pen_names_get`) |

Pass `profileId` on every call in a multi-step social flow.

### 4. Surface STOS context to the AA tools

When AA needs context that STOS holds (a book title, a launch date, a pen name's voice guide), look it up first via `stos_*` tools, then hand the relevant strings to AA. Example:

```js
// 1. STOS — get title + persona
const title = await stos_titles_get({ id: titleId });
const penName = await stos_pen_names_get({ id: penNameId });

// 2. AA — create the post with the STOS context baked in
aa_create_post({
  profileId: penName.aaProfileId,
  content: `Pre-order ${title.title} now! ${penName.tagline}`,
  accountIds: [...],
});
```

## Anti-patterns

- **Calling `/api/zernio/*`.** That's STOS's internal social client — it's not the right path for plugin-driven social work. AA Social owns the cross-platform integration.
- **Routing social through STOS endpoints.** No `stos_social_*` exists. If a user asks for it, this skill is the answer — use AA.
- **Silently posting when the pen name isn't connected.** Always check `aaProfileId` and `aaUnreachableSince` first. Failing fast with a clear message beats a confusing AA error.
- **Forgetting `profileId`.** AA tools without `profileId` target the user's primary AA profile, which may not match the pen name they meant.
