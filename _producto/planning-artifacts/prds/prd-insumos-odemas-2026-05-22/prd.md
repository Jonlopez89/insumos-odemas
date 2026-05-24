---
title: PRD — insumos-odemas
project: insumos-odemas
status: draft
created: 2026-05-22
updated: 2026-05-24
owner: Jonathan
language: es-MX
---

# PRD — insumos-odemas

## Resumen ejecutivo

**insumos-odemas** es un sistema que **automatiza la consolidación distrital quincenal** de la solicitud de insumos de papel para los 226 servicentros (centros de copiado) de Office Depot México. Sustituye el día completo que hoy invierte el coordinador de guardia en armar `SolicitusDeInsumosTodos.xlsx` a mano con un flujo en el que el motor predice la demanda por tienda × SKU, sugiere cantidades respetando el techo de presupuesto individual, presenta histórico de transferencias TFS para informar decisiones y hace visibles las pérdidas que hoy se contabilizan como consumo. **El jefe de servicentro no opera el sistema en v1**: sigue capturando altas y bajas en SIM como hoy. El sistema toma SIM como **input externo unidireccional** de saldo de inventario. La salida es bit-a-bit compatible con el `SolicitusDeInsumosTodos.xlsx` que el comprador corporativo recibe hoy. **Ahorro v1:** un día completo de trabajo del coordinador de guardia por ciclo (~52 días-hombre/año entre los tres).

---

## 1. Visión

Un sistema que **predice, sugiere, valida, traza y consolida** la solicitud quincenal distrital de insumos para los 226 servicentros de OD México — **automatizando la consolidación que hoy hace a mano el coordinador de guardia**, haciendo visibles las pérdidas que hoy se contabilizan como consumo, y respetando SIM como herramienta única de captura de inventario del jefe (el sistema lee de SIM, no escribe ni sustituye su flujo). El v1 reduce un día completo de consolidación a una sesión de revisión y aprobación; el cálculo manual de 6h/quincena del jefe **no es dolor que v1 ataque directamente** — se atiende indirectamente vía mejor forecast del coordinador (ver §3-bis).

## 2. Contexto operativo

### Universo del producto

- **226 servicentros** distribuidos en formato **Express** (tiendas más chicas con menos espacio físico) y **Estándar** (formato completo). Ambos formatos manejan el mismo catálogo de servicios e insumos; el formato no afecta la lógica de forecast.
- **Tres coordinadores** (Carlos, Marco, Eduardo) operan bajo un **modelo híbrido**:
  - **Consolidación quincenal:** rotación de guardia semanal. El coordinador de guardia esa semana procesa todo lo que entró, sin importar zona.
  - **Excepciones mid-cycle:** asignación territorial fija. Cada coordinador atiende exclusivamente las tiendas de su distrito asignado (zona + formato), preservando conocimiento y relación con cada jefe.
  - **La rotación de guardia entre los tres coordinadores es operativa, no la gobierna el sistema** — los coordinadores la organizan ellos. El sistema la consume como input cuando el coordinador la configura (calendario opcional), o cada coordinador opera su bandeja territorial sin orquestación central.
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

- **SIM** — sistema interno donde los jefes capturan altas y bajas de inventario de bodega. La captura es manual y, bajo presión operativa, **frecuentemente imprecisa**. Provee visibilidad de tránsito (ALLOC, TFS) a los coordinadores. **SIM sigue siendo la herramienta única del jefe en v1; el sistema nuevo no la sustituye ni le ofrece UI al jefe.** El sistema nuevo lee snapshots periódicos de SIM como input de saldo (lectura unidireccional, modelo (C) Coexistencia — ver §10.4). La calidad sucia del dato en SIM se mitiga vía factor de merma esperada por tienda × mes (C-9) y backtesting, **no vía conteo asistido** (fuera de scope v1).
- **Excel + correo electrónico** — vehículo principal del flujo de solicitud actual. Cada jefe llena un Excel con su solicitud; el coordinador de guardia arma `SolicitusDeInsumosTodos.xlsx` consolidando ~113 solicitudes por ciclo. El archivo `docs/SolicitusDeInsumosTodos.xlsx` está en el repositorio como referencia del formato actual. **En v1 el jefe sigue enviando su Excel al coordinador como hoy**; el sistema apoya al coordinador en la consolidación.
- **WhatsApp / llamadas / correo informal** — canal de excepciones mid-cycle entre jefe y coordinador, sin estructura, sin trazabilidad, sin priorización. **En v1 el canal entre jefe y coordinador sigue informal**; lo que sí pasa al sistema es la decisión del coordinador territorial (C-10 reformulada).

### Restricciones del entorno

El stack tecnológico obligatorio (Java + Spring Boot + Angular sobre GCP), las convenciones de datos (locale es-MX, fechas `dd/MM/yyyy`, encoding CSV variable, `BigDecimal` para montos, fail-loud sobre datos faltantes), y la disciplina de testing (golden datasets, backtesting, property-based testing con jqwik, contract testing) están definidas en `_producto/project-context.md`. No se repiten aquí.

## 3. El problema

**La consolidación distrital quincenal del coordinador es costosa, lenta, opaca y no escala a 226 tiendas.** El dolor v1 vive en el coordinador. El dolor histórico del jefe se documenta pero **se atiende solo indirectamente** vía mejor forecast del coordinador (ver §3-bis).

### Dolor del Coordinador de distrito

| Código | Dolor |
|---|---|
| **D-coord-1** | **Un día completo** por consolidación de guardia rotativa, ~1 consolidación por semana en la red. **~52 días-hombre/año** entre los tres coordinadores. |
| **D-coord-2** | Si una tienda no envía su Excel a tiempo, el coordinador debe **detectarlo manualmente**. Una tienda olvidada se queda sin pedido esa quincena → quiebre garantizado. |
| **D-coord-3** | Validar `cantidad × costo ≤ techo de presupuesto` para ~113 tiendas × N SKUs **a mano en Excel** es lento y propenso a error aritmético. |
| **D-coord-4** | Las **oportunidades de transferencia entre tiendas (TFS)** dependen de la memoria del coordinador. En 226 tiendas se le escapan casos. La ejecución de TFS vive en otro proceso/programa; v1 solo aporta visibilidad del histórico para informar la decisión. |
| **D-coord-5** | Sin trazabilidad: cuando el coordinador recorta una tienda de 10 a 4 piezas, **en ningún lado queda escrito por qué**. Sin evidencia para auditoría ni para retroalimentar al jefe. |
| **D-coord-6** | El canal informal de excepciones (llamadas, WhatsApp, correo) **no se prioriza ni se conserva**. Una alerta urgente puede perderse en la avalancha. v1 no estructura el canal jefe→coordinador (sigue informal); sí estructura la decisión coordinador→comprador (C-10). |
| **D-coord-7** | Sin **dashboard agregado del distrito en vivo**: cuántas tiendas faltan responder, cuánto presupuesto se ha consumido, qué SKUs concentran riesgo, qué tiendas piden por encima de su techo. |

### Dolor del flujo de excepción mid-cycle (parcialmente atendido en v1)

| Código | Dolor | Tratamiento v1 |
|---|---|---|
| **D-exc-1** | Canal disperso entre jefe y coordinador (WhatsApp + correo + llamada). | **Fuera de scope v1** — sigue informal. Se reabre en v1.1. |
| **D-exc-2** | No hay priorización entre múltiples alertas simultáneas. | **Parcial** — la bandeja del coordinador territorial muestra excepciones registradas con severidad y SKU, pero las alertas iniciales siguen entrando por canal informal. |
| **D-exc-3** | El coordinador decide TFS vs compra directa **sin ver en pantalla** qué tienda cercana tiene exceso del SKU faltante. | **Atendido** vía contexto agregado en la bandeja del coordinador territorial (saldo, ALLOC pendiente, histórico TFS) — C-10 FR-092. |
| **D-exc-4** | No queda trazado quién decidió qué, cuándo, ni por qué. | **Atendido** desde el momento en que el coordinador registra la decisión en el sistema — bitácora C-11. |
| **D-exc-5** | La compra extraordinaria al proveedor **no se refleja en el saldo de planeación** → el siguiente ciclo arranca con dato falso. | **Atendido** — FR-095 reformulado refleja la compra extraordinaria registrada en el saldo del modelo de planeación; SIM se sincroniza por su flujo propio (no por este sistema). |

### Dolor cross-corte — pérdidas invisibles

| Código | Dolor |
|---|---|
| **D-merma-1** | El sistema actual no detecta el patrón **"venta plana o decreciente + solicitudes de insumo crecientes"** que sugiere merma. Los coordinadores lo intuyen pero no lo sistematizan. Cada ciclo perdido es dinero quemado sin trazabilidad. v1 lo atiende vía merma esperada por tienda × mes configurable (C-9 reformulada). |

## 3-bis. Dolor histórico fuera del alcance v1

Estos dolores son reales pero **v1 no los ataca directamente**. Se documentan para mantener visibilidad del actor en el ecosistema y para informar v2. El jefe sigue calculando a mano si quiere — el sistema no le quita ni le da nada en v1; lo que sí ocurre es que **un mejor forecast del coordinador reduce indirectamente la presión sobre el jefe** al disminuir quiebres y rechazos en mostrador.

### Dolor del Jefe de servicentro (histórico, no atendido directamente en v1)

| Código | Dolor |
|---|---|
| **D-jefe-1** | **Hasta 6 horas por quincena** dedicadas al cálculo manual, compitiendo con la atención al personal de servicentro y a clientes en mostrador. *No es ahorro v1.* |
| **D-jefe-2** | El **conteo físico de bodega** se hace bajo presión operativa y queda mal registrado en SIM. Los saldos sucios contaminan el siguiente ciclo de planeación. *v1 no sanea SIM; mitiga vía factor de merma esperada (C-9) y backtesting.* |
| **D-jefe-3** | El cálculo es **empírico, a ojo, en función de la temporada** — sin método formal ni fórmula reproducible. Conocimiento en la cabeza del jefe; si rota personal, su reemplazo arranca de cero. *v1 no le da herramienta al jefe; el coordinador opera con motor estacional.* |
| **D-jefe-4** | Sin visibilidad de inventario en tránsito (ALLOC pendiente, TFS en camino) → riesgo de pedir de más por desconocer lo que ya viene. *El coordinador sí ve esta visibilidad en v1.* |
| **D-jefe-5** | Cuando hay excepciones (ventas extraordinarias), debe **interrumpir su día** para armar un WhatsApp / correo / llamada, y **esperar autorización mientras el cliente lo mira en mostrador**. *Sigue ocurriendo en v1; canal jefe→coordinador queda informal.* |

**Concesión epistémica central:** v1 asume la divergencia físico-teórico en SIM como precondición y la mitiga, no la resuelve. Trade-off consciente del owner: scope manejable a cambio de no atacar la causa raíz. Documentado también en §11.16 y R-1.

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
| **Validación de presupuesto a escala** | Calcular `cantidad × costo ≤ techo` para ~113 tiendas × N SKUs a mano es lento y propenso a error aritmético. | Validación automática por tienda, sugerencia de recorte cuando excede, indicador agregado del consumo distrital. |
| **Histórico y análisis cruzado de TFS** | Requiere mirar el universo entero (exceso vs déficit, semana a semana) — imposible mantenerlo en hojas separadas con el ciclo operativo del coordinador. | Histórico TFS ingestado semanalmente desde `TFS_*.csv`, visualización cruzada por tienda, SKU y periodo. Alimenta como input al motor de forecast. *v1 no decide ni ejecuta TFS — esa decisión vive en otro proceso/programa.* |
| **Merma esperada por tienda configurable + detección estadística residual** | Mantener un valor de merma por tienda × mes ajustable por coordinador y aplicarlo al forecast es trabajo de motor, no de fórmula. | Valor inicial computado del histórico de mermas; coordinador territorial ajusta mensualmente; alimenta el motor de forecast como factor; detección estadística residual surface candidatos para registro humano. |
| **Trazabilidad y auditoría** | Excel no registra quién cambió qué y por qué. Si el comprador audita, el rastro está perdido. | Bitácora automática: sugerencia original → override → razón estructurada → timestamp → actor. Exportable. |
| **Fail-loud sobre datos faltantes** | Excel silencia errores con celdas vacías. Si falta equivalencia SKU venta↔insumo, se infiere o se ignora. | Excepciones de negocio explícitas (`EquivalenciaNoDefinidaException`, `MinimoPackNoConfiguradoException`, etc.) abortan el ciclo de esa tienda hasta resolver — ver `project-context.md`. |
| **Reflejo automático en el siguiente ciclo** | Cada Excel es un punto en el tiempo; el siguiente arranca de cero. | El estado persiste; lo decidido la quincena pasada alimenta el motor de la próxima, incluyendo compras de excepción registradas. |

### La cuenta dura del statu quo

- **Coordinador:** 1 día por consolidación × ~52 consolidaciones/año (cada semana hay una) ≈ **~52 días-hombre/año** en consolidación, repartidos entre los tres en rotación. **Este es el ahorro v1.**
- **TFS no informadas:** cada transferencia que el coordinador no recordó se traduce en compra extra al proveedor. v1 da visibilidad del histórico TFS (semana a semana, cruzado por tienda y SKU) para informar mejor las decisiones que se toman fuera del sistema. Baseline cuantitativo `[ASSUMPTION: medir en pilotaje]` — **ahorro intangible que el histórico empieza a hacer visible**.
- **Mermas invisibles:** sin detección sistemática, hoy se contabilizan como consumo. Es dinero que sale sin trazabilidad ni rendición de cuentas. v1 las modela como **merma esperada configurable** y surface candidatos residuales para registro.
- **Cálculo manual del jefe (~6h × 226 tiendas × ~26 ciclos = decenas de miles de horas/año):** **NO es ahorro v1**. Se documenta en §3-bis y queda como upside futuro para v2 si se decide volver al jefe.

Excel no es competidor del sistema en v1. **Excel es la herramienta del coordinador que el sistema sustituye.**

## 5. Objetivos del producto

Cuatro objetivos, en orden de prioridad:

### O1 — Sustituir el cálculo manual de la consolidación distrital del coordinador

El coordinador de guardia deja de armar `SolicitusDeInsumosTodos.xlsx` a mano. El sistema consolida automáticamente las ~113 solicitudes del ciclo, valida techos individuales, sugiere recortes cuando exceden, presenta histórico TFS cruzado para informar decisiones, aplica factor de merma esperada al forecast y entrega al comprador un export bit-a-bit compatible con el formato actual. **El día completo del coordinador se reduce a una sesión de revisión y aprobación.**

### O2 — Eliminar quiebres y sobre-inventario

Ningún servicentro debe quedarse sin papel en su ciclo normal, **y** ningún servicentro debe acumular papel ocioso. La métrica operativa más visible es **tasa de quiebres en mostrador → cerca de cero** acompañada de **rotación de inventario en rangos sanos por tienda**. Se logra vía mejor forecast del coordinador, no vía intervención sobre el jefe.

### O3 — Sustituir la decisión humana sobre TFS por visibilidad histórica

El sistema entrega al coordinador el dato cruzado de TFS semana a semana (por tienda, SKU, periodo) para informar las decisiones de transferencia que **se siguen tomando en otro sistema/proceso**. v1 no decide ni ejecuta TFS — solo registra histórico y lo alimenta al motor de forecast.

### O4 — Hacer visibles las pérdidas hoy invisibles

Las mermas dejan de ser folclore distrital. El sistema computa **merma esperada por tienda × mes** desde histórico, el coordinador territorial la **ajusta**, y la merma esperada **alimenta el motor de forecast** como factor. Además, una capa de **detección estadística residual** surface candidatos sospechosos para registro humano del coordinador, acumulados para revisión trimestral.

## 6. Métricas de éxito (preliminar)

Las metas cuantitativas finales se acuerdan con Finanzas en pilotaje. Las propuestas iniciales son:

| Métrica | Baseline | Objetivo v1 | Cómo se mide |
|---|---|---|---|
| **Tiempo del coordinador por consolidación** | 1 día completo | **< 2 h** | Telemetría: tiempo entre primera apertura de la bandeja y export del consolidado. **Métrica principal v1.** |
| **Tasa de quiebre (cliente rechazado por falta de insumo)** | `[OPEN: baseline a recolectar en pilotaje]` | **Reducción ≥ 80%** | Contador en mostrador + reporte semanal de tiendas piloto. |
| **Sobre-inventario (cobertura en semanas de venta)** | `[OPEN: baseline]` | Por tienda, mantener en rango sano (a definir con Finanzas) | Reporte del sistema: cobertura por SKU/tienda. |
| **TFS registradas en histórico y analizadas por coordinador** | 0 (memoria del coordinador) | `[OPEN: meta v1]` | Contador en sistema. **Visibilidad, no decisión** — la TFS se ejecuta fuera del sistema. |
| **MAPE / WAPE del motor de forecast** | n/a | `[OPEN: acordar con Finanzas en backtesting]` | `BacktestingSuite.java` sobre histórico real — ver disciplina de testing en `project-context.md`. |
| **Mermas detectadas y registradas** | 0 (intuición) | `[OPEN: meta v1]` | Contador en sistema; revisión trimestral con coordinadores. Entrada: histórico de mermas + configuración mensual del coordinador. |
| **Override del coordinador — ciclos 1-3 (calibración)** | n/a | **≥ X% por ciclo con razón estructurada registrada** (X a calibrar; propuesta inicial: 30%) | Contador en sistema con desglose por razón estructurada. **En esta fase el override es ORO de información para calibrar el motor**, no señal de desconfianza. Decisión Dr. Quinn (D-1 híbrido). |
| **Override del coordinador — ciclo 4+ (confianza)** | n/a | **< 20% por ciclo** | Contador en sistema. Se activa una vez que MAPE esté estabilizado y firmado con Finanzas. Hasta entonces, esta métrica se reporta solo como vista secundaria. |

### Contra-métricas (señales de que el sistema está fallando aunque las métricas principales se vean bien)

- **Tasa de override > 50% por ciclo** en alguna tienda *después de la fase de calibración*: el coordinador no confía en la sugerencia; el sistema está sobreajustando o pidiendo cosas absurdas.
- **Override estancado bajo X% durante los ciclos 1-3** (la inversa de la métrica de calibración): el coordinador está corrigiendo poco, el sistema no recibe señal — investigar si la sugerencia es buena o si el coordinador está siendo pasivo.
- **Excepciones mid-cycle creciendo después del lanzamiento**: el forecast inicial subestima sistemáticamente; calibrar el factor de merma esperada.
- **Pedidos urgentes al proveedor (fuera de ciclo) creciendo**: o el forecast falla o la coexistencia (C) con SIM está degradando.
- **`SolicitusDeInsumosTodos.xlsx` exportado pero el comprador sigue pidiendo correcciones**: incompatibilidad downstream con el flujo del comprador, ajustar el formato.
- **Reducción del tiempo del coordinador pero crecimiento de quiebres**: la automatización está omitiendo casos que el coordinador detectaba manualmente.
- **Divergencia entre saldo SIM y consumo real reportado por venta crece** (acoplada con R-11): señal de que la coexistencia (C) se está rompiendo silenciosamente. Investigar SLA de frescura del snapshot SIM, captura del jefe en SIM, o factor de merma esperada mal calibrado.

---

## 7. Personas y User Journeys

### 7.1 Stakeholder externo — Jefe de servicentro (no es usuario del sistema en v1)

- **Quién es:** empleado de una tienda específica, 25-35 años. Lidera al personal del servicentro, gestiona la solicitud de insumos, atiende problemas operativos del área. No multitask cross-departamento.
- **Relación con el sistema en v1: cero interacción.** Sigue capturando altas y bajas en SIM como hoy y enviando su Excel de solicitud por correo al coordinador como hoy. El sistema no le provee UI ni le sustituye nada.
- **Por qué sigue documentado como persona:** mantener visibilidad del actor en el ecosistema, informar v2, y permitir que el equipo de UX no pierda la perspectiva del extremo del flujo. Su dolor histórico está en §3-bis.

### 7.2 Persona — Coordinador de distrito (usuario interactivo principal de v1)

- **Quién es:** uno de tres (Carlos, Marco, Eduardo). Full-time en servicentro pero responsabilidades más allá del inventario.
- **Universo asignado:** las 226 tiendas se reparten territorialmente entre los tres por zona + formato. Cada tienda tiene un coordinador territorial asignado para excepciones; la consolidación quincenal opera por guardia rotativa global semanal organizada por los coordinadores fuera del sistema.

**Tres modos de operación en el sistema:**

- **(a) Bandeja de guardia rotativa — consolidación.** Cuando le toca semana de guardia, abre la bandeja con las ~113 tiendas del ciclo, ve sugerencias de recorte por presupuesto, histórico TFS cruzado, factor de merma aplicado, override con razón estructurada, aprueba el consolidado, exporta a comprador.
- **(b) Bandeja territorial — excepciones y mermas.** Atiende las alertas mid-cycle que llegan por canal informal del jefe (WhatsApp/correo/llamada, fuera del sistema) y las **registra en el sistema** con decisión: TFS de emergencia gestionada fuera del sistema, escalamiento a comprador para compra directa con contexto, rechazo. También revisa candidatos de merma estadística residual del territorio.
- **(c) Administración.** Configura merma esperada mensual por tienda (input para C-4), umbrales de detección, lista de razones estructuradas. Algunas funciones pueden requerir rol de Administrador.

- **Cómo opera hoy:** miércoles arranca la consolidación. Recibe ~113 correos con Excel adjuntos, valida techo de presupuesto a mano por tienda, recorta cuando excede, se acuerda de TFS oportunas (por memoria), arma `SolicitusDeInsumosTodos.xlsx` y lo manda al comprador. Tiempo: un día completo.
- **Qué lo hace feliz:** **dejar de armar `SolicitusDeInsumosTodos.xlsx` a mano**. Recuperar el día completo que hoy se va en consolidación.

### 7.3 Persona — Comprador corporativo (stakeholder pasivo)

- **Rol único** que recibe el consolidado del coordinador y materializa órdenes de compra al proveedor. **Fuera del alcance del sistema** — no opera la UI. **Sí es consumidor downstream**: el export del sistema debe ser compatible con su flujo actual basado en `SolicitusDeInsumosTodos.xlsx` recibido por correo.

### 7.4 User Journey AS-IS — Cálculo quincenal del jefe (UJ-1) — *contexto histórico, sigue ocurriendo fuera del sistema*

UJ-1 sigue documentado para mantener visibilidad del flujo real del jefe que el sistema NO ataca en v1. Ver detalle completo en `addendum.md` (apéndice "UJ-1 AS-IS — Cálculo manual del jefe"). Resumen: ciclo quincenal → conteo físico → captura en SIM → cálculo empírico estacional en Excel → correo al coordinador, hasta 6 horas/quincena. **Dolores asociados:** D-jefe-1..5 (§3-bis).

### 7.5 User Journey AS-IS — Consolidación del coordinador (UJ-2)

1. (Miércoles, semana de guardia rotativa) coordinador entra al correo, encuentra ~113 Excel adjuntos.
2. Recolecta solicitudes; controla manualmente quién no respondió.
3. Consulta SIM para saldos, ALLOC pendiente y TFS en tránsito.
4. Valida cada solicitud contra su techo individual; recorta tiendas que exceden.
5. Por memoria, identifica oportunidades de TFS entre tiendas con exceso/déficit; **detecta** y **solicita** la TFS dentro de la jornada, pero la **ejecución** sigue por proceso paralelo (fuera del alcance v1).
6. Compila `SolicitusDeInsumosTodos.xlsx`.
7. Envía por correo al comprador.

**Tiempo:** un día completo. **Dolores:** D-coord-1..7.

### 7.6 User Journey AS-IS — Excepción mid-cycle (UJ-3) — *parcialmente atendido por v1*

**Importante:** este journey arranca con el jefe disparando una alerta informal al coordinador. **En v1 el sistema NO gobierna los pasos 1-2**; el jefe sigue avisando por canal informal (WhatsApp/correo/llamada) como hoy. Lo que sí pasa al sistema es la decisión y el escalamiento a partir del paso 3.

1. *(Fuera del sistema)* Jefe detecta quiebre inminente por venta extraordinaria o consumo más rápido de lo previsto.
2. *(Fuera del sistema)* Contacta a **su coordinador territorial** por WhatsApp / correo / llamada — sin canal único.
3. **(En el sistema)** Coordinador territorial **registra** la excepción en su bandeja con SKU, cantidad estimada faltante, urgencia, motivo estructurado y tienda. El sistema muestra contexto agregado: saldo SIM actual, ALLOC pendiente, histórico TFS reciente para el SKU.
4. **(En el sistema)** Coordinador decide caso por caso: gestionar TFS de emergencia (ejecución sigue fuera del sistema; el coordinador solo registra la decisión y el resultado), escalamiento al comprador para compra directa al proveedor (sistema notifica con contexto), o rechazo con razón.
5. *(Fuera del sistema)* Compra extraordinaria al proveedor (si autorizada) ocurre fuera del ciclo normal con el flujo del comprador.
6. **(En el sistema)** La compra extraordinaria autorizada se **refleja en el saldo del modelo de planeación** del sistema para que el siguiente ciclo parta de la realidad.

**Dolores parcialmente atendidos:** D-exc-3, D-exc-4, D-exc-5 (ver §3 tabla de tratamiento). **No atendidos en v1:** D-exc-1 (canal jefe→coordinador sigue informal), D-exc-2 (priorización solo aplica una vez registrada).

### 7.7 User Journey TO-BE — Consolidación del coordinador con el sistema (UJ-2-TB)

1. **(Lunes-martes)** Jefes envían su solicitud por correo al coordinador como hoy. *Fuera del sistema.*
2. **(Miércoles AM)** Coordinador de guardia abre la bandeja de la semana. Sistema muestra ~113 tiendas con su estado: capturada / pendiente captura del jefe / con recorte sugerido / aprobada / exportada. Snapshot SIM más reciente cargado como saldo inicial.
3. **(Miércoles AM)** Por cada tienda, el sistema presenta: cantidad sugerida por SKU con trazo explicable (incluyendo factor de merma esperada aplicado), validación de techo presupuestal individual, recorte sugerido si excede, histórico TFS cruzado para informar contexto.
4. **(Miércoles PM)** Coordinador revisa, aplica overrides con razón estructurada (cuando proceda — recordatorio: en ciclos 1-3 el override es ORO de calibración, ver §6), aprueba tienda por tienda o en lote.
5. **(Jueves)** Coordinador exporta el consolidado a `SolicitusDeInsumosTodos.xlsx` y lo envía al comprador con CC configurable. Hash del archivo y destinatarios en bitácora.
6. **(Continuo durante la semana)** Coordinadores territoriales atienden excepciones mid-cycle (UJ-3) en paralelo.

**Tiempo objetivo:** < 2h por consolidación. **Dolores atendidos:** D-coord-1..7. **Métrica de éxito principal v1:** §6 tiempo del coordinador.

## 8. Capacidades y Requisitos Funcionales

Las capacidades del sistema se agrupan en **doce áreas**. Cada FR tiene identificador estable (FR-NNN) y se referencia desde el resto del PRD. Cuando una capacidad responde directamente a un dolor de §3, se nombra.

### 8.1 Datos maestros (C-1)

Base operativa que el resto de capacidades consume.

- **FR-001** — Mantener catálogo de **tiendas** con: ID (String), nombre, distrito, **coordinador territorial asignado**, **formato (Express / Estándar)**, **ciclo quincenal asignado** (semana 1-y-3 o 2-y-4), **techo presupuestal semanal MXN**, **merma esperada mensual configurable (MXN o piezas, por confirmar — alimenta C-4)**.
- **FR-002** — Mantener catálogo de **SKUs de insumo** (lo comprable al proveedor): código, descripción, presentación, contenido (piezas por presentación), costo MXN. Preservar literal `"Matarial y tamaño"` por contrato con el equipo de datos.
- **FR-003** — Mantener **tabla de equivalencia** SKU venta ↔ SKU insumo. Toda venta procesada debe encontrar su equivalencia; sin equivalencia → `EquivalenciaNoDefinidaException` y aborto del ciclo de esa tienda (fail-loud).
- **~~FR-004~~** — *Eliminado.* La rotación de guardia entre coordinadores es operativa y la organizan ellos fuera del sistema. El sistema puede consumirla como calendario opcional configurado por el administrador (parte de FR-005), pero no la gobierna.
- **FR-005** — Mantener **datos de coordinadores**: identidad, distrito territorial asignado (lista de tiendas), credenciales, rol. Opcionalmente, calendario informativo de quién está de guardia esa semana (consumido como input, no gobernado por el sistema).

### 8.2 Ingesta y normalización de datos (C-2)

Carga de archivos canónicos en `docs/` al modelo del sistema. Las convenciones (encoding, parsing dual de fechas, separadores, tipos exactos, fail-loud) están en `project-context.md` y no se repiten.

- **FR-010** — Ingestar archivos canónicos del inventario en `docs/` por **descubrimiento por patrón de nombre** (`ALLOC_YYYY_MM_DD.csv`, `TFS_YYYY_MM_DD.csv`, `Entrega-directa-tienda*.csv`). Nuevos archivos sin reconfiguración del sistema.
- **FR-011** — Detectar **encoding** por archivo en orden: UTF-8 BOM → cp1252. Registrar encoding detectado en MDC.
- **FR-012** — Validar invariantes de ingesta documentadas en `project-context.md` (ej. `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE` en ALLOC, `TRAN_CODE ↔ DECODE` en Entrega-directa). Comportamiento fail-loud cuando aplica.
- **FR-013** — Parsear `Cantidad por semana` de `Presupuesto-tiendas.csv` (formato `"$X,XXX"`) y `Semana que debe solicitar` (lenguaje natural → `List<Integer>` ISO). Patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`.
- **FR-014** *(nuevo)* — Ingestar **snapshots periódicos de SIM** (saldo de inventario por tienda × SKU) como input externo unidireccional. Frecuencia, formato y dueño del extract: OQ-117. Política fail-loud para snapshot ausente / saldo negativo / inconsistente: OQ-114. Reemplaza al conteo asistido como input de saldo.
- **FR-015** *(nuevo)* — Ingestar **histórico de mermas** como input externo para alimentar C-9 (cálculo inicial de merma esperada por tienda × mes). Formato, fuente y frecuencia: OQ-113.

### 8.3 ~~Conteo asistido de bodega (C-3)~~ — *Eliminado, fuera de scope v1*

C-3 (FR-020..025) se elimina de v1 por decisión del owner (C-O2). El jefe sigue capturando inventario en SIM como hoy; el sistema no provee UI de conteo. Justificación y trade-off en §11.12 y §11.16. La calidad sucia de SIM se mitiga vía factor de merma esperada (C-9) y backtesting, no vía conteo asistido. **Reabre en v1.1** si se decide volver al jefe.

### 8.4 Forecast de demanda quincenal (C-4) — Dolor: D-coord-1 (atendido indirectamente D-jefe-3)

Motor de predicción por tienda × SKU. Es el corazón estructural del valor v1: el coordinador deja de calcular sobre intuición y opera sobre sugerencias del motor.

- **FR-030** — Calcular **demanda esperada de venta** para el próximo ciclo por tienda × SKU venta, usando modelo estacional configurable (familia de modelos en Java puro definida en `project-context.md`).
- **FR-031** — Convertir demanda de venta a **demanda de insumo** aplicando la tabla de equivalencia.
- **FR-032** — **Cuantizar** la demanda de insumo al múltiplo del empaque mínimo (`Contenido` en `Skus_insumos`). Nunca redondear hacia abajo si hay demanda.
- **FR-033** — Aplicar **componente estacional anual** (regreso a clases, fin de mes, cierres fiscales, exámenes) basado en histórico ≥ 12 meses.
- **FR-034** — Generar **trazo explicable** del cálculo por SKU: histórico utilizado, ventana, factor estacional, **factor de merma esperada aplicado**, ajuste por inventario en tránsito. Visible en hover en UI.
- **FR-035** *(nuevo)* — Aplicar **factor de merma esperada por tienda × mes** (configurado en C-9) como parámetro del modelo de forecast. El trazo explicable debe declarar el valor del factor aplicado y su origen (computado del histórico o ajustado manualmente por coordinador).

### 8.5 Sugerencia de pedido (C-5) — Dolor: D-coord-1, D-coord-3 (atendido D-jefe-4 indirectamente)

Calcula la cantidad final a pedir considerando saldo SIM (vía snapshot) y tránsito. **El actor que ve la sugerencia es el coordinador en su bandeja de consolidación**, no el jefe.

- **FR-040** — Para cada tienda × ciclo, calcular **cantidad sugerida por SKU** = `demanda_quincenal_insumo + buffer_seguridad − saldo_SIM_snapshot − ALLOC_pendiente − TFS_entrante`, cuantizada al empaque mínimo. El factor de merma esperada (FR-035) ya está aplicado en `demanda_quincenal_insumo`.
- **FR-041** — Mostrar al **coordinador en su bandeja de consolidación** la sugerencia por tienda × SKU con su trazo explicable.
- **~~FR-042~~** — *Eliminado.* El jefe no opera el sistema en v1, por lo tanto no aplica override del jefe. El override del coordinador se consolida con FR-064.
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
- **FR-061** — Por cada tienda, mostrar estado: **pendiente solicitud del jefe** (correo no llegado) / **solicitud recibida** (Excel capturado o ingresado) / sugerencia del motor lista / con recorte sugerido / aprobada / exportada. Sin estado "pendiente captura jefe" porque el jefe no opera el sistema.
- **~~FR-062~~** — *Eliminado.* La alerta de captura del jefe no aplica — el jefe no captura en el sistema. El coordinador detecta tiendas sin solicitud por correo como hoy.
- **FR-063** — Mostrar **agregados del ciclo**: presupuesto consumido total, número de tiendas con solicitud pendiente, SKUs con riesgo concentrado, costo estimado del consolidado.
- **FR-064** — Permitir al coordinador **modificar cantidades** (override) en cualquier tienda con razón estructurada obligatoria (lista predefinida + texto libre opcional); modificaciones en bitácora. Consolida lo que era FR-042. En ciclos 1-3 estos overrides son input crítico de calibración (ver §6 D-1 híbrido).
- **FR-065** — Permitir **aprobar el consolidado** y dispararlo al export downstream (C-12).

### 8.8 Histórico y análisis de transferencias TFS (C-8) — Dolor: D-coord-4

**v1 NO sugiere ni decide TFS.** Las transferencias se gestionan en otro proceso/programa. v1 solo provee histórico semanal y visualización cruzada para informar las decisiones que el coordinador toma fuera del sistema, y alimenta el motor de forecast con ese histórico.

- **~~FR-070~~** — *Eliminado.* No hay detección de pares exceso/déficit en v1.
- **~~FR-071~~** — *Eliminado.* No hay sugerencia de cantidad transferible en v1.
- **~~FR-072~~** — *Eliminado.* No hay sugerencia para aceptar/modificar/descartar — la TFS se decide fuera del sistema.
- **FR-073** *(reformulado)* — **Registrar histórico de TFS ejecutadas** ingestando `TFS_*.csv` semanalmente. Este histórico cumple dos funciones: (a) visibilidad para el coordinador (análisis cruzado), (b) input al motor de forecast (C-4) para mejorar la predicción de saldos efectivos.
- **FR-074** *(nuevo)* — **Visualización cruzada del histórico TFS** al coordinador: filtros por tienda origen, tienda destino, SKU, periodo (semana/mes). Tablero exportable a CSV.
- **FR-075** *(nuevo)* — Cuando el coordinador territorial registra una excepción mid-cycle (C-10) y decide gestionar una TFS de emergencia *fuera del sistema*, puede **registrar manualmente la decisión y el resultado** para que quede en bitácora; el saldo del sistema **no se modifica directamente** desde aquí (la TFS se refleja en el histórico cuando aparezca en `TFS_*.csv` del siguiente extract).

### 8.9 Merma esperada por tienda y detección estadística residual (C-9) — Dolor: D-merma-1

Modelo de dos capas: **(1) merma esperada configurable** que alimenta el motor de forecast como factor (input determinístico que absorbe el sesgo sistemático del dato sucio en SIM), y **(2) detección estadística residual** que surface lo que la merma esperada no explica.

- **FR-080** *(reformulado)* — Detectar **patrones estadísticos sospechosos** por tienda × SKU: `consumo_real_observado > venta_esperada_considerando_merma_configurada`, alta variabilidad inter-ciclo más allá de banda esperada. **Sin conteo asistido** — se opera sobre snapshot SIM + venta histórica.
- **FR-081** — Generar **candidatos de merma residual** clasificados por severidad (divergencia vs banda esperada). Surfacing en bandeja del coordinador territorial asignado.
- **FR-082** *(ampliado)* — Permitir al coordinador **capturar / modificar / descartar** cada candidato residual (SKU, cantidad estimada, motivo, evidencia opcional) **y configurar la merma esperada mensual por tienda** (acción nueva — gobierna el factor que alimenta C-4 vía FR-035).
- **FR-083** — Acumular registro de mermas confirmadas para revisión trimestral con dirección. Exportable.
- **FR-084** *(nuevo)* — El sistema **computa merma esperada por tienda × mes** desde el histórico de mermas ingestado (FR-015) como **valor inicial sugerido**; el coordinador territorial lo ajusta mensualmente desde la UI de administración (FR-082).
- **FR-085** *(nuevo)* — La **merma esperada configurada alimenta el motor de forecast** (C-4) vía FR-035. Cambios en el factor son trazados en bitácora con timestamp + coordinador + valor anterior + valor nuevo + razón estructurada.

### 8.10 Registro de excepción mid-cycle por el coordinador (C-10) — Dolor: D-exc-3, D-exc-4, D-exc-5 (parciales)

**Decisión D-2 — conservar canal coordinador → comprador.** El canal jefe → coordinador sigue informal (WhatsApp/correo/llamada como hoy, fuera del sistema). El coordinador territorial **entra a la bandeja del sistema para registrar** la excepción que recibió por canal informal, decidir con contexto, y escalar al comprador con trazo.

- **~~FR-090~~** — *Eliminado.* El jefe no opera el sistema en v1. La alerta inicial sigue siendo informal.
- **FR-091** *(reformulado)* — El **coordinador territorial** registra manualmente una excepción mid-cycle en su bandeja con: tienda, SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado, origen del aviso (WhatsApp/correo/llamada/inferencia propia). La excepción **se asigna automáticamente al territorio del coordinador** que la registra (no hay routing — él ya es el destinatario por construcción).
- **FR-092** — Mostrar al coordinador en su bandeja **contexto agregado** para la excepción: saldo SIM más reciente de la tienda, ALLOC pendiente, TFS en tránsito y histórico TFS reciente para el SKU (C-8). **Sin sugerencia automática de TFS de emergencia** — la decisión y la ejecución viven fuera del sistema.
- **FR-093** *(reformulado)* — Permitir al coordinador **decidir y registrar** la respuesta: gestionar TFS de emergencia fuera del sistema (registrar solo decisión, no ejecución — ver FR-075), escalamiento a comprador para compra directa, rechazo (con razón), espera al ciclo normal.
- **FR-094** — Cuando se escala a comprador, **notificarlo** por canal acordado (correo, integración) con contexto + decisión del coordinador + datos de la tienda y el SKU.
- **FR-095** — Reflejar en el saldo de planeación cualquier **compra extraordinaria autorizada** (cuando el comprador confirme), para que el siguiente ciclo parta de la realidad. SIM se sincroniza por su propio flujo (no por este sistema).

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

- **NFR-E-1** — Soportar **226 tiendas activas en el modelo de datos**. **Usuarios humanos concurrentes: hasta 3 coordinadores + 1 administrador** (no hay capturas paralelas de jefes en v1).
- **~~NFR-E-2~~** — *Eliminado.* No hay 113 jefes capturando en paralelo — el jefe no opera el sistema en v1.
- **NFR-E-3** — El motor de forecast batch debe **procesar el universo completo** (226 tiendas × N SKUs) en una ventana razonable que no bloquee la operación del miércoles. **`[OPEN: cuantificar N de SKUs por tienda — bloqueante escalado por Winston (R-8). Sin un orden de magnitud concreto, "minutos no horas" es deseo, no NFR accionable. Decisión cae implícitamente en architecture.md.]`**
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
- **NFR-S-2** — Roles operativos v1: **Coordinador** (acceso a sus tiendas territoriales para excepciones y configuración de merma esperada + bandeja de guardia rotativa cuando aplica) y **Administrador** (configura datos maestros, umbrales de detección de mermas, lista de razones estructuradas). **Rol "Jefe de servicentro" eliminado de v1** — el jefe no opera el sistema.
- **NFR-S-3** — **MFA obligatorio** para roles administrativos.
- **NFR-S-4** — Datos sensibles (presupuestos, costos) **no se exponen** a roles que no los necesitan operativamente.

### 9.6 Calidad del dato como pilar (NFR-Datos)

Este NFR es estructural — el sistema solo es tan bueno como el conteo.

- **NFR-DAT-1** — **Fail-loud** ante datos faltantes o ambiguos: equivalencia SKU no definida, presupuesto no configurado, encoding ambiguo, periodicidad presupuestal no reconocida, snapshot SIM ausente o con saldo inconsistente → excepciones checked específicas que abortan el ciclo de esa tienda (heredado de `project-context.md`). **Política específica para SIM sucio en OQ-114.**
- **~~NFR-DAT-2~~** — *Eliminado.* No hay conteo asistido en v1; NFR perdió razón de ser.
- **NFR-DAT-3** — Encoding de archivos detectado y **registrado en logs** por archivo procesado, incluyendo el snapshot SIM (FR-014).
- **NFR-DAT-4** *(nuevo)* — **SLA de frescura del snapshot SIM** (acoplado con R-11 y OQ-117): el sistema debe rechazar operar sobre snapshot SIM con antigüedad mayor a `[OPEN: 48h propuesto]` y notificar al administrador.

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

### 10.4 SIM — decisión cerrada: modelo (C) Coexistencia con lectura unidireccional

**SIM** es el sistema interno donde los jefes de servicentro capturan altas y bajas de inventario en bodega de forma manual. Los coordinadores tienen acceso de lectura para ver ALLOC y TFS.

**Decisión del owner (2026-05-24, OQ-103 cerrada):** **modelo (C) Coexistencia con lectura unidireccional**. El sistema nuevo NO escribe a SIM. SIM es fuente **unidireccional** (lectura de snapshots periódicos vía CSV extract). La verdad operativa del coordinador (planeación, decisiones, bitácora) vive en el sistema nuevo. La verdad contable de inventario sigue en SIM. **Divergencia aceptada como precondición**; monitoreada por R-11.

**Objeción documentada de Winston (Architect, Party Mode 2026-05-24):** *"(C) coexistencia con doble captura es garantía de divergencia — dos sistemas, dos verdades, ningún ganador. Es el peor de los tres en producción aunque parezca el más seguro políticamente."*

**Justificación del owner Jonathan:** el jefe NO captura en el sistema nuevo (clarificación C-O7 del change signal), por lo tanto **"doble captura del jefe" no aplica como riesgo** — el jefe sigue capturando UNA SOLA VEZ en SIM como hoy. La divergencia que sí existe es entre **SIM contable** y **plan operativo del coordinador**, no entre dos UIs del jefe. Trade-off aceptado: scope manejable v1 a cambio de no resolver el problema epistémico de fondo (concesión central declarada en §11.16, R-1, R-11).

**Modelos (A) y (B) rechazados** — razonamiento completo en `addendum.md` apéndice "Opciones SIM consideradas".

**Información operacional pendiente (no bloquea la decisión, sí condiciona el diseño):**

- Frecuencia, formato y dueño del extract SIM → snapshot del sistema (OQ-117).
- Política fail-loud sobre snapshot ausente o con saldo inconsistente (OQ-114).
- ¿El sistema publica el pedido aprobado a SIM como ALLOC esperado o eso lo hace el flujo actual de SIM? (OQ-116).

### 10.5 Datos maestros adicionales requeridos por el sistema

Más allá de lo que viene en `docs/`, el sistema necesita gobernar:

- **Mapeo Tienda → Coordinador territorial** (para C-10 contexto de excepciones).
- **Calendario informativo de rotación de guardia** (opcional, consumido como input — no gobernado por el sistema; los coordinadores la organizan fuera).
- **Merma esperada mensual por tienda** (gobernada por coordinador territorial vía FR-082 + FR-084; alimenta el motor de forecast vía FR-035).
- **Umbrales configurables** para C-9 detección estadística residual (qué desviación residual considerar "sospechosa" más allá de la merma esperada configurada).
- **Plazos del ciclo de consolidación** del coordinador — semana ISO + día (miércoles por defecto). Plazos de captura del jefe NO aplican porque el jefe no captura en el sistema.
- **Lista de razones estructuradas** para overrides del coordinador, recortes, excepciones mid-cycle (configurable por admin).

### 10.6 Integraciones externas

- **SIM** — **lectura unidireccional de snapshots** (CSV extract). Frecuencia, formato y dueño del extract: OQ-117. Decisión modelo (C) cerrada en §10.4. No hay escritura del sistema hacia SIM en v1.
- **Histórico de mermas** — input externo para C-9 (FR-015). Dueño, formato, frecuencia: OQ-113.
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

El sistema **provee la herramienta**; la capacitación de los coordinadores y el anuncio oficial son **responsabilidad de la célula de UX / Eficiencia operativa** y del área operativa. Crítico para la adopción, pero fuera del producto. **Identificado como riesgo (R-2).**

### 11.11 Captura del jefe de servicentro en el sistema nuevo

El jefe **sigue capturando altas y bajas en SIM como hoy** y sigue enviando su Excel de solicitud por correo al coordinador. **v1 no provee UI/UX para el jefe.** Reabre en v1.1 si se decide volver al jefe.

### 11.12 Conteo asistido de bodega

La captura del conteo físico sigue siendo manual en SIM como hoy. **v1 no aporta UI dedicada al conteo.** La calidad sucia se mitiga vía factor de merma esperada (C-9) y backtesting.

### 11.13 Decisión y ejecución de transferencias TFS

El sistema **solo registra histórico desde `TFS_*.csv`** y lo visualiza para informar decisiones. **La decisión y la ejecución de TFS viven en otro proceso/programa** fuera de v1.

### 11.14 Rotación de guardia entre coordinadores

**El sistema no la gobierna.** Los coordinadores la organizan operativamente fuera del sistema. Sí puede consumirse como calendario informativo opcional configurado por admin (FR-005), pero no es una capacidad gobernada.

### 11.15 Canal estructurado de alertas mid-cycle disparadas por el jefe

El jefe **sigue contactando al coordinador por canales informales** (WhatsApp/correo/llamada). **El sistema no provee bandeja de entrada del jefe ni canal estructurado jefe→coordinador.** Lo que sí estructura el sistema es la decisión y el escalamiento del coordinador hacia el comprador (C-10 reformulada).

### 11.16 Saneamiento del dato de inventario en SIM

**v1 NO sanea SIM.** La calidad del dato sucio se asume como precondición y se mitiga vía factor de merma esperada (C-9) y backtesting, no vía conteo asistido. **Esta es la concesión epistémica central de v1** (diagnosticada por Dr. Quinn en Party Mode 2026-05-24 y firmada conscientemente por el owner): v1 automatiza el efecto, no la causa raíz. Documentado también en R-1 con mitigación específica.

## 12. Riesgos

Riesgos materiales identificados en Discovery. Cada uno con dueño tentativo y mitigación propuesta. Final en `architecture.md` y plan de pilotaje.

### R-1 — Calidad sucia del dato en SIM contamina el sistema nuevo *(severidad: alta)*

- **Descripción:** SIM hoy tiene saldos imprecisos porque el conteo de los jefes se hace bajo presión. El sistema lee snapshot SIM como saldo inicial; sin conteo asistido (C-3 eliminada en v1), opera sobre el dato tal cual. **Esta es la concesión epistémica central de v1** (§11.16).
- **Mitigación triple:**
  1. **Factor de merma esperada por tienda × mes** (C-9 reformulada, FR-035) absorbe parte del sesgo sistemático como parámetro del modelo de forecast.
  2. **Backtesting sobre histórico real** (`BacktestingSuite.java`, NFR-T-5) cuantifica el impacto en MAPE antes del piloto — bloqueante R-9.
  3. **Monitoreo continuo de divergencia** entre predicción del sistema y venta real reportada (contra-métrica nueva en §6) alerta cuando SIM se desvía críticamente — acoplado con R-11.
- **Dueño tentativo:** PM + arquitectura.

### R-2 — Resistencia al cambio operativo *(severidad: media — reducida tras scope cut)*

- **Descripción:** los coordinadores pueden seguir cruzando correos por costumbre y desconfiar de la sugerencia inicial. **Severidad bajó significativamente** porque el scope v1 no toca a 226 jefes — solo a 3 coordinadores. El grupo afectado es pequeño y conocido.
- **Mitigación:** plan formal de capacitación para los 3 coordinadores antes del go-live; iteración rápida en los primeros 3 ciclos con override como ORO de calibración (§6 D-1); revisiones quincenales con los coordinadores en los primeros 60 días para construir confianza.
- **Dueño tentativo:** coordinadores + sponsor (dirección de operaciones).

### R-3 — Definición de "merma sospechosa residual" — falsos positivos erosionan confianza *(severidad: media — más interpretable tras C-9 reformulada)*

- **Descripción:** umbrales mal calibrados generan demasiados candidatos residuales falsos; el coordinador descarta sistemáticamente y deja de mirar la bandeja → la detección se vuelve folclore.
- **Mitigación más fácil tras C-9 reformulada:** ahora la "merma" tiene una **expectativa configurada** por el coordinador (merma esperada mensual por tienda), así que un falso positivo es divergencia respecto a una expectativa que él mismo configuró — mucho más interpretable que un umbral abstracto. Umbrales conservadores en v1 (priorizar precision sobre recall), iteración trimestral con datos reales.
- **Dueño tentativo:** PM + coordinadores piloto + Finanzas.

### R-4 — Incompatibilidad downstream con el comprador

- **Descripción:** el export `SolicitusDeInsumosTodos.xlsx`-compatible se rompe si la estructura cambia o si el comprador modifica su flujo. Cada incompatibilidad obliga al coordinador a corregir manualmente — pierde el ahorro.
- **Mitigación:** revisar `docs/SolicitusDeInsumosTodos.xlsx` para fijar contrato; involucrar al comprador en pilotaje desde semana 1; tests automatizados de export contra fixture firmada (NFR-T-2).
- **Dueño tentativo:** arquitectura + PM.

### R-5 — Dependencia técnica y organizacional de SIM *(severidad: media — reducida tras decisión (C))*

- **Descripción:** con lectura unidireccional (modelo C cerrado en §10.4), **no se necesita API ni dueño técnico de SIM**. Solo se necesita un **extract programado** (CSV) a una ubicación compartida que el sistema lea. La dependencia organizacional se limita a quien produzca el extract.
- **Mitigación:** identificar al dueño operativo del extract SIM (OQ-117); acordar formato, frecuencia y SLA de frescura; mecanismo de aviso si el extract falla o se atrasa.
- **Dueño tentativo:** IT o dueño operativo de SIM + arquitectura.

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

### R-11 — Divergencia silenciosa entre SIM contable y plan operativo del coordinador *(severidad: alta — nuevo riesgo)*

- **Descripción:** con coexistencia (C) cerrada en §10.4, si el snapshot SIM no se actualiza a tiempo, o si la captura manual del jefe en SIM se vuelve más floja (al ver que el sistema nuevo "hace todo"), el sistema planea sobre datos viejos. **La objeción de Winston en Party Mode 2026-05-24** se materializa precisamente aquí: las dos verdades pueden divergir silenciosamente y la consolidación opera sobre la divergencia.
- **Mitigación triple:**
  1. **SLA de frescura de snapshot SIM** (NFR-DAT-4, propuesta inicial 48h máximo de antigüedad). Si se excede, el sistema rechaza operar y notifica al administrador.
  2. **Contra-métrica de divergencia predicción vs venta** (§6) monitorea el sesgo agregado del sistema; si crece sistemáticamente, señal de que SIM se está desviando.
  3. **Revisión trimestral con dueño operativo de SIM** + coordinadores para auditar muestras de tiendas y reconciliar manualmente cuando aplique.
- **Dueño tentativo:** arquitectura + IT + dueño operativo de SIM.

## 13. Open Questions

Lista consolidada de decisiones abiertas. Cada una tiene contexto, propuesta inicial y dueño tentativo.

### OQ-101 — Frecuencia real de excepciones mid-cycle

Provisional `[ASSUMPTION: 1-3 por semana en la red de 226]`. **Medir post-lanzamiento** con telemetría. No bloquea v1; afecta el diseño visual de prominencia de la bandeja del coordinador territorial. **Dueño:** PM.

### OQ-102 — Estructura exacta de `SolicitusDeInsumosTodos.xlsx`

Revisar el archivo de referencia en `docs/`. Definir el contrato del export (columnas, orden, tipos, encabezados, hoja). Fijar como fixture inmutable para tests de contrato (NFR-T-4). **Dueño:** arquitectura + PM.

### OQ-103 — Modelo de integración con SIM — **CERRADA (2026-05-24)**

Decisión del owner: **modelo (C) Coexistencia con lectura unidireccional**. Ver §10.4 para detalle, objeción documentada de Winston, justificación, y el trade-off epistémico (§11.16). Modelos (A) y (B) rechazados — razonamiento en `addendum.md`.

### OQ-104 — ¿`Presupuesto-tiendas.csv` distingue formato Express / Estándar?

Verificar al inspeccionar el archivo. Si no lo distingue, levantar data master nueva. **Dueño:** análisis de datos.

### OQ-105 — Costo logístico estimado para sugerencia de TFS — **CERRADA (2026-05-24)**

No aplica. **TFS no se decide en v1** — el sistema solo registra histórico (C-8 reformulada, FR-073). El costo logístico, si importa, es input de la decisión que se toma en otro proceso/programa fuera del sistema.

### OQ-106 — Estructura del histórico de mermas y umbral residual *(reformulada)*

Tras C-9 reformulada, la pregunta cambió de "qué umbral cuenta como merma" a: **¿cuál es la estructura del histórico de mermas (input externo, OQ-113) y qué umbral residual cuenta como sospechoso una vez configurada la merma esperada mensual?** Requiere análisis de datos históricos + criterio de coordinadores. **Dueño:** PM + coordinadores piloto + análisis de datos.

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

### OQ-113 — Histórico de mermas: dueño + formato + fuente *(nueva)*

Necesario para C-9 reformulada y FR-015. ¿De dónde viene el histórico de mermas que el sistema usa para computar la merma esperada inicial por tienda × mes? Archivo nuevo en `docs/`, sistema interno existente, captura manual histórica del coordinador? Formato y frecuencia. **Dueño:** PM + Operaciones.

### OQ-114 — Política fail-loud sobre SIM sucio sin conteo asistido *(nueva)*

Si el snapshot SIM trae saldo negativo / ausente / inconsistente para una tienda × SKU específica, ¿qué hace el sistema? **Opciones:** (a) abortar el forecast de esa tienda con `SaldoSIMInconsistenteException`; (b) marcar warning y continuar usando factor de merma esperada como compensación; (c) asumir cero. Necesita matriz tipo-defecto → acción (escalado por Murat TEA en Party Mode). **Dueño:** arquitectura + PM.

### OQ-115 — Origen de excepciones mid-cycle sin jefe disparándolas *(nueva)*

Sin el jefe disparando alertas en el sistema, ¿cómo entra una excepción al sistema para que el coordinador la registre? **Opciones:** (a) detección automática por patrón en snapshot SIM (consumo anómalo, saldo bajo, ventas pico); (b) coordinador territorial detecta manualmente desde dashboard al revisar territorio; (c) coordinador registra cuando recibe aviso informal del jefe (decisión D-2 escogió esta). Acoplado con FR-091 reformulado. **Dueño:** PM + coordinadores.

### OQ-116 — ¿El sistema publica el pedido aprobado a SIM como ALLOC esperado? *(nueva)*

Si el comprador genera la OC desde el export del sistema y ALLOC se materializa, ¿se refleja en SIM como ALLOC esperado vía este sistema o eso lo hace el flujo actual de SIM independientemente? Relevante para la coherencia de la coexistencia (C) — si nada publica el ALLOC esperado a SIM, el snapshot SIM siguiente quedará desactualizado en el bucle. **Dueño:** arquitectura + dueño operativo de SIM.

### OQ-117 — Snapshot SIM: SLA de frescura, frecuencia, formato, dueño del extract *(nueva)*

Para que la coexistencia (C) no degrade silenciosamente (acoplado con R-11 y NFR-DAT-4). **Propuesta inicial:** snapshot diario, formato CSV con encoding UTF-8 BOM o cp1252 según convención del producto, dueño operativo a confirmar, SLA de frescura ≤48h. **Dueño:** arquitectura + IT.

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
- **SIM:** sistema interno actual donde el jefe registra movimientos manuales de bodega (altas/bajas). Captura manual, dato hoy sucio. En v1 el sistema nuevo lee snapshots de SIM como input unidireccional; SIM **sigue siendo la herramienta única del jefe**.
- **Snapshot SIM:** extracción periódica unidireccional del saldo de inventario en SIM hacia el sistema nuevo (FR-014). **Reemplaza al conteo asistido como input de saldo en v1.** Frecuencia, formato y dueño: OQ-117.
- **Merma esperada:** valor configurado mensualmente por tienda que representa la pérdida prevista por consumo no atribuible a venta. Base inicial computada desde histórico de mermas (FR-084); ajustable por coordinador territorial (FR-082); alimenta el motor de forecast como factor (FR-035).
- **Excepción mid-cycle:** evento entre ciclos quincenales que rompe la planeación (venta extraordinaria, consumo más rápido del previsto). En v1 el aviso jefe→coordinador sigue informal; el registro coordinador→sistema y el escalamiento coord→comprador son los pasos que el sistema sí estructura (C-10 reformulada).
- **Override:** modificación manual del coordinador (no del jefe — el jefe no opera el sistema en v1) sobre una cantidad sugerida por el sistema, con razón estructurada capturada. **En ciclos 1-3 es ORO de calibración** (§6 D-1 híbrido).
- **Conteo asistido:** *(término histórico — fuera de alcance v1)*. UI dedicada al conteo físico que originalmente iba a sustituir la captura del jefe en SIM. **Eliminado de v1**; reabre en v1.1 si se decide volver al jefe.
- **MAPE / WAPE:** *Mean Absolute Percentage Error* / *Weighted Absolute Percentage Error*. Métricas de calidad del motor de forecast.
- **Fail-loud:** disciplina de manejo de errores en la que toda condición ambigua o dato faltante dispara una excepción explícita en lugar de degradarse silenciosamente.

---

**Estado actual del PRD:** `draft` — secciones 1-14 reescritas el 2026-05-24 conforme al change signal `.change-signal-2026-05-24.md` post-Party-Mode. **Pendiente para `status: final`:** Reviewer Gate con bloqueos críticos resueltos (R-4 inspección de `SolicitusDeInsumosTodos.xlsx`, R-7 MAPE/WAPE con Finanzas, R-8 spike de forecasting Java, NFR-E-3 cuantificación N SKUs, R-9 backtesting precede al piloto, identificación de comprador piloto por nombre).

