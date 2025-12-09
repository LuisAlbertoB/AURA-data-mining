# 📈 Análisis de KPIs para Clustering y Estructura de Datos Analítica - AURA

> **Fecha:** 2025-12-05  
> **Referencia:** [Documentación de Persistencia](./data-persistence.md)  
> **Contexto:** Análisis Exploratorio de Datos (EDA) y Clustering de Usuarios

---

El siguiente documento presenta el análisis de los **Key Performance Indicators (KPIs)**, la estructura de la base de datos analítica y el conjunto de instrucciones SQL, diseñado específicamente para la fase de Análisis Exploratorio de Datos (EDA) y la posterior clusterización de usuarios para la plataforma AURA.

## 💎 1. Key Performance Indicators (KPIs) para Clustering AURA

Bajo el contexto de AURA (Jóvenes, Apoyo Emocional, Abuso de Sustancias, Aislamiento Social), los siguientes KPIs son críticos para la detección de patrones de riesgo en un modelo no supervisado:

### KPI 1: Ratio de Reciprocidad Social

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Aislamiento Social**. Mide el equilibrio de las relaciones sociales. Un ratio muy superior a 1 (sigue a muchos, pocos seguidores) puede indicar una búsqueda activa de validación sin ser correspondido, mientras que un ratio muy bajo puede reflejar un comportamiento pasivo o de retiro. |
| **Fuente de Persistencia** | `aura_social.user_profiles` |
| **Operación (Alto Nivel)** | Proporción entre el total de usuarios que el joven sigue y el total de sus seguidores. |
| **Operación (Bajo Nivel)** | $$ R = \frac{\text{followers\_count}}{\text{following\_count}} $$ |

### KPI 2: Días desde Última Conexión (Retirada)

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Aislamiento Social / Abuso de Sustancias**. Es el indicador más directo de retiro o abandono de la plataforma, el cual puede correlacionarse con el inicio de una crisis de salud mental o el escalamiento de un problema de aislamiento social severo. |
| **Fuente de Persistencia** | `aura_messaging.users` |
| **Operación (Alto Nivel)** | Cálculo de la diferencia temporal entre el momento de la extracción y la última marca de tiempo de actividad registrada. |
| **Operación (Bajo Nivel)** | $$ D_{last} = \frac{\text{EXTRACT(EPOCH FROM(NOW() - last\_seen\_at))}}{86400} $$ *(Resultado en días)* |

### KPI 3: Ratio de Mensajes Nocturnos

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Abuso de Sustancias / Apoyo Emocional**. Proxy de un desorden del ritmo circadiano (insomnio o actividad anormal). El insomnio es un síntoma clave asociado con ansiedad, depresión y, a menudo, con abuso de sustancias. |
| **Fuente de Persistencia** | `aura_messaging.messages` (`created_at`) |
| **Operación (Alto Nivel)** | Porcentaje de mensajes enviados entre las 1:00 AM y las 5:00 AM sobre el total de mensajes enviados por el usuario. |
| **Operación (Bajo Nivel)** | $$ R_{night} = \frac{\sum(\text{messages.created\_at entre 1:00 y 5:00})}{\sum(\text{total messages})} \times 100 $$ |

### KPI 4: Índice de Apatía del Perfil (Incompletitud)

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Apoyo Emocional**. Un perfil incompleto puede ser un indicador de anhedonia (pérdida de placer o interés) o apatía, ya que el joven no invierte esfuerzo en su identidad digital. |
| **Fuente de Persistencia** | `aura_social.user_profiles` (`bio`), `aura_social.complete_profiles` |
| **Operación (Alto Nivel)** | Variable booleana que indica la ausencia de campos clave de autodefinición (`bio` en user_profiles y la ausencia de registro en complete_profiles). |
| **Operación (Bajo Nivel)** | $$ A_{empty} = \text{CASE WHEN(bio IS NULL AND complete\_profiles IS NULL, 1, 0)} $$ |

### KPI 5: Índice de Negatividad en Texto (NLP)

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Apoyo Emocional / Abuso de Sustancias**. Mide el tono emocional predominante en el contenido generado por el usuario. Un alto índice de negatividad en posts, comentarios y mensajes puede ser un indicador temprano de crisis, depresión o riesgo de autolesión. |
| **Fuente de Persistencia** | `aura_social.posts` (`content`), `aura_social.comments` (`content`), `aura_messaging.messages` (`content`) |
| **Operación (Alto Nivel)** | Promedio ponderado de la probabilidad de sentimiento negativo detectado por modelos de lenguaje (RoBERTa/BERT) en todos los textos del usuario. |
| **Operación (Bajo Nivel)** | $$ I_{neg} = \frac{\sum P(Negative|Text_i)}{N_{textos}} $$ *(Calculado externamente vía Python/NLP)* |

### KPI 6: Densidad de Participación Comunitaria

| Aspecto | Descripción Contextual (AURA) |
| :--- | :--- |
| **Contexto AURA** | **Aislamiento Social**. Mide la amplitud de la red de apoyo. Una baja densidad (pocas categorías de comunidad) sugiere que el joven se relaciona en círculos muy cerrados o no tiene intereses diversificados. |
| **Fuente de Persistencia** | `aura_social.community_members`, `communities` (`category`) |
| **Operación (Alto Nivel)** | Conteo de categorías únicas de las comunidades a las que pertenece un usuario (ej. 'Meditación', 'Deportes', 'Gaming'). |
| **Operación (Bajo Nivel)** | $$ D_{com} = \text{COUNT DISTINCT(communities.category) por user\_id} $$ |

---

## 💾 2. Estructura de la Base de Datos Analítica (`aura_data_miner_db`)

El nuevo servicio de minería implementará su propia base de datos, separada de los microservicios, para almacenar el resultado del proceso ETL (el Vector de Características del Usuario).

*   **Propósito:** Servir como fuente de datos limpia, unificada, vectorizada y normalizada para el modelo de Clustering.
*   **DBMS:** PostgreSQL (mismo contenedor `aura_postgres` o instancia dedicada)

### Tabla `analitica.user_feature_vector`

| Campo | Tipo de Dato | Descripción | Clave/Uso |
| :--- | :--- | :--- | :--- |
| `id` | `SERIAL / BIGINT` | Clave primaria autoincremental para el registro. | PK |
| `user_id_raiz` | `UUID` | Identificador único y anónimo del usuario (del Auth Service). | FK Lógica / Índice |
| `extraction_date` | `DATE` | Fecha de la ejecución del proceso ETL. | Partición/Índice |
| `reciprocity_ratio_norm` | `FLOAT` | **KPI 1:** Ratio normalizado. | Feature Vector |
| `days_since_last_seen_norm` | `FLOAT` | **KPI 2:** Días desde la última conexión (normalizado). | Feature Vector |
| `ratio_night_messages` | `FLOAT` | **KPI 3:** Ratio de actividad nocturna. | Feature Vector |
| `is_profile_incomplete` | `BOOLEAN` | **KPI 4:** Índice de apatía (Boolean 0/1). | Feature Vector |
| `sentiment_negativity_index` | `FLOAT` | **KPI 5:** Índice de negatividad (NLP). | Feature Vector |
| `num_community_categories` | `INTEGER` | **KPI 6:** Densidad de categorías. | Feature Vector |
| `int_gaming_one_hot` | `BOOLEAN` | Variable binaria: ¿Tiene interés 'Gaming'? (Ejemplo de vectorización) | Feature Vector |
| `comm_voluntariado_one_hot` | `BOOLEAN` | Variable binaria: Pertenece a comunidad 'Voluntariado'. | Feature Vector |
| `cluster_label` | `VARCHAR` | Espacio reservado para almacenar el resultado (Label) del algoritmo de clustering (ej. Cluster A, Alto Riesgo). | Resultado |

---

## 🧠 3. Análisis de Sentimiento con NLP (Implementación Python)

Para el cálculo del **KPI 5**, se utilizarán modelos Transformer pre-entrenados en español. Se sugiere la implementación de `UMUTeam/roberta-spanish-sentiment-analysis` o `dccuchile/bert-base-spanish-wwm-uncased`.

### Flujo de Trabajo Sugerido

1.  **Extracción SQL:** Obtener textos crudos (posts, comentarios, mensajes) de la BD.
2.  **Procesamiento Python:** Ejecutar inferencia de sentimiento.
3.  **Agregación:** Calcular promedio de negatividad por usuario.
4.  **Persistencia:** Guardar resultado en `user_feature_vector`.

### Ejemplo de Implementación (Python)

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification, pipeline
import torch
import pandas as pd

# Configuración del Modelo (Sugerencia: RoBERTa Spanish Sentiment)
model_name = "UMUTeam/roberta-spanish-sentiment-analysis"
# Alternativa: "dccuchile/bert-base-spanish-wwm-uncased" (requiere fine-tuning para sentiment)

print(f"🔄 Cargando modelo NLP: {model_name}...")
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

# Crear pipeline de clasificación
sentiment_pipeline = pipeline("sentiment-analysis", model=model, tokenizer=tokenizer, return_all_scores=True)

def calculate_negativity_index(texts):
    """
    Calcula el índice de negatividad promedio para una lista de textos.
    Retorna un valor entre 0.0 y 1.0.
    """
    if not texts or len(texts) == 0:
        return 0.0
    
    negativity_scores = []
    
    # Procesamiento por lotes para eficiencia
    for text in texts:
        if not text or len(text.strip()) < 3: continue
        
        # Truncar textos muy largos (límite de tokens del modelo)
        text = text[:512] 
        
        try:
            result = sentiment_pipeline(text)[0]
            # Buscar score de la etiqueta 'NEG' o equivalente
            neg_score = next((item['score'] for item in result if item['label'] in ['NEG', 'NEGATIVE']), 0.0)
            negativity_scores.append(neg_score)
        except Exception as e:
            print(f"⚠️ Error procesando texto: {e}")
            
    if not negativity_scores:
        return 0.0
        
    return sum(negativity_scores) / len(negativity_scores)

# Ejemplo de uso con DataFrame extraído de SQL
# df_texts = pd.read_sql("SELECT user_id, content FROM ...", db_connection)
# user_negativity = df_texts.groupby('user_id')['content'].apply(list).apply(calculate_negativity_index)
```

---

## 🛠️ 4. Instructivo SQL para la Extracción (EDA)

El siguiente conjunto de instrucciones SQL utiliza **Common Table Expressions (CTEs)** para unificar y calcular los KPIs desde las bases de datos de origen (`aura_auth`, `aura_social`, `aura_messaging`).

> **Nota:** Se asume el uso de Foreign Data Wrappers o una conexión única que acceda a los diferentes esquemas/bases de datos.

```sql
-- 1. CTE: Definición de la base de usuarios y Unificación de IDs
WITH UserBase AS (
    SELECT
        A.user_id AS user_id_raiz,
        S.id AS profile_id_social,
        M.id AS profile_id_messaging,
        S.followers_count, -- KPI 1: Followers
        S.following_count, -- KPI 1: Following
        M.last_seen_at, -- KPI 2: Last Seen At
        S.bio IS NULL AS profile_bio_missing, -- KPI 4: Bio NULL
        CP.user_id IS NULL AS complete_profile_missing -- KPI 4: CompleteProfile NULL
    FROM
        aura_auth.users A
    LEFT JOIN
        aura_social.user_profiles S ON A.user_id = S.user_id -- Relación Auth -> Social
    LEFT JOIN
        aura_messaging.users M ON S.id = M.profile_id -- Relación Social -> Messaging
    LEFT JOIN
        aura_social.complete_profiles CP ON S.id = CP.user_id -- Existencia de Complete Profile
),

-- 2. CTE: Métricas de Mensajería (KPI 2 y KPI 3)
MessagingMetrics AS (
    SELECT
        sender_profile_id AS profile_id_messaging,
        COUNT(id) AS total_messages_sent,
        -- KPI 3: Conteo de mensajes nocturnos (1 AM a 5 AM)
        SUM(CASE
            WHEN EXTRACT(HOUR FROM created_at) BETWEEN 1 AND 5 THEN 1
            ELSE 0
        END) AS messages_night_count
    FROM
        aura_messaging.messages
    GROUP BY 1
),

-- 3. CTE: Extracción de Textos para NLP (Preparación para Python)
-- Nota: Esta CTE no calcula el KPI directamente, sino que prepara los datos.
-- El cálculo real del KPI 5 se hace en Python. Aquí solo contamos volumen para referencia.
TextContentMetrics AS (
    SELECT
        user_id AS profile_id_social,
        COUNT(id) AS total_text_items
    FROM (
        SELECT user_id, id FROM aura_social.posts WHERE content IS NOT NULL
        UNION ALL
        SELECT user_id, id FROM aura_social.comments WHERE content IS NOT NULL
    ) AllTexts
    GROUP BY 1
),

-- 4. CTE: Métricas Comunitarias (KPI 6)
CommunityMetrics AS (
    SELECT
        CM.user_id AS profile_id_social,
        -- KPI 6: Conteo de categorías únicas de comunidad
        COUNT(DISTINCT C.category) AS distinct_community_categories,
        -- Ejemplo de Vectorización: ¿Pertenece a una comunidad de 'Voluntariado'?
        MAX(CASE WHEN C.category = 'Voluntariado' THEN 1 ELSE 0 END) AS is_in_voluntariado
    FROM
        aura_social.community_members CM
    INNER JOIN
        aura_social.communities C ON CM.community_id = C.id
    GROUP BY 1
)

-- SELECT FINAL: Construcción del Vector de Características
SELECT
    UB.user_id_raiz,
    UB.profile_id_social,

    -- KPI 1: Ratio de Reciprocidad Social
    CASE
        WHEN UB.followers_count = 0 THEN 0 -- Asumir 0 si no hay seguidores (o usar 1 para evitar infinito, dependiendo del modelo)
        ELSE UB.following_count::NUMERIC / UB.followers_count
    END AS reciprocity_ratio,

    -- KPI 2: Días desde Última Conexión
    EXTRACT(EPOCH FROM (NOW() - UB.last_seen_at)) / (60 * 60 * 24) AS days_since_last_seen,

    -- KPI 3: Ratio de Mensajes Nocturnos
    CASE
        WHEN MM.total_messages_sent = 0 THEN 0
        ELSE MM.messages_night_count::NUMERIC / MM.total_messages_sent
    END AS ratio_night_messages,
    
    -- KPI 4: Índice de Apatía del Perfil
    (UB.profile_bio_missing AND UB.complete_profile_missing) AS is_profile_incomplete_kpi,

    -- KPI 5: Frecuencia de Sentimiento Negativo (Placeholder para valor calculado en Python)
    -- En SQL solo podemos dejar el espacio o poner NULL, ya que requiere procesamiento externo.
    NULL::FLOAT AS sentiment_negativity_index,

    -- KPI 6: Densidad de Participación Comunitaria
    COALESCE(CM.distinct_community_categories, 0) AS num_community_categories,
    
    -- Características de Vectorización (Ejemplos)
    COALESCE(CM.is_in_voluntariado, 0) AS comm_voluntariado_one_hot
    
FROM
    UserBase UB
LEFT JOIN
    MessagingMetrics MM ON UB.profile_id_messaging = MM.profile_id_messaging
LEFT JOIN
    TextContentMetrics TM ON UB.profile_id_social = TM.profile_id_social
LEFT JOIN
    CommunityMetrics CM ON UB.profile_id_social = CM.profile_id_social
ORDER BY
    days_since_last_seen DESC;
```
