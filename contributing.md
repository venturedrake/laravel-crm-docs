# Contributing

[[toc]]

## Overview

Thank you for your interest in contributing to Laravel CRM! Contributions are welcome via pull requests on GitHub.

## Development Setup

1. Fork and clone the repository
2. Install dependencies:

```bash
composer install
npm install
```

3. Build frontend assets:

```bash
npm run build
```

## Code Style

Laravel CRM follows the Laravel coding style enforced by [Laravel Pint](https://laravel.com/docs/pint):

```bash
composer format      # Run Laravel Pint formatter
composer format-test # Dry-run formatting check
```

## Testing

Run the test suite:

```bash
composer test
```

Tests use Orchestra Testbench with SQLite in-memory database.

## Pull Requests

- Fork the repository and create your branch from `master`
- Write tests for any new functionality
- Run the formatter before submitting
- Ensure all tests pass
- Write a clear description of your changes

## Bug Reports

File bug reports on the [GitHub Issues](https://github.com/venturedrake/laravel-crm/issues) page with:

- A clear description of the issue
- Steps to reproduce
- Expected vs actual behaviour
- Laravel and package version numbers

