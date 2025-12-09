# 📈 Análisis de KPIs para Clustering - AURA

> **Fecha:** 2025-12-09  
> **Versión:** 2.0  
> **Referencia:** [Documentación de Persistencia](./data-persistence.md), [Reporte de Clustering](./reporte_clustering_aura.md)

---

Documento que presenta el análisis de los **Key Performance Indicators (KPIs)**, la estructura de la base de datos analítica y las instrucciones SQL para el Análisis Exploratorio de Datos (EDA) y clustering de usuarios en la plataforma AURA.

---

## 📑 Tabla de Contenidos

1. [KPIs para Clustering](#-1-key-performance-indicators-kpis)
2. [Base de Datos Analítica](#-2-base-de-datos-analítica)
3. [Análisis de Sentimiento NLP](#-3-análisis-de-sentimiento-nlp)
4. [Instructivo SQL para Extracción](#-4-instructivo-sql-para-extracción)
5. [Procesamiento en Tiempo Real](#-5-procesamiento-en-tiempo-real)

---

## 💎 1. Key Performance Indicators (KPIs)

Bajo el contexto AURA (Jóvenes, Apoyo Emocional, Abuso de Sustancias, Aislamiento Social), estos 6 KPIs son críticos para detectar patrones de riesgo:

### KPI 1: Ratio de Reciprocidad Social

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Aislamiento Social.** Un ratio alto (sigue a muchos, pocos seguidores) indica búsqueda de validación sin reciprocidad. |
| **Fuente** | `aura_social.user_profiles` |
| **Fórmula** | `following_count / followers_count` |

### KPI 2: Días desde Última Conexión

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Aislamiento/Retiro.** Indicador directo de abandono de la plataforma, correlacionado con crisis de salud mental. |
| **Fuente** | `aura_messaging.users` (`last_seen_at`) |
| **Fórmula** | `(NOW() - last_seen_at) / 86400` días |

### KPI 3: Ratio de Mensajes Nocturnos

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Desorden Circadiano.** Proxy de insomnio asociado con ansiedad, depresión o abuso de sustancias. |
| **Fuente** | `aura_messaging.messages` |
| **Fórmula** | `mensajes(1-5AM) / total_mensajes × 100%` |

### KPI 4: Índice de Apatía del Perfil

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Apoyo Emocional.** Perfil incompleto indica anhedonia o apatía (falta de inversión en identidad digital). |
| **Fuente** | `user_profiles.bio`, `complete_profiles` |
| **Fórmula** | `bio IS NULL AND complete_profile IS NULL` |

### KPI 5: Índice de Negatividad en Texto (NLP)

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Crisis Emocional.** Alto índice de negatividad en contenido generado es indicador temprano de depresión o riesgo de autolesión. |
| **Fuente** | Posts, Comments, Messages |
| **Fórmula** | `Σ P(Negative|Text) / N_textos` (via Transformers) |

### KPI 6: Densidad de Participación Comunitaria

| Aspecto | Descripción |
|:--------|:------------|
| **Contexto AURA** | **Red de Apoyo.** Baja densidad sugiere círculos cerrados o falta de intereses diversificados. |
| **Fuente** | `community_members`, `communities.category` |
| **Fórmula** | `COUNT DISTINCT(category)` |

---

## 💾 2. Base de Datos Analítica

El servicio de clustering implementa su propia base de datos `aura_data_miner` para almacenar el resultado del proceso ETL.

### Tabla `user_feature_vector`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | SERIAL | Clave primaria |
| `user_id_raiz` | UUID | ID del usuario (Auth Service) |
| `extraction_date` | TIMESTAMP | Fecha del ETL |
| `reciprocity_ratio_norm` | FLOAT | KPI 1 normalizado [0-1] |
| `days_since_last_seen_norm` | FLOAT | KPI 2 normalizado [0-1] |
| `ratio_night_messages` | FLOAT | KPI 3 |
| `is_profile_incomplete` | BOOLEAN | KPI 4 |
| `sentiment_negativity_index` | FLOAT | KPI 5 (NLP) [0-1] |
| `num_community_categories_norm` | FLOAT | KPI 6 normalizado [0-1] |
| `cluster_label` | VARCHAR | Resultado: ALTO_RIESGO, RIESGO_MODERADO, BAJO_RIESGO |

---

## 🧠 3. Análisis de Sentimiento NLP

### Modelo Utilizado

**UMUTeam/roberta-spanish-sentiment-analysis** - RoBERTa fine-tuned para español.

### Implementación

```python
from transformers import pipeline

# Inicializar pipeline de sentimiento
sentiment_pipeline = pipeline(
    "sentiment-analysis",
    model="UMUTeam/roberta-spanish-sentiment-analysis",
    return_all_scores=True
)

def calculate_negativity_index(texts: list) -> float:
    """Calcula índice de negatividad promedio [0-1]."""
    if not texts:
        return 0.0
    
    scores = []
    for text in texts:
        result = sentiment_pipeline(text[:512])[0]
        neg_score = next(
            (item['score'] for item in result 
             if item['label'] in ['NEG', 'NEGATIVE']),
            0.0
        )
        scores.append(neg_score)
    
    return sum(scores) / len(scores) if scores else 0.0
```

---

## 🛠️ 4. Instructivo SQL para Extracción

### CTE Unificado para Extracción de KPIs

```sql
WITH UserBase AS (
    SELECT
        up.user_id AS user_id_raiz,
        up.id AS profile_id_social,
        up.followers_count,
        up.following_count,
        up.bio IS NULL AS profile_bio_missing,
        cp.user_id IS NULL AS complete_profile_missing
    FROM aura_social.user_profiles up
    LEFT JOIN aura_social.complete_profiles cp ON up.id = cp.user_id
    WHERE up.is_active = TRUE
),

MessagingMetrics AS (
    SELECT
        u.profile_id,
        u.last_seen_at,
        COALESCE(m.total_messages, 0) AS total_messages,
        COALESCE(m.night_messages, 0) AS night_messages
    FROM aura_messaging.users u
    LEFT JOIN (
        SELECT 
            sender_profile_id,
            COUNT(*) AS total_messages,
            SUM(CASE WHEN EXTRACT(HOUR FROM created_at) BETWEEN 1 AND 5 THEN 1 ELSE 0 END) AS night_messages
        FROM aura_messaging.messages WHERE is_deleted = FALSE
        GROUP BY sender_profile_id
    ) m ON u.id = m.sender_profile_id
),

CommunityMetrics AS (
    SELECT
        cm.user_id AS profile_id_social,
        COUNT(DISTINCT c.category) AS distinct_categories
    FROM aura_social.community_members cm
    JOIN aura_social.communities c ON cm.community_id = c.id
    GROUP BY cm.user_id
)

SELECT
    ub.user_id_raiz,
    -- KPI 1: Reciprocidad
    CASE WHEN ub.followers_count > 0 
         THEN ub.following_count::NUMERIC / ub.followers_count 
         ELSE 0 END AS reciprocity_ratio,
    -- KPI 2: Días inactivo
    EXTRACT(EPOCH FROM (NOW() - mm.last_seen_at)) / 86400 AS days_since_last_seen,
    -- KPI 3: Ratio nocturno
    CASE WHEN mm.total_messages > 0 
         THEN mm.night_messages::NUMERIC / mm.total_messages 
         ELSE 0 END AS ratio_night_messages,
    -- KPI 4: Perfil incompleto
    (ub.profile_bio_missing AND ub.complete_profile_missing) AS is_profile_incomplete,
    -- KPI 5: Sentimiento (calculado en Python)
    NULL::FLOAT AS sentiment_negativity_index,
    -- KPI 6: Comunidades
    COALESCE(cm.distinct_categories, 0) AS num_community_categories
FROM UserBase ub
LEFT JOIN MessagingMetrics mm ON ub.profile_id_social = mm.profile_id
LEFT JOIN CommunityMetrics cm ON ub.profile_id_social = cm.profile_id_social;
```

---

## 🔄 5. Procesamiento en Tiempo Real

### CDC con PostgreSQL LISTEN/NOTIFY

El sistema v2.0 implementa **Change Data Capture** para procesar cambios incrementalmente:

```sql
-- Trigger para notificar cambios en mensajes
CREATE OR REPLACE FUNCTION notify_messages_change() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('aura_data_change', json_build_object(
        'table', 'messages',
        'user_id', COALESCE(NEW.sender_profile_id, OLD.sender_profile_id),
        'timestamp', NOW()
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER messages_change_trigger
    AFTER INSERT OR UPDATE ON messages
    FOR EACH ROW EXECUTE FUNCTION notify_messages_change();
```

### Pipeline de Streaming

Cuando llega una notificación:

1. **Extracción incremental** - Solo datos del usuario afectado
2. **Recálculo de KPIs** - Actualizar vector de características
3. **Predicción de riesgo** - Clasificar con el ensamble
4. **Notificación WebSocket** - Broadcast a clientes conectados

### Tablas Monitoreadas

| Base de Datos | Tablas con Triggers |
|:--------------|:--------------------|
| `aura_messaging` | `messages`, `users` |
| `aura_social` | `posts`, `comments`, `user_profiles`, `community_members` |

---

*Documento actualizado para AURA Clustering Service v2.0*
