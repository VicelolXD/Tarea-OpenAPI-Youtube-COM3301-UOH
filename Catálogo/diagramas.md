# Diagramas — Catálogo editorial y derechos

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class Channel {
        +UUID channelId
        +UUID creatorId
        +String name
        +String description
        +String imageUrl
    }

    class CatalogItem {
        +UUID catalogItemId
        +UUID channelId
        +UUID videoAssetId
        +String title
        +String description
        +List~String~ tags
        +String category
        +String status
        +DateTime publishedAt
    }

    class Playlist {
        +UUID playlistId
        +UUID channelId
        +String title
    }

    class PlaylistItem {
        +UUID catalogItemId
        +Integer position
    }

    class RightsClaim {
        +UUID claimId
        +UUID catalogItemId
        +UUID claimantId
        +String reason
        +String status
    }

    class PolicyRestriction {
        +UUID restrictionId
        +UUID catalogItemId
        +String type
        +String status
        +DateTime appliedAt
        +DateTime liftedAt
    }

    Channel "1" --> "*" CatalogItem : publica
    Channel "1" --> "*" Playlist : organiza
    Playlist "1" --> "*" PlaylistItem : contiene
    PlaylistItem "*" --> "1" CatalogItem : referencia
    CatalogItem "1" --> "*" RightsClaim : puede tener
    CatalogItem "1" --> "*" PolicyRestriction : acumula historial de
```

**Notas de diseño:**
- `CatalogItem.videoAssetId` es solo una **referencia** (un UUID) al
  `VideoAsset` que vive en el contexto de Publicación. No se copian sus
  campos técnicos (renditions, duración, status de procesamiento) — eso
  rompería el principio de modelo propio por contexto.
- `PolicyRestriction` se modela como historial (puede haber varias en el
  tiempo, activas o levantadas), no como un único campo de estado, porque
  RF-C6 pide explícitamente mantener historial de política.

## Diagrama de secuencia — "Desde asset técnico listo hasta contenido publicado"

Este flujo conecta el evento de salida del Contexto 1 (Publicación) con la
publicación editorial, y muestra cómo Catálogo notifica a los demás
contextos al publicar.

```mermaid
sequenceDiagram
    participant Pub as Publicación
    participant Cat as Catálogo (este contexto)
    actor Creador
    participant Desc as Descubrimiento
    participant Mon as Monetización
    participant Ads as Publicidad

    Pub-->>Cat: evento video.asset.ready_for_publishing
    Cat->>Cat: Crear CatalogItem (status=draft)

    Creador->>Cat: PATCH /catalog-items/{id} (título, descripción, tags, miniatura)
    Cat-->>Creador: 200 CatalogItem actualizado

    Creador->>Cat: POST /catalog-items/{id}/publish
    Cat->>Cat: Verificar restricciones de política activas

    alt Sin restricciones bloqueantes
        Cat->>Cat: status = published
        Cat-->>Desc: evento catalog.item.published
        Cat-->>Mon: evento catalog.item.published
        Cat-->>Ads: evento catalog.item.published
        Cat-->>Creador: 200 publicado
    else Restricción activa (ej. reclamación de derechos aceptada)
        Cat-->>Creador: 409 no se puede publicar
    end
```

**Por qué este flujo valida bien la frontera entre contextos:** Catálogo
es el único que decide *si* algo es publicable (revisa sus propias
restricciones), pero no le dice a Descubrimiento *cómo* rankear el
contenido, ni a Monetización *cuánto* pagar — solo notifica el cambio de
estado vía evento y cada contexto decide qué hacer con esa información
(RNF-4: preferir eventos asíncronos, evitar acoplamiento).
