# Widget Test Page

A test page for bidirectional iframe widget communication.

## Token

The widget receives a JWT token via query parameter (`?bmtoken=...`). The token payload contains:

- `custom-widget-name` - Display name of the widget
- `businessId` - Business context
- `email` - User email
- `sub` - User subject
- `iat` - Issued at timestamp
- `exp` - Expiration timestamp
- `jti` - Unique token identifier

### Reading the Token

```javascript
const token = new URLSearchParams(window.location.search).get('bmtoken');
```

### Verifying the Token (Recommended)

For production widgets, you should verify the JWT signature using Botmaker's public key. The public key can be fetched and cached:

```javascript
import { jwtVerify, importJWK } from 'jose';

const PUBLIC_KEY_URL = 'https://c.botmaker.com/botmaker/identities/botmaker-identity-1.json';

let cachedPublicKey = null;

async function getPublicKey() {
    if (cachedPublicKey) {
        return cachedPublicKey;
    }
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
        const { businessId, jti, email, sub } = payload;
        return { valid: true, payload };
    } catch (error) {
        console.error('Token verification failed:', error);
        return { valid: false, error };
    }
}
```

> **Note:** The public key rarely changes, so caching it for the lifetime of your application is safe and recommended to avoid unnecessary network requests.

## Communication Protocol

### Message Types

| Type | Direction | Purpose |
|------|-----------|---------|
| `BM_WIDGET_EVENT` | Parent → Widget | Events from the parent application |
| `BM_WIDGET_ACTION` | Widget → Parent | Actions triggered by the widget |
| `BM_WIDGET_ACTION_RESPONSE` | Parent → Widget | Response to widget actions |

### Receiving Events

```javascript
window.addEventListener('message', (event) => {
  if (event.data?.messageType === 'BM_WIDGET_EVENT') {
    const { event: eventName, screen, payload, timestamp } = event.data;
    // Handle the event
  }
});
```

### Sending Actions

```javascript
const message = {
  messageType: 'BM_WIDGET_ACTION',
  widgetName: 'your-widget-name',
  action: 'fillInput',
  screen: 'CHATS',
  payload: { text: 'Hello world' },
  requestId: 'req_' + Math.random().toString(36).substring(2, 11),
};

window.parent.postMessage(message, '*');
```

### Available Actions (CHATS Screen)

| Action | Payload | Description |
|--------|---------|-------------|
| `fillInput` | `{ text: string }` | Fills the chat input with the provided text |
| `sendMessage` | `{ text: string }` | Sends a message to the current customer |

### Available Events (CHATS Screen)

| Event | Payload | Description |
|-------|---------|-------------|
| `chat:selected` | `{ customerId, customerName?, platform? }` | User selected a chat |
| `chat:messageSent` | `{ customerId, messageType, messageId? }` | Message was sent |
