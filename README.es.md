> 🌐 [Read in English](./README.md)

# Custom Widgets - Guía para Desarrolladores

Construye widgets personalizados que se integran con la consola de operadores de Botmaker. Los widgets se ejecutan dentro de iframes y se comunican con la aplicación host a través de un protocolo seguro de postMessage.

## Tabla de Contenidos

- [Primeros Pasos](#primeros-pasos)
- [Autenticación](#autenticación)
- [Protocolo de Comunicación](#protocolo-de-comunicación)
- [Referencia de Eventos](#referencia-de-eventos)
- [Referencia de Acciones](#referencia-de-acciones)
- [Seguridad](#seguridad)
- [Widgets de Demostración](#widgets-de-demostración)

## Primeros Pasos

Los widgets personalizados se insertan como iframes dentro de la consola de operadores de Botmaker. Cada widget:
- Recibe un token JWT para autenticación
- Escucha eventos de la aplicación host
- Puede enviar acciones de vuelta a la aplicación host
- Está limitado a pantallas específicas (CHATS, TICKETS, CONTACTS, etc.)

## Autenticación

### Entrega del Token

El widget recibe un token JWT a través del parámetro de URL:

```
https://tu-widget.com?bmtoken=eyJhbGciOiJSUzI1NiIs...
```

```javascript
const token = new URLSearchParams(window.location.search).get('bmtoken');
```

### Contenido del Token

| Campo | Descripción |
|-------|-------------|
| `custom-widget-name` | Nombre del widget |
| `businessId` | Identificador del negocio |
| `email` | Email del usuario autenticado |
| `sub` | Sujeto del usuario |
| `iat` | Fecha de emisión |
| `exp` | Fecha de expiración |
| `jti` | Identificador único del token |

### Verificación del Token (Recomendado)

Para widgets en producción, verifica la firma JWT usando la clave pública de Botmaker:

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
        console.error('Falló la verificación del token:', error);
        return { valid: false, error };
    }
}
```

> **Nota:** La clave pública raramente cambia — almacenarla en caché durante el ciclo de vida de tu aplicación es seguro y recomendado.

## Protocolo de Comunicación

### Tipos de Mensaje

| Tipo | Dirección | Propósito |
|------|-----------|-----------|
| `BM_WIDGET_EVENT` | Host → Widget | Eventos desde la aplicación host |
| `BM_WIDGET_ACTION` | Widget → Host | Acciones disparadas por el widget |
| `BM_WIDGET_ACTION_RESPONSE` | Host → Widget | Respuesta a acciones del widget |

### Recibir Eventos

```javascript
window.addEventListener('message', (event) => {
    if (event.data?.messageType === 'BM_WIDGET_EVENT') {
        const { event: eventName, screen, payload, timestamp } = event.data;
        console.log(`Recibido ${eventName} en ${screen}:`, payload);
    }
});
```

### Enviar Acciones

```javascript
function sendAction(action, screen, payload) {
    const message = {
        messageType: 'BM_WIDGET_ACTION',
        widgetName: 'nombre-de-tu-widget', // Debe coincidir con el nombre registrado
        action,
        screen,
        payload,
        requestId: 'req_' + Math.random().toString(36).substring(2, 11),
    };
    window.parent.postMessage(message, '*');
}
```

### Recibir Respuestas de Acciones

```javascript
window.addEventListener('message', (event) => {
    if (event.data?.messageType === 'BM_WIDGET_ACTION_RESPONSE') {
        const { requestId, success, error, data } = event.data;
        if (success) {
            console.log(`Acción ${requestId} exitosa:`, data);
        } else {
            console.error(`Acción ${requestId} falló:`, error);
        }
    }
});
```

## Referencia de Eventos

### Eventos Comunes (Todas las Pantallas)

#### `widget:ready`

Se emite cuando el iframe del widget se carga y el canal de comunicación se establece.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `screen` | `string` | Identificador de la pantalla actual |

### Eventos de la Pantalla CHATS

#### `chat:selected`

Se emite cuando un agente selecciona una conversación.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `customerId` | `string` | Identificador único del cliente |
| `customerName` | `string?` | Nombre del cliente |
| `platform` | `string?` | Plataforma de mensajería (ej: "WHATSAPP", "WEBCHAT") |

#### `chat:messageSent`

Se emite cuando se envía un mensaje en el chat actual.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `customerId` | `string` | Identificador del cliente |
| `messageType` | `string` | Tipo de mensaje enviado |
| `messageId` | `string?` | Identificador único del mensaje |

### Eventos de la Pantalla CONTACTS

#### `contacts:companySelected`

Se emite cuando se selecciona una empresa en la vista de contactos.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `companyId` | `string` | Identificador único de la empresa |

### Eventos de la Pantalla TICKETS

#### `ticket:selected`

Se emite cuando un agente abre o selecciona un ticket.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `ticketCode` | `string` | Identificador del ticket (ej: "TK-1234") |
| `subject` | `string` | Asunto del ticket |
| `state` | `string` | Estado actual: `NEW`, `OPEN`, `CLOSED`, `CANCELLED` |
| `priority` | `string` | Prioridad: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `MINOR` |
| `assignee` | `object?` | Agente asignado `{ name: string, id: string }` |
| `supportTeam` | `string?` | Nombre del equipo de soporte asignado |
| `metrics` | `object` | Métricas de SLA (ver [Métricas de SLA](#payload-de-métricas-de-sla)) |
| `creationDate` | `string` | Fecha de creación ISO 8601 |
| `author` | `object?` | Autor del ticket `{ firstName: string, lastName: string }` |

#### `ticket:statusChanged`

Se emite cuando el estado del ticket actualmente visualizado cambia.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `ticketCode` | `string` | Identificador del ticket |
| `previousState` | `string` | Estado anterior |
| `newState` | `string` | Nuevo estado |
| `metrics` | `object` | Métricas de SLA actualizadas |

#### `tickets:queueSummary`

Se emite cuando la lista de tickets se carga o actualiza. Contiene datos resumidos de todos los tickets visibles.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `tickets` | `array` | Array de resúmenes de tickets (ver abajo) |
| `total` | `number` | Cantidad total de tickets |

Cada ticket en el array contiene:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `ticketCode` | `string` | Identificador del ticket |
| `subject` | `string` | Asunto del ticket |
| `state` | `string` | Estado actual |
| `priority` | `string` | Nivel de prioridad |
| `assignee` | `object?` | Agente asignado |
| `metrics` | `object` | Métricas de SLA |
| `creationDate` | `string` | Fecha de creación ISO 8601 |

### Payload de Métricas de SLA

El campo `metrics` es un `Record<string, MetricData>` donde las claves son tipos de métrica:

- `TIME_IN_CURRENT_STATE` — Tiempo que el ticket lleva en su estado actual
- `TIME_WAITING_ASSIGNATION_IN_CURRENT_SUPPORT_TEAM` — Tiempo esperando asignación de agente
- `TOTAL_TIME_IN_CURRENT_SUPPORT_TEAM` — Tiempo total con el equipo de soporte actual
- `TIME_TO_FIRST_RESPONSE` — Tiempo hasta la primera respuesta
- `TIME_TO_RESOLVE` — Tiempo total para resolver el ticket

Cada métrica tiene la siguiente estructura:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `ticksMillis` | `number` | Milisegundos transcurridos |
| `finished` | `boolean` | Si la métrica dejó de contar |
| `isPaused` | `boolean` | Si la métrica está actualmente pausada |
| `pauseReason` | `string?` | Razón de pausa: `BY_CALENDAR` o `BY_STATE` |
| `thresholdMillis` | `number?` | Umbral de SLA en milisegundos (de la política de SLA) |
| `isBreached` | `boolean?` | Si se superó el umbral de SLA |
| `breachDate` | `number?` | Timestamp de cuándo se incumplió/incumplirá el SLA |

## Referencia de Acciones

### Acciones de la Pantalla CHATS

#### `fillInput`

Rellena el campo de entrada del chat con el texto proporcionado sin enviarlo.

```javascript
sendAction('fillInput', 'CHATS', { text: 'Hola, ¿en qué puedo ayudarte?' });
```

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `text` | `string` | Máx 5000 caracteres, no vacío | Texto a colocar en el campo de entrada |

#### `sendMessage`

Envía un mensaje al cliente actual en el chat activo.

```javascript
sendAction('sendMessage', 'CHATS', { text: '¡Tu pedido ha sido enviado!' });
```

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `text` | `string` | Máx 5000 caracteres, no vacío | Texto del mensaje a enviar |

### Acciones de la Pantalla TICKETS

#### `navigateToTicket`

Navega la vista de tickets a un ticket específico por su código.

```javascript
sendAction('navigateToTicket', 'TICKETS', { ticketCode: 'TK-1234' });
```

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `ticketCode` | `string` | Máx 50 caracteres, no vacío | Código del ticket al que navegar |

## Seguridad

### Validación de Origen

Todas las acciones entrantes se validan contra los dominios permitidos configurados del widget. La aplicación host:

1. Verifica el origen del mensaje contra la lista de dominios permitidos del widget
2. Soporta subdominios comodín (ej: `*.example.com`)
3. Valida la estructura del mensaje y tipos de payload
4. Sanitiza los payloads de texto (elimina etiquetas script y event handlers)
5. Aplica contexto de pantalla (las acciones deben coincidir con la pantalla actual)

### Mejores Prácticas

- **Verifica el token JWT** en producción para asegurar que las solicitudes provienen de Botmaker
- **Valida los orígenes de eventos** — verifica `event.origin` antes de procesar mensajes
- **No almacenes datos sensibles** en el widget — el token JWT expira
- **Usa HTTPS** para la URL de tu widget
- **Mantén nombres de widget únicos** por negocio

## Widgets de Demostración

Widgets de ejemplo están disponibles en el directorio [`demos/`](./demos/):

| Demo | Descripción |
|------|-------------|
| [SLA Guardian](./demos/sla-guardian.html) | Dashboard de monitoreo de SLA en tiempo real para la pantalla TICKETS. Muestra la cola de tickets agrupada por urgencia de SLA, temporizadores en vivo y alertas de incumplimiento. |

Para probar un demo localmente, configura un widget personalizado en el panel de administración apuntando al archivo de demo servido desde un servidor HTTP local.
