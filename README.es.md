<h1 align="center">coodle-moodle53agentskills</h1>

<p align="center">
  <em>Tu agente de programación, experto en Moodle 5.3.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/moodle-5.3%20LTS-orange?style=flat-square" alt="Moodle 5.3 LTS">
  <img src="https://img.shields.io/badge/skills-4-blue?style=flat-square" alt="4 skills">
  <img src="https://img.shields.io/badge/references-36-blue?style=flat-square" alt="36 archivos de referencia">
  <img src="https://img.shields.io/badge/license-GPL--3.0-green?style=flat-square" alt="Licencia GPL-3.0">
</p>

<p align="center"><a href="README.md">English</a> · <a href="README.es.md">Español</a></p>

---

Si le pides a un agente de programación genérico que cree un plugin de Moodle, obtendrás un paquete PHP genérico: cadenas de texto codificadas, SQL directo contra nombres de tablas con prefijo, ausencia de comprobaciones de capacidades, ningún `db/upgrade.php` y un estilo que falla phpcs desde la primera línea. Si se lo pides para *Moodle 5.3*, es peor: sus datos de entrenamiento son anteriores a la reestructuración del directorio `public/`, Bootstrap 5, PHPUnit 11, los scopes obligatorios en rutas y una larga lista de APIs renombradas o eliminadas.

Este repositorio resuelve ese problema. Es un conjunto de **skills para agentes** destiladas sin conexión a partir de la documentación oficial de desarrollo de Moodle 5.3 — [moodledev.io/docs/5.3](https://moodledev.io/docs/5.3) y la [política de estilo de código](https://moodledev.io/general/development/policies/codingstyle) — junto con las notas `UPGRADING.md` del núcleo para 5.0, 5.1, 5.2 y 5.3. Está empaquetado para que pueda usarlo un agente compatible con Claude Code, Codex o `AGENTS.md`. El agente carga un router compacto por tema y solo los archivos de referencia que necesita la tarea: sin descargar contenido de la red, sin boilerplate de Docusaurus y sin adivinar a partir de datos de entrenamiento desactualizados.

Esta es la edición 5.3 de [coodle-moodle45agentskills](https://github.com/catalanml/coodle-moodle45agentskills). Es un repositorio y un plugin separados, así que ambas ediciones pueden instalarse a la vez.

## Tabla de contenidos

- [Cómo funciona](#cómo-funciona)
- [Qué incluye](#qué-incluye)
- [Qué cambió respecto a la edición 4.5](#qué-cambió-respecto-a-la-edición-45)
- [Instalación](#instalación)
- [Sesión de ejemplo](#sesión-de-ejemplo)
- [Principios de diseño](#principios-de-diseño)
- [Precisión y verificación](#precisión-y-verificación)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Actualizar el contenido](#actualizar-el-contenido)
- [Contribuir](#contribuir)
- [Licencia](#licencia)

## Cómo funciona

Cada skill es una carpeta con un router `SKILL.md` y un directorio `references/`. La descripción del frontmatter del router indica al agente *cuándo* cargarla; el cuerpo del router le indica *qué* archivo de referencia cubre la tarea. Una pregunta sobre `$DB->get_records()` carga una referencia de base de datos de unas 370 líneas, no todo el conjunto.

```
Tú: «Añade a mi plugin local una ruta de API REST que liste las insignias de un usuario».

El agente carga: moodle-plugin-development → references/common-files.md          (esqueleto del plugin)
                 moodle-core-apis           → references/webservices-advanced.md  (#[route], #[scopeset])
                                            → references/access-roles.md          (comprobación de capacidades)
                 moodle-coding-style        → SKILL.md                            (mientras escribe)
```

Las skills se activan automáticamente: nunca tienes que invocarlas manualmente.

## Qué incluye

### `moodle-plugin-development`

Anatomía de plugins, requisitos de cada tipo y la guía de migración de 4.5 a 5.3.

| Referencia | Cubre |
|---|---|
| `common-files.md` | Nombres frankenstyle, version.php, lang/, todos los archivos db/, autoload de classes/, settings.php, lib.php, soporte de composer.json, la estructura `public/` de 5.1 |
| `upgrading-to-5.3.md` | **Nueva.** Lista de cambios obligatorios y tablas Antiguo → Nuevo de APIs eliminadas o renombradas en 5.0, 5.1, 5.2 y 5.3 |
| `activity-modules.md` | Plugins mod: callbacks de lib.php, flags FEATURE_* (incl. `FEATURE_MOD_OTHERPURPOSE`), integración con la página de resumen de actividades, cadenas `modulename_summary`/`_tip` |
| `blocks.md` / `local-plugins.md` | API de block_base y configuración de instancia; estructura de plugins locales y cuándo usarlos |
| `themes.md` | config.php, herencia de Boost, SCSS, Bootstrap 5, variables de color de iconos de actividad, plantillas reubicadas, eliminación de Classic |
| `filters.md` / `course-formats.md` | `\core_filters\text_filter`; `core_courseformat\base`, clases de salida, navegación lineal de 5.3, cambios en el selector de actividades |
| `repositories.md` / `enrolment.md` | Contrato de la clase repository; flujo de enrol_plugin y API del subsistema |
| `question-types.md` | Plugins qtype y qbank, restauración de qtype, **plugins de reglas de acceso al cuestionario (`quizaccess_`)**, bancos de preguntas a nivel de módulo |
| `other-plugin-types.md` | Catálogo de tipos de plugin de 5.3, proveedores/ubicaciones de IA, tiny, antivirus, availability, customfield, sms, deprecación de tipos de plugin |
| `subplugins.md` | db/subplugins.json con `subplugintypes`, tipos de campo de base de datos, herramientas de libro, pruebas Behat de TinyMCE, migración de formatos de curso |
| `workflow-guides.md` | Flujo de trabajo con Git, reestructuración `public/`, API de actualización, profiling, migración a Bootstrap 5, requisitos de 5.3 |

### `moodle-core-apis`

Una referencia por API del núcleo; el router asigna tarea → archivo.

| Referencia | Cubre |
|---|---|
| `database.md` | Métodos $DB/DML, placeholders, transacciones, helpers sql_*, XMLDB/DDL (Oracle eliminado) |
| `forms.md` | Ciclo de vida de moodleform, catálogo de elementos, validación, elementos de archivo |
| `output-templates.md` | $PAGE/$OUTPUT, renderers, renderables, Mustache, humandate/humantimeperiod |
| `events-tasks.md` | Clases de eventos y observadores; tareas programadas y adhoc |
| `caching.md` / `files.md` | Definiciones MUC y cachés versionadas; File API, pluginfile, áreas de borrador |
| `access-roles.md` | Contextos, capacidades, has_capability/require_login |
| `webservices.md` | Funciones `\core_external\external_api`, db/services.php, core/ajax |
| `webservices-advanced.md` | Subsistema de routing, **`#[scopeset]` / `#[unscoped_resource]` obligatorios**, clases de scope, shortlinks, guía de estilo REST, pruebas y archivos de WS |
| `react.md` | **Nueva.** Componentes React: herramientas de build, import map, auto-init, temas, pruebas, profiler |
| `privacy.md` | Clases provider de GDPR, metadatos, exportación/borrado |
| `hooks-di.md` | Hooks API, inyección de dependencias, \core\clock, atributos de deprecación |
| `javascript.md` | Módulos AMD/ESM, grunt, core/str, `Modal.create()`, proceso de deprecación de JS |
| `javascript-advanced.md` | Librería reactive, comboboxsearch, reconocimiento de YUI heredado |
| `backup-restore.md` | Tareas y stepslib de backup/restore para actividades, bloques y herramientas de administración |
| `more-subsystems.md` | Editor, plagio, xAPI, favoritos, tiempo, comunicación, IA, SMS, insignias, OpenTelemetry, availability, analytics, Check API |
| `files-advanced.md` | Exploración de archivos, conversores de documentos, almacenamiento interno, drivers DML, utilidades CLI de privacidad |
| `other-apis.md` | Cadenas, mensajería, navegación, report builder, campos personalizados, comentarios, ajustes de administración, calendario, etiquetas, grupos, locks |

### `moodle-coding-style`

El estilo de código oficial para PHP, JavaScript y SQL en `SKILL.md`, más la **nueva** `references/css.md` con la política de estilo CSS/SCSS.

### `moodle-testing`

| Referencia | Cubre |
|---|---|
| `phpunit.md` | PHPUnit 11 (desde 5.0), atributos vs anotaciones, generadores, sinks, cómo arreglar pruebas rotas |
| `behat.md` | Estructura de .feature, etiquetas, pasos personalizados, generadores de entidades |
| `ci-tools.md` | phpcs + moodle-cs, moodle-plugin-ci en GitHub Actions con la matriz PHP/BD de 5.3, referencias de QA testing |

## Qué cambió respecto a la edición 4.5

- **3 referencias nuevas:** `upgrading-to-5.3.md`, `react.md`, `moodle-coding-style/references/css.md`.
- **32 páginas nuevas** de la documentación incorporadas: reglas de acceso al cuestionario, resumen de actividades, navegación lineal, scopes y shortlinks de rutas, la guía de estilo REST, React (7 páginas), soporte de Composer, la reestructuración del código, las guías de frontend y de deprecación de JS, humandate, comentarios, insignias, OpenTelemetry, restauración de qtype, la guía para escribir pruebas PHPUnit y QA testing.
- **Todas las referencias existentes se volvieron a destilar** contra el árbol 5.3 y se contrastaron con las notas `UPGRADING.md` de 5.0–5.3. Muchas páginas de devdocs no han cambiado desde 4.5 aunque la API sí, así que a menudo las notas son la única fuente de un cambio. Cuando corresponde, cada referencia empieza con un resumen de los cambios desde 4.5.

## Instalación

El repositorio usa un único directorio `skills/` compartido y manifiestos ligeros para cada integración de agente compatible.

### Claude Code

```bash
/plugin marketplace add catalanml/coodle-moodle53agentskills
/plugin install coodle-moodle53agentskills@coodle53
```

El marketplace se llama `coodle53` para no colisionar con el marketplace `coodle` de la edición 4.5. También puedes copiar las skills al directorio de skills de un proyecto sin publicarlas. Usa un directorio a nivel de proyecto si también conservas las skills de 4.5, porque los nombres de las skills son los mismos:

```bash
cp -r skills/* /ruta/a/tu/proyecto/.claude/skills/
```

### Codex

`.codex-plugin/plugin.json` declara el directorio compartido `skills/`. Instala el repositorio como plugin desde la interfaz de plugins de Codex (`/plugins`) o apunta Codex a un clon local. `AGENTS.md` le da a Codex las instrucciones de enrutamiento incluso sin soporte de plugins.

### Otros agentes (`AGENTS.md`)

Cualquier agente que lea `AGENTS.md` (OpenCode, Cursor, Copilot CLI, …) puede usar el conjunto: clona este repositorio en una ubicación estable y añade una línea a las instrucciones de agente de tu proyecto:

```markdown
Antes de escribir código para Moodle 5.3, lee /ruta/a/coodle-moodle53agentskills/AGENTS.md
y sigue el enrutamiento de skills que describe.
```

Las skills son markdown plano: sin hooks, sin scripts y sin dependencias de ejecución.

## Sesión de ejemplo

> **Tú:** Mi formato de curso de 4.5 falla en 5.3. Arréglalo y añade navegación lineal.
>
> **Agente:** *carga `moodle-plugin-development` (upgrading-to-5.3, course-formats). Sustituye los menús de sección basados en arrays por `core_courseformat\output\local\content\section\controlmenu` con objetos `action_menu_link`, cambia los enlaces `addModule` por `newModule` con `data-sectionid`, deja de depender del ajuste obsoleto `maxsections`, sobrescribe `uses_linear_navigation()` y sube `$plugin->requires` a la versión del núcleo 5.3.0.*

## Principios de diseño

- **Destilado, no copiado.** Cada archivo de referencia es una reescritura compacta y autocontenida (100–470 líneas) con los datos esenciales: nombres de clases, firmas, rutas de archivos y ejemplos de código reales.
- **Router + referencias.** Los archivos `SKILL.md` son pequeños y apuntan exactamente a un archivo por tema.
- **Solo datos documentados.** El contenido proviene de la documentación oficial y de las notas de actualización del núcleo, no de la memoria del modelo. Cuando ambas fuentes callan, los archivos lo indican en lugar de inventar una API.
- **Fiel a 5.3 antes que a la documentación.** Cuando una página de devdocs está desactualizada (por ejemplo ModalFactory, eliminado en 5.2; Oracle, eliminado en 5.0; `set_section_number()`; Atto), las skills enseñan la API actual y señalan la discrepancia.

## Precisión y verificación

Fuentes:

- [moodle/devdocs](https://github.com/moodle/devdocs) `docs/` (el árbol 5.3, etiquetado `main (5.3)`) en el commit `0367ab8911f86646e1fefdf07a59237c43057be7`, 5 de octubre de 2026: 220 archivos Markdown/MDX (los 188 del árbol 4.5 más 32 nuevos). 75 archivos cambiaron o se añadieron respecto al árbol 4.5 (incluidas imágenes y fragmentos de código).
- Las páginas no versionadas de estilo de código, PHP, PHPUnit y Behat en `general/development/` en el mismo commit.
- `UPGRADING.md` de `moodle/moodle`, rama `MOODLE_503_STABLE` (5.3 publicado el 5 de octubre de 2026), secciones 5.0 a 5.3.

Cada página de contenido del árbol 5.3 (excepto la documentación de la app móvil) está asignada a una referencia. Los datos esenciales se verificaron contra estas fuentes. Cuando devdocs y UPGRADING discrepan en un nombre de clase, namespace, firma o ruta, el conflicto se resolvió contra el código fuente de `MOODLE_503_STABLE` (por ejemplo `\core_course\cm_info` y no `\course\cm_info` como escribe UPGRADING; `\core\router\route`; `uses_linear_navigation()` estático; `public/admin/tool/behat/cli/`). La recuperación se probó con agentes nuevos en tareas específicas de 5.3 para cada skill.

## Limitaciones conocidas

- Muchas páginas de devdocs de 5.3 no han cambiado desde 4.5. Cuando UPGRADING indica una eliminación pero no un reemplazo, los archivos dicen «none listed» en lugar de adivinar.
- La API del libro de calificaciones (`grade_update()`), la API de eventos completa y los bancos de preguntas a nivel de módulo (`mod_qbank`) tienen poca o ninguna cobertura en devdocs; se documentan a partir de UPGRADING o se señalan.
- Algunas APIs aparecen solo por nombre en UPGRADING (por ejemplo, los métodos de reintento suave de tareas y algunos namespaces de hooks); los archivos las marcan como no verificadas.
- La documentación de la app de Moodle (móvil) queda fuera del alcance a propósito.

## Actualizar el contenido

Para actualizar: vuelve a clonar devdocs. Cuando 5.3 tenga versión propia, sus páginas pasarán de `docs/` a `versioned_docs/version-5.3/`. Compara con el commit `0367ab8`, relee las páginas modificadas y las nuevas entradas de `UPGRADING.md` en `MOODLE_503_STABLE`, y vuelve a verificar. 5.3 es una versión LTS, así que los cambios tras su publicación deberían ser lentos.

## Contribuir

1. Haz un fork del repositorio y crea una rama desde `main`.
2. Respeta las reglas: markdown de GitHub plano, sin sintaxis de Docusaurus, datos tomados de la documentación oficial o de las notas de actualización (cita la página fuente en tu PR) y routers breves.
3. Un tema por archivo de referencia; la amplitud va en los archivos `other-*`.

## Licencia

GPL-3.0-or-later. Contenido derivado de la documentación para desarrolladores de Moodle y del núcleo de Moodle, que forman parte del proyecto Moodle (GPL).
