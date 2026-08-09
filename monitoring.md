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
| `timeout` | `integer` | Column default `30` — see the note under [Configuration](#configuration) |
| `is_active` | `boolean` | Toggle the monitor on/off |
| `uptime_enabled` | `boolean` | Run uptime checks |
| `ssl_enabled` | `boolean` | Run SSL certificate checks |
| `last_status_code` | `integer` | Last observed HTTP status |
| `last_checked_at` | `datetime` | When the monitor was last checked |
| `last_status_changed_at` | `datetime` | When the up/down state last changed |
| `down_since_at` | `datetime` | When the endpoint started being down |
| `notified_at` | `datetime` | When the last downtime alert was sent |
| `perf_notified_at` | `datetime` | When the last slow-response alert was sent |
| `recovered_notified_at` | `datetime` | When the last recovery alert was sent |
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

When a monitor's status changes from up to down, or its SSL certificate is approaching expiry, a mail notification is sent to the monitor's owner and assignee.

| Notification | Sent when |
|---|---|
| `MonitorDownNotification` | The endpoint has been down for longer than the debounce window |
| `MonitorPerformanceNotification` | A check comes back `slow` — response time over the performance threshold |
| `MonitorRecoveredNotification` | The endpoint comes back up, and a down or slow alert actually preceded it |

## Alert Rate Limiting

Each alert type carries its own "last sent" timestamp on the monitor, so an ongoing incident produces one email rather than one per check. Before 2.4.0 only downtime was rate-limited this way, so a monitor sitting just over its performance threshold sent one email per check — a 5-minute monitor mailed its owner 288 times a day.

| Alert | Timestamp | Window | Key |
|---|---|---|---|
| Down | `notified_at` | Debounce before the first alert | `monitoring.down_debounce_minutes` (2) |
| Slow | `perf_notified_at` | Minimum gap between alerts | `monitoring.perf_alert_rate_limit_minutes` (60) |
| Recovered | `recovered_notified_at` | Minimum gap between alerts | `monitoring.recovered_alert_rate_limit_minutes` (60) |
| SSL | `ssl_notified_at` | Minimum gap between alerts | `monitoring.ssl_alert_rate_limit_hours` (24) |

Two details worth knowing:

- **A recovery alert only fires when a down or slow alert preceded it.** A monitor that flickered without ever tripping an alert recovers silently, rather than announcing a resolution to an incident nobody was told about.
- **The down alert is debounced, not rate-limited.** `down_since_at` records when the endpoint first failed, and the alert is held until it has been down for `down_debounce_minutes`. A monitor's own `downtime_minutes_before_alert` overrides the config value when set. `notified_at` is written *before* the notification is dispatched, so a job retry after a downstream failure cannot re-send it.

`notified_at`, `perf_notified_at` and `down_since_at` are all cleared when the endpoint comes back up, so the next incident starts from a clean slate.

> **Note:** The four rate-limit keys are read by `RunMonitorCheck` but are **not present in the shipped config file**. They fall back to the defaults above; add them to your published `config/laravel-crm.php` to change them. See [Configuration → Monitoring](/configuration#monitoring).

## Configuration

```php
// config/laravel-crm.php
'monitoring' => [
    'default_frequency_minutes' => env('LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES', 5),
    'default_ssl_days_before_expiry_alert' => env('LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT', 14),
    'request_timeout_seconds' => env('LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS', 15),
    'ssl_recheck_hours' => env('LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS', 12),
    'max_response_bytes' => env('LARAVEL_CRM_MONITORING_MAX_RESPONSE_BYTES', 5 * 1024 * 1024),
    'allow_private_targets' => env('LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES` | `5` | Default check frequency for new monitors |
| `LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT` | `14` | Days before SSL expiry to trigger an alert |
| `LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS` | `15` | HTTP request timeout used for every check |
| `LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS` | `12` | How often to re-check SSL certificates |
| `LARAVEL_CRM_MONITORING_MAX_RESPONSE_BYTES` | `5242880` (5 MiB) | Cap on the response body a check will read |
| `LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS` | `false` | Allow monitors to target private/loopback IPs |

> **Note:** There are two timeouts and they are not the same thing. `crm_monitors.timeout` is a per-monitor column whose schema default is **30**, carried for forward compatibility. The timeout actually applied to every HTTP and SSL check is `monitoring.request_timeout_seconds`, which defaults to **15** — `MonitorCheckService` reads the config value, not the column.

> **Note:** `max_response_bytes` caps how much of a response body a check will read, so a monitored endpoint streaming an unbounded response cannot exhaust the queue worker's memory. It ships as 5 MiB in the config file, but the code falls back to **2 MiB** when the key is absent — so an install with a `config/laravel-crm.php` published before 2.4.0 gets 2 MiB until the key is added.

## Permissions

The package seeds the following permissions for the Monitoring module:

- `view crm monitors`
- `create crm monitors`
- `edit crm monitors`
- `delete crm monitors`

These are wired up in `MonitorPolicy`. `MonitorCheckPolicy` delegates every method to `MonitorPolicy` against the check's parent monitor, so there is no separate permission for checks.

> **Important:** Neither **Manager** nor **Employee** holds any `crm monitors` permission. Monitoring is Owner/Admin only under the stock roles — grant the permissions explicitly under **Settings → Roles** if others need it. See [Roles](/roles).
