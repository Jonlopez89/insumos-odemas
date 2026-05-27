---
title: PRD — insumos-odemas
project: insumos-odemas
status: draft
created: 2026-05-22
updated: 2026-05-26
owner: Jonathan
language: es-MX
---

# PRD — insumos-odemas

## Resumen ejecutivo

**insumos-odemas** es un sistema que **automatiza la consolidación distrital semanal** de la solicitud de insumos de papel para los 226 servicentros (centros de copiado) de Office Depot México. Sustituye las **9 horas que hoy invierte cada coordinador, semana a semana**, en armar a mano su parte del `SolicitusDeInsumosTodos.xlsx` con un flujo en el que el motor **predice la demanda por tienda × SKU basándose ~99% en el histórico de ventas** (cadencia semanal del motor; unidad quincenal por tienda), aplica un **único factor de pérdida configurable por tipo de papel** (≈0.9% de la venta, calibrable) como colchón, ajusta la cantidad a pedir descontando lo que ya está **asignado/en tránsito (ALLOC/TFS)**, sugiere cantidades respetando el techo de presupuesto individual y presenta histórico de transferencias TFS más un **dashboard de validación pedido-vs-recibido**. **El producto se concibe en 3 fases (encuadre del owner 2026-05-26): (1) Predicción ← v1; (2) Inventario/solicitud; (3) Rebajas/merma precisa.** v1 cubre la Fase 1 más el soporte a la consolidación; la **medición precisa de merma por tienda es Fase 3, fuera de v1**. **El jefe de servicentro no opera el sistema en v1**: sigue capturando altas y bajas en SIM como hoy, y v1 **no fuerza captura ni baja de inventario** (basarse en el conteo SIM "siempre deja torcida" la solicitud — owner 2026-05-26). El sistema toma SIM como **input externo unidireccional** de saldo (actualización diaria por extract nocturno) y se apoya en ALLOC/TFS como cross-check. La salida es **compatible con la estructura validada** de `docs/SolicitusDeInsumosTodos.xlsx`; **el coordinador opera en pantalla, no en Excel — el Excel es el output hacia compras, no el input** (FR-061). **Canal oficial único para solicitudes esporádicas y excepciones:** correo electrónico (WA y llamadas existen como notificación informal previa, pero no entran al flujo del sistema). **Ahorro v1:** de 9h/coord/semana a < 2h/coord/semana ≈ **~1,092 horas-hombre/año** entre los tres coordinadores (~136 días-hombre/año a 8h/día). Cada coordinador trabaja su distrito todas las semanas; la guardia rotativa aplica solo a excepciones inter-distritales, no a consolidación regular.

---

## 1. Visión

Un sistema que **predice, sugiere, valida, traza y consolida** la solicitud distrital de insumos para los 226 servicentros de OD México — **automatizando la consolidación semanal que hoy hace a mano cada coordinador para su propio distrito** (cadencia semanal del motor; unidad quincenal por tienda — semana 1y3 vs 2y4 alternadas). La predicción se basa **~99% en el histórico de ventas y ~1% en un factor de pérdida** (un único factor configurable por tipo de papel, ≈0.9% de la venta), respetando SIM como herramienta única de captura de inventario del jefe (el sistema lee snapshots de SIM, no escribe ni sustituye su flujo, **y no fuerza la captura de inventario**) y procesando solo solicitudes esporádicas o excepciones formalizadas por correo electrónico. **El producto vive en 3 fases — (1) Predicción, (2) Inventario/solicitud, (3) Rebajas/merma precisa — y v1 entrega la Fase 1 más el soporte a la consolidación;** la medición precisa de merma por tienda es Fase 3. El v1 reduce un día completo de consolidación por coordinador por semana a una sesión de revisión y aprobación; el cálculo manual de 6h/quincena del jefe **no es dolor que v1 ataque directamente** — se atiende indirectamente vía mejor forecast del coordinador (ver §3-bis).

## 2. Contexto operativo

### Universo del producto

- **226 servicentros** distribuidos en formato **Express** (tiendas más chicas con menos espacio físico) y **Estándar** (formato completo). Ambos formatos manejan el mismo catálogo de servicios e insumos; el formato no afecta la lógica de forecast.
- **Tres coordinadores** (Carlos, Marco, Eduardo) operan con **asignación territorial fija** (zona + formato). Cada coordinador trabaja **siempre** las tiendas de su distrito asignado, **tanto para consolidación regular semanal como para excepciones mid-cycle**. La "guardia rotativa" entre coordinadores aplica únicamente a tareas inter-distritales (consolidación agregada cuando se requiera, excepciones cross-territorio, etc.) y la organizan ellos fuera del sistema — el sistema no la gobierna ni la consume.
- **Comprador corporativo** (rol único) recibe los concentrados de los tres coordinadores por correo y materializa las órdenes de compra al proveedor. **Fuera del alcance del sistema**, pero consumidor downstream — el output del sistema debe ser compatible con su flujo actual.

### Cadencia operativa

- Cada tienda tiene un **ciclo quincenal** asignado en `Presupuesto-tiendas.csv`: "semana 1 y 3" o "semana 2 y 4" (numeración ISO 8601). Aproximadamente la mitad del universo de tiendas (~113) pide cada semana. **Por lo tanto, cada coordinador consolida su distrito todas las semanas** (no una semana sí, una no): el motor opera con **cadencia semanal** aunque la unidad de pedido por tienda sea quincenal.
- La **consolidación normalmente arranca los miércoles**. Lunes-martes los jefes envían su solicitud por correo; miércoles-jueves cada coordinador consolida su distrito; los tres concentrados se entregan al comprador antes del viernes.
- Las **excepciones mid-cycle** (ventas extraordinarias, faltantes inesperados, disparos por temporalidades o por uso interno) ocurren con frecuencia indeterminada `[ASSUMPTION: 1-3 por semana en la red completa, a medir post-lanzamiento]`. **El canal oficial único es el correo electrónico**: las llamadas y WhatsApp del jefe son notificación informal previa pero **no disparan atención formal del sistema ni del coordinador** hasta que el jefe formaliza por correo. El correo se dirige al coordinador **territorial** de la tienda afectada.

### Presupuesto por tienda

- `Presupuesto-tiendas.csv` define un **techo MXN semanal individual por tienda**. Estos techos ya están calibrados por venta histórica — tiendas que venden más reciben presupuesto mayor.
- **No hay pool distrital ni reasignación de MXN entre tiendas.** El coordinador valida que cada solicitud individual ≤ techo de esa tienda; si excede, recorta a esa tienda.
- Las **transferencias físicas entre tiendas (TFS)** son un mecanismo paralelo que **no consume presupuesto de la tienda receptora** — son flujo de inventario, no de gasto.

### Herramientas y sistemas actuales

- **SIM** — sistema interno donde los jefes capturan altas y bajas de inventario de bodega. La captura es manual y, bajo presión operativa, **frecuentemente imprecisa**. Provee visibilidad de tránsito (ALLOC, TFS) a los coordinadores. **SIM sigue siendo la herramienta única del jefe en v1; el sistema nuevo no la sustituye ni le ofrece UI al jefe.** El sistema nuevo lee snapshots periódicos de SIM como input de saldo (lectura unidireccional, modelo (C) Coexistencia — ver §10.4). La calidad sucia del dato en SIM se mitiga vía factor de merma esperada por tienda × mes (C-9) y backtesting, **no vía conteo asistido** (fuera de scope v1).
- **Excel + correo electrónico** — vehículo principal del flujo de solicitud actual. Cada jefe llena un Excel con su solicitud; cada coordinador arma su parte del `SolicitusDeInsumosTodos.xlsx` consolidando las ~38 solicitudes semanales de su distrito (~113 entre los tres). El archivo `docs/SolicitusDeInsumosTodos.xlsx` está en el repositorio como referencia del formato actual. **En v1 el jefe sigue enviando su Excel al coordinador como hoy**; el sistema apoya al coordinador en la consolidación.
- **WhatsApp / llamadas** — canal **informal previo**, no oficial. Existen como mecanismo de aviso anticipado del jefe pero **no entran al flujo formal del sistema ni del coordinador**: la atención formal arranca cuando el jefe formaliza su excepción por correo electrónico al coordinador territorial. **En v1 el canal jefe → coordinador formal es exclusivamente correo electrónico**; lo que pasa al sistema es la decisión y el escalamiento del coordinador territorial (C-10 reformulada).

### Restricciones del entorno

El stack tecnológico obligatorio (Java + Spring Boot + Angular sobre GCP), las convenciones de datos (locale es-MX, fechas `dd/MM/yyyy`, encoding CSV variable, `BigDecimal` para montos, fail-loud sobre datos faltantes), y la disciplina de testing (golden datasets, backtesting, property-based testing con jqwik, contract testing) están definidas en `_producto/project-context.md`. No se repiten aquí.

## 3. El problema

**La consolidación distrital del coordinador es costosa, lenta, opaca y no escala a 226 tiendas operando con cadencia semanal.** El dolor v1 vive en el coordinador. El dolor histórico del jefe se documenta pero **se atiende solo indirectamente** vía mejor forecast del coordinador (ver §3-bis).

### Dolor del Coordinador de distrito

| Código | Dolor |
|---|---|
| **D-coord-1** | **9 horas por cada coordinador, todas las semanas.** Cada coordinador consolida su propio distrito semana a semana — la espera de los Excel de los jefes, el cruce manual con SIM, la validación contra techo presupuestal y el armado del consolidado consumen 9h efectivas (cronometrado por owner 2026-05-25). **~1,404 horas-hombre/año** entre los tres coordinadores (9h × 3 × ~52 semanas) ≈ ~175 días-hombre/año a 8h/día. |
| **D-coord-2** | Si una tienda no envía su Excel a tiempo, el coordinador no la persigue (perseguir tiendas atrasadas no es su responsabilidad, ver D-coord-7) — **la tienda salta a la siguiente semana aunque no esté en su ciclo nominal** (1y3 vs 2y4), generando carry-over informal sin registro automático. Hoy ese carry-over vive en la cabeza del coord; si se le pasa, la tienda queda sin pedido y se garantiza quiebre. |
| **D-coord-3** | Validar `cantidad × costo ≤ techo de presupuesto` para ~38 tiendas/semana × N SKUs **a mano en Excel** es lento y propenso a error aritmético. **Operativamente los coordinadores no se han excedido del techo**, así que el dolor es **preventivo (consistencia, escala, auditabilidad), no curativo** — no se trata de rescatar al coord del precipicio, se trata de quitarle la carga de chequeo manual y dejar evidencia de que se validó. |
| **D-coord-4** | Las **oportunidades de transferencia entre tiendas (TFS)** dependen de la memoria del coordinador. En 226 tiendas se le escapan casos. La ejecución de TFS vive en otro proceso/programa; v1 solo aporta visibilidad del histórico para informar la decisión. |
| **D-coord-5** | Sin trazabilidad: cuando el coordinador recorta una tienda de 10 a 4 piezas, **en ningún lado queda escrito por qué**. Sin evidencia para auditoría ni para retroalimentar al jefe. |
| **D-coord-6** | El **canal informal previo** (WhatsApp, llamadas) **no es canal de entrada al flujo**: el sistema solo procesa excepciones que el jefe formaliza por **correo electrónico**. Aun así, en el correo no hay priorización nativa — una alerta urgente puede perderse en la avalancha de la bandeja del coord. v1 **historia** la decisión y escalamiento (C-10), no resuelve la priorización en línea del canal correo. |
| **D-coord-7** | **Perseguir tiendas que no enviaron su solicitud no es responsabilidad del coordinador — es del jefe del servicentro.** El coordinador necesita **visibilidad pasiva** del estado de cada tienda en su distrito (envió / pendiente / con recorte / aprobada / carry-over de semana anterior) para planear, no para perseguir. Hoy no tiene **dashboard agregado** que muestre cuánto presupuesto se ha consumido, qué SKUs concentran riesgo, qué tiendas piden por encima de su techo, o qué tiendas vienen arrastrando carry-over. |

### Dolor del flujo de excepción mid-cycle (parcialmente atendido en v1)

| Código | Dolor | Tratamiento v1 |
|---|---|---|
| **D-exc-1** | El jefe puede contactar por WA o llamada pero **solo el correo electrónico formaliza la solicitud**. El canal informal previo existe pero no entra al sistema hasta que el jefe genera correo. | **Historiado** en v1 — el correo es el único disparador del flujo; WA/llamada quedan como ruido previo no procesable por el sistema. La avalancha de correos en sí misma no se prioriza. Se reabre en v1.1. |
| **D-exc-2** | Las excepciones mid-cycle tienen **dos disparadores conocidos**: (a) **temporalidades** — eventos del calendario comercial (Hot Sale, BTS, Buen Fin, Navidad, etc.) que disparan demanda atípica; (b) **consumo más rápido de lo previsto** (incluida la pérdida no anticipada) que agota inventario antes de tiempo. Hoy ninguno de los dos se anticipa sistemáticamente. | **Parcial / atendido**: (a) temporalidades atendidas por componente estacional del motor (FR-033), apoyado en que la estacionalidad **ya viene embebida en el histórico de ventas YoY** (C-O39). (b) El consumo no-venta se absorbe vía el **factor único de pérdida** (C-9 reformulada). Las alertas siguen llegando por correo del jefe y se registran post-hoc. |
| **D-exc-3** | El coordinador decide TFS vs compra directa **sin ver en pantalla** qué tienda cercana tiene exceso del SKU faltante. La validación de inventario por tienda **vive en SIM** (decisión cerrada en §10.4 — refuerza coexistencia C). | **Atendido** vía contexto agregado en la bandeja del coordinador territorial (saldo SIM snapshot, ALLOC pendiente, histórico TFS) — C-10 FR-092. SIM sigue siendo fuente única de validación de inventario; el sistema solo lee snapshots. |
| **D-exc-4** | No queda trazado quién decidió qué, cuándo, ni por qué. | **Atendido** desde el momento en que el coordinador registra la decisión en el sistema — bitácora C-11. |
| **D-exc-5** | La compra extraordinaria al proveedor **no se refleja en el saldo de planeación** del siguiente ciclo, y además **hoy no se descuenta del techo presupuestal de la tienda** — la compra esporádica consume presupuesto que el ciclo regular no reconoce, generando doble contabilidad invisible. | **Atendido** — FR-095 refleja la compra extraordinaria registrada en el saldo del modelo de planeación. **Nueva regla (FR-054)**: el presupuesto disponible del ciclo regular se calcula como `techo_semanal − compras_esporádicas_del_periodo`, leyendo `Entrega-directa-tienda.csv` (TRAN_CODE 20 "Purchases"). SIM se sincroniza por su flujo propio (no por este sistema). |

### Dolor cross-corte — pérdidas invisibles

| Código | Dolor |
|---|---|
| **D-merma-1** | El sistema actual no detecta el patrón **"venta plana o decreciente + solicitudes de insumo crecientes"** que sugiere pérdida. Los coordinadores lo intuyen pero no lo sistematizan. **Decisión del owner 2026-05-26: la pérdida es un gasto único, no dos componentes** — *"al final del día es un gasto, es uno solo"*. v1 la modela como **un único `factor_perdida` configurable por tipo de papel** (≈0.9% de la venta, calibrable), aplicado como colchón al forecast (C-9 reformulada, FR-035). **La medición precisa de la pérdida por tienda (cuánto se rebajó vs cuánto se compró) es la Fase 3 del producto, fuera de v1** (C-O28/C-O30). La detección estadística residual (FR-080) es una capa secundaria opcional para surface lo atípico, no el corazón de v1. *(Revierte el modelo dual D-4.)* |

## 3-bis. Dolor histórico fuera del alcance v1

Estos dolores son reales pero **v1 no los ataca directamente**. Se documentan para mantener visibilidad del actor en el ecosistema y para informar v2. El jefe sigue calculando a mano si quiere — el sistema no le quita ni le da nada en v1; lo que sí ocurre es que **un mejor forecast del coordinador reduce indirectamente la presión sobre el jefe** al disminuir quiebres y rechazos en mostrador.

### Dolor del Jefe de servicentro (histórico, no atendido directamente en v1)

| Código | Dolor |
|---|---|
| **D-jefe-1** | **Hasta 6 horas por quincena** dedicadas al cálculo manual, compitiendo con la atención al personal de servicentro y a clientes en mostrador. *No es ahorro v1.* |
| **D-jefe-2** | El **conteo físico de bodega** se hace bajo presión operativa y queda mal registrado en SIM. Los saldos sucios contaminan el siguiente ciclo de planeación. *v1 no sanea SIM; mitiga vía factor de merma esperada (C-9) y backtesting.* |
| **D-jefe-3** | El cálculo es **empírico, a ojo, en función de la temporada** — sin método formal ni fórmula reproducible. Conocimiento en la cabeza del jefe; si rota personal, su reemplazo arranca de cero. *v1 no le da herramienta al jefe; el coordinador opera con motor estacional.* |
| **D-jefe-4** | Sin visibilidad de inventario en tránsito (ALLOC pendiente, TFS en camino) → riesgo de pedir de más por desconocer lo que ya viene. *El coordinador sí ve esta visibilidad en v1.* |
| **D-jefe-5** | Cuando hay excepciones (ventas extraordinarias), aunque el jefe pueda avisar primero por WhatsApp o llamada, **tiene que formalizar por correo electrónico** para que la solicitud entre al flujo del coordinador, y **esperar autorización mientras el cliente lo mira en mostrador**. *Sigue ocurriendo en v1; el correo es el único canal formal.* |

**Concesión epistémica central:** v1 asume la divergencia físico-teórico en SIM como precondición y la **mitiga parcialmente vía el factor único de pérdida** (≈0.9% de la venta, configurable por tipo de papel), no la resuelve. Además, v1 **no se ancla al conteo SIM** para calcular la demanda — la base es la venta (~99%), y el ajuste del pedido usa ALLOC/TFS como cross-check (owner 2026-05-26). Trade-off consciente del owner: scope manejable a cambio de no atacar la causa raíz. La medición precisa de la pérdida es Fase 3. Documentado también en §11.16, §11.17 y R-1.

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
| **Validación de presupuesto a escala (preventiva)** | Calcular `cantidad × costo ≤ techo` para ~38 tiendas/semana × N SKUs a mano es lento y propenso a error aritmético, **aunque operativamente los coords no se hayan excedido**. El dolor real es de consistencia, escala y auditoría — no de rescate. | Validación automática por tienda, sugerencia de recorte cuando excede, descuento de compras esporádicas del techo (FR-054), indicador agregado del consumo distrital con evidencia trazable. |
| **Histórico y análisis cruzado de TFS** | Requiere mirar el universo entero (exceso vs déficit, semana a semana) — imposible mantenerlo en hojas separadas con el ciclo operativo del coordinador. | Histórico TFS ingestado semanalmente desde `TFS_*.csv`, visualización cruzada por tienda, SKU y periodo. Alimenta como input al motor de forecast. *v1 no decide ni ejecuta TFS — esa decisión vive en otro proceso/programa.* |
| **Factor de pérdida + detección estadística residual** | Aplicar un factor de pérdida calibrable por tipo de papel sobre el forecast y, encima, detectar consumo atípico no explicado por la venta, es trabajo de motor sobre el universo de 226 tiendas — no de fórmula mantenida a mano. | Coordinador territorial configura un único `factor_perdida` por tipo de papel (default global ≈0.9%); el motor lo aplica como ajuste sobre la demanda de venta; una capa secundaria de detección estadística residual surface candidatos atípicos para registro humano. *(La medición precisa de pérdida por tienda es Fase 3, fuera de v1.)* |
| **Trazabilidad y auditoría** | Excel no registra quién cambió qué y por qué. Si el comprador audita, el rastro está perdido. | Bitácora automática: sugerencia original → override → razón estructurada → timestamp → actor. Exportable. |
| **Fail-loud sobre datos faltantes** | Excel silencia errores con celdas vacías. Si falta equivalencia SKU venta↔insumo, se infiere o se ignora. | Excepciones de negocio explícitas (`EquivalenciaNoDefinidaException`, `MinimoPackNoConfiguradoException`, etc.) abortan el ciclo de esa tienda hasta resolver — ver `project-context.md`. |
| **Reflejo automático en el siguiente ciclo** | Cada Excel es un punto en el tiempo; el siguiente arranca de cero. | El estado persiste; lo decidido la quincena pasada alimenta el motor de la próxima, incluyendo compras de excepción registradas. |

### La cuenta dura del statu quo

- **Coordinador:** 9h por consolidación × ~52 semanas/año × 3 coordinadores ≈ **~1,404 horas-hombre/año** (~175 días-hombre/año a 8h/día) en consolidación. **Este es el ahorro v1** — cada coordinador trabaja su distrito semana a semana, no rotan la consolidación regular. Objetivo: bajar de 9h/coord/semana a < 2 h/coord/semana ≈ **ahorro estimado ~1,092 horas-hombre/año** (~136 días-hombre/año a 8h/día).
- **TFS no informadas:** cada transferencia que el coordinador no recordó se traduce en compra extra al proveedor. v1 da visibilidad del histórico TFS (semana a semana, cruzado por tienda y SKU) para informar mejor las decisiones que se toman fuera del sistema. Baseline cuantitativo `[ASSUMPTION: medir en pilotaje]` — **ahorro intangible que el histórico empieza a hacer visible**.
- **Pérdidas invisibles:** sin un factor explícito, hoy la pérdida se contabiliza como consumo legítimo y contamina la señal de demanda. v1 la modela como **un único factor de pérdida configurable por tipo de papel** (≈0.9% de la venta, calibrable) — un colchón pequeño, no un modelo elaborado. Surface candidatos residuales atípicos para registro. **El desglose fino y la medición precisa por tienda son Fase 3** (rebaja de inventario), fuera de v1.
- **Cálculo manual del jefe (~6h × 226 tiendas × ~26 ciclos = decenas de miles de horas/año):** **NO es ahorro v1**. Se documenta en §3-bis y queda como upside futuro para v2 si se decide volver al jefe.

Excel no es competidor del sistema en v1. **Excel es la herramienta del coordinador que el sistema sustituye.**

## 5. Objetivos del producto

Cuatro objetivos, en orden de prioridad:

### O1 — Sustituir el cálculo manual de la consolidación distrital semanal del coordinador

Cada coordinador deja de armar a mano su parte del `SolicitusDeInsumosTodos.xlsx` semana a semana. El sistema consolida automáticamente las ~38 solicitudes semanales de cada distrito, valida techos individuales con descuento de compras esporádicas del periodo, sugiere recortes cuando exceden, presenta histórico TFS cruzado y dashboard de validación pedido-vs-recibido para informar decisiones, aplica el factor único de pérdida por tipo de papel al forecast y produce el export compatible con la estructura validada del archivo de referencia. **Las 9 horas por coordinador por semana se reducen a < 2 h ≈ ~1,404 horas-hombre/año bajan a ~312 horas-hombre/año, ahorro estimado ~1,092 horas-hombre/año entre los tres (~136 días-hombre/año a 8h/día).**

### O2 — Eliminar quiebres y sobre-inventario

Ningún servicentro debe quedarse sin papel en su ciclo normal, **y** ningún servicentro debe acumular papel ocioso. La métrica operativa más visible es **tasa de quiebres en mostrador → cerca de cero** acompañada de **rotación de inventario en rangos sanos por tienda**. Se logra vía mejor forecast del coordinador, no vía intervención sobre el jefe.

### O3 — Informar (no sustituir) la decisión humana sobre TFS con visibilidad histórica

El sistema entrega al coordinador el dato cruzado de TFS semana a semana (por tienda, SKU, periodo) para **informar** las decisiones de transferencia que **se siguen tomando en otro sistema/proceso**. v1 no decide ni ejecuta TFS — solo registra histórico, lo expone con filtros, y lo alimenta al motor de forecast como input. La decisión sigue siendo humana; el sistema le quita la dependencia de memoria.

### O4 — Incorporar un factor de pérdida calibrable y hacer visible el consumo atípico

La pérdida deja de ser folclore distrital y entra al forecast como **un único factor configurable por tipo de papel** (≈0.9% de la venta, calibrable en piloto). El coordinador territorial lo ajusta desde el aplicativo a discreción; default global si no se configura. El factor **alimenta el motor de forecast** como ajuste porcentual sobre la demanda de venta (decisión D-9, revierte D-4). Una capa **secundaria** de detección estadística residual surface candidatos de consumo atípico para registro humano, acumulados para revisión periódica. **La medición precisa de la pérdida por tienda — cuánto se rebajó vs cuánto se compró — es la Fase 3 del producto, fuera de v1** (C-O28/C-O30). O4 de v1 se reduce a "aplicar un colchón de pérdida calibrable", no a "medir merma".

## 6. Métricas de éxito (preliminar)

Las metas cuantitativas finales se acuerdan con Finanzas en pilotaje. Las propuestas iniciales son:

| Métrica | Baseline | Objetivo v1 | Cómo se mide |
|---|---|---|---|
| **Tiempo del coordinador por consolidación semanal** | **9 h / coord / semana** (cronometrado por owner 2026-05-25) = ~1,404 horas-hombre/año entre los tres ≈ ~175 días-hombre/año a 8h/día | **< 2 h / coord / semana** (~312 horas-hombre/año entre los tres → ahorro ~1,092 horas-hombre/año ≈ ~136 días-hombre/año a 8h/día) | Telemetría: tiempo entre primera apertura de la bandeja y export del consolidado por coordinador y por semana. **Métrica principal v1.** Vista desagregada por coordinador (los 3 deberían tener tiempos similares si los distritos son comparables). |
| **Tasa de quiebre (cliente rechazado por falta de insumo)** | `[OPEN: baseline cuantitativo de quiebres a recolectar en pilotaje]` — baseline de tiempo del coord ya cerrado (9h) | **Reducción ≥ 80%** | Contador en mostrador + reporte semanal de tiendas piloto. |
| **Sobre-inventario (cobertura en semanas de venta)** | `[OPEN: baseline]` — a recolectar en pilotaje | Por tienda, mantener en rango sano (a definir con Finanzas, R-7) | Reporte del sistema: cobertura por SKU/tienda. |
| **TFS registradas en histórico y analizadas por coordinador** | 0 (memoria del coordinador) | `[OPEN: meta v1 — métrica de output, no outcome — calibrar en pilotaje]` | Contador en sistema. **Visibilidad, no decisión** — la TFS se ejecuta fuera del sistema. |
| **MAPE / WAPE del motor de forecast** | n/a | `[OPEN: acordar con Finanzas en backtesting — bloqueo Reviewer Gate R-7]` | `BacktestingSuite.java` sobre histórico real — ver disciplina de testing en `project-context.md`. |
| **Candidatos de consumo atípico detectados y registrados** | 0 (no se sistematiza hoy) | `[OPEN: meta v1 — calibrar en pilotaje]` | Contador en sistema de candidatos residuales (FR-081) registrados por el coordinador. **Capa secundaria** — la medición precisa de pérdida por tienda es Fase 3, no v1. |
| **Factor de pérdida configurado por tipo de papel** *(seguimiento, no gate)* | Default global ≈0.9% al go-live | **Coordinadores ajustan por tipo de papel conforme observan necesidad** — sin meta absoluta | Reporte de configuración. **No es gate del go-live** (el default global ya cubre el caso base); es indicador de maduración del modelo. |
| **Override del coordinador — ciclos 1-3 (calibración)** | n/a | **≥ 25% por ciclo con razón estructurada registrada** (umbral firmado; meta de calibración óptima: ≥30%) | Contador en sistema con desglose por razón estructurada. **En esta fase el override es ORO de información para calibrar el motor**, no señal de desconfianza. Decisión Dr. Quinn (D-1 híbrido); umbral X=25% fijado en update 2026-05-25. |
| **Override del coordinador — ciclo 4+ (confianza)** | n/a | **< 20% por ciclo** | Contador en sistema. Se activa una vez que MAPE esté estabilizado y firmado con Finanzas. Hasta entonces, esta métrica se reporta solo como vista secundaria. |

### Contra-métricas (señales de que el sistema está fallando aunque las métricas principales se vean bien)

- **Tasa de override > 50% por ciclo** en alguna tienda *después de la fase de calibración*: el coordinador no confía en la sugerencia; el sistema está sobreajustando o pidiendo cosas absurdas.
- **Override estancado bajo 25% durante los ciclos 1-3** (la inversa de la métrica de calibración): el coordinador está corrigiendo poco, el sistema no recibe señal — investigar si la sugerencia es buena o si el coordinador está siendo pasivo.
- **Excepciones mid-cycle creciendo > 20% mes-sobre-mes después del lanzamiento**: el forecast inicial subestima sistemáticamente; calibrar el factor de pérdida por tipo de papel, o revisar la cobertura de eventos comerciales en FR-033.
- **Pedidos urgentes al proveedor (fuera de ciclo) creciendo > 15% mes-sobre-mes**: o el forecast falla o la coexistencia (C) con SIM está degradando.
- **`SolicitusDeInsumosTodos.xlsx` exportado pero el comprador sigue pidiendo correcciones**: incompatibilidad downstream con el flujo del comprador, ajustar el formato (depende de R-4).
- **Reducción del tiempo del coordinador pero crecimiento ≥ 10% de quiebres**: la automatización está omitiendo casos que el coordinador detectaba manualmente.
- **Divergencia entre saldo SIM y consumo real reportado por venta crece > 10% mes-sobre-mes** (acoplada con R-11): señal de que la coexistencia (C) se está rompiendo silenciosamente. Investigar SLA de frescura del snapshot SIM, captura del jefe en SIM, o factor de pérdida mal calibrado.
- **Carry-over creciente de pedidos tardíos en alguna tienda** (más de 1 ciclo arrastrado en > 20% de las semanas): señal de que el jefe está sistemáticamente atrasándose y la tienda termina servida de forma inconsistente. Acoplado con OQ-118.

---

## 7. Personas y User Journeys

### 7.1 Stakeholder externo — Jefe de servicentro (no es usuario del sistema en v1)

- **Quién es:** empleado de una tienda específica, 25-35 años. Lidera al personal del servicentro, gestiona la solicitud de insumos, atiende problemas operativos del área. No multitask cross-departamento.
- **Relación con el sistema en v1: cero interacción.** Sigue capturando altas y bajas en SIM como hoy y enviando su Excel de solicitud por correo al coordinador como hoy. El sistema no le provee UI ni le sustituye nada.
- **Por qué sigue documentado como persona:** mantener visibilidad del actor en el ecosistema, informar v2, y permitir que el equipo de UX no pierda la perspectiva del extremo del flujo. Su dolor histórico está en §3-bis.

### 7.2 Persona — Coordinador de distrito (usuario interactivo principal de v1)

- **Quién es:** uno de tres (Carlos, Marco, Eduardo). Full-time en servicentro pero responsabilidades más allá del inventario.
- **Universo asignado:** las 226 tiendas se reparten territorialmente entre los tres por zona + formato (~75 tiendas por coord). **Cada coordinador trabaja su distrito todas las semanas** — la asignación territorial es fija para consolidación regular y excepciones. La "guardia rotativa" aplica solo a tareas inter-distritales y la organizan ellos fuera del sistema.

**Dos modos de operación en el sistema:**

- **(a) Bandeja distrital — consolidación semanal.** Todas las semanas abre la bandeja **en pantalla** (no en Excel) con las ~38 tiendas de su distrito en ciclo esa semana (las que están en 1y3 o 2y4 según corresponda), ve sugerencias con el factor único de pérdida aplicado, validación de presupuesto descontando compras esporádicas del periodo, histórico TFS cruzado más el dashboard de validación pedido-vs-recibido, estado de cada tienda (incluyendo carry-over de semana anterior y "pendiente solicitud" como visibilidad pasiva), aplica overrides con razón estructurada, aprueba el consolidado, y **exporta el Excel como output hacia compras** (el Excel es el handoff, no el input).
- **(b) Bandeja territorial — excepciones y consumo atípico.** Atiende las excepciones mid-cycle que el jefe **formaliza por correo electrónico** (canal único). WA/llamadas previas no entran al sistema. Registra cada excepción con decisión: TFS de emergencia gestionada fuera del sistema, escalamiento a comprador para compra directa con contexto, rechazo. También revisa candidatos de consumo atípico residual del territorio (FR-080) y configura el **factor único de pérdida por tipo de papel** (FR-082).
- **(c) Administración** *(rol separado o función con permiso elevado).* Configura datos maestros, umbrales de detección, lista de razones estructuradas, lista de eventos comerciales. Requiere rol de Administrador.

- **Cómo opera hoy:** miércoles arranca la consolidación de su distrito. Recibe ~38 correos con Excel adjuntos, valida techo de presupuesto a mano por tienda, recorta cuando excede, se acuerda de TFS oportunas (por memoria), arma su parte del `SolicitusDeInsumosTodos.xlsx` y lo manda al comprador. Tiempo: un día completo, todas las semanas.
- **Qué lo hace feliz:** **dejar de armar a mano su parte del `SolicitusDeInsumosTodos.xlsx` semana a semana**. Recuperar el día completo que hoy se va en consolidación.

### 7.3 Persona — Comprador corporativo (stakeholder pasivo)

- **Hugo** — rol único que recibe los concentrados de los tres coordinadores y materializa órdenes de compra al proveedor. **Fuera del alcance del sistema** — no opera la UI. **Sí es consumidor downstream**: el export del sistema debe ser compatible con su flujo actual basado en `SolicitusDeInsumosTodos.xlsx` recibido por correo. **Identificado como comprador piloto (2026-05-25, NFR-T-2 cerrada)** — firma golden datasets y la fixture inmutable del export.

### 7.4 User Journey AS-IS — Cálculo quincenal del jefe (UJ-1) — *contexto histórico, sigue ocurriendo fuera del sistema*

UJ-1 sigue documentado para mantener visibilidad del flujo real del jefe que el sistema NO ataca en v1. Ver detalle completo en `addendum.md` (apéndice "UJ-1 AS-IS — Cálculo manual del jefe"). Resumen: ciclo quincenal → conteo físico → captura en SIM → cálculo empírico estacional en Excel → correo al coordinador, hasta 6 horas/quincena. **Dolores asociados:** D-jefe-1..5 (§3-bis).

### 7.5 User Journey AS-IS — Consolidación del coordinador (UJ-2)

1. (Miércoles, todas las semanas) coordinador entra al correo, encuentra ~38 Excel adjuntos de tiendas de su distrito en ciclo esa semana.
2. Recolecta solicitudes; observa quién no respondió (no persigue — es responsabilidad del jefe). Las tiendas atrasadas saltan a la siguiente semana en carry-over informal.
3. Consulta SIM para saldos, ALLOC pendiente y TFS en tránsito.
4. Valida cada solicitud contra su techo individual; recorta tiendas que exceden (operativamente raro — el dolor es de chequeo manual, no de rescate).
5. Por memoria, identifica oportunidades de TFS entre tiendas con exceso/déficit; **detecta** y **solicita** la TFS dentro de la jornada, pero la **ejecución** sigue por proceso paralelo (fuera del alcance v1).
6. Compila su parte del `SolicitusDeInsumosTodos.xlsx`.
7. Envía por correo al comprador (los tres concentrados llegan al comprador semana a semana).

**Tiempo:** un día completo, semana a semana. **Dolores:** D-coord-1..7.

### 7.6 User Journey UJ-3 — Excepción mid-cycle — AS-IS + TO-BE parcial

**Importante:** el flujo se desdobla explícitamente en **UJ-3-AS-IS** (lo que pasa fuera del sistema antes y después del trabajo del coord) y **UJ-3-TB** (lo que el coord opera dentro del sistema).

#### UJ-3-AS-IS — Lo que sigue ocurriendo fuera del sistema

1. Jefe detecta quiebre inminente por venta extraordinaria, evento comercial no anticipado, o consumo más rápido de lo previsto (incluido uso interno alto).
2. Jefe puede contactar a su coordinador territorial por **WhatsApp o llamada como aviso previo informal** — esto **no inicia atención formal del coordinador ni del sistema**.
3. Para que la excepción entre al flujo, el jefe debe **formalizar por correo electrónico** dirigido a su coordinador territorial. El correo es el único canal oficial.
4. *(Después del paso 6 de UJ-3-TB)* Si la compra extraordinaria al proveedor es autorizada, ocurre fuera del ciclo normal con el flujo del comprador.

#### UJ-3-TB — Lo que el coordinador opera dentro del sistema

1. Coordinador territorial recibe el correo formalizado, abre su bandeja territorial en el sistema.
2. **Registra** la excepción con: tienda, SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado (incluyendo categoría de disparador: temporalidad / uso interno / otra). Origen del aviso = correo electrónico (campo fijo).
3. Sistema muestra contexto agregado: saldo SIM más reciente, ALLOC pendiente, TFS en tránsito y histórico TFS reciente para el SKU.
4. Coordinador **decide y registra** la respuesta: gestionar TFS de emergencia fuera del sistema (registrar solo decisión y resultado), escalamiento al comprador para compra directa (sistema notifica con contexto), o rechazo con razón.
5. Si escalamiento: sistema notifica al comprador con todo el contexto + datos de tienda y SKU.
6. Si compra extraordinaria es autorizada: la **refleja en el saldo del modelo de planeación** del sistema (FR-095) **y descuenta del techo presupuestal de la tienda** para el ciclo regular siguiente (FR-054). SIM se sincroniza por su propio flujo.

**Dolores atendidos:** D-exc-3 (contexto en bandeja), D-exc-4 (bitácora), D-exc-5 (saldo + presupuesto reflejados). **Historiado (no atendido en línea):** D-exc-1 (canal correo es único, pero la avalancha de correos no se prioriza nativamente), D-exc-2 (disparadores temporalidad / uso interno reconocidos, pero la atención sigue siendo post-hoc).

### 7.7 User Journey TO-BE — Consolidación del coordinador con el sistema (UJ-2-TB)

1. **(Lunes-martes)** Jefes envían su solicitud por correo al coordinador territorial como hoy. *Fuera del sistema.*
2. **(Miércoles AM)** Cada coordinador abre la bandeja distrital de la semana. Sistema muestra las ~38 tiendas de su distrito en ciclo esa semana con su estado: **solicitud recibida** / **pendiente solicitud** (correo del jefe no llegado, visible pasivo — no es acción del coord) / **carry-over de semana anterior** / con recorte sugerido / aprobada / exportada. Snapshot SIM más reciente cargado como saldo inicial; compras esporádicas del periodo descontadas del techo presupuestal.
3. **(Miércoles AM)** Por cada tienda, el sistema presenta: cantidad sugerida por SKU con trazo explicable (incluyendo el factor único de pérdida aplicado por tipo de papel, factor estacional por evento comercial activo si corresponde, y el ajuste por ALLOC/TFS en tránsito), validación de techo presupuestal individual con descuento de compras esporádicas, recorte sugerido si excede, histórico TFS cruzado y dashboard de validación pedido-vs-recibido para informar contexto.
4. **(Miércoles PM)** Coordinador revisa, aplica overrides con razón estructurada (cuando proceda — recordatorio: en ciclos 1-3 el override es ORO de calibración, ver §6), aprueba tienda por tienda o en lote.
5. **(Jueves)** Coordinador exporta su parte del consolidado distrital al formato compatible con `SolicitusDeInsumosTodos.xlsx` y lo envía al comprador con CC configurable. Hash del archivo y destinatarios en bitácora.
6. **(Continuo durante la semana)** Cada coordinador atiende excepciones mid-cycle (UJ-3-TB) de su distrito en paralelo, registrando las excepciones que el jefe formaliza por correo.

**Tiempo objetivo:** < 2h por coordinador por semana (estado estable post-calibración; en ciclos 1-3 con override ≥ 25% será mayor — es objetivo de estado estable, no de ciclo 1, consistente con que los overrides son ORO de calibración). **Dolores atendidos:** D-coord-1..7. **Métrica de éxito principal v1:** §6 tiempo del coordinador.

## 8. Capacidades y Requisitos Funcionales

Las capacidades del sistema se agrupan en **doce áreas**. Cada FR tiene identificador estable (FR-NNN) y se referencia desde el resto del PRD. Cuando una capacidad responde directamente a un dolor de §3, se nombra.

### 8.1 Datos maestros (C-1)

Base operativa que el resto de capacidades consume.

- **FR-001** — Mantener catálogo de **tiendas** con: ID (String), nombre, distrito, **coordinador territorial asignado**, **formato (Express / Estándar)**, **ciclo quincenal asignado** (semana 1-y-3 o 2-y-4), **techo presupuestal semanal MXN**. *El factor de pérdida NO vive en el catálogo de tienda — vive a nivel **tipo de papel** (ver FR-007 y FR-082 reformulados, decisión D-9 2026-05-26).*
- **FR-007** *(reformulado por D-9, 2026-05-26)* — Mantener **catálogo de tipos de papel** (`tipo_papel`) derivado del atributo `Material` de `Skus_insumos.csv`. Cada tipo de papel agrupa uno o más SKUs de insumo. Es la unidad de configuración del **único `factor_perdida`** (C-9) — la granularidad operativa que el coordinador entiende ("Opalina", "Bond", etc.), no el SKU individual ni la combinación tipo+tamaño. *(Antes era catálogo de categorías tipo_papel+size_papel para el modelo dual; el colapso a factor único por tipo de papel simplifica la granularidad.)*
- **FR-002** — Mantener catálogo de **SKUs de insumo** (lo comprable al proveedor): código, descripción, presentación, contenido (piezas por presentación), costo MXN. Preservar literal `"Matarial y tamaño"` por contrato con el equipo de datos.
- **FR-003** — Mantener **tabla de equivalencia** SKU venta ↔ SKU insumo. Toda venta procesada debe encontrar su equivalencia; sin equivalencia → `EquivalenciaNoDefinidaException` y aborto del ciclo de esa tienda (fail-loud).
- **~~FR-004~~** — *Eliminado.* La rotación de guardia entre coordinadores es operativa y la organizan ellos fuera del sistema; el sistema **no la gobierna ni la consume** (decisión C-O20 / F-12, 2026-05-25).
- **FR-005** — Mantener **datos de coordinadores**: identidad, distrito territorial asignado (lista de tiendas), credenciales, rol. **El sistema no mantiene calendario de guardia rotativa** (eliminado per C-O20).
- **FR-006** *(nuevo)* — Mantener **catálogo de eventos comerciales** con ventanas de fechas y factor multiplicador estimado por evento. Lista inicial propuesta (pendiente de confirmación con Operaciones, D-5): Hot Sale, Back to School (BTS), Buen Fin, Navidad/Reyes, regreso a oficinas post-vacacional, Día de las Madres / del Padre / del Niño, cierre fiscal (oct-dic). Cada evento alimenta FR-033 con su factor (calibrado por backtesting si no se conoce a priori).

### 8.2 Ingesta y normalización de datos (C-2)

Carga de archivos canónicos en `docs/` al modelo del sistema. Las convenciones (encoding, parsing dual de fechas, separadores, tipos exactos, fail-loud) están en `project-context.md` y no se repiten.

- **FR-010** — Ingestar archivos canónicos del inventario en `docs/` por **descubrimiento por patrón de nombre** (`ALLOC_YYYY_MM_DD.csv`, `TFS_YYYY_MM_DD.csv`, `Entrega-directa-tienda*.csv`). Nuevos archivos sin reconfiguración del sistema.
- **FR-011** — Detectar **encoding** por archivo en orden: UTF-8 BOM → cp1252. Registrar encoding detectado en MDC.
- **FR-012** — Validar invariantes de ingesta documentadas en `project-context.md` (ej. `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE` en ALLOC, `TRAN_CODE ↔ DECODE` en Entrega-directa). Comportamiento fail-loud cuando aplica.
- **FR-013** — Parsear `Cantidad por semana` de `Presupuesto-tiendas.csv` (formato `"$X,XXX"`) y `Semana que debe solicitar` (lenguaje natural → `List<Integer>` ISO). Patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`.
- **FR-014** *(nuevo)* — Ingestar **snapshots periódicos de SIM** (saldo de inventario por tienda × SKU) como input externo unidireccional. Cadencia: diaria, una actualización por la noche (decisión owner 2026-05-25, OQ-117). Formato y dueño del extract: pendientes (OQ-117 parcial). Política fail-loud para snapshot ausente / saldo negativo / inconsistente: OQ-114. Reemplaza al conteo asistido como input de saldo.
- **~~FR-015~~** *(eliminado por decisión owner 2026-05-25)* — La ingesta de histórico de mermas no será posible. Se reemplaza por **configuración manual del coordinador territorial** del factor único de pérdida por tipo de papel (FR-082 reformulado), con default global ≈0.9% (FR-084 reformulado). Esto destraba completamente F-02 (la mitigación de R-1 ya no depende de un histórico inexistente).
- **FR-016** *(nuevo, decisión D-8, 2026-05-26)* — Ingestar **ventas con granularidad semanal**. Hoy el histórico disponible es **mensual**; el motor requiere cadencia semanal (cada semana hay tiendas en ciclo). Fuente objetivo: **Big Data / BigQuery** (extract de ventas por semana × tienda × SKU venta), a confirmar con Fernando (OQ-123). Si la fuente semanal no está disponible a tiempo, fallback documentado: derivar de mensual por prorrateo — con la advertencia de que introduce error en la cadencia que el owner quiere por control. *(BigQuery aquí es **almacén de lectura de datos**, no BigQuery ML — distinción a aclarar contra la regla de `project-context.md`; ver §10.6.)*
- **FR-017** *(nuevo, decisión D-10, 2026-05-26)* — Ingestar **ALLOC y TFS** (asignaciones y transferencias: lo pedido, lo en curso, lo que va a llegar, lo entregado) como input del cálculo del pedido (FR-040) y del dashboard de validación pedido-vs-recibido (FR-067). Fuente objetivo: **RMS / BigQuery** — los coordinadores no acceden RMS; el sistema lo consume vía extract (a confirmar con Fernando, OQ-124). Convive con la ingesta por patrón de `ALLOC_*.csv` / `TFS_*.csv` (FR-010) mientras se define la fuente productiva.

### 8.3 ~~Conteo asistido de bodega (C-3)~~ — *Eliminado, fuera de scope v1*

C-3 (FR-020..025) se elimina de v1 por decisión del owner (C-O2). El jefe sigue capturando inventario en SIM como hoy; el sistema no provee UI de conteo. Justificación y trade-off en §11.12 y §11.16. La calidad sucia de SIM se mitiga vía factor de merma esperada (C-9) y backtesting, no vía conteo asistido. **Reabre en v1.1** si se decide volver al jefe.

### 8.4 Forecast de demanda — cadencia dual (C-4) — Dolor: D-coord-1 (atendido indirectamente D-jefe-3)

Motor de predicción por tienda × SKU. Es el corazón estructural del valor v1: el coordinador deja de calcular sobre intuición y opera sobre sugerencias del motor. **Cadencia explícita: el motor calcula semanalmente** (porque cada semana hay ~113 tiendas en su ciclo de pedido — las de 1y3 vs 2y4 alternadas); **la unidad de pedido individual por tienda es quincenal** (cada tienda recibe un pedido cada 2 semanas).

- **FR-030** — Calcular **demanda esperada de venta semanalmente** para el próximo ciclo de cada tienda en ciclo activo esa semana, por tienda × SKU venta, usando modelo estacional configurable (familia de modelos en Java puro definida en `project-context.md`). La unidad temporal interna del modelo y el output al coordinador es el ciclo quincenal de la tienda.
- **FR-031** — Convertir demanda de venta a **demanda de insumo** aplicando la tabla de equivalencia.
- **FR-032** — **Cuantizar** la demanda de insumo al múltiplo del empaque mínimo (`Contenido` en `Skus_insumos`). Nunca redondear hacia abajo si hay demanda.
- **FR-033** — Aplicar **componente estacional** basado en histórico ≥ 12 meses **y catálogo de eventos comerciales FR-006**. Lista inicial de eventos modelados (D-5, pendiente de confirmación con Operaciones): **Hot Sale, Back to School (BTS), Buen Fin, Navidad/Reyes, regreso a oficinas post-vacacional, Día de las Madres / del Padre / del Niño, cierre fiscal (oct-dic)**. Cada evento con su ventana de fechas y factor multiplicador (calibrado por backtesting cuando no se conozca a priori).
- **FR-034** *(reformulado por D-9)* — Generar **trazo explicable** del cálculo por SKU: histórico de ventas utilizado, ventana, factor estacional + evento comercial activo, **factor único de pérdida aplicado (% de la venta)** con indicación de nivel del que provino (factor por tipo de papel o default global), ajuste por inventario en tránsito (ALLOC/TFS). Visible en hover en UI. *Nota de implementación: la opción técnica concreta para "aplicar factor" (multiplicador sobre output, ajuste de histórico — ver F-05) es decisión de `architecture.md`; afecta R-7 (MAPE acordado) y R-8 (spike forecasting).*
- **FR-035** *(reformulado por D-9, 2026-05-26 — revierte el modelo dual D-4)* — Aplicar **un único `factor_perdida`** (porcentaje de la demanda de venta) resuelto por jerarquía `factor_por_tipo_papel → default_global (≈0.9%)` (configurado en C-9 vía FR-082). El motor lo consume como ajuste porcentual sobre la demanda de venta predicha. El trazo explicable (FR-034) declara el valor aplicado, su naturaleza (% de la venta), y el nivel del que provino. La opción técnica concreta de cómo se aplica es decisión de `architecture.md` (F-05).

### 8.5 Sugerencia de pedido (C-5) — Dolor: D-coord-1, D-coord-3 (atendido D-jefe-4 indirectamente)

Calcula la cantidad final a pedir considerando saldo SIM (vía snapshot) y tránsito. **El actor que ve la sugerencia es el coordinador en su bandeja de consolidación**, no el jefe.

- **FR-040** — Para cada tienda × ciclo, calcular **cantidad sugerida por SKU** = `demanda_quincenal_insumo + buffer_seguridad − saldo_SIM_snapshot − ALLOC_pendiente − TFS_entrante`, cuantizada al empaque mínimo. El factor único de pérdida (FR-035) ya está aplicado en `demanda_quincenal_insumo`. **`ALLOC_pendiente` y `TFS_entrante`** se obtienen de FR-017 (lo asignado/en tránsito que ya va a llegar) — el owner enfatizó que el cálculo debe considerar "lo que había pedido, lo que está en curso, lo que va a llegar" (C-O33). **`buffer_seguridad`** es un parámetro configurable por administración (ver §10.5) que representa el inventario de protección frente a variabilidad inter-ciclo no capturada por el factor; valor inicial sugerido = 1 semana de cobertura de venta promedio, ajustable por tienda × SKU. *(Resuelve F-Done-1 — buffer_seguridad ahora definido.)* *Nota (C-O35): v1 no fuerza la captura de inventario; `saldo_SIM_snapshot` se usa como dato disponible pero la base del cálculo es la venta + ALLOC/TFS, no el conteo físico — basarse en el conteo "siempre deja torcida" la solicitud.*
- **FR-041** — Mostrar al **coordinador en su bandeja de consolidación** la sugerencia por tienda × SKU con su trazo explicable.
- **~~FR-042~~** — *Eliminado.* El jefe no opera el sistema en v1, por lo tanto no aplica override del jefe. El override del coordinador se consolida con FR-064.
- **FR-043** — Para SKUs sin equivalencia definida, **no incluir** en sugerencia y emitir `EquivalenciaNoDefinidaException` que aborta el ciclo de esa tienda hasta resolverse.

### 8.6 Validación de presupuesto y sugerencia de recorte (C-6) — Dolor: D-coord-3

Garantiza que la solicitud por tienda no exceda su techo individual.

- **FR-050** — Calcular **costo total por tienda** = `Σ (cantidad_sugerida × costo_SKU)` en MXN.
- **FR-051** — Si `costo_total > presupuesto_disponible`, **sugerir recorte automático** priorizando reducir los SKUs con **menor riesgo de quiebre**, donde riesgo de quiebre se define operacionalmente como: `(cobertura_en_semanas_post_recorte ≥ 2 semanas)` con priorización descendente — primero se recortan los SKUs cuya cobertura post-recorte sigue siendo más alta. *(Resuelve F-Done-2 — criterio de riesgo de quiebre operable.)*
- **FR-052** — Mostrar al coordinador, en la consolidación, las tiendas que requieren recorte con la sugerencia aplicada y permitir **aceptar / modificar / rechazar** con razón.
- **FR-053** — Persistir cada decisión de recorte con timestamp, coordinador ejecutor, razón.
- **FR-054** *(nuevo, decisión C-O17)* — **Descontar compras esporádicas del periodo del techo presupuestal disponible.** Calcular `presupuesto_disponible = techo_semanal_tienda − Σ compras_esporadicas_del_periodo`, leyendo `Entrega-directa-tienda.csv` (categoría TRAN_CODE 20 "Purchases"). Si una tienda tuvo compras esporádicas autorizadas (FR-095 escalamiento → comprador), el presupuesto disponible del ciclo regular queda reducido. Mostrar al coordinador el desglose `techo − consumido_espor → disponible`. Compras esporádicas registradas en el sistema dentro del periodo cuentan independientemente de su origen (excepción mid-cycle vía FR-095 o ingesta histórica vía `Entrega-directa-tienda.csv`).

### 8.7 Concentrado distrital — Bandeja semanal por coordinador (C-7) — Dolor: D-coord-1, D-coord-2, D-coord-7

UI del coordinador para procesar las solicitudes de su distrito que entran cada miércoles. **Bandeja por coordinador territorial** (no rotativa) — cada uno ve solo sus ~38 tiendas en ciclo esa semana.

- **FR-060** — Mostrar a cada coordinador la **bandeja de tiendas de su distrito con ciclo activo esa semana** (~38 tiendas en promedio).
- **FR-061** *(reformulado, decisiones D-6 + D-7 + D-9/C-O31 2026-05-26)* — Por cada tienda, mostrar estado: **solicitud recibida** / **pendiente solicitud** (correo del jefe no llegado — **visible pasivo, no es acción del coord**, solo informativo; perseguir es responsabilidad del jefe per D-coord-7) / **carry-over de semana anterior** (tienda que no envió a tiempo en semana N−1 y se procesa en semana N aunque no esté en su ciclo nominal — ver FR-066) / sugerencia del motor lista / con recorte sugerido / aprobada / exportada. **Sin estado "pendiente captura jefe"** porque el jefe no opera el sistema. **Mecanismo de entrada de la solicitud (resuelto C-O31, 2026-05-26):** el coordinador **no transcribe Excel** — el sistema **genera** la sugerencia del motor por tienda y el coordinador trabaja sobre ella **en pantalla**. El Excel del jefe sigue llegando por correo como contexto/respaldo, pero el flujo del coord es la pantalla del sistema; el Excel del sistema es el **output hacia compras** (FR-110), no el input. Marcar "solicitud recibida" es un flag (manual o derivado de la llegada del correo); v1 NO parsea automáticamente el adjunto del jefe.
- **~~FR-062~~** — *Eliminado.* El coord no persigue tiendas atrasadas (D-coord-7). La visibilidad pasiva de FR-061 cumple la función informativa sin requerir acción.
- **FR-063** — Mostrar **agregados del distrito en la semana**: presupuesto consumido total (con desglose `techo − compras_esporádicas`), número de tiendas con solicitud pendiente, número de tiendas en carry-over, SKUs con riesgo concentrado, costo estimado del consolidado.
- **FR-064** — Permitir al coordinador **modificar cantidades** (override) en cualquier tienda con razón estructurada obligatoria (lista predefinida + texto libre opcional); modificaciones en bitácora. Consolida lo que era FR-042. En ciclos 1-3 estos overrides son input crítico de calibración (ver §6 D-1 híbrido).
- **FR-065** — Permitir **aprobar el consolidado** del distrito y dispararlo al export downstream (C-12).
- **FR-066** *(nuevo, decisión D-6 = modelar carry-over)* — **Regla automática de carry-over de pedidos tardíos.** Si una tienda en ciclo nominal de la semana N no envió solicitud antes del corte del miércoles, el sistema:
  1. Marca la tienda como **carry-over** en la bandeja de la semana N (estado visible pasivo, no procesable en N).
  2. **Reinscribe automáticamente la tienda al ciclo de la semana N+1** aunque su ciclo nominal sea N+2, manteniendo su ciclo nominal original para semanas posteriores (un único salto).
  3. Genera **entrada en bitácora** del carry-over con timestamp, tienda, semana origen, semana destino.
  4. **Recalcula presupuesto disponible para la semana N+1.** **Regla (resuelve finding high del Validate 2026-05-26):** el techo presupuestal es **semanal y rígido por tienda — NO acumula**. La solicitud arrastrada se valida contra el **techo de la semana destino N+1** (el techo no consumido de la semana N **no se traslada ni se suma**), descontando además las compras esporádicas del periodo (FR-054). Es decir: `presupuesto_disponible_N+1 = techo_semanal_N+1 − compras_esporádicas_del_periodo`. *[CONFIRMAR CON OWNER: regla elegida por consistencia con "techo semanal rígido, sin pool"; validar que el negocio no espera acumulación del techo no usado.]*
  5. Surface el carry-over al coordinador territorial en su bandeja de la semana N+1 con etiqueta visual.
  
  La regla aplica solo a tiendas individuales que se atrasan; no aplica masivamente.
- **FR-067** *(nuevo, decisión C-O37, 2026-05-26)* — **Dashboard de validación pedido-vs-recibido.** Por proveedor → tienda → SKU, mostrar: necesidad sugerida, **últimos pedidos** (cantidad asignada/pedida), y **recibos** (cantidad recibida real), con la **brecha** resaltada. Resuelve el caso operativo que el owner nombró: *"pido 10 paquetes de opalina y solo me llegan 2; he pedido 50 y recibí 20"*. Se alimenta de ALLOC/TFS (FR-017): `QTY_ALLOCATED` vs `QTY_RECEIVED`. Permite al coordinador detectar tiendas con déficit crónico de surtido que generan necesidad recurrente no atribuible a venta.

### 8.8 Histórico y análisis de transferencias TFS (C-8) — Dolor: D-coord-4

**v1 NO sugiere ni decide TFS.** Las transferencias se gestionan en otro proceso/programa. v1 solo provee histórico semanal y visualización cruzada para informar las decisiones que el coordinador toma fuera del sistema, y alimenta el motor de forecast con ese histórico.

- **~~FR-070~~** — *Eliminado.* No hay detección de pares exceso/déficit en v1.
- **~~FR-071~~** — *Eliminado.* No hay sugerencia de cantidad transferible en v1.
- **~~FR-072~~** — *Eliminado.* No hay sugerencia para aceptar/modificar/descartar — la TFS se decide fuera del sistema.
- **FR-073** *(reformulado)* — **Registrar histórico de TFS ejecutadas** ingestando `TFS_*.csv` semanalmente. Este histórico cumple dos funciones: (a) visibilidad para el coordinador (análisis cruzado), (b) input al motor de forecast (C-4) para mejorar la predicción de saldos efectivos.
- **FR-074** *(nuevo)* — **Visualización cruzada del histórico TFS** al coordinador: filtros por tienda origen, tienda destino, SKU, periodo (semana/mes). Tablero exportable a CSV.
- **FR-075** *(nuevo)* — Cuando el coordinador territorial registra una excepción mid-cycle (C-10) y decide gestionar una TFS de emergencia *fuera del sistema*, puede **registrar manualmente la decisión y el resultado** para que quede en bitácora; el saldo del sistema **no se modifica directamente** desde aquí (la TFS se refleja en el histórico cuando aparezca en `TFS_*.csv` del siguiente extract).

### 8.9 Factor de pérdida y detección estadística residual (C-9) — Dolor: D-merma-1

**Reescritura entera (decisión D-9, 2026-05-26 — REVIERTE el modelo dual D-4):** la pérdida deja de modelarse como dos componentes separados (uso interno + error de operador). El owner determinó que **"al final del día es un gasto, es uno solo"** y que la base de la predicción es **99% ventas, 1% lo demás** — el factor de pérdida es un colchón pequeño sobre la venta, no un modelo elaborado. v1 modela la pérdida como **un único `factor_perdida` configurable**, expresado como **porcentaje de la demanda de venta** (valor inicial sugerido ≈ **0.9%**, calibrable en piloto: 1%, 3%, 5%…).

**Granularidad de configuración (decisión D-9):** un solo factor configurable por **tipo de papel** (`tipo_papel`, derivado del atributo `Material` de `Skus_insumos.csv`). No se desdobla por (tienda × categoría) ni se separa en dos componentes. Conserva mínima estructura — el coordinador entiende "Opalina", "Bond", etc. **Default global ≈ 0.9%** para tipos de papel no configurados explícitamente (FR-084).

**Encuadre de 3 fases (decisión C-O28/C-O30, 2026-05-26):** la **medición precisa de la pérdida por tienda** (ej. "compré 1000, rebajé 900, tienes 10% de merma en esta ubicación") requiere tocar el inventario de rebajas, que es la **Fase 3** del producto. **v1 (Fase 1 — predicción) NO mide merma por tienda**: solo aplica el factor único como colchón sobre el forecast. La detección residual (FR-080) es una capa opcional y secundaria, no el corazón de v1. Ver §11.16 y §11.17.

- **FR-080** *(reformulado por D-9)* — Capa **opcional y secundaria** de detección de patrones sospechosos por tienda × SKU: `residual = consumo_real_observado − venta_esperada − perdida_aplicada`. Si `residual` excede banda estadística esperada (umbral inicial sugerido ≥ 2σ, calibrable — OQ-123), generar candidato para revisión humana. **No es el mecanismo principal de v1** — el factor único cubre el caso base; la detección residual surface lo atípico para registro. **Sin conteo asistido**; opera sobre venta histórica + factor configurado. *(La medición precisa de pérdida por tienda es Fase 3, fuera de v1.)*
- **FR-081** *(reformulado por D-9)* — Generar **candidatos residuales** clasificados por severidad (divergencia vs banda esperada) para surfacing en la bandeja del coordinador territorial. **Sin atribución entre categorías** (ya no hay dos componentes — OQ-120 cerrada N/A). El candidato es simplemente "consumo no explicado por venta + factor de pérdida configurado".
- **FR-082** *(reformulado por D-9)* — Permitir al coordinador territorial:
  1. **Configurar el `factor_perdida` (% de la venta) por tipo de papel** — gobierna el default que alimenta C-4 vía FR-035. Valor inicial global ≈ 0.9%.
  2. **Capturar / modificar / descartar** cada candidato residual de FR-081 (SKU, cantidad estimada, motivo, evidencia opcional) para registro histórico.
  3. Toda configuración del factor y todo registro de candidato con su propio histórico de cambios trazado (C-11).
- **FR-083** — Acumular registro de candidatos confirmados para revisión periódica con dirección. Exportable. *(El desglose fino de causas de pérdida es de Fase 3.)*
- **FR-084** *(reformulado por D-9)* — **Default global ≈ 0.9% de la venta** para cualquier tipo de papel no configurado explícitamente. El coordinador territorial ajusta el factor por tipo de papel **desde el aplicativo, a discreción**, conforme observe la necesidad operativa (sin obligación pre-go-live). Calibrable en piloto.
- **FR-085** *(reformulado por D-9)* — El **factor único de pérdida alimenta el motor de forecast** (C-4 vía FR-035) como un ajuste porcentual sobre la demanda de venta, resuelto por jerarquía `factor_por_tipo_papel → default_global`. Cambios en el factor son trazados en bitácora con timestamp + coordinador + tipo de papel afectado + valor anterior + valor nuevo + razón estructurada.

### 8.10 Registro de excepción mid-cycle por el coordinador (C-10) — Dolor: D-exc-3, D-exc-4, D-exc-5 (parciales)

**Decisión D-2 + C-O12/C-O14/C-O19 — conservar canal coordinador → comprador con canal oficial único = correo electrónico.** El canal jefe → coordinador sigue **fuera del sistema** pero **se formaliza exclusivamente por correo electrónico**; WhatsApp y llamadas existen como notificación informal previa pero no entran al flujo. El coordinador territorial **entra a la bandeja del sistema para registrar** la excepción que recibió por correo, decidir con contexto, y escalar al comprador con trazo.

- **~~FR-090~~** — *Eliminado.* El jefe no opera el sistema en v1. La alerta inicial llega exclusivamente por correo electrónico fuera del sistema.
- **FR-091** *(reformulado por C-O12 + OQ-115 cerrada con opción c)* — El **coordinador territorial** registra manualmente una excepción mid-cycle en su bandeja con: tienda, SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado, **categoría de disparador (temporalidad / uso interno / otra — derivada de D-exc-2)**, **origen del aviso = correo electrónico (campo fijo, canal único per C-O12)**. La excepción **se asigna automáticamente al territorio del coordinador** que la registra (no hay routing — él ya es el destinatario por construcción). **Eliminado "inferencia propia" de los orígenes** — no hay flujo de detección automática en v1 (OQ-115 cerrada con opción c, resuelve F-17).
- **FR-092** — Mostrar al coordinador en su bandeja **contexto agregado** para la excepción: saldo SIM más reciente de la tienda (fuente única de validación de inventario per C-O16), ALLOC pendiente, TFS en tránsito y histórico TFS reciente para el SKU (C-8). **Sin sugerencia automática de TFS de emergencia** — la decisión y la ejecución viven fuera del sistema.
- **FR-093** *(reformulado)* — Permitir al coordinador **decidir y registrar** la respuesta: gestionar TFS de emergencia fuera del sistema (registrar solo decisión, no ejecución — ver FR-075), escalamiento a comprador para compra directa, rechazo (con razón), espera al ciclo normal.
- **FR-094** — Cuando se escala a comprador, **notificarlo** por canal acordado (correo, integración) con contexto + decisión del coordinador + datos de la tienda y el SKU.
- **FR-095** *(acoplado con FR-054)* — Reflejar en el saldo de planeación cualquier **compra extraordinaria autorizada** (cuando el comprador confirme) **y descontarla del techo presupuestal de la tienda** para el ciclo regular siguiente (vía FR-054). El siguiente ciclo parte de la realidad — tanto en saldo de inventario como en presupuesto disponible. SIM se sincroniza por su propio flujo (no por este sistema).

### 8.11 Trazabilidad y auditoría (C-11) — Dolor: D-coord-5, D-exc-4

Bitácora universal de cambios y decisiones.

- **FR-100** — Para cada cantidad sugerida modificada (override, recorte, TFS aceptada/rechazada, alerta atendida), **registrar entrada en bitácora** con: timestamp, actor, valor original, valor nuevo, razón estructurada.
- **FR-101** — Mostrar en UI (hover sobre fila modificada) la **cadena completa**: `original modelo X → Jefe Carlos 07:48: Y → Coord. Marco 09:12: aprobado`.
- **FR-102** — Panel lateral **"Cambios de hoy"** exportable a CSV.
- **FR-103** — MDC en logs con `tiendaId`, `skuId`, `cicloId`, `usuarioId` (heredado de `project-context.md`).
- **FR-104** — Bitácora **inmutable**. Para corregir un error se crea nueva entrada que referencia la anterior.

### 8.12 Export downstream al comprador (C-12)

Genera el archivo de handoff al flujo del comprador.

- **FR-110** — Generar archivo consolidado equivalente a `SolicitusDeInsumosTodos.xlsx` aprobado por el coordinador, **como output hacia compras** (el coordinador trabaja en pantalla; el Excel es el handoff, FR-061). **Formato compatible** con la estructura del archivo de referencia en `docs/SolicitusDeInsumosTodos.xlsx` (estructura validada por owner 2026-05-25, R-4/OQ-102 cerrada). El archivo de referencia se usa como contrato literal del export — toda divergencia downstream con el flujo del comprador (Hugo) constituye defecto bloqueante. **Layout del reporte de necesidad de demanda (OQ-126, decisión D-11 = abierta):** el owner evalúa con los coordinadores dos vistas candidatas — (A) por SKU → tiendas + cantidades, o (B) por proveedor → tienda → necesidades. La estructura del export al comprador (FR-110) es la del archivo de referencia; la vista de trabajo/consulta del coordinador puede ofrecer una o ambas, a definir con los coordinadores.
- **FR-111** — Permitir **exportar bajo demanda** y **enviar por correo** al comprador con CC configurable.
- **FR-112** — Registrar cada export en bitácora: timestamp, coordinador ejecutor, hash del archivo, destinatarios.

---

## 9. Requisitos No-Funcionales transversales

Las convenciones de calidad, testing, stack y datos están en `_producto/project-context.md` y se asumen como base. Aquí solo se nombran los NFR específicos del producto que afectan el diseño y la operación.

### 9.1 Escala y concurrencia (NFR-Escala)

- **NFR-E-1** — Soportar **226 tiendas activas en el modelo de datos**. **Usuarios humanos concurrentes: hasta 3 coordinadores + 1 administrador** (no hay capturas paralelas de jefes en v1).
- **~~NFR-E-2~~** — *Eliminado.* No hay 113 jefes capturando en paralelo — el jefe no opera el sistema en v1.
- **NFR-E-3** *(cuantificada 2026-05-25)* — El motor de forecast batch debe **procesar el universo completo** en una ventana razonable que no bloquee la operación del miércoles. **Orden de magnitud confirmado** (inspección de `docs/Cruce-de-skus-venta-insumo.xlsx` + `docs/Skus_insumos.csv`): **~38 SKUs de venta** y **~10 SKUs de insumo** por tienda. Universo del motor = 226 tiendas × 38 SKUs venta = **~8,600 series temporales** (Holt-Winters / ARIMA) → cuantización final a **~2,260 cantidades** (226 × 10 insumos). Este universo es trivial computacionalmente: <5 min en una sola JVM, ampliamente dentro de la ventana operativa. **Cerrado como bloqueo del Reviewer Gate.** La elección técnica concreta de modelo y paralelización queda en `architecture.md`.
- **NFR-E-4** — Cada coordinador debe poder abrir su bandeja distrital con **~38 tiendas** y verla cargada en menos de 3 segundos.

### 9.2 Performance percibida (NFR-Perf)

- **NFR-P-1** — Edición numérica en grilla del coordinador dispara **recálculo optimista en cliente** (RxJS `debounceTime(150)`) con confirmación servidor; recalcular fila, acumulado por proveedor y restante de presupuesto en **<100 ms** (heredado de `project-context.md`).
- **NFR-P-2** — Autosave silencioso con indicador tipo Google Docs ("Guardado hace 3 seg"). El usuario nunca pierde trabajo.

### 9.3 Disponibilidad (NFR-Disp)

- **NFR-D-1** — Disponibilidad **≥ 99% durante ventanas operativas** (lunes a viernes 06:00-22:00 hora CDMX). Ventanas fuera de operación tolerables para mantenimiento.
- **NFR-D-2** — En caso de caída, **las decisiones del coordinador en la bandeja distrital y territorial deben poder reanudarse** sin pérdida del trabajo previo (overrides aplicados, recortes aceptados, excepciones registradas, configuraciones de factores). El estado persiste server-side.

### 9.4 Auditabilidad y trazabilidad (NFR-Audit)

- **NFR-A-1** — Toda decisión de negocio (override, recorte, TFS aceptada, alerta atendida, export) queda **trazada** con timestamp, actor identificado, valor original, valor nuevo, razón estructurada (C-11).
- **NFR-A-2** — Logs server-side con **MDC poblado** con `tiendaId`, `skuId`, `cicloId`, `usuarioId` para toda operación de negocio (heredado de `project-context.md`).
- **NFR-A-3** — Bitácora **inmutable**. Correcciones se hacen como nuevas entradas que referencian la anterior, nunca modificando la original.

### 9.5 Seguridad y control de acceso (NFR-Sec)

- **NFR-S-1** — Autenticación de usuarios con **identidad individual** (no compartida). Patrón sugerido (Auth0 con JWT) tomado de la arquitectura de referencia Tomaturno; decisión final en `architecture.md`.
- **NFR-S-2** — Roles operativos v1: **Coordinador** (acceso a sus tiendas territoriales para consolidación semanal regular, excepciones mid-cycle y configuración del factor único de pérdida por tipo de papel) y **Administrador** (configura datos maestros, catálogo de eventos comerciales, umbrales de detección residual, lista de razones estructuradas, parámetro `buffer_seguridad` por tienda × SKU). **Rol "Jefe de servicentro" eliminado de v1** — el jefe no opera el sistema. **Override aplica también al Administrador** cuando opere en modo coordinador (resuelve F-16).
- **NFR-S-3** — **MFA obligatorio** para roles administrativos.
- **NFR-S-4** — Datos sensibles (presupuestos, costos) **no se exponen** a roles que no los necesitan operativamente.

### 9.6 Calidad del dato como pilar (NFR-Datos)

Este NFR es estructural — el sistema solo es tan bueno como el conteo.

- **NFR-DAT-1** — **Fail-loud** ante datos faltantes o ambiguos: equivalencia SKU no definida, presupuesto no configurado, encoding ambiguo, periodicidad presupuestal no reconocida, snapshot SIM ausente o con saldo inconsistente → excepciones checked específicas que abortan el ciclo de esa tienda (heredado de `project-context.md`). **Política específica para SIM sucio en OQ-114.**
- **~~NFR-DAT-2~~** — *Eliminado.* No hay conteo asistido en v1; NFR perdió razón de ser.
- **NFR-DAT-3** — Encoding de archivos detectado y **registrado en logs** por archivo procesado, incluyendo el snapshot SIM (FR-014).
- **NFR-DAT-4** *(cerrada cadencia 2026-05-25)* — **SLA de frescura del snapshot SIM** (acoplado con R-11 y OQ-117): el extract de SIM se ejecuta **diariamente por la noche** (no es posible en línea, decisión owner 2026-05-25). El sistema debe rechazar operar sobre snapshot SIM con antigüedad mayor a **36 horas** (ventana de 24h del ciclo + 12h de buffer para retraso del extract) y notificar al administrador. Formato y dueño operativo del extract: pendientes en OQ-117 parcialmente cerrada.

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
- **NFR-T-2** — Golden datasets versionados en `src/test/resources/fixtures/golden/vN/` con `EXPECTED_OUTPUT.csv` firmado por **Hugo** (comprador piloto identificado 2026-05-25) + `DECISION_LOG.md`. Inmutable una vez shipped.
- **NFR-T-3** — Property-based testing con jqwik sobre cuantización y presupuesto. Invariantes obligatorias enunciadas en `project-context.md`.
- **NFR-T-4** — Contract testing Pact / Spring Cloud Contract entre Spring Boot y Angular. Contrato YAML antes de implementar.
- **NFR-T-5** — Backtesting suite sobre histórico real para acordar MAPE/WAPE con Finanzas.
- **NFR-T-6** — Mutation testing con PIT sobre paquete `forecasting.*`. Threshold ≥ 80% mutantes eliminados antes de mergear a `main`.

### 9.11 Internacionalización (NFR-i18n)

- **NFR-I-1** — V1 **solo es-MX**. No se diseña con i18n diferida pero tampoco se cierra la puerta — strings de UI vivirán en archivo de recursos para evitar reescritura si en el futuro se internacionaliza.
- **NFR-I-2** — Identificadores de código en **inglés**; comentarios y documentación de negocio en **español** (alineado con `project-context.md`).

### 9.12 Observabilidad (NFR-Obs)

- **NFR-O-1** — Métricas de operación expuestas: tiempo de consolidación por coordinador (semanal), número de excepciones mid-cycle registradas por semana, tasa de override por tienda (desglosada por ciclos 1-3 vs ciclo 4+), número de carry-overs por semana, latencia del motor de forecast, errores de ingesta por archivo, frescura del snapshot SIM.
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

### 10.4 SIM — decisión cerrada: modelo (C) Coexistencia con lectura unidireccional

**SIM** es el sistema interno donde los jefes de servicentro capturan altas y bajas de inventario en bodega de forma manual. Los coordinadores tienen acceso de lectura para ver ALLOC y TFS.

**Decisión del owner (2026-05-24, OQ-103 cerrada):** **modelo (C) Coexistencia con lectura unidireccional**. El sistema nuevo NO escribe a SIM. SIM es fuente **unidireccional** (lectura de snapshots periódicos vía CSV extract). **SIM sigue siendo la fuente única de validación de inventario por tienda** (decisión C-O16, 2026-05-25 — refuerza que el sistema nuevo NO genera su propia visión de inventario, solo lee snapshots). La verdad operativa del coordinador (planeación, decisiones, bitácora) vive en el sistema nuevo. La verdad contable de inventario sigue en SIM. **Divergencia aceptada como precondición**; monitoreada por R-11.

**Objeción documentada de Winston (Architect, Party Mode 2026-05-24):** *"(C) coexistencia con doble captura es garantía de divergencia — dos sistemas, dos verdades, ningún ganador. Es el peor de los tres en producción aunque parezca el más seguro políticamente."*

**Honestidad epistémica (post-F-03, 2026-05-25):** la justificación original del owner replanteó el riesgo de Winston ("doble UI del jefe") como un riesgo distinto ("divergencia entre SIM contable y plan operativo"), pero **R-11 reconoce textualmente que la objeción de Winston se materializa precisamente en este nuevo riesgo**. Es decir, no es un riesgo distinto con nombre nuevo — es la misma objeción de Winston con etiqueta operacional. Las tres mitigaciones documentadas en R-11 son **detección + escalamiento humano** (SLA de frescura, contra-métrica de divergencia, revisión trimestral) — **ninguna es engineered mitigation**. Es decir: el modelo (C) no elimina la divergencia que Winston nombró, la **vigila**. Eso puede ser trade-off razonable, pero el PRD lo dice así de directo: la divergencia es estructural, y v1 la detecta antes de que sea grande, no la previene.

**Justificación del owner Jonathan:** el jefe NO captura en el sistema nuevo (clarificación C-O7 del change signal), por lo tanto **"doble captura del jefe" no aplica como riesgo** — el jefe sigue capturando UNA SOLA VEZ en SIM como hoy. Trade-off aceptado: scope manejable v1 a cambio de no resolver el problema epistémico de fondo, mitigando parcialmente vía el factor único de pérdida (concesión central declarada en §11.16, R-1, R-11).

**Refuerzo 2026-05-26 (C-O35):** v1 **no fuerza la captura ni la baja de inventario** en tiendas — *"es una fiesta complicada"* y *"basarse en el inventario actual siempre nos deja torcidos"*. Por eso la base del forecast es la **venta (~99%)**, no el conteo SIM, y el ajuste del pedido usa **ALLOC/TFS** (lo asignado/en tránsito) como cross-check (FR-017, FR-040). SIM sigue siendo input de saldo disponible, pero deja de ser el ancla del cálculo de demanda. Esto reduce la severidad de R-1.

**Modelos (A) y (B) rechazados** — razonamiento completo en `addendum.md` apéndice "Opciones SIM consideradas".

**Información operacional pendiente (no bloquea la decisión, sí condiciona el diseño):**

- Frecuencia, formato y dueño del extract SIM → snapshot del sistema (OQ-117).
- Política fail-loud sobre snapshot ausente o con saldo inconsistente (OQ-114).
- ¿El sistema publica el pedido aprobado a SIM como ALLOC esperado o eso lo hace el flujo actual de SIM? (OQ-116).

### 10.5 Datos maestros adicionales requeridos por el sistema

Más allá de lo que viene en `docs/`, el sistema necesita gobernar:

- **Mapeo Tienda → Coordinador territorial** (para C-10 contexto de excepciones y para bandeja distrital C-7).
- **Catálogo de tipos de papel (`tipo_papel`)** derivado de `Skus_insumos.csv` atributo `Material` (FR-007 reformulado). Cada tipo de papel agrupa uno o más SKUs de insumo.
- **Factor único de pérdida (`factor_perdida`, % de la venta) por tipo de papel** — gobernado por coordinador territorial vía FR-082; default global ≈0.9% si no configurado (FR-084); alimenta el motor vía FR-035 con resolución jerárquica `factor_por_tipo_papel → default_global`. *(Decisión D-9 2026-05-26 — revierte el modelo dual de dos factores por categoría tipo_papel+size_papel con override por tienda.)*
- **`buffer_seguridad` por tienda × SKU** (gobernado por administración; valor inicial sugerido = 1 semana de cobertura de venta promedio; consumido por FR-040).
- **Umbrales configurables** para C-9 detección estadística residual (qué desviación residual considerar "sospechosa" más allá del factor aplicado — umbral inicial ≥ 2σ, OQ-123).
- **Catálogo de eventos comerciales** con ventanas de fechas y factor multiplicador (FR-006); consumido por FR-033.
- **Plazos del ciclo de consolidación** del coordinador — semana ISO + día (miércoles por defecto). Plazos de captura del jefe NO aplican porque el jefe no captura en el sistema.
- **Lista de razones estructuradas** para overrides del coordinador, recortes, excepciones mid-cycle (configurable por admin).

**Eliminado del listado de datos maestros gobernados:**
- Calendario informativo de rotación de guardia (decisión C-O20, F-12). El sistema NO lo gobierna NI lo consume.
- Ingesta de histórico de mermas (decisión owner 2026-05-25, FR-015 eliminado). El factor único de pérdida se configura 100% manualmente por el coord; no hay seed desde dato histórico ingestado.
- Catálogo de categorías tipo_papel+size_papel y los dos factores separados con override por tienda (decisión D-9 2026-05-26 — colapsados a un único factor por tipo de papel).

### 10.6 Integraciones externas

- **SIM** — **lectura unidireccional de snapshots** (CSV extract). Cadencia diaria nocturna confirmada (decisión owner 2026-05-25, OQ-117). Formato y dueño operativo del extract: pendientes en arquitectura. Decisión modelo (C) cerrada en §10.4. No hay escritura del sistema hacia SIM en v1.
- **Big Data / BigQuery — ventas semanales** *(nuevo, D-8, 2026-05-26)* — fuente objetivo para ventas con granularidad semanal (FR-016), hoy solo disponibles a nivel mensual. A confirmar disponibilidad/formato con Fernando (OQ-123).
- **RMS / BigQuery — ALLOC y TFS** *(nuevo, D-10, 2026-05-26)* — fuente objetivo para asignaciones y transferencias (lo pedido, en curso, por llegar, entregado) que alimentan FR-017/FR-040/FR-067. Los coordinadores no acceden RMS; el sistema lo consume vía extract. A confirmar con Fernando (OQ-124).
- ⚠️ **Distinción a aclarar (no bloqueante):** `project-context.md` prohíbe **dependencias gestionadas de GCP (Vertex AI, BigQuery ML)** sin RFC formal. Las fuentes anteriores usan **BigQuery como almacén de lectura de datos**, no BigQuery ML — uso distinto del prohibido. Documentar la distinción y abrir RFC ligero si arquitectura lo considera necesario.
- *(eliminado)* — Histórico de mermas como input externo: no se podrá ingestar (decisión owner 2026-05-25). El factor único de pérdida (C-9) se configura 100% manualmente por el coord.
- **Correo (SMTP) o servicio de email** — para notificar al comprador con el export (FR-094, FR-110). Patrón sugerido (SendGrid) tomado de arquitectura Tomaturno; decisión final en `architecture.md`.
- **Auth0 (probable)** — autenticación con MFA para roles administrativos (NFR-S-1, NFR-S-3). Patrón Tomaturno; decisión final en arquitectura.
- **Cloud Storage** — para evidencias adjuntas en captura de mermas residuales (FR-082) y para hash/almacenamiento de exports históricos (FR-112).

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

Tras Discovery se confirmó que **los presupuestos y la demanda mantienen los pedidos siempre debajo del techo físico** de la bodega — no es problema real. **No se modela** capacidad física como atributo de Tienda. Casos raros de saturación se resuelven con **override del coordinador (FR-064)** y/o ajuste manual del techo presupuestal por administración. *(FR-042 fue eliminado en el rewrite 2026-05-24 — el jefe no opera el sistema, por lo tanto su override original ya no existe; se consolida en FR-064.)*

### 11.6 Aprendizaje automático del override del coordinador

El sistema **registra** los overrides con razón estructurada (FR-100), pero **no aprende automáticamente** a ajustarse a ellos en v1. La iteración del motor ante patrones de override sistemáticos es trabajo de calibración manual del equipo. **Posible v2.**

### 11.7 Aplicación móvil nativa

V1 es **web responsive** que funciona en computadora del coordinador. **No hay app nativa** iOS/Android. **Posible v2** si la operación de campo lo demanda.

### 11.8 Reportería BI / ejecutiva fuera del dashboard operativo

El dashboard operativo del coordinador (FR-063) y del administrador (NFR-O-2) son lo que entrega v1. **Reportería ejecutiva, exportes para Power BI / Looker, integraciones con DWH** son trabajo posterior.

### 11.9 Internacionalización fuera de es-MX

V1 **solo es-MX** (NFR-I-1). No se diseña con i18n diferida pero se preserva separación de strings de UI para no cerrar la puerta.

### 11.10 Capacitación, change management y comunicación oficial

El sistema **provee la herramienta**; la capacitación de los coordinadores y el anuncio oficial son **responsabilidad de la célula de UX / Eficiencia operativa** y del área operativa. Crítico para la adopción, pero fuera del producto. **Identificado como riesgo (R-2).**

### 11.11 Captura del jefe de servicentro en el sistema nuevo

El jefe **sigue capturando altas y bajas en SIM como hoy** y sigue enviando su Excel de solicitud por correo al coordinador. **v1 no provee UI/UX para el jefe.** Reabre en v1.1 si se decide volver al jefe.

### 11.12 Conteo asistido de bodega

La captura del conteo físico sigue siendo manual en SIM como hoy. **v1 no aporta UI dedicada al conteo.** La calidad sucia se mitiga vía factor de merma esperada (C-9) y backtesting.

### 11.13 Decisión y ejecución de transferencias TFS

El sistema **solo registra histórico desde `TFS_*.csv`** y lo visualiza para informar decisiones. **La decisión y la ejecución de TFS viven en otro proceso/programa** fuera de v1.

### 11.14 Rotación de guardia entre coordinadores

**El sistema no la gobierna ni la consume.** Los coordinadores la organizan 100% operativamente fuera del sistema. *(Eliminada por C-O20 / F-12 la cláusula opcional de FR-005 que permitía consumirla como calendario informativo — quedaba ambivalente entre out-of-scope y FR opcional, ahora es out-of-scope explícito.)*

### 11.15 Canal estructurado de alertas mid-cycle disparadas por el jefe

**El sistema solo procesa excepciones formalizadas por correo electrónico.** WhatsApp y llamadas existen como notificación informal previa entre jefe y coordinador, pero **no son canal de entrada al sistema** y no inician atención formal del coordinador hasta que el jefe formaliza por correo (decisión C-O12/C-O14/C-O19, 2026-05-25). El sistema no provee bandeja de entrada del jefe ni canal estructurado en línea jefe→coordinador. Lo que sí estructura el sistema es la decisión y el escalamiento del coordinador hacia el comprador (C-10 reformulada).

### 11.16 Saneamiento del dato de inventario en SIM

**v1 NO sanea SIM.** La calidad del dato sucio se asume como precondición. Tras la decisión 2026-05-26 (C-O35), v1 además **no se ancla al conteo SIM** para calcular demanda — la base es la venta (~99%) y el ajuste usa ALLOC/TFS. La pérdida se mitiga vía el **factor único configurable por tipo de papel** (≈0.9%, FR-082) y backtesting, no vía conteo asistido. **Esta es la concesión epistémica central de v1** (diagnosticada por Dr. Quinn en Party Mode 2026-05-24 y firmada conscientemente por el owner): v1 automatiza el efecto, no la causa raíz. El default global del factor cubre el caso base desde el go-live; el coordinador lo afina por tipo de papel a discreción. La detección residual (FR-080) y los overrides del coord en bandeja (FR-064) son los amortiguadores secundarios. Documentado también en R-1.

### 11.17 Medición precisa de merma por tienda (Fase 3), inventario forzado y solicitud 100% automática

**Encuadre de 3 fases del owner (C-O28/C-O30, 2026-05-26):** el producto vive en tres fases — **(1) Predicción**, **(2) Inventario/solicitud**, **(3) Rebajas/merma precisa**. **v1 es la Fase 1** (predicción) más el soporte a la consolidación de la solicitud. Quedan **fuera de v1**:

- **Medición precisa de la pérdida por tienda** ("compré 1000, rebajé 900 según lo que vendiste → 10% de merma en esta ubicación; cada tienda es distinta"). Requiere tocar el inventario de rebajas — es **Fase 3**. v1 solo aplica el factor único como colchón.
- **Forzar la captura o baja de inventario en las tiendas.** El owner lo descarta explícitamente para v1 (*"es una fiesta complicada"*). v1 se basa en ventas + ALLOC/TFS.
- **Solicitud 100% automática sin intervención humana ni correo** (la visión plena de Fase 2: *"que se genere en automático, sin que pasen por esos [coordinadores] ni que alguien mande un correo"*). En v1 el coordinador sí interviene (revisa y aprueba en pantalla — decisión D-12). La automatización plena es fase futura.
- **Alerta proactiva de quiebre por tienda** ("la tienda X se está quedando sin el producto Y"). Hoy no existe el mecanismo; el único que sabe es la operación de la tienda (C-O38). Futuro.

## 12. Riesgos

Riesgos materiales identificados en Discovery. Cada uno con dueño tentativo y mitigación propuesta. Final en `architecture.md` y plan de pilotaje.

### R-1 — Calidad sucia del dato en SIM contamina el sistema nuevo *(severidad: media — reducida 2026-05-26)*

- **Descripción:** SIM hoy tiene saldos imprecisos porque el conteo de los jefes se hace bajo presión. **Severidad reducida tras la decisión 2026-05-26 (C-O35):** v1 **deja de anclar el cálculo de demanda al conteo SIM** — la base es la venta (~99%) y el ajuste del pedido usa ALLOC/TFS (lo asignado/en tránsito) como cross-check. El conteo SIM ya no es el insumo crítico, así que su suciedad contamina mucho menos. Sigue siendo la concesión epistémica central (§11.16), pero el daño potencial baja.
- **Mitigación:**
  1. **Forecast sales-driven (~99% ventas)** — el cálculo no descansa en el conteo físico sucio sino en el histórico de ventas, que es dato más confiable.
  2. **Factor único de pérdida configurable por tipo de papel** (C-9 reformulada, FR-082, FR-084): default global ≈0.9% desde el go-live (cubre el caso base sin esperar configuración), afinable por el coord. *(Decisión D-9 2026-05-26 — revierte el modelo dual; la mitigación se simplifica y deja de depender de configurar muchas categorías antes del go-live.)*
  3. **ALLOC/TFS como cross-check** (FR-017, FR-067 dashboard pedido-vs-recibido) detectan déficit de surtido que de otro modo se confundiría con pérdida.
  4. **Backtesting sobre histórico real** (`BacktestingSuite.java`, NFR-T-5) cuantifica el impacto en MAPE antes del piloto — acoplado con R-9.
  5. **Monitoreo continuo de divergencia** predicción vs venta real (contra-métrica §6, umbral > 10% mes-sobre-mes) — acoplado con R-11.
- **Nota:** el sub-riesgo "arranca con todo en 0" del modelo dual **desaparece** — ahora hay un default global ≈0.9% activo desde el primer ciclo; no se arranca ciego.
- **Dueño tentativo:** PM + arquitectura + 3 coordinadores territoriales.

### R-2 — Resistencia al cambio operativo *(severidad: media — reducida tras scope cut)*

- **Descripción:** los coordinadores pueden seguir cruzando correos por costumbre y desconfiar de la sugerencia inicial. **Severidad bajó significativamente** porque el scope v1 no toca a 226 jefes — solo a 3 coordinadores. El grupo afectado es pequeño y conocido.
- **Mitigación:** plan formal de capacitación para los 3 coordinadores antes del go-live; iteración rápida en los primeros 3 ciclos con override como ORO de calibración (§6 D-1); revisiones quincenales con los coordinadores en los primeros 60 días para construir confianza.
- **Dueño tentativo:** coordinadores + sponsor (dirección de operaciones).

### R-3 — Falsos positivos de detección residual erosionan confianza *(severidad: baja — reducida tras D-9; detección es capa secundaria)*

- **Descripción:** umbrales mal calibrados generan demasiados candidatos residuales falsos; el coordinador descarta sistemáticamente y deja de mirar → la detección se vuelve folclore. **Severidad reducida tras D-9:** la detección residual (FR-080) deja de ser el corazón del modelo de pérdida — es una **capa secundaria opcional**. El caso base lo cubre el factor único; la detección solo surface lo atípico. Un falso positivo molesta menos porque no es el mecanismo principal.
- **Mitigación:** umbral conservador inicial (≥ 2σ, OQ-123), priorizar precision sobre recall, iteración periódica con datos reales. Sin atribución entre categorías (ya no hay dos — OQ-120 cerrada N/A), el candidato es simplemente "consumo no explicado por venta + factor".
- **Dueño tentativo:** PM + coordinadores piloto.

### R-4 — Incompatibilidad downstream con el comprador *(severidad: media — reducida tras validación de estructura)*

- **Descripción:** el export `SolicitusDeInsumosTodos.xlsx`-compatible se rompe si la estructura cambia futuramente o si el comprador modifica su flujo. Cada incompatibilidad obliga al coordinador a corregir manualmente — pierde el ahorro. **Severidad reducida** tras owner validar que la estructura del archivo de referencia es la correcta (2026-05-25, OQ-102 cerrada).
- **Mitigación:** contrato literal = estructura del archivo de referencia validada; involucrar a **Hugo (comprador piloto identificado)** desde semana 1 del pilotaje; tests automatizados de export contra fixture firmada por Hugo (NFR-T-2 ahora con dueño concreto).
- **Dueño tentativo:** arquitectura + PM + Hugo (comprador).

### R-5 — Dependencia técnica y organizacional de SIM *(severidad: media — reducida tras decisión (C))*

- **Descripción:** con lectura unidireccional (modelo C cerrado en §10.4), **no se necesita API ni dueño técnico de SIM**. Solo se necesita un **extract programado** (CSV) a una ubicación compartida que el sistema lea. La dependencia organizacional se limita a quien produzca el extract.
- **Mitigación:** identificar al dueño operativo del extract SIM (OQ-117); acordar formato, frecuencia y SLA de frescura; mecanismo de aviso si el extract falla o se atrasa.
- **Dueño tentativo:** IT o dueño operativo de SIM + arquitectura.

### R-6 — Tasa de override del coordinador alta inicialmente erosiona la propuesta de valor

- **Descripción:** en los primeros ciclos el motor no tiene calibración fina; el coordinador puede overridear el 60%+ de las sugerencias. Si esto se vuelve patrón, el sistema queda como "Excel sofisticado".
- **Mitigación:** definir **contra-métrica de override < 20% por ciclo** (§6) como gate de éxito; iterar motor rápido en pilotaje; capturar razón estructurada de cada override para análisis.
- **Dueño tentativo:** PM + arquitectura.

### R-7 — Acuerdo MAPE/WAPE con Finanzas se posterga *(severidad: media — reencuadrada 2026-05-26)*

- **Descripción:** sin un número objetivo acordado con Finanzas, no hay vara de medida formal del motor.
- **Reencuadre 2026-05-26 (C-O40):** el owner fijó un **criterio de éxito pragmático que NO depende de un MAPE formal**: la base es la venta YoY y el criterio es *"sin desviaciones importantes vs lo que vendí el año pasado"*, con un lazo de autocorrección pedido-vs-venta-real (*"te mandé 1500, vendiste 1200, te faltan 300 que me pides"*). Esto **deja de ser bloqueo de tesis** — el producto puede arrancar y demostrar valor con el criterio de desviación vs histórico. El número formal con Finanzas sigue siendo deseable pero pasa a tarea paralela, no precondición.
- **Mitigación:** `BacktestingSuite.java` sobre histórico real cuantifica desviación vs YoY desde el primer sprint útil; sesión con Finanzas cuando el owner la agende, ya con datos de soporte.
- **Dueño:** Owner (Jonathan) → Finanzas. PM apoya.

### R-8 — Performance del motor de forecast a escala 226 × 38 SKUs venta *(severidad: baja — reducida tras cuantificar NFR-E-3)*

- **Descripción:** Java puro con Smile/Commons Math puede no cumplir el SLA cuando se corre sobre el universo completo si los modelos seleccionados son pesados. **Severidad reducida tras NFR-E-3 cuantificado** (2026-05-25): universo total = 226 × 38 = ~8,600 series temporales — orden de magnitud trivial para Holt-Winters/ARIMA en una sola JVM.
- **Estatus 2026-05-25:** Owner confirmó que **queda pendiente** el spike de validación — sigue siendo bloqueo del Reviewer Gate pero con expectativa baja de problemas (el universo es trivial).
- **Mitigación:** prototipado temprano del motor en el primer sprint útil; medir tiempos con histórico real; activar el "trigger objetivo para reevaluar Python como microservicio" de `project-context.md` solo si los umbrales se rompen — improbable dado el orden de magnitud.
- **Dueño:** arquitectura.

### R-9 — Pilotaje sin baseline cuantitativo *(severidad: media — baseline de tiempo cerrado, otros pendientes)*

- **Descripción:** si arrancamos pilotaje sin medir el statu quo (tiempo del coordinador por semana, tasa de quiebre, sobre-inventario), no podemos demostrar mejora.
- **Estatus 2026-05-25 (parcial):**
  - ✅ **Tiempo del coordinador baseline cerrado:** 9 horas/semana/coordinador (confirmado por owner; cronometrado de la operación actual). Aplicado en §6 métricas.
  - ⏳ **Tasa de quiebre y sobre-inventario baseline:** siguen pendientes de recolectar en pilotaje.
  - ⏳ **Backtesting sobre histórico real:** sigue pendiente (acoplado con R-8 spike).
- **Mitigación:** **antes del go-live**, recolectar las dos métricas restantes en 3-5 tiendas piloto durante un par de ciclos (recuento de quiebres en mostrador, snapshot de bodega vía SIM y vía verificación física puntual). El tiempo del jefe NO es métrica baseline en v1 — el jefe no opera el sistema y su tiempo es upside de v2.
- **Dueño:** PM + coordinadores piloto.

### R-10 — Express vs Estándar — supuesto de "mismo catálogo" se rompe

- **Descripción:** confirmamos en Discovery que ambos formatos manejan mismo catálogo de servicios, pero si en operación real Express maneja un subset reducido y el motor lo ignora, hay sobre-pedido.
- **Mitigación:** validar el supuesto al inspeccionar `Skus_insumos.csv` y `Presupuesto-tiendas.csv` (sección §10.3); si surge diferencia, modelar como subset por formato.
- **Dueño tentativo:** arquitectura + análisis de datos.

### R-11 — Divergencia silenciosa entre SIM contable y plan operativo del coordinador *(severidad: alta — nuevo riesgo)*

- **Descripción:** con coexistencia (C) cerrada en §10.4, si el snapshot SIM no se actualiza a tiempo, o si la captura manual del jefe en SIM se vuelve más floja (al ver que el sistema nuevo "hace todo"), el sistema planea sobre datos viejos. **La objeción de Winston en Party Mode 2026-05-24** se materializa precisamente aquí: las dos verdades pueden divergir silenciosamente y la consolidación opera sobre la divergencia.
- **Mitigación triple:**
  1. **SLA de frescura de snapshot SIM** (NFR-DAT-4, propuesta inicial 48h máximo de antigüedad). Si se excede, el sistema rechaza operar y notifica al administrador.
  2. **Contra-métrica de divergencia predicción vs venta** (§6) monitorea el sesgo agregado del sistema; si crece sistemáticamente, señal de que SIM se está desviando.
  3. **Revisión trimestral con dueño operativo de SIM** + coordinadores para auditar muestras de tiendas y reconciliar manualmente cuando aplique.
- **Dueño tentativo:** arquitectura + IT + dueño operativo de SIM.

### R-12 — Dependencia de extracción de ventas semanales y ALLOC/TFS desde RMS/BigQuery *(severidad: media — nuevo 2026-05-26)*

- **Descripción:** la decisión de cadencia semanal (D-8) y el cálculo del pedido con ALLOC/TFS (D-10) dependen de extraer datos de **RMS / BigQuery**, sistemas que los coordinadores no tocan y cuya disponibilidad, formato y dueño operativo no están confirmados. Hoy las ventas solo están a nivel mensual. Si la fuente semanal no se materializa, la cadencia semanal que el owner quiere por control se degrada (habría que derivar de mensual, con error).
- **Mitigación:** confirmar con **Fernando** la existencia, formato y SLA de los extracts (OQ-123, OQ-124) antes de comprometer la cadencia semanal en `architecture.md`; fallback documentado de derivar semanal de mensual por prorrateo mientras tanto (FR-016), con la advertencia de error asociada. Aclarar la distinción BigQuery-lectura vs BigQuery ML frente a `project-context.md` (posible RFC ligero).
- **Dueño tentativo:** PM + Fernando + arquitectura.

## 13. Open Questions

Lista consolidada de decisiones abiertas. Cada una tiene contexto, propuesta inicial y dueño tentativo.

### OQ-101 — Frecuencia real de excepciones mid-cycle

Provisional `[ASSUMPTION: 1-3 por semana en la red de 226]`. **Medir post-lanzamiento** con telemetría. No bloquea v1; afecta el diseño visual de prominencia de la bandeja del coordinador territorial. **Dueño:** PM.

### OQ-102 — Estructura exacta de `SolicitusDeInsumosTodos.xlsx` — **CERRADA (2026-05-25)**

**Decisión del owner:** la estructura del archivo de referencia `docs/SolicitusDeInsumosTodos.xlsx` está validada y se usa como contrato literal del export (FR-110). El archivo de referencia se fija como **fixture inmutable** para tests de contrato (NFR-T-4) y será firmado por **Hugo** (comprador) en la fase de golden datasets (NFR-T-2).

### OQ-103 — Modelo de integración con SIM — **CERRADA (2026-05-24)**

Decisión del owner: **modelo (C) Coexistencia con lectura unidireccional**. Ver §10.4 para detalle, objeción documentada de Winston, justificación, y el trade-off epistémico (§11.16). Modelos (A) y (B) rechazados — razonamiento en `addendum.md`.

### OQ-104 — ¿`Presupuesto-tiendas.csv` distingue formato Express / Estándar?

Verificar al inspeccionar el archivo. Si no lo distingue, levantar data master nueva. **Dueño:** análisis de datos.

### OQ-105 — Costo logístico estimado para sugerencia de TFS — **CERRADA (2026-05-24)**

No aplica. **TFS no se decide en v1** — el sistema solo registra histórico (C-8 reformulada, FR-073). El costo logístico, si importa, es input de la decisión que se toma en otro proceso/programa fuera del sistema.

### OQ-106 — Estructura del histórico de mermas y umbral residual — **CERRADA N/A (2026-05-26)**

Quedó **huérfana** (señalado en el Validate 2026-05-26): preguntaba por la estructura de un histórico de mermas que **ya no se ingesta** (FR-015 eliminado) y por un modelo dual que **ya no existe** (D-9 colapsa a factor único). Se cierra N/A. La pregunta viva sobre el **umbral de detección residual** se traslada a **OQ-123**.

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

El sistema necesita data maestra que no existe en `docs/`: asignación territorial Tienda → Coordinador, merma esperada mensual por tienda, umbrales residuales de mermas, plazos de consolidación, lista de razones estructuradas. **Decisión:** ¿hay un admin operativo? ¿es el PM? ¿es IT? **Dueño:** PM + dirección.

### OQ-113-A + OQ-113-B — Histórico de mermas: **CERRADAS COMO N/A (2026-05-25, segunda iteración del owner)**

**Confirmación del owner (segunda iteración 2026-05-25):** la ingesta del histórico de mermas **no será posible**. Se reemplaza el approach: el coordinador configura el factor manualmente desde día 1. **FR-015 eliminado.** OQ-113-A y OQ-113-B se cierran como N/A. **F-02 (grieta estructural de R-1) cerrado** — la mitigación ya no depende de un dato que no existe; depende del juicio operativo del coord. *(Nota 2026-05-26: D-9 simplificó además de dos factores por categoría tipo_papel+size_papel a un único factor por tipo de papel; ver FR-082/FR-084 reformulados.)*

### OQ-114 — Política fail-loud sobre SIM sucio sin conteo asistido *(nueva)*

Si el snapshot SIM trae saldo negativo / ausente / inconsistente para una tienda × SKU específica, ¿qué hace el sistema? **Opciones:** (a) abortar el forecast de esa tienda con `SaldoSIMInconsistenteException`; (b) marcar warning y continuar usando factor de merma esperada como compensación; (c) asumir cero. Necesita matriz tipo-defecto → acción (escalado por Murat TEA en Party Mode). **Dueño:** arquitectura + PM.

### OQ-115 — Origen de excepciones mid-cycle sin jefe disparándolas — **CERRADA (2026-05-25)**

**Decisión:** opción (c) refinada — el coordinador territorial registra cuando recibe **correo electrónico** del jefe (canal oficial único, decisiones C-O12/C-O14/C-O19). Opciones (a) detección automática y (b) detección manual desde dashboard se rechazan en v1 — no hay flujo de inferencia propia del coord (resuelve F-17). FR-091 reformulado refleja esta decisión con `origen_aviso = correo electrónico` como campo fijo. Opciones (a) y (b) quedan como posibles para v2 si se decide ampliar la detección.

### OQ-116 — ¿El sistema publica el pedido aprobado a SIM como ALLOC esperado? *(nueva)*

Si el comprador genera la OC desde el export del sistema y ALLOC se materializa, ¿se refleja en SIM como ALLOC esperado vía este sistema o eso lo hace el flujo actual de SIM independientemente? Relevante para la coherencia de la coexistencia (C) — si nada publica el ALLOC esperado a SIM, el snapshot SIM siguiente quedará desactualizado en el bucle. **Dueño:** arquitectura + dueño operativo de SIM.

### OQ-117 — Snapshot SIM: SLA de frescura, frecuencia, formato, dueño del extract — **PARCIALMENTE CERRADA (2026-05-25)**

**Confirmación del owner (2026-05-25):**
- ✅ **Frecuencia:** diaria, una actualización por la noche. **En línea NO es posible.** NFR-DAT-4 endurecido a 36h de antigüedad máxima (24h ciclo + 12h buffer).
- ⏳ **Formato:** pendiente — propuesta inicial CSV con encoding UTF-8 BOM o cp1252 según convención del producto.
- ⏳ **Dueño operativo del extract:** pendiente — a confirmar con IT en `architecture.md`.

**Estado:** la parte crítica de la coexistencia (cadencia + SLA) está cerrada; formato y dueño son detalles operativos que arquitectura puede resolver con IT. **Sale del Reviewer Gate como bloqueante** — los detalles restantes no impiden ascender a `status: final` siempre que `architecture.md` los resuelva antes del go-live. **Dueño:** arquitectura + IT.

### OQ-118 — Carry-over de pedidos tardíos — **CERRADA (2026-05-25, decisión D-6)**

**Decisión:** v1 **sí modela el carry-over** automáticamente vía FR-066 — cuando una tienda no envió solicitud antes del corte del miércoles en su ciclo nominal, salta a la siguiente semana con registro automático en bitácora, ajuste de presupuesto disponible, y surfacing visual en la bandeja del coord. Opciones (b) ignorar carry-over y (c) registro manual quedan rechazadas — v1 lo modela automáticamente. Acoplado con FR-066 y FR-061 estado "carry-over de semana anterior".

### OQ-119 — Visibilidad del estado "pendiente solicitud del jefe" en bandeja — **CERRADA (2026-05-25, decisión D-7)**

**Decisión:** opción (b) **visible pasivo** — la bandeja del coord muestra el estado "pendiente solicitud (correo del jefe no llegado)" como información, **no como acción del coord** (perseguir tiendas atrasadas es responsabilidad del jefe, D-coord-7). Permite detectar carry-over y planear. Opción (a) ocultar queda rechazada — pierde visibilidad operativa. Implementado en FR-061 reformulado.

### OQ-120 — Atribución de candidatos residuales entre uso interno vs error de operador — **CERRADA N/A (2026-05-26)**

**Moot tras D-9:** al colapsar la pérdida a un **único factor** (ya no hay dos categorías — uso interno vs error de operador), no hay nada que atribuir. El candidato residual es simplemente "consumo no explicado por venta + factor de pérdida". Se cierra N/A.

### OQ-123 — Umbral de detección estadística residual *(nueva, hereda de OQ-106)*

¿Qué desviación residual (`consumo − venta_esperada − perdida_aplicada`) cuenta como "candidato atípico" en FR-080? **Propuesta inicial:** ≥ 2σ (conservador, prioriza precision sobre recall), calibrable en piloto. Capa secundaria — no bloquea v1. **Dueño:** PM + coordinadores piloto + análisis de datos.

### OQ-124 — Fuente y disponibilidad de ventas semanales (Big Data/BigQuery) *(nueva, D-8)*

Hoy las ventas están a nivel mensual; el motor requiere granularidad semanal (FR-016). ¿Big Data/BigQuery puede entregar ventas por semana × tienda × SKU venta? ¿Formato, SLA, dueño? Si no, fallback de prorrateo mensual (con error). **Bloqueo de la cadencia semanal — confirmar con Fernando.** **Dueño:** PM + Fernando + análisis de datos.

### OQ-125 — Fuente y extracción de ALLOC/TFS desde RMS/BigQuery *(nueva, D-10)*

ALLOC/TFS (asignado/en tránsito/entregado) alimentan FR-017/FR-040/FR-067. ¿Fuente confirmada en RMS/BigQuery? ¿Quién (Fernando), qué disponibilidad, qué formato, qué SLA? Los coords no acceden RMS. **Dueño:** PM + Fernando + arquitectura.

### OQ-126 — Layout del reporte de necesidad de demanda *(nueva, D-11 = abierta)*

Dos vistas candidatas para la consulta del coordinador: (A) por SKU → tiendas+cantidades; (B) por proveedor → tienda → necesidades. El owner lo decide con los coordinadores. El export al comprador (FR-110) mantiene la estructura del archivo de referencia independientemente de la vista de consulta. **Dueño:** PM + coordinadores.

### OQ-121 — Decisión MXN-vs-piezas para factores de uso interno y error de operador — **CERRADA (2026-05-25)**

**Decisión del owner (2026-05-25):** piezas. **SUPERADA por D-9 (2026-05-26):** el factor único de pérdida se expresa como **porcentaje de la demanda de venta** (≈0.9%), no en piezas — consistente con el encuadre "1% de la venta". F-18 ya no aplica.

### OQ-122 — Mecanismo de seed de los dos factores de merma — **CERRADA (2026-05-25), luego SUPERADA por D-9 (2026-05-26)**

**Decisión del owner 2026-05-25:** default = 0 piezas, configuración manual de dos factores por categoría tipo_papel+size_papel. **SUPERADA por D-9 (2026-05-26):** ya no hay dos factores ni categorías tipo_papel+size_papel ni default 0 — hay **un único factor por tipo de papel con default global ≈0.9%**. FR-082 y FR-084 reformulados consecuentemente.

## 14. Glosario

- **Servicentro:** centro de copiado dentro de una tienda Office Depot México que atiende clientes con servicios de impresión, fotocopia, encuadernación.
- **Jefe de servicentro:** empleado de la tienda responsable de operar el servicentro. Hoy calcula manualmente la solicitud de insumos quincenal.
- **Coordinador (de distrito):** uno de tres roles (Carlos, Marco, Eduardo) que supervisa tiendas asignadas territorialmente para excepciones y rota guardia semanal para consolidación.
- **Comprador (corporativo):** rol corporativo que materializa órdenes de compra al proveedor. Recibe el consolidado del coordinador. Fuera del alcance del sistema. **En v1 = Hugo** (identificado 2026-05-25, comprador piloto que firma golden datasets per NFR-T-2).
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
- **SIM:** sistema interno actual donde el jefe registra movimientos manuales de bodega (altas/bajas). Captura manual, dato hoy sucio. En v1 el sistema nuevo lee snapshots de SIM como input unidireccional; SIM **sigue siendo la herramienta única del jefe**.
- **Snapshot SIM:** extracción periódica unidireccional del saldo de inventario en SIM hacia el sistema nuevo (FR-014). **Reemplaza al conteo asistido como input de saldo en v1.** Frecuencia, formato y dueño: OQ-117.
- **Merma / pérdida (actualizado 2026-05-26):** gasto agregado de inventario no atribuible a venta (uso interno + desperdicio juntos — *"al final del día es un gasto, es uno solo"*). En v1 se modela como **un único factor** (ver "Factor de pérdida"); **no se desdobla** en componentes (revierte el modelo dual de 2026-05-25). La medición precisa por tienda es de Fase 3.
- **Factor de pérdida (`factor_perdida`, nuevo 2026-05-26):** único factor configurable que representa la pérdida como **porcentaje de la demanda de venta** (default global ≈0.9%, calibrable en piloto). Configurable por **tipo de papel** (no por tienda, no dual). Alimenta el motor de forecast vía FR-035 con resolución `factor_por_tipo_papel → default_global`. Decisión D-9.
- **Tipo de papel (`tipo_papel`, nuevo 2026-05-26):** unidad de configuración del factor de pérdida — derivada del atributo `Material` de `Skus_insumos.csv` (FR-007). La granularidad operativa que el coordinador entiende ("Opalina", "Bond", etc.). Reemplaza a la "categoría de insumo (tipo_papel+size_papel)" del modelo dual.
- **Fase 1 / Fase 2 / Fase 3 (nuevo 2026-05-26):** encuadre del owner del producto. **Fase 1 = Predicción** (el forecast de demanda — v1). **Fase 2 = Inventario/solicitud** (generación y aprobación de la solicitud; v1 cubre la consolidación con intervención del coord; la solicitud 100% automática es futura). **Fase 3 = Rebajas/merma precisa** (medición de cuánto se rebajó vs compró por tienda; fuera de v1).
- **Dashboard de validación (nuevo 2026-05-26):** vista pedido-vs-recibido (FR-067) que cruza necesidad sugerida, último pedido/asignado y recibido real por proveedor → tienda → SKU, resaltando la brecha. Caza el caso *"pido 10 y me llegan 2"*.
- **Canal oficial (nuevo):** **correo electrónico** como única vía formal por la que una solicitud esporádica o una excepción mid-cycle entra al flujo del sistema. WhatsApp y llamadas existen como notificación informal previa pero no son canal de entrada (decisiones C-O12/C-O14/C-O19, 2026-05-25).
- **Cadencia del forecast (nuevo):** el motor calcula **semanalmente** porque cada semana hay tiendas en su ciclo de pedido (las de semana 1y3 vs 2y4 alternadas). La **unidad de pedido individual por tienda es quincenal** — cada tienda recibe un pedido cada 2 semanas.
- **Eventos comerciales (nuevo):** ventanas de demanda atípica que el motor debe considerar en FR-033. Lista inicial (D-5, pendiente confirmación con Operaciones): Hot Sale, Back to School (BTS), Buen Fin, Navidad/Reyes, regreso a oficinas post-vacacional, Día de las Madres / del Padre / del Niño, cierre fiscal (oct-dic). Cada evento con ventana de fechas y factor multiplicador estimado, calibrado por backtesting cuando no se conozca a priori.
- **Carry-over (nuevo):** mecanismo automático por el cual una tienda que no envió solicitud antes del corte del miércoles en su ciclo nominal salta a la siguiente semana, aunque no esté en su ciclo nominal originalmente. Implementado por FR-066, surface visual en FR-061.
- **Excepción mid-cycle:** evento entre ciclos quincenales que rompe la planeación (venta extraordinaria, evento comercial no anticipado, consumo más rápido del previsto, uso interno elevado). En v1 el aviso jefe→coordinador es exclusivamente por **correo electrónico** (canal oficial); el registro coordinador→sistema y el escalamiento coord→comprador son los pasos que el sistema sí estructura (C-10 reformulada). Sinónimo: "alerta mid-cycle" (usado intercambiablemente en el documento).
- **Override:** modificación manual del **coordinador o administrador** (no del jefe — el jefe no opera el sistema en v1) sobre una cantidad sugerida por el sistema, con razón estructurada capturada. **En ciclos 1-3 es ORO de calibración** (§6 D-1 híbrido; umbral X=25%).
- **Coordinador territorial (nuevo):** uno de los tres coordinadores (Carlos, Marco, Eduardo) responsable de un distrito específico de ~75 tiendas. Atiende todas las semanas la consolidación regular y las excepciones de su distrito. Asignación fija (no rotativa).
- **`buffer_seguridad` (nuevo):** parámetro configurable por administración (datos maestros) por tienda × SKU que representa el inventario de protección frente a variabilidad inter-ciclo no capturada por el factor de pérdida. Valor inicial sugerido = 1 semana de cobertura de venta promedio. Consumido por FR-040.
- **Conteo asistido:** *(término histórico — fuera de alcance v1)*. UI dedicada al conteo físico que originalmente iba a sustituir la captura del jefe en SIM. **Eliminado de v1**; reabre en v1.1 si se decide volver al jefe.
- **MAPE / WAPE:** *Mean Absolute Percentage Error* / *Weighted Absolute Percentage Error*. Métricas de calidad del motor de forecast.
- **Fail-loud:** disciplina de manejo de errores en la que toda condición ambigua o dato faltante dispara una excepción explícita en lugar de degradarse silenciosamente.

---

**Estado actual del PRD:** `draft` — actualizado el **2026-05-26** (tercera ronda) conforme a:
- `.change-signal-2026-05-26.md` (tercera ronda — **reversa del modelo dual a factor único de pérdida por tipo de papel**, encuadre de 3 fases, forecast 99% ventas, ventas semanales desde BigQuery, ALLOC/TFS como input del pedido, FR-061 resuelto, dashboard de validación, inventario no forzado).
- `validation-report.md` del Validate 2026-05-26 (Grade Fair: 0C/3H/7M/2L).
- Transcripción de reunión `docs/Iniciativa Vector - Indicadores Ambientales_2605.docx`.
- Base previa: reescritura 2026-05-25 (`.change-signal-2026-05-25.md`) + respuestas del owner a los 8 bloqueos del Reviewer Gate.

**Decisiones firmadas en update 2026-05-26:** **D-9** (factor único de pérdida por tipo de papel — revierte D-4), **D-8** (ventas semanales desde Big Data/BigQuery), **D-12** (v1 = reporte + bandeja con coord; solicitud automática = futuro), **D-11** (layout del reporte abierto, OQ-126). FR-061 resuelto (coord en pantalla, Excel es output). FR-066 paso 4 (regla de presupuesto en carry-over) enunciada — pendiente de confirmación del owner. OQ-106 y OQ-120 cerradas N/A. OQ-123..OQ-126 abiertas. R-1, R-3, R-7 severidad reducida; R-12 nuevo.

**Findings del Validate 2026-05-26 resueltos en esta ronda:** FR-061 (high) ✅, sub-riesgo R-1 sin gate (high) ✅ disuelto, OQ-120/OQ-106 (medium) ✅ cerradas N/A. **FR-066 carry-over (high):** regla enunciada, pendiente confirmación owner. Findings medium/low remanentes (UJ del comprador, SolicitusDeInsumosTodos.xlsx en project-context.md, FR-110 bit-a-bit, drift glosario) → diferidos.

**Reviewer Gate — estado post-respuestas owner 2026-05-25 (parte 3):**

✅ **Cerrados (5 de 8):**

1. ✅ **R-4 / OQ-102** — Estructura del archivo `docs/SolicitusDeInsumosTodos.xlsx` validada por owner. Se usa como contrato literal del export.
2. ✅ **NFR-E-3** — Cuantificado con inspección directa: ~10 SKUs insumo + ~38 SKUs venta × 226 tiendas = ~8,600 series temporales. Universo trivial computacionalmente.
3. ✅ **Comprador piloto identificado** — **Hugo**. Aplicado en §7.3, NFR-T-2, glosario.
4. ✅ **OQ-113-A + OQ-113-B (F-02)** — **Cerradas como N/A** (segunda iteración del owner 2026-05-25): la ingesta de histórico no será posible. Se reemplaza por configuración manual del coord por categoría tipo_papel+size_papel (FR-082/FR-084 reformulados). FR-015 eliminado. Grieta estructural de F-02 cerrada — la mitigación ya no depende de un dato que no existe.
5. ✅ **OQ-117 cadencia** — Frecuencia diaria nocturna confirmada; no es posible en línea. NFR-DAT-4 endurecido a 36h. Formato y dueño operativo siguen pendientes pero NO bloquean — arquitectura los resuelve con IT.

⏳ **Siguen abiertos (3 de 8):**

6. ⏳ **R-7 / OQ-107** — MAPE/WAPE con Finanzas. Owner gestiona en paralelo; pendiente por negocio (confirmado nuevamente 2026-05-25).
7. ⏳ **R-8** — Spike de forecasting Java sobre datos reales. Severidad baja tras cuantificar NFR-E-3 (universo trivial), pero el spike sigue siendo precondición de pilotaje.
8. ⏳ **R-9** — Baseline de tiempo cerrado (9h/coord/semana). Faltan baseline de tasa de quiebre + sobre-inventario (recolectables en pilotaje) y backtesting sobre histórico real.

**Decisiones nuevas firmadas en update 2026-05-25:** ~~D-4 (modelo dual con dos factores separados)~~ **→ REVERTIDA por D-9 (2026-05-26): factor único de pérdida por tipo de papel**. D-6 (carry-over modelado), D-7 (visible pasivo), OQ-115 cerrada (origen excepción = correo), R-4/OQ-102, NFR-E-3, NFR-T-2 = Hugo, OQ-117 cadencia. ~~OQ-121 (piezas)~~ y ~~OQ-122 (default 0 + config por categoría)~~ **superadas por D-9** (el factor es % de la venta por tipo de papel, default global ≈0.9%). OQ-113-A/B cerradas como N/A.

**Sub-riesgo "arranca con todo en 0" — DISUELTO (2026-05-26):** con factor único y default global ≈0.9% activo desde el go-live, el sistema ya no arranca ciego; la métrica "100% de categorías configuradas como gate" se retira (ya no aplica).

**Reviewer Gate — estado tras update 2026-05-26:** R-7 reencuadrado (criterio = "sin desviaciones vs YoY", deja de ser bloqueo de tesis). R-8 severidad baja (algoritmo simplificado). R-9 baseline de tiempo cerrado (9h); faltan baselines de quiebre/sobre-inventario + backtesting. **Bloqueos externos restantes:** spike Java (R-8), backtesting+baselines (R-9), sesión Finanzas (R-7, opcional). **Nuevo:** R-12 (confirmar fuentes RMS/BigQuery con Fernando).

**Decisiones pendientes restantes (no bloqueantes):** D-5 (lista de eventos comerciales con Operaciones), FR-066 paso 4 (confirmar regla de presupuesto en carry-over), OQ-123 (umbral residual), OQ-124/OQ-125 (fuentes BigQuery/RMS con Fernando), OQ-126 (layout del reporte con coords), distinción BigQuery-lectura vs ML (posible RFC ligero).

