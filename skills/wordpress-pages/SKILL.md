---
name: wordpress-pages
description: Use when the user wants to update or create static pages on their connected WordPress site (About, Contact, Books, Privacy, etc.). Triggers on "update my About page", "create a Books page", "edit the Contact page", "what does my homepage look like?".
---

# WordPress pages

Pages are the static content on a WordPress site (About, Contact, Books, Privacy, Terms, etc.). Different from posts: no categories, no tags, usually long-lived.

## Tools

```js
stos_wp_pages_list({ status, search, page, per_page })
stos_wp_pages_get({ id })
stos_wp_pages_create({ fields: { title, content, excerpt, slug, status, featured_media } })
stos_wp_pages_update({ id, fields: { ... } })
stos_wp_pages_delete({ id, force })
```

All tools take an optional `penNameId` — the user can connect one WordPress site per pen name plus an account default. Omit for the account default; pass a pen name id (from `stos_pen_names_list`) to act on that pen name's site. When the user has multiple pen names and it's ambiguous which site they mean, ask.

Field shapes match `wordpress-posts` minus `categories`, `tags`, and `date` (pages don't schedule the same way).

## Patterns

### Refresh the About page

```js
// Find the page first
const list = await stos_wp_pages_list({ search: "About" });
const aboutPage = list.data[0];

stos_wp_pages_update({
  id: aboutPage.id,
  fields: {
    content: "<p>Updated bio...</p>",
  }
});
```

### Create a new "Books" landing page

```js
stos_wp_pages_create({
  fields: {
    title: "Books",
    slug: "books",
    content: "<h2>Available now</h2><p>...</p>",
    status: "publish",
  }
});
```

## Anti-patterns

- Do not delete pages without confirming. Pages often anchor menu items and breaking links is silent.
- Do not change the slug of a published page without warning the user — it breaks any external link to that URL.

## Composes well with

- `wordpress-media` — featured images for landing-style pages
