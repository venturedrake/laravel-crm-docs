# Configuration

[[toc]]

After publishing the package assets, a configuration file will be located at `config/laravel-crm.php`. Below is a reference for all available settings.

## CRM Owner

The primary owner of the CRM. This must be set to the email address of a registered user so you can access the CRM initially.

```php
'crm_owner' => env('LARAVEL_CRM_OWNER', ''),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_OWNER` | `''` | Email address of the CRM owner |

## Teams Support

Enable multi-tenant support using Laravel Jetstream or Spark teams. Each team acts as a separate account with its own users, contacts, leads, etc.

```php
'teams' => env('LARAVEL_CRM_TEAMS', false),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_TEAMS` | `false` | Enable teams/multi-tenant support |

> **Important:** Only enable this if you are using Jetstream or Spark teams. Enabling without the feature installed will break your installation. This requires Spatie Permissions v5+ with teams support enabled. See the [Spatie docs](https://spatie.be/docs/laravel-permission/v5/basic-usage/teams-permissions) for setup.

> **Note:** This is unrelated to the user teams feature within the CRM itself, which is simply a way of grouping users.

## Default Settings

These defaults are also used each time a user creates a new team when team support is enabled.

```php
'currency' => env('LARAVEL_CRM_CURRENCY', 'USD'),
'country' => env('LARAVEL_CRM_COUNTRY', 'United States'),
'language' => env('LARAVEL_CRM_LANGUAGE', 'english'),
'timezone' => env('LARAVEL_CRM_TIMEZONE', 'UTC'),
'date_format' => env('LARAVEL_CRM_DATE_FORMAT', 'Y-m-d'),
'time_format' => env('LARAVEL_CRM_TIME_FORMAT', 'g:i A'),
'tax_name' => env('LARAVEL_CRM_TAX_NAME', 'Tax'),
'tax_rate' => env('LARAVEL_CRM_TAX_RATE', null),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_CURRENCY` | `USD` | Default currency code |
| `LARAVEL_CRM_COUNTRY` | `United States` | Default country |
| `LARAVEL_CRM_LANGUAGE` | `english` | Default language |
| `LARAVEL_CRM_TIMEZONE` | `UTC` | Default timezone |
| `LARAVEL_CRM_DATE_FORMAT` | `Y-m-d` | PHP date format string |
| `LARAVEL_CRM_TIME_FORMAT` | `g:i A` | PHP time format string |
| `LARAVEL_CRM_TAX_NAME` | `Tax` | Label for tax on quotes/invoices |
| `LARAVEL_CRM_TAX_RATE` | `null` | Default tax rate percentage |

## Route Subdomain

Serve the CRM on a subdomain, e.g. `https://crm.yourdomain.com`.

```php
'route_subdomain' => env('LARAVEL_CRM_ROUTE_SUBDOMAIN', null),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_ROUTE_SUBDOMAIN` | `null` | Subdomain for CRM routes |

## Route Prefix

Define the URL prefix for the CRM. Set to a subfolder like `crm` or leave blank to serve from the root.

```php
'route_prefix' => env('LARAVEL_CRM_ROUTE_PREFIX', 'crm'),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_ROUTE_PREFIX` | `crm` | URL prefix (e.g. `/crm`) |

> **Tip:** Use a subfolder prefix if you are installing the CRM into an existing Laravel project that already has its own routes and controllers.

## Route Middleware

Add any custom middleware to the CRM routes.

```php
'route_middleware' => [],
```

Pass an array of middleware class names or aliases to apply to all CRM routes.

## Database Table Prefix

All CRM database tables are prefixed with this value.

```php
'db_table_prefix' => env('LARAVEL_CRM_DB_TABLE_PREFIX', 'crm_'),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_DB_TABLE_PREFIX` | `crm_` | Prefix for all CRM tables |

> **Tip:** If installing into an existing project, keep the default `crm_` prefix to avoid table name conflicts.

## Database Field Encryption

Encrypt personal information in certain database fields as an added layer of privacy protection.

```php
'encrypt_db_fields' => env('LARAVEL_CRM_ENCRYPT_DB_FIELDS', false),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_ENCRYPT_DB_FIELDS` | `false` | Enable encryption of sensitive fields |

## User Interface

The CRM comes with a built-in user interface. Disable this if you want to build your own frontend.

```php
'user_interface' => env('LARAVEL_CRM_USER_INTERFACE', true),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_USER_INTERFACE` | `true` | Enable the built-in UI |

> **Note:** When disabled, routes using the CRM route prefix will not load, preventing users from accessing the default views.

## Optional Modules

Enable or disable CRM modules based on your business needs. Remove a module from the array to disable it.

```php
'modules' => [
    'leads',
    'deals',
    'quotes',
    'orders',
    'invoices',
    'deliveries',
    'purchase-orders',
    'teams',
    'chat',
    'email-marketing',
    'sms-marketing',
    'features',
    'monitoring',
],
```

| Module | Description |
|---|---|
| `leads` | Lead management and pipeline tracking |
| `deals` | Deal/opportunity tracking through pipeline stages |
| `quotes` | Quote generation with line items and PDF export |
| `orders` | Order management with fulfillment tracking |
| `invoices` | Invoice management with payment tracking and PDF export |
| `deliveries` | Delivery tracking for physical goods |
| `purchase-orders` | Purchase order management for suppliers |
| `teams` | User team grouping within the CRM |
| `chat` | Live chat with embeddable widget — see [Chat](/chat) |
| `email-marketing` | Email campaigns and templates — see [Email Marketing](/email-marketing) |
| `sms-marketing` | SMS campaigns and templates — see [SMS Marketing](/sms-marketing) |
| `features` | Public feature-request and voting board — see [Features](/features) |
| `monitoring` | Uptime and SSL monitoring for HTTP/HTTPS endpoints — see [Monitoring](/monitoring) |

> **Tip:** If you sell digital products or services, you can remove `deliveries` since it won't be relevant.

## Monitoring

Defaults for the [Monitoring](/monitoring) module. These are used by `MonitorService` when creating monitors without explicit values, and by the `RunMonitorCheck` job when scheduling and evaluating checks.

```php
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
| `LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES` | `5` | Default check frequency for new monitors (minutes) |
| `LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT` | `14` | Days before SSL expiry to trigger an alert |
| `LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS` | `15` | Default HTTP request timeout (seconds) |
| `LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS` | `12` | How often to re-check SSL certificates |
| `LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS` | `false` | Allow monitors to target private/loopback IPs (off by default to prevent SSRF) |

## Portal

Settings for the public-facing [Portal](/portal) (feature board, signed quote/invoice/purchase-order links).

```php
'portal' => [
    'allow_registration' => env('LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION` | `false` | Allow visitors to self-register on the portal so they can vote and comment on features |

> **Note:** When enabled, `/p/register` writes rows to the host application's `users` table and dispatches Laravel's `Registered` event for each signup.

## Features

Settings for the [Features](/features) module.

```php
'features' => [
    'view_dedup_minutes' => env('LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES', 60),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES` | `60` | View de-duplication window in minutes. Set to `0` to record every page view. |

## Update Notifications

Show or hide package update notifications for CRM users.

```php
'update_notifications' => env('LARAVEL_CRM_UPDATE_NOTIFICATIONS', true),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_UPDATE_NOTIFICATIONS` | `true` | Show update notifications |

## Models with Global

In multi-tenant mode, certain model tables can have rows marked as global (shared across all teams) using a `global` column.

```php
'model_with_global' => [
    'settings',
],
```

By default, only `settings` are global. Add other model names to share their data across teams.
