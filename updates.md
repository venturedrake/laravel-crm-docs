# Updates

[[toc]]

## Overview

The CRM tells you when it needs attention. Two surfaces do this:

- The **updates page** at `/crm/updates`, which shows the installed version, the latest published version, and a "How to update" card with the exact commands to run.
- The **system check banner**, a dismissible bar rendered above the page content on every CRM screen when something is outstanding.

Both are driven by `Services\SystemCheckService`, and both are silent when `update_notifications` is off.

The commands themselves are covered end to end in the [Upgrade Guide](/upgrading); this page describes the machinery.

## The two commands

```bash
php artisan laravelcrm:upgrade
php artisan laravelcrm:update
```

They do deliberately different jobs.

### `laravelcrm:upgrade`

> Republish Laravel CRM assets and clear caches. Safe to run on every composer install — touches no database

Republishes built assets, prunes stale content-hashed build output, publishes Flasher assets and clears cached config, routes and views.

**It also warns about drifted published views.** It md5-compares every published blade under `resources/views/vendor/laravel-crm` against the copy the package ships, and names both the views that have drifted and the ones the package no longer ships at all. `PdfTemplateRegistry::LEGACY_VIEWS` is skipped, since a preserved override is the entire point of those. This is how a view frozen at an older component contract used to go unnoticed until it 500'd a page in production, with nothing anywhere in the upgrade path saying so.

The check **only warns** — it never fails a composer run, because it fires from a `post-autoload-dump` hook on production boxes. Re-publish a view it names, or diff it and re-apply your edits. See the [Upgrade Guide](/upgrading#views-to-re-publish).

**It never opens a database connection, and it never prompts.** That is what makes it safe to fire from a composer hook: during a build the database may be unreachable, mid-migration, or belong to a different release, and there is nobody at a TTY to answer a question. It also exits successfully — rather than failing the composer run — when the application is not yet in a state to publish into.

### `laravelcrm:update`

> Apply Laravel CRM database migrations, seed data and backfills

```bash
php artisan laravelcrm:update --force
```

Calls `laravelcrm:upgrade` first, so one command by hand still does everything, then runs migrations, the base seeder, the lookup-data seeders and the one-time data backfills, and stamps `db_version` on success.

`--force` skips the production confirmation prompt, for deploy scripts. Since 2.4.0 the command **exits non-zero** when a migration or the base seeder fails, so a `&&` chain stops where it should.

| | `laravelcrm:upgrade` | `laravelcrm:update` |
|---|---|---|
| Republishes built assets | Yes | Yes (calls `upgrade` first) |
| Prunes stale content-hashed build files | Yes | Yes |
| Clears cached config, routes, views | Yes | Yes |
| Publishes Flasher assets | Yes | Yes |
| Warns about drifted published views | Yes | Yes |
| Publishes migration stubs | No | Yes |
| Runs `migrate` | No | Yes |
| Runs seeders and data backfills | No | Yes |
| Stamps `db_version` | No | Yes |
| Opens a database connection | Never | Yes |
| Prompts | Never | Only on an interactive production console, unless `--force` |

## The composer hook

`laravelcrm:install` writes this into the host application's `composer.json`:

```json
"scripts": {
    "post-autoload-dump": [
        "@php artisan package:discover --ansi",
        "@php artisan laravelcrm:upgrade --ansi"
    ]
}
```

From then on, every `composer install` and `composer update` republishes CRM assets and clears caches on its own — including a production `composer install --no-dev`, which is why the hook is `post-autoload-dump` rather than `post-update-cmd`.

The line must come **after** `package:discover`; that is what makes the package's artisan commands resolvable.

Installs from before 2.4.0 add the line by hand once, then run `composer dump-autoload` to confirm it fires. See [Upgrade Guide → After upgrading](/upgrading#after-upgrading).

> **Important:** Remove the line before you remove the package. `post-autoload-dump` fires on `composer remove venturedrake/laravel-crm` too, at which point `laravelcrm:upgrade` no longer exists and composer reports a non-zero exit code.

## System-check banner

The banner is the `crm-system-check` Livewire component, backed by `SystemCheckService`. It produces three alert types.

| Alert | Meaning | What to run |
|---|---|---|
| `upgrade_required` | The schema is behind the initial release — the `crm_settings` table is missing, or the host `users` table never got `crm_access` / `last_online_at` / `current_crm_team_id`. Every other check reads one of those, so this short-circuits the rest. | `php artisan laravelcrm:update` |
| `update_available` | A newer release exists than the one installed. Compared with `version_compare()`, so `2.2.0` is correctly older than `2.10.0`. | `composer update venturedrake/laravel-crm` |
| `db_update_required` | The database is behind the code. | `php artisan laravelcrm:update` |

`db_update_required` fires on any of three independent signals, because each has a blind spot on its own:

- a `db_update_*` flag still sitting at `0`;
- a missing or stale [`db_version` marker](#database-version-marker);
- an unrun migration **belonging to this package** — those loaded from `database/updates`, plus published stubs matched back to the `.stub` set by filename.

That last check is deliberately scoped. The host application's own migrations and other packages' migrations live in the same directories and the same migrator paths, and an unrun one of those says nothing about whether the CRM's database is behind — it would otherwise raise a banner telling the operator to run `laravelcrm:update` over someone else's schema change. The check is also guarded on the migration repository existing, so a host that has never migrated reports `upgrade_required` instead.

### Caching and dismissal

Alerts are cached for **300 seconds** under `app.crm-system-check`. Each alert set has a short signature; dismissing the banner stores that signature against the user, so the banner stays hidden until the underlying alerts actually change.

### Gating

The banner renders only when `update_notifications` is enabled, and only for users holding `view crm updates`. The dismiss action re-checks that permission server-side.

The `laravel-crm.updates.index` route carries `can:view crm updates` as well. Before 2.4.0 the route was auth-only while the sidebar link was gated, so anyone could read the install's version and update state by typing the URL.

> **Note:** `view crm updates` is one of the few permissions that is not part of a create/view/edit/delete quad. Owner and Admin receive it with `Permission::all()`; Manager and Employee do not hold it. See [Roles](/roles).

## Database Version Marker

`laravelcrm:install` and `laravelcrm:update` stamp a `db_version` setting with `config('laravel-crm.version')` on success. `SystemCheckService` reads it back to answer "is the code ahead of the database?".

The existing `version` setting cannot carry this. `Http/Middleware/Settings` overwrites `version` with the config value on the first web request after a deploy, so it always reports the code's version and can never reveal that the database is behind it.

Two details worth knowing:

- **A missing marker counts as behind.** Either the install predates the marker, or it was never stamped — both want the same action.
- **It is written install-wide.** `db_version` and the `db_update_*` flags describe the schema, which is install-wide, but settings are team-scoped under `laravel-crm.teams` and a console command has no authenticated user. They are read and written with the team scope dropped, and a flag counts as pending when *any* of its rows holds `0`, so per-team duplicates left by older versions cannot mask a genuinely outstanding update.

## Configuration

```php
// config/laravel-crm.php
'update_notifications' => env('LARAVEL_CRM_UPDATE_NOTIFICATIONS', true),

'docs_url' => env('LARAVEL_CRM_DOCS_URL', 'https://github.com/venturedrake/laravel-crm'),

'upgrade_guide_url' => env('LARAVEL_CRM_UPGRADE_GUIDE_URL', 'https://laravelcrm.com/docs/2.x/upgrading'),
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_UPDATE_NOTIFICATIONS` | `true` | Show the updates page banner and alerts |
| `LARAVEL_CRM_DOCS_URL` | `https://github.com/venturedrake/laravel-crm` | Where "View version X details" points — release notes |
| `LARAVEL_CRM_UPGRADE_GUIDE_URL` | `https://laravelcrm.com/docs/2.x/upgrading` | Where every "Upgrade guide" link points — the updates page and the banner |

The two URLs are separate because they answer different questions: `docs_url` answers *what is in this release?*, `upgrade_guide_url` answers *how do I install it?*. Override either if you host your own documentation.

See [Configuration](/configuration) for the full config reference.
