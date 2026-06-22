# Diagramas — Descubrimiento y personalización

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class IndexEntry {
        +UUID catalogItemId
        +UUID channelId
        +String title
        +List~String~ tags
        +String category
        +Float popularityScore
        +DateTime publishedAt
    }

    class Impression {
        +UUID userId
        +String sessionId
        +UUID catalogItemId
        +String surface
        +DateTime occurredAt
    }

    class Click {
        +UUID userId
        +String sessionId
        +UUID catalogItemId
        +String surface
        +DateTime occurredAt
    }

    class WatchSignal {
        +UUID userId
        +UUID catalogItemId
        +Integer watchedSeconds
        +DateTime occurredAt
    }

    class InterestProfile {
        +UUID userId
        +Map~String,Float~ categoryWeights
        +DateTime updatedAt
    }

    class RankedItem {
        +UUID catalogItemId
        +Float score
        +String reason
    }

    IndexEntry "1" --> "*" Impression : recibe
    IndexEntry "1" --> "*" Click : recibe
    IndexEntry "1" --> "*" WatchSignal : recibe
    Impression "*" --> "1" InterestProfile : alimenta
    Click "*" --> "1" InterestProfile : alimenta
    WatchSignal "*" --> "1" InterestProfile : alimenta
    InterestProfile "1" --> "*" RankedItem : produce vía ranking
    IndexEntry "1" ..> "1" RankedItem : se traduce en
```

**Notas de diseño:**
- `IndexEntry` es la representación propia de este contexto del concepto
  "video" — liviana y orientada a ranking, distinta de `CatalogItem`
  (Catálogo) y de `VideoAsset` (Publicación). No duplica metadata
  completa: solo lo necesario para buscar/rankear (título, tags,
  categoría, popularidad).
- `RankedItem` es lo que efectivamente devuelven las APIs (search, feed,
  related, trending): una referencia + score + razón, nunca el contenido
  completo.

## Diagrama de secuencia — "Indexación + recomendación + captura de señal"

Muestra cómo este contexto consume el evento de publicación de Catálogo
para construir su índice, y cómo registra señales que alimentan futuras
recomendaciones — cerrando el ciclo de personalización.

```mermaid
sequenceDiagram
    participant Cat as Catálogo
    participant Desc as Descubrimiento (este contexto)
    actor Viewer
    participant Pub as Publicación

    Cat-->>Desc: evento catalog.item.published
    Desc->>Desc: POST /index/entries (indexar IndexEntry)

    Viewer->>Desc: GET /feed?userId=...
    Desc->>Desc: Calcular ranking usando InterestProfile
    Desc-->>Viewer: 200 FeedPage [RankedItem...]

    Viewer->>Desc: POST /signals/impressions (vio el ítem en el feed)
    Viewer->>Pub: reproduce el video (fuera de este contexto)
    Pub-->>Desc: evento playback.session.completed (watch time)
    Desc->>Desc: Actualizar InterestProfile del usuario

    Note over Desc: El próximo /feed de este usuario ya refleja<br/>la nueva señal de interés.
```

**Por qué este flujo valida bien la frontera entre contextos:**
Descubrimiento nunca decide si el contenido es visible (eso ya lo filtró
Catálogo antes de emitir el evento de publicación) ni almacena el stream
(eso es de Publicación) ni guarda comentarios o likes (eso es de
Audiencia) — solo indexa referencias y aprende patrones de interés a
partir de señales que otros contextos le notifican.
