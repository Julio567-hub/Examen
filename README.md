# Sistema de Inventario — Documentación del Proyecto

Sistema distribuido de gestión de inventario compuesto por tres servicios independientes que se comunican entre sí a través de una VPN Tailscale.

---

## Tabla de Contenidos

- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Diagrama de Base de Datos](#diagrama-de-base-de-datos)
- [Stack Tecnológico](#stack-tecnológico)
- [Instrucciones de Despliegue](#instrucciones-de-despliegue)
- [Variables de Entorno](#variables-de-entorno)
- [Endpoints disponibles](#endpoints-disponibles)

---

## Arquitectura del Sistema

El sistema está dividido en tres máquinas con responsabilidades separadas:

```
                        ┌─────────────────────────────────────────────┐
                        │             Red Tailscale (VPN)             │
                        └─────────────────────────────────────────────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
  ┌───────────────────────┐   ┌───────────────────────┐   ┌───────────────────────┐
  │      MÁQUINA 1        │   │      MÁQUINA 2        │   │      MÁQUINA 3        │
  │   Aplicación Spring   │   │    Base de Datos      │   │  Servicio Reportes   │
  │                       │   │                       │   │                       │
  │  ┌─────────────────┐  │   │  ┌─────────────────┐  │   │  ┌─────────────────┐  │
  │  │  Spring Boot 4  │  │   │  │  PostgreSQL 16  │  │   │  │  FastAPI Python │  │
  │  │   Java 21       │  │   │  │                 │  │   │  │   Python 3.11   │  │
  │  │   Thymeleaf     │  │   │  │  Base: Comida   │  │   │  │   Uvicorn       │  │
  │  │   Spring Sec.   │  │   │  │                 │  │   │  └─────────────────┘  │
  │  └────────┬────────┘  │   │  └─────────────────┘  │   │           │           │
  │           │           │   │                       │   │           │           │
  │  Puerto: 8080         │   │  Puerto: 5432         │   │  Puerto: 8081         │
  │                       │   │                       │   │                       │
  │  ┌─────────────────┐  │   │  ┌─────────────────┐  │   └───────────────────────┘
  │  │    pgAdmin 4    │  │   │  │    pgAdmin 4    │  │
  │  └─────────────────┘  │   │  └─────────────────┘  │
  │  Puerto: 5050         │   │  Puerto: 5050         │
  └───────────┬───────────┘   └───────────────────────┘
              │                            ▲
              │   JDBC (DB_HOST:5432)      │
              └────────────────────────────┘
              │
              │   HTTP (REPORT_SERVICE_URL:8081)
              └──────────────────────────────────► MÁQUINA 3
```

### Flujo de comunicación

- **Máquina 1 → Máquina 2**: Spring Boot se conecta a PostgreSQL vía JDBC usando la IP Tailscale de Máquina 2.
- **Máquina 1 → Máquina 3**: Spring actúa como proxy — cuando el usuario accede a `/reportes/*`, Spring hace una llamada HTTP interna al servicio de reportes de Máquina 3.
- **Máquina 3 → Máquina 2**: El servicio de reportes se conecta directamente a PostgreSQL para ejecutar las consultas de análisis.
- **Máquina 2**: Solo expone la base de datos. No hace llamadas a ningún otro servicio.

---

## Diagrama de Base de Datos

```
┌──────────────────────────┐
│         categoria        │
├──────────────────────────┤
│ PK  id          BIGSERIAL│
│     nombre      VARCHAR  │◄─────────────────────┐
│     descripcion VARCHAR  │                       │
└──────────────────────────┘                       │
                                                   │
┌──────────────────────────┐                       │
│        proveedores       │                       │
├──────────────────────────┤                       │
│ PK  id          BIGSERIAL│◄──────────┐           │
│     nombre      VARCHAR  │           │           │
│     contacto    VARCHAR  │           │           │
│     telefono    VARCHAR  │           │           │
│     email       VARCHAR  │           │           │
│     direccion   VARCHAR  │           │           │
└──────────────────────────┘           │           │
                                       │           │
┌──────────────────────────────────────┴───────────┴──────────┐
│                          productos                           │
├──────────────────────────────────────────────────────────────┤
│ PK  id           BIGSERIAL                                   │
│     nombre       VARCHAR       NOT NULL                      │
│     descripcion  VARCHAR(500)                                │
│     precio       FLOAT         NOT NULL                      │
│     stock_actual INTEGER       NOT NULL  DEFAULT 0           │
│     stock_minimo INTEGER       NOT NULL  DEFAULT 5           │
│ FK  categoria_id BIGINT        → categoria(id)               │
│ FK  proveedor_id BIGINT        → proveedores(id)             │
└──────────────────────────────────────────────────────────────┘
          │                          │
          │ 1:N                      │ 1:N
          ▼                          ▼
┌──────────────────────────────────────────────────────────────┐
│                         movimientos                          │
├──────────────────────────────────────────────────────────────┤
│ PK  id           BIGSERIAL                                   │
│ FK  producto_id  BIGINT        → productos(id)  NOT NULL     │
│     cantidad     INTEGER       NOT NULL                      │
│     tipo         VARCHAR(10)   CHECK (ENTRADA | SALIDA)      │
│     fecha        TIMESTAMP     NOT NULL  DEFAULT NOW()       │
│ FK  proveedor_id BIGINT        → proveedores(id)             │
│     motivo       VARCHAR(100)                                │
│     observacion  VARCHAR(255)                                │
└──────────────────────────────────────────────────────────────┘
```

### Relaciones

| Relación | Tipo | Descripción |
|---|---|---|
| categoria → productos | 1:N | Una categoría agrupa muchos productos |
| proveedores → productos | 1:N | Un proveedor suministra muchos productos |
| productos → movimientos | 1:N | Un producto puede tener muchos movimientos |
| proveedores → movimientos | 1:N | Un proveedor puede aparecer en muchos movimientos (opcional) |

---

## Stack Tecnológico

| Máquina | Tecnología | Versión |
|---|---|---|
| Máquina 1 | Spring Boot | 4.0.2 |
| Máquina 1 | Java | 21 |
| Máquina 1 | Thymeleaf | — |
| Máquina 1 | Spring Security | — |
| Máquina 1 | Spring Data JPA + Hibernate | — |
| Máquina 2 | PostgreSQL | 16 |
| Máquina 2 | pgAdmin 4 | latest |
| Máquina 3 | Python | 3.11 |
| Máquina 3 | FastAPI | 0.115.5 |
| Máquina 3 | Uvicorn | 0.32.1 |
| Máquina 3 | psycopg2 | 2.9.10 |
| Todas | Docker + Docker Compose | — |
| Todas | Tailscale VPN | — |

---

## Instrucciones de Despliegue

### Requisitos previos

En **cada máquina** instalar:
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/Mac) o Docker Engine (Linux)
- [Tailscale](https://tailscale.com/download)

### Paso 1 — Conectar las máquinas con Tailscale

En **cada máquina**:

1. Instalar Tailscale y hacer login con la misma cuenta en las 3 máquinas.
2. Obtener la IP Tailscale de cada una:
   - **Windows**: clic en el ícono de Tailscale en la barra de tareas → aparece la IP `100.x.x.x`
   - **Linux**: ejecutar `tailscale ip -4`
3. Anotar las 3 IPs:
   ```
   Máquina 1 (Spring)   → 100.x.x.1
   Máquina 2 (BD)       → 100.x.x.2
   Máquina 3 (Reportes) → 100.x.x.3
   ```
4. Verificar conectividad desde Máquina 1:
   ```bash
   ping 100.x.x.2   # debe responder
   ping 100.x.x.3   # debe responder
   ```

---

### Paso 2 — Levantar Máquina 2 (Base de Datos) — PRIMERO

```bash
cd maquina2/

# Crear el archivo de variables de entorno
cp .env.example .env

# Levantar los contenedores
docker compose up -d

# Verificar que está healthy (esperar ~20 segundos)
docker compose ps
```

Esperar hasta ver `healthy` en la columna Status antes de continuar.

**Verificar que la base de datos está lista:**
```bash
docker exec inventario-db pg_isready -U postgres -d Comida
# Debe responder: /var/run/postgresql:5432 - accepting connections
```

**Acceder a pgAdmin** (opcional): http://IP_MAQUINA2:5050
- Email: `admin@inventario.com`
- Contraseña: `admin123`

---

### Paso 3 — Levantar Máquina 3 (Servicio de Reportes)

```bash
cd servicio-reportes/

# Editar el .env con la IP Tailscale de Máquina 2
cp .env.example .env
```

Editar `.env`:
```env
DB_HOST=100.x.x.2      # ← IP Tailscale de Máquina 2
DB_PORT=5432
DB_NAME=Comida
DB_USER=postgres
DB_PASSWORD=12345
```

```bash
# Levantar
docker compose up -d

# Verificar que está healthy
docker compose ps

# Verificar conexión a la BD
curl http://localhost:8081/health
# Respuesta esperada: {"status":"ok","db":"conectada"}
```

---

### Paso 4 — Levantar Máquina 1 (Aplicación Spring Boot) — ÚLTIMO

```bash
cd proyecto/

# Crear el archivo de variables de entorno
cp .env.example .env
```

Editar `.env`:
```env
DB_HOST=100.x.x.2                         # ← IP Tailscale de Máquina 2
DB_PORT=5432
DB_NAME=Comida
DB_USER=postgres
DB_PASSWORD=12345
REPORT_SERVICE_URL=http://100.x.x.3:8081  # ← IP Tailscale de Máquina 3
APP_USER=admin
APP_PASSWORD=admin123
```

> ⚠️ **Si estás probando todo en una sola computadora** (Windows/Mac con Docker Desktop), usar `host.docker.internal` en lugar de las IPs Tailscale:
> ```env
> DB_HOST=host.docker.internal
> REPORT_SERVICE_URL=http://host.docker.internal:8081
> ```

```bash
# Levantar (la primera vez tarda 2-5 minutos porque compila el JAR)
docker compose up -d

# Ver el progreso en tiempo real
docker compose logs -f
```

Cuando aparezca esta línea en los logs, la aplicación está lista:
```
Started PrimeraApiApplication in X.XXX seconds
```

---

### Paso 5 — Verificar que todo funciona

| Servicio | URL | Credenciales |
|---|---|---|
| Aplicación principal | `http://IP_MAQUINA1:8080` | admin / admin123 |
| Health Spring | `http://IP_MAQUINA1:8080/actuator/health` | — |
| Reportes health | `http://IP_MAQUINA1:8080/reportes/health` | — |
| pgAdmin | `http://IP_MAQUINA2:5050` | admin@inventario.com / admin123 |
| Swagger Reportes | `http://IP_MAQUINA3:8081/docs` | — |

**Prueba de extremo a extremo:**
```bash
# ¿Spring conecta con la BD?
curl http://IP_MAQUINA1:8080/actuator/health
# {"status":"UP"}

# ¿Spring conecta con Reportes?
curl http://IP_MAQUINA1:8080/reportes/health
# {"status":"ok","db":"conectada"}

# ¿Los reportes funcionan?
curl http://IP_MAQUINA1:8080/reportes/stock-bajo
```

---

### Detener los servicios

```bash
# En cada máquina, dentro de la carpeta del proyecto:
docker compose down

# Para eliminar también los volúmenes (borra los datos de la BD):
docker compose down -v
```

---

## Variables de Entorno

### Máquina 2 — Base de Datos (`.env`)

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `DB_NAME` | Nombre de la base de datos | `Comida` |
| `DB_USER` | Usuario de PostgreSQL | `postgres` |
| `DB_PASSWORD` | Contraseña de PostgreSQL | `12345` |
| `PGADMIN_EMAIL` | Email para pgAdmin | `admin@inventario.com` |
| `PGADMIN_PASSWORD` | Contraseña de pgAdmin | `admin123` |

### Máquina 3 — Servicio de Reportes (`.env`)

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_HOST` | IP Tailscale de Máquina 2 | `100.x.x.2` |
| `DB_PORT` | Puerto de PostgreSQL | `5432` |
| `DB_NAME` | Nombre de la base de datos | `Comida` |
| `DB_USER` | Usuario de PostgreSQL | `postgres` |
| `DB_PASSWORD` | Contraseña de PostgreSQL | `12345` |

### Máquina 1 — Aplicación Spring (`.env`)

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_HOST` | IP Tailscale de Máquina 2 | `100.x.x.2` |
| `DB_PORT` | Puerto de PostgreSQL | `5432` |
| `DB_NAME` | Nombre de la base de datos | `Comida` |
| `DB_USER` | Usuario de PostgreSQL | `postgres` |
| `DB_PASSWORD` | Contraseña de PostgreSQL | `12345` |
| `REPORT_SERVICE_URL` | URL del servicio de reportes | `http://100.x.x.3:8081` |
| `APP_USER` | Usuario para login en la app | `admin` |
| `APP_PASSWORD` | Contraseña para login en la app | `admin123` |

---

## Endpoints disponibles

### Aplicación Spring (Máquina 1 — puerto 8080)

**Vistas web (Thymeleaf):**

| Método | URL | Descripción |
|---|---|---|
| GET | `/productos` | Lista de productos |
| GET | `/productos/nuevo` | Formulario nuevo producto |
| GET | `/productos/{id}/editar` | Editar producto |
| GET | `/categorias` | Lista de categorías |
| GET | `/proveedores` | Lista de proveedores |
| GET | `/movimientos` | Lista de movimientos con filtros |
| GET | `/movimientos/nuevo` | Formulario nuevo movimiento |
| GET | `/movimientos/{id}` | Detalle de un movimiento |
| GET | `/reportes` | Dashboard de reportes |

**API REST:**

| Método | URL | Descripción |
|---|---|---|
| GET | `/inventario` | Listar productos |
| GET | `/inventario/{id}` | Buscar producto por ID |
| POST | `/inventario/guardar/producto` | Crear producto |
| PUT | `/inventario/editar/{id}` | Editar producto |
| DELETE | `/inventario/eliminar/{id}` | Eliminar producto |
| GET | `/inventario/filtrar/precio/mayor_a?precio=` | Filtrar por precio |
| GET | `/inventario/filtrar/categoria?nombre=` | Filtrar por categoría |
| GET | `/inventario/proveedores` | Listar proveedores |
| POST | `/inventario/proveedores/guardar` | Crear proveedor |
| PUT | `/inventario/proveedores/editar/{id}` | Editar proveedor |
| DELETE | `/inventario/proveedores/eliminar/{id}` | Eliminar proveedor |
| GET | `/inventario/movimientos` | Listar movimientos |
| POST | `/inventario/movimientos/registrar` | Registrar movimiento |
| GET | `/inventario/movimientos/filtrar` | Filtrar movimientos |
| GET | `/inventario/movimientos/producto/{id}` | Movimientos por producto |
| GET | `/reportes/stock-bajo` | Reporte: stock bajo |
| GET | `/reportes/mas-movidos?fecha_inicio=&fecha_fin=` | Reporte: más movidos |
| GET | `/reportes/valor-inventario` | Reporte: valor inventario |
| GET | `/reportes/movimientos?fecha_inicio=&fecha_fin=` | Reporte: movimientos |
| GET | `/reportes/resumen-proveedor` | Reporte: por proveedor |
| GET | `/actuator/health` | Health check |

### Servicio de Reportes (Máquina 3 — puerto 8081)

| Método | URL | Descripción |
|---|---|---|
| GET | `/reportes/stock-bajo` | Productos con stock bajo |
| GET | `/reportes/stock-bajo/csv` | Exportar stock bajo a CSV |
| GET | `/reportes/mas-movidos?fecha_inicio=&fecha_fin=` | Top 10 más movidos |
| GET | `/reportes/valor-inventario` | Valor del inventario por categoría |
| GET | `/reportes/movimientos?fecha_inicio=&fecha_fin=[&tipo=]` | Movimientos por fecha |
| GET | `/reportes/resumen-proveedor` | Resumen por proveedor |
| GET | `/health` | Health check |
| GET | `/docs` | Documentación interactiva (Swagger) |
