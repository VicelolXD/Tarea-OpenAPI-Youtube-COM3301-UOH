# Diagramas — Audiencia, comunidad y engagement

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class Subscription {
        +UUID userId
        +UUID channelId
        +DateTime subscribedAt
    }

    class Reaction {
        +UUID reactionId
        +UUID catalogItemId
        +UUID userId
        +String type
    }

    class Comment {
        +UUID commentId
        +UUID catalogItemId
        +UUID userId
        +UUID parentCommentId
        +String text
        +Boolean pinned
        +DateTime createdAt
    }

    class Notification {
        +UUID notificationId
        +UUID userId
        +String type
        +Boolean read
    }

    class HistoryEntry {
        +UUID userId
        +UUID catalogItemId
        +DateTime viewedAt
    }

    class CommunityPost {
        +UUID postId
        +UUID channelId
        +String text
        +DateTime createdAt
    }

    Subscription "*" --> "1" Notification : puede disparar
    Comment "1" --> "*" Comment : respuestas (thread)
    Comment "*" --> "1" Notification : puede disparar
    CommunityPost "1" --> "*" Comment : recibe
    CommunityPost "1" --> "*" Reaction : recibe
```

**Notas de diseño:**
- `Comment` se auto-referencia con `parentCommentId` para modelar threads
  (RF-A3), en vez de tener una clase separada `Reply`.
- Ninguna de estas clases guarda metadata editorial del video ni del
  canal — solo `catalogItemId`/`channelId` como referencia, validando su
  existencia contra Catálogo cuando haga falta, tal como exige la
  restricción de diseño del contexto.
- `Subscription` es deliberadamente simple (sin precio, sin nivel): la
  versión "de pago" (membresías) es una clase completamente distinta que
  vive en Monetización.

## Diagrama de secuencia — "Publicación dispara notificación + reacción genera señal"

Cubre dos integraciones clave: consumir el evento de Catálogo para
notificar suscriptores, y emitir una señal de engagement consumida por
Descubrimiento (paso 5 del escenario integrador: "Registrar like →
Audiencia").

```mermaid
sequenceDiagram
    participant Cat as Catálogo
    participant Aud as Audiencia (este contexto)
    actor Viewer
    participant Desc as Descubrimiento

    Cat-->>Aud: evento catalog.item.published
    Aud->>Aud: Buscar suscriptores del canal
    loop Por cada suscriptor
        Aud->>Aud: Crear Notification (type=new_video)
    end

    Viewer->>Aud: POST /catalog-items/{id}/reactions (like)
    Aud->>Aud: Registrar Reaction
    Aud-->>Desc: evento audience.engagement.signal (agregado)

    Note over Desc: Descubrimiento usa esta señal para<br/>ajustar ranking y recomendaciones futuras.
```

**Por qué este flujo valida bien la frontera entre contextos:** Audiencia
reacciona a que algo se publicó (Catálogo) generando notificaciones, y
produce señales de interacción, pero nunca decide qué se recomienda
después (eso es responsabilidad exclusiva de Descubrimiento) ni almacena
metadata editorial — solo enriquece la relación social alrededor del
contenido.
