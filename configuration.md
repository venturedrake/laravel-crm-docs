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

> **Note:** This key is about **host-application teams** — the Jetstream or Spark tenant a user is currently switched to. It is unrelated to the CRM's own `teams` module (`crm_teams`), which groups users inside a single tenant. See [Teams](/teams) for how the two differ.

## Host Team Model

The Eloquent model in the host application that represents a team a user can switch to — typically Jetstream's or a starter kit's `App\Models\Team`. When set, the **+ New enterprise** link in the CRM header uses this model to create the team and switches the user's current team to it.

```php
'host_team_model' => env('LARAVEL_CRM_HOST_TEAM_MODEL'),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_HOST_TEAM_MODEL` | `null` | Fully-qualified host team model. Leave unset to auto-detect via the user's `ownedTeams()` relationship |

> **Tip:** Set this if your host application's own `teams.create` route sits behind middleware the CRM cannot satisfy — Jetstream's `hasNoTeam`, for instance. See [Teams → Creating a Team from the CRM](/teams#creating-a-team-from-the-crm).

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
    'max_response_bytes' => env('LARAVEL_CRM_MONITORING_MAX_RESPONSE_BYTES', 5 * 1024 * 1024),
    'allow_private_targets' => env('LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES` | `5` | Default check frequency for new monitors (minutes) |
| `LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT` | `14` | Days before SSL expiry to trigger an alert |
| `LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS` | `15` | HTTP request timeout used by the check service (seconds) |
| `LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS` | `12` | How often to re-check SSL certificates |
| `LARAVEL_CRM_MONITORING_MAX_RESPONSE_BYTES` | `5242880` (5 MiB) | Cap on the response body a check will read, so a monitored endpoint streaming an unbounded response cannot exhaust the queue worker's memory |
| `LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS` | `false` | Allow monitors to target private/loopback IPs (off by default to prevent SSRF) |

> **Note:** `max_response_bytes` ships as 5 MiB in the config file, but `MonitorCheckService` falls back to **2 MiB** when the key is absent. An install with a published `config/laravel-crm.php` written before 2.4.0 therefore gets 2 MiB until the key is added. Add it explicitly if the value matters to you.

Four further monitoring keys are **read by the code but absent from the shipped config file**. They fall back to the defaults below; add them to your published config to change them.

| Key | Default | Description |
|---|---|---|
| `monitoring.perf_alert_rate_limit_minutes` | `60` | Minimum gap between slow-response alerts for one monitor |
| `monitoring.recovered_alert_rate_limit_minutes` | `60` | Minimum gap between recovery alerts for one monitor |
| `monitoring.down_debounce_minutes` | `2` | How long an endpoint must stay down before a downtime alert fires |
| `monitoring.ssl_alert_rate_limit_hours` | `24` | Minimum gap between SSL expiry alerts for one monitor |

See [Monitoring → Alert Rate Limiting](/monitoring#alert-rate-limiting).

## Portal

Settings for the public-facing [Portal](/portal) (feature board, signed quote/invoice/purchase-order links).

```php
'portal' => [
    'team_id' => env('LARAVEL_CRM_PORTAL_TEAM_ID'),
    'allow_registration' => env('LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_PORTAL_TEAM_ID` | `null` | Pin the portal to a single team. **Optional** — leave unset to give every team its own board |
| `LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION` | `false` | Allow visitors to self-register on the portal so they can vote and comment on features |

> **Note:** When registration is enabled, `/p/register` writes rows to the host application's `users` table and dispatches Laravel's `Registered` event for each signup.

> **Important:** `portal.team_id` is no longer required. Under multi-tenant teams mode every team has its own board at `/p/features/team/{id}`, and bare `/p/features` resolves the board from the URL, the session, the signed-in user's current team, or — where only one team has a board — that team. Setting it is a hard single-tenant lock that 404s every feature outside that team. It is ignored when teams mode is off. See [Portal → Portal Teams](/portal#portal-teams).

## API

Settings for the [REST API](/api). These two throttle `POST /crm/api/v2/auth/token` per email address, on top of the IP-keyed `throttle:6,1` limiter the route already carries.

```php
'api' => [
    'token_attempts_per_account' => env('LARAVEL_CRM_API_TOKEN_ATTEMPTS_PER_ACCOUNT', 5),
    'token_attempts_decay_seconds' => env('LARAVEL_CRM_API_TOKEN_ATTEMPTS_DECAY_SECONDS', 600),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_API_TOKEN_ATTEMPTS_PER_ACCOUNT` | `5` | Failed token requests allowed per email address per window |
| `LARAVEL_CRM_API_TOKEN_ATTEMPTS_DECAY_SECONDS` | `600` | Length of that window, in seconds |

The counter is incremented only on a failed attempt and cleared on success. See [API → Rate limits](/api#rate-limits).

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

When this is off, the system-check banner renders nothing and the **Updates** sidebar item is hidden. See [Updates](/updates).

## Documentation URLs

Where the CRM's own in-app links point. Override either if you host your own documentation.

```php
'docs_url' => env('LARAVEL_CRM_DOCS_URL', 'https://github.com/venturedrake/laravel-crm'),

'upgrade_guide_url' => env('LARAVEL_CRM_UPGRADE_GUIDE_URL', 'https://laravelcrm.com/docs/2.x/upgrading'),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_DOCS_URL` | `https://github.com/venturedrake/laravel-crm` | Target of the "View version X details" link — release notes |
| `LARAVEL_CRM_UPGRADE_GUIDE_URL` | `https://laravelcrm.com/docs/2.x/upgrading` | Target of every "Upgrade guide" link — the updates page and the system check banner |

The two are separate because they answer different questions: `docs_url` answers *what is in this release?*, `upgrade_guide_url` answers *how do I install it?*.

## Localisation

The package ships four locale directories under `resources/lang`:

| Locale | Contents |
|---|---|
| `en` | The complete key set — every other locale falls back to this one |
| `en_au` | A small regional override file (≈30 keys): Australian spellings and terms such as *ABN Number* and *postcode* |
| `en_gb` | The same, for British spellings |
| `fa` | Persian — a full translation, new in 2.4.0 |

> **Note:** `en_au` and `en_gb` are deliberately partial. They override only the keys whose wording differs regionally and fall through to `en` for everything else, so they need no maintenance when new keys are added.
>
> `fa` is a full translation, but it is currently missing the ~62 keys added during the 2.4.0 cycle — the invitation emails, the PDF template picker, the system-check banner and the decimal-quantity validation messages. Those strings render in English for a Persian-locale user until the translation catches up.

## Models with Global

In multi-tenant mode, certain model tables can have rows marked as global (shared across all teams) using a `global` column.

```php
'model_with_global' => [
    'settings',
],
```

By default, only `settings` are global. Add other model names to share their data across teams.
