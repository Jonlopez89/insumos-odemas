---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - '_producto/project-context.md'
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md'
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/addendum.md'
workflowType: 'architecture'
project_name: 'insumos-odemas'
user_name: 'Jonathan'
date: '2026-05-26'
lastStep: 8
status: 'complete'
completedAt: '2026-05-27'
---

# Documento de Decisiones de Arquitectura — insumos-odemas

_Este documento se construye de forma colaborativa, paso a paso. Las secciones se van agregando conforme avanzamos en cada decisión arquitectónica en conjunto._

## Análisis del Contexto del Proyecto

### Resumen de requisitos

**Requisitos funcionales (~50 FRs activos en 10 áreas de capacidad):**

El sistema automatiza la consolidación distrital semanal de la solicitud de insumos
de papel para 226 servicentros. Arquitectónicamente, los FRs se agrupan en dos
naturalezas distintas:

- **Núcleo batch / motor (C-1, C-2, C-4, C-5, C-6, C-9):** datos maestros, ingesta
  y normalización de archivos heterogéneos, forecast por tienda × SKU (cadencia
  semanal, unidad quincenal), conversión venta→insumo vía cruce autoritativo,
  cuantización al empaque mínimo, validación de techo presupuestal con recorte
  sugerido, y aplicación del factor único de pérdida por tipo de papel. Es un
  **pipeline composable de 8 pasos** (parsing → validación → catálogo → joins →
  forecast → cuantización → ventana-presupuesto → output), cada paso testeable
  en aislamiento (NFR-T-1). El `ForecastingEngine` (Java puro) es una pieza de
  ese pipeline, no el sistema entero.
- **Interactivo / web (C-7, C-8, C-10, C-11, C-12):** bandeja distrital del
  coordinador (sugerencias, overrides con razón estructurada, recortes, agregados),
  histórico TFS cruzado, dashboard pedido-vs-recibido (FR-067), registro de
  excepciones mid-cycle, bitácora inmutable de auditoría, y export al comprador
  compatible con `SolicitusDeInsumosTodos.xlsx`.

**Requisitos no-funcionales que dirigen la arquitectura:**

- **Escala (NFR-E):** ~8,600 series temporales (226 × ~38 SKUs venta) → ~2,260
  cantidades de insumo. Lote completo <5 min en una sola JVM. Concurrencia humana
  máxima: **3 coordinadores + 1 administrador**. La escala NO es el factor de
  diseño dominante; lo es la correctitud.
- **Performance percibida (NFR-P):** recálculo optimista en cliente <100 ms con
  confirmación servidor (RxJS `debounceTime(150)`) → frontend Angular reactivo.
  Autosave silencioso.
- **Auditabilidad (NFR-A):** bitácora **inmutable** — correcciones como entradas
  nuevas que referencian la anterior; MDC con `tiendaId/skuId/cicloId/usuarioId`.
- **Calidad del dato (NFR-DAT):** fail-loud con excepciones checked específicas;
  detección y registro de encoding por archivo; SLA de frescura del snapshot SIM
  de 36 h (rechazar operar si se excede).
- **Seguridad (NFR-S):** identidad individual (Auth0 sugerido), MFA para roles
  administrativos, roles Coordinador / Administrador, datos sensibles (presupuestos,
  costos) no expuestos a roles que no los necesitan.
- **Testing (NFR-T):** golden datasets versionados e inmutables firmados por Hugo,
  property-based con jqwik sobre cuantización/presupuesto, contract testing
  (Pact / Spring Cloud Contract), backtesting sobre histórico real, mutation
  testing PIT ≥80% en `forecasting.*`.
- **Locale (NFR-L) y A11y (NFR-A11Y):** es-MX (fechas `dd/MM/yyyy`, semanas ISO,
  MXN, cantidades discretas con equivalencia piezas↔cajas); WCAG 2.1 AA mínimo.

**Escala y complejidad:**

- Dominio primario: **full-stack con núcleo batch de forecasting** (web interactivo
  de bajo volumen + motor analítico Java).
- Nivel de complejidad: **media** — concentrada en correctitud del dominio y calidad
  del dato sucio/heterogéneo, no en escala ni concurrencia.
- Componentes arquitectónicos estimados: ~5–7 (SPA Angular, API Spring Boot,
  motor de forecast, capa de ingesta/pipeline, persistencia PostgreSQL con esquemas
  raw/staging/mart, almacenamiento de archivos/exports, notificación por correo).

### Restricciones técnicas y dependencias

- **Stack obligatorio (no negociable, `project-context.md`):** Java + Spring Boot
  (Docker) → PostgreSQL en Cloud SQL → Angular en Firebase Hosting. **GCP exclusivo.**
  Forecasting en **Java puro** (Smile / Apache Commons Math) tras interfaz
  `ForecastingEngine`. Sin Python / Vertex AI / BigQuery ML sin RFC formal.
- **Sin integraciones externas en vivo en v1** (decisión 2026-05-26b): ventas
  semanales y ALLOC/TFS llegan como **reportes que aporta el owner** (mismo patrón
  de ingesta que `docs/`); SIM como **lectura unidireccional de snapshots** (extract
  nocturno, modelo C). Desaparece la dependencia de BigQuery/RMS de v1.
- **Convenciones de datos transversales:** fechas `dd/MM/yyyy[ HH:mm]` con parser
  dual; encoding UTF-8 BOM → cp1252 con detección registrada; `BigDecimal` para
  montos (MXN, scale=4); IDs de tienda/SKU como `String`; distinción `""` vacío
  vs columna ausente; literal `"Matarial y tamaño"` preservado por contrato.
- **Entorno local del equipo:** Windows 11 + PowerShell 5.1.
- **Decisiones tácitas del PRD a validar aquí** (addendum A3.3): bitácora inmutable
  (¿event sourcing / append-only?), export XLSX bit-a-bit compatible, recálculo
  reactivo en cliente.
- **Patrones de referencia (Tomaturno, ilustrativos no obligatorios):** Auth0+JWT,
  SendGrid, Cloud Run + Docker, Cloud SQL, Cloud Storage, Firebase Hosting.

### Concerns transversales identificados

- **Fail-loud por defecto:** excepciones checked por escenario de negocio
  (`EquivalenciaNoDefinidaException`, `MinimoPackNoConfiguradoException`,
  `PresupuestoExcedidoException`, `PeriodicidadPresupuestoIndefinidaException`,
  `SaldoSIMInconsistenteException`, `StatusDesconocidoException`). Nunca degradar
  silenciosamente, nunca interpolar.
- **Trazabilidad y auditoría:** bitácora inmutable + MDC en logs + cadena de
  override visible en UI + export a CSV. Es requisito de primera clase, no añadido.
- **Pureza de la capa de dominio:** reglas de conversión y cuantización en POJOs
  testeables sin Spring; framework por fuera del núcleo.
- **Locale es-MX:** transversal en backend (parsing) y frontend (presentación).
- **Calidad del dato:** detección de encoding, parsing dual de fechas, validación
  de invariantes de ingesta (`QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE`,
  `TRAN_CODE ↔ DECODE`), SLA de frescura SIM.
- **Disciplina de testing:** golden / backtesting / property-based / contract /
  mutation — condiciona la forma del código y el plan de entrega.
- **Seguridad por rol:** autenticación individual, MFA admin, segregación de datos
  sensibles.

### Preguntas abiertas delegadas a este documento

OQ-108 (cold-start <12 meses), OQ-114 (política fail-loud ante SIM sucio/ausente/
negativo), OQ-116 (publicación de ALLOC esperado a SIM), OQ-117 parcial (formato y
dueño del extract SIM), F-05 (implementación interna del factor de pérdida:
multiplicador sobre output vs ajuste de histórico), y la elección de herramienta de
migraciones (Flyway vs Liquibase).

## Evaluación de Starter Templates

### Dominio técnico primario

**Full-stack con núcleo batch de forecasting**: API Java/Spring Boot (contenerizada,
objetivo Cloud Run) + SPA Angular (objetivo Firebase Hosting) sobre GCP. Stack
mandado por `project-context.md`, sin decisión de lenguaje.

### Starters considerados

| Opción | Veredicto | Razón |
|---|---|---|
| **Spring Initializr** (`start.spring.io`) | ✅ Adoptado (backend) | Generador oficial de Spring Boot; control total de dependencias y estructura. |
| **Angular CLI** (`ng new`) | ✅ Adoptado (frontend) | Generador oficial; v21 con esbuild y zoneless por defecto. |
| **JHipster** (full-stack generado) | ❌ Rechazado | Scaffolding CRUD-céntrico y convenciones opinadas que pelean con la regla "dominio antes que framework" (sin `@Service`/`@Autowired` en lógica core) y con la disciplina de golden datasets / backtesting / jqwik. El núcleo de v1 es dominio específico, no CRUD. |
| Starters full-stack JS (T3, RedwoodJS, Nx-only) | ❌ No aplica | Asumen backend JS/TS; el backend es Java mandado. |

### Starter seleccionado: Spring Initializr + Angular CLI (limpios, monorepo)

**Justificación:** dos generadores oficiales en un monorepo (`/backend` + `/frontend`)
dan el control fino que exige el dominio fail-loud (parsing de CSV sucios, cuantización,
bitácora inmutable, motor de forecast) sin arrastrar scaffolding que luego habría que
desmontar. Maven se alinea con las referencias existentes (`project-context.md` cita
`mvn`; testing con PIT/jqwik/Pact bien soportado en Maven).

**Línea de versiones objetivo** *(rango — el pin exacto se confirma en la story de
inicialización contra la imagen base corporativa, per `project-context.md`)*:

- **Spring Boot 3.5.x** (último parche; hoy 3.5.14) — línea madura, Spring Framework 6.
- **Java 21 LTS** — mínimo `project-context.md` es 17; 21 es el LTS maduro objetivo.
- **Angular 21.2.x** (último parche) + `@angular/cli` alineado.
- **Maven** como build tool del backend.

**Comando de inicialización — backend** *(primera story de implementación; no ejecutar
en fase de planeación)*:

```bash
# Vía Spring Initializr (start.spring.io) — Maven, Java 21, Spring Boot 3.5.x
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.5.14 \
  -d javaVersion=21 \
  -d groupId=mx.com.officedepot.insumos \
  -d artifactId=insumos-odemas-api \
  -d packaging=jar \
  -d dependencies=web,data-jpa,postgresql,validation,security,oauth2-resource-server,flyway,actuator,testcontainers \
  -o backend.zip
```

> Migraciones: `flyway` listado de forma tentativa; Flyway vs Liquibase se decide
> formalmente en las decisiones de arquitectura (paso siguiente).

**Comando de inicialización — frontend** *(primera story de implementación)*:

```bash
# Angular CLI v21.2.x — SCSS, routing, modo estricto
ng new insumos-odemas-web --style=scss --routing=true --strict=true --ssr=false
```

**Dependencias fuera del catálogo de Initializr** (se añaden al `pom.xml` después,
versiones a fijar en la story de init):

- **Motor de forecast:** Smile **o** Apache Commons Math (Java puro, tras interfaz
  `ForecastingEngine`).
- **Lectura/escritura XLSX:** Apache POI (cruce de SKUs `*.xlsx` y export
  `SolicitusDeInsumosTodos.xlsx`).
- **Parsing CSV robusto:** Jackson CSV / univocity (encoding dual, fechas duales).
- **Testing:** jqwik (property-based), PIT (mutation), Pact / Spring Cloud Contract.

### Decisiones arquitectónicas que fija el starter

- **Lenguaje & runtime:** Java 21 LTS, Spring Boot 3.5.x, Maven (`pom.xml`).
- **Estilado frontend:** SCSS; Angular standalone + zoneless (default v21).
- **Build tooling:** Maven (backend) + esbuild vía Angular CLI (frontend).
- **Testing base:** JUnit 5 + Spring Boot Test + Testcontainers (backend); Karma/Jasmine
  o Vitest según default de Angular CLI v21 (frontend). Capas avanzadas (jqwik, PIT,
  Pact, backtesting) se añaden sobre esta base.
- **Organización de código:** monorepo `/backend` (paquete `mx.com.officedepot.insumos`,
  con `forecasting.*` aislado para PIT) + `/frontend` (modelos `api/` vs `view/`
  separados per `project-context.md`).
- **Modo estricto:** Angular `strict: true` + `strictNullChecks` (no negociable).

**Nota:** la inicialización con estos comandos debe ser la **primera story de
implementación**, una vez aprobado este `architecture.md` (la regla de bloqueo de
`project-context.md` lo impide hasta entonces).

## Decisiones Arquitectónicas Centrales

### Análisis de prioridad de decisiones

**Críticas (bloquean implementación):** proveedor de auth (Auth0), motor de
persistencia (híbrido JPA+SQL), herramienta de migraciones (Flyway), modelo de
ejecución del batch de forecast, formato de contrato de error backend↔frontend.

**Importantes (moldean la arquitectura):** estado del frontend (Signals), componente
de grid editable, contract testing, caché de datos maestros, landing de ingesta en
Cloud Storage, CI/CD.

**Diferidas (post-MVP / dependen de terceros):** cold-start de tiendas <12 meses
(OQ-108), publicación de ALLOC esperado a SIM (OQ-116), caché distribuido, escalado
horizontal, app móvil y BI/DWH (out-of-scope PRD §11).

### Arquitectura de datos

- **Motor:** PostgreSQL en Cloud SQL (mandato). Esquemas `raw` (CSV/XLSX cargados sin
  transformar), `staging` (validados/normalizados), `mart` (presentación) — mandato
  `project-context.md`.
- **Acceso a datos (decisión): híbrido.** JPA/Hibernate para entidades de dominio y
  CRUD (tiendas, SKUs, factor de pérdida, excepciones, bitácora); **SQL directo
  (JdbcTemplate)** para ingesta masiva, joins venta↔insumo del pipeline y agregados
  del `mart`. Encaja con el pipeline composable de 8 pasos (NFR-T-1) y evita que el
  ORM estorbe en lo analítico.
- **Migraciones (decisión): Flyway** (SQL plano versionado, gestionado por el BOM de
  Spring Boot 3.5). Una migration por PR; nunca editar una ya aplicada — se crea otra.
- **Tipos exactos:** `NUMERIC(p,s)` ↔ `BigDecimal` (scale=4 intermedio); `BIGINT` para
  cantidades; `TIMESTAMPTZ`/`DATE`; IDs de tienda/SKU como `VARCHAR`. `snake_case`.
- **Bitácora inmutable (valida decisión tácita A3.3):** tabla **append-only**
  (`bitacora_evento`), solo INSERT — sin UPDATE/DELETE; cada corrección es una entrada
  nueva que referencia la anterior (FR-104). Es append-only, **no** event sourcing
  completo. Reforzado por permisos de BD del rol de aplicación.
- **Caché:** datos maestros (tiendas, SKUs, cruce, eventos comerciales, factor de
  pérdida) en caché en memoria por ciclo (Spring Cache + Caffeine), invalidada al
  reingestar. **Sin caché distribuido/Redis en v1** (3 usuarios, carga batch).
- **Política SIM sucio (OQ-114, propuesta):** snapshot ausente/saldo negativo/
  inconsistente para tienda×SKU → `SaldoSIMInconsistenteException`, abortar el ciclo
  de esa tienda (fail-loud); el resto del distrito continúa. Snapshot global con
  antigüedad >36h → rechazar operar y notificar admin (NFR-DAT-4).

### Autenticación y seguridad

- **Proveedor (decisión): Auth0** con OIDC/JWT (patrón Tomaturno, experiencia previa
  del equipo). Backend = **Spring Security OAuth2 Resource Server** validando el JWT;
  no se emiten credenciales propias.
- **Autorización:** roles `COORDINADOR` y `ADMINISTRADOR` desde claims del token.
  **Scoping territorial obligatorio:** el coordinador solo accede a las tiendas de su
  distrito — filtro a nivel de servicio/consulta por `coordinadorId`, no solo por rol
  (un coordinador no puede ver el distrito de otro).
- **MFA:** obligatorio para `ADMINISTRADOR` (configurado en Auth0) — NFR-S-3.
- **Datos sensibles:** presupuestos y costos no se exponen a roles que no los necesitan
  operativamente (NFR-S-4); segregación a nivel de DTO/proyección.
- **Secretos:** GCP Secret Manager (credenciales Cloud SQL, client secret de Auth0,
  API key de correo). Nada de secretos en código ni en imágenes.

### Patrones de API y comunicación

- **Estilo:** REST + JSON, versionado bajo `/api/v1`. Documentación con
  **springdoc-openapi** (OpenAPI 3, gestionado por el BOM).
- **Errores (decisión): RFC 7807 `ProblemDetail`** (nativo en Spring 6). Las
  excepciones checked de negocio (`EquivalenciaNoDefinidaException`,
  `PresupuestoExcedidoException`, `SaldoSIMInconsistenteException`, …) se mapean a
  `ProblemDetail` con `type/title/detail/instance` + código de incidente copiable.
  Esto conecta el fail-loud del backend con los **3 niveles de error de UX**
  (NFR-UX-3): el backend grita estructurado, el frontend traduce a inline/negocio/
  sistema.
- **Contract testing (decisión): Spring Cloud Contract** (productor = Spring Boot;
  stubs consumidos por el frontend). Contrato definido **antes** de implementar
  cualquiera de los dos lados (NFR-T-4).
- **Batch de forecast (decisión):** servicio Cloud Run con endpoint protegido
  disparado por **Cloud Scheduler** (semanal, alineado al ciclo lunes→miércoles) +
  **trigger manual** desde la UI de administración. `min-instances=1` en ventana
  operativa para evitar cold start. El universo (~8,600 series) corre <5 min en una
  sola JVM (NFR-E-3). Alternativa registrada: Cloud Run Jobs si se desea desacoplar
  el batch del servicio HTTP.

### Arquitectura de frontend

- **Estado (decisión): Angular Signals en servicios** + RxJS solo para flujos
  asíncronos (el recálculo optimista con `debounceTime(150)`, NFR-P-1). Aprovecha el
  modo zoneless por defecto de v21. **Sin NgRx** (sobredimensionado para 3 usuarios).
- **Componentes:** standalone (default v21); modelos `models/api/` (response backend)
  separados de `models/view/` (vista) — `project-context.md`.
- **Grid editable (recomendación): AG Grid (Community)** para la bandeja distrital
  (FR-060/064): edición en celda, navegación por teclado completa (NFR-A11Y-3),
  virtual scrolling para ~38 filas × N SKUs. Angular Material table + CDK como fallback
  sin licencia; Enterprise solo si se requieren features avanzadas (decisión de costo
  a confirmar).
- **Routing:** Angular Router con guards por rol. **Modo estricto** (`strict`,
  `strictNullChecks`) no negociable. `LOCALE_ID='es-MX'`.

### Infraestructura y despliegue

- **Backend:** Cloud Run + Docker (imagen del API y, opcionalmente, del job batch).
- **Frontend:** Firebase Hosting (SPA Angular, build esbuild de v21).
- **Base de datos:** Cloud SQL PostgreSQL.
- **Almacenamiento (decisión):** Cloud Storage como landing de ingesta
  (`inbound/`: reportes del owner — ventas semanales, ALLOC/TFS — y snapshots SIM,
  descubiertos por patrón de nombre FR-010) y como destino de exports/evidencias
  (`outbound/`: `SolicitusDeInsumosTodos.xlsx` generado + hash, evidencias FR-082).
- **CI/CD (recomendación): Cloud Build** (GCP-native): build+test (unit, jqwik,
  contract) → **PIT sobre `forecasting.*` con gate ≥80%** → build Docker → deploy
  Cloud Run (backend) + Firebase Hosting (frontend). GitHub Actions como alternativa
  si el repo vive en GitHub.
- **Config por entorno:** Spring profiles `dev`/`staging`/`prod`; configuración
  externa vía env vars + Secret Manager; Angular `environments`.
- **Observabilidad:** Spring Boot Actuator + Cloud Logging/Monitoring; MDC poblado
  (`tiendaId/skuId/cicloId/usuarioId`); dashboard de operación para admin (NFR-O-2).
- **Correo (recomendación): SendGrid** (patrón Tomaturno) para notificar al comprador
  (FR-094/110/111); alternativa SMTP corporativo. Decisión final a confirmar.

### Análisis de impacto de decisiones

**Secuencia de implementación sugerida:**

1. Inicialización del monorepo (Initializr + Angular CLI) — primera story.
2. Esquema de BD `raw`/`staging`/`mart` + Flyway baseline + entidades de dominio.
3. Capa de ingesta (parsing fail-loud, encoding/fechas duales) sobre Cloud Storage.
4. Núcleo de dominio: cruce SKU, cuantización, factor de pérdida, presupuesto —
   POJOs puros testeables (jqwik, golden, PIT).
5. `ForecastingEngine` (Smile/Commons Math) + backtesting.
6. API REST + contrato (Spring Cloud Contract) + ProblemDetail.
7. Frontend: auth Auth0, bandeja distrital (AG Grid + Signals), bitácora, export.
8. Batch (Cloud Scheduler) + observabilidad + CI/CD.

**Dependencias entre componentes:**

- El contrato (Spring Cloud Contract) precede a backend y frontend → ambos lo consumen.
- ProblemDetail (backend) ↔ 3 niveles de error UX (frontend) son dos extremos del
  mismo acuerdo fail-loud.
- La persistencia híbrida condiciona el pipeline (SQL para joins/mart) y el dominio
  (JPA para entidades) — la frontera se define en el paso de patrones (siguiente).
- Auth0 + scoping territorial condiciona toda consulta de la bandeja (filtro por
  `coordinadorId`).

## Patrones de Implementación y Reglas de Consistencia

### Puntos de conflicto identificados

~7 áreas donde agentes distintos elegirían diferente: nombrado (BD/API/código),
organización de paquetes, formato de dinero/fecha en JSON, mapeo snake_case↔camelCase,
manejo de error punta-a-punta, patrón de estado en Angular, y generación de IDs.
La mayoría de convenciones base ya están en `project-context.md` y se heredan; aquí
se cierran las que faltan.

### Patrones de nombrado

**Base de datos** (hereda `snake_case` de `project-context.md`):

- Tablas: sustantivo singular en `snake_case`, calificadas por esquema:
  `staging.tienda`, `mart.sugerencia_pedido`, `raw.alloc`.
- PK surrogate: `id` (`BIGINT GENERATED ALWAYS AS IDENTITY`). Claves naturales de
  negocio (`tienda_id`, `sku_insumo_id`) se conservan como `VARCHAR` aparte, nunca
  como PK numérica.
- FK: `<tabla_referida>_id` (`coordinador_id`, `tienda_id`).
- Índices: `idx_<tabla>_<columnas>`; únicos: `uq_<tabla>_<columnas>`.

**API REST:**

- Endpoints: sustantivo **plural** en `kebab-case`, jerárquico:
  `GET /api/v1/distritos/{distritoId}/bandeja`, `POST /api/v1/sugerencias/{id}/override`.
- Path params: `{camelCase}`. Query params y campos JSON: **camelCase**.
- Versionado en path: `/api/v1`.

**Código Java:**

- Paquete raíz: `mx.com.officedepot.insumos`. Clases `PascalCase`, métodos/vars
  `camelCase`. Identificadores en **inglés**; comentarios de negocio en **español**
  (`project-context.md`).
- Excepciones checked de negocio: `<Caso>Exception`
  (`EquivalenciaNoDefinidaException`, `PresupuestoExcedidoException`).
- Interfaces de dominio sin sufijo `I`; implementaciones con sufijo descriptivo
  (`ForecastingEngine` → `HoltWintersForecastingEngine`).

**Código Angular:**

- Archivos `kebab-case` (`bandeja-distrital.component.ts`), clases `PascalCase`.
- Servicios `XxxService`; signals expuestos como `readonly` (`tiendas`, `cargando`,
  `error`). Modelos `models/api/` (response backend) ≠ `models/view/`
  (`project-context.md`).

### Patrones de estructura

**Organización de paquetes backend (decisión — dominio puro al centro,
hexagonal-lite):**

```
mx.com.officedepot.insumos
├── dominio/         POJOs puros sin Spring (cruce, cuantización, presupuesto, pérdida)
├── forecasting/     ForecastingEngine + impl — AISLADO para PIT (≥80%)
├── ingesta/         parsers, detección encoding, descubrimiento por patrón
├── bandeja/         consolidación distrital
├── excepciones/     registro de excepciones mid-cycle
├── bitacora/        auditoría append-only
├── exportacion/     generación XLSX
├── api/             controllers REST, DTOs, mappers, @ControllerAdvice
├── persistencia/    entidades JPA, repositorios, DAOs JdbcTemplate
└── config/          configuración Spring + seguridad
```

`dominio/` y `forecasting/` **no tienen anotaciones Spring** (sin `@Service`/
`@Autowired`, `project-context.md`). Los adaptadores (`api`, `persistencia`) dependen
hacia adentro, nunca al revés.

**Tests:** Maven estándar `src/test/java` espejando paquetes; golden fixtures
inmutables en `src/test/resources/fixtures/golden/vN/` (`project-context.md`).
Frontend: `*.spec.ts` co-localizados.

### Patrones de formato

**Respuesta API:** **sin wrapper** — representación directa del recurso. Errores
**solo** vía RFC 7807 `ProblemDetail` (decidido). Nunca `{data, error}`.

**Dinero en JSON (decisión): string decimal** (`"82000.00"`) para preservar la
fidelidad de `BigDecimal` y evitar el float de JS. El cliente lo parsea solo para
**recálculo optimista** (preview); el servidor recalcula en `BigDecimal` y es
autoridad (NFR-P-1). Moneda MXN implícita; si conviven divisas → badge explícito.

**Cantidades:** **number entero** (`long`) — piezas/packs nunca con decimales.

**Fechas:** transporte ISO 8601 UTC en JSON; display `dd/MM/yyyy` en UI; semanas
ISO 8601 (`project-context.md`, NFR-L).

**Mapeo de casing:** BD `snake_case` ↔ Java `camelCase` (mapeo explícito Jackson,
**sin reflection mágica** sobre datos de negocio) ↔ JSON `camelCase`.

**Status/códigos pendientes de glosario** (ALLOC.STATUS, TFS.STATUS, TRAN_CODE):
se conservan como `String` crudo; cualquier ramificación pasa por
`interpretarStatus(...)` central → `StatusDesconocidoException` si no se reconoce
(`project-context.md`). `""` vacío legítimo ≠ `null` — no colapsar.

### Patrones de comunicación

**Eventos de bitácora:** entradas append-only con `tipo_evento` (enum String en
pasado: `OVERRIDE_APLICADO`, `RECORTE_ACEPTADO`, `EXCEPCION_REGISTRADA`,
`EXPORT_GENERADO`, `FACTOR_PERDIDA_MODIFICADO`). Payload uniforme:
`{actor, timestamp, valorOriginal, valorNuevo, razonEstructurada, entradaPreviaId}`.

**Estado Angular (decisión):** actualizaciones **inmutables** sobre signals; un
servicio por feature expone signals `readonly` + métodos que reemplazan estado
(nunca mutación in-place). RxJS solo para asincronía/`debounceTime(150)`.

**Logging:** SLF4J con MDC poblado (`tiendaId/skuId/cicloId/usuarioId`,
`project-context.md`). `INFO` flujo de negocio, `WARN` invariante violada no fatal,
`ERROR` excepción de negocio que aborta ciclo. Sin `System.out`.

### Patrones de proceso

**Manejo de error punta-a-punta:** excepción checked de dominio →
`@ControllerAdvice` → `ProblemDetail` (con código de incidente) → interceptor HTTP
Angular → traducción a los **3 niveles de UX** (inline/negocio/sistema, NFR-UX-3).
El backend grita estructurado; el frontend traduce a es-MX claro (NFR-UX-4).

**Estados de carga:** patrón `{cargando, error, datos}` por feature; autosave
silencioso con indicador "Guardado hace 3 seg" (NFR-P-2); acciones reversibles con
toast "Deshacer" 8s, **no** `confirm()` modal (NFR-UX-2).

**Validación:** backend autoritativo (Bean Validation en DTO + invariantes de dominio
en POJOs); frontend optimista que el servidor confirma. Invariantes de cuantización/
presupuesto verificadas con jqwik (`project-context.md`).

**IDs:** surrogate `BIGINT IDENTITY` para entidades internas (bitácora, excepción,
sugerencia); claves naturales `String` para tienda/SKU. Sin UUID salvo necesidad de
generación cliente-side (no aplica en v1).

**Reintentos:** ingesta y correo con backoff exponencial acotado; fail-loud si se
agota (notificar admin, NFR-O-3). Nunca reintento silencioso infinito.

### Lineamientos de cumplimiento

**Todo agente DEBE:**

- Mantener `dominio/` y `forecasting/` libres de Spring; lógica de negocio en POJOs.
- Usar `BigDecimal` (scale=4) para dinero y transportarlo como string en JSON.
- Lanzar la excepción checked específica ante dato faltante/ambiguo — nunca degradar,
  nunca interpolar, nunca default 0 (fail-loud).
- Mapear CSV/JSON explícitamente (sin reflection mágica); preservar literal
  `"Matarial y tamaño"`.
- Poblar MDC en toda operación de negocio.

**Anti-patrones (prohibidos):**

- `double`/`float` para dinero; `BigDecimal` como número JSON.
- `@Service`/`@Autowired` en lógica de dominio core.
- Wrapper de respuesta `{data,...}`; stacktrace al usuario.
- Colapsar `""` a `null`; asumir encoding o formato de fecha sin detección.
- `.subscribe()` sin teardown; `any` en TS fuera de boundaries; `outline:none`.

## Estructura del Proyecto y Fronteras

### Árbol de directorios completo (monorepo)

El código se agrega al repo existente como dos subproyectos (`backend/`, `frontend/`)
junto a `docs/`, `_producto/`, `_bmad/` ya presentes.

```
insumos-odemas/                      # raíz del repo existente
├── README.md
├── cloudbuild.yaml                  # CI/CD GCP-native
├── .gitignore                       # + ~$*.xlsx (lock files de Excel)
├── docs/                            # (existente) insumos de datos CSV/XLSX
├── _producto/                       # (existente) artefactos BMad
│
├── contracts/                       # Spring Cloud Contract (compartido FE/BE)
│   └── bandeja/, sugerencias/, excepciones/   # *.groovy/*.yml por endpoint
│
├── backend/
│   ├── pom.xml                      # Maven, Spring Boot 3.5.x, Java 21
│   ├── Dockerfile                   # imagen Cloud Run
│   ├── src/main/java/mx/com/officedepot/insumos/
│   │   ├── InsumosOdemasApplication.java
│   │   ├── config/                  # SecurityConfig(Auth0), CacheConfig, JacksonConfig, OpenApiConfig
│   │   ├── api/                     # ADAPTADOR entrante
│   │   │   ├── controller/          # BandejaController, SugerenciaController, ExcepcionController,
│   │   │   │                        #   DatosMaestrosController, FactorPerdidaController,
│   │   │   │                        #   DashboardValidacionController, BitacoraController,
│   │   │   │                        #   ExportController, ForecastBatchController
│   │   │   ├── dto/                 # request/response (camelCase, dinero como string)
│   │   │   ├── mapper/              # dominio ↔ DTO (explícito, sin reflection)
│   │   │   └── error/               # GlobalExceptionHandler @ControllerAdvice → ProblemDetail
│   │   ├── dominio/                 # NÚCLEO PURO — sin Spring
│   │   │   ├── modelo/              # Tienda, SkuVenta, SkuInsumo, Ciclo, TipoPapel, EventoComercial (POJOs/records)
│   │   │   ├── cruce/               # equivalencia venta↔insumo (autoridad)
│   │   │   ├── cuantizacion/        # Cuantizador (múltiplo empaque mínimo)
│   │   │   ├── presupuesto/         # ValidadorPresupuesto + recorte por riesgo de quiebre
│   │   │   ├── perdida/             # FactorPerdida (resolución jerárquica tipo_papel→default)
│   │   │   └── error/               # excepciones checked: EquivalenciaNoDefinidaException, etc.
│   │   ├── forecasting/             # AISLADO para PIT ≥80%
│   │   │   ├── ForecastingEngine.java          # interfaz
│   │   │   ├── HoltWintersForecastingEngine.java
│   │   │   ├── estacionalidad/      # componente estacional + eventos comerciales (FR-033)
│   │   │   └── backtesting/         # soporte BacktestingSuite
│   │   ├── ingesta/                 # ADAPTADOR de entrada de archivos
│   │   │   ├── descubrimiento/      # patrón de nombre (ALLOC_*, TFS_*, snapshot SIM, reportes owner)
│   │   │   ├── encoding/            # detección UTF-8 BOM → cp1252 (log MDC)
│   │   │   ├── parser/              # un parser por archivo (fechas duales, "$X,XXX", List<Integer> semanas)
│   │   │   └── validacion/          # invariantes (QTY_ALLOCATED=…, TRAN_CODE↔DECODE)
│   │   ├── bandeja/                 # consolidación distrital, estado por tienda, agregados
│   │   ├── excepciones/             # registro mid-cycle + escalamiento a comprador
│   │   ├── bitacora/                # auditoría append-only (servicio + tipo_evento)
│   │   ├── exportacion/             # generación XLSX compatible (Apache POI)
│   │   └── persistencia/            # ADAPTADOR saliente
│   │       ├── jpa/                 # @Entity + repositories (dominio/CRUD, esquemas staging/mart)
│   │       └── jdbc/                # DAOs JdbcTemplate (ingesta raw, joins pipeline, agregados mart)
│   ├── src/main/resources/
│   │   ├── application.yml          # + application-{dev,staging,prod}.yml
│   │   └── db/migration/            # Flyway: V1__raw_schema.sql, V2__staging.sql, V3__mart.sql, V4__bitacora.sql…
│   └── src/test/
│       ├── java/.../               # unit (espejo) + jqwik (cuantización/presupuesto) + BacktestingSuite.java
│       └── resources/fixtures/golden/v1/   # EXPECTED_OUTPUT.csv (firmado por Hugo) + DECISION_LOG.md
│
└── frontend/
    ├── package.json                 # Angular 21.2.x
    ├── angular.json
    ├── tsconfig.json                # strict + strictNullChecks
    ├── firebase.json                # Firebase Hosting
    └── src/
        ├── main.ts
        ├── styles.scss
        ├── environments/            # environment.{ts,prod.ts}
        └── app/
            ├── app.config.ts        # LOCALE_ID='es-MX', registerLocaleData, Auth0, interceptor
            ├── app.routes.ts        # rutas + guards por rol
            ├── core/                # auth (Auth0), http-error.interceptor (ProblemDetail→3 niveles), guards, autosave
            ├── shared/              # UI reusable, pipes es-MX (moneda MXN, cantidad+equivalencia, %), a11y, toast Deshacer
            ├── models/
            │   ├── api/             # contratos de response del backend
            │   └── view/            # modelos de vista del componente
            └── features/
                ├── bandeja-distrital/      # AG Grid editable + signals (C-7)
                ├── dashboard-validacion/   # pedido-vs-recibido (FR-067)
                ├── excepciones/            # bandeja territorial mid-cycle (C-10)
                ├── factor-perdida/         # config por tipo de papel + candidatos residuales (C-9)
                ├── tfs-historico/          # visualización cruzada (C-8)
                ├── bitacora/               # cadena de cambios + "Cambios de hoy" (C-11)
                └── administracion/         # datos maestros, eventos, umbrales, buffer (rol admin)
```

### Fronteras arquitectónicas

**Frontera de API:** `/api/v1/**`. Auth0 JWT validado en el filtro de Spring Security
(edge); el **scoping territorial** (filtro por `coordinadorId`) vive en la capa de
servicio, no en el controller. Documentación OpenAPI en `/swagger-ui`.

**Frontera de dominio (regla dura):** `dominio/` y `forecasting/` son el núcleo puro
— sin Spring, sin JPA, sin IO. `api/`, `ingesta/` y `persistencia/` son adaptadores
que dependen **hacia adentro**; el núcleo no conoce a los adaptadores. Esta frontera
es lo que hace testeable el pipeline en aislamiento (NFR-T-1) y mutable con PIT.

**Frontera de datos:** esquemas `raw` (ingesta cruda, vía `jdbc/`) → `staging`
(validado, JPA) → `mart` (sugerencias/agregados de presentación, JPA lectura +
`jdbc/` para agregación). Ninguna capa superior escribe en `raw`.

**Frontera de componentes (frontend):** cada `feature/` es autónoma; se comunican solo
vía servicios de `core/` que exponen signals `readonly`. Prohibido importar de feature
a feature directamente.

### Mapeo de requisitos a estructura

| Área de capacidad (FR) | Backend | Frontend |
|---|---|---|
| C-1 Datos maestros (FR-001..007) | `dominio/modelo` + `persistencia/jpa` + `api/controller/DatosMaestrosController` | `features/administracion` |
| C-2 Ingesta (FR-010..017) | `ingesta/**` + `persistencia/jdbc` (raw) | — |
| C-4 Forecast (FR-030..035) | `forecasting/**` + `dominio/{cruce,cuantizacion,perdida}` | trazo explicable en `bandeja-distrital` (hover) |
| C-5 Sugerencia pedido (FR-040..043) | `bandeja` + `dominio` | `bandeja-distrital` |
| C-6 Presupuesto/recorte (FR-050..054) | `dominio/presupuesto` | `bandeja-distrital` (desglose techo−espor.) |
| C-7 Bandeja distrital (FR-060..067) | `bandeja` + `api/BandejaController` | `bandeja-distrital`, `dashboard-validacion` |
| C-8 Histórico TFS (FR-073..075) | `ingesta` (TFS) + `mart` | `tfs-historico` |
| C-9 Factor pérdida + residual (FR-080..085) | `dominio/perdida` + detección residual | `factor-perdida` |
| C-10 Excepciones (FR-091..095) | `excepciones/**` | `excepciones` |
| C-11 Trazabilidad (FR-100..104) | `bitacora/**` (append-only) | `bitacora` |
| C-12 Export (FR-110..112) | `exportacion/**` (POI) | acción en `bandeja-distrital` |

**Concerns transversales:**
- **Auth/seguridad:** `backend/config/SecurityConfig` + `frontend/core/auth` (Auth0).
- **Errores:** `backend/api/error` (ProblemDetail) ↔ `frontend/core/http-error.interceptor`.
- **Locale es-MX:** `backend/dominio` (parsing) + `frontend/shared` (pipes) + `app.config`.
- **Auditoría:** `backend/bitacora` + MDC en logs + `frontend/features/bitacora`.

### Puntos de integración

**Comunicación interna:** Angular (Firebase Hosting) → REST `/api/v1` (Cloud Run) →
Cloud SQL. Frontend consume el contrato de Spring Cloud Contract; ambos lados lo
comparten desde `contracts/`.

**Integraciones externas:**
- **Auth0** — OIDC/JWT (login + MFA admin).
- **Cloud Storage** — landing `inbound/` (reportes del owner, snapshot SIM) y destino
  `outbound/` (export XLSX + hash, evidencias). SIM y reportes del owner entran como
  **archivos**, no APIs en vivo.
- **SendGrid** — notificación al comprador (export) y escalamientos.
- **Cloud SQL** — PostgreSQL.

**Flujo de datos (end-to-end):**
```
Cloud Storage inbound → ingesta(raw) → validación(staging) →
dominio(cruce+cuantización+pérdida) + forecasting → presupuesto →
mart(sugerencias) → API → bandeja Angular → override/aprobación →
exportación(XLSX) → Cloud Storage outbound + SendGrid → comprador
```

### Integración con el flujo de desarrollo

- **Dev:** backend `mvn spring-boot:run` (perfil `dev`, Cloud SQL Auth Proxy local o
  Testcontainers); frontend `ng serve`. Contratos generan stubs para desarrollo
  desacoplado.
- **Build:** `cloudbuild.yaml` → `mvn verify` (unit + jqwik + contract) → PIT
  `forecasting.*` (gate ≥80%) → `docker build` backend; `ng build` frontend.
- **Deploy:** backend → Cloud Run (Docker); frontend → Firebase Hosting; migraciones
  Flyway al arranque del backend; batch disparado por Cloud Scheduler.

## Resultados de Validación de Arquitectura

### Validación de coherencia ✅

**Compatibilidad de decisiones:** stack mutuamente compatible y verificado en la web
(may-2026): Spring Boot 3.5.x ↔ Java 21 ↔ Spring Cloud 2025.0 (Spring Cloud Contract);
Angular 21.2.x ↔ AG Grid 35.3.0 (soporta el LTS de Angular); Auth0 ↔ Spring Security
OAuth2 Resource Server; Flyway gestionado por el BOM de Boot 3.5. Sin decisiones
contradictorias.

**Consistencia de patrones:** el nombrado, fail-loud, bitácora append-only y el núcleo
de dominio puro (hexagonal-lite) se refuerzan entre sí; ningún patrón pelea con una
decisión de stack.

**Alineación de estructura:** el monorepo soporta los dos targets de despliegue GCP
(Cloud Run + Firebase Hosting); el aislamiento de `dominio/` y `forecasting/` habilita
PIT (≥80%) y jqwik; los esquemas raw/staging/mart soportan la persistencia híbrida.

### Validación de cobertura de requisitos ✅

**Capacidades (FR):** las 12 áreas activas (C-1, C-2, C-4..C-12) están mapeadas a
ubicaciones concretas (tabla de mapeo). C-3 (conteo asistido) está fuera de scope por
decisión del owner — no es gap. ~50 FRs activos cubiertos.

**NFR:** Escala (batch <5 min en una JVM + caché por ciclo), Performance (signals +
`debounceTime(150)`), Auditoría (append-only + MDC), Calidad del dato (fail-loud +
detección encoding + SLA SIM 36h), Seguridad (Auth0 + MFA + scoping territorial +
segregación de datos sensibles), Testing (golden/jqwik/contract/backtesting/PIT
soportados por la estructura), Locale/A11y/UX (pipes es-MX + AG Grid teclado +
ProblemDetail→3 niveles), Observabilidad (Actuator + Cloud Logging + dashboard admin).

### Validación de preparación para implementación ✅

**Completitud de decisiones:** decisiones críticas documentadas con versiones (como
rangos, per `project-context.md`); auth, persistencia, migraciones, batch y contrato
de error resueltos.

**Completitud de estructura:** árbol completo, fronteras explícitas, puntos de
integración y flujo de datos end-to-end definidos.

**Completitud de patrones:** ~7 puntos de conflicto cerrados con ejemplos y
anti-patrones; convenciones de nombrado, comunicación y proceso especificadas.

### Análisis de brechas

**Críticas (bloquean implementación):** ninguna. La secuencia de fundación
(init → esquema → ingesta → dominio → motor) es ejecutable con lo documentado.

**Importantes (resolver antes de su story dependiente, no de la aprobación):**
- **Formato y cadencia de los reportes del owner** (ventas semanales, ALLOC/TFS) —
  bloquea la story de ingesta de esos archivos (FR-016/017). Dueño: PM + owner.
- **Formato y dueño del extract SIM (OQ-117 parcial)** — bloquea la ingesta del
  snapshot SIM (FR-014). Dueño: arquitectura + IT.
- **Cold-start de tiendas <12 meses (OQ-108)** — fallback documentado ("sin sugerencia"
  fail-loud); el tratamiento fino (histórico distrital / bootstrap) queda diferido.
- **OQ-116** (publicar ALLOC esperado a SIM) — afecta el bucle de coexistencia; diferido.
- **Spike de forecasting Java (R-8)** y **backtesting + baselines (R-9)** —
  precondiciones de pilotaje (no de la arquitectura).

**Menores:**
- Licenciamiento AG Grid Community vs Enterprise (decisión de costo).
- SendGrid vs SMTP corporativo (confirmar).

### Issues de validación atendidos

- **F-05 resuelto (implementación del factor de pérdida):** se aplica como
  **multiplicador sobre la demanda de venta predicha, ANTES de la conversión a insumo
  (FR-031) y la cuantización (FR-032)** — `demanda_ajustada = V × (1 + f)`. Observable
  e independiente del método interno del modelo, consistente con el criterio de
  aceptación de FR-035. Se descarta el ajuste sobre el histórico para v1.
- **OQ-114 (SIM sucio):** política propuesta en "Arquitectura de datos" —
  `SaldoSIMInconsistenteException` que aborta el ciclo de esa tienda; el distrito
  continúa; snapshot >36h rechaza operar y notifica admin.
- **Compatibilidad Spring Cloud Contract:** confirmada via release train 2025.0.

### Checklist de completitud de la arquitectura

**Análisis de requisitos**
- [x] Contexto del proyecto analizado a fondo
- [x] Escala y complejidad evaluadas
- [x] Restricciones técnicas identificadas
- [x] Concerns transversales mapeados

**Decisiones arquitectónicas**
- [x] Decisiones críticas documentadas con versiones
- [x] Stack tecnológico completamente especificado
- [x] Patrones de integración definidos
- [x] Consideraciones de performance atendidas

**Patrones de implementación**
- [x] Convenciones de nombrado establecidas
- [x] Patrones de estructura definidos
- [x] Patrones de comunicación especificados
- [x] Patrones de proceso documentados

**Estructura del proyecto**
- [x] Estructura de directorios completa definida
- [x] Fronteras de componentes establecidas
- [x] Puntos de integración mapeados
- [x] Mapeo requisitos→estructura completo

### Evaluación de preparación

**Estado general:** READY FOR IMPLEMENTATION — los 16 ítems del checklist están en
`[x]` y no hay brechas críticas. Las brechas *Important* son dependencias externas o
decisiones diferidas que el PRD ya clasificó como precondiciones de pilotaje; gatean
stories específicas (ingesta, forecast), no la fundación ni la aprobación del documento.

**Nivel de confianza:** alto para la fundación (init/esquema/dominio/motor); medio para
las stories de ingesta y forecast hasta cerrar formatos de reporte y el spike (R-8).

**Fortalezas clave:**
- Núcleo de dominio puro y aislado → testeable (jqwik/golden/PIT) e independiente del
  framework, alineado con la disciplina de `project-context.md`.
- Fail-loud coherente punta-a-punta (excepción checked → ProblemDetail → 3 niveles UX).
- Escala deliberadamente no sobre-ingenierizada (3 usuarios, ~8,600 series).
- Sin dependencia de BigQuery/RMS en v1 (ingesta por archivos).

**Áreas de mejora futura:**
- Cold-start formal (OQ-108) y posible microservicio Python si el spike rompe SLA.
- Integración R/W con SIM (Fase 2/3), medición precisa de merma (Fase 3).
- BI/DWH y app móvil (out-of-scope v1).

### Handoff a implementación

**Guía para agentes de IA:**
- Seguir las decisiones arquitectónicas exactamente como están documentadas.
- Usar los patrones de implementación de forma consistente.
- Respetar las fronteras (dominio puro; adaptadores hacia adentro).
- Consultar este documento ante cualquier duda arquitectónica.

**Primera prioridad de implementación:** inicialización del monorepo (story de init) con
los comandos de Spring Initializr + Angular CLI documentados en "Evaluación de Starter
Templates" — confirmando el pin exacto de versiones contra la imagen base corporativa.
