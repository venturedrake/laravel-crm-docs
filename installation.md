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
6. Prompt you to create an owner user (or grant access to an existing user)

### Step 3. Access the CRM

Navigate to `http://<yoursite>/crm` (or whatever you set `LARAVEL_CRM_ROUTE_PREFIX` to). Log in with the owner credentials you created during installation.

## Additional Setup Commands

After installation, you can run these optional commands from your host application:

```bash
# Add another user with CRM access
php artisan laravelcrm:add-user

# Generate sample data for development
php artisan laravelcrm:sample-data
```

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
