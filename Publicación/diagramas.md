# Diagramas — Publicación y distribución de contenido

GitHub renderiza estos bloques ```mermaid``` automáticamente como diagramas
al ver este archivo en el repositorio (no hace falta exportar imágenes).

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class UploadSession {
        +UUID uploadId
        +UUID creatorId
        +String status
        +Integer bytesUploaded
        +Integer totalBytes
        +DateTime createdAt
    }

    class VideoAsset {
        +UUID videoAssetId
        +UUID creatorId
        +String status
        +Float durationSeconds
        +DateTime createdAt
    }

    class Rendition {
        +String quality
        +String codec
        +Integer bitrateKbps
        +String url
    }

    class ProcessingJob {
        +UUID jobId
        +UUID videoAssetId
        +String status
        +Integer retryCount
    }

    class PlaybackSession {
        +UUID sessionId
        +UUID videoAssetId
        +UUID viewerId
        +Integer watchedSeconds
        +String qualityUsed
        +Integer interruptions
    }

    class LiveStream {
        +UUID liveStreamId
        +UUID creatorId
        +String status
        +String ingestUrl
        +UUID recordedVideoAssetId
    }

    UploadSession "1" --> "1" VideoAsset : produce al completarse
    VideoAsset "1" --> "1" ProcessingJob : dispara
    VideoAsset "1" --> "*" Rendition : tiene
    VideoAsset "1" --> "*" PlaybackSession : es reproducido en
    LiveStream "1" --> "0..1" VideoAsset : genera grabación VOD
```

**Notas de diseño:**
- `VideoAsset` aquí es el *archivo/stream técnico*, sin título ni descripción
  (eso vive en `CatalogItem` dentro del contexto de Catálogo — es un modelo
  distinto, no la misma clase compartida).
- `LiveStream` y `UploadSession` son ambos *productores* de `VideoAsset`,
  pero por caminos distintos (subida directa vs. grabación de un live).

## Diagrama de secuencia — "Un creador sube un video hasta que queda listo para publicar"

Este es el flujo end-to-end de RF-P1 → RF-P2, mostrando además la
integración con el contexto de Catálogo (consume el evento que este
contexto emite).

```mermaid
sequenceDiagram
    actor Creador
    participant Pub as Publicación (este contexto)
    participant Cat as Catálogo editorial

    Creador->>Pub: POST /uploads
    Pub-->>Creador: 201 UploadSession (status=uploading)

    loop Subida por partes
        Creador->>Pub: PUT /uploads/{id}/parts
        Pub-->>Creador: 200 OK
    end

    Creador->>Pub: POST /uploads/{id}/complete
    Pub->>Pub: Validar integridad de la subida

    alt Subida válida
        Pub->>Pub: Crear VideoAsset (status=pending)
        Pub->>Pub: Disparar ProcessingJob automáticamente
        Pub-->>Creador: 200 VideoAsset (status=processing)
        Pub->>Pub: Generar renditions (360p/720p/1080p)
        Pub->>Pub: VideoAsset.status = ready
        Pub-->>Cat: evento video.asset.ready_for_publishing
        Cat->>Cat: Crear borrador editorial vinculado al videoAssetId
    else Subida corrupta/incompleta
        Pub-->>Creador: 422 Error
    end
```

**Por qué este flujo valida bien la frontera entre contextos:** Publicación
nunca decide si el contenido es público ni le pone título — solo avisa
"técnicamente está listo". Es Catálogo quien decide qué hacer con esa
notificación (crear un borrador, esperar a que el creador complete
metadata, etc.). Si Publicación intentara también publicar el contenido,
estaríamos mezclando dos responsabilidades de negocio distintas — justo
lo que la "regla del proyecto" del enunciado dice que hay que evitar.
