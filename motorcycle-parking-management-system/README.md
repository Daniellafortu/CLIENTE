# MotoPark

Aplicación para administrar un parqueadero de motos con Next.js, PostgreSQL y Drizzle. La zona horaria de operación es `America/Bogota`.

## Publicar en Netlify

Netlify puede crear una base PostgreSQL integrada automáticamente. No necesitas instalar PostgreSQL ni crear las tablas a mano: el proyecto ya incluye la integración y las migraciones. Esta opción requiere un plan compatible con Netlify Database y consume créditos según el uso.

**¿MongoDB sería más fácil?** Esta aplicación está construida y probada con PostgreSQL y Drizzle. Cambiar a MongoDB exigiría rehacer la capa de datos y trasladar los registros; no basta con cambiar una variable. Para publicarla en Netlify, la base integrada ya evita tener que crear una base externa.

1. Descarga el proyecto completo y súbelo a un repositorio **privado** de GitHub. Incluye `netlify.toml` y la carpeta `netlify/database/migrations/`. Nunca subas `.env`, `.env.local`, `backups/`, `.next/` ni `node_modules/`.
2. En [Netlify](https://app.netlify.com/) selecciona **Add new project → Import an existing project**, conecta GitHub y elige el repositorio. Netlify detectará Next.js y usará el `netlify.toml` incluido.
3. En la configuración de variables de entorno del proyecto añade `PARK_ADMIN_USER` (por ejemplo, `admin`) y `PARK_ADMIN_PASSWORD` (una contraseña propia y única de al menos 12 caracteres). Si puedes elegir el alcance, permite **Builds** y **Functions**. No uses la contraseña de ejemplo ni publiques estos valores.
4. Publica o vuelve a desplegar. El paquete `@netlify/database` permite que Netlify prepare su base y aplique las migraciones de `netlify/database/migrations/` antes de publicar. Puedes comprobarla en **Data & Storage → Database**; si tu cuenta no la crea automáticamente, usa **Create a database manually** y vuelve a desplegar.
5. Abre la dirección `netlify.app` que ofrece Netlify e inicia sesión con el usuario y la contraseña que configuraste. En Netlify Database no hace falta poner `DATABASE_URL` ni `BACKUP_DIR`: Netlify proporciona `NETLIFY_DB_URL` automáticamente. No dejes una `DATABASE_URL` antigua o local, porque tiene prioridad sobre la base de Netlify.
6. Revisa **Data & Storage → Database → Backups** y la retención de tu plan. Descarga también copias desde **Respaldos** dentro de la aplicación y conserva una copia externa.

Estas migraciones nativas son para una base Netlify **nueva**. Si ya tienes datos reales en otra base, respáldalos y prepara su traslado; no se copian automáticamente. Las variables de administrador solo crean la cuenta inicial cuando la base está vacía. Para futuras modificaciones de esquema, incorpora la migración tanto en `drizzle/` como en `netlify/database/migrations/`.

Si deseas usar un PostgreSQL **externo** con Netlify, configura su conexión pública como `DATABASE_URL`, aplica previamente `node scripts/migrate.cjs` a esa base y administra sus respaldos con el proveedor correspondiente. No ejecutes las migraciones de Drizzle sobre una base que ya tenga las tablas sin preparar su historial de migraciones.

## Publicar en línea con Railway

Necesitas el **proyecto completo**, no solo el archivo Excel, y una cuenta en [GitHub](https://github.com/) y en [Railway](https://railway.com/). El alojamiento y la base de datos pueden generar cargos: revisa los límites y precios de tu plan.

1. Descarga o exporta el proyecto completo desde tu editor y descomprime el archivo si es necesario. Debes ver `package.json`, `src/`, `drizzle/`, `scripts/`, `railway.json` y `drizzle.config.ts` en la carpeta principal. **No subas** el archivo `.env`, las carpetas `node_modules/`, `.next/` ni `backups/`: pueden contener datos sensibles o archivos temporales.
2. En GitHub crea un repositorio **privado** y sube el contenido de la carpeta del proyecto, no la carpeta contenedora. Incluye `drizzle/` con su carpeta `meta/`, `scripts/`, `.nvmrc` y `railway.json`. También puedes conectar el proyecto con GitHub desde tu editor, si tiene esa opción.
3. En Railway selecciona **New Project → Deploy from GitHub repo** y elige el repositorio. Si el primer despliegue falla porque todavía no hay base de datos, continúa con los pasos siguientes y vuelve a desplegar.
4. En el mismo proyecto de Railway, añade una base desde **+ New → Database → PostgreSQL**. Entra al servicio de la aplicación, abre **Variables**, elige **Add Reference Variable** y selecciona `DATABASE_URL` del servicio PostgreSQL. La referencia habitual es `${{Postgres.DATABASE_URL}}`, aunque el nombre puede variar si cambias el nombre del servicio.
5. En **Variables** del servicio de la aplicación agrega también:
   - `PARK_ADMIN_USER`: el usuario administrador que tú elijas; entre 3 y 60 letras, números, puntos, guiones o guiones bajos.
   - `PARK_ADMIN_PASSWORD`: una contraseña propia, larga y única, de **al menos 12 caracteres**. No uses la contraseña de demostración. Guárdala en un lugar seguro.
6. Aplica los cambios y vuelve a desplegar el servicio. El archivo `railway.json` ejecuta automáticamente `node scripts/migrate.cjs` antes de iniciar: crea las tablas en una base PostgreSQL **nueva y vacía** y aplica futuras migraciones versionadas. El servicio solo se publica cuando pasa la comprobación de salud.
7. En el servicio de la aplicación abre **Settings → Networking → Generate Domain**. Railway te dará una dirección HTTPS que podrás compartir. Entra con el usuario y la contraseña que elegiste en el paso 5.
8. Para conservar las copias automáticas entre despliegues, agrega un **Volume** al servicio de la aplicación con ruta de montaje `/data` y configura `BACKUP_DIR` con el valor `/data/backups`. Además, conserva una copia externa y comprueba periódicamente que puedas restaurarla.

Las credenciales de las variables solo crean el administrador **la primera vez**, cuando la base está vacía. Si luego las cambias en Railway, la contraseña de una cuenta existente no cambia: hazlo desde el menú de perfil de la aplicación.

### Si la aplicación se publica fuera de Railway

Crea PostgreSQL en Railway desde **+ New → Database → PostgreSQL**. Abre el servicio de PostgreSQL y, en **Settings → Networking**, activa **Public Access** si aún no existe. En **Variables**, copia `DATABASE_PUBLIC_URL` y pégala como `DATABASE_URL` en el proveedor donde publicarás la aplicación; no compartas esa dirección porque contiene la contraseña de la base. La referencia `${{Postgres.DATABASE_URL}}` solo funciona entre servicios del mismo proyecto de Railway.

En una base nueva, el proveedor externo también debe ejecutar una vez `node scripts/migrate.cjs` con `DATABASE_URL` configurada, por ejemplo como comando de predespliegue. La configuración automática de `railway.json` se aplica únicamente cuando la aplicación se despliega en Railway. Es preferible mantener aplicación y base en el mismo proyecto de Railway para usar su red privada.

**Datos existentes:** el código y los registros de una vista previa no se trasladan automáticamente a la nueva base de Railway. Antes de trasladar datos reales, haz una copia de seguridad y planifica la importación de PostgreSQL. No ejecutes la migración inicial sobre una base que ya tenga estas tablas sin preparar su historial de migraciones.

## Desarrollo y base de datos

Usa Node.js 22, instala las dependencias con `npm install`, copia `.env.example` como `.env` y configura `DATABASE_URL`. Para una base nueva y vacía ejecuta `node scripts/migrate.cjs` y luego `npm run dev`. En una base local que ya se creó con `drizzle-kit push`, no vuelvas a aplicar la migración inicial sin una línea base; para desarrollo puedes seguir usando `npx drizzle-kit push`.

En una instalación local sin credenciales configuradas, el primer acceso es `admin` / `Admin123!`; cámbialo desde el perfil. En Railway y Netlify se requieren las credenciales propias indicadas arriba. Sin una sesión válida solo se muestra el formulario de inicio de sesión; el panel, las operaciones y los reportes requieren autenticación.

Las horas de entrada y salida son generadas por el servidor. La placa activa cuenta con un índice único parcial. Los pagos y cierres se serializan por fecha, y los ajustes de cierres conservan su auditoría.

## Nombre del parqueadero

El administrador puede abrir **Configuración** o pulsar el lápiz junto al logo, escribir el nombre y guardar los cambios. El nombre se muestra en el menú, la pantalla de acceso y los archivos Excel; también se incluye en los respaldos.

## Exportar el reporte mensual

En **Reporte mensual**, selecciona el mes y pulsa **Exportar Excel**. El archivo `.xlsx` incluye una hoja con los totales y el resumen diario, y otra con los movimientos pagados del mes, sus cuidadores, casco y comentarios. La descarga está disponible únicamente para el administrador.

## Respaldos

Mientras el servidor Node esté activo, se escribe automáticamente una instantánea JSON por día en `backups/` (o en `BACKUP_DIR`), con permisos restringidos y retención de 30 copias. También se puede descargar una copia desde la pantalla de respaldos. En producción usa un volumen persistente, protege las copias —contienen hashes de contraseña— y replica fuera del servidor. Un archivo en el disco temporal de un contenedor puede desaparecer al redesplegar.

En Netlify, las funciones no mantienen un proceso activo de forma continua y no se usan las copias programadas en disco de la aplicación. Si utilizas Netlify Database, revisa sus copias automáticas en **Data & Storage → Database → Backups** y la retención de tu plan; con una base externa, configura las copias con su proveedor. La descarga manual de JSON sigue disponible para el administrador.

Para una copia PostgreSQL independiente del proceso de la aplicación, configura además un cron diario, por ejemplo: `0 2 * * * /ruta/al/proyecto/scripts/backup.sh`. Este usa `pg_dump`, requiere `DATABASE_URL`, y conserva 30 días. Restaura un archivo `.dump` con `pg_restore`; las copias JSON son instantáneas de los datos de negocio y no incluyen sesiones temporales.
