---
title: PRD — insumos-odemas
project: insumos-odemas
status: draft
created: 2026-05-22
updated: 2026-05-22
owner: Jonathan
language: es-MX
---

# PRD — insumos-odemas

## Resumen ejecutivo

**insumos-odemas** es un sistema de planeación quincenal de insumos de papel para los 226 servicentros (centros de copiado) de Office Depot México. Predice la demanda por tienda y SKU, sugiere la cantidad a pedir respetando el techo de presupuesto individual de cada tienda, asiste el conteo físico de bodega para sanear el dato de inventario, identifica oportunidades de transferencia (TFS) entre tiendas antes de comprar al proveedor, y detecta mermas hoy invisibles. Sustituye un proceso manual basado en Excel + correo + SIM que cuesta hasta **6 horas por quincena al jefe de cada servicentro** y **un día completo de trabajo al coordinador de guardia** que consolida ~113 solicitudes por ciclo. La salida del sistema es bit-a-bit compatible con `SolicitusDeInsumosTodos.xlsx`, el formato que el comprador corporativo recibe hoy.

---

## 1. Visión

Un sistema que **predice, sugiere, valida y traza** la solicitud quincenal de insumos para los 226 servicentros de OD México — eliminando el cálculo manual del jefe de servicentro, automatizando la consolidación distrital del coordinador, y haciendo visibles las pérdidas que hoy se contabilizan como consumo.

## 2. Contexto operativo

### Universo del producto

- **226 servicentros** distribuidos en formato **Express** (tiendas más chicas con menos espacio físico) y **Estándar** (formato completo). Ambos formatos manejan el mismo catálogo de servicios e insumos; el formato no afecta la lógica de forecast.
- **Tres coordinadores** (Carlos, Marco, Eduardo) operan bajo un **modelo híbrido**:
  - **Consolidación quincenal:** rotación de guardia semanal. El coordinador de guardia esa semana procesa todo lo que entró, sin importar zona.
  - **Excepciones mid-cycle:** asignación territorial fija. Cada coordinador atiende exclusivamente las tiendas de su distrito asignado (zona + formato), preservando conocimiento y relación con cada jefe.
- **Comprador corporativo** (rol único) recibe el concentrado consolidado por correo y materializa las órdenes de compra al proveedor. **Fuera del alcance del sistema**, pero consumidor downstream — el output del sistema debe ser compatible con su flujo actual.

### Cadencia operativa

- Cada tienda tiene un **ciclo quincenal** asignado en `Presupuesto-tiendas.csv`: "semana 1 y 3" o "semana 2 y 4" (numeración ISO 8601). Aproximadamente la mitad del universo de tiendas (~113) pide cada semana.
- La **consolidación normalmente arranca los miércoles**. Lunes-martes los jefes envían su solicitud por correo; miércoles-jueves el coordinador de guardia consolida; el `SolicitusDeInsumosTodos.xlsx` se entrega al comprador antes del viernes.
- Las **excepciones mid-cycle** (ventas extraordinarias, faltantes inesperados) ocurren con frecuencia indeterminada `[ASSUMPTION: 1-3 por semana en la red completa, a medir post-lanzamiento]` y siguen un canal informal (llamada / WhatsApp / correo) hacia el coordinador **territorial** de la tienda afectada.

### Presupuesto por tienda

- `Presupuesto-tiendas.csv` define un **techo MXN semanal individual por tienda**. Estos techos ya están calibrados por venta histórica — tiendas que venden más reciben presupuesto mayor.
- **No hay pool distrital ni reasignación de MXN entre tiendas.** El coordinador valida que cada solicitud individual ≤ techo de esa tienda; si excede, recorta a esa tienda.
- Las **transferencias físicas entre tiendas (TFS)** son un mecanismo paralelo que **no consume presupuesto de la tienda receptora** — son flujo de inventario, no de gasto.

### Herramientas y sistemas actuales

- **SIM** — sistema interno donde se registran movimientos de bodega (altas y bajas). La captura es manual y, bajo presión operativa, **frecuentemente imprecisa**. Provee visibilidad de tránsito (ALLOC, TFS) a los coordinadores. La interfaz entre el sistema nuevo y SIM es decisión arquitectónica abierta — ver `architecture.md`.
- **Excel + correo electrónico** — vehículo principal del flujo de solicitud. Cada jefe llena un Excel con su solicitud; el coordinador de guardia arma `SolicitusDeInsumosTodos.xlsx` consolidando ~113 solicitudes por ciclo. El archivo `docs/SolicitusDeInsumosTodos.xlsx` está en el repositorio como referencia del formato actual.
- **WhatsApp / llamadas / correo informal** — canal de excepciones mid-cycle, sin estructura, sin trazabilidad, sin priorización.

### Restricciones del entorno

El stack tecnológico obligatorio (Java + Spring Boot + Angular sobre GCP), las convenciones de datos (locale es-MX, fechas `dd/MM/yyyy`, encoding CSV variable, `BigDecimal` para montos, fail-loud sobre datos faltantes), y la disciplina de testing (golden datasets, backtesting, property-based testing con jqwik, contract testing) están definidas en `_producto/project-context.md`. No se repiten aquí.

## 3. El problema

El cálculo manual del pedido y la consolidación distrital son **costosos, lentos, imprecisos, opacos y no escalan a 226 tiendas**. Los puntos de dolor se distribuyen entre los tres actores y un dolor cross-corte de pérdidas invisibles.

### Dolor del Jefe de servicentro

| Código | Dolor |
|---|---|
| **D-jefe-1** | **Hasta 6 horas por quincena** dedicadas al cálculo manual, compitiendo con la atención al personal de servicentro y a clientes en mostrador. Varía por volumen de la tienda. |
| **D-jefe-2** | El **conteo físico de bodega** se hace bajo presión operativa y queda mal registrado en SIM. Los saldos sucios contaminan el siguiente ciclo de planeación. |
| **D-jefe-3** | El cálculo es **empírico, a ojo, en función de la temporada** — sin método formal ni fórmula reproducible. Conocimiento en la cabeza del jefe; si rota personal, su reemplazo arranca de cero. |
| **D-jefe-4** | Sin visibilidad de inventario en tránsito (ALLOC pendiente, TFS en camino) → riesgo de pedir de más por desconocer lo que ya viene. |
| **D-jefe-5** | Cuando hay excepciones (ventas extraordinarias), debe **interrumpir su día** para armar un WhatsApp / correo / llamada, y **esperar autorización mientras el cliente lo mira en mostrador**. Es fricción operativa, no ambigüedad de roles — la cadena de accountability ya es clara hoy (jefe avisa, coordinador decide, comprador autoriza). |

### Dolor del Coordinador de distrito

| Código | Dolor |
|---|---|
| **D-coord-1** | **Un día completo** por consolidación, repetible cada ~3 semanas en rotación. **~72 días-hombre/año** entre los tres coordinadores. |
| **D-coord-2** | Si una tienda no envía su Excel a tiempo, el coordinador debe **detectarlo manualmente**. Una tienda olvidada se queda sin pedido esa quincena → quiebre garantizado. |
| **D-coord-3** | Validar `cantidad × costo ≤ techo de presupuesto` para ~113 tiendas × N SKUs **a mano en Excel** es lento y propenso a error aritmético. |
| **D-coord-4** | Las **oportunidades de transferencia entre tiendas (TFS)** dependen de la memoria del coordinador. En 226 tiendas se le escapan casos. **Cada TFS perdida es una compra al proveedor que se pudo evitar** — dinero quemado, hoy no contabilizado. |
| **D-coord-5** | Sin trazabilidad: cuando el coordinador recorta una tienda de 10 a 4 piezas, **en ningún lado queda escrito por qué**. Sin evidencia para auditoría ni para retroalimentar al jefe. |
| **D-coord-6** | El canal de excepciones (llamadas, WhatsApp, correo) **no se prioriza ni se conserva**. Una alerta urgente puede perderse en la avalancha. |
| **D-coord-7** | Sin **dashboard agregado del distrito en vivo**: cuántas tiendas faltan responder, cuánto presupuesto se ha consumido, qué SKUs concentran riesgo, qué tiendas piden por encima de su techo. |

### Dolor del flujo de excepción mid-cycle

| Código | Dolor |
|---|---|
| **D-exc-1** | Canal disperso (WhatsApp + correo + llamada), sin bandeja única, sin garantía de quién recibe primero. |
| **D-exc-2** | No hay priorización entre múltiples alertas simultáneas. |
| **D-exc-3** | El coordinador decide TFS vs compra directa **sin ver en pantalla** qué tienda cercana tiene exceso del SKU faltante. |
| **D-exc-4** | No queda trazado quién decidió qué, cuándo, ni por qué. |
| **D-exc-5** | La compra extraordinaria al proveedor **no se refleja en el saldo de planeación** → el siguiente ciclo arranca con dato falso. |

### Dolor cross-corte — pérdidas invisibles

| Código | Dolor |
|---|---|
| **D-merma-1** | El sistema actual no detecta el patrón **"venta plana o decreciente + solicitudes de insumo crecientes"** que sugiere merma. Los coordinadores lo intuyen pero no lo sistematizan. Cada ciclo perdido es dinero quemado sin trazabilidad. |

## 4. Por qué no Excel

> *"Esto se podría hacer en Excel."*

Es una pregunta legítima — y la respuesta no es retórica. Excel **es** suficiente para algunos casos: una tienda, pocos SKUs, un humano dedicado, sin necesidad de auditoría. Pero el universo de este producto **no es ese caso**.

### Lo que Excel sí puede hacer

- Cálculo aritmético de `cantidad × costo ≤ techo` para una tienda individual.
- Forecast estacional simple si alguien escribe y mantiene las fórmulas.
- Plantilla compartida para llenar y enviar por correo.
- Tabla pivote para consolidar al final del ciclo.

### Lo que Excel rompe a escala de 226 tiendas

| Capacidad | Por qué Excel no | Por qué el sistema sí |
|---|---|---|
| **Multi-tienda concurrente con verdad única** | Un archivo, un dueño, no se sincroniza. Se vuelven "el Excel de Carlos" y "el Excel de Marco" — verdades divergentes. | Estado en servidor, ~113 jefes capturando en paralelo, coordinador ve la consolidación en vivo. |
| **Conteo asistido de bodega** | Excel no estructura el conteo; el jefe lo hace en papel o a ojo y transcribe — fuente del dato sucio en SIM. | UI dedicada al conteo por SKU/bin, validación de orden de magnitud, ~15 min en lugar de 6 horas. |
| **Validación de presupuesto a escala** | Calcular `cantidad × costo ≤ techo` para ~113 tiendas × N SKUs a mano es lento y propenso a error aritmético. | Validación automática por tienda, sugerencia de recorte cuando excede, indicador agregado del consumo distrital. |
| **Detección de mermas (anomalía consumo vs venta)** | Para detectar "venta plana + alta solicitud" en 226 tiendas habría que recrear el motor analítico en cada Excel. | Motor central cruza histórico de venta × consumo de insumo × cruce SKU, alerta al coordinador para registro humano asistido. |
| **Sugerencia de TFS entre tiendas** | Requiere mirar el universo entero (exceso vs déficit) simultáneo — imposible en hojas separadas. | Motor central detecta pares "tienda A con exceso + tienda B con déficit" y sugiere transferencia **antes** de compra al proveedor. Cada TFS sugerida es dinero ahorrado. |
| **Trazabilidad y auditoría** | Excel no registra quién cambió qué y por qué. Si el comprador audita, el rastro está perdido. | Bitácora automática: sugerencia original → override → razón estructurada → timestamp → actor. Exportable. |
| **Canal estructurado de excepciones** | WhatsApp + correo + llamada son inenarrables, sin priorización, sin estado. | Bandeja por coordinador territorial, alertas con SKU + cantidad faltante + urgencia + motivo, decisión registrada y trazada. |
| **Fail-loud sobre datos faltantes** | Excel silencia errores con celdas vacías. Si falta equivalencia SKU venta↔insumo, se infiere o se ignora. | Excepciones de negocio explícitas (`EquivalenciaNoDefinidaException`, `MinimoPackNoConfiguradoException`, etc.) abortan el ciclo de esa tienda hasta resolver — ver `project-context.md`. |
| **Reflejo automático en el siguiente ciclo** | Cada Excel es un punto en el tiempo; el siguiente arranca de cero. | El estado persiste; lo decidido la quincena pasada alimenta el motor de la próxima, incluyendo compras de excepción. |

### La cuenta dura del statu quo

- **Jefe de servicentro:** hasta 6 horas/quincena × 226 tiendas × ~26 ciclos/año ≈ **hasta 35,000 horas-hombre/año** en cálculo manual. Aun reduciendo a una fracción, el ahorro es masivo.
- **Coordinador:** 1 día por consolidación × ~52 consolidaciones/año (cada semana hay una) ≈ **~52 días-hombre/año** en consolidación, repartidos entre los tres en rotación.
- **TFS perdidas:** cada transferencia que el coordinador no recordó se traduce en compra extra al proveedor. Baseline cuantitativo `[ASSUMPTION: medir en pilotaje]`, pero es **gasto evitable que hoy no se mide**.
- **Mermas invisibles:** sin detección sistemática, hoy se contabilizan como consumo. Es dinero que sale sin trazabilidad ni rendición de cuentas.

Excel no es competidor del sistema. **Excel es el statu quo cuyo costo el sistema vuelve visible.**

## 5. Objetivos del producto

Cuatro objetivos, en orden de prioridad:

### O1 — Eliminar el cálculo manual del jefe de servicentro

El jefe deja de calcular cuánto pedir. El sistema lo hace por él contra el histórico real, el inventario asistido y el calendario estacional. El jefe **valida y aprueba** en su pantalla en lugar de **calcular y rendir cuentas** en Excel. Captura el conteo de bodega con UI dedicada, no a ojo en papel.

### O2 — Eliminar quiebres y sobre-inventario

Ningún servicentro debe quedarse sin papel en su ciclo normal, **y** ningún servicentro debe acumular papel ocioso. La métrica operativa más visible es **tasa de quiebres en mostrador → 0** acompañada de **rotación de inventario en rangos sanos por tienda**.

### O3 — Automatizar el concentrado distrital

El coordinador deja de armar `SolicitusDeInsumosTodos.xlsx` a mano. El sistema consolida automáticamente, valida techos individuales, sugiere recortes cuando exceden, sugiere TFS entre tiendas con exceso y déficit, y entrega al comprador un export bit-a-bit compatible con el formato actual. El día completo del coordinador se reduce a una sesión de revisión y aprobación.

### O4 — Hacer visibles las pérdidas hoy invisibles

Las mermas dejan de ser folclore distrital. El sistema las **señala** con evidencia cuantitativa (patrón consumo de insumo > venta esperada para esa tienda), las **registra** con captura del coordinador territorial, y las **acumula** para revisión trimestral con dirección.

## 6. Métricas de éxito (preliminar)

Las metas cuantitativas finales se acuerdan con Finanzas en pilotaje. Las propuestas iniciales son:

| Métrica | Baseline | Objetivo v1 | Cómo se mide |
|---|---|---|---|
| **Tiempo del jefe por ciclo** | hasta 6 h | **< 30 min** | Telemetría: tiempo entre login y submit del pedido. |
| **Tiempo del coordinador por consolidación** | 1 día completo | **< 2 h** | Telemetría: tiempo entre primera apertura de la bandeja y export del consolidado. |
| **Tasa de quiebre (cliente rechazado por falta de insumo)** | `[OPEN: baseline a recolectar en pilotaje]` | **Reducción ≥ 80%** | Contador en mostrador + reporte semanal de tiendas piloto. |
| **Sobre-inventario (cobertura en semanas de venta)** | `[OPEN: baseline]` | Por tienda, mantener en rango sano (a definir con Finanzas) | Reporte del sistema: cobertura por SKU/tienda. |
| **TFS sugeridas y aceptadas** | 0 (memoria del coordinador) | `[OPEN: meta v1]` | Contador en sistema. Métrica complementaria de ahorro al proveedor. |
| **MAPE / WAPE del motor de forecast** | n/a | `[OPEN: acordar con Finanzas en backtesting]` | `BacktestingSuite.java` sobre histórico real — ver disciplina de testing en `project-context.md`. |
| **Mermas detectadas y registradas** | 0 (intuición) | `[OPEN: meta v1]` | Contador en sistema; revisión trimestral con coordinadores. |
| **Tasa de override del coordinador sobre la sugerencia del sistema** | n/a | **< 20% por ciclo** (señal de confianza) | Contador en sistema. |

### Contra-métricas (señales de que el sistema está fallando aunque las métricas principales se vean bien)

- **Tasa de override > 50% por ciclo** en alguna tienda: el coordinador no confía en la sugerencia; el sistema está sobreajustando o pidiendo cosas absurdas.
- **Tiempo del jefe creciendo en lugar de bajando**: la UI de conteo está más fricción que valor; revisar.
- **Excepciones mid-cycle creciendo después del lanzamiento**: el forecast inicial subestima sistemáticamente; calibrar.
- **Pedidos urgentes al proveedor (fuera de ciclo) creciendo**: o el forecast falla o las TFS sugeridas no son útiles.
- **`SolicitusDeInsumosTodos.xlsx` exportado pero el comprador sigue pidiendo correcciones**: incompatibilidad downstream con el flujo del comprador, ajustar el formato.
- **Reducción del tiempo del coordinador pero crecimiento de quiebres**: la automatización está omitiendo casos que el coordinador detectaba manualmente.

---

## 7. Personas y User Journeys

### 7.1 Persona — Jefe de servicentro

- **Quién es:** empleado de una tienda específica, 25-35 años. Lidera al personal del servicentro, gestiona la solicitud de insumos, atiende problemas operativos del área. No multitask cross-departamento.
- **Cómo opera hoy:** ciclo quincenal asignado por `Presupuesto-tiendas.csv`. Hace conteo físico en bodega (a veces a la carrera bajo presión operativa), registra bajas en SIM (con riesgo de dato sucio), calcula a ojo en función de la temporada cuánto pedir, llena un Excel y lo manda por correo al coordinador. Tiempo total: hasta 6 horas/quincena.
- **Cómo va a operar con el sistema:** abre la app, hace conteo asistido por SKU en pantalla (~15 min), revisa la cantidad sugerida con su trazo explicable, hace override si tiene contexto local que el sistema no ve (con razón estructurada capturada), aprueba y manda. Ante quiebre inminente fuera de ciclo, dispara una alerta estructurada con un botón.
- **Qué teme:** no perder control — la cadena de accountability ya está clara hoy. Sí teme la **fricción operativa** de seguir levantando la mano (interrumpir el día, esperar autorización mientras el cliente lo mira en mostrador).
- **Qué lo hace feliz:** no preocuparse por si va a haber inventario suficiente. Eliminar la ansiedad operativa, no solo la tarea.

### 7.2 Persona — Coordinador de distrito

- **Quién es:** uno de tres (Carlos, Marco, Eduardo). Full-time en servicentro pero responsabilidades más allá del inventario.
- **Universo asignado:** las 226 tiendas se reparten territorialmente entre los tres por zona + formato. Cada tienda tiene un coordinador territorial asignado para excepciones; pero la consolidación quincenal opera por guardia rotativa global semanal.
- **Cómo opera hoy:** miércoles arranca la consolidación. Recibe ~113 correos con Excel adjuntos, valida techo de presupuesto a mano por tienda, recorta cuando excede, se acuerda de TFS oportunas (por memoria), arma `SolicitusDeInsumosTodos.xlsx` y lo manda al comprador. Tiempo: un día completo.
- **Cómo va a operar con el sistema:** entra a su bandeja de la semana, ve estado en vivo de las ~113 tiendas, recibe sugerencias de recorte y de TFS automáticamente, valida o ajusta con razón capturada, aprueba el consolidado y exporta. Para excepciones mid-cycle, atiende las alertas de **sus** tiendas (territorio asignado) desde una bandeja propia, decide TFS de emergencia vs escalamiento a comprador, registra la decisión.
- **Qué lo hace feliz:** **dejar de armar `SolicitusDeInsumosTodos.xlsx` a mano**. Recuperar el día completo que hoy se va en consolidación.

### 7.3 Persona — Comprador corporativo (stakeholder pasivo)

- **Rol único** que recibe el consolidado del coordinador y materializa órdenes de compra al proveedor. **Fuera del alcance del sistema** — no opera la UI. **Sí es consumidor downstream**: el export del sistema debe ser compatible con su flujo actual basado en `SolicitusDeInsumosTodos.xlsx` recibido por correo.

### 7.4 User Journey AS-IS — Cálculo quincenal del jefe (UJ-1)

1. Llega el ciclo quincenal asignado (semana 1-y-3 o 2-y-4 ISO).
2. Jefe deja la atención al mostrador, va a la bodega del servicentro.
3. Hace conteo físico de SKUs de papel. Bajo presión, puede saltar SKUs o estimar a ojo.
4. Vuelve a la computadora, abre SIM, captura bajas observadas. Si el conteo fue impreciso, el dato queda sucio en SIM.
5. Abre el Excel corporativo de solicitud.
6. Estima a ojo en función de la temporada cuánto papel necesita para los próximos ~14 días por SKU.
7. Llena el Excel y lo envía por correo al coordinador.
8. (Aguas abajo) el coordinador consolida; el comprador genera la OC.

**Tiempo:** hasta 6 horas. **Dolores:** D-jefe-1..5.

### 7.5 User Journey AS-IS — Consolidación del coordinador (UJ-2)

1. (Miércoles, semana de guardia rotativa) coordinador entra al correo, encuentra ~113 Excel adjuntos.
2. Recolecta solicitudes; controla manualmente quién no respondió.
3. Consulta SIM para saldos, ALLOC pendiente y TFS en tránsito.
4. Valida cada solicitud contra su techo individual; recorta tiendas que exceden.
5. Por memoria, identifica oportunidades de TFS entre tiendas con exceso/déficit; **detecta** y **solicita** la TFS dentro de la jornada, pero la **ejecución** sigue por proceso paralelo (fuera del alcance v1).
6. Compila `SolicitusDeInsumosTodos.xlsx`.
7. Envía por correo al comprador.

**Tiempo:** un día completo. **Dolores:** D-coord-1..7.

### 7.6 User Journey AS-IS — Excepción mid-cycle (UJ-3)

1. Jefe detecta quiebre inminente por venta extraordinaria o consumo más rápido de lo previsto.
2. Contacta a **su coordinador territorial** (modelo de routing territorial, no rotativo) por WhatsApp / correo / llamada — sin canal único.
3. Coordinador valida el caso, decide caso por caso (no es lineal): TFS de emergencia, escalamiento al comprador para compra directa al proveedor, o rechazo si no procede.
4. Si escala al comprador, este puede autorizar o rechazar.
5. Compra extraordinaria al proveedor (si autorizada) ocurre fuera del ciclo normal.
6. **Ningún paso queda trazado en sistema de planeación.** El siguiente ciclo arranca con saldo desincronizado.

**Dolores:** D-exc-1..5.

## 8. Capacidades y Requisitos Funcionales

Las capacidades del sistema se agrupan en **doce áreas**. Cada FR tiene identificador estable (FR-NNN) y se referencia desde el resto del PRD. Cuando una capacidad responde directamente a un dolor de §3, se nombra.

### 8.1 Datos maestros (C-1)

Base operativa que el resto de capacidades consume.

- **FR-001** — Mantener catálogo de **tiendas** con: ID (String), nombre, distrito, **coordinador territorial asignado**, **formato (Express / Estándar)**, **ciclo quincenal asignado** (semana 1-y-3 o 2-y-4), **techo presupuestal semanal MXN**.
- **FR-002** — Mantener catálogo de **SKUs de insumo** (lo comprable al proveedor): código, descripción, presentación, contenido (piezas por presentación), costo MXN. Preservar literal `"Matarial y tamaño"` por contrato con el equipo de datos.
- **FR-003** — Mantener **tabla de equivalencia** SKU venta ↔ SKU insumo. Toda venta procesada debe encontrar su equivalencia; sin equivalencia → `EquivalenciaNoDefinidaException` y aborto del ciclo de esa tienda (fail-loud).
- **FR-004** — Mantener **calendario de rotación de guardia** entre los tres coordinadores (semana ISO → coordinador de guardia esa semana). Editable.
- **FR-005** — Mantener **datos de coordinadores**: identidad, distrito territorial asignado (lista de tiendas), credenciales, rol.

### 8.2 Ingesta y normalización de datos (C-2)

Carga de archivos canónicos en `docs/` al modelo del sistema. Las convenciones (encoding, parsing dual de fechas, separadores, tipos exactos, fail-loud) están en `project-context.md` y no se repiten.

- **FR-010** — Ingestar archivos canónicos del inventario en `docs/` por **descubrimiento por patrón de nombre** (`ALLOC_YYYY_MM_DD.csv`, `TFS_YYYY_MM_DD.csv`, `Entrega-directa-tienda*.csv`). Nuevos archivos sin reconfiguración del sistema.
- **FR-011** — Detectar **encoding** por archivo en orden: UTF-8 BOM → cp1252. Registrar encoding detectado en MDC.
- **FR-012** — Validar invariantes de ingesta documentadas en `project-context.md` (ej. `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE` en ALLOC, `TRAN_CODE ↔ DECODE` en Entrega-directa). Comportamiento fail-loud cuando aplica.
- **FR-013** — Parsear `Cantidad por semana` de `Presupuesto-tiendas.csv` (formato `"$X,XXX"`) y `Semana que debe solicitar` (lenguaje natural → `List<Integer>` ISO). Patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`.

### 8.3 Conteo asistido de bodega (C-3) — Dolor: D-jefe-2

Reemplaza la captura manual en SIM con UI dedicada al conteo físico.

- **FR-020** — Permitir al jefe **iniciar sesión de conteo** para su tienda en su ciclo activo.
- **FR-021** — Mostrar por SKU el **saldo esperado** (reconstrucción `Σ entradas − Σ consumos esperados`).
- **FR-022** — Capturar **conteo físico real**. Validar orden de magnitud contra esperado; resaltar divergencias significativas.
- **FR-023** — Cuando la divergencia supera umbral configurable, **señalar candidato de merma** automáticamente para revisión del coordinador territorial (alimenta C-9).
- **FR-024** — Persistir conteo con timestamp, jefe ejecutor, observaciones libres. Inmutable una vez cerrado.
- **FR-025** — Reflejar las **bajas reconciliadas** del conteo en el saldo del sistema. Decisión arquitectónica abierta de cómo coexistir / sincronizar con SIM — ver `architecture.md`.

### 8.4 Forecast de demanda quincenal (C-4) — Dolor: D-jefe-3

Motor de predicción por tienda × SKU.

- **FR-030** — Calcular **demanda esperada de venta** para el próximo ciclo por tienda × SKU venta, usando modelo estacional configurable (familia de modelos en Java puro definida en `project-context.md`).
- **FR-031** — Convertir demanda de venta a **demanda de insumo** aplicando la tabla de equivalencia.
- **FR-032** — **Cuantizar** la demanda de insumo al múltiplo del empaque mínimo (`Contenido` en `Skus_insumos`). Nunca redondear hacia abajo si hay demanda.
- **FR-033** — Aplicar **componente estacional anual** (regreso a clases, fin de mes, cierres fiscales, exámenes) basado en histórico ≥ 12 meses.
- **FR-034** — Generar **trazo explicable** del cálculo por SKU: histórico utilizado, ventana, factor estacional, ajuste por inventario en tránsito. Visible en hover en UI.

### 8.5 Sugerencia de pedido (C-5) — Dolor: D-jefe-1, D-jefe-4

Calcula la cantidad final a pedir considerando inventario asistido y tránsito.

- **FR-040** — Para cada tienda × ciclo, calcular **cantidad sugerida por SKU** = `demanda_quincenal_insumo + buffer_seguridad − inventario_bodega − ALLOC_pendiente − TFS_entrante`, cuantizada.
- **FR-041** — Mostrar al jefe la sugerencia con su trazo explicable.
- **FR-042** — Permitir **override del jefe** con campo obligatorio de **razón estructurada** (lista predefinida + texto libre opcional). Override registrado en bitácora.
- **FR-043** — Para SKUs sin equivalencia definida, **no incluir** en sugerencia y emitir `EquivalenciaNoDefinidaException` que aborta el ciclo de esa tienda hasta resolverse.

### 8.6 Validación de presupuesto y sugerencia de recorte (C-6) — Dolor: D-coord-3

Garantiza que la solicitud por tienda no exceda su techo individual.

- **FR-050** — Calcular **costo total por tienda** = `Σ (cantidad_sugerida × costo_SKU)` en MXN.
- **FR-051** — Si `costo_total > techo_semanal_tienda`, **sugerir recorte automático** priorizando reducir los SKUs con menor riesgo de quiebre.
- **FR-052** — Mostrar al coordinador, en la consolidación, las tiendas que requieren recorte con la sugerencia aplicada y permitir **aceptar / modificar / rechazar** con razón.
- **FR-053** — Persistir cada decisión de recorte con timestamp, coordinador ejecutor, razón.

### 8.7 Concentrado distrital — Bandeja de la semana (C-7) — Dolor: D-coord-1, D-coord-2, D-coord-7

UI del coordinador de guardia para procesar las solicitudes que entran cada miércoles.

- **FR-060** — Mostrar al coordinador de guardia la **bandeja de tiendas con ciclo activo esa semana** (~113 tiendas).
- **FR-061** — Por cada tienda, mostrar estado: pendiente de captura del jefe / capturada / con recorte sugerido / con TFS sugerida / aprobada / exportada.
- **FR-062** — **Alertar de tiendas que no han capturado** dentro del plazo configurable antes del cierre del ciclo. Recordatorio configurable al jefe.
- **FR-063** — Mostrar **agregados del ciclo**: presupuesto consumido total, número de tiendas pendientes, SKUs con riesgo concentrado, costo estimado del consolidado.
- **FR-064** — Permitir al coordinador **modificar cantidades** en cualquier tienda con razón estructurada; modificaciones en bitácora.
- **FR-065** — Permitir **aprobar el consolidado** y dispararlo al export downstream (C-12).

### 8.8 Sugerencia de TFS entre tiendas (C-8) — Dolor: D-coord-4

Identifica oportunidades de transferencia antes de compra al proveedor.

- **FR-070** — Detectar **pares (tienda exceso, tienda déficit)** del mismo SKU dentro del ciclo activo.
- **FR-071** — Sugerir la **cantidad transferible** considerando exceso real en origen, déficit en destino, restricciones geográficas básicas. Costo logístico estimado `[OPEN: a definir con operaciones]`.
- **FR-072** — Mostrar la sugerencia al coordinador como candidato; permitir **aceptar / modificar / descartar** con razón estructurada.
- **FR-073** — Cuando se acepta, **registrar la decisión** y reflejarla en los saldos del modelo de planeación. **La ejecución física del TFS** (orden de movimiento, logística) **es proceso separado fuera del alcance v1** — el sistema solo señala y rastrea.

### 8.9 Detección de mermas asistida (C-9) — Dolor: D-merma-1

Sistema híbrido humano-máquina para visibilizar pérdidas hoy invisibles.

- **FR-080** — Identificar **patrones sospechosos** por tienda × SKU: consumo de insumo > venta esperada, divergencia conteo asistido vs reconstrucción, alta variabilidad inter-ciclo.
- **FR-081** — Generar **candidatos de merma** clasificados por severidad. Surfacing en bandeja del coordinador territorial asignado.
- **FR-082** — Permitir al coordinador **capturar / modificar / descartar** cada candidato. Captura incluye: SKU, cantidad estimada, motivo (lista + texto libre), evidencia (foto opcional, nota).
- **FR-083** — Acumular registro de mermas confirmadas para revisión trimestral con dirección. Exportable.

### 8.10 Canal estructurado de alertas mid-cycle (C-10) — Dolor: D-jefe-5, D-exc-1..5

Reemplaza WhatsApp + correo + llamada con flujo trazado en sistema.

- **FR-090** — Permitir al jefe **disparar alerta de quiebre inminente** con: SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado.
- **FR-091** — Rutear automáticamente la alerta a la **bandeja del coordinador territorial asignado** a esa tienda (modelo territorial, no rotativo).
- **FR-092** — Mostrar al coordinador en su bandeja **contexto agregado**: saldo actual de la tienda, ALLOC pendiente, TFS en tránsito, sugerencias inmediatas (tiendas cercanas con exceso del SKU para TFS de emergencia).
- **FR-093** — Permitir al coordinador **decidir y registrar** la respuesta: TFS de emergencia (dispara flujo C-8 ad-hoc), escalamiento a comprador para compra directa, rechazo (con razón), espera al ciclo normal.
- **FR-094** — Cuando se escala a comprador, **notificarlo** por canal acordado (correo, integración) con contexto + decisión del coordinador.
- **FR-095** — Reflejar en el saldo de planeación cualquier **compra extraordinaria** autorizada, para que el siguiente ciclo parta de la realidad.

### 8.11 Trazabilidad y auditoría (C-11) — Dolor: D-coord-5, D-exc-4

Bitácora universal de cambios y decisiones.

- **FR-100** — Para cada cantidad sugerida modificada (override, recorte, TFS aceptada/rechazada, alerta atendida), **registrar entrada en bitácora** con: timestamp, actor, valor original, valor nuevo, razón estructurada.
- **FR-101** — Mostrar en UI (hover sobre fila modificada) la **cadena completa**: `original modelo X → Jefe Carlos 07:48: Y → Coord. Marco 09:12: aprobado`.
- **FR-102** — Panel lateral **"Cambios de hoy"** exportable a CSV.
- **FR-103** — MDC en logs con `tiendaId`, `skuId`, `cicloId`, `usuarioId` (heredado de `project-context.md`).
- **FR-104** — Bitácora **inmutable**. Para corregir un error se crea nueva entrada que referencia la anterior.

### 8.12 Export downstream al comprador (C-12)

Genera el archivo de handoff al flujo del comprador.

- **FR-110** — Generar archivo consolidado equivalente a `SolicitusDeInsumosTodos.xlsx` aprobado por el coordinador. **Formato compatible** con el archivo de referencia en `docs/SolicitusDeInsumosTodos.xlsx` `[ASSUMPTION: estructura por confirmar al inspeccionar el archivo en Contexto Técnico — §10]`.
- **FR-111** — Permitir **exportar bajo demanda** y **enviar por correo** al comprador con CC configurable.
- **FR-112** — Registrar cada export en bitácora: timestamp, coordinador ejecutor, hash del archivo, destinatarios.

---

## 9. Requisitos No-Funcionales transversales

Las convenciones de calidad, testing, stack y datos están en `_producto/project-context.md` y se asumen como base. Aquí solo se nombran los NFR específicos del producto que afectan el diseño y la operación.

### 9.1 Escala y concurrencia (NFR-Escala)

- **NFR-E-1** — Soportar **226 tiendas activas** concurrentes en el modelo de datos.
- **NFR-E-2** — Soportar **~113 jefes de servicentro capturando en paralelo** en las semanas pico (ciclo 1-y-3 o ciclo 2-y-4).
- **NFR-E-3** — El motor de forecast batch debe **procesar el universo completo** (226 tiendas × N SKUs) en una ventana razonable que no bloquee la operación del miércoles `[OPEN: cuantificar en architecture.md tras prototipado — orden de magnitud objetivo: minutos, no horas]`.
- **NFR-E-4** — El coordinador de guardia debe poder abrir su bandeja con **~113 tiendas** y verla cargada en menos de 3 segundos.

### 9.2 Performance percibida (NFR-Perf)

- **NFR-P-1** — Edición numérica en grilla del jefe / coordinador dispara **recálculo optimista en cliente** (RxJS `debounceTime(150)`) con confirmación servidor; recalcular fila, acumulado por proveedor y restante de presupuesto en **<100 ms** (heredado de `project-context.md`).
- **NFR-P-2** — Autosave silencioso con indicador tipo Google Docs ("Guardado hace 3 seg"). El usuario nunca pierde trabajo.

### 9.3 Disponibilidad (NFR-Disp)

- **NFR-D-1** — Disponibilidad **≥ 99% durante ventanas operativas** (lunes a viernes 06:00-22:00 hora CDMX). Ventanas fuera de operación tolerables para mantenimiento.
- **NFR-D-2** — En caso de caída, el conteo y la captura del jefe **deben poder reanudarse** sin pérdida del trabajo previo (estado persiste server-side).

### 9.4 Auditabilidad y trazabilidad (NFR-Audit)

- **NFR-A-1** — Toda decisión de negocio (override, recorte, TFS aceptada, alerta atendida, export) queda **trazada** con timestamp, actor identificado, valor original, valor nuevo, razón estructurada (C-11).
- **NFR-A-2** — Logs server-side con **MDC poblado** con `tiendaId`, `skuId`, `cicloId`, `usuarioId` para toda operación de negocio (heredado de `project-context.md`).
- **NFR-A-3** — Bitácora **inmutable**. Correcciones se hacen como nuevas entradas que referencian la anterior, nunca modificando la original.

### 9.5 Seguridad y control de acceso (NFR-Sec)

- **NFR-S-1** — Autenticación de usuarios con **identidad individual** (no compartida). Patrón sugerido (Auth0 con JWT) tomado de la arquitectura de referencia Tomaturno; decisión final en `architecture.md`.
- **NFR-S-2** — Roles operativos: **Jefe de servicentro** (acceso solo a su tienda), **Coordinador** (acceso a sus tiendas territoriales para excepciones + bandeja de guardia rotativa cuando aplica), **Administrador** (configura datos maestros, calendario de guardia, umbrales de detección de mermas).
- **NFR-S-3** — **MFA obligatorio** para roles administrativos.
- **NFR-S-4** — Datos sensibles (presupuestos, costos) **no se exponen** a roles que no los necesitan operativamente.

### 9.6 Calidad del dato como pilar (NFR-Datos)

Este NFR es estructural — el sistema solo es tan bueno como el conteo.

- **NFR-DAT-1** — **Fail-loud** ante datos faltantes o ambiguos: equivalencia SKU no definida, presupuesto no configurado, encoding ambiguo, periodicidad presupuestal no reconocida → excepciones checked específicas que abortan el ciclo de esa tienda (heredado de `project-context.md`).
- **NFR-DAT-2** — **Calidad de UX del conteo** es prioridad de primer nivel — la UI debe asistir el conteo de forma que se complete bien en ~15 minutos en lugar de hacerse a la carrera en 5.
- **NFR-DAT-3** — Encoding de archivos detectado y **registrado en logs** por archivo procesado.

### 9.7 Locale y representación (NFR-Locale)

Heredado de `project-context.md`. Se enuncian los que afectan UX directamente.

- **NFR-L-1** — `LOCALE_ID = 'es-MX'`, `registerLocaleData(localeEsMX)` en Angular.
- **NFR-L-2** — **Fechas**: mostrar `dd/MM/yyyy`; transportar JSON como ISO 8601.
- **NFR-L-3** — **Semanas**: ISO 8601 (lunes a domingo). Compradores y coordinadores piensan en semanas, no en fechas sueltas (`"Sem 12 · 16-22 mar 2026"`).
- **NFR-L-4** — **Moneda**: MXN con separador de miles y dos decimales (`$1,250.00`). Totales grandes abreviados con tooltip (`$1.2M` / hover `$1,234,567.89`).
- **NFR-L-5** — **Cantidades discretas**: enteros sin decimales, separador de miles, unidad pegada al número (`1,250 cajas`). Mostrar siempre equivalencia (`18 piezas = 3 cajas`).
- **NFR-L-6** — **Plazos**: días hábiles, no naturales (`5 días hábiles`).

### 9.8 Accesibilidad (NFR-A11y)

Heredado de `project-context.md` — base WCAG 2.1 AA mínimo.

- **NFR-A11Y-1** — Contraste 4.5:1 en texto, 3:1 en componentes interactivos.
- **NFR-A11Y-2** — Toda celda editable con `aria-label` contextualizado.
- **NFR-A11Y-3** — Navegación por teclado completa en grillas (Tab/Shift+Tab/Enter/Esc). `:focus-visible` con outline 2px; `outline:none` prohibido.

### 9.9 UX no negociable (NFR-UX)

Heredado de `project-context.md` y enunciado aquí porque condiciona el diseño visual del producto.

- **NFR-UX-1** — **Trazabilidad visible**: toda cantidad sugerida con affordance de explicación (ícono `info` con tooltip); valor sugerido vs override diferenciados tipográficamente; auditoría visible al hacer hover.
- **NFR-UX-2** — **Acciones reversibles > acciones confirmadas**: toast con "Deshacer" durante 8s, no `confirm()` modal.
- **NFR-UX-3** — **Errores con quién, qué, por qué y qué hacer ahora** en tres niveles: inline (campo), negocio (fila/sección), sistema (toast con código de incidente copiable).
- **NFR-UX-4** — Tone es-MX claro y no técnico (`"La cantidad debe ser múltiplo de 6 (empaque mínimo)"`, no `"Invalid input: not divisible by pack_size"`).

### 9.10 Mantenibilidad y testabilidad (NFR-Test)

Heredado de `project-context.md`. Se nombran porque afectan la arquitectura y el plan de entrega.

- **NFR-T-1** — Pipeline composable: cada paso del flujo (parsing → validación → catálogo → joins → forecast → cuantización → presupuesto-window → output) testeable en aislamiento.
- **NFR-T-2** — Golden datasets versionados en `src/test/resources/fixtures/golden/vN/` con `EXPECTED_OUTPUT.csv` firmado por el comprador piloto + `DECISION_LOG.md`. Inmutable una vez shipped.
- **NFR-T-3** — Property-based testing con jqwik sobre cuantización y presupuesto. Invariantes obligatorias enunciadas en `project-context.md`.
- **NFR-T-4** — Contract testing Pact / Spring Cloud Contract entre Spring Boot y Angular. Contrato YAML antes de implementar.
- **NFR-T-5** — Backtesting suite sobre histórico real para acordar MAPE/WAPE con Finanzas.
- **NFR-T-6** — Mutation testing con PIT sobre paquete `forecasting.*`. Threshold ≥ 80% mutantes eliminados antes de mergear a `main`.

### 9.11 Internacionalización (NFR-i18n)

- **NFR-I-1** — V1 **solo es-MX**. No se diseña con i18n diferida pero tampoco se cierra la puerta — strings de UI vivirán en archivo de recursos para evitar reescritura si en el futuro se internacionaliza.
- **NFR-I-2** — Identificadores de código en **inglés**; comentarios y documentación de negocio en **español** (alineado con `project-context.md`).

### 9.12 Observabilidad (NFR-Obs)

- **NFR-O-1** — Métricas de operación expuestas: tiempo de captura por jefe, tiempo de consolidación por coordinador, número de alertas mid-cycle por semana, tasa de override por tienda, latencia del motor de forecast, errores de ingesta por archivo.
- **NFR-O-2** — Dashboard de operación visible para administrador del sistema con las métricas anteriores.
- **NFR-O-3** — Alertas automáticas a admin cuando: ingesta de un archivo falla, tasa de excepciones de negocio crece anormalmente, motor de forecast tarda fuera de SLA.

---

## 10. Contexto técnico y dependencias

Esta sección **no decide arquitectura** — esa decisión es de `_producto/planning-artifacts/architecture.md`. Aquí se enumeran las dependencias técnicas, restricciones y decisiones abiertas que el PRD necesita exponer para que arquitectura las resuelva.

### 10.1 Stack obligatorio

Heredado sin negociación de `_producto/project-context.md`:

- Backend: **Java + Spring Boot** contenerizado con Docker.
- Base de datos: **PostgreSQL** sobre **Cloud SQL** en GCP.
- Frontend: **Angular**, hosting en **Firebase Hosting**.
- Plataforma cloud: **GCP exclusivamente** (no AWS, no Azure).
- Forecasting: **Java puro** (Smile o Apache Commons Math) detrás de interfaz `ForecastingEngine`. No Python, no Vertex AI, no BigQuery ML sin RFC.
- Entorno local del equipo: Windows 11 + PowerShell 5.1.

### 10.2 Archivos canónicos de datos

Fuente de verdad documentada en `_producto/project-context.md`. Resumen:

| Archivo | Rol |
|---|---|
| `Cruce-de-skus-venta-insumo.xlsx` | Equivalencia venta ↔ insumo (autoridad). |
| `historico-de-ventas-2024-2025-2026.csv` | Histórico de ventas por tienda/mes/SKU. |
| `Skus_insumos.csv` | Catálogo de SKUs de insumo. |
| `Presupuesto-tiendas.csv` | Ventana presupuestal y periodicidad por tienda. |
| `ALLOC_*.csv` | Órdenes de surtido WH → tienda. |
| `TFS_*.csv` | Transferencias entre tiendas. |
| `Entrega-directa-tienda.csv` | Compras y transferencias directas en tienda. |
| **`SolicitusDeInsumosTodos.xlsx`** ⚠️ | **Hallazgo del Discovery del PRD:** formato consolidado actual que el coordinador envía al comprador. NO está en el inventario canónico de `project-context.md` — se debe **agregar** allí y revisar su estructura para diseñar el export del sistema (FR-110). |

Las convenciones de parseo, encoding, tipos exactos y dialectos están en `project-context.md` (sección "Insumos de datos") y se asumen.

### 10.3 Atributos del modelo de Tienda pendientes de confirmar en datos

El modelo de Tienda en el sistema necesita los siguientes atributos. La fuente exacta para cada uno debe verificarse al inspeccionar los archivos:

- **Identificador (String).** Confirmado en `historico-de-ventas` como `location_key` (entero → tratar como String).
- **Distrito asignado.** Está en `Presupuesto-tiendas.csv` como columna `Distrito` (confirmado en `project-context.md`).
- **Formato (Express / Estándar).** `[OPEN: verificar si Presupuesto-tiendas.csv ya lo distingue, o si requiere data master nueva]`. Captura inicial: probable que sí esté en algún catálogo OD que no está en `docs/` todavía.
- **Coordinador territorial asignado.** **No existe en `docs/` hoy.** Es **data master nueva** que el sistema debe capturar al onboarding (mapeo Tienda → uno de {Carlos, Marco, Eduardo}).
- **Ciclo quincenal asignado.** Confirmado en `Presupuesto-tiendas.csv` columna `Semana que debe solicitar` → `List<Integer>` ISO.
- **Techo presupuestal semanal MXN.** Confirmado en `Presupuesto-tiendas.csv` columna `Cantidad por semana` (string `"$X,XXX"`).

### 10.4 SIM — decisión arquitectónica abierta

**SIM** es el sistema interno donde HOY los jefes de servicentro capturan altas y bajas de inventario en bodega de forma manual. Los coordinadores tienen acceso de lectura para ver ALLOC y TFS.

Hay tres modelos de interacción posibles con SIM en v1 del sistema nuevo:

- **(A) Integración (read + write):** el sistema lee saldos de SIM como fuente de verdad inicial; cuando el conteo asistido del jefe genera bajas reconciliadas, las escribe a SIM. **Ventaja:** una sola verdad. **Riesgo:** la calidad del dato en SIM hoy es sucia; integrar sin sanear arrastra el problema. Requiere acuerdo con dueño de SIM (¿IT? ¿Operaciones?).
- **(B) Reemplazo de la parte de servicentro de SIM:** el sistema se vuelve la nueva fuente de verdad para inventario de servicentros; SIM deja de ser usado para ese flujo. **Ventaja:** dato limpio desde día uno. **Riesgo:** alto — afecta a otros consumidores de SIM (no caracterizados), requiere migración de datos y posiblemente cambio organizacional.
- **(C) Coexistencia:** el sistema nuevo opera con su propia base de datos; SIM sigue capturándose en paralelo (doble captura). **Ventaja:** sin dependencia técnica con SIM. **Riesgo:** duplica el trabajo manual del jefe — **antitético al objetivo del proyecto**.

**Decisión a tomar en `architecture.md` con dueño de SIM e IT.** La recomendación preliminar del PRD es **(A) integración con saneamiento progresivo**: el sistema nuevo se vuelve la UI primaria del jefe; las bajas reconciliadas del conteo asistido se escriben a SIM; SIM sigue siendo registro contable hasta que se valide migración.

**Información a recolectar antes de decidir:**

- ¿Quién es el dueño operativo y técnico de SIM?
- ¿Qué otros consumidores tiene SIM además de servicentro?
- ¿Tiene SIM API o solo UI? Si API, contrato disponible.
- ¿Hay restricciones legales / contables para que un sistema nuevo escriba a SIM?

### 10.5 Datos maestros adicionales requeridos por el sistema

Más allá de lo que viene en `docs/`, el sistema necesita gobernar:

- **Mapeo Tienda → Coordinador territorial** (para C-10 routing de excepciones).
- **Calendario de rotación de guardia semanal** entre los tres coordinadores (para C-7 visibilidad de bandeja).
- **Umbrales configurables** para C-9 detección de mermas (qué desviación considerar "sospechosa").
- **Plazos del ciclo de pedido** (cuándo se cierra la captura del jefe, cuándo se cierra la consolidación del coordinador) — semana ISO + día (miércoles por defecto).
- **Lista de razones estructuradas** para overrides, recortes, alertas, decisiones (configurable por admin).

### 10.6 Integraciones externas

- **SIM** — ver §10.4. Lectura/escritura por definir.
- **Correo (SMTP) o servicio de email** — para notificar al comprador con el export (FR-094, FR-110). Patrón sugerido (SendGrid) tomado de arquitectura Tomaturno; decisión final en `architecture.md`.
- **Auth0 (probable)** — autenticación con MFA para roles administrativos (NFR-S-1, NFR-S-3). Patrón Tomaturno; decisión final en arquitectura.
- **Cloud Storage** — para evidencias adjuntas en captura de mermas (FR-082) y para hash/almacenamiento de exports históricos (FR-112).

### 10.7 Restricciones operacionales heredadas

- **Datos de entrada en esta fase:** archivos planos (CSV/XLSX) en `docs/`, exportados desde sistemas internos de OD. **No se asume acceso directo a bases productivas.**
- **Encoding de CSVs:** validar caso por caso (UTF-8 BOM o cp1252 indistintamente).
- **Locale:** `es-MX`; verificación de separador decimal por archivo antes de parsear.
- **No introducir Python ni servicios gestionados de ML/AI** sin RFC formal (`project-context.md` §"Motor de forecasting").

---

## 11. Out of scope explícito

Lo siguiente **NO está en el alcance de v1**. Cada exclusión se justifica para evitar arrastres futuros.

### 11.1 Órdenes de compra al proveedor

El comprador corporativo recibe el consolidado y materializa las OC al proveedor con su flujo actual. **El sistema no genera, transmite ni rastrea OC**. La interfaz con el comprador es el export `SolicitusDeInsumosTodos.xlsx`-compatible (FR-110) por correo.

### 11.2 Ejecución física de transferencias (TFS)

El sistema **sugiere y registra la decisión** de TFS (C-8, FR-073) pero **no ejecuta** el movimiento físico: orden de transferencia, logística del traslado, asentamiento contable. Ese proceso paralelo sigue operando donde vive hoy (probablemente SIM u otro módulo).

### 11.3 Insumos NO-papel

El sistema v1 cubre **únicamente insumos de papel** del servicentro. Los otros insumos (toners, cartuchos, suministros varios) están reconocidos como necesidad futura pero **fuera del primer alcance**.

### 11.4 Módulos de SIM no relacionados con bodega de servicentro

SIM tiene otros usos que no son materia de este sistema. Solo se interactúa con SIM en la zona de altas/bajas de inventario de bodega de servicentro.

### 11.5 Modelado de capacidad física de bodega

Tras Discovery se confirmó que **los presupuestos y la demanda mantienen los pedidos siempre debajo del techo físico** de la bodega — no es problema real. **No se modela** capacidad física como atributo de Tienda. Casos raros de saturación se resuelven con override manual del jefe (FR-042).

### 11.6 Aprendizaje automático del override del coordinador

El sistema **registra** los overrides con razón estructurada (FR-100), pero **no aprende automáticamente** a ajustarse a ellos en v1. La iteración del motor ante patrones de override sistemáticos es trabajo de calibración manual del equipo. **Posible v2.**

### 11.7 Aplicación móvil nativa

V1 es **web responsive** que funciona en computadora del jefe / coordinador. **No hay app nativa** iOS/Android. **Posible v2** si la operación de campo lo demanda.

### 11.8 Reportería BI / ejecutiva fuera del dashboard operativo

El dashboard operativo del coordinador (FR-063) y del administrador (NFR-O-2) son lo que entrega v1. **Reportería ejecutiva, exportes para Power BI / Looker, integraciones con DWH** son trabajo posterior.

### 11.9 Internacionalización fuera de es-MX

V1 **solo es-MX** (NFR-I-1). No se diseña con i18n diferida pero se preserva separación de strings de UI para no cerrar la puerta.

### 11.10 Capacitación, change management y comunicación oficial

El sistema **provee la herramienta**; la capacitación de los jefes y coordinadores y el anuncio oficial de que la tarea pasa a coordinadores son **responsabilidad de la célula de UX / Eficiencia operativa** y del área operativa. Crítico para la adopción, pero fuera del producto. **Identificado como riesgo (R-2).**

## 12. Riesgos

Riesgos materiales identificados en Discovery. Cada uno con dueño tentativo y mitigación propuesta. Final en `architecture.md` y plan de pilotaje.

### R-1 — Calidad sucia del dato en SIM contamina el sistema nuevo

- **Descripción:** SIM hoy tiene saldos imprecisos porque el conteo de los jefes se hace bajo presión. Si el sistema lee saldos de SIM como condición inicial, arrastra el problema y el motor de forecast opera sobre basura.
- **Mitigación:** el primer ciclo del sistema obliga un **conteo asistido inicial** (C-3) por tienda antes de activar las sugerencias automáticas; el conteo asistido se vuelve la fuente primaria; SIM se sincroniza desde el sistema, no al revés.
- **Dueño tentativo:** arquitectura + operaciones.

### R-2 — Resistencia al cambio operativo

- **Descripción:** los jefes pueden no confiar en la sugerencia y seguir calculando "a ojo" en paralelo; los coordinadores pueden seguir cruzando correos por costumbre. La adopción no es técnica, es organizacional.
- **Mitigación:** plan formal de capacitación + anuncio oficial de que la tarea pasa a coordinadores; identificación temprana de jefes "champions" para pilotaje; iteración rápida en las primeras semanas para construir confianza.
- **Dueño tentativo:** célula de UX / Eficiencia operativa + dirección de operaciones.

### R-3 — Definición de "merma sospechosa" — falsos positivos erosionan confianza

- **Descripción:** umbrales mal calibrados generan demasiados candidatos falsos; el coordinador descarta sistemáticamente y deja de mirar la bandeja → la detección de mermas se vuelve folclore.
- **Mitigación:** umbrales conservadores en v1 (priorizar precision sobre recall), iteración trimestral con datos reales, **definir métrica de calidad de detección** (precision objetivo `[OPEN: a definir con coordinadores]`).
- **Dueño tentativo:** PM + coordinadores piloto + Finanzas (validar el costo del falso negativo vs falso positivo).

### R-4 — Incompatibilidad downstream con el comprador

- **Descripción:** el export `SolicitusDeInsumosTodos.xlsx`-compatible se rompe si la estructura cambia o si el comprador modifica su flujo. Cada incompatibilidad obliga al coordinador a corregir manualmente — pierde el ahorro.
- **Mitigación:** revisar `docs/SolicitusDeInsumosTodos.xlsx` para fijar contrato; involucrar al comprador en pilotaje desde semana 1; tests automatizados de export contra fixture firmada (NFR-T-2).
- **Dueño tentativo:** arquitectura + PM.

### R-5 — Dependencia técnica y organizacional de SIM

- **Descripción:** sin claridad sobre el dueño y la API de SIM, el modelo de integración (§10.4) queda abierto y bloquea decisiones de arquitectura.
- **Mitigación:** identificar al dueño de SIM antes de cerrar `architecture.md`; si la integración no es viable en v1, decidir coexistencia consciente con plan de migración v2.
- **Dueño tentativo:** PM + IT.

### R-6 — Tasa de override del coordinador alta inicialmente erosiona la propuesta de valor

- **Descripción:** en los primeros ciclos el motor no tiene calibración fina; el coordinador puede overridear el 60%+ de las sugerencias. Si esto se vuelve patrón, el sistema queda como "Excel sofisticado".
- **Mitigación:** definir **contra-métrica de override < 20% por ciclo** (§6) como gate de éxito; iterar motor rápido en pilotaje; capturar razón estructurada de cada override para análisis.
- **Dueño tentativo:** PM + arquitectura.

### R-7 — Acuerdo MAPE/WAPE con Finanzas se posterga

- **Descripción:** sin un número objetivo acordado con Finanzas, no hay vara de medida del motor. El proyecto queda expuesto a "ya pero ¿qué tan bueno es?".
- **Mitigación:** sesión formal con Finanzas en las primeras 4 semanas para acordar baseline; `BacktestingSuite.java` corre sobre histórico real desde el primer sprint útil.
- **Dueño tentativo:** PM + Finanzas.

### R-8 — Performance del motor de forecast a escala 226 × N SKUs

- **Descripción:** Java puro con Smile/Commons Math puede no cumplir el SLA cuando se corre sobre el universo completo si los modelos seleccionados son pesados.
- **Mitigación:** prototipado temprano del motor en el primer sprint útil; medir tiempos con histórico real; activar el "trigger objetivo para reevaluar Python como microservicio" de `project-context.md` si los umbrales se rompen.
- **Dueño tentativo:** arquitectura.

### R-9 — Pilotaje sin baseline cuantitativo

- **Descripción:** si arrancamos pilotaje sin medir el statu quo (horas del jefe, tasa de quiebre, sobre-inventario), no podemos demostrar mejora.
- **Mitigación:** **antes del go-live**, recolectar baseline en 3-5 tiendas piloto durante un par de ciclos (tiempo del jefe cronometrado, recuento de quiebres, snapshot de bodega).
- **Dueño tentativo:** PM + coordinadores piloto.

### R-10 — Express vs Estándar — supuesto de "mismo catálogo" se rompe

- **Descripción:** confirmamos en Discovery que ambos formatos manejan mismo catálogo de servicios, pero si en operación real Express maneja un subset reducido y el motor lo ignora, hay sobre-pedido.
- **Mitigación:** validar el supuesto al inspeccionar `Skus_insumos.csv` y `Presupuesto-tiendas.csv` (sección §10.3); si surge diferencia, modelar como subset por formato.
- **Dueño tentativo:** arquitectura + análisis de datos.

## 13. Open Questions

Lista consolidada de decisiones abiertas. Cada una tiene contexto, propuesta inicial y dueño tentativo.

### OQ-101 — Frecuencia real de excepciones mid-cycle

Provisional `[ASSUMPTION: 1-3 por semana en la red de 226]`. **Medir post-lanzamiento** con telemetría. No bloquea v1; afecta el diseño visual de prominencia de la bandeja del coordinador territorial. **Dueño:** PM.

### OQ-102 — Estructura exacta de `SolicitusDeInsumosTodos.xlsx`

Revisar el archivo de referencia en `docs/`. Definir el contrato del export (columnas, orden, tipos, encabezados, hoja). Fijar como fixture inmutable para tests de contrato (NFR-T-4). **Dueño:** arquitectura + PM.

### OQ-103 — Modelo de integración con SIM

Tres modelos en §10.4: (A) integración, (B) reemplazo, (C) coexistencia. Decisión depende de dueño de SIM, su API y otros consumidores. **Dueño:** arquitectura + IT.

### OQ-104 — ¿`Presupuesto-tiendas.csv` distingue formato Express / Estándar?

Verificar al inspeccionar el archivo. Si no lo distingue, levantar data master nueva. **Dueño:** análisis de datos.

### OQ-105 — Costo logístico estimado para sugerencia de TFS

C-8 (FR-071) considera "costo logístico estimado" para validar que la TFS es preferible a compra al proveedor. Hoy no se modela. **Decisión:** ¿costo fijo por TFS, costo por kilómetro, costo cero? Operaciones tiene que decir. **Dueño:** operaciones + PM.

### OQ-106 — Umbrales cuantitativos de "merma sospechosa"

C-9 detecta candidatos por patrón de desviación; el umbral concreto (cuánta divergencia consumo vs venta cuenta como merma) requiere análisis de datos históricos + criterio de los coordinadores. **Dueño:** PM + coordinadores piloto + análisis de datos.

### OQ-107 — Objetivo de MAPE / WAPE acordado con Finanzas

Sin número, no hay vara de medida del motor. **Sesión formal a agendar en las primeras 4 semanas.** **Dueño:** PM + Finanzas.

### OQ-108 — Ventana de histórico mínimo para forecast

¿Tiendas nuevas con < 12 meses de histórico — cómo se tratan? Opciones: (a) usar histórico distrital agregado, (b) bootstrap con tienda similar, (c) recoger 12 meses antes de activar sugerencia. **Dueño:** arquitectura.

### OQ-109 — Selección de tiendas piloto

¿Cuántas tiendas, qué mezcla Express/Estándar, qué distritos, cuántos ciclos antes de generalizar? **Dueño:** PM + coordinadores.

### OQ-110 — Flujo cuando el comprador rechaza una solicitud extraordinaria

UJ-3 confirmado: el comprador **puede rechazar** una compra directa al proveedor. ¿Qué hace el coordinador en ese caso? ¿Vuelve a sugerir TFS? ¿El jefe avisa al cliente? El flujo de rechazo no está detallado. **Dueño:** PM + operaciones.

### OQ-111 — Scope de "otros insumos" para v2

Confirmado que insumos no-papel están fuera de v1. Para planeación de v2 conviene saber el universo: toners, cartuchos, suministros varios. **Dueño:** PM + operaciones.

### OQ-112 — Datos maestros nuevos: ¿quién los gobierna?

El sistema necesita data maestra que no existe en `docs/`: asignación territorial Tienda → Coordinador, calendario de rotación de guardia, umbrales de mermas, plazos de ciclo, lista de razones estructuradas. **Decisión:** ¿hay un admin operativo? ¿es el PM? ¿es IT? **Dueño:** PM + dirección.

## 14. Glosario

- **Servicentro:** centro de copiado dentro de una tienda Office Depot México que atiende clientes con servicios de impresión, fotocopia, encuadernación.
- **Jefe de servicentro:** empleado de la tienda responsable de operar el servicentro. Hoy calcula manualmente la solicitud de insumos quincenal.
- **Coordinador (de distrito):** uno de tres roles (Carlos, Marco, Eduardo) que supervisa tiendas asignadas territorialmente para excepciones y rota guardia semanal para consolidación.
- **Comprador (corporativo):** rol corporativo que materializa órdenes de compra al proveedor. Recibe el consolidado del coordinador. Fuera del alcance del sistema.
- **Ciclo quincenal:** periodicidad de solicitud de insumos por tienda. Cada tienda tiene asignado "semana 1 y 3" o "semana 2 y 4" del mes (numeración ISO 8601).
- **Bodega:** espacio físico **dentro del servicentro de la tienda** donde se almacena el papel. No es el almacén central (WH).
- **WH:** *warehouse* — almacén central de Office Depot México que surte a las tiendas vía ALLOC.
- **ALLOC:** orden de surtido del WH a la tienda. Captura en archivos `ALLOC_*.csv`.
- **TFS:** *transfer* — transferencia física de inventario entre tiendas. Captura en archivos `TFS_*.csv`. **No consume presupuesto de la tienda receptora.**
- **SKU venta:** identificador del producto vendido al cliente final (`item_id_venta`).
- **SKU insumo:** identificador del producto comprable al proveedor (caja, paquete). Es sobre el que se cuantiza al empaque mínimo.
- **Cruce SKU:** tabla autoritativa que mapea SKU venta → SKU insumo (`Cruce-de-skus-venta-insumo.xlsx`).
- **Empaque mínimo / cuantización:** múltiplo de piezas (`Contenido` en `Skus_insumos`) al que toda cantidad sugerida debe redondearse hacia arriba.
- **Techo presupuestal semanal:** monto MXN máximo que una tienda puede gastar en un ciclo. `Cantidad por semana` en `Presupuesto-tiendas.csv`. **Rígido por tienda — no se mueve entre tiendas.**
- **SIM:** sistema interno actual donde se registran movimientos manuales de bodega. Captura manual, dato hoy sucio.
- **Excepción mid-cycle:** evento entre ciclos quincenales que rompe la planeación (venta extraordinaria, consumo más rápido del previsto, descubrimiento de merma).
- **Override:** modificación manual de una cantidad sugerida por el sistema, con razón estructurada capturada.
- **MAPE / WAPE:** *Mean Absolute Percentage Error* / *Weighted Absolute Percentage Error*. Métricas de calidad del motor de forecast.
- **Fail-loud:** disciplina de manejo de errores en la que toda condición ambigua o dato faltante dispara una excepción explícita en lugar de degradarse silenciosamente.

---

**Estado actual del PRD:** `draft` — secciones 1-14 completas. Pendiente: Reviewer Gate + Finalize.

