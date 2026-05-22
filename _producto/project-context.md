---
project_name: 'insumos-odemas'
user_name: 'Jonathan'
date: '2026-05-22'
sections_completed: ['discovery', 'technology_stack', 'language_and_quality', 'data_inputs']
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

---

## Insumos de datos (CSV/XLSX en `docs/`)

### Inventario canónico

Los siguientes archivos son la fuente de verdad para el pipeline. Sus nombres, columnas y formatos pueden cambiar — cualquier cambio se registra aquí antes de modificar código.

| Archivo | Rol | Cardinalidad |
|---|---|---|
| `Cruce-de-skus-venta-insumo.xlsx` | Equivalencia venta↔insumo (autoridad) | 1 SKU venta → 1 SKU insumo |
| `historico-de-ventas-2024-2025-2026.csv` | Histórico de ventas por tienda/mes/SKU | grano: (year, month, item_id_venta, location_key) |
| `Skus_insumos.csv` | Catálogo de SKUs de insumo (lo que se compra a proveedor) | 1 fila por `SkuInsumo` |
| `Presupuesto-tiendas.csv` | Ventana presupuestal y periodicidad de pedido por tienda | 1 fila por tienda |
| `ALLOC_2026_04_27.csv` | Órdenes de surtido WH → tienda | snapshot fechado |
| `TFS_2026_04_27.csv` | Transferencias entre tiendas | snapshot fechado |
| `Entrega-directa-tienda.csv` | Compras y transferencias directas registradas en tienda | snapshot fechado |

Los archivos ALLOC/TFS/Entrega vienen con sufijo de fecha (`*_YYYY_MM_DD.csv`). El parser debe descubrirlos por patrón, no por nombre fijo.

### Modelo de cruce de SKU (decisión registrada)

`item_id_venta` (en `historico-de-ventas-*.csv`) y `sku_venta_id` (en `Cruce-de-skus-venta-insumo.xlsx`) son **el mismo identificador**, solo nombrado distinto por origen. El join autoritativo del pipeline es:

```
historico.item_id_venta  =  cruce.sku_venta_id  →  cruce.sku_insumo  =  Skus_insumos.SkuInsumo
```

`sku_insumo` es el código del artículo **comprable al proveedor** (caja/paquete/etc). Es el SKU sobre el que se cuantiza y se aplica el empaque mínimo.

La columna `sku_venta` del histórico (ej. `54693`) coincide a veces directamente con `SkuInsumo` — esto es un mapeo embebido por el equipo de datos, **no se usa como autoridad**. Se ignora salvo para conciliación de auditoría.

**Invariante (fail-loud):**
- Todo `item_id_venta` del histórico debe encontrarse en `Cruce-de-skus-venta-insumo.xlsx.sku_venta_id`. Si no → `EquivalenciaNoDefinidaException`, abortar ciclo de esa tienda. NUNCA inferir ni interpolar.

### Reglas de parseo por archivo

**`historico-de-ventas-2024-2025-2026.csv`**

- Periodo expresado como `(year, month)` enteros — NO `dd/MM/yyyy`. Reconstruir a `YearMonth` o primer día del mes según necesidad.
- `item_id_venta` y `sku_venta`: tratar como `String` aunque parezcan numéricos.
- `location_key` (entero): mapea a `Tienda.id`. String en dominio.
- `sales_quantity` y `tickets`: `long`. `sales_amount`: `BigDecimal(scale=4)`.

**`Cruce-de-skus-venta-insumo.xlsx`**

- Columnas: `sku_venta_id` (String) ↔ `sku_insumo` (String).
- Es XLSX, no CSV. Decisión pendiente de arquitectura: convertir a CSV en pipeline de ingesta o leer XLSX directo. Registrar la decisión en `architecture.md`.

**`Skus_insumos.csv`**

- Cabecera literal `"Matarial y tamaño"` (typo de origen). **Mapeo conservador**: el POJO usa exactamente `"Matarial y tamaño"` como nombre de columna en el parser. Si el equipo de datos corrige el typo, el parser debe fallar (es un cambio de contrato, no una mejora silenciosa).
- `Presentación` ∈ {`PAQUETE`, `CAJA C/Xp_Ypz`, ...}. `Contenido` = piezas por presentación (`long`). Cuantización opera sobre `Contenido` como múltiplo mínimo.
- `Costo`: `BigDecimal(scale=4)`, MXN.

**`Presupuesto-tiendas.csv`**

- `Cantidad por semana` viene como string `"$82,000"` (símbolo + miles con coma + comillas para escapar). Parser dedicado: strip `$`, strip `,`, → `BigDecimal(scale=2)`.
- `Semana que debe solicitar` viene en lenguaje natural (`"Solicitan insumo semana 1 y 3"`). Se modela como **`List<Integer>` de números de semana ISO 8601**:
  - `"semana 1 y 3"` → `[1, 3]`
  - `"semana 2 y 4"` → `[2, 4]`
  - Cualquier patrón no reconocido → `PeriodicidadPresupuestoIndefinidaException`, fail-loud.
- `Distrito` y `Tienda`: `String`.

**`ALLOC_*.csv`**

- `RELEASE_DATE` y `FECHA_DE_VIGENCIA_OC`: formato `dd/MM/yyyy`.
- `FECHA_CREATION_DATE_ORDEN`: formato `dd/MM/yyyy HH:mm` (con hora). La misma columna puede contener ambos formatos en distintas filas — parser dual con fallback ordenado.
- `papalEspacial` (sic, typo en origen): admite `""` legítimo. NO colapsar a `null`.
- `STATUS` y `STATUS_ORDEN`: **glosario pendiente** (ver "Riesgos abiertos").
- `WH`: warehouse id (`String`). `TO_LOC`: tienda destino (`String`).
- Cantidades (`QTY_ALLOCATED`, `QTY_TRANSFERRED`, `QTY_RECEIVED`, `CTD_PENDIENTE`): `long`. Invariante: `QTY_ALLOCATED = QTY_TRANSFERRED + CTD_PENDIENTE` (validar en parsing; si no se cumple, registrar warning con MDC poblado).

**`TFS_*.csv`**

- `CREATE_DATE` mezcla `dd/MM/yyyy` y `dd/MM/yyyy HH:mm` en la misma columna entre filas. Parser dual obligatorio.
- `FROM_LOC` y `TO_LOC`: `String` (pueden ser warehouse o tienda, no asumir por rango numérico).
- `STATUS` ∈ {A, I, S, ...}: **glosario pendiente**.
- Trailing comma en la cabecera (`STATUS,`); el último campo (`-`) parece ser placeholder. Documentar y descartar en el mapeo.

**`Entrega-directa-tienda.csv`**

- `TRAN_DATE` y `POST_DATE`: formato `dd/MM/yyyy`.
- `TRAN_CODE` ∈ {20=`Purchases`, 30=`Transfers In`, ...}: el archivo trae también la columna `DECODE` que ya descodifica el `TRAN_CODE`. **Usar `DECODE` como autoridad** y validar contra `TRAN_CODE` (fail-loud si difieren).
- `SUPPLIER`: `String` aunque parezca numérico.
- `STORE_NAME` incluye prefijo `"NN-NombreTienda"`. Para joins usar `LOCATION` (numérico → String), nunca `STORE_NAME`.
- `TOTAL_COST` y `TOTAL_RETAIL`: `BigDecimal(scale=4)`.

### Encoding y CSV dialect (transversal)

- Encoding: intentar UTF-8 BOM (`utf-8-sig`) primero, fallback `cp1252`. Loguear encoding detectado por archivo en MDC.
- Delimitador: coma. Comillas para escapar (`Presupuesto-tiendas.csv` lo demuestra con `"$82,000"`).
- Separador decimal: punto en CSVs revisados. Los XLSX pueden traer coma — muestrear antes de parsear.

### Riesgos abiertos (registrar, no resolver en código)

1. **Glosario `TRAN_CODE` y `STATUS`** (ALLOC.STATUS, ALLOC.STATUS_ORDEN, TFS.STATUS, Entrega.TRAN_CODE/DECODE): no existe diccionario formal en `docs/`. Política provisional: el parser conserva el código crudo como `String`. Cualquier consumidor que necesite ramificar lógica por status debe llamar a un método `interpretarStatus(...)` central; si el valor recibido no está en la lista conocida → `StatusDesconocidoException`, fail-loud. Pendiente: pedir glosario oficial al equipo de operaciones.
2. **Conversión XLSX→CSV de `Cruce-de-skus-venta-insumo.xlsx`**: decisión pendiente en `architecture.md`. Mientras tanto, el archivo se trata como input directo XLSX.
3. **Doble identificador en histórico (`item_id_venta` vs `sku_venta`)**: el segundo es un mapeo embebido del equipo de datos. Si en algún ciclo diverge respecto al cruce autoritativo, se loguea como anomalía pero no se aborta — el join autoritativo gana.
4. **Lock files de Excel**: `docs/~$*.xlsx` aparece cuando un .xlsx está abierto en Excel. Agregar `~$*.xlsx` a `.gitignore` para evitar commits accidentales.
