# Email Marketing

[[toc]]

## Overview

The email marketing module lets you send broadcast email campaigns to subscribed CRM contacts using reusable templates. Campaigns can be drafted, scheduled, and sent to a recipient list. Open and click tracking is captured via tracking pixel and redirect URLs, and unsubscribes are honoured against the contact's email subscription flag.

The module is enabled via the `email-marketing` entry in the [`modules`](/configuration#optional-modules) configuration array and is gated by the `@hasemailmarketingenabled` Blade directive.

## Models

| Model | Table | Description |
|---|---|---|
| `EmailTemplate` | `{prefix}email_templates` | Reusable email template (subject + body) |
| `EmailCampaign` | `{prefix}email_campaigns` | A scheduled or sent campaign |
| `EmailCampaignRecipient` | `{prefix}email_campaign_recipients` | One row per recipient with status |
| `EmailCampaignClick` | `{prefix}email_campaign_clicks` | Tracked click event |

## Email Template

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Template name |
| `subject` | `string` | Default subject line |
| `preview_text` | `string` | Preview text shown by email clients |
| `body` | `longText` | HTML body, edited via TinyMCE in the CRM UI |
| `is_system` | `boolean` | Flagged when the template ships with the package |

System templates are seeded by the `seed_laravel_crm_email_templates` migration. Custom templates can be created at any time via **Settings → Email Templates**.

## Email Campaign

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs |
| `campaign_id` | `string` | Human-readable ID (e.g. `EC1001`) |
| `name` | `string` | Internal campaign name |
| `subject`, `preview_text`, `body` | various | Email content (copied from the chosen template, then editable) |
| `email_template_id` | `integer` | Source template |
| `status` | `enum` | `draft`, `scheduled`, `sending`, `sent`, `cancelled`, `failed` |
| `scheduled_at` | `datetime` | When the campaign is queued to send |
| `timezone` | `string` | Timezone of the scheduled time |
| `audience_type` | `string` | Who the campaign goes to: `all`, `labels`, `organizations`, `pipeline_stages` or `people` (default `all`, since 2.7.0) |
| `audience_ids` | `json` | Ids of the chosen labels, organizations, pipeline stages or people; empty for `all` (since 2.7.0) |
| `sent_at` | `datetime` | When sending finished |
| `total_recipients` | `integer` | Total recipient count |
| `opens_count`, `unique_opens_count` | `integer` | Open metrics |
| `clicks_count`, `unique_clicks_count` | `integer` | Click metrics |
| `unsubscribes_count` | `integer` | Unsubscribes attributed to the campaign |

### Helpers

```php
$campaign->isEditable();    // true when status === 'draft'
$campaign->isCancellable(); // true when status === 'scheduled'
$campaign->openRate();      // unique opens / total recipients (%)
$campaign->clickRate();     // unique clicks / total recipients (%)
```

## Audiences

Since 2.7.0 a campaign targets an audience, chosen on the create and edit forms:

| `audience_type` | Recipients |
|---|---|
| `all` | Every person with a subscribed email |
| `labels` | People with any of the chosen labels |
| `organizations` | People whose organization is one of those chosen |
| `pipeline_stages` | People with a lead or deal in any of the chosen stages |
| `people` | The chosen people |

Recipients are always people with a subscribed email address, and an address is sent to once even if it matches twice. A filtered audience with nothing chosen matches nobody; it never falls back to everyone. The form shows a live recipient count as you choose. Labels and pipeline stages are listed in full; people and organizations are searched on the server as you type, up to 20 matches at a time.

- **New campaigns save as drafts.** Choosing **Send now** asks you to confirm with a second save.
- **An empty audience can't be sent.** The form flags it on the audience field.
- **A draft can be scheduled from its edit page.**

Campaigns created before 2.7.0 get `audience_type = all`, so they keep sending to all subscribed contacts.

## Placeholders

Template and campaign content uses single-brace placeholders such as `{first_name}`. The email and SMS renderers also accept `{{first_name}}` and replace it whole, so content seeded that way before 2.7.0 renders correctly.

- **The subject is substituted too.** Since 2.7.0 `EmailCampaignMessage::build()` fills in the subject's placeholders. Like the preview text, the subject is not HTML-escaped. Before, subjects went out with their placeholders unreplaced, and the preview text was escaped twice.
- **The preview drawer** on campaigns and templates shows the subject and fills in the subject and preview text with a sample recipient.
- **Sample data** from `php artisan laravelcrm:sample-data` writes `{key}` placeholders since 2.7.0 ([#137](https://github.com/venturedrake/laravel-crm/issues/137)). Demo data seeded earlier may still show `{{product_name}}`, `{{discount}}` or `{{order_id}}` literally, as those placeholders never existed; run `php artisan laravelcrm:sample-data --fresh` to replace it.

## Sending Pipeline

1. Draft a campaign in **Marketing → Email Campaigns** (or duplicate an existing one).
2. Choose its [audience](#audiences). Only contacts with a subscribed email are eligible.
3. Schedule the campaign or choose **Send now** and confirm.
4. The scheduled `laravelcrm:email-campaigns-dispatch` command (every minute) finds due campaigns and queues per-recipient jobs:
   - `Jobs/SendEmailCampaignRecipient` — renders and sends the email via `Mail/EmailCampaignMessage`.
5. Each rendered email contains a tracking pixel (`/p/email/o/{token}.gif`), rewritten click links (`/p/email/c/{token}`), and an unsubscribe link (`/p/email/u/{token}`).

> Tracking routes live in `src/Http/email-tracking-routes.php` and are registered **outside** the `web` middleware group so they work from any email client.

## Scheduled Command

```bash
php artisan laravelcrm:email-campaigns-dispatch
```

Auto-registered to run **every minute** by the package service provider. Add `php artisan schedule:run` to your application's cron for it to take effect.

## Service

`VentureDrake\LaravelCrm\Services\EmailCampaignService` handles create/update orchestration. `EmailTemplateService` handles template CRUD.

