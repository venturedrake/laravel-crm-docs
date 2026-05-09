# Security

[[toc]]

## Overview

Laravel CRM is designed with security and data privacy best practices. Several features provide layers of protection for sensitive data.

## Authentication

The CRM uses your application's existing authentication system. All CRM routes are protected by the `auth.laravel-crm` middleware. Users must have the `crm_access` attribute set to `true` on their user model.

## Authorization

Every CRM entity has a corresponding Laravel Policy in `src/Policies/`. Permissions are managed via the [Spatie Permission](https://spatie.be/docs/laravel-permission) package with CRM-specific roles and permissions.

See [Roles](/roles) and [Permissions](/permissions) for details.

## Field Encryption

Sensitive personal data (names, emails, phone numbers) can be encrypted at rest in the database. Enable via:

```env
LARAVEL_CRM_ENCRYPT_DB_FIELDS=true
```

Then run the encryption command:

```bash
php artisan laravelcrm:encrypt
```

Encrypted fields are declared in each model's `$encryptable` array and handled transparently by the `LaravelEncryptableTrait`.


## Reporting Vulnerabilities

If you discover a security vulnerability, please email [andrew@laravelcrm.com](mailto:andrew@laravelcrm.com). All security vulnerabilities will be promptly addressed.

