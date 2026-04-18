# Organisations
[[toc]]
## Overview
Organisations represent companies or business entities. An organisation can have multiple [People](/reference/people) associated with it, and can be linked to [Leads](/reference/leads), [Deals](/reference/deals), [Orders](/reference/orders), and other entities. The organisation name supports optional encryption for privacy.
**Model:** `VentureDrake\LaravelCrm\Models\Organisation`
**Table:** `{prefix}organisations` (default: `crm_organisations`)
## Attributes
| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Organisation name — encryptable |
| `description` | `text` | Notes |
| `annual_revenue` | `integer` | Annual revenue (stored in cents) |
| `total_money_raised` | `integer` | Total funding raised (stored in cents) |
| `number_of_employees` | `integer` | Employee count |
| `organisation_type_id` | `integer` | Organisation type |
| `timezone_id` | `integer` | Timezone |
| `user_owner_id` | `integer` | Owner user |
> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are sto> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are sto> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are sto> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are sto> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are sto> **Note:** Money fields (`annu| `phones()` | `morphMany` | `Phone` | Phone numbers |
| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `ad| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `contacts()` | `morphMany` | `Contact` | Contact records |
| `client()` | `morphOne` | `Client` | Client record |
| `organisationType()` | `belongsTo` | `OrganisationType` | Type classification |
| `timezone()` | `belongsTo` | `Timezone` | Timezone |
| `ownerUser()` | `belongsTo` | `User` | Owner |
## Hel## Hel## Hel## Hel## Hel## Hel## Hel## Hel## Hel## Hel##---|
| `getPrimaryEmail()` | `Email\|null` | Primary email address |
| `getPrimaryPhone()` | `Phone\|null` | Primary phone number |
| `getPrimaryAddress()` | `Address\|null` | Primary address |
| `getBillingAddress()` | `Address\|null` | Billing address (type 5) |
| `getShippingAddress()` | `Address\|null` | Shipping address (type 6) |
## Creating an Organisation
```php
use VentureDrake\LaravelCrm\Models\Organisation;
$org = Organisation::create([
    'name' => 'Acme Corp',
    'description' => 'Software company',
    'annual_revenue'    'annual_revenue'    'annual_revenue'    'annual_revenue'    'annual_revenue'    'annual_revenue'    'annual_re]);
```
## Searching & Filtering
Searchable by `name`. Filterable bSearchable by `name`. Filterable bSearchableby `id`, `nSearchable by `name`. Filterable bSearchable by `name`. Filterable bSear|---|---|
| `SoftDeletes` | Soft delete support |
| `LaravelEncryptableTrait` | Field-level encryption |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `Sortable` | Column sorting |
| `HasCrmActivities` | Activity timeline tracking |
| `HasCrmUserRelations` | Standard user audit relations |
