# Quotes, Orders & Invoices

[[toc]]

## Quotes

Quotes allow you to create formal proposals that can be sent to contacts, accepted, or rejected.

### List & Board Views

Quotes support both list and Kanban board views, similar to leads and deals. The list view shows quote number, title, contact, value, and status.

### Creating a Quote

The quote form includes:

- **Contact** — Select a person and/or organisation.
- **Deal** — Optionally link to an existing deal.
- **Product Lines** — Add products with quantity, price, tax, and discount.
- **Terms & Description** — Text fields for terms and conditions.

### Quote Actions

- **Send** — Email the quote directly to the contact.
- **Download** — Generate and download a PDF of the quote.
- **Accept / Reject** — Mark the quote as accepted or rejected.
- **Unaccept / Unreject** — Reverse accept/reject status.

### Portal

Each quote has a public portal URL (`/p/quotes/{external_id}`) that allows the recipient to view and respond to the quote without a CRM login.

## Orders

Orders track confirmed sales, typically created after a deal is won or a quote is accepted.

### Creating an Order

The order form includes:

- **Contact** — Person and/or organisation.
- **Product Lines** — Products with quantity, price, and tax.
- **Delivery Address** — Shipping address for the order.

### Order Actions

- **Download** — Generate a PDF of the order.
- **Create Delivery** — Create a delivery record from the order.

## Invoices

Invoices manage billing for orders and deals.

### Creating an Invoice

Invoices can be created standalone or from an existing order or deal. The form includes:

- **Contact** — Person and/or organisation.
- **Invoice Lines** — Products/services with quantity, price, and tax.
- **Due Date** — Payment due date.
- **Terms** — Payment terms.

### Invoice Actions

- **Send** — Email the invoice to the contact.
- **Download** — Generate a PDF.
- **Pay** — Record a payment against the invoice.

### Portal

Invoices have a public portal URL (`/p/invoices/{external_id}`) for recipients to view and pay invoices.

## Deliveries

Deliveries track the fulfilment of orders.

### Creating a Delivery

Deliveries are typically created from an order using the **Create Delivery** action. They include delivery product lines and delivery address information.

### Delivery Actions

- **Download** — Generate a PDF delivery note.

## Purchase Orders

Purchase orders manage orders placed with suppliers.

### Creating a Purchase Order

The form includes supplier organisation, product lines, and delivery details. Purchase orders can also be created in bulk using the multiple creation feature.

### Purchase Order Actions

- **Send** — Email the purchase order to the supplier.
- **Download** — Generate a PDF.

