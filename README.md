# Grupo 5 — Pedidos / Order Management

Microservicio responsable del ciclo de vida de un pedido dentro del ecosistema **Mini Marketplace Cloud UTEM**.

## Descripción del servicio

Este servicio es el núcleo del ecosistema: recibe la creación de un pedido desde Grupo 4 (Checkout), administra la máquina de estados del pedido y publica eventos que consumen Grupo 6 (Despacho), Grupo 7 (Reportería), Grupo 8 (Pagos/Notificaciones) y otros servicios del ecosistema.

## URLs

| Entorno | URL |
|---|---|
| Mock (Prism / Render) | `https://grupo5-pedidos-mock.onrender.com` |
| Producción (E3) | `https://api-grupo5-pedidos.onrender.com/v1` *(pendiente)* |
| Local | `http://localhost:8050/v1` |

## Endpoints expuestos

| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/orders` | Crear pedido (con Idempotency-Key) |
| GET | `/orders/{orderId}` | Obtener pedido por ID |
| PATCH | `/orders/{orderId}/status` | Transicionar estado |
| GET | `/users/{userId}/orders` | Listar pedidos de un usuario (paginado) |

## Estados del pedido

```
CREATED → PAYMENT_PENDING → PAID → STOCK_RESERVED → READY_TO_SHIP → SHIPPED → DELIVERED
                         ↘                                                    ↘ CANCELLED
                          FAILED ←────────────── PAYMENT_REJECTED ────────────┘
```

**Reglas de transición:**
- `PAYMENT_APPROVED` (desde Grupo 8) → transición automática a `PAID`, genera envío síncrono con Grupo 6.
- `PAYMENT_REJECTED` (desde Grupo 8) → transición a `FAILED`, emite `ORDER_CANCELLED`.
- `SHIPMENT_IN_TRANSIT` (desde Grupo 6) → transición a `SHIPPED`.
- `SHIPMENT_DELIVERED` (desde Grupo 6) → transición a `DELIVERED`.

## Eventos publicados

Todos los eventos siguen el sobre estándar `EventEnvelope` del ecosistema:

```yaml
eventId: uuid         # Identificador único del evento (llave de idempotencia)
eventType: string     # Tipo en UPPER_SNAKE_CASE
version: string       # Versión del contrato (Ej: "1.0")
occurredAt: date-time # Timestamp UTC
producer: string      # "group-5-pedidos"
correlationId: uuid?  # UUID de trazabilidad transversal
payload: object       # Contenido en camelCase
```

> **Nota:** Los montos monetarios se transmiten como enteros en CLP (sin decimales) para evitar errores de redondeo.

| Evento | Cuándo se emite | Consumidores |
|---|---|---|
| `ORDER_CREATED` | Al registrar un pedido exitosamente (`201 Created`) | Grupo 7 (Reportería), Grupo 8 (Notificaciones) |
| `ORDER_STATUS_CHANGED` | En cada transición válida de la máquina de estados | Grupo 7 (Reportería), Grupo 8 (Notificaciones) |
| `ORDER_CANCELLED` | Por rechazo de pago o evento de contingencia | Grupo 7 (Reportería), Grupo 8 (Notificaciones) |

## Eventos consumidos

| Evento | Origen | Estado | Lógica interna |
|---|---|---|---|
| `PAYMENT_APPROVED` | Grupo 8 (Pagos) | ✅ Sincronizado | Transiciona a `PAID`, crea envío síncrono (`POST /api/v1/shipments`) con Grupo 6 |
| `PAYMENT_REJECTED` | Grupo 8 (Pagos) | ✅ Sincronizado | Transiciona a `FAILED`, emite `ORDER_CANCELLED` |
| `SHIPMENT_IN_TRANSIT` | Grupo 6 (Despacho) | 🔴 Bloqueado — mitigado vía Polling | Transiciona a `SHIPPED` |
| `SHIPMENT_DELIVERED` | Grupo 6 (Despacho) | 🔴 Bloqueado — mitigado vía Polling | Transiciona a `DELIVERED` |

### ⚠️ Mitigación de bloqueo con Grupo 6 (Despacho)

Grupo 6 tiene implementado el patrón Outbox pero su worker no está activo para inyectar eventos al broker. Como mitigación temporal, Grupo 5 implementa un **worker de polling** que consulta periódicamente:

```
GET /api/v1/shipments?orderId={orderId}
```

**Headers obligatorios para las llamadas de polling:**

```
X-Request-Id: <UUIDv4 aleatorio generado por el worker>
X-Correlation-Id: <UUID original del pedido>
X-Consumer: "G5-Pedidos"
```

**Mapeo de estados:**

| Respuesta Grupo 6 | Estado interno G5 |
|---|---|
| `status: "IN_TRANSIT"` | `SHIPPED` |
| `status: "DELIVERED"` | `DELIVERED` |

**Garantía de idempotencia:** Se valida el `eventId` de cada evento recibido contra un registro histórico (`UNIQUE constraint`) para ignorar reintentos duplicados del broker.

## Estructura del repositorio

```
Grupo5-Pedidos/
├── contrato/
│   ├── openapi.yaml                  # Contrato REST oficial (fuente de verdad)
│   ├── events.md                     # Contrato de eventos asíncronos
│   └── postman_collection.json       # Colección de pruebas contra el mock
├── docs/
│   ├── modelo-de-datos.md            # Entidades, campos y relaciones
│   └── entregables-E2.md             # Evidencia de entregables Fase 2
├── mock/
│   └── README.md                     # Documentación del mock en Render
└── README.md
```

## Integración con otros grupos

| Grupo | Rol | Tipo | Estado |
|---|---|---|---|
| Grupo 2 | Valida JWT (`POST /auth/validate`) | REST | ✅ |
| Grupo 4 | Invoca `POST /orders` al confirmar checkout | REST | ✅ |
| Grupo 6 | Recibe `PAYMENT_APPROVED`, expone `GET /api/v1/shipments` | REST + Evento | 🔴 Polling activo |
| Grupo 7 | Consume `ORDER_CREATED`, `ORDER_STATUS_CHANGED`, `ORDER_CANCELLED` para reportería | Evento | ✅ |
| Grupo 8 | Publica `PAYMENT_APPROVED/REJECTED`; consume `ORDER_CREATED` para notificaciones | Evento | ✅ |

## Headers obligatorios (estándar ecosistema)

```
Authorization: Bearer <jwt>
Idempotency-Key: <uuid>       # obligatorio en POST /orders
X-Correlation-Id: <uuid>      # recomendado en todas las requests
```

## Tecnologías

- **Mock:** Prism (OpenAPI) desplegado en Render
- **Implementación (E3):** Node.js / Express + PostgreSQL en Render
- **CI/CD:** GitHub Actions

## Cómo probar el mock localmente

```bash
# Importar la colección en Postman
# Archivo: contrato/postman_collection.json
# Variable baseUrl: https://grupo5-pedidos-mock.onrender.com
```

Ver `mock/README.md` para más detalles.
Ver `contrato/events.md` para el contrato completo de comunicación asíncrona.
