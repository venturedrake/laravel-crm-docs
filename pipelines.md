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

Pipelines and stages are managed through **Settings → Pipelines** and **Settings → Pipeline Stages** in the CRM UI. Stages are assigned colours that show on the Kanban board.

- **Creating a stage.** The stages index has a **Create pipeline stage** button, which opens `/crm/pipeline-stages/create` ([#114](https://github.com/venturedrake/laravel-crm/issues/114)). The form defaults to the first pipeline, and a new stage goes after the pipeline's last one ([#138](https://github.com/venturedrake/laravel-crm/issues/138)).

> **Removed in 2.7.0:** the v1 pipeline and pipeline stage create/store routes ([#107](https://github.com/venturedrake/laravel-crm/issues/107)), which rendered views v2 never shipped and returned a 500. `/crm/pipelines/create` is a 404, and the `StorePipelineRequest` and `StorePipelineStageRequest` classes are gone.

### Reorder stages

Since 2.7.0 stages can be reordered, saving through `PipelineStageService` ([#123](https://github.com/venturedrake/laravel-crm/issues/123)).

Open **Settings → Pipelines → [pipeline]**. In the **Stages** list, drag a stage by its handle or use its **Move up** and **Move down** buttons (also available with Tab and Enter/Space). You can also open a stage under **Settings → Pipeline Stages** and choose **Reorder stages** in its pipeline card.

Each move saves automatically and shows a confirmation. Wait for it to finish before making another move. Refreshing the page preserves the saved order; the lead, deal or quote board uses that same column order when loaded. Reordering preserves stage identities, colours, probabilities and the records assigned to them. Editing controls require permission to edit the pipeline and its stages.

New stages append to the end. For example, after creating Negotiation, move it before Closed Won and Closed Lost. Moving a stage to another pipeline also appends it to that pipeline; changing only its name or description keeps its position.

If another administrator changes the order or adds, removes or moves a stage while your page is open, your outdated move saves nothing. The current list reloads with an explanation so you can try again. If a request fails, reload to check the saved order before retrying.

A pipeline supports up to 127 consecutive stage positions. If older positions have gaps and reach that limit, reorder the existing stages to close the gaps before adding another.

## Board View

[Leads](/leads), [Deals](/deals), and [Quotes](/quotes) all support a Kanban-style board view that groups records by `pipeline_stage_id`. Cards are dragged between columns using SortableJS, and stage transitions persist through the underlying entity's update path. A card can only be moved to a stage of the board's own pipeline.

Since 2.7.0 the boards are paged. Each column renders 20 cards with a **Load more** button below them; set `LARAVEL_CRM_BOARD_PER_COLUMN` to change the page size (see [Configuration → Kanban Boards](/configuration#kanban-boards)). Each column header shows the column's full count and the total value of every record in it, one line per currency. Before, a board rendered every record, and one with a few thousand records took up to 25 seconds and 49 MB of HTML. Board search finds contacts by name with field encryption on.

`LeadBoard`, `DealBoard` and `QuoteBoard` extend the `PipelineBoard` base class, which pages its own queries. A subclass narrows the cards in `boardScope()`, calling the parent first, and qualifies columns with `$this->column('title')`, because a search joins the people and organizations tables. Boards that extend `KanbanBoard` directly are not paged. On the quote board, a card's Send item opens a Send drawer shared by the whole page. See the [Upgrade Guide](/upgrading#boards-load-one-page-per-column) if you extend a board.

Each card carries a `data-record-id` attribute, and the drop handler collects **only** those. Every card is followed in the DOM by its own delete-confirm `<dialog>`, so before 2.4.1 the handler posted arrays like `['3', 'modalDeleteLead3', '7']` — the server then tried to update a record with a non-numeric id and 500'd, and the interleaved dialogs inflated `pipeline_stage_order` to 1, 3, 5 rather than 1, 2, 3.

Server-side, the batch is resolved, filtered and renumbered in a transaction, and **every** record in it is authorized. Previously only the first resolvable record was checked, so a record the user had no right to edit was reordered as long as an editable one led the list. See [Security](/security#kanban-reordering).

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

