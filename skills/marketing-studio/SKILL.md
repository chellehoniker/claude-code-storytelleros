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

### Signup forms + custom domains

Two surfaces let authors collect new subscribers without leaving STOS.

| Surface | What | Where |
|---|---|---|
| Hosted signup page | Public landing at `/f/<slug>` | `stos_email_signup_forms` |
| JS embed snippet | `<script async src="/api/forms/<slug>/embed.js"></script>` — drops a self-mounting iframe on any author site | Same row, same slug |
| Custom domain | CNAME `subscribe.<author-domain>` → app + TXT record on `_stos-form-verification.<domain>` for ownership proof | `stos_email_custom_domains` |

`form_type` is `landing_page` | `embed` | `popup`. `target_list_ids` + `target_tag_ids` decide where new subscribers land — they must belong to the same pen name as the form. `double_opt_in=true` (default) sends a confirmation email; the recipient clicks to flip subscription status from `pending` to `subscribed`.

Public submission endpoint:

```
POST /api/forms/<slug>/submit          # email, first_name, last_name, _hp (honeypot)
GET  /api/forms/confirm/<token>        # double-opt-in confirmation
```

Custom domain verification:

```js
stos_email_custom_domains_create({ fields: { pen_name_id, domain: 'subscribe.example.com' }})
// → set DNS records shown in the UI, then:
stos_email_custom_domains_verify({ id: '...' })  // checks the TXT record live
```

### Templates + activities

`stos_email_templates_user` stores author-saved designs (full templates, `is_section=false`) and reusable section snippets (`is_section=true`). `design_json` holds a block array; in-code starter templates remain in the campaign-create gallery as a read-only seed.

`stos_email_contact_activities` is the per-contact timeline that powers the subscriber-profile page. Triggers populate it automatically for subscribed / unsubscribed / confirmed / bounced / complained / added_to_list / removed_from_list / tagged / untagged / email_sent / email_opened / email_clicked / enrolled_in_automation / exited_automation / completed_automation. Manual inserts are supported for `note_added` and for backfill from external ESPs.

## Orchestration: STOS as ESP-of-record

A common use case is **author sends through a third-party ESP and uses STOS as the source of truth for audit, analytics, and cross-channel orchestration**. The MCP surface is designed to support this — every author-owned table carries source-provenance columns:

| Column | Use |
|---|---|
| `external_source` | Short slug of the originating system (`flodesk`, `mailerlite`, `kit`, `convertkit`, `manual`, `form:<slug>`, `stos-native`, etc.) |
| `external_id` | The id of this record in that external system — for dedupe + linking on re-sync |
| `external_url` | Deep-link back to the original record so Cowork can open it |
| `imported_at` | When STOS first saw this row (distinct from `created_at`, which is when STOS itself wrote the row) |

All four are nullable. STOS-native records (created via the dashboard or default MCP calls) leave them NULL.

### Typical flows

**Log a campaign that was sent through Flodesk:**

```js
// 1. Create the campaign in STOS with external_source so it's clear where it came from.
const campaign = stos_email_campaigns_create({
  fields: {
    pen_name_id: '<pen>',
    campaign_name: 'February newsletter',
    subject_line: 'February news',
    status: 'sent',
    external_source: 'flodesk',
    external_id: 'flo_abc123',
    external_url: 'https://app.flodesk.com/campaigns/abc123',
    imported_at: '2026-02-15T10:00:00Z',
  },
});

// 2. Backfill per-recipient sends + open/click events in one bulk call.
stos_email_campaigns_bulk_sends({
  id: campaign.id,
  external_source: 'flodesk',
  rows: [
    { email: 'reader@example.com', external_id: 'flo_msg_xyz', sent_at: '2026-02-15T10:00:00Z',
      opens: [{ at: '2026-02-15T11:23:00Z' }],
      clicks: [{ at: '2026-02-15T11:24:10Z', url: 'https://...' }] },
    ...
  ],
});

// 3. STOS computes aggregates (total_sent, unique_opens, unique_clicks)
//    from the inserted rows. The campaign now shows up in analytics
//    alongside STOS-native campaigns.
```

**Import a list of subscribers from MailerLite:**

```js
// For each subscriber:
stos_email_contacts_create({ fields: {
  email, first_name, last_name,
  external_source: 'mailerlite',
  external_id: 'ml_sub_id',
  imported_at: '2026-01-10T...',
}});
// Then attach to the appropriate list/tag the same way.
```

**Log a historical event from an old send:**

```js
stos_email_contact_activities_create({ fields: {
  contact_id, type: 'email_sent', description: 'Old newsletter',
  occurred_at: '2025-11-01T10:00:00Z',
  external_source: 'mailerlite',
  external_id: 'ml_campaign_id',
}});
```

### When NOT to use the bulk path

If the author wants STOS to actually **send** (open rate tracked via STOS pixel, unsubscribe via STOS footer, etc.), use the native flow:

```js
stos_email_campaigns_send({ id })  // STOS sends, STOS tracks
```

The bulk path is for read-only mirroring of sends that happened elsewhere — opens / clicks come from the source ESP's analytics, not from STOS's pixel (because STOS never put a pixel in those emails).

## Articles and webinars (read-only)

```js
stos_articles_list()   // recent Indie Author Magazine articles
stos_webinars_list()   // Indie Author Training webinars and talks
```

These two surface global content feeds — not author-owned records. They're read-only references with no create / update / delete: pull current articles or upcoming webinars into a campaign, a newsletter, or a recommendation. There is no FAQs tool — FAQ content is managed inside StorytellerOS directly.

## Anti-patterns

- **Trying to post to social through `stos_*`.** There's no `stos_social_*` tool. Use the `social-handoff` skill.
- **Sending a campaign with no `pen_name_id`.** Send will refuse it. Pen name drives sender identity and audience scope.
- **Forgetting list_ids / tag_ids.** Audience resolves to zero.
- **Treating unsubscribe as global.** It's per pen name — siblings under the same user are independent audiences.
- **Mass-mailing a personal list without consent confirmation.** If the user says "send to all my subscribers", check the resolved audience size first and surface that count before triggering the send.
- **Building an automation step before setting `automation_id`.** The step's audience and sender resolve through the automation; the step itself is just content + delay.
- **Flipping an automation to `active` without testing.** Test the rendered content first by creating a temporary throwaway list, enrolling yourself, and watching the step fire — there's no separate "test send" for automations the way there is for campaigns.
