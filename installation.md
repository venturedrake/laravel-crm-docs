# Installation

[[toc]]

## Requirements

- PHP ^8.2
- MySQL 5.7+ / MariaDB 10.2.7+
- Laravel 11, 12, or 13
- Livewire 3 or 4

## Install Laravel CRM

### Step 1. Require the Package

```bash
composer require venturedrake/laravel-crm
```

### Step 2. Run the Installer

The installer publishes config, migrations, and assets, runs migrations, seeds the database, and creates your initial owner user:

```bash
php artisan laravelcrm:install
```

The installer will:

1. Publish the configuration file to `config/laravel-crm.php`
2. Publish database migrations
3. Publish frontend assets to `public/vendor/laravel-crm/`
4. Run migrations (creates all `crm_`-prefixed tables)
5. Seed default data (roles, permissions, pipeline stages, settings)
6. Patch your `User` model with the `HasCrmAccess` and `HasCrmTeams` traits (it prompts first, and skips if they are already there) — see [Users → Setup](/users#setup)
7. Add `@php artisan laravelcrm:upgrade --ansi` to your `composer.json` `post-autoload-dump` scripts — see [The composer hook](#the-composer-hook)
8. **Prompt you to choose which optional modules to enable** — the 13 entries listed under [Optional Modules](/configuration#optional-modules): leads, deals, quotes, orders, invoices, deliveries, purchase-orders, teams, chat, email-marketing, sms-marketing, features, monitoring
9. Prompt you to create an owner user (or grant access to an existing user)

For non-interactive installs you can pre-select modules:

```bash
# Enable everything
php artisan laravelcrm:install --modules=all

# Enable a specific subset
php artisan laravelcrm:install --modules=leads,deals,quotes,invoices
```

### Step 3. Access the CRM

Navigate to `http://<yoursite>/crm` (or whatever you set `LARAVEL_CRM_ROUTE_PREFIX` to). Log in with the owner credentials you created during installation.

## The composer hook

The installer writes this into your application's `composer.json`:

```json
"scripts": {
    "post-autoload-dump": [
        "@php artisan package:discover --ansi",
        "@php artisan laravelcrm:upgrade --ansi"
    ]
}
```

From then on, every `composer install` and `composer update` republishes CRM assets and clears cached config, routes and views on its own — including a production `composer install --no-dev`. `laravelcrm:upgrade` never opens a database connection and never prompts, so it is safe to run unattended mid-build.

The line must come **after** `package:discover`; that is what makes the package's artisan commands resolvable. If the installer could not edit the file it prints the line for you to add by hand. See [Updates](/updates).

## Additional Setup Commands

After installation, you can run these optional commands from your host application:

```bash
# Republish assets and clear caches — no database access, safe unattended
php artisan laravelcrm:upgrade

# Apply migrations, seed data and run backfills after a package update
php artisan laravelcrm:update

# Add another user with CRM access
php artisan laravelcrm:add-user

# Generate sample data for development
php artisan laravelcrm:sample-data
```

See the [Upgrade Guide](/upgrading#what-each-command-does) for what `laravelcrm:upgrade` and `laravelcrm:update` each do and where they belong in a deploy.

## Localisation

The package ships `en`, plus `en_au` and `en_gb` regional override files and a Persian (`fa`) translation. See [Configuration → Localisation](/configuration#localisation) for what each covers and where `fa` currently falls back to English.

## Route Prefix

By default, the CRM is accessible at `/crm`. To change this, set the `LARAVEL_CRM_ROUTE_PREFIX` environment variable or update `config/laravel-crm.php`.

If you set the route prefix to blank (serving from root), update your `routes/web.php` to avoid conflicts with the CRM's routes:

```php
Route::middleware(['auth'])->get('/dashboard', function () {
    return redirect('/');
})->name('dashboard');
```

## Field Encryption

To encrypt sensitive database fields (names, emails, phones) after installation:

```bash
php artisan laravelcrm:encrypt
```

To decrypt them later:

```bash
php artisan laravelcrm:decrypt
```

Enable encryption in your `.env`:

```env
LARAVEL_CRM_ENCRYPT_DB_FIELDS=true
```
