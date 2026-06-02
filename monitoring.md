# Monitoring

[[toc]]

## Overview

The monitoring module turns the CRM into a lightweight uptime and SSL monitor for any HTTP/HTTPS endpoint you care about — your website, your API, customer-facing services, status pages, etc. Each monitor is checked on a schedule, response times are graphed, and alerts are emailed when an endpoint goes down or its SSL certificate is approaching expiry.

The module is enabled via the `monitoring` entry in the [`modules`](/configuration#optional-modules) configuration array.

## Surfaces

| Surface | Route | Description |
|---|---|---|
| Index | `/crm/monitors` | List of monitors with a 7-day performance sparkline |
| Show | `/crm/monitors/{external_id}` | Detail page with response-time chart and recent checks |
| Create / Edit | `/crm/monitors/create`, `/crm/monitors/{external_id}/edit` | CRUD forms |

## Models

| Model | Table | Description |
|---|---|---|
| `Monitor` | `{prefix}monitors` | The endpoint being monitored |
| `MonitorCheck` | `{prefix}monitor_checks` | An individual check result (response time, status code, errors) |

## Monitor

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in routes |
| `monitor_id` | `string` | Human-readable ID (e.g. `M0001`) |
| `name` | `string` | Display name (falls back to `host` or `url`) |
| `description` | `text` | Optional notes |
| `type` | `string` | `http` or `https` |
| `url` | `string(1024)` | Full URL to check |
| `host` | `string` | Hostname extracted from URL (used for SSL checks) |
| `method` | `string(16)` | HTTP method (default `GET`) |
| `headers` | `json` | Custom request headers |
| `body` | `text` | Optional request body |
| `expected_status_code` | `integer` | Expected HTTP status (default `200`) |
| `interval` | `integer` | Check frequency in minutes (default `5`) |
| `timeout` | `integer` | Request timeout in seconds (default `30`) |
| `is_active` | `boolean` | Toggle the monitor on/off |
| `uptime_enabled` | `boolean` | Run uptime checks |
| `ssl_enabled` | `boolean` | Run SSL certificate checks |
| `last_status_code` | `integer` | Last observed HTTP status |
| `last_checked_at` | `datetime` | When the monitor was last checked |
| `last_status_changed_at` | `datetime` | When the up/down state last changed |
| `down_since_at` | `datetime` | When the endpoint started being down |
| `notified_at` | `datetime` | When the last downtime alert was sent |
| `ssl_status` | `string` | `valid`, `expiring_soon`, `expired`, etc. |
| `ssl_issuer` | `string` | SSL issuer common name |
| `ssl_expires_at` | `datetime` | SSL expiry timestamp |
| `ssl_notified_at` | `datetime` | When the last SSL alert was sent |

### Relationships

| Method | Type | Description |
|---|---|---|
| `checks()` | `hasMany` | `MonitorCheck`, ordered by `checked_at` desc |
| `customFieldValues()` | `morphMany` | Custom field values |
| `ownerUser()` / `assignedToUser()` | `belongsTo` | CRM user owner / assignee |

### Traits

| Trait | Description |
|---|---|
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom-field support |
| `SoftDeletes` | Soft delete support |

## Monitor Check

Each scheduled run records a `MonitorCheck` row, used to draw the response-time chart and to evaluate uptime/SSL transitions.

| Attribute | Type | Description |
|---|---|---|
| `type` | `string` | `http` (uptime) or `ssl` |
| `status` | `string` | `up`, `down`, `valid`, `expired`, etc. |
| `response_time` | `integer` | Response time in milliseconds (uptime checks only) |
| `status_code` | `integer` | Observed HTTP status |
| `error_message` | `text` | Reason for failure if any |
| `response_body` | `longText` | Captured response body for failed checks |
| `ssl_expires_at` | `datetime` | Captured SSL expiry (SSL checks only) |
| `checked_at` | `datetime` | When the check ran |

Response time is measured via Guzzle's `on_stats` callback (transfer time), so it reflects actual network round-trip rather than wall-clock time.

## Charts

The show page renders a **response-time bar chart** with a period selector for **24 hours, 7 days, 30 days, 90 days, or 365 days**. The configured **performance threshold** is overlaid as a dotted line so you can see at a glance which checks crossed it.

The index page renders a **7-day sparkline** alongside each monitor for quick visual comparison.

## Scheduling Checks

The package registers a scheduled task that picks up due monitors and dispatches `RunMonitorCheck` jobs. As long as you have Laravel's scheduler running:

```cron
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

…monitors will be checked at their configured intervals.

You can also trigger a check manually from the CLI:

```bash
php artisan laravelcrm:monitor-check {monitor_id?}
```

If `monitor_id` is omitted the command picks up all monitors that are due.

## SSRF Protection

Because any CRM admin can create a monitor, the package guards against requests to internal infrastructure (loopback, private, link-local, and reserved IP ranges). Any monitor URL that resolves to such an address is rejected before the HTTP request is made.

To allow private targets (for example, when monitoring services on the same private network), set:

```env
LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS=true
```

## Notifications

When a monitor's status changes from up to down, or its SSL certificate is approaching expiry, a mail notification is sent to the monitor's owner and assignee. Notifications are de-duplicated using `notified_at` / `ssl_notified_at` so you don't get repeated emails for an ongoing incident.

## Configuration

```php
// config/laravel-crm.php
'monitoring' => [
    'default_frequency_minutes' => env('LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES', 5),
    'default_ssl_days_before_expiry_alert' => env('LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT', 14),
    'request_timeout_seconds' => env('LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS', 15),
    'ssl_recheck_hours' => env('LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS', 12),
    'allow_private_targets' => env('LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES` | `5` | Default check frequency for new monitors |
| `LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT` | `14` | Days before SSL expiry to trigger an alert |
| `LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS` | `15` | Default HTTP request timeout |
| `LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS` | `12` | How often to re-check SSL certificates |
| `LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS` | `false` | Allow monitors to target private/loopback IPs |

## Permissions

The package seeds the following permissions for the Monitoring module:

- `view monitors`
- `create monitors`
- `edit monitors`
- `delete monitors`
- `view monitor checks`

These are wired up in `MonitorPolicy` and `MonitorCheckPolicy`.
