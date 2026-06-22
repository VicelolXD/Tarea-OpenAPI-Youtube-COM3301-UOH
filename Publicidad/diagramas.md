# Diagramas — Publicidad y marketplace de anunciantes

## Diagrama de clases (conceptual)

```mermaid
classDiagram
    class Advertiser {
        +UUID advertiserId
        +String companyName
        +String contactEmail
    }

    class Campaign {
        +UUID campaignId
        +UUID advertiserId
        +String objective
        +Float budgetAmount
        +String status
        +Date startDate
        +Date endDate
    }

    class Creative {
        +UUID creativeId
        +UUID campaignId
        +String type
        +String assetUrl
        +String policyStatus
    }

    class TargetingCriteria {
        +UUID campaignId
        +List~String~ countries
        +List~String~ ageRanges
        +List~String~ contentCategories
    }

    class AdDecision {
        +UUID decisionId
        +UUID catalogItemId
        +Boolean adSelected
        +UUID campaignId
        +UUID creativeId
    }

    class Impression {
        +UUID impressionId
        +UUID decisionId
        +UUID catalogItemId
        +UUID campaignId
        +Boolean billable
    }

    Advertiser "1" --> "*" Campaign : posee
    Campaign "1" --> "*" Creative : usa
    Campaign "1" --> "1" TargetingCriteria : define
    Campaign "1" --> "*" AdDecision : participa en
    AdDecision "1" --> "0..1" Impression : produce
```

**Notas de diseño:**
- `AdDecision` modela explícitamente el resultado de "evaluar una
  oportunidad de visualización", incluyendo el caso `adSelected=false`
  (no mostrar nada) — RF-F6 lo pide expresamente.
- Ninguna clase aquí contiene metadata editorial del video (eso es
  Catálogo) ni datos del creador o su saldo (eso es Monetización):
  `catalogItemId` es solo una referencia para targeting/elegibilidad.
- `Impression.billable` distingue entre "se sirvió" y "cuenta para
  facturación", porque RF-F7 separa medición bruta de gasto real.

## Diagrama de secuencia — "Elegir anuncio → impresión facturable → ingreso para el creador"

Cubre el paso 3 del escenario integrador ("Elegir anuncio → Publicidad")
y su conexión con el paso 7 ("Atribuir ingreso al creador →
Monetización"), cerrando el ciclo completo del proyecto.

```mermaid
sequenceDiagram
    participant Pub as Publicación
    participant Ads as Publicidad (este contexto)
    participant Cat as Catálogo
    participant Mon as Monetización

    Pub->>Ads: POST /ad-decisions (oportunidad: catalogItemId, slotType=pre_roll)
    Ads->>Cat: GET /catalog-items/{id}/visibility (consulta síncrona)
    Cat-->>Ads: 200 visible=true

    Ads->>Ads: Evaluar campañas activas + targeting + presupuesto
    alt Hay un anuncio elegible
        Ads-->>Pub: 200 AdDecision (adSelected=true, creativeId)
        Pub->>Pub: Reproducir el anuncio antes del video
        Pub->>Ads: POST /impressions
        Ads->>Ads: Crear Impression (billable=false)
        Ads->>Ads: POST /impressions/{id}/confirm-billable
        Ads->>Ads: Impression.billable = true
        Ads-->>Mon: evento ad.revenue.generated (monto, catalogItemId, channelId)
        Note over Mon: Monetización confirma el RevenueEvent<br/>y acredita el saldo del creador.
    else No hay anuncio elegible / presupuesto agotado
        Ads-->>Pub: 200 AdDecision (adSelected=false)
    end
```

**Por qué este flujo valida bien la frontera entre contextos:**
Publicidad consulta a Catálogo solo para confirmar visibilidad (consulta
síncrona, permitida por RNF-4), pero nunca decide publicar ni
despublicar nada. Y aunque Publicidad es quien cobra al anunciante,
**no paga directamente al creador** — solo emite un evento de ingreso
generado; es Monetización quien decide cómo repartirlo. Esto evita
exactamente el error típico listado en el enunciado: "Payouts en
Publicidad".
