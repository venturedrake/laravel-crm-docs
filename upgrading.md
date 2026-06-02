# Upgrade Guide

[[toc]]

## Upgrading from 2.2.x to 2.3.0

Version 2.3.0 introduces two new optional modules — a public **Features** voting board and an **uptime / SSL Monitoring** module — plus a redesigned public portal and quality-of-life improvements to file uploads and the installer. There are no schema-breaking changes; follow the standard [Upgrading Within 2.x](#upgrading-within-2-x) steps.

### What's new

- **Features** — public roadmap board with voting, comments, status tracking, view analytics, and email notifications. See [Features](/features).
- **Monitoring** — uptime and SSL monitoring for HTTP/HTTPS endpoints, with response-time charts, sparklines, SSL expiry alerts, and SSRF protection. See [Monitoring](/monitoring).
- **Portal redesign** — public quote, invoice, purchase-order, and feature pages rebuilt on Tailwind v4 + DaisyUI v5 + MaryUI with a top navbar, theme toggle, and toast notifications. See [Portal](/portal).
- **File-upload improvements** — drag-and-drop dropzone, upload progress bar, deferred upload, and per-component max-size / allowed-types validation.
- **Installer module selection** — `laravelcrm:install` now prompts which modules to enable, or accepts `--modules=all` / `--modules=leads,deals,...` for non-interactive installs.

### Enabling the new modules

The new modules are added to the `modules` array in the published config file. When you run `laravelcrm:update` the migrations for `crm_features*`, `crm_monitors`, and `crm_monitor_checks` will be applied.

To enable them, ensure the following entries exist in `config/laravel-crm.php`:

```php
'modules' => [
    // ... existing modules
    'features',
    'monitoring',
],
```

Optional environment variables (all have sensible defaults):

```env
# Features
LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES=60
LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION=false

# Monitoring
LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES=5
LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT=14
LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS=15
LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS=12
LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS=false
```

If you use the Monitoring module, make sure Laravel's scheduler is running (`* * * * * php artisan schedule:run`) so monitor checks fire on their configured intervals.

### Breaking changes

None. PHP 8.2+ and Laravel 11+ requirements introduced in 2.2.0 are unchanged.

## Upgrading from 2.1.x to 2.2.0

Version 2.2.0 introduces a JSON REST API and adds page titles throughout the UI. There are no schema-breaking changes — follow the standard [Upgrading Within 2.x](#upgrading-within-2-x) steps.

### What's new

- **REST API** — Sanctum-authenticated JSON API at `/crm/api/v2` exposing 8 resourceful entities (`leads`, `products`, `organizations`, `people`, `deals`, `quotes`, `orders`, `invoices`) plus auth endpoints. See [API](/api).
- **Page titles** — Every CRM page now sets a descriptive `<title>` tag for better browser tabs, history, and SEO.

### Breaking changes

- **PHP requirement** — Minimum PHP version is now **8.2** (was 8.1).
- **Laravel requirement** — Minimum Laravel version is now **11** (was 10).

If you're on PHP 8.1 or Laravel 10, upgrade those first before pulling 2.2.0.

### Optional: enable the REST API

If you want to use the new API, install Sanctum in the host application — see [API → Installation](/api).

## Upgrading from 1.x to 2.x

Version 2.x is a major rewrite of the user interface and frontend stack. The backend API and model layer remain largely compatible.

### Breaking Changes

- **UI stack**: Bootstrap 4 + jQuery replaced with Tailwind CSS v4 + DaisyUI v5 + MaryUI
- **Livewire**: Upgraded from Livewire 2 to Livewire 3 or 4. All Livewire components have been rewritten.
- **PHP requirement**: Minimum PHP version is now 8.1 (was 7.3)
- **Laravel requirement**: Minimum Laravel version is now 10 (was 6)
- **Views**: All Blade views have been rewritten. If you published and customized views, you will need to re-apply customizations to the new templates.

### New Modules

- **Chat** — Live chat with embeddable visitor widget — see [Chat](/chat)
- **Email Marketing** — Campaign and template management with open/click tracking — see [Email Marketing](/email-marketing)
- **SMS Marketing** — Campaign and template management via ClickSend — see [SMS Marketing](/sms-marketing)

### Step 1. Update Package

```bash
composer require venturedrake/laravel-crm
```

### Step 2. Publish & Migrate

```bash
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="config"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
```

### Step 3. Run the v2 Migration Helper

The package ships a one-shot command that backfills new 2.x columns, normalises existing data, and seeds new lookup tables required by the rewritten UI:

```bash
php artisan laravelcrm:v2
```

Run this **once** after migrating from a 1.x installation.

### Step 4. Update Permissions & Custom Fields

```bash
php artisan laravelcrm:permissions
php artisan laravelcrm:fields
```

### Step 5. Clear Caches

```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

## Upgrading Within 2.x

Follow these steps when upgrading between 2.x releases.

### Step 1. Update the Package

```bash
composer require venturedrake/laravel-crm
```

### Step 2. Run the Update Routine

The `laravelcrm:update` command re-publishes config/migrations/assets, runs new migrations, and re-seeds permissions, labels, and custom fields in one step:

```bash
php artisan laravelcrm:update
```

If you prefer to run each step manually, the equivalent commands are:

```bash
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
php artisan laravelcrm:permissions
php artisan laravelcrm:labels
php artisan laravelcrm:fields
```

## General Upgrade Tips

- **Back up your database** before upgrading to any new version.
- **Review the [changelog](https://github.com/venturedrake/laravel-crm/blob/master/CHANGELOG.md)** for breaking changes and new features.
- **Clear caches** after upgrading:

```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

- **Run tests** to ensure your customisations still work with the updated package.
