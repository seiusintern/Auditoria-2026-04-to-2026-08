# SEIUS APU — Documentación del Sistema

> **Sistema de Análisis de Precios Unitarios y Gestión de Presupuestos de Obra**
> Versión del documento: 1.0 · Fecha: 2026-08-27

---

## Tabla de Contenidos

1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Stack tecnológico](#2-stack-tecnológico)
3. [Arquitectura del sistema](#3-arquitectura-del-sistema)
4. [Modelo de dominio](#4-modelo-de-dominio)
5. [Módulos funcionales](#5-módulos-funcionales)
6. [Integración con Schneider Electric](#6-integración-con-schneider-electric)
7. [Generación de reportes (PDF / Excel)](#7-generación-de-reportes-pdf--excel)
8. [Modelo de datos (PostgreSQL)](#8-modelo-de-datos-postgresql)
9. [Infraestructura y despliegue](#9-infraestructura-y-despliegue)
10. [Configuración y variables de entorno](#10-configuración-y-variables-de-entorno)
11. [Seguridad y manejo de secretos](#11-seguridad-y-manejo-de-secretos)
12. [Colección Postman — Schneider API](#12-colección-postman--schneider-api)
13. [Estructura de carpetas](#13-estructura-de-carpetas)
14. [Comandos útiles de desarrollo](#14-comandos-útiles-de-desarrollo)
15. [Glosario](#15-glosario)

---

## 1. Resumen ejecutivo

**SEIUS APU** es una aplicación web empresarial orientada a la **construcción** que permite:

- Gestionar un **banco de insumos** (equipos, mano de obra y materiales) con precios geográficos por provincia.
- Crear y mantener **rubros / matrices APU** (*Análisis de Precios Unitarios*) calculando automáticamente rendimientos y costos.
- Armar **presupuestos de obra** estructurados en capítulos, rubros, programación mensual y costos indirectos.
- Consultar en línea **precios y disponibilidad** de productos Schneider Electric directamente desde la API oficial.
- Generar reportes profesionales en **PDF vectorial** y **Excel** para imprimir, compartir o presentar a clientes.

El sistema está pensado para el mercado ecuatoriano (moneda USD, provincias EC, cliente distribuidor de Schneider Electric) y se entrega como una sola aplicación web full-stack: el backend Spring Boot sirve tanto la API como la UI Vaadin, sin necesidad de un servidor frontend separado.

---

## 2. Stack tecnológico

| Capa               | Tecnología                                          |
|--------------------|-----------------------------------------------------|
| Lenguaje           | Java 22                                             |
| Framework backend  | Spring Boot 3.2.4                                   |
| Framework UI       | Vaadin 24.3.7 (Flow + Lit + componentes Lumo)       |
| ORM                | Spring Data JPA / Hibernate                         |
| Base de datos      | PostgreSQL 16                                       |
| Build backend      | Maven 3.9                                           |
| Build / bundle FE  | Vite 5.1 + Rollup                                   |
| Lenguaje FE        | TypeScript 5.3                                      |
| Generación PDF     | openhtmltopdf-pdfbox 1.0.10 (vectorial)            |
| Generación Excel   | Apache POI 5.2.5 (formato `.xlsx`)                  |
| HTTP client        | Spring `RestClient`                                 |
| Contenedorización  | Docker multi-stage, Docker Compose                  |
| Reverse proxy      | Traefik (gestionado por Dokploy)                    |
| Despliegue         | Dokploy (panel web) sobre VPS                       |
| API externa        | Schneider Electric (api.se.com) — OAuth2            |

---

## 3. Arquitectura del sistema

El proyecto sigue una **arquitectura hexagonal** (Ports & Adapters) ligeramente simplificada, con cuatro capas bien separadas dentro de `src/main/java/com/seius/apu`:

```
com.seius.apu
├── APU.java                       ← Punto de entrada Spring Boot
│
├── domain/                        ← Núcleo: entidades puras y modelos
│   └── model/
│       └── insumo/TipoRecurso     ← Enum: EQUIPO / MANO_DE_OBRA / MATERIAL
│
├── usecase/                       ← Lógica de aplicación / servicios
│   ├── dto/                       ← Objetos de transferencia entre capas
│   └── service/                   ← Servicios transaccionales (@Service)
│
├── infrastructure/                ← Adaptadores hacia el exterior
│   ├── database/
│   │   ├── entity/                ← Entidades JPA (@Entity)
│   │   └── repository/            ← Spring Data + repositorios custom
│   └── schneider/                 ← Cliente HTTP + token manager + DTOs
│       ├── dto/                   ← Mapeos de la API de Schneider
│       ├── SchneiderProperties    ← @ConfigurationProperties
│       ├── SchneiderConfig        ← Bean RestClient + token
│       ├── SchneiderTokenManager  ← Cache OAuth2 thread-safe
│       ├── SchneiderApiClient     ← Endpoints REST
│       └── SchneiderException
│
└── ui/                            ← Adaptador de entrada: vistas Vaadin
    ├── MainLayout                 ← Layout principal (AppLayout + SideNav)
    ├── HomeView                   ← Redirige "/" → "/insumos"
    ├── InsumoView                 ← Banco de insumos
    ├── ApuView / ApuEditorDialog  ← Matrices APU
    ├── ConsultaSchneiderView      ← Consulta precios Schneider
    └── presupuesto/               ← Módulo de presupuestos
        ├── PresupuestoView        ← Listado + creación
        ├── PresupuestoDetalleView ← Editor de presupuesto
        ├── PresupuestoModalDialog ← Diálogo nuevo presupuesto
        ├── PresupuestoFilaDTO
        └── componentes/           ← Sub-pestañas del editor
            ├── PresupuestoDetalleTab
            ├── ProgramacionTab
            ├── CostosIndirectosTab
            ├── ReportesTab
            ├── SeleccionInsumoDialog
            └── SeleccionarApuModalDialog
```

### Principios aplicados

- **Inversión de dependencias**: la UI depende de los servicios de `usecase/`, nunca directamente de la infraestructura.
- **Transacciones**: los métodos de escritura están anotados con `@Transactional`; las consultas intensivas con `@Transactional(readOnly = true)`.
- **Validación de esquema**: `spring.jpa.hibernate.ddl-auto=validate` — la app no arranca si el modelo JPA no coincide con la BD.
- **Plantillas HTML/CSS para PDF**: viven en `src/main/resources/templates/seius/`, se renderizan con openhtmltopdf.

---

## 4. Modelo de dominio

El dominio gira alrededor de tres conceptos principales: **Insumos → APUs → Presupuestos**, con Schneider como fuente externa opcional de precios.

```
┌─────────────────────┐    ┌──────────────────────┐    ┌────────────────────────�
│  Insumo (recurso)   │───▶│   Rubro (APU)        │───▶│  Rubro en Presupuesto  │
│                     │    │                      │    │                        │
│ - equipo            │    │ - equipo             │    │ - cantidad             │
│ - mano de obra      │    │ - mano de obra       │    │ - precio unitario      │
│ - material          │    │ - material           │    │ - subtotal             │
│ - precios por       │    │ - rendimiento        │    │                        │
│   provincia         │    │ - costo total        │    │ agrupado en capítulos  │
└─────────────────────┘    └──────────────────────�    └────────────────────────┘
          ▲                                                       │
          │                                                       ▼
┌─────────────────────�                                ┌────────────────────────┐
│ InsumoGrupo         │                                │ Presupuesto             │
│ RubroGrupo          │                                │  - capítulos           │
└─────────────────────┘                                │  - rubros              │
                                                      │  - programación        │
                                                      │  - costos indirectos   │
                                                      │  - reportes            │
                                                      └────────────────────────┘
```

### Conceptos clave

- **Insumo**: recurso indivisible (equipo, persona-meses, material). Tiene un precio base y precios variantes por provincia (`InsumoPrecioProvinciaEntity`).
- **Rubro / APU**: "receta" que combina insumos con cantidades y rendimientos para producir una unidad de obra. El cálculo del costo se realiza en `ApuServiceImpl`.
- **Presupuesto**: instancia de obra con uno o varios capítulos. Cada capítulo contiene rubros con cantidades, dando el costo total.

---

## 5. Módulos funcionales

### 5.1 Banco de Insumos (`/insumos`)

Vista principal: `InsumoView`.

Funcionalidad:
- ABM de insumos agrupados por `InsumoGrupo` (categorías).
- Cada insumo tiene un `TipoRecurso` (`EQUIPO`, `MANO_DE_OBRA`, `MATERIAL`).
- Edición de **precios por provincia** (`InsumoPrecioProvinciaEntity`).
- Búsqueda en vivo y filtros.

### 5.2 Análisis de Precios Unitarios (`/apu`)

Vista principal: `ApuView` + `ApuEditorDialog`.

Funcionalidad:
- Gestión de **grupos de rubros** (`RubroGrupo`).
- Creación de **rubros / matrices APU** combinando insumos.
- Cálculos automáticos:
  - **Rendimiento en horas** = `1 / (rendimiento_diario / 8)`
  - **Costo equipo / mano de obra** = `cantidad × costo_hora × rendimiento_horas`
  - **Costo material** = `cantidad × precio`
- Vista por acordeón para agrupar rubros por grupo.

### 5.3 Presupuestos de Obra (`/presupuestos`)

Vista principal: `PresupuestoView` + `PresupuestoDetalleView` (con pestañas).

Estructura del editor (4 pestañas):

| Pestaña              | Componente                | Función                                                       |
|----------------------|---------------------------|---------------------------------------------------------------|
| **Detalle**          | `PresupuestoDetalleTab`   | Capítulos y rubros del presupuesto                            |
| **Programación**     | `ProgramacionTab`         | Distribución mensual del avance físico y financiero           |
| **Costos Indirectos**| `CostosIndirectosTab`     | Porcentajes de administración, utilidad, impuestos, etc.      |
| **Reportes**         | `ReportesTab`             | Generación de PDF y Excel, configuración del pie de página    |

Funcionalidad adicional:
- **Clonar presupuesto** (`PresupuestoService#clonarPresupuesto`): duplica un presupuesto completo con sus capítulos y rubros.
- **Eliminación en cascada**: borrar un presupuesto elimina también sus costos indirectos asociados.
- Configuración persistente del pie de página: `gerenteProyecto`, `representanteLegal`, `empresaConstructora` (se imprimen en todos los PDFs).

### 5.4 Consulta Schneider (`/consulta-schneider`)

Vista principal: `ConsultaSchneiderView`.

Permite ingresar:
- `catalogNumber` (referencia Schneider, p. ej. `RMCA61BD`).
- `quantity` (cantidad).

Y muestra todos los campos devueltos por la API oficial:
- Precio lista, precio neto, total
- Moneda y fecha del precio
- Grupo de material
- Stock (vía `standardLeadTime`)
- Mensajes de la API

### 5.5 Mi Cuenta (`/cuenta`)

Reservada en el menú lateral. Pendiente de implementación.

---

## 6. Integración con Schneider Electric

La aplicación consume dos endpoints REST de `api.se.com`:

### 6.1 Product Availability V2

```
GET {apiBaseUrl}/v2/sales-operation/product-availability/products/{reference}?quantity=N
→ Stock y lead time.
```

### 6.2 Product Net Price (Distributor)

```
GET {apiBaseUrl}/v1/customer-journey/distributor/product-netprice/product-reference
   ?catalog-number=X&quantity=Y&currencyCode=USD
→ Precio lista, precio neto y total.
```

### 6.3 Autenticación OAuth2 (client_credentials)

```
POST {tokenUrl}
Authorization: Basic base64(client_id : client_secret)
Content-Type:  application/x-www-form-urlencoded
Body:          grant_type=client_credentials

→ 200 { "access_token": "...", "expires_in": 3600, ... }
```

Implementación en `SchneiderTokenManager`:

| Aspecto                  | Detalle                                                                                  |
|--------------------------|------------------------------------------------------------------------------------------|
| Cache                    | `AtomicReference<CachedToken>` — lectura sin lock.                                       |
| Concurrencia             | `ReentrantLock` — un solo hilo refresca, los demás esperan (double-check under lock).    |
| Buffer de expiración     | `schneider.token-refresh-buffer-seconds=60` — evita tokens expirados en vuelo.           |
| Recuperación ante 401    | `SchneiderApiClient#invalidate()` dispara un nuevo intento con token fresco.             |
| Diagnóstico de arranque  | `@PostConstruct` loggea si las credenciales están presentes o faltan.                    |

### 6.4 DTOs de la integración

| DTO                                  | Propósito                                        |
|--------------------------------------|--------------------------------------------------|
| `SchneiderAvailabilityResponse`      | Stock y lead time                                |
| `SchneiderPriceResponse`             | Precios (lista, neto, total)                     |
| `SchneiderProductInformationResponse`| Información general del producto                 |
| `SchneiderProductCatalogResponse`    | Catálogo                                         |
| `SchneiderCombinedProductInfo`       | Vista consolidada para la UI                     |
| `SchneiderProductAvailability`       | Modelo de dominio (en `domain/model/`)           |

`SchneiderCatalogService` orquesta la combinación de precio + disponibilidad para la vista.

---

## 7. Generación de reportes (PDF / Excel)

### 7.1 Reportes disponibles

Todos se generan desde `ReportePdfService` (no confundir con el nombre — también genera Excel):

| Reporte                                | Método                                    |
|----------------------------------------|-------------------------------------------|
| Reporte general del presupuesto        | `generarReportePresupuestoGeneral(id)`     |
| Análisis de Precios Unitarios (APU)    | `generarReporteApu(id)`                    |
| Precios unitarios elementales          | `generarReportePreciosUnitarios(id)`       |
| Reporte individual por tipo de recurso  | `generarReportePorTipoRecurso(id, tipo)`   |

### 7.2 Tecnología PDF

- **Motor**: `openhtmltopdf-pdfbox` (renderiza HTML/CSS 2.1 → PDF vectorial).
- **Ventajas**: nitidez en impresión profesional, tamaño de archivo reducido vs. alternativas basadas en imágenes.
- **Silenciado de logs**: `XRLog.setLoggingEnabled(false)` para evitar ruido en consola.
- **Plantillas**: HTML + CSS en `src/main/resources/templates/seius/`.

### 7.3 Tecnología Excel

- **Motor**: Apache POI (`XSSFWorkbook`) → archivos `.xlsx`.
- Permite al usuario abrir, editar o analizar los datos en Excel/LibreOffice sin perder estructura.

### 7.4 Cabecera y pie de página compartidos

Todos los reportes comparten:
- **Encabezado**: nombre del proyecto.
- **Pie de página**: configuración persistente del presupuesto (`gerenteProyecto`, `representanteLegal`, `empresaConstructora`).

Los campos son **opcionales** y **nullable** para mantener compatibilidad con presupuestos antiguos.

---

## 8. Modelo de datos (PostgreSQL)

Entidades JPA en `infrastructure/database/entity/`:

| Entidad                          | Tabla                     | Propósito                                                   |
|----------------------------------|---------------------------|-------------------------------------------------------------|
| `InsumoEntity`                   | `insumos`                 | Recurso base (equipo, mano de obra, material)               |
| `InsumoGrupoEntity`              | `insumos_grupos`          | Categoría de insumo                                         |
| `InsumoPrecioProvinciaEntity`    | `insumos_precios_provincia` | Precios del insumo por provincia                          |
| `RubroEntity`                    | `rubros`                  | Matriz APU (receta de obra)                                 |
| `RubroGrupoEntity`               | `rubros_grupos`           | Categoría de rubro                                          |
| `ApuDetalleEntity`               | `apu_detalles`            | Línea de detalle (insumo + cantidad + rendimiento)          |
| `PresupuestoEntity`              | `presupuestos`            | Cabecera del presupuesto + pie de página de reportes        |
| `PresupuestoCapituloEntity`      | `presupuestos_capitulos`  | Capítulo dentro de un presupuesto                           |
| `PresupuestoRubroEntity`         | `presupuestos_rubros`     | Rubro dentro de un capítulo                                 |
| `PresupuestoRubroDetalleEntity`  | `presupuestos_rubros_detalles` | Desglose de un rubro del presupuesto                   |
| `PresupuestoInsumoPrecioEntity`  | `presupuestos_insumos_precios` | Precio snapshot del insumo al momento del presupuesto |
| `PresupuestoCostosIndirectosEntity` | `presupuestos_costos_indirectos` | Costos indirectos del presupuesto                    |

### Repositorios

- Repositorios **Spring Data JPA** (`JpaRepository`) — `InsumoJpaRepository`, `InsumoPrecioProvinciaJpaRepository`.
- Repositorios **custom** con lógica adicional — `PresupuestoRepository`, `RubroRepository`, etc.

---

## 9. Infraestructura y despliegue

### 9.1 Docker

`Dockerfile` multi-stage:

```dockerfile
# Etapa 1: build
FROM maven:3.9-eclipse-temurin-22 AS build
- Instala Node.js 20 (necesario para Vaadin/Vite)
- mvn dependency:go-offline
- mvn ... vaadin-maven-plugin:24.3.7:prepare-frontend
- mvn ... vaadin-maven-plugin:24.3.7:build-frontend
- mvn package -Dvaadin.productionMode=true -DskipTests

# Etapa 2: runtime
FROM eclipse-temurin:22-jre-jammy
- Copia el .jar con el frontend ya inyectado
- EXPOSE 8090
- ENTRYPOINT ["java", "-Dvaadin.productionMode=true", "-jar", "app.jar"]
```

### 9.2 Docker Compose

Dos servicios en `docker-compose.yml`:

| Servicio     | Imagen                        | Puerto host → contenedor | Notas                                                  |
|--------------|-------------------------------|--------------------------|--------------------------------------------------------|
| `postgres-db`| `postgres:16-alpine`          | `127.0.0.1:5435 → 5432`  | Solo localhost en dev; interno en Dokploy.             |
| `apu-app`    | build local                   | (interno 8090)           | Expuesto a través de Traefik con TLS automático.        |

Redes:
- `apu-network` (bridge) — red interna entre los dos servicios.
- `dokploy-network` (external) — red compartida con el reverse proxy.

Volumen persistente: `postgres_apu_data` → `/var/lib/postgresql/data`.

### 9.3 Despliegue en producción

- **Plataforma**: Dokploy sobre VPS.
- **Proxy / TLS**: Traefik con Let's Encrypt (labels declarados en `docker-compose.yml`).
- **Host público**: `apu.seius.com.ec` (configurable vía label `traefik.http.routers.seius-apu.rule`).
- **Healthcheck Postgres**: `pg_isready` cada 10 s.
- **Restart policy**: `restart: always` en ambos servicios.

---

## 10. Configuración y variables de entorno

### 10.1 `application.properties` (valores por defecto / fallback)

```properties
server.port=8090

spring.datasource.url=jdbc:postgresql://localhost:5432/APU
spring.datasource.username=postgres
spring.datasource.password=admin
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update          # ⚠ En prod idealmente: validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

schneider.token-url=${SCHNEIDER_TOKEN_URL:https://api.se.com/token}
schneider.api-base-url=${SCHNEIDER_API_BASE_URL:https://api.se.com}
schneider.client-id=${SCHNEIDER_CLIENT_ID:}
schneider.client-secret=${SCHNEIDER_CLIENT_SECRET:}
schneider.scope=${SCHNEIDER_SCOPE:}
schneider.country=${SCHNEIDER_COUNTRY:EC}
schneider.currency=${SCHNEIDER_CURRENCY:USD}
schneider.token-refresh-buffer-seconds=60
```

### 10.2 `.env.example` (plantilla — nunca se commitea `.env`)

```env
POSTGRES_DB=apu_db
POSTGRES_USER=seius
POSTGRES_PASSWORD=cambiar_en_produccion

SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-db:5432/apu_db
SPRING_DATASOURCE_USERNAME=seius
SPRING_DATASOURCE_PASSWORD=cambiar_en_produccion

SCHNEIDER_TOKEN_URL=https://api.se.com/token
SCHNEIDER_API_BASE_URL=https://api.se.com
SCHNEIDER_CLIENT_ID=tu_client_id_aqui
SCHNEIDER_CLIENT_SECRET=tu_client_secret_aqui
SCHNEIDER_SCOPE=
SCHNEIDER_COUNTRY=EC
SCHNEIDER_CURRENCY=USD
```

### 10.3 Flujo de variables

```
.env (local)            → docker-compose → contenedores
Dokploy Environment     → docker-compose → contenedores
application.properties → valores por defecto / fallback si no hay env var
```

---

## 11. Seguridad y manejo de secretos

- ✅ **`.env` está en `.gitignore`** — nunca se commitea.
- ✅ **Credenciales reales solo viven en Dokploy** (producción) o en `.env` local (desarrollo).
- ✅ **`docker-compose.yml` NO contiene secretos** — solo referencias a variables.
- ✅ **`POSTGRES_PASSWORD` debe ser IGUAL a `SPRING_DATASOURCE_PASSWORD`** (lo recalca el `.env.example`).
- ⚠ **No usar `seius`, `admin` u otras contraseñas débiles en producción**.

Auditoría de credenciales Schneider al arranque (`@PostConstruct` en `SchneiderTokenManager`):
- ✅ Si están presentes: loggea longitudes y URLs.
- ❌ Si faltan: loggea WARN con los nombres de variables a definir.

---

## 12. Colección Postman — Schneider API

Archivo: `SEIUS SA EC.postman_collection (1).json`

Tres requests listos para importar en Postman:

| Request                  | Método | URL                                                                                     |
|--------------------------|--------|-----------------------------------------------------------------------------------------|
| **Token API**            | POST   | `https://api.se.com/token`                                                              |
| **Product Availability V2** | GET  | `https://api.se.com/v2/sales-operation/product-availability/products/RMCA61BD?quantity=12` |
| **Net Price**            | GET    | `https://api.se.com/v1/customer-journey/distributor/product-netprice/product-reference?catalog-number=RMCA61BD&quantity=10` |

### Variables de colección

| Variable          | Descripción                                         |
|-------------------|-----------------------------------------------------|
| `customer_name`   | `SEIUS SA EC`                                       |
| `client_id`       | Tu client_id de Schneider                           |
| `client_secret`   | Tu client_secret de Schneider                       |
| `access_token`    | Token OAuth2 (se setea automáticamente desde el test del request "Token API") |
| `tokenExpiry`     | Timestamp de expiración calculado por el test        |

### Test automático en "Token API"

Tras cada request de token, el test:
1. Verifica status `200`/`201` y presencia de `access_token`.
2. Guarda `access_token` en la variable de colección.
3. Calcula expiración = `now + (expires_in - 60) × 1000` (60 s de margen) y la guarda en `tokenExpiry`.

> ℹ️ La app Java reproduce esta misma lógica en `SchneiderTokenManager` (buffer configurable vía `schneider.token-refresh-buffer-seconds`).

---

## 13. Estructura de carpetas

```
Seius-Apu/
├── .env.example                       # Plantilla de variables de entorno
├── .gitignore
├── Dockerfile                         # Build multi-stage (Maven + Node + JRE)
├── docker-compose.yml                 # Postgres + app + redes Traefik
├── package.json                       # Dependencias Vaadin/Vite
├── package-lock.json
├── pom.xml                            # Spring Boot + Vaadin + PDF + Excel
├── vite.config.ts                     # Configuración Vite (generada por Vaadin)
├── vite.generated.ts
├── tsconfig.json
├── types.d.ts
│
├── frontend/                          # Frontend Vaadin (TS + Lit)
│   ├── styles/
│   └── generated/                     # Generado por Vaadin (no editar)
│
├── src/main/
│   ├── java/com/seius/apu/            # Backend Java (ver §3)
│   └── resources/
│       ├── application.properties
│       └── templates/seius/           # Plantillas HTML para PDF
│
├── src/main/java/com/seius/apu/ui/presupuesto/componentes/
│   ├── ProgramacionTab.java
│   ├── CostosIndirectosTab.java
│   ├── ReportesTab.java
│   ├── PresupuestoDetalleTab.java
│   ├── SeleccionInsumoDialog.java
│   └── SeleccionarApuModalDialog.java
│
├── target/                            # Artefactos de build Maven
├── node_modules/                      # Dependencias Node (Vaadin)
│
├── README.md                          # (placeholder)
├── DOCUMENTATION.md                   # ← este archivo
├── nbactions.xml                      # Acciones de NetBeans IDE
├── nb-configuration.xml
│
├── "OFERTA SEIUS 2026.docx"          # Documento comercial (referencia)
├── "Captura de pantalla 2026-08-24 160735.png"  # Captura de UI
└── "SEIUS SA EC.postman_collection (1).json"    # Colección Postman Schneider
```

---

## 14. Comandos útiles de desarrollo

### Build local con Maven

```bash
# Compilar y empaquetar (modo desarrollo)
mvn clean package

# Empaquetar para producción (con bundle frontend inyectado)
mvn clean package -Pproduction
mvn clean package -Dvaadin.productionMode=true
```

### Levantar con Docker Compose (desarrollo local)

```bash
# 1) Crear tu archivo .env a partir de la plantilla
cp .env.example .env
# 2) Editar .env y rellenar valores reales
# 3) Construir y levantar
docker compose up -d --build
# 4) Ver logs
docker compose logs -f apu-app
```

### Conectarse a Postgres (dev)

```bash
docker exec -it seius-apu-postgres psql -U seius -d apu_db
# Host: localhost   Puerto: 5435
```

### Inspeccionar logs de Schneider

```bash
docker compose logs apu-app | grep -i schneider
```

---

## 15. Glosario

| Término | Significado |
|---------|-------------|
| **APU** | Análisis de Precios Unitarios — descomposición de una unidad de obra en insumos con cantidades y rendimientos. |
| **Rubro** | "Receta" de una unidad de obra en el sistema APU. |
| **Insumo** | Recurso base (equipo, persona-meses, material). |
| **Presupuesto** | Instancia de obra que agrupa capítulos, rubros, programación y costos indirectos. |
| **Capítulo** | Agrupador de rubros dentro de un presupuesto. |
| **Costos indirectos** | Porcentajes adicionales al costo directo: administración, utilidad, impuestos, etc. |
| **Schneider Electric** | Fabricante de equipamiento eléctrico cuya API oficial (`api.se.com`) consulta precios y stock. |
| **Dokploy** | Plataforma de despliegue self-hosted con Traefik integrado, usada para producción. |
| **Traefik** | Reverse proxy con auto-TLS vía Let's Encrypt, configurado por labels en `docker-compose.yml`. |
| **Hexagonal** | Estilo de arquitectura que separa dominio, casos de uso y adaptadores externos. |

---

> Documento generado automáticamente a partir del código fuente del repositorio.
> Si modificas la arquitectura, los endpoints o las variables de entorno, **actualiza también este archivo**.
