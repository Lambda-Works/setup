# 🧱 Guía de setup para proyectos de Lambda Works

---

## 📌 Índice

1. Setup repo  
2. Frontend (Next.js)  
3. Backend (NestJS + Prisma + PostgreSQL) — incluye PostgreSQL + Prisma  
4. Workspaces  
5. Estructura final (repo)  
6. `.gitignore`  
7. VPS — estructura, releases, PM2  
8. HestiaCP — proxy, dominio, SSL, DNS, error Let’s Encrypt  
9. Deploy en el VPS (releases + build monorepo)  
10. Entorno de testing

---

> **Arquitectura por defecto de esta guía:** frontend (Next.js) y backend (NestJS) en el **mismo VPS** - Hestia (proxy + SSL) y PM2.

## 1. 🚀 Setup repo

Crear repo con nombre `<nombre-proyecto>` en github [Crear repositorio](https://github.com/new).

Ahora en local:
```bash
mkdir <nombre-proyecto>
cd <nombre-proyecto>
git init
echo "# <nombre-proyecto>" >> README.md
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/lambdaworksar/<nombre-proyecto>.git
git push -u origin main
```

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


## 4. 📦 Workspaces
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
    "build": "npm run build --workspaces",
    "start": "npm run start --workspaces"
  }
}
```

Instalar todo:

```bash
npm install
```

## 5. 🧩 Estructura final (repo)

```text
<nombre-proyecto>/
  ├── frontend/
  ├── backend/
  ├── package.json
  └── .gitignore
```
(Tambien se sugiere crear una carpeta shared pero todavia no encontré motivos para hacerlo)

## 6. 🧹 .gitignore

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
```

Nota: gran momento para hacer un segundo commit:
```bash
git add .
git commit -m "project setup"
git push
```

## 7. 🖥️ VPS


### 7.1 Árbol recomendado 
(no tocar nada todavía, sólo para ver como vamos a trabajar)
```text
/var/www/<nombre-proyecto>/
  ├── current/          → symlink a la release activa (lo usa PM2)
  ├── releases/         → una carpeta por versión desplegada
  └── shared/           → datos que NO se versionan y sobreviven a cada deploy
```

| Carpeta | Rol |
|--------|-----|
| `releases/` | Cada deploy agrega una carpeta nueva (`release-1`, `release-2`, …). Ahí vive el código de esa versión (checkout del repo + `node_modules` + build). |
| `current/` | **No** es una copia del código: es un **enlace simbólico** a *una* carpeta dentro de `releases/`. Cambiar deploy = cambiar a qué release apunta `current`. |
| `shared/` | `.env` de producción, uploads, certificados locales si aplica, etc. No se borra al publicar una release nueva. |

### 7.2 Setup inicial
Primero logearse en el servidor:
```bash
ssh root@72.61.50.55
```
Ahora si empezamos a crear la estructura:
```bash
mkdir -p /var/www/<nombre-proyecto>/{releases,shared}
```

`current/` **no** se crea con `mkdir`: aparece cuando hagas el primer `ln -sfn` (ver más abajo).

### 7.3 Cómo se crea una release (cada deploy)

En esta guía cada release es un `git clone` nuevo en una carpeta con nombre fijo (`release-1`, `release-2`, …). Es simple y permite rollback instantáneo.


```bash
cd /var/www/<nombre-proyecto>/releases
git clone https://github.com/lambdaworksar/<nombre-proyecto> release-<n>
cd release-<n>
npm ci
npm run build --workspaces
```


### 7.4 Primera vez: apuntar `current`

`current` debe ser un symlink a la release que querés en producción:

```bash
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n> /var/www/<nombre-proyecto>/current
```
Ahora el current esta apuntando a `release-<n>`

### Mantenimiento:

### 7.5 Deploy siguiente

```bash
cd /var/www/<nombre-proyecto>/releases
git clone <url-del-repo> release-<n+1>
cd release-<n+1>
npm ci
npm run build --workspaces

# Cambiar la versión activa
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n+1> /var/www/<nombre-proyecto>/current

pm2 reload <nombre-proyecto>-backend
pm2 reload <nombre-proyecto>-web
```

### 7.6 Rollback

Si `release-<n+1>` falla, volvé el puntero:

```bash
ln -sfn /var/www/<nombre-proyecto>/releases/release-<n> /var/www/<nombre-proyecto>/current
pm2 reload /var/www/<nombre-proyecto>/shared/ecosystem.config.js
```

### 7.7 Limpieza de releases viejas

Las carpetas en `releases/` **no** se borran solas. Política típica: conservar las últimas N releases y borrar el resto **solo** cuando estés seguro de que no necesitás rollback a ellas.

**No borres** la release a la que apunta `current` (comprobá con `readlink -f /var/www/<nombre-proyecto>/current`).

### 7.8 ⚙️ PM2 (process manager)

Primero verificar que puertos están disponibles para usar con el comando:
```bash
get-ports
```
Va a retornar dos números de puerto que vamos a usar en el siguiente archivo:
```bash
nano /var/www/<nombre-proyecto>/shared/ecosystem.config.js
```
Con el contenido:
```js
module.exports = {
  apps: [
    {
      name: "<nombre-proyecto>-backend",
      cwd: "/var/www/<nombre-proyecto>/current",
      script: "npm",
      args: "run start -w backend",
      env: {
        NODE_ENV: "production",
        PORT: <PUERTO-BACKEND>
      }
    },
    {
      name: "<nombre-proyecto>-web",
      cwd: "/var/www/<nombre-proyecto>/current",
      script: "npm",
      args: "run start -w frontend",
      env: {
        NODE_ENV: "production",
        PORT: <PUERTO-FRONTEND>
      }
    }
  ]
};
```

`name` debe ser **único** entre los procesos gestionados por PM2 en ese servidor. Convención: `<nombre-proyecto>-backend` y `<nombre-proyecto>-web`.

**Puerto (`PORT`):** también debe ser **único por proceso que escucha** en la misma interfaz (p. ej. `127.0.0.1`). Dos apps no pueden compartir el mismo puerto (de eso se encarga `get-ports`).


Run:

```bash
# Levanta (o suma) los procesos definidos en ecosystem (API, front, etc.)
pm2 start /var/www/<nombre-proyecto>/current/ecosystem.config.js

# Guarda la lista actual de procesos PM2 para que sobreviva al reinicio del servicio pm2
pm2 save

# Genera el comando de arranque al boot del sistema (systemd/openrc); hay que copiar/ejecutar la línea que muestra
pm2 startup
```


## 8. 🌐 HestiaCP: dominio, proxy y SSL (backend y frontend)

### 8.1 Crear templates desde VPS
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
### 8.2 Dominios y DNS

**Dominios recomendados:**

|Front/Back | Entorno | Ejemplo |
|--------|------------|---------|
|Backend | Producción | `<nombre-proyecto>.api.lambdaworks.ar` |
|Backend | Desarrollo | `dev.<nombre-proyecto>.api.lambdaworks.ar` |
|Frontend| Producción | `<nombre-proyecto>.lambdaworks.ar` |
|Frontend| Desarrollo | `dev.<nombre-proyecto>.lambdaworks.ar` |

Desde Vercel (https://vercel.com/lambda-works-projects/~/domains/lambdaworks.ar) agregar los registros DNS de tipo A para los 4 dominios con value 72.61.50.55
Por ejemplo:
![](img/registros-dns.png)

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

### 8.3 Error (Let's Encrypt / ACME)

Puede aparecer el error (por algún motivo sólo lo vi para dominios de backend):
```Error: Let's Encrypt validation status 400 (test.api.lambdaworks.ar). Details: 403:"72.61.50.55: Invalid response from http://test.api.lambdaworks.ar/.well-known/acme-challenge/UvQi8XxbfjddTF8qi6vs6_M9BkqYwan3HDlxbhjIJ78: 404"```

En ese caso ejecutar en el VPS:

```bash
v-delete-web-domain LambdaWorks <dominio>
v-add-web-domain LambdaWorks <dominio>
v-add-letsencrypt-domain LambdaWorks <dominio>
```
Y volver a asignar el template desde hestia.


## 9. 🚀 Deploy en el VPS (releases + build monorepo)

Cada release incluye **frontend y backend**. En la raíz del repo:

```bash
cd /var/www/<nombre-proyecto>/releases
git clone <repo> release-<n+1>
cd release-<n+1>

npm ci
npm run build --workspaces

ln -sfn /var/www/<nombre-proyecto>/releases/release-<n+1> /var/www/<nombre-proyecto>/current

# pm2: recargar API y web individualmente
pm2 reload <nombre-proyecto>-backend
pm2 reload <nombre-proyecto>-web

# O todo junto con:
pm2 reload /var/www/<nombre-proyecto>/shared/ecosystem.config.js 
```


## 10. 🧪 Entorno de testing

Dominios sugeridos (misma convención que producción, con prefijo `dev.`):

```text
dev.<nombre-proyecto>.api.lambdaworks.ar   → backend
dev.<nombre-proyecto>.lambdaworks.ar      → frontend (Next.js)
```

Crear un proyecto como ya hicimos, las unicas diferencias son al momento de clonar el repo para releases, clonar directamente la rama `dev`:
```bash
git clone -b dev --single-branch https://github.com/lambdaworksar/<nombre-proyecto>-dev release-<n>

```

Y en el `ecosystem.config.js` usar `NODE_ENV: "development"`
```js
module.exports = {
  apps: [
    {
      name: "<nombre-proyecto>-dev-backend",
      cwd: "/var/www/<nombre-proyecto>-dev/current",
      script: "npm",
      args: "run start -w backend",
      env: {
        NODE_ENV: "development",
        PORT: <PUERTO-BACKEND-DEV>
      }
    },
    {
      name: "<nombre-proyecto>-dev-web",
      cwd: "/var/www/<nombre-proyecto>-dev/current",
      script: "npm",
      args: "run start -w frontend",
      env: {
        NODE_ENV: "development",
        PORT: <PUERTO-FRONTEND-DEV>
      }
    }
  ]
};
```

Después se debería manejar como un proyecto separado de producción con políticas de mergeo.


### Extras (después agregar donde corresponda):
# PostgreSQL en VPS - Lambda Works (setup inicial + acceso + clon prod->test)

---

## 1) Convencion de nombres (obligatoria)

Para mantener orden entre proyectos:

- **DB produccion**: `<proyecto>_backend_prod`
- **DB testing/dev**: `<proyecto>_backend_dev`
- **Usuario app**: `app_<proyecto>_backend`

Ejemplo para Optica Marani:

- `opticamarani_backend_prod`
- `opticamarani_backend_dev`
- `app_opticamarani_backend`

Reglas:

- siempre minusculas
- usar `_` para separar
- sin espacios, tildes ni guiones

---

## 2) Requisitos en el VPS

Instalar PostgreSQL (si no esta):

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Verificar que este activo:

```bash
sudo systemctl status postgresql
```

---

## 3) Primera creacion de DB (Optica Marani)

Variables del ejemplo:

- `DB_PROD=opticamarani_backend_prod`
- `DB_DEV=opticamarani_backend_dev`
- `APP_USER=app_opticamarani_backend`

Entrar a PostgreSQL como admin del sistema:

```bash
sudo -u postgres psql
```

Crear usuario (una sola vez por proyecto):

```sql
CREATE USER app_opticamarani_backend WITH PASSWORD 'CAMBIAR_PASSWORD_SEGURA';
```

Crear bases y asignar owner:

```sql
CREATE DATABASE opticamarani_backend_prod OWNER app_opticamarani_backend;
CREATE DATABASE opticamarani_backend_dev OWNER app_opticamarani_backend;
```

Permisos explicitos:

```sql
GRANT ALL PRIVILEGES ON DATABASE opticamarani_backend_prod TO app_opticamarani_backend;
GRANT ALL PRIVILEGES ON DATABASE opticamarani_backend_dev TO app_opticamarani_backend;
\q
```

---

## 4) Como acceder despues (dia a dia)

Conexion directa por consola:

```bash
psql -h localhost -U app_opticamarani_backend -d opticamarani_backend_prod -W
psql -h localhost -U app_opticamarani_backend -d opticamarani_backend_dev -W
```

En backend (`.env`) usar una URL por entorno.

Produccion:

```env
DATABASE_URL=postgresql://app_opticamarani_backend:CAMBIAR_PASSWORD_SEGURA@localhost:5432/opticamarani_backend_prod
```

Testing/dev:

```env
DATABASE_URL=postgresql://app_opticamarani_backend:CAMBIAR_PASSWORD_SEGURA@localhost:5432/opticamarani_backend_dev
```

Si usan Prisma, recordar:

```bash
cd backend
npx prisma migrate deploy   # prod
npx prisma migrate dev      # dev local/testing
npx prisma generate
```

---

## 5) Copiar DB de produccion a testing (prod -> dev)

Este flujo sirve para probar con datos reales sin tocar produccion.

### 5.1 Dump de produccion

Formato custom (`-Fc`) recomendado:

```bash
PGPASSWORD='CAMBIAR_PASSWORD_SEGURA' pg_dump \
  -h localhost \
  -U app_opticamarani_backend \
  -d opticamarani_backend_prod \
  -Fc \
  -f /tmp/opticamarani_prod.dump
```

### 5.2 Recrear base de testing limpia

```bash
sudo -u postgres psql -c "DROP DATABASE IF EXISTS opticamarani_backend_dev;"
sudo -u postgres psql -c "CREATE DATABASE opticamarani_backend_dev OWNER app_opticamarani_backend;"
```

### 5.3 Restore en testing

```bash
PGPASSWORD='CAMBIAR_PASSWORD_SEGURA' pg_restore \
  -h localhost \
  -U app_opticamarani_backend \
  -d opticamarani_backend_dev \
  --no-owner \
  --no-privileges \
  /tmp/opticamarani_prod.dump
```

### 5.4 Verificacion rapida

```bash
PGPASSWORD='CAMBIAR_PASSWORD_SEGURA' psql \
  -h localhost \
  -U app_opticamarani_backend \
  -d opticamarani_backend_dev \
  -c "\dt"
```

### 5.5 Recomendaciones de seguridad

- nunca restaurar sobre `*_prod`
- hacer backup antes de cada restore
- si hay datos sensibles, anonimizar luego del restore (clientes, telefonos, emails, etc.)

---

## 6) Comandos utiles de operacion

Listar bases:

```bash
sudo -u postgres psql -c "\l"
```

Listar usuarios/roles:

```bash
sudo -u postgres psql -c "\du"
```

Cambiar password del usuario app:

```bash
sudo -u postgres psql -c "ALTER USER app_opticamarani_backend WITH PASSWORD 'NUEVA_PASSWORD';"
```

Backup rapido:

```bash
PGPASSWORD='CAMBIAR_PASSWORD_SEGURA' pg_dump \
  -h localhost -U app_opticamarani_backend -d opticamarani_backend_prod \
  > /tmp/opticamarani_backend_prod_$(date +%F).sql
```

---

## 7) Checklist por cada nuevo proyecto/cliente

1. Definir `<proyecto>_backend_prod`, `<proyecto>_backend_dev`, `app_<proyecto>_backend`.
2. Crear usuario app (si no existe), DB prod y DB dev.
3. Probar conexion con `psql` en ambas.
4. Configurar `DATABASE_URL` por entorno en `.env`.
5. Correr migraciones Prisma segun entorno.
6. Definir rutina de backup y (si aplica) rutina de clon `prod -> dev`.

