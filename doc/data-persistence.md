# 📊 Documentación de Persistencia de Datos - Plataforma AURA

> **Fecha:** 2025-12-09  
> **Versión:** V5 + Real-Time v2.0

---

## 📑 Tabla de Contenidos

1. [Arquitectura General](#-arquitectura-general)
2. [Infraestructura de Base de Datos](#-infraestructura-de-base-de-datos)
3. [Credenciales de Acceso](#-credenciales-de-acceso)
4. [Servicio de Autenticación](#-servicio-de-autenticación)
5. [Servicio de Mensajería](#-servicio-de-mensajería)
6. [Servicio Social](#-servicio-social)
7. [Servicio de Clustering (Data Miner)](#-servicio-de-clustering-data-miner)
8. [Sistema en Tiempo Real (CDC)](#-sistema-en-tiempo-real-cdc)
9. [Broker de Mensajes (RabbitMQ)](#-broker-de-mensajes-rabbitmq)
10. [Almacenamiento de Medios (Cloudinary)](#-almacenamiento-de-medios-cloudinary)
11. [Diagrama de Flujo de Datos](#-diagrama-de-flujo-de-datos)

---

## 🏗️ Arquitectura General

La plataforma AURA implementa una arquitectura de **microservicios**:

| Componente | Puerto | Base de Datos | ORM |
|:-----------|:------:|:--------------|:----|
| Gateway Service | 3000 | - | - |
| Auth Service | 3001 | aura_auth | Prisma |
| Social Service | 3002 | aura_social | Sequelize |
| Messaging Service | 3003 | aura_messaging | Sequelize |
| Notifications Service | 3004 | aura_notifications | Prisma |
| **Clustering Service** | **8001** | **aura_data_miner** | **SQLAlchemy** |

---

## 🗄️ Infraestructura de Base de Datos

### PostgreSQL (Versión 15-alpine)

```yaml
Container: aura_postgres
Image: postgres:15-alpine
Ports: 5432:5432
Volume: postgres_data:/var/lib/postgresql/data
```

**Bases de datos:**
- `aura_auth` - Autenticación y usuarios
- `aura_messaging` - Sistema de chat y grupos
- `aura_notifications` - Tokens FCM
- `aura_social` - Red social (perfiles, posts, comunidades)
- `aura_data_miner` - **Base analítica de clustering**

**Extensiones habilitadas:**
- `uuid-ossp` - Generación de UUIDs
- `pg_trgm` - Búsquedas por similitud

---

## 🔐 Credenciales de Acceso

### PostgreSQL

| Parámetro | Valor |
|:----------|:------|
| Host | `db` (Docker) / `localhost` (dev) |
| Puerto | `5432` |
| Usuario | `postgres` |
| Contraseña | `postgres` |

### URLs de Conexión

| Servicio | URL |
|:---------|:----|
| Auth | `postgresql://postgres:postgres@db:5432/aura_auth` |
| Social | `postgresql://postgres:postgres@db:5432/aura_social` |
| Messaging | `postgresql://postgres:postgres@db:5432/aura_messaging` |
| Clustering | `postgresql://postgres:postgres@db:5432/aura_data_miner` |

---

## 🔑 Servicio de Autenticación

**ORM:** Prisma | **BD:** `aura_auth`

### Tabla: `users`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `user_id` | UUID | PK, Identificador único |
| `username` | VARCHAR(100) | Nombre de usuario |
| `email` | VARCHAR(100) | Correo electrónico |
| `password_hash` | VARCHAR(255) | Hash bcrypt |
| `id_role` | Int | FK → roles (1=admin, 2=user) |
| `created_at` | DateTime | Fecha de registro |

---

## 💬 Servicio de Mensajería

**ORM:** Sequelize | **BD:** `aura_messaging`

### Tabla: `users`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | UUID | PK |
| `profile_id` | UUID | Referencia a social-service |
| `username` | VARCHAR(100) | Nombre de usuario |
| `is_online` | Boolean | Estado de conexión |
| `last_seen_at` | DateTime | **KPI 2: Última actividad** |

### Tabla: `messages`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | UUID | PK |
| `sender_profile_id` | UUID | Remitente |
| `content` | TEXT | Contenido del mensaje |
| `created_at` | DateTime | **KPI 3: Hora de envío** |
| `is_deleted` | Boolean | Soft delete |

---

## 🌐 Servicio Social

**ORM:** Sequelize | **BD:** `aura_social`

### Tabla: `user_profiles`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | UUID | PK del perfil social |
| `user_id` | UUID | FK → auth-service |
| `display_name` | VARCHAR(100) | Nombre visible |
| `bio` | VARCHAR(500) | **KPI 4: Bio (apatía si NULL)** |
| `followers_count` | Integer | **KPI 1: Seguidores** |
| `following_count` | Integer | **KPI 1: Siguiendo** |
| `is_active` | Boolean | Perfil activo |

### Tabla: `posts`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | UUID | PK |
| `user_id` | UUID | Autor |
| `content` | TEXT | **KPI 5: Texto para NLP** |
| `is_active` | Boolean | Post activo |

### Tabla: `communities`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `id` | UUID | PK |
| `name` | VARCHAR(100) | Nombre de la comunidad |
| `category` | ENUM | **KPI 6: Categoría** |

### Tabla: `community_members`

| Campo | Tipo | Descripción |
|:------|:-----|:------------|
| `community_id` | UUID | FK → communities |
| `user_id` | UUID | FK → user_profiles |
| `role` | ENUM | Rol del miembro |

---

## 🔮 Servicio de Clustering (Data Miner)

**ORM:** SQLAlchemy | **BD:** `aura_data_miner` | **Puerto:** 8001

### Tabla: `user_feature_vector`

| Campo | Tipo | KPI | Descripción |
|:------|:-----|:---:|:------------|
| `id` | SERIAL | - | PK |
| `user_id_raiz` | UUID | - | ID usuario (Auth) |
| `extraction_date` | TIMESTAMP | - | Fecha del ETL |
| `reciprocity_ratio_norm` | FLOAT | 1 | Ratio seguidores/seguidos |
| `days_since_last_seen_norm` | FLOAT | 2 | Días inactivo |
| `ratio_night_messages` | FLOAT | 3 | % mensajes nocturnos |
| `is_profile_incomplete` | BOOLEAN | 4 | Perfil vacío |
| `sentiment_negativity_index` | FLOAT | 5 | Negatividad NLP |
| `num_community_categories_norm` | FLOAT | 6 | Categorías de comunidad |
| `cluster_label` | VARCHAR | - | ALTO_RIESGO / MODERADO / BAJO |

---

## 🔄 Sistema en Tiempo Real (CDC)

### Arquitectura Change Data Capture

El Clustering Service v2.0 implementa CDC usando PostgreSQL `LISTEN/NOTIFY`:

```mermaid
graph LR
    subgraph "Microservicios"
        MSG[(aura_messaging)]
        SOC[(aura_social)]
    end
    
    subgraph "Triggers"
        MSG --> |INSERT/UPDATE| T1[pg_notify]
        SOC --> |INSERT/UPDATE| T2[pg_notify]
    end
    
    subgraph "Clustering Service"
        T1 --> L[DB Listener]
        T2 --> L
        L --> P[Streaming Pipeline]
        P --> WS[WebSocket Manager]
    end
    
    subgraph "Clientes"
        WS --> R[React Dashboard]
    end
```

### Triggers SQL Instalados

| Base de Datos | Tablas Monitoreadas |
|:--------------|:--------------------|
| `aura_messaging` | `messages`, `users` |
| `aura_social` | `posts`, `comments`, `user_profiles`, `community_members` |

### Flujo de Tiempo Real

1. Usuario envía mensaje/post
2. Trigger PostgreSQL ejecuta `pg_notify('aura_data_change', ...)`
3. DatabaseListener captura la notificación
4. StreamingPipeline recalcula KPIs del usuario
5. WebSocketManager broadcast a clientes conectados

### Endpoints WebSocket

| Endpoint | Canal | Uso |
|:---------|:------|:----|
| `/api/v2/clustering/ws/live` | clustering | Actualizaciones generales |
| `/api/v2/clustering/ws/alerts` | alerts | Alertas críticas |

---

## 🐰 Broker de Mensajes (RabbitMQ)

**Versión:** 3-management-alpine

| Parámetro | Valor |
|:----------|:------|
| URL AMQP | `amqp://guest:guest@rabbitmq:5672` |
| Management | `http://localhost:15672` |

### Configuración de Notificaciones

| Elemento | Nombre |
|:---------|:-------|
| Exchange | `notifications_exchange` |
| Queue | `notifications_queue` |
| Routing Key | `notifications_routing_key` |

---

## ☁️ Almacenamiento de Medios (Cloudinary)

| Parámetro | Valor |
|:----------|:------|
| Cloud Name | `derzzspjo` |
| API Key | `967388682229272` |
| Max File Size | 100 MB |
| Max Files | 10 por request |

---

## 📊 Diagrama de Flujo de Datos

```mermaid
flowchart TB
    subgraph Frontend["🖥️ Frontend"]
        Cliente[Cliente React/Mobile]
    end
    
    subgraph Gateway["🌐 Gateway :3000"]
        GW[API Gateway]
    end
    
    subgraph Services["Microservicios"]
        Auth["🔐 Auth :3001"]
        Social["🌐 Social :3002"]
        Msg["💬 Messaging :3003"]
    end
    
    subgraph Clustering["🔮 Clustering :8001"]
        ETL[ETL Pipeline]
        Ensemble[Modelo Ensemble]
        RT[Real-Time System]
    end
    
    subgraph Databases["🗄️ PostgreSQL"]
        AuthDB[(aura_auth)]
        SocialDB[(aura_social)]
        MsgDB[(aura_messaging)]
        MinerDB[(aura_data_miner)]
    end
    
    Cliente --> GW
    GW --> Auth --> AuthDB
    GW --> Social --> SocialDB
    GW --> Msg --> MsgDB
    
    AuthDB -.-> ETL
    SocialDB -.-> ETL
    MsgDB -.-> ETL
    
    SocialDB -.-> |pg_notify| RT
    MsgDB -.-> |pg_notify| RT
    
    ETL --> MinerDB
    MinerDB --> Ensemble
    
    RT --> |WebSocket| Cliente
```

---

## 🔗 Relaciones entre Servicios

```
┌──────────────────┐
│   Auth Service   │
│   users.user_id  │ ◄───── Fuente primaria de identidad
└────────┬─────────┘
         │
         ├──────────────────────────────────┐
         ▼                                  ▼
┌────────────────────┐           ┌────────────────────────┐
│   Social Service   │           │   Messaging Service    │
│ user_profiles.user_id │        │   users.profile_id     │
└────────┬───────────┘           └────────────┬───────────┘
         │                                    │
         └────────────────┬───────────────────┘
                          ▼
              ┌────────────────────────┐
              │   Clustering Service   │
              │  user_feature_vector   │
              │  (datos consolidados)  │
              └────────────────────────┘
```

---

## 📝 Notas Importantes

1. **UUIDs:** Todos los IDs principales usan UUIDs v4
2. **Soft Delete:** Campo `is_active` en lugar de eliminación física
3. **Tiempo Real:** CDC vía `pg_notify` para actualizaciones inmediatas
4. **JWT:** Todos los servicios comparten `JWT_SECRET`
5. **Medios:** Archivos en Cloudinary, solo URLs en BD

---

*Documento actualizado para AURA Platform v5 + Clustering Service v2.0*
