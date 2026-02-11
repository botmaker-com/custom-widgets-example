> 🌐 [Leer en Español](./README.es.md)

# Custom Widgets - Developer Guide

Build custom widgets that integrate with Botmaker's operator console. Widgets run inside iframes and communicate with the host application through a secure postMessage protocol.

## Table of Contents

- [Getting Started](#getting-started)
- [Authentication](#authentication)
- [Communication Protocol](#communication-protocol)
- [Events Reference](#events-reference)
- [Actions Reference](#actions-reference)
- [Security](#security)
- [Demo Widgets](#demo-widgets)

## Getting Started

Custom widgets are embedded as iframes within the Botmaker operator console. Each widget:
- Receives a JWT token for authentication
- Listens for events from the host application
- Can send actions back to the host application
- Is scoped to specific screens (CHATS, TICKETS, CONTACTS, etc.)

## Authentication

### Token Delivery

The widget receives a JWT token via query parameter:

```
https://your-widget.com?bmtoken=eyJhbGciOiJSUzI1NiIs...
```

```javascript
const token = new URLSearchParams(window.location.search).get('bmtoken');
```

### Token Payload

| Field | Description |
|-------|-------------|
| `custom-widget-name` | Display name of the widget |
| `businessId` | Business context identifier |
| `email` | Authenticated user's email |
| `sub` | User subject |
| `iat` | Issued at timestamp |
| `exp` | Expiration timestamp |
| `jti` | Unique token identifier |

### Token Verification (Recommended)

For production widgets, verify the JWT signature using Botmaker's public key:

```javascript
import { jwtVerify, importJWK } from 'jose';

const PUBLIC_KEY_URL = 'https://c.botmaker.com/botmaker/identities/botmaker-identity-1.json';

let cachedPublicKey = null;

async function getPublicKey() {
    if (cachedPublicKey) return cachedPublicKey;
    const response = await fetch(PUBLIC_KEY_URL);
    const jwk = await response.json();
    cachedPublicKey = await importJWK(jwk, 'RS256');
    return cachedPublicKey;
}

async function verifyToken(token) {
    try {
        const publicKey = await getPublicKey();
        const { payload } = await jwtVerify(token, publicKey, {
            algorithms: ['RS256'],
            issuer: 'botmaker.com',
        });
        return { valid: true, payload };
    } catch (error) {
        console.error('Token verification failed:', error);
        return { valid: false, error };
    }
}
```

> **Note:** The public key rarely changes — caching it for the lifetime of your application is safe and recommended.

## Communication Protocol

### Message Types

| Type | Direction | Purpose |
|------|-----------|---------|
| `BM_WIDGET_EVENT` | Host → Widget | Events from the host application |
| `BM_WIDGET_ACTION` | Widget → Host | Actions triggered by the widget |
| `BM_WIDGET_ACTION_RESPONSE` | Host → Widget | Response to widget actions |

### Receiving Events

```javascript
window.addEventListener('message', (event) => {
    if (event.data?.messageType === 'BM_WIDGET_EVENT') {
        const { event: eventName, screen, payload, timestamp } = event.data;
        console.log(`Received ${eventName} on ${screen}:`, payload);
    }
});
```

### Sending Actions

```javascript
function sendAction(action, screen, payload) {
    const message = {
        messageType: 'BM_WIDGET_ACTION',
        widgetName: 'your-widget-name', // Must match the registered widget name
        action,
        screen,
        payload,
        requestId: 'req_' + Math.random().toString(36).substring(2, 11),
    };
    window.parent.postMessage(message, '*');
}
```

### Receiving Action Responses

```javascript
window.addEventListener('message', (event) => {
    if (event.data?.messageType === 'BM_WIDGET_ACTION_RESPONSE') {
        const { requestId, success, error, data } = event.data;
        if (success) {
            console.log(`Action ${requestId} succeeded:`, data);
        } else {
            console.error(`Action ${requestId} failed:`, error);
        }
    }
});
```

## Events Reference

### Common Events (All Screens)

#### `widget:ready`

Emitted when the widget iframe is loaded and the communication channel is established.

| Field | Type | Description |
|-------|------|-------------|
| `screen` | `string` | Current screen identifier |

### CHATS Screen Events

#### `chat:selected`

Emitted when an agent selects a chat conversation.

| Field | Type | Description |
|-------|------|-------------|
| `customerId` | `string` | Unique customer identifier |
| `customerName` | `string?` | Customer display name |
| `platform` | `string?` | Messaging platform (e.g., "WHATSAPP", "WEBCHAT") |

#### `chat:messageSent`

Emitted when a message is sent in the current chat.

| Field | Type | Description |
|-------|------|-------------|
| `customerId` | `string` | Customer identifier |
| `messageType` | `string` | Type of message sent |
| `messageId` | `string?` | Unique message identifier |

### CONTACTS Screen Events

#### `contacts:companySelected`

Emitted when a company is selected in the contacts view.

| Field | Type | Description |
|-------|------|-------------|
| `companyId` | `string` | Unique company identifier |

### TICKETS Screen Events

#### `ticket:selected`

Emitted when an agent opens/selects a ticket.

| Field | Type | Description |
|-------|------|-------------|
| `ticketCode` | `string` | Ticket identifier (e.g., "TK-1234") |
| `subject` | `string` | Ticket subject line |
| `state` | `string` | Current state: `NEW`, `OPEN`, `CLOSED`, `CANCELLED` |
| `priority` | `string` | Priority: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `MINOR` |
| `assignee` | `object?` | Assigned agent `{ name: string, id: string }` |
| `supportTeam` | `string?` | Assigned support team name |
| `metrics` | `object` | SLA metrics (see [SLA Metrics](#sla-metrics-payload)) |
| `creationDate` | `string` | ISO 8601 creation date |
| `author` | `object?` | Ticket author `{ firstName: string, lastName: string }` |

#### `ticket:statusChanged`

Emitted when the currently viewed ticket's state changes.

| Field | Type | Description |
|-------|------|-------------|
| `ticketCode` | `string` | Ticket identifier |
| `previousState` | `string` | Previous state |
| `newState` | `string` | New state |
| `metrics` | `object` | Updated SLA metrics |

#### `tickets:queueSummary`

Emitted when the tickets list is loaded or refreshed. Contains summary data for all visible tickets.

| Field | Type | Description |
|-------|------|-------------|
| `tickets` | `array` | Array of ticket summaries (see below) |
| `total` | `number` | Total ticket count |

Each ticket in the array contains:

| Field | Type | Description |
|-------|------|-------------|
| `ticketCode` | `string` | Ticket identifier |
| `subject` | `string` | Ticket subject |
| `state` | `string` | Current state |
| `priority` | `string` | Priority level |
| `assignee` | `object?` | Assigned agent |
| `metrics` | `object` | SLA metrics |
| `creationDate` | `string` | ISO 8601 creation date |

### SLA Metrics Payload

The `metrics` field is a `Record<string, MetricData>` where keys are metric types:

- `TIME_IN_CURRENT_STATE` — Time the ticket has been in its current state
- `TIME_WAITING_ASSIGNATION_IN_CURRENT_SUPPORT_TEAM` — Time waiting for agent assignment
- `TOTAL_TIME_IN_CURRENT_SUPPORT_TEAM` — Total time with the current support team
- `TIME_TO_FIRST_RESPONSE` — Time until the first response is sent
- `TIME_TO_RESOLVE` — Total time to resolve the ticket

Each metric has the following structure:

| Field | Type | Description |
|-------|------|-------------|
| `ticksMillis` | `number` | Elapsed milliseconds |
| `finished` | `boolean` | Whether the metric has stopped counting |
| `isPaused` | `boolean` | Whether the metric is currently paused |
| `pauseReason` | `string?` | Pause reason: `BY_CALENDAR` or `BY_STATE` |
| `thresholdMillis` | `number?` | SLA threshold in milliseconds (from SLA policy) |
| `isBreached` | `boolean?` | Whether the SLA threshold has been exceeded |
| `breachDate` | `number?` | Timestamp of when the SLA was/will be breached |

## Actions Reference

### CHATS Screen Actions

#### `fillInput`

Fills the chat input field with the provided text without sending it.

```javascript
sendAction('fillInput', 'CHATS', { text: 'Hello, how can I help?' });
```

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `text` | `string` | Max 5000 chars, non-empty | Text to place in the input |

#### `sendMessage`

Sends a message to the current customer in the active chat.

```javascript
sendAction('sendMessage', 'CHATS', { text: 'Your order has been shipped!' });
```

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `text` | `string` | Max 5000 chars, non-empty | Message text to send |

### TICKETS Screen Actions

#### `navigateToTicket`

Navigates the tickets view to a specific ticket by its code.

```javascript
sendAction('navigateToTicket', 'TICKETS', { ticketCode: 'TK-1234' });
```

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `ticketCode` | `string` | Max 50 chars, non-empty | Ticket code to navigate to |

## Security

### Origin Validation

All incoming actions are validated against the widget's configured allowed domains. The host application:

1. Checks the message origin against the widget's allowed domain list
2. Supports wildcard subdomains (e.g., `*.example.com`)
3. Validates message structure and payload types
4. Sanitizes text payloads (strips script tags and event handlers)
5. Enforces screen context (actions must match the current screen)

### Best Practices

- **Verify the JWT token** in production to ensure requests come from Botmaker
- **Validate event origins** — check `event.origin` before processing messages
- **Don't store sensitive data** in the widget — the JWT token expires
- **Use HTTPS** for your widget URL
- **Keep widget names unique** per business

## Demo Widgets

Example widgets are available in the [`demos/`](./demos/) directory:

| Demo | Description |
|------|-------------|
| [SLA Guardian](./demos/sla-guardian.html) | Real-time SLA monitoring dashboard for the TICKETS screen. Shows ticket queue grouped by SLA urgency, live countdown timers, and breach alerts. |

To test a demo locally, configure a custom widget in the admin panel pointing to the demo file served from a local HTTP server.
