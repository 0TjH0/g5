# Contrato de Eventos — Grupo 5: Pedidos (Order Management)

Este documento define la estructura y el comportamiento de la comunicación asíncrona del servicio de Pedidos (`order-service`). Todos los eventos transmitidos a través del bus del ecosistema utilizan el sobre estándar `EventEnvelope`:

```yaml
EventEnvelope:
  eventId: uuid            # Identificador único del evento (Llave de idempotencia)
  eventType: string        # Tipo de evento en UPPER_SNAKE_CASE
  version: string          # Versión del contrato del evento (Ej: "1.0")
  occurredAt: date-time    # Timestamp UTC en que se generó el evento
  producer: string         # Identificador del emisor: "group-5-pedidos"
  correlationId: uuid?     # UUID de trazabilidad transversal de la solicitud
  payload: object          # Contenido del evento en formato camelCase

{
  "eventId": "evt-3f2a1b00-0000-4000-8000-000000000001",
  "eventType": "ORDER_CREATED",
  "version": "1.0",
  "occurredAt": "2026-06-20T10:00:00Z",
  "producer": "group-5-pedidos",
  "correlationId": "99999999-8888-4777-9666-555555555555",
  "payload": {
    "orderId": "ORD-20260620-001",
    "userId": "e9d8c7b6-a543-2109-8765-fedcba098765",
    "status": "CREATED",
    "totalAmount": 1599980,
    "currency": "CLP",
    "items": [
      {
        "productId": "f0e9d8c7-b6a5-4321-0987-fedcba098765",
        "name": "Notebook Lenovo IdeaPad",
        "quantity": 2,
        "unitPrice": 799990,
        "subtotal": 1599980
      }

{
  "eventId": "evt-3f2a1b00-0000-4000-8000-000000000002",
  "eventType": "ORDER_STATUS_CHANGED",
  "version": "1.0",
  "occurredAt": "2026-06-20T10:05:00Z",
  "producer": "group-5-pedidos",
  "correlationId": "99999999-8888-4777-9666-555555555555",
  "payload": {
    "orderId": "ORD-20260620-001",
    "userId": "e9d8c7b6-a543-2109-8765-fedcba098765",
    "previousStatus": "PAYMENT_PENDING",
    "status": "PAID"
  }
}

{
  "eventId": "evt-3f2a1b00-0000-4000-8000-000000000003",
  "eventType": "ORDER_CANCELLED",
  "version": "1.0",
  "occurredAt": "2026-06-20T10:10:00Z",
  "producer": "group-5-pedidos",
  "correlationId": "99999999-8888-4777-9666-555555555555",
  "payload": {
    "orderId": "ORD-20260620-001",
    "userId": "e9d8c7b6-a543-2109-8765-fedcba098765",
    "status": "CANCELLED",
    "reason": "PAYMENT_REJECTED"
  }
}

{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440001",
  "eventType": "PAYMENT_APPROVED",
  "version": "1.0",
  "occurredAt": "2026-06-20T10:04:00Z",
  "producer": "g8-pagos",
  "correlationId": "99999999-8888-4777-9666-555555555555",
  "payload": {
    "paymentId": "PAY-a1b2c3d4-e5f6-7890-1234-56789abcdef0",
    "orderId": "ORD-20260620-001",
    "userId": "e9d8c7b6-a543-2109-8765-fedcba098765",
    "amount": 1599980,
    "currency": "CLP"
  }
}












    ]
  }
}
