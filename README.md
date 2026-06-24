# Tarea OpenAPI YouTube — COM3301 UOH

Modelado de arquitectura de software para una plataforma estilo YouTube,
aplicando **Domain-Driven Design (DDD)** y especificaciones **OpenAPI 3.0**.

---

## Integrantes

- Vicente [Apellido]
- [Nombre compañero] [Apellido]

**Curso:** Análisis y Diseño de Software — COM3301  
**Universidad:** Universidad de O'Higgins  
**Fecha de entrega:** 24 de junio de 2026

---

## Descripción del proyecto

El sistema modela una plataforma de video (estilo YouTube) dividida en
**6 Bounded Contexts**, cada uno con responsabilidades claramente
delimitadas siguiendo los principios de DDD.

Cada contexto tiene su propio modelo de datos y lenguaje ubicuo —
por ejemplo, "Video" significa cosas distintas en cada contexto:

| Contexto | "Video" es... |
|---|---|
| Publicación | Un archivo técnico a procesar |
| Catálogo | Un ítem editorial con título y descripción |
| Descubrimiento | Un candidato de ranking |
| Audiencia | Un objeto que recibe likes y comentarios |
| Monetización | Una unidad monetizable |
| Publicidad | Una oportunidad de inventario publicitario |

---

## Estructura del repositorio

```
/
├── 📁 Publicación/
│   ├── openapi.yaml       ← Especificación OpenAPI
│   └── diagramas.md       ← Diagrama de clases + secuencia (UML)
│
├── 📁 Catálogo/
│   ├── openapi.yaml
│   └── diagramas.md
│
├── 📁 Descubrimiento/
│   ├── openapi.yaml
│   └── diagramas.md
│
├── 📁 Audiencia/
│   ├── openapi.yaml
│   └── diagramas.md
│
├── 📁 Monetización/
│   ├── openapi.yaml
│   └── diagramas.md
│
├── 📁 Publicidad/
│   ├── openapi.yaml
│   └── diagramas.md
│
└── 📄 escenario-integrador.md  ← Flujo completo de punta a punta
```

---

## Los 6 Bounded Contexts

### 1. Publicación y distribución de contenido
Gestiona el ciclo de vida **técnico** del video: subida, procesamiento
(transcoding), reproducción y transmisión en vivo. No decide si el
contenido es visible ni gestiona metadata editorial.

### 2. Catálogo editorial y derechos
Define **qué contenido existe públicamente** y bajo qué reglas puede
mostrarse (visibilidad, restricciones territoriales, reclamaciones de
derechos, moderación).

### 3. Descubrimiento y personalización
Ayuda al usuario a **encontrar contenido relevante**: búsqueda, feeds
personalizados, contenido relacionado y tendencias. Aprende de las
señales de interacción del usuario.

### 4. Audiencia, comunidad y engagement
Gestiona la **relación social** entre usuarios, creadores y contenido:
suscripciones, likes, comentarios, notificaciones e historial personal.

### 5. Monetización del ecosistema creador
Define cómo los creadores **ganan y cobran dinero**: elegibilidad,
membresías, propinas, atribución de ingresos publicitarios y pagos.

### 6. Publicidad y marketplace de anunciantes
Gestiona el **negocio publicitario B2B**: campañas, creativos,
targeting, decisión de entrega de anuncios y facturación a anunciantes.

---

## Escenario integrador

El archivo `escenario-integrador.md` muestra el flujo completo de
punta a punta cruzando los 6 contextos:

```
Viewer ve recomendación  →  Descubrimiento
Valida visibilidad       →  Catálogo
Elige anuncio            →  Publicidad
Reproduce el video       →  Publicación
Registra el like         →  Audiencia
Registra watch time      →  Publicación → Descubrimiento
Atribuye ingreso         →  Publicidad → Monetización
```

---

## Eventos de integración entre contextos

| Evento | Emisor | Consumidor |
|---|---|---|
| `video.asset.ready_for_publishing` | Publicación | Catálogo |
| `catalog.item.published` | Catálogo | Descubrimiento, Audiencia, Monetización, Publicidad |
| `playback.session.completed` | Publicación | Descubrimiento |
| `audience.engagement.signal` | Audiencia | Descubrimiento |
| `ad.revenue.generated` | Publicidad | Monetización |

---

## Cómo visualizar las especificaciones OpenAPI

1. Ir a **https://editor.swagger.io**
2. Copiar el contenido de cualquier `openapi.yaml`
3. Pegarlo en el editor — la documentación se renderiza automáticamente

## Cómo visualizar los diagramas UML

Los archivos `diagramas.md` usan bloques **Mermaid** que GitHub
renderiza automáticamente al ver el archivo en el repositorio.
