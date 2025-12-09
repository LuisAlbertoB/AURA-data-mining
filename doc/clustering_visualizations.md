# 📊 Guía de Interpretación: Visualizaciones de Clustering AURA

> **Fecha:** 2025-12-09  
> **Servicio:** AURA Data Miner & Clustering Service v2.0  
> **Contexto:** Interpretación de gráficos para detección de riesgos psicoemocionales

---

Este documento explica la funcionalidad, interpretación y utilidad clínica de las visualizaciones generadas por el servicio de clustering. Incluye tanto los gráficos SVG estáticos (API v1) como los datos en tiempo real para ChartJS (API v2).

---

## 📑 Índice

1. [Dashboard General](#1-dashboard-general)
2. [Proyección PCA de Riesgo](#2-proyección-pca-de-riesgo)
3. [Distribución de Riesgo](#3-distribución-de-riesgo)
4. [Perfil de Clusters (Radar)](#4-perfil-de-clusters-radar-chart)
5. [Histograma de Severidad](#5-histograma-de-severidad)
6. [Métricas de Calidad](#6-métricas-de-calidad)
7. [API v2: Datos para ChartJS](#7-api-v2-datos-para-chartjs)
8. [WebSocket: Actualizaciones en Tiempo Real](#8-websocket-actualizaciones-en-tiempo-real)

---

## 1. Dashboard General

**Endpoint:** `GET /api/v1/clustering/visualize/dashboard`

El dashboard consolida las métricas clave y las 4 visualizaciones principales en una sola vista. Es el punto de partida recomendado para evaluar el estado general de la población de usuarios.

### Métricas Clave

| Métrica | Descripción |
|:--------|:------------|
| **Total Usuarios** | Número de usuarios analizados en la última ejecución |
| **Alto Riesgo (%)** | Porcentaje con clasificación ALTO RIESGO (prioridad de intervención) |
| **Silhouette Score** | Calidad técnica del clustering (cercano a 1 = grupos bien definidos) |

---

## 2. Proyección PCA de Riesgo

**Endpoints:**
- SVG: `GET /api/v1/clustering/visualize/scatter`
- JSON: `GET /api/v2/clustering/data/scatter`

### ¿Qué muestra?

Gráfico de dispersión donde cada punto representa un usuario. Usa **PCA (Análisis de Componentes Principales)** para reducir los 6 KPIs a 2 dimensiones visualizables.

### Interpretación

| Color | Nivel de Riesgo |
|:------|:----------------|
| 🔴 Rojo | Alto Riesgo - Intervención prioritaria |
| 🟡 Amarillo | Riesgo Moderado - Monitoreo |
| 🟢 Verde | Bajo Riesgo - Normal |

- **Agrupación:** Puntos cercanos tienen comportamientos similares
- **Separación clara:** Indica que el modelo distingue eficazmente los perfiles de riesgo

### Utilidad Clínica

Permite identificar visualmente si los usuarios en riesgo forman un grupo compacto (patrón homogéneo) o si están dispersos (diferentes tipos de riesgo que requieren intervenciones diferenciadas).

---

## 3. Distribución de Riesgo

**Endpoints:**
- SVG: `GET /api/v1/clustering/visualize/distribution`
- JSON: `GET /api/v2/clustering/data/distribution`

### ¿Qué muestra?

Gráfico de barras que cuantifica usuarios por categoría de riesgo.

### Interpretación

- **Altura de barra:** Número absoluto de usuarios
- **Etiqueta:** Conteo y porcentaje del total

### Utilidad Clínica

Proporciona visión rápida de la carga de trabajo para el equipo de intervención.

> ⚠️ **Alerta:** Si el porcentaje de "Alto Riesgo" supera el 20%, puede indicar un evento sistémico o necesidad de ajustar la sensibilidad del modelo.

---

## 4. Perfil de Clusters (Radar Chart)

**Endpoints:**
- SVG: `GET /api/v1/clustering/visualize/radar`
- JSON: `GET /api/v2/clustering/data/radar`

### ¿Qué muestra?

Gráfico de radar comparando el promedio de los 6 KPIs normalizados para cada nivel de riesgo.

### Ejes del Radar

| KPI | Descripción | Alto valor indica... |
|:----|:------------|:---------------------|
| Reciprocidad | Balance de relaciones sociales | Mayor conexión social |
| Inactividad | Días sin conectarse | Mayor retiro |
| Mensajes Nocturnos | Actividad 1-5 AM | Desorden circadiano |
| Perfil Incompleto | Datos de perfil vacíos | Apatía |
| Negatividad NLP | Sentimiento en textos | Crisis emocional |
| Comunidades | Categorías de participación | Red de apoyo amplia |

### Utilidad Clínica

Es la herramienta más potente para **diagnóstico diferencial**. Permite entender *por qué* un grupo es considerado de riesgo:

- **Ejemplo A:** Cluster de riesgo con alta Negatividad → Depresión activa
- **Ejemplo B:** Cluster de riesgo con alta Inactividad + bajo Comunidades → Retiro pasivo/aislamiento

---

## 5. Histograma de Severidad

**Endpoints:**
- SVG: `GET /api/v1/clustering/visualize/severity`
- JSON: `GET /api/v2/clustering/data/severity-histogram`

### ¿Qué muestra?

Distribución del **Índice de Severidad de Anomalía (ASI)**, un puntaje continuo de 0 a 100.

### Interpretación de Colores

| Rango | Color | Significado |
|:------|:------|:------------|
| 0-30 | 🟢 Verde | Normal |
| 30-60 | 🟡 Amarillo | Monitoreo |
| 60-100 | 🔴 Rojo | Atención urgente |

### Utilidad Clínica

Ayuda a **priorizar dentro del grupo de Alto Riesgo**. Los usuarios con ASI > 80 son los casos más anómalos y requieren atención inmediata.

---

## 6. Métricas de Calidad

**Endpoint:** `GET /api/v1/clustering/visualize/metrics`

### Métricas Técnicas

| Métrica | Descripción | Valor saludable |
|:--------|:------------|:----------------|
| **Silhouette Score** | Similitud intra-cluster vs inter-cluster | ≥ 0.3 |
| **Calinski-Harabasz** | Dispersión entre/dentro de clusters | Mayor = mejor |

### Utilidad Técnica

Sirve para que el equipo de Data Science monitoree la salud del modelo. Una caída drástica sugiere que los patrones de comportamiento han cambiado y el modelo necesita re-entrenamiento.

---

## 7. API v2: Datos para ChartJS

La API v2 proporciona datos estructurados en formato JSON directamente compatible con Chart.js para integración con dashboards React.

### Endpoints Disponibles

| Endpoint | Tipo de Gráfico | Descripción |
|:---------|:----------------|:------------|
| `/api/v2/clustering/data/distribution` | Bar/Pie | Conteo por nivel de riesgo |
| `/api/v2/clustering/data/scatter` | Scatter | Proyección PCA de usuarios |
| `/api/v2/clustering/data/radar` | Radar | Perfil de KPIs por cluster |
| `/api/v2/clustering/data/severity-histogram` | Bar | Distribución de ASI |
| `/api/v2/clustering/data/kpi-trends?hours=24` | Line | Tendencias temporales |
| `/api/v2/clustering/data/high-risk-users` | Table | Lista de usuarios prioritarios |

### Ejemplo de Respuesta (Distribution)

```json
{
  "labels": ["BAJO RIESGO", "RIESGO MODERADO", "ALTO RIESGO"],
  "datasets": [{
    "data": [150, 45, 12],
    "backgroundColor": ["#10B981", "#F59E0B", "#EF4444"]
  }],
  "meta": {
    "total_users": 207,
    "high_risk_percentage": 5.8,
    "last_updated": "2025-12-09T14:30:00Z"
  }
}
```

---

## 8. WebSocket: Actualizaciones en Tiempo Real

### Conexión

```javascript
const ws = new WebSocket('ws://localhost:8001/api/v2/clustering/ws/live');
```

### Canales Disponibles

| Canal | Endpoint | Uso |
|:------|:---------|:----|
| `clustering` | `/ws/live` | Actualizaciones generales |
| `alerts` | `/ws/alerts` | Solo alertas críticas |

### Tipos de Mensajes

| Tipo | Descripción | Acción sugerida |
|:-----|:------------|:----------------|
| `INITIAL_STATE` | Estado al conectar | Inicializar dashboard |
| `USER_RISK_UPDATE` | Cambio de riesgo de un usuario | Actualizar gráficas |
| `CRITICAL_ALERT` | Usuario pasó a Alto Riesgo | Mostrar notificación urgente |
| `DISTRIBUTION_UPDATE` | Cambio en distribución general | Refrescar bar chart |

### Ejemplo de Mensaje CRITICAL_ALERT

```json
{
  "type": "CRITICAL_ALERT",
  "payload": {
    "priority": "URGENT",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "severity_index": 85.2,
    "factors": [
      "Inactividad de 14 días",
      "Ratio nocturno 67%",
      "Índice negatividad NLP 0.82"
    ],
    "suggested_action": "Contacto prioritario por profesional de salud mental"
  },
  "server_timestamp": "2025-12-09T14:32:15Z"
}
```

---

## 📋 Resumen de Decisiones

| Situación | Visualización recomendada |
|:----------|:--------------------------|
| Evaluación general rápida | Dashboard |
| Ver carga de trabajo | Distribución de Riesgo |
| Entender *por qué* hay riesgo | Radar Chart |
| Priorizar casos urgentes | Histograma de Severidad |
| Monitoreo continuo | WebSocket + ChartJS |
| Análisis de tendencias | KPI Trends (Line Chart) |

---

*Documento actualizado para AURA Clustering Service v2.0 - Sistema de Detección de Riesgos Psicoemocionales*
