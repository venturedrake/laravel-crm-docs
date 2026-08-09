# Xero

[[toc]]

## Overview

Laravel CRM integrates with [Xero](https://www.xero.com/) accounting software to synchronise contacts, products, invoices, and purchase orders between the two systems. The integration is powered by the [`dcblogdev/laravel-xero`](https://github.com/dcblogdev/laravel-xero) package.

## Requirements

- A Xero account with API access.
- The `dcblogdev/laravel-xero` package (included as a dependency).
- Xero OAuth2 credentials (Client ID and Client Secret) from the [Xero Developer Portal](https://developer.xero.com/).

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

## Scheduled Syncs

The following syncs are auto-registered in the service provider's scheduler:

- `xero:keep-alive` — every 5 minutes (keeps OAuth token active)
- `laravelcrm:xero contacts` — every 10 minutes
- `laravelcrm:xero products` — every 10 minutes

These only run when Xero credentials are configured.

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
