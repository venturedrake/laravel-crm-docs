# Laravel CRM Documentation

Documentation content for [Laravel CRM](https://github.com/venturedrake/laravel-crm).

## Structure

This is a **branch-per-version** repository: every documentation page lives at the repository root, and the branch you are on determines the version. The `2.x` branch holds the docs published at `laravelcrm.com/docs/2.x/*`.

```
documentation.md   # Navigation index — every page must be linked from here
*.md               # One file per documentation page
```

Page files are plain markdown with no front matter. Each page opens with an `# H1`, a blank line, and `[[toc]]` on line 3 — `documentation.md` and this file are the only exceptions. Internal links are root-relative and extensionless, e.g. `[Orders](/orders)`.

## Usage

This repo is consumed by the Laravel CRM docs site via a symlink or clone into `storage/docs/`.
