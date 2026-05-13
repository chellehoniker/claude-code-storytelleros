---
name: marketing-studio
description: Use when the user wants to manage email lists, contacts, tags, email campaigns, articles, FAQs, or webinars. EXCLUDES social — social work routes to the Author Automations plugin via the `social-handoff` skill. Triggers on "add subscribers to [list]", "draft a newsletter", "send my latest campaign", "publish an article", "add a webinar".
---

# Marketing Studio

## What this covers vs. social

| Channel | Skill |
|---|---|
| Email (lists, campaigns, sends) | `marketing-studio` (this) |
| Articles / long-form posts | `marketing-studio` |
| FAQs, webinars | `marketing-studio` |
| Social posts / scheduling / campaigns | `social-handoff` → AA plugin |

This skill never calls a STOS social tool. There isn't one.

## Email

### Subscribers + lists + tags

```js
stos_email_contacts_list({ })  // optional filters
stos_email_contacts_create({ fields: { email, first_name, last_name, ... } })
stos_email_lists_list()
stos_email_lists_create({ fields: { list_name, description } })
stos_email_tags_list()
```

### Campaigns

```js
stos_email_campaigns_list()
stos_email_campaigns_get({ id })
stos_email_campaigns_create({ fields: { subject, preview_text, body_html, list_ids: [...] } })
```

### Send

```js
stos_email_campaigns_send({ id })                          // live send
stos_email_campaigns_send({ id, testEmails: ['a@b.com'] }) // test send
stos_email_campaigns_send({ id, scheduledDate: 'ISO' })    // schedule
```

> Note: the live-send engine is currently a stub and returns 503. The endpoint is plumbed and will start working when the platform's send infra ships. For now, drafts save fine but `_send` will surface a friendly "next update" error.

## Articles, FAQs, webinars

```js
stos_articles_list()
stos_articles_create({ fields: { title, slug, body_html, status, published_at } })
stos_faqs_list()
stos_faqs_create({ fields: { question, answer, category } })
stos_webinars_list()
stos_webinars_create({ fields: { title, description, start_at, end_at, registration_url } })
```

## Anti-patterns

- **Trying to post to social through `stos_*`.** There's no `stos_social_*` tool. Use the `social-handoff` skill.
- **Forgetting to associate campaigns with lists.** A campaign with no `list_ids` will fail to send (when send ships) or send to zero recipients.
- **Mass-mailing a personal list without consent confirmation.** If the user says "send to all my subscribers", check the list size first — surface that count before triggering the send.
