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

### Audience model

Subscription is **per (contact, pen name)**, not per contact. One email contact belongs to the user, but its subscribed/unsubscribed state lives on `email_pen_name_subscriptions` rows — one row per audience (pen name).

- A reader can be subscribed under pen name A and unsubscribed under pen name B even though it's the same email address.
- Unsubscribe links scope to the sending pen name only. Never globally unsubscribe a contact in response to one audience opting out.
- Every campaign **must** have a `pen_name_id`. Send will refuse a campaign without one.
- Sender identity (from-name, from-address, SMTP credentials, branding) lives on `email_pen_name_senders` keyed by `pen_name_id`. One sender row per pen name.

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
stos_email_campaigns_create({ fields: { subject, preview_text, body_html, pen_name_id, list_ids: [...], tag_ids: [...] } })
```

### Before you can send

Verify all of the following before calling `_send`. The send will fail otherwise:

1. Campaign has `pen_name_id` set.
2. That pen name has an `email_pen_name_senders` row configured — the user sets this up at `/dashboard/settings/email-connection`.
3. Campaign has at least one of `list_ids` or `tag_ids` selected (otherwise the audience resolves to zero).

If the user says "send to all my subscribers", resolve the audience count first and surface it before triggering.

### Send

One tool, three modes — the params decide which:

```js
// Live send — fires immediately
stos_email_campaigns_send({ id })

// Schedule — stamps scheduled_at; the cron picks it up within ~1 min of that time
stos_email_campaigns_send({ id, scheduledDate: '2026-05-20T18:00:00Z' })

// Test send — renders + delivers to the listed addresses only.
// No tracking, no recipient writes; subject is prefixed "[Test]".
stos_email_campaigns_send({ id, testEmails: ['me@example.com'] })
```

`scheduledDate` is an ISO-8601 string. There is no separate `stos_email_campaigns_schedule` tool — scheduling is just `_send` with `scheduledDate`.

### Tracking

Every live (non-test) campaign HTML is instrumented automatically:

- A tracking pixel records opens.
- Outbound links are rewritten to record clicks before redirecting.

Counters:

- `email_campaign_sends.open_count` / `click_count` — per recipient.
- `email_campaigns.total_opens`, `unique_opens`, `total_clicks`, `unique_clicks` — campaign aggregates.

Test sends do not write tracking.

### Automations (sequences / drip / welcome series)

Automations are author-defined sequences of emails sent to one contact over time. Three database surfaces:

- `stos_email_automations` — the sequence definition (pen name, status, trigger type, trigger config).
- `stos_email_automation_steps` — ordered emails within a sequence with `delay_value` + `delay_unit` between them.
- `stos_email_automation_enrollments` — one row per (automation, contact) pair tracking position and status.

Trigger types:

| `trigger_type`     | `trigger_config` shape                                            | Fires when                                                                  |
|--------------------|--------------------------------------------------------------------|------------------------------------------------------------------------------|
| `list_subscribed`  | `{ "list_id": "<uuid>" }`                                          | Contact added to that list (DB trigger; instant).                            |
| `tag_added`        | `{ "tag_id": "<uuid>" }`                                           | Contact gets that tag (DB trigger; instant).                                 |
| `date_field`       | `{ "custom_field": "<key>", "offset_days": 0, "recurring": "yearly" }` | Daily cron at 09:00 UTC scans contacts where `custom_fields[key]` ≈ today.   |
| `manual`           | `{}`                                                               | Only via `stos_email_automations_enroll`.                                    |

Author the sequence:

```js
const auto = stos_email_automations_create({ fields: {
  pen_name_id, name: 'Welcome series',
  trigger_type: 'list_subscribed',
  trigger_config: { list_id: '<reader-magnet-list>' },
  status: 'active',
}});

stos_email_automation_steps_create({ fields: {
  automation_id: auto.id, position: 0, delay_value: 0,  delay_unit: 'minutes',
  subject_line: 'Your free book is here', email_body_html: '<p>…</p>',
}});
stos_email_automation_steps_create({ fields: {
  automation_id: auto.id, position: 1, delay_value: 3,  delay_unit: 'days',
  subject_line: 'Day 3 — what to read next', email_body_html: '<p>…</p>',
}});
```

Trigger semantics:

- **Active subscriptions only.** Enrollments require the contact to hold a `status='subscribed'` row on the automation's pen name. Manual enrollment silently skips ineligible contacts.
- **Unsubscribe exits.** When `exit_on_unsubscribe=true` (the default), an unsubscribe flips the contact's enrollments on that pen name to `status='exited'`.
- **One enrollment per (automation, contact).** Re-enrolling a contact who exited requires deleting their old enrollment first via `stos_email_automation_enrollments_delete`.
- **Pause vs Archive.** `status='paused'` on an automation stops new enrollments but preserves in-flight ones (their steps still fire). `status='archived'` is for historical record-keeping; treat as read-only.

Manual / API enrollment:

```js
stos_email_automations_enroll({ id: auto.id, contactIds: ['…','…'] })
// → { enrolled: 47, skipped: 3, automationStatus: 'active' }
```

Tracking parity:

Automation emails use the same open/click pipeline as campaign emails. Per-recipient counters live on `email_campaign_sends` keyed by `(enrollment_id, automation_step_id)`; the table's `campaign_id` is NULL for these rows.

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
- **Sending a campaign with no `pen_name_id`.** Send will refuse it. Pen name drives sender identity and audience scope.
- **Forgetting list_ids / tag_ids.** Audience resolves to zero.
- **Treating unsubscribe as global.** It's per pen name — siblings under the same user are independent audiences.
- **Mass-mailing a personal list without consent confirmation.** If the user says "send to all my subscribers", check the resolved audience size first and surface that count before triggering the send.
- **Building an automation step before setting `automation_id`.** The step's audience and sender resolve through the automation; the step itself is just content + delay.
- **Flipping an automation to `active` without testing.** Test the rendered content first by creating a temporary throwaway list, enrolling yourself, and watching the step fire — there's no separate "test send" for automations the way there is for campaigns.
