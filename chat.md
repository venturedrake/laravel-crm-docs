# Chat

[[toc]]

## Overview

The chat module provides a live-chat inbox inside the CRM and an embeddable widget that can be installed on any external website. Visitors start conversations from the widget, and CRM agents reply from the **Chat** section. Messages are delivered in real-time via broadcasting (with polling fallback), and visitors can be promoted to a CRM [Person](/people) record.

The chat module is enabled via the `chat` entry in the [`modules`](/configuration#optional-modules) configuration array and is gated by the `@haschatenabled` Blade directive.

## Models

| Model | Table | Description |
|---|---|---|
| `ChatWidget` | `{prefix}chat_widgets` | An installable widget configuration |
| `ChatVisitor` | `{prefix}chat_visitors` | A site visitor identified by a token cookie |
| `ChatConversation` | `{prefix}chat_conversations` | A thread of messages between a visitor and the CRM |
| `ChatMessage` | `{prefix}chat_messages` | Individual messages from `user` (agent), `visitor`, or `system` |
| `ChatVisitorPageView` | `{prefix}chat_visitor_page_views` | Page-view trail for a visitor |

## Chat Widget

`ChatWidget` is the configuration for an embeddable widget. Each widget has a unique `public_key` used in the embed snippet so multiple sites can be served from a single CRM.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs |
| `public_key` | `string` | Public 64-char key used in the embed snippet |
| `name` | `string` | Internal widget name |
| `welcome_message` | `string` | Greeting shown when a visitor opens the widget |
| `color` | `string` | Hex colour for the widget chrome (default `#2563eb`) |
| `position` | `string` | `bottom-right` or `bottom-left` |
| `allowed_origins` | `array` | Optional CORS allow-list of origins permitted to embed |
| `is_active` | `boolean` | Toggle the widget on/off |

Widgets are managed through **Settings → Chat Widgets**.

### Embed Snippet

Each widget exposes a JavaScript snippet via `ChatWidget::embedSnippet()` that you paste into the host site's HTML:

```html
<!-- Laravel CRM Chat Widget -->
<script>(function(){var s=document.createElement('script');s.async=1;s.src='https://your-crm.test/p/chat/{publicKey}.js';document.head.appendChild(s);})();</script>
```

The widget loader injects an iframe served from the CRM, so the visitor-facing UI lives entirely in the CRM application.

### Public Routes

Widget endpoints are registered **outside** the `web` middleware group (no CSRF or session) so they can be used cross-origin without errors:

| Route | Description |
|---|---|
| `/p/chat/{publicKey}` | Visitor-facing widget HTML (rendered as iframe) |
| `/p/chat/{publicKey}.js` | Widget loader JavaScript |
| Additional JSON endpoints | Conversation polling, message create, visitor identification |

## Chat Visitor

| Attribute | Type | Description |
|---|---|---|
| `visitor_token` | `string` | 64-char token persisted in the visitor's browser |
| `name`, `email` | `string` | Optional details captured from the widget |
| `ip_address`, `user_agent`, `country_code` | `string` | Request context |
| `current_url`, `last_seen_at` | `string`/`datetime` | Last known location and activity |
| `person_id` | `integer` | Set when a visitor is promoted to a CRM `Person` |

## Chat Conversation

| Attribute | Type | Description |
|---|---|---|
| `chat_id` | `string` | Human-readable ID (e.g. `C1001`) |
| `subject` | `string` | Optional subject line |
| `status` | `enum` | `open`, `pending`, or `closed` |
| `user_assigned_id` | `integer` | Assigned CRM user (agent) |
| `last_message_at` | `datetime` | Timestamp of the most recent message |
| `closed_at` | `datetime` | When the conversation was closed |
| `lead_id` | `integer` | Optional [Lead](/leads) the conversation was converted to |

## Chat Message

| Attribute | Type | Description |
|---|---|---|
| `sender_type` | `string` | `user`, `visitor`, or `system` |
| `sender_id` | `integer` | FK to the relevant sender row |
| `body` | `text` | Message body |
| `read_at` | `datetime` | When the recipient last read the message |

## Realtime

Messages are broadcast on the public channel `crm-chat.{conversation_external_id}` via the `ChatMessageSent` event. The agent UI subscribes via Echo with a `wire:poll` fallback so polling continues to work when broadcasting is not configured.

## Service

`VentureDrake\LaravelCrm\Services\ChatService` orchestrates visitor lookup/creation, conversation opening, message creation, and lead promotion. Use it from custom integrations rather than writing to the models directly.

## CRM Inbox

Inside the CRM, agents work conversations from **Chat** in the sidebar:

- **Conversations list** (`crm-chat-index`) — paginated list of open/pending/closed conversations.
- **Conversation view** (`crm-chat-show`) — agent reply view with realtime/polled message updates and conversation actions (assign, close, convert to lead, promote visitor to person).

