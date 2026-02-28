# ⚡ Fastify API de Ventas — JavaScript Modular

API REST de ventas construida con **Fastify 4**, **PostgreSQL** (vía Prisma ORM), **Redis** (caché) y autenticación **JWT**. Arquitectura modular en capas: routes → service → repository. Sin TypeScript, sin clases — JavaScript funcional puro.

---

## Tabla de contenidos

1. [Requisitos](#requisitos)
2. [Instalación](#instalación)
3. [Variables de entorno](#variables-de-entorno)
4. [Levantar PostgreSQL y Redis](#levantar-postgresql-y-redis)
5. [Migraciones de base de datos](#migraciones-de-base-de-datos)
6. [Ejecutar en desarrollo](#ejecutar-en-desarrollo)
7. [Ejecutar en producción](#ejecutar-en-producción)
8. [Scripts disponibles](#scripts-disponibles)
9. [Endpoints principales](#endpoints-principales)
10. [Estructura del proyecto](#estructura-del-proyecto)
11. [Licencia](#licencia)

---

## Requisitos

| Herramienta | Versión mínima |
|-------------|----------------|
| Node.js     | 18.x o superior |
| npm         | 9.x o superior |
| PostgreSQL   | 14.x o superior |
| Redis       | 6.x o superior (para caché) |

---

## Instalación

```bash
# 1. Clonar el repositorio
git clone https://github.com/AntoniDB/FastifyImplementacion.git
cd FastifyImplementacion

# 2. Instalar dependencias
npm install
```

---

## Variables de entorno

Crea un archivo **`.env`** en la raíz del proyecto (puedes copiar `.env.example` como punto de partida):

```bash
cp .env.example .env
```

Edita `.env` con tus valores reales:

| Variable        | Descripción                                              | Valor por defecto          |
|-----------------|----------------------------------------------------------|----------------------------|
| `NODE_ENV`      | Entorno de ejecución (`development`, `production`, `test`) | `development`            |
| `PORT`          | Puerto en el que escucha el servidor                     | `3000`                     |
| `DATABASE_URL`  | Cadena de conexión a PostgreSQL (**requerida**)          | —                          |
| `REDIS_URL`     | URL de conexión a Redis                                  | `redis://localhost:6379`   |
| `JWT_SECRET`    | Clave secreta para firmar tokens JWT (mínimo 32 caracteres, **requerida**) | — |
| `JWT_EXPIRES_IN`| Tiempo de expiración del token JWT                       | `7d`                       |

### Ejemplo de `.env`

```dotenv
NODE_ENV=development
PORT=3000

# PostgreSQL
DATABASE_URL="postgresql://postgres:password@localhost:5432/ventas_db"

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=cambia_esto_por_una_clave_secreta_de_al_menos_32_caracteres
JWT_EXPIRES_IN=7d
```

> ⚠️ **Nunca subas `.env` al repositorio.** Está incluido en `.gitignore`.

---

## Levantar PostgreSQL y Redis

### Con Docker Compose (recomendado)

Si no tienes PostgreSQL ni Redis instalados localmente, puedes levantarlos con Docker:

```bash
# Crear el archivo docker-compose.yml en la raíz (si no existe)
cat > docker-compose.yml << 'EOF'
version: '3.9'
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: ventas_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"

volumes:
  postgres_data:
EOF

# Iniciar los servicios
docker compose up -d
```

### Con instalación local

Si ya tienes PostgreSQL instalado, crea la base de datos manualmente:

```sql
CREATE DATABASE ventas_db;
```

---

## Migraciones de base de datos

Después de configurar `.env` y tener PostgreSQL en marcha:

```bash
# Ejecutar migraciones (crea/actualiza las tablas)
npm run db:migrate

# Regenerar el cliente Prisma (necesario después de cambiar schema.prisma)
npm run db:generate
```

---

## Ejecutar en desarrollo

```bash
npm run dev
```

El servidor se iniciará con **nodemon** (recarga automática ante cambios) en `http://localhost:3000`.

---

## Ejecutar en producción

```bash
npm start
```

---

## Scripts disponibles

| Script             | Descripción                                              |
|--------------------|----------------------------------------------------------|
| `npm run dev`      | Inicia el servidor con nodemon (recarga en caliente)     |
| `npm start`        | Inicia el servidor en modo producción                    |
| `npm run db:migrate` | Ejecuta migraciones de Prisma contra la BD             |
| `npm run db:generate` | Regenera el cliente Prisma                            |
| `npm run db:studio` | Abre Prisma Studio (interfaz visual de la BD)           |

---

## Endpoints principales

### Salud

| Método | Ruta       | Descripción                  | Auth |
|--------|------------|------------------------------|------|
| GET    | `/health`  | Estado del servidor          | No   |

### Autenticación (`/api/auth`)

| Método | Ruta                  | Descripción                          | Auth |
|--------|-----------------------|--------------------------------------|------|
| POST   | `/api/auth/register`  | Registrar nuevo usuario              | No   |
| POST   | `/api/auth/login`     | Iniciar sesión, devuelve token JWT   | No   |
| GET    | `/api/auth/me`        | Datos del usuario autenticado        | Sí   |

### Ventas (`/api/sales`)

| Método | Ruta               | Descripción                                    | Auth  |
|--------|--------------------|------------------------------------------------|-------|
| GET    | `/api/sales`       | Listar ventas (soporta `?page=1&limit=10&status=PENDING`) | Sí |
| GET    | `/api/sales/:id`   | Obtener una venta por ID (con caché Redis)     | Sí    |
| POST   | `/api/sales`       | Crear nueva venta                              | Sí    |
| PATCH  | `/api/sales/:id`   | Actualizar estado de una venta                 | Sí    |
| DELETE | `/api/sales/:id`   | Eliminar una venta                             | Admin |

> 💡 Incluir el token JWT en el header: `Authorization: Bearer <token>`

---

## Estructura del proyecto

```
FastifyImplementacion/
├── prisma/
│   └── schema.prisma          # Modelos y relaciones de la BD
├── src/
│   ├── main.js                # Punto de entrada — arranca el servidor
│   ├── app.js                 # Configura Fastify, registra plugins y módulos
│   ├── config/
│   │   └── env.js             # Valida variables de entorno con Zod
│   ├── modules/
│   │   ├── sales/
│   │   │   ├── sales.module.js     # Plugin Fastify del módulo
│   │   │   ├── sales.routes.js     # URLs, métodos HTTP y middlewares
│   │   │   ├── sales.service.js    # Lógica de negocio
│   │   │   ├── sales.repository.js # Consultas a la BD (Prisma)
│   │   │   └── sales.schema.js     # Schemas Zod (validación/DTOs)
│   │   └── auth/
│   │       ├── auth.module.js
│   │       └── auth.routes.js
│   └── shared/
│       ├── plugins/
│       │   ├── prisma.plugin.js    # Cliente PostgreSQL compartido
│       │   ├── jwt.plugin.js       # Autenticación JWT
│       │   ├── cors.plugin.js      # Configuración CORS
│       │   └── redis.plugin.js     # Conexión Redis
│       └── utils/
│           └── sanitize.js         # Sanitización de inputs
├── .env                       # Variables de entorno (no subir a git)
├── .env.example               # Plantilla de variables de entorno
└── package.json
```

---

## Licencia

Este repositorio no incluye un archivo de licencia. Todos los derechos reservados al autor.
