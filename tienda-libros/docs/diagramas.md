# Diagramas de recorrido — tienda-libros

Basados en `api/src/server.js` y `worker/worker.js`.

**Nota de rutas:** Express monta `/libros` y `/pedidos` (sin prefijo). El prefijo `/api`
lo elimina Nginx (`location /api/ { proxy_pass http://api:3000/; }`), por eso
`GET /api/libros` llega al handler como `GET /libros`.

---

## 1. Lectura `GET /api/libros` — sistema de caché (Hit / Miss)

Patrón *cache-aside* sobre Redis, con TTL de 60 s (`EX: 60`). No existe caché en
memoria local: Redis **es** la caché.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente (navegador)
    participant N as Nginx :8080
    participant A as Express GET /libros
    participant R as Redis (caché)
    participant P as PostgreSQL

    C->>N: GET /api/libros
    Note over N: location /api/ → proxy_pass api:3000/<br/>quita el prefijo /api/
    N->>A: GET /libros
    A->>R: GET libros
    alt HIT — valor en caché
        R-->>A: JSON serializado (clave "libros")
        A->>A: JSON.parse(enCache)
        A-->>N: 200 OK + JSON desde caché
        N-->>C: 200 OK (sin consultar la BD)
    else MISS — clave inexistente o expirada (TTL 60 s)
        R-->>A: (nil)
        A->>P: SELECT id, titulo, autor, precio FROM libros ORDER BY id
        P-->>A: rows (resultset)
        A->>R: SET libros = JSON.stringify(rows) EX 60
        R-->>A: OK (reescribir caché)
        A-->>N: 200 OK + rows desde la BD
        N-->>C: 200 OK (próximas peticiones serán HIT hasta 60 s)
    end
```

- **Hit (línea 16 de server.js):** `return res.json(JSON.parse(enCache))` → respuesta
  directa, la BD no participa.
- **Miss (líneas 17-19):** consulta SQL, rellena la caché con `EX 60` y responde;
  la siguiente petición dentro de 60 s será Hit.

---

## 2. Escritura `POST /api/pedidos` — cola `cola:facturas` + worker asíncrono

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente (navegador)
    participant N as Nginx :8080
    participant A as Express POST /pedidos
    participant P as PostgreSQL
    participant R as Redis (lista cola:facturas)
    participant W as worker.js (while true)

    Note over W: Al arrancar: await cache.connect()
    W->>R: BRPOP cola:facturas 0 (queda bloqueado, espera infinita)

    rect rgb(232, 245, 253)
        Note over C,A: Parte SÍNCRONA (el cliente espera)
        C->>N: POST /api/pedidos { libroId, unidades }
        N->>A: POST /pedidos (prefijo /api/ eliminado)
        A->>A: express.json() parsea el body
        A->>A: Valida libroId y unidades
        alt Datos incompletos
            A-->>N: 400 { error: "Faltan datos" }
            N-->>C: 400 Bad Request
        else Datos correctos
            A->>P: INSERT INTO pedidos (libro_id, unidades) VALUES ($1,$2) RETURNING id
            P-->>A: rows[0].id = (id)
            A->>R: LPUSH cola:facturas (id)
            R-->>A: OK (mensaje encolado)
            A-->>N: 201 { pedido: (id) }
            N-->>C: 201 Created (respuesta INMEDIATA, sin generar factura aún)
        end
    end

    rect rgb(232, 253, 233)
        Note over R,W: Parte ASÍNRONA (desacoplada, ocurre después de la respuesta)
        R-->>W: tarea.element = (id) → desbloquea BRPOP
        W->>W: console.log("Generando la factura del pedido", (id))
        W->>R: BRPOP cola:facturas 0 (vuelve a bloquearse esperando la siguiente)
    end
```

**Claves:**
- La API **nunca habla directamente con el worker**: solo deja el `id` con `LPUSH`
  (línea 29 de server.js). El worker lo retira con `BRPOP` bloqueante timeout `0`
  = espera infinita (línea 9 de worker.js), API de node-redis v4 → `{ element: (id) }`.
- El `201` se responde **antes** de generar la factura → desacoplamiento
  productor/consumidor; si el worker está caído, los pedidos siguen acumulándose en la cola.
- `worker.js` hoy solo hace `console.log` de la factura (no la persiste).
