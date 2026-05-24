---
title: Addendum — PRD insumos-odemas
prd: prd.md
created: 2026-05-24
language: es-MX
---

# Addendum — PRD insumos-odemas

Material que el owner ha pedido conservar pero que no pertenece al cuerpo principal del PRD: detalle histórico, alternativas consideradas y rechazadas con razonamiento, sizing y contexto que servirá a `architecture.md` y a la solución técnica downstream.

---

## A1. UJ-1 AS-IS — Cálculo quincenal del jefe (referenciado desde §7.4 del PRD)

Este journey está fuera del alcance de v1 (el jefe no opera el sistema, §11.11), pero se conserva como referencia del flujo real que sigue ocurriendo.

1. Llega el ciclo quincenal asignado al servicentro (semana 1-y-3 o 2-y-4 ISO, según `Presupuesto-tiendas.csv`).
2. Jefe deja la atención al mostrador y se traslada a la bodega del servicentro.
3. Hace conteo físico de SKUs de papel. Bajo presión, puede saltar SKUs o estimar a ojo.
4. Vuelve a la computadora, abre SIM, captura bajas observadas. Si el conteo fue impreciso, el dato queda sucio en SIM.
5. Abre el Excel corporativo de solicitud.
6. Estima a ojo en función de la temporada cuánto papel necesita para los próximos ~14 días por SKU.
7. Llena el Excel y lo envía por correo al coordinador.
8. (Aguas abajo) el coordinador consolida; el comprador genera la OC.

**Tiempo total:** hasta 6 horas por quincena. **Dolores asociados:** D-jefe-1..5 (§3-bis del PRD).

**Lo que v1 cambia para este flujo:** nada directamente. El jefe lo sigue ejecutando igual. Lo que cambia es lo que ocurre con su Excel cuando llega al coordinador (UJ-2-TB, §7.7 del PRD).

**Lo que v1 podría cambiar en v1.1 o v2:** reintroducir conteo asistido (C-3) + UI para el jefe + integración R/W con SIM. Trade-off epistémico documentado en §11.16 del PRD: v1 conscientemente NO ataca esta causa raíz.

---

## A2. Opciones de integración con SIM consideradas y rechazadas (referenciado desde §10.4 del PRD)

La decisión cerrada del owner es **(C) Coexistencia con lectura unidireccional**. Las dos alternativas evaluadas y rechazadas:

### (A) Integración bidireccional (read + write) con saneamiento progresivo — RECHAZADA

**Descripción:** el sistema lee saldos de SIM como fuente de verdad inicial; cuando el conteo asistido del jefe (C-3) genera bajas reconciliadas, las escribe a SIM; SIM se vuelve registro contable consolidado.

**Ventajas:** una sola verdad de inventario; saneamiento progresivo del dato sucio histórico.

**Razones del rechazo:**
- Requiere C-3 (conteo asistido), que el owner cortó del scope v1 (C-O2).
- Requiere acuerdo formal con dueño de SIM (¿IT? ¿Operaciones?) y posiblemente cambios contractuales para que un sistema externo escriba a SIM.
- Requiere caracterizar otros consumidores de SIM para evitar romper sus flujos.
- Sin C-3, la integración R/W arrastra el problema de dato sucio sin saneamiento — pierde su ventaja principal.

### (B) Reemplazo de la parte de servicentro de SIM — RECHAZADA

**Descripción:** el sistema se vuelve la nueva fuente de verdad para inventario de servicentros; SIM deja de usarse para ese flujo; se migra la captura del jefe al sistema nuevo.

**Ventajas:** dato limpio desde día uno; sin doble verdad.

**Razones del rechazo:**
- Requiere que el jefe opere el sistema, que el owner cortó del scope v1 (C-O5, C-O7).
- Afecta a otros consumidores de SIM no caracterizados — riesgo organizacional alto.
- Requiere migración de datos históricos y posible cambio organizacional grande.
- Cuelga del cronograma de v1, que el owner prefiere mantener manejable.

### (C) Coexistencia con lectura unidireccional — DECISIÓN ADOPTADA

**Descripción y justificación completas en §10.4 del PRD.**

**Objeción documentada de Winston** (Party Mode 2026-05-24): *"(C) coexistencia con doble captura es garantía de divergencia — dos sistemas, dos verdades, ningún ganador."* — válida en abstracto pero atenuada en este caso específico porque el jefe NO captura en el sistema nuevo (clarificación C-O7). La divergencia que sí existe es entre SIM contable y plan operativo del coordinador, monitoreada por R-11 y mitigada por SLA de frescura del snapshot (NFR-DAT-4, OQ-117).

**Trade-off firmado por el owner:** scope manejable v1 a cambio de no atacar la causa raíz (concesión central declarada en §11.16, R-1).

---

## A3. Inputs y contexto downstream para `architecture.md`

Material recopilado durante Discovery y change-signal que `architecture.md` debe consumir pero que no pertenece al PRD:

### A3.1 Patrones técnicos heredados de Tomaturno (referencia, no decisión)

- Auth0 con JWT para autenticación.
- SendGrid para notificaciones por correo (FR-094, FR-110).
- Firebase Hosting para Angular SPA.
- Cloud Run + Docker para Spring Boot.
- Cloud SQL Postgres.
- Cloud Storage para evidencias y exports.

`architecture.md` decide si adopta estos patrones tal cual o los ajusta.

### A3.2 Forecasting — librerías candidatas y trigger objetivo

Stack mandado por `project-context.md`: **Java puro** (Smile o Apache Commons Math) detrás de interfaz `ForecastingEngine`. **No Python, no Vertex AI, no BigQuery ML sin RFC formal.**

**Trigger objetivo para reevaluar Python como microservicio** (heredado de `project-context.md`): si el spike de Winston (R-8) demuestra que Java puro no cumple SLA del NFR-E-3 ("minutos no horas") en universo completo 226 × N SKUs, abrir RFC para evaluar Python como microservicio aislado detrás de la misma interfaz `ForecastingEngine`.

### A3.3 Decisión arquitectónica implícita en el PRD pendiente de validar en `architecture.md`

Winston (Party Mode 2026-05-24) señaló que el PRD escogió arquitectura sin admitirlo en varios lugares:
- **RxJS `debounceTime(150)`** en NFR-P-1 — asume frontend Angular reactivo.
- **Bitácora inmutable** en NFR-A-3 — asume event sourcing o append-only en persistencia.
- **Export bit-a-bit compatible** en FR-110 — asume capacidad de generar XLSX idéntico al de referencia.

`architecture.md` debe validar (o cuestionar) estas decisiones tácitas.

---

## A4. Decisiones pendientes que NO entraron al PRD pero quedan documentadas

Estas son consultas y decisiones que surgieron durante el change signal o sesiones previas y que `architecture.md` o trabajos downstream deben resolver:

- **N de SKUs por tienda:** orden de magnitud sin cuantificar. Bloqueante de NFR-E-3 (R-8). Escalado por Winston.
- **Comprador piloto por nombre:** quién firma `EXPECTED_OUTPUT.csv` de los golden datasets (NFR-T-2). Sin esto, NFR-T-2 es ficción. Escalado por Murat TEA.
- **Sesión formal con Finanzas** para acordar baseline MAPE/WAPE (R-7, OQ-107) — agendar en las primeras 4 semanas. Sin número, no hay vara de medida del motor.
- **Spike de forecasting Java de 2 semanas** sobre 1 tienda × 12 meses de histórico real (R-8) — escalado por Winston. Más crítico ahora porque sin conteo asistido, la calidad del forecast descansa enteramente en el motor + factor de merma esperada.
- **Backtesting precede al piloto, no lo acompaña** (R-9) — escalado por Murat + Dr. Quinn.

---

**Fin del addendum.**
