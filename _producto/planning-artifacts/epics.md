---
stepsCompleted: [1, 2]
inputDocuments:
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md'
  - '_producto/planning-artifacts/architecture.md'
  - '_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/addendum.md'
  - '_producto/project-context.md'
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
