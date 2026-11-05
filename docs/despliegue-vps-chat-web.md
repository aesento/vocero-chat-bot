# Desplegar Vocero con chat web en un VPS

Esta guía despliega la rama `feat/067-chat-web` en un repositorio independiente.
Esa rama incluye el conector de chat web y aún no tiene una imagen publicada
propia; Compose debe construir la imagen del CRM desde el código de la rama.
Antes del despliegue, la rama debe estar subida al nuevo repositorio que elijas:
el fork viejo no se usa como destino de esta instalación.

## Antes de empezar

- VPS Linux con Docker Engine y el plugin Docker Compose. Recomiendo 2 vCPU y
  4 GB de RAM para construir Next.js en el mismo servidor.
- Dominio, por ejemplo `crm.tudominio.com`, con un registro DNS `A` apuntando
  a la IP del VPS.
- Puertos TCP 80 y 443 abiertos al VPS; conserva SSH en el puerto que uses.
- Acceso SSH con un usuario que pueda ejecutar Docker.

Postgres y Caddy se ejecutan en la red interna del Compose. No abras el puerto
5432 a Internet. Caddy solicita y renueva HTTPS automáticamente.

## Instalar

Clona la rama con el chat web:

```bash
git clone --branch feat/067-chat-web https://github.com/aesento/vocero-chat-bot.git vocero
cd vocero
```

Reemplaza `USUARIO/REPO_NUEVO` por la ruta del repositorio independiente una vez
creado y con la rama `feat/067-chat-web` publicada.

Con Node 22 y pnpm disponibles, genera los secretos locales y valida el entorno:

```bash
corepack enable
pnpm install --frozen-lockfile
pnpm init-env --domain crm.tudominio.com
```

Edita `.env` y deja estas dos variables con tu dominio real y el canal deseado:

```dotenv
DOMAIN=crm.tudominio.com
APP_BASE_URL=https://crm.tudominio.com
CHANNELS=whatsapp,webchat
```

Para usar solo chat web, sin mostrar WhatsApp en la instancia, usa
`CHANNELS=webchat`. La app igual requiere que `pnpm init-env` genere los
secretos obligatorios. No compartas ni subas `.env` al repositorio.

Construye la imagen del código de la rama y arranca la aplicación:

```bash
docker compose build --build-arg SOURCE_COMMIT="$(git rev-parse HEAD)" app
docker compose up -d
docker compose ps
```

El Compose de esta rama levanta PostgreSQL, el CRM, Caddy y Nea. Las migraciones
se aplican al iniciar el contenedor del CRM. Nea queda disponible para cuando
conectes la IA desde Vocero; la bandeja funciona manualmente mientras tanto.

Confirma el estado:

```bash
curl --fail https://crm.tudominio.com/api/health
docker compose logs --tail=100 app caddy
```

En el resultado de `/api/health`, comprueba `ok: true`, `mediaWritable: true`
y que `features.channels` incluya `webchat`.

## Activar el widget

1. Abre `https://crm.tudominio.com`, crea la organización y entra como dueño.
2. En **Configuración → Chat web**, registra el origen del sitio donde
   aparecerá el chat, por ejemplo `https://www.tudominio.com` (sin ruta ni
   barra final).
3. Copia el fragmento que muestra la pantalla y pégalo antes de `</body>` en
   el sitio del negocio.
4. Abre el sitio publicado en HTTPS y envía un mensaje. Debe aparecer en la
   Bandeja con el canal **Chat web**.
5. Si quieres respuestas automáticas, configura el proveedor en **Ajustes → IA**,
   revisa que Nea esté conectado y enciende el agente. Sin eso, una persona
   puede contestar desde la Bandeja.

El origen permitido es el dominio del sitio del negocio, no el dominio del
CRM. Usa el origen exacto (`https://sitio.example`); cada subdominio adicional
se agrega por separado en la configuración del chat web.

## Actualizar esta instalación

```bash
git pull --ff-only
docker compose build --build-arg SOURCE_COMMIT="$(git rev-parse HEAD)" app
docker compose up -d
docker compose ps
```

Haz una copia de seguridad de PostgreSQL y del volumen `vocero_app_data` antes
de actualizar. La base y los adjuntos viven en volúmenes persistentes; no borres
los volúmenes al recrear los contenedores.

## Operación segura

- Mantén `.env` fuera de Git y limita su lectura al usuario administrador.
- Deja PostgreSQL y Nea solo en la red interna de Docker; publica únicamente
  80/443 en Caddy y el puerto SSH.
- Configura copias de seguridad fuera del VPS y prueba la restauración antes
  de guardar datos reales.
- El despliegue utiliza esta rama del fork; no actualices con `git pull` de
  `main` del proyecto original, porque no contiene todavía estos cambios.
