# Overview

[[toc]]

## Introduction

Laravel CRM 2.x ships with a complete, ready-to-use web interface built with Tailwind CSS v4, DaisyUI v5, MaryUI, and Livewire 3 or 4. The UI provides full CRUD management for all CRM entities, a Kanban-style pipeline board, global search, activity tracking, and role-based navigation — all accessible at `/crm` in your application.

## Accessing the CRM

Once installed, navigate to `http://<yoursite>/crm`. If you are not authenticated, you will be redirected to your application's login page. After logging in, users with CRM access will see the dashboard.

## Layout Structure

The CRM interface uses a responsive layout with sidebar navigation:

- **Sidebar** — Contains the main navigation menu, organized into logical sections. Navigation items are permission-gated, so users only see what they have access to.
- **Header** — Theme toggle, the user menu, and — when [teams](/teams) are enabled — a **team switcher** dropdown listing the tenants the signed-in user belongs to, with a check against the current one and a **+ New enterprise** item.
- **Content Area** — The main content area where pages are rendered. Most pages follow a consistent card-based layout. A [system-check banner](/updates#system-check-banner) sits above it when the install needs attention.

## Navigation

The sidebar navigation is organized into the following sections:

### Sales Pipeline
- **Dashboard** — Overview with summary cards showing totals for leads, deals, people, and organisations.
- **Leads** — Manage unqualified prospects (list and board views).
- **Deals** — Track qualified opportunities through pipeline stages (list and board views).
- **Quotes** — Create and manage quotes linked to deals (list and board views).

### Activity
- **Activity** — Unified view of all activities including tasks, notes, calls, meetings, lunches, and files.

### Marketing
- **Email Campaigns** — Create, schedule, and send email marketing campaigns to subscribed contacts.
- **Email Templates** — Reusable email templates with TinyMCE rich-text editing.
- **SMS Campaigns** — Create, schedule, and send SMS marketing campaigns via ClickSend.
- **SMS Templates** — Reusable SMS message templates.

### Chat
- **Conversations** — Live chat inbox for visitor conversations from the embeddable chat widget.

### Fulfilment
- **Orders** — Manage orders with line items and downloadable PDFs.
- **Invoices** — Create, send, and track invoice payments with PDF downloads.
- **Deliveries** — Track order deliveries with downloadable PDFs.
- **Purchase Orders** — Manage supplier purchase orders.

### Contacts
- **People** — Individual contact directory.
- **Organisations** — Company/organisation directory.
- **Users** — CRM user management and [invitations](/users#invitations).
- **Teams** — CRM user teams (when the `teams` module is enabled).

### Feedback & Monitoring
- **Features** — Public feature-request and voting board, with a shareable roadmap link. See [Features](/features).
- **Monitors** — Uptime and SSL monitoring for HTTP/HTTPS endpoints. See [Monitoring](/monitoring).

### Catalogue
- **Products** — Product catalogue with search and autocomplete.

### Administration
- **Settings** — [General settings](/settings) (split across a tab per entity), [PDF templates](/pdf-templates), roles and permissions, pipelines, pipeline stages, product categories, tax rates, labels, lead sources, custom fields, custom field groups, chat widgets, and integrations (Xero, ClickSend).
- **Updates** — Installed version, available updates, and the commands to run. See [Updates](/updates).

> **Note:** [Customers](/customers) have full CRUD routes under `/crm/customers` but no sidebar item in the rebuilt 2.x navigation — reach them by URL or from a linked person or organisation. The underlying model is named `Client`; the routes, permissions and UI all say *Customers*.

## Common UI Patterns

### List Views

Most entities provide a paginated table view with:

- **Search** — Text search across key fields.
- **Filters** — Filter by owner, labels, and other entity-specific criteria.
- **Sorting** — Click column headers to sort ascending/descending.
- **Actions** — View, edit, and delete buttons on each row.
- **Create Button** — Prominently placed button to add new records.

### Board Views

Leads, deals, and quotes offer a Kanban-style board view where records are displayed as cards organized by pipeline stage. Cards can be dragged between stages using SortableJS. Switch between list and board views using the toggle buttons.

### Detail Views

Each entity's detail page typically shows:

- **Header** — Title, ID, and status information with action buttons (edit, delete, and entity-specific actions like "Convert to Deal" or "Mark as Won").
- **Details Card** — Key attributes displayed in a structured layout.
- **Related Records** — Associated people, organisations, products, and other linked entities displayed via reusable Livewire sub-components.
- **Activity Timeline** — A chronological feed of notes, tasks, calls, meetings, and file uploads associated with the record.
- **Custom Fields** — Any custom fields configured for the entity type.

### Forms

Create and edit forms use a consistent card-based layout with:

- **Autocomplete Fields** — People, organisations, and products use autocomplete for quick selection.
- **Dynamic Product Lines** — Deals, quotes, orders, invoices, deliveries, and purchase orders support adding multiple product line items with quantity, price, tax rate, and subtotal calculations. Quantities accept up to three decimal places.
- **PDF Template Picker** — Quote, order, invoice, delivery, and purchase-order forms carry a **PDF template** select pinning one of the five shipped layouts to that record, or leaving it to follow the Settings default. See [PDF Templates](/pdf-templates).
- **Address Fields** — Structured address input with multiple address support.
- **Email & Phone Fields** — Multiple email addresses and phone numbers with type selection (work, home, etc.).
- **Label Selection** — Tag records with colour-coded labels.
- **Custom Fields** — Dynamically rendered custom fields based on field group configuration.

### PDF Documents

Quotes, orders, invoices, deliveries, and purchase orders generate downloadable PDF documents via `barryvdh/laravel-dompdf`, rendered through one of five pickable layouts. See [PDF Templates](/pdf-templates).

A **Preview** action beside every download button — on the show page and on index rows — renders the real generated PDF in a slide-over with pdf.js, so a document can be checked before sending without a trip through the browser's download tray. On quotes, invoices and purchase orders a **Get link** button beside it hands over the same 14-day signed portal URL that gets emailed to the recipient.

### Portal Pages

Quotes, invoices, and purchase orders have public-facing portal pages at unique external URLs (`/p/quotes/{external_id}`, `/p/invoices/{external_id}`, `/p/purchase-orders/{external_id}`). These allow recipients to view, accept, or reject a quote, view an invoice, or respond to a purchase order without needing a CRM login.

Those pages render **the record's own PDF template**, chrome-free and pinned to a light theme, so what a customer reads on screen is the document they download rather than a separate HTML layout that drifts away from it.

The public **feature board** lives on the same portal at `/p/features`, with a per-team board at `/p/features/team/{id}` on a multi-tenant install. See [Portal](/portal).

### Toast Notifications

The UI uses MaryUI toast notifications (`Mary\Traits\Toast`) for user feedback on actions.

