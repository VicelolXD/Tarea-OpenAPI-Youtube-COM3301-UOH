# Diagramas — Monetización del ecosistema creador

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class MonetizationStatus {
        +UUID channelId
        +String status
        +DateTime evaluatedAt
    }

    class MembershipTier {
        +UUID tierId
        +UUID channelId
        +String name
        +Float price
        +String currency
    }

    class Membership {
        +UUID membershipId
        +UUID viewerId
        +UUID tierId
        +DateTime startedAt
    }

    class Tip {
        +UUID tipId
        +UUID viewerId
        +UUID catalogItemId
        +Float amount
        +String currency
    }

    class RevenueEvent {
        +UUID revenueEventId
        +UUID channelId
        +UUID catalogItemId
        +String source
        +Float amount
        +String status
        +Float creatorShareAmount
    }

    class Balance {
        +UUID channelId
        +Float availableAmount
        +String currency
    }

    class Payout {
        +UUID payoutId
        +UUID channelId
        +Float amount
        +String status
        +DateTime requestedAt
    }

    MonetizationStatus "1" --> "1" Balance : habilita
    MembershipTier "1" --> "*" Membership : se compra como
    Membership "1" --> "1" RevenueEvent : genera
    Tip "1" --> "1" RevenueEvent : genera
    RevenueEvent "*" --> "1" Balance : acredita
    Balance "1" --> "*" Payout : origina
```

**Notas de diseño:**
- `RevenueEvent` es la clase central: unifica publicidad, membresías y
  propinas bajo un mismo concepto de "ingreso a repartir", con un campo
  `source` que distingue el origen. Esto evita tener tres tablas de
  ingreso paralelas con lógica de reparto duplicada.
- `RevenueEvent` no almacena los datos de la campaña publicitaria ni del
  anunciante — eso vive en Publicidad. Solo llega el monto ya calculado
  vía `externalReference`, manteniendo la frontera entre "cuánto pagó el
  anunciante" (Publicidad) y "cuánto le toca al creador" (Monetización).
- `Balance` es un acumulado derivado de `RevenueEvent` confirmados, no
  una clase que se edita directamente — solo cambia vía eventos de
  ingreso confirmados o pagos procesados.

## Diagrama de secuencia — "Del ingreso publicitario al pago al creador"

Este es el tramo final del escenario integrador completo (reproducción →
like → watch time → **atribución de ingresos**), mostrando cómo este
contexto consume el ingreso ya confirmado por Publicidad y lo convierte
en saldo disponible para retiro.

```mermaid
sequenceDiagram
    participant Ads as Publicidad
    participant Mon as Monetización (este contexto)
    actor Creador

    Ads-->>Mon: evento ad.revenue.confirmed (monto, catalogItemId, channelId)
    Mon->>Mon: POST /revenue-events (status=pending, source=advertising)
    Mon->>Mon: POST /revenue-events/{id}/confirm
    Mon->>Mon: Aplicar revenue share configurado
    Mon->>Mon: Acreditar Balance del canal
    Mon-->>Creador: evento revenue.attributed (notificación)

    Creador->>Mon: GET /channels/{id}/balance
    Mon-->>Creador: 200 Balance disponible

    Creador->>Mon: POST /channels/{id}/payouts (amount)
    alt Saldo suficiente y sobre el mínimo
        Mon->>Mon: Crear Payout (status=requested)
        Mon-->>Creador: 201 Payout en proceso
    else Saldo insuficiente
        Mon-->>Creador: 409 Error
    end
```

**Por qué este flujo valida bien la frontera entre contextos:**
Monetización confía en el monto que le entrega Publicidad (no
recalcula impresiones ni precios de subasta — eso sería invadir el
contexto de Publicidad) y se limita a aplicar las reglas de reparto y
gestionar el ciclo de vida del dinero del creador (saldo → retiro →
pago), que es exactamente su responsabilidad según RF-M3 a RF-M5.
