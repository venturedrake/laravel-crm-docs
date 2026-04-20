# Upgrade Guide

[[toc]]

## Upgrading from 1.x to 2.x

Version 2.x is a major rewrite of the user interface and frontend stack. The backend API and model layer remain largely compatible.

### Breaking Changes

- **UI stack**: Bootstrap 4 + jQuery replaced with Tailwind CSS v4 + DaisyUI v5 + MaryUI
- **Livewire**: Upgraded from Livewire 2 to Livewire 3. All Livewire components have been rewritten.
- **PHP requirement**: Minimum PHP version is now 8.1 (was 7.3)
- **Laravel requirement**: Minimum Laravel version is now 10 (was 6)
- **Views**: All Blade views have been rewritten. If you published and customized views, you will need to re-apply customizations to the new templates.

### New Modules

- **Deliveries** — Track physical shipments from orders
- **Purchase Orders** — Manage supplier purchase orders

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

### Step 3. Update Permissions

```bash
php artisan laravelcrm:permissions
```

### Step 4. Clear Caches

```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

## Upgrading Within 2.x

Follow these steps when upgrading between 2.x releases.

### Step 1. Update Package & Publish Assets

```bash
composer require venturedrake/laravel-crm
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
```

### Step 2. Run the Database Seeder

```bash
php artisan db:seed --class="VentureDrake\LaravelCrm\Database\Seeders\LaravelCrmTablesSeeder"
```

### Step 3. Update Permissions

```bash
php artisan laravelcrm:permissions
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
