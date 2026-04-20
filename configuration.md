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

> **Tip:** If you sell digital products or services, you can remove `deliveries` since it won't be relevant.

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
