# FICHA DE REQUERIMIENTOS DE IMPLANTACIÓN — `tienda-libros`

**Versión de la ficha:** 1.0 · **Fecha:** 14/10/2026
**Alcance:** puesta en producción real de la aplicación `tienda-libros`.
**Arquitectura actual:** distribuida N-Tier, 5 servicios orquestados con Docker Compose (`proxy`, `api`, `worker`, `db`, `cache`).

> **Leyenda:** ✅ = **YA VIENE configurado** en el repositorio · ⚠️ = **SE DEBE AÑADIR / configurar externamente** para producción.

---

## 1) Hardware o instancia

### ✅ Lo que ya trae
- **Nada que escale por sí solo, pero nada impide escalar:** el `docker-compose.yml` no fija límites ni peticiones de recursos, así que los 5 servicios compartirán todo lo disponible en el host.
- Composición preparada para **un único host** (red por defecto de Compose, sin dependencias externas).

### ⚠️ Lo que hay que añadir / configurar

| Recomendación | Detalle | Motivo |
|---|---|---|
| **Instancia mínima** | 2 vCPU / 4 GB RAM / 40 GB SSD | Nginx + 2× Node + Postgres 16 + Redis 7 conviven en el mismo host. |
| **Instancia recomendada** | 4 vCPU / 8 GB RAM / 80 GB SSD | Margen para picos de lectura y copias de seguridad. |
| **Añadir límites de recursos** | `deploy.resources.limits` (o `mem_limit`/`cpus`) en `api`, `worker`, `db` y `cache` del compose | ⚠️ **Hoy no existe** → un _memory leak_ en `api` o una consulta pesada en `db` pueden tumbar el host entero. |
| **Separar base de datos** | Idealmente Postgres en instancia/volumen dedicado | El compose monta todo en una sola máquina. |
| **Disco para copias** | Almacenamiento externo (S3 u objeto similar) | Ver apartado 6. |
| **Capacidad de carga** | Estimar ~50–100 req/s por instancia Nginx con `nginx:1.27-alpine` | La caché Redis (TTL 60 s) absorbe la mayoría de lecturas. |

---

## 2) Software y versiones

### ✅ Lo que ya trae
| Componente | Versión | Origen |
|---|---|---|
| Nginx (proxy + estáticos) | `nginx:1.27-alpine` | imagen oficial, `docker-compose.yml:3` |
| Node.js (API) | `node:20-alpine` | `api/Dockerfile:1` |
| Node.js (worker) | `node:20-alpine` | `worker/Dockerfile:1` |
| PostgreSQL | `postgres:16-alpine` | `docker-compose.yml:37` |
| Redis | `redis:7-alpine` | `docker-compose.yml:53` |
| Dependencias API | `express ^4.19.2`, `pg ^8.11.5`, `redis ^4.6.13`, `jsonwebtoken ^9.0.2` | `api/package.json` |
| Dependencias worker | `redis ^4.6.13` | `worker/package.json` |
| Docker Compose | orquestación de los 5 servicios | `docker-compose.yml` |

- Las imágenes van **pinneadas por versión mayor** y el API arranca con `NODE_ENV: production` (`docker-compose.yml:16`).

### ⚠️ Lo que hay que añadir / configurar
- **Motor de contenedores:** instalar Docker Engine + plugin Compose v2 en el servidor de producción.
- **Registry de imágenes:** crear uno (GHCR/ECR/Docker Hub) y publicar imágenes **etiquetadas con versión**, porque hoy el compose usa `build: ./api` y `build: ./worker` (compilación local, sin tag). Ver apartados 8 y 9.
- **Bloquear versiones exactas:** `^` en `package.json` permite _minor/patch_ flotantes → generar y versionar `package-lock.json` e instalar con `npm ci` en los Dockerfiles (hoy usan `npm install --omit=dev`).
- **Herramientas externas a instalar:**
  - Certbot o cliente ACME (si TLS no lo aporta el proveedor/cloud).
  - `pg_dump`/`pg_restore` (incluidos en la imagen, pero conviene versión de herramientas local para dumps).
  - Cliente de copias a la nube (`rclone`, `aws cli`, etc.).
  - Agente de monitorización (Prometheus Node Exporter / Datadog / equivalentes).
- **Definir un `docker-compose.prod.yml`** aparte (override), sin tocar el actual.
- **Actualizaciones:** adoptar un ciclo de _patch_ mensual y revisión trimestral de imágenes base.

---

## 3) Red (dominio, DNS, puertos, cortafuegos)

### ✅ Lo que ya trae
- **Un único puerto publicado:** `8080:80` en el `proxy` (`docker-compose.yml:5`).
- **Puertos internos sin publicar (bien):** `api:3000`, `db:5432`, `cache:6379` **no** se exponen al exterior → solo accesibles desde la red interna de Compose.
- Enrutamiento interno definido: Nginx sirve `/` desde `/usr/share/nginx/html` con `try_files` (SPA) y `location /api/` con `proxy_pass http://api:3000/` que **elimina el prefijo `/api/`** (`nginx/default.conf:12-16`).
- Cabeceras básicas de proxy: `Host` y `X-Real-IP`.

### ⚠️ Lo que hay que añadir / configurar
- **Dominio:** registrar/definir el FQDN (p. ej. `tiendalibros.ejemplo.com`).
- **DNS:** registro **A/AAAA** hacia la IP pública del servidor (o CNAME a balanceador). TTL 300–3600 s.
- **Puertos:**
  - Exponer **80 → 443** (hoy solo está el 8080 hacia el 80 interno de Nginx). Cambiar el mapeo y redirigir 80 a 443.
  - Cerrar todo lo demás.
- **TLS/HTTPS:** hoy Nginx solo hace `listen 80` → **obligatorio añadir bloque `listen 443 ssl http2;`**, certificado (Let's Encrypt o comercial), HSTS y redirección 301 de 80 a 443. *Bloqueador de producción.*
- **`server_name`:** sustituir el comodín `server_name _;` por el dominio real.
- **Cabeceras de proxy correctas:** añadir `X-Forwarded-For`, `X-Forwarded-Proto` y `X-Forwarded-Host` (hoy solo hay `X-Real-IP`), para que la IP real llegue a la aplicación y a los logs.
- **Balanceador/CDN (opcional pero recomendado):** LB gestionado delante del proxy.
- **Cortafuegos (reglas sugeridas):**
  - **Entrada permitida:** 80, 443, y 22 solo desde IP de administración.
  - **Entrada bloqueada:** 3000, 5432, 6379, 8080.
  - **Salida permitida:** 443 (actualizaciones, ACME, backup), 53/DNS.
  - Añadir **fail2ban** o equivalente contra fuerza bruta en 22/443.
- **Grupos de seguridad / firewall del proveedor** (AWS SG, UFW, etc.) documentados y versionados.

---

## 4) Seguridad y secretos

### ✅ Lo que ya trae
- **Gestión de secretos por variable de entorno:** `POSTGRES_PASSWORD` y `JWT_SECRET` se inyectan desde el entorno, con plantilla `.env.example` y **`.env` excluido de git** (`.gitignore:5-7`).
- **SQL parametrizado** con `$1, $2` (`server.js:26-28`) → sin inyección SQL.
- **Aislamiento de red:** Postgres y Redis solo accesibles internamente.
- **Volumen de configuración y estáticos montados en solo lectura** (`:ro`, `docker-compose.yml:7-8`).
- **`NODE_ENV: production`.**
- El `.env.example` avisa explícitamente de cambiar los valores por defecto.

### ⚠️ Lo que hay que añadir / configurar

**Bloqueadores (impiden publicar tal cual está):**
1. **`JWT_SECRET` está inyectado pero NO se usa.** `jsonwebtoken` está en `api/package.json:13`, pero `server.js` **nunca lo importa**. En consecuencia:
   - `POST /api/pedidos` está **completamente sin autenticación**.
   - No hay control de acceso ni emisión/validación de tokens. → Implementar el middleware JWT real (o retirar la dependencia y declararlo explícitamente como API pública controlada).
2. **Sin TLS** (ver apartado 3) → credenciales y tráfico en claro.
3. **Redis sin contraseña:** `redis://cache:6379` (`docker-compose.yml:19,31`) sin `--requirepass`. → Añadir `requirepass`/ACL y usar `redis://:PASSWORD@cache:6379`.

**Configuración de cabecera / límite de tasa (todo ausente hoy):**
- **Cabeceras de seguridad:** añadir `helmet` o equivalentes en Nginx: `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`.
- **CORS:** no está definido → restringir al dominio propio.
- **Rate limiting:** `limit_req_zone` en Nginx (especialmente en `POST /api/pedidos`) y _throttling_ contra fuerza bruta.
- **Protección anti-CSRF** si se implementa sesión basada en cookies.
- **XSS en el front-end:** `index.html` usa `innerHTML` con datos de la API (`index.html:14-16`) → sanitizar/escapar o usar `textContent`.
- **Validación de entrada:** hoy solo se comprueba presencia (`if (!libroId || !unidades)`), no tipos ni rangos → validar tipos y límites.

**Higiene de contenedores:**
- **Ejecutar como usuario no raíz:** ninguno de los dos `Dockerfile` declara `USER` → añadir `USER node` / `USER nginx`.
- **Escanear vulnerabilidades** de imágenes (Trivy/Dependabot) en CI.
- **Endurecer Docker:** `no-new-privileges`, filesystem de solo lectura donde sea posible, _seccomp_/_AppArmor_.
- **Secretos:** migrar de `.env` a un gestor real (Vault, SOPS+age, secretos del proveedor, `docker secrets` en Swarm). Rotación definida.
- **Principio de mínimo privilegio en BD:** el usuario `tienda` es propietario del esquema; considerar usuario de aplicación de solo lectura/escritura limitada.

---

## 5) Entornos

### ✅ Lo que ya trae
- **Un único entorno implícito** (desarrollo/local), definido por `docker-compose.yml`.
- Parámetros externos mediante `.env` + `.env.example`, lo que facilita crear variantes.
- El `.gitignore` ya contempla `.env.*` con excepción de `.env.example`, permitiendo varios ficheros de entorno.

### ⚠️ Lo que hay que añadir / configurar
- **Formalizar entornos:** `dev` → `staging` (o `pre`) → `prod`, con **un fichero `.env` por entorno y ninguno compartido**.
- **Diferenciar configuración por entorno:**
  - `POSTGRES_PASSWORD` y `JWT_SECRET` **distintos por entorno** (nunca reutilizar los de desarrollo).
  - `NODE_ENV` y nivel de log por entorno.
- **Crear `docker-compose.prod.yml`** (override) con: límites de recursos, TLS, `requirepass` de Redis, sin seed data, usuarios no raíz.
- **Semántica de despliegue:**
  - `dev`: arranque local con `docker compose up --build`.
  - `staging`: réplica de prod con datos sintéticos para pruebas de aceptación.
  - `prod`: despliegue solo desde la rama principal, con aprobación.
- **Gestión de configuración externa:** parámetros de Nginx y _secrets_ fuera del repositorio.
- **Separar datos semilla:** `db/init.sql` inserta 3 libros de ejemplo (`init.sql:15-18`) → **no debe cargarse en producción**; separar `init.sql` (esquema) de `seed.sql` (datos de prueba solo en dev/staging).
- **Proteger el entorno de producción:** acceso SSH con llave, sin login de password, acceso restringido por IP.

---

## 6) Datos y copias de seguridad

### ✅ Lo que ya trae
- **Persistencia de Postgres** mediante volumen nombrado `datos-db` (`docker-compose.yml:43`).
- **Persistencia de Redis** mediante volumen `datos-cache` + política RDB `--save 60 1` (snapshot cada 60 s si hay ≥1 cambio) (`docker-compose.yml:54`).
- **Inicialización automática del esquema** con `db/init.sql` (tablas `libros`, `pedidos`) al crear el volumen por primera vez.
- **Healthcheck de Postgres** (`pg_isready -U tienda`, 10 s/5 s/5 intentos) con `depends_on: condition: service_healthy` → la API no arranca antes que la BD (`docker-compose.yml:45-49`).
- El `.gitignore` excluye dumps (`*.sql.gz`, `dump*.sql`).

### ⚠️ Lo que hay que añadir / configurar
- **Política de copias de seguridad (hoy NO existe ninguna):**
  - **Postgres:** `pg_dump` diario (formato _custom_, `Fc`) + **copias semanales completas**; retención: 7 diarias + 4 semanales + 12 mensuales (regla 3-2-1).
  - **Redis:** no requiere backup de negocio (caché + cola temporal); si se desea, exportar RDB ocasional.
  - **Código/infraestructura:** respaldar `docker-compose*.yml`, `nginx/`, `.env` cifrado y los `Dockerfile`.
- **Almacenamiento off-site cifrado** (S3 con SSE, o servidor remoto), con **cifrado en reposo y en tránsito**.
- **Automatizar** con cron/systemd-timer o servicio gestionado; sin intervención manual.
- **Alertas de fallo** de backup (ver apartado 7).
- **Plan de retención y borrado** (RGPD/LOPDGDD): datos de clientes/pedidos.
- **Pruebas de restauración periódicas** (mensual) — una copia no probada no es una copia.
- **Herramienta de migraciones de esquema:** hoy `init.sql` solo se ejecuta **al crear el volumen vacío**; cambios posteriores del esquema **no se aplican automáticamente** → introducir migraciones versionadas (Flyway, Knex, etc.) y probarlas en staging.
- **Alta disponibilidad (si aplica):** réplica de Postgres y estrategia de conmutación; Redis con _persistence_/AOF según tolerancia.
- **Vaciar o no la caché** en cada despliegue (hoy, al reiniciar Redis, la caché se reconstruye sola en ≤60 s gracias al TTL).

---

## 7) Monitorización y registros (logs)

### ✅ Lo que ya trae
- **Punto de salud HTTP:** `GET /salud` devuelve `{ estado: "ok" }` (`server.js:12`).
- **Salida a stdout** en API y worker (`console.log`/`console.error`), capturable con `docker logs`.
- **Logs con contexto de dominio:** hit/miss de caché y encolado en la API; "Generando la factura del pedido X" en el worker.
- **Reinicio automático** de los 5 servicios con `restart: unless-stopped`.
- **Healthcheck de Postgres** operativo.

### ⚠️ Lo que hay que añadir / configurar
- **Healthcheck de la API en Compose:** hoy **solo `db` tiene healthcheck**; añadir `healthcheck` contra `/salud` para `api` (y referencia a `proxy`). *`/salud` existe pero nadie la vigila.*
- **Logs estructurados:** pasar de `console.log` a JSON (pino/winston) con **nivel de log por entorno** y **ID de correlación** por petición.
- **Rotación y gestión de logs:** `json-file` con `max-size`/`max-file`, o driver `local`/`syslog`; sin ella, `docker logs` crece sin límite.
- **Centralización** (ELK, Loki, CloudWatch, Datadog…).
- **Métricas:** exportar métricas de Nginx y de la API; instalar **node_exporter** a nivel de host.
- **Monitorización 24×7:** Prometheus + Grafana o SaaS, con paneles de latencia, tasa de error, caché hit-rate, profundidad de `cola:facturas`.
- **Alertas definidas:**

  | Señal | Umbral sugerido | Acción |
  |---|---|---|
  | `/salud` caída o > 2 s | 3 fallos seguidos | Página/Slack al guardia |
  | Errores 5xx | > 1% durante 5 min | Revisión inmediata |
  | CPU / memoria host | > 85% durante 10 min | Escalar |
  | Espacio en disco | > 85% | Liberar/ampliar |
  | Caché Redis hit-rate | < 50% | Revisar TTL |
  | Profundidad `cola:facturas` | creciente > 15 min | **Worker caído ⇒ facturas sin generar** |
  | Último backup | antigüedad > 26 h | Reintentar y avisar |

- **Auditoría de acceso:** logs de Nginx con IP real (`X-Forwarded-For`) y retención definida.
- **Sondeas externos** (UptimeRobot/Grafana Cloud) sobre el dominio público.

---

## 8) Fases del despliegue

### ✅ Lo que ya trae
- Arranque determinista con Compose y **orden garantizado por `depends_on`** (`api` espera a `db` _healthy_; `worker` espera a `cache`).
- **Reinicio automático** ante caída.
- Configuración separada del código (`.env`).
- **Semilla de datos inicial** automática en el primer arranque.

### ⚠️ Lo que hay que añadir / configurar

> **Nota:** hoy **no hay pipeline CI/CD, ni registry, ni imágenes versionadas** (`build:` local). Todo ello debe crearse antes de estas fases.

| Fase | Contenido |
|---|---|
| **F0 — Requisitos previos** | Dominio + DNS, instancia, cortafuegos, TLS, registry, gestor de secretos, CI/CD, monitorización básica, `.env` de producción generado con valores nuevos. |
| **F1 — CI** | Lint + tests + `npm ci` + build de imágenes + **escaneo de vulnerabilidades** + publicación en registry con tag = versión/SHA. |
| **F2 — Staging** | Despliegue con datos sintéticos (sin seed de demo). Validación: `/salud`, `GET /api/libros` (hit y miss), `POST /api/pedidos` → 201 y mensaje en `cola:facturas` procesado por el worker. |
| **F3 — Copia previa** | **Snapshot de Postgres obligatorio antes de tocar producción.** |
| **F4 — Publicación (blue/green o recreativo)** | Arrancar versión nueva → smoke test → conmutar. Usar **tags inmutables**, nunca `latest`. |
| **F5 — Verificación post-despliegue** | Smoke tests: `GET /salud`, `GET /api/libros`, `POST /api/pedidos`, carga del front vía HTTPS, revisión de logs 15 min, métricas y cola de facturas. |
| **F6 — Cierre** | Retirar versión antigua, documentar en _changelog_, confirmar backup correcto. |

- **Ventanas de despliegue:** laborables 09:00–13:00, con personal disponible. Sin despliegues viernes por la tarde ni en fechas sensibles.
- **Despliegue cero-downtime (recomendado):** el frontend usa `try_files` con `index.html` sin _cache-busting_ → añadir versionado de assets o cabeceras `no-cache` en `index.html`.

---

## 9) Plan de vuelta atrás (rollback)

### ✅ Lo que ya trae
- **`restart: unless-stopped`** → auto-sanación ante caída del proceso (no ante versión defectuosa).
- **Datos persistentes** en volúmenes nombrados → sobreviven a recrear contenedores.
- **Caché autolimitada** (TTL 60 s) → tras un reinicio se regenera sola en ≤1 minuto, sin intervención.
- La cola `cola:facturas` vive en Redis con persistencia RDB → tolera reinicios breves.

### ⚠️ Lo que hay que añadir / configurar

**Requisito previo (hoy NO existe):** para poder volver atrás hacen falta **imágenes versionadas en un registry** y **etiquetas inmutables**. Con `build: ./api` local no hay versión anterior a la que recurrir.

**Niveles de rollback:**

| Nivel | Situación | Acción | Riesgo de pérdida de datos |
|---|---|---|---|
| **1. Rollback de aplicación** | Defecto en API/worker/frontend | `docker compose up -d` con el tag de la versión anterior en `docker-compose.prod.yml`; validar con smoke tests | Nulo (esquema compatible) |
| **2. Rollback de esquema** | Migración BD defectuosa | Restaurar el **dump pre-despliegue** (F3) y desplegar versión anterior | **Pérdida de pedidos** creados tras el dump ⇒ pedir confirmación antes |
| **3. Rollback completo** | Host o infraestructura dañada | Reconstruir host + restaurar backup + apuntar DNS | Máximo ⇒ última opción |

**Procedimiento operativo de rollback:**
1. **Criterios de activación:** >5% de 5xx, `/salud` caída >2 min, imposibilidad de crear pedidos, o pérdida de integridad de datos.
2. **Autorización:** dev puede activar nivel 1; niveles 2 y 3 requieren aprobación de responsable.
3. **Pasos nivel 1:** identificar la última versión buena → actualizar el tag → `docker compose -f docker-compose.prod.yml up -d` → smoke tests → verificar cola de facturas → documentar.
4. **Plazo objetivo:** < 15 minutos (nivel 1), < 60 minutos (nivel 3).
5. **Comunicación:** avisar a soporte y, si hay impacto visible, publicar aviso.
6. **Post-mortem** en 48 h con acciones correctivas.

**⚠️ Puntos débiles del rollback actual a corregir:**
- **Sin registry ni tags** → añadirlos (ver F0/F1).
- **Sin pruebas automatizadas** → no hay forma objetiva de decidir si la versión es buena.
- **El worker solo hace `console.log`** (`worker.js:10`) y **no persiste facturas** → en producción, un problema del worker provoca **pérdida silenciosa de facturas**. Persistir en Postgres y añadir **reintentos + DLQ**.
- **El worker usa `BRPOP` con timeout 0** (`worker.js:9`): la tarea extraída se pierde si el proceso muere justo después. Usar `LMOVE`/lista de procesamiento fiable (patrón _reliable queue_).
- **Datos semilla en `init.sql`** → en restauraciones no debe cargarse el seed de demo.

---

## Resumen de hallazgos críticos (bloqueadores antes de publicar)

| # | Hallazgo | Evidencia |
|---|---|---|
| 1 | **Sin TLS/HTTPS** | `nginx/default.conf:2` solo `listen 80` |
| 2 | **Sin autenticación:** `POST /api/pedidos` abierto; `JWT_SECRET` inyectado pero sin usar (`jsonwebtoken` nunca se importa) | `server.js`, `api/package.json:13`, `docker-compose.yml:20` |
| 3 | **Redis sin contraseña** | `docker-compose.yml:19,31` |
| 4 | **Sin copias de seguridad** de Postgres | `docker-compose.yml` |
| 5 | **Sin registry ni imágenes versionadas** → imposible rollback | `docker-compose.yml:14,29` (`build:`) |
| 6 | **Sin límites de recursos** | `docker-compose.yml` |
| 7 | **`/salud` sin healthcheck en Compose** (solo `db` lo tiene) | `docker-compose.yml:45` |
| 8 | **Contenedores como raíz** (sin `USER`) | `api/Dockerfile`, `worker/Dockerfile` |
| 9 | **XSS potencial** por `innerHTML` | `web/index.html:14-16` |
| 10 | **Worker no persiste facturas** + `BRPOP` sin garantías | `worker.js:9-10` |
| 11 | **Seed de demo en `init.sql`** | `db/init.sql:15-18` |
| 12 | **Logs sin rotar ni estructurar**; sin monitorización ni alertas | `server.js`, `worker.js` |
