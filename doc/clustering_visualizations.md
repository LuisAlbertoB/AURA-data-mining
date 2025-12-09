# Guía de Interpretación: Visualizaciones de Clustering AURA

> **Fecha:** 2025-12-05  
> **Servicio:** AURA Data Miner & Clustering Service  
> **Contexto:** Interpretación de gráficos generados por el API de Clustering para la detección de riesgos.

---

Este documento explica la funcionalidad, interpretación y utilidad clínica de cada una de las visualizaciones generadas por el servicio de clustering. Estas gráficas están diseñadas para facilitar la toma de decisiones por parte del equipo de psicólogos y administradores de la plataforma.

## 1. Dashboard General (`/visualize/dashboard`)

El dashboard consolida las métricas clave y las 4 visualizaciones principales en una sola vista. Es el punto de partida recomendado para evaluar el estado general de la población de usuarios.

**Métricas Clave:**
*   **Total Usuarios:** Número de usuarios analizados en la última ejecución.
*   **Alto Riesgo:** Porcentaje de usuarios clasificados como "ALTO RIESGO" (prioridad de intervención).
*   **Silhouette Score:** Calidad técnica del clustering (valores cercanos a 1 indican grupos bien definidos).

---

## 2. Proyección PCA de Riesgo (`/visualize/scatter`)

### ¿Qué muestra?
Un gráfico de dispersión (Scatter Plot) donde cada punto representa un usuario. Dado que el análisis utiliza 6 dimensiones (KPIs), se utiliza **PCA (Análisis de Componentes Principales)** para reducir la complejidad a 2 dimensiones (Eje X e Y) y permitir su visualización en un plano.

### Interpretación
*   **Colores:** Indican el nivel de riesgo asignado por el ensamble de modelos.
    *   🔴 **Rojo:** Alto Riesgo
    *   🟡 **Amarillo:** Riesgo Moderado
    *   🟢 **Verde:** Bajo Riesgo
*   **Agrupación:** Los puntos cercanos entre sí tienen comportamientos similares.
*   **Separación:** Si los puntos rojos están claramente separados de los verdes, el modelo está distinguiendo eficazmente los perfiles de riesgo.

### Utilidad Clínica
Permite identificar visualmente si los usuarios en riesgo forman un grupo compacto (patrón de riesgo homogéneo) o si están dispersos (diferentes tipos de riesgo).

---

## 3. Distribución de Riesgo (`/visualize/distribution`)

### ¿Qué muestra?
Un gráfico de barras que cuantifica cuántos usuarios caen en cada categoría de riesgo.

### Interpretación
*   **Altura de la barra:** Número absoluto de usuarios.
*   **Etiqueta:** Muestra el conteo y el porcentaje del total.

### Utilidad Clínica
Proporciona una visión rápida de la carga de trabajo para el equipo de intervención. Si el porcentaje de "Alto Riesgo" es inusualmente alto (ej. >20%), puede indicar un evento sistémico o la necesidad de ajustar la sensibilidad del modelo.

---

## 4. Perfil de Clusters (Radar Chart) (`/visualize/radar`)

### ¿Qué muestra?
Un gráfico de radar (o araña) que compara el promedio de los 6 KPIs normalizados para cada cluster identificado por K-Means.

### Interpretación
*   **Ejes:** Cada radio representa un KPI (ej. Reciprocidad, Negatividad, Inactividad). Escala de 0 a 1.
*   **Polígonos:** Cada línea de color representa un cluster diferente.
*   **Cluster de Riesgo (⚠️):** El cluster identificado automáticamente como riesgoso suele mostrar valores altos en KPIs negativos (Inactividad, Negatividad) y bajos en positivos (Reciprocidad, Comunidad).

### Utilidad Clínica
Es la herramienta más potente para el **diagnóstico diferencial**. Permite entender *por qué* un grupo es considerado de riesgo.
*   *Ejemplo:* Un cluster puede ser de riesgo por "Alta Negatividad" (depresión activa), mientras que otro puede serlo por "Alta Inactividad y Aislamiento" (retiro pasivo).

---

## 5. Histograma de Severidad (`/visualize/severity`)

### ¿Qué muestra?
La distribución de frecuencias del **Índice de Severidad de Anomalía (ASI)**, un puntaje continuo de 0 a 100.

### Interpretación
*   **Eje X:** Índice de Severidad (0 = Normal, 100 = Crítico).
*   **Colores:** Las barras cambian de color según los umbrales de riesgo (Verde < 30, Amarillo 30-60, Rojo > 60).
*   **Forma:** Una distribución cargada a la izquierda es saludable. Una "cola larga" hacia la derecha indica la presencia de casos extremos.

### Utilidad Clínica
Ayuda a priorizar dentro del grupo de "Alto Riesgo". Los usuarios en el extremo derecho del histograma (ASI > 80) son los casos más anómalos y urgentes, que requieren atención inmediata.

---

## 6. Métricas de Calidad (`/visualize/metrics`)

### ¿Qué muestra?
Un resumen técnico de la calidad del agrupamiento.

*   **Silhouette Score:** Mide qué tan similar es un objeto a su propio cluster en comparación con otros clusters.
*   **Calinski-Harabasz:** Mide la dispersión entre clusters y dentro de los clusters.

### Utilidad Técnica
Sirve para que el equipo de Data Science monitoree la salud del modelo. Una caída drástica en estas métricas sugiere que los patrones de comportamiento de los usuarios han cambiado y el modelo necesita re-entrenamiento.
