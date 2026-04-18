# Getting Started

[[toc]]

## Overview

There are several ways to get started with Laravel CRM, depending on your needs and technical requirements. Choose the option that best fits your situation.

## 1. Laravel CRM Package

Install the Laravel CRM package into your existing Laravel application. This is ideal if you already have a Laravel project and want to add CRM functionality to it.

- **Best for:** Developers who want full control and customisation within an existing Laravel app.
- **Requirements:** An existing Laravel application, PHP, MySQL/MariaDB, Composer.

```bash
composer require venturedrake/laravel-crm
```

Follow the full [Installation Guide](/installation) for step-by-step setup instructions including migrations, configuration, and user model setup.

## 2. Self-Hosted Project

A complete, ready-to-go Laravel CRM project that you can clone and run immediately. This is the fastest way to get a fully functional CRM up and running on your own server.

- **Best for:** Teams who want a standalone CRM without integrating into an existing app.
- **Requirements:** A server with PHP, MySQL/MariaDB, and Composer.

```bash
git clone --depth=1 https://github.com/venturedrake/laravel-crm-starter.git
cd laravel-crm-starter
composer install
cp .env.example .env
php artisan key:generate
php artisan laravelcrm:install
```

See the [Quick Start](/quickstart) guide for full details.

## 3. Cloud Hosting

Deploy Laravel CRM to the cloud using [Laravel Cloud](https://cloud.laravel.com/) for a managed hosting experience. Laravel Cloud handles infrastructure, scaling, and deployments so you can focus on your business.

- **Best for:** Teams who want self-hosted control without managing servers.
- **Requirements:** A Laravel Cloud account.

Get started by deploying the self-hosted project to Laravel Cloud:

1. Create a [Laravel Cloud](https://cloud.laravel.com/) account.
2. Connect your repository containing the Laravel CRM starter project.
3. Configure your environment variables and database.
4. Deploy.

## 4. SaaS Version

A fully managed, hosted version of Laravel CRM — no installation, no servers, no maintenance. Sign up and start using your CRM immediately.

- **Best for:** Businesses who want to get started instantly without any technical setup.
- **Requirements:** None — just a web browser.

**Coming soon.** Visit [laravelcrm.com](https://laravelcrm.com) for updates and to join the waitlist.

## Which Option Should I Choose?

| Option | Technical Skill | Setup Time | Server Management | Customisation |
|---|---|---|---|---|
| **Package** | Advanced | Medium | You manage | Full |
| **Self-Hosted Project** | Intermediate | Quick | You manage | Full |
| **Cloud Hosting** | Intermediate | Quick | Managed | Full |
| **SaaS Version** | None | Instant | Managed | Limited |

