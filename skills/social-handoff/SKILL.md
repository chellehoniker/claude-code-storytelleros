---
name: social-handoff
description: Use when the user wants to post, schedule, run a social campaign, work their social Inbox, read social analytics, or manage connected social accounts. Triggers on "post", "schedule", "campaign", "tweet", "publish to Instagram/TikTok/Threads/Facebook/LinkedIn/X/Pinterest/Reddit/Bluesky/YouTube Shorts/Google Business/Discord", "social media", "my queue", "my inbox", "best time to post". Everything runs on the StorytellerOS connector's stos_social_* tools; no second connector is needed.
---

# Social Studio from Claude

## The rule

**All social work runs through the StorytellerOS connector.** The `stos_social_*`
tools cover posting, scheduling, campaigns, the Inbox, analytics and account
settings. Do not look for, suggest, or wait for a second social connector or
plugin. If a `stos_social_*` tool is missing, the StorytellerOS connector is not
connected or not enabled: route to `stos-setup`.

## 1. Scope to the pen name first

```js
stos_pen_names_list()              // match the pen name the user means
stos_social_accounts_list({ penNameId })   // what is connected under it
```

Pass `penNameId` on **every** social call in the flow. Without it the call targets
the active pen name, which may not be the one the user meant.

- A refusal that says social posting isn't set up for the pen name: stop and tell
  the user to set it up in StorytellerOS (Settings, then Pen Name Details). Don't
  retry under another pen name.
- An account that needs reconnecting (`stos_social_accounts_health`): say which one
  and point the user to Social Studio accounts in the app. Connecting a new social
  account also happens in the app.

## 2. Pull the writing context from StorytellerOS

Look up what the post is about before drafting: `stos_titles_get`,
`stos_title_variations_list` (ASINs and formats live on the variation),
`stos_pen_names_get` (voice, guides, tagline), `stos_social_guides_get`. Write in
the pen name's voice.

## 3. Pick the tool

| User intent | Tool |
|---|---|
| Post now, schedule, draft or queue a post | `stos_social_posts_validate` first, then `stos_social_posts_create` |
| Upload media for a post | `stos_social_media_upload_url` (PUT the bytes, reuse `publicUrl`) or a Media Studio asset (`stos_media_assets_list`) |
| Check a caption or file before posting | `stos_social_validate` (post-length, media, subreddit, full dry run), `stos_social_hashtags_check` |
| Change, retry, archive, unpublish a post | `stos_social_posts_update`, `_retry`, `_archive`, `_unpublish`; swap media with `stos_social_posts_media_edit` |
| Copy a post | `stos_social_posts_duplicate` |
| What's in my queue? | `stos_social_queues_list`, `stos_social_queues_preview`, `stos_social_queues_next` |
| Launch campaign for a title | the campaign flow below |
| Labels, calendar notes, archive | `stos_social_labels`, `stos_social_post_labels`, `stos_social_calendar_notes`, `stos_social_archived_posts` |
| DMs, comments, reviews | `stos_social_inbox_*` (list, reply, private reply, like, hide, pin, react) |
| Comment-to-DM and Inbox flows | `stos_social_comment_automations_*`, `stos_social_inbox_flows_*` |
| How did my posts do? When should I post? | `stos_social_analytics` (report: posts, best-time, follower-stats, ...) |
| Platform options (boards, subreddits, flairs, playlists, TikTok privacy) | `stos_social_account_options` |
| Account settings (defaults, Facebook Page, LinkedIn identity, Discord, Messenger menu, ice breakers, groups) | `stos_social_account_update`, `stos_social_account_messaging_settings`, `stos_social_account_groups` |
| RSS auto-drafting | `stos_social_rss_feeds` |
| Find something to post | `stos_social_find_content`, `stos_social_reddit`, `stos_social_youtube_transcript` |
| Ads (boost a post, run ads) | the `ads-studio` skill |

## 4. Campaign flow (a launch, a month of posts)

1. `stos_social_campaigns_create` with the title, pen name and objective.
2. `stos_social_campaign_sample_generate`: one look-check image. Show it; approve the
   look (`stos_social_campaigns_update`) before going on.
3. `stos_social_campaign_plan_generate`: generate the plan, or save a reviewed one.
4. `stos_social_campaign_captions_generate`, then `stos_social_campaign_media_generate`
   and poll `stos_social_campaign_media_status`. Overlays:
   `stos_social_campaign_text_overlays_apply`; a carousel as one video:
   `stos_social_campaign_carousel_video`.
5. Review with the user, then `stos_social_campaign_schedule`. Each platform's queue
   sets the timing. Relay the returned warnings.

## Anti-patterns

- **Publishing without review.** Show the account, the content and the time, and get
  a yes before anything goes live.
- **Forgetting `penNameId`** in a multi-step flow.
- **Telling the user to install another connector** for social work. The
  StorytellerOS connector covers it.
- **Calling internal app endpoints** instead of the `stos_social_*` tools.
