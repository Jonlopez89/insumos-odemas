# PRD Quality Review — insumos-odemas (2026-05-22, updated 2026-05-25 iter 4)

## Overall verdict

Después de cuatro iteraciones en 24 horas el PRD llega a un punto inusual de coherencia narrativa para un documento todavía en `draft`: la tesis es nombrable en una frase, la concesión epistémica central (operar sobre SIM sucio sin sanearlo) está declarada cinco veces de formas consistentes, y la decisión más resbalosa — qué hacer con la pérdida invisible — se reescribió desde "factor seedeado desde histórico" a "default = 0 + configuración manual progresiva a discreción del coord", lo cual es una rara muestra de scope honesto bajando expectativas en lugar de subirlas. Lo que sostiene el documento son §10.4 (modelo SIM (C) con objeción de Winston textualmente preservada), §8.9 (modelo dual de pérdida bien estructurado), §6 (métricas y contra-métricas calibradas a la fase) y §11 (out-of-scope detallado con tratamiento de cada exclusión). Lo que sigue poniendo en riesgo el documento como input para `architecture.md` son: (a) tres bloqueos Reviewer Gate todavía abiertos (R-7 MAPE, R-8 spike Java, R-9 backtesting + pilot baselines), correctamente documentados como acciones externas no decisiones de scope; (b) un sub-riesgo aceptado por el owner (sistema arranca con todo en 0, sin obligación de configurar pre-go-live) que delega la calidad inicial del forecast a la detección residual + overrides — el PRD lo declara honestamente pero deja a `architecture.md` el problema de cómo opera el motor en ciclos con factores en cero; (c) algunos huecos de definición todavía vivos (`buffer_seguridad` ahora sí definido por FR-040, pero el mecanismo de atribución residual OQ-120 sigue abierto y FR-081 depende de él).

## Decision-readiness — adequate

El PRD nombra sus decisiones cerradas y las firma con fecha. §10.4 conserva textualmente la objeción de Winston sobre el modelo (C) ("garantía de divergencia — dos sistemas, dos verdades, ningún ganador") y le añade en la cuarta iteración una sección de **honestidad epistémica** que admite que la objeción no se neutraliza, solo se vigila ("ninguna [mitigación] es engineered mitigation"). Esa admisión costaría poco esconderla y el PRD escoge no esconderla — eso es decision-readiness real, no smoothing.

§6 desdobla la métrica de override en dos filas (ciclos 1-3 ORO de calibración ≥ 25% vs ciclo 4+ confianza < 20%) absorbiendo la objeción de Dr. Quinn con un trade-off explícito y firmado. La métrica de "cobertura de configuración manual de factores" en §6 está bien marcada como **"seguimiento, no gate"** — el PRD se cuida de no convertir un indicador de uso en un gate ficticio. Eso es disciplina.

Donde la dimensión queda *adequate* y no *strong*: los tres bloqueos Reviewer Gate restantes (R-7, R-8, R-9) están **fuera del control del PRD** y el documento lo reconoce, pero al menos uno tiene implicación de diseño que el PRD no nombra: si R-7 (MAPE objetivo con Finanzas) sale fuera de un rango asumible, el modelo dual de factores (C-9) puede no ser suficiente — el PRD no nombra qué pasa con la arquitectura del motor si Finanzas pide un MAPE menor a, digamos, 15%. El sub-riesgo aceptado en R-1 ("arranca con todo en 0") debería tener una contra-métrica explícita en §6 que monitoree el riesgo: hoy se infiere de "Cobertura de configuración manual" pero no se nombra cuándo ese indicador dispara una intervención.

### Findings
- **high** Sub-riesgo R-1 aceptado pero sin gate operacional explícito (§11.16 + §12 R-1 + §6 fila "Cobertura de configuración manual") — El owner aceptó conscientemente que en ciclos tempranos el motor opera ciegamente sobre categorías en 0 y descansa en detección residual (FR-080) + overrides (FR-064). El PRD declara la concesión pero la métrica que la monitorea está marcada "seguimiento, no gate". Si la cobertura no crece o crece mal, no hay umbral pre-acordado para reaccionar. *Fix:* añadir contra-métrica concreta (ej. "si ≥X% de categorías sigue en 0 después de N ciclos, escalar a sponsor para decidir intervención") o documentar explícitamente que la trayectoria es aceptada sin gate.
- **medium** Tres bloqueos Reviewer Gate documentados como externos, pero R-7 (MAPE Finanzas) tiene implicación de diseño no nombrada (§9.10 NFR-T-5 + §12 R-7) — Si Finanzas pide un MAPE muy estricto, el modelo dual + Holt-Winters/ARIMA puede no llegar; el PRD no nombra qué rango de MAPE asume implícitamente. *Fix:* documentar el rango asumido o nombrar explícitamente que `architecture.md` debe definir la respuesta si el target sale fuera de un rango razonable.
- **medium** OQ-120 (mecanismo de atribución residual entre uso interno vs error de operador) sigue abierta y FR-081 depende de ella (§8.9 FR-081 + §13 OQ-120) — FR-081 promete "sugerencia de atribución preliminar... mecanismo a definir en OQ-120". Sin mecanismo cerrado, FR-081 no es implementable. Está marcado como decisión de pilotaje, pero el PRD pasa a `architecture.md` un FR cuyo comportamiento aún no se ha decidido. *Fix:* o cerrar OQ-120 con la recomendación inicial (opción 1 heurística confirmable, ya propuesta) antes de Validate final, o documentar que FR-081 ship con atribución 100% manual hasta que cierre.

## Substance over theater — strong

Las cuatro variantes de teatro están bajo control.

**Persona theater:** §7 tiene exactamente tres personas y dos de ellas están explícitamente marcadas como NO usuarios del sistema (Jefe = stakeholder externo cero interacción; Comprador = stakeholder pasivo downstream). Solo el Coordinador es load-bearing — aparece como actor en C-5..C-12 con tres modos detallados (bandeja distrital, bandeja territorial, administración). Es persona-spec, no persona-furniture. Que mantengan al Jefe como persona documentada con justificación explícita ("mantener visibilidad del actor en el ecosistema, informar v2") es la decisión correcta — eliminarlo escondería el trade-off que §3-bis declara abiertamente.

**Vision theater:** §1 declara explícitamente lo que NO ataca («el cálculo manual de 6h/quincena del jefe **no es dolor que v1 ataque directamente**»). Esta segunda mitad — nombrar lo que el sistema deja fuera — es lo que distingue visión real de visión-furniture.

**Innovation theater:** §4 "Por qué no Excel" no es retórica. Cada fila nombra una capacidad concreta donde Excel rompe y el sistema sí puede (validación a escala con descuento de compras esporádicas, histórico cruzado TFS, modelo dual de pérdida con jerarquía categoría→override, trazabilidad inmutable, fail-loud, reflejo automático entre ciclos). No hay celebración hueca de "automatización"; hay capacidades específicas con FR asociados.

**NFR theater:** §9 tiene umbrales concretos. NFR-E-1 cuantifica concurrencia (3 coords + 1 admin); NFR-E-3 ahora cuantificado (8,600 series, <5 min); NFR-E-4 (bandeja en <3s); NFR-D-1 (99% en ventana lunes-viernes 06:00-22:00 CDMX); NFR-DAT-4 (36h máximo); NFR-T-6 (≥80% mutantes PIT). Las heredadas de `project-context.md` tienen bounds (4.5:1 contraste, debounce 150ms, autosave 8s). No hay un solo "el sistema debe ser robusto" / "razonable" / "eficiente" sin bound.

### Findings
- (sin findings — la dimensión sostiene).

## Strategic coherence — strong

La tesis es nombrable en una frase: **"Automatizar el día semanal del coordinador (no del jefe) ahorra ~1,092 horas-hombre/año; SIM sigue siendo única fuente de captura del jefe; las pérdidas se desdoblan en dos componentes configurables manualmente a discreción del coord."** O1..O4 derivan sin gaps: O1 es el ahorro directo (9h → <2h × 3 coords × 52 semanas), O2 es el efecto downstream del forecast (cero quiebres + rotación sana), O3 es informar TFS sin decidir, O4 es desdoblar la merma en dos categorías visibles.

La coherencia se nota en cómo cada decisión grande se traza a través de múltiples secciones:

- **Scope-cut C-3 (conteo asistido):** §8.3 (capacidad eliminada) → §3-bis (qué dolor del jefe queda sin atender) → R-1 (consecuencia: dato sucio como precondición) → §8.9 C-9 (mitigación: dos factores configurables) → §11.12 (out-of-scope explícito) → §11.16 (concesión epistémica central) → addendum A1 (UJ-1 AS-IS preservado).
- **Modelo dual (C-9):** §3 D-merma-1 (problema desdoblado) → §5 O4 (objetivo de visibilidad por componente) → §6 (métricas separadas por componente) → §8.9 (capacidad con dos factores independientes + jerarquía + detección residual) → §10.5 (datos maestros con jerarquía gobernada) → §14 (glosario con dos entradas separadas).
- **Canal correo único:** §2 (cadencia operativa) → §3 D-exc-1 (tratamiento "historiado") → §7.6 UJ-3-AS-IS/TB → §8.10 (C-10 reformulada con `origen_aviso = correo` campo fijo) → §11.15 (out-of-scope canal estructurado) → §14 (glosario "Canal oficial").

Cinco secciones distintas contando la misma historia coherente, en tres dimensiones distintas. Eso es coherencia estratégica.

Las **métricas de éxito en §6** validan la tesis: tiempo del coordinador <2h es la métrica principal de v1. Las contra-métricas están genuinamente disconfirmatorias: "Reducción del tiempo del coordinador pero crecimiento ≥ 10% de quiebres" detecta exactamente el modo de fallo más obvio (automatización que se come casos que el manual capturaba), y la nueva "Divergencia SIM vs venta real > 10%" acoplada con R-11 está bien construida para detectar la objeción de Winston materializándose.

### Findings
- (sin findings — la dimensión sostiene).

## Done-ness clarity — adequate

Esta era la dimensión más débil en la versión anterior y se mejoró: `buffer_seguridad` ahora está definido en FR-040 (parámetro configurable, valor inicial sugerido = 1 semana de cobertura de venta promedio); `riesgo de quiebre` ahora está operacionalizado en FR-051 (cobertura post-recorte ≥ 2 semanas, priorización descendente). Las "F-Done-1" y "F-Done-2" del review anterior están cerradas.

Sigue habiendo huecos:

- **FR-061** declara estado "**solicitud recibida** (Excel capturado o ingresado)". "Capturado o ingresado" sigue siendo ambiguo: el PRD no aclara si el coordinador transcribe el Excel del jefe al sistema, si el sistema parsea attachments de correo, o si el jefe sigue mandando un Excel y el sistema solo refleja "se recibió" como flag manual del coord. Sin esta aclaración, FR-061 no es testable y bloquea el diseño de la bandeja distrital.
- **FR-080** ("residual excede banda estadística esperada (umbral OQ-106 reformulada)") — sigue dependiendo de OQ-106, que sigue abierta. El umbral residual no está cerrado. El PRD lo reconoce vía la OQ pero deja FR-080 sin criterio testable.
- **FR-081** depende de OQ-120 (atribución preliminar entre uso interno vs error de operador) — sin OQ-120 cerrada, FR-081 no es implementable más allá de "surface candidato con cantidad y SKU".
- **FR-110** "**Formato compatible** con la estructura del archivo de referencia... contrato literal del export". Aunque R-4/OQ-102 está cerrada, "compatible" como verbo no está definido a nivel bit. El propio PRD enuncia en NFR-T-2 que Hugo firma la fixture inmutable — bien — pero el PRD podría nombrar explícitamente qué cuenta como divergencia bloqueante (¿orden de columnas?, ¿formato de cada celda?, ¿encoding del XLSX?, ¿metadatos?). Sin ese nivel, "defecto bloqueante" es adjetivo.
- **FR-066 (carry-over)** es relativamente robusto pero el paso 4 dice "**Recalcula presupuesto disponible para la semana N+1** considerando que la tienda no consumió en N" — ¿significa que el techo de N se traslada a N+1 (acumulación)?, ¿o se descarta el techo de N?, ¿o el techo de N+1 se ajusta cómo exactamente? Es una regla de negocio crítica para FR-054 (cálculo de presupuesto disponible) y no está cerrada operacionalmente.
- **FR-014** (snapshot SIM) hereda OQ-117 parcialmente abierta (formato y dueño operativo). El SLA de frescura ya está fijado (NFR-DAT-4 = 36h), pero el formato concreto del CSV de snapshot no — y eso bloquea la implementación de la ingesta.
- **FR-082 / FR-084** describen la configuración manual de los dos factores pero no enuncia bounds operacionales: ¿hay límite superior razonable de un factor en piezas? ¿el sistema valida cordura del input (ej. "el factor es 80% del consumo histórico" → alerta)? ¿qué pasa si el coord configura un factor mayor que la demanda esperada? Sin estos bounds, el cofactor puede convertir el forecast en absurdo y no hay guardarraíl.

Es una dimensión que se mejoró sensiblemente vs. la iteración anterior (de *thin* a *adequate*) pero sigue requiriendo una pasada de criterios de aceptación por FR antes de generar stories. Probablemente el equipo lo hará en la etapa de epics, pero conviene nombrarlo.

### Findings
- **high** FR-061 estado "solicitud recibida (Excel capturado o ingresado)" sin aclarar cómo entra el Excel al sistema (§8.7 FR-061) — Bloquea diseño de bandeja distrital: ¿transcripción manual?, ¿parseo de attachment?, ¿flag manual del coord?, ¿ingesta de archivo plano del jefe? Es la decisión que define cuánto trabajo manual queda en el coord pre-revisión. *Fix:* abrir OQ explícita ("¿cómo entran las solicitudes del jefe al sistema?") con tres opciones (a) transcripción manual del coord, (b) ingesta de Excel desde carpeta compartida, (c) sistema parsea correo. Decisión owner antes de Validate.
- **high** FR-066 paso 4 "recalcula presupuesto disponible para semana N+1" sin regla cerrada (§8.7 FR-066) — Define un comportamiento crítico de FR-054 sin enunciarlo. ¿Se acumula techo de N+N+1 cuando hay carry-over? ¿Se descarta N? El PRD no responde y la decisión cambia significativamente el comportamiento del sistema. *Fix:* enunciar la regla explícitamente o abrir OQ.
- **medium** FR-080 + FR-081 dependen de OQ-106 y OQ-120 abiertas (§8.9 FR-080/FR-081 + §13 OQ-106/OQ-120) — Dos FRs centrales del modelo dual de pérdida no son implementables sin las dos OQs. *Fix:* cerrar al menos OQ-120 con la recomendación inicial (opción 1, heurística confirmable) antes de Validate; OQ-106 puede quedar como decisión de pilotaje siempre que se enuncie el comportamiento default ("hasta umbral acordado, FR-080 surface todo residual ≥ 1σ").
- **medium** FR-082 / FR-084 sin bounds operacionales (§8.9) — La configuración manual de factores en piezas no tiene validación de cordura. *Fix:* añadir FR (o nota en FR-082) que defina validación mínima — ej. "alerta si factor > X% de la demanda esperada del periodo".
- **medium** FR-014 hereda OQ-117 con formato pendiente (§8.2 FR-014 + §13 OQ-117) — SLA cerrado, formato CSV no. *Fix:* enunciar formato propuesto como decisión arquitectónica concreta (CSV con encoding UTF-8 BOM o cp1252, columnas mínimas) o tratarlo como decisión de `architecture.md` con bounds claras.
- **low** FR-110 "compatible" sin criterio bit-a-bit (§8.12 FR-110) — Aunque la fixture inmutable de Hugo cubre el caso operacional, el PRD podría nombrar explícitamente el criterio. *Fix:* añadir línea en FR-110 que diga "compatible = idéntico estructuralmente al fixture firmado por Hugo (orden de columnas, formatos de celda, encoding)".

## Scope honesty — strong

El PRD trata las omisiones como ciudadanas de primera. §11 tiene 16 sub-secciones de out-of-scope, cada una con justificación (no solo "X queda fuera" sino "X queda fuera porque... y reabre en v1.1 si..."). §3-bis es una sección entera dedicada a documentar el dolor histórico del jefe que v1 NO ataca — es exactamente la pieza que un PRD honesto debe tener cuando se hace un scope-cut grande.

`[ASSUMPTION]` tags se usan con discreción y precisión: OQ-101 frecuencia de excepciones mid-cycle (`[ASSUMPTION: 1-3 por semana en la red de 226]`, marcado como "a medir post-lanzamiento"); contadores TFS y baselines de quiebre marcados explícitamente como `[OPEN: baseline a recolectar en pilotaje]`. No se asumen y se ignoran — se asumen y se nombran.

Las **decisiones explícitamente firmadas con fecha** (C-O2, C-O5, C-O7, C-O12, C-O14, C-O16, C-O17, C-O19, C-O20, D-1..D-7) constituyen una bitácora de decisiones embebida en el documento. Que el .decision-log.md tenga 774 líneas y el PRD las refleje en su cuerpo es disciplina notable.

Las **contra-métricas en §6** son scope honesto en forma de operación: el PRD se da herramientas concretas para detectarse fallando (override > 50% sostenido, excepciones creciendo > 20% MoM, pedidos urgentes > 15% MoM, divergencia SIM > 10% MoM, carry-over > 1 ciclo en > 20% semanas).

La **honestidad epistémica añadida en §10.4** (post-F-03) — la admisión textual de que las mitigaciones de R-11 son "detección + escalamiento humano... ninguna es engineered mitigation" — es excepcional. Pocos PRDs nombran este nivel de honestidad sobre sus propias mitigaciones.

Lo único que rebaja un poco la dimensión: el sub-riesgo aceptado en R-1 ("arranca con todo en 0") está documentado pero no tiene métrica gate. Es honesto que esté documentado; sería más honesto tener un umbral pre-acordado de "si la cobertura no crece a X% en N ciclos, esto es un problema". El PRD se quedó a mitad de camino aquí.

### Findings
- (sin findings adicionales — ya cubierto en Decision-readiness "Sub-riesgo R-1 sin gate operacional"; la dimensión sostiene).

## Downstream usability — adequate

Este PRD es chain-top (alimenta `architecture.md` → UX → epics → stories), así que la dimensión pesa más.

**Glosario:** §14 es robusto — 30+ entradas, cada noun de dominio definido. Las dos entradas críticas nuevas (uso interno, error de operador) están bien diferenciadas y traceables. "Coordinador territorial" definido como term nuevo aclara el modo de operación (asignación fija, no rotativa).

**IDs FR/NFR/UJ/OQ:** ID space coherente con eliminaciones explícitas marcadas con `~~strikethrough~~` (FR-004, FR-015, FR-020-025, FR-042, FR-062, FR-070-072, FR-090, NFR-E-2, NFR-DAT-2). Eso preserva continuidad histórica sin romper referencias.

**Cross-references:** §3 nombra dolores que §8 atiende con FRs nominados ("Dolor: D-coord-1, D-coord-3"); §6 cita FR-082, FR-081, FR-064 explícitamente; §7.6 acopla UJ-3-TB con FR-091, FR-092, FR-093, FR-094, FR-095. Es referenciable.

Donde la dimensión se queda en *adequate* y no *strong*:

- **UJ persona linkage:** §7.4 UJ-1 nombra Jefe, §7.5 UJ-2 nombra Coordinador. Bien. Pero **el PRD no tiene UJ explícita para Comprador** — Hugo es persona definida (§7.3) pero no tiene UJ. Puede ser intencional (no opera UI, solo recibe export), pero al menos podría aparecer como UJ-X "Recepción del export por el comprador" con un paso para hacer explícito el handoff (recibe correo → abre archivo → verifica estructura → procesa OC).
- **Cross-ref ocasionalmente roto:** §8.9 dice "umbral OQ-106 reformulada" pero §13 OQ-106 dice "Tras C-9 reformulada, la pregunta cambió..." — la OQ-106 reformulada cambia el _objeto_ de la pregunta (ya no es "qué umbral cuenta como merma" sino "qué estructura tiene el histórico y qué umbral residual cuenta como sospechoso"), pero ese histórico **ya no existe** (FR-015 eliminado). OQ-106 hoy es una pregunta huérfana — pregunta por la estructura de algo que no se va a ingestar. Hay que reformularla de nuevo o cerrar la parte de histórico y dejar solo el umbral residual.
- **`SolicitusDeInsumosTodos.xlsx`** aparece consistentemente como término referencial pero §10.2 lo marca como "**Hallazgo del Discovery: NO está en el inventario canónico de `project-context.md`**" — se anota la inconsistencia con la fuente de verdad pero no se la cierra. Habría que actualizar `project-context.md` o documentar en este PRD que es input nuevo gobernado por el sistema, no fuente de datos heredada.
- **El término "merma"** sigue apareciendo en lugares no aclarados — ej. R-3 "definición de 'merma sospechosa residual'", §11.16 "calidad sucia se mitiga vía los dos factores configurables", glosario "lo que coloquialmente se llama 'merma'". El glosario aclara que merma se desdobla en dos componentes, pero el uso textual sigue mezclando coloquial ("merma") con técnico ("uso interno + error de operador") sin marca consistente. Para downstream (especialmente UX que tiene que decidir el wording de la UI), conviene una regla más estricta: ¿la UI dice "merma"?, ¿"uso interno"?, ¿"pérdida"?

### Findings
- **medium** OQ-106 reformulada pero su objeto (estructura del histórico de mermas) ya no existe (§13 OQ-106 + FR-015 eliminado) — La OQ pregunta por la estructura de un histórico que el owner decidió no ingestar. Es OQ huérfana — la pregunta ya no aplica. *Fix:* re-reformular OQ-106 a "umbral residual estadístico sobre snapshot SIM + venta histórica + factores configurados", eliminando referencia al histórico. O cerrar como N/A y abrir OQ nueva para el umbral.
- **medium** Comprador (Hugo) sin UJ explícita (§7.3 + §7.6) — Persona definida pero handoff downstream no journey-modeled. Para downstream architecture, conviene tener el paso explícito. *Fix:* añadir UJ-4 "Recepción del export por el comprador" con 3-4 pasos (recibe correo → abre XLSX → verifica fixture → procesa OC en flujo actual).
- **medium** `SolicitusDeInsumosTodos.xlsx` falta de `project-context.md` no resuelto (§10.2) — Inconsistencia con fuente de verdad documentada pero no cerrada. *Fix:* o actualizar `project-context.md` para incluirlo como archivo canónico, o documentar en este PRD que es input nuevo gobernado por el sistema (no fuente heredada).
- **low** Drift glosario "merma" — uso coloquial vs técnico inconsistente (§14 + §3 D-merma-1 + §11.16 + R-3) — El glosario aclara pero el cuerpo del PRD sigue mezclando. *Fix:* establecer regla — "merma" solo aparece entre comillas como término coloquial; el sistema y la UI usan "uso interno" / "error de operador" / "pérdida".

## Shape fit — strong

Este PRD ajusta correctamente a su shape. Es **multi-stakeholder B2B híbrido con MVP problem-solving**: tres tipos de usuario (jefe-stakeholder-externo, coordinador-actor-principal, comprador-stakeholder-downstream), tres modos de operación del actor principal, y un objetivo de "automatizar lo que ya se hace a mano" — no es greenfield innovation, es internal tool con UX que importa porque el coord pasa 9h/semana ahí.

Las personas y UJs están **bien dosificadas para el shape**: tres personas (no doce), dos con explicación explícita de por qué siguen documentadas a pesar de no ser usuarios, journeys AS-IS + TO-BE para los flujos críticos (UJ-1 jefe en addendum, UJ-2 + UJ-2-TB coordinador, UJ-3-AS-IS + UJ-3-TB excepción). Es exactamente lo que pediría una shape multi-stakeholder con UX que importa.

Para chain-top que alimenta `architecture.md`: el PRD respeta la separación con disciplina. §10 entera dice explícitamente "no decide arquitectura — esa decisión es de `architecture.md`" y enumera dependencias técnicas, restricciones y OQs sin pre-decidir solución. Addendum A3.3 enumera "decisiones arquitectónicas implícitas en el PRD pendientes de validar" (RxJS debounce, bitácora inmutable como event sourcing, export bit-a-bit XLSX) — el PRD se autocensura proactivamente sobre dónde se metió sin querer en arquitectura. Eso es shape-awareness avanzada.

NFRs trazables a arquitectura: cada NFR tiene un dueño implícito (stack, performance percibida, disponibilidad, auditabilidad, seguridad, datos, locale, accesibilidad, UX, testabilidad, i18n, observabilidad) y cada uno deja a `architecture.md` la decisión técnica concreta con bound pre-acordado.

### Findings
- (sin findings — la dimensión sostiene).

## Mechanical notes

- **Glosario drift "merma":** ya marcado como Downstream usability finding. Coloquial vs técnico mezclado en el cuerpo.
- **ID continuity:** FR-006 nuevo, FR-007 nuevo, FR-014 nuevo, FR-054 nuevo, FR-066 nuevo, FR-074 nuevo, FR-075 nuevo — todos numerados consistentemente. Eliminaciones con strikethrough preservadas. ID space saludable.
- **Assumptions Index roundtrip:** los `[ASSUMPTION]` inline (OQ-101 frecuencia excepciones, baselines TFS/quiebre, etc.) están todos rastreables a OQs o métricas marcadas `[OPEN:]`. No vi `[ASSUMPTION]` huérfano.
- **UJ persona linkage:** UJ-1 ↔ Jefe (§7.1), UJ-2 ↔ Coordinador (§7.2), UJ-3-AS-IS/TB ↔ Coordinador + Jefe-externo (§7.6), UJ-2-TB ↔ Coordinador (§7.7). Sin floating UJs. Comprador (Hugo, §7.3) sin UJ explícita — ya marcado como Downstream finding.
- **Cross-refs resueltos:** verifiqué C-9 ↔ FR-035 ↔ FR-080..085 ↔ §6 métricas; C-10 ↔ FR-091..095 ↔ §7.6 UJ-3-TB; FR-054 ↔ FR-095 ↔ §3 D-exc-5. Roundtrip limpio.
- **Estado del documento:** §status = `draft` correctamente puesto — los 3 bloqueos Reviewer Gate restantes (R-7, R-8, R-9) son acciones externas, no scope decisions, y el PRD lo declara. No promover a `final` sin cerrar los 3 ni cerrar OQ-120 (mecanismo atribución) ni resolver el hueco de FR-061 ("Excel capturado o ingresado").
- **Strikethrough preservados:** ~~FR-004~~, ~~FR-015~~, ~~FR-020-025~~, ~~FR-042~~, ~~FR-062~~, ~~FR-070~~, ~~FR-071~~, ~~FR-072~~, ~~FR-090~~, ~~NFR-E-2~~, ~~NFR-DAT-2~~, ~~C-3~~. Eliminaciones explícitas — no fantasmas.
- **Reviewer Gate footer:** §"Estado actual del PRD" al final del cuerpo del PRD documenta los 5 cerrados / 3 abiertos con detalle. Es la pieza correcta para que el siguiente Validate sepa exactamente qué quedó pendiente.

---

**Conteo de findings:**
- Critical: 0
- High: 3
- Medium: 7
- Low: 1

**Verdict por dimensión:**
- Decision-readiness: adequate
- Substance over theater: strong
- Strategic coherence: strong
- Done-ness clarity: adequate (mejorada desde *thin*)
- Scope honesty: strong
- Downstream usability: adequate
- Shape fit: strong

**Comparativo informal vs. iteración anterior (no diff formal):** el PRD subió desde "Poor" hacia algo equivalente a "Good con bloqueos externos documentados". Cinco bloqueos Reviewer Gate cerrados, la reescritura del modelo de pérdida en C-9 simplificó el problema más que complicarlo (raro), y la honestidad epistémica añadida en §10.4 (post-F-03) es excepcional. Lo que queda son 3 bloqueos externos (R-7, R-8, R-9) que el PRD no puede cerrar por sí mismo, más una pequeña deuda de done-ness (FR-061 "capturado o ingresado", FR-066 paso 4 sobre presupuesto, OQ-106 reformulación huérfana, OQ-120 atribución) que sí depende del PRD. El documento está decision-ready para iniciar `architecture.md` con los bloqueos externos gestionados en paralelo, **siempre que** se aclaren los huecos de FR-061 y FR-066, y se cierre OQ-120 con la recomendación inicial (opción heurística confirmable).
