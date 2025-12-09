# 📊 Documentación de Persistencia de Datos - Plataforma AURA

> **Fecha de generación:** 2025-12-05  
> **Versión:** V5  
> **Autor:** Análisis automatizado del sistema

---

## 📑 Tabla de Contenidos

1. [Arquitectura General](#arquitectura-general)
2. [Infraestructura de Base de Datos](#infraestructura-de-base-de-datos)
3. [Credenciales de Acceso](#credenciales-de-acceso)
4. [Servicio de Autenticación (auth-service-aura)](#servicio-de-autenticación-auth-service-aura)
5. [Servicio de Mensajería (messaging-service-aura)](#servicio-de-mensajería-messaging-service-aura)
6. [Servicio de Notificaciones (notifications-service-aura)](#servicio-de-notificaciones-notifications-service-aura)
7. [Servicio Social (social-service-aura)](#servicio-social-social-service-aura)
8. [Broker de Mensajes (RabbitMQ)](#broker-de-mensajes-rabbitmq)
9. [Almacenamiento de Medios (Cloudinary)](#almacenamiento-de-medios-cloudinary)
10. [Diagrama de Flujo de Datos](#diagrama-de-flujo-de-datos)

---

## 🏗️ Arquitectura General

La plataforma AURA implementa una arquitectura de **microservicios** con los siguientes componentes principales:

| Componente | Puerto | Base de Datos | ORM |
|------------|--------|---------------|-----|
| Gateway Service | 3000 | - | - |
| Auth Service | 3001 | aura_auth | Prisma |
| Social Service | 3002 | aura_social | Sequelize |
| Messaging Service | 3003 | aura_messaging | Sequelize |
| Notifications Service | 3004 | aura_notifications | Prisma |

---

## 🗄️ Infraestructura de Base de Datos

### PostgreSQL (Versión 15-alpine)

**Configuración del contenedor:**
```yaml
Container: aura_postgres
Image: postgres:15-alpine
Ports: 5432:5432
Volume: postgres_data:/var/lib/postgresql/data
```

**Bases de datos creadas:**
- `aura_auth` - Autenticación y usuarios
- `aura_messaging` - Sistema de chat y grupos
- `aura_notifications` - Tokens de dispositivos FCM
- `aura_social` - Red social (perfiles, posts, comunidades)

**Extensiones habilitadas:**
- `uuid-ossp` - Generación de UUIDs
- `pg_trgm` - Búsquedas por similitud de texto

### 🐳 Contenedores Docker y Estrategia de Despliegue

**¿La Base de Datos se encuentra centralizada y desacoplada?**

**SÍ**, la arquitectura implementa una estrategia híbrida que combina centralización de infraestructura con desacoplamiento lógico:

1.  **Infraestructura Centralizada (Nivel Contenedor):**
    *   Existe **un único contenedor Docker** (`aura_postgres`) ejecutando la instancia de PostgreSQL.
    *   Esto simplifica la gestión de recursos, backups y mantenimiento a nivel de sistema operativo.
    *   Reduce el overhead de memoria que implicaría tener 4 contenedores de base de datos separados.

2.  **Desacoplamiento Lógico (Nivel Datos):**
    *   Cada microservicio tiene su **propia base de datos independiente** (`aura_auth`, `aura_social`, etc.).
    *   **Aislamiento:** Un servicio no puede acceder a las tablas de otro servicio directamente (aunque compartan la instancia física).
    *   **Independencia de Esquema:** Cada servicio gestiona sus propias migraciones y modelos (Prisma vs Sequelize) sin interferir con los demás.

**Ventajas de esta aproximación:**
*   **Facilidad de Desarrollo:** Se levanta todo el entorno con un solo `docker-compose up`.
*   **Escalabilidad Futura:** Si un servicio crece mucho (ej. `aura_social`), su base de datos puede migrarse fácilmente a una instancia física separada o un servicio gestionado (RDS) sin cambiar el código, solo actualizando la variable de entorno `DATABASE_URL`.


---

## 🔐 Credenciales de Acceso

### PostgreSQL - Producción (Docker Compose)

| Parámetro | Valor |
|-----------|-------|
| **Host** | `db` (contenedor) / `localhost` (desarrollo) |
| **Puerto** | `5432` |
| **Usuario** | `postgres` |
| **Contraseña** | `postgres` |
| **URL de Conexión Auth** | `postgresql://postgres:postgres@db:5432/aura_auth?schema=public` |
| **URL de Conexión Messaging** | `postgresql://postgres:postgres@db:5432/aura_messaging?schema=public` |
| **URL de Conexión Notifications** | `postgresql://postgres:postgres@db:5432/aura_notifications?schema=public` |
| **URL de Conexión Social** | `postgresql://postgres:postgres@db:5432/aura_social` |

### PostgreSQL - Desarrollo Local (Auth Service)

| Parámetro | Valor |
|-----------|-------|
| **Usuario** | `auth_service_user` |
| **Contraseña** | `ctLk3z1XhI5yIK3` |
| **Base de Datos** | `auth_service_db` |

### PostgreSQL - Desarrollo Local (Social Service)

| Parámetro | Valor |
|-----------|-------|
| **Usuario** | `social_service_pg_user` |
| **Contraseña** | `social_service_secure_pass` |
| **Base de Datos** | `social_service_pg_db` |

### RabbitMQ

| Parámetro | Valor Producción | Valor Desarrollo |
|-----------|------------------|------------------|
| **URL** | `amqp://guest:guest@rabbitmq:5672` | `amqp://admin:admin@localhost:5672` |
| **Puerto AMQP** | `5672` | `5672` |
| **Puerto Management** | `15672` | `15672` |
| **Usuario** | `guest` | `admin` |
| **Contraseña** | `guest` | `admin` |

### JWT Secret

| Servicio | Secret |
|----------|--------|
| Todos los servicios | `pezcadofrito.1` |

---

## 🔑 Servicio de Autenticación (auth-service-aura)

**ORM:** Prisma  
**Base de datos:** `aura_auth`  
**Archivo de esquema:** `auth-service-aura/prisma/schema.prisma`

### Tabla: `roles`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id_role` | `Int` | PK, Auto-increment | Identificador único del rol |
| `role_name` | `VARCHAR(50)` | Unique | Nombre del rol (ej: 'admin', 'user') |
| `created_at` | `DateTime` | Default: now() | Fecha de creación del rol |

**Valores predefinidos:**
- `id_role: 1` → `admin`
- `id_role: 2` → `user`

---

### Tabla: `users`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `user_id` | `UUID` | PK, Default: uuid() | Identificador único del usuario |
| `username` | `VARCHAR(100)` | Unique, Not Null | Nombre de usuario único |
| `email` | `VARCHAR(100)` | Unique, Not Null | Correo electrónico del usuario |
| `password_hash` | `VARCHAR(255)` | Not Null | Hash bcrypt de la contraseña |
| `id_role` | `Int` | FK → roles, Default: 2 | Rol asignado al usuario |
| `created_at` | `DateTime` | Default: now() | Fecha de registro |

**Relaciones:**
- `role` → Relación con tabla `roles` (N:1)
- `passwordResets` → Relación con tabla `password_resets` (1:N)
- `deviceTokens` → Relación con tabla `device_tokens` (1:N)

---

### Tabla: `device_tokens`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `Int` | PK, Auto-increment | Identificador único del registro |
| `user_id` | `UUID` | FK → users, Not Null | Usuario propietario del token |
| `token` | `VARCHAR(255)` | Not Null | Token FCM del dispositivo |
| `created_at` | `DateTime` | Default: now() | Fecha de registro del token |

**Restricciones:**
- Índice único compuesto: `[user_id, token]` - Previene tokens duplicados por usuario

---

### Tabla: `password_resets`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `Int` | PK, Auto-increment | Identificador único |
| `user_id` | `UUID` | FK → users, Not Null | Usuario que solicitó el reset |
| `token` | `VARCHAR(64)` | Unique, Not Null | Token único para reset de contraseña |
| `expires_at` | `DateTime` | Not Null | Fecha de expiración del token |
| `created_at` | `DateTime` | Default: now() | Fecha de creación de la solicitud |
| `used` | `Boolean` | Default: false | Indica si el token ya fue utilizado |

---

## 💬 Servicio de Mensajería (messaging-service-aura)

**ORM:** Sequelize  
**Base de datos:** `aura_messaging`  
**Directorio de modelos:** `messaging-service-aura/src/infrastructure/database/models/`

### Tabla: `users`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador interno del usuario en mensajería |
| `profile_id` | `UUID` | Unique, Not Null | ID del perfil completo del usuario (referencia externa a social-service) |
| `username` | `VARCHAR(100)` | Unique, Not Null | Nombre de usuario |
| `display_name` | `VARCHAR(150)` | Nullable | Nombre a mostrar en chats |
| `avatar_url` | `VARCHAR(500)` | Nullable | URL del avatar del usuario |
| `is_online` | `Boolean` | Default: false | Estado de conexión en tiempo real |
| `last_seen_at` | `DateTime` | Nullable | Última vez visto en línea |
| `is_active` | `Boolean` | Default: true | Cuenta activa |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `conversations`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la conversación |
| `participant1_profile_id` | `UUID` | Not Null | ID del perfil del primer participante |
| `participant2_profile_id` | `UUID` | Not Null | ID del perfil del segundo participante |
| `last_message_id` | `UUID` | Nullable, FK → messages | ID del último mensaje enviado |
| `last_message_at` | `DateTime` | Nullable | Fecha del último mensaje |
| `participant1_status` | `ENUM` | Default: 'active' | Estado del participante 1: `active`, `archived`, `blocked` |
| `participant2_status` | `ENUM` | Default: 'active' | Estado del participante 2: `active`, `archived`, `blocked` |
| `unread_count_1` | `Integer` | Default: 0 | Mensajes no leídos por participante 1 |
| `unread_count_2` | `Integer` | Default: 0 | Mensajes no leídos por participante 2 |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `chat_groups`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del grupo |
| `name` | `VARCHAR(200)` | Not Null | Nombre del grupo |
| `description` | `TEXT` | Nullable | Descripción del grupo |
| `image_url` | `VARCHAR(500)` | Nullable | URL de imagen del grupo |
| `group_type` | `ENUM` | Default: 'community' | Tipo de grupo: `community`, `activity`, `private` |
| `creator_profile_id` | `UUID` | Not Null | ID del perfil del creador |
| `external_id` | `UUID` | Nullable | ID externo (referencia a comunidad en social-service) |
| `max_members` | `Integer` | Nullable | Límite máximo de miembros |
| `is_public` | `Boolean` | Default: true | Visibilidad pública del grupo |
| `status` | `ENUM` | Default: 'active' | Estado: `active`, `inactive`, `archived` |
| `settings` | `JSON` | Default: {} | Configuraciones adicionales del grupo |
| `member_count` | `Integer` | Default: 0 | Contador de miembros |
| `last_message_at` | `DateTime` | Nullable | Fecha del último mensaje |
| `location` | `JSON` | Nullable | Ubicación geográfica (para actividades) |
| `scheduled_at` | `DateTime` | Nullable | Fecha programada (para actividades) |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `group_members`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la membresía |
| `group_id` | `UUID` | Not Null, FK → chat_groups | ID del grupo |
| `profile_id` | `UUID` | Not Null | ID del perfil del miembro |
| `role` | `ENUM` | Default: 'member' | Rol: `owner`, `admin`, `moderator`, `member` |
| `status` | `ENUM` | Default: 'active' | Estado: `active`, `muted`, `banned`, `left`, `pending` |
| `nickname` | `VARCHAR(100)` | Nullable | Apodo dentro del grupo |
| `last_read_message_id` | `UUID` | Nullable | ID del último mensaje leído |
| `unread_count` | `Integer` | Default: 0 | Contador de mensajes no leídos |
| `muted_until` | `DateTime` | Nullable | Fecha hasta la cual está silenciado |
| `joined_at` | `DateTime` | Default: NOW | Fecha de unión al grupo |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `messages`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del mensaje |
| `conversation_id` | `UUID` | Nullable, FK → conversations | ID de conversación (para chats 1-1) |
| `group_id` | `UUID` | Nullable, FK → chat_groups | ID de grupo (para chats grupales) |
| `sender_profile_id` | `UUID` | Not Null | ID del perfil del remitente |
| `content` | `TEXT` | Not Null | Contenido del mensaje |
| `message_type` | `ENUM` | Default: 'text' | Tipo: `text`, `image`, `video`, `audio`, `file`, `system` |
| `status` | `ENUM` | Default: 'sent' | Estado: `sent`, `delivered`, `read`, `failed` |
| `media_url` | `VARCHAR(500)` | Nullable | URL del archivo multimedia |
| `reply_to_id` | `UUID` | Nullable, FK → messages | ID del mensaje al que responde |
| `is_edited` | `Boolean` | Default: false | Indica si fue editado |
| `is_deleted` | `Boolean` | Default: false | Indica si fue eliminado |
| `metadata` | `JSON` | Default: {} | Metadatos adicionales |
| `created_at` | `DateTime` | Auto | Fecha de envío |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

## 🔔 Servicio de Notificaciones (notifications-service-aura)

**ORM:** Prisma  
**Base de datos:** `aura_notifications`  
**Archivo de esquema:** `notifications-service-aura/prisma/schema.prisma`

### Tabla: `user_devices`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: uuid() | Identificador único del registro |
| `userId` | `String` | Indexed | ID del usuario (viene de auth-service) |
| `fcmToken` | `String` | Unique | Token FCM único por dispositivo |
| `deviceInfo` | `String` | Nullable | Información del dispositivo (navegador, OS, etc.) |
| `createdAt` | `DateTime` | Default: now() | Fecha de registro |
| `updatedAt` | `DateTime` | Auto | Fecha de actualización |

**Configuración adicional:**
- **Exchange RabbitMQ:** `notifications_exchange`
- **Queue RabbitMQ:** `notifications_queue`
- **Routing Key:** `notifications_routing_key`

---

## 🌐 Servicio Social (social-service-aura)

**ORM:** Sequelize  
**Base de datos:** `aura_social`  
**Directorio de modelos:** `social-service-aura/src/infrastructure/database/models/`

### Tabla: `user_profiles`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del perfil |
| `user_id` | `UUID` | Unique, Not Null | ID del usuario (referencia a auth-service) |
| `display_name` | `VARCHAR(100)` | Not Null | Nombre a mostrar - REQUERIDO |
| `bio` | `VARCHAR(500)` | Nullable | Biografía del usuario |
| `avatar_url` | `VARCHAR(500)` | Not Null | URL del avatar (Cloudinary) |
| `cover_url` | `TEXT` | Nullable | URL de imagen de portada |
| `location` | `VARCHAR(100)` | Nullable | Ubicación del usuario |
| `website` | `VARCHAR(255)` | Nullable | Sitio web personal |
| `birth_date` | `DATE` | Nullable | Fecha de nacimiento |
| `followers_count` | `Integer` | Not Null, Default: 0 | Número de seguidores |
| `following_count` | `Integer` | Not Null, Default: 0 | Número de usuarios seguidos |
| `posts_count` | `Integer` | Not Null, Default: 0 | Número de publicaciones |
| `is_verified` | `Boolean` | Not Null, Default: false | Cuenta verificada |
| `is_active` | `Boolean` | Not Null, Default: true | Cuenta activa |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

**Índices:**
- `user_id` (unique)
- `display_name`
- `is_active`
- `is_verified`
- `created_at`

---

### Tabla: `complete_profiles`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único |
| `user_id` | `UUID` | Unique, Not Null | ID del usuario |
| `full_name` | `VARCHAR(100)` | Nullable | Nombre completo |
| `age` | `Integer` | Nullable, Range: 1-120 | Edad del usuario |
| `profile_picture` | `VARCHAR(500)` | Nullable | URL de la foto de perfil |
| `bio` | `TEXT` | Nullable, Max: 500 chars | Descripción breve |
| `hobbies` | `JSONB` | Nullable, Default: [] | Array de hobbies principales |
| `location` | `VARCHAR(100)` | Nullable | Ubicación del usuario |
| `website` | `VARCHAR(200)` | Nullable, Must be URL | Sitio web personal |
| `phone` | `VARCHAR(20)` | Nullable | Número de teléfono |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `posts`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la publicación |
| `user_id` | `UUID` | Not Null | ID del usuario autor |
| `content` | `TEXT` | Nullable | Contenido textual del post |
| `type` | `ENUM` | Not Null, Default: 'text' | Tipo: `text`, `image`, `video`, `text_image`, `text_video` |
| `visibility` | `ENUM` | Not Null, Default: 'public' | Visibilidad: `public`, `private`, `friends` |
| `location` | `VARCHAR(255)` | Nullable | Ubicación del post |
| `tags` | `JSONB` | Nullable | Etiquetas del post |
| `metadata` | `JSONB` | Nullable | Metadatos adicionales |
| `likes_count` | `Integer` | Not Null, Default: 0 | Contador de likes |
| `comments_count` | `Integer` | Not Null, Default: 0 | Contador de comentarios |
| `shares_count` | `Integer` | Not Null, Default: 0 | Contador de compartidos |
| `is_active` | `Boolean` | Not Null, Default: true | Estado activo del post |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `comments`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del comentario |
| `post_id` | `UUID` | Not Null, FK → posts, Indexed | ID del post comentado |
| `user_id` | `UUID` | Not Null, Indexed | ID del usuario autor |
| `parent_id` | `UUID` | Nullable, FK → comments, Indexed | ID del comentario padre (respuestas) |
| `content` | `TEXT` | Not Null | Contenido del comentario |
| `likes_count` | `Integer` | Not Null, Default: 0 | Contador de likes |
| `replies_count` | `Integer` | Not Null, Default: 0 | Contador de respuestas |
| `level` | `Integer` | Not Null, Default: 1 | Nivel de anidación |
| `is_edited` | `Boolean` | Not Null, Default: false | Indica si fue editado |
| `is_active` | `Boolean` | Not Null, Default: true, Indexed | Estado activo |
| `edited_at` | `DateTime` | Nullable | Fecha de última edición |
| `created_at` | `DateTime` | Auto, Indexed | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `likes`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del like |
| `user_id` | `UUID` | Not Null, Indexed | ID del usuario que dio el like |
| `likeable_type` | `ENUM` | Not Null, Indexed | Tipo de entidad: `post`, `comment` |
| `likeable_id` | `UUID` | Not Null, Indexed | ID de la entidad likes |
| `type` | `ENUM` | Not Null, Default: 'like', Indexed | Tipo de reacción: `like`, `dislike`, `love`, `angry`, `sad`, `wow` |
| `created_at` | `DateTime` | Not Null, Default: NOW | Fecha del like |

**Restricciones:**
- Índice único compuesto: `[user_id, likeable_id, likeable_type]` - Un usuario solo puede dar un like por entidad

---

### Tabla: `media`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del archivo |
| `post_id` | `UUID` | Not Null, FK → posts, Indexed | ID del post al que pertenece |
| `type` | `ENUM` | Not Null, Indexed | Tipo: `image`, `video`, `audio`, `document` |
| `url` | `TEXT` | Not Null | URL del archivo en Cloudinary |
| `public_id` | `VARCHAR(500)` | Nullable, Indexed | Public ID de Cloudinary |
| `original_name` | `VARCHAR(500)` | Nullable | Nombre original del archivo |
| `mime_type` | `VARCHAR(100)` | Nullable | Tipo MIME del archivo |
| `size` | `BIGINT` | Nullable | Tamaño en bytes |
| `width` | `Integer` | Nullable | Ancho en píxeles (imágenes/videos) |
| `height` | `Integer` | Nullable | Alto en píxeles (imágenes/videos) |
| `duration` | `Float` | Nullable | Duración en segundos (videos/audios) |
| `format` | `VARCHAR(20)` | Nullable | Formato del archivo (jpg, png, mp4, etc.) |
| `alt_text` | `TEXT` | Nullable | Texto alternativo para accesibilidad |
| `metadata` | `JSONB` | Nullable | Metadatos adicionales |
| `order_position` | `Integer` | Not Null, Default: 1, Indexed | Posición de orden en el post |
| `is_primary` | `Boolean` | Not Null, Default: false | Archivo principal del post |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

---

### Tabla: `interests`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único del interés |
| `user_id` | `UUID` | Not Null, Indexed | ID del usuario propietario |
| `name` | `VARCHAR(100)` | Not Null, Indexed | Nombre del interés |
| `category` | `VARCHAR(50)` | Nullable, Indexed | Categoría (deportes, música, etc.) |
| `created_at` | `DateTime` | Not Null, Default: NOW | Fecha de creación |

**Restricciones:**
- Índice único compuesto: `[user_id, name]` - Un usuario no puede tener intereses duplicados

---

### Tabla: `user_preferences`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único |
| `user_id` | `UUID` | Unique, Not Null, Indexed | ID del usuario |
| `preferences` | `JSONB` | Not Null, Default: [] | Array de preferencias |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

**Valores válidos para preferences:**
- `Deportes`, `Arte`, `Música`, `Lectura`, `Tecnología`
- `Naturaleza`, `Voluntariado`, `Gaming`, `Fotografía`
- `Cocina`, `Baile`, `Meditación`

---

### Tabla: `communities`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la comunidad |
| `creator_id` | `UUID` | Not Null, FK → user_profiles, Indexed | ID del creador |
| `name` | `VARCHAR(100)` | Not Null | Nombre de la comunidad |
| `description` | `TEXT` | Nullable | Descripción de la comunidad |
| `category` | `ENUM` | Not Null, Indexed | Categoría de la comunidad |
| `tags` | `JSONB` | Nullable, Default: [] | Etiquetas de la comunidad |
| `community_image_url` | `VARCHAR(500)` | Nullable | URL de la imagen |
| `members_count` | `Integer` | Default: 1 | Cantidad de miembros |
| `is_active` | `Boolean` | Default: true, Indexed | Comunidad activa |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

**Valores válidos para category:**
- `Deportes`, `Arte`, `Música`, `Lectura`, `Tecnología`
- `Naturaleza`, `Voluntariado`, `Gaming`, `Fotografía`
- `Cocina`, `Baile`, `Meditación`

---

### Tabla: `community_members`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la membresía |
| `community_id` | `UUID` | Not Null, FK → communities, Indexed | ID de la comunidad |
| `user_id` | `UUID` | Not Null, FK → user_profiles, Indexed | ID del usuario miembro |
| `role` | `ENUM` | Default: 'member' | Rol: `creator`, `admin`, `moderator`, `member` |
| `joined_at` | `DateTime` | Not Null, Default: NOW | Fecha de unión |

**Restricciones:**
- Índice único compuesto: `[community_id, user_id]` - Un usuario solo puede ser miembro una vez

---

### Tabla: `friendships`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | `UUID` | PK, Default: UUIDv4 | Identificador único de la amistad |
| `requester_id` | `UUID` | Not Null, FK → user_profiles, Indexed | ID del solicitante |
| `addressee_id` | `UUID` | Not Null, FK → user_profiles, Indexed | ID del destinatario |
| `status` | `ENUM` | Default: 'pending', Indexed | Estado: `pending`, `accepted`, `rejected`, `blocked` |
| `requested_at` | `DateTime` | Not Null, Indexed | Fecha de solicitud |
| `responded_at` | `DateTime` | Nullable | Fecha de respuesta |
| `is_active` | `Boolean` | Not Null, Default: true, Indexed | Relación activa |
| `created_at` | `DateTime` | Auto | Fecha de creación |
| `updated_at` | `DateTime` | Auto | Fecha de actualización |

**Restricciones:**
- Índice único compuesto: `[requester_id, addressee_id]` - Solo una solicitud por par de usuarios
- Validación: `requester_id ≠ addressee_id` - No puedes enviarte solicitud a ti mismo

---

## 🐰 Broker de Mensajes (RabbitMQ)

**Versión:** 3-management-alpine  
**Contenedor:** `aura_rabbitmq`

### Credenciales de Acceso

| Parámetro | Valor |
|-----------|-------|
| **URL AMQP** | `amqp://guest:guest@rabbitmq:5672` |
| **URL Management** | `http://localhost:15672` |
| **Usuario** | `guest` |
| **Contraseña** | `guest` |

### Configuración de Notificaciones

| Elemento | Nombre |
|----------|--------|
| **Exchange** | `notifications_exchange` |
| **Queue** | `notifications_queue` |
| **Routing Key** | `notifications_routing_key` |

### Flujo de Eventos

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────────────┐
│  Auth Service   │──────│    RabbitMQ     │──────│  Notifications Service  │
│  (Publisher)    │      │   (Exchange)    │      │     (Consumer)          │
└─────────────────┘      └─────────────────┘      └─────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   notifications_queue │
                    └───────────────────────┘
```

**Eventos publicados por Auth Service:**
- Registro de usuario nuevo
- Cambios de contraseña
- Actualizaciones de tokens de dispositivo

---

## ☁️ Almacenamiento de Medios (Cloudinary)

### Credenciales de Acceso

| Parámetro | Valor |
|-----------|-------|
| **Cloud Name** | `derzzspjo` |
| **API Key** | `967388682229272` |
| **API Secret** | `BOF1fG3Hx-stAkgKdJgehh23iDw` |

### Configuración de Uploads

| Parámetro | Valor |
|-----------|-------|
| **Tamaño máximo de archivo** | 100 MB |
| **Número máximo de archivos** | 10 por request |

### Tipos de medios soportados

- Imágenes: `jpg`, `png`, `gif`, `webp`
- Videos: `mp4`, `mov`, `avi`
- Audios: `mp3`, `wav`
- Documentos: `pdf`, `doc`, `docx`

---

## 📊 Diagrama de Flujo de Datos

```mermaid
flowchart TB
    subgraph Frontend["🖥️ Frontend (React/Mobile)"]
        Cliente[Cliente]
    end
    
    subgraph Gateway["🌐 Gateway Service :3000"]
        GW[API Gateway]
    end
    
    subgraph Auth["🔐 Auth Service :3001"]
        AuthAPI[Auth API]
        AuthDB[(aura_auth)]
    end
    
    subgraph Social["🌐 Social Service :3002"]
        SocialAPI[Social API]
        SocialDB[(aura_social)]
    end
    
    subgraph Messaging["💬 Messaging Service :3003"]
        MsgAPI[Messaging API]
        MsgDB[(aura_messaging)]
    end
    
    subgraph Notifications["🔔 Notifications Service :3004"]
        NotifAPI[Notifications API]
        NotifDB[(aura_notifications)]
    end
    
    subgraph Infrastructure["🏗️ Infraestructura"]
        RabbitMQ[(RabbitMQ)]
        Cloudinary[☁️ Cloudinary]
    end
    
    Cliente --> GW
    GW --> AuthAPI
    GW --> SocialAPI
    GW --> MsgAPI
    GW --> NotifAPI
    
    AuthAPI --> AuthDB
    SocialAPI --> SocialDB
    MsgAPI --> MsgDB
    NotifAPI --> NotifDB
    
    AuthAPI --> RabbitMQ
    RabbitMQ --> NotifAPI
    
    SocialAPI --> Cloudinary
    MsgAPI --> Cloudinary
```

---

## 🔗 Relaciones entre Servicios

### Referencia Cruzada de user_id

El campo `user_id` es la clave principal para relacionar datos entre servicios:

```
┌──────────────────┐
│   Auth Service   │
│   users.user_id  │ ◄───────── Fuente primaria de identidad
└────────┬─────────┘
         │
         ├──────────────────────────────────────┐
         │                                      │
         ▼                                      ▼
┌────────────────────┐              ┌────────────────────────┐
│   Social Service   │              │   Messaging Service    │
│ user_profiles.user_id │           │   users.profile_id     │
│  posts.user_id     │              │ (referencia a        │
│  comments.user_id  │              │  user_profiles.id)   │
│  likes.user_id     │              └────────────────────────┘
└────────────────────┘
         │
         ▼
┌────────────────────────┐
│  Notifications Service │
│  user_devices.userId   │
└────────────────────────┘
```

---

## 📝 Notas Importantes

1. **UUIDs:** Todos los identificadores principales utilizan UUIDs v4 para garantizar unicidad global.

2. **Timestamps:** Todas las tablas incluyen campos `created_at` y `updated_at` para auditoría.

3. **Soft Delete:** Se utiliza el campo `is_active` en lugar de eliminación física.

4. **Eventos Asíncronos:** Los eventos entre servicios se comunican a través de RabbitMQ.

5. **Almacenamiento de Medios:** Todos los archivos multimedia se almacenan en Cloudinary, guardando solo URLs en la base de datos.

6. **JWT Token:** Todos los servicios comparten el mismo `JWT_SECRET` para validación de tokens.

---

*Documento generado automáticamente mediante análisis del código fuente.*
