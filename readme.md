# 🧱 Guía de setup para proyectos de Lambda Works

---

## 📌 Índice

1. Setup repo  
1.1 CI/CD — GitHub Actions  
2. Frontend (Next.js)  
3. Backend (NestJS + Prisma + PostgreSQL) — incluye PostgreSQL + Prisma  
4. Docker Compose  
5. Workspaces  
6. Estructura final (repo)  
7. `.gitignore`  
8. VPS — estructura, releases, puertos y Docker  
9. HestiaCP — proxy, dominio, SSL, DNS, error Let’s Encrypt  
10. Deploy en el VPS (releases + build monorepo)  
11. Entorno de testing

---

> **Arquitectura por defecto de esta guía:** frontend (Next.js) y backend (NestJS) en el **mismo VPS** — Hestia (proxy + SSL) y Docker Compose.

## 1. 🚀 Setup repo

Crear repo con nombre `<nombre-proyecto>` en github [Crear repositorio](https://github.com/new).

Ahora en local:
```bash
mkdir <nombre-proyecto>
cd <nombre-proyecto>
git init
mkdir -p .github/workflows
cp /ruta/al/template/ci.yml .github/workflows/ci.yml
echo "# <nombre-proyecto>" >> README.md
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:Lambda-Works/<nombre-proyecto>.git
git push -u origin main

# Crear y pushear ramas principales
git checkout -b develop
git push -u origin develop
git checkout -b production
git push -u origin production
git checkout develop
```

> **Ramas del proyecto:** cada repositorio debe contener al menos tres ramas principales:
> - `main` — código estable de producción.
> - `develop` — desarrollo activo (se deploya al entorno de testing).
> - `production` — reflejo exacto de lo que está en el VPS de producción (gestionada por el Hub y GitHub Actions).

## 1.1 🔄 CI/CD — GitHub Actions

Crear el archivo `.github/workflows/ci.yml` con el siguiente contenido:

```yaml
name: CI - Docker Compose Dev

env:
  FRONTEND_PATH: frontend
  BACKEND_PATH: backend
  BACKEND_PORT: 3001
  FRONTEND_PORT: 3000
  HEALTH_ENDPOINT: /health
  POSTGRES_SERVICE: postgres

on:
  push:
    branches: [develop]
  pull_request:
    branches: [develop, main]

jobs:
  build-and-test:
    name: Build & Verify Dev Environment
    runs-on: ubuntu-latest

    steps:
      # 1. Descargar el código del repositorio
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Copiar el archivo de variables de entorno
      - name: Setup environment variables
        run: cp .env.example .env

      # 3. Verificar build de producción del Frontend (detecta errores de compilación)
      - name: Verify Frontend production build
        run: |
          echo "Building Frontend with Dockerfile to verify no compile errors..."
          docker build -f ${{ env.FRONTEND_PATH }}/Dockerfile -t frontend-build-check .
          echo "✓ Frontend production build successful"

      # 4. Levantar los servicios con Docker Compose y esperar healthchecks
      - name: Build and start Docker Compose services
        run: docker compose up -d --build --wait

      # 5. Esperar a que los servicios arranquen
      - name: Wait for services to start
        run: |
          echo "Waiting 15 seconds for Backend and Frontend to initialize..."
          sleep 15

      # 6. Verificar que el Backend responde (healthcheck)
      - name: Verify Backend is running
        run: |
          echo "Checking Backend at http://localhost:${{ env.BACKEND_PORT }}${{ env.HEALTH_ENDPOINT }} ..."
          curl -f -s --max-time 30 \
            -o /dev/null \
            -w "Backend Response: HTTP %{http_code}\n" \
            http://localhost:${{ env.BACKEND_PORT }}${{ env.HEALTH_ENDPOINT }}

      # 7. Verificar que el Frontend responde (siguiendo redirects)
      - name: Verify Frontend is running
        run: |
          echo "Checking Frontend at http://localhost:${{ env.FRONTEND_PORT }} ..."
          curl -L -f -s --max-time 30 \
            -o /dev/null \
            -w "Frontend Response: HTTP %{http_code}\n" \
            http://localhost:${{ env.FRONTEND_PORT }}

      # 8. Si falló algo, mostrar logs para debug
      - name: Print logs on failure
        if: failure()
        run: |
          echo "========== DOCKER COMPOSE LOGS =========="
          docker compose logs --tail 100

      # 9. Limpiar contenedores y recursos
      - name: Cleanup
        if: always()
        run: docker compose down -v
```

> **Nota:** este workflow se dispara automáticamente en cada push a la rama `develop` y en cada pull request dirigido a `develop` o `main`. Verifica que el frontend compila, levanta los servicios con Docker Compose, espera a que estén healthy y comprueba que ambos respondan por HTTP.

## 2. ⚛️ Frontend (Next.js)


```bash
npx create-next-app@latest frontend --disable-git
```

Respuestas recomendadas del asistente interactivo:

```text
✔ Would you like to use TypeScript? … Yes
✔ Which linter would you like to use? › ESLint
✔ Would you like to use React Compiler? … No
✔ Would you like to use Tailwind CSS? … Yes
✔ Would you like your code inside a `src/` directory? … Yes
✔ Would you like to use App Router? (recommended) … Yes
✔ Would you like to customize the import alias (`@/*` by default)? … No
✔ Would you like to include AGENTS.md to guide coding agents to write up-to-date Next.js code? … Yes
```

Estructura:

```text
/frontend
```

## 3. 🧠 Backend (NestJS + Prisma + PostgreSQL)


```bash
npx @nestjs/cli new backend --skip-git
```

Estructura:

```text
/backend
```
##### 🗄️ Base de datos (PostgreSQL + Prisma)

```bash
cd backend
npm install prisma @prisma/client
npx prisma init
```

##### Comandos de Prisma

**Con `npx` (desde `backend/`):**

1. **`npx prisma generate`** — Regenera el cliente `@prisma/client` a partir de `schema.prisma`. Corrélo después de cambiar el esquema o de clonar el repo.
2. **`npx prisma migrate dev`** — Crea una migración SQL nueva si el esquema cambió, la aplica a la base de desarrollo y vuelve a generar el cliente. Te pide un nombre descriptivo para la migración.
3. **`npx prisma studio`** — Abre Prisma Studio en el navegador (servidor local) para ver y editar registros con la misma conexión que usa la app.


**Uso típico en el día a día**

```bash
cd backend
npx prisma migrate dev    # (2) cuando cambiás modelos en schema.prisma
npx prisma generate       # (1) si solo necesitás el cliente (p. ej. CI o tras pull sin migraciones nuevas)
npx prisma studio         # (3) inspección / datos de prueba
```

**Post-instalación o antes de commitear cambios al esquema (recomendado)**

1. **`npx prisma validate`** — comprueba que `schema.prisma` sea válido.
2. **`npx prisma format`** — formatea el schema.
3. **`npx prisma generate`** — asegura el cliente al día con el schema actual.

```bash
cd backend
npx prisma validate
npx prisma format
npx prisma generate
```

Otros útiles (según necesidad): `npx prisma db pull` (introspección desde una DB existente), `npx prisma migrate deploy` (aplicar migraciones en staging/prod sin modo interactivo). Consultá `npx prisma --help` y `npx prisma <comando> --help`.


## 4. 🐳 Docker Compose

Cada proyecto debe incluir los siguientes archivos en la raíz para poder deployar desde el Hub.

**`docker-compose.yml`** (desarrollo local):

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "3001:3001"
    env_file: ./backend/.env
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    env_file: ./frontend/.env
```

**`docker-compose.prod.yml`** (producción — usado por el Hub):

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "${PUERTO_BACKEND}:3001"
    env_file: ./backend/.env
  frontend:
    build: ./frontend
    ports:
      - "${PUERTO_FRONTEND}:3000"
    env_file: ./frontend/.env
```

> **Importante:** El archivo `docker-compose.prod.yml` es el que el Hub utiliza para ejecutar el deploy en el VPS. Los puertos se inyectan desde el `.env` en `shared/` (symlinkeado como `.env` en el directorio de la release). Docker Compose los lee automáticamente.
>
> **Convenciones por defecto:**
> - Backend escucha en el puerto `3001` dentro del container.
> - Frontend escucha en el puerto `3000` dentro del container.
> - Los puertos externos (`PUERTO_BACKEND`, `PUERTO_FRONTEND`) los asigna `get-ports` y se persisten en `shared/.env`.
> - El backend debe exponer un endpoint `GET /health` que responda HTTP 200 para que el CI pueda verificar que levantó correctamente.

---


## 5. 📦 Workspaces
`package.json` en root del proyecto:

```json
{
  "private": true,
  "workspaces": [
    "frontend",
    "backend"
  ],
  "scripts": {
    "dev": "npm run dev --workspaces",
    "build": "docker compose -f docker-compose.prod.yml build",
    "start": "docker compose -f docker-compose.prod.yml up -d",
    "stop": "docker compose -f docker-compose.prod.yml down"
  }
}
```

> **Nota:** `dev` corre los proyectos en local con npm. Los comandos `build`, `start` y `stop` usan Docker Compose y están pensados para el deploy en el VPS. Ver sección de Docker Compose más abajo.

Instalar todo:

```bash
npm install
```

## 6. 🧩 Estructura final (repo)

```text
<nombre-proyecto>/
  ├── frontend/
  ├── backend/
  ├── docker-compose.yml
  ├── docker-compose.prod.yml
  ├── package.json
  └── .gitignore
```
(Tambien se sugiere crear una carpeta shared pero todavia no encontré motivos para hacerlo)

## 7. 🧹 .gitignore

Base recomendada:

```gitignore
# Dependencias
node_modules/

# Variables de entorno y secretos
.env
.env.*
!.env.example

# Builds y caches
dist/
.next/
coverage/

# Logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Prisma local (solo si usás SQLite)
prisma/dev.db*

# Docker
.dockerignore
docker-compose.override.yml
```

Nota: gran momento para hacer un segundo commit:
```bash
git add .
git commit -m "project setup + ci"
git push
```

## 8. 🖥️ VPS


### 8.1 Árbol recomendado 
(no tocar nada todavía, sólo para ver como vamos a trabajar)
```text
/var/www/<nombre-proyecto>/
  ├── current/          → symlink a la release activa (lo usa Docker)
  ├── releases/         → una carpeta por versión desplegada
  └── shared/           → datos que NO se versionan y sobreviven a cada deploy
```

| Carpeta | Rol |
|--------|-----|
| `releases/` | Cada deploy agrega una carpeta nueva (`release-1`, `release-2`, …). Ahí vive el código de esa versión (checkout del repo + `node_modules` + build). |
| `current/` | **No** es una copia del código: es un **enlace simbólico** a *una* carpeta dentro de `releases/`. Cambiar deploy = cambiar a qué release apunta `current`. |
| `shared/` | `.env` de producción, uploads, certificados locales si aplica, etc. No se borra al publicar una release nueva. |

### 8.2 Setup inicial
Primero logearse en el servidor:
```bash
ssh root@72.61.50.55
```
Ahora si empezamos a crear la estructura:
```bash
mkdir -p /var/www/<nombre-proyecto>/{releases,shared}
```

`current/` **no** se crea con `mkdir`: aparece cuando hagas el primer `ln -sfn` (ver más abajo).

### 8.3 Cómo se crea una release (cada deploy)

En esta guía cada release es un `git clone` nuevo en una carpeta con nombre fijo (`release-1`, `release-2`, …). Es simple y permite rollback instantáneo.


```bash
cd /var/www/<nombre-proyecto>/releases
git clone git@github.com:Lambda-Works/<nombre-proyecto>.git release-<n>
cd release-<n>

# Build con Docker Compose
docker compose -f docker-compose.prod.yml build
```


### 8.4 Primera vez: apuntar `current`

`current` debe ser un symlink a la release que querés en producción:

```bash
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n> /var/www/<nombre-proyecto>/current
```
Ahora el current esta apuntando a `release-<n>`

### Mantenimiento:

### 8.5 Deploy siguiente

```bash
cd /var/www/<nombre-proyecto>/releases
git clone git@github.com:Lambda-Works/<nombre-proyecto>.git release-<n+1>
cd release-<n+1>

# Symlink al .env de shared (Docker busca .env en el directorio actual)
ln -sf ../shared/.env .env

# Build y levantar
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml up -d

# Cambiar la versión activa
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n+1> /var/www/<nombre-proyecto>/current
```

### 8.6 Rollback

Si `release-<n+1>` falla, volvé el puntero:

```bash
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n> /var/www/<nombre-proyecto>/current

# Con Docker Compose, volver a levantar desde la release anterior:
cd /var/www/<nombre-proyecto>/current
ln -sf ../shared/.env .env
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml up -d
```

### 8.7 Limpieza de releases viejas

Las carpetas en `releases/` **no** se borran solas. Política típica: conservar las últimas N releases y borrar el resto **solo** cuando estés seguro de que no necesitás rollback a ellas.

**No borres** la release a la que apunta `current` (comprobá con `readlink -f /var/www/<nombre-proyecto>/current`).

### 8.8 🐳 Puertos y variables de entorno

**get-ports** devuelve puertos libres en el VPS:
```bash
get-ports 2
```
Respuesta de ejemplo:
```json
{
  "start_port": 3000,
  "requested": 2,
  "ports": ["3000", "3002"]
}
```

Con esos puertos se escribe el archivo `shared/.env`:
```bash
nano /var/www/<nombre-proyecto>/shared/.env
```
```env
PUERTO_BACKEND=3000
PUERTO_FRONTEND=3002
```

**El symlink al .env** se crea después de cada clone. Docker Compose busca `.env` en el directorio donde se ejecuta:
```bash
cd /var/www/<nombre-proyecto>/current
ln -sf ../shared/.env .env
docker compose -f docker-compose.prod.yml up -d
```

> **Importante:** los puertos deben ser únicos por proyecto. Dos proyectos no pueden compartir el mismo puerto en la misma interfaz — de eso se encarga `get-ports`.



## 9. 🌐 HestiaCP: dominio, proxy y SSL (backend y frontend)

### 9.1 Crear templates desde VPS
```bash
cd /usr/local/hestia/data/templates/web/nginx/

cp node-proxy.tpl node-<PUERTO-BACKEND>.tpl
cp node-proxy.tpl node-<PUERTO-FRONTEND>.tpl

cp node-proxy.stpl node-<PUERTO-BACKEND>.stpl
cp node-proxy.stpl node-<PUERTO-FRONTEND>.stpl
```

Para los 4 archivos cambiar las lineas 
```
        proxy_pass         http://127.0.0.1:3001;
```
Por:
```
        proxy_pass         http://127.0.0.1:<PUERTO>;
```

#### Lo anterior ya está automatizado:
Usar comando
```bash
create-proxy-templates <PUERTO-BACKEND> <PUERTO-FRONTEND>
```

Una vez creados los templates recargar el config de Hestia desde la consola:
```bash
v-rebuild-web-domains user
```
### 9.2 Dominios y DNS

**Dominios recomendados:**

|Front/Back | Entorno | Ejemplo |
|--------|------------|---------|
|Backend | Producción | `<nombre-proyecto>.api.lambdaworks.ar` |
|Backend | Desarrollo | `dev.<nombre-proyecto>.api.lambdaworks.ar` |
|Frontend| Producción | `<nombre-proyecto>.lambdaworks.ar` |
|Frontend| Desarrollo | `dev.<nombre-proyecto>.lambdaworks.ar` |

**Agregar registros DNS desde el VPS** (subdominios de `lambdaworks.ar`):
```bash
hestia v-add-dns-record LambdaWorks lambdaworks.ar <subdominio> A 72.61.50.55
```
Ejemplo para un proyecto `hub`:
```bash
hestia v-add-dns-record LambdaWorks lambdaworks.ar hub A 72.61.50.55
hestia v-add-dns-record LambdaWorks lambdaworks.ar hub.api A 72.61.50.55
hestia v-add-dns-record LambdaWorks lambdaworks.ar dev.hub A 72.61.50.55
hestia v-add-dns-record LambdaWorks lambdaworks.ar dev.hub.api A 72.61.50.55
```

> Si el proyecto usa un dominio externo (no subdominio de `lambdaworks.ar`), ver la sección de extras al final.

**Pasos en Hestia (repetir por cada dominio de API):**
0. Ingresar a Hestia (https://72.61.50.55:8083/login/) con usuario `user`
1. `User` → Click en LambdaWorks
2. `Web` → `Add Web Domain` (Si aparece un cartel rojo igualmente continuar).
3. Poner el nombre de dominio, guardar y esperar que el vhost exista.
![](img/add-web-domain.png)
4. `Web` → `Edit Domain` (El ícono de un lápiz en la fila del dominio recién creado).
5. Tildar el checkbox de `Enable SSL for this domain` y los 3 que aparecen.
![](img/checkboxes.png) 
6. En `Advanced Options` elegir el Proxy Template que creamos antes, importante recordar cual era de backend y cual de frontend.
7. Guardar.

Ya debería estar funcionando todo OK. El único error que encontré que puede ocurrir es el siguiente:

### 9.3 Error (Let's Encrypt / ACME)

Puede aparecer el error (por algún motivo sólo lo vi para dominios de backend):
```Error: Let's Encrypt validation status 400 (test.api.lambdaworks.ar). Details: 403:"72.61.50.55: Invalid response from http://test.api.lambdaworks.ar/.well-known/acme-challenge/UvQi8XxbfjddTF8qi6vs6_M9BkqYwan3HDlxbhjIJ78: 404"```

En ese caso ejecutar en el VPS:

```bash
v-delete-web-domain LambdaWorks <dominio>
v-add-web-domain LambdaWorks <dominio>
v-add-letsencrypt-domain LambdaWorks <dominio>
```
Y volver a asignar el template desde hestia.


## 10. 🚀 Deploy en el VPS (releases + Docker Compose)

Cada release incluye **frontend y backend**:

```bash
cd /var/www/<nombre-proyecto>/releases
git clone git@github.com:Lambda-Works/<nombre-proyecto>.git release-<n+1>
cd release-<n+1>

# Symlink al .env de shared
ln -sf ../shared/.env .env

# Build y levantar (Docker Compose lee las vars del .env)
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml up -d

# Cambiar la versión activa
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n+1> /var/www/<nombre-proyecto>/current
```

Para **detener** los servicios:
```bash
cd /var/www/<nombre-proyecto>/current
docker compose -f docker-compose.prod.yml down
```


## 11. 🧪 Entorno de testing

Dominios sugeridos (misma convención que producción, con prefijo `dev.`):

```text
dev.<nombre-proyecto>.api.lambdaworks.ar   → backend
dev.<nombre-proyecto>.lambdaworks.ar      → frontend (Next.js)
```

**Mismo repo, distinta rama.** El entorno de desarrollo usa el mismo repositorio que producción pero clona la rama `develop`:

```bash
cd /var/www/<nombre-proyecto>-dev/releases
git clone -b develop --single-branch git@github.com:Lambda-Works/<nombre-proyecto>.git release-<n>
cd release-<n>

# Symlink al .env de shared (con puertos de dev y NODE_ENV=development)
ln -sf ../shared/.env .env

docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml up -d

ln -sfn /var/www/<nombre-proyecto>-dev/releases/release-<n> /var/www/<nombre-proyecto>-dev/current
```

El `shared/.env` de desarrollo lleva `NODE_ENV=development` y puertos distintos a los de producción, obtenidos con `get-ports 2`.

```env
# /var/www/<nombre-proyecto>-dev/shared/.env
NODE_ENV=development
PUERTO_BACKEND=3004
PUERTO_FRONTEND=3005
```

Después se debería manejar como un proyecto separado de producción con políticas de mergeo de `develop` → `production`.


### Extras (comandos útiles de Hestia):

```
# DNS — subdominios de lambdaworks.ar
hestia v-add-dns-record LambdaWorks lambdaworks.ar <subdominio> A 72.61.50.55

# Dominios externos (<dominio>, por ejemplo: github.com)
hestia v-add-dns-domain LambdaWorks <dominio>
# revertir paso anterior
# hestia v-delete-dns-domain LambdaWorks <dominio>
hestia v-add-dns-record LambdaWorks <dominio> @ A 72.61.50.55
hestia v-add-dns-record LambdaWorks <dominio> www CNAME <dominio>. # importante el punto final
hestia v-add-dns-record LambdaWorks <dominio> api A 72.61.50.55




# Para eliminar un registro:
# Listamos los registros para oservar el ID
hestia v-list-dns-records LambdaWorks lambdaworks.ar
hestia v-delete-dns-record LambdaWorks lambdaworks.ar <id>



# crear dominio
hestia v-add-web-domain LambdaWorks test.lambdaworks.ar
# eliminar dominio
# hestia v-delete-web-domain LambdaWorks test.lambdaworks.ar

# agregar certificado SSL
hestia v-add-letsencrypt-domain LambdaWorks test.lambdaworks.ar

# json con puertos libres
# {
#   "start_port": 3000,
#   "requested": 2,
#   "ports": [
#     "3000",
#     "3002"
#   ]
# }
get-ports <n>


create-proxy-templates <n1> <n2> ... <nn>
hestia v-rebuild-web-domains LambdaWorks

hestia v-change-web-domain-proxy-tpl LambdaWorks test.lambdaworks.ar node-<n>

hestia v-rebuild-web-domain LambdaWorks test.lambdaworks.ar

# otros comandos
# listar dominios
hestia v-list-web-domains LambdaWorks
# ver registros DNS
hestia v-list-dns-records LambdaWorks lambdaworks.ar



```
