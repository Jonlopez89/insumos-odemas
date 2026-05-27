# PRD Quality Review — insumos-odemas

**Fecha:** 2026-05-26b
**Grade global:** **Good**
**Conteo de hallazgos:** 0 Critical / 0 High / 4 Medium / 3 Low

---

## Overall verdict

El PRD aguanta como documento de decisión maduro: tiene tesis explícita (consolidación distrital del coordinador, no el jefe), trade-offs firmados con honestidad epistémica poco común (la objeción de Winston sobre el modelo (C) se cita textual y no se disuelve, §10.4), y la ronda 2026-05-26b cerró limpiamente los tres High de Done-ness del Validate previo. Lo que está en riesgo no es el cuerpo del PRD sino su **changelog**: el pie de documento (§14 footer, líneas 912-944) quedó sin actualizar y ahora **contradice directamente las decisiones del cuerpo** — sigue diciendo que FR-066 está "pendiente de confirmación del owner", que las ventas vienen "desde Big Data/BigQuery", y que OQ-124/125/126 están abiertas, cuando el cuerpo ya las cerró. Es drift de severidad media (no afecta la lógica funcional, pero un lector que llegue al footer extrae la conclusión opuesta). El PRD es **candidato a `status: final`** una vez saneado ese footer y reconocidos los bloqueos externos como out-of-band; no hay nada estructural pendiente en los FRs.

---

## 1. Decision-readiness — **strong**

El PRD decide y lo admite. Las decisiones grandes están nombradas como decisiones, no enterradas: modelo (C) de SIM (§10.4, OQ-103 cerrada), reversa del modelo dual al factor único (D-9), carry-over reducido a surfacing pasivo (D-13), ventas/ALLOC/TFS como reporte del owner (OQ-124/125). El trade-off de cada una nombra qué se cede: §10.4 "Honestidad epistémica" reconoce que el modelo (C) **no elimina** la divergencia que Winston nombró, la **vigila** — y que "ninguna [de las tres mitigaciones de R-11] es engineered mitigation". Eso es exactamente lo contrario de smoothear todo a neutral.

Las Open Questions que siguen abiertas son genuinamente abiertas (OQ-108 cold-start, OQ-114 SIM sucio, OQ-123 umbral residual) y tienen dueño. Las cerradas se marcan CERRADA con fecha y decisión. R-11 está calificada "severidad: alta" sin maquillaje.

### Findings
- Ninguno que mueva el veredicto.

## 2. Substance over theater — **strong**

No hay relleno. Tres personas exactas (Jefe / Coordinador / Comprador), cada una con función declarada: el Jefe se mantiene **explícitamente como no-usuario** y el PRD justifica por qué se conserva ("mantener visibilidad del actor… informar v2", §7.1) en lugar de fingir que es usuario. La sección §4 "Por qué no Excel" es una defensa de tesis real, no innovation theater — responde a la objeción que de hecho recibió ("esto se podría hacer en Excel") con la cuenta dura. Los NFR llevan umbrales product-specific (NFR-E-3 ~8,600 series, NFR-DAT-4 36h, NFR-E-4 <3s, NFR-P-1 <100ms), no boilerplate "debe ser escalable".

### Findings
- **low** Persona Comprador delgada como UJ (§7.3) — Hugo no tiene UJ propio, solo aparece como consumidor downstream. Es correcto para un stakeholder pasivo (ver Shape fit), pero conviene que el PRD diga una línea explícita "sin UJ por diseño" para que UX no lo busque. *Fix:* añadir nota "stakeholder pasivo, sin UJ" en §7.3.

## 3. Strategic coherence — **strong**

Hay tesis y el documento la sostiene end-to-end: *el dolor v1 vive en el coordinador, no en el jefe* (§3), y todo lo que sigue se ordena bajo ella — los cuatro objetivos (§5) están priorizados, los FRs eliminados (FR-004, FR-042, FR-062, FR-070..072, FR-090, FR-015, NFR-E-2, NFR-DAT-2) se tacharon **porque dejaron de servir a la tesis** (el jefe fuera, TFS no se decide, conteo asistido fuera), no por conveniencia. El encuadre de 3 fases (Predicción / Inventario / Rebajas) da un arco claro de qué entra a v1 y por qué. Las contra-métricas (§6) están nombradas y son señales reales de falla (override >50% post-calibración, divergencia SIM >10% m/m), no métricas de vanidad.

### Findings
- **medium** Varias Success Metrics principales siguen sin baseline ni meta numérica (§6): tasa de quiebre `[OPEN: baseline]`, sobre-inventario `[OPEN: baseline]`, MAPE/WAPE `[OPEN: acordar con Finanzas]`, TFS registradas `[OPEN: meta]`. Es honesto (se declara "a recolectar en pilotaje") y consistente con R-7/R-9, pero significa que solo **una** métrica (tiempo del coordinador, 9h→<2h) es accionable hoy. Para un PRD que aspira a `final`, la tesis se valida con una sola vara. *Fix:* aceptable diferir si R-9 (recolección de baseline pre-go-live) queda como precondición explícita de pilotaje; ya lo está — basta señalar que la métrica de tiempo es la única gate v1.

## 4. Done-ness clarity — **strong** (era el flanco débil del Validate previo; los 3 High están cerrados)

Esta era la dimensión floja en el Validate 2026-05-26 (Fair). La ronda b la elevó: los tres FRs señalados ahora llevan criterio de aceptación observable. FR-040 define `buffer_seguridad` (antes F-Done-1) y FR-051 define "riesgo de quiebre" operacionalmente (`cobertura post-recorte ≥ 2 semanas`, antes F-Done-2). El patrón fail-loud está enunciado por FR concreto, no como adjetivo.

Quedan pocos adjetivos sin acotar y son menores (ver findings).

### Findings
- **low** "Ventana razonable que no bloquee la operación del miércoles" (NFR-E-3) y "rangos sanos" / "rango sano" (O2, §6 sobre-inventario) son adjetivos sin bound numérico. NFR-E-3 se rescata a sí mismo después con "<5 min", pero "rango sano de rotación" sigue colgado de R-7/Finanzas. *Fix:* marcar "rango sano" como `[OPEN: definir con Finanzas, R-7]` (ya implícito, hacerlo explícito en O2).
- **low** FR-067 (dashboard pedido-vs-recibido) no tiene criterio de "done" explícito más allá de la descripción de columnas; para un FR nuevo es aceptable pero conviene una condición verificable (ej. "la brecha se resalta cuando recibido < pedido"). *Fix:* añadir una frase de aceptación a FR-067.

## 5. Scope honesty — **strong**

Sección §11 "Out of scope" de 17 entradas, cada una justificada, es de las más honestas que se ven — incluye la concesión epistémica central nombrada como tal (§11.16: "v1 automatiza el efecto, no la causa raíz") y la cita a Dr. Quinn que la diagnosticó. Los `[ASSUMPTION: …]` están tagueados inline (frecuencia de excepciones, baseline TFS) y la mayoría tienen dueño/plan de medición. La densidad de open-items es alta pero **apropiada a un PRD que aún no es green-light-to-build** y que reconoce 3 bloqueos externos pendientes (R-7/R-8/R-9). De-scoping siempre explícito (FR tachados con razón y decisión-código).

### Findings
- **medium** No hay un índice consolidado de `[ASSUMPTION]` al final (la rúbrica lo pide como roundtrip). Los assumptions viven inline (§2 frecuencia excepciones, §4 baseline TFS, OQ-101) pero no hay sección "Assumptions Index" que los liste. Para un PRD de este tamaño (946 líneas) eso dificulta el roundtrip. *Fix:* añadir un apéndice "Índice de supuestos" o confirmar que la convención del proyecto no lo exige.

## 6. Downstream usability — **adequate** (bueno en lo funcional; arrastra drift de changelog)

Este PRD es chain-top (alimenta architecture.md → stories), así que la dimensión pesa. Lo funcional está bien: Glosario completo y actualizado (§14), IDs FR contiguos con tachados visibles (no se reciclan números), cross-refs que resuelven (FR-035→FR-082→FR-084 cadena coherente, FR-054↔FR-095 acoplados explícitamente). Cada sección se sostiene sola vía términos del Glosario.

Lo que baja el veredicto de strong a adequate es el **drift del changelog footer** (ver Mechanical notes — es donde más muerde porque architecture.md leerá ese footer como estado oficial).

### Findings
- **medium** El changelog footer (líneas 912-944) **contradice el cuerpo** en cuatro puntos tras la ronda b — ver detalle en Mechanical notes. Para downstream esto es el hallazgo de mayor impacto: un lector de architecture.md que confíe en el footer concluirá que FR-066 sigue abierto, que las ventas vienen de BigQuery (cuando OQ-124 cerró eso), y que OQ-124/125/126 están abiertas (cuando están cerradas en §13). *Fix:* reescribir el bloque footer para reflejar el estado 2026-05-26b (FR-066 cerrado por D-13; ventas/ALLOC/TFS = reporte del owner; OQ-124/125/126 cerradas; eliminar "Fernando ruta crítica" y "distinción BigQuery-lectura vs ML").

## 7. Shape fit — **strong**

La forma encaja con el producto. Es una herramienta interna multi-rol pero de un solo operador interactivo dominante (3 coordinadores idénticos en función) → el PRD usa shape de **capability spec** (12 áreas C-1..C-12, FRs densos) y lo complementa con UJs porque sí hay flujo humano relevante (consolidación semanal, excepción mid-cycle). No sobre-formaliza: el Jefe no recibe UJ TO-BE (correcto, no opera v1), el Comprador no recibe UJ (correcto, pasivo). Los UJs AS-IS/TO-BE están bien separados (UJ-3 explícitamente desdoblado AS-IS / TB). SMs son operacionales (tiempo, quiebre, override) no user-facing de consumo masivo — apropiado para tool interno.

### Findings
- Ninguno.

---

## Verificación explícita de los 3 High previos

### High #1 (Done-ness) — FR-066 carry-over con `[CONFIRMAR CON OWNER]` → **CERRADO** (en el FR; ver drift residual)

FR-066 (§8.7, línea 331) ahora se titula **"Surfacing pasivo de pedidos tardíos (sin regla de arrastre)"** y dice textual: *"v1 NO modela el carry-over como regla del sistema: no reinscribe automáticamente la tienda a otro ciclo, no recalcula ni traslada presupuesto, y no ejecuta una mecánica de arrastre con bitácora propia. El faltante de una tienda que no pidió se autocorrige de forma natural en el siguiente ciclo… El techo presupuestal sigue siendo semanal y rígido por tienda (sin acumulación ni pool)."* El tag `[CONFIRMAR CON OWNER]` y la "regla de presupuesto en carry-over" desaparecieron del FR; el propio FR cierra con *"Esta reducción cierra el finding high del Validate 2026-05-26 sobre el paso 4: ya no existe una regla de presupuesto en disputa."* OQ-118 (§13) registra la reversa (D-6 → D-13). **El High está cerrado en el cuerpo.**
**Pero** el changelog footer (línea 944) todavía lista *"FR-066 paso 4 (confirmar regla de presupuesto en carry-over)"* como decisión pendiente, y la línea 920 dice *"FR-066 carry-over (high): regla enunciada, pendiente confirmación owner."* → drift residual capturado como hallazgo medium #6.2.

### High #2 (Done-ness) — FR-035/FR-034 delegaban la mecánica del factor a architecture.md sin criterio propio → **CERRADO**

FR-035 (§8.4, línea 300) ahora trae **criterio de aceptación observable**: *"para una tienda con demanda de venta esperada `V` (FR-030) y factor resuelto `f`, la demanda ajustada cumple `demanda_ajustada = V × (1 + f)` sobre la demanda de venta, antes de la conversión a insumo (FR-031) y la cuantización (FR-032); el trazo explicable (FR-034) declara V, f, el nivel del que provino f… y el resultado."* La delegación a architecture.md se acota correctamente a la **opción técnica interna** ("multiplicador sobre output, ajuste del histórico, etc."), dejando claro que *"el criterio de aceptación anterior es observable e independiente del método interno."* FR-034 complementa con el trazo explicable (qué campos expone). **El criterio de done es ahora testeable sin esperar a architecture.md. Cerrado.**

### High #3 (Done-ness) — FR-030 no acotaba el "done" ni el comportamiento con tiendas < 12 meses → **CERRADO**

FR-030 (§8.4, línea 295) ahora trae **"Criterio de aceptación (done)"**: *"para una tienda con ≥ 12 meses de histórico de ventas, el motor produce un valor de demanda esperada por tienda × SKU venta para el ciclo objetivo, acompañado del trazo de FR-034; el backtesting (BacktestingSuite.java) sobre histórico real produce la desviación vs ventas YoY usada como criterio de éxito (R-7/C-O40)."* Y el cold-start queda fail-loud y deferido: *"Tiendas con < 12 meses de histórico (cold-start): el tratamiento… queda abierto en OQ-108 (dueño: arquitectura); hasta resolverlo, una tienda sin histórico suficiente se marca explícitamente como 'sin sugerencia del motor' en la bandeja (fail-loud — no se inventa un número…)."* El comportamiento con <12 meses está **definido como conducta v1 observable** (marca "sin sugerencia", trato manual del coord), con la política fina deferida a OQ-108 sin que eso deje el FR sin criterio. **Cerrado.**

---

## Hallazgos nuevos introducidos por los cambios (por severidad)

**Critical:** ninguno.

**High:** ninguno.

**Medium:**

- **medium #6.1 — SMs sin baseline accionable (preexistente, no introducido):** ver Dimensión 3. Solo la métrica de tiempo es gate v1. Aceptable si R-9 queda como precondición de pilotaje.
- **medium #6.2 — Drift del changelog footer post-ronda-b (INTRODUCIDO por la ronda):** líneas 912-944 no se actualizaron al cerrar OQ-124/125/126 y D-13. Cuatro contradicciones concretas con el cuerpo:
  1. Línea 913 / 918: *"D-8 (ventas semanales desde Big Data/BigQuery)"* y *"ventas semanales desde BigQuery"* — **contradice** OQ-124 cerrada (§13, línea 855) y §10.6, que pusieron las ventas como **reporte del owner sin BigQuery**.
  2. Línea 918 / 920 / 944: FR-066 *"pendiente de confirmación del owner"* / *"regla enunciada, pendiente confirmación owner"* — **contradice** FR-066 cerrado por D-13 (línea 331) y OQ-118 (línea 837).
  3. Línea 942 / 944: *"R-12 (confirmar fuentes RMS/BigQuery con Fernando)"* y *"OQ-124/OQ-125 (fuentes BigQuery/RMS con Fernando)"* como pendientes — **contradice** OQ-124/125 cerradas (líneas 855-861) y R-12 reencuadrada a "reporte del owner" (línea 754), donde *"Fernando deja de ser ruta crítica"*.
  4. Línea 918: *"D-11 (layout del reporte abierto, OQ-126)"* — **contradice** OQ-126 cerrada con Opción A (línea 863) y FR-110 (línea 388).
  *Fix:* reescribir el bloque de cierre (912-944) para reflejar 2026-05-26b. Es el único hallazgo que toca downstream con fuerza.
- **medium #6.3 — Sin Índice de supuestos consolidado:** ver Dimensión 5. Roundtrip de `[ASSUMPTION]` no verificable sin índice.
- **medium #6.4 — OQ-121/OQ-122 marcadas "SUPERADA por D-9" pero conservadas en §13 (líneas 867-873):** correcto historiar, pero ambas siguen redactadas en presente como si fueran preguntas; conviene un encabezado "[HISTÓRICO]" para que no se relean como abiertas. Menor.

**Low:**

- **low #7.1** — Comprador sin UJ explícitamente declarado "por diseño" (§7.3) — ver Dim. 2.
- **low #7.2** — Adjetivos sin bound: "ventana razonable", "rango sano de rotación" — ver Dim. 4.
- **low #7.3** — FR-067 sin frase de aceptación verificable — ver Dim. 4.

---

## Mechanical notes

- **Glosario:** completo, actualizado a 2026-05-26b (entradas nuevas para Factor de pérdida, Tipo de papel, Fases 1/2/3, Dashboard de validación, Canal oficial, Cadencia del forecast, Carry-over actualizado). Sin drift de términos detectado en el cuerpo — "factor único de pérdida", "tipo de papel", "carry-over pasivo" usados consistentemente entre FRs, §6, §10.4 y §11.
- **Continuidad de IDs:** FR-NNN contiguos con tachados visibles (FR-004, FR-015, FR-020..025/C-3, FR-042, FR-062, FR-070..072, FR-090 eliminados y marcados; no se reciclan números). NFR-E-2 y NFR-DAT-2 tachados. OQ-101..126 con saltos explicables (OQ-113-A/B, OQ-120/121/122 históricas). Sin duplicados ni cross-refs rotos detectados en el cuerpo.
- **Cross-refs:** resuelven — FR-035→FR-082/FR-084 (cadena del factor), FR-054↔FR-095 (acople presupuesto/compra extraordinaria), FR-066→FR-061 (carry-over surfacing), R-1/R-11→§10.4/§11.16. Las referencias a OQ y D-codes en el cuerpo apuntan a entradas existentes.
- **⚠️ Drift principal — changelog footer (912-944):** el único bloque desincronizado. Detallado en hallazgo medium #6.2. No es drift de glosario (case/plural) sino **drift de estado**: el footer narra la ronda anterior. Es el bloqueo cosmético-pero-importante para `status: final`.
- **Assumptions Index roundtrip:** no hay índice; assumptions viven inline (medium #6.3).
- **UJ persona linkage:** UJ-1 → Jefe (§7.1/§7.4, en addendum), UJ-2/UJ-2-TB → Coordinador (§7.2), UJ-3 AS-IS/TB → Coordinador territorial + Jefe como origen externo. Todos nombran persona definida. Sin UJ flotantes.

---

## ¿Candidato a `status: final`?

**Sí, con un saneamiento previo no estructural.** El cuerpo del PRD (FRs, NFRs, riesgos, OQs, glosario) está en estado final: los 3 High de Done-ness cerrados, scope honesto, tesis coherente, criterios de aceptación testeables. **Lo que bloquea el ascenso a `final`:**

1. **(Bloqueo de documento, fácil) Sanear el changelog footer (912-944)** para que no contradiga el cuerpo — hallazgo medium #6.2. Sin esto, downstream (architecture.md) lee estado obsoleto.
2. **(Bloqueos externos, out-of-band — no del PRD) R-7 (sesión Finanzas, ya reencuadrado a no-bloqueante de tesis), R-8 (spike forecasting Java) y R-9 (baselines de quiebre/sobre-inventario + backtesting).** Estos son precondiciones de **pilotaje**, no de `final` del PRD — el propio PRD los declara como tareas paralelas/precondiciones de go-live, no de aprobación del documento. Pueden quedar como riesgos abiertos en un PRD `final`.

Recomendación: aplicar el fix del footer (#6.2) y opcionalmente los 3 medium restantes; con eso el PRD asciende a `status: final` con los bloqueos externos reconocidos como gates de pilotaje, no de documento.
