# Guía Profesional de Docker Compose
## Arquitectura Contenedorizada: React + FastAPI + PostgreSQL

**Versión:** 2.0  
**Fecha:** 26/02/2026  
**Stack:** React 18+ · FastAPI 0.100+ · PostgreSQL 16 · Docker Compose v2

---

## Tabla de Contenidos

1. [Fundamentos Técnicos](#1-fundamentos-técnicos)
2. [Arquitectura del Sistema](#2-arquitectura-del-sistema)
3. [Estructura Profesional del Proyecto](#3-estructura-profesional-del-proyecto)
4. [Dockerización del Frontend (React)](#4-dockerización-del-frontend-react)
5. [Dockerización del Backend (FastAPI)](#5-dockerización-del-backend-fastapi)
6. [Servicio PostgreSQL](#6-servicio-postgresql)
7. [docker-compose.yml Profesional](#7-docker-composeyml-profesional)
8. [Comunicación entre Servicios](#8-comunicación-entre-servicios)
9. [Modo Desarrollo vs Producción](#9-modo-desarrollo-vs-producción)
10. [Seguridad y Buenas Prácticas](#10-seguridad-y-buenas-prácticas)
11. [Automatización y DevOps](#11-automatización-y-devops)
12. [Problemas Reales y Troubleshooting](#12-problemas-reales-y-troubleshooting)

---

# 1. Fundamentos Técnicos

## 1.1 ¿Qué es Docker?

Docker es una plataforma de contenedorización que empaqueta aplicaciones y todas sus dependencias en unidades aisladas llamadas **contenedores**. A diferencia de las máquinas virtuales, los contenedores comparten el kernel del sistema operativo host, lo que los hace extremadamente ligeros y rápidos de iniciar.

La razón fundamental por la que Docker transformó la industria no es simplemente "empaquetar aplicaciones", sino que **elimina la divergencia entre entornos**. El clásico problema "en mi máquina funciona" desaparece porque el contenedor ES el entorno. Un contenedor construido en tu laptop de desarrollo se comporta de manera idéntica en un servidor de producción en AWS.

Docker utiliza internamente tecnologías del kernel Linux: **namespaces** para el aislamiento de procesos, **cgroups** para la limitación de recursos, y **union filesystems** (OverlayFS) para la gestión eficiente de capas de imagen.

```
┌──────────────────────────────────────────────┐
│              MÁQUINA VIRTUAL                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  App A   │  │  App B   │  │  App C   │   │
│  │  Bins/   │  │  Bins/   │  │  Bins/   │   │
│  │  Libs    │  │  Libs    │  │  Libs    │   │
│  │ Guest OS │  │ Guest OS │  │ Guest OS │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────────────────────────────────┐    │
│  │           HYPERVISOR                 │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │            HOST OS                   │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│              CONTENEDORES                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  App A   │  │  App B   │  │  App C   │   │
│  │  Bins/   │  │  Bins/   │  │  Bins/   │   │
│  │  Libs    │  │  Libs    │  │  Libs    │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────────────────────────────────┐    │
│  │         DOCKER ENGINE                │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │            HOST OS                   │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

**Decisión arquitectónica:** Usamos contenedores en lugar de VMs porque nuestro stack (React + FastAPI + PostgreSQL) no necesita aislamiento a nivel de kernel. Los contenedores ofrecen arranque en milisegundos, menor consumo de memoria y mayor densidad de servicios por host.

## 1.2 ¿Qué es Docker Compose?

Docker Compose es una herramienta de **orquestación declarativa** para definir y ejecutar aplicaciones multi-contenedor. En lugar de ejecutar múltiples comandos `docker run` con decenas de flags, se declara el estado deseado del sistema en un archivo YAML (`docker-compose.yml`), y Compose se encarga de materializarlo.

Compose resuelve tres problemas fundamentales:

1. **Orquestación de dependencias:** Define el orden de arranque y las relaciones entre servicios (la base de datos debe estar lista antes de que el backend intente conectarse).
2. **Gestión de red:** Crea automáticamente una red interna donde los servicios se descubren por nombre DNS.
3. **Reproducibilidad:** Un solo archivo describe todo el sistema. Cualquier ingeniero puede levantar el entorno completo con `docker compose up`.

**Versión importante:** Docker Compose v2 (integrado como plugin de Docker CLI: `docker compose`) reemplaza a la v1 (el binario independiente `docker-compose`). En esta guía usamos **Compose v2** exclusivamente. La diferencia no es solo sintáctica: v2 tiene mejor rendimiento, soporte nativo para BuildKit, y profiles.

## 1.3 Imágenes vs Contenedores

Esta distinción es **crucial** y frecuentemente malentendida:

| Concepto | Analogía | Descripción |
|----------|----------|-------------|
| **Imagen** | Clase (OOP) | Template inmutable de solo lectura. Compuesto por capas ordenadas. |
| **Contenedor** | Instancia (OOP) | Proceso en ejecución creado a partir de una imagen. Tiene estado mutable. |

```
IMAGEN (inmutable, capas de solo lectura)
┌─────────────────────────┐
│  Capa 5: COPY app/ .    │  ← Tu código
│  Capa 4: RUN pip install│  ← Dependencias
│  Capa 3: RUN apt-get    │  ← Paquetes del SO
│  Capa 2: ENV PYTHONPATH  │  ← Variables
│  Capa 1: python:3.12-slim│  ← Imagen base
└─────────────────────────┘
         │
         │  docker run / docker compose up
         ▼
CONTENEDOR (capa de escritura efímera)
┌─────────────────────────┐
│  Capa R/W: logs, tmp,   │  ← Escritura temporal
│  pid files, sockets     │
├─────────────────────────┤
│  Capas de imagen (R/O)  │  ← Compartidas entre contenedores
└─────────────────────────┘
```

**Por qué esto importa en producción:** Las capas se comparten entre contenedores. Si tienes 10 contenedores basados en `python:3.12-slim`, la imagen base se almacena una sola vez en disco. Docker utiliza Copy-on-Write (CoW): un contenedor solo consume espacio adicional cuando escribe datos nuevos. Esto es lo que permite ejecutar cientos de contenedores en un solo host.

**Implicación práctica:** Cada instrucción en un Dockerfile crea una capa. El orden de las instrucciones afecta directamente al caché de build. Las instrucciones que cambian con menor frecuencia deben ir primero (instalar dependencias del SO), y las que cambian con mayor frecuencia al final (copiar código fuente). Esto reduce drásticamente los tiempos de build.

## 1.4 Networking Interno de Docker

Docker crea redes virtuales que permiten la comunicación entre contenedores. Existen varios drivers de red:

| Driver | Uso | Aislamiento |
|--------|-----|-------------|
| `bridge` | Red privada por defecto. Contenedores en la misma red se comunican. | Medio |
| `host` | El contenedor usa la red del host directamente. | Ninguno |
| `none` | Sin conectividad de red. | Total |
| `overlay` | Redes multi-host (Docker Swarm). | Alto |

**En Docker Compose**, se crea automáticamente una red bridge personalizada para cada proyecto. El nombre sigue la convención `{nombre_proyecto}_default`. Dentro de esta red:

```
Red Docker: myproject_default (172.18.0.0/16)
┌──────────────────────────────────────────────────┐
│                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │ frontend │    │ backend  │    │    db     │   │
│  │172.18.0.2│    │172.18.0.3│    │172.18.0.4│   │
│  │  :3000   │───▶│  :8000   │───▶│  :5432   │   │
│  └──────────┘    └──────────┘    └──────────┘   │
│                                                  │
│  DNS interno resuelve nombres de servicio:       │
│  "backend" → 172.18.0.3                         │
│  "db"      → 172.18.0.4                         │
└──────────────────────────────────────────────────┘
         │
    ports: "3000:3000"  (solo frontend expuesto al host)
         │
         ▼
┌──────────────────┐
│   HOST / Usuario │
│   localhost:3000 │
└──────────────────┘
```

**Decisión crítica: redes personalizadas vs red por defecto.** Siempre debemos definir redes explícitas en producción. La red `default` funciona, pero las redes personalizadas permiten segmentación: el frontend no necesita acceso directo a la base de datos. Esto implementa el **principio de menor privilegio** a nivel de red.

## 1.5 Volúmenes

Los contenedores son **efímeros** por diseño. Cuando un contenedor se destruye, todos los datos escritos en su capa de escritura desaparecen. Los volúmenes resuelven esto proporcionando almacenamiento persistente.

Existen tres mecanismos principales:

### Named Volumes (Volúmenes con nombre)
Gestionados completamente por Docker. Son la opción recomendada para persistencia en producción.

```yaml
volumes:
  postgres_data:  # Docker gestiona la ubicación en disco

services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

Los datos se almacenan en `/var/lib/docker/volumes/postgres_data/_data` en el host. Docker controla los permisos y el ciclo de vida.

### Bind Mounts (Montajes de enlace)
Mapean un directorio del host directamente al contenedor. Esenciales para desarrollo (hot reload), pero problemáticos en producción por acoplamiento al filesystem del host.

```yaml
services:
  backend:
    volumes:
      - ./backend/app:/app  # Directorio local → contenedor
```

### tmpfs Mounts
Almacenamiento en memoria RAM. Útil para datos temporales sensibles que no deben persistir en disco (tokens, caché de sesiones).

```yaml
services:
  backend:
    tmpfs:
      - /tmp
```

**Decisión para nuestro sistema:**
- PostgreSQL → Named volume (`postgres_data`). Los datos de la BD DEBEN persistir entre reinicios.
- Backend en desarrollo → Bind mount para hot reload del código.
- Backend en producción → Sin bind mount. El código va dentro de la imagen.
- Frontend en desarrollo → Bind mount para hot reload.
- Frontend en producción → Imagen estática con Nginx. Sin volúmenes.

## 1.6 Ciclo de Vida de Servicios

Comprender el ciclo de vida es fundamental para diagnosticar problemas:

```
    docker compose up
          │
          ▼
┌─────────────────┐
│   CREATED       │  Contenedor creado, no iniciado
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   RUNNING       │  Proceso principal ejecutándose
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────────┐
│ PAUSED │ │  EXITED    │  Proceso terminó (código 0 o error)
└────────┘ └──────┬─────┘
                  │
                  ▼
           ┌────────────┐
           │  REMOVED    │  docker compose down
           └────────────┘
```

**Restart policies** controlan qué ocurre cuando un contenedor se detiene:

| Policy | Comportamiento |
|--------|----------------|
| `no` | No reiniciar nunca (defecto) |
| `always` | Reiniciar siempre, incluyendo al iniciar Docker daemon |
| `on-failure` | Reiniciar solo si el proceso sale con código != 0 |
| `unless-stopped` | Como `always`, pero no reiniciar si fue detenido manualmente |

**Recomendación de producción:** Usar `unless-stopped` para servicios como PostgreSQL y el backend. Esto sobrevive a reinicios del servidor, pero respeta paradas manuales para mantenimiento.

---

# 2. Arquitectura del Sistema

## 2.1 Visión General

```
                    INTERNET
                       │
                       ▼
              ┌────────────────┐
              │   NAVEGADOR    │
              │   (Usuario)    │
              └───────┬────────┘
                      │  HTTP :80 / :443
                      ▼
┌──────────────────────────────────────────────────────────┐
│                    DOCKER HOST                           │
│                                                          │
│  ┌─── Red: frontend_net ───────────────────────────┐     │
│  │                                                 │     │
│  │  ┌──────────────────────┐                       │     │
│  │  │     FRONTEND         │                       │     │
│  │  │  ┌───────────────┐   │                       │     │
│  │  │  │    Nginx       │   │                       │     │
│  │  │  │  (puerto 80)   │   │                       │     │
│  │  │  │  Sirve React   │   │                       │     │
│  │  │  │  build estático│   │                       │     │
│  │  │  │  + proxy_pass  │───┼──── /api/* ────┐     │     │
│  │  │  └───────────────┘   │               │     │     │
│  │  └──────────────────────┘               │     │     │
│  │                                          │     │     │
│  └──────────────────────────────────────────┼─────┘     │
│                                              │           │
│  ┌─── Red: backend_net ─────────────────────┼─────┐     │
│  │                                          │     │     │
│  │  ┌──────────────────────┐               │     │     │
│  │  │      BACKEND         │◄──────────────┘     │     │
│  │  │  ┌───────────────┐   │                      │     │
│  │  │  │   Uvicorn     │   │                      │     │
│  │  │  │  (puerto 8000) │   │                      │     │
│  │  │  │   FastAPI      │   │                      │     │
│  │  │  └───────┬───────┘   │                      │     │
│  │  └──────────┼───────────┘                      │     │
│  │             │ postgresql://db:5432/appdb        │     │
│  │             ▼                                   │     │
│  │  ┌──────────────────────┐                      │     │
│  │  │     POSTGRESQL       │                      │     │
│  │  │  ┌───────────────┐   │                      │     │
│  │  │  │  postgres 16  │   │                      │     │
│  │  │  │ (puerto 5432) │   │   Sin exposición     │     │
│  │  │  │  SOLO interno │   │   al host            │     │
│  │  │  └───────────────┘   │                      │     │
│  │  │  📁 Volume:          │                      │     │
│  │  │  postgres_data       │                      │     │
│  │  └──────────────────────┘                      │     │
│  │                                                 │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## 2.2 Separación de Servicios y Responsabilidades

Cada servicio tiene una responsabilidad única y bien definida:

**Frontend (React + Nginx):**
- Servir la aplicación SPA (Single Page Application) como archivos estáticos.
- Actuar como **reverse proxy** hacia el backend para las rutas `/api/*`.
- Manejar compresión gzip, caché de assets estáticos, y TLS termination.
- En producción, Nginx es el único punto de entrada desde Internet.

**Backend (FastAPI + Uvicorn):**
- Exponer la API REST/GraphQL.
- Ejecutar la lógica de negocio.
- Gestionar autenticación, autorización, y validación.
- Conectarse a PostgreSQL para operaciones CRUD.
- Ejecutar migraciones de esquema de base de datos (Alembic).

**Base de Datos (PostgreSQL):**
- Almacenamiento persistente de datos.
- Integridad transaccional (ACID).
- No se expone fuera de la red Docker. Nunca.

**Justificación de la arquitectura de dos redes:**

Se utilizan dos redes Docker (`frontend_net` y `backend_net`) en lugar de una sola. El backend pertenece a ambas redes, actuando como puente. El frontend NO puede comunicarse directamente con PostgreSQL. Esto aplica el principio de **defensa en profundidad**: si el contenedor frontend es comprometido, el atacante no tiene ruta de red hacia la base de datos.

## 2.3 Flujo de Comunicación

**Petición típica del usuario:**

```
1. Usuario abre http://miapp.com
   → Nginx sirve index.html + bundle JS de React

2. React hace fetch("/api/users")
   → Nginx intercepta /api/* y hace proxy_pass a http://backend:8000

3. FastAPI recibe GET /users
   → SQLAlchemy ejecuta SELECT * FROM users
   → Conexión a postgresql://db:5432/appdb

4. PostgreSQL retorna resultados
   → FastAPI serializa con Pydantic
   → Nginx retorna JSON al navegador
   → React renderiza los datos
```

**Decisión de diseño: Nginx como proxy.** Aunque React podría hacer peticiones directamente a `http://backend:8000`, esto crea problemas de CORS y expone el backend al exterior. Usar Nginx como proxy unifica todo bajo un solo dominio, eliminando CORS y simplificando TLS.

---

# 3. Estructura Profesional del Proyecto

```
project/
│
├── frontend/
│   ├── Dockerfile                # Build de la imagen React
│   ├── Dockerfile.dev            # Imagen para desarrollo con hot reload
│   ├── nginx/
│   │   └── default.conf          # Configuración de Nginx para producción
│   ├── package.json
│   ├── package-lock.json
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   │   └── api.js            # Cliente HTTP configurado
│   │   └── pages/
│   └── .dockerignore
│
├── backend/
│   ├── Dockerfile                # Build de la imagen FastAPI
│   ├── Dockerfile.dev            # Imagen para desarrollo
│   ├── requirements.txt          # Dependencias Python pinned
│   ├── requirements-dev.txt      # Dependencias adicionales de desarrollo
│   ├── alembic.ini               # Configuración de migraciones
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py               # Punto de entrada FastAPI
│   │   ├── config.py             # Configuración centralizada (Pydantic Settings)
│   │   ├── database.py           # Engine y Session de SQLAlchemy
│   │   ├── models/               # Modelos ORM
│   │   │   ├── __init__.py
│   │   │   └── user.py
│   │   ├── schemas/              # Esquemas Pydantic
│   │   │   ├── __init__.py
│   │   │   └── user.py
│   │   ├── routers/              # Endpoints agrupados
│   │   │   ├── __init__.py
│   │   │   └── users.py
│   │   ├── services/             # Lógica de negocio
│   │   └── middleware/
│   ├── tests/
│   │   ├── conftest.py
│   │   └── test_users.py
│   └── .dockerignore
│
├── database/
│   ├── init/
│   │   └── 01-init.sql           # Script SQL de inicialización
│   └── backups/                  # Directorio para dumps
│
├── scripts/
│   ├── start-dev.sh              # Arranque modo desarrollo
│   ├── start-prod.sh             # Arranque modo producción
│   ├── backup-db.sh              # Script de backup PostgreSQL
│   └── wait-for-it.sh            # Script para esperar servicios
│
├── docker-compose.yml            # Configuración base
├── docker-compose.dev.yml        # Override para desarrollo
├── docker-compose.prod.yml       # Override para producción
├── .env                          # Variables de entorno (NO commitear)
├── .env.example                  # Template de variables
├── .gitignore
├── Makefile                      # Comandos simplificados
└── README.md
```

## 3.1 Justificación de la Estructura

**Separación por servicio:** Cada servicio tiene su propio directorio con su Dockerfile. Esto permite builds independientes, versionado separado, y la posibilidad de que diferentes equipos trabajen en paralelo sin conflictos.

**Dockerfiles separados (dev/prod):** En lugar de un solo Dockerfile con targets multi-stage para todo, tener `Dockerfile` (producción) y `Dockerfile.dev` (desarrollo) simplifica la lectura y evita errores al construir la imagen incorrecta. Algunos equipos prefieren un único Dockerfile con `--target`; ambos enfoques son válidos. Aquí priorizamos claridad.

**docker-compose override pattern:** Docker Compose soporta archivos de override. Al ejecutar `docker compose -f docker-compose.yml -f docker-compose.dev.yml up`, el segundo archivo sobrescribe o extiende al primero. Esto permite una base compartida con variaciones por entorno. Es el patrón recomendado oficialmente.

**Directorio `scripts/`:** Scripts de shell para automatización. Un `Makefile` o `justfile` en la raíz proporciona una interfaz unificada (`make dev`, `make prod`, `make backup`).

**`.env.example`:** Se commitea al repositorio como documentación de qué variables son necesarias. El `.env` real con credenciales se excluye del control de versiones.

---

# 4. Dockerización del Frontend (React)

## 4.1 Dockerfile de Producción (Multi-Stage Build)

```dockerfile
# ============================================================
# frontend/Dockerfile — Producción
# Multi-stage build: Node (build) → Nginx (serve)
# ============================================================

# ── Stage 1: Build ──────────────────────────────────────────
FROM node:20-alpine AS builder

# Crear usuario no-root para el build
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copiar SOLO archivos de dependencias primero (caché de capas)
COPY package.json package-lock.json ./

# Instalar dependencias con ci (más rápido y determinista que npm install)
RUN npm ci --only=production=false

# Ahora copiar el código fuente
COPY . .

# Argumento de build para la URL del API
# Se inyecta en build time porque React embebe las variables en el bundle
ARG VITE_API_URL=/api
ENV VITE_API_URL=${VITE_API_URL}

# Generar build de producción
RUN npm run build

# ── Stage 2: Serve ──────────────────────────────────────────
FROM nginx:1.27-alpine AS production

# Eliminar configuración default de Nginx
RUN rm /etc/nginx/conf.d/default.conf

# Copiar configuración personalizada
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Copiar build estático desde el stage anterior
COPY --from=builder /app/dist /usr/share/nginx/html

# Crear usuario no-root para Nginx
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

# Exponer puerto 80 (documentación, no funcional)
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:80/health || exit 1

# Ejecutar como no-root
USER nginx

CMD ["nginx", "-g", "daemon off;"]
```

## 4.2 Justificación Técnica del Multi-Stage Build

El multi-stage build es **obligatorio** para producción por tres razones:

1. **Tamaño de imagen:** La imagen de Node con `node_modules` puede pesar 1-2 GB. La imagen final con solo Nginx + archivos estáticos pesa ~25 MB. Esto reduce drásticamente el tiempo de deploy y la superficie de ataque.

2. **Seguridad:** La imagen final no contiene Node.js, npm, código fuente, ni dependencias de desarrollo. Un atacante que comprometa el contenedor frontend solo tiene acceso a archivos HTML/CSS/JS estáticos y al binario de Nginx.

3. **Caché de capas:** Al copiar `package.json` y `package-lock.json` antes del código fuente, Docker reutiliza la capa de `npm ci` mientras las dependencias no cambien. Esto reduce builds de 3-5 minutos a segundos cuando solo cambia código.

## 4.3 Configuración de Nginx

```nginx
# frontend/nginx/default.conf

# Configuración upstream del backend
upstream backend_api {
    server backend:8000;
    # Si tuvieras múltiples replicas:
    # server backend-1:8000;
    # server backend-2:8000;
}

server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # ── Compresión gzip ────────────────────────────────────
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml;

    # ── Caché de assets estáticos ──────────────────────────
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # ── Proxy hacia el backend ─────────────────────────────
    location /api/ {
        proxy_pass http://backend_api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 60s;

        # WebSocket support (si usas conexiones en tiempo real)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # ── Health check endpoint ──────────────────────────────
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # ── SPA: todas las rutas a index.html ──────────────────
    location / {
        try_files $uri $uri/ /index.html;
    }

    # ── Seguridad: ocultar versión de Nginx ────────────────
    server_tokens off;

    # ── Headers de seguridad ───────────────────────────────
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

**`try_files $uri $uri/ /index.html`:** Esta directiva es crítica para SPAs. Cuando el usuario navega a `/users/123`, Nginx primero busca un archivo literal `/users/123`. Al no encontrarlo, sirve `index.html`, y React Router maneja la ruta en el cliente.

**`proxy_pass http://backend_api/`:** La barra final es importante. Con `proxy_pass http://backend_api/`, una petición a `/api/users` se reenvía como `/users` al backend. Sin la barra, se reenviaría como `/api/users`.

## 4.4 Dockerfile de Desarrollo

```dockerfile
# ============================================================
# frontend/Dockerfile.dev — Desarrollo con Hot Reload
# ============================================================
FROM node:20-alpine

WORKDIR /app

# Instalar dependencias primero para caché
COPY package.json package-lock.json ./
RUN npm ci

# El código se monta como bind mount, no se copia
# COPY . .  ← NO hacer esto en desarrollo

EXPOSE 5173

# Vite dev server con hot reload
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

**¿Por qué no copiar código en desarrollo?** Porque usamos bind mounts. El directorio `./frontend/src` del host se monta en `/app/src` del contenedor. Cuando guardas un archivo, Vite detecta el cambio y recarga el navegador instantáneamente. Copiar el código al build haría que los cambios no se reflejen hasta reconstruir la imagen.

## 4.5 .dockerignore del Frontend

```
# frontend/.dockerignore
node_modules
dist
build
.git
.gitignore
.env
.env.*
*.md
.vscode
.idea
coverage
```

**Crucial:** Incluir `node_modules` en `.dockerignore`. Sin esto, Docker copiaría los `node_modules` del host (que pueden tener binarios compilados para tu SO) al contenedor (Linux Alpine). Esto causa errores crípticos de módulos nativos. Las dependencias deben instalarse siempre DENTRO del contenedor.

---

# 5. Dockerización del Backend (FastAPI)

## 5.1 Dockerfile de Producción

```dockerfile
# ============================================================
# backend/Dockerfile — Producción
# ============================================================
FROM python:3.12-slim AS base

# Prevenir creación de .pyc y buffering de stdout
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Pip
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Instalar dependencias del sistema necesarias para psycopg2
# (compilador C y headers de PostgreSQL)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq-dev \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario no-root
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

WORKDIR /app

# ── Stage: Dependencies ────────────────────────────────────
FROM base AS dependencies

# Copiar solo requirements para caché de dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ── Stage: Production ──────────────────────────────────────
FROM dependencies AS production

# Copiar código de la aplicación
COPY ./app ./app
COPY ./alembic ./alembic
COPY ./alembic.ini .

# Asignar propiedad al usuario no-root
RUN chown -R appuser:appuser /app

# Cambiar a usuario no-root
USER appuser

# Puerto de la aplicación
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Gunicorn con workers Uvicorn para producción
# Workers = (2 × CPU cores) + 1
CMD ["gunicorn", "app.main:app", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--workers", "4", \
     "--bind", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--timeout", "120", \
     "--graceful-timeout", "30", \
     "--keep-alive", "5"]
```

## 5.2 Decisiones Técnicas Explicadas

### Uvicorn vs Gunicorn en producción

**Uvicorn** es un servidor ASGI de alto rendimiento basado en uvloop. Es excelente para desarrollo, pero en producción tiene una limitación: es single-process. Si el proceso muere, el servicio cae.

**Gunicorn** es un process manager probado en batalla. Al combinarlo con workers de tipo `UvicornWorker`, obtenemos lo mejor de ambos mundos: Gunicorn gestiona múltiples procesos worker (reinicio automático si un worker muere, distribución de carga), y cada worker ejecuta Uvicorn para manejar peticiones ASGI.

```
Gunicorn (master)
├── UvicornWorker 1 (PID 10) ← Maneja requests async
├── UvicornWorker 2 (PID 11) ← Maneja requests async
├── UvicornWorker 3 (PID 12) ← Maneja requests async
└── UvicornWorker 4 (PID 13) ← Maneja requests async
```

**Fórmula de workers:** `(2 × núcleos_CPU) + 1`. Para un contenedor con 2 CPU asignadas: 5 workers. Más workers no siempre es mejor; cada uno consume ~50-100 MB de RAM.

### `PYTHONUNBUFFERED=1`

Sin esta variable, Python bufferiza la salida estándar. En un contenedor, esto significa que los logs no aparecen en `docker compose logs` hasta que el buffer se llena. Con `PYTHONUNBUFFERED=1`, cada línea de log aparece inmediatamente. Indispensable para debugging y monitoreo.

### `PYTHONDONTWRITEBYTECODE=1`

Previene la creación de archivos `.pyc`. En un contenedor de producción, estos archivos solo ocupan espacio en la capa de escritura sin beneficio de rendimiento significativo (el contenedor se destruye y recrea con cada deploy).

## 5.3 requirements.txt (Dependencias Pinned)

```
# backend/requirements.txt
# SIEMPRE versiones pinned para reproducibilidad

# Framework
fastapi==0.115.6
uvicorn[standard]==0.34.0
gunicorn==23.0.0

# Base de datos
sqlalchemy==2.0.36
psycopg2-binary==2.9.10
alembic==1.14.1

# Validación y configuración
pydantic==2.10.4
pydantic-settings==2.7.1

# Seguridad
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.18

# HTTP
httpx==0.28.1

# Utilidades
python-dotenv==1.0.1
```

**¿Por qué `psycopg2-binary` en lugar de `psycopg2`?** El paquete `psycopg2` requiere compilar extensiones C contra las headers de libpq instaladas en el sistema. `psycopg2-binary` incluye una versión precompilada de libpq. Esto simplifica el Dockerfile y acelera el build. Para producción de alta escala, algunos equipos prefieren `psycopg2` compilado contra la versión exacta de libpq del servidor PostgreSQL, pero `psycopg2-binary` es perfectamente válido para la mayoría de casos.

## 5.4 Código Base del Backend

### Configuración centralizada con Pydantic Settings

```python
# backend/app/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    """
    Configuración centralizada. Los valores se cargan de variables de entorno.
    Pydantic Settings valida tipos automáticamente.
    """
    # Base de datos
    POSTGRES_HOST: str = "db"
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str = "appuser"
    POSTGRES_PASSWORD: str  # Obligatorio, sin default
    POSTGRES_DB: str = "appdb"

    # Aplicación
    APP_NAME: str = "My App API"
    DEBUG: bool = False
    ALLOWED_ORIGINS: list[str] = ["http://localhost:3000"]

    # Seguridad
    SECRET_KEY: str  # Obligatorio
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    @property
    def database_url(self) -> str:
        return (
            f"postgresql://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}"
            f"@{self.POSTGRES_HOST}:{self.POSTGRES_PORT}/{self.POSTGRES_DB}"
        )

    @property
    def async_database_url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}"
            f"@{self.POSTGRES_HOST}:{self.POSTGRES_PORT}/{self.POSTGRES_DB}"
        )

    class Config:
        env_file = ".env"
        case_sensitive = True


@lru_cache
def get_settings() -> Settings:
    """Singleton: la configuración se parsea una sola vez."""
    return Settings()
```

### Punto de entrada FastAPI

```python
# backend/app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.database import engine, Base
from app.routers import users

settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Ciclo de vida de la aplicación.
    Se ejecuta al arrancar y al apagar.
    """
    # Startup: crear tablas si no existen (solo desarrollo)
    # En producción, usar Alembic exclusivamente
    if settings.DEBUG:
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown: cerrar conexiones
    await engine.dispose()


app = FastAPI(
    title=settings.APP_NAME,
    docs_url="/docs" if settings.DEBUG else None,  # Desactivar Swagger en prod
    redoc_url="/redoc" if settings.DEBUG else None,
    lifespan=lifespan,
)

# CORS — solo necesario si el frontend NO pasa por Nginx proxy
# Con Nginx proxy_pass, CORS no es necesario (mismo origen)
if settings.DEBUG:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.ALLOWED_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

# Registrar routers
app.include_router(users.router, prefix="/users", tags=["users"])


@app.get("/health")
async def health_check():
    """Endpoint para healthchecks de Docker y load balancers."""
    return {"status": "healthy"}
```

### Conexión a la Base de Datos

```python
# backend/app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase

from app.config import get_settings

settings = get_settings()

# Engine con pool de conexiones
engine = create_async_engine(
    settings.async_database_url,
    echo=settings.DEBUG,         # Log SQL queries en modo debug
    pool_size=20,                # Conexiones en el pool
    max_overflow=10,             # Conexiones extra permitidas
    pool_timeout=30,             # Segundos esperando una conexión
    pool_recycle=1800,           # Reciclar conexiones cada 30 min
    pool_pre_ping=True,          # Verificar conexión antes de usarla
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


class Base(DeclarativeBase):
    """Clase base para todos los modelos ORM."""
    pass


async def get_db() -> AsyncSession:
    """Dependency injection para obtener sesión de BD."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

## 5.5 Dockerfile de Desarrollo

```dockerfile
# ============================================================
# backend/Dockerfile.dev — Desarrollo con Hot Reload
# ============================================================
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Dependencias del sistema
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq-dev \
        curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Instalar dependencias (incluye herramientas de desarrollo)
COPY requirements.txt requirements-dev.txt ./
RUN pip install --no-cache-dir -r requirements.txt -r requirements-dev.txt

# El código se monta como bind mount
EXPOSE 8000

# Uvicorn con --reload para hot reload
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

```
# backend/requirements-dev.txt
pytest==8.3.4
pytest-asyncio==0.24.0
httpx==0.28.1
ruff==0.8.6
mypy==1.14.1
debugpy==1.8.11
```

---

# 6. Servicio PostgreSQL

## 6.1 Configuración del Servicio

PostgreSQL se configura directamente en `docker-compose.yml` sin Dockerfile propio. La imagen oficial es suficiente para la mayoría de casos.

**Decisiones clave:**

### Persistencia con Named Volume

```yaml
volumes:
  postgres_data:
    driver: local
```

El directorio `/var/lib/postgresql/data` dentro del contenedor es donde PostgreSQL almacena todos los datos (tablas, índices, WAL). Montamos un named volume aquí. Esto significa que al ejecutar `docker compose down`, los datos persisten. Solo `docker compose down -v` destruye los volúmenes.

**Trampa común:** Si NO montas un volumen, un `docker compose down` seguido de `docker compose up` crea un contenedor PostgreSQL nuevo y vacío. Todos los datos se pierden. Esto es la causa #1 de pérdida de datos en equipos que empiezan con Docker.

### Inicialización Automática

La imagen oficial de PostgreSQL ejecuta automáticamente todos los scripts `.sql` y `.sh` en `/docker-entrypoint-initdb.d/` cuando la base de datos se crea por primera vez. Esto solo ocurre si el volumen de datos está vacío.

```sql
-- database/init/01-init.sql
-- Este script se ejecuta SOLO en la primera creación de la BD

-- Crear extensiones necesarias
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Crear esquema de la aplicación
CREATE SCHEMA IF NOT EXISTS app;

-- Ajustes de rendimiento (override de postgresql.conf)
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '768MB';
ALTER SYSTEM SET maintenance_work_mem = '128MB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET work_mem = '4MB';
ALTER SYSTEM SET max_connections = 100;
```

**Orden de ejecución:** Los scripts se ejecutan en orden alfabético. Por eso los prefijamos con números (`01-init.sql`, `02-seed.sql`).

### Variables de Entorno Requeridas

```
POSTGRES_USER=appuser         # NO usar "postgres" (superusuario)
POSTGRES_PASSWORD=...         # Contraseña fuerte
POSTGRES_DB=appdb             # Nombre de la base de datos
```

**Seguridad:** Creamos un usuario dedicado (`appuser`) en lugar de usar el superusuario `postgres`. El backend se conecta con permisos limitados. Si la aplicación es comprometida, el atacante no tiene permisos de superusuario en PostgreSQL.

## 6.2 Script de Backup

```bash
#!/bin/bash
# scripts/backup-db.sh
# Uso: ./scripts/backup-db.sh

set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="./database/backups"
BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.sql.gz"

# Crear directorio si no existe
mkdir -p "${BACKUP_DIR}"

# Dump comprimido usando el contenedor de PostgreSQL
docker compose exec -T db pg_dump \
    -U "${POSTGRES_USER:-appuser}" \
    -d "${POSTGRES_DB:-appdb}" \
    --format=custom \
    --compress=9 \
    --verbose \
    > "${BACKUP_FILE}"

echo "Backup creado: ${BACKUP_FILE}"
echo "Tamaño: $(du -h ${BACKUP_FILE} | cut -f1)"

# Eliminar backups con más de 7 días
find "${BACKUP_DIR}" -name "backup_*.sql.gz" -mtime +7 -delete
echo "Backups antiguos eliminados"
```

## 6.3 Restauración

```bash
#!/bin/bash
# Restaurar un backup específico
docker compose exec -T db pg_restore \
    -U appuser \
    -d appdb \
    --clean \
    --if-exists \
    --verbose \
    < ./database/backups/backup_20260225_120000.sql.gz
```

---

# 7. docker-compose.yml Profesional

## 7.1 Archivo Base

```yaml
# docker-compose.yml
# ═══════════════════════════════════════════════════════════
# Archivo base de Docker Compose para el sistema completo
# Se usa combinado con docker-compose.dev.yml o docker-compose.prod.yml
# ═══════════════════════════════════════════════════════════

# ── Redes ──────────────────────────────────────────────────
# Dos redes aisladas para segmentación de servicios
networks:
  frontend_net:
    driver: bridge
    # El frontend y el backend comparten esta red
  backend_net:
    driver: bridge
    # Solo el backend y la base de datos

# ── Volúmenes ─────────────────────────────────────────────
# Named volumes gestionados por Docker
volumes:
  postgres_data:
    driver: local
    # Persistencia de datos PostgreSQL
    # Ubicación física: /var/lib/docker/volumes/project_postgres_data/_data

# ── Servicios ─────────────────────────────────────────────
services:

  # ╔═══════════════════════════════════════════════════════╗
  # ║  FRONTEND — React + Nginx                            ║
  # ╚═══════════════════════════════════════════════════════╝
  frontend:
    build:
      context: ./frontend           # Directorio con el Dockerfile
      dockerfile: Dockerfile        # Dockerfile de producción por defecto
      args:
        VITE_API_URL: /api          # Se embebe en el bundle de React
    container_name: app-frontend    # Nombre fijo para facilitar referencia
    restart: unless-stopped         # Reinicio automático excepto parada manual
    ports:
      - "${FRONTEND_PORT:-80}:80"   # Mapeado de puerto con default
    networks:
      - frontend_net                # Solo red frontend
    depends_on:
      backend:
        condition: service_healthy  # Esperar a que backend pase healthcheck
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1",
             "--spider", "http://localhost:80/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 128M              # Nginx usa muy poca memoria
          cpus: "0.25"

  # ╔═══════════════════════════════════════════════════════╗
  # ║  BACKEND — FastAPI + Gunicorn/Uvicorn                ║
  # ╚═══════════════════════════════════════════════════════╝
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app-backend
    restart: unless-stopped
    # NO exponemos el puerto al host en producción
    # El backend solo es accesible vía Nginx (frontend_net)
    expose:
      - "8000"                      # Puerto interno, no mapeado al host
    networks:
      - frontend_net                # Para recibir peticiones de Nginx
      - backend_net                 # Para conectar con PostgreSQL
    depends_on:
      db:
        condition: service_healthy  # Esperar a que PostgreSQL esté listo
    environment:
      # Variables inyectadas desde .env
      - POSTGRES_HOST=db            # Nombre del servicio Docker (DNS interno)
      - POSTGRES_PORT=5432
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - SECRET_KEY=${SECRET_KEY}
      - DEBUG=${DEBUG:-false}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s            # Dar tiempo a Gunicorn para iniciar workers
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  # ╔═══════════════════════════════════════════════════════╗
  # ║  DATABASE — PostgreSQL 16                            ║
  # ╚═══════════════════════════════════════════════════════╝
  db:
    image: postgres:16-alpine       # Alpine = imagen más pequeña (~80 MB)
    container_name: app-db
    restart: unless-stopped
    # NO usar "ports:" — la BD no se expone al host
    expose:
      - "5432"                      # Solo accesible dentro de backend_net
    networks:
      - backend_net                 # SOLO red backend, aislada del frontend
    volumes:
      - postgres_data:/var/lib/postgresql/data          # Datos persistentes
      - ./database/init:/docker-entrypoint-initdb.d     # Scripts de inicio
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      # Optimizaciones de rendimiento
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    # Parámetros de PostgreSQL inyectados como comando
    command:
      - "postgres"
      - "-c" 
      - "max_connections=100"
      - "-c"
      - "shared_buffers=256MB"
      - "-c"
      - "log_statement=mod"         # Log INSERT/UPDATE/DELETE (no SELECT)
      - "-c"
      - "log_min_duration_statement=200"  # Log queries > 200ms
```

## 7.2 Explicación Detallada

### `networks` — Segmentación de Red

```yaml
networks:
  frontend_net:
    driver: bridge
  backend_net:
    driver: bridge
```

Creamos dos redes bridge. `frontend` pertenece solo a `frontend_net`. `db` pertenece solo a `backend_net`. `backend` pertenece a ambas, actuando como puente controlado. Esto implementa el principio de **menor privilegio a nivel de red**: un contenedor frontend comprometido no puede alcanzar PostgreSQL.

### `depends_on` con `condition: service_healthy`

```yaml
depends_on:
  db:
    condition: service_healthy
```

Sin `condition: service_healthy`, `depends_on` solo espera a que el contenedor ARRANQUE (estado `running`), no a que el servicio esté LISTO. PostgreSQL puede tardar 5-15 segundos en inicializar. Sin healthcheck, FastAPI intenta conectarse a un PostgreSQL que aún no acepta conexiones y falla con `ConnectionRefused`.

El healthcheck de PostgreSQL usa `pg_isready`, una herramienta incluida en la imagen oficial que verifica si el servidor acepta conexiones. Solo cuando retorna exitosamente, Docker considera el servicio `healthy` y permite que los servicios dependientes arranquen.

### `expose` vs `ports`

```yaml
# Backend
expose:
  - "8000"    # Puerto visible SOLO dentro de las redes Docker

# Frontend
ports:
  - "80:80"   # Puerto mapeado al HOST (accesible desde Internet)
```

`expose` documenta el puerto interno y lo hace accesible entre contenedores en la misma red, pero NO lo mapea al host. `ports` mapea `host:contenedor` y lo hace accesible desde fuera de Docker.

**Regla de producción:** Solo el frontend (Nginx) debe usar `ports`. El backend y la base de datos usan `expose` exclusivamente. Si necesitas acceso temporal a PostgreSQL para debugging, usa `docker compose exec db psql` en lugar de exponer el puerto.

### `restart: unless-stopped`

El contenedor se reinicia automáticamente ante caídas, incluido un reinicio del servidor host. Pero si un operador ejecuta `docker compose stop` para mantenimiento, el contenedor NO se reinicia automáticamente hasta que se ejecute `docker compose start`.

### `deploy.resources.limits`

```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: "1.0"
```

Limita los recursos que el contenedor puede consumir. Esto previene que un servicio con memory leak consuma toda la RAM del host y mate a los demás. En producción, estos límites son obligatorios.

## 7.3 Override de Desarrollo

```yaml
# docker-compose.dev.yml
# ═══════════════════════════════════════════════════════════
# Override para desarrollo local
# Uso: docker compose -f docker-compose.yml -f docker-compose.dev.yml up
# ═══════════════════════════════════════════════════════════

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev    # Dockerfile de desarrollo
    ports:
      - "5173:5173"                 # Puerto de Vite dev server
    volumes:
      - ./frontend/src:/app/src     # Hot reload del código
      - ./frontend/public:/app/public
      - /app/node_modules           # Excluir node_modules del bind mount
    environment:
      - NODE_ENV=development
      - VITE_API_URL=http://localhost:8000  # En dev, sin proxy Nginx

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"                 # Exponer backend directamente en dev
    volumes:
      - ./backend/app:/app/app      # Hot reload del código Python
      - ./backend/tests:/app/tests
      - ./backend/alembic:/app/alembic
    environment:
      - DEBUG=true
      - ALLOWED_ORIGINS=["http://localhost:5173"]

  db:
    ports:
      - "5432:5432"                 # Exponer PostgreSQL para herramientas locales
                                    # (pgAdmin, DBeaver, etc.)
```

**`/app/node_modules` como volumen anónimo:**

```yaml
volumes:
  - ./frontend/src:/app/src
  - /app/node_modules           # ← Volumen anónimo
```

Esto es un patrón importante. El bind mount `./frontend/src:/app/src` monta el código del host. Pero si el host tiene un directorio `node_modules` diferente (o vacío), pisaría el `node_modules` instalado dentro del contenedor. El volumen anónimo `/app/node_modules` "protege" ese directorio: Docker lo gestiona internamente y no se ve afectado por el bind mount del host.

## 7.4 Override de Producción

```yaml
# docker-compose.prod.yml
# ═══════════════════════════════════════════════════════════
# Override para producción
# Uso: docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
# ═══════════════════════════════════════════════════════════

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile        # Multi-stage con Nginx
      args:
        VITE_API_URL: /api
    ports:
      - "80:80"
      - "443:443"                   # Si se configura TLS en Nginx

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile        # Con Gunicorn
    # Sin ports: — solo accesible vía Nginx
    environment:
      - DEBUG=false

  db:
    # Sin ports: — aislado completamente
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # Contraseña fuerte de producción
```

## 7.5 Archivo .env

```bash
# .env — Variables de entorno
# NUNCA commitear este archivo. Usar .env.example como template.

# ── PostgreSQL ─────────────────────────────────────────────
POSTGRES_USER=appuser
POSTGRES_PASSWORD=S3cur3_P@ssw0rd_Ch4ng3_M3!
POSTGRES_DB=appdb

# ── Backend ────────────────────────────────────────────────
SECRET_KEY=your-256-bit-secret-key-here-generate-with-openssl
DEBUG=false

# ── Frontend ───────────────────────────────────────────────
FRONTEND_PORT=80

# ── Compose ────────────────────────────────────────────────
COMPOSE_PROJECT_NAME=myapp
```

```bash
# .env.example — Template (se commitea al repositorio)
POSTGRES_USER=appuser
POSTGRES_PASSWORD=CHANGE_ME
POSTGRES_DB=appdb
SECRET_KEY=CHANGE_ME
DEBUG=false
FRONTEND_PORT=80
COMPOSE_PROJECT_NAME=myapp
```

---

# 8. Comunicación entre Servicios

## 8.1 DNS Interno de Docker

Docker Compose incluye un servidor DNS embebido (128.0.0.11) que resuelve los nombres de servicio definidos en `docker-compose.yml` a las direcciones IP de los contenedores.

```
┌─────────────────────────────────────────────┐
│  Docker DNS (128.0.0.11)                    │
│                                             │
│  "frontend"  → 172.18.0.2                  │
│  "backend"   → 172.18.0.3                  │
│  "db"        → 172.18.0.4                  │
│                                             │
│  También resuelve:                          │
│  "app-frontend"  → 172.18.0.2  (container) │
│  "app-backend"   → 172.18.0.3  (container) │
│  "app-db"        → 172.18.0.4  (container) │
└─────────────────────────────────────────────┘
```

Cuando FastAPI necesita conectarse a PostgreSQL, usa `db` como hostname:

```
postgresql://appuser:password@db:5432/appdb
                               ^^
                    Nombre del servicio en docker-compose.yml
```

Docker resuelve `db` a la IP del contenedor PostgreSQL. Esto funciona automáticamente sin ninguna configuración adicional, siempre que ambos contenedores estén en la misma red Docker.

**Punto crítico:** El DNS solo resuelve nombres dentro de la misma red Docker. Como `frontend` y `db` están en redes diferentes (`frontend_net` y `backend_net`), el frontend NO puede resolver `db`. Esto es intencional.

## 8.2 Conexión FastAPI → PostgreSQL

La conexión usa SQLAlchemy con el connection string que referencia el servicio `db`:

```python
# La variable POSTGRES_HOST=db se define en docker-compose.yml
DATABASE_URL = "postgresql://appuser:password@db:5432/appdb"
```

**Pool de conexiones:** SQLAlchemy mantiene un pool de conexiones persistentes. Esto evita el overhead de crear una nueva conexión TCP + handshake SSL + autenticación PostgreSQL en cada petición HTTP. Con `pool_size=20` y `max_overflow=10`, el backend puede manejar hasta 30 conexiones simultáneas.

**`pool_pre_ping=True`:** Antes de entregar una conexión del pool a la aplicación, SQLAlchemy envía un `SELECT 1` para verificar que la conexión sigue viva. Esto previene el error "connection reset by peer" cuando PostgreSQL cierra conexiones idle.

## 8.3 Conexión React → FastAPI

### En Producción (con Nginx proxy)

```
Navegador                      Docker
   │                             │
   │  GET /api/users             │
   ├────────────────────────────►│ Nginx (frontend:80)
   │                             │    │
   │                             │    │ proxy_pass http://backend:8000/
   │                             │    │
   │                             │    ▼
   │                             │ FastAPI (backend:8000)
   │                             │    │
   │  HTTP 200 [{...}]          │    │
   │◄────────────────────────────┤    │
```

React hace `fetch("/api/users")`. Nginx intercepta la ruta `/api/*` y reenvía la petición a `http://backend:8000`. Como la petición va al mismo dominio y puerto que sirvió la página, **no hay problema de CORS**.

Configuración del cliente HTTP en React:

```javascript
// frontend/src/services/api.js
const API_BASE_URL = import.meta.env.VITE_API_URL || "/api";

export async function fetchFromAPI(endpoint, options = {}) {
  const url = `${API_BASE_URL}${endpoint}`;
  
  const response = await fetch(url, {
    headers: {
      "Content-Type": "application/json",
      ...options.headers,
    },
    ...options,
  });

  if (!response.ok) {
    throw new Error(`API Error: ${response.status}`);
  }

  return response.json();
}

// Uso:
// fetchFromAPI("/users")         →  GET /api/users
// fetchFromAPI("/users", {
//   method: "POST",
//   body: JSON.stringify(data),
// })
```

### En Desarrollo (sin Nginx proxy)

En desarrollo, React se ejecuta en Vite dev server (puerto 5173) y el backend en Uvicorn (puerto 8000). Son dominios diferentes (`localhost:5173` vs `localhost:8000`), por lo que **CORS es obligatorio**.

```python
# backend/app/main.py — Solo en modo DEBUG
if settings.DEBUG:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:5173"],  # Origen exacto de Vite
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

Y en React:

```javascript
// En desarrollo, VITE_API_URL=http://localhost:8000
// Definido en docker-compose.dev.yml
const API_BASE_URL = import.meta.env.VITE_API_URL || "/api";
```

## 8.4 Manejo de CORS: Guía Definitiva

CORS (Cross-Origin Resource Sharing) es un mecanismo de seguridad del navegador que bloquea peticiones entre diferentes orígenes. Un "origen" se compone de protocolo + dominio + puerto.

```
Mismo origen (CORS no aplica):
  http://miapp.com/         (página)
  http://miapp.com/api/users (API)
  → Mismo protocolo, dominio y puerto

Diferente origen (CORS requerido):
  http://localhost:5173     (React dev server)
  http://localhost:8000     (FastAPI)
  → Mismo protocolo y dominio, DIFERENTE puerto
```

**La regla de oro:** En producción, configura Nginx como proxy para que frontend y API estén en el mismo origen. Esto elimina la necesidad de CORS por completo. CORS solo debe habilitarse en desarrollo.

**Errores comunes:**
- `allow_origins=["*"]` en producción: Esto permite que CUALQUIER sitio web haga peticiones a tu API. Un atacante podría hacer que el navegador de un usuario envíe peticiones autenticadas a tu API desde un sitio malicioso.
- Olvidar `allow_credentials=True` cuando usas cookies o tokens en headers.
- No incluir los headers `Authorization` en `allow_headers`.

---

# 9. Modo Desarrollo vs Producción

## 9.1 Comparación Completa

| Aspecto | Desarrollo | Producción |
|---------|------------|------------|
| **Frontend server** | Vite dev server (HMR) | Nginx sirve build estático |
| **Backend server** | Uvicorn con `--reload` | Gunicorn + Uvicorn workers |
| **Código** | Bind mount (edición en vivo) | COPY en imagen (inmutable) |
| **Debug** | Habilitado, Swagger visible | Deshabilitado, sin docs |
| **CORS** | Habilitado (puertos diferentes) | No necesario (Nginx proxy) |
| **DB acceso** | Puerto 5432 expuesto al host | Sin exposición al host |
| **Logs SQL** | `echo=True` (todas las queries) | Solo queries > 200ms |
| **Restart policy** | `no` (manual) | `unless-stopped` |
| **Recursos** | Sin límites | CPU/RAM limitados |
| **Imágenes** | Dockerfile.dev (herramientas) | Dockerfile (multi-stage, slim) |
| **Variables** | DEBUG=true | DEBUG=false |

## 9.2 Comandos de Arranque

```bash
# ── Desarrollo ─────────────────────────────────────────────
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# ── Producción ─────────────────────────────────────────────
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# ── Detener (preservando datos) ────────────────────────────
docker compose down

# ── Detener Y eliminar volúmenes (DESTRUYE DATOS) ─────────
docker compose down -v
```

## 9.3 Makefile para Simplificar

```makefile
# Makefile
.PHONY: dev prod down logs backup migrate test

# ── Desarrollo ─────────────────────────────────────────────
dev:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

dev-detached:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build -d

# ── Producción ─────────────────────────────────────────────
prod:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# ── Gestión ────────────────────────────────────────────────
down:
	docker compose down

destroy:
	docker compose down -v --rmi all

restart:
	docker compose restart

logs:
	docker compose logs -f --tail=100

logs-backend:
	docker compose logs -f backend

logs-db:
	docker compose logs -f db

# ── Base de datos ──────────────────────────────────────────
migrate:
	docker compose exec backend alembic upgrade head

migrate-create:
	docker compose exec backend alembic revision --autogenerate -m "$(msg)"

backup:
	./scripts/backup-db.sh

psql:
	docker compose exec db psql -U $(POSTGRES_USER) -d $(POSTGRES_DB)

# ── Testing ────────────────────────────────────────────────
test:
	docker compose exec backend pytest -v

test-coverage:
	docker compose exec backend pytest --cov=app --cov-report=html

# ── Utilidades ─────────────────────────────────────────────
shell-backend:
	docker compose exec backend bash

shell-frontend:
	docker compose exec frontend sh

status:
	docker compose ps
	@echo ""
	@docker compose top
```

---

# 10. Seguridad y Buenas Prácticas

## 10.1 Variables Secretas

**Nunca hardcodear credenciales en `docker-compose.yml` ni en Dockerfiles.**

```yaml
# ❌ NUNCA hacer esto
environment:
  POSTGRES_PASSWORD: mi_password_insegura

# ✅ Referencia a .env
environment:
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Para producción avanzada, usar Docker Secrets o un vault externo (HashiCorp Vault, AWS Secrets Manager):

```yaml
# docker-compose.prod.yml con secrets (Docker Swarm o workaround)
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

## 10.2 Usuarios No Root

Todos los Dockerfiles deben crear y usar un usuario sin privilegios:

```dockerfile
# Python
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Alpine/Node
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Nginx (imagen oficial ya tiene usuario 'nginx')
USER nginx
```

**¿Por qué importa?** Si un atacante explota una vulnerabilidad en la aplicación y obtiene ejecución de código dentro del contenedor, con usuario root podría:
- Montar el filesystem del host (si tiene capabilities)
- Modificar binarios del sistema
- Escalar privilegios

Con usuario no-root, el daño es limitado al contexto de ese usuario.

## 10.3 Reducción de Superficie de Ataque

```dockerfile
# Usar imágenes slim/alpine (no full)
FROM python:3.12-slim    # ~150 MB vs ~1 GB de python:3.12
FROM node:20-alpine      # ~130 MB vs ~1 GB de node:20
FROM postgres:16-alpine  # ~80 MB vs ~400 MB de postgres:16

# Eliminar caché de paquetes
RUN apt-get update && \
    apt-get install -y --no-install-recommends paquete && \
    rm -rf /var/lib/apt/lists/*

# No instalar herramientas de debug en producción
# (curl se necesita para healthcheck, pero nada más)
```

## 10.4 No Exponer la Base de Datos

```yaml
# ❌ NUNCA en producción
db:
  ports:
    - "5432:5432"    # Accesible desde Internet

# ✅ Solo accesible dentro de Docker
db:
  expose:
    - "5432"         # Solo contenedores en la misma red
```

Si necesitas acceso de emergencia a PostgreSQL en producción:

```bash
# Usar exec en lugar de exponer puertos
docker compose exec db psql -U appuser -d appdb

# O crear un túnel SSH temporal
ssh -L 5432:localhost:5432 user@production-server
```

## 10.5 Escaneo de Vulnerabilidades

```bash
# Escanear imagen con Docker Scout
docker scout cves app-backend:latest

# Escanear con Trivy (herramienta de Aqua Security)
docker run --rm \
    -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image app-backend:latest
```

## 10.6 .dockerignore Completo

```
# .dockerignore global
.git
.gitignore
.env
.env.*
*.md
LICENSE
docker-compose*.yml
.vscode
.idea
__pycache__
*.pyc
.pytest_cache
.mypy_cache
node_modules
dist
build
coverage
.coverage
*.log
```

---

# 11. Automatización y DevOps

## 11.1 Comandos Esenciales de Docker Compose

```bash
# ── Ciclo de vida ──────────────────────────────────────────
docker compose up                     # Arrancar todos los servicios
docker compose up -d                  # Arrancar en background
docker compose up --build             # Reconstruir imágenes antes de arrancar
docker compose down                   # Parar y eliminar contenedores
docker compose down -v                # + eliminar volúmenes (DATOS)
docker compose down --rmi all         # + eliminar imágenes
docker compose restart                # Reiniciar todos
docker compose restart backend        # Reiniciar solo uno

# ── Observación ────────────────────────────────────────────
docker compose ps                     # Estado de servicios
docker compose logs -f                # Logs en tiempo real
docker compose logs -f backend        # Logs de un servicio
docker compose logs --tail=50 backend # Últimas 50 líneas
docker compose top                    # Procesos por contenedor
docker compose stats                  # Uso de recursos en vivo

# ── Ejecución ──────────────────────────────────────────────
docker compose exec backend bash      # Shell interactivo
docker compose exec db psql -U appuser -d appdb  # PostgreSQL CLI
docker compose run --rm backend pytest  # Ejecutar comando y eliminar contenedor

# ── Build ──────────────────────────────────────────────────
docker compose build                  # Build de todas las imágenes
docker compose build --no-cache       # Build limpio (sin caché)
docker compose build backend          # Build de un solo servicio

# ── Escalado ───────────────────────────────────────────────
docker compose up -d --scale backend=3  # 3 instancias del backend
```

## 11.2 Profiles de Compose

Los profiles permiten definir servicios opcionales que solo arrancan cuando se activan:

```yaml
# En docker-compose.yml
services:
  # Servicios core (siempre arrancan)
  frontend:
    # ...
  backend:
    # ...
  db:
    # ...

  # Servicios opcionales
  pgadmin:
    image: dpage/pgadmin4:latest
    profiles: ["debug"]           # Solo arranca con profile "debug"
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    networks:
      - backend_net

  redis:
    image: redis:7-alpine
    profiles: ["cache"]           # Solo arranca con profile "cache"
    expose:
      - "6379"
    networks:
      - backend_net

  mailhog:
    image: mailhog/mailhog
    profiles: ["debug"]
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - backend_net
```

```bash
# Arrancar solo servicios core
docker compose up -d

# Arrancar core + herramientas de debug
docker compose --profile debug up -d

# Arrancar core + caché
docker compose --profile cache up -d

# Arrancar todo
docker compose --profile debug --profile cache up -d
```

## 11.3 Script de Arranque Inteligente

```bash
#!/bin/bash
# scripts/start-dev.sh

set -euo pipefail

# Colores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}🚀 Iniciando entorno de desarrollo...${NC}"

# Verificar que Docker está corriendo
if ! docker info > /dev/null 2>&1; then
    echo -e "${RED}❌ Docker no está corriendo. Inícialo primero.${NC}"
    exit 1
fi

# Verificar que .env existe
if [ ! -f .env ]; then
    echo -e "${YELLOW}⚠️  Archivo .env no encontrado. Copiando .env.example...${NC}"
    cp .env.example .env
    echo -e "${YELLOW}   → Edita .env con tus valores antes de continuar.${NC}"
    exit 1
fi

# Verificar que los puertos no están ocupados
for port in 5173 8000 5432; do
    if lsof -Pi :$port -sTCP:LISTEN -t > /dev/null 2>&1; then
        echo -e "${RED}❌ Puerto $port ya está en uso.${NC}"
        exit 1
    fi
done

# Construir y arrancar
echo -e "${GREEN}📦 Construyendo imágenes...${NC}"
docker compose -f docker-compose.yml -f docker-compose.dev.yml build

echo -e "${GREEN}🏗️  Levantando servicios...${NC}"
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Esperar a que los servicios estén healthy
echo -e "${YELLOW}⏳ Esperando a que los servicios estén listos...${NC}"
timeout 120 bash -c 'until docker compose ps | grep -q "healthy"; do sleep 2; done'

# Ejecutar migraciones
echo -e "${GREEN}🗃️  Ejecutando migraciones...${NC}"
docker compose exec -T backend alembic upgrade head

echo ""
echo -e "${GREEN}✅ Entorno de desarrollo listo!${NC}"
echo -e "   Frontend: http://localhost:5173"
echo -e "   Backend:  http://localhost:8000"
echo -e "   API Docs: http://localhost:8000/docs"
echo -e "   DB:       localhost:5432"
echo ""
echo -e "   Logs: docker compose logs -f"
echo -e "   Parar: docker compose down"
```

## 11.4 CI/CD Básico con GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository }}

jobs:
  # ── Tests ────────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create .env for testing
        run: |
          cat > .env << EOF
          POSTGRES_USER=testuser
          POSTGRES_PASSWORD=testpassword
          POSTGRES_DB=testdb
          SECRET_KEY=test-secret-key-not-for-production
          DEBUG=true
          EOF

      - name: Build and start services
        run: |
          docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
          docker compose exec -T backend alembic upgrade head

      - name: Run backend tests
        run: docker compose exec -T backend pytest -v --tb=short

      - name: Run frontend tests
        run: docker compose exec -T frontend npm test -- --run

      - name: Cleanup
        if: always()
        run: docker compose down -v

  # ── Build y Push de imágenes ─────────────────────────────
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}-backend:latest

      - name: Build and push Frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}-frontend:latest

  # ── Deploy a VPS ─────────────────────────────────────────
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
            docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
            docker image prune -f
```

## 11.5 Despliegue en VPS

### Preparación del servidor

```bash
# En el VPS (Ubuntu 24.04)
# 1. Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 2. Instalar Docker Compose plugin
sudo apt-get install docker-compose-plugin

# 3. Configurar firewall
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# 4. Clonar repositorio
git clone https://github.com/user/myapp.git /opt/myapp
cd /opt/myapp

# 5. Configurar .env de producción
cp .env.example .env
nano .env  # Editar con credenciales de producción

# 6. Arrancar
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

---

# 12. Problemas Reales y Troubleshooting

## 12.1 Contenedores que No Arrancan

### Síntoma: El contenedor se reinicia en bucle

```bash
# Diagnóstico
docker compose ps                    # Verás "Restarting" o "Exit(1)"
docker compose logs backend          # Ver el error específico
```

**Causas comunes:**
- Falta una variable de entorno requerida (`SECRET_KEY`, `POSTGRES_PASSWORD`)
- El archivo `main.py` tiene un import error
- Puerto ya ocupado por otro proceso

```bash
# Verificar variables
docker compose exec backend env | grep POSTGRES

# Verificar puertos
sudo lsof -i :8000
```

### Síntoma: `ModuleNotFoundError` en el backend

El Dockerfile no instaló las dependencias correctamente, o el bind mount pisó el directorio de dependencias.

```bash
# Reconstruir sin caché
docker compose build --no-cache backend
```

## 12.2 Problemas de Red

### Síntoma: `Connection refused` entre servicios

```
sqlalchemy.exc.OperationalError:
  could not connect to server: Connection refused
  Is the server running on host "db" (172.18.0.4) and
  accepting TCP/IP connections on port 5432?
```

**Causas y soluciones:**

1. **PostgreSQL aún no está listo.** El healthcheck con `condition: service_healthy` debería prevenir esto. Si no lo usas, el backend intenta conectar antes de que PostgreSQL esté inicializado.

2. **Los servicios están en redes diferentes.** Verifica que backend y db comparten `backend_net`.

3. **El hostname es incorrecto.** Debe ser el nombre del servicio en `docker-compose.yml` (`db`), no `localhost` ni una IP hardcodeada.

```bash
# Verificar conectividad desde el backend
docker compose exec backend ping db
docker compose exec backend curl -v telnet://db:5432

# Verificar redes
docker network ls
docker network inspect myapp_backend_net
```

### Síntoma: Frontend no puede alcanzar el backend

```
GET http://localhost:8000/api/users → ERR_CONNECTION_REFUSED
```

**En desarrollo:** Verificar que el backend expone el puerto (`ports: "8000:8000"`).

**En producción:** El frontend no debe llamar a `localhost:8000`. Debe llamar a `/api/users` (ruta relativa) y Nginx hace el proxy. Verificar que `VITE_API_URL=/api` está configurado correctamente en build time.

## 12.3 Race Conditions con la Base de Datos

### El problema clásico

```
backend_1  | sqlalchemy.exc.OperationalError: FATAL:
backend_1  |   database "appdb" does not exist
```

FastAPI arranca antes de que PostgreSQL haya terminado de crear la base de datos. Aunque `depends_on` garantiza que el contenedor de PostgreSQL se inicia primero, no garantiza que el servicio PostgreSQL dentro del contenedor esté listo.

### Solución definitiva: healthcheck + depends_on condition

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    timeout: 5s
    retries: 10
    start_period: 30s

backend:
  depends_on:
    db:
      condition: service_healthy
```

### Solución complementaria: retry en la aplicación

```python
# backend/app/database.py
import time
from sqlalchemy import create_engine
from sqlalchemy.exc import OperationalError

def create_engine_with_retry(url, max_retries=5, delay=2):
    """Intenta conectar con reintentos exponenciales."""
    for attempt in range(max_retries):
        try:
            engine = create_engine(url)
            # Verificar conexión
            with engine.connect() as conn:
                conn.execute(text("SELECT 1"))
            return engine
        except OperationalError:
            if attempt < max_retries - 1:
                wait = delay * (2 ** attempt)  # Backoff exponencial
                print(f"DB no disponible. Reintentando en {wait}s...")
                time.sleep(wait)
            else:
                raise
```

## 12.4 Migraciones Fallidas

### Problema: Alembic no puede conectar a la BD

```bash
# Ejecutar migraciones manualmente
docker compose exec backend alembic upgrade head

# Si falla, verificar la URL de conexión
docker compose exec backend python -c "
from app.config import get_settings
print(get_settings().database_url)
"
```

### Problema: Conflicto de migraciones

Dos desarrolladores crearon migraciones en paralelo, generando dos "heads" en el historial de Alembic.

```bash
# Ver el historial
docker compose exec backend alembic history

# Merge de heads
docker compose exec backend alembic merge heads -m "merge branches"
```

### Problema: La migración es irreversible

```bash
# Ver estado actual
docker compose exec backend alembic current

# Downgrade a una revisión específica
docker compose exec backend alembic downgrade abc123

# Downgrade un paso
docker compose exec backend alembic downgrade -1
```

## 12.5 Errores Comunes de Principiantes

### 1. "Mis cambios no se reflejan"

**Causa:** No se reconstruyó la imagen después de cambiar el Dockerfile o las dependencias.

```bash
# SIEMPRE usar --build al cambiar Dockerfiles o requirements
docker compose up --build
```

### 2. "El contenedor tiene datos viejos de la BD"

**Causa:** El volumen persiste los datos. Los scripts de `initdb.d` solo se ejecutan cuando el volumen está vacío.

```bash
# Para reiniciar la BD desde cero
docker compose down -v    # ¡Elimina TODOS los volúmenes!
docker compose up --build
```

### 3. "node_modules da errores de módulos nativos"

**Causa:** Se copió `node_modules` del host (macOS/Windows) al contenedor (Linux). Las extensiones nativas son incompatibles.

**Solución:** Agregar `node_modules` a `.dockerignore` y al volumen anónimo:

```yaml
volumes:
  - ./frontend:/app
  - /app/node_modules    # Protege node_modules del bind mount
```

### 4. "Los logs no aparecen"

**Causa para Python:** Falta `PYTHONUNBUFFERED=1`.  
**Causa para Node:** La aplicación escribe a un archivo en lugar de stdout.

**Regla:** En contenedores, SIEMPRE escribir logs a stdout/stderr. Docker los captura automáticamente. No usar archivos de log dentro del contenedor.

### 5. "El build tarda una eternidad"

**Causa:** El orden de instrucciones en el Dockerfile invalida el caché frecuentemente.

```dockerfile
# ❌ Mal: cualquier cambio en el código invalida la caché de dependencias
COPY . .
RUN pip install -r requirements.txt

# ✅ Bien: las dependencias se cachean independientemente del código
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 6. "Permission denied en volúmenes"

**Causa:** El usuario dentro del contenedor (e.g., UID 1000) no tiene permisos sobre los archivos montados del host.

```bash
# Verificar UIDs
docker compose exec backend id
# uid=1000(appuser) gid=1000(appuser)

ls -la ./backend/app/
# Si los archivos son de root en el host, el contenedor no puede leerlos
```

**Solución:** Asegurar que el UID del usuario en el contenedor coincide con el UID del usuario del host, o usar `chown` en el Dockerfile.

### 7. "Disco lleno"

Docker acumula imágenes, contenedores detenidos y volúmenes huérfanos.

```bash
# Ver uso de disco
docker system df

# Limpiar TODO lo no utilizado (imágenes, contenedores, redes, caché)
docker system prune -a --volumes

# Limpiar solo imágenes sin usar
docker image prune -a

# Limpiar solo volúmenes huérfanos
docker volume prune
```

---

# Apéndice A: Referencia Rápida de Comandos

```bash
# ── Desarrollo ─────────────────────────────────────────────
make dev                              # Arrancar desarrollo
make logs                             # Ver todos los logs
make logs-backend                     # Ver logs del backend
make psql                             # Conectar a PostgreSQL
make migrate                          # Ejecutar migraciones
make test                             # Ejecutar tests

# ── Producción ─────────────────────────────────────────────
make prod                             # Arrancar producción
make backup                           # Backup de la BD

# ── Limpieza ───────────────────────────────────────────────
make down                             # Parar servicios
make destroy                          # Eliminar todo
docker system prune -a                # Liberar disco
```

# Apéndice B: Checklist Pre-Deploy

```
□ .env configurado con credenciales de producción
□ .env NO está en el repositorio (verificar .gitignore)
□ SECRET_KEY generado con: openssl rand -hex 32
□ POSTGRES_PASSWORD es fuerte (20+ caracteres aleatorios)
□ DEBUG=false
□ Swagger/ReDoc desactivado (docs_url=None)
□ CORS configurado con orígenes específicos (no "*")
□ PostgreSQL NO expone puertos al host
□ Imágenes construidas con multi-stage build
□ Usuarios no-root en todos los contenedores
□ Healthchecks configurados en todos los servicios
□ Restart policy: unless-stopped
□ Límites de recursos (CPU/RAM) definidos
□ Volúmenes de datos con named volumes
□ Backups automatizados
□ Logs van a stdout/stderr
□ Firewall configurado (solo 80/443 abiertos)
□ TLS/SSL configurado (Let's Encrypt + Nginx)
```
