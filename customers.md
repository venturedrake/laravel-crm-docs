# Customers

[[toc]]

## Overview

Customers are a v1 entity that 2.x keeps for data compatibility only. **There is no Customers screen in 2.x**, and nothing in the 2.x forms creates or picks a customer. Use [People](/people) and [Organisations](/organizations) instead.

The model and table stay because the [v1 → v2 upgrade](/upgrading) migrates existing customer data into them, and leads, deals, quotes and orders still carry a `customer_id` column.

**Model:** `VentureDrake\LaravelCrm\Models\Customer`
**Table:** `{prefix}customers` (default: `crm_customers`)

> **Note:** In v1 the model was `Client` and the table `crm_clients`. `laravelcrm:v2` renames the table to `crm_customers`, renames the `clientable_*` morph columns to `customerable_*` and `client_id` to `customer_id` on leads, deals, quotes and orders, and rewrites stored `Models\Client` class names to `Models\Customer`. There is no `Client` model in 2.x.

## Removed in 2.5.0

The leftover v1 `/crm/customers` pages were removed in 2.5.0, along with their `laravel-crm.customers.*` routes, `CustomerController`, the empty `customers/index` view and the unused `StoreClientRequest` / `UpdateClientRequest` classes. They never worked in v2: opening `/crm/customers` failed with `Call to undefined method Illuminate\Database\Eloquent\Builder::sortable()` ([#93](https://github.com/venturedrake/laravel-crm/issues/93)), and the other pages pointed at views that no longer exist.

Host code that calls `route('laravel-crm.customers.index')`, or any sibling, now throws `RouteNotFoundException`. The model, its observer and policy, and the `crm customers` permissions are unchanged. See [Upgrading to 2.5.0](/upgrading#customers-routes-removed).

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID (set by `CustomerObserver`) |
| `team_id` | `integer` | Team (multi-tenant installs) |
| `name` | `string` | Customer name. Nullable, and encrypted when field encryption is on |
| `customerable_type` | `string` | Polymorphic type of the underlying person or organisation. Nullable |
| `customerable_id` | `integer` | Polymorphic ID. Nullable |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

The `user_created_id`, `user_updated_id`, `user_deleted_id` and `user_restored_id` columns are filled by `CustomerObserver`.

## Computed Attributes

### name

Returns the customer's own `name` when it is set, and otherwise the name of the underlying person or organisation.

```php
$customer->name; // "Acme Corp"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `customerable()` | `morphTo` | `Person\|Organization` | The underlying person or organisation, if any |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `contacts()` | `morphMany` | `Contact` | Related contacts |

### From other models

| Model | Method | Works? |
|---|---|---|
| `Lead` | `customer()` — `belongsTo(Customer::class)` on `customer_id` | Yes |
| `Deal`, `Quote`, `Order` | `client()` — `belongsTo(Customer::class)` | **No.** The foreign key is inferred from the method name as `client_id`, a column that no longer exists, so it always returns `null`. Read `customer_id` and load the model with `Customer::find()` |
| `Person`, `Organization` | `client()` — `morphOne(Customer::class, 'clientable')` | **No.** It queries `clientable_type` / `clientable_id`, which the customers table does not have, so loading it throws a `QueryException`. Query `Customer::where('customerable_type', $person->getMorphClass())->where('customerable_id', $person->id)` instead |

These `client()` methods are v1 leftovers that were not renamed when the table and columns were.

## Usage

```php
use VentureDrake\LaravelCrm\Models\Customer;

// The customer a lead belongs to
$customer = $lead->customer;

// Customers for an organisation
$customers = Customer::where('customerable_type', $organization->getMorphClass())
    ->where('customerable_id', $organization->id)
    ->get();

// The underlying person or organisation
$customer->customerable;
```

## Permissions

`CustomerPolicy` maps onto the `create` / `view` / `edit` / `delete crm customers` permissions. With no Customers screen, they gate nothing in the 2.x UI. Under the stock roles Employee holds them and Manager does not — see [Roles](/roles).

## Searching & Filtering

Searchable by `name`. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `HasEncryptableFields` | Field-level encryption of `name` |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasCrmUserRelations` | Standard user tracking relations |
