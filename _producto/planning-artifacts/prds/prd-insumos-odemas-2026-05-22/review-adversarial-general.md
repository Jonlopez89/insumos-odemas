---
title: Revisión adversarial — PRD insumos-odemas
prd_target: prd.md
reviewer_role: Reviewer cínico (Validate intent del skill bmad-prd)
created: 2026-05-25
language: es-MX
status: review-output
---

# Revisión adversarial cínica — PRD insumos-odemas

> Tono: revisor escéptico, profesional, sin paciencia para *hand-waving*. Cada hallazgo cita sección/ID y, donde aplica, **comilla literal** del PRD. La severidad es **impacto sobre la utilidad operativa del PRD** (¿puede `architecture.md` o `bmad-create-epics-and-stories` consumirlo sin tener que volver a hacer Discovery?).
>
> Marco de validación: las 6 reviewer gates explícitamente declaradas bloqueantes en `.decision-log.md` (R-4, R-7, R-8, NFR-E-3, R-9, comprador piloto) **siguen abiertas**. El PRD reconoce esto al final (§799), pero el cuerpo está redactado como si las hubiera asimilado. Eso es la primera tensión que esta revisión tiene que iluminar.

---

## Resumen ejecutivo de la revisión

El PRD reescrito el 2026-05-24 es un trabajo serio de scope-cut y de honestidad epistémica en algunas zonas (§11.16, R-1, R-11). Pero **la honestidad declarada en las anotaciones no se sostiene cuando uno lee el cuerpo principal con ojo cínico**: el resumen ejecutivo, §1 Visión y §5 O1 hacen claims fuertes ("bit-a-bit compatible", "sustituye el día completo del coordinador", "el comprador recibe export compatible") cuyo soporte está en blockers abiertos del Reviewer Gate. El producto firmado por el owner es coherente; el documento que lo describe **mezcla aspiración y especificación** y un lector apurado se llevaría la primera.

**Stance:** El PRD **no está listo para `status: final`**. Tiene tres clases de problema:
1. **Promesas que no están sustentadas por mitigaciones engineered** (compatibilidad bit-a-bit, factor de merma esperada como mitigación primaria de R-1, "minutos no horas" del motor).
2. **Out-of-scope que silenciosamente reentra por la puerta de atrás** vía FR-005 (calendario opcional), FR-075 (registro manual de TFS), §11.5 que invoca un FR ya eliminado.
3. **Inconsistencias residuales del rewrite** — referencias muertas, NFRs que mencionan al jefe siendo que el jefe no opera el sistema, glosario que aún define el override del jefe por implicación.

Estos problemas son arreglables; varios son edits de superficie. Pero hay un nudo duro que no es edit: **el factor de merma esperada (C-9, FR-035) hace todo el trabajo pesado en el PRD** — es mitigación de R-1, justificación de no atacar la causa raíz (§11.16), tres FRs nuevos (FR-035, FR-082, FR-084, FR-085), una métrica de éxito (O4), una contra-métrica, y la base epistémica para que SIM sucio no contamine el forecast. Y todo descansa sobre OQ-113 (estructura del histórico de mermas) **abierta**, sin dueño confirmado, sin formato, sin frecuencia. Es decir: **la mitigación central del riesgo central depende de un input cuya existencia operativa nadie ha confirmado**.

---

## Hallazgos

### F-01 [crítico] — "Bit-a-bit compatible con SolicitusDeInsumosTodos.xlsx" se promete en resumen ejecutivo mientras R-4 sigue abierto

**Cita literal §Resumen ejecutivo (línea 15):** *"La salida es bit-a-bit compatible con el `SolicitusDeInsumosTodos.xlsx` que el comprador corporativo recibe hoy."*

**Cita literal §12 R-4 (línea 651):** *"el export `SolicitusDeInsumosTodos.xlsx`-compatible se rompe si la estructura cambia"* — riesgo activo, mitigación: *"revisar `docs/SolicitusDeInsumosTodos.xlsx` para fijar contrato"*.

**Cita literal §FR-110 (línea 366):** *"Formato compatible con el archivo de referencia en `docs/SolicitusDeInsumosTodos.xlsx` `[ASSUMPTION: estructura por confirmar al inspeccionar el archivo en Contexto Técnico — §10]`."*

**Cita literal §13 OQ-102 (línea 710):** *"Revisar el archivo de referencia en `docs/`. Definir el contrato del export"* — OWNERSHIP: *"arquitectura + PM"*. Abierta.

**Cita literal §799 (cierre):** *"Pendiente para `status: final`: Reviewer Gate con bloqueos críticos resueltos (R-4 inspección de `SolicitusDeInsumosTodos.xlsx`...)"*.

**El problema:** el resumen ejecutivo afirma una garantía operativa ("bit-a-bit compatible") que **no se puede afirmar hasta que alguien abra `docs/SolicitusDeInsumosTodos.xlsx` y caracterice su estructura**. El archivo **está en `docs/`** — verificado en este review — y nadie lo ha inspeccionado todavía según la propia OQ-102 y el decision log. "Bit-a-bit" es lenguaje contractual: si el comprador hace `diff` contra el export y aparecen diferencias en columnas, orden, hoja, encoding, formato de fecha o tipo numérico, la promesa se rompe. **No es "compatible" — es "aspira a ser compatible cuando alguien haga el trabajo de R-4"**. Esta línea del resumen ejecutivo está vendiéndole al lector lo que el sistema *quiere* ser, no lo que está especificado.

**Impacto:** un sponsor leyendo solo el resumen cierra el documento creyendo que el contrato downstream ya está cerrado. No lo está.

**Acción sugerida:** sustituir *"La salida es bit-a-bit compatible con..."* por algo como *"La salida está diseñada para ser compatible con `SolicitusDeInsumosTodos.xlsx` — contrato exacto del export pendiente de definir en R-4/OQ-102 antes del go-live"*. O, mejor: **inspeccionar el archivo ahora** (está en `docs/`) y cerrar OQ-102 antes de validar el PRD. Es trabajo de minutos, no de sprint.

---

### F-02 [crítico] — El factor de merma esperada (C-9) carga todo el peso de R-1 sin que su input exista

**Cita §11.16 (línea 622):** *"v1 NO sanea SIM. La calidad del dato sucio se asume como precondición y se mitiga vía factor de merma esperada (C-9) y backtesting, no vía conteo asistido. Esta es la concesión epistémica central de v1"*.

**Cita §R-1 (línea 631-634):** mitigación triple — *"(1) Factor de merma esperada por tienda × mes (C-9 reformulada, FR-035) absorbe parte del sesgo sistemático... (2) Backtesting... (3) Monitoreo continuo de divergencia..."*.

**Cita FR-084 (línea 338):** *"El sistema computa merma esperada por tienda × mes desde el histórico de mermas ingestado (FR-015) como valor inicial sugerido"*.

**Cita FR-015 (línea 273):** *"Ingestar histórico de mermas como input externo para alimentar C-9 (cálculo inicial de merma esperada por tienda × mes). Formato, fuente y frecuencia: OQ-113."*

**Cita OQ-113 (línea 754):** *"Necesario para C-9 reformulada y FR-015. ¿De dónde viene el histórico de mermas que el sistema usa para computar la merma esperada inicial por tienda × mes? Archivo nuevo en `docs/`, sistema interno existente, captura manual histórica del coordinador? Formato y frecuencia. **Dueño:** PM + Operaciones."*

**El problema:** la lógica encadenada es: SIM tiene dato sucio → mitigamos con factor de merma esperada → el factor se calcula desde histórico de mermas → **el histórico de mermas es OQ-113 abierta, sin dueño confirmado, sin formato, sin saber siquiera si existe como dato hoy**. La opción "captura manual histórica del coordinador" en OQ-113 sugiere que tal vez **el dato simplemente no existe** y habría que construirlo. Si OQ-113 se resuelve con "no hay histórico de mermas, hay que empezar a recolectarlo", entonces:

- FR-084 no puede operar en ciclo 1 (sin valor inicial computado).
- FR-035 opera sin valor de factor de merma (¿con qué default?).
- R-1 pierde su mitigación principal.
- §11.16 pierde su justificación epistémica (la concesión es real, pero la "mitigación" es vapor).

**El backtesting (mitigación 2 de R-1)** depende del spike de Winston (R-8) y de cuantificar N SKUs (NFR-E-3), ambas también abiertas. El monitoreo de divergencia (mitigación 3) es contra-métrica de telemetría — no es mitigación, es detección posterior al daño.

**Esto no es un edit, es una grieta estructural.** El PRD asume que la merma esperada es un dato compaginable; el owner debe responder OQ-113 antes de que esta mitigación sea defendible. Mientras tanto, el PRD está afirmando que mitiga un riesgo con un mecanismo cuyo combustible no está garantizado.

**Acción sugerida:** OQ-113 debe pasar a la lista de bloqueos del Reviewer Gate junto con las otras seis. Si la respuesta es "no hay histórico", entonces hay que decidir explícitamente qué hace v1 en el ciclo 1 (¿factor de merma = 0 por defecto, surface al coordinador?) y declararlo en el PRD.

---

### F-03 [alto] — Modelo (C) Coexistencia: la objeción de Winston está documentada con honor, pero R-11 es contención, no mitigación

**Cita §10.4 (línea 516):** objeción de Winston citada textual — *"(C) coexistencia con doble captura es garantía de divergencia... Es el peor de los tres en producción aunque parezca el más seguro políticamente."*

**Cita §10.4 (línea 518):** justificación del owner — *"el jefe NO captura en el sistema nuevo... entonces 'doble captura del jefe' no aplica como riesgo... La divergencia que sí existe es entre SIM contable y plan operativo del coordinador, no entre dos UIs del jefe."*

**Cita R-11 (línea 693):** *"con coexistencia (C) cerrada en §10.4, si el snapshot SIM no se actualiza a tiempo, o si la captura manual del jefe en SIM se vuelve más floja (al ver que el sistema nuevo 'hace todo'), el sistema planea sobre datos viejos. La objeción de Winston en Party Mode 2026-05-24 se materializa precisamente aquí"*.

**El problema:** la justificación del owner replantea el riesgo de Winston ("doble UI del jefe") como un riesgo distinto ("divergencia entre SIM contable y plan operativo"), pero **R-11 reconoce textualmente que la objeción de Winston se materializa precisamente en este nuevo riesgo**. Es decir, la justificación dice "no es ese riesgo, es este otro" y luego R-11 dice "este otro es el mismo riesgo que Winston nombró". El sleight-of-hand está en el cambio de etiqueta, no en cambiar la realidad.

Las mitigaciones de R-11:
1. SLA de frescura 48h — `[OPEN: 48h propuesto]` en NFR-DAT-4 (no es decisión, es propuesta).
2. Contra-métrica de divergencia — observación post-hoc, no prevención.
3. Revisión trimestral con dueño operativo de SIM — quien aún no se ha identificado (OQ-117).

**Ninguna de las tres es engineered mitigation; todas son detección + escalamiento humano.** La objeción de Winston era estructural ("garantía de divergencia"), y la respuesta del PRD es "vamos a vigilarla". Vigilar una divergencia que tu propio arquitecto te dijo que es estructural no la elimina — solo te avisa antes de que el daño sea grande. Eso puede ser un trade-off razonable, pero **el PRD debería decirlo así de directo, no envolverlo en "trade-off aceptado por el owner"**.

**Acción sugerida:** reformular la justificación del owner en §10.4 para reconocer explícitamente que R-11 es la misma objeción de Winston con nombre nuevo, y que las tres mitigaciones son **detección, no prevención**. La honestidad ya está en R-1 ("concesión epistémica central"); aplicarla con el mismo rigor a §10.4.

---

### F-04 [alto] — "Visibilidad histórica TFS" (C-8 reformulada) — capacidad pasiva sin valor cuantificado

**Cita O3 (línea 151):** *"Sustituir la decisión humana sobre TFS por visibilidad histórica"*. **Sustituir** es palabra fuerte; lo que el PRD entrega es **complementar**, no **sustituir** — la decisión sigue siendo humana, solo recibe más datos en la pantalla.

**Cita FR-073 reformulado (línea 326):** *"Registrar histórico de TFS ejecutadas ingestando `TFS_*.csv` semanalmente. Este histórico cumple dos funciones: (a) visibilidad para el coordinador (análisis cruzado), (b) input al motor de forecast (C-4) para mejorar la predicción de saldos efectivos."*

**Cita métrica de éxito §6:** *"TFS registradas en histórico y analizadas por coordinador — Baseline: 0 (memoria del coordinador) — Objetivo v1: `[OPEN: meta v1]`"*.

**Cita §4 tabla "Por qué no Excel":** *"Histórico TFS ingestado semanalmente desde `TFS_*.csv`, visualización cruzada por tienda, SKU y periodo. Alimenta como input al motor de forecast. *v1 no decide ni ejecuta TFS — esa decisión vive en otro proceso/programa.*"*

**El problema:** C-8 reformulada quedó reducida a **tres FRs** (FR-073, FR-074, FR-075) que conjuntamente significan: "ingerimos el CSV, lo mostramos en una tabla con filtros, y el coordinador puede anotar a mano una decisión que tomó fuera del sistema". Esto **no es una capacidad** — es una vista de datos con un campo de texto libre. La meta cuantitativa es `[OPEN]`. El "valor" prometido es indirecto ("alimenta al forecast"), pero el PRD no especifica **cómo** el motor de forecast usa el histórico TFS (¿ajuste de saldo en FR-040? ¿feature del modelo en FR-030?). Y para colmo, **O3 dice "sustituir la decisión humana"** cuando lo que se entrega es una pantalla.

Adicionalmente, **FR-075 abre una puerta peligrosa**: *"el coordinador puede registrar manualmente la decisión y el resultado"*. Esto es captura manual en el sistema de algo que ya pasó fuera del sistema → segunda verdad → riesgo de divergencia entre lo que el coordinador anota y lo que `TFS_*.csv` reporta del siguiente extract → exactamente la trampa que la coexistencia (C) ya tiene en SIM, replicada en TFS. **Out-of-scope §11.13 dice "decisión y ejecución viven fuera del sistema"; FR-075 dice "registra manualmente la decisión"**. Eso es ambivalencia: ¿la decisión está en el sistema o no?

**Acción sugerida:**
1. Bajar O3 de "sustituir" a "informar" (lo que realmente entrega el PRD).
2. Cerrar la meta cuantitativa de la métrica TFS o aceptar que es métrica de output, no de outcome, y reportarla como tal.
3. Especificar cómo el histórico TFS entra al motor (¿feature? ¿corrección de saldo?), o admitir que es **input para v2 del motor** y que en v1 solo es tabla con filtros.
4. Resolver la tensión FR-075 vs §11.13: o la decisión vive en el sistema (entonces no es out-of-scope), o no vive (entonces FR-075 no debería existir).

---

### F-05 [alto] — Motor Java + factor de merma esperada: el camino del input al cálculo está hand-waved frente a las restricciones de stack

**Cita FR-035 (línea 288):** *"Aplicar factor de merma esperada por tienda × mes (configurado en C-9) como parámetro del modelo de forecast. El trazo explicable debe declarar el valor del factor aplicado y su origen"*.

**Cita FR-040 (línea 294):** *"cantidad_sugerida por SKU = demanda_quincenal_insumo + buffer_seguridad − saldo_SIM_snapshot − ALLOC_pendiente − TFS_entrante, cuantizada al empaque mínimo. El factor de merma esperada (FR-035) ya está aplicado en demanda_quincenal_insumo."*

**Cita project-context.md §"Motor de forecasting":** *"Modelos iniciales permitidos: media móvil ponderada, suavizado exponencial / Holt-Winters, ARIMA. Nada de redes neuronales o ML 'sofisticado' sin justificación cuantitativa."* + *"No agregar Python al stack sin RFC formal"*.

**El problema:** *"Aplicar factor de merma esperada como parámetro del modelo"* es lenguaje vago. Las opciones técnicas concretas, en términos de Holt-Winters / ARIMA / media móvil ponderada (los tres modelos permitidos en `project-context.md`), son:

- **(a) Multiplicador sobre el output del modelo** (`demanda = forecast × (1 + merma_pct)`). Trivial, pero ignora estacionalidad del propio fenómeno de merma.
- **(b) Feature en el modelo** (regresor exógeno). Holt-Winters básico no acepta regresores; ARIMA sí (ARIMAX). Smile no documenta ARIMAX explícito — habría que verificar.
- **(c) Ajuste del histórico de entrada antes de ajustar el modelo** (`venta_efectiva = venta + merma_estimada`). Cambia el régimen de la serie.

**El PRD elige una opción técnica sin nombrarla.** Cualquiera de las tres tiene implicaciones de implementación, costo computacional, e interaccción con el spike de Winston (R-8) que está abierto. La frase *"ya está aplicado en demanda_quincenal_insumo"* en FR-040 sugiere implícitamente (a) o (c), pero no lo especifica — y la decisión cae implícitamente en `architecture.md` sin que el PRD reconozca que **la elección afecta el MAPE que se acuerda con Finanzas (R-7)**.

Adicionalmente, *"El trazo explicable debe declarar el valor del factor aplicado y su origen"* es UX sobre algo que internamente puede ser opaco (un peso dentro de un Holt-Winters). Generar trazo explicable de un modelo estadístico no es trivial — exige caracterizar el modelo escogido y diseñar el formato del trazo. El PRD lo trata como si fuera transparente.

**Acción sugerida:** declarar en §10 o en C-9 las tres opciones técnicas y **escalar la decisión a `architecture.md`** explícitamente (no implícitamente). Reconocer que la elección impacta R-7 y R-8.

---

### F-06 [alto] — NFR-E-3 marcado como bloqueante en el cuerpo del PRD: anti-patrón de Validate

**Cita NFR-E-3 (línea 380):** *"El motor de forecast batch debe procesar el universo completo (226 tiendas × N SKUs) en una ventana razonable que no bloquee la operación del miércoles. `[OPEN: cuantificar N de SKUs por tienda — bloqueante escalado por Winston (R-8). Sin un orden de magnitud concreto, 'minutos no horas' es deseo, no NFR accionable. Decisión cae implícitamente en architecture.md.]`"*

**El problema:** un NFR cuyo enunciado **dice de sí mismo** "esto es deseo, no NFR accionable" **no es un NFR**. Es una nota al margen. Un PRD que se valida con NFRs así no debería pasar Validate. La marca `[OPEN]` y la honestidad de "no accionable" son ejemplares para un draft, **pero el PRD está siendo evaluado para `status: final`** según el frontmatter y el cierre §799. Un NFR no-accionable es un agujero en el contrato que arquitectura va a tener que rellenar adivinando o estimando — y si la estimación falla, no hay enunciado contractual contra el cual reclamar.

Esto vale para **otros NFRs**:
- NFR-D-1 *"≥ 99% durante ventanas operativas"* — bien definido.
- NFR-D-2 *"el conteo y la captura del jefe deben poder reanudarse sin pérdida"* — **referencia al conteo y al jefe, que están out-of-scope**. NFR muerto. (Ver F-09.)
- NFR-E-4 *"<3 segundos"* — bien definido.
- NFR-DAT-4 *"snapshot SIM con antigüedad mayor a `[OPEN: 48h propuesto]`"* — también `[OPEN]`.

**Acción sugerida:** o el owner cuantifica N (orden de magnitud, no número exacto — "centenas", "miles", "decenas de miles") y NFR-E-3 se vuelve testeable, o el PRD reconoce explícitamente que NFR-E-3 quedará deferred a `architecture.md` y no es validable en este nivel. Equivalente para NFR-DAT-4.

---

### F-07 [alto] — Decisión D-1 (override híbrido) tiene métrica con variable libre X sin valor inicial firmado

**Cita §6 tabla métricas — fila "Override del coordinador — ciclos 1-3":** *"≥ X% por ciclo con razón estructurada registrada (X a calibrar; propuesta inicial: 30%)"*.

**Cita §6 contra-métricas:** *"Override estancado bajo X% durante los ciclos 1-3 (la inversa de la métrica de calibración)"*.

**Cita .decision-log.md §D-1:** *"Decisión: dos métricas escalonadas... Ciclos 1-3: override ≥X% por ciclo... (X propuesta inicial: 30%)"*.

**El problema:** D-1 está marcada como **resuelta** en el decision log, pero **X sigue siendo variable libre**. "Propuesta inicial 30%" no es decisión — es punto de partida para una conversación. La métrica que sale al go-live tiene que tener un valor numérico contra el cual el coordinador y Finanzas puedan medir. Si el equipo aterriza en ciclo 1 con "≥X%" y nadie firmó X, **el reporte de la métrica no se puede generar** o se genera con el placeholder.

Adicionalmente, el contra-señal "estancado bajo X%" es la inversa booleana de la métrica, lo cual significa que **un coordinador que overridee al 28% en ciclo 1 estaría "estancado" según contra-métrica pero "no cumpliendo" según métrica principal** — el sistema tendría dos alarmas simultáneas por el mismo dato. La lógica está bien, pero el umbral único significa que la zona de confort tiene grosor cero.

**Acción sugerida:** o fijar X (ej. 25% como umbral mínimo, ≥30% como meta de calibración óptima), o agendar la calibración de X con Finanzas como parte del Reviewer Gate. No dejar la fórmula con variable libre en el PRD validado.

---

### F-08 [alto] — D-2 y D-3 declaradas resueltas en decision log pero los FRs reformulados conservan ambigüedad

**Cita .decision-log.md D-2:** decisión es *"CONSERVAR CANAL COORDINADOR → COMPRADOR"*. Aplicada en §8.10.

**Cita FR-091 reformulado (línea 346):** *"El coordinador territorial registra manualmente una excepción mid-cycle en su bandeja con: tienda, SKU, cantidad estimada faltante, fecha esperada de quiebre, urgencia, motivo estructurado, origen del aviso (WhatsApp/correo/llamada/inferencia propia)."*

**Cita OQ-115 (línea 762):** *"Sin el jefe disparando alertas en el sistema, ¿cómo entra una excepción al sistema para que el coordinador la registre? Opciones: (a) detección automática por patrón en snapshot SIM (consumo anómalo, saldo bajo, ventas pico); (b) coordinador territorial detecta manualmente desde dashboard al revisar territorio; (c) coordinador registra cuando recibe aviso informal del jefe (decisión D-2 escogió esta). Acoplado con FR-091 reformulado."*

**El problema:** D-2 dice "decisión: conservar canal coord→comprador", pero **OQ-115 sigue lista como abierta** con tres opciones y una nota *"decisión D-2 escogió esta"* refiriéndose a la opción (c). Si D-2 ya decidió, **OQ-115 debe cerrarse**, no quedar listada con las tres opciones como si todavía estuviera por decidir. Esto deja al lector con la duda: ¿el sistema también tendrá detección automática (opción a) o solo registro manual (opción c)? FR-091 implica solo (c), pero OQ-115 abierta sugiere que (a) sigue en debate.

Equivalente para D-3: *"UJ-3 con jefe fuera del sistema: MANTENER COMO AS-IS CONTEXTO"*. §7.6 quedó reescrita con marcas `(En el sistema)` y `(Fuera del sistema)`. Bien. Pero el journey **mezcla un AS-IS (lo que pasa hoy) con un TO-BE parcial (lo que pasa en el sistema nuevo)** — la sección dice "AS-IS — parcialmente atendido por v1", lo cual es un híbrido confuso. UJ-2-TB en §7.7 sí es claramente TO-BE. UJ-3 debería tener su propio TO-BE (pasos del coordinador *en el sistema*) separado del AS-IS (lo que pasa fuera).

**Acción sugerida:** cerrar OQ-115 con su decisión, eliminar opciones (a) y (b) o moverlas a addendum como rechazadas. Reorganizar §7.6 como UJ-3 AS-IS (fuera del sistema) + UJ-3 TO-BE (en el sistema, parcial).

---

### F-09 [alto] — NFR-D-2 sigue mencionando "captura del jefe" y "conteo" — referencias muertas

**Cita NFR-D-2 (línea 391):** *"En caso de caída, el conteo y la captura del jefe deben poder reanudarse sin pérdida del trabajo previo (estado persiste server-side)."*

**Cita NFR-P-1 (línea 385):** *"Edición numérica en grilla del jefe / coordinador dispara recálculo optimista en cliente"*.

**Cita NFR-O-1 (línea 461):** *"Métricas de operación expuestas: tiempo de captura por jefe, tiempo de consolidación por coordinador..."*

**El problema:** el rewrite del 2026-05-24 eliminó al jefe del sistema (C-O7, §7.1 "Stakeholder externo — Jefe de servicentro... cero interacción"), pero **estos tres NFRs todavía mencionan al jefe como usuario del sistema**. Son referencias muertas que el rewrite no propagó. NFR-D-2 además habla del "conteo", que ya no existe (C-3 eliminada). NFR-O-1 propone medir "tiempo de captura por jefe" — métrica imposible cuando el jefe no captura.

Esto es ruido residual del rewrite. Para un PRD que está a punto de validarse, este tipo de referencia muerta **degrada la credibilidad del documento** — un lector que detecta una inconsistencia se pregunta qué más quedó sin propagar.

**Acción sugerida:** búsqueda y reemplazo de "jefe" en §9 NFRs:
- NFR-D-2: eliminar "conteo" y "captura del jefe", redactar como persistencia del trabajo del coordinador.
- NFR-P-1: eliminar "del jefe", dejar solo coordinador.
- NFR-O-1: eliminar "tiempo de captura por jefe".

Equivalente en cualquier otro lugar.

---

### F-10 [medio] — Out-of-scope §11.5 invoca FR-042 que fue eliminado

**Cita §11.5 (línea 578):** *"Casos raros de saturación se resuelven con override manual del jefe (FR-042)."*

**Cita FR-042 eliminado (línea 296):** *"~~FR-042~~ — Eliminado. El jefe no opera el sistema en v1, por lo tanto no aplica override del jefe. El override del coordinador se consolida con FR-064."*

**El problema:** §11.5 sigue refiriéndose a FR-042 como mecanismo de mitigación, pero FR-042 ya no existe en v1. Es referencia muerta. Si el caso raro de saturación de bodega se quiere resolver, ahora tendría que ser vía override del coordinador (FR-064) — pero el coordinador no sabe la capacidad física de la bodega, ese era el conocimiento del jefe. **El mecanismo invocado en §11.5 ya no puede activarse**.

**Acción sugerida:** o reformular §11.5 para reconocer que la mitigación de saturación de bodega ya no es vía override del jefe (porque el jefe no opera el sistema), o explicitar que el coordinador puede aplicar override por petición informal del jefe (vía WhatsApp/correo, mismo canal de excepciones de §11.15). Lo importante es que no quede una referencia a un FR que no existe.

---

### F-11 [medio] — "Sustituir el cálculo manual de la consolidación" en O1 ignora pasos críticos que siguen siendo manuales

**Cita O1 (línea 142):** *"Sustituir el cálculo manual de la consolidación distrital del coordinador... El día completo del coordinador se reduce a una sesión de revisión y aprobación."*

**Cita §1 Visión (línea 21):** *"El v1 reduce un día completo de consolidación a una sesión de revisión y aprobación"*.

**Cita UJ-2-TB §7.7 paso 4 (línea 244):** *"Coordinador revisa, aplica overrides con razón estructurada... aprueba tienda por tienda o en lote."*

**Cita §6 métrica principal:** *"Tiempo del coordinador por consolidación — Baseline: 1 día completo — Objetivo v1: < 2 h"*.

**El problema:** la "sesión de revisión y aprobación" no es un acto trivial cuando son **~113 tiendas con N SKUs cada una**. Si el motor sugiere razonablemente y el coordinador acepta el 80% de las sugerencias en un click ("aprobar en lote"), pero el 20% requiere intervención manual con razón estructurada (la métrica del propio PRD reconoce ≥30% de override esperado en ciclos 1-3), entonces:

- 113 tiendas × 30% override × N SKUs × tiempo de captura de razón estructurada = ¿cuánto tiempo realmente?
- Mas: revisión de las tiendas que no requieren override (¿el coordinador las mira de un vistazo o las aprueba a ciegas?).
- Mas: revisión del consolidado agregado, comparación con presupuesto, decisiones de TFS fuera del sistema, escritura de correo al comprador.

El cálculo "< 2 h" en la métrica es un compromiso ambicioso pero **el PRD no muestra el cálculo de cómo llega a ese número**. Es razonablemente plausible si el override en ciclo 4+ baja a <20%; en ciclos 1-3 con ≥30% override puede ser optimista. Y si no se cumple, el caso de ROI ("recuperar un día") se debilita inmediatamente.

**Acción sugerida:** o el PRD muestra el cálculo de los 2h en el addendum (con supuestos de tiempo por tienda, tasa de override, tiempo por razón estructurada), o reconoce que la métrica de tiempo es objetivo de estado estable (post-calibración) y que en ciclos 1-3 será mayor — lo cual es consistente con que los overrides 1-3 son "ORO de calibración".

---

### F-12 [medio] — FR-005 reintroduce calendario de guardia como input opcional — out-of-scope §11.14 lo prohíbe explícitamente

**Cita FR-005 (línea 262):** *"Mantener datos de coordinadores: identidad, distrito territorial asignado (lista de tiendas), credenciales, rol. Opcionalmente, calendario informativo de quién está de guardia esa semana (consumido como input, no gobernado por el sistema)."*

**Cita §11.14 (línea 614):** *"Rotación de guardia entre coordinadores — El sistema no la gobierna. Los coordinadores la organizan operativamente fuera del sistema. Sí puede consumirse como calendario informativo opcional configurado por admin (FR-005), pero no es una capacidad gobernada."*

**El problema:** §11.14 dice "el sistema NO la gobierna pero sí la consume opcional vía FR-005". FR-005 dice "calendario informativo de quién está de guardia esa semana (consumido como input)". Esto es out-of-scope que reentra por la puerta de atrás. **Si el sistema consume el calendario, lo está usando para algo** — probablemente para decidir a quién mostrarle la bandeja de guardia en C-7. Si lo usa, **es funcionalidad** (con UI para configurarlo, validaciones, modo de fallback cuando no esté configurado, comportamiento documentado). Decir "opcional, no gobernado" es una manera de tener la funcionalidad sin diseñarla.

Tres opciones honestas:
- **(a)** El sistema realmente no necesita el calendario — entonces eliminarlo de FR-005 y de §11.14, y que el coordinador de guardia simplemente acceda a la bandeja con sus credenciales (la bandeja muestra todo, él filtra mentalmente lo que le toca).
- **(b)** El sistema sí necesita el calendario para alguna decisión (¿enrutar notificaciones? ¿pre-cargar la bandeja del coordinador de guardia?) — entonces es FR completo, no "opcional", con su UI y validaciones.
- **(c)** Calendario es UX pasiva (decoración informativa) — entonces dejarlo explícito como tal.

Tal como está, el PRD tiene la cosa ambivalente y `architecture.md` va a tener que decidir sin guía.

**Acción sugerida:** elegir (a), (b) o (c) y declararlo.

---

### F-13 [medio] — D-coord-6 atenuado: el dolor del canal informal *no* se ataca en v1 — el PRD dice "fuera de scope" y luego incluye una solución parcial

**Cita D-coord-6 (línea 69):** *"El canal informal de excepciones (llamadas, WhatsApp, correo) no se prioriza ni se conserva. Una alerta urgente puede perderse en la avalancha. v1 no estructura el canal jefe→coordinador (sigue informal); sí estructura la decisión coordinador→comprador (C-10)."*

**Cita tabla D-exc-2 (línea 77):** *"D-exc-2 — No hay priorización entre múltiples alertas simultáneas. Tratamiento: Parcial — la bandeja del coordinador territorial muestra excepciones registradas con severidad y SKU, pero las alertas iniciales siguen entrando por canal informal."*

**El problema:** la avalancha de alertas urgentes (D-coord-6) y la falta de priorización (D-exc-2) son problemas que ocurren **antes de que la excepción llegue al sistema**. La "priorización" en la bandeja del coordinador opera solo sobre lo que el coordinador ya registró, que es necesariamente lo que él decidió priorizar (porque tuvo que decidir manualmente entrar a la bandeja a capturarla). **Es priorización post-priorización humana.** El dolor real (que llegue una alerta urgente y se pierda en WhatsApp) no se ataca.

Esto no es un bug, es una decisión consciente del scope cut (D-2). Pero el PRD podría ser más limpio diciendo: "D-coord-6 y D-exc-2 son **historiados** en v1, no atendidos". El lenguaje "estructura la decisión" sugiere que el sistema agrega valor cuando, en términos de los dolores listados, el valor es solo post-hoc (registro y trazabilidad, no priorización en línea).

**Acción sugerida:** ajustar texto en tablas D-coord y D-exc para distinguir "atendido" (mecanismo en línea), "parcial" (cubierto post-hoc), e "historiado" (registro/trazabilidad sin atención al evento en sí).

---

### F-14 [medio] — Métricas con `[OPEN]` no son métricas: TFS, mermas y MAPE quedaron sin números

**Cita §6 tabla:**
- *"Tasa de quiebre — Baseline: `[OPEN: baseline a recolectar en pilotaje]` — Objetivo: Reducción ≥ 80%"*
- *"Sobre-inventario — Baseline: `[OPEN: baseline]` — Objetivo: Por tienda, mantener en rango sano (a definir con Finanzas)"*
- *"TFS registradas — Objetivo: `[OPEN: meta v1]`"*
- *"MAPE/WAPE — Objetivo: `[OPEN: acordar con Finanzas en backtesting]`"*
- *"Mermas detectadas — Objetivo: `[OPEN: meta v1]`"*

**El problema:** **5 de las 7 métricas de éxito tienen `[OPEN]`** en baseline o objetivo. Si Finanzas o sponsor preguntan "¿cuándo sabemos que v1 fue exitoso?", el PRD responde "cuando acordemos eso después". Esto no es métrica de éxito — es lista de cosas que vamos a medir, sin acuerdo sobre qué constituye éxito.

La métrica del tiempo del coordinador (<2h) y la tasa de override (≥30% en ciclos 1-3, <20% en ciclo 4+) son las únicas con números concretos, y la segunda tiene variable libre (F-07).

**Esto está acoplado con R-7** (acuerdo con Finanzas) y **R-9** (baseline cuantitativo), ambas en los bloqueos del Reviewer Gate. Pero el PRD presenta la tabla de métricas como si fuera un compromiso, cuando es más bien un compromiso a establecer compromisos.

**Acción sugerida:** o se completan los `[OPEN]` antes de Validate (con la sesión de Finanzas R-7 y la recolección de baseline R-9), o el PRD reconoce que §6 es "framework de métricas a calibrar en piloto" y no afirma metas. La actual mezcla — algunas con número, otras con `[OPEN]` — es lo peor de ambos mundos.

---

### F-15 [medio] — "Contra-métricas" listadas sin umbrales — no son métricas, son hipótesis

**Cita §6 contra-métricas:**
- *"Tasa de override > 50% por ciclo en alguna tienda después de la fase de calibración"* — esto sí tiene número.
- *"Override estancado bajo X% durante los ciclos 1-3"* — X sin valor (F-07).
- *"Excepciones mid-cycle creciendo después del lanzamiento"* — sin umbral.
- *"Pedidos urgentes al proveedor creciendo"* — sin umbral.
- *"`SolicitusDeInsumosTodos.xlsx` exportado pero el comprador sigue pidiendo correcciones"* — binaria, OK.
- *"Reducción del tiempo del coordinador pero crecimiento de quiebres"* — sin umbral de "crecimiento".
- *"Divergencia entre saldo SIM y consumo real reportado por venta crece"* — sin umbral.

**El problema:** las contra-métricas como "creciendo" o "crece" sin umbral son **hipótesis cualitativas**, no contra-métricas. ¿Crece 5%? ¿20%? ¿Comparado contra qué baseline? Si esto es lo que se monitorea para detectar que la coexistencia (C) se está rompiendo (R-11) o que el motor está fallando, **el evento de alarma queda librado al juicio del operador**, no hay disparo automático.

**Acción sugerida:** o se proponen umbrales iniciales (`>10% mes-sobre-mes` por ejemplo), o se acepta que las contra-métricas son "observables a calibrar" y se mueven a un anexo para no inflar §6.

---

### F-16 [bajo] — Glosario: "Override" excluye al jefe pero deja la definición ambigua para administradores

**Cita §14 Glosario:** *"Override: modificación manual del coordinador (no del jefe — el jefe no opera el sistema en v1) sobre una cantidad sugerida por el sistema..."*

**El problema:** la definición es correcta pero tangencial. ¿Un administrador (NFR-S-2) puede aplicar override? El PRD no lo dice. El glosario solo aclara quién NO lo hace (el jefe), no quién SÍ además del coordinador. Si el administrador modifica una cantidad sugerida con razón estructurada, ¿es override? Probablemente sí, pero el glosario no lo confirma.

**Acción sugerida:** expandir definición a "modificación manual del coordinador o administrador (no del jefe)" si el administrador puede hacer overrides. Si no puede, decirlo.

---

### F-17 [bajo] — "Excepción mid-cycle" en glosario y §7.6 incluyen "inferencia propia" del coordinador como origen del aviso — ese origen no aparece en ningún flujo

**Cita FR-091 (línea 346):** *"origen del aviso (WhatsApp/correo/llamada/inferencia propia)"*.

**El problema:** "inferencia propia" del coordinador sugiere que el coordinador puede detectar una excepción mirando datos (¿desde dónde? ¿qué tablero?) y registrarla. Pero **el PRD no especifica qué tablero usa el coordinador para hacer esa inferencia**. En la bandeja territorial (§7.2 modo b), ¿hay algún dashboard que surface tiendas con consumo anómalo, saldo SIM bajo, o ventas pico? Si sí, no está en ningún FR. Si no, "inferencia propia" significa que el coordinador adivina con qué información tiene en su cabeza — lo cual no es muy distinto del statu quo.

OQ-115 menciona la opción (a) detección automática por patrón, pero D-2 escogió la opción (c) registro manual cuando llega aviso informal. Si "inferencia propia" se admite como origen, ¿no es eso un pedacito de (b) detección manual desde dashboard, que debería tener su propio FR de visualización?

**Acción sugerida:** o se elimina "inferencia propia" de los orígenes admitidos (entonces la excepción siempre viene de aviso informal externo), o se añade un FR pequeño en C-7 o C-10 que diga cómo el coordinador inspecciona su territorio para inferir excepciones (qué columnas/filtros del dashboard).

---

### F-18 [bajo] — Catálogo de Tienda en FR-001 incluye atributos cuya fuente está marcada `[OPEN]`

**Cita FR-001 (línea 258):** *"Mantener catálogo de tiendas con: ID (String), nombre, distrito, coordinador territorial asignado, formato (Express / Estándar), ciclo quincenal asignado (semana 1-y-3 o 2-y-4), techo presupuestal semanal MXN, merma esperada mensual configurable (MXN o piezas, por confirmar — alimenta C-4)."*

**El problema:** *"MXN o piezas, por confirmar"* es decisión técnica abierta dentro de un FR. **Si es MXN, la merma absorbe tanto cambios de precio como cambios de unidades; si es piezas, solo de unidades pero requiere conversión cuando se aplica al cálculo de costo total.** Esto afecta directamente cómo se aplica el factor en FR-035 y cómo se valida contra presupuesto en C-6. No es trivia.

Adicionalmente, OQ-104 sigue abierta sobre si "formato (Express/Estándar)" está en `Presupuesto-tiendas.csv` o requiere data master nueva. FR-001 lo lista como atributo del catálogo asumiendo que estará disponible.

**Acción sugerida:** cerrar la decisión MXN-vs-piezas antes de Validate (es decisión de pocos minutos con coordinadores). Para OQ-104, inspeccionar `Presupuesto-tiendas.csv` (está en `docs/`) y cerrar.

---

### F-19 [bajo] — `addendum.md` A2 cita la objeción de Winston con matiz que el cuerpo del PRD no aplica

**Cita addendum.md A2 (línea 67):** *"válida en abstracto pero atenuada en este caso específico porque el jefe NO captura en el sistema nuevo (clarificación C-O7). La divergencia que sí existe es entre SIM contable y plan operativo del coordinador, monitoreada por R-11 y mitigada por SLA de frescura del snapshot (NFR-DAT-4, OQ-117)."*

**El problema:** el addendum dice que NFR-DAT-4 **mitiga** la divergencia. El cuerpo del PRD §10.4 dice que R-11 **monitorea** la divergencia (no la elimina). NFR-DAT-4 SLA de 48h con `[OPEN]` no es mitigación de divergencia — es threshold para rechazar operar sobre snapshot viejo. Si el snapshot está fresco pero el jefe capturó mal en SIM las 48h anteriores, NFR-DAT-4 no aplica y la divergencia persiste. El addendum sobreestima ligeramente lo que NFR-DAT-4 entrega.

**Acción sugerida:** alinear el addendum con el cuerpo — la SLA de frescura controla obsolescencia del snapshot, no divergencia por captura sucia. La divergencia por captura sucia se mitiga por el factor de merma esperada (F-02) y se detecta por contra-métrica.

---

### F-20 [bajo] — "Día completo del coordinador" en §1 y §5 se cita como ≈52 días/año, pero §3 D-coord-1 lo computa también como 52 — los números no aportan independientemente

**Cita §1 Visión (línea 21):** *"~52 días-hombre/año entre los tres"*.

**Cita D-coord-1 (línea 64):** *"~52 días-hombre/año entre los tres coordinadores"*.

**Cita §4 cuenta dura (línea 130):** *"~52 días-hombre/año"*.

**El problema:** la cifra ~52 días-hombre/año se repite literalmente tres veces. Es coherente, pero **se presenta cada vez como si fuera dato duro independiente** cuando es el mismo cálculo: 52 semanas × 1 día. No es robustez, es eco. Si alguien cuestiona el número una vez (¿realmente las 52 semanas tienen consolidación, considerando vacaciones, festivos, semanas sin ciclo?), las tres citas caen juntas.

Un cálculo más honesto sería: ~52 ciclos/año (1 ciclo/semana en promedio) × ~1 día/ciclo × tasa de realización (probablemente ~85-90% dado vacaciones/festivos) ≈ ~44-47 días-hombre/año efectivos. La cifra ~52 es techo, no realización.

**Acción sugerida:** o se cita con margen ("hasta 52 días-hombre/año, ~44 días efectivos descontando vacaciones"), o se reconoce que es aproximación.

---

## Cierre del review

El PRD es producto de **mucho trabajo bien hecho** — el rewrite del 2026-05-24 corrigió cosas reales del draft inicial y la documentación de decisiones es ejemplar (decision log + addendum + change signal). El owner ha demostrado disposición a aceptar trade-offs duros (concesión epistémica de §11.16 firmada, scope cut del jefe, override híbrido como ORO de calibración).

**Pero el documento todavía mezcla aspiraciones con especificaciones, y eso es lo que un revisor de Validate tiene que llamar.** Las seis reviewer gates declaradas (R-4, R-7, R-8, NFR-E-3, R-9, comprador piloto) **siguen siendo bloqueos reales**: el cuerpo principal escribe como si las hubiera resuelto (resumen ejecutivo, O1, métricas), pero el cierre §799 lo desmiente. La discrepancia entre cuerpo y cierre es lo que hace que este PRD no pueda pasar de `draft` a `final` todavía.

**Recomendación al owner:** antes de invocar `bmad-prd validate` para Reviewer Gate completo, atender al menos:

1. **Inspeccionar `docs/SolicitusDeInsumosTodos.xlsx` ahora** (está ahí, es trabajo de minutos) y cerrar R-4/OQ-102 + ajustar el lenguaje del resumen ejecutivo (F-01).
2. **Cerrar OQ-113** (origen del histórico de mermas). Sin esto, la mitigación principal de R-1 es vapor (F-02).
3. **Cuantificar X de la métrica de override** y NFR-E-3 (F-06, F-07).
4. **Propagar el rewrite a NFRs y §11.5** para eliminar referencias muertas al jefe / a FR-042 / al conteo (F-09, F-10).
5. **Resolver la ambivalencia FR-005 / §11.14** sobre el calendario opcional de guardia (F-12).
6. **Decidir MXN-vs-piezas** para la merma esperada (F-18).

El resto de los hallazgos son edits de superficie (eco numérico, glosario, contra-métricas con umbrales) y se pueden absorber en el ciclo de polish editorial del Reviewer Gate. Pero los seis bloqueos arriba más los bloqueos ya declarados (R-7, R-8, R-9, comprador piloto) tienen que estar resueltos.

---

**Fin de la revisión adversarial.**
