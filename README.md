# Gestor de tareas en dos contenedores Docker

API sencilla para guardar y consultar tareas, que corre en **dos contenedores comunicados**:

| Contenedor | Imagen | Qué hace |
|---|---|---|
| `app` | Propia (`app/Dockerfile`, basada en `node:22-alpine`) | API REST en Node.js + Express. Guarda y consulta tareas. |
| `db` | Oficial `postgres:17-alpine` | Base de datos PostgreSQL donde quedan guardadas las tareas. |

Los dos están en la red `red_tareas`. La app encuentra a la base por el **nombre del servicio** (`db`), no por IP.

```
  El navegador / curl
         │  http://localhost:3000
         ▼
 ┌──────────────────┐      red_tareas      ┌──────────────────┐
 │  app (Node.js)   │ ───────────────────▶ │  db (PostgreSQL) │
 │  puerto 3000     │   host "db":5432     │  puerto 5432     │
 └──────────────────┘                      └──────────────────┘
                                                   │
                                            volumen datos_db
```

## Requisitos

- Docker Desktop Windows o Docker Engine + Docker Compose v2 (Linux).
- Git.


## Inicio

```bash
git clone https://github.com/malejangel/gestor-tareas-devops.git
cd gestor-tareas-devops
```

1. se crea el archivo `.env` a partir del ejemplo y cambia la clave:

   ```bash
   # Linux / macOS / Git Bash
   cp .env.example .env
   # Windows (PowerShell)
   copy .env.example .env
   ```

   Variables que debe tener `.env`:

   | Variable | Para qué |
   |---|---|
   | `DB_USER` | Usuario de PostgreSQL |
   | `DB_PASSWORD` | Clave de PostgreSQL (invéntala) |
   | `DB_NAME` | Nombre de la base de datos |
   | `APP_PORT` | Puerto de tu equipo para la API (opcional, por defecto 3000) |

2. Iniciar los dos contenedores:

   ```bash
   docker compose up -d --build
   ```

3. Comprueba que ambos estén corriendo y `healthy`:

   ```bash
   docker compose ps
   ```

## ejemplo

Guardar una tarea y luego la consultarla; el resultado consultado debe ser igual a lo guardado.

```bash
docker compose exec app node scripts/operacion-ejemplo.js
```

Resultado esperado:

```
Resultado esperado: {"titulo":"Estudiar Docker","descripcion":"Repasar redes y volumenes","completada":false}
Resultado obtenido: {"titulo":"Estudiar Docker","descripcion":"Repasar redes y volumenes","completada":false}
OPERACION DE EJEMPLO: OK
```

De manera manual:

```bash
curl -X POST http://localhost:3000/tareas -H "Content-Type: application/json" -d "{\"titulo\":\"Entregar proyecto\",\"descripcion\":\"Subir el documento\"}"
curl http://localhost:3000/tareas/1
curl http://localhost:3000/salud
```

## Orden de inicio

1. Arranca `db`. Su `healthcheck` (`pg_isready`) dice cuándo acepta conexiones.
2. `app` tiene `depends_on: condition: service_healthy`, así que **espera** a que `db` esté sana antes de arrancar.
3. La primera vez, PostgreSQL ejecuta `db/init.sql` y crea la tabla `tareas`.
4. Por si acaso, la app además reintenta la conexión 10 veces.

## Apagar

```bash
docker compose down        # apaga y conserva los datos
docker compose down -v     # apaga y BORRA los datos (volumen)
```

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `Falta DB_PASSWORD, revisa tu archivo .env` | No creaste el `.env` | Haz el paso 1 |
| `port is already allocated` / `puerto 3000 en uso` | Otro programa usa el 3000 | Cambia `APP_PORT=3001` en `.env` y usa ese puerto |
| `Cannot connect to the Docker daemon` / `open //./pipe/docker_engine` | Docker Desktop no está abierto | Abre Docker Desktop y espera a que diga *Running* |
| `password authentication failed` | Cambiaste la clave en `.env` después de crear el volumen | `docker compose down -v` y vuelve a levantar |
| `relation "tareas" does not exist` | El volumen ya existía antes de `init.sql` | `docker compose down -v` y vuelve a levantar |

## Integración continua (Jenkins)

El `Jenkinsfile` construye la imagen, levanta los contenedores, ejecuta la operación de ejemplo (falla si el resultado no coincide) y retira todo al terminar, pase o falle. Se dispara solo con Poll SCM cada ~2 minutos. Necesita una credencial *Secret text* en Jenkins con id `tareas-db-password`.
