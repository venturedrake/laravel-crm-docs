# Xero

[[toc]]

## Overview

Laravel CRM integrates with [Xero](https://www.xero.com/) accounting software to synchronise contacts, products, invoices, and purchase orders between the two systems. The integration is powered by the [`dcblogdev/laravel-xero`](https://github.com/dcblogdev/laravel-xero) package.

## Requirements

- A Xero account with API access.
- The `dcblogdev/laravel-xero` package, installed by your application (see below).
- Xero OAuth2 credentials (Client ID and Client Secret) from the [Xero Developer Portal](https://developer.xero.com/).

Since 2.4.5, Laravel CRM only *suggests* `dcblogdev/laravel-xero`. It does not require it. Install it yourself:

```bash
composer require dcblogdev/laravel-xero:1.1.3
```

> **Important:** No published version of `dcblogdev/laravel-xero` allows `guzzlehttp/guzzle` 8 (1.1.3 requires `^6|^7`, later releases `^7.9.3`), so installing it **pins your app to Guzzle 7**. A stock Laravel 13.32+ app resolves Guzzle 8, so you have to choose between Xero sync and Guzzle 8 until upstream ships a Guzzle 8 release. Upgrading an existing install? Add the package in the same deploy as the upgrade — see [Upgrading to 2.4.5](/upgrading#upgrading-to-245).

### When the package is not installed

Without `dcblogdev/laravel-xero`, the CRM runs normally and the Xero integration is inert:

- **Settings → Integrations → Xero** shows an alert naming the missing package in place of the **Connect** button.
- The connect and disconnect routes stay registered, so links to them still resolve, but they redirect back to the settings screen.
- `php artisan laravelcrm:xero` exits non-zero and prints the install command.
- Product, invoice and purchase-order sync is a no-op, and the `XeroToken` observer and `XeroTenant` middleware are not registered.
- Stored tokens and `crm_xero_*` rows are kept. Installing the package restores the connection without re-authorising, as long as the stored refresh token has not expired while the package was gone (nothing runs `xero:keep-alive` in that time).

There is no banner, so the settings screen is the only place this shows in the UI. The exception is the scheduler: if you still have a published `config/xero.php` with Xero credentials set, the [scheduled syncs](#scheduled-syncs) are still registered and fail on every run. If you are dropping Xero, remove `XERO_CLIENT_ID` and `XERO_CLIENT_SECRET` from your `.env`.

Every entry point checks `VentureDrake\LaravelCrm\Support\XeroIntegration` first. Use `XeroIntegration::installed()` to check that the package is present, and `XeroIntegration::connected()` to check that it is present *and* connected. Use them in your own code in place of `\Dcblogdev\Xero\Facades\Xero::isConnected()`, which is a fatal `Class not found` when the package is absent.

## Configuration

### 1. Set Environment Variables

Add your Xero OAuth2 credentials to your `.env` file:

```env
XERO_CLIENT_ID=your-client-id
XERO_CLIENT_SECRET=your-client-secret
XERO_REDIRECT_URI=https://yoursite.com/xero/callback
```

### 2. Publish Xero Config

Publish the Xero package configuration if you haven't already:

```bash
php artisan vendor:publish --provider="Dcblogdev\Xero\XeroServiceProvider"
```

## Connecting to Xero

Navigate to **Settings → Integrations → Xero** in the CRM interface. Click the **Connect** button to initiate the OAuth2 flow. You will be redirected to Xero to authorize the connection. Once authorized, you will be redirected back to the CRM with an active connection.

The connected Xero tenant name is displayed on the integration page once authenticated.

To disconnect, click the **Disconnect** button on the same page.

## Sync Settings

Once connected, the Xero integration page provides toggle switches to enable or disable syncing for each data type:

| Setting | Description |
|---|---|
| **Contacts** | Sync Xero contacts as CRM organisations and people |
| **Products** | Sync Xero items as CRM products |
| **Quotes** | Sync quotes (planned) |
| **Invoices** | Sync invoices (planned) |

Toggle the settings on or off and click **Update** to save.

## Syncing Data

Data synchronisation is performed via the Artisan command:

```bash
php artisan laravelcrm:xero {model}
```

Where `{model}` is one of: `contacts`, `products`, `quotes`, or `invoices`.

### Contacts Sync

```bash
php artisan laravelcrm:xero contacts
```

When contacts sync is enabled, this command:

- Fetches all contacts from Xero.
- Creates or updates a CRM **Organisation** for each Xero contact, matched by Xero Contact ID.
- If the Xero contact has a first name or last name, creates or updates a CRM **Person** linked to the organisation.
- Syncs the contact's email address as the person's primary work email.
- Stores the Xero Contact ID mapping in the `xero_contacts` and `xero_people` tables for future syncs.

The organisation owner is set to the user configured as `crm_owner` in the `laravel-crm` config.

### Products Sync

```bash
php artisan laravelcrm:xero products
```

When products sync is enabled, this command:

- Fetches all items from Xero.
- Creates or updates a CRM **Product** for each Xero item, matched by Xero Item ID.
- Syncs product code, name, description, purchase/sales account codes, tax rate, and unit price.
- **Two-way sync:** CRM products that don't exist in Xero (but have a product code) are automatically pushed to Xero.
- Stores the Xero Item ID mapping in the `xero_items` table.

The unit price pushed to Xero is the product's own price. Before 2.5.0 the push read the first matching row in `crm_product_prices`, and that table also holds [variation](/product-variations) prices, so a variation's price could be sent as the product's.

## Line Items

When an invoice or purchase order is pushed to Xero, each line is built by `VentureDrake\LaravelCrm\Support\XeroLineItem`:

- **`Description`** is the line's `displayName()`: the product name, plus the [variation](/product-variations) name when the line has one (`Widget — Large`).
- **`ItemCode`** is the variation's SKU only when an item with that code has already been synced from Xero. Otherwise it is the parent product's Xero item code, or failing that its product code, as before 2.5.0.

Variations are never created as Xero items. If you want a variation to post against its own Xero item, create that item in Xero with the variation's SKU as its code and let the [products sync](#products-sync) bring it across. Invoices pulled in from Xero are stored without a variation.

## Scheduled Syncs

The following syncs are auto-registered in the service provider's scheduler:

- `xero:keep-alive` — every 5 minutes (keeps OAuth token active)
- `laravelcrm:xero contacts` — every 10 minutes
- `laravelcrm:xero products` — every 10 minutes

These are registered whenever Xero credentials are configured (`xero.clientId` and `xero.clientSecret`). They do **not** check that `dcblogdev/laravel-xero` is installed. Without the package but with credentials still configured, `xero:keep-alive` fails as an undefined command every 5 minutes, and `laravelcrm:xero` exits non-zero with the install command every 10 minutes. Remove the credentials if you are not using Xero — see [When the package is not installed](#when-the-package-is-not-installed).

## Data Model

The integration uses intermediate models to track the mapping between CRM records and Xero records:

| CRM Model | Xero Model | Xero ID Field | Description |
|---|---|---|---|
| `Organization` | `XeroContact` | `contact_id` | Maps organisations to Xero contacts |
| `Person` | `XeroPerson` | `contact_id` | Maps people to Xero contacts |
| `Product` | `XeroItem` | `item_id` | Maps products to Xero items |
| `Invoice` | `XeroInvoice` | `invoice_id` | Maps invoices to Xero invoices |
| `PurchaseOrder` | `XeroPurchaseOrder` | `purchase_order_id` | Maps purchase orders to Xero purchase orders |

Each Xero model is related to its CRM counterpart via a `morphOne` or `belongsTo` relationship, and is assigned a UUID `external_id` on creation.
