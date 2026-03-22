# Guía de operación y contribución

Este documento define las reglas operativas para actualizar y extender ai-dev-os.

---

## 1. Política de actualización por sección

### 1.1 Frecuencia de actualización y alcance de impacto

| Sección | Frecuencia de actualización | Alcance de impacto | Notas |
|---------|----------------------------|-------------------|-------|
| `01_philosophy/` | Extremadamente rara | Todos los proyectos | Los cambios en la filosofía central requieren un incremento de versión MAJOR |
| `02_decision-criteria/` | Rara | Todos los proyectos | Los cambios en criterios de decisión requieren verificaciones de consistencia con las directrices posteriores |
| `03_guidelines/common/` | Media | Todos los proyectos | Los cambios en estándares comunes requieren verificación del impacto en todos los frameworks |
| `03_guidelines/frameworks/` | Alta | Proyectos que usan el FW relevante | Mantener al día con las actualizaciones del framework |
| `04_ai-prompts/` | Media | Usuarios de asistentes IA | Las mejoras de prompts se pueden hacer libremente |
| `templates/` | Media | Solo proyectos nuevos | Sin impacto en proyectos existentes |

### 1.2 Reglas para actualizaciones

**Común a todas las secciones:**

- No incluir descripciones específicas del proyecto (nombres de servicios específicos, terminología de dominio específica)
- Cuando se necesiten ejemplos concretos, usar ejemplos genéricos con marcadores de posición como `{domain}`, `{Payment Service}`, etc.
- Después de los cambios, actualizar la estructura de directorios en `README.md` para reflejar el estado más reciente

**Política de idiomas:**

- `01_philosophy/` y `02_decision-criteria/` contienen **contenido de ejemplo en inglés** — después de clonar, **reescríbalos en su idioma nativo** (el pensamiento abstracto y los marcos de toma de decisiones se expresan mejor en el idioma nativo para preservar matices)
- Todas las demás secciones (`03_guidelines/`, `04_ai-prompts/`, `templates/`) deben escribirse en **inglés** — para compatibilidad con IA y accesibilidad internacional
- Las guías de operación multilingües se mantienen en `docs/i18n/` (JA, ZH, KO, ES)
- Al agregar o actualizar contenido, siempre seguir esta política de idiomas

**Actualizar `01_philosophy/`:**

- Contiene contenido de ejemplo (inglés). Después de clonar, **reescríbalo en su idioma nativo**
- Al agregar nuevos principios, verificar que no haya contradicciones con los principios existentes
- Las eliminaciones y cambios importantes se tratan como versiones MAJOR
- También verificar la consistencia con `02_decision-criteria/` y secciones inferiores

**Actualizar `02_decision-criteria/`:**

- Contiene contenido de ejemplo (inglés). Después de clonar, **reescríbalo en su idioma nativo**
- Al cambiar criterios de decisión, verificar si las secciones correspondientes en `03_guidelines/` también necesitan actualizaciones
- Al agregar nuevos ejes de decisión, verificar si existen directrices correspondientes

**Actualizar `03_guidelines/`:**

- Tener cuidado de no duplicar contenido entre `common/` y `frameworks/`
- Revisar periódicamente si el contenido que debería ser compartido está escrito solo en archivos específicos del framework
- Ver "2. Agregar directrices de framework" para más detalles

**Actualizar `04_ai-prompts/`:**

- Verificar que las rutas de directrices referenciadas en los prompts realmente existan
- `skills/` debe mantener el formato SKILL.md de Claude Code (frontmatter + procedures)

**Actualizar `templates/`:**

- Las plantillas son para proyectos nuevos. No aplicar automáticamente a proyectos existentes
- Los archivos de configuración (pyproject.toml, configuración de ruff, etc.) deben ser consistentes con los estándares de `03_guidelines/`

---

## 2. Agregar directrices de framework

### 2.1 Criterios de adición

Condiciones para agregar nuevas directrices de framework:

| Condición | Requerido/Recomendado |
|-----------|----------------------|
| Usado en 2 o más proyectos | Requerido |
| Patrones específicos del framework que no se pueden expresar en `common/` | Requerido |
| El framework está en versión estable (v1.0+) | Recomendado |
| Se espera mantenimiento a largo plazo | Recomendado |

### 2.2 Estructura de directorios

```text
03_guidelines/frameworks/{framework-name}/
├── overview.md            # Definición del stack tecnológico (requerido)
├── project-structure.md   # Estructura de directorios (requerido)
└── ...                    # Directrices específicas del framework
```

### 2.3 Pasos de adición

#### Step 1: Crear overview.md

El punto de entrada requerido para todas las directrices de framework. Incluir:

```markdown
# {Framework} Technology Stack

## Core
- Framework: {Name} v{Version}
- Language: Python (con type hints, mypy/pyright modo estricto)
- Package Manager: {uv / pip / poetry}

## Recommended Libraries
{Listar bibliotecas recomendadas por categoría}

## Prerequisites
{Especificar la relación con las directrices de common/}
```

#### Step 2: Crear project-structure.md

Directrices de estructura de directorios. Seguir estos principios:

- Aplicar tal cual los conceptos ya definidos en `common/` (vertical slices, reglas de dependencia, etc.)
- Solo describir las convenciones de directorio específicas del framework
- Usar marcadores de posición como `{domain}` para ejemplos concretos

#### Step 3: Agregar directrices específicas del framework

Relación con las directrices correspondientes de `common/`:

| Patrón | Enfoque |
|--------|---------|
| Se aplica el contenido común tal cual | No crear archivo en el lado del framework |
| Contenido que extiende lo común | Crear en el lado del framework, referenciando lo común y describiendo solo las diferencias |
| Conceptos específicos del framework | Crear solo en el lado del framework |

**Reglas de nomenclatura de archivos:**

- Usar el mismo nombre cuando corresponda con `common/` (ej. `common/security.md` → `python-cli/security.md`)
- Usar nombres descriptivos para conceptos específicos del framework (ej. `dependency-injection.md`, `cli-commands.md`)

#### Step 4: Crear plantillas

```text
templates/{framework-name}/
├── CLAUDE.md.template     # Plantilla CLAUDE.md (requerido)
├── submodule-setup.sh     # Script de configuración (recomendado)
├── .claude/skills/        # Claude Code skills (recomendado)
└── {configuration files}  # pyproject.toml, configuración de ruff, etc.
```

#### Step 5: Actualizar README.md

- Agregar el nuevo framework a la estructura de directorios
- Agregar instrucciones específicas del framework a la sección de introducción si es necesario

### 2.4 Principio de separación de responsabilidades con common/

```text
common/          → "Qué hacer" (reglas independientes del lenguaje/FW)
frameworks/xxx/  → "Cómo implementar" (patrones de implementación específicos del FW)
```

### Ejemplo: Validación

- `common/validation.md` → "La validación del lado del servidor es obligatoria" "Se recomienda Pydantic"
- `frameworks/python-cli/validation.md` → "Patrón de validación con modelos Pydantic y análisis de argumentos Typer"
- `frameworks/fastapi/validation.md` → "Validación del cuerpo de solicitud con modelos Pydantic e inyección de dependencias"

### Ejemplo: Manejo de errores

- `common/error-handling.md` → "Clasificación de errores" "Principios para la visualización al usuario"
- `frameworks/python-cli/error-handling.md` → "Jerarquía de excepciones personalizadas con códigos de salida Typer"

### 2.5 Lista de verificación de adición

- [ ] Se han creado overview.md y project-structure.md
- [ ] No se incluyen descripciones específicas del proyecto
- [ ] No hay duplicación de contenido con `common/`
- [ ] Se incluyen referencias apropiadas a `common/` donde corresponda
- [ ] Se ha creado `templates/{framework}/CLAUDE.md.template`
- [ ] Se ha actualizado la estructura de directorios en `README.md`
- [ ] Se publica como versión MINOR o superior

---

## 3. Versionado y gestión de cambios

### 3.1 Determinar el tipo de cambio

| Cambio | Versión | Ejemplo |
|--------|---------|---------|
| Cambios fundamentales en la filosofía de diseño | MAJOR | Cambiar los tres pilares, invertir reglas de dependencia |
| Cambios importantes en criterios de decisión | MAJOR | Cambios fundamentales en umbrales de abstracción |
| Agregar directrices de framework | MINOR | Crear `frameworks/fastapi/` |
| Mejorar/extender directrices existentes | MINOR | Agregar nuevos patrones a directrices de seguridad |
| Agregar prompts/skills | MINOR | Agregar nuevas definiciones de skills |
| Correcciones tipográficas/mejoras de redacción | PATCH | Corregir erratas, reemplazar ejemplos |
| Cambios solo en plantillas | PATCH | Ajustes menores en archivos de configuración |

### 3.2 Manejo de cambios incompatibles

Los siguientes se tratan como cambios incompatibles que requieren una versión MAJOR:

- Eliminación o renombrado de archivos (rompe las referencias de ruta en CLAUDE.md)
- Cambios en la estructura de directorios
- Abolición o inversión de reglas existentes

**Al mover archivos:**

1. Dejar un archivo de redirección de una línea en la ruta anterior: "Moved to: `new-path`"
2. Eliminar el archivo de redirección en la siguiente versión MAJOR

---

Languages: [English](../../operation-guide.md) | [日本語](../ja/operation-guide.md) | [简体中文](../zh-CN/operation-guide.md) | [한국어](../ko/operation-guide.md) | Español
