---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md'
  - '_producto/planning-artifacts/architecture.md'
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/addendum.md'
  - '_producto/project-context.md'
  - '_producto/planning-artifacts/ux-design-specification.md'
totalEpics: 5
totalStories: 51
lastUpdated: '2026-05-28'
---

# insumos-odemas - Epic Breakdown

## Overview

Este documento provee el desglose completo de épicas e historias para **insumos-odemas**, descomponiendo los requisitos del PRD, la Arquitectura y las convenciones de `project-context.md` en historias implementables. (No existe documento UX independiente; los requisitos UX están embebidos en el PRD §9.7–9.9 y `project-context.md`.)

## Requirements Inventory

### Functional Requirements

> Convención: solo se listan FRs **activos**. Los FRs eliminados en el PRD (`~~FR-004~~`, `~~FR-015~~`, `~~FR-042~~`, `~~FR-062~~`, `~~FR-070/071/072~~`, `~~FR-090~~`) se omiten deliberadamente. Las áreas C-3 (conteo asistido) está fuera de scope v1.

**C-1 — Datos maestros**

- **FR-001**: Mantener catálogo de **tiendas** con ID (String), nombre, distrito, coordinador territorial asignado, formato (Express/Estándar), ciclo quincenal (semana 1-y-3 o 2-y-4) y techo presupuestal semanal MXN. El factor de pérdida NO vive en la tienda.
- **FR-002**: Mantener catálogo de **SKUs de insumo** (comprable al proveedor): código, descripción, presentación, contenido (piezas/presentación), costo MXN. Preservar literal `"Matarial y tamaño"` por contrato con el equipo de datos.
- **FR-003**: Mantener **tabla de equivalencia** SKU venta ↔ SKU insumo. Toda venta procesada debe encontrar equivalencia; sin equivalencia → `EquivalenciaNoDefinidaException` y aborto del ciclo de esa tienda (fail-loud).
- **FR-005**: Mantener **datos de coordinadores**: identidad, distrito territorial asignado (lista de tiendas), credenciales, rol. Sin calendario de guardia rotativa.
- **FR-006**: Mantener **catálogo de eventos comerciales** con ventanas de fechas y factor multiplicador estimado por evento (Hot Sale, BTS, Buen Fin, Navidad/Reyes, regreso a oficinas, Día de las Madres/Padre/Niño, cierre fiscal). Alimenta FR-033.
- **FR-007**: Mantener **catálogo de tipos de papel** (`tipo_papel`) derivado del atributo `Material` de `Skus_insumos.csv`. Agrupa uno o más SKUs de insumo; es la unidad de configuración del único `factor_perdida`.

**C-2 — Ingesta y normalización**

- **FR-010**: Ingestar archivos canónicos en `docs/` por **descubrimiento por patrón de nombre** (`ALLOC_YYYY_MM_DD.csv`, `TFS_YYYY_MM_DD.csv`, `Entrega-directa-tienda*.csv`). Nuevos archivos sin reconfiguración.
- **FR-011**: Detectar **encoding** por archivo en orden UTF-8 BOM → cp1252; registrar el detectado en MDC.
- **FR-012**: Validar invariantes de ingesta (`QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE`, `TRAN_CODE ↔ DECODE`); comportamiento fail-loud cuando aplica.
- **FR-013**: Parsear `Cantidad por semana` (`"$X,XXX"`) y `Semana que debe solicitar` (lenguaje natural → `List<Integer>` ISO). Patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`.
- **FR-014**: Ingestar **snapshots periódicos de SIM** (saldo por tienda × SKU) como input externo unidireccional; cadencia diaria nocturna. Política fail-loud para snapshot ausente/negativo/inconsistente (OQ-114).
- **FR-016**: Ingestar **ventas con granularidad semanal** provistas como reporte/archivo aportado por el owner (CSV/XLSX, mismo patrón de ingesta). Sin integración a BigQuery en v1. Fallback de prorrateo mensual→semanal con advertencia si un ciclo carece de corte semanal.
- **FR-017**: Ingestar **ALLOC y TFS** (lo pedido, en curso, por llegar, entregado) como input del cálculo del pedido (FR-040) y del dashboard pedido-vs-recibido (FR-067). Reporte aportado por el owner; sin integración RMS/BigQuery en v1.

**C-4 — Forecast de demanda (cadencia dual)**

- **FR-030**: Calcular **demanda esperada de venta semanalmente** por tienda × SKU venta para el ciclo activo, con modelo estacional (Java puro). Unidad temporal de salida = ciclo quincenal de la tienda. Tiendas con <12 meses de histórico (cold-start, OQ-108) se marcan **"sin sugerencia del motor"** (fail-loud, no se inventa número).
- **FR-031**: Convertir demanda de venta a **demanda de insumo** aplicando la tabla de equivalencia.
- **FR-032**: **Cuantizar** la demanda de insumo al múltiplo del empaque mínimo (`Contenido`). Nunca redondear hacia abajo si hay demanda.
- **FR-033**: Aplicar **componente estacional** basado en histórico ≥12 meses + catálogo de eventos comerciales (FR-006), cada evento con ventana de fechas y factor multiplicador.
- **FR-034**: Generar **trazo explicable** por SKU: histórico usado, ventana, factor estacional + evento activo, factor único de pérdida aplicado (% de venta) con nivel de origen, ajuste por ALLOC/TFS en tránsito. Visible en hover.
- **FR-035**: Aplicar **un único `factor_perdida`** (% de la demanda de venta) resuelto por jerarquía `factor_por_tipo_papel → default_global (≈0.9%)`. Criterio de aceptación: `demanda_ajustada = V × (1 + f)` sobre la demanda de venta, **antes** de conversión (FR-031) y cuantización (FR-032); el trazo declara V, f y nivel de origen.

**C-5 — Sugerencia de pedido**

- **FR-040**: Calcular **cantidad sugerida por SKU** = `demanda_quincenal_insumo + buffer_seguridad − saldo_SIM_snapshot − ALLOC_pendiente − TFS_entrante`, cuantizada al empaque mínimo. `buffer_seguridad` configurable por administración (inicial ≈ 1 semana de cobertura). El factor de pérdida ya está aplicado en la demanda.
- **FR-041**: Mostrar al **coordinador en su bandeja** la sugerencia por tienda × SKU con su trazo explicable.
- **FR-043**: Para SKUs sin equivalencia definida, **no incluir** en sugerencia y emitir `EquivalenciaNoDefinidaException` que aborta el ciclo de esa tienda hasta resolverse.

**C-6 — Validación de presupuesto y recorte**

- **FR-050**: Calcular **costo total por tienda** = `Σ (cantidad_sugerida × costo_SKU)` en MXN.
- **FR-051**: Si `costo_total > presupuesto_disponible`, **sugerir recorte automático** priorizando reducir SKUs con menor riesgo de quiebre (cobertura post-recorte ≥ 2 semanas; recortar primero los de cobertura más alta).
- **FR-052**: Mostrar las tiendas que requieren recorte con la sugerencia aplicada y permitir **aceptar / modificar / rechazar** con razón.
- **FR-053**: Persistir cada decisión de recorte con timestamp, coordinador ejecutor y razón.
- **FR-054**: **Descontar compras esporádicas del periodo** del techo: `presupuesto_disponible = techo_semanal − Σ compras_esporadicas_del_periodo` (leyendo `Entrega-directa-tienda.csv`, TRAN_CODE 20 "Purchases"). Mostrar el desglose `techo − consumido → disponible`.

**C-7 — Concentrado distrital (bandeja semanal por coordinador)**

- **FR-060**: Mostrar a cada coordinador la **bandeja de tiendas de su distrito con ciclo activo esa semana** (~38 tiendas).
- **FR-061**: Por cada tienda, mostrar estado (solicitud recibida / pendiente solicitud — visible pasivo / carry-over de semana anterior — visible pasivo / sugerencia lista / con recorte / aprobada / exportada). El coordinador **trabaja en pantalla sobre la sugerencia del motor**; el Excel del jefe es contexto, el Excel del sistema es output (FR-110). v1 no parsea el adjunto del jefe.
- **FR-063**: Mostrar **agregados del distrito**: presupuesto consumido total (con desglose `techo − esporádicas`), tiendas con solicitud pendiente, tiendas en carry-over, SKUs con riesgo concentrado, costo estimado del consolidado.
- **FR-064**: Permitir **modificar cantidades** (override) en cualquier tienda con razón estructurada obligatoria (lista predefinida + texto libre opcional); modificaciones en bitácora. En ciclos 1-3 los overrides son ORO de calibración.
- **FR-065**: Permitir **aprobar el consolidado** del distrito y dispararlo al export downstream (C-12).
- **FR-066**: **Surfacing pasivo de pedidos tardíos (sin regla de arrastre).** Tienda que no envió antes del corte se muestra como `carry-over`/`sin pedido esta semana` (visible pasivo). v1 NO reinscribe automáticamente ni recalcula/traslada presupuesto; el faltante se autocorrige vía forecast sales-driven + posición de inventario.
- **FR-067**: **Dashboard de validación pedido-vs-recibido.** Por proveedor → tienda → SKU: necesidad sugerida, últimos pedidos (asignado/pedido) y recibos reales, con la brecha resaltada. Se alimenta de ALLOC/TFS (`QTY_ALLOCATED` vs `QTY_RECEIVED`).

**C-8 — Histórico y análisis de transferencias TFS** *(v1 NO sugiere ni decide TFS)*

- **FR-073**: **Registrar histórico de TFS ejecutadas** ingestando `TFS_*.csv` semanalmente. Doble función: (a) visibilidad al coordinador, (b) input al motor de forecast.
- **FR-074**: **Visualización cruzada del histórico TFS**: filtros por tienda origen, tienda destino, SKU, periodo. Tablero exportable a CSV.
- **FR-075**: Cuando el coordinador gestiona una TFS de emergencia fuera del sistema, permitir **registrar manualmente la decisión y el resultado** para bitácora; el saldo del sistema no se modifica directamente desde aquí.

**C-9 — Factor de pérdida y detección estadística residual**

- **FR-080**: Capa **opcional y secundaria** de detección de patrones sospechosos por tienda × SKU: `residual = consumo_real − venta_esperada − perdida_aplicada`; si excede banda (≥ 2σ inicial, OQ-123) generar candidato para revisión humana.
- **FR-081**: Generar **candidatos residuales** clasificados por severidad para surfacing en la bandeja del coordinador territorial. Sin atribución entre categorías (factor único).
- **FR-082**: Permitir al coordinador territorial: (1) **configurar el `factor_perdida` (% de venta) por tipo de papel** (gobierna el default que alimenta C-4 vía FR-035); (2) capturar/modificar/descartar candidatos residuales (SKU, cantidad, motivo, evidencia opcional); (3) todo trazado en bitácora (C-11).
- **FR-083**: Acumular registro de candidatos confirmados para revisión periódica con dirección. Exportable.
- **FR-084**: **Default global ≈ 0.9% de la venta** para tipos de papel no configurados. El coordinador ajusta por tipo de papel desde el aplicativo a discreción.
- **FR-085**: El **factor único de pérdida alimenta el motor de forecast** (C-4 vía FR-035) como ajuste porcentual, resuelto por jerarquía `factor_por_tipo_papel → default_global`. Cambios trazados en bitácora (timestamp + coordinador + tipo de papel + valor anterior/nuevo + razón).

**C-10 — Registro de excepción mid-cycle por el coordinador**

- **FR-091**: El **coordinador territorial registra** una excepción mid-cycle: tienda, SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado, categoría de disparador (temporalidad/uso interno/otra), **origen del aviso = correo electrónico (campo fijo)**. Se asigna automáticamente a su territorio.
- **FR-092**: Mostrar al coordinador **contexto agregado**: saldo SIM más reciente, ALLOC pendiente, TFS en tránsito y histórico TFS reciente para el SKU. Sin sugerencia automática de TFS.
- **FR-093**: Permitir al coordinador **decidir y registrar** la respuesta: TFS de emergencia fuera del sistema (solo decisión/resultado), escalamiento a comprador para compra directa, rechazo con razón, espera al ciclo normal.
- **FR-094**: Al escalar a comprador, **notificarlo** por canal acordado con contexto + decisión del coordinador + datos de tienda y SKU.
- **FR-095**: Reflejar en el saldo de planeación cualquier **compra extraordinaria autorizada** y descontarla del techo presupuestal de la tienda para el ciclo regular siguiente (vía FR-054).

**C-11 — Trazabilidad y auditoría**

- **FR-100**: Para cada cantidad modificada (override, recorte, TFS aceptada/rechazada, excepción atendida), **registrar entrada en bitácora**: timestamp, actor, valor original, valor nuevo, razón estructurada.
- **FR-101**: Mostrar en UI (hover sobre fila modificada) la **cadena completa** de cambios.
- **FR-102**: Panel lateral **"Cambios de hoy"** exportable a CSV.
- **FR-103**: MDC en logs con `tiendaId`, `skuId`, `cicloId`, `usuarioId`.
- **FR-104**: Bitácora **inmutable**: para corregir un error se crea nueva entrada que referencia la anterior.

**C-12 — Export downstream al comprador**

- **FR-110**: Generar archivo consolidado **compatible con la estructura validada de `docs/SolicitusDeInsumosTodos.xlsx`** (contrato literal del export), como output hacia compras. Vista de trabajo del coordinador organizada **por SKU → tiendas + cantidades** (Opción A, D-11).
- **FR-111**: Permitir **exportar bajo demanda** y **enviar por correo** al comprador con CC configurable.
- **FR-112**: Registrar cada export en bitácora: timestamp, coordinador ejecutor, hash del archivo, destinatarios.

### NonFunctional Requirements

**Escala y concurrencia**
- **NFR-E-1**: Soportar 226 tiendas en el modelo de datos; usuarios humanos concurrentes hasta 3 coordinadores + 1 administrador.
- **NFR-E-3**: El motor de forecast batch procesa el universo completo (~8,600 series temporales) en <5 min en una sola JVM, dentro de la ventana operativa del miércoles.
- **NFR-E-4**: Cada coordinador abre su bandeja distrital (~38 tiendas) y la ve cargada en <3 s.

**Performance percibida**
- **NFR-P-1**: Edición numérica dispara recálculo optimista en cliente (RxJS `debounceTime(150)`) con confirmación servidor; recalcular fila, acumulado por proveedor y restante de presupuesto en <100 ms.
- **NFR-P-2**: Autosave silencioso con indicador tipo Google Docs ("Guardado hace 3 seg"); el usuario nunca pierde trabajo.

**Disponibilidad**
- **NFR-D-1**: Disponibilidad ≥99% durante ventanas operativas (L–V 06:00–22:00 CDMX).
- **NFR-D-2**: Tras una caída, las decisiones del coordinador (overrides, recortes, excepciones, configuraciones) se reanudan sin pérdida; el estado persiste server-side.

**Auditabilidad**
- **NFR-A-1**: Toda decisión de negocio queda trazada (timestamp, actor, valor original, valor nuevo, razón estructurada).
- **NFR-A-2**: Logs server-side con MDC poblado (`tiendaId/skuId/cicloId/usuarioId`).
- **NFR-A-3**: Bitácora inmutable; correcciones como entradas nuevas que referencian la anterior.

**Seguridad y control de acceso**
- **NFR-S-1**: Autenticación con identidad individual (no compartida); patrón Auth0 + JWT.
- **NFR-S-2**: Roles v1: Coordinador (sus tiendas territoriales) y Administrador (datos maestros, eventos, umbrales, razones, `buffer_seguridad`). Override aplica también al Administrador en modo coordinador.
- **NFR-S-3**: MFA obligatorio para roles administrativos.
- **NFR-S-4**: Datos sensibles (presupuestos, costos) no se exponen a roles que no los necesitan.

**Calidad del dato**
- **NFR-DAT-1**: Fail-loud ante datos faltantes/ambiguos (equivalencia, presupuesto, encoding, periodicidad, snapshot SIM ausente/inconsistente) → excepciones checked que abortan el ciclo de esa tienda.
- **NFR-DAT-3**: Encoding detectado y registrado en logs por archivo procesado, incluido el snapshot SIM.
- **NFR-DAT-4**: SLA de frescura del snapshot SIM = 36 h máximo de antigüedad; si se excede, rechazar operar y notificar al administrador.

**Locale y representación**
- **NFR-L-1**: `LOCALE_ID = 'es-MX'`, `registerLocaleData(localeEsMX)` en Angular.
- **NFR-L-2**: Fechas mostradas `dd/MM/yyyy`; transporte JSON ISO 8601.
- **NFR-L-3**: Semanas ISO 8601 (lunes a domingo), con contexto (`"Sem 12 · 16–22 mar 2026"`).
- **NFR-L-4**: Moneda MXN con separador de miles y dos decimales; totales grandes abreviados con tooltip.
- **NFR-L-5**: Cantidades discretas enteras con separador de miles y unidad pegada; mostrar siempre equivalencia (`18 piezas = 3 cajas`).
- **NFR-L-6**: Plazos en días hábiles, no naturales.

**Accesibilidad**
- **NFR-A11Y-1**: Contraste 4.5:1 en texto, 3:1 en componentes interactivos.
- **NFR-A11Y-2**: Toda celda editable con `aria-label` contextualizado.
- **NFR-A11Y-3**: Navegación por teclado completa en grillas (Tab/Shift+Tab/Enter/Esc); `:focus-visible` 2px; `outline:none` prohibido.

**UX no negociable**
- **NFR-UX-1**: Trazabilidad visible (ícono info + tooltip; sugerido vs override diferenciados tipográficamente; auditoría en hover).
- **NFR-UX-2**: Acciones reversibles > confirmadas (toast "Deshacer" 8 s, no `confirm()` modal).
- **NFR-UX-3**: Errores con quién/qué/por qué/qué hacer ahora, en 3 niveles (inline/negocio/sistema con código de incidente copiable).
- **NFR-UX-4**: Tone es-MX claro y no técnico.

**Mantenibilidad y testabilidad**
- **NFR-T-1**: Pipeline composable; cada paso (parsing → validación → catálogo → joins → forecast → cuantización → presupuesto-window → output) testeable en aislamiento.
- **NFR-T-2**: Golden datasets versionados en `src/test/resources/fixtures/golden/vN/` con `EXPECTED_OUTPUT.csv` firmado por Hugo + `DECISION_LOG.md`; inmutable una vez shipped.
- **NFR-T-3**: Property-based testing con jqwik sobre cuantización y presupuesto (invariantes de `project-context.md`).
- **NFR-T-4**: Contract testing (Spring Cloud Contract / Pact); contrato YAML antes de implementar.
- **NFR-T-5**: Backtesting suite sobre histórico real para acordar MAPE/WAPE con Finanzas (criterio pragmático = desviación vs YoY).
- **NFR-T-6**: Mutation testing con PIT sobre `forecasting.*`; ≥80% mutantes eliminados antes de mergear a `main`.

**Internacionalización**
- **NFR-I-1**: v1 solo es-MX; strings de UI en archivo de recursos para no cerrar la puerta a i18n futura.
- **NFR-I-2**: Identificadores de código en inglés; comentarios y documentación de negocio en español.

**Observabilidad**
- **NFR-O-1**: Métricas de operación expuestas (tiempo de consolidación por coordinador, excepciones/semana, tasa de override por ciclos 1-3 vs 4+, carry-overs/semana, latencia del motor, errores de ingesta por archivo, frescura del snapshot SIM).
- **NFR-O-2**: Dashboard de operación visible para administrador con las métricas anteriores.
- **NFR-O-3**: Alertas automáticas a admin cuando falla ingesta, crece anormalmente la tasa de excepciones de negocio, o el motor tarda fuera de SLA.

### Additional Requirements

> Requisitos técnicos derivados de `architecture.md` (`status: complete`, READY FOR IMPLEMENTATION) que impactan la creación de épicas e historias.

**🚀 Starter / inicialización (impacta Épica 1, Historia 1 — PRIMERA story de implementación):**
- **AR-Init**: Inicializar el monorepo con dos generadores oficiales: **Spring Initializr** (Maven, Java 21 LTS, Spring Boot 3.5.x, `groupId=mx.com.officedepot.insumos`, `artifactId=insumos-odemas-api`, deps: `web, data-jpa, postgresql, validation, security, oauth2-resource-server, flyway, actuator, testcontainers`) + **Angular CLI v21.2.x** (`ng new`, SCSS, routing, `strict`, sin SSR). Estructura `/backend` + `/frontend`. Pin exacto de versiones confirmado contra la imagen base corporativa. Bloqueado hasta aprobar `architecture.md`.

**Arquitectura de datos y persistencia:**
- **AR-DB**: PostgreSQL en Cloud SQL con esquemas `raw` (CSV/XLSX sin transformar) → `staging` (validado) → `mart` (presentación). Acceso híbrido: JPA/Hibernate para entidades de dominio/CRUD; JdbcTemplate para ingesta masiva, joins del pipeline y agregados del `mart`.
- **AR-Migrations**: Flyway (SQL versionado por el BOM de Boot 3.5); una migration por PR; nunca editar una ya aplicada.
- **AR-Types**: `NUMERIC(p,s)` ↔ `BigDecimal` (scale=4); `BIGINT` para cantidades; `TIMESTAMPTZ`/`DATE`; IDs tienda/SKU como `VARCHAR`; PK surrogate `BIGINT GENERATED ALWAYS AS IDENTITY`; `snake_case`.
- **AR-Bitacora**: Tabla **append-only** `bitacora_evento` (solo INSERT, sin UPDATE/DELETE), reforzada por permisos de BD; cada corrección referencia la entrada previa (FR-104). Append-only, no event sourcing completo.
- **AR-Cache**: Datos maestros en caché en memoria por ciclo (Spring Cache + Caffeine), invalidada al reingestar. Sin caché distribuido/Redis en v1.
- **AR-SIM-Policy**: OQ-114 → snapshot ausente/negativo/inconsistente para tienda×SKU = `SaldoSIMInconsistenteException` (aborta ciclo de esa tienda, el distrito continúa); snapshot global >36 h = rechazar operar y notificar admin.

**Autenticación y seguridad:**
- **AR-Auth**: Auth0 (OIDC/JWT); backend = Spring Security OAuth2 Resource Server validando JWT. Roles `COORDINADOR`/`ADMINISTRADOR` desde claims. **Scoping territorial** (filtro por `coordinadorId` en capa de servicio, no solo por rol). MFA admin en Auth0. Secretos en GCP Secret Manager.

**API y comunicación:**
- **AR-API**: REST + JSON versionado en `/api/v1`; documentación springdoc-openapi (OpenAPI 3). Endpoints en plural kebab-case; campos JSON camelCase; sin wrapper de respuesta.
- **AR-Errors**: Errores **solo** vía RFC 7807 `ProblemDetail` (nativo Spring 6); excepciones checked de negocio mapeadas a `ProblemDetail` (type/title/detail/instance + código de incidente). Conecta el fail-loud del backend con los 3 niveles de error UX (NFR-UX-3) vía `@ControllerAdvice` → interceptor HTTP Angular.
- **AR-Contracts**: **Spring Cloud Contract** (productor = Spring Boot; stubs consumidos por el frontend). Contrato definido en `contracts/` **antes** de implementar cualquiera de los dos lados (NFR-T-4).
- **AR-Money-JSON**: Dinero en JSON como **string decimal** (`"82000.00"`) para preservar `BigDecimal`; cantidades como `number` entero (`long`).

**Batch de forecast:**
- **AR-Batch**: Servicio Cloud Run con endpoint protegido disparado por **Cloud Scheduler** (semanal, alineado al ciclo lunes→miércoles) + **trigger manual** desde la UI de administración; `min-instances=1` en ventana operativa. Alternativa: Cloud Run Jobs.
- **AR-F05**: El factor de pérdida se aplica como **multiplicador sobre la demanda de venta predicha, ANTES de la conversión a insumo y la cuantización** (`demanda_ajustada = V × (1 + f)`); se descarta el ajuste sobre el histórico para v1.

**Frontend:**
- **AR-FE-State**: Angular **Signals** en servicios (actualizaciones inmutables, signals `readonly`) + RxJS solo para flujos asíncronos (`debounceTime(150)`). Modo zoneless por defecto v21. Sin NgRx.
- **AR-Grid**: **AG Grid (Community)** para la bandeja distrital (edición en celda, navegación por teclado, virtual scrolling); Angular Material table + CDK como fallback. Componentes standalone; modelos `models/api/` vs `models/view/`.
- **AR-Routing**: Angular Router con guards por rol; modo estricto (`strict`, `strictNullChecks`); `LOCALE_ID='es-MX'`.

**Infraestructura, despliegue y observabilidad:**
- **AR-Infra**: Cloud Run + Docker (backend, opcionalmente job batch); Firebase Hosting (frontend); Cloud SQL PostgreSQL.
- **AR-Storage**: Cloud Storage como landing de ingesta (`inbound/`: reportes del owner + snapshots SIM, descubiertos por patrón) y destino de exports/evidencias (`outbound/`: `SolicitusDeInsumosTodos.xlsx` + hash, evidencias FR-082).
- **AR-CICD**: Cloud Build (GCP-native): build+test (unit, jqwik, contract) → **PIT sobre `forecasting.*` gate ≥80%** → build Docker → deploy Cloud Run + Firebase Hosting. Config por entorno con Spring profiles + Secret Manager.
- **AR-Obs**: Spring Boot Actuator + Cloud Logging/Monitoring; MDC poblado; dashboard de operación para admin (NFR-O-2).
- **AR-Email**: SendGrid (patrón Tomaturno) para notificar al comprador (FR-094/110/111); alternativa SMTP corporativo.

**Estructura y fronteras (regla dura):**
- **AR-Hexagonal**: Organización backend hexagonal-lite — `dominio/` y `forecasting/` son núcleo puro **sin Spring/JPA/IO**; `api/`, `ingesta/`, `persistencia/` son adaptadores que dependen hacia adentro. `forecasting/` aislado para PIT ≥80%. POI para XLSX; Jackson CSV/univocity para parsing.

### UX Design Requirements

> No existe un documento UX independiente; estos requisitos UX-DR se derivan de la sección "UX no negociable" de `project-context.md`, del PRD §9.7–9.9, y de los componentes/pipes de frontend que `architecture.md` nombra explícitamente. Cada UX-DR es lo bastante específico para generar historias con criterios de aceptación verificables.

- **UX-DR1 — Trazabilidad visible de cada cantidad**: ícono `info` con tooltip que muestra origen del número, ventana de histórico usada, factor estacional/evento aplicado y factor de pérdida; ningún número crítico se renderiza sin contexto. (Componente reusable de explicación/trazo, consumido por la bandeja.)
- **UX-DR2 — Diferenciación sugerido vs override**: valor sugerido (gris/itálica) vs override (negro/bold + badge `Modificado por {user} {hh:mm}`); hover sobre fila modificada muestra la cadena de auditoría completa.
- **UX-DR3 — Recálculo optimista en cliente**: edición numérica dispara recálculo (RxJS `debounceTime(150)`) con confirmación servidor; recalcular fila + acumulado por proveedor + restante de presupuesto en <100 ms.
- **UX-DR4 — Autosave silencioso**: indicador tipo Google Docs ("Guardado hace 3 seg"); estado persistido server-side.
- **UX-DR5 — Acciones reversibles**: componente toast "Deshacer" (8 s) en lugar de `confirm()` modal.
- **UX-DR6 — Manejo de error en 3 niveles**: interceptor HTTP que traduce `ProblemDetail` del backend a (a) inline en campo, (b) banner de negocio en fila/sección con acción sugerida, (c) toast de sistema con código de incidente copiable + reintento con backoff. Mensajes en es-MX claro (`"La cantidad debe ser múltiplo de 6 (empaque mínimo)"`).
- **UX-DR7 — Pipes/formatters de locale de negocio MX (reusables en `frontend/shared`)**:
  - Cantidades discretas: enteros sin decimales, separador de miles, unidad pegada (`1,250 cajas`); equivalencia piezas↔cajas (`18 piezas = 3 cajas`).
  - Empaque mínimo: input rechaza no-múltiplos y sugiere el cercano.
  - Moneda MXN: `$1,250.00`; totales grandes abreviados con tooltip (`$1.2M`).
  - Porcentajes: un decimal máximo, con signo y color (`+12.5%` verde / `−8.3%` rojo, U+2212).
  - Fechas con contexto de semana ISO (`Sem 12 · 16–22 mar 2026`).
- **UX-DR8 — Accesibilidad WCAG 2.1 AA en grillas**: contraste 4.5:1/3:1, `aria-label` contextualizado por celda editable (`"Cantidad sugerida para papel opalina, tienda 28, semana 1, 18 cajas, editable"`), navegación por teclado completa, `:focus-visible` 2px.

### FR Coverage Map

> Mapeo FR → épica. Garantiza que ningún FR activo quede sin cubrir (56 FRs + NFR-O-*).

**Épica 1 — Fundación e ingesta**
- FR-001: Épica 1 — catálogo de tiendas (datos maestros)
- FR-002: Épica 1 — catálogo de SKUs de insumo
- FR-003: Épica 1 — tabla de equivalencia venta↔insumo (fail-loud)
- FR-005: Épica 1 — datos de coordinadores
- FR-006: Épica 1 — catálogo de eventos comerciales (consumido por E2)
- FR-007: Épica 1 — catálogo de tipos de papel (consumido por E2/E4)
- FR-010: Épica 1 — ingesta por descubrimiento de patrón de nombre
- FR-011: Épica 1 — detección de encoding (UTF-8 BOM → cp1252)
- FR-012: Épica 1 — validación de invariantes de ingesta
- FR-013: Épica 1 — parseo de presupuesto + periodicidad
- FR-014: Épica 1 — ingesta de snapshot SIM (unidireccional)
- FR-016: Épica 1 — ingesta de ventas semanales (reporte del owner)
- FR-017: Épica 1 — ingesta de ALLOC/TFS (reporte del owner)
- FR-073: Épica 1 — registro de histórico TFS (ingesta); alimenta motor (E2) y visualización (E4)
- FR-103: Épica 1 — MDC en logs (infraestructura de logging)

**Épica 2 — Motor de pronóstico y cálculo de sugerencia**
- FR-030: Épica 2 — demanda esperada de venta semanal
- FR-031: Épica 2 — conversión venta→insumo
- FR-032: Épica 2 — cuantización al empaque mínimo
- FR-033: Épica 2 — componente estacional + eventos comerciales
- FR-034: Épica 2 — generación del trazo explicable (se muestra en E3)
- FR-035: Épica 2 — aplicación del factor único de pérdida
- FR-040: Épica 2 — cálculo de cantidad sugerida por SKU
- FR-043: Épica 2 — SKU sin equivalencia excluido (fail-loud)
- FR-050: Épica 2 — costo total por tienda
- FR-051: Épica 2 — recorte sugerido por riesgo de quiebre (cálculo)
- FR-054: Épica 2 — presupuesto disponible (descuento de esporádicas); desglose mostrado en E3
- FR-084: Épica 2 — default global del factor (≈0.9%)
- FR-085: Épica 2 — el factor alimenta el motor (resolución jerárquica)

**Épica 3 — Bandeja de consolidación distrital**
- FR-041: Épica 3 — mostrar sugerencia con trazo en la bandeja
- FR-052: Épica 3 — aceptar/modificar/rechazar recorte (interactivo)
- FR-053: Épica 3 — persistir decisión de recorte
- FR-060: Épica 3 — bandeja de tiendas del distrito con ciclo activo
- FR-061: Épica 3 — estado por tienda (coord trabaja en pantalla)
- FR-063: Épica 3 — agregados del distrito en la semana
- FR-064: Épica 3 — override con razón estructurada
- FR-065: Épica 3 — aprobar el consolidado y disparar export
- FR-066: Épica 3 — surfacing pasivo de carry-over (sin regla de arrastre)
- FR-067: Épica 3 — dashboard pedido-vs-recibido
- FR-100: Épica 3 — registrar entrada en bitácora
- FR-101: Épica 3 — cadena completa de cambios en hover
- FR-102: Épica 3 — panel "Cambios de hoy" exportable a CSV
- FR-104: Épica 3 — bitácora inmutable (append-only)
- FR-110: Épica 3 — export XLSX compatible con la estructura de referencia
- FR-111: Épica 3 — exportar bajo demanda y enviar por correo
- FR-112: Épica 3 — registrar cada export en bitácora

**Épica 4 — Bandeja territorial: excepciones, factor y TFS**
- FR-074: Épica 4 — visualización cruzada del histórico TFS
- FR-075: Épica 4 — registro manual de TFS de emergencia
- FR-080: Épica 4 — detección estadística residual (capa opcional)
- FR-081: Épica 4 — candidatos residuales clasificados por severidad
- FR-082: Épica 4 — config del factor por tipo de papel + gestión de candidatos
- FR-083: Épica 4 — acumular candidatos confirmados para revisión
- FR-091: Épica 4 — registro de excepción mid-cycle (origen = correo)
- FR-092: Épica 4 — contexto agregado para la excepción
- FR-093: Épica 4 — decidir y registrar la respuesta
- FR-094: Épica 4 — notificar al comprador con contexto
- FR-095: Épica 4 — reflejar compra extraordinaria autorizada

**Épica 5 — Automatización y observabilidad operativa**
- NFR-O-1: Épica 5 — métricas de operación expuestas
- NFR-O-2: Épica 5 — dashboard de operación para administrador
- NFR-O-3: Épica 5 — alertas automáticas a admin

> NFRs transversales (NFR-E, NFR-P, NFR-D, NFR-A, NFR-S, NFR-DAT, NFR-L, NFR-A11Y, NFR-UX, NFR-T, NFR-I) y los requisitos adicionales de arquitectura (AR-*) y de diseño (UX-DR*) **no se mapean a una épica única**: se aplican como criterios de aceptación / definition-of-done dentro de las historias de las épicas correspondientes (p. ej. AR-Init → E1; AR-F05 → E2; AR-Grid/AR-Errors/UX-DR* → E3; AR-Email → E4; AR-CICD → E1 + E5).

## Epic List

### Épica 1: Fundación de la plataforma e ingesta de datos confiable
El **administrador** inicia sesión con identidad individual, gobierna los datos maestros (tiendas, SKUs de insumo, equivalencia venta↔insumo, coordinadores, tipos de papel, eventos comerciales) e **ingesta + valida** todos los archivos canónicos (ventas semanales, ALLOC/TFS, snapshot SIM, presupuestos) con comportamiento **fail-loud** y retroalimentación clara. Al cerrar la épica, los insumos de datos son confiables, los datos maestros están configurados, y el sistema es desplegable de punta a punta (walking skeleton sobre el pipeline de CI/CD).
**FRs covered:** FR-001, FR-002, FR-003, FR-005, FR-006, FR-007, FR-010, FR-011, FR-012, FR-013, FR-014, FR-016, FR-017, FR-073, FR-103.
**Notas de implementación:** consolida `AR-Init` (Spring Initializr + Angular CLI, monorepo `/backend` + `/frontend`), `AR-DB` (esquemas `raw`/`staging`/`mart`), `AR-Migrations` (Flyway baseline), `AR-Types`, `AR-Storage` (Cloud Storage `inbound/`), `AR-Auth` (Auth0 + scoping territorial + MFA admin), `AR-CICD` (Cloud Build + gate PIT). Estas capacidades comparten los archivos núcleo de persistencia/ingesta → consolidadas con historias ordenadas. Hereda convenciones de datos de `project-context.md` (encoding/fechas duales, BigDecimal, IDs String, literal `"Matarial y tamaño"`).

### Épica 2: Motor de pronóstico y cálculo de la sugerencia de pedido
El sistema **genera, por tienda × SKU, una cantidad de insumo sugerida** — cuantizada al empaque mínimo, ajustada por el factor único de pérdida (default global ≈0.9%), estacionalidad/eventos comerciales, y descuento de saldo SIM + ALLOC/TFS — acompañada del **trazo explicable** y **verificable por backtesting vs histórico real** (criterio pragmático "sin desviaciones importantes vs YoY"). Corre en batch (trigger manual) y persiste las sugerencias en el `mart`. Es la **Fase 1 (Predicción)**, corazón de v1, y constituye la **frontera de riesgo** del producto (spike R-8, backtesting R-9).
**FRs covered:** FR-030, FR-031, FR-032, FR-033, FR-034, FR-035, FR-040, FR-043, FR-050, FR-051, FR-054, FR-084, FR-085.
**Notas de implementación:** núcleo de dominio puro + `forecasting/` aislado (`AR-Hexagonal`), factor como multiplicador antes de conversión/cuantización (`AR-F05`), batch en Cloud Run (`AR-Batch`). Disciplina de testing intensiva: golden datasets firmados por Hugo (NFR-T-2), property-based jqwik (NFR-T-3), backtesting (NFR-T-5), mutation PIT ≥80% (NFR-T-6). **Funciona desde go-live con el default global** — independiente de la UI de configuración del factor (Épica 4).

### Épica 3: Bandeja de consolidación distrital del coordinador
El **coordinador** abre su bandeja semanal en pantalla, ve las ~38 tiendas en ciclo con sus sugerencias y trazo, valida el presupuesto con desglose `techo − esporádicas` y recorte sugerido, aplica **overrides con razón estructurada**, consulta agregados del distrito y el **dashboard pedido-vs-recibido**, aprueba el consolidado y **exporta el Excel compatible** al comprador por correo. Cada decisión queda en **bitácora inmutable**. Entrega el objetivo O1 — el ahorro de 9h → <2h por coordinador por semana.
**FRs covered:** FR-041, FR-052, FR-053, FR-060, FR-061, FR-063, FR-064, FR-065, FR-066, FR-067, FR-100, FR-101, FR-102, FR-104, FR-110, FR-111, FR-112.
**Notas de implementación:** AG Grid + Angular Signals (`AR-Grid`, `AR-FE-State`), recálculo optimista <100ms (NFR-P-1), autosave (NFR-P-2), toast Deshacer (NFR-UX-2); bitácora append-only (`AR-Bitacora`); ProblemDetail → 3 niveles de error UX (`AR-Errors`, NFR-UX-3); contrato Spring Cloud Contract antes de implementar (`AR-Contracts`, NFR-T-4); export XLSX con Apache POI. Concentra los UX-DR1–8 y los NFR-L/A11y.

### Épica 4: Bandeja territorial — excepciones, factor de pérdida y análisis TFS
El **coordinador territorial** registra y resuelve **excepciones mid-cycle** (formalizadas por correo) con contexto agregado (saldo SIM, ALLOC pendiente, TFS en tránsito e histórico) y escalamiento al comprador; **configura el factor de pérdida por tipo de papel** y revisa **candidatos residuales** de consumo atípico; consulta el **histórico TFS cruzado** con filtros.
**FRs covered:** FR-074, FR-075, FR-080, FR-081, FR-082, FR-083, FR-091, FR-092, FR-093, FR-094, FR-095.
**Notas de implementación:** reutiliza auth, bitácora y notificación por correo de épicas previas (`AR-Email` SendGrid para escalamiento). La config del factor (FR-082) **enriquece** el default que ya alimentaba el motor en Épica 2 — no lo bloquea (preserva la autonomía de E2). Acoplamiento FR-095 ↔ FR-054 (compra extraordinaria descuenta del techo del ciclo siguiente).

### Épica 5: Automatización y observabilidad operativa
El **administrador** monitorea la operación (tiempo de consolidación por coordinador, excepciones/semana, tasa de override por fase de calibración, carry-overs, latencia del motor, errores de ingesta por archivo, frescura del snapshot SIM) en un dashboard y **recibe alertas automáticas**; el batch del motor corre **programado** (Cloud Scheduler, ciclo lunes→miércoles) sin necesidad de trigger manual.
**FRs/NFRs covered:** NFR-O-1, NFR-O-2, NFR-O-3 · `AR-Batch` (automatización Cloud Scheduler) · `AR-Obs` (Spring Boot Actuator + Cloud Logging/Monitoring).
**Notas de implementación:** la observabilidad de bajo nivel (MDC, logging estructurado) ya está en Épica 1; aquí se entrega la capa **admin-facing** (dashboard + alertas) y la automatización del ciclo operativo.

**Flujo de dependencias:** Épica 1 → Épica 2 → Épica 3 → Épica 4 → Épica 5. Cada épica es autónoma y entrega valor independientemente; ninguna requiere una épica posterior para funcionar.

---

## Épica 1: Fundación de la plataforma e ingesta de datos confiable

**Goal:** El administrador inicia sesión con identidad individual, gobierna los datos maestros (tiendas, SKUs de insumo, equivalencia venta↔insumo, coordinadores, tipos de papel, eventos comerciales) e ingesta + valida todos los archivos canónicos (ventas semanales, ALLOC/TFS, snapshot SIM, presupuestos) con comportamiento fail-loud y retroalimentación clara. Al cerrar la épica, los insumos de datos son confiables, los datos maestros están configurados, y el sistema es desplegable de punta a punta (walking skeleton sobre el pipeline de CI/CD).

**FRs cubiertos:** FR-001, FR-002, FR-003, FR-005, FR-006, FR-007, FR-010, FR-011, FR-012, FR-013, FR-014, FR-016, FR-017, FR-073, FR-103.

### Story 1.1: Walking skeleton — monorepo, CI/CD y deploy hello-world

Como **arquitecto y DevOps del proyecto**,
quiero **un monorepo `/backend` + `/frontend` inicializado con Spring Initializr + Angular CLI y desplegado de punta a punta a GCP vía Cloud Build**,
para **tener un walking skeleton verificable que respete los pins de versión corporativos antes de escribir lógica de negocio**.

**Acceptance Criteria:**

**Dado** una rama limpia sin código de aplicación
**Cuando** se ejecuta el script de bootstrap del monorepo
**Entonces** existe el directorio `/backend` con un proyecto Spring Initializr generado con `groupId=mx.com.officedepot.insumos`, `artifactId=insumos-odemas-api`, Maven, Java 21 LTS, Spring Boot 3.5.x
**Y** las dependencias declaradas en `pom.xml` son exactamente: `web, data-jpa, postgresql, validation, security, oauth2-resource-server, flyway, actuator, testcontainers`
**Y** existe el directorio `/frontend` con un proyecto Angular CLI v21.2.x generado con SCSS, routing habilitado, `strict: true`, sin SSR

**Dado** el monorepo recién inicializado
**Cuando** un desarrollador ejecuta `./mvnw spring-boot:run` y `ng serve` localmente
**Entonces** el backend expone `GET /api/v1/health` devolviendo `{ "status": "UP" }`
**Y** el frontend renderiza una landing `Hola insumos-odemas` que consume `/api/v1/health` y muestra el status

**Dado** un PR contra `main` con cualquier cambio
**Cuando** se dispara el pipeline de Cloud Build
**Entonces** la build ejecuta secuencialmente: `mvn verify` → `ng build --configuration=production` → construcción de imagen Docker del backend → despliegue a Cloud Run (`backend`) → despliegue a Firebase Hosting (`frontend`)
**Y** el pipeline falla si cualquier paso retorna non-zero

**Dado** las versiones de runtime corporativas vigentes
**Cuando** se versionan los `pom.xml` y `package.json`
**Entonces** Java 21 LTS, Spring Boot 3.5.x y Angular v21.2.x quedan pinneados a versión exacta (no rango)
**Y** existe un documento `docs/version-pinning.md` que justifica cada pin contra la imagen base corporativa

**Dado** un endpoint protegido genérico añadido en historias futuras
**Cuando** se documenta el contrato OpenAPI vía springdoc-openapi
**Entonces** `GET /v3/api-docs` y `/swagger-ui.html` están accesibles en el deploy de Cloud Run
**Y** la documentación incluye automáticamente cualquier endpoint nuevo sin configuración adicional

---

### Story 1.2: Persistencia base — Flyway baseline, esquemas raw/staging/mart y tabla bitácora append-only

Como **desarrollador del pipeline**,
quiero **los esquemas Postgres y la tabla de bitácora inmutable creados vía Flyway baseline**,
para **que cada historia siguiente pueda añadir su propia migración sin reconfigurar la base de datos**.

**Acceptance Criteria:**

**Dado** una instancia Cloud SQL Postgres provisionada
**Cuando** se ejecuta `mvn flyway:migrate` por primera vez
**Entonces** existen los esquemas `raw`, `staging` y `mart`, todos vacíos de tablas de negocio
**Y** la migración baseline queda registrada como `V1__baseline.sql` y nunca se modifica en commits posteriores

**Dado** el esquema `mart`
**Cuando** se aplica la baseline
**Entonces** existe la tabla `mart.bitacora_evento` con columnas: `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`, `timestamp_evento TIMESTAMPTZ NOT NULL`, `actor VARCHAR(120) NOT NULL`, `entidad_tipo VARCHAR(80) NOT NULL`, `entidad_id VARCHAR(120) NOT NULL`, `valor_anterior JSONB`, `valor_nuevo JSONB`, `razon_estructurada VARCHAR(120)`, `razon_libre TEXT`, `referencia_evento_previo BIGINT REFERENCES mart.bitacora_evento(id)`, `correlation_id UUID NOT NULL`
**Y** existen triggers `bitacora_no_update` y `bitacora_no_delete` que lanzan excepción ante cualquier `UPDATE`/`DELETE` sobre la tabla
**Y** el rol Postgres de la aplicación tiene `INSERT, SELECT` pero `NO UPDATE, NO DELETE` sobre `mart.bitacora_evento`

**Dado** las convenciones AR-Types
**Cuando** se revisa la migración baseline
**Entonces** todos los tipos cumplen: `NUMERIC(p,4)` para montos, `BIGINT` para cantidades, `TIMESTAMPTZ` para fechas con hora, `DATE` para fechas puras, `VARCHAR` para IDs de tienda/SKU, `snake_case` en todos los nombres
**Y** las primary keys de tablas futuras se documentan como `BIGINT GENERATED ALWAYS AS IDENTITY`

**Dado** un desarrollador que clona el repo por primera vez
**Cuando** sigue las instrucciones de `README.md` sección "Base de datos local"
**Entonces** puede levantar un Postgres local (Docker), correr Flyway baseline y conectarse con las credenciales documentadas
**Y** el README aclara que cada PR añade exactamente una migración (Vn__descripcion.sql) y nunca edita una ya aplicada

---

### Story 1.3: Auth0 + scoping territorial + MFA admin

Como **coordinador y administrador**,
quiero **iniciar sesión con identidad individual mediante Auth0 y recibir solo los recursos de mi alcance territorial**,
para **que cada coordinador vea únicamente las tiendas de su distrito y la administración exija MFA**.

**Acceptance Criteria:**

**Dado** un tenant Auth0 configurado con la aplicación `insumos-odemas`
**Cuando** un usuario navega a la URL del frontend sin sesión
**Entonces** Angular Router lo redirige al login universal de Auth0
**Y** tras autenticación exitosa el JWT firmado regresa con claims `roles: ["COORDINADOR"]` o `roles: ["ADMINISTRADOR"]` y `coordinador_id` (cuando aplica)

**Dado** un JWT recibido por el backend
**Cuando** llega al endpoint `/api/v1/test-protegido`
**Entonces** Spring Security OAuth2 Resource Server valida la firma contra el JWKS de Auth0
**Y** sin JWT válido devuelve `401` con `ProblemDetail` (`type: about:blank, title: Unauthorized`)
**Y** con JWT válido pero rol insuficiente devuelve `403` con `ProblemDetail` (`title: Forbidden`)
**Y** con JWT válido y rol suficiente devuelve `200`

**Dado** un coordinador autenticado con `coordinador_id=C-NORTE-01`
**Cuando** llama a un endpoint que devuelve tiendas
**Entonces** la capa de servicio aplica un filtro `WHERE coordinador_id = :coordinadorId` derivado del JWT — no del query string
**Y** el filtro se aplica **antes** del control de rol, no después
**Y** intentar pasar `coordinadorId` distinto en query/body es ignorado silenciosamente (no escala privilegio)

**Dado** un usuario con rol `ADMINISTRADOR` registrado en Auth0
**Cuando** intenta autenticarse sin MFA habilitado
**Entonces** Auth0 le exige enrolar MFA antes de emitir el token
**Y** sin MFA enrolado no se emite token y el usuario ve la pantalla de enrolamiento

**Dado** los secretos del tenant Auth0 (`client_id`, `client_secret`, `audience`, `issuer`)
**Cuando** el backend arranca
**Entonces** los lee desde **GCP Secret Manager** y no desde `application.properties`
**Y** ningún secreto aparece en logs, en repo, ni en variables de entorno literales

**Dado** Angular Router con guards por rol
**Cuando** un coordinador intenta navegar a una ruta marcada como `ADMINISTRADOR`
**Entonces** el guard lo redirige a `/forbidden` con mensaje `"Esta sección requiere rol Administrador"`
**Y** la navegación no carga el módulo de admin (lazy-load bloqueado)

---

### Story 1.4: Logging estructurado MDC + interceptor de errores ProblemDetail

Como **desarrollador y administrador**,
quiero **que toda excepción de negocio se transforme automáticamente en `ProblemDetail` RFC 7807 con código de incidente, y que los logs lleven MDC con `tiendaId`/`skuId`/`cicloId`/`usuarioId`**,
para **que debugging post-mortem sea viable, los errores sean auditables y la UI pueda traducirlos a los 3 niveles UX consistentemente**.

**Acceptance Criteria:**

**Dado** un servlet filter de logging activado en el backend
**Cuando** llega cualquier request HTTP con JWT y headers `X-Tienda-Id`, `X-Sku-Id`, `X-Ciclo-Id`
**Entonces** el filter popula MDC con `tiendaId`, `skuId`, `cicloId`, `usuarioId` (este último del claim `sub` del JWT)
**Y** el MDC se limpia al final de la request (no fuga entre requests del mismo thread)

**Dado** SLF4J + Logback configurados con appender JSON
**Cuando** cualquier código loggea con `log.info("mensaje")`
**Entonces** el log emitido es JSON con campos: `timestamp`, `level`, `message`, `logger`, `thread`, `mdc.tiendaId`, `mdc.skuId`, `mdc.cicloId`, `mdc.usuarioId`, `mdc.correlationId`
**Y** Cloud Logging muestra los campos MDC como labels estructurados

**Dado** un `@ControllerAdvice` global registrado
**Cuando** un controlador lanza cualquier excepción checked de negocio (`EquivalenciaNoDefinidaException`, `PeriodicidadPresupuestoIndefinidaException`, `SaldoSIMInconsistenteException`, etc.)
**Entonces** se mapea a `ProblemDetail` (Spring 6 nativo) con `type: https://insumos-odemas.officedepot.com.mx/problems/{slug-del-tipo}`, `title` en español, `detail` con contexto, `instance: /api/v1/...`, y propiedad extension `incidente: <UUID>` único
**Y** el mismo `incidente` aparece en el log de la excepción para correlación
**Y** excepciones no controladas (`RuntimeException` genéricas) mapean a `500` con `title: "Error interno"`, `detail: "Contacta a soporte con el código de incidente"` (sin stacktrace expuesto al cliente)

**Dado** Angular con un HTTP interceptor global
**Cuando** el backend responde con `ProblemDetail`
**Entonces** el interceptor lee `incidente`, `title`, `detail`, `type` y dispara un stub de los 3 niveles UX (UX-DR6): inline en campo si el `type` empata con un validation error, banner de negocio para excepciones de dominio, toast de sistema para `500`
**Y** el stub muestra el código de incidente copiable en errores de sistema

**Dado** Spring Boot Actuator habilitado
**Cuando** un operador consulta `GET /actuator/health`
**Entonces** devuelve estado del backend, conectividad a Cloud SQL y disponibilidad de Auth0 JWKS
**Y** `GET /actuator/info` expone `git.commit.id` y versión del artifact (sin secretos)
**Y** `GET /actuator/metrics` expone métricas JVM y de timer por endpoint

**Dado** un test de integración del @ControllerAdvice
**Cuando** se lanza una `EquivalenciaNoDefinidaException("tienda=28, sku_venta=54693")`
**Entonces** el cliente recibe HTTP `422` con `ProblemDetail.type` terminado en `/equivalencia-no-definida`, `title: "Equivalencia no definida"`, `detail` incluye `tienda=28` y `sku_venta=54693`, y `incidente` es un UUID parseable

---

### Story 1.5: Catálogo de tiendas (CRUD admin)

Como **administrador**,
quiero **crear, leer, actualizar y desactivar tiendas con sus atributos operativos (distrito, coordinador territorial, formato, ciclo quincenal, techo presupuestal)**,
para **que la base de 226 tiendas esté configurada y disponible como datos maestros del pipeline**.

**Acceptance Criteria:**

**Dado** la migración Flyway de Épica 1.5
**Cuando** se aplica
**Entonces** existe `mart.tienda` con columnas: `id VARCHAR PRIMARY KEY`, `nombre VARCHAR NOT NULL`, `distrito VARCHAR NOT NULL`, `coordinador_id BIGINT REFERENCES mart.coordinador(id) NULL` (NULL temporal hasta que exista 1.6), `formato VARCHAR CHECK (formato IN ('EXPRESS', 'ESTANDAR'))`, `ciclo_quincenal VARCHAR CHECK (ciclo_quincenal IN ('1Y3', '2Y4'))`, `techo_presupuestal_semanal NUMERIC(12,2) NOT NULL CHECK (techo_presupuestal_semanal > 0)`, `activa BOOLEAN NOT NULL DEFAULT TRUE`
**Y** NO existe columna `factor_perdida` en `tienda` (el factor vive en `tipo_papel`, ver 1.7)

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/tiendas` con un payload válido
**Entonces** se persiste la tienda y devuelve `201 Created` con la representación completa
**Y** se registra una entrada en `bitacora_evento` con `entidad_tipo=tienda`, `entidad_id=<id>`, `valor_anterior=null`, `valor_nuevo=<JSON>`, `actor=<usuarioId>`

**Dado** una tienda existente
**Cuando** un administrador llama `PUT /api/v1/tiendas/{id}` con cambios
**Entonces** se actualiza y se registra en bitácora con `valor_anterior` y `valor_nuevo` completos
**Y** si el cambio incluye `coordinador_id`, la regla de exclusividad de coordinador (definida en 1.6) se valida

**Dado** una tienda existente
**Cuando** un administrador llama `DELETE /api/v1/tiendas/{id}`
**Entonces** se marca `activa = FALSE` (soft delete), no se borra físicamente
**Y** la tienda inactiva queda excluida de listados por defecto (`GET /api/v1/tiendas` filtra `activa=true` salvo `?incluir_inactivas=true`)

**Dado** Caffeine cache habilitado para `tienda`
**Cuando** se hace `POST`, `PUT` o `DELETE`
**Entonces** la cache de `tienda` se invalida (la siguiente lectura recarga de BD)
**Y** lecturas consecutivas sin cambios sirven desde cache (verificable por métrica `cache.hits`)

**Dado** un usuario con rol `COORDINADOR`
**Cuando** intenta llamar `POST /api/v1/tiendas`
**Entonces** recibe `403 Forbidden` con `ProblemDetail`
**Y** la operación no se registra en bitácora (sin side effects)

**Dado** la UI admin Angular standalone
**Cuando** un administrador navega a `/admin/tiendas`
**Entonces** ve una tabla con paginación servidor-side, filtros por distrito y formato, y un botón "Nueva tienda"
**Y** el form respeta accesibilidad WCAG 2.1 AA (UX-DR8): labels asociados, `:focus-visible` 2px, navegación por teclado completa

---

### Story 1.6: Catálogo de coordinadores y sincronización con Auth0

Como **administrador**,
quiero **dar de alta, modificar y desactivar coordinadores con sus tiendas territoriales asignadas, manteniendo sincronía con Auth0**,
para **que el scoping territorial de 1.3 funcione con datos reales y la asignación distrito ↔ coordinador sea explícita y auditable**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.6
**Cuando** se aplica
**Entonces** existe `mart.coordinador` con `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`, `nombre VARCHAR NOT NULL`, `email VARCHAR UNIQUE NOT NULL`, `auth0_subject VARCHAR UNIQUE NOT NULL`, `rol VARCHAR CHECK (rol IN ('COORDINADOR', 'ADMINISTRADOR')) NOT NULL`, `activo BOOLEAN NOT NULL DEFAULT TRUE`
**Y** la FK `tienda.coordinador_id` pasa de NULL a `NOT NULL` mediante una migración que requiere todas las tiendas asignadas como precondición (validación en flyway:migrate)

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/coordinadores` con `email`, `nombre`, `rol`, `tiendas_asignadas: [...]`
**Entonces** se crea el registro en `mart.coordinador`, se llama a Auth0 Management API para invitar al usuario por email (MFA forzado si `rol = ADMINISTRADOR`), se persiste `auth0_subject` retornado, y se asignan las tiendas
**Y** se registra en bitácora la creación + cada asignación de tienda

**Dado** una tienda actualmente asignada al coordinador A
**Cuando** un administrador la reasigna al coordinador B vía `PUT /api/v1/tiendas/{id}` o `PUT /api/v1/coordinadores/{id}/tiendas`
**Entonces** se actualiza `tienda.coordinador_id = B`
**Y** se registra en bitácora: desasignación de A (`valor_anterior=A, valor_nuevo=B`) y asignación a B, ambos con timestamp único y `referencia_evento_previo` enlazando
**Y** ninguna tienda activa queda con `coordinador_id = NULL`

**Dado** un coordinador activo con tiendas asignadas
**Cuando** un administrador intenta desactivarlo (`DELETE /api/v1/coordinadores/{id}`)
**Entonces** la operación falla con `ProblemDetail` `409 Conflict` (`title: "Coordinador con tiendas asignadas"`) hasta que sus tiendas hayan sido reasignadas a otro coordinador activo
**Y** el mensaje sugiere el endpoint de reasignación

**Dado** la sincronización con Auth0
**Cuando** un administrador cambia el rol de un coordinador de `COORDINADOR` a `ADMINISTRADOR`
**Entonces** se actualiza el rol en Auth0 (vía Management API) y se le exige re-enrolamiento de MFA en su siguiente login
**Y** el cambio queda en bitácora con razón estructurada obligatoria

**Dado** un coordinador inactivo
**Cuando** intenta autenticarse
**Entonces** Auth0 le permite obtener token (no es responsabilidad de Auth0 saber si está activo), pero el backend rechaza con `403` (`title: "Coordinador inactivo"`)
**Y** sus tiendas previamente asignadas siguen mostrando el coordinador inactivo hasta reasignación explícita — no se "huérfanan" silenciosamente

---

### Story 1.7: Catálogo de SKUs de insumo y tipos de papel

Como **administrador**,
quiero **gestionar el catálogo de SKUs comprables al proveedor y sus tipos de papel (la unidad de configuración del factor de pérdida)**,
para **que el motor de pronóstico (E2) pueda convertir venta a insumo y E4 pueda configurar el factor por tipo de papel**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.7
**Cuando** se aplica
**Entonces** existe `mart.tipo_papel` (`id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `nombre VARCHAR UNIQUE NOT NULL`, `derivado_de_material_csv VARCHAR NOT NULL` documentando el valor original del atributo `Material`)
**Y** existe `mart.sku_insumo` (`codigo VARCHAR PRIMARY KEY`, `descripcion VARCHAR NOT NULL`, `presentacion VARCHAR NOT NULL`, `contenido BIGINT NOT NULL CHECK (contenido > 0)` piezas/presentación, `costo NUMERIC(p,4) NOT NULL CHECK (costo > 0)` MXN, `tipo_papel_id BIGINT REFERENCES mart.tipo_papel(id) NOT NULL`, `activo BOOLEAN NOT NULL DEFAULT TRUE`)

**Dado** un parser de `Skus_insumos.csv`
**Cuando** lee el archivo
**Entonces** mapea la cabecera literal `"Matarial y tamaño"` (con typo) al campo POJO `materialYTamano`
**Y** si la cabecera del CSV viene corregida como `"Material y tamaño"`, el parser lanza `CabeceraCSVInesperadaException` (fail-loud — cambio de contrato, no mejora silenciosa)
**Y** existe un test unitario que verifica ambos casos

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/skus-insumo/upload-masivo` con `Skus_insumos.csv`
**Entonces** se parsean todas las filas, se derivan los `tipo_papel` únicos del atributo `Material`, se insertan los SKUs nuevos, se actualizan los existentes (por `codigo`)
**Y** la respuesta incluye contador de creados/actualizados/errores con detalle por fila
**Y** cada cambio queda en bitácora

**Dado** un CRUD admin para tipos de papel
**Cuando** un administrador crea, edita o desactiva un tipo de papel
**Entonces** las operaciones quedan en bitácora
**Y** desactivar un tipo de papel con SKUs activos asociados falla con `409 Conflict`

**Dado** Caffeine cache para `sku_insumo` y `tipo_papel`
**Cuando** ocurre cualquier escritura
**Entonces** la cache se invalida y la siguiente lectura recarga de BD

**Dado** la UI admin
**Cuando** un administrador navega a `/admin/skus-insumo`
**Entonces** ve listado paginado con filtros por tipo de papel y por estado (activo/inactivo)
**Y** un panel de upload-masivo con preview de cambios antes de confirmar

---

### Story 1.8: Tabla de equivalencia SKU venta ↔ SKU insumo (fail-loud preparada)

Como **administrador**,
quiero **mantener la tabla autoritativa de equivalencias `sku_venta_id → sku_insumo` y disponer de la excepción checked `EquivalenciaNoDefinidaException`**,
para **que el motor (E2) pueda convertir ventas a insumo y los faltantes se detecten fail-loud sin contaminar el `mart`**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.8
**Cuando** se aplica
**Entonces** existe `mart.equivalencia_venta_insumo` (`sku_venta_id VARCHAR PRIMARY KEY`, `sku_insumo VARCHAR NOT NULL REFERENCES mart.sku_insumo(codigo)`, `vigencia_desde DATE NOT NULL`, `vigencia_hasta DATE NULL`, `actualizado_en TIMESTAMPTZ NOT NULL DEFAULT now()`)

**Dado** la clase Java `EquivalenciaNoDefinidaException`
**Cuando** se inspecciona en código
**Entonces** es `checked` (`extends Exception`), no `RuntimeException`
**Y** su constructor obliga a recibir `tiendaId` y `skuVentaId` y los incluye en `getMessage()`
**Y** existe un test que verifica que el `@ControllerAdvice` la mapea a `ProblemDetail 422` con `type` terminado en `/equivalencia-no-definida` (ya cubierto en 1.4)

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/equivalencias/upload-masivo` con `Cruce-de-skus-venta-insumo.xlsx`
**Entonces** el archivo XLSX se parsea con Apache POI, se insertan/actualizan equivalencias por `sku_venta_id`, y la respuesta incluye contador de creados/actualizados/errores
**Y** filas con `sku_insumo` que no existe en `mart.sku_insumo` se rechazan con detalle (no se inserta orphan FK)
**Y** cada cambio queda en bitácora

**Dado** un administrador autenticado
**Cuando** llama `GET /api/v1/equivalencias/faltantes`
**Entonces** el endpoint cruza `staging.venta_semanal.sku_venta` (poblado tras 1.12) contra `mart.equivalencia_venta_insumo` y devuelve la lista de `sku_venta_id` sin equivalencia activa
**Y** si `staging.venta_semanal` está vacío (antes de 1.12), devuelve `[]` y un header `X-Datos-Disponibles: false`

**Dado** la UI admin en `/admin/equivalencias`
**Cuando** un administrador busca por `sku_venta_id` o `sku_insumo`
**Entonces** ve la coincidencia con vigencia y fecha de última actualización
**Y** puede editar la equivalencia abriendo una nueva vigencia (no sobreescribe la histórica)

---

### Story 1.9: Catálogo de eventos comerciales

Como **administrador**,
quiero **configurar eventos comerciales con su ventana de fechas y factor multiplicador**,
para **que el motor de pronóstico (E2) pueda aplicar el componente estacional**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.9
**Cuando** se aplica
**Entonces** existe `mart.evento_comercial` (`id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `nombre VARCHAR NOT NULL`, `fecha_inicio DATE NOT NULL`, `fecha_fin DATE NOT NULL CHECK (fecha_fin >= fecha_inicio)`, `factor_multiplicador NUMERIC(8,4) NOT NULL CHECK (factor_multiplicador > 0)`, `activo BOOLEAN NOT NULL DEFAULT TRUE`)

**Dado** la baseline data de la migración
**Cuando** se aplica por primera vez
**Entonces** la tabla queda pre-poblada con los 8 eventos canónicos: Hot Sale, BTS (Back-to-School), Buen Fin, Navidad/Reyes, regreso a oficinas, Día de las Madres, Día del Padre, Día del Niño, cierre fiscal — con fechas placeholder marcadas explícitamente como pendientes de confirmación (D-5)
**Y** los factores iniciales son `1.0` (neutros) hasta que el administrador los ajuste

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/eventos-comerciales` o `PUT /api/v1/eventos-comerciales/{id}`
**Entonces** se valida que `fecha_inicio < fecha_fin` y `factor > 0`
**Y** se rechaza con `400` si el mismo `nombre` ya tiene un evento activo con ventana de fechas que solapa
**Y** cada cambio queda en bitácora

**Dado** Caffeine cache para `evento_comercial`
**Cuando** ocurre escritura
**Entonces** la cache se invalida

**Dado** la UI admin en `/admin/eventos-comerciales`
**Cuando** un administrador navega
**Entonces** ve listado calendárico (timeline anual) con eventos pendientes de confirmar destacados visualmente
**Y** puede editar `factor` y `ventana_fechas` con razón estructurada en bitácora

---

### Story 1.10: Plumbing de ingesta CSV — detección de encoding y descubrimiento por patrón

Como **desarrollador de los parsers**,
quiero **un servicio reutilizable que descubra archivos en Cloud Storage por patrón de nombre y detecte su encoding**,
para **que cada parser concreto (1.11, 1.12) no reimplemente la lógica común**.

**Acceptance Criteria:**

**Dado** un bucket Cloud Storage `insumos-odemas-inbound` provisionado vía Terraform/gcloud
**Cuando** se inspecciona el IAM
**Entonces** el Service Account del backend tiene permisos `storage.objects.list` y `storage.objects.get` sobre el bucket
**Y** ninguna otra cuenta tiene acceso de escritura (los archivos se suben por proceso externo del owner — fuera de scope v1)

**Dado** el servicio `IngestaArchivoService`
**Cuando** se llama `listarPorPatron(patron, prefijo)`
**Entonces** devuelve la lista de objetos del bucket cuyo `name` empata el regex `patron` bajo el `prefijo` dado
**Y** los resultados se ordenan ascendentemente por fecha embebida en el nombre cuando el patrón la captura (`ALLOC_YYYY_MM_DD.csv`)
**Y** se cachean en Caffeine con TTL 60s (el bucket no recibe escrituras frecuentes)

**Dado** un `InputStream` de archivo
**Cuando** se llama `detectarEncoding(stream)`
**Entonces** el servicio intenta primero leer como UTF-8 con BOM; si falla la validación (caracteres `?`/no representables en muestra de las primeras 1000 líneas), reintenta como `cp1252`; si ambos fallan, lanza `EncodingNoDetectableException` (checked)
**Y** loguea (con MDC poblado por nombre de archivo) el encoding detectado
**Y** existe test con tres fixtures: UTF-8 con BOM (detecta UTF-8), cp1252 con acentos (detecta cp1252), binario aleatorio (lanza excepción)

**Dado** un administrador autenticado
**Cuando** llama `GET /api/v1/ingesta/archivos?patron=ALLOC_\d{4}_\d{2}_\d{2}\.csv`
**Entonces** recibe lista de archivos pendientes en `inbound/` que empatan el patrón con su fecha embebida y tamaño
**Y** el endpoint requiere rol `ADMINISTRADOR`

**Dado** que este servicio es solo plumbing
**Cuando** se revisa el código
**Entonces** **NO** parsea contenido CSV/XLSX de ningún archivo concreto — solo lista, detecta encoding y abre streams
**Y** está en el paquete `ingesta.infraestructura`, no en `ingesta.dominio` (adaptador, no núcleo)

---

### Story 1.11: Ingesta de presupuestos de tiendas

Como **administrador**,
quiero **subir `Presupuesto-tiendas.csv` y poblar la tabla de presupuestos con la cantidad semanal MXN y las semanas ISO de solicitud por tienda**,
para **que el motor (E2) pueda validar contra presupuesto y la bandeja (E3) muestre el desglose `techo − esporádicas`**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.11
**Cuando** se aplica
**Entonces** existe `staging.presupuesto_tienda` (`id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `tienda_id VARCHAR NOT NULL REFERENCES mart.tienda(id)`, `cantidad_semanal NUMERIC(12,2) NOT NULL CHECK (cantidad_semanal > 0)`, `semanas_solicitud JSONB NOT NULL`, `vigencia_desde DATE NOT NULL`, `vigencia_hasta DATE NULL`, `ingestado_en TIMESTAMPTZ NOT NULL DEFAULT now()`)
**Y** existe índice único `(tienda_id, vigencia_desde)`

**Dado** el parser de `Presupuesto-tiendas.csv`
**Cuando** lee la celda `"$82,000"` o `"$1,234,567.89"`
**Entonces** strip de `$` y `,`, conversión a `BigDecimal(scale=2)` → `82000.00` o `1234567.89`
**Y** celdas vacías o no parseables lanzan `MontoPresupuestoInvalidoException` para esa fila

**Dado** el parser de `Semana que debe solicitar`
**Cuando** lee `"Solicitan insumo semana 1 y 3"`, `"semana 2 y 4"`, `"Semanas 1, 3"`, `"sem 2 y 4"`
**Entonces** mapea a `List<Integer>` de números de semana ISO: `[1, 3]`, `[2, 4]`, `[1, 3]`, `[2, 4]` respectivamente
**Y** cualquier patrón no reconocido (`"cuando se acuerde"`, `"variable"`, `""`) lanza `PeriodicidadPresupuestoIndefinidaException` (checked) con `tiendaId` y texto crudo en el mensaje
**Y** la excepción aborta solo esa fila — el resto del archivo continúa procesándose

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/ingesta/presupuestos` adjuntando el CSV
**Entonces** el endpoint usa `IngestaArchivoService.detectarEncoding` (de 1.10), parsea todas las filas, persiste las válidas en `staging.presupuesto_tienda` cerrando vigencias anteriores (`vigencia_hasta = new_vigencia_desde - 1 day`)
**Y** devuelve un reporte `{ filas_procesadas, filas_ok, errores: [{fila, tienda_id, mensaje, texto_crudo}] }`
**Y** cada upload genera entrada en `bitacora_evento` con `entidad_tipo='ingesta_presupuesto'`, `hash_archivo`, `total_filas`, `total_errores`

**Dado** la UI admin en `/admin/ingesta/presupuestos`
**Cuando** un administrador sube un archivo
**Entonces** ve preview de las primeras 20 filas con su parseo (cantidad parseada, semanas parseadas, marca de error si aplica) **antes** de confirmar la ingesta
**Y** confirmar dispara el `POST`
**Y** tras procesar, ve un estado por tienda: última vigencia, monto, semanas, fecha de última ingesta

---

### Story 1.12: Ingesta de archivos canónicos restantes (ventas, ALLOC, TFS, Entrega, SIM)

Como **administrador operativo**,
quiero **ingestar los cinco archivos canónicos transaccionales con validaciones fail-loud por archivo**,
para **que los datos de ventas, surtidos, transferencias, compras directas y saldos SIM estén disponibles para el motor (E2), la bandeja distrital (E3) y la bandeja territorial (E4)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 1.12
**Cuando** se aplica
**Entonces** existen las tablas en `raw` y `staging` para cada archivo:
**Y** `raw.venta_mensual` + `staging.venta_semanal` para ventas
**Y** `raw.alloc` + `staging.alloc` para ALLOC
**Y** `raw.tfs` + `staging.tfs` para TFS
**Y** `raw.entrega_directa` + `staging.entrega_directa` para Entrega-directa-tienda
**Y** `raw.snapshot_sim` + `staging.snapshot_sim` para SIM (saldo por tienda × SKU)
**Y** todas las cantidades son `BIGINT`, montos `NUMERIC(p,4)`, IDs `VARCHAR`, fechas `TIMESTAMPTZ`/`DATE` según corresponda

**Dado** el parser de ventas (`historico-de-ventas-*.csv`)
**Cuando** procesa una fila con `(year, month)` enteros
**Entonces** reconstruye `YearMonth` y promueve a `staging.venta_semanal` con fallback de prorrateo lineal mensual→semanal ISO si no existe corte semanal en el archivo
**Y** loguea advertencia por cada `(tienda, mes)` prorrateado: `"Prorrateo mensual→semanal aplicado por ausencia de corte semanal"`
**Y** `item_id_venta`, `sku_venta`, `location_key` se persisten como `VARCHAR` (no Integer)

**Dado** el parser de ALLOC (`ALLOC_YYYY_MM_DD.csv`)
**Cuando** procesa el archivo
**Entonces** parsea `RELEASE_DATE` y `FECHA_DE_VIGENCIA_OC` como `dd/MM/yyyy`
**Y** parsea `FECHA_CREATION_DATE_ORDEN` con parser dual: primero `dd/MM/yyyy HH:mm`, fallback `dd/MM/yyyy`
**Y** valida la invariante `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE` por fila; violación → warning loggeado (con MDC poblado de `tiendaId, skuId`), la fila se persiste pero se marca `invariante_violada=true` en `staging.alloc`
**Y** la columna `papelEspacial` (sic) admite `""` legítimo sin colapsar a `null`

**Dado** el parser de TFS (`TFS_YYYY_MM_DD.csv`)
**Cuando** procesa el archivo
**Entonces** parsea `CREATE_DATE` con parser dual `dd/MM/yyyy HH:mm` o `dd/MM/yyyy`
**Y** `FROM_LOC`/`TO_LOC` son `VARCHAR` (pueden ser warehouse o tienda; no se infiere por rango numérico)
**Y** la trailing comma en cabecera (`STATUS,`) y el último campo placeholder (`-`) se descartan documentadamente
**Y** `STATUS` se persiste crudo (`A`, `I`, `S`, etc.) — no se interpreta hasta que exista glosario formal (riesgo abierto)

**Dado** el parser de Entrega-directa-tienda
**Cuando** procesa una fila
**Entonces** valida que `TRAN_CODE ↔ DECODE` empaten (ej. `TRAN_CODE=20` → `DECODE='Purchases'`; `TRAN_CODE=30` → `DECODE='Transfers In'`)
**Y** discrepancia → `DecodeInconsistenteException` (checked), la fila se rechaza fail-loud con detalle de qué se esperaba
**Y** `LOCATION` se usa como join contra `tienda.id` (no `STORE_NAME` con prefijo `"NN-Nombre"`)
**Y** `TOTAL_COST` y `TOTAL_RETAIL` son `BigDecimal(scale=4)`

**Dado** el parser de snapshot SIM
**Cuando** procesa un archivo
**Entonces** valida que el snapshot tiene fecha de generación (`FECHA_SNAPSHOT`) dentro de los últimos `36h` respecto a `now()`
**Y** si la antigüedad > 36h, rechaza el archivo completo con `SnapshotSIMVencidoException` y notifica al administrador (email vía SendGrid en E5, log + bitácora en E1)
**Y** filas con `saldo_unidades < 0` o `saldo_unidades` ausente lanzan `SaldoSIMInconsistenteException(tiendaId, skuInsumo)` (checked) que aborta SOLO esa fila — el resto del batch continúa
**Y** las filas rechazadas se cuentan y se reportan al admin con detalle

**Dado** cada upload de los cinco archivos
**Cuando** termina
**Entonces** se persiste en `mart.bitacora_evento` una entrada con `entidad_tipo='ingesta_<tipo>'`, `hash_archivo`, `encoding_detectado`, `total_filas`, `total_errores`, `errores_resumen JSON`
**Y** el admin ve un dashboard en `/admin/ingesta` con estado por archivo: última fecha de ingesta, última fecha de datos, total filas, errores pendientes, encoding detectado

**Dado** golden fixtures en `src/test/resources/fixtures/ingesta/`
**Cuando** se ejecuta la suite de tests
**Entonces** existe un fixture por archivo con: archivo de input (muestra representativa), `EXPECTED_OUTPUT.csv` con las filas que deben terminar en `staging.*`, `DECISION_LOG.md` documentando cada caso de error esperado
**Y** los tests fallan si el output diverge byte-a-byte (o con tolerancia documentada)
**Y** los fixtures están firmados por Hugo (comprador piloto) en su `DECISION_LOG.md`

**Dado** un administrador autenticado
**Cuando** llama `POST /api/v1/ingesta/{tipo}` con `tipo ∈ {ventas, alloc, tfs, entrega, sim}`
**Entonces** el endpoint procesa el archivo y devuelve el reporte estructurado (filas procesadas, OK, errores con detalle, encoding detectado)
**Y** los endpoints requieren rol `ADMINISTRADOR`

**Dado** que el formato del snapshot SIM aún no está confirmado con el owner (OQ-117 parcial)
**Cuando** se implementa el parser SIM
**Entonces** el parser se aísla detrás de la interfaz `SnapshotSIMParser` con implementación `SnapshotSIMParserV1` basada en el formato placeholder documentado
**Y** existe un test que falla intencionalmente con mensaje claro `"Definir formato SIM con owner antes de go-live"` si la implementación V1 no se ha reemplazado por una concreta

---

## Épica 2: Motor de pronóstico y cálculo de la sugerencia de pedido

**Goal:** El sistema genera, por tienda × SKU, una cantidad de insumo sugerida — cuantizada al empaque mínimo, ajustada por el factor único de pérdida (default global ≈0.9%), estacionalidad/eventos comerciales, y descuento de saldo SIM + ALLOC/TFS — acompañada del trazo explicable y verificable por backtesting vs histórico real (criterio pragmático "sin desviaciones importantes vs YoY"). Corre en batch (trigger manual) y persiste las sugerencias en el `mart`. Es la Fase 1 (Predicción), corazón de v1, y constituye la frontera de riesgo del producto (spike R-8, backtesting R-9).

**FRs cubiertos:** FR-030, FR-031, FR-032, FR-033, FR-034, FR-035, FR-040, FR-043, FR-050, FR-051, FR-054, FR-084, FR-085.

### Story 2.1: `ForecastingEngine` — contrato y esqueleto del módulo aislado

Como **arquitecto del módulo de pronóstico**,
quiero **el paquete `forecasting/` definido como núcleo puro Java sin dependencias de Spring/JPA/IO, con la interfaz `ForecastingEngine` y sus POJOs de salida**,
para **que el motor sea testeable en aislamiento, alcanzable por PIT ≥80% y desacoplado del framework**.

**Acceptance Criteria:**

**Dado** la migración Flyway de Épica 2 / Story 2.1
**Cuando** se ejecuta `mvn compile`
**Entonces** existe el paquete `mx.com.officedepot.insumos.forecasting` con subpaquetes `engine`, `modelo`, `trazo`
**Y** la interfaz `ForecastingEngine` declara `ResultadoForecast predecir(SerieTemporal historico, VentanaPronostico ventana) throws CapacidadInsuficienteHistoricoException`
**Y** los records POJO `ResultadoForecast`, `TrazoForecast`, `SerieTemporal`, `VentanaPronostico`, `PuntoSemanal` quedan definidos como tipos inmutables

**Dado** ArchUnit configurado en la suite de tests
**Cuando** se ejecuta el test `ForecastingArchitectureTest`
**Entonces** verifica que ningún archivo bajo `forecasting/**` importa `org.springframework.*`, `jakarta.persistence.*`, `org.hibernate.*` o `java.io.File`
**Y** verifica que ningún archivo bajo `forecasting/**` lleva anotaciones `@Service`, `@Component`, `@Repository`, `@Autowired`
**Y** el test falla si cualquier nueva clase del paquete rompe estas reglas

**Dado** la excepción checked `CapacidadInsuficienteHistoricoException`
**Cuando** se inspecciona
**Entonces** extiende `Exception` (no `RuntimeException`)
**Y** su constructor obliga a `(String tiendaId, String skuVentaId, int semanasDisponibles)`
**Y** su `getMessage()` incluye los tres datos formateados

**Dado** el `pom.xml` del backend
**Cuando** se inspecciona la sección de PIT
**Entonces** está configurado con `targetClasses=mx.com.officedepot.insumos.forecasting.*` (preparado para gate de 2.11)
**Y** el módulo `forecasting/` es alcanzable como path independiente para PIT scope

---

### Story 2.2: Forecast baseline semanal — media móvil ponderada y cold-start fail-loud

Como **el motor de pronóstico**,
quiero **calcular la demanda esperada semanal por tienda × SKU venta usando media móvil ponderada sobre el histórico de ventas, y marcar fail-loud las series con menos de 12 meses de histórico**,
para **producir una predicción baseline robusta y nunca inventar un número para tiendas nuevas**.

**Acceptance Criteria:**

**Dado** la implementación `MediaMovilPonderadaEngine implements ForecastingEngine`
**Cuando** recibe una `SerieTemporal` con ≥52 puntos semanales
**Entonces** aplica media móvil con ventana de 13 semanas y pesos decrecientes exponencialmente (peso mayor a semanas recientes)
**Y** aplica clipping de outliers a ±3σ sobre la ventana
**Y** devuelve `ResultadoForecast` con `demandaEsperada` como `BigDecimal(scale=4)` y `trazo.ventanaHistoricoUsada` describiendo las semanas y pesos

**Dado** una `SerieTemporal` con `puntos.size() < 52`
**Cuando** se llama `predecir`
**Entonces** lanza `CapacidadInsuficienteHistoricoException(tiendaId, skuVentaId, semanasDisponibles)`
**Y** la excepción NO aborta toda la tienda — solo se propaga al orquestador (2.10) que la captura y marca ese par tienda × SKU como "sin sugerencia"

**Dado** la decisión técnica de librería (Smile vs Apache Commons Math)
**Cuando** se inspecciona el repo
**Entonces** existe un ADR en `docs/adr/ADR-001-libreria-forecasting.md` que registra la elección con su justificación
**Y** la dependencia está pinneada a versión exacta en `pom.xml`

**Dado** un golden fixture con serie sintética de 52 semanas de ventas constantes (100 unidades/semana)
**Cuando** se ejecuta `predecir`
**Entonces** la demanda esperada devuelve `100 ± 1%`
**Y** el test queda en `src/test/java/.../forecasting/MediaMovilPonderadaEngineTest.java`

**Dado** un golden fixture con serie de tendencia ascendente lineal (de 50 a 150 en 52 semanas)
**Cuando** se ejecuta `predecir`
**Entonces** la demanda esperada se sitúa por encima del promedio histórico (>100) — el modelo captura la tendencia reciente

**Dado** el `TrazoForecast` emitido
**Cuando** se inspecciona el JSON serializado
**Entonces** contiene `ventanaHistoricoUsada` (rango semanas), `pesosAplicados` (array de pesos), `demandaBaseline`, `outliersClippeados` (contador)
**Y** NO contiene aún componente estacional ni factor de pérdida (esos se agregan en 2.4 y 2.6)

---

### Story 2.3: Backtesting suite y golden fixtures firmados

Como **el equipo de producto**,
quiero **una suite de backtesting que mida MAPE/WAPE del motor sobre histórico real y se firme por Hugo (comprador piloto)**,
para **que el criterio de calidad del motor sea verificable, no retórico, y bloquee merges que degraden el modelo**.

**Acceptance Criteria:**

**Dado** la suite `BacktestingSuite.java` en `src/test/java/.../forecasting/`
**Cuando** se ejecuta `mvn test -Dtest=BacktestingSuite`
**Entonces** carga datos reales desde `src/test/resources/fixtures/backtesting/historico-2024.csv` (subset anonimizado del histórico real)
**Y** trunca a `ventas hasta 2024-12-31` como input, `2025-Q1` como ground truth
**Y** corre `MediaMovilPonderadaEngine.predecir` para cada tienda × SKU
**Y** calcula `MAPE = mean(|predicho − real| / real × 100)` y `WAPE = Σ|predicho − real| / Σ|real| × 100` agregados y por tienda × SKU

**Dado** el reporte de backtesting
**Cuando** termina la ejecución
**Entonces** escribe `target/backtesting/report-{timestamp}.csv` con columnas `tienda_id`, `sku_venta_id`, `demanda_predicha`, `demanda_realizada`, `error_absoluto`, `error_porcentual`
**Y** escribe un resumen `target/backtesting/summary-{timestamp}.json` con WAPE global, MAPE global, percentiles 50/75/90/99 del error porcentual

**Dado** el `DECISION_LOG.md` del golden v1
**Cuando** se inspecciona
**Entonces** contiene el threshold acordado con Hugo en formato literal (ej. "WAPE global ≤ 25% en histórico 2024 → 2025_Q1, acordado 2026-MM-DD, firmado por Hugo Sánchez, comprador piloto")
**Y** está firmado con commit de Hugo en el repo (o, si no tiene acceso, con check-off explícito que Jonathan dejó en el archivo)

**Dado** un cambio al motor que degrada WAPE por encima del threshold
**Cuando** corre el pipeline en Cloud Build
**Entonces** `BacktestingSuite` falla con mensaje "WAPE actual X% > threshold Y% acordado con Hugo en v1/DECISION_LOG.md"
**Y** el merge a `main` queda bloqueado hasta que (a) se restaure WAPE bajo threshold, o (b) Hugo firme un nuevo threshold en `v2/DECISION_LOG.md`

**Dado** el directorio `src/test/resources/fixtures/golden/v1/`
**Cuando** se inspecciona
**Entonces** contiene `EXPECTED_OUTPUT.csv` (sugerencias esperadas para una muestra de 20 tiendas × 50 SKUs), `DECISION_LOG.md` con la firma de Hugo, `INPUT_DATA/` con los CSVs de entrada del golden
**Y** el directorio `v1/` es inmutable: un test de integridad verifica que el hash SHA256 de cada archivo coincide con el documentado en `MANIFEST.sha256`
**Y** cualquier cambio a la sugerencia esperada se hace en `v2/`, nunca editando `v1/`

---

### Story 2.4: Componente estacional y eventos comerciales

Como **el motor de pronóstico**,
quiero **aplicar componente estacional basado en patrones YoY del histórico y multiplicar por el factor del evento comercial cuando la ventana cae dentro de uno**,
para **que la predicción refleje la realidad estacional (Buen Fin, BTS, Hot Sale) y no solo el promedio plano**.

**Acceptance Criteria:**

**Dado** el servicio `ComponenteEstacional` en `forecasting/`
**Cuando** recibe `(serie_historica, ventana_pronostico)` con histórico ≥24 meses (2 años YoY)
**Entonces** identifica semanas comparables YoY (mismo número ISO en años previos)
**Y** calcula `ratio = venta_semana_X / promedio_anual` para cada año disponible
**Y** devuelve `factor_estacional` = promedio de ratios

**Dado** el servicio `ComponenteEstacional`
**Cuando** recibe histórico con menos de 24 meses
**Entonces** devuelve `factor_estacional = 1.0` (neutral)
**Y** loguea warning "componente estacional neutralizado por histórico insuficiente"

**Dado** la consulta a `mart.evento_comercial`
**Cuando** la `ventana_pronostico` cae total o parcialmente dentro de un evento activo
**Entonces** se aplica `factor_evento = evento_comercial.factor_multiplicador`
**Y** si múltiples eventos solapan, se toma el **MAYOR** factor entre ellos (no producto, para no inflar artificialmente)

**Dado** la fórmula de ajuste estacional
**Cuando** se compone con el baseline (2.2)
**Entonces** `demanda_estacional = demanda_baseline × factor_estacional × factor_evento`
**Y** el trazo se enriquece con `factor_estacional`, `factor_evento`, `evento_aplicado` (nombre del evento o `null`)

**Dado** un golden fixture con serie que muestra pico de 2x en semana ISO 47 en 3 años YoY
**Cuando** se predice para semana 47 del año siguiente
**Entonces** el componente estacional devuelve `~2.0`
**Y** si además hay evento "Buen Fin" con factor 1.5 en esa semana, el `factor_evento` aplicado es 1.5

**Dado** un test de regresión con ventana sin eventos
**Cuando** se predice
**Entonces** `factor_evento = 1.0` y el trazo lo declara explícitamente
**Y** la demanda final es igual a `demanda_baseline × factor_estacional`

---

### Story 2.5: Factor de pérdida jerárquico — tabla y resolver (sin UI, solo motor)

Como **el motor de pronóstico**,
quiero **resolver el factor de pérdida por SKU vía la jerarquía `factor_por_tipo_papel → default_global (0.9%)`**,
para **que cada tipo de papel pueda tener su propio % de pérdida sin requerir configuración por SKU individual, con un fallback razonable**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 2.5
**Cuando** se aplica
**Entonces** existe `mart.factor_perdida_tipo_papel` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `tipo_papel_id BIGINT NOT NULL REFERENCES mart.tipo_papel(id)`, `factor_porcentual NUMERIC(8,4) NOT NULL CHECK (factor_porcentual >= 0 AND factor_porcentual <= 1)`, `vigencia_desde DATE NOT NULL`, `vigencia_hasta DATE NULL`, `actualizado_por VARCHAR NOT NULL`, `razon_estructurada VARCHAR`, `razon_libre TEXT`
**Y** existe `mart.parametro_sistema` con `clave VARCHAR PK`, `valor VARCHAR NOT NULL`, `tipo VARCHAR CHECK (tipo IN ('DECIMAL','INTEGER','STRING','BOOLEAN'))`, `actualizado_por VARCHAR NOT NULL`, `actualizado_en TIMESTAMPTZ NOT NULL DEFAULT now()`
**Y** la migración hace seed: `('forecast.factor_perdida.default_global', '0.009', 'DECIMAL', 'system')`

**Dado** el servicio `ResolverFactorPerdida` en `forecasting/`
**Cuando** se invoca `resolver(skuInsumoCodigo, fechaCiclo)`
**Entonces** busca el `tipo_papel_id` del SKU (vía `sku_insumo`), busca factor vigente en `factor_perdida_tipo_papel` (`vigencia_desde <= fechaCiclo AND (vigencia_hasta IS NULL OR vigencia_hasta >= fechaCiclo)`)
**Y** si existe, devuelve `(factor_porcentual, nivel_origen=TIPO_PAPEL, tipo_papel_id)`
**Y** si no existe, devuelve `(default_global, nivel_origen=DEFAULT_GLOBAL, tipo_papel_id)` y loguea fallback

**Dado** que `ResolverFactorPerdida` vive en `forecasting/` (núcleo puro)
**Cuando** se inspecciona
**Entonces** no depende de Spring ni JPA directamente — recibe los datos de factor pre-cargados por un adaptador (port en sentido hexagonal)
**Y** existe la implementación adaptador en `persistencia/` que consulta la BD y los inyecta al servicio

**Dado** el endpoint mínimo de admin `PUT /api/v1/admin/factor-perdida/tipo-papel/{tipoPapelId}`
**Cuando** un administrador envía `{factor_porcentual, razon_estructurada, razon_libre}`
**Entonces** cierra la vigencia anterior (`vigencia_hasta = ayer`), inserta nueva vigencia, registra en bitácora con `valor_anterior` y `valor_nuevo`
**Y** la UI completa con la lógica de configuración la entrega E4 (FR-082); este endpoint es el plumbing mínimo

**Dado** un test del resolver
**Cuando** SKU "PAPEL-001" tiene `tipo_papel=opalina` con factor 0.012 vigente
**Entonces** `resolver("PAPEL-001", 2026-06-01)` devuelve `(0.012, TIPO_PAPEL, tipo_papel_id_opalina)`

**Dado** un test del resolver
**Cuando** SKU "PAPEL-002" tiene `tipo_papel=carbon` sin factor configurado
**Entonces** `resolver("PAPEL-002", 2026-06-01)` devuelve `(0.009, DEFAULT_GLOBAL, tipo_papel_id_carbon)`

---

### Story 2.6: Conversión venta → insumo y exclusión fail-loud sin equivalencia

Como **el motor de pronóstico**,
quiero **aplicar el factor de pérdida sobre la demanda de venta, convertirla a demanda de insumo vía la tabla de equivalencia, y abortar fail-loud el ciclo de la tienda completa si encuentro un SKU venta sin equivalencia**,
para **que la conversión sea trazable conforme a AR-F05 y nunca produzca insumo "inferido" a partir de venta no mapeada**.

**Acceptance Criteria:**

**Dado** el servicio `ConvertirVentaAInsumo`
**Cuando** recibe `(tiendaId, skuVentaId, demandaVenta, factorPerdida, nivelOrigenFactor)`
**Entonces** calcula `demanda_ajustada = demandaVenta × (1 + factorPerdida)` ANTES de mapear (AR-F05)
**Y** consulta `mart.equivalencia_venta_insumo` por `sku_venta_id`
**Y** si existe equivalencia activa, devuelve `(skuInsumoCodigo, demandaInsumoPiezas=demanda_ajustada)`
**Y** si no existe, lanza `EquivalenciaNoDefinidaException(tiendaId, skuVentaId)`

**Dado** la `EquivalenciaNoDefinidaException` lanzada en el contexto del orquestador batch
**Cuando** el orquestador la captura (2.10)
**Entonces** **aborta TODO el ciclo de esa tienda** (todos los SKUs de la tienda quedan sin sugerencia)
**Y** persiste en bitácora `entidad_tipo='aborto_ciclo_tienda'` con `tienda_id`, `causa='EquivalenciaNoDefinidaException'`, `sku_venta_culprit=skuVentaId`
**Y** continúa procesando las demás tiendas

**Dado** un test de la fórmula AR-F05
**Cuando** `demandaVenta=100`, `factorPerdida=0.01`
**Entonces** `demanda_ajustada = 100 × 1.01 = 101`
**Y** el orden importa: si el factor se aplicara DESPUÉS de la conversión, el resultado sería diferente cuando la equivalencia no es 1:1; el test verifica el orden ANTES

**Dado** un test de exclusión fail-loud
**Cuando** se procesa una serie con `sku_venta_id="99999"` sin equivalencia
**Entonces** se lanza `EquivalenciaNoDefinidaException` con mensaje conteniendo `"tienda="` y `"sku_venta=99999"`
**Y** no se persiste ninguna sugerencia para la tienda afectada

**Dado** el trazo emitido
**Cuando** se inspecciona tras conversión exitosa
**Entonces** incluye `demanda_venta_original`, `factor_perdida_aplicado`, `nivel_origen_factor`, `demanda_ajustada`, `sku_insumo_resuelto`

---

### Story 2.7: Cuantización al empaque mínimo (property-based)

Como **el motor de pronóstico**,
quiero **cuantizar la demanda de insumo al múltiplo del empaque mínimo del proveedor, nunca redondeando hacia abajo si hay demanda**,
para **que las cantidades sugeridas sean realmente solicitables y no se generen pedidos imposibles de cumplir**.

**Acceptance Criteria:**

**Dado** el servicio `CuantizarAEmpaqueMinimo` en `forecasting/`
**Cuando** recibe `(contenido, demandaPiezas)` con `demandaPiezas > 0`
**Entonces** devuelve `ceil(demandaPiezas / contenido) × contenido`

**Dado** el mismo servicio
**Cuando** recibe `demandaPiezas <= 0`
**Entonces** devuelve `0`

**Dado** jqwik configurado con `@Property`
**Cuando** se ejecuta la propiedad `cantidadEsMultiploDeContenido`
**Entonces** para todo `(demanda > 0, contenido > 0)` generado aleatoriamente, `output % contenido == 0`

**Dado** jqwik con `@Property cantidadNoRedondeaAbajo`
**Cuando** se ejecuta
**Entonces** para todo `(demanda > 0, contenido > 0)`, `output >= ceil(demanda / contenido) * contenido`
**Y** equivalentemente, `output * contenido >= demanda` siempre

**Dado** jqwik con `@Property demandaCeroProduceCantidadCero`
**Cuando** se ejecuta con `demanda = 0`
**Entonces** `output == 0`

**Dado** jqwik con `@Property demandaPositivaMinimaProduceUnPaquete`
**Cuando** `demanda = 1` y `contenido = 100`
**Entonces** `output = 100` (nunca redondea a 0 si hay demanda)

**Dado** el trazo emitido tras cuantización
**Cuando** se inspecciona
**Entonces** incluye `demanda_insumo_piezas`, `contenido_empaque`, `cantidad_cuantizada_piezas`, `cantidad_paquetes = cantidad / contenido`

---

### Story 2.8: Sugerencia final, descuento SIM/ALLOC/TFS y persistencia del trazo

Como **el motor de pronóstico**,
quiero **aplicar la fórmula completa de sugerencia descontando saldo SIM, ALLOC pendiente y TFS entrante, y persistir el resultado con su trazo JSONB para consumo en E3**,
para **que la cantidad sugerida refleje la necesidad real ajustada por inventario y tránsito**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 2.8
**Cuando** se aplica
**Entonces** existe `mart.sugerencia_pedido` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `ciclo_id VARCHAR NOT NULL`, `tienda_id VARCHAR NOT NULL REFERENCES mart.tienda(id)`, `sku_insumo VARCHAR NOT NULL REFERENCES mart.sku_insumo(codigo)`, `cantidad_sugerida BIGINT NOT NULL CHECK (cantidad_sugerida >= 0)`, `costo_unitario NUMERIC(12,4) NOT NULL`, `costo_total NUMERIC(12,4) NOT NULL`, `trazo JSONB NOT NULL`, `generada_en TIMESTAMPTZ NOT NULL DEFAULT now()`, `UNIQUE(ciclo_id, tienda_id, sku_insumo)`
**Y** la migración inserta `('forecast.buffer_seguridad_semanas', '1', 'INTEGER', 'system')` en `parametro_sistema`

**Dado** el servicio `CalcularSugerencia`
**Cuando** se invoca para una tienda × SKU
**Entonces** calcula `demanda_quincenal = demanda_semana_1 + demanda_semana_2` del ciclo
**Y** `buffer_seguridad = cuantizar(demanda_promedio_semanal × buffer_seguridad_semanas)` (parametro_sistema)
**Y** consulta `saldo_SIM` desde `staging.snapshot_sim` más reciente para `(tienda_id, sku_insumo)`
**Y** consulta `ALLOC_pendiente = Σ CTD_PENDIENTE` desde `staging.alloc` con `STATUS != 'completado'` para `(tienda_id, sku_insumo)`
**Y** consulta `TFS_entrante = Σ QTY_TRANSFERRED` desde `staging.tfs` con `TO_LOC = tienda_id, sku_insumo`, `CREATE_DATE` o entrega futura

**Dado** los componentes calculados
**Cuando** se aplica la fórmula
**Entonces** `cantidad_neta = demanda_quincenal + buffer − SIM − ALLOC − TFS`
**Y** `cantidad_final = cuantizar(max(0, cantidad_neta))`
**Y** si `cantidad_final == 0`, NO se persiste fila en `mart.sugerencia_pedido` (no se generan filas con cantidad cero)
**Y** si `cantidad_final > 0`, se persiste con `costo_total = cantidad_final × sku_insumo.costo`

**Dado** el trazo JSONB persistido
**Cuando** se inspecciona
**Entonces** contiene la estructura completa: `{ historico: { ventana_semanas, pesos, demanda_baseline }, estacionalidad: { factor_estacional, factor_evento, evento_aplicado }, perdida: { factor, nivel_origen }, conversion: { sku_venta_id, sku_insumo_codigo, demanda_ajustada_piezas }, cuantizacion: { contenido_empaque, demanda_piezas, cantidad_cuantizada }, descuentos: { sim, alloc, tfs }, buffer: { semanas, piezas }, resultado: { cantidad_final, costo_unitario, costo_total } }`

**Dado** un test con datos sintéticos
**Cuando** tienda=`T1`, SKU=`S1` con `demanda_quincenal=200 piezas`, `buffer=100`, `SIM=50`, `ALLOC=30`, `TFS=20`, `contenido=100`, `costo=15.00`
**Entonces** `cantidad_neta = 200+100-50-30-20 = 200` → cuantizado `200` → 2 paquetes
**Y** la fila persistida tiene `cantidad_sugerida=200`, `costo_total=3000.00`

**Dado** un test de borde
**Cuando** `SIM > demanda_quincenal + buffer`
**Entonces** `cantidad_neta < 0` → cuantizado `0` → NO se persiste fila

**Dado** la serialización JSON del trazo (vía Jackson)
**Cuando** se inspecciona el JSON resultante
**Entonces** los campos monetarios se serializan como string (`"15.00"`) preservando `BigDecimal` (AR-Money-JSON)
**Y** las cantidades como `long` literal (`200`)

---

### Story 2.9: Validación presupuestal, descuento de esporádicas y recorte sugerido por riesgo de quiebre

Como **el motor de pronóstico**,
quiero **calcular el costo total por tienda restándole las compras esporádicas del periodo y, si excede presupuesto, sugerir un recorte priorizando reducir SKUs con menor riesgo de quiebre**,
para **que la sugerencia respete el techo presupuestal y el coordinador reciba un punto de partida razonable para decidir**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 2.9
**Cuando** se aplica
**Entonces** existe `mart.validacion_presupuesto_tienda` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `ciclo_id VARCHAR NOT NULL`, `tienda_id VARCHAR NOT NULL REFERENCES mart.tienda(id)`, `costo_total_pre_recorte NUMERIC(14,4) NOT NULL`, `techo_quincenal NUMERIC(14,4) NOT NULL`, `esporadicas_periodo NUMERIC(14,4) NOT NULL`, `presupuesto_disponible NUMERIC(14,4) NOT NULL`, `estado VARCHAR CHECK (estado IN ('EN_RANGO','CON_RECORTE','IMPOSIBLE_AJUSTAR')) NOT NULL`, `recorte_total NUMERIC(14,4)`, `generada_en TIMESTAMPTZ NOT NULL DEFAULT now()`, `UNIQUE(ciclo_id, tienda_id)`
**Y** existe la columna `recorte_propuesto JSONB` añadida a `mart.sugerencia_pedido`

**Dado** el servicio `ValidarPresupuesto` por tienda × ciclo
**Cuando** se invoca
**Entonces** calcula `costo_total = Σ(cantidad_sugerida × costo_unitario)` desde `mart.sugerencia_pedido`
**Y** calcula `esporadicas_periodo = Σ TOTAL_COST` desde `staging.entrega_directa` con `TRAN_CODE=20` (Purchases), `tienda_id`, `TRAN_DATE` dentro del periodo del ciclo
**Y** calcula `techo_quincenal = tienda.techo_presupuestal_semanal × 2`
**Y** `presupuesto_disponible = techo_quincenal − esporadicas_periodo`

**Dado** que `costo_total <= presupuesto_disponible`
**Cuando** termina la validación
**Entonces** persiste `estado='EN_RANGO'`, `recorte_total=0`
**Y** no modifica ninguna fila de `mart.sugerencia_pedido.recorte_propuesto`

**Dado** que `costo_total > presupuesto_disponible`
**Cuando** se ejecuta el algoritmo de recorte
**Entonces** calcula `cobertura_semanas[sku] = cantidad_sugerida / demanda_semanal_promedio` para cada SKU de la tienda
**Y** ordena SKUs por `cobertura_semanas` descendente (los más holgados primero)
**Y** itera recortando 1 empaque (`contenido` piezas) por SKU
**Y** antes de recortar valida `cobertura_post_recorte >= 2 semanas`; si no se cumple, salta al siguiente SKU
**Y** continúa hasta `costo_total <= presupuesto_disponible` o agotar SKUs recortables

**Dado** que el algoritmo recorta exitosamente
**Cuando** termina
**Entonces** persiste `estado='CON_RECORTE'`, `recorte_total = costo_total_original − costo_total_post_recorte`
**Y** para cada SKU recortado actualiza `sugerencia_pedido.recorte_propuesto = { cantidad_original, cantidad_recortada, motivo: "Ajuste presupuestal — cobertura holgada" }`
**Y** las filas de `sugerencia_pedido` NO se modifican en su `cantidad_sugerida` (el recorte es una sugerencia, no una mutación; el coordinador acepta en E3)

**Dado** que el algoritmo no puede llegar al presupuesto sin violar la regla de 2 semanas
**Cuando** se termina la iteración
**Entonces** persiste `estado='IMPOSIBLE_AJUSTAR'`, `recorte_total = recorte_máximo_alcanzable`
**Y** lanza `RecorteImposibleException` (checked) capturada por el orquestador, que registra en bitácora con `entidad_tipo='recorte_imposible'`, `tienda_id`, `gap_presupuestal`

**Dado** jqwik con `@Property recorteRespetaPresupuesto`
**Cuando** se ejecuta sobre escenarios generados aleatoriamente CON_RECORTE
**Entonces** para todo escenario, `Σ(costo × cantidad_post_recorte) <= presupuesto_disponible`

**Dado** jqwik con `@Property recorteRespetaCoberturaMinima`
**Cuando** se ejecuta
**Entonces** para todo SKU recortado, `cobertura_post_recorte >= 2`

**Dado** un test con escenario concreto
**Cuando** costo_total=100,000, techo_semanal=40,000 (×2=80,000), esporadicas=20,000 → presupuesto=60,000 → recorte_necesario=40,000
**Entonces** el algoritmo recorta empezando por SKUs con cobertura 5+ semanas, deja de recortar cuando llega a 60,000 o no puede más sin violar la regla
**Y** los SKUs con cobertura 2.5 semanas NO se tocan

---

### Story 2.10: Orquestador batch y endpoint trigger manual

Como **administrador**,
quiero **disparar manualmente la ejecución del motor de pronóstico para un ciclo dado, procesando el universo completo (~8,600 series) en menos de 5 minutos**,
para **regenerar las sugerencias cuando los datos de entrada cambien o cuando el ciclo lo requiera**.

**Acceptance Criteria:**

**Dado** el servicio `ForecastBatchOrquestador`
**Cuando** se invoca `ejecutar(cicloId)`
**Entonces** carga el universo de tiendas activas con sus SKUs venta de histórico ≥1 semana
**Y** para cada tienda crea un task que ejecuta el pipeline completo: forecast (2.2) → estacional (2.4) → factor (2.5) → conversión (2.6) → cuantización (2.7) → sugerencia (2.8) → validación + recorte (2.9)
**Y** los tasks se ejecutan en `ForkJoinPool` con `parallelism = Runtime.getRuntime().availableProcessors() − 1`

**Dado** que durante el procesamiento de una tienda se lanza `EquivalenciaNoDefinidaException` o `CapacidadInsuficienteHistoricoException` o `RecorteImposibleException`
**Cuando** el task la captura
**Entonces** persiste la causa en bitácora con `entidad_tipo='aborto_ciclo_tienda'` o `'serie_sin_sugerencia'` (granularidad según la excepción)
**Y** el task termina sin propagar la excepción al pool
**Y** la siguiente tienda continúa procesándose sin interrupción

**Dado** que se re-ejecuta el mismo `ciclo_id`
**Cuando** el orquestador arranca
**Entonces** primero borra `mart.sugerencia_pedido WHERE ciclo_id = X` y `mart.validacion_presupuesto_tienda WHERE ciclo_id = X` en una transacción
**Y** persiste en bitácora `entidad_tipo='reemplazo_ciclo'`, `actor`, `ciclo_id`, `total_sugerencias_borradas`
**Y** luego procesa desde cero

**Dado** el endpoint `POST /api/v1/admin/forecast/ejecutar`
**Cuando** un administrador autenticado envía `{ ciclo_id: "2026-W22" }`
**Entonces** valida que el `ciclo_id` no tiene un job EN_COLA o EJECUTANDO
**Y** crea registro `mart.forecast_job` con `id UUID`, `ciclo_id`, `estado='EN_COLA'`, `disparado_por`, `disparado_en`
**Y** dispara el orquestador asíncronamente
**Y** devuelve `202 Accepted` con header `Location: /api/v1/admin/forecast/jobs/{jobId}` y body `{ job_id, estado: "EN_COLA" }`

**Dado** el endpoint `GET /api/v1/admin/forecast/jobs/{jobId}`
**Cuando** un administrador lo consulta
**Entonces** devuelve estado actual: `EN_COLA | EJECUTANDO | COMPLETADO | FALLIDO`
**Y** durante EJECUTANDO incluye métricas en tiempo real: `tiendas_procesadas`, `tiendas_total`, `errores_por_tipo: { sin_equivalencia: N, sin_historico: M, recorte_imposible: K }`, `duracion_ms_actual`
**Y** tras COMPLETADO incluye `sugerencias_persistidas`, `tiendas_abortadas`, `duracion_ms_total`

**Dado** la métrica Micrometer `forecast.batch.duration_seconds` tag `{ciclo_id, estado}`
**Cuando** completa una ejecución
**Entonces** la métrica registra el tiempo total
**Y** se expone vía `/actuator/metrics` y a Cloud Monitoring

**Dado** un test de integración con universo simulado de ~8,600 series (puede ser generado sintéticamente)
**Cuando** se ejecuta `ForecastBatchOrquestador.ejecutar`
**Entonces** completa en menos de 5 minutos en una JVM single-instance
**Y** el test falla si supera el SLA (NFR-E-3)

**Dado** el endpoint `POST /api/v1/admin/forecast/ejecutar`
**Cuando** un usuario con rol `COORDINADOR` intenta llamarlo
**Entonces** devuelve `403 Forbidden` con `ProblemDetail`
**Y** ningún job se crea

---

### Story 2.11: Gate de calidad — PIT ≥80%, jqwik en CI y contrato YAML para E3

Como **el equipo de ingeniería**,
quiero **PIT mutation testing ≥80% en `forecasting.*`, jqwik property-based con las 4 invariantes obligatorias y el contrato Spring Cloud Contract YAML del endpoint que E3 consumirá**,
para **que el motor no pueda regresar a `main` sin demostrar calidad medible, y E3 pueda empezar contra stubs antes de que el backend implemente el endpoint**.

**Acceptance Criteria:**

**Dado** el `pom.xml` con `pitest-maven` plugin
**Cuando** se inspecciona la configuración
**Entonces** `targetClasses=mx.com.officedepot.insumos.forecasting.*`
**Y** `targetTests=mx.com.officedepot.insumos.forecasting.*Test`
**Y** `mutationThreshold=80` (el build falla si <80% mutantes eliminados)
**Y** `excludedClasses` lista cualquier POJO/record obvio si es necesario, documentado en `pom.xml`

**Dado** el paso "PIT gate" en `cloudbuild.yaml`
**Cuando** se ejecuta el pipeline
**Entonces** corre `mvn org.pitest:pitest-maven:mutationCoverage`
**Y** el resultado HTML queda como artifact del build
**Y** si `mutationThreshold` no se alcanza, el paso falla y el pipeline aborta

**Dado** la configuración de jqwik
**Cuando** se ejecuta `mvn test`
**Entonces** las 4 propiedades obligatorias corren con al menos 1000 casos generados cada una:
**Y** `cantidadEsMultiploDeContenido` (2.7)
**Y** `cantidadNoRedondeaAbajoNiACero` (2.7)
**Y** `recorteRespetaPresupuesto` (2.9)
**Y** `skuSinEquivalenciaNoApareceEnOutput` (2.6 — generador sintetiza ventas con un sku_venta sin equivalencia y verifica que el output no lo incluye)

**Dado** el archivo de contrato `backend/src/test/resources/contracts/forecast-sugerencia-v1.yaml`
**Cuando** se inspecciona
**Entonces** describe el endpoint `GET /api/v1/sugerencias/{ciclo_id}` con query `tienda_id` opcional
**Y** la respuesta `200 OK` tiene esquema: array de `{ tienda_id, sku_insumo, descripcion_sku, presentacion, contenido, cantidad_sugerida (long), costo_unitario (string), costo_total (string), recorte_propuesto: { cantidad_original, cantidad_recortada, motivo } | null, trazo: { ... estructura completa ... } }`
**Y** la respuesta `404` cuando `ciclo_id` no existe
**Y** la respuesta `403` cuando el JWT no tiene acceso al `tienda_id` (scoping territorial de 1.3)

**Dado** el plugin Spring Cloud Contract
**Cuando** se ejecuta `mvn install`
**Entonces** se generan stubs publicados localmente bajo `groupId.artifactId-stubs`
**Y** el frontend (E3) puede consumir esos stubs en modo `WireMock` para desarrollo offline sin backend

**Dado** la serialización JSON del trazo y montos
**Cuando** un cliente consume el endpoint vía stub
**Entonces** los `costo_unitario` y `costo_total` aparecen como strings decimales (`"15.00"`, `"3000.00"`) — AR-Money-JSON
**Y** las cantidades como `long` literal (`200`)

**Dado** el archivo `backend/CONTRACTS.md`
**Cuando** un desarrollador lo lee
**Entonces** explica cómo regenerar stubs (`mvn install`), dónde se publican, cómo consumirlos desde Angular (`WireMock` o `@AutoConfigureStubRunner`)
**Y** establece que cambios al YAML rompen el build hasta regenerar — el contrato es fuente de verdad

**Dado** el paso "Contract verify" en `cloudbuild.yaml`
**Cuando** se ejecuta el pipeline
**Entonces** corre los tests generados de Spring Cloud Contract que verifican que la implementación del controller respeta el contrato
**Y** si la implementación diverge del YAML, el paso falla con mensaje claro de qué campo/status diverge

---

## Épica 3: Bandeja de consolidación distrital del coordinador

**Goal:** El coordinador abre su bandeja semanal en pantalla, ve las ~38 tiendas en ciclo con sus sugerencias y trazo, valida el presupuesto con desglose `techo − esporádicas` y recorte sugerido, aplica overrides con razón estructurada, consulta agregados del distrito y el dashboard pedido-vs-recibido, aprueba el consolidado y exporta el Excel compatible al comprador por correo. Cada decisión queda en bitácora inmutable. Entrega el objetivo O1 — el ahorro de 9h → <2h por coordinador por semana.

**FRs cubiertos:** FR-041, FR-052, FR-053, FR-060, FR-061, FR-063, FR-064, FR-065, FR-066, FR-067, FR-100, FR-101, FR-102, FR-104, FR-110, FR-111, FR-112.

### Story 3.1: Shell de bandeja distrital y endpoint base de sugerencias

Como **coordinador**,
quiero **abrir mi bandeja distrital y ver de inmediato las ~38 tiendas con ciclo activo y sus sugerencias del motor**,
para **arrancar mi sesión semanal con todo el contexto operativo sin armarlo a mano**.

**Acceptance Criteria:**

**Dado** un coordinador autenticado (de 1.3)
**Cuando** navega a `/bandeja`
**Entonces** Angular Router resuelve el guard `COORDINADOR` y carga el módulo standalone de bandeja
**Y** un usuario con rol `ADMINISTRADOR` también puede entrar (modo coordinador con override según NFR-S-2)
**Y** cualquier otro rol o ausencia de sesión redirige a login o a `/forbidden`

**Dado** la pantalla de bandeja cargada
**Cuando** se renderiza por primera vez
**Entonces** el header muestra `Coordinador: {nombre}`, `Distrito: {distrito}`, `Ciclo activo: Sem {N} · {rango_fechas} ISO 8601`
**Y** el layout es responsivo desktop 1366×768 mínimo con navegación lateral colapsable

**Dado** el endpoint `GET /api/v1/sugerencias?ciclo_id={cicloId}`
**Cuando** el frontend lo consume sin pasar `coordinador_id`
**Entonces** el backend deriva `coordinador_id` desde el JWT (scoping territorial de 1.3)
**Y** devuelve solo las tiendas del distrito del coordinador con sus sugerencias del ciclo activo
**Y** intentar pasar `coordinador_id` distinto en query es ignorado (no escala scope)

**Dado** la respuesta del endpoint para un distrito con ~38 tiendas y ~10 SKUs típicos por tienda
**Cuando** el frontend la recibe
**Entonces** se mapea de `models/api/SugerenciaApi` a `models/view/SugerenciaView` con mapper explícito (sin reflection, sin `any`)
**Y** el primer render del grid principal completa en menos de 3 segundos (NFR-E-4) medido por `PerformanceObserver` en cliente

**Dado** que 2.11 publica stubs de Spring Cloud Contract de `forecast-sugerencia-v1.yaml`
**Cuando** el frontend corre en modo desarrollo
**Entonces** consume los stubs (vía `@AutoConfigureStubRunner` o `WireMock`) sin necesidad de backend corriendo
**Y** el contrato es la única fuente de verdad de los nombres/tipos de campos — divergencia rompe build

**Dado** la falla del backend (error 5xx) en el primer fetch
**Cuando** el frontend la recibe
**Entonces** el interceptor HTTP (de 1.4) muestra toast de sistema con código de incidente copiable
**Y** la bandeja muestra estado vacío con mensaje "No se pudo cargar la bandeja. Reintenta o contacta a soporte con el código {incidente}"

---

### Story 3.2: Pipes y formatters locale MX en `frontend/shared`

Como **frontend del producto**,
quiero **un conjunto de pipes Angular standalone que formateen cantidades, moneda, porcentajes y fechas conforme a las reglas locale MX no negociables**,
para **que ninguna pantalla reimplemente formato y todas hablen el mismo lenguaje al usuario**.

**Acceptance Criteria:**

**Dado** el módulo `frontend/shared/pipes/`
**Cuando** se inspecciona
**Entonces** existen los pipes Angular standalone: `CantidadDiscretaPipe`, `EquivalenciaPiezasACajasPipe`, `MonedaMXNPipe`, `PorcentajePipe`, `FechaSemanaIsoPipe`, `DiasHabilesPipe`
**Y** `LOCALE_ID='es-MX'` está configurado en `app.config.ts` con `registerLocaleData(localeEsMX)`

**Dado** `CantidadDiscretaPipe`
**Cuando** recibe `1250` con unidad `"cajas"`
**Entonces** devuelve `"1,250 cajas"` (separador de miles, unidad pegada con espacio simple)
**Y** con valor `0` devuelve `"0 cajas"`, nunca `"NaN"` ni vacío

**Dado** `EquivalenciaPiezasACajasPipe`
**Cuando** recibe `(piezas=18, contenido=6)`
**Entonces** devuelve `"18 piezas = 3 cajas"`
**Y** con `piezas=18, contenido=6, formato='compacto'` devuelve `"3 cajas"`

**Dado** `MonedaMXNPipe`
**Cuando** recibe `1250.5`
**Entonces** devuelve `"$1,250.50"`
**Y** con valores `>100000` activa modo abreviado: `$1.2M` con `title="$1,234,567.89"` para tooltip nativo
**Y** acepta segundo argumento `incluirSimboloMoneda=false` que omite el `$`

**Dado** `PorcentajePipe`
**Cuando** recibe `0.125`
**Entonces** devuelve `"+12.5%"` con color verde (clase CSS `pct-pos`)
**Y** con `−0.083` devuelve `"−8.3%"` usando **U+2212** (MINUS SIGN) **no guion ASCII**, color rojo (clase `pct-neg`)
**Y** con `0` devuelve `"0.0%"` color neutral

**Dado** `FechaSemanaIsoPipe`
**Cuando** recibe `2026-03-16` (lunes)
**Entonces** devuelve `"Sem 12 · 16–22 mar 2026"` (semana ISO 8601, lunes a domingo, mes abreviado en español)
**Y** acepta segundo argumento `formato='solo-semana'` que devuelve `"Sem 12"`

**Dado** `DiasHabilesPipe`
**Cuando** recibe `5`
**Entonces** devuelve `"5 días hábiles"` (no `"5 días naturales"`, NFR-L-6)

**Dado** tests unitarios por pipe en `frontend/shared/pipes/*.spec.ts`
**Cuando** se ejecuta `ng test`
**Entonces** cada pipe tiene tests de casos típicos, edge cases (cero, negativo, undefined, null) y locale
**Y** ningún pipe devuelve `"undefined"` o `"NaN"` ante input inválido — devuelve `""` o un placeholder documentado

---

### Story 3.3: Componente `TrazoForecastComponent` reusable

Como **coordinador**,
quiero **ver el trazo explicable de cada cantidad sugerida (histórico, estacional, factor, descuentos) con un clic o hover**,
para **decidir con confianza si acepto o ajusto el número del motor, alineado con UX-DR1**.

**Acceptance Criteria:**

**Dado** el componente standalone `TrazoForecastComponent`
**Cuando** se le pasa `[trazo]` (estructura JSONB de 2.8)
**Entonces** renderiza secciones colapsadas: "Histórico" (ventana, pesos, baseline), "Estacionalidad" (factor estacional, factor evento, evento aplicado), "Pérdida" (factor, nivel de origen `TIPO_PAPEL` o `DEFAULT_GLOBAL`), "Descuentos" (SIM, ALLOC, TFS con cifras), "Cuantización" (piezas → unidades comprables)
**Y** los valores monetarios y cantidades usan los pipes de 3.2

**Dado** el ícono `info` adyacente a cada celda de cantidad en el grid
**Cuando** el usuario lo hover con el mouse
**Entonces** abre un Material/CDK Overlay con el `TrazoForecastComponent` posicionado relativo al ícono
**Y** el overlay se cierra al `mouseleave` con delay de 200ms (para permitir mover el cursor al overlay)

**Dado** el ícono `info` con foco por teclado
**Cuando** el usuario presiona `Enter` o `Space`
**Entonces** abre el overlay sticky (no se cierra al perder hover)
**Y** `Esc` lo cierra; `Tab` mueve foco dentro del overlay

**Dado** un trazo con `nivel_origen=DEFAULT_GLOBAL`
**Cuando** se renderiza la sección "Pérdida"
**Entonces** muestra `"Factor 0.9% (default global)"` con badge ámbar `"Sin configuración por tipo de papel"`
**Y** con `nivel_origen=TIPO_PAPEL` muestra `"Factor 1.2% (opalina · configurado por Lupita 12-mar)"` con badge azul

**Dado** un test de accesibilidad con axe-core
**Cuando** se ejecuta sobre `TrazoForecastComponent`
**Entonces** no hay violaciones de contraste (NFR-A11Y-1)
**Y** todos los elementos interactivos tienen `aria-label`
**Y** `:focus-visible` con outline 2px en todos los elementos enfocables

**Dado** un trazo con secciones faltantes (ej. sin evento estacional aplicado)
**Cuando** se renderiza
**Entonces** la sección se muestra con valor neutro (`"Sin evento aplicado (factor 1.0)"`) — nunca se oculta para que el usuario sepa que el cálculo es completo

---

### Story 3.4: Grid principal de la bandeja — AG Grid con virtual scrolling

Como **coordinador**,
quiero **ver mis ~38 tiendas y sus SKUs sugeridos en una grilla performante con navegación por teclado completa**,
para **poder revisar el universo de mi distrito sin lag y operar como power user**.

**Acceptance Criteria:**

**Dado** la dependencia AG Grid Community en `package.json` pinneada
**Cuando** la bandeja carga
**Entonces** renderiza AG Grid con virtual scrolling activado (`rowBuffer=10`, virtualización vertical y horizontal)
**Y** filas = tiendas del distrito, columnas dinámicas = SKUs activos en al menos una tienda del distrito (no se muestran SKUs en cero universal)

**Dado** una celda con cantidad sugerida
**Cuando** se renderiza
**Entonces** muestra `cantidad_sugerida` formateada con `CantidadDiscretaPipe` ("18 piezas" o "3 cajas" según preferencia del coord — toggle en header), ícono `info` para el trazo (3.3), y badge de estado de recorte si aplica (`"Recorte sugerido"` ámbar)

**Dado** la separación de modelos AR-Grid
**Cuando** se inspecciona el código
**Entonces** existe `frontend/models/api/sugerencia.api.ts` (response cruda del backend) y `frontend/models/view/sugerencia.view.ts` (lo que el componente usa)
**Y** un mapper `sugerenciaApiToView()` los conecta — el componente nunca toca el modelo de API directamente
**Y** ninguna interfaz usa `any`

**Dado** un usuario navegando por teclado
**Cuando** presiona `Tab` desde el header
**Entonces** el foco llega a la primera celda del grid
**Y** flechas direccionales mueven entre celdas (AG Grid nativo)
**Y** `Enter` entra a modo edición (en historias 3.6+); por ahora abre el trazo (3.3)
**Y** `Esc` regresa foco al header
**Y** `Shift+Tab` reverso navega correctamente

**Dado** el filtro de búsqueda/búsqueda libre AG Grid
**Cuando** un usuario escribe `"opalina"`
**Entonces** se filtran columnas de SKU cuya descripción coincide
**Y** filtra filas por tienda escribiendo `"T-NORTE-28"` en input dedicado de tienda
**Y** filtros se acumulan (AND)

**Dado** axe-core ejecutado sobre el grid
**Cuando** se inspecciona
**Entonces** cada celda editable tiene `aria-label` contextualizado (de UX-DR8): `"Cantidad sugerida para Papel Opalina, tienda 28, semana 1, 18 cajas, editable"`
**Y** contraste 4.5:1 en texto, 3:1 en componentes interactivos (NFR-A11Y-1)

**Dado** un escenario con 50 tiendas × 15 SKUs (worst case del piloto)
**Cuando** se mide tiempo de scroll y respuesta
**Entonces** scroll mantiene 60fps en monitor 1366×768
**Y** primer render <3s (NFR-E-4)

---

### Story 3.5: Diferenciación visual sugerido vs override y bitácora en hover

Como **coordinador**,
quiero **distinguir a primera vista las cantidades que aún son sugeridas por el motor de las que ya tienen override humano, y ver la cadena completa de cambios al pasar el mouse**,
para **operar con auditoría visible cumpliendo UX-DR2 y FR-101**.

**Acceptance Criteria:**

**Dado** una celda con cantidad sugerida sin modificar
**Cuando** se renderiza
**Entonces** el número está en color gris `#666666`, fuente itálica, sin badge
**Y** el contraste contra el fondo cumple 4.5:1 (NFR-A11Y-1)

**Dado** una celda con override aplicado
**Cuando** se renderiza
**Entonces** el número está en color negro `#000000`, bold, con badge arriba a la derecha `"Modificado por {user} {hh:mm}"`
**Y** el badge tiene fondo distintivo (color azul claro `#E3F2FD`) y borde

**Dado** el endpoint `GET /api/v1/sugerencias/{id}/bitacora`
**Cuando** un coordinador lo consulta
**Entonces** devuelve la cadena cronológica de cambios de `mart.bitacora_evento` filtrada por `entidad_tipo='sugerencia_pedido' AND entidad_id={id}`
**Y** cada entrada incluye `timestamp`, `actor`, `valor_anterior`, `valor_nuevo`, `razon_estructurada`, `razon_libre`

**Dado** una fila con override
**Cuando** el usuario hover sobre cualquier celda de la fila
**Entonces** se muestra popup con cadena: `"Original modelo 180 → Lupita 07:48: 200 (ajuste_demanda_local) → Víctor 09:12: 195 (decision_negocio)"`
**Y** el popup se cierra al `mouseleave` con delay 200ms

**Dado** una fila sin overrides
**Cuando** se hover
**Entonces** no se muestra cadena de cambios (solo el trazo de 3.3 si es el ícono info)
**Y** el badge `Modificado por` está ausente

**Dado** una fila con override → modificación de override → vuelta al valor original
**Cuando** se inspecciona la cadena
**Entonces** todos los cambios aparecen secuencialmente, incluida la vuelta
**Y** la bitácora subyacente es inmutable (FR-104, AR-Bitacora) — cada cambio es nueva fila, no UPDATE

---

### Story 3.6: Override con razón estructurada y recálculo optimista

Como **coordinador**,
quiero **editar una cantidad sugerida con razón estructurada obligatoria, ver el recálculo inmediato (<100ms) y tener una ventana de 8 segundos para deshacer**,
para **operar fluidamente sin perder trazabilidad y sin modales bloqueantes**.

**Acceptance Criteria:**

**Dado** una celda editable en el grid
**Cuando** el coordinador entra a modo edición (`Enter` por teclado o doble-click)
**Entonces** el input acepta solo enteros positivos
**Y** valida que el valor sea múltiplo de `sku_insumo.contenido` — si no, muestra error inline `"La cantidad debe ser múltiplo de 6 (empaque mínimo). ¿Querías decir 18 o 24?"` (UX-DR6 inline + sugerencia)
**Y** `Esc` cancela sin enviar; `Enter` confirma y dispara modal de razón

**Dado** el modal de razón estructurada
**Cuando** se abre
**Entonces** muestra lista de razones predefinidas (`RazonOverrideEnum`): "Ajuste de demanda local", "Evento no capturado", "Inventario no reflejado en SIM", "Decisión de negocio", "Otro"
**Y** si selecciona "Otro" obliga a llenar campo de texto libre (min 5 caracteres)
**Y** si selecciona cualquier otra razón, el texto libre es opcional pero recomendado

**Dado** el coordinador confirma la razón
**Cuando** se cierra el modal
**Entonces** RxJS `debounceTime(150)` evita doble disparo
**Y** se envía `PATCH /api/v1/sugerencias/{id}` con `{ cantidad, razon_estructurada, razon_libre }`
**Y** el backend registra en bitácora con `valor_anterior`, `valor_nuevo`, `actor=usuarioId`, `razon_estructurada`, `razon_libre`
**Y** la celda actualiza su estilo a override (3.5) inmediatamente (optimistic update)

**Dado** el recálculo optimista en cliente
**Cuando** la edición confirma
**Entonces** se recalcula localmente: total de la fila (cantidad × costo), acumulado por proveedor en el panel de agregados (3.8), y restante de presupuesto disponible
**Y** todo el recálculo termina en <100ms medido por `performance.now()` (NFR-P-1)
**Y** si el servidor responde con cantidad diferente (por validación de pack), se reconcilia el valor con animación de "ajustado por servidor"

**Dado** un toast "Deshacer" tras la edición
**Cuando** aparece
**Entonces** se muestra con texto `"Cantidad modificada de 180 a 200 en {tienda} · {SKU}"` y botón `Deshacer`
**Y** permanece visible 8 segundos (UX-DR5)
**Y** click en `Deshacer` envía otro `PATCH` con valor original + `razon_estructurada='deshacer_cambio_previo'` (queda en bitácora — no se borra el original)

**Dado** un coordinador editando una celda
**Cuando** el servidor responde con error (red caída, 5xx)
**Entonces** la celda vuelve a su valor previo
**Y** se muestra toast de sistema con código de incidente (UX-DR6 nivel 3)
**Y** la bitácora del intento fallido NO se persiste server-side (la transacción se revierte)

---

### Story 3.7: Autosave silencioso e indicador tipo Google Docs

Como **coordinador**,
quiero **saber en todo momento que mi trabajo está guardado y nunca perderlo si algo se va mal**,
para **operar con tranquilidad sin pulsar "Guardar" manualmente, cumpliendo UX-DR4 y NFR-P-2**.

**Acceptance Criteria:**

**Dado** el header de la bandeja
**Cuando** se renderiza
**Entonces** muestra un indicador discreto a la derecha: `"Guardado hace 3 seg"` en estado normal
**Y** durante una operación de save: `"Guardando…"` con spinner pequeño
**Y** ante error: `"Error de guardado — reintentando en 5s"` en color ámbar

**Dado** una sugerencia con campo `actualizada_en TIMESTAMPTZ` en `mart.sugerencia_pedido`
**Cuando** se actualiza vía PATCH
**Entonces** `actualizada_en = now()` server-side
**Y** la respuesta del PATCH incluye `actualizada_en` para que el cliente reconcile su estado local

**Dado** el cliente con polling cada 5s del estado de su última sugerencia editada
**Cuando** detecta divergencia entre su `actualizada_en` local y la del servidor (porque otro coord o admin editó)
**Entonces** muestra indicador `"Tiene cambios nuevos — actualizando…"` y refresca esa fila desde servidor
**Y** si el coordinador estaba editando esa fila, se le advierte con modal `"Otro usuario modificó esta fila. Tu cambio sobrescribirá. ¿Continuar?"`

**Dado** una operación de PATCH que falla por timeout o 5xx
**Cuando** el cliente la captura
**Entonces** reintenta con backoff exponencial (2s, 4s, 8s) hasta 3 intentos
**Y** durante los reintentos el indicador muestra `"Guardando… (reintento 2/3)"`
**Y** si los 3 intentos fallan, muestra toast de sistema (UX-DR6) y persiste el cambio pendiente en `localStorage` para recuperación en el próximo login

**Dado** el coordinador cierra el navegador con cambios pendientes en `localStorage`
**Cuando** vuelve a abrir la bandeja
**Entonces** se le muestra modal `"Tienes 3 cambios sin guardar de tu sesión anterior. ¿Reintentar o descartar?"`
**Y** "Reintentar" dispara los PATCH; "Descartar" limpia `localStorage` con confirmación explícita

**Dado** la política de conflicto v1 (último-en-escribir gana)
**Cuando** dos coordinadores editan la misma celda en una ventana de 100ms
**Entonces** ambos PATCH se procesan, el segundo gana en `actualizada_en`
**Y** la bitácora conserva los dos cambios — el primero NO se pierde, solo no es el valor "actual"

---

### Story 3.8: Agregados del distrito en panel lateral

Como **coordinador**,
quiero **un panel lateral con métricas vivas del distrito (presupuesto consumido, tiendas pendientes, carry-overs, SKUs con riesgo concentrado, costo total)**,
para **tener contexto agregado mientras trabajo en cada tienda sin cambiar de pantalla**.

**Acceptance Criteria:**

**Dado** el panel lateral de agregados en la bandeja
**Cuando** se renderiza
**Entonces** muestra cinco bloques: (a) "Presupuesto consumido" con desglose `techo {monto} − esporádicas {monto} → disponible {monto}` (FR-054 desglose visible), (b) "Tiendas pendientes" (contador + lista colapsable), (c) "Carry-overs" (contador + lista), (d) "SKUs con riesgo concentrado" (top 3), (e) "Costo total consolidado"

**Dado** los cinco bloques
**Cuando** se inspeccionan los valores
**Entonces** usan `MonedaMXNPipe` para presupuestos y costos, `CantidadDiscretaPipe` para conteos
**Y** los presupuestos grandes (>$100,000) usan modo abreviado con tooltip (`$1.2M` / hover `$1,234,567.89`)

**Dado** el coordinador aplica un override en una celda (3.6)
**Cuando** el recálculo optimista corre
**Entonces** todos los agregados del panel se recalculan en cliente en <100ms (NFR-P-1)
**Y** sin nueva llamada al backend para el agregado (se calcula del estado local)

**Dado** el algoritmo de "SKUs con riesgo concentrado"
**Cuando** se computa
**Entonces** identifica SKUs donde ≥3 tiendas piden cantidades altas (>1.5× la media del distrito de ese SKU)
**Y** muestra top 3 ordenados por número de tiendas y por exceso relativo
**Y** click sobre un SKU filtra la grid principal a esas tiendas

**Dado** un escenario test con 38 tiendas, 10 SKUs cada una
**Cuando** se mide recálculo
**Entonces** los cinco agregados completan en menos de 100ms en un laptop de spec mínima (1366×768, 4GB RAM)

**Dado** que el coordinador colapsa el panel
**Cuando** clickea el botón de toggle
**Entonces** el panel se oculta animado en 200ms y la grid principal aprovecha el ancho liberado
**Y** la preferencia se persiste en `localStorage` por usuario

---

### Story 3.9: Estado por tienda y surfacing pasivo de carry-over

Como **coordinador**,
quiero **ver el estado de cada tienda de mi distrito (lista, pendiente, carry-over, con recorte, aprobada, exportada) y reconocer pasivamente las tardías sin que el sistema las re-inscriba automáticamente**,
para **operar conforme a FR-061 y FR-066 (sin regla de arrastre en v1)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 3.9
**Cuando** se aplica
**Entonces** existe `mart.estado_tienda_ciclo` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `ciclo_id VARCHAR NOT NULL`, `tienda_id VARCHAR NOT NULL REFERENCES mart.tienda(id)`, `estado VARCHAR NOT NULL CHECK (estado IN ('PENDIENTE_SOLICITUD','CARRY_OVER','SUGERENCIA_LISTA','CON_RECORTE','APROBADA','EXPORTADA'))`, `actualizado_en TIMESTAMPTZ NOT NULL DEFAULT now()`, `actualizado_por VARCHAR`, `UNIQUE(ciclo_id, tienda_id)`

**Dado** un ciclo recién ejecutado por el motor (2.10)
**Cuando** se persisten las sugerencias
**Entonces** cada tienda con sugerencias arranca en estado `SUGERENCIA_LISTA` o `CON_RECORTE` (según validación presupuestal de 2.9)
**Y** tiendas sin sugerencia (motor no produjo nada — cold-start o equivalencia faltante) arrancan en `PENDIENTE_SOLICITUD`

**Dado** el sidebar de la bandeja
**Cuando** se renderiza la lista de tiendas
**Entonces** cada tienda lleva badge con su estado y color: gris (PENDIENTE), ámbar (CARRY_OVER), azul (SUGERENCIA_LISTA), naranja (CON_RECORTE), verde (APROBADA), verde oscuro (EXPORTADA)
**Y** click sobre el badge filtra la grid principal a esa tienda

**Dado** una tienda que en el ciclo anterior quedó sin enviar pedido
**Cuando** arranca el nuevo ciclo
**Entonces** su estado se marca como `CARRY_OVER` automáticamente (job nocturno o computado on-read)
**Y** **NO se reinscribe automáticamente al ciclo nuevo** (FR-066) — sigue apareciendo en la bandeja como recordatorio pasivo
**Y** **NO se recalcula ni traslada presupuesto** — el faltante se autocorrige vía forecast sales-driven en el ciclo nuevo

**Dado** un coordinador que decide actuar sobre una tienda en `CARRY_OVER`
**Cuando** aplica overrides
**Entonces** la tienda pasa de `CARRY_OVER` a `SUGERENCIA_LISTA` (o `CON_RECORTE` si los overrides exceden presupuesto)
**Y** la transición queda en bitácora

**Dado** la transición de estados
**Cuando** ocurre cualquier cambio
**Entonces** se persiste en `mart.estado_tienda_ciclo` y se registra en `mart.bitacora_evento` con `valor_anterior` y `valor_nuevo` del estado
**Y** la UI refresca el badge en tiempo real (al recibir confirmación del PATCH)

---

### Story 3.10: Aceptar / Modificar / Rechazar recorte sugerido

Como **coordinador**,
quiero **decidir explícitamente sobre cada recorte propuesto por el motor (aceptar, modificar a otro número, o rechazar)**,
para **mantener control sobre el ajuste presupuestal y dejar trazabilidad de cada decisión (FR-052, FR-053)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 3.10
**Cuando** se aplica
**Entonces** existe `mart.decision_recorte` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `ciclo_id VARCHAR NOT NULL`, `tienda_id VARCHAR NOT NULL`, `sku_insumo VARCHAR NOT NULL`, `decision VARCHAR CHECK (decision IN ('ACEPTAR','MODIFICAR','RECHAZAR'))`, `cantidad_original BIGINT NOT NULL`, `cantidad_final BIGINT NOT NULL`, `razon_estructurada VARCHAR NOT NULL`, `razon_libre TEXT`, `coordinador_id BIGINT NOT NULL`, `decidida_en TIMESTAMPTZ NOT NULL DEFAULT now()`, `UNIQUE(ciclo_id, tienda_id, sku_insumo)`

**Dado** una tienda con estado `CON_RECORTE`
**Cuando** el coordinador la abre
**Entonces** ve un panel con SKUs recortados por el motor: tabla con `SKU`, `cantidad_original`, `cantidad_recortada`, `motivo` (de 2.9)
**Y** tres botones por SKU: `Aceptar`, `Modificar`, `Rechazar`

**Dado** el coordinador clickea "Aceptar"
**Cuando** confirma
**Entonces** la cantidad de la celda en el grid principal pasa a `cantidad_recortada`
**Y** se registra en `mart.decision_recorte` con `decision='ACEPTAR'`, `cantidad_final=cantidad_recortada`, `razon_estructurada='recorte_aceptado_motor'`
**Y** la celda muestra el estilo de override (3.5) con badge `"Recorte aceptado por {user} {hh:mm}"`

**Dado** el coordinador clickea "Modificar"
**Cuando** abre el modal
**Entonces** muestra slider/input con rango `[0, cantidad_original]`, default `cantidad_recortada`, lista de razones (`recorte_modificado_riesgo_quiebre`, `recorte_modificado_evento_anticipado`, `otro`)
**Y** confirmar persiste `decision='MODIFICAR'`, `cantidad_final={valor_input}`, `razon_estructurada`

**Dado** el coordinador clickea "Rechazar"
**Cuando** confirma con razón obligatoria (modal con `recorte_rechazado_demanda_critica`, `recorte_rechazado_inventario_bajo`, `otro`)
**Entonces** la cantidad NO cambia (queda en `cantidad_original`)
**Y** se persiste `decision='RECHAZAR'`, `cantidad_final=cantidad_original`, `razon_estructurada`
**Y** la tienda queda **excediendo presupuesto deliberadamente** — el panel de agregados (3.8) lo refleja en rojo
**Y** la aprobación del consolidado (3.12) requerirá razón adicional para esta tienda

**Dado** una tienda con todos los recortes resueltos (aceptados/modificados/rechazados)
**Cuando** se inspecciona
**Entonces** el badge de estado pasa de `CON_RECORTE` a `SUGERENCIA_LISTA` (si presupuesto OK) o se mantiene `CON_RECORTE` (si rechazos dejan excediendo)

**Dado** UX-DR5 (acciones reversibles)
**Cuando** se confirma cualquier decisión de recorte
**Entonces** aparece toast "Deshacer" 8s con descripción
**Y** click en "Deshacer" inserta nueva fila en `decision_recorte` con `decision='ACEPTAR'` (de la decisión previa) y `razon_estructurada='deshacer_decision_previa'`

---

### Story 3.11: Dashboard pedido-vs-recibido

Como **coordinador**,
quiero **una pantalla dedicada que cruza por proveedor → tienda → SKU la necesidad sugerida vs los pedidos asignados/efectivos y los recibos reales**,
para **detectar brechas operativas y ajustar mis sugerencias futuras con evidencia (FR-067)**.

**Acceptance Criteria:**

**Dado** la ruta `/bandeja/pedido-vs-recibido`
**Cuando** un coordinador autenticado navega
**Entonces** el guard `COORDINADOR` resuelve y se renderiza la pantalla con tabla agrupada Proveedor → Tienda → SKU
**Y** el scoping territorial filtra solo tiendas del distrito del coordinador

**Dado** el endpoint `GET /api/v1/pedido-vs-recibido?rango_inicio=...&rango_fin=...`
**Cuando** se llama
**Entonces** consulta cruzando `mart.sugerencia_pedido` (necesidad sugerida), `staging.alloc` (`QTY_ALLOCATED` vs `QTY_TRANSFERRED`, último pedido efectivo y asignado), recibos reales (`QTY_RECEIVED`)
**Y** devuelve registros con `proveedor` (derivado de `staging.entrega_directa.SUPPLIER` o equivalente), `tienda_id`, `sku_insumo`, `necesidad_sugerida`, `pedido_asignado`, `pedido_efectivo`, `recibo_real`, `brecha_porcentual = recibo_real / pedido_efectivo`

**Dado** la tabla en pantalla
**Cuando** se renderiza
**Entonces** cada fila colorea la columna `brecha_porcentual`: **verde** ≥95%, **ámbar** 80-94%, **rojo** <80%
**Y** ordena por brecha ascendente por default (peores casos arriba)

**Dado** los filtros de la pantalla
**Cuando** el usuario los aplica
**Entonces** puede filtrar por rango de fechas (default últimos 30 días), proveedor (dropdown), tienda (dropdown), SKU (búsqueda libre)
**Y** los filtros persisten en query params para compartir el link

**Dado** el botón "Exportar a CSV"
**Cuando** el coordinador lo clickea
**Entonces** descarga `pedido-vs-recibido-{coordinador}-{fecha}.csv` con todas las filas filtradas
**Y** las columnas usan los formatters de 3.2 cuando se exportan (cantidades enteras, fechas dd/MM/yyyy)

**Dado** que esta vista es informativa, no decisional
**Cuando** se inspecciona
**Entonces** NO modifica `sugerencia_pedido` ni bloquea aprobación
**Y** está claro en el header `"Vista informativa — no afecta el ciclo actual"`

---

### Story 3.12: Aprobación del consolidado distrital

Como **coordinador**,
quiero **aprobar mi consolidado del ciclo cuando todo está resuelto, con validaciones previas que me protegen de aprobar a medias**,
para **disparar el export al comprador sabiendo que no hay nada pendiente (FR-065)**.

**Acceptance Criteria:**

**Dado** la bandeja con varias tiendas en estados mixtos
**Cuando** el coordinador clickea "Aprobar consolidado"
**Entonces** se ejecuta pre-validación: (a) ninguna tienda con `estado=PENDIENTE_SOLICITUD`, (b) tiendas con `CON_RECORTE` que excedan presupuesto solo si TODOS sus recortes tienen `decision` registrada en `mart.decision_recorte`, (c) todos los overrides con `razon_estructurada` no nula

**Dado** que la pre-validación falla
**Cuando** se inspecciona la respuesta
**Entonces** se muestra modal listando las tiendas problemáticas con el problema específico (`"T-NORTE-28: Recorte SKU PAPEL-001 sin decisión"`, `"T-SUR-12: Override sin razón en SKU PAPEL-014"`)
**Y** click en cada item navega a esa tienda+SKU en el grid

**Dado** que la pre-validación pasa
**Cuando** se confirma la aprobación con modal "¿Confirmas aprobar el consolidado de N tiendas por $X total?"
**Entonces** se persiste el cambio: todas las tiendas pasan a `estado=APROBADA` en `mart.estado_tienda_ciclo`
**Y** se registra en bitácora con `entidad_tipo='aprobacion_consolidado'`, `coordinador_id`, `ciclo_id`, `total_tiendas`, `costo_total`
**Y** la UI bloquea más ediciones — el grid se vuelve read-only con banner `"Consolidado aprobado por {coordinador} {timestamp}"`

**Dado** un consolidado aprobado
**Cuando** el coordinador intenta editar una celda
**Entonces** AG Grid bloquea la edición y muestra inline `"Consolidado aprobado — para modificar requiere reabrir explícitamente"`
**Y** existe botón "Reabrir consolidado" que requiere razón estructurada obligatoria (`error_detectado`, `cambio_solicitado_por_jefe`, `otro`) y registra reapertura en bitácora

**Dado** que la aprobación dispara el export downstream (3.13)
**Cuando** se confirma
**Entonces** la generación del XLSX inicia asíncronamente
**Y** el coordinador ve indicador "Generando archivo de export…" y al completar se le habilita el botón "Exportar y enviar" (de 3.14)

**Dado** UX-DR5
**Cuando** se aprueba el consolidado
**Entonces** aparece toast `"Consolidado aprobado · {N} tiendas · ${costo} total"` con botón "Deshacer" 8s
**Y** "Deshacer" reabre el consolidado y registra reapertura con `razon_estructurada='deshacer_aprobacion_inmediata'`

---

### Story 3.13: Export XLSX compatible con `SolicitusDeInsumosTodos.xlsx`

Como **el sistema**,
quiero **generar un archivo XLSX byte-compatible con la estructura validada de `docs/SolicitusDeInsumosTodos.xlsx` y subirlo a Cloud Storage**,
para **que el comprador reciba un archivo idéntico al que hoy llena a mano, sin reentrenamiento (FR-110)**.

**Acceptance Criteria:**

**Dado** la dependencia Apache POI en `pom.xml`
**Cuando** se inspecciona
**Entonces** la versión está pinneada a la versión documentada en el ADR de XLSX export
**Y** existe el servicio `ExportXlsxService` en `api/adaptador/`

**Dado** el servicio `ExportXlsxService.generar(cicloId, coordinadorId)`
**Cuando** se invoca
**Entonces** lee `mart.sugerencia_pedido` filtrado por `ciclo_id` y `coordinador_id`
**Y** produce un workbook con la estructura exacta de `docs/SolicitusDeInsumosTodos.xlsx`: misma cabecera (nombres, orden, formato), mismo número de columnas, mismas hojas si las hay
**Y** la vista de trabajo del coordinador es **por SKU → tiendas + cantidades** (Opción A, D-11): filas = SKU, columnas = tiendas, celdas = cantidad

**Dado** un test de regresión del export
**Cuando** se compara el archivo generado contra `docs/SolicitusDeInsumosTodos.xlsx` como fixture
**Entonces** las cabeceras (primera fila) coinciden byte-a-byte
**Y** el número y orden de columnas coinciden
**Y** los tipos de celda (string, número, fórmula) coinciden por columna
**Y** divergencia rompe build con mensaje claro de qué celda diverge

**Dado** que el archivo se sube a Cloud Storage
**Cuando** se persiste
**Entonces** queda en `outbound/{ciclo_id}/{distrito}/SolicitusDeInsumos-{hash_sha256}.xlsx`
**Y** se registra el `gs://` URI completo en respuesta del servicio
**Y** el archivo tiene metadata: `coordinador_id`, `ciclo_id`, `total_tiendas`, `total_skus`, `costo_total`, `generado_en`

**Dado** que el equipo de datos corrige el typo `"Matarial y tamaño"` → `"Material y tamaño"` en el archivo de referencia
**Cuando** se ejecuta el test de regresión
**Entonces** **falla** con mensaje claro `"Cabecera divergió de fixture — confirmar cambio de contrato con comprador"` (no es mejora silenciosa)

**Dado** un escenario con consolidado de 38 tiendas × 10 SKUs
**Cuando** se mide tiempo de generación
**Entonces** completa en menos de 10 segundos en una instancia Cloud Run mínima
**Y** la métrica `export.xlsx.duration_seconds` se expone vía Actuator

---

### Story 3.14: Exportar bajo demanda y envío por correo

Como **coordinador**,
quiero **enviar el consolidado exportado al comprador por correo con CC configurable, viendo preview antes de enviar**,
para **cerrar mi ciclo sin salir de la herramienta y con bitácora del envío (FR-111, FR-112)**.

**Acceptance Criteria:**

**Dado** un consolidado en estado `APROBADA`
**Cuando** el coordinador clickea "Exportar y enviar"
**Entonces** abre modal con: campo `destinatario_principal` pre-cargado con el comprador asignado (de `mart.comprador`), campos `cc[]` editables, asunto pre-llenado `"Insumos OD — Ciclo Sem {N}/{año} — Distrito {distrito}"`, cuerpo en es-MX con resumen

**Dado** el modal de envío
**Cuando** el coordinador clickea "Vista previa"
**Entonces** se descarga (sin enviar) el XLSX generado para revisión local
**Y** se muestra preview del cuerpo del correo

**Dado** el coordinador confirma envío
**Cuando** clickea "Enviar"
**Entonces** el backend genera (o regenera si han pasado cambios) el XLSX (3.13)
**Y** llama a SendGrid (AR-Email) con XLSX adjunto, destinatarios, asunto, cuerpo
**Y** registra en `mart.bitacora_evento` con `entidad_tipo='export'`, `entidad_id={hash_archivo}`, `valor_nuevo={destinatarios, cc, asunto, hash_archivo}`, `actor={coordinador_id}`, `timestamp`
**Y** el estado de las tiendas del consolidado pasa de `APROBADA` a `EXPORTADA`

**Dado** la integración con SendGrid
**Cuando** la API responde con error transitorio
**Entonces** se reintenta con backoff exponencial (5s, 15s, 60s)
**Y** tras 3 intentos fallidos se muestra toast de sistema (UX-DR6) con código de incidente
**Y** el estado de las tiendas NO cambia a `EXPORTADA` (solo si el envío fue exitoso)

**Dado** los secretos de SendGrid
**Cuando** el backend los lee
**Entonces** vienen de GCP Secret Manager (no de `application.properties`)
**Y** la configuración de `from`, `from_name`, `reply_to` está parametrizada por entorno (dev/staging/prod)

**Dado** un consolidado en estado `EXPORTADA`
**Cuando** el coordinador navega a la bandeja
**Entonces** ve banner `"Consolidado exportado el {fecha} a {comprador}, copia a {N} destinatarios. Reenviar"`
**Y** el botón "Reenviar" permite nuevo envío con bitácora separada (no sobrescribe la anterior)

**Dado** que SendGrid envía el correo exitosamente
**Cuando** se inspecciona el adjunto
**Entonces** es el mismo XLSX subido a Cloud Storage (mismo hash SHA256)
**Y** el nombre del archivo en el correo es `SolicitusDeInsumos-{ciclo}-{distrito}.xlsx` (legible, sin hash)

---

### Story 3.15: Panel "Cambios de hoy" exportable a CSV

Como **coordinador**,
quiero **un panel lateral con la lista cronológica de TODOS mis cambios del día (overrides, decisiones de recorte, aprobaciones, exports) y poder exportarlo a CSV**,
para **tener auditoría rápida sin abrir reportes pesados (FR-102)**.

**Acceptance Criteria:**

**Dado** el panel lateral colapsable "Cambios de hoy"
**Cuando** se abre
**Entonces** se renderiza lista cronológica descendente (más reciente arriba) de todos los eventos del coordinador en el día actual (zona horaria CDMX, `America/Mexico_City`)
**Y** cada entrada muestra: `hh:mm`, ícono del tipo de acción, tienda + SKU si aplica, valor anterior → valor nuevo, razón estructurada

**Dado** el endpoint `GET /api/v1/bitacora/cambios-hoy`
**Cuando** se llama sin parámetros
**Entonces** filtra `mart.bitacora_evento WHERE actor = {jwt.subject} AND timestamp_evento >= today_at_00_CDMX AND timestamp_evento < tomorrow_at_00_CDMX`
**Y** ordena por `timestamp_evento DESC`
**Y** devuelve entrada de cada acción: override, decisión de recorte, aprobación de consolidado, export

**Dado** que un administrador en modo coordinador opera en el sistema
**Cuando** consulta sus cambios de hoy
**Entonces** ve sus propios cambios (filtro por `actor = jwt.subject`), no los del coordinador titular del distrito (a menos que sea el mismo subject)

**Dado** el botón "Exportar a CSV"
**Cuando** el coordinador lo clickea
**Entonces** descarga `cambios-hoy-{coordinador}-{fecha}.csv` con columnas: `timestamp`, `accion`, `tienda_id`, `tienda_nombre`, `sku_insumo`, `descripcion_sku`, `valor_anterior`, `valor_nuevo`, `razon_estructurada`, `razon_libre`
**Y** las fechas en CSV usan formato `dd/MM/yyyy HH:mm` (consistente con `docs/`)

**Dado** un coordinador que no ha hecho cambios hoy
**Cuando** abre el panel
**Entonces** muestra estado vacío `"Sin cambios registrados hoy"` con ícono ilustrativo
**Y** el botón "Exportar a CSV" se deshabilita

**Dado** una entrada de bitácora con `referencia_evento_previo`
**Cuando** se renderiza en el panel
**Entonces** se indica visualmente la cadena (ej. flecha hacia arriba mostrando que es una corrección de evento previo)
**Y** click expande detalles enlazando al evento anterior por id

**Dado** que la bitácora es inmutable (FR-104, AR-Bitacora)
**Cuando** se inspecciona la consulta del endpoint
**Entonces** es solo SELECT — sin UPDATE/DELETE
**Y** correcciones aparecen como nuevas entradas con `referencia_evento_previo` apuntando al original

---

## Épica 4: Bandeja territorial — excepciones, factor de pérdida y análisis TFS

**Goal:** El coordinador territorial registra y resuelve excepciones mid-cycle (formalizadas por correo) con contexto agregado (saldo SIM, ALLOC pendiente, TFS en tránsito e histórico) y escalamiento al comprador; configura el factor de pérdida por tipo de papel y revisa candidatos residuales de consumo atípico; consulta el histórico TFS cruzado con filtros.

**FRs cubiertos:** FR-074, FR-075, FR-080, FR-081, FR-082, FR-083, FR-091, FR-092, FR-093, FR-094, FR-095.

### Story 4.1: Shell de bandeja territorial y navegación por secciones

Como **coordinador territorial**,
quiero **una pantalla dedicada para mis responsabilidades territoriales (excepciones, factor de pérdida, candidatos residuales, histórico TFS) con navegación clara entre secciones**,
para **gestionar lo que no es ciclo regular sin contaminar la bandeja semanal**.

**Acceptance Criteria:**

**Dado** un coordinador autenticado
**Cuando** navega a `/territorial`
**Entonces** el guard `COORDINADOR` resuelve y se renderiza el módulo standalone territorial
**Y** el layout muestra cuatro secciones navegables (tabs o sidebar): "Excepciones mid-cycle", "Factor de pérdida", "Candidatos residuales", "Histórico TFS"
**Y** un usuario con rol `ADMINISTRADOR` puede acceder (modo coordinador, NFR-S-2)

**Dado** el header de la pantalla
**Cuando** se renderiza
**Entonces** muestra `Coordinador: {nombre}`, `Distrito: {distrito}` y un resumen con contadores: `Excepciones abiertas: N`, `Candidatos pendientes: M` (badge rojo si M > 5)
**Y** los contadores se cargan desde endpoints respectivos en paralelo, sin bloquear el render

**Dado** el endpoint resumen `GET /api/v1/territorial/resumen`
**Cuando** se llama
**Entonces** devuelve `{ excepciones_abiertas, candidatos_pendientes, factor_default_global, ultima_actualizacion_factor }` filtrado por el distrito del coordinador
**Y** el scoping territorial aplica (filtro `WHERE coordinador_id = jwt.subject`)

**Dado** las cuatro secciones del shell
**Cuando** el coordinador navega entre ellas
**Entonces** Angular Router usa rutas hijas `/territorial/excepciones`, `/territorial/factor`, `/territorial/candidatos`, `/territorial/tfs`
**Y** cada ruta lazy-load su módulo correspondiente
**Y** el estado de filtros/scroll se preserva en cada sección al volver

**Dado** axe-core sobre el shell
**Cuando** se ejecuta
**Entonces** la navegación es accesible por teclado (Tab entre tabs/links, Enter activa)
**Y** cada sección tiene `aria-label` descriptivo
**Y** el foco activo es visible con `:focus-visible` 2px

---

### Story 4.2: Configuración del factor de pérdida por tipo de papel

Como **coordinador territorial**,
quiero **revisar y modificar el factor de pérdida por tipo de papel (y el default global) con razón estructurada obligatoria**,
para **calibrar el motor de pronóstico cuando observo cambios sistemáticos en pérdidas, alimentando FR-035 / FR-082 / FR-085**.

**Acceptance Criteria:**

**Dado** la sección `/territorial/factor`
**Cuando** se renderiza
**Entonces** muestra tabla con todos los tipos de papel: columna `tipo_papel`, `factor_actual` (% con un decimal), `nivel_origen` (badge `TIPO_PAPEL` o `DEFAULT_GLOBAL`), `vigencia_desde`, `actualizado_por`, `actualizado_en`
**Y** los tipos sin factor configurado muestran el default global con badge ámbar `"Sin configuración — usando default global"`

**Dado** un tipo de papel en la tabla
**Cuando** el coordinador clickea "Editar"
**Entonces** abre modal con: input `nuevo_factor_porcentual` (rango 0–100, hasta 2 decimales, transformado a `BigDecimal(scale=4)` server-side), dropdown `razon_estructurada` (`recalibracion_periodica`, `tendencia_observada`, `cambio_de_proveedor`, `error_correccion`, `otro`), textarea `razon_libre` opcional (obligatoria si `razon_estructurada='otro'`)
**Y** preview muestra `"Cambio: 0.9% → 1.2% (+0.3pp)"` antes de confirmar

**Dado** el coordinador confirma el cambio
**Cuando** envía PUT `/api/v1/factor-perdida/tipo-papel/{id}`
**Entonces** el backend cierra la vigencia anterior (`vigencia_hasta = ayer`), inserta nueva fila en `mart.factor_perdida_tipo_papel` con vigencia abierta
**Y** registra en `mart.bitacora_evento` con `entidad_tipo='factor_perdida'`, `valor_anterior={factor_porcentual:0.009}`, `valor_nuevo={factor_porcentual:0.012}`, `razon_estructurada`, `razon_libre`, `actor=coordinador_id`
**Y** la tabla en UI refresca y muestra el nuevo factor con badge `TIPO_PAPEL`

**Dado** un coordinador con rol `COORDINADOR` (no admin)
**Cuando** intenta editar el `default_global`
**Entonces** el endpoint `PUT /api/v1/parametro-sistema/forecast.factor_perdida.default_global` requiere rol `ADMINISTRADOR` — devuelve `403`
**Y** la UI muestra el input deshabilitado con tooltip `"Solo administrador puede modificar el default global"`

**Dado** un administrador
**Cuando** modifica el default global con razón estructurada
**Entonces** el endpoint persiste el cambio en `mart.parametro_sistema` y bitácora
**Y** un toast confirma `"Default global actualizado de 0.9% a 1.1%"`
**Y** la siguiente ejecución del motor (E2) usará el nuevo default para tipos de papel sin configuración específica

**Dado** UX-DR5 (acciones reversibles)
**Cuando** el coordinador modifica un factor
**Entonces** aparece toast "Deshacer" 8s
**Y** "Deshacer" inserta nueva entrada en `factor_perdida_tipo_papel` con `razon_estructurada='deshacer_cambio_previo'` y vuelve al valor anterior — la cadena en bitácora queda completa

**Dado** una mutación al factor de un tipo de papel
**Cuando** las sugerencias del ciclo activo ya están generadas
**Entonces** el cambio NO recalcula automáticamente las sugerencias existentes
**Y** la UI muestra banner informativo `"Cambio aplicado. Re-ejecutar el motor del ciclo actual desde /admin/forecast para que las sugerencias reflejen el nuevo factor"`

---

### Story 4.3: Registrar excepción mid-cycle

Como **coordinador territorial**,
quiero **registrar una excepción mid-cycle (avisada por correo) con todos sus campos estructurados y asignación automática a mi territorio**,
para **dejar trazabilidad antes de decidir y empezar a trabajar con el contexto disponible (FR-091)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 4.3
**Cuando** se aplica
**Entonces** existe `mart.excepcion_midcycle` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `tienda_id VARCHAR NOT NULL REFERENCES mart.tienda(id)`, `sku_insumo VARCHAR NOT NULL REFERENCES mart.sku_insumo(codigo)`, `cantidad_estimada_faltante BIGINT NOT NULL CHECK (cantidad_estimada_faltante > 0)`, `fecha_esperada_quiebre DATE NOT NULL`, `urgencia VARCHAR CHECK (urgencia IN ('BAJA','MEDIA','ALTA','CRITICA')) NOT NULL`, `motivo_estructurado VARCHAR CHECK (motivo_estructurado IN ('venta_excepcional','evento_no_capturado','inventario_no_reflejado','otro')) NOT NULL`, `motivo_libre TEXT`, `categoria_disparador VARCHAR CHECK (categoria_disparador IN ('temporalidad','uso_interno','otra')) NOT NULL`, `origen_aviso VARCHAR NOT NULL DEFAULT 'correo_electronico' CHECK (origen_aviso='correo_electronico')`, `coordinador_id BIGINT NOT NULL REFERENCES mart.coordinador(id)`, `estado VARCHAR CHECK (estado IN ('ABIERTA','EN_RESPUESTA','RESUELTA','RECHAZADA')) NOT NULL DEFAULT 'ABIERTA'`, `registrada_en TIMESTAMPTZ NOT NULL DEFAULT now()`

**Dado** la sección `/territorial/excepciones`
**Cuando** un coordinador clickea "Nueva excepción"
**Entonces** abre form con campos: dropdown `tienda` (solo tiendas del distrito), búsqueda autocompleta `sku_insumo` (filtra por descripción), input `cantidad_estimada_faltante`, datepicker `fecha_esperada_quiebre` (no permite fechas pasadas), dropdown `urgencia`, dropdown `motivo_estructurado`, textarea `motivo_libre` opcional, dropdown `categoria_disparador`
**Y** el campo `origen_aviso` aparece pre-llenado y deshabilitado con valor `correo_electronico` y tooltip `"Definido por proceso operativo (FR-091)"`

**Dado** el coordinador confirma el registro
**Cuando** envía POST `/api/v1/excepciones`
**Entonces** el backend valida que `tienda_id` pertenezca al distrito del coordinador (scoping territorial)
**Y** persiste en `mart.excepcion_midcycle` con `coordinador_id={jwt.subject}` y `estado='ABIERTA'`
**Y** registra entrada en `mart.bitacora_evento` con `entidad_tipo='excepcion_midcycle'`, `valor_nuevo={JSON completo}`, `actor=coordinador_id`

**Dado** que el coordinador intenta registrar una excepción para una tienda fuera de su distrito
**Cuando** se valida en backend
**Entonces** devuelve `403 Forbidden` con `ProblemDetail` `title="Tienda fuera del territorio del coordinador"`
**Y** el form en UI nunca debería listarla — esto es defensa en profundidad

**Dado** la lista de excepciones en la sección
**Cuando** se renderiza
**Entonces** muestra tabla ordenada por `urgencia` descendente (CRITICA → BAJA) y luego `registrada_en` descendente
**Y** cada fila muestra `tienda`, `sku_insumo`, `cantidad_estimada_faltante`, `fecha_esperada_quiebre`, `urgencia` (badge color), `estado`, acción "Abrir"
**Y** filtros: por `estado`, `urgencia`, `tienda`, rango de fechas

**Dado** UX-DR5
**Cuando** el coordinador registra una excepción
**Entonces** aparece toast "Deshacer" 8s
**Y** "Deshacer" cambia `estado='RECHAZADA'` con `razon_estructurada='deshacer_registro_inmediato'` (no se borra — bitácora preservada)

---

### Story 4.4: Contexto agregado de la excepción

Como **coordinador territorial**,
quiero **al abrir una excepción ver inmediatamente el contexto agregado del SKU en esa tienda (saldo SIM, ALLOC pendiente, TFS en tránsito, histórico TFS reciente)**,
para **decidir la respuesta con evidencia (FR-092), sin sugerencia automática del sistema**.

**Acceptance Criteria:**

**Dado** una excepción abierta
**Cuando** el coordinador clickea "Abrir"
**Entonces** se renderiza pantalla detalle con el form de la excepción (read-only) y panel lateral "Contexto agregado"
**Y** el panel carga datos vía `GET /api/v1/excepciones/{id}/contexto`

**Dado** el endpoint de contexto
**Cuando** se llama
**Entonces** devuelve `saldo_sim` (más reciente para `(tienda, sku_insumo)` desde `staging.snapshot_sim`), `alloc_pendiente` (`Σ CTD_PENDIENTE` con `STATUS != completado`), `tfs_en_transito` (`Σ QTY_TRANSFERRED` con `TO_LOC=tienda`, fecha futura o pendiente), `historico_tfs_90dias` (lista de TFS últimos 90 días que involucran la tienda — origen o destino — para ese SKU)

**Dado** el panel "Contexto agregado" en UI
**Cuando** se renderiza
**Entonces** muestra cuatro bloques con valores y mini-gráficos: SIM (cantidad + fecha del snapshot + badge si snapshot > 24h), ALLOC pendiente (cantidad + lista de últimas 3 órdenes), TFS en tránsito (cantidad + origen + fecha esperada), Histórico TFS 90d (timeline con flechas dirección origen→destino)

**Dado** el snapshot SIM está fuera de SLA (>36h, definido en AR-SIM-Policy)
**Cuando** se intenta cargar el contexto
**Entonces** el panel muestra warning `"Snapshot SIM con antigüedad >36h — datos pueden estar desactualizados"` en rojo
**Y** el coordinador puede continuar pero queda advertido

**Dado** el contexto cargado
**Cuando** se inspecciona
**Entonces** NO contiene ninguna "sugerencia automática" del sistema (FR-092 explícito)
**Y** todos los números son evidencia cruda lista para que el coordinador decida

**Dado** que cualquier excepción puede involucrar SKUs sin equivalencia o histórico insuficiente
**Cuando** se carga el contexto
**Entonces** si falta SIM o TFS, los bloques muestran `"Sin datos disponibles"` en lugar de cero falso
**Y** la diferencia se distingue visualmente (gris claro vs negro)

---

### Story 4.5: Decidir y registrar respuesta a la excepción + escalamiento al comprador

Como **coordinador territorial**,
quiero **elegir una de las cuatro respuestas a una excepción (TFS de emergencia, escalar a comprador, rechazar, esperar) y registrar la decisión completa con bitácora**,
para **resolver la excepción dejando trazabilidad y, si corresponde, notificar al comprador con contexto (FR-093, FR-094, FR-095)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 4.5
**Cuando** se aplica
**Entonces** existe `mart.respuesta_excepcion` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `excepcion_id BIGINT NOT NULL REFERENCES mart.excepcion_midcycle(id)`, `tipo_respuesta VARCHAR CHECK (tipo_respuesta IN ('TFS_EMERGENCIA','ESCALAR_COMPRADOR','RECHAZAR','ESPERAR_CICLO_NORMAL')) NOT NULL`, `detalle JSONB NOT NULL`, `razon_estructurada VARCHAR`, `razon_libre TEXT`, `coordinador_id BIGINT NOT NULL`, `respondida_en TIMESTAMPTZ NOT NULL DEFAULT now()`

**Dado** una excepción `ABIERTA`
**Cuando** el coordinador clickea "Responder"
**Entonces** abre wizard con 4 opciones mutuamente excluyentes: `TFS de emergencia (fuera del sistema)`, `Escalar a comprador`, `Rechazar excepción`, `Esperar al ciclo normal`
**Y** cada opción muestra descripción breve y consecuencia (`"El saldo del sistema no se modifica"`, `"Se enviará correo al comprador con contexto"`, etc.)

**Dado** la opción "TFS de emergencia (fuera del sistema)"
**Cuando** se selecciona
**Entonces** muestra form: `tienda_origen` (dropdown del distrito), `cantidad`, `fecha_realizada`, `resultado` (`'recibida_completa'`, `'recibida_parcial'`, `'no_concretada'`), `nota`
**Y** persiste en `respuesta_excepcion.detalle JSONB` el form completo
**Y** **NO modifica `staging.snapshot_sim` ni `staging.tfs`** — solo bitácora (FR-075)
**Y** `excepcion.estado='RESUELTA'`

**Dado** la opción "Escalar a comprador"
**Cuando** se selecciona
**Entonces** muestra preview del correo: destinatario (comprador asignado de `mart.comprador`), asunto `"Excepción de insumos — {tienda} — {sku_insumo}"`, cuerpo con resumen de la excepción + contexto agregado (de 4.4) + decisión del coordinador
**Y** permite editar CC, agregar nota adicional al cuerpo
**Y** al confirmar, llama a SendGrid (AR-Email) con secretos de Secret Manager
**Y** persiste `tipo_respuesta='ESCALAR_COMPRADOR'`, `detalle={destinatarios, asunto, hash_cuerpo}`, `excepcion.estado='EN_RESPUESTA'`

**Dado** la opción "Rechazar excepción"
**Cuando** se selecciona
**Entonces** obliga a `razon_estructurada` (`falsa_alarma`, `ya_resuelto`, `no_aplica`, `otro`) y `razon_libre` opcional
**Y** persiste `tipo_respuesta='RECHAZAR'`, `excepcion.estado='RECHAZADA'`

**Dado** la opción "Esperar al ciclo normal"
**Cuando** se selecciona
**Entonces** persiste `tipo_respuesta='ESPERAR_CICLO_NORMAL'`, `excepcion.estado='RESUELTA'`
**Y** sin notificaciones automáticas

**Dado** una compra extraordinaria autorizada vía el escalamiento al comprador
**Cuando** el comprador efectivamente registra la compra en `Entrega-directa-tienda.csv` con `TRAN_CODE=20` (Purchases) y la próxima ingesta (1.12) la procesa
**Entonces** el cálculo de `presupuesto_disponible` en E2.9 (FR-054) automáticamente la descuenta del techo del ciclo siguiente
**Y** **no requiere lógica adicional aquí** — el acoplamiento es vía el archivo de entrega directa, no vía endpoint dedicado

**Dado** la falla de SendGrid en el escalamiento
**Cuando** se reintenta con backoff exponencial 3 veces y todos fallan
**Entonces** la respuesta se persiste con `detalle.envio_fallido=true`, `excepcion.estado='EN_RESPUESTA'`
**Y** toast de sistema (UX-DR6) con código de incidente
**Y** botón "Reintentar envío" disponible en la pantalla de la excepción

**Dado** UX-DR5
**Cuando** el coordinador confirma una respuesta
**Entonces** aparece toast "Deshacer" 8s
**Y** "Deshacer" revierte `excepcion.estado` al anterior y registra nueva entrada en `respuesta_excepcion` con `razon_estructurada='deshacer_respuesta_inmediata'` (bitácora preservada)

---

### Story 4.6: Detección estadística residual y generación de candidatos

Como **el sistema**,
quiero **un job que detecte tienda × SKU con consumo atípico (residual ≥ 2σ tras descontar venta esperada y pérdida aplicada) y genere candidatos clasificados por severidad**,
para **que el coordinador territorial revise patrones sospechosos sin tener que cazar manualmente (FR-080, FR-081)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 4.6
**Cuando** se aplica
**Entonces** existe `mart.candidato_residual` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `tienda_id VARCHAR NOT NULL`, `sku_insumo VARCHAR NOT NULL`, `periodo_inicio DATE NOT NULL`, `periodo_fin DATE NOT NULL`, `consumo_real BIGINT NOT NULL`, `venta_esperada BIGINT NOT NULL`, `perdida_aplicada BIGINT NOT NULL`, `residual BIGINT NOT NULL`, `desviacion_sigmas NUMERIC(8,4) NOT NULL`, `severidad VARCHAR CHECK (severidad IN ('BAJO','MEDIO','ALTO','CRITICO')) NOT NULL`, `estado VARCHAR CHECK (estado IN ('PENDIENTE','CAPTURADO','DESCARTADO')) NOT NULL DEFAULT 'PENDIENTE'`, `generado_en TIMESTAMPTZ NOT NULL DEFAULT now()`, `UNIQUE(tienda_id, sku_insumo, periodo_inicio, periodo_fin)`
**Y** existe entrada en `mart.parametro_sistema` `('residual.umbral_sigma', '2.0', 'DECIMAL', 'system')`

**Dado** el job `DeteccionResidualBatch`
**Cuando** se ejecuta
**Entonces** itera por tienda × SKU activo, computa `consumo_real` (suma de salidas en `staging.snapshot_sim` diferenciales o equivalente), `venta_esperada` (desde `mart.sugerencia_pedido.trazo.demanda_baseline`), `perdida_aplicada` (`venta_esperada × factor_perdida`), `residual = consumo_real − venta_esperada − perdida_aplicada`
**Y** calcula `σ_serie` (desviación estándar histórica del residual para ese par tienda × SKU)
**Y** calcula `desviacion_sigmas = |residual| / σ_serie`

**Dado** la clasificación de severidad
**Cuando** `desviacion_sigmas > umbral_sigma` (default 2.0)
**Entonces** genera/actualiza candidato con severidad: `BAJO` (2-3σ), `MEDIO` (3-4σ), `ALTO` (4-5σ), `CRITICO` (>5σ)
**Y** si `desviacion_sigmas <= umbral_sigma`, no se genera candidato

**Dado** el endpoint admin `POST /api/v1/admin/deteccion-residual/ejecutar`
**Cuando** un administrador lo dispara
**Entonces** lanza el job asíncrono (similar a 2.10) y devuelve `202 Accepted` con `Location` del job
**Y** la programación nocturna automática queda para E5 (Cloud Scheduler)

**Dado** la lista de candidatos en `/territorial/candidatos`
**Cuando** se renderiza
**Entonces** muestra tabla con filtros: tienda, SKU, severidad, estado (default `PENDIENTE`), rango de generación
**Y** ordenada por `severidad` descendente y luego `generado_en` descendente
**Y** scope al distrito del coordinador

**Dado** las invariantes del cálculo
**Cuando** se inspecciona el código
**Entonces** `residual` puede ser positivo (consumo > esperado + pérdida) o negativo (consumo < esperado + pérdida)
**Y** ambos casos generan candidato si `|residual| / σ > umbral` — la dirección queda visible en UI

**Dado** que el factor único de pérdida no atribuye entre categorías (FR-080 sin atribución)
**Cuando** se genera un candidato
**Entonces** **no se intenta inferir** si es merma, robo, uso interno o evento — solo se reporta la anomalía
**Y** la atribución es responsabilidad humana en 4.7

---

### Story 4.7: Gestión de candidatos residuales — capturar / modificar / descartar

Como **coordinador territorial**,
quiero **revisar cada candidato residual y decidir si lo capturo (con motivo y evidencia), modifico la cantidad confirmada, o descarto el candidato**,
para **gestionar consumo atípico con trazabilidad acumulable y exportable para revisión periódica (FR-082 parte 2, FR-083)**.

**Acceptance Criteria:**

**Dado** la migración Flyway de 4.7
**Cuando** se aplica
**Entonces** existe `mart.candidato_confirmado` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `candidato_id BIGINT NOT NULL REFERENCES mart.candidato_residual(id)`, `cantidad_confirmada BIGINT NOT NULL`, `motivo VARCHAR CHECK (motivo IN ('merma_documentada','uso_interno_autorizado','evento_anomalo','otro')) NOT NULL`, `motivo_libre TEXT`, `url_evidencia VARCHAR`, `coordinador_id BIGINT NOT NULL`, `confirmado_en TIMESTAMPTZ NOT NULL DEFAULT now()`

**Dado** un candidato con estado `PENDIENTE`
**Cuando** el coordinador clickea "Capturar"
**Entonces** abre form con campos: `cantidad_confirmada` (default = `|residual|` del candidato, editable), dropdown `motivo`, textarea `motivo_libre` opcional, upload de evidencia (PDF/imagen, sube a Cloud Storage `outbound/evidencias/{candidato_id}-{filename}`)
**Y** confirmar persiste en `mart.candidato_confirmado` y cambia `candidato_residual.estado='CAPTURADO'`
**Y** registra en bitácora con `entidad_tipo='candidato_residual_capturado'`

**Dado** un candidato con estado `PENDIENTE`
**Cuando** el coordinador clickea "Descartar"
**Entonces** abre modal con dropdown `razon_estructurada` (`falso_positivo`, `no_relevante`, `dato_origen_incorrecto`, `otro`) y `razon_libre` opcional
**Y** persiste `candidato_residual.estado='DESCARTADO'` con bitácora

**Dado** un candidato ya `CAPTURADO`
**Cuando** se inspecciona la lista
**Entonces** aparece con badge verde y enlace al detalle de la captura
**Y** desde el detalle el coordinador puede ver la evidencia subida (descarga desde Cloud Storage) y modificar la cantidad confirmada (genera nueva entrada en `candidato_confirmado` con `referencia_evento_previo` bitácora)

**Dado** la vista de candidatos confirmados acumulados
**Cuando** el coordinador navega a `/territorial/candidatos/confirmados`
**Entonces** muestra tabla con todos los `candidato_confirmado` del distrito con filtros: motivo, rango de fechas, tienda, SKU
**Y** botón "Exportar a CSV" descarga la lista con columnas: `fecha`, `tienda`, `sku`, `cantidad_confirmada`, `motivo`, `motivo_libre`, `url_evidencia`, `coordinador`

**Dado** el upload de evidencia
**Cuando** se sube un archivo
**Entonces** se valida que sea PDF o imagen (PNG/JPG), tamaño <10 MB
**Y** se sube a Cloud Storage con metadata `coordinador_id`, `candidato_id`, `subido_en`
**Y** la URL pública firmada (signed URL) se persiste en `candidato_confirmado.url_evidencia`
**Y** la signed URL caduca a las 24h por seguridad — descarga regenera signed URL si es necesario

**Dado** UX-DR5
**Cuando** el coordinador captura o descarta un candidato
**Entonces** aparece toast "Deshacer" 8s con descripción
**Y** "Deshacer" inserta nueva entrada en bitácora y vuelve `candidato_residual.estado='PENDIENTE'` (no se borra el confirmado — bitácora preservada)

---

### Story 4.8: Histórico TFS cruzado y registro manual de TFS de emergencia

Como **coordinador territorial**,
quiero **una tabla de TFS cruzada con filtros y la posibilidad de registrar TFS de emergencia hechas fuera del sistema**,
para **tener visibilidad y bitácora completa de transferencias entre tiendas sin que el saldo del sistema se modifique directamente (FR-074, FR-075)**.

**Acceptance Criteria:**

**Dado** la sección `/territorial/tfs`
**Cuando** se renderiza
**Entonces** muestra tabla con todas las TFS ejecutadas desde `staging.tfs` (ingestado en 1.12) filtrada al distrito del coordinador
**Y** columnas: `CREATE_DATE`, `FROM_LOC` (con badge si es del distrito), `TO_LOC` (con badge si es del distrito), `sku_insumo`, `QTY_TRANSFERRED`, `STATUS` (crudo, sin interpretar)
**Y** ordenada por `CREATE_DATE` descendente por default

**Dado** los filtros disponibles
**Cuando** el coordinador los aplica
**Entonces** puede filtrar por `tienda_origen` (dropdown), `tienda_destino` (dropdown), `sku_insumo` (búsqueda libre), rango de fechas
**Y** los filtros persisten en query params

**Dado** el botón "Exportar a CSV"
**Cuando** el coordinador lo clickea
**Entonces** descarga `historico-tfs-{distrito}-{fecha}.csv` con las filas filtradas
**Y** los formatos respetan locale MX (`dd/MM/yyyy`)

**Dado** la migración Flyway de 4.8 (registro manual)
**Cuando** se aplica
**Entonces** existe `mart.tfs_manual_externa` con `id BIGINT GENERATED ALWAYS AS IDENTITY PK`, `from_loc VARCHAR NOT NULL`, `to_loc VARCHAR NOT NULL`, `sku_insumo VARCHAR NOT NULL REFERENCES mart.sku_insumo(codigo)`, `cantidad BIGINT NOT NULL CHECK (cantidad > 0)`, `fecha_realizada DATE NOT NULL`, `motivo VARCHAR NOT NULL`, `resultado VARCHAR CHECK (resultado IN ('recibida_completa','recibida_parcial','no_concretada')) NOT NULL`, `cantidad_recibida BIGINT`, `coordinador_id BIGINT NOT NULL`, `registrada_en TIMESTAMPTZ NOT NULL DEFAULT now()`

**Dado** el botón "Registrar TFS de emergencia fuera del sistema"
**Cuando** el coordinador lo clickea
**Entonces** abre form: `from_loc`, `to_loc`, `sku_insumo`, `cantidad` solicitada, `fecha_realizada` (no futura), `motivo` libre, `resultado` dropdown, `cantidad_recibida` (obligatoria si `resultado != 'no_concretada'`)
**Y** valida que al menos una de las dos tiendas esté en el distrito del coordinador (scoping)
**Y** persiste en `mart.tfs_manual_externa` con bitácora `entidad_tipo='tfs_manual_externa'`

**Dado** una TFS manual registrada
**Cuando** se inspecciona el saldo SIM en otras pantallas
**Entonces** **NO se modifica ni recalcula** desde `tfs_manual_externa` — el saldo solo proviene de `staging.snapshot_sim` (FR-075 explícito)
**Y** una vista informativa puede mostrar la TFS manual junto al histórico de TFS oficiales con badge distintivo (`Registro manual — saldo no modificado`)

**Dado** la lista combinada de TFS oficiales y manuales en pantalla
**Cuando** se renderiza
**Entonces** las TFS manuales aparecen con fondo distinto (color amarillo claro) y badge `"Manual"` claramente visible
**Y** se pueden filtrar como `Solo manuales` o `Solo oficiales` además de los filtros estándar

---

## Épica 5: Automatización y observabilidad operativa

**Goal:** El administrador monitorea la operación (tiempo de consolidación por coordinador, excepciones/semana, tasa de override por fase de calibración, carry-overs, latencia del motor, errores de ingesta por archivo, frescura del snapshot SIM) en un dashboard y recibe alertas automáticas; el batch del motor corre programado (Cloud Scheduler, ciclo lunes→miércoles) sin necesidad de trigger manual.

**FRs/NFRs cubiertos:** NFR-O-1, NFR-O-2, NFR-O-3 · `AR-Batch` (automatización Cloud Scheduler) · `AR-Obs` (Spring Boot Actuator + Cloud Logging/Monitoring).

### Story 5.1: Cloud Scheduler para batch del motor (programación automática)

Como **administrador**,
quiero **que el batch del motor de pronóstico se dispare automáticamente cada lunes a las 02:00 CDMX con el `ciclo_id` calculado por el sistema**,
para **no depender de un trigger manual y garantizar que las bandejas estén listas para los coordinadores el lunes en la mañana (AR-Batch)**.

**Acceptance Criteria:**

**Dado** un Service Account dedicado `forecast-scheduler-sa@{proyecto}.iam.gserviceaccount.com`
**Cuando** se provisionan los recursos vía Terraform o `gcloud`
**Entonces** el SA tiene rol `roles/run.invoker` SOLO sobre el servicio backend de Cloud Run
**Y** NO tiene ningún otro rol — principio de privilegio mínimo
**Y** la configuración queda versionada en `infra/scheduler/`

**Dado** el Cloud Scheduler job `forecast-batch-weekly`
**Cuando** se inspecciona
**Entonces** está configurado con cron `0 2 * * 1` (lunes 02:00) en zona horaria `America/Mexico_City`
**Y** target es HTTP POST a `https://{backend-url}/api/v1/admin/forecast/ejecutar`
**Y** auth via OIDC token del SA dedicado
**Y** body literal: `{ "ciclo_id": "{{=auto}}" }` — el backend interpreta `auto` como semana ISO actual

**Dado** el endpoint `/api/v1/admin/forecast/ejecutar` recibe `ciclo_id="auto"`
**Cuando** se procesa
**Entonces** calcula `ciclo_id` = año + semana ISO actual con formato `YYYY-W{NN}` (ej. `2026-W22`)
**Y** verifica que el `ciclo_id` no tenga ya un job EN_COLA o EJECUTANDO (de 2.10)
**Y** si está libre, dispara el batch normalmente
**Y** si está ocupado, devuelve `409 Conflict` y Scheduler registra el fallo (con retry)

**Dado** Scheduler retry policy
**Cuando** una invocación falla con `5xx` o timeout
**Entonces** se reintenta con backoff exponencial hasta 3 intentos (delays 30s, 90s, 270s)
**Y** tras 3 intentos fallidos dispara alerta a admin (alimenta 5.4)

**Dado** Cloud Run deploy del backend
**Cuando** se inspecciona la configuración
**Entonces** durante la ventana operativa L–V 06:00–22:00 CDMX, `min-instances=1` (sin cold-start para los coordinadores)
**Y** fuera de esa ventana, `min-instances=0` (optimización de costos)
**Y** la configuración se aplica vía Cloud Build deploy con perfiles (`prod-ventana-operativa` vs `prod-no-operativa`)

**Dado** que el batch programado completa exitosamente
**Cuando** se inspecciona el dashboard (de 5.3)
**Entonces** la tarjeta "Último batch del motor" muestra `Disparado por: scheduler-weekly`, `Duración: X min`, `Tiendas: N`, `Errores: M`
**Y** la bitácora registra `entidad_tipo='forecast_batch'`, `actor='scheduler-weekly'`, `ciclo_id`, métricas

**Dado** que un admin necesita re-disparar manualmente
**Cuando** llama el endpoint desde la UI o vía curl
**Entonces** funciona normalmente sin conflicto con el Scheduler (a menos que un job esté EN_COLA o EJECUTANDO ya)
**Y** el sistema soporta ambos triggers simultáneamente

---

### Story 5.2: Métricas de operación expuestas

Como **administrador**,
quiero **siete métricas operativas expuestas vía Micrometer y Cloud Monitoring (consolidación, excepciones, override, carry-over, latencia, errores ingesta, frescura SIM)**,
para **medir cómo va la operación y alimentar el dashboard y las alertas (NFR-O-1)**.

**Acceptance Criteria:**

**Dado** Micrometer configurado en el backend (ya base por Actuator de 1.4)
**Cuando** se inspecciona el código
**Entonces** las siguientes métricas están registradas con sus tags:
**Y** `consolidacion.duracion_segundos` (Timer): tag `coordinador_id`, `ciclo_id` — medido desde primer fetch de bandeja hasta export exitoso
**Y** `excepciones.creadas` (Counter): tag `coordinador_id`, `tipo='midcycle'` — incrementa al crear excepción
**Y** `override.tasa` (Gauge): tag `coordinador_id`, `ciclo_id`, `fase` (`calibracion` ciclos 1-3, `produccion` ciclos 4+) — ratio overrides/sugerencias del ciclo
**Y** `carryover.cantidad` (Gauge): tag `ciclo_id` — número de tiendas con `estado=CARRY_OVER` al cierre del ciclo
**Y** `forecast.batch.duracion_segundos` (Timer, ya existente de 2.10)
**Y** `ingesta.errores.cantidad` (Counter): tag `tipo_archivo`, `tipo_error` — incrementa con cada excepción de parser
**Y** `sim.snapshot.frescura_horas` (Gauge): updateada con cada ingesta SIM exitosa

**Dado** el endpoint `/actuator/metrics`
**Cuando** un admin lo consulta
**Entonces** lista todas las métricas registradas y permite consultar cada una con `/actuator/metrics/{nombre}?tag=coordinador_id:C-NORTE-01`
**Y** requiere rol `ADMINISTRADOR` (Spring Security)

**Dado** la integración con Cloud Monitoring
**Cuando** Micrometer exporta
**Entonces** las métricas custom aparecen en Cloud Monitoring bajo namespace `custom.googleapis.com/insumos_odemas/*`
**Y** las series históricas se preservan al menos 90 días (configurado en Monitoring)

**Dado** un test de integración por métrica
**Cuando** se ejecutan
**Entonces** verifican que cada métrica se incrementa/registra correctamente bajo un escenario simulado
**Y** los tags se aplican correctamente

**Dado** el cálculo de `fase` (calibración vs producción)
**Cuando** se computa
**Entonces** se basa en `ciclo_numero_secuencial` del coordinador (ciclos 1-3 desde su primer ciclo → `calibracion`, 4+ → `produccion`)
**Y** la tabla `mart.coordinador_ciclos` registra fecha de inicio operativo de cada coordinador para calcular esto

---

### Story 5.3: Dashboard de operación admin-facing

Como **administrador**,
quiero **un dashboard con visualizaciones claras de las métricas operativas en tiempo casi-real**,
para **monitorear la salud del sistema sin abrir Cloud Monitoring directamente (NFR-O-2)**.

**Acceptance Criteria:**

**Dado** la ruta `/admin/dashboard`
**Cuando** un administrador navega
**Entonces** el guard `ADMINISTRADOR` resuelve y se renderiza la pantalla
**Y** un usuario con rol `COORDINADOR` recibe `403 Forbidden`

**Dado** el layout del dashboard
**Cuando** se renderiza
**Entonces** muestra cuatro secciones: (a) **Cards** con números clave (excepciones de la semana, carry-overs del último ciclo, tiempo promedio de consolidación, latencia última ejecución del motor), (b) **Charts de tendencias** (línea semana vs anterior, mes vs anterior) usando ng2-charts o Chart.js, (c) **Tabla "Top issues"** (top 5 tiendas con más carry-overs, top 5 SKUs con más excepciones, top 3 archivos con errores de ingesta), (d) **Panel "Estado de jobs"** (último batch del motor: `disparado_por`, `timestamp`, `duración_min`, `tiendas_procesadas`, `errores_por_tipo`; última detección residual: similar)

**Dado** el endpoint `GET /api/v1/admin/dashboard/resumen`
**Cuando** se llama
**Entonces** devuelve el JSON con todas las cards y stats agregados desde `mart.*` + Cloud Monitoring API
**Y** la respuesta cachea en Caffeine 30s (las métricas no cambian más rápido que eso)

**Dado** los filtros del dashboard
**Cuando** un administrador los aplica
**Entonces** puede filtrar por rango de fechas (default últimos 7 días), distrito, coordinador
**Y** los filtros se reflejan en query params para compartir el link
**Y** todos los charts y cards se recalculan con el filtro aplicado

**Dado** el auto-refresh
**Cuando** el dashboard está abierto
**Entonces** se refresca cada 60s vía polling (sin WebSockets en v1)
**Y** un indicador discreto muestra `"Última actualización: hace X seg"`

**Dado** una métrica con valor anómalo (ej. excepciones semana > umbral)
**Cuando** se renderiza la card correspondiente
**Entonces** se colorea (ámbar/rojo) y enlaza a la sección detallada
**Y** click sobre la card filtra el dashboard a esa métrica/distrito

**Dado** que el dashboard usa los pipes locale MX de 3.2
**Cuando** se inspeccionan los valores
**Entonces** las cantidades, porcentajes y fechas siguen las reglas de formato (ya probadas en 3.2)
**Y** los rangos de fecha se muestran con `FechaSemanaIsoPipe` (ej. `"Sem 22 · 25-31 may 2026"`)

---

### Story 5.4: Alertas automáticas al administrador

Como **administrador**,
quiero **recibir alertas automáticas cuando algo va mal (ingesta falla, excepciones explotan, motor se tarda, SIM se vuelve viejo)**,
para **reaccionar sin tener que estar revisando el dashboard manualmente (NFR-O-3)**.

**Acceptance Criteria:**

**Dado** la configuración de alertas en `infra/alerting/`
**Cuando** se inspecciona
**Entonces** existen cuatro alert policies versionadas (Terraform `.tf` o `gcloud` scripts):
**Y** `alert_falla_ingesta` que dispara cuando logs Cloud Logging tienen `severity=ERROR AND labels.entidad_tipo:"ingesta_"` con frecuencia > 1 en 5 min
**Y** `alert_tasa_excepciones` que dispara cuando `excepciones.semana > media_movil_7_dias × 1.5`
**Y** `alert_latencia_motor` que dispara cuando `forecast.batch.duracion_segundos > 420` (7 min, NFR-E-3 margen +40%)
**Y** `alert_sim_obsoleto` que dispara cuando `sim.snapshot.frescura_horas > 36` (AR-SIM-Policy)

**Dado** una alerta se dispara
**Cuando** Cloud Monitoring la procesa
**Entonces** envía email vía Notification Channel a la lista de administradores configurada
**Y** el email contiene: severidad, métrica violada, valor actual vs umbral, link al dashboard filtrado, link al log relevante

**Dado** el Notification Channel email
**Cuando** se configura
**Entonces** apunta a un grupo (`admins@officedepot.com.mx` u otra lista corporativa) en lugar de a personas específicas
**Y** la configuración no incluye PII en el cuerpo del email (solo nombres de métricas, valores, links)

**Dado** la configuración opcional de Slack webhook
**Cuando** está habilitada en variables de entorno
**Entonces** las alertas también se envían a un canal Slack configurado
**Y** el mensaje Slack incluye reacciones rápidas (botones de ack/silencio si el webhook soporta acciones)

**Dado** las cuatro alertas configuradas
**Cuando** se ejecuta un test de fuego (manual o vía script `infra/alerting/test-fire.sh`)
**Entonces** se puede simular cada condición y verificar que la alerta dispara correctamente
**Y** la documentación en `infra/alerting/README.md` explica cómo correr este test

**Dado** una alerta sostenida (condición no se resuelve)
**Cuando** continúa durante 30 min
**Entonces** Cloud Monitoring re-envía notificación (escalamiento simple)
**Y** la frecuencia de re-envío es configurable por alert policy

---

### Story 5.5: Cloud Scheduler para detección residual nocturna

Como **administrador**,
quiero **que el job de detección residual (de E4.6) corra automáticamente cada noche**,
para **que los coordinadores lleguen cada mañana con los candidatos del día anterior listos en su bandeja territorial**.

**Acceptance Criteria:**

**Dado** la infra de Scheduler de 5.1 (Service Account + OIDC)
**Cuando** se inspecciona
**Entonces** existe un segundo Cloud Scheduler job `deteccion-residual-daily`
**Y** está configurado con cron `0 3 * * *` (diario 03:00 CDMX) en zona horaria `America/Mexico_City`
**Y** target es HTTP POST a `https://{backend-url}/api/v1/admin/deteccion-residual/ejecutar`
**Y** usa el mismo SA dedicado con `roles/run.invoker`

**Dado** el job dispara exitosamente
**Cuando** el backend procesa
**Entonces** ejecuta el job de E4.6 sin diferencia respecto al trigger manual
**Y** los candidatos generados quedan en `mart.candidato_residual` para revisión por coordinadores

**Dado** Scheduler retry policy
**Cuando** una invocación falla
**Entonces** se reintenta con backoff exponencial (3 intentos: 30s, 90s, 270s)
**Y** tras fallar 3 veces, dispara alerta a admin (alimenta 5.4 — añadir `alert_deteccion_residual_falla` si quieres formalmente; quedó implícita en este AC)

**Dado** un escenario donde el job nocturno tarda más de lo esperado y se solapa con el batch del motor del lunes (5.1)
**Cuando** ambos están EJECUTANDO
**Entonces** ambos jobs son independientes — operan sobre diferentes tablas (`candidato_residual` vs `sugerencia_pedido`)
**Y** ningún recurso compartido los bloquea (sin lock pesimista)

**Dado** que un admin necesita re-ejecutar manualmente
**Cuando** dispara desde `/admin/jobs` o vía endpoint (de E4.6)
**Entonces** funciona normalmente sin conflicto con el Scheduler
**Y** la bitácora registra el actor diferenciado (`actor='scheduler-daily'` vs `actor={admin_id}`)

**Dado** un test de integración de la cadena Scheduler → backend → job
**Cuando** se ejecuta en ambiente de staging
**Entonces** verifica que el job se dispara correctamente, completa, y persiste candidatos
**Y** la métrica `deteccion_residual.duracion_segundos` (similar a `forecast.batch.duracion_segundos`) queda registrada en Cloud Monitoring
**Y** el dashboard de 5.3 incluye esta métrica en el panel "Estado de jobs"










