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

## Sending Pipeline

1. Draft a campaign in **Marketing → Email Campaigns** (or duplicate an existing one).
2. Choose recipients from the CRM contacts. Only contacts with a subscribed email are eligible.
3. Schedule the campaign or send immediately.
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

