# Pipelines

[[toc]]

## Overview

Pipelines define the workflow that [Leads](/leads), [Deals](/deals), and [Quotes](/quotes) move through. Each pipeline contains an ordered set of **pipeline stages** that drive the Kanban board view. A pipeline is bound to a specific model type (lead, deal, or quote) so different entities can have independent workflows.

**Models:**
- `VentureDrake\LaravelCrm\Models\Pipeline`
- `VentureDrake\LaravelCrm\Models\PipelineStage`
- `VentureDrake\LaravelCrm\Models\PipelineStageProbability`

**Tables:**
- `{prefix}pipelines` (default: `crm_pipelines`)
- `{prefix}pipeline_stages` (default: `crm_pipeline_stages`)
- `{prefix}pipeline_stage_probabilities` (default: `crm_pipeline_stage_probabilities`)

## Pipeline Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs |
| `name` | `string` | Pipeline display name |
| `model` | `string` | The entity class this pipeline applies to (e.g. `Lead`, `Deal`, `Quote`) |
| `team_id` | `integer` | Team scope when teams are enabled |

## Pipeline Stage Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs |
| `name` | `string` | Stage name (e.g. `Qualified`, `Proposal Sent`) |
| `description` | `text` | Optional stage description |
| `pipeline_id` | `integer` | Parent pipeline |
| `pipeline_stage_probability_id` | `integer` | Optional win-probability bucket |
| `order` | `integer` | Sort order within the pipeline |
| `color` | `string` | Hex colour used on the Kanban board |

## Relationships

### Pipeline

| Method | Type | Description |
|---|---|---|
| `pipelineStages()` | `hasMany` | Stages belonging to the pipeline |
| `leads()` | `hasMany` | Leads using this pipeline |
| `deals()` | `hasMany` | Deals using this pipeline |
| `quotes()` | `hasMany` | Quotes using this pipeline |

### Pipeline Stage

| Method | Type | Description |
|---|---|---|
| `pipeline()` | `belongsTo` | Parent pipeline |
| `pipelineStageProbability()` | `belongsTo` | Optional probability bucket |
| `leads()` | `hasMany` | Leads currently in this stage |
| `deals()` | `hasMany` | Deals currently in this stage |
| `quotes()` | `hasMany` | Quotes currently in this stage |

## Managing Pipelines

Pipelines and stages are managed through **Settings → Pipelines** and **Settings → Pipeline Stages** in the CRM UI. Stages can be reordered via drag-and-drop and assigned colours that show on the Kanban board.

## Board View

[Leads](/leads), [Deals](/deals), and [Quotes](/quotes) all support a Kanban-style board view that groups records by `pipeline_stage_id`. Cards are dragged between columns using SortableJS, and stage transitions persist through the underlying entity's update path.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

