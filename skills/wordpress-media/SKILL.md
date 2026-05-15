---
name: wordpress-media
description: Use when the user wants to upload, list, or manage images on their connected WordPress site, or set a featured image on a post or page. Triggers on "upload this image to WordPress", "use this as the featured image", "add to my media library", "list my recent uploads".
---

# WordPress media

Images and files in the connected WordPress site's media library. Most often used as featured images for posts and pages.

## Tools

```js
stos_wp_media_list({ page, per_page })
stos_wp_media_get({ id })
stos_wp_media_create({ fields: { url?, data_base64?, filename?, mime?, title?, alt? } })
stos_wp_media_delete({ id, force })
```

## Two upload modes

### URL ingest (preferred)

Pass a public URL — WordPress fetches and stores the bytes.

```js
const media = await stos_wp_media_create({
  fields: {
    url: "https://example.com/cover.jpg",
    alt: "Book cover for Demon at Dusk",
  }
});
```

### Inline base64 (firewalled / non-public sources)

When the source isn't reachable from the user's WordPress server, pass the bytes directly.

```js
stos_wp_media_create({
  fields: {
    filename: "cover.jpg",
    mime: "image/jpeg",
    data_base64: "...",     // base64-encoded image bytes
    alt: "Book cover",
    title: "Demon at Dusk cover",
  }
});
```

## Featured image flow

```js
// 1. Upload
const media = await stos_wp_media_create({ fields: { url: "https://...", alt: "..." } });

// 2. Attach
stos_wp_posts_update({ id: postId, fields: { featured_media: media.id } });
```

## Anti-patterns

- Do not upload the same file twice if you can reuse an existing one. List first, upload only when nothing matches.
- Always pass `alt` text. WordPress sites should have it for accessibility and SEO.

## Composes well with

- `wordpress-posts`, `wordpress-pages`, `wordpress-products` — anywhere a `featured_media` ID is needed
- `blog-post-and-newsletter` — generates an image and uploads it as part of the pipeline
