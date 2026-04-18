# Overview

[[toc]]

## Introduction

Laravel CRM ships with a complete, ready-to-use web interface built with Bootstrap, jQuery, and Livewire. The UI provides full CRUD management for all CRM entities, a Kanban-style pipeline board, global search, activity tracking, and role-based navigation — all accessible at `/crm` in your application.

## Accessing the CRM

Once installed, navigate to `http://<yoursite>/crm`. If you are not authenticated, you will be redirected to your application's login page. After logging in, users with CRM access will see the dashboard.

## Layout Structure

The CRM interface uses a fixed top navbar with a sidebar navigation layout:

- **Top Navbar** — Displays the application name, global search bar, team switcher (if enabled), and user dropdown menu.
- **Left Sidebar** — Contains the main navigation menu, organized into logical sections with dividers. Navigation items are permission-gated, so users only see what they have access to.
- **Content Area** — The main content area where pages are rendered. Most pages follow a consistent card-based layout.

## Navigation

The sidebar navigation is organized into the following sections:

### Sales Pipeline
- **Dashboard** — Overview with summary cards showing totals for leads, deals, people, and organisations.
- **Leads** — Manage unqualified prospects (list and board views).
- **Deals** — Track qualified opportunities through pipeline stages (list and board views).
- **Quotes** — Create and manage quotes linked to deals (list and board views).

### Activity
- **Activity** — Unified view of all activities including tasks, notes, calls, meetings, lunches, and files.

### Fulfilment
- **Orders** — Manage orders with line items and downloadable PDFs.
- **Invoices** — Create, send, and track invoice payments with PDF downloads.
- **Deliveries** — Track order deliveries with downloadable PDFs.
- **Purchase Orders** — Manage supplier purchase orders.

### Contacts
- **Clients** — Manage client records that link people and organisations.
- **Organisations** — Company/organisation directory.
- **People** — Individual contact directory.
- **Users** — CRM user management and invitations.
- **Teams** — Team management (when enabled).

### Catalogue
- **Products** — Product catalogue with search and autocomplete.

### Administration
- **Settings** — CRM configuration, including feature toggles.
- **Updates** — Check for package updates.

## Common UI Patterns

### List Views

Most entities (leads, deals, people, organisations, etc.) provide a paginated table view with:

- **Search** — Text search across key fields.
- **Filters** — Filter by owner, labels, and other entity-specific criteria.
- **Sorting** — Click column headers to sort ascending/descending.
- **Actions** — View, edit, and delete buttons on each row.
- **Create Button** — Prominently placed button to add new records.

### Board Views

Leads, deals, and quotes offer a Kanban-style board view where records are displayed as cards organized by pipeline stage. You can switch between list and board views using the toggle buttons.

### Detail Views

Each entity's detail page typically shows:

- **Header** — Title, ID, and status information with action buttons (edit, delete, and entity-specific actions like "Convert to Deal" or "Mark as Won").
- **Details Card** — Key attributes displayed in a structured layout.
- **Related Records** — Associated people, organisations, products, and other linked entities.
- **Activity Timeline** — A chronological feed of notes, tasks, calls, meetings, and file uploads associated with the record.
- **Custom Fields** — Any custom fields configured for the entity type.

### Forms

Create and edit forms use a consistent card-based layout with:

- **Autocomplete Fields** — People, organisations, and products use AJAX-powered autocomplete for quick selection.
- **Dynamic Product Lines** — Deals, quotes, orders, and invoices support adding multiple product line items with quantity, price, tax rate, and subtotal calculations.
- **Address Fields** — Structured address input with multiple address support.
- **Email & Phone Fields** — Multiple email addresses and phone numbers with type selection (work, home, etc.).
- **Label Selection** — Tag records with colour-coded labels.
- **Custom Fields** — Dynamically rendered custom fields based on field group configuration.

### PDF Documents

Quotes, orders, invoices, and deliveries can generate downloadable PDF documents. Quotes and invoices can also be emailed directly to contacts via a send action.

### Portal Pages

Quotes and invoices have public-facing portal pages accessible via unique external URLs. These allow recipients to view, accept, or reject quotes and view/pay invoices without needing a CRM login.
