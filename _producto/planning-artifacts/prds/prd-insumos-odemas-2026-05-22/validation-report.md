# Validation Report — insumos-odemas

- **PRD:** `_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md`
- **Rubric:** `.claude/skills/bmad-prd/assets/prd-validation-checklist.md`
- **Run at:** 2026-05-26T21:35:40-06:00
- **Grade:** Fair
- **Revisores:** rúbrica de calidad (7 dimensiones)

## Overall verdict

PRD inusualmente honesto y bien argumentado para su etapa: la tesis (*automatizar la consolidación distrital del coordinador, no atacar la causa raíz del dato sucio*) está nombrada, las concesiones epistémicas están expuestas sin maquillaje (§10.4, §11.16, R-1, R-11 reconocen textualmente que el modelo (C) "vigila pero no previene" la divergencia que Winston nombró), y la reversa D-9 al factor único de pérdida está bien propagada en el cuerpo principal. Cuatro de siete dimensiones son **strong** (Decision-readiness, Substance, Strategic coherence, Shape fit), pero la calificación queda en **Fair** por tres hallazgos high.

Lo que está en riesgo es la **done-ness de los FR del motor** (FR-030..035, FR-040 mezclan condición verificable con decisiones diferidas a `architecture.md` sin criterio de aceptación propio) y la **scope honesty de cara a un go-live**: FR-066 paso 4 es un Open Question disfrazado de FR cerrado, y la densidad de open-items sobre dependencias de datos no confirmadas (D-8/D-10, OQ-124/OQ-125) sigue alta para un PRD que pretende destrabar arquitectura. Los rastros del modelo dual viejo en glosario/ejemplos son cosméticos pero erosionan confianza en un chain-top. Es un `draft` sólido que aún no debe leerse como green-light.

*Comparación con el Validate del 2026-05-26 previo (Grade Fair, 0C/3H/7M/2L): el perfil de hallazgos mejoró (ahora 0C/3H/5M/1L) pero la calificación sigue gateada por los mismos 3 hallazgos high.*

## Dimension verdicts

- Decision-readiness — **strong**
- Substance over theater — **strong**
- Strategic coherence — **strong**
- Done-ness clarity — **thin**
- Scope honesty — **adequate**
- Downstream usability — **adequate**
- Shape fit — **strong**

## Findings by severity

### Critical (0)

_Ninguno._

### High (3)

**[Done-ness clarity]** — FR-035 / FR-034 delegan la mecánica del factor a arquitectura sin criterio de aceptación propio (§8.4, FR-034/FR-035)
"La opción técnica concreta de cómo se aplica es decisión de `architecture.md` (F-05)". El motor de pérdida es central a O4 y el propio PRD dice que la elección "afecta R-7 (MAPE)"; sin acotar el modo de aplicación, una historia downstream no tiene condición testeable.
Fix: dejar que arquitectura elija la implementación, pero fijar en el FR la consecuencia observable (p. ej. "para una tienda con venta esperada V y factor f, la demanda ajustada cumple `demanda = V × (1+f)` antes de cuantización; el trazo declara V, f, nivel y resultado"), de modo que el criterio de aceptación exista independiente del método interno.

**[Done-ness clarity]** — FR-030 no acota "done" del forecast (§8.4, FR-030)
"modelo estacional configurable (familia de modelos en Java puro…)" no define qué resultado verificar ni cómo se comporta con tiendas de < 12 meses (abierto en OQ-108). Es el FR más cargado de valor y el menos verificable.
Fix: añadir criterio de aceptación basado en el backtesting (NFR-T-5) — "el forecast produce una cantidad por tienda × SKU × ciclo cuya desviación vs YoY queda dentro del criterio acordado (R-7)" — y resolver o referir explícitamente el caso de tienda nueva como precondición de done.

**[Downstream usability]** — Glosario de "Coordinador" contradice el modelo de rol del cuerpo (§14 glosario vs §2/§7.2)
El glosario dice "rota guardia semanal para consolidación"; el cuerpo insiste en asignación fija y consolidación no rotada ("Cada coordinador trabaja su distrito todas las semanas… la guardia rotativa aplica solo a tareas inter-distritales"). El glosario es la fuente que UX/arquitectura source-extraen — un downstream lo modelaría mal.
Fix: reescribir la entrada: "supervisa un distrito fijo asignado territorialmente; consolida su distrito todas las semanas; la guardia rotativa aplica solo a tareas inter-distritales y no la gobierna el sistema".

### Medium (5)

**[Strategic coherence]** — Objetivos O2/O4 sin métrica cerrada mientras O1 sí (§6)
La métrica de tiempo del coordinador está cerrada (baseline 9h, meta <2h), pero quiebre, sobre-inventario, MAPE/WAPE y candidatos atípicos siguen `[OPEN: baseline a recolectar en pilotaje]`. Para un PRD con impacto financiero, dos de cuatro objetivos no son verificables hoy.
Fix: aceptar explícitamente que O1 es el objetivo gateado para v1 y O2/O4 son objetivos de pilotaje-a-confirmar, o cerrar al menos una meta direccional (p. ej. rango de cobertura sano) con Finanzas antes de subir a `final`.

**[Done-ness clarity]** — FR-066 paso 4 es una regla financiera con done condicional (§8.7, FR-066)
La fórmula de presupuesto en carry-over está marcada `[CONFIRMAR CON OWNER]`; su criterio de aceptación depende de una decisión abierta. Es una regla con efecto en MXN presentada como FR "cerrado" sin firma.
Fix: extraer la regla a un Open Question (OQ nueva) hasta confirmarla, o confirmarla y quitar el marcador; no dejar una regla con efecto en MXN como FR cerrado sin firma.

**[Scope honesty]** — Densidad de open-items sobre fuentes de datos vs. pretensión de destrabar arquitectura (§10.6, R-12, OQ-124/OQ-125)
El pilar de cadencia semanal (D-8) y el cálculo del pedido con tránsito (D-10) dependen de extracts RMS/BigQuery "a confirmar con Fernando", aún sin dueño/formato/SLA. Está marcado honestamente, pero es una dependencia de datos no confirmada bajo una decisión presentada como firmada.
Fix: elevar la confirmación de la fuente semanal a precondición explícita de `architecture.md` y marcar D-8/D-10 como "firmadas condicionalmente a OQ-124/OQ-125"; documentar el umbral de error del fallback de prorrateo para que arquitectura sepa cuándo es inaceptable.

**[Downstream usability]** — FR-101 ejemplo de bitácora incluye al "Jefe" como actor (§8.11, FR-101)
El ejemplo "`original modelo X → Jefe Carlos 07:48: Y → Coord. Marco 09:12: aprobado`" enseña una cadena imposible: el jefe no opera el sistema en v1 (§7.1, FR-090 eliminado). Puede inducir un modelo de datos con actor "jefe".
Fix: cambiar el ejemplo a actores reales de v1, p. ej. "`original modelo X → Coord. Marco 09:12: Y (override, razón) → aprobado`".

**[Downstream usability]** — SLA de frescura del snapshot SIM con dos valores (§12 R-11 mit. 1 = 48h vs §9.6 NFR-DAT-4 / OQ-117 = 36h)
Número contradictorio para el mismo control: R-11 dice "propuesta inicial 48h máximo" mientras NFR-DAT-4 y OQ-117 dicen 36h (24h + 12h buffer).
Fix: alinear R-11 a 36h (el valor endurecido y cerrado en NFR-DAT-4) y eliminar la "propuesta inicial 48h".

### Low (1)

**[Scope honesty]** — Open-item disfrazado de FR cerrado (§8.7, FR-066 paso 4)
Ya señalado en Done-ness; cuenta también como ruido de scope honesty porque un open-item incrustado como FR (en vez de declarado como OQ) infla la apariencia de cierre y el conteo de FR "cerrados".
Fix: convertir a OQ hasta confirmación del owner.

## Mechanical notes

- **Drift de glosario (modelo dual → factor único):** el término "factor de merma esperada" (lenguaje del modelo dual) sobrevive en D-jefe-2 (línea 94), C-3 §8.3 (línea 289), §11.12 (línea 649) y OQ-114 opción (b) (línea 825), mientras el cuerpo nuevo usa "factor único de pérdida". OQ-114 opción (b) propone "continuar usando factor de merma esperada como compensación" — terminología pre-D-9. Normalizar a "factor único de pérdida".
- **Continuidad de IDs OQ:** OQ-121 y OQ-122 aparecen después de OQ-126 (líneas 872 y 876), fuera de orden numérico. No rompe referencias (ambas cerradas/superadas) pero desordena la lectura. Reubicar tras OQ-120 o anotar el salto.
- **Cross-ref a hallazgos internos (F-NN, C-O-NN):** el PRD referencia "F-05", "F-Done-1/2", "F-16/F-17/F-18", "C-O2..C-O40" como identificadores estables, pero no hay índice que los resuelva dentro del PRD ni del addendum. Downstream no puede dereferenciarlos. Añadir un mini-índice o convertirlos en prosa.
- **Roundtrip de Assumptions:** los `[ASSUMPTION]` están inline (OQ-101, baselines TFS §4, ventana de excepciones §2) pero no hay un Índice de Assumptions consolidado al final como la rúbrica espera. Para un chain-top conviene. Severidad baja.
- **D-IDs sin tabla de decisiones:** D-1, D-2, D-5..D-12 se citan por todo el documento (decisiones firmadas) pero no hay tabla única que liste cada D-NN con su enunciado; están dispersas en notas de FR y en el cierre §14. Un Decision Log tabular ayudaría a downstream.
- **Continuidad FR:** los rangos eliminados (FR-004, 015, 020-025, 042, 062, 070-072, 090) están todos tachados con razón inline — buena higiene. No se detectaron duplicados ni FR huérfanos referenciados.
- **Secciones requeridas:** presentes y completas para los stakes (Visión, Contexto, Problema, Objetivos, Métricas con contra-métricas, Personas/UJ, FR por capacidad, NFR, Contexto técnico, Out-of-scope, Riesgos, Open Questions, Glosario).

## Reviewer files

- `review-rubric.md`
