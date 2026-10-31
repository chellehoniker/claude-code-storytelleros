---
name: blog-post-and-newsletter
description: Use when the user wants to publish a blog post AND turn it into a newsletter in one step. The flagship WordPress + Email Studio flow. Triggers on "write and send a blog post about X", "blog this and email the list", "draft a post and a newsletter on X", "publish to WP and send to my readers".
---

# Blog post → newsletter (flagship composite flow)

This is the highest-leverage WordPress + Email Studio combo. The user gives you a topic; you draft once, publish to WordPress with a featured image, then queue the newsletter pointing at the published URL — without copy-pasting between tools.

## The pipeline

```text
1. Discovery       → confirm site is connected         (stos_wp_site)
2. Draft post      → write title + body                (in-conversation)
3. Featured image  → generate in Media Studio          (stos_media_generate, then
                                                        stos_media_generation_status)
4. Upload image    → push to WP media library          (stos_wp_media_create)
5. Publish post    → save + publish on WP              (stos_wp_posts_create)
6. Compose email   → newsletter using the same body    (stos_email_campaigns_create)
7. Confirm         → show user the WP URL + draft id
```

Skip steps the user explicitly opts out of (e.g. "no image" → skip 3-4). The post saves as a draft unless the user says to publish; then set status `publish` in step 5.

## Step-by-step

### 1. Confirm the connection

```js
const site = await stos_wp_site();
// site.siteName, site.pluginVersion, site.hasWooCommerce, etc.
```

If this 404s, route to `wordpress-connect`.

### 2. Draft the post

Compose the title and HTML body in conversation. Aim for the user's voice — pull pen-name guidance via `stos_pen_names_list` if the user has a primary pen name with a brand voice doc.

Default structure:
- 1-line lede
- 2-3 paragraph development
- 1-paragraph practical takeaway / call to action
- HTML, not Markdown

### 3. Generate featured image

Use Media Studio on the StorytellerOS connector (it uses the image provider the user has configured). Aim 16:9, evocative-not-literal.

`sizes` must name at least one platform size. Valid keys: `instagram_feed`, `instagram_story`, `tiktok`, `facebook_feed`, `linkedin`, `twitter`, `pinterest`, `youtube_thumb`, `bluesky`, `threads`, `googlebusiness`, `reddit`, `telegram`, `snapchat`, `whatsapp`, `discord`. The 16:9 ones (`twitter`, `youtube_thumb`, `bluesky`, `reddit`, `discord`) suit a blog header; an unknown key is refused with "Unknown size key".

```js
await stos_media_generate({
  taskType: "image",
  prompt: "Watercolor sketch of a writer at dawn, soft pastel palette",
  sizes: ["twitter"],  // required for images; "twitter" is 1200×675 (16:9), a good blog header
});
// poll until the job completes; the finished image then appears in the media library
const jobs = await stos_media_generation_status({});
const assets = await stos_media_assets_list({ source: "generated", kind: "image" });
const imageUrl = /* the new asset's public url */;
```

### 4. Upload to WP media library

```js
const media = await stos_wp_media_create({
  fields: {
    url: imageUrl,
    alt: "Watercolor sketch of a writer at dawn",
    title: "Featured image: " + postTitle,
  }
});
```

### 5. Publish the post

```js
const post = await stos_wp_posts_create({
  fields: {
    title: postTitle,
    content: postHtml,
    excerpt: postExcerpt,
    status: "draft",           // publish only after the user says so
    featured_media: media.id,
    tags: postTags,            // string[]
    categories: postCategoryIds, // number[] — list via stos_wp_categories_list first
  }
});
const wpUrl = post.link; // user-facing URL
```

### 6. Draft the newsletter

```js
stos_email_campaigns_create({
  fields: {
    campaign_name: postTitle,   // required
    pen_name_id: penNameId,     // required to send; the sender comes from the pen name
    subject_line: postTitle,
    preheader: postExcerpt,
    email_body_html: `${teaserHtml /* the lede + a paragraph, not the whole post */}<p><a href="${wpUrl}">Read the full post on the blog →</a></p>`,
    list_ids: defaultListIds,   // ask user OR pull "Main list" via stos_email_lists_list
  }
});
```

The campaign starts as a draft. Confirm with the user before sending — never auto-send.

### 7. Confirm

Reply with:
- The published post URL
- The newsletter's name + a "Send when ready in Email Studio → Campaigns" line

## Decision points to surface

Before running silently, confirm:

| Decision | Default if user doesn't say |
|---|---|
| Publish or draft? | Draft. Ask before publishing. |
| Categories | Skip if user doesn't mention. Don't auto-assign. |
| Tags | Up to 5 you infer from content. Confirm. |
| Image style | Watercolor / soft / brand-matching when no preference. |
| Newsletter list | "Main list" if exactly one obvious match; otherwise ask. |
| Send newsletter or save as draft? | Always draft. The user sends from Email Studio. |

## Anti-patterns

- Do not send the newsletter automatically. The user reviews and sends.
- Do not skip the upload step. Linking directly to a StorytellerOS-hosted image can break when its link expires; the copy in the WordPress media library is the one to use.
- Do not auto-create new categories. If the user mentions one that doesn't exist, ask before creating with `stos_wp_categories_create`.
- Do not stuff the newsletter body with the entire post. A teaser + "read more →" link converts better and respects the WP site as the canonical home.

## Composes well with

- `wordpress-connect` — when the connection is missing
- `wordpress-media` — featured image work in detail
- `marketing-studio` — list / send the campaign once it's drafted
