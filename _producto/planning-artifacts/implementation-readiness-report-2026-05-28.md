---
project_name: insumos-odemas
date: 2026-05-28
stepsCompleted: ['discovery', 'prd_analysis', 'epic_coverage', 'ux_alignment', 'epic_quality_review', 'final_assessment']
overallStatus: 'NEEDS WORK'
assessor: 'Claude (bmad-check-implementation-readiness)'
documentsIncluded:
  prd:
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/addendum.md
  architecture:
    - _producto/planning-artifacts/architecture.md
  epics:
    - _producto/planning-artifacts/epics.md
  ux:
    - _producto/planning-artifacts/ux-design-specification.md
  supporting:
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/validation-report.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/review-adversarial-general.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/review-rubric.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/.decision-log.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/.change-signal-2026-05-24.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/.change-signal-2026-05-25.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/.change-signal-2026-05-26.md
    - _producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/.change-signal-2026-05-26b.md
    - _producto/planning-artifacts/ux-design-directions.html
---

# Implementation Readiness Assessment Report

**Date:** 2026-05-28
**Project:** insumos-odemas

## Step 1 — Inventario de documentos

### PRD
- **Documento principal (whole):** `_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md` (136 KB, modificado 2026-05-27)
- **Addendum:** `_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/addendum.md` (7 KB, 2026-05-25)
- **Validación / revisión / rúbrica / decision log / change-signals:** presentes en la misma carpeta (registros del ciclo de revisión, no se reanalizan como PRD).
- **Sharded:** no se encontró carpeta `prd/` con `index.md`. ✅ Sin duplicado de formato.

### Arquitectura
- **Documento principal (whole):** `_producto/planning-artifacts/architecture.md` (45 KB, 2026-05-27)
- **Sharded:** no se encontró carpeta `architecture/`. ✅ Sin duplicado de formato.

### Epics & Stories
- **Documento principal (whole):** `_producto/planning-artifacts/epics.md` (37 KB, 2026-05-27)
- **Sharded:** no se encontró carpeta `epics/`. ✅ Sin duplicado de formato.
- **Stories:** no se encontró carpeta dedicada de stories (`stories/`, `epic-*/story-*.md`). ⚠️ A confirmar con el usuario si las historias detalladas están embebidas en `epics.md` o aún por generar.

### UX Design
- **Especificación principal (whole):** `_producto/planning-artifacts/ux-design-specification.md` (62 KB, 2026-05-28)
- **Direcciones / mockups:** `_producto/planning-artifacts/ux-design-directions.html` (20 KB, 2026-05-27)
- **Sharded:** no se encontró carpeta `ux/`. ✅ Sin duplicado de formato.
- **Pipeline WDS:** `design-artifacts/{A-Product-Brief, B-Trigger-Map, C-UX-Scenarios, D-Design-System, E-Development}` existen pero están **vacíos** — no aportan al assessment.

### Implementación / Tests
- `_producto/implementation-artifacts/` vacío (esperado en fase de planeación).
- `_producto/test-artifacts/` vacío (esperado — Murat/TEA aún no ha producido test design).

### Issues detectados
- ⚠️ **WARNING:** no hay carpeta de **stories** independiente. Hay que validar si `epics.md` ya contiene historias granuladas con criterios de aceptación, o si se asume el patrón "epics + stories embebidas".
- ⚠️ **WARNING:** carpetas `design-artifacts/*` y `_producto/test-artifacts/` vacías — fuera del scope inmediato de PRD/UX/Arch/Epics, pero relevantes para el veredicto final de implementation readiness.
- ✅ **Sin duplicados de formato** (whole vs sharded) en ninguno de los cuatro documentos críticos.

---

## Step 2 — PRD Analysis

Fuente: `_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md` (estado `final`, actualizado 2026-05-26) + `addendum.md`. El PRD ha pasado 4 rondas de change-signal y un Reviewer Gate (Grade Good, 0C/0H/4M/3L en update 2026-05-26b).

### Requisitos Funcionales (FRs)

Agrupados por capacidad C-1..C-12. Los FR con ~~tachado~~ están **eliminados** del scope v1 (los conservo como evidencia de trazabilidad y para detectar arrastres en epics).

**C-1 — Datos maestros**
- **FR-001** — Catálogo de tiendas con ID (String), nombre, distrito, coordinador territorial asignado, formato (Express/Estándar), ciclo quincenal (1y3 / 2y4), techo presupuestal semanal MXN.
- **FR-002** — Catálogo de SKUs de insumo (código, descripción, presentación, contenido = piezas por presentación, costo MXN). Preservar literal `"Matarial y tamaño"`.
- **FR-003** — Tabla de equivalencia SKU venta ↔ SKU insumo; sin equivalencia → `EquivalenciaNoDefinidaException` (fail-loud).
- **~~FR-004~~** — Eliminado (rotación de guardia fuera del sistema).
- **FR-005** — Datos de coordinadores (identidad, distrito territorial asignado, credenciales, rol).
- **FR-006** — Catálogo de eventos comerciales (ventanas + factor multiplicador): Hot Sale, BTS, Buen Fin, Navidad/Reyes, regreso a oficinas, Día de las Madres/Padre/Niño, cierre fiscal — pendiente confirmación Operaciones (D-5).
- **FR-007** — Catálogo de tipos de papel (`tipo_papel`) derivado de `Material` de `Skus_insumos.csv`. Unidad de configuración del `factor_perdida`.

**C-2 — Ingesta y normalización**
- **FR-010** — Ingesta por descubrimiento de patrón de nombre (`ALLOC_YYYY_MM_DD.csv`, `TFS_*.csv`, `Entrega-directa-tienda*.csv`).
- **FR-011** — Detección de encoding: UTF-8 BOM → cp1252; registrar en MDC.
- **FR-012** — Validar invariantes de ingesta (ej. `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE`, `TRAN_CODE ↔ DECODE`).
- **FR-013** — Parsear `Cantidad por semana` (`"$X,XXX"`) y `Semana que debe solicitar` (lenguaje natural → `List<Integer>` ISO); patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`.
- **FR-014** — Ingestar snapshots periódicos de SIM (cadencia diaria nocturna; formato/dueño en OQ-117 parcial).
- **~~FR-015~~** — Eliminado (ingesta de histórico de mermas no posible; factor manual del coord lo reemplaza).
- **FR-016** — Ingesta de **ventas con granularidad semanal** como reporte aportado por el owner (sin BigQuery v1).
- **FR-017** — Ingesta de **ALLOC y TFS** como reporte aportado por el owner; input de FR-040 y FR-067 (sin RMS/BigQuery v1).

**C-3 — Conteo asistido de bodega** — **CAPACIDAD ELIMINADA DE v1** (FR-020..025 fuera de scope; reabre en v1.1).

**C-4 — Forecast de demanda (cadencia dual)**
- **FR-030** — Calcular demanda esperada de venta **semanalmente** por tienda × SKU venta para el próximo ciclo activo. Cold-start (< 12 meses) → marcar "sin sugerencia del motor", fail-loud (OQ-108 abierta).
- **FR-031** — Convertir demanda de venta a demanda de insumo aplicando la tabla de equivalencia.
- **FR-032** — Cuantizar al múltiplo del empaque mínimo (`Contenido`). Nunca redondear hacia abajo.
- **FR-033** — Componente estacional basado en histórico ≥ 12 meses + catálogo de eventos comerciales FR-006.
- **FR-034** — Trazo explicable por SKU: histórico, ventana, factor estacional + evento, factor de pérdida (con nivel `tipo_papel` o `default_global`), ajuste por ALLOC/TFS.
- **FR-035** — Aplicar `factor_perdida` único resuelto por jerarquía `factor_por_tipo_papel → default_global (≈0.9%)` como ajuste porcentual sobre la demanda de venta antes de conversión a insumo y cuantización.

**C-5 — Sugerencia de pedido**
- **FR-040** — `cantidad_sugerida = demanda_quincenal_insumo + buffer_seguridad − saldo_SIM_snapshot − ALLOC_pendiente − TFS_entrante`, cuantizada.
- **FR-041** — Mostrar al coordinador en bandeja la sugerencia por tienda × SKU con trazo explicable.
- **~~FR-042~~** — Eliminado (no hay override del jefe; consolidado en FR-064).
- **FR-043** — Para SKUs sin equivalencia → no incluir + `EquivalenciaNoDefinidaException` aborta el ciclo de esa tienda.

**C-6 — Validación de presupuesto y sugerencia de recorte**
- **FR-050** — Costo total por tienda = `Σ (cantidad_sugerida × costo_SKU)` MXN.
- **FR-051** — Si excede, sugerir recorte priorizando reducir SKUs con menor riesgo de quiebre (`cobertura_post_recorte ≥ 2 semanas`).
- **FR-052** — Coordinador puede aceptar/modificar/rechazar con razón.
- **FR-053** — Persistir cada decisión de recorte (timestamp + ejecutor + razón).
- **FR-054** — `presupuesto_disponible = techo_semanal − Σ compras_esporádicas_del_periodo` (lectura `Entrega-directa-tienda.csv` TRAN_CODE 20).

**C-7 — Bandeja semanal del coordinador**
- **FR-060** — Bandeja con tiendas del distrito en ciclo activo esa semana (~38 tiendas).
- **FR-061** — Estado por tienda: solicitud recibida / pendiente solicitud / carry-over / sugerencia lista / con recorte / aprobada / exportada. El coordinador trabaja en pantalla; el Excel del sistema es OUTPUT, no input.
- **~~FR-062~~** — Eliminado.
- **FR-063** — Agregados del distrito: presupuesto consumido (con desglose `techo − esporádicas`), tiendas pendientes, en carry-over, SKUs con riesgo concentrado, costo del consolidado.
- **FR-064** — Override del coordinador con razón estructurada (lista predefinida + texto libre opcional); en bitácora. Ciclos 1-3 = ORO de calibración.
- **FR-065** — Aprobar consolidado y disparar export downstream.
- **FR-066** — **Surfacing pasivo** de pedidos tardíos (sin regla de arrastre; sin recalcular presupuesto).
- **FR-067** — Dashboard pedido-vs-recibido (proveedor → tienda → SKU): necesidad / pedido / recibido / brecha.

**C-8 — Histórico y análisis de TFS** (sin decisión ni sugerencia)
- **~~FR-070, FR-071, FR-072~~** — Eliminados.
- **FR-073** — Registrar histórico de TFS ejecutadas (ingesta `TFS_*.csv`).
- **FR-074** — Visualización cruzada con filtros (tienda origen/destino, SKU, periodo); exportable CSV.
- **FR-075** — Registrar manualmente decisión/resultado de TFS de emergencia (sin modificar saldo).

**C-9 — Factor de pérdida y detección residual**
- **FR-080** — Capa secundaria de detección de patrones: `residual = consumo_real − venta_esperada − perdida_aplicada`; umbral inicial ≥ 2σ (OQ-123).
- **FR-081** — Generar candidatos residuales clasificados por severidad para surfacing.
- **FR-082** — Coordinador territorial: (1) configurar `factor_perdida` por tipo de papel; (2) capturar/modificar/descartar candidato residual; (3) bitácora.
- **FR-083** — Acumular registro de candidatos confirmados (exportable).
- **FR-084** — **Default global ≈ 0.9% de la venta** para tipos de papel no configurados.
- **FR-085** — Factor alimenta motor (vía FR-035); cambios trazados con valor antes/después + razón.

**C-10 — Excepción mid-cycle**
- **~~FR-090~~** — Eliminado (jefe no opera el sistema).
- **FR-091** — Coordinador territorial registra excepción: tienda, SKU, cantidad estimada, fecha de quiebre, urgencia, motivo estructurado, categoría de disparador (temporalidad/uso interno/otra), origen = correo electrónico (campo fijo).
- **FR-092** — Mostrar contexto agregado: saldo SIM, ALLOC pendiente, TFS en tránsito y histórico.
- **FR-093** — Decidir y registrar respuesta: TFS de emergencia fuera, escalamiento a comprador, rechazo, espera al ciclo normal.
- **FR-094** — Notificar al comprador con contexto + decisión + datos.
- **FR-095** — Reflejar compra extraordinaria en saldo de planeación **y descontar del techo** vía FR-054.

**C-11 — Trazabilidad y auditoría**
- **FR-100** — Bitácora por cada cantidad modificada: timestamp, actor, valor antes/después, razón estructurada.
- **FR-101** — UI mostrar cadena completa (hover).
- **FR-102** — Panel "Cambios de hoy" exportable a CSV.
- **FR-103** — MDC en logs (`tiendaId`, `skuId`, `cicloId`, `usuarioId`).
- **FR-104** — Bitácora **inmutable**; correcciones = nuevas entradas referenciando la anterior.

**C-12 — Export downstream al comprador**
- **FR-110** — Generar archivo compatible con `SolicitusDeInsumosTodos.xlsx` (estructura validada por owner 2026-05-25 = contrato literal). Layout de trabajo del coord = Opción A (SKU → tiendas + cantidades). Dashboard FR-067 conserva vista `proveedor → tienda → SKU`.
- **FR-111** — Exportar bajo demanda y enviar por correo al comprador con CC configurable.
- **FR-112** — Registrar cada export en bitácora (timestamp, ejecutor, hash, destinatarios).

**Total FRs vigentes en v1: ~58** (excluyendo los marcados como eliminados FR-004, FR-015, FR-020..025, FR-042, FR-062, FR-070..072, FR-090).

### Requisitos No-Funcionales (NFRs)

**9.1 Escala y concurrencia (NFR-Escala)**
- **NFR-E-1** — 226 tiendas en el modelo; ≤ 3 coordinadores + 1 admin concurrentes.
- **~~NFR-E-2~~** — Eliminado (no hay 113 jefes en paralelo).
- **NFR-E-3** — Motor batch sobre ~8,600 series temporales (226 × 38 SKUs venta) → < 5 min JVM única. Cerrado como bloqueante.
- **NFR-E-4** — Bandeja distrital (~38 tiendas) carga en < 3 s.

**9.2 Performance percibida (NFR-Perf)**
- **NFR-P-1** — Recálculo optimista en cliente con `debounceTime(150)`; recalcular fila + acumulado + restante de presupuesto en < 100 ms.
- **NFR-P-2** — Autosave silencioso ("Guardado hace 3 seg"); sin pérdida de trabajo.

**9.3 Disponibilidad (NFR-Disp)**
- **NFR-D-1** — ≥ 99% en ventana operativa L-V 06:00-22:00 CDMX.
- **NFR-D-2** — Reanudación sin pérdida de overrides/recortes/excepciones tras caída.

**9.4 Auditabilidad (NFR-Audit)**
- **NFR-A-1** — Toda decisión trazada (timestamp + actor + antes/después + razón estructurada).
- **NFR-A-2** — MDC en logs.
- **NFR-A-3** — Bitácora inmutable.

**9.5 Seguridad (NFR-Sec)**
- **NFR-S-1** — Autenticación individual (patrón sugerido Auth0 + JWT).
- **NFR-S-2** — Roles v1: Coordinador (sus tiendas + factor por tipo de papel) y Administrador (datos maestros, eventos comerciales, umbrales, razones, `buffer_seguridad`). Override aplica también al Admin operando como coord.
- **NFR-S-3** — MFA obligatorio para roles administrativos.
- **NFR-S-4** — Datos sensibles (presupuestos, costos) sin exposición a roles no autorizados.

**9.6 Calidad del dato (NFR-Datos)**
- **NFR-DAT-1** — Fail-loud ante datos faltantes/ambiguos (excepciones checked).
- **~~NFR-DAT-2~~** — Eliminado.
- **NFR-DAT-3** — Encoding registrado en logs por archivo procesado.
- **NFR-DAT-4** — SLA frescura snapshot SIM ≤ 36h (24h ciclo + 12h buffer); rechazar operar si excede.

**9.7 Locale (NFR-Locale)**
- **NFR-L-1** — `LOCALE_ID = 'es-MX'` + `registerLocaleData(localeEsMX)`.
- **NFR-L-2** — Fechas display `dd/MM/yyyy`; JSON ISO 8601.
- **NFR-L-3** — Semanas ISO 8601 (`"Sem 12 · 16-22 mar 2026"`).
- **NFR-L-4** — MXN `$1,250.00`; totales grandes abreviados con tooltip.
- **NFR-L-5** — Cantidades enteras + separador miles + unidad pegada (`1,250 cajas`); mostrar equivalencia (`18 piezas = 3 cajas`).
- **NFR-L-6** — Plazos en días hábiles.

**9.8 Accesibilidad (NFR-A11y, WCAG 2.1 AA)**
- **NFR-A11Y-1** — Contraste 4.5:1 texto, 3:1 interactivos.
- **NFR-A11Y-2** — Toda celda editable con `aria-label` contextualizado.
- **NFR-A11Y-3** — Navegación por teclado completa en grillas; `:focus-visible` 2px; `outline:none` prohibido.

**9.9 UX no negociable (NFR-UX)**
- **NFR-UX-1** — Trazabilidad visible: tooltip info + diferenciación tipográfica sugerido/override + auditoría hover.
- **NFR-UX-2** — Acciones reversibles > confirmadas: toast "Deshacer" 8s, no `confirm()` modal.
- **NFR-UX-3** — Errores en 3 niveles (inline / negocio / sistema) con quién-qué-por qué-qué hacer ahora.
- **NFR-UX-4** — Tone es-MX claro, no técnico.

**9.10 Mantenibilidad y testabilidad (NFR-Test)**
- **NFR-T-1** — Pipeline composable (parsing → validación → catálogo → joins → forecast → cuantización → presupuesto → output).
- **NFR-T-2** — Golden datasets `src/test/resources/fixtures/golden/vN/` con `EXPECTED_OUTPUT.csv` firmado por **Hugo** + `DECISION_LOG.md`; inmutables.
- **NFR-T-3** — Property-based testing con jqwik sobre cuantización y presupuesto.
- **NFR-T-4** — Contract testing Pact / Spring Cloud Contract con YAML antes de implementar.
- **NFR-T-5** — Backtesting suite sobre histórico real para acordar MAPE/WAPE con Finanzas.
- **NFR-T-6** — Mutation testing PIT sobre `forecasting.*` ≥ 80% mutantes muertos antes de merge a `main`.

**9.11 Internacionalización (NFR-i18n)**
- **NFR-I-1** — Solo es-MX en v1; strings de UI en archivo de recursos.
- **NFR-I-2** — Identificadores en inglés; comentarios/docs de negocio en español.

**9.12 Observabilidad (NFR-Obs)**
- **NFR-O-1** — Métricas: tiempo de consolidación por coord, excepciones por semana, tasa de override (desglose ciclos 1-3 vs 4+), carry-overs por semana, latencia motor, errores de ingesta, frescura snapshot SIM.
- **NFR-O-2** — Dashboard de operación para el administrador.
- **NFR-O-3** — Alertas automáticas a admin (ingesta fallida, excepciones anómalas, motor fuera de SLA).

**Total NFRs vigentes en v1: ~37** (excluyendo NFR-E-2 y NFR-DAT-2 eliminados).

### Requisitos / Restricciones adicionales (constraints, supuestos, datos maestros)

- **Stack obligatorio (§10.1):** Java + Spring Boot + Docker + Cloud Run; PostgreSQL en Cloud SQL; Angular en Firebase Hosting; forecasting Java puro (Smile / Commons Math) tras interfaz `ForecastingEngine` — sin Python, sin Vertex AI / BigQuery ML sin RFC.
- **Datos maestros adicionales (§10.5):** Mapeo Tienda→Coordinador, `tipo_papel` derivado de `Material`, `factor_perdida` por tipo de papel (default global 0.9%), `buffer_seguridad` por tienda × SKU (≈ 1 semana de cobertura), umbrales de detección residual, catálogo de eventos comerciales, plazos del ciclo (miércoles default), lista de razones estructuradas.
- **SIM (§10.4):** modelo (C) Coexistencia con lectura unidireccional — sin escritura del sistema a SIM. Snapshot diario nocturno, SLA 36h.
- **Integraciones externas (§10.6):** SIM (lectura), ventas semanales (reporte del owner), ALLOC/TFS (reporte del owner), correo (SendGrid candidato), Auth0 (candidato), Cloud Storage.
- **Out-of-scope explícito (§11):** OCs al proveedor; ejecución física de TFS; insumos no-papel; módulos de SIM ajenos a bodega; modelado de capacidad física; aprendizaje automático del override; app móvil nativa; reportería BI/ejecutiva fuera del dashboard operativo; i18n fuera es-MX; capacitación/change-management; captura del jefe; conteo asistido; decisión y ejecución TFS; rotación de guardia; canal estructurado de alertas; saneamiento SIM; medición precisa de merma por tienda (Fase 3); inventario forzado; solicitud 100% automática.
- **Riesgos vivos (§12):** R-1 calidad sucia SIM (media reducida), R-2 resistencia al cambio (media reducida), R-3 falsos positivos detección residual (baja), R-4 incompat. comprador (media reducida), R-5 dependencia SIM (media), R-6 override alto inicial, R-7 acuerdo MAPE Finanzas (paralelo, no bloqueo de tesis), R-8 performance motor (baja — universo trivial), R-9 baseline cuantitativo (parcial — tiempo cerrado, falta quiebre/sobre-inventario/backtesting), R-10 Express vs Estándar, R-11 divergencia silenciosa SIM vs plan (alta), R-12 reportes del owner (baja).
- **Open Questions abiertas (§13):** OQ-101 (frecuencia excepciones — telemetría), OQ-104 (Express/Estándar en presupuesto), OQ-107 (MAPE/WAPE Finanzas), OQ-108 (cold-start < 12 meses), OQ-109 (selección tiendas piloto), OQ-110 (rechazo del comprador), OQ-111 (scope v2), OQ-112 (gobierno data master nueva), OQ-114 (política fail-loud SIM sucio), OQ-116 (publicación ALLOC esperado a SIM), OQ-117 parcial (formato + dueño extract SIM), OQ-123 (umbral detección residual).
- **Decisiones pendientes (no bloqueantes del PRD):** D-5 (lista de eventos comerciales), formato/cadencia de reportes del owner.

### Evaluación inicial de completitud del PRD

- ✅ **Cobertura funcional clara y trazable:** 12 capacidades agrupadas, FRs numerados, cada FR mapeado a un dolor de §3 o a una decisión registrada. Eliminados conservados con tachado — trazabilidad inversa perfecta.
- ✅ **NFRs explícitos** con métricas concretas en mayoría de casos (3 s para bandeja, < 100 ms recálculo, 36 h SLA SIM, 80% PIT, 99% disponibilidad).
- ✅ **Criterios de aceptación observables** en los FRs críticos (FR-030, FR-035) tras el update 2026-05-26b.
- ✅ **Out-of-scope explícito y justificado** en 17 incisos — bajo riesgo de scope creep.
- ✅ **Riesgos con dueños tentativos y mitigaciones** documentadas.
- ⚠️ **OQ-107 (MAPE Finanzas), OQ-108 (cold-start), OQ-114 (política fail-loud SIM sucio) siguen abiertas** — afectan diseño de épicas de motor y de ingesta SIM, pero el owner las reencuadró como **precondiciones de pilotaje, no de aprobación del documento**.
- ⚠️ **Decisión D-5** (lista de eventos comerciales) y formato/cadencia de reportes del owner pendientes — afectan FR-006 / FR-016 / FR-017.
- ⚠️ **NFR-T-5 (backtesting)** depende de archivos que el owner aún debe producir (ventas semanales históricas con corte semanal). Sin ese input el motor no puede validarse antes del go-live.

**Conclusión Step 2:** el PRD es lo suficientemente maduro y trazable para alimentar análisis de cobertura de épicas. Las gaps abiertas (OQ-107/108/114, D-5, formato de reportes del owner) son **gestionadas explícitamente como riesgos** y no impiden iniciar implementación de capacidades que **no dependen** de ellas (UI del coordinador, ingesta de los CSVs conocidos, framework de pruebas, trazabilidad/bitácora).

---

## Step 3 — Epic Coverage Validation

Fuente: `_producto/planning-artifacts/epics.md` (5 épicas + FR Coverage Map). El documento incluye un mapa explícito FR→épica que voy a verificar contrastándolo contra el inventario activo del PRD.

### Cobertura — Matriz FR → Épica

| FR | Texto PRD (resumen) | Cobertura en épicas | Estado |
|----|--------------------|---------------------|--------|
| **FR-001** | Catálogo de tiendas + atributos clave | Épica 1 | ✓ Cubierto |
| **FR-002** | Catálogo de SKUs de insumo + literal `"Matarial y tamaño"` | Épica 1 | ✓ Cubierto |
| **FR-003** | Equivalencia venta↔insumo (fail-loud) | Épica 1 | ✓ Cubierto |
| **FR-005** | Datos de coordinadores | Épica 1 | ✓ Cubierto |
| **FR-006** | Catálogo de eventos comerciales | Épica 1 (consumido por E2) | ✓ Cubierto |
| **FR-007** | Catálogo de tipos de papel (`Material`) | Épica 1 (consumido por E2/E4) | ✓ Cubierto |
| **FR-010** | Ingesta por patrón de nombre | Épica 1 | ✓ Cubierto |
| **FR-011** | Detección de encoding UTF-8 BOM → cp1252 | Épica 1 | ✓ Cubierto |
| **FR-012** | Validar invariantes de ingesta | Épica 1 | ✓ Cubierto |
| **FR-013** | Parseo presupuesto + periodicidad | Épica 1 | ✓ Cubierto |
| **FR-014** | Ingesta snapshot SIM (unidireccional) | Épica 1 + AR-SIM-Policy | ✓ Cubierto |
| **FR-016** | Ventas semanales como reporte del owner | Épica 1 | ✓ Cubierto |
| **FR-017** | ALLOC/TFS como reporte del owner | Épica 1 | ✓ Cubierto |
| **FR-030** | Demanda esperada de venta semanal | Épica 2 | ✓ Cubierto |
| **FR-031** | Conversión venta→insumo | Épica 2 | ✓ Cubierto |
| **FR-032** | Cuantización al empaque mínimo | Épica 2 | ✓ Cubierto |
| **FR-033** | Componente estacional + eventos | Épica 2 | ✓ Cubierto |
| **FR-034** | Trazo explicable por SKU | Épica 2 (render en E3) | ✓ Cubierto |
| **FR-035** | Aplicar `factor_perdida` con jerarquía | Épica 2 + AR-F05 | ✓ Cubierto |
| **FR-040** | Cantidad sugerida por SKU | Épica 2 | ✓ Cubierto |
| **FR-041** | Mostrar sugerencia en bandeja del coord | Épica 3 | ✓ Cubierto |
| **FR-043** | SKU sin equivalencia → fail-loud | Épica 2 | ✓ Cubierto |
| **FR-050** | Costo total por tienda | Épica 2 | ✓ Cubierto |
| **FR-051** | Recorte sugerido por riesgo de quiebre | Épica 2 (cálculo) + E3 (UI) | ✓ Cubierto |
| **FR-052** | Aceptar/modificar/rechazar recorte | Épica 3 | ✓ Cubierto |
| **FR-053** | Persistir decisión de recorte | Épica 3 | ✓ Cubierto |
| **FR-054** | `disponible = techo − esporádicas` | Épica 2 (cálculo) + E3 (desglose UI) | ✓ Cubierto |
| **FR-060** | Bandeja de tiendas del distrito | Épica 3 | ✓ Cubierto |
| **FR-061** | Estado por tienda + trabajo en pantalla | Épica 3 | ✓ Cubierto |
| **FR-063** | Agregados del distrito | Épica 3 | ✓ Cubierto |
| **FR-064** | Override con razón estructurada | Épica 3 | ✓ Cubierto |
| **FR-065** | Aprobar consolidado y exportar | Épica 3 | ✓ Cubierto |
| **FR-066** | Surfacing pasivo de carry-over | Épica 3 | ✓ Cubierto |
| **FR-067** | Dashboard pedido-vs-recibido | Épica 3 | ✓ Cubierto |
| **FR-073** | Ingesta histórico TFS | Épica 1 | ✓ Cubierto |
| **FR-074** | Visualización cruzada TFS | Épica 4 | ✓ Cubierto |
| **FR-075** | Registro manual de TFS de emergencia | Épica 4 | ✓ Cubierto |
| **FR-080** | Detección estadística residual | Épica 4 | ✓ Cubierto |
| **FR-081** | Candidatos residuales por severidad | Épica 4 | ✓ Cubierto |
| **FR-082** | Configuración factor + gestión candidatos | Épica 4 | ✓ Cubierto |
| **FR-083** | Acumular candidatos confirmados | Épica 4 | ✓ Cubierto |
| **FR-084** | Default global ≈ 0.9% | Épica 2 | ✓ Cubierto |
| **FR-085** | Factor alimenta el motor | Épica 2 | ✓ Cubierto |
| **FR-091** | Registro de excepción mid-cycle | Épica 4 | ✓ Cubierto |
| **FR-092** | Contexto agregado para excepción | Épica 4 | ✓ Cubierto |
| **FR-093** | Decidir y registrar respuesta | Épica 4 | ✓ Cubierto |
| **FR-094** | Notificar al comprador | Épica 4 + AR-Email | ✓ Cubierto |
| **FR-095** | Reflejar compra extraordinaria autorizada | Épica 4 (acopla con FR-054) | ✓ Cubierto |
| **FR-100** | Registrar entrada en bitácora | Épica 3 | ✓ Cubierto |
| **FR-101** | Cadena de cambios en hover | Épica 3 | ✓ Cubierto |
| **FR-102** | Panel "Cambios de hoy" → CSV | Épica 3 | ✓ Cubierto |
| **FR-103** | MDC en logs | Épica 1 | ✓ Cubierto |
| **FR-104** | Bitácora inmutable | Épica 3 + AR-Bitacora | ✓ Cubierto |
| **FR-110** | Export XLSX compatible | Épica 3 | ✓ Cubierto |
| **FR-111** | Enviar export por correo | Épica 3 | ✓ Cubierto |
| **FR-112** | Bitácora de export | Épica 3 | ✓ Cubierto |

### Cobertura NFR

Las NFRs de §9.1–9.12 del PRD están **explícitamente declaradas como cross-cutting**: el documento las trata como criterios de aceptación / definition-of-done dentro de las historias de las épicas a las que aplican (no como historias propias). Solo **NFR-O-1/2/3 (observabilidad)** se mapean explícitamente a una épica dedicada (Épica 5).

| Grupo NFR | Tratamiento | Estado |
|-----------|-------------|--------|
| NFR-E-* (Escala) | DoD transversal; NFR-E-3 en E2; NFR-E-4 en E3 | ✓ |
| NFR-P-* (Performance) | DoD en E3 (UX-DR3) | ✓ |
| NFR-D-* (Disponibilidad) | DoD en E1 (Cloud Run config) + E3 (persistencia de estado) | ✓ |
| NFR-A-* (Auditabilidad) | DoD en E3 (AR-Bitacora) | ✓ |
| NFR-S-* (Seguridad) | DoD en E1 (AR-Auth + Auth0 + MFA) | ✓ |
| NFR-DAT-* (Calidad del dato) | DoD en E1 (AR-SIM-Policy + fail-loud) | ✓ |
| NFR-L-* (Locale) | DoD en E3 (UX-DR7 pipes/formatters) | ✓ |
| NFR-A11Y-* (Accesibilidad) | DoD en E3 (UX-DR8) | ✓ |
| NFR-UX-* (UX no negociable) | DoD en E3 (UX-DR1..6) | ✓ |
| NFR-T-* (Test) | DoD en cada épica (E1 CI/CD, E2 jqwik+PIT+backtesting, E3 contract testing) | ✓ |
| NFR-I-* (i18n) | DoD en E3 (strings en archivo de recursos) | ✓ |
| NFR-O-* (Observabilidad) | Épica 5 (explícito) | ✓ |

### Cobertura de requisitos adicionales (`AR-*` y `UX-DR*`)

El documento epics inventaría 17 `AR-*` derivados de la arquitectura y 8 `UX-DR*` derivados del PRD §9 + project-context, y los mapea a las épicas. Cobertura completa cruzada: `AR-Init` y `AR-CICD` → E1; `AR-F05`, `AR-Batch`, `AR-Hexagonal` → E2; `AR-Grid`, `AR-FE-State`, `AR-Errors`, `AR-Bitacora`, `AR-Contracts`, `AR-Money-JSON` → E3; `AR-Email` → E4. UX-DR1..8 → E3.

### Estadísticas de cobertura

- **Total FRs activos en el PRD:** **56**
- **FRs explícitamente mapeados en `epics.md` (FR Coverage Map):** **56**
- **FRs sin cobertura:** **0**
- **Porcentaje de cobertura:** **100%**
- **FRs en `epics.md` que NO están en el PRD:** **0** (no hay overreach)
- **FRs eliminados (PRD ~~tachado~~) que **erróneamente** aparecen en epics:** **0** (epics excluye correctamente FR-004, FR-015, FR-020..025, FR-042, FR-062, FR-070/071/072, FR-090)

### Hallazgos secundarios (no son gaps de cobertura, son riesgos de implementación)

1. ⚠️ **`epics.md` NO contiene historias detalladas con criterios de aceptación.** Solo describe el alcance de cada épica + lista de FRs cubiertos + notas de implementación. La **descomposición fina en stories** (con AC verificables, estimación, dependencias entre stories) **aún no existe** — esto es esperable en BMad cuando se ha hecho solo epic breakdown (Steps 1-2 del flujo de epics, según el frontmatter `stepsCompleted: [1, 2]`), pero **es la siguiente entrega obligatoria antes de Phase 4**.
2. ⚠️ **OQ abiertas referenciadas en épicas pero sin historia de resolución explícita:**
   - **OQ-108 (cold-start < 12 meses):** FR-030 marca la tienda como "sin sugerencia del motor" pero no hay historia que diseñe el flujo de manejo (alternativa: bootstrap / agregado distrital).
   - **OQ-123 (umbral residual ≥ 2σ):** descrito en FR-080 como "calibrable en piloto", no aparece como historia en Épica 4 (admin config).
   - **D-5 (lista de eventos comerciales):** FR-006 está cubierto en Épica 1, pero **la lista en sí depende de Operaciones** — no es bloqueante de implementación si se modela como dato configurable, pero el contenido viene fuera del sistema.
3. ⚠️ **Formato y cadencia de los reportes del owner (FR-016, FR-017)** sigue pendiente. Las stories de ingesta no pueden cerrarse de DoD sin esta especificación. **Bloqueante operativo, no de planeación.**
4. ⚠️ **NFR-T-5 (backtesting)** requiere un dataset histórico de ventas con corte semanal que el owner aún no ha entregado. Sin él, Épica 2 puede construirse pero **no validarse** antes del go-live. Acoplado con R-9 (baseline pendiente).
5. ⚠️ **Sin glosario adicional de `STATUS`/`TRAN_CODE`** (riesgo abierto en `project-context.md`): afecta historias de ingesta de Épica 1 (FR-012). El epics lo asume implícito; la política provisional (`StatusDesconocidoException` fail-loud) está cubierta por AR-SIM-Policy y por la convención general de fail-loud, pero **el glosario operativo** falta.

**Conclusión Step 3:** la cobertura FR→épica es **100% completa y simétrica** (sin FRs faltantes, sin FRs huérfanos en epics). El documento `epics.md` está bien construido como **descomposición de épicas**. Sin embargo, la **descomposición a nivel de stories con criterios de aceptación verificables aún no existe** — es la siguiente entrega obligatoria antes de poder iniciar Phase 4. Los riesgos restantes son de **insumos externos (reportes del owner, glosarios, decisiones operativas)** y **calibración con pilotaje**, no de cobertura de requisitos.

---

## Step 4 — UX Alignment Assessment

### Estado del documento UX

**✅ ENCONTRADO Y COMPLETO.** `_producto/planning-artifacts/ux-design-specification.md` (1074 líneas) con frontmatter `status: 'complete'`, `stepsCompleted: [1..14]`. Documento generado a partir de `project-context.md` + PRD + addendum + `architecture.md` + `epics.md` (orden secuencial correcto del pipeline BMad). Complementado por `ux-design-directions.html` (mockups de dirección visual).

### UX ↔ PRD — Alineación

| Aspecto PRD | Cobertura en UX | Veredicto |
|-------------|-----------------|-----------|
| Usuarios principales (3 coords + admin) | Sección "Usuarios Objetivo" identifica explícitamente a Carlos/Marco/Eduardo + admin; correctamente excluye al jefe en v1 y al comprador (downstream) | ✅ Alineado |
| Objetivo O1 (9h → <2h) | "Meta de UX medible" lo declara como North Star del journey héroe (UJ-A) | ✅ Alineado |
| UJ-2-TB Consolidación (PRD §7.7) | UJ-A "Consolidación distrital semanal" — flowchart Mermaid completo, deriva explícitamente de UJ-2-TB | ✅ Alineado |
| UJ-3-TB Excepción mid-cycle (PRD §7.6) | UJ-B "Excepción mid-cycle" — flowchart Mermaid, deriva de UJ-3-TB, canal único correo | ✅ Alineado |
| FR-061 (coord trabaja en pantalla, no Excel) | "Defining Experience": la celda como unidad de decisión + la tienda como unidad de compromiso | ✅ Alineado |
| FR-067 dashboard pedido-vs-recibido | `DashboardBrecha` custom component definido | ✅ Alineado |
| FR-066 carry-over surfacing pasivo | "Visibilidad pasiva sin acoso" + `ChipEstadoTienda` con estado `carry-over` | ✅ Alineado |
| FR-064 override con razón estructurada | `OverrideEditor` con fricción asimétrica ±15% + dropdown de razones tipificadas | ✅ Alineado (con refinamiento — ver hallazgos) |
| FR-030 cold-start (<12 meses) | `EstadoColdStart` componente "honesto" + andamios con semanas crudas; mensaje "sin sugerencia del motor — aún aprendo esta tienda" | ✅ Alineado |
| FR-110 export = output | Explícito: "Excel deja de ser herramienta de trabajo, pasa a ser handoff" | ✅ Alineado |
| NFR-P-1 (recálculo <100ms) + autosave | "Effortless Interactions": Enter=aceptar, recálculo optimista <100ms, autosave tipo Google Docs | ✅ Alineado |
| NFR-UX-2 (deshacer en lugar de modal) | Toast "Deshacer" 8s explícito en "Feedback Patterns" | ✅ Alineado |
| NFR-UX-3 (errores 3 niveles) | "Feedback Patterns" — inline / negocio / sistema con código de incidente copiable | ✅ Alineado |
| NFR-L-1..6 locale es-MX | Sección "Pipes es-MX" + ejemplos `1,250 cajas`, `Sem 12 · 16-22 mar 2026`, `$1,250.00` | ✅ Alineado |
| NFR-A11Y-1..3 (WCAG 2.1 AA) | "Accessibility Considerations" + `aria-label` contextual + teclado completo + `:focus-visible` 2px | ✅ Alineado |
| FR-100..104 bitácora visible | `CadenaBitacora` + panel "Cambios de hoy" exportable a CSV | ✅ Alineado |
| Plataforma web responsive (PRD §11.7 sin nativa) | "Platform Strategy" reitera "Sin app nativa"; viewport laptop 1366×768 dimensionado | ✅ Alineado |

### UX ↔ Arquitectura — Alineación

| Decisión arquitectónica (`architecture.md`) | Soporte/uso en UX | Veredicto |
|---------------------------------------------|-------------------|-----------|
| AR-Init (Angular CLI 21.2.x, strict, zoneless) | "Platform Strategy" → SPA Angular en Firebase Hosting | ✅ |
| AR-FE-State (Signals + RxJS solo para asincronía + zoneless) | Implícito en "Recálculo optimista <100 ms" (mecanismo no especificado, pero compatible) | ✅ |
| AR-Grid (AG Grid Community + Material/CDK fallback) | UX selecciona explícitamente **AG Grid Community** + **PrimeNG** como design system tematizado a OD; documenta lo que Community **no trae** (row-grouping custom, master/detail) y lo lista como componentes custom | ✅ con divergencia menor (ver hallazgo 1) |
| AR-Routing (Router con guards por rol + LOCALE_ID='es-MX') | "Navigation Patterns": app-shell con cambio de modo (distrital/territorial/dashboard) sin perder estado, scoping territorial | ✅ |
| AR-Auth (Auth0 OIDC/JWT + MFA admin + scoping territorial) | UJ-A inicia con "Login (Auth0)"; "Concurrencia: 3 coordinadores con territorios disjuntos → sin colisión por construcción" | ✅ |
| AR-Errors (RFC 7807 ProblemDetail + interceptor → 3 niveles UX) | "Sistema de errores 3 niveles" — `http-error.interceptor` traduce ProblemDetail a inline/negocio/sistema | ✅ |
| AR-Bitacora (append-only table) | `CadenaBitacora` consume bitácora inmutable; cadena hover quién-qué-por qué | ✅ |
| AR-Money-JSON (BigDecimal como string) | Pipes es-MX en `frontend/shared` consumen JSON tipado vía modelos `view/` | ✅ |
| AR-Contracts (Spring Cloud Contract antes de implementar) | UX dice "Contrato antes de implementar; los componentes de datos consumen el contrato compartido" | ✅ |
| AR-Email (SendGrid) | UJ-B "Notificar comprador (SendGrid)" | ✅ |
| AR-Hexagonal (frontera `dominio/forecasting`) | No aplica directamente al frontend; UX no contradice | ✅ |
| Listado de componentes custom en epics (UX-DR1..8) | Cada UX-DR materializado como componente concreto (`OverrideEditor`, `PanelTrazo`, `BarraPresupuestoTienda`, `ChipEstadoTienda`, `EstadoColdStart`, `DashboardBrecha`, `SelectorSemanaISO`, pipes es-MX) | ✅ |

### Hallazgos / brechas de alineación

1. **AG Grid Community vs Material fallback — divergencia menor de design system.** `architecture.md` lista AG Grid Community como **primario** con Angular Material + CDK como **fallback**. La UX selecciona **PrimeNG + AG Grid Community** y descarta implícitamente Material/CDK. Esto es **una decisión más restrictiva que la arquitectura, no contradictoria**; pero conviene reconciliar en `architecture.md` para que el equipo no quede ambivalente. **Acción:** confirmar que la línea oficial es "PrimeNG + AG Grid Community", actualizar `architecture.md` para reflejarlo. Licenciamiento AG Grid Community sigue como decisión menor abierta (ya señalada en `architecture.md` §"Análisis de brechas").

2. **Fricción asimétrica ±15% — constante de UX sin FR/NFR codificándola.** La UX introduce un **umbral de banda ±15%** para distinguir override-en-banda vs override-fuera-de-banda, con UX distinta (motivo de 1 toque vs confirmación consciente + motivo obligatorio). El PRD habla de override + razón estructurada (FR-064) pero **no codifica el umbral**. **Acción:** el umbral debe **(a)** quedar como parámetro configurable en datos maestros (admin), o **(b)** declararse como constante de diseño en el `architecture.md` / catálogo de constantes UX. Sin esto, el equipo de implementación puede ponerlo cableado y dificultar calibración en piloto.

3. **Lista de razones estructuradas — UX asume taxonomía mínima.** La UX nombra al menos dos razones explícitas (`"corrijo dato SIM sucio"`, `"no confío aún"`) que considera necesarias para no contaminar la señal de calibración. El PRD/epics tratan la lista de razones como **configurable por admin** (FR-064 + datos maestros §10.5). **Acción:** crear story explícita en Épica 1 (datos maestros) que prepoble la lista de razones con la taxonomía mínima sugerida por la UX, no dejarla 100% en blanco al go-live.

4. **Aprobación por tienda como unidad de compromiso — refina FR-065.** La UX introduce una **doble unidad de compromiso**: aprobar tienda (gesto único por tienda) + aprobar consolidado (gesto único distrital). FR-065 solo dice "aprobar el consolidado del distrito". Esto es un **enriquecimiento** de UX consistente con el modelo "celda = decisión, tienda = compromiso, distrito = export". **Acción:** confirmar que la story de aprobación en Épica 3 contempla ambos niveles (per-tienda y per-distrito), o documentar la decisión en `epics.md` / acta de UX para que no se pierda al desglosar.

5. **Tokens de marca Office Depot — supuesto aún por reconciliar.** Commit `31dcf07` (2026-05-28) registra: *"institucional Office Depot (tokens derivados como supuesto ⚠️ a reconciliar con el PDF de rebranding)"*. La UX define color/tipografía OD pero está **pendiente la validación contra el PDF oficial de rebranding**. **Acción:** reconciliar tokens antes de implementar `Fase 1` del Implementation Roadmap UX; no es bloqueante de la fundación pero sí de cualquier UI visible.

6. **Pipeline WDS (`design-artifacts/{A..E}`) vacío.** El proyecto tiene la estructura del pipeline WDS de Freya pero no hay artefactos en ninguna de las 5 fases. La especificación UX vive en `_producto/planning-artifacts/ux-design-specification.md` (pipeline BMad, no WDS). **Acción:** simplemente declarar que la fuente única de UX es el archivo de `_producto/planning-artifacts/`, y limpiar/usar `design-artifacts/` para entregables visuales (mockups, design system exportado) si se desea — o eliminar si se decide no usar.

### Warnings

- ⚠️ **Sin documento de design system / tokens exportable.** La UX define los componentes y patrones pero no produce un archivo de design tokens (CSS variables, paleta, escalas tipográficas) listo para consumo de frontend. Esto es una **brecha de entregable** que va a aparecer en la primera story de UI (`MatrizBandeja` en Épica 3): el equipo no podrá empezar sin tokens. **Acción:** generar `tokens.scss` o equivalente como parte de la story de inicialización del frontend (E1), con valores derivados de la UX hasta que el PDF de rebranding cierre.
- ⚠️ **Sin wireframes de baja/alta fidelidad en `design-artifacts/E-Development/`.** La UX especifica componentes verbalmente y con mermaid flowcharts; **no hay screen mockups**. Para una matriz tienda × SKU con densidad alta en 1366×768, ver mockups antes de implementar reduce riesgo de re-trabajo. **Acción:** producir al menos 3-5 wireframes anclados (bandeja distrital, bandeja territorial, dashboard pedido-vs-recibido, override en banda, override fuera de banda) antes de la primera story de Épica 3.

**Conclusión Step 4:** la UX está **completa, alineada con PRD y arquitectura**, sin contradicciones críticas. Las 6 brechas detectadas son refinamientos de bajo a medio impacto que pueden cerrarse durante las primeras 2 stories de Épica 1 + setup del frontend de Épica 3 (tokens, prepoblado de razones, decisión del umbral ±15%, reconciliación de marca, wireframes). Ninguna brecha bloquea la aprobación general.

---

## Step 5 — Epic Quality Review

### A. Foco en valor de usuario (no milestone técnico)

| Épica | Título | ¿Valor de usuario? | Veredicto |
|-------|--------|--------------------|-----------|
| **E1** | Fundación de la plataforma e ingesta de datos confiable | **Híbrida**. El cuerpo describe "El **administrador** inicia sesión con identidad individual, gobierna los datos maestros... e ingesta + valida todos los archivos canónicos". Hay valor para el rol **Administrador** (gobierno de catálogos + ingesta confiable). Pero también incluye "walking skeleton sobre el pipeline de CI/CD" → componente técnico. | 🟡 **Aceptable con observación.** El framing centra al admin como usuario, lo cual es legítimo. La parte técnica (walking skeleton) está justificada como precondición de despliegue, pero debería **descomponerse** en (a) historias de admin gobernando catálogos y (b) historias técnicas de fundación. |
| **E2** | Motor de pronóstico y cálculo de la sugerencia de pedido | **El motor no es operado por un humano directamente** — el admin lo dispara manualmente (trigger manual + `min-instances=1`) y los resultados los consumen las épicas 3-4. Pero el documento argumenta que **"persiste las sugerencias en el `mart`"** y que es **"Fase 1 (Predicción), corazón de v1"**. El valor es para todos los usuarios downstream, pero ningún usuario humano interactúa con esta épica de forma autónoma. | 🟠 **Major Issue.** Esta épica se acerca a un milestone técnico ("Motor de pronóstico"). Para hacerla pasar el filtro de valor de usuario, una de dos: (a) **incluir** explícitamente la story "Admin dispara batch manual de forecast y ve resultados en el dashboard de operación" como entregable visible al admin; (b) **fusionar** con Épica 3 cuando la bandeja distrital se construya, declarando que E2 es un sub-objetivo técnico de E3. El documento ya menciona "Funciona desde go-live con el default global — independiente de la UI de configuración del factor (Épica 4)", pero no enuncia un usuario consumiendo el output de E2 sin pasar por E3 primero. |
| **E3** | Bandeja de consolidación distrital del coordinador | "El **coordinador** abre su bandeja semanal en pantalla, ve las ~38 tiendas... aplica overrides... aprueba el consolidado y **exporta el Excel compatible** al comprador por correo. Entrega el objetivo O1 — el ahorro de 9h → <2h por coordinador por semana." | ✅ **Excelente.** Valor claro, medible, atado al objetivo principal del producto. |
| **E4** | Bandeja territorial — excepciones, factor de pérdida y análisis TFS | "El **coordinador territorial** registra y resuelve excepciones mid-cycle... configura el factor de pérdida por tipo de papel y revisa candidatos residuales..." | ✅ **Excelente.** Tres capacidades de valor para el mismo usuario, agrupadas por la unidad mental "territorial". |
| **E5** | Automatización y observabilidad operativa | "El **administrador** monitorea la operación... en un dashboard y **recibe alertas automáticas**; el batch del motor corre programado..." | ✅ **Bueno.** Valor explícito para el admin (dashboard + alertas) + automatización del batch. Reduce el toque manual de E2 a su forma productiva. |

### B. Independencia y dependencias entre épicas

✅ **Sin forward dependencies.** El documento declara explícitamente al final: *"Cada épica es autónoma y entrega valor independientemente; ninguna requiere una épica posterior para funcionar."*

Verificación cruzada:

| Épica | Depende de | ¿Forward dep? |
|-------|------------|---------------|
| E1 | — (standalone) | ✅ No |
| E2 | E1 (datos maestros + ingesta) | ✅ Backward, OK |
| E3 | E1 + E2 (sugerencias persistidas) | ✅ Backward, OK |
| E4 | E1 + E2 + E3 (auth, bitácora, factor, notif) | ✅ Backward, OK — texto: "reutiliza auth, bitácora y notificación por correo de épicas previas" |
| E5 | E1 (logging infra) + automatiza E2 | ✅ Backward, OK |

**Hallazgo positivo:** la nota sobre E2 ("Funciona desde go-live con el default global — independiente de la UI de configuración del factor (Épica 4)") demuestra que las dependencias entre E2 y E4 están deliberadamente **invertidas hacia el camino sano** — E2 funciona sin E4, no al revés.

### C. Story sizing y AC — **GAP CRÍTICO**

🔴 **VIOLACIÓN CRÍTICA: `epics.md` NO contiene historias detalladas.**

El frontmatter del documento confirma el estado parcial: `stepsCompleted: [1, 2]` (solo dos pasos del workflow `bmad-create-epics-and-stories`). El documento cubre:

- ✅ Requirements Inventory (FRs / NFRs / AR-* / UX-DR* extraídos)
- ✅ FR Coverage Map (mapeo FR→épica)
- ✅ Epic List con descripción de alto nivel + FRs cubiertos + notas de implementación
- ❌ **NO hay descomposición a stories** con título, persona, acción, beneficio
- ❌ **NO hay criterios de aceptación** (Given/When/Then o equivalente) por story
- ❌ **NO hay estimación o priorización intra-épica** de stories
- ❌ **NO hay timing de creación de tablas/migrations por story**
- ❌ **NO hay archivos `stories/` o `epic-*/story-*.md` separados**

**Impacto:** sin stories detalladas, **Phase 4 (Implementation) NO puede iniciar bajo el flujo BMad estándar.** El equipo de dev necesita historias con AC para:
1. Tener una unidad de trabajo asignable.
2. Validar implementación contra criterios verificables.
3. Hacer test design (Murat/TEA) acoplado a historias.
4. Acordar contratos antes de implementar (NFR-T-4 + AR-Contracts).

**Severidad:** crítica — bloquea Phase 4. Es **el bloqueante #1** del veredicto de readiness.

### D. Calidad de los AC implícitos en el PRD

Como mitigación parcial, observo que **el PRD sí incluye criterios de aceptación explícitos en algunos FRs críticos** (resuelto en update 2026-05-26b):

- ✅ **FR-030** tiene AC observable: "para una tienda con ≥ 12 meses de histórico... el motor produce un valor de demanda esperada por tienda × SKU venta para el ciclo objetivo, acompañado del trazo de FR-034..."
- ✅ **FR-035** tiene AC observable: "para una tienda con demanda de venta esperada `V` y factor resuelto `f`, la demanda ajustada cumple `demanda_ajustada = V × (1 + f)` sobre la demanda de venta, antes de la conversión a insumo y la cuantización; el trazo explicable declara `V`, `f`, el nivel del que provino `f` y el resultado."

Pero la mayoría de FRs siguen siendo enunciados de capacidad, no AC ejecutables. Faltan AC para FR-014/016/017 (ingesta), FR-040 (cálculo de sugerencia completo), FR-051 (recorte), FR-064 (override taxonomía), FR-067 (dashboard pedido-vs-recibido), FR-110 (export con tolerancia de comparación bit-a-bit), etc.

### E. Starter Template Story

✅ **Sí declarado.** `epics.md` enuncia: **"AR-Init: ... — PRIMERA story de implementación"**. Esto cumple el requisito del workflow create-epics-and-stories ("if Architecture specifies starter template, Epic 1 Story 1 must be 'Set up initial project from starter template'"). El comando exacto está en `architecture.md` §"Evaluación de Starter Templates".

⚠️ **Pero la story como tal no existe como artefacto independiente** (es una nota dentro de E1). Cuando se descomponga, la story de init debe documentar:
- AC1: monorepo `/backend` + `/frontend` creado con los comandos exactos del arch.
- AC2: `pom.xml` con deps listadas + pin de versión exacto contra imagen base.
- AC3: `ng new` con flags `--style=scss --routing=true --strict=true --ssr=false`.
- AC4: walking skeleton ejecutable (`mvn spring-boot:run` + `ng serve` levantan, hello-world end-to-end).
- AC5: pipeline `cloudbuild.yaml` con stages mínimas.

### F. Greenfield indicators

| Indicador esperado | Presente en planeación |
|--------------------|------------------------|
| Story de initial project setup | ✅ AR-Init en E1 (sin story file) |
| Configuración de dev environment | ✅ Architecture §"Integración con el flujo de desarrollo" |
| CI/CD pipeline temprano | ✅ AR-CICD mapeado a E1 |
| Migration timing (Flyway, una por PR) | ✅ AR-Migrations en `architecture.md` |
| Tablas creadas cuando se necesitan | ⚠️ No verificable sin stories — convención `architecture.md` lo respeta, pero sin stories no hay timing por feature |

### Resumen de hallazgos por severidad

#### 🔴 Críticas

1. **AUSENCIA DE STORIES.** `epics.md` se detiene en el nivel de épica + FR Coverage Map. No hay descomposición a stories con AC. **Bloquea Phase 4.** Acción: completar el workflow `bmad-create-epics-and-stories` (pasos 3-N) o usar `bmad-create-story` por épica para producir los archivos de historias. Hugo (comprador piloto) y Murat (TEA) están bloqueados hasta tenerlas.

#### 🟠 Mayores

2. **Épica 2 borderline como milestone técnico.** El motor de forecast no tiene usuario humano interactivo en su definición actual. Recomendación: (a) explicitar story "Admin dispara batch manual + ve resultados básicos" como entregable de E2; o (b) declarar E2 como sub-objetivo técnico fusionado en la planeación de E3 (con tracking interno pero un solo epic visible). La opción (a) preserva la independencia entre epics; (b) cumple mejor el filtro de valor de usuario.

3. **Mezcla de admin user-value + infra técnica en E1.** Recomendación: al descomponer E1 en stories, separar (a) stories de **setup técnico** (AR-Init, AR-DB, AR-Migrations, AR-CICD, AR-Auth) de (b) stories de **valor de usuario admin** (gobierno de catálogos, ingesta + validación con feedback). Las primeras son enablers; las segundas son lo que el admin reconoce como "puedo hacer X".

4. **Falta AC verificable en la mayoría de FRs** (solo FR-030 y FR-035 lo tienen). Recomendación: durante la descomposición a stories, **derivar AC verificables** para cada FR, especialmente para ingesta (FR-014/016/017), cálculo de sugerencia completo (FR-040), recorte (FR-051), override taxonomía (FR-064), dashboard (FR-067) y export (FR-110, incluye fixture de comparación con tolerancia documentada).

#### 🟡 Menores

5. **Umbral fricción asimétrica ±15% no codificado** (heredado del Step 4) — debe quedar como constante de UX configurable o parámetro admin antes de implementar `OverrideEditor`.

6. **Lista de razones estructuradas vacía al go-live** — requiere story de prepoblado en E1 con la taxonomía mínima que la UX necesita.

7. **Wireframes ausentes** — recomendado producir 3-5 wireframes ancla antes de la primera story de UI en E3 (`MatrizBandeja`).

8. **Design tokens OD no exportados** — bloquea cualquier story de UI hasta tener `tokens.scss` (o equivalente) generado. Reconciliar contra PDF de rebranding cuando exista.

9. **Glosario `STATUS` / `TRAN_CODE` pendiente** — `project-context.md` documenta política provisional (fail-loud con `StatusDesconocidoException`); el glosario operativo sigue por pedirse a Operaciones.

### Compliance checklist por épica

| Criterio | E1 | E2 | E3 | E4 | E5 |
|----------|----|----|----|----|----|
| Entrega valor de usuario | 🟡 mixto | 🟠 borderline | ✅ | ✅ | ✅ |
| Funciona independientemente | ✅ | ✅ | ✅ | ✅ | ✅ |
| Stories apropiadamente dimensionadas | ❌ sin stories | ❌ sin stories | ❌ sin stories | ❌ sin stories | ❌ sin stories |
| Sin forward dependencies | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tablas creadas cuando se necesitan | ❓ a verificar en stories | ❓ | ❓ | ❓ | ❓ |
| AC claros | ❌ | 🟡 parcial (FR-030/035) | ❌ | ❌ | ❌ |
| Trazabilidad a FRs | ✅ | ✅ | ✅ | ✅ | ✅ |

**Conclusión Step 5:** la arquitectura de épicas es **sólida en independencia, cobertura y trazabilidad**, pero **el documento no completó la descomposición a stories** — paso obligatorio antes de Phase 4. Adicionalmente, E2 debería reforzarse con valor de usuario explícito (admin dispara batch + ve resultado), E1 debería separar enablers técnicos de stories admin, y los AC a nivel de story siguen pendientes para casi todos los FRs.

---

## Summary and Recommendations

### Overall Readiness Status

# 🟠 NEEDS WORK

El cuerpo de planeación (PRD, Architecture, UX, Epics) es **maduro, trazable y consistente entre sí**. La cobertura FR→épica es **100%** y la arquitectura está formalmente *READY FOR IMPLEMENTATION* en su propio frontmatter. Sin embargo, **el flujo BMad estándar requiere stories detalladas con criterios de aceptación verificables antes de iniciar Phase 4**, y esa descomposición **no está hecha**. Esa única brecha — junto con un puñado de insumos externos y refinamientos UX — es lo que separa al proyecto del veredicto `READY`.

**Lo que sí está listo:**
- ✅ PRD `status: final`, validado 4 rondas, Reviewer Gate Grade Good, 0 Critical / 0 High abiertos.
- ✅ Architecture `status: complete`, `READY FOR IMPLEMENTATION`, 16/16 checklist ✅, sin brechas críticas.
- ✅ UX `status: complete`, 14/14 pasos, journeys derivados explícitamente del PRD, componentes mapeados a la arquitectura.
- ✅ Epics breakdown con FR Coverage Map 100% completo, sin forward dependencies, sin FRs huérfanos.
- ✅ Convenciones de datos, testing, stack y entorno cerradas en `project-context.md`.

**Lo que falta:**
- ❌ Stories detalladas con AC verificables (`bmad-create-epics-and-stories` se detuvo en step 2 de N).
- ⚠️ Insumos del owner (formato y cadencia de reportes de ventas semanales + ALLOC/TFS).
- ⚠️ Insumos operativos (glosario `STATUS`/`TRAN_CODE`, lista de razones estructuradas, lista de eventos comerciales).
- ⚠️ Activos de UX (tokens OD reconciliados con PDF de rebranding, wireframes ancla).
- ⚠️ Precondiciones de pilotaje (spike R-8 forecast Java, backtesting R-9, baseline de quiebre/sobre-inventario).

---

### Critical Issues Requiring Immediate Action

#### 🔴 #1 — Descomponer Épicas a Stories con criterios de aceptación

**Bloqueante #1 de Phase 4.** Los 5 epics están descritos a alto nivel pero sin stories. Sin stories, el dev no puede tomar trabajo asignable, el TEA (Murat) no puede diseñar tests, y los contratos Spring Cloud Contract no se pueden definir antes de implementar.

**Acción concreta:** invocar `bmad-create-story` (o continuar el workflow `bmad-create-epics-and-stories` desde el step 3) para producir, por épica:
- Una historia por capacidad o sub-capacidad (típicamente 4-8 historias por épica).
- Para cada historia: persona + acción + beneficio (formato BMad), AC verificables, dependencias intra-épica, tablas/migrations que la historia crea, story points / tamaño relativo.
- **Primera historia OBLIGATORIA en E1:** "Inicializar monorepo con Spring Initializr + Angular CLI" (los comandos exactos están en `architecture.md`).

**Output esperado:** archivos `_producto/planning-artifacts/epics/epic-1/story-N.md` (o estructura equivalente) — entre 25 y 40 stories totales estimadas para v1.

#### 🔴 #2 — Refinar Épica 2 para anclar valor de usuario

El motor de forecast como épica autónoma roza el milestone técnico. **Acción:** agregar a la descomposición de E2 al menos una historia visible al admin (ej. "Admin dispara batch manual desde UI de administración y consulta resultados básicos del último run en el `mart`") para que la épica produzca valor reconocible por un usuario humano, no solo persistencia para consumidores downstream.

#### 🔴 #3 — Fijar formato y cadencia de los reportes del owner

**Bloqueante operativo de E1 (stories de ingesta FR-016, FR-017).** El owner aporta ventas semanales y ALLOC/TFS como archivos, sin BigQuery/RMS en v1. **Acción:** Jonathan especifica el formato exacto (columnas, encoding, naming convention) y la cadencia (lunes 06:00 CDMX, semana ISO N−1, etc.) antes de que arranque la implementación de las stories de ingesta. Documentar en `_producto/planning-artifacts/` como anexo.

---

### Issues Mayores a resolver en paralelo (no bloquean inicio si se delimitan stories)

4. **Reconciliar tokens de diseño OD** con el PDF de rebranding antes de la primera story de UI en E3.
5. **Codificar el umbral ±15%** de fricción asimétrica (parámetro admin o constante de UX/arch).
6. **Story de prepoblado de razones estructuradas** en E1 con la taxonomía mínima de la UX.
7. **Glosario operativo** `STATUS` (ALLOC.STATUS, ALLOC.STATUS_ORDEN, TFS.STATUS) y `TRAN_CODE` (ya hay `DECODE` autoritativo) — pedirlo a Operaciones para acotar `StatusDesconocidoException`.
8. **Lista oficial de eventos comerciales** (D-5) — confirmar con Operaciones para alimentar FR-006 / FR-033.
9. **AC verificables para FRs críticos** sin AC todavía: FR-014/016/017 (ingesta), FR-040 (cálculo completo), FR-051 (recorte), FR-064 (override), FR-067 (dashboard), FR-110 (export con tolerancia bit-a-bit).
10. **Wireframes ancla** (3-5) para E3 antes de la primera story de UI: bandeja distrital, bandeja territorial, dashboard pedido-vs-recibido, override en banda, override fuera de banda.

---

### Issues Menores (refinamientos)

11. Reconciliar AG Grid Community como decisión primaria (no como "fallback") en `architecture.md`.
12. Aclarar que `design-artifacts/` (pipeline WDS) no es la fuente — la fuente es `_producto/planning-artifacts/ux-design-specification.md`. Limpiar o reutilizar `design-artifacts/` para mockups exportados.
13. Documentar la "aprobación por tienda" como sub-decisión de FR-065 cuando se desglose la story de aprobación en E3.
14. Exportar `tokens.scss` (o equivalente) en la story de inicialización del frontend de E1.

---

### Recommended Next Steps (orden cronológico)

1. **Esta semana — Producir stories:** completar `bmad-create-epics-and-stories` (pasos 3-N) o invocar `bmad-create-story` para cada épica. Empezar por **E1 Story 1 (init del monorepo)** y producir las 6-8 historias siguientes de E1 (datos maestros + ingesta + walking skeleton).
2. **Esta semana — Cerrar insumos del owner:** Jonathan documenta formato + cadencia de reportes de ventas y ALLOC/TFS.
3. **Próxima semana — Cerrar insumos operativos:** pedir glosario `STATUS`/`TRAN_CODE`, lista de eventos comerciales (D-5) y lista mínima de razones estructuradas a Operaciones.
4. **Próxima semana — Wireframes:** producir los 3-5 wireframes ancla de E3 antes de descomponer E3 en stories detalladas.
5. **Próximas 2 semanas — Spike Java forecast (R-8):** ejecutar el spike de 2 semanas sobre 1 tienda × 12 meses de histórico real para validar Smile / Commons Math y descartar la necesidad de RFC para Python.
6. **Próximas 2 semanas — Sesión Finanzas (R-7):** agendar para acordar criterio formal (o confirmar que el criterio pragmático YoY del owner es suficiente).
7. **Sprint 1 (asumiendo stories ya listas):** ejecutar E1 Story 1 (init) + Story 2 (esquema BD + Flyway baseline) + Story 3 (auth Auth0). Walking skeleton ejecutable.
8. **Sprint 2:** E1 Stories de ingesta (FR-010..013) + master data CRUD. Cierra E1.
9. **Sprint 3-4:** E2 motor + golden datasets + backtesting. Cierra MAPE/WAPE evidence base.

---

### Métricas del assessment

| Métrica | Valor |
|---------|-------|
| FRs activos en PRD | 56 |
| FRs cubiertos en epics | 56 (100%) |
| FRs sin cobertura | 0 |
| FRs con AC verificable explícito | 2 (FR-030, FR-035) — el resto se infiere de la descripción del FR |
| NFRs activos en PRD | ~37 |
| NFRs explícitamente mapeados a una épica | NFR-O-1..3 → E5 (el resto son cross-cutting / DoD) |
| Épicas definidas | 5 |
| Épicas con forward dependencies | 0 |
| Épicas que delivery valor de usuario claro | 3 (E3, E4, E5); 1 mixta (E1); 1 borderline (E2) |
| Stories detalladas escritas | **0** ← bloqueante |
| Critical findings | 3 |
| Major findings | 7 |
| Minor findings | 4 |
| Documentos críticos faltantes (PRD/Arch/UX/Epics) | 0 — todos presentes y completos en su scope |

---

### Final Note

Este assessment identificó **14 hallazgos en 3 categorías de severidad** (3 críticos, 7 mayores, 4 menores) a través de 5 dimensiones (Discovery, PRD, Cobertura, UX, Calidad de Épicas).

El proyecto **no está listo para iniciar Phase 4 con el workflow BMad estricto**, pero la distancia es **claramente recorrible en 1-2 semanas** de trabajo de planeación adicional: producir las stories, cerrar formato de reportes del owner, reconciliar tokens de marca, y solicitar los glosarios operativos pendientes.

**Lo positivo importa:** el cuerpo de planeación entre PRD, Architecture, UX y Epics es **internamente consistente, completamente trazable y arquitectónicamente sólido**. No hay decisiones contradictorias entre los cuatro documentos, no hay FRs huérfanos, no hay forward dependencies entre épicas, y los riesgos están explícitamente dueños y mitigaciones definidas. Es un punto de partida sano para implementación una vez resuelto el #1.

Estos hallazgos pueden usarse para mejorar los artefactos antes de proceder, o pueden documentarse como riesgos asumidos si se decide arrancar implementación con stories generadas on-the-fly por historia (modo Agile más relajado) — aunque esta segunda vía rompería la disciplina BMad de "contratos antes de implementar" (NFR-T-4 + AR-Contracts) y debería ser una decisión consciente del owner.

---

**Assessor:** Claude (skill `bmad-check-implementation-readiness`)
**Fecha:** 2026-05-28
**Versión del PRD evaluado:** 2026-05-26b (4ª ronda)
**Versión de Architecture evaluado:** 2026-05-26 (status complete, completedAt 2026-05-27)
**Versión de UX evaluado:** 2026-05-27 (status complete)
**Versión de Epics evaluado:** stepsCompleted [1, 2] (descomposición de stories pendiente)

