# Lead Sources

[[toc]]

## Overview

Lead sources track where a [Lead](/leads) originated from (e.g. `Website`, `Referral`, `Cold Call`, `Trade Show`). Sources are managed in the CRM UI and assigned to leads via the `lead_source_id` foreign key, enabling reporting on which channels are producing the most pipeline.

**Model:** `VentureDrake\LaravelCrm\Models\LeadSource`
**Table:** `{prefix}lead_sources` (default: `crm_lead_sources`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Source name |
| `team_id` | `integer` | Team scope when teams are enabled |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `leads()` | `hasMany` | `Lead` | Leads attributed to this source |

## Managing Lead Sources

Lead sources are managed through **Settings → Lead Sources** in the CRM UI.

## Seeding Default Lead Sources

```bash
php artisan laravelcrm:lead-sources
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

