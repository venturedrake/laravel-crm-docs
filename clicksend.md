# ClickSend

[[toc]]

## Overview

ClickSend is the SMS gateway used by the [SMS Marketing](/sms-marketing) module to send campaign messages. The integration is built on the ClickSend REST API v3 with HTTP Basic authentication.

**Service:** `VentureDrake\LaravelCrm\Services\ClickSendService`

## Configuration

Credentials are stored as CRM settings (not `.env` values) and are managed via **Settings → Integrations → ClickSend** in the CRM UI.

| Setting key | Description |
|---|---|
| `clicksend_username` | Your ClickSend account username |
| `clicksend_api_key` | Your ClickSend API key |
| `clicksend_default_from` | Default sender ID for outbound SMS (number or alphanumeric, subject to country rules) |

## Connecting

1. Create a [ClickSend account](https://clicksend.com/?u=47224) and retrieve your API key from the ClickSend dashboard.
2. In the CRM, go to **Settings → Integrations → ClickSend** and enter your username, API key, and default sender ID.
3. The integration activates immediately — no restart or `.env` change is required.

## Usage

`ClickSendService` is injected automatically via the service container. It is used internally by the SMS campaign dispatch jobs but can also be called directly:

```php
use VentureDrake\LaravelCrm\Services\ClickSendService;

$clickSend = app(ClickSendService::class);

if ($clickSend->isConfigured()) {
    $result = $clickSend->sendSms(
        to: '+64211234567',
        body: 'Hello from Laravel CRM!',
    );

    // $result = ['ok' => true, 'message_id' => '...', 'status' => 'SUCCESS', 'error' => null]
}
```

## API Endpoint

All requests go to `https://rest.clicksend.com/v3` using HTTP Basic auth (`username:api_key`).

## Related

- [SMS Marketing](/sms-marketing) — campaigns and templates powered by this integration.

