# AI Dev OS Rules — Python

[![Lint & Link Check](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml/badge.svg)](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)

> Directrices de 4 capas para proyectos Python (FastAPI, Flask, Typer CLI, etc.).
> Se agrega como git submodule, referenciado por asistentes de codificación IA.

**Parte del ecosistema [AI Dev OS](https://github.com/yunbow/ai-dev-os).**

## ¿Por qué estas reglas?

AI Dev OS Rules proporcionan a su asistente de codificación IA **estándares concretos y verificables** en lugar de instrucciones vagas:

- **13 reglas comunes** — Nomenclatura, manejo de errores, seguridad, pruebas, registro, i18n y más
- **Reglas específicas del framework** — Patrones de FastAPI, mejores prácticas de Python CLI
- **Resolución de conflictos integrada** — Specificity Cascade resuelve la prioridad de reglas automáticamente
- **Versionado y auditable** — Fijar a una etiqueta, ver diferencias, revisar en PRs

## Inicio rápido

```bash
npx ai-dev-os init --rules python --plugin claude-code
```

> Configura todo automáticamente. Consulte [AI Dev OS CLI](https://github.com/yunbow/ai-dev-os-cli).

<details>
<summary>Configuración manual</summary>

### Agregar como submodule

```bash
cd /path/to/your-project
git submodule add https://github.com/yunbow/ai-dev-os-rules-python.git docs/ai-dev-os
git submodule update --init
```

### Configurar con plantillas (Python CLI)

```bash
bash docs/ai-dev-os/templates/python-cli/submodule-setup.sh
```

### Editar CLAUDE.md

Copie `templates/python-cli/CLAUDE.md.template` como `./CLAUDE.md` y complete el nombre del proyecto y las directrices específicas.

### Actualizar submodule

```bash
git submodule update --remote docs/ai-dev-os
```

</details>

## Contenido incluido

| Capa | Ruta | Contenido |
|------|------|-----------|
| L1 — Filosofía | `01_philosophy/` | Principios, modelos mentales, antipatrones |
| L2 — Criterios de decisión | `02_decision-criteria/` | Abstracción, selección tecnológica, arquitectura, errores, seguridad |
| L3 — Directrices comunes | `03_guidelines/common/` | 13 reglas: código, nomenclatura, validación, errores, registro, seguridad, pruebas, etc. |
| L3 — Directrices del framework | `03_guidelines/frameworks/` | [FastAPI](../../../03_guidelines/frameworks/fastapi/README.md), [Python CLI](../../../03_guidelines/frameworks/python-cli/README.md) |
| Plantillas | `templates/` | [Scaffolding de Python CLI](../../../templates/README.md) |

## Specificity Cascade

Cuando las reglas entran en conflicto, **el número menor gana**.

| Prioridad | Capa | Ejemplo |
|-----------|------|---------|
| 1 (Más alta) | Directrices específicas del framework | `03_guidelines/frameworks/python-cli/*` |
| 2 | Directrices comunes | `03_guidelines/common/*` |
| 3 | Criterios de decisión | `02_decision-criteria/*` |
| 4 | Filosofía de diseño | `01_philosophy/*` |

<details>
<summary>Estructura de directorios</summary>

```text
ai-dev-os/
├── docs/
│   ├── operation-guide.md        # Guía de operación y contribución
│   └── i18n/                     # Guías multilingües
│       ├── ja/                   #   日本語
│       ├── zh-CN/                #   简体中文
│       ├── ko/                   #   한국어
│       └── es/                   #   Español
│
├── 01_philosophy/                # Filosofía de diseño [Ejemplo - reescribir en su idioma nativo]
│   ├── principles.md             #   Tres pilares: Correctness, Observability, Pragmatism
│   ├── mental-models.md          #   10 marcos de pensamiento
│   └── anti-patterns.md          #   Patrones a evitar (con ejemplos de código)
│
├── 02_decision-criteria/         # Criterios de decisión [Ejemplo - reescribir en su idioma nativo]
│   ├── abstraction.md            #   Momento y umbrales para la abstracción
│   ├── technology-selection.md   #   Marco de selección tecnológica
│   ├── architecture.md           #   Diseño de API, gestión de estado, organización de módulos
│   ├── error-strategy.md         #   Clasificación de errores, reintentos, patrón Result
│   └── security-vs-ux.md        #   Prioridad y equilibrio de medidas de seguridad
│
├── 03_guidelines/                # Directrices [Inglés]
│   ├── common/                   #   Comunes (independientes de lenguaje/FW)
│   │   ├── code.md               #     Estándares de codificación
│   │   ├── naming.md             #     Convenciones de nomenclatura
│   │   ├── validation.md         #     Validación
│   │   ├── error-handling.md     #     Manejo de errores
│   │   ├── logging.md            #     Registro
│   │   ├── security.md           #     Seguridad
│   │   ├── rate-limiting.md      #     Limitación de velocidad
│   │   ├── testing.md            #     Pruebas
│   │   ├── performance.md        #     Rendimiento
│   │   ├── cors.md               #     CORS
│   │   ├── env.md                #     Variables de entorno
│   │   ├── cicd.md               #     CI/CD
│   │   └── i18n.md               #     Internacionalización
│   │
│   └── frameworks/               #   Específicos del framework (ver cada README.md)
│       ├── fastapi/              #     → [README.md](../../../03_guidelines/frameworks/fastapi/README.md)
│       └── python-cli/           #     → [README.md](../../../03_guidelines/frameworks/python-cli/README.md)
│
│
└── templates/                    # Plantillas de proyecto [Inglés]
    └── python-cli/               #     → [README.md](../../../templates/README.md)
```

</details>

<details>
<summary>Operaciones y versionado</summary>

Para políticas de actualización, pasos de adición de frameworks y detalles de versionado, consulte **[docs/operation-guide.md](../../../docs/operation-guide.md)**.

### Guía de frecuencia de actualización

| Sección | Frecuencia | Alcance |
|---------|-----------|---------|
| `01_philosophy/` | Extremadamente rara | Todos los proyectos (cambio MAJOR) |
| `02_decision-criteria/` | Rara | Todos los proyectos |
| `03_guidelines/common/` | Media | Todos los proyectos |
| `03_guidelines/frameworks/` | Alta | Solo proyectos del FW relevante |
| `templates/` | Media | Solo proyectos nuevos |

### Agregar frameworks

Para agregar un nuevo framework (ej., FastAPI, Flask, Django):

1. Crear `overview.md` y `project-structure.md` en `03_guidelines/frameworks/{framework}/`
2. Aplicar separación de responsabilidades con `common/` (reglas comunes → common, patrones específicos del FW → frameworks)
3. Preparar plantillas en `templates/{framework}/`
4. Actualizar la estructura de directorios en este README

Consulte [docs/operation-guide.md](../../../docs/operation-guide.md) para los pasos detallados y la lista de verificación.

**Versionado** — Gestionado con versionado semántico (etiquetas git).

| Tipo de cambio | Versión | Ejemplo |
|---------------|---------|---------|
| Cambios importantes en filosofía/criterios | MAJOR | v2.0.0 |
| Adición/mejora de directrices | MINOR | v1.1.0 |
| Correcciones tipográficas, complementos | PATCH | v1.0.1 |

Fijar el submodule a una etiqueta específica:

```bash
cd docs/ai-dev-os
git checkout v1.2.0
cd ../..
git add docs/ai-dev-os
git commit -m "chore: pin ai-dev-os to v1.2.0"
```

</details>

## Política de idiomas

- `01_philosophy/` y `02_decision-criteria/` contienen **contenido de ejemplo en inglés**. Después de clonar, **reescríbalos en su idioma nativo** para preservar los matices del pensamiento abstracto y los marcos de toma de decisiones de su equipo.
- Todas las demás secciones se escriben en **inglés** para la compatibilidad con IA y la accesibilidad internacional.
- Las guías de operación multilingües están disponibles en `docs/i18n/`.

## Relacionados

| Repositorio | Descripción |
|---|---|
| [ai-dev-os](https://github.com/yunbow/ai-dev-os) | Especificación y teoría del framework |
| [rules-typescript](https://github.com/yunbow/ai-dev-os-rules-typescript) | Directrices de TypeScript / Next.js / Node.js |
| [plugin-claude-code](https://github.com/yunbow/ai-dev-os-plugin-claude-code) | Skills, Hooks y Agents para Claude Code |
| [plugin-kiro](https://github.com/yunbow/ai-dev-os-plugin-kiro) | Steering Rules y Hooks para Kiro |
| [plugin-cursor](https://github.com/yunbow/ai-dev-os-plugin-cursor) | Cursor Rules (.mdc) para desarrollo guiado por directrices |
| [cli](https://github.com/yunbow/ai-dev-os-cli) | Automatización de configuración — `npx ai-dev-os init` |
| [benchmark](https://github.com/yunbow/ai-dev-os-benchmark) | Benchmark cuantitativo — impacto de directrices en calidad |

## Licencia

[MIT](../../../LICENSE)

---

Languages: [English](../../../README.md) | [日本語](../ja/README.md) | [简体中文](../zh-CN/README.md) | [한국어](../ko/README.md) | Español
