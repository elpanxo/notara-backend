# notara-backend

Repositorio de microservicios del backend de **Notara**, plataforma de aprendizaje de inglés mediante música. Contiene todos los servicios que conforman la capa de negocio y datos de la aplicación.

---

## Arquitectura

```
notara-backend/
├── ms-usuarios/          # Microservicio de autenticación y usuarios (Spring Boot, Puerto 8081)
├── ms-canciones/         # Microservicio de canciones y letras (Node.js/Fastify, Puerto 3002)
├── ms-notas-metas/       # Microservicio de notas y metas (Spring Boot, Puerto 8083)
├── api-gateway/          # API Gateway Node.js (Express, Puerto 3000)
├── api-getaway-001/      # API Gateway Spring Cloud Gateway (Puerto 8086)
├── eureka-server/        # Servidor de descubrimiento de servicios (Puerto 8761)
└── docker-compose.yml    # Stack completo de servicios
```

Cada microservicio corre en su propio contenedor Docker. La comunicación interna se realiza a través de la red Docker interna. Solo el API Gateway y el Gateway Spring Cloud están expuestos hacia el frontend.

---

## Tecnologías

| Servicio | Tecnología | Base de datos |
|---|---|---|
| ms-usuarios | Spring Boot 3.2 + Spring Security + JWT | PostgreSQL |
| ms-canciones | Node.js 20 + Fastify | MongoDB + Redis |
| ms-notas-metas | Spring Boot 3.3 | PostgreSQL |
| api-gateway | Node.js 20 + Express | — |
| api-getaway-001 | Spring Cloud Gateway | — |
| eureka-server | Spring Cloud Netflix Eureka | — |

---

## Variables de entorno

Crea un archivo `.env` en la raíz del repositorio con las siguientes variables:

```env
# JWT
JWT_SECRET=tu_secreto_jwt_largo_y_seguro
JWT_REFRESH_SECRET=otro_secreto_para_refresh_token

# Spotify API (obtener en https://developer.spotify.com)
SPOTIFY_CLIENT_ID=tu_client_id
SPOTIFY_CLIENT_SECRET=tu_client_secret
```

---

## Levantar el stack localmente

### Requisitos previos

- Docker Desktop instalado y corriendo
- Docker Compose v2+
- Archivo `.env` configurado (ver sección anterior)

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/elpanxo/notara-backend.git
cd notara-backend

# 2. Crear el archivo .env con las variables requeridas
cp .env.example .env   # editar con tus valores reales

# 3. Construir y levantar todos los servicios
docker compose up --build

# 4. Verificar que los servicios están corriendo
docker compose ps
```

### Orden de inicio

Docker Compose gestiona automáticamente el orden mediante `depends_on` con `condition: service_healthy`:

1. `postgres-usuarios` y `postgres-notas-metas` (bases de datos)
2. `mongodb` y `redis`
3. `eureka-server`
4. `ms-usuarios`, `ms-canciones`, `ms-notas-metas`
5. `api-gateway`, `api-getaway-001`

---

## Puertos expuestos

| Servicio | Puerto local |
|---|---|
| API Gateway (Node) | `3000` |
| API Gateway (Spring) | `8086` |
| ms-usuarios | `8081` |
| ms-canciones | `3002` |
| ms-notas-metas | `8083` |
| Eureka Dashboard | `8761` |
| PostgreSQL usuarios | `5432` |
| PostgreSQL notas | `5433` |
| MongoDB | `27017` |
| Redis | `6379` |

---

## Persistencia de datos

Los datos críticos se persisten mediante **named volumes** de Docker. Esto garantiza que la información no se pierda al reiniciar o recrear los contenedores.

| Volumen | Servicio | Justificación |
|---|---|---|
| `postgres_usuarios_data` | PostgreSQL usuarios | Datos de cuentas de usuario |
| `postgres_notas_metas_data` | PostgreSQL notas | Notas y metas del usuario |
| `mongo_data` | MongoDB | Caché de canciones y letras |
| `redis_data` | Redis | Caché de tokens y sesiones |

Se eligieron **named volumes** sobre bind mounts porque Docker gestiona su ciclo de vida de forma independiente al sistema de archivos del host, lo que facilita el despliegue en EC2 sin depender de rutas absolutas del sistema operativo.

---

## Pipeline CI/CD

El repositorio cuenta con un pipeline en **GitHub Actions** (`.github/workflows/deploy-backend.yml`) que se activa al hacer push sobre la rama `deploy`.

### Flujo del pipeline

```
Push a rama deploy
       │
       ▼
  Checkout código
       │
       ▼
  Login a Amazon ECR
       │
       ▼
  Build imagen ms-canciones
  Push a ECR
       │
       ▼
  Build imagen notara-bff
  Push a ECR
       │
       ▼
  Deploy en EC2 via AWS SSM
  (sin abrir puerto SSH)
       │
       ▼
  Verificar estado del deploy
```

### Secrets requeridos en GitHub

Configurar en `Settings > Secrets and variables > Actions`:

```
EC2_BACKEND_INSTANCE_ID   # ID de la instancia EC2 del backend
JWT_SECRET                # Mismo valor que en .env
JWT_REFRESH_SECRET
SPOTIFY_CLIENT_ID
SPOTIFY_CLIENT_SECRET
```

Las credenciales AWS se configuran en el runner self-hosted mediante IAM Role, por lo que no se almacenan como secrets.

---

## Endpoints principales

### API Gateway (puerto 3000)

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/auth/register` | Registro de usuario |
| `POST` | `/auth/login` | Login, retorna JWT |
| `POST` | `/auth/refresh` | Renovar access token |
| `GET` | `/users/me` | Perfil del usuario autenticado |
| `GET` | `/songs/search?q=` | Buscar canciones en Spotify |
| `GET` | `/songs/:id` | Metadatos de una canción |
| `GET` | `/songs/:id/lyrics` | Letra de la canción |
| `GET` | `/songs/:id/lesson-type` | Tipo de lección recomendado |
| `GET` | `/health` | Estado del gateway |

---

## Tests

### ms-canciones (Node.js)

```bash
cd ms-canciones
npm install
npm test              # ejecutar todos los tests
npm run test:unit     # solo tests unitarios
npm run test:coverage # con reporte de cobertura
```

### ms-usuarios y ms-notas-metas (Java)

```bash
cd ms-usuarios
./mvnw test

cd ms-notas-metas
./mvnw test
```

---

## Despliegue en AWS EC2

Este backend está diseñado para correr en una instancia EC2 **en subred privada**, accesible únicamente desde el frontend a través de Security Groups de AWS. El frontend (subred pública) se comunica con este backend mediante su IP privada.

```
Internet
   │
   ▼
[EC2 Frontend - IP pública]
   │  (Security Group: solo puerto 8080 expuesto)
   ▼
[EC2 Backend - IP privada]
   │  (Security Group: acepta tráfico solo desde EC2 Frontend)
   ▼
[EC2 Data - IP privada]
   │  (Security Group: acepta tráfico solo desde EC2 Backend)
```
