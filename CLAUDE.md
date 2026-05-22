# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Critical: current project phase

This repo is in **planning phase**. There is **no source code yet** — only data inputs (`docs/`), BMad tooling (`_bmad/`), planning artifacts (`_producto/`), and design artifacts (`design-artifacts/`).

**Hard rule — do not bypass without asking the user first:** until `_producto/planning-artifacts/architecture.md` exists and is approved, you must NOT:

- Install packages (`mvn`, `npm install`, `pip install`, …) or initialize environments.
- Create files outside `docs/`, `_bmad/`, `_producto/`, `design-artifacts/` — no `pom.xml`, `package.json`, `angular.json`, `Dockerfile`, no `src/` / `frontend/` / `backend/` trees.
- Pin specific versions (propose ranges only).
- Generate executable Java / TypeScript / SQL.

Permitted work in this phase: read/inspect files under `docs/`, answer domain questions, produce markdown analysis and diagrams, draft BMad artifacts in `_producto/` and `design-artifacts/`.

The authoritative source for these rules — plus all stack, data, and quality conventions — is **`_producto/project-context.md`**. Read it before doing anything substantive. If it conflicts with this file, project-context.md wins.

## What this project is

**insumos-odemas** — supply forecasting and allocation system for Office Depot México's in-store print centers ("centro de copiado"). Predicts paper/supply demand per store per cycle, quantizes to supplier pack minimums, and respects per-store budget windows. Inputs are CSV/XLSX exports under `docs/` (sales history, allocations, store budgets, SKU catalog, direct-to-store deliveries).

Architecture target (not yet approved, see `docs/arcitecturaPrueba.png` reference from sister project Tomaturno):

- Angular (Firebase Hosting) → Java/Spring Boot API (Cloud Run + Docker) → Cloud SQL Postgres + Cloud Storage.
- Forecasting engine in **Java only** (Smile / Apache Commons Math) behind a `ForecastingEngine` interface. **No Python, no Vertex AI, no BigQuery ML** without a formal RFC.

## Repository layout

```
docs/                       Domain inputs — CSV/XLSX exports, reference PNGs, PRD docx. Source of truth for data conventions.
_producto/                  BMad output (Spanish). Artifacts the team produces.
  project-context.md        ⚠️ Read first. Active rules, stack constraints, data conventions, testing discipline.
  planning-artifacts/       architecture.md, PRD, etc. (currently empty)
  implementation-artifacts/ (currently empty)
  test-artifacts/           Test design / reviews / traceability (currently empty)
design-artifacts/           WDS pipeline outputs: A-Product-Brief → E-Development (currently empty)
_bmad/                      BMad-Method tooling (installer-managed; gitignored except as listed below)
  config.toml               Installer-managed — DO NOT edit by hand. Use _bmad/custom/ for durable overrides.
  config.user.toml          Personal install answers (also installer-managed).
  custom/config.toml        Team overrides committed to repo (safe to edit).
  custom/config.user.toml   Personal pinned overrides (gitignored).
  scripts/resolve_config.py Resolves layered config.
```

Note: `.gitignore` excludes `_bmad/` broadly *but* this repo commits `_bmad/` anyway — treat the installer-managed files as read-only and put durable changes in `_bmad/custom/`.

## Environment

- **Windows 11 + PowerShell 5.1** is the assumed shell. Do not assume PowerShell 7 or POSIX bash unless the user opts in. Use `$null`, `$env:VAR`, `;` chaining (`&&`/`||` are parse errors in 5.1).
- Deploy target: **GCP only**. Do not propose AWS/Azure equivalents.
- Output language for BMad-generated documents: **Español** (`document_output_language` in `_bmad/config.toml`). The user (`Jonathan`) prefers Spanish for artifacts; code identifiers stay in English unless project-context.md says otherwise.

## Data conventions (non-obvious, high-impact on correctness)

These are the conventions you will trip over if you skim. Full list in `_producto/project-context.md`.

- **Dates in source CSVs:** `dd/MM/yyyy` or `dd/MM/yyyy HH:mm` (Mexican format). Never parse as ISO. In Java: `DateTimeFormatter.ofPattern("dd/MM/yyyy")` with `Locale.of("es","MX")`.
- **CSV encoding:** try UTF-8 BOM first, fall back to `cp1252`. Corporate exports vary. Log the detected encoding per file.
- **Decimal separator:** `.` in CSVs reviewed so far, but XLSX exports may use `,` — sample before parsing.
- **Money:** `java.math.BigDecimal` with scale=4 in intermediates. **Never** `double`/`float`. Currency is MXN.
- **Store IDs and SKU IDs:** treat as `String` even when they look numeric (leading zeros / future prefixes).
- **Empty vs null in CSVs:** `""` is a legitimate empty value (e.g. `papalEspacial` column in ALLOC); do not collapse to `null` without an explicit decision.
- **Fail-loud is the default:** missing equivalence, missing budget, or ambiguous encoding → checked exception and abort that store's cycle. Never silently default to 0, never interpolate.

## Testing discipline (will apply once code exists)

Captured here because it constrains how the eventual code is shaped — not yet executable.

- Golden datasets versioned at `src/test/resources/fixtures/golden/vN/` with `EXPECTED_OUTPUT.csv` signed by the pilot buyer + `DECISION_LOG.md`. `vN/` is immutable once shipped; bump to `v(N+1)` to change.
- Property-based testing with **jqwik** over quantization and budget invariants.
- Contract testing between Spring Boot and Angular via **Pact** or **Spring Cloud Contract** — YAML contract defined before either side implements.
- Backtesting suite (`BacktestingSuite.java`) on real historical CSVs — this is where the MAPE/WAPE number agreed with Finanzas comes from, not from a slide.
- Mutation testing with **PIT** on `forecasting.*`, ≥80% mutants killed before merge to `main`.

## Commands

There is no build system yet, so no `build`, `test`, `lint`, or `run` commands exist. The only tooling currently runnable is the BMad config resolver:

```powershell
python _bmad/scripts/resolve_config.py
```

Once `architecture.md` lands, the Maven / npm / Angular CLI commands belong here.

## Working with BMad in this repo

The BMad-Method skills are surfaced through Claude's Skill tool (look for `bmad-*` entries in the skill list). Highlights for this project:

- `_producto/project-context.md` is generated by the BMad workflow and is the single source of truth for "agent rules". Update it through the right BMad skill rather than hand-editing whenever possible.
- The user works in Spanish; respond and write artifacts in Spanish unless the user switches language.
- BMad agents have personas (Mary/Analyst, John/PM, Winston/Architect, Amelia/Dev, Sally/UX, Murat/Test Architect, Freya/WDS-UX, Saga/WDS-Analyst, Mimir/WDS-Builder, etc.). When the user addresses one by name, invoke the matching `bmad-agent-*` or `wds-agent-*` skill.
