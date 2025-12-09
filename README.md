# Proyecto de Minería de Datos

Este repositorio contiene varios proyectos y servicios relacionados con la minería de datos y el análisis de datos.

## Estructura de Directorios

A continuación se describe el contenido de cada directorio principal dentro de este proyecto.

### 📂 `clustering-service-aura/`

Este directorio contiene un microservicio desarrollado con FastAPI para realizar tareas de clustering. A juzgar por sus dependencias, el servicio incluye funcionalidades para:

*   **API Web:** Exposición de endpoints utilizando `FastAPI`.
*   **Procesamiento de Datos:** Uso de `pandas` y `numpy` para la manipulación de datos y `scikit-learn` para los algoritmos de clustering.
*   **Análisis de Lenguaje Natural (NLP):** Integración de modelos de `transformers` y `torch` para el procesamiento de texto.
*   **Base de Datos:** Interacción con una base de datos PostgreSQL (`psycopg2-binary`) a través de `SQLAlchemy`, con manejo de migraciones mediante `Alembic`.

### 📂 `doc/`

Este directorio está destinado a contener toda la documentación relevante del proyecto. Esto puede incluir diagramas de arquitectura, especificaciones de la API (como archivos OpenAPI/Swagger), guías de usuario y cualquier otro documento que ayude a entender el funcionamiento y diseño de los servicios.