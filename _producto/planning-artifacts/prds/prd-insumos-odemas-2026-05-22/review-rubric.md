# PRD Quality Review — insumos-odemas

## Overall verdict

Este es un PRD inusualmente honesto y bien argumentado para su etapa: la tesis (automatizar la consolidación distrital del coordinador, no atacar la causa raíz del dato sucio) está nombrada, las concesiones epistémicas están expuestas sin maquillaje (§10.4, §11.16, R-1, R-11 reconocen textualmente que el modelo (C) "vigila pero no previene" la divergencia que Winston nombró), y la reversa D-9 al factor único de pérdida está bien propagada en el cuerpo principal. Lo que está en riesgo es la **done-ness de los FR del motor** (FR-030..035, FR-040 mezclan condición verificable con decisiones diferidas a `architecture.md` sin criterio de aceptación propio) y la **scope honesty de cara a un go-live**: FR-066 paso 4 es un Open Question disfrazado de FR cerrado, y la densidad de open-items sobre dependencias de datos no confirmadas (D-8/D-10, OQ-124/OQ-125) sigue alta para un PRD que pretende destrabar arquitectura. Es un `draft` sólido que aún no debe leerse como green-light; los rastros del modelo dual viejo en glosario/ejemplos son cosméticos pero erosionan confianza en un chain-top.

## Decision-readiness — strong

El PRD trata sus decisiones como decisiones, no como "consideraciones". La concesión epistémica central de SIM está expuesta con una franqueza notable: §10.4 cita la objeción de Winston ("(C) coexistencia con doble captura es garantía de divergencia — dos sistemas, dos verdades, ningún ganador") y, en lugar de neutralizarla, el párrafo "Honestidad epistémica (post-F-03)" reconoce textualmente que "no es un riesgo distinto con nombre nuevo — es la misma objeción de Winston con etiqueta operacional" y que "ninguna [mitigación] es engineered mitigation… el modelo (C) no elimina la divergencia… la vigila". Eso es exactamente lo que pide esta dimensión: el trade-off nombrado con lo que se cede, no solo lo que se elige. R-1 y R-11 refuerzan la misma postura sin suavizarla.

Las contra-métricas de §6 (divergencia SIM vs venta > 10% MoM, override > 50% post-calibración, pedidos urgentes > 15% MoM) son señales reales de fallo, no adornos. Los trade-offs de scope (jefe fuera de v1, TFS solo histórico, conteo asistido eliminado) están enmarcados con el costo aceptado, no como ganancias gratis.

El único punto que baja la nota de "perfecto" a "strong" sólido: FR-066 paso 4 introduce una regla de negocio con consecuencia financiera real (techo no acumula en carry-over) marcada `[CONFIRMAR CON OWNER]` — una decisión todavía abierta presentada dentro de un FR redactado como cerrado (ver Scope honesty).

## Substance over theater — strong

Poco relleno. Las tres personas (§7) son exactamente las que el producto necesita y cada una está justificada por su relación con el sistema, no por completitud: el coordinador es usuario interactivo, el comprador (Hugo) es consumidor downstream con función concreta (firma golden datasets, NFR-T-2), y el jefe de servicentro está marcado explícitamente como "**no es usuario del sistema en v1**" con justificación de por qué se documenta igual ("informar v2", "que el equipo de UX no pierda la perspectiva del extremo del flujo"). Esa etiqueta convierte lo que sería persona-theater en contexto deliberado y honesto — ver Shape fit.

§4 "Por qué no Excel" es sustancia ganada, no furniture: distingue lo que Excel sí puede ("cálculo aritmético para una tienda", "tabla pivote para consolidar") de lo que rompe a escala (validación de 38 tiendas × N SKUs, trazabilidad, fail-loud), con la cuenta dura del statu quo. Los NFR llevan umbrales product-specific (NFR-E-4 bandeja < 3s; NFR-P-1 recálculo < 100 ms con `debounceTime(150)`; NFR-DAT-4 frescura del snapshot), no boilerplate de "el sistema debe ser escalable".

## Strategic coherence — strong

Hay tesis explícita y las features la sirven. La tesis: "El dolor v1 vive en el coordinador" (§3), y todo el cuerpo se ordena alrededor de eso — los objetivos O1-O4 están en orden de prioridad declarado, las capacidades C-1..C-12 mapean a dolores D-coord-* nombrados por código, y los dolores fuera de alcance (jefe) se aíslan en §3-bis con el argumento de que se atienden "solo indirectamente vía mejor forecast del coordinador". El MVP es coherentemente de tipo "problem-solving" (sustituir cálculo manual), no un catálogo de capacidades.

Las métricas principales validan la tesis, no actividad: la métrica #1 es tiempo del coordinador (9h → <2h), que es precisamente lo que la tesis promete. El encuadre de 3 fases (Predicción / Inventario-solicitud / Rebajas-merma) da un arco de producto con v1 = Fase 1 claramente delimitado.

El riesgo de coherencia que sí existe es menor pero real: §6 marca como `[OPEN]` los baselines de quiebre y sobre-inventario (O2) y el MAPE/WAPE (acoplado a R-7). La tesis financiera ("eliminar quiebres y sobre-inventario") tiene métricas declaradas pero sin baseline ni meta numérica cerrada — la tesis de tiempo está blindada, la tesis de inventario todavía no. Es honesto (lo marca), pero significa que dos de los cuatro objetivos no son aún medibles.

### Findings
- **medium** Objetivos O2/O4 sin métrica cerrada mientras O1 sí (§6) — La métrica de tiempo del coordinador está cerrada (baseline 9h, meta <2h), pero quiebre, sobre-inventario, MAPE/WAPE y candidatos atípicos siguen `[OPEN: baseline a recolectar en pilotaje]`. Para un PRD con impacto financiero, dos de cuatro objetivos no son verificables hoy. *Fix:* aceptar explícitamente que O1 es el objetivo gateado para v1 y O2/O4 son objetivos de pilotaje-a-confirmar, o cerrar al menos una meta direccional (p. ej. rango de cobertura sano) con Finanzas antes de subir a `final`.

## Done-ness clarity — thin

Esta es la dimensión más débil y la que más importa para el chain-top hacia historias. El problema no es vaguedad de adjetivos sueltos (el PRD evita en general "razonable"/"amigable"), sino que **varios FR del corazón del producto delegan su consecuencia verificable a `architecture.md` sin dejar un criterio de aceptación propio**.

FR-035 y FR-034 son el caso más claro: ambos dicen que el factor de pérdida se aplica "como ajuste porcentual sobre la demanda de venta" pero cierran con "La opción técnica concreta de cómo se aplica es decisión de `architecture.md` (F-05)". Un ingeniero de historias no puede escribir un criterio de aceptación testeable para FR-035 sin saber si el 0.9% multiplica el output, ajusta el histórico, o entra como término aditivo — y el PRD nota que esa elección "afecta R-7 (MAPE)". FR-030 ("modelo estacional configurable (familia de modelos en Java puro definida en `project-context.md`)") no acota qué cuenta como "done": ¿qué modelo, qué error tolerado, qué pasa con tiendas < 12 meses (eso vive abierto en OQ-108)?

En contraste, FR-040, FR-051 y FR-054 sí están bien aterrizados: FR-040 da la fórmula completa de cantidad sugerida con cada término definido y resuelve `buffer_seguridad` (1 semana de cobertura, configurable); FR-051 operacionaliza "riesgo de quiebre" como "cobertura post-recorte ≥ 2 semanas" con orden de priorización; FR-054 da la fórmula `techo − compras_esporádicas`. Estos demuestran que el autor sabe escribir un FR con consecuencia verificable — por eso la inconsistencia con FR-030..035 destaca.

FR-066 paso 4 falla en done-ness por otra vía: la regla está enunciada con fórmula (`presupuesto_disponible_N+1 = techo_semanal_N+1 − compras_esporádicas`) pero etiquetada `[CONFIRMAR CON OWNER]`, así que su "done" es condicional a una decisión no tomada.

### Findings
- **high** FR-035 / FR-034 delegan la mecánica del factor a arquitectura sin criterio de aceptación propio (§8.4, FR-034/FR-035) — "La opción técnica concreta de cómo se aplica es decisión de `architecture.md` (F-05)". El motor de pérdida es central a O4 y el propio PRD dice que la elección "afecta R-7 (MAPE)"; sin acotar el modo de aplicación, una historia downstream no tiene condición testeable. *Fix:* dejar que arquitectura elija la implementación, pero fijar en el FR la consecuencia observable (p. ej. "para una tienda con venta esperada V y factor f, la demanda ajustada cumple `demanda = V × (1+f)` antes de cuantización; el trazo declara V, f, nivel y resultado"), de modo que el criterio de aceptación exista independiente del método interno.
- **high** FR-030 no acota "done" del forecast (§8.4, FR-030) — "modelo estacional configurable (familia de modelos en Java puro…)" no define qué resultado verificar ni cómo se comporta con tiendas de < 12 meses (abierto en OQ-108). Es el FR más cargado de valor y el menos verificable. *Fix:* añadir criterio de aceptación basado en el backtesting (NFR-T-5) — "el forecast produce una cantidad por tienda × SKU × ciclo cuya desviación vs YoY queda dentro del criterio acordado (R-7)" — y resolver o referir explícitamente el caso de tienda nueva como precondición de done.
- **medium** FR-066 paso 4 es una regla financiera con done condicional (§8.7, FR-066) — la fórmula de presupuesto en carry-over está marcada `[CONFIRMAR CON OWNER]`; su criterio de aceptación depende de una decisión abierta. *Fix:* extraer la regla a un Open Question (OQ nueva) hasta confirmarla, o confirmarla y quitar el marcador; no dejar una regla con efecto en MXN como FR "cerrado" sin firma.

## Scope honesty — adequate

La sección Out-of-scope (§11) hace trabajo real: 17 exclusiones, cada una justificada y muchas con su destino futuro (v1.1 / v2) y su FR-eliminado trazado (FR-004, FR-015, FR-020..025, FR-042, FR-062, FR-070..072, FR-090, NFR-E-2, NFR-DAT-2). Los `[ASSUMPTION]` están presentes donde corresponde (frecuencia de excepciones OQ-101, baselines de TFS, etc.). Los tachados de FR/NFR eliminados con razón inline son ejemplares para downstream — no hay des-scoping silencioso.

Lo que baja la nota de `strong` a `adequate` es la **densidad de open-items relativa a las pretensiones del documento**. El PRD es `draft` y lo dice, pero el cierre (§14, líneas finales) lista como "bloqueos externos restantes" el spike Java (R-8), backtesting+baselines (R-9), sesión Finanzas (R-7) y la confirmación de fuentes RMS/BigQuery (R-12), y mantiene abiertas OQ-107/108/110/111/112/114/116/123/124/125/126. Para la dimensión donde más pesa — las **fuentes de datos productivas** — D-8 (ventas semanales) y D-10 (ALLOC/TFS) son decisiones firmadas que *dependen* de extracts no confirmados ("a confirmar con Fernando"). El PRD es honesto al marcarlo (R-12, OQ-124/OQ-125) pero el efecto neto es que la cadencia semanal —pilar del motor— se apoya en un input cuya existencia no está confirmada, con fallback de prorrateo "con error". Eso es scope honesty bien ejecutada en la forma, pero es un nivel de incertidumbre de datos que un lector debe entender como bloqueante de arquitectura, no como detalle.

El FR-066 paso 4 también cuenta aquí: un open-item incrustado como FR en vez de declarado como OQ infla la apariencia de cierre.

### Findings
- **medium** Densidad de open-items sobre fuentes de datos vs. pretensión de destrabar arquitectura (§10.6, R-12, OQ-124/OQ-125) — el pilar de cadencia semanal (D-8) y el cálculo del pedido con tránsito (D-10) dependen de extracts RMS/BigQuery "a confirmar con Fernando", aún sin dueño/formato/SLA. Está marcado honestamente, pero es una dependencia de datos no confirmada bajo una decisión presentada como firmada. *Fix:* elevar la confirmación de la fuente semanal a precondición explícita de `architecture.md` y marcar D-8/D-10 como "firmadas condicionalmente a OQ-124/OQ-125"; documentar el umbral de error del fallback de prorrateo para que arquitectura sepa cuándo es inaceptable.
- **low** Open-item disfrazado de FR cerrado (§8.7, FR-066 paso 4) — ya señalado en Done-ness; cuenta también como ruido de scope honesty porque infla el conteo de FR "cerrados". *Fix:* convertir a OQ hasta confirmación del owner.

## Downstream usability — adequate

Para un chain-top esto importa, y el PRD lo soporta razonablemente. Hay glosario (§14) extenso; los IDs FR son estables y referenciados desde dolores, UJs y riesgos; los UJ nombran personas de §7 por etiqueta (UJ-1 jefe, UJ-2 coordinador, UJ-3 excepción). Las capacidades C-1..C-12 dan agrupación limpia para epics. El export (FR-110) declara contrato literal contra `SolicitusDeInsumosTodos.xlsx` y NFR-T-2/T-4 fijan fixture y contract testing — buen anclaje para historias.

Lo que erosiona la usabilidad downstream son **rastros del modelo viejo y un par de inconsistencias internas** que un extractor automático arrastraría como verdad:

1. El glosario de "Coordinador (de distrito)" (línea 884) todavía dice "rota guardia semanal para consolidación" — esto **contradice directamente** §2 y §7.2, que insisten en que la asignación es fija y la consolidación NO se rota ("Cada coordinador trabaja su distrito todas las semanas… la guardia rotativa aplica solo a tareas inter-distritales"). Un downstream que lea el glosario como autoridad modelará mal el rol.
2. FR-101 muestra como ejemplo de cadena de bitácora "`original modelo X → Jefe Carlos 07:48: Y → Coord. Marco 09:12: aprobado`" — pero el jefe **no opera el sistema** (§7.1, FR-090 eliminado, glosario de Override), así que no puede ser actor en la bitácora. El ejemplo enseña un flujo imposible en v1.
3. R-11 mitigación #1 dice "SLA de frescura… propuesta inicial **48h** máximo" mientras NFR-DAT-4 y OQ-117 dicen **36h** (24h + 12h buffer). Dos números para el mismo SLA.

Ninguno bloquea por sí solo, pero en un chain-top de alto riesgo los tres son trampas para el extractor.

### Findings
- **high** Glosario de "Coordinador" contradice el modelo de rol del cuerpo (§14 glosario vs §2/§7.2) — el glosario dice "rota guardia semanal para consolidación"; el cuerpo dice asignación fija y consolidación no rotada. El glosario es la fuente que UX/arquitectura source-extraen. *Fix:* reescribir la entrada: "supervisa un distrito fijo asignado territorialmente; consolida su distrito todas las semanas; la guardia rotativa aplica solo a tareas inter-distritales y no la gobierna el sistema".
- **medium** FR-101 ejemplo de bitácora incluye al "Jefe" como actor (§8.11, FR-101) — el jefe no opera el sistema en v1; el ejemplo enseña una cadena imposible y puede inducir un modelo de datos con actor "jefe". *Fix:* cambiar el ejemplo a actores reales de v1, p. ej. "`original modelo X → Coord. Marco 09:12: Y (override, razón) → aprobado`".
- **medium** SLA de frescura del snapshot SIM con dos valores (§12 R-11 mitigación 1 = 48h vs §9.6 NFR-DAT-4 / OQ-117 = 36h) — número contradictorio para el mismo control. *Fix:* alinear R-11 a 36h (el valor endurecido y cerrado en NFR-DAT-4) y eliminar la "propuesta inicial 48h".

## Shape fit — strong

El PRD acierta su forma. Es una herramienta interna multi-rol con motor de forecast y trazabilidad financiera, no un consumer product — y el documento la trata así: los UJ existen porque hay flujos operativos reales (consolidación semanal, excepción mid-cycle desdoblada en AS-IS/TO-BE), pero no se sobre-formaliza con personas decorativas ni journeys para roles que no operan. La decisión de documentar al jefe de servicentro como persona §7.1 **sin** convertirlo en usuario está justificada explícitamente y sirve a v2 (no es theater: la etiqueta "no es usuario del sistema en v1" lo deja inequívoco y el UJ-1 vive como AS-IS de contexto, además relegado al addendum). El comprador Hugo sin UJ propia es correcto para un consumidor downstream pasivo — forzarle un journey sería el over-formalizing que la rúbrica penaliza.

La constraint traceability (stack obligatorio, convenciones de datos, fail-loud) se delega correctamente a `project-context.md` sin re-litigar, y §10 separa con disciplina lo que el PRD expone de lo que arquitectura decide. La forma es la correcta y está ejecutada con criterio.

## Mechanical notes

- **Drift de glosario (modelo dual → factor único):** el término "**factor de merma esperada**" (lenguaje del modelo dual) sobrevive en D-jefe-2 (línea 94), C-3 §8.3 (línea 289), §11.12 (línea 649) y OQ-114 opción (b) (línea 825), mientras el cuerpo nuevo usa "factor único de pérdida". OQ-114 opción (b) incluso propone "continuar usando factor de merma esperada como compensación" — terminología pre-D-9. Recomendado: normalizar a "factor único de pérdida" en todas las ocurrencias.
- **Continuidad de IDs OQ:** OQ-121 y OQ-122 aparecen **después** de OQ-126 (líneas 872 y 876), fuera de orden numérico. No rompe referencias (ambas están cerradas/superadas) pero desordena la lectura. Recomendado: reubicar tras OQ-120 o anotar el salto.
- **Cross-ref a hallazgos internos (F-NN, C-O-NN):** el PRD referencia "F-05", "F-Done-1/2", "F-16/F-17/F-18", "C-O2..C-O40" como si fueran identificadores estables, pero no hay índice que los resuelva dentro del PRD ni del addendum. Downstream no puede dereferenciarlos. Recomendado: o un mini-índice de findings/change-ordinals, o convertirlos en prosa.
- **Roundtrip de Assumptions:** los `[ASSUMPTION]` están inline (OQ-101 frecuencia de excepciones, baselines de TFS §4, ventana de excepciones §2) pero **no hay un Índice de Assumptions** al final como la rúbrica espera. Para un chain-top conviene un índice consolidado. Severidad baja.
- **D-IDs sin tabla de decisiones:** D-1, D-2, D-5..D-12 se citan por todo el documento (decisiones firmadas) pero no hay una tabla única que liste cada D-NN con su enunciado; están dispersas en notas de FR y en el cierre §14. Un Decision Log tabular ayudaría a downstream.
- **Continuidad FR:** los rangos eliminados (FR-004, 015, 020-025, 042, 062, 070-072, 090) están todos tachados con razón inline — buena higiene. No se detectaron duplicados ni FR huérfanos referenciados.
- **Secciones requeridas:** presentes y completas para los stakes (Visión, Contexto, Problema, Objetivos, Métricas con contra-métricas, Personas/UJ, FR por capacidad, NFR, Contexto técnico, Out-of-scope, Riesgos, Open Questions, Glosario).
