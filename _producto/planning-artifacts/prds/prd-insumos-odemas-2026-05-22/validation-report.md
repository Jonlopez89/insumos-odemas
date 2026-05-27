# Validation Report — insumos-odemas

- **PRD:** `_producto/planning-artifacts/prds/prd-insumos-odemas-2026-05-22/prd.md`
- **Rubric:** `.claude/skills/bmad-prd/assets/prd-validation-checklist.md`
- **Run at:** 2026-05-26
- **Grade:** Fair

## Overall verdict

Después de cuatro iteraciones en 24 horas el PRD llega a un punto inusual de coherencia narrativa para un documento todavía en `draft`: la tesis es nombrable en una frase, la concesión epistémica central (operar sobre SIM sucio sin sanearlo) está declarada cinco veces de formas consistentes, y la decisión más resbalosa —qué hacer con la pérdida invisible— se reescribió desde "factor seedeado desde histórico" a "default = 0 + configuración manual progresiva a discreción del coord", lo cual es una rara muestra de scope honesto bajando expectativas en lugar de subirlas. El PRD subió desde "Poor" en la corrida anterior hacia algo equivalente a "Good con bloqueos externos documentados".

Lo que sostiene el documento son §10.4 (modelo SIM (C) con objeción de Winston textualmente preservada), §8.9 (modelo dual de pérdida bien estructurado), §6 (métricas y contra-métricas calibradas a la fase) y §11 (out-of-scope detallado con tratamiento de cada exclusión). Lo que sigue poniéndolo en riesgo como input para `architecture.md` son: (a) tres bloqueos Reviewer Gate todavía abiertos (R-7 MAPE, R-8 spike Java, R-9 backtesting + pilot baselines), correctamente documentados como acciones externas no decisiones de scope; (b) un sub-riesgo aceptado por el owner (sistema arranca con todo en 0) que delega la calidad inicial del forecast a detección residual + overrides, sin gate operacional; (c) algunos huecos de definición todavía vivos: FR-061 "Excel capturado o ingresado" sin aclarar el mecanismo de entrada, FR-066 paso 4 sin regla cerrada para el presupuesto en carry-over, OQ-106 reformulación huérfana porque el histórico ya no se ingesta, y OQ-120 mecanismo de atribución residual abierta. El grade **Fair** deriva mecánicamente de los 3 findings altos; ninguna dimensión está broken ni thin.

## Dimension verdicts

- Decision-readiness — **adequate**
- Substance over theater — **strong**
- Strategic coherence — **strong**
- Done-ness clarity — **adequate** (mejorada desde *thin* en validate anterior)
- Scope honesty — **strong**
- Downstream usability — **adequate**
- Shape fit — **strong**

## Findings by severity

### Critical (0)

_Ninguno._

### High (3)

**[Decision-readiness]** — Sub-riesgo R-1 aceptado pero sin gate operacional explícito (§11.16 + §12 R-1 + §6 fila "Cobertura de configuración manual")
El owner aceptó conscientemente que en ciclos tempranos el motor opera ciegamente sobre categorías en 0 y descansa en detección residual (FR-080) + overrides (FR-064). El PRD declara la concesión pero la métrica que la monitorea está marcada "seguimiento, no gate". Si la cobertura no crece o crece mal, no hay umbral pre-acordado para reaccionar.
Fix: añadir contra-métrica concreta (ej. "si ≥X% de categorías sigue en 0 después de N ciclos, escalar a sponsor para decidir intervención") o documentar explícitamente que la trayectoria es aceptada sin gate.

**[Done-ness clarity]** — FR-061 estado "solicitud recibida (Excel capturado o ingresado)" sin aclarar cómo entra el Excel al sistema (§8.7 FR-061)
Bloquea diseño de bandeja distrital: ¿transcripción manual del coord?, ¿parseo de attachment de correo?, ¿flag manual del coord?, ¿ingesta de archivo plano del jefe? Es la decisión que define cuánto trabajo manual queda en el coord pre-revisión.
Fix: abrir OQ explícita ("¿cómo entran las solicitudes del jefe al sistema?") con tres opciones (a) transcripción manual del coord, (b) ingesta de Excel desde carpeta compartida, (c) sistema parsea correo. Decisión owner antes de Validate.

**[Done-ness clarity]** — FR-066 paso 4 "recalcula presupuesto disponible para semana N+1" sin regla cerrada (§8.7 FR-066)
Define un comportamiento crítico de FR-054 sin enunciarlo. ¿Se acumula techo de N+N+1 cuando hay carry-over? ¿Se descarta N? ¿Cómo exactamente se ajusta N+1? El PRD no responde y la decisión cambia significativamente el comportamiento del sistema.
Fix: enunciar la regla explícitamente en FR-066 o abrir OQ explícita.

### Medium (7)

**[Decision-readiness]** — R-7 (MAPE Finanzas) con implicación de diseño no nombrada (§9.10 NFR-T-5 + §12 R-7)
Si Finanzas pide un MAPE muy estricto, el modelo dual + Holt-Winters/ARIMA puede no llegar; el PRD no nombra qué rango de MAPE asume implícitamente.
Fix: documentar el rango asumido o nombrar explícitamente que `architecture.md` debe definir la respuesta si el target sale fuera de un rango razonable.

**[Decision-readiness]** — OQ-120 abierta y FR-081 depende de ella (§8.9 FR-081 + §13 OQ-120)
FR-081 promete "sugerencia de atribución preliminar... mecanismo a definir en OQ-120". Sin mecanismo cerrado, FR-081 no es implementable. El PRD pasa a `architecture.md` un FR cuyo comportamiento aún no se ha decidido.
Fix: cerrar OQ-120 con la recomendación inicial (opción 1, heurística confirmable ya propuesta) antes de Validate final, o documentar que FR-081 ship con atribución 100% manual hasta que cierre.

**[Done-ness clarity]** — FR-080 + FR-081 dependen de OQ-106 y OQ-120 abiertas (§8.9 FR-080/FR-081 + §13 OQ-106/OQ-120)
Dos FRs centrales del modelo dual de pérdida no son implementables sin las dos OQs cerradas. El umbral residual estadístico y el mecanismo de atribución preliminar son piezas que se pasan a `architecture.md` sin estar definidas.
Fix: cerrar al menos OQ-120 con la recomendación inicial antes de Validate; OQ-106 puede quedar como decisión de pilotaje siempre que se enuncie el comportamiento default ("hasta umbral acordado, FR-080 surface todo residual ≥ 1σ").

**[Done-ness clarity]** — FR-082 / FR-084 sin bounds operacionales de cordura (§8.9)
La configuración manual de los dos factores en piezas no tiene validación de cordura. ¿Hay límite superior razonable? ¿El sistema alerta si el factor supera X% de la demanda esperada? Sin estos bounds, el cofactor puede convertir el forecast en absurdo.
Fix: añadir FR o nota en FR-082 con validación mínima — ej. "alerta si factor > X% de la demanda esperada del periodo".

**[Done-ness clarity]** — FR-014 hereda OQ-117 con formato del snapshot pendiente (§8.2 FR-014 + §13 OQ-117)
SLA de frescura cerrado (NFR-DAT-4 = 36h) pero el formato concreto del CSV de snapshot no está enunciado, lo que bloquea la implementación de la ingesta en `architecture.md`.
Fix: enunciar formato propuesto como decisión arquitectónica concreta (CSV con encoding UTF-8 BOM o cp1252, columnas mínimas declaradas) o tratarlo como decisión de `architecture.md` con bounds claras y dueño.

**[Downstream usability]** — OQ-106 reformulada pero su objeto ya no existe (§13 OQ-106 + FR-015 eliminado)
La OQ pregunta por la estructura de un histórico que el owner decidió no ingestar (parte 3 del decision log). Es OQ huérfana — la pregunta ya no aplica.
Fix: re-reformular OQ-106 a "umbral residual estadístico sobre snapshot SIM + venta histórica + factores configurados", eliminando referencia al histórico; o cerrar como N/A y abrir OQ nueva para el umbral.

**[Downstream usability]** — Comprador (Hugo) sin UJ explícita (§7.3 + §7.6)
Persona definida pero handoff downstream no journey-modeled. Para `architecture.md` y UX, conviene tener el paso explícito de recepción del export — define la frontera del sistema con precisión.
Fix: añadir UJ-4 "Recepción del export por el comprador" con 3-4 pasos (recibe correo → abre XLSX → verifica fixture → procesa OC en flujo actual fuera del sistema).

**[Downstream usability]** — SolicitusDeInsumosTodos.xlsx falta en project-context.md no resuelto (§10.2)
Inconsistencia con la fuente de verdad del proyecto documentada en el propio PRD pero no cerrada. `project-context.md` lista el inventario canónico de archivos en `docs/` pero no incluye este XLSX que ahora es contrato del export.
Fix: actualizar `project-context.md` (vía `bmad-generate-project-context`) para incluirlo como archivo canónico, o documentar en este PRD que es input nuevo gobernado por el sistema.

### Low (2)

**[Done-ness clarity]** — FR-110 "compatible" sin criterio bit-a-bit operacional (§8.12 FR-110)
Aunque la fixture inmutable de Hugo (NFR-T-2) cubre el caso operacional, "compatible" como verbo no está definido a nivel bit.
Fix: añadir línea en FR-110: "compatible = idéntico estructuralmente al fixture firmado por Hugo (orden de columnas, formatos de celda, encoding XLSX)".

**[Downstream usability]** — Drift glosario: uso coloquial vs técnico de "merma" inconsistente (§14 + §3 D-merma-1 + §11.16 + R-3)
El glosario aclara que "merma" se desdobla en dos componentes (uso interno + error de operador), pero el cuerpo del PRD sigue mezclando coloquial ("merma") con técnico sin marca consistente.
Fix: establecer regla — "merma" solo aparece entre comillas como término coloquial; el sistema y la UI usan "uso interno" / "error de operador" / "pérdida".

## Mechanical notes

- **ID continuity:** FR-006, FR-007, FR-014, FR-054, FR-066, FR-074, FR-075 numerados consistentemente. Eliminaciones (~~FR-004~~, ~~FR-015~~, ~~FR-020-025~~, ~~FR-042~~, ~~FR-062~~, ~~FR-070-072~~, ~~FR-090~~, ~~NFR-E-2~~, ~~NFR-DAT-2~~, ~~C-3~~) preservadas con strikethrough. ID space saludable.
- **Assumptions Index roundtrip:** los `[ASSUMPTION]` inline están todos rastreables a OQs o métricas marcadas `[OPEN:]`. Sin huérfanos detectados.
- **UJ persona linkage:** UJ-1 ↔ Jefe, UJ-2 ↔ Coordinador, UJ-3-AS-IS/TB ↔ Coordinador + Jefe-externo, UJ-2-TB ↔ Coordinador. Comprador (Hugo) sin UJ — marcado como Downstream finding.
- **Cross-refs resueltos:** C-9 ↔ FR-035 ↔ FR-080..085 ↔ §6 métricas; C-10 ↔ FR-091..095 ↔ §7.6 UJ-3-TB; FR-054 ↔ FR-095 ↔ §3 D-exc-5. Roundtrip limpio.
- **Estado del documento:** frontmatter `status: draft` correctamente puesto. **No promover a `final`** sin cerrar los 3 bloqueos externos (R-7, R-8, R-9), OQ-120, el hueco de FR-061 y el hueco de FR-066.
- **Reviewer Gate footer:** la sección "Estado actual del PRD" al final del cuerpo documenta los 5 bloqueos cerrados / 3 abiertos con detalle.

## Reviewer files

- `review-rubric.md`
