# Despliegue en VPS (Óptica Marani)

Esta guía resume cómo alojar la aplicación en un servidor Linux (VPS): **frontend estático**, **API Node (NestJS)** y **PostgreSQL**. También indica cómo servir **archivos estáticos** si más adelante movés las recetas u otros adjuntos al disco en lugar de guardarlos en base de datos.

## Requisitos en el servidor

- Node.js 20+ (para compilar y ejecutar el backend).
- PostgreSQL 14+.
- Nginx (u otro reverse proxy) frente a la API y al sitio estático.
- Certificado TLS (recomendado: [Certbot](https://certbot.eff.org/) con el plugin de Nginx).

## Variables de entorno

**Backend** (`backend/.env` o variables del proceso systemd):

| Variable | Descripción |
|----------|-------------|
| `DATABASE_URL` | Cadena de conexión PostgreSQL (Prisma). |
| `JWT_SECRET` | Secreto para firmar tokens del panel admin. |
| `PORT` | Puerto donde escucha la API (ej. `3000`). |
| `FRONTEND_URL` | Origen del front en producción (ej. `https://tudominio.com`). Puede ser una lista separada por comas si hay varios orígenes. |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Solo para `npm run prisma:seed` (crear usuario admin inicial). |

**Frontend** (en build time, `frontend/.env.production`):

| Variable | Descripción |
|----------|-------------|
| `VITE_API_URL` | URL pública de la API, ej. `https://tudominio.com` si Nginx enruta `/api` al backend, o `https://api.tudominio.com`. |

## Base de datos

En el VPS:

```bash
cd backend
npm ci
npx prisma migrate deploy
npm run prisma:seed   # crea/actualiza usuario admin
```

Para generar datos de prueba de cotizaciones:

```bash
cd backend
npm run seed:quotes
```

(Elimina cotizaciones anteriores con código `MAR-SEED-*` y crea 100 nuevas; podés cambiar la cantidad con `SEED_QUOTES_COUNT=50`.)

## Build

```bash
# Frontend
cd frontend && npm ci && npm run build
# Salida: frontend/dist

# Backend
cd backend && npm ci && npm run build
# Salida: backend/dist
```

En producción el backend se ejecuta con `node dist/main.js` (o `npm run start:prod` desde `backend`).

## Cómo “alojar los archivos” en el VPS

### 1. Sitio web (HTML/CSS/JS del cotizador)

El resultado de `vite build` es un directorio **`dist/`** con `index.html` y `assets/`. Es contenido **100 % estático**: copiá ese árbol a un directorio servido por Nginx, por ejemplo:

```text
/var/www/opticamarani/html/
  index.html
  assets/
    ...
```

Permisos típicos: usuario `www-data` o el usuario del servicio web, lectura para el proceso de Nginx.

### 2. API detrás del mismo dominio

Convención habitual: Nginx sirve el front en `/` y **reenvía** `/api` al proceso Node en `localhost:3000`:

- El navegador pide `https://tudominio.com/api/quotes`.
- Nginx hace `proxy_pass` a `http://127.0.0.1:3000/api/...`.
- En el front, `VITE_API_URL` puede quedar vacío o ser `https://tudominio.com` para que las rutas relativas `/api/...` apunten al mismo host.

Ejemplo de bloques (ajustá rutas y nombres de servidor):

```nginx
server {
    listen 443 ssl;
    server_name tudominio.com;

    root /var/www/opticamarani/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 15m;
    }
}
```

`client_max_body_size` es importante porque las recetas adjuntas van como JSON con **base64** (límite similar en el backend: ~12 MB).

### 3. Archivos en disco (opcional, futuro)

Hoy las recetas pueden guardarse en el campo `prescription` (texto o data URL). Si en el futuro preferís **subir archivos al VPS**:

1. Creá un directorio dedicado, ej. `/var/www/opticamarani/uploads/`, con permisos solo para el usuario que ejecuta la API.
2. Serví ese directorio **solo para lectura** con Nginx (`location /uploads/ { alias /var/www/opticamarani/uploads/; }`) o exponé URLs firmadas desde la API.
3. En la aplicación habría que añadir endpoints `multipart` y guardar en disco en lugar de base64; no está incluido en la versión actual.

Mientras uses base64 en la base de datos, **no hace falta** un volumen de uploads separado: el “alojamiento” del contenido es la propia PostgreSQL.

## Proceso systemd (ejemplo)

Unidad mínima para el backend (ruta y usuario según tu servidor):

```ini
[Service]
WorkingDirectory=/opt/opticamarani/backend
ExecStart=/usr/bin/node dist/main.js
EnvironmentFile=/opt/opticamarani/backend/.env
Restart=on-failure
```

Tras editar: `sudo systemctl daemon-reload && sudo systemctl enable --now opticamarani-api`.

## Resumen

- **Archivos del sitio**: carpeta `frontend/dist` → directorio servido por Nginx (`root` + `try_files` para SPA).
- **API**: proceso Node escuchando en un puerto local; Nginx hace `proxy_pass` bajo `/api`.
- **Recetas como adjuntos**: en la implementación actual van en la BD; si pasás a archivos en disco, usá un directorio bajo `/var/www/...` con permisos estrictos y, si aplica, una `location` de Nginx o URLs servidas por la API.
