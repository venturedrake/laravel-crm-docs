# Security

[[toc]]

## Overview

Laravel CRM takes security seriously. The package implements multiple layers of protection to ensure that only authorised users can access CRM data and perform actions.

## Authentication

All CRM routes (except public portal pages) are protected by the `auth.laravel-crm` middleware stack, which includes:

- **Authentication** — Users must be logged in via your application's standard authentication system. Unauthenticated users are redirected to the login page.
- **CRM Access Check** — The `HasCrmAccess` middleware verifies that the authenticated user has the `crm_access` flag enabled. Users without access receive a `403 Forbidden` response.

## Authorisation

### Roles & Permissions

Laravel CRM uses a role-based access control system powered by [Spatie Laravel Permission](https://spatie.be/docs/laravel-permission). Every CRM route is protected by permission checks using Laravel's `can` middleware.

Key permissions include:

- `view crm leads`, `view crm deals`, `view crm people`, `view crm organisations`, etc.
- `view crm settings`, `view crm updates`

See the [Roles](/reference/roles) and [Permissions](/reference/permissions) reference for full details.

### Policies

Each CRM model has a dedicated policy class that controls `viewAny`, `view`, `create`, `update`, and `delete` actions. These policies are applied at the route level to ensure consistent authorisation.

### Owner Auto-Assignment

The first user matching the `crm_owner` config value is automatically assigned the **Owner** role with full permissions. When teams are enabled, the team creator is assigned the Owner role for that team.

## Team Isolation

When teams are enabled, CRM data is scoped to the current team. The `TeamsPermission` middleware ensures that users can only access data belonging to their active team. Team-scoped permissions prevent cross-team data access.

## Public Portal Pages

Quotes and invoices have public-facing portal pages accessible via unique UUID-based URLs (e.g., `/p/quotes/{external_id}`). These pages:

- Do **not** require authentication.
- Are protected by unguessable UUIDs rather than sequential IDs.
- Only expose the specific quote or invoice — no other CRM data is accessible.

## CSRF Protection

All CRM forms and POST/PUT/DELETE requests are protected by Laravel's built-in CSRF token verification.

## Soft Deletes

Most CRM models use soft deletes, meaning records are not permanently removed from the database when deleted. This provides an audit trail and the ability to restore accidentally deleted records.

## Audit Trail

CRM models track which user created, updated, deleted, and restored each record via foreign key relationships (`user_created_id`, `user_updated_id`, `user_deleted_id`, `user_restored_id`). See the [Users](/reference/users) reference for details.

## Reporting Security Vulnerabilities

If you discover a security vulnerability in Laravel CRM, please report it responsibly. **Do not open a public GitHub issue for security vulnerabilities.**

Instead, please email security concerns directly to **security@venturedrake.com**. We will acknowledge your report promptly and work with you to understand and address the issue before any public disclosure.

When reporting, please include:

- A description of the vulnerability and its potential impact.
- Steps to reproduce the issue.
- Any relevant code snippets or proof of concept.

We appreciate responsible disclosure and will credit reporters (with permission) in our release notes.
