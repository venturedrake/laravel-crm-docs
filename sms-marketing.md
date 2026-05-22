# SMS Marketing

[[toc]]

## Overview

The SMS marketing module sends broadcast SMS campaigns to subscribed CRM contacts via the [ClickSend](https://clicksend.com/?u=47224) HTTP API. Campaigns can be drafted, scheduled, and sent. Click tracking is captured via shortened redirect URLs, and unsubscribes are honoured against the contact's phone subscription flag.

The module is enabled via the `sms-marketing` entry in the [`modules`](/configuration#optional-modules) configuration array and is gated by the `@hassmsmarketingenabled` Blade directive.

## Models

| Model | Table | Description |
|---|---|---|
| `SmsTemplate` | `{prefix}sms_templates` | Reusable SMS template |
| `SmsCampaign` | `{prefix}sms_campaigns` | A scheduled or sent SMS campaign |
| `SmsCampaignRecipient` | `{prefix}sms_campaign_recipients` | One row per recipient with delivery status |
| `SmsCampaignClick` | `{prefix}sms_campaign_clicks` | Tracked click event |

## SMS Template

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Template name |
| `body` | `text` | SMS body |
| `is_system` | `boolean` | Flagged when shipped with the package |

System templates are seeded by the `seed_laravel_crm_sms_templates` migration. Custom templates can be created at any time via **Settings → SMS Templates**.

## SMS Campaign

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs |
| `campaign_id` | `string` | Human-readable ID (e.g. `SC1001`) |
| `name` | `string` | Internal campaign name |
| `body` | `text` | SMS message body |
| `from` | `string` | Sender ID (number or alphanumeric, depending on country) |
| `sms_template_id` | `integer` | Source template |
| `status` | `enum` | `draft`, `scheduled`, `sending`, `sent`, `cancelled`, `failed` |
| `scheduled_at` | `datetime` | When the campaign is queued to send |
| `timezone` | `string` | Timezone of the scheduled time |
| `sent_at` | `datetime` | When sending finished |
| `total_recipients` | `integer` | Total recipient count |
| `sent_count`, `delivered_count`, `failed_count` | `integer` | Delivery metrics |
| `clicks_count`, `unique_clicks_count` | `integer` | Click metrics |
| `unsubscribes_count` | `integer` | Unsubscribes attributed to the campaign |

## Sending Pipeline

1. Draft a campaign in **Marketing → SMS Campaigns** (or duplicate an existing one).
2. Choose recipients from the CRM contacts. Only contacts with a subscribed phone number are eligible.
3. Schedule the campaign or send immediately.
4. The scheduled `laravelcrm:sms-campaigns-dispatch` command (every minute) finds due campaigns and queues:
   - `Jobs/MaterialiseSmsCampaignRecipients` — expands the recipient list.
   - `Jobs/SendSmsCampaignRecipient` — sends each message via `Sms/SmsCampaignMessage` and `ClickSendService`.
5. Each rendered message has its links rewritten through `/p/sms/c/{token}` for click tracking and includes an unsubscribe path at `/p/sms/u/{token}`.

> Tracking routes live in `src/Http/email-tracking-routes.php` (shared with email tracking) and are registered **outside** the `web` middleware group so SMS click links work without a session.

## ClickSend Configuration

SMS sending is delegated to `VentureDrake\LaravelCrm\Services\ClickSendService`, which calls `https://rest.clicksend.com/v3` with HTTP Basic auth. Credentials are stored as CRM settings (managed via **Settings → Integrations → ClickSend**), not as `.env` values:

| Setting key | Description |
|---|---|
| `clicksend_username` | Your ClickSend username |
| `clicksend_api_key` | Your ClickSend API key |
| `clicksend_default_from` | Default sender ID for outbound SMS |

## Scheduled Command

```bash
php artisan laravelcrm:sms-campaigns-dispatch
```

Auto-registered to run **every minute** by the package service provider. Add `php artisan schedule:run` to your application's cron for it to take effect.

## Service

`VentureDrake\LaravelCrm\Services\SmsCampaignService` handles create/update orchestration. `SmsTemplateService` handles template CRUD. `ClickSendService` is the underlying HTTP client.

