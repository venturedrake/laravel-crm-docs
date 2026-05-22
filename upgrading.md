# Upgrade Guide

[[toc]]

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
