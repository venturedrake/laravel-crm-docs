# Upgrade Guide

[[toc]]

## Upgrading from >= 0.2

Follow these steps when upgrading from version 0.2 or later.

### Step 1. Update Package & Publish Assets

```bash
composer require venturedrake/laravel-crm
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="config"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
```

### Step 2. Run the Database Seeder

```bash
php artisan db:seed --class="VentureDrake\LaravelCrm\Database\Seeders\LaravelCrmTablesSeeder"
```

### Step 3. Update Permissions (Teams Only)

If you are using teams support, run the permissions update command:

```bash
php artisan laravelcrm:permissions
```

### Step 4. Run the Update Command

```bash
php artisan laravelcrm:update
```

This command applies any necessary database updates for the new version.

## Upgrading from < 0.2

If you are upgrading from a version before 0.2, there are additional steps required due to breaking changes.

### Step 1. Update Package & Publish Assets

```bash
composer require venturedrake/laravel-crm
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="config"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
```

### Step 2. Delete Previously Published Views

Remove the old package views that were published to your application:

```bash
rm -rf resources/views/vendor/laravel-crm
```

### Step 3. Update User Model

Add the `HasCrmAccess`, `HasCrmTeams` and `HasRoles` traits to your `App\User` model:

```php
use Spatie\Permission\Traits\HasRoles;
use VentureDrake\LaravelCrm\Traits\HasCrmAccess;
use VentureDrake\LaravelCrm\Traits\HasCrmTeams;

class User extends Authenticatable
{
    use HasRoles;
    use HasCrmAccess;
    use HasCrmTeams;

    // ...
}
```

See the [Installation](/installation) guide for full details on user model setup.

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
