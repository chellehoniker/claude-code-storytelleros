---
name: wordpress-posts
description: Use when the user wants to draft, edit, schedule, publish, or delete blog posts on their connected WordPress site. Triggers on "write a blog post", "draft a post about X", "publish my post", "update the last post", "schedule this for tomorrow", "delete the post called X".
---

# WordPress posts

The user's connected WordPress site is the source of truth for blog posts. This skill writes to it through StorytellerOS.

## Tools

```js
stos_wp_posts_list({ status, search, page, per_page })
stos_wp_posts_get({ id })
stos_wp_posts_create({ fields: { title, content, excerpt, slug, status, date, categories, tags, featured_media } })
stos_wp_posts_update({ id, fields: { ... } })
stos_wp_posts_delete({ id, force })   // omit force or pass false to trash; force=true to permanently delete
```

## Field shapes

| Field | Type | Notes |
|---|---|---|
| `title` | string | Plain text. |
| `content` | string | HTML allowed — preserved by WordPress. Most authors prefer paragraphs separated by `\n\n`. |
| `excerpt` | string | Short summary used in feeds. |
| `slug` | string | URL fragment. Omit to let WordPress generate from the title. |
| `status` | `draft` \| `publish` \| `pending` \| `private` \| `future` | `future` requires `date` in the future. |
| `date` | ISO 8601 | Future date for scheduled publish. |
| `categories` | `number[]` | Category IDs. Look up via `stos_wp_categories_list`. |
| `tags` | `string[]` | Tag NAMES (WordPress find-or-creates). |
| `featured_media` | number | Media ID. Look up via `stos_wp_media_list` or upload via `stos_wp_media_create`. |

## Patterns

### Draft a new post

```js
const post = await stos_wp_posts_create({
  fields: {
    title: "Why I Killed My Side Plot",
    content: "<p>Three rewrites in, the side plot was dead weight. Here's how I cut it without losing the emotional through-line.</p>",
    excerpt: "Why side plots fail and what to do about them.",
    status: "draft",
    tags: ["craft", "revision"],
  }
});
```

### Schedule for later

```js
stos_wp_posts_create({
  fields: {
    title: "Launch day post",
    content: "<p>...</p>",
    status: "future",
    date: "2026-06-01T08:00:00Z",
  }
});
```

### Publish an existing draft

```js
stos_wp_posts_update({ id: 123, fields: { status: "publish" } });
```

### Add a featured image from a public URL

```js
const media = await stos_wp_media_create({
  fields: { url: "https://stos-cdn/cover-2026-launch.jpg", alt: "Launch banner" }
});
stos_wp_posts_update({ id: 123, fields: { featured_media: media.id } });
```

## Anti-patterns

- Do not store post content as Markdown — WordPress doesn't render it natively. Convert to HTML before sending.
- Do not pass tag IDs to `tags`. WordPress's REST API for our plugin accepts tag names only; IDs are categories' domain.
- Do not assume the user wants to publish. Default to `status: "draft"` unless they explicitly say "publish".

## Composes well with

- `wordpress-media` — upload featured images
- `blog-post-and-newsletter` — full pipeline that drafts post + image + newsletter from one prompt
