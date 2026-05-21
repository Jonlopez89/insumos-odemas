---
project_name: 'insumos-odemas'
user_name: 'Jonathan'
date: '2026-05-20'
sections_completed: ['discovery', 'technology_stack', 'language_and_quality']
existing_patterns_found: 0
---

# Project Context for AI Agents

_Este archivo contiene reglas críticas y patrones que los agentes de IA deben seguir al implementar código en este proyecto. Enfoque en detalles no obvios que los agentes podrían pasar por alto._

---

## Stack tecnológico y restricciones de entorno

### REGLA ACTIVA — Bloqueo de implementación

Hasta que exista `_producto/planning-artifacts/architecture.md` aprobado, los agentes **NO deben**:

- Instalar paquetes (`mvn dependency`, `npm install`, `pip install`, etc.) ni inicializar entornos.
- Crear archivos fuera de `docs/`, `_bmad/`, `_producto/`: incluye `pom.xml`, `package.json`, `angular.json`, `Dockerfile`, scripts de Cloud Build, carpetas `src/`, `frontend/`, `backend/`.
- Decidir versiones específicas — solo proponer rangos.
- Generar código Java/TypeScript/SQL ejecutable.

**Trabajo permitido en esta fase:** leer e inspeccionar archivos en `docs/`, responder preguntas sobre los datos y el dominio, producir análisis y diagramas en markdown, redactar artefactos BMad en `_producto/`.

Si el agente cree tener razón para una excepción, debe **preguntar al usuario antes de actuar**. No autorizar la excepción por cuenta propia.

---

### Stack corporativo obligatorio (no negociable)

| Capa | Tecnología |
|------|------------|
| Backend | **Java + Spring Boot**, contenerizado con Docker |
| Base de datos | **PostgreSQL** desplegada como **Cloud SQL** en GCP |
| Frontend | **Angular** |
| Hosting frontend | **Firebase Hosting** |
| Plataforma cloud | **Google Cloud Platform** (no AWS, no Azure) |

### Motor de forecasting (decisión arquitectónica registrada)

**Decisión:** forecasting en **Java puro** (Smile o Apache Commons Math), encapsulado tras una interfaz `ForecastingEngine`. **No** se introduce Python ni Vertex AI / BigQuery ML en esta fase.

**Modelos iniciales permitidos:** media móvil ponderada, suavizado exponencial / Holt-Winters, ARIMA. Nada de redes neuronales o ML "sofisticado" sin justificación cuantitativa.

**Restricción operacional:**
- No agregar Python al stack sin RFC formal aprobado por arquitectura + IT.
- No agregar dependencias gestionadas de GCP (Vertex AI, BigQuery ML) sin la misma vía.

**Trigger objetivo para reevaluar Python como microservicio:** cuando ocurra **cualquiera** de las siguientes:
- MAPE/WAPE del motor Java en backtesting > umbral acordado con Finanzas.
- Volumen ≥ 50 tiendas activas, o ≥ varios miles de SKUs activos con historia.
- Requisito de modelos jerárquicos / por categoría que Smile no cubra.

### Arquitectura de referencia (ilustrativa, no obligatoria)

`docs/arcitecturaPrueba.png` muestra la arquitectura del proyecto hermano **Tomaturno**. Patrón sugerido a evaluar para *insumos-odemas*:

- Angular App (Firebase Hosting) → API Java/Spring Boot (Cloud Run + Docker) → Cloud SQL (PostgreSQL) + Cloud Storage (evidencias/archivos).
- Auth: **Auth0** con MFA/JWT para roles administrativos.
- Servicios externos plausibles: **SendGrid** (correo), **Sinch** (SMS/OTP).

El equipo puede divergir de este patrón si la justificación queda registrada en `architecture.md`.

### Restricciones de entorno (siempre vigentes)

- Sistema local del equipo: **Windows 11 + PowerShell 5.1** (no asumir PowerShell 7 ni bash nativo).
- Despliegue: **GCP exclusivamente** (no AWS / Azure).
- Datos de entrada en esta fase: archivos planos (CSV/XLSX) en `docs/`, exportados desde sistemas internos de OD. **No se asume acceso directo a bases productivas.**
- **Encoding de CSVs:** validar caso por caso (esperar `utf-8-sig` o `cp1252` indistintamente — los exports corporativos varían).
- **Locale:** `es-MX`. Decimales con punto en los CSVs revisados, pero los archivos Excel pueden traer coma como separador decimal — verificar antes de parsear.

---

## Reglas específicas del lenguaje y disciplina de calidad

### Convenciones de datos (transversales — alto impacto en correctitud)

- **Fechas en CSVs de origen:** formato `dd/MM/yyyy` o `dd/MM/yyyy HH:mm` (formato mexicano). NUNCA parsear como ISO sin validación. En Java: `DateTimeFormatter.ofPattern("dd/MM/yyyy")` con `Locale.of("es", "MX")`.
- **Encoding al leer CSV:** intentar `UTF-8 BOM` (utf-8-sig); si falla, `cp1252`. Registrar en log el encoding detectado por archivo.
- **Separador decimal:** punto (`.`) en CSVs revisados; los Excel pueden traer coma. Verificar con muestreo antes de parsear.
- **Cantidades monetarias:** SIEMPRE `java.math.BigDecimal` con scale=4 en cálculos intermedios. NUNCA `double` ni `float`. Moneda MXN.
- **Cantidades enteras (unidades, packs):** `long` para totales; `int` solo si el dominio acota explícitamente.
- **IDs de tienda y SKU:** tratar como `String` aunque parezcan numéricos (pueden traer ceros a la izquierda o adquirir prefijos).
- **NULL vs vacío en CSV:** distinguir `""` (vacío legítimo, ej. `papalEspacial` en ALLOC) de columna ausente. No colapsar a `null` sin decisión explícita.

### Disciplina de testing (innegociable antes de código)

- **Golden dataset versionado** en `src/test/resources/fixtures/golden/vN/` con `EXPECTED_OUTPUT.csv` firmado por el comprador piloto + `DECISION_LOG.md`. Una vez versionada, `vN/` NO se modifica — se crea `v(N+1)`. Output del motor debe coincidir byte-a-byte (o con tolerancia documentada por línea).
- **Property-based testing con jqwik** sobre cuantización y presupuesto. Invariantes obligatorias:
  - `∀ output: cantidad % unidadMinimaProveedor == 0`
  - `∀ output: cantidad >= ceil(demandaEsperada / unidadMinima) * unidadMinima` (nunca redondear hacia abajo si hay demanda)
  - `∀ tienda, ciclo: sum(costo) <= presupuesto(tienda, ciclo)`
  - `∀ SKU sin equivalencia definida: output NO lo contiene` (fail-loud)
- **Contract testing** entre Spring Boot y Angular: **Pact** o **Spring Cloud Contract**. Contrato YAML del endpoint definido antes de implementar; ambos lados consumen el mismo contrato.
- **Backtesting suite** en `src/test/java/.../BacktestingSuite.java`. Corre el motor sobre histórico real (ej. `ventas_2023.csv` → predice `2024_Q1`), compara contra realizado. **De aquí sale el número de MAPE/WAPE acordado con Finanzas**, no de un slide.
- **Mutation testing con PIT** (pitest.org) sobre paquete `forecasting.*`. Threshold mínimo 80% de mutantes eliminados antes de mergear a `main`.
- **Fail-loud por defecto:** si falta equivalencia, presupuesto, o el encoding es ambiguo → excepción checked y abortar ciclo de esa tienda. NUNCA rellenar con 0, NUNCA asumir default, NUNCA interpolar silenciosamente.
- **Pipeline composable:** cada paso del flujo (parsing → validación → catálogo → joins → forecast → cuantización → presupuesto-window → output) debe ser una unidad testeable en aislamiento. El forecast como "función pura" cubre solo 1 de 8 pasos.

### Java + Spring Boot

- **Java 17 LTS o superior** (records, pattern matching, text blocks). Confirmar versión disponible en imagen base corporativa antes de fijar.
- **Capa de dominio antes que framework:** reglas de conversión consumo→compra y cuantización viven en POJOs testeables sin Spring. No usar `@Service` ni `@Autowired` en lógica de negocio core.
- **Excepciones checked específicas** por escenario de negocio: `EquivalenciaNoDefinidaException`, `MinimoPackNoConfiguradoException`, `PresupuestoExcedidoException`, etc. Obligan al caller a decidir aborto vs degradación manual. Trazabilidad reversa hecha tipo.
- **Logging:** SLF4J con MDC poblado con `tiendaId`, `skuId`, `cicloId`. Sin esto, debugging post-mortem y respuesta a auditoría son imposibles.
- **Sin reflection sobre datos de negocio.** Mapeo CSV→POJO explícito (Jackson CSV o equivalente). No introspección mágica.
- **`Optional<T>`** solo como tipo de retorno, NUNCA como campo de clase ni parámetro.

### TypeScript + Angular

- **`strict: true` y `strictNullChecks: true`** en `tsconfig.json` — no negociable.
- **`any` prohibido** salvo en boundaries de integración (parseo crudo); siempre con mapper inmediato al tipo declarado.
- **RxJS:** suscripciones manejadas con `takeUntilDestroyed` o `async pipe`. `.subscribe()` sin teardown es bug, no estilo.
- **Modelos separados:** `models/api/` (response del backend) ≠ `models/view/` (lo que ve el componente). El backend cambia; el frontend no debe romperse en cascada.
- **Locale:** `LOCALE_ID = 'es-MX'`, `registerLocaleData(localeEsMX)`.
- **Fechas:** mostrar `dd/MM/yyyy`; transportar JSON como ISO 8601 (UTC).

### UX no negociable (presentación al usuario)

**Trazabilidad y override visibles:**
- Toda cantidad sugerida lleva affordance de explicación (ícono `info` con tooltip): origen del número, ventana de histórico usada, ajuste estacional aplicado. Ningún número crítico se renderiza sin su contexto.
- Valor sugerido vs valor override se diferencian tipográficamente: sugerido (gris/itálica) vs override (negro/bold + badge `Modificado por {user} {hh:mm}`).
- Auditoría visible en UI: hover sobre fila modificada muestra cadena `Original modelo X → Lupita 07:48: Y → Víctor 09:12: aprobado`. Panel lateral "Cambios de hoy" exportable a CSV.

**Edición y feedback:**
- Toda edición numérica dispara recálculo optimista en cliente (`RxJS debounceTime(150)`), con confirmación servidor. Recalcular fila, acumulado por proveedor y restante de presupuesto en <100ms.
- Acciones reversibles > acciones confirmadas: toast con "Deshacer" durante 8s, no `confirm()` modal.
- Autosave silencioso con indicador tipo Google Docs ("Guardado hace 3 seg"). El usuario nunca pierde trabajo, y lo sabe.

**Accesibilidad (WCAG 2.1 AA mínimo):**
- Contraste 4.5:1 en texto, 3:1 en componentes interactivos.
- Toda celda editable con `aria-label` contextualizado: `Cantidad sugerida para papel opalina, tienda 28, semana 1, 18 cajas, editable`.
- Navegación por teclado completa en grillas (Tab/Shift+Tab/Enter/Esc). Power users no tocan el mouse.
- `:focus-visible` con outline 2px. `outline:none` está prohibido.

**Error states (tres niveles):**
- **Inline (campo):** mensajes en `es-MX` claro, no técnicos. `La cantidad debe ser múltiplo de 6 (empaque mínimo)`, no `Invalid input: not divisible by pack_size`.
- **Negocio (fila/sección):** banner contextual con acción sugerida. `El total excede el presupuesto. Reducir cantidad o solicitar ampliación.`
- **Sistema (global):** toast rojo con código de incidente copiable + reintento con backoff. NUNCA stacktrace.
- Regla maestra: todo error tiene **quién, qué, por qué y qué hacer ahora**. Coincide con la regla fail-loud del backend — backend grita, frontend traduce.

**Locale de negocio MX (no solo `LOCALE_ID`):**
- **Cantidades discretas:** enteros sin decimales, `number:'1.0-0'`. Separador de miles arriba de 1,000: `1,250 cajas`. Unidad siempre pegada al número.
- **Empaque mínimo:** input rechaza valores no múltiplos y sugiere el cercano. Mostrar siempre equivalencia: `18 piezas = 3 cajas`. Usuario piensa en cajas, sistema en piezas, UI traduce ambas direcciones.
- **Moneda MXN:** `CurrencyPipe:'MXN':'symbol-narrow':'1.2-2'` → `$1,250.00`. Totales grandes (>$100,000) abreviados con tooltip: `$1.2M` / hover `$1,234,567.89`. Nunca mezclar MXN y USD sin badge de divisa.
- **Porcentajes:** un decimal máximo (`12.5%`, no `12.47%`). Variaciones con signo y color: `+12.5%` verde, `−8.3%` rojo (U+2212, no guion).
- **Fechas con contexto semanal:** `Sem 12 · 16–22 mar 2026`. Compradores piensan en semanas ISO 8601 (lunes a domingo), no en fechas sueltas.
- **Plazos:** días hábiles, no naturales (`5 días hábiles`).

### SQL / PostgreSQL

- **`snake_case`** para tablas y columnas (`historico_ventas`, `sku_insumo`). Convención uniforme con maestros existentes de OD si los hay.
- **Tipos exactos:** `NUMERIC(p,s)` para montos (no `FLOAT`/`REAL`). `BIGINT` para cantidades. `TIMESTAMPTZ` para fechas con hora; `DATE` para fechas puras.
- **Schemas separados:** `raw` (CSVs cargados sin transformación), `staging` (limpios/validados), `mart` (presentación). Sin esto, debugging del pipeline es imposible.
- **Migrations:** Flyway o Liquibase (decisión en `architecture.md`). Una migration por PR. NUNCA editar migrations ya aplicadas — se crea una nueva.
