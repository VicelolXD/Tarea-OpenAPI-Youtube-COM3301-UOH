# Escenario Integrador — Flujo de punta a punta

Este diagrama muestra cómo interactúan los 6 bounded contexts en el
escenario completo: desde que un viewer recibe una recomendación hasta
que el creador recibe el ingreso publicitario correspondiente.

## Diagrama de secuencia UML

```mermaid
sequenceDiagram
    actor Viewer
    participant Desc as Descubrimiento
    participant Cat as Catálogo
    participant Pub as Publicación
    participant Ads as Publicidad
    participant Aud as Audiencia
    participant Mon as Monetización

    %% Paso 1: El viewer recibe una recomendación
    Viewer->>Desc: GET /feed?userId=...
    Desc->>Desc: Calcular ranking con InterestProfile
    Desc-->>Viewer: 200 FeedPage [RankedItem...]

    %% Paso 2: Catálogo valida que el contenido es visible
    Viewer->>Cat: (implícito) ¿Este contenido es visible?
    Cat->>Cat: Verificar restricciones activas
    Cat-->>Viewer: visible=true

    %% Paso 3: Publicidad decide el anuncio a mostrar
    Viewer->>Ads: (vía Publicación) POST /ad-decisions
    Ads->>Cat: GET /catalog-items/{id}/visibility
    Cat-->>Ads: visible=true, sin restricciones de marca
    Ads->>Ads: Evaluar campañas activas + targeting
    Ads-->>Pub: AdDecision (adSelected=true, creativeId)

    %% Paso 4: Publicación reproduce el contenido
    Viewer->>Pub: GET /video-assets/{id}/playback-info
    Pub-->>Viewer: 200 PlaybackInfo (manifestUrl, calidades)
    Viewer->>Pub: POST /playback-sessions (inicia reproducción)
    Pub-->>Viewer: 201 PlaybackSession

    %% Paso 5: Audiencia registra el like del viewer
    Viewer->>Aud: POST /catalog-items/{id}/reactions (like)
    Aud->>Aud: Registrar Reaction
    Aud-->>Desc: evento audience.engagement.signal
    Note over Desc: Descubrimiento actualiza el<br/>ranking del contenido.

    %% Paso 6: Publicación registra el watch time
    Viewer->>Pub: POST /playback-sessions/{id}/complete
    Pub-->>Desc: evento playback.session.completed (watch time)
    Note over Desc: Descubrimiento actualiza el<br/>InterestProfile del viewer.

    %% Paso 7: Publicidad confirma impresión y genera ingreso
    Pub->>Ads: POST /impressions (impresión del anuncio)
    Ads->>Ads: POST /impressions/{id}/confirm-billable
    Ads-->>Mon: evento ad.revenue.generated (monto, channelId)

    %% Paso 8: Monetización atribuye el ingreso al creador
    Mon->>Mon: POST /revenue-events (source=advertising)
    Mon->>Mon: POST /revenue-events/{id}/confirm
    Mon->>Mon: Aplicar revenue share → acreditar Balance
    Mon-->>Mon: evento revenue.attributed
    Note over Mon: El creador ya tiene saldo<br/>disponible para retirar.
```

## Resumen de responsabilidades por paso

| Paso | Contexto | Acción |
|------|----------|--------|
| 1 | Descubrimiento | Genera el feed personalizado usando el InterestProfile del viewer |
| 2 | Catálogo | Valida que el contenido es visible (sin restricciones activas) |
| 3 | Publicidad | Decide qué anuncio mostrar según campañas activas y targeting |
| 4 | Publicación | Entrega el stream técnico al viewer y registra la sesión |
| 5 | Audiencia | Registra el like y emite señal de engagement a Descubrimiento |
| 6 | Publicación | Registra el watch time y lo notifica a Descubrimiento |
| 7 | Publicidad | Confirma la impresión como facturable y notifica el ingreso a Monetización |
| 8 | Monetización | Aplica el revenue share y acredita el saldo del creador |

## Eventos de integración entre contextos

| Evento | Emisor | Consumidor | Propósito |
|--------|--------|------------|-----------|
| `video.asset.ready_for_publishing` | Publicación | Catálogo | Avisar que el asset técnico está listo |
| `catalog.item.published` | Catálogo | Descubrimiento, Audiencia, Monetización, Publicidad | Avisar que el contenido es público |
| `catalog.item.unpublished` | Catálogo | Descubrimiento, Publicidad | Remover del índice y detener anuncios |
| `playback.session.completed` | Publicación | Descubrimiento | Señal de watch time para ranking |
| `audience.engagement.signal` | Audiencia | Descubrimiento | Señal de likes/reacciones para ranking |
| `audience.user.subscribed` | Audiencia | Audiencia (notificaciones) | Disparar notificaciones de nuevo contenido |
| `ad.revenue.generated` | Publicidad | Monetización | Transferir ingreso publicitario confirmado |
| `revenue.attributed` | Monetización | — | El creador tiene saldo disponible |
| `campaign.budget.exhausted` | Publicidad | Publicidad | Pausar entrega de anuncios de esa campaña |
