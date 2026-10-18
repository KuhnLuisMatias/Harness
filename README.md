# Agentes .NET para Claude Code

![Claude Code](https://img.shields.io/badge/Claude%20Code-subagentes-8A2BE2)
![.NET](https://img.shields.io/badge/.NET-C%23%20%7C%20ASP.NET%20Core-512BD4)
![EF Core](https://img.shields.io/badge/EF%20Core-migraciones-68217A)
![Estado](https://img.shields.io/badge/estado-plantilla%20en%20uso-informational)

Configuración multiagente para **Claude Code** orientada a proyectos **.NET (C# / ASP.NET Core con EF Core)**. Un **coordinador** que nunca ejecuta alinea el pedido con el usuario, aprueba el plan y delega el trabajo en cuatro subagentes especializados. El flujo es *spec-first*, con TDD, verificación en capas y disciplina de evidencia.

## ¿Qué es esto?

No es un proyecto .NET: es un conjunto de **reglas y subagentes** que le dan a Claude Code un proceso de trabajo reproducible sobre un repositorio .NET.

- **Nadie declara éxito sin evidencia.** Toda afirmación técnica se rastrea con `archivo:línea` o `comando → resultado`.
- **El que planifica no ejecuta; el que ejecuta no se aprueba a sí mismo.** Separación estricta de roles.
- **Los cambios se prueban contra la especificación**, no contra el código.
- **Git es de solo lectura** para los agentes: no hay `commit`, `push`, `rebase`, `reset`, `checkout`, `switch`, `merge`, `pull`, `revert`, `cherry-pick` ni `apply`.

## Roles

| Agente | Modelo | Hace | Tools |
| --- | --- | --- | --- |
| `coordinador` | opus | Entiende el pedido, alinea, aprueba el plan y delega. **Nunca ejecuta.** | `Agent(...)`, `Read`, `Grep`, `Glob`, `AskUserQuestion` |
| `analisis` | opus | Mapea código, diagnostica bugs, redacta `ISA`/`PLAN`/`ESTADO`. Solo escribe en `docs/`. | `Read`, `Grep`, `Glob`, `Bash`, `Write`, `Edit` |
| `implementacion` | sonnet | Escribe el código del plan, test primero, solo en los archivos permitidos. | `Read`, `Grep`, `Glob`, `Edit`, `Write`, `Bash` |
| `pruebas` | sonnet | Traduce las specs `S#` a tests y emite veredicto con evidencia. | `Read`, `Grep`, `Glob`, `Edit`, `Write`, `Bash` |
| `adversario` | opus | Intenta romper el cambio antes de cerrarlo. Solo lee y verifica. | `Read`, `Grep`, `Glob`, `Bash` |

El coordinador es el **agente principal** (setting `agent` en `settings.json`) y solo puede delegar en esos cuatro subagentes.

## Flujo de trabajo

1. **Autorizar** — Explicar, investigar, comparar y revisar son solo lectura: se delegan a `analisis` y se termina.
2. **Contexto** — Se lee `CLAUDE.md`, `docs/adr/`, `docs/external/`, `docs/LECCIONES.md` si existen.
3. **Alineación** — Pedido concreto: se sigue. Pedido ambiguo: 3 preguntas específicas en una sola llamada y se frena.
4. **Planificar** — Cambio trivial: plan de 3 líneas en el chat. Cambio real: `analisis` redacta `ISA.md` y `PLAN.md` (su escritura pide confirmación), y el usuario aprueba explícitamente. **Nada se toca antes de la aprobación.**
5. **Implementar** — Por cortes verticales; primero el flujo mínimo end-to-end.
6. **Probar** — `pruebas` valida cada corte contra las specs `S#`.
7. **Estado** — `analisis` actualiza `docs/<feature>/ESTADO.md`.
8. **Atacar** — En alto riesgo, `adversario` antes de cerrar.
9. **Corregir con tope** — Máximo 3 intentos por problema. Lo que siga abierto pasa a `necesita_decision`.

## Estructura del repositorio

```
.
├─ .claude/
│  ├─ agents/
│  │  ├─ coordinador.md      # orquesta y delega; no ejecuta
│  │  ├─ analisis.md         # investigación, ISA/PLAN/ESTADO
│  │  ├─ implementacion.md   # código del plan, test primero
│  │  ├─ pruebas.md          # tests y veredicto por S#
│  │  └─ adversario.md       # revisión adversarial
│  └─ settings.json          # agente por defecto + permisos
└─ CLAUDE.md                 # reglas comunes para todos los agentes
```

Documentos que los agentes generan **por feature** (no incluidos en este repo):

```
docs/
├─ <feature>/
│  ├─ ISA.md        # qué y por qué (problema, specs S#, aceptación)
│  ├─ PLAN.md       # cómo (diseño, archivos permitidos, cortes T#, verificación)
│  ├─ ESTADO.md     # seguimiento (hecho / pendiente / bloqueos / próxima acción)
│  └─ EVIDENCIA.md  # pruebas ejecutadas y sus resultados
├─ adr/             # decisiones de arquitectura
├─ external/        # documentación externa de referencia
└─ LECCIONES.md     # aprendizajes reutilizables (fecha | incidente | clase | guarda)
```

## Requisitos

- [Claude Code](https://code.claude.com/docs) (conviene una versión moderna, por el uso de subagentes y `Agent`).
- .NET SDK (el que use el proyecto: 6, 8, ...).
- Git.

## Cómo empezar

1. Cloná este repo (o copiá `.claude/` y `CLAUDE.md` a la raíz de tu proyecto .NET).
2. Abrí Claude Code en la raíz del repositorio. El agente por defecto es `coordinador`.
3. Describí lo que querés hacer. El coordinador decide si alcanza con leer o si hay que planificar, implementar y probar.

```bash
# Opción explícita: arrancar con el coordinador como agente principal
claude --agent coordinador
```

> `Agent(analisis, implementacion, pruebas, adversario)` funciona cuando el coordinador corre como agente principal, que es el caso de uso previsto.

## Permisos y seguridad

`settings.json` define la política en tres niveles:

- **`allow`** — sin preguntar: lectura de git (`status`, `diff`, `log`, `blame`, `show`), `dotnet build`, `dotnet test` y `dotnet format --verify-no-changes`.
- **`ask`** — requiere confirmación: `git stash` y `git branch`; `mv`, `rm`, `rmdir`; `dotnet ef migrations add`, `dotnet ef database update`; `dotnet run`; y la edición de los documentos que exigen aprobación (`CLAUDE.md`, `docs/adr/**`, `docs/LECCIONES.md`, `**/ISA.md` y `**/PLAN.md`).
- **`deny`** — bloqueado: `git commit`/`push`/`rebase`/`reset`/`clean`/`checkout`/`restore`/`switch`/`merge`/`pull`/`revert`/`cherry-pick`/`apply`/`am`/`stash drop`/`stash clear`; `dotnet ef database drop`; edición de `.claude/**`; y **lectura de secretos** (`.env*`, `*.pem`, `*.key`, `*.pfx`, `*.p12`, `secrets/**`, `.ssh/**`, `credentials.json`, `service-account*.json`, `.aws/credentials`, los user secrets de .NET y `appsettings.Production/Staging/Local.json`).

`appsettings.Development.json` **no** está bloqueado a propósito: los agentes necesitan leer la configuración local. Si tu entorno guarda secretos reales ahí, movelos a user secrets.

Un `dotnet format` mutante (sin `--verify-no-changes`) no está en la lista blanca y cae en confirmación.

### Límites de esta política

- Las reglas `Read(...)` aplican a las herramientas de archivo de Claude Code, **no** a comandos de Bash (`cat .env`, `type`, `Get-Content`). En modo `default` esos comandos piden confirmación, pero no están bloqueados.
- `dotnet build`, `dotnet test` y `dotnet run` ejecutan código del repositorio con acceso al entorno y a la configuración. Sin sandbox, la política no impide que ese código lea o imprima secretos: `dotnet run` quedó en `ask` por ese motivo.
- La política de `settings.json` no impone restricciones **por agente** (por ejemplo, "`pruebas` solo escribe tests"): eso vive en el prompt.
- Los documentos protegidos están en `ask`, no en `deny`, para que el flujo de análisis pueda crearlos y actualizarlos sin saltear la aprobación del usuario.

## Endurecimiento recomendado (fuera de este repo)

`settings.json` no reemplaza un sandbox. Para un proyecto con secretos reales, sumá:

- **Contenedor / sandbox** para ejecutar `dotnet` sin acceso a las credenciales del host.
- **Hooks `PreToolUse`** en `.claude/settings.json` que rechacen lecturas de secretos por Bash y operaciones de git mutante más allá del allowlist.
- **User secrets o variables de entorno** para el desarrollo local, y `appsettings.Development.json` sin secretos reales.

## Estados de operación

Todo cierre usa uno de estos estados, sin inventar éxito parcial:

| Estado | Significa |
| --- | --- |
| `completado` | Criterios de aceptación cumplidos y evidencia suficiente. |
| `completado_con_advertencias` | Objetivo cumplido, con advertencias no bloqueantes. |
| `bloqueado` | No puede continuar por validación, seguridad, permisos o falta de evidencia. |
| `necesita_decision` | Falta una decisión humana para continuar. |
| `fallido` | Evidencia de que terminó sin cumplir el objetivo. |
| `resultado_desconocido` | No se puede determinar si se produjeron efectos. |

## Convenciones .NET asumidas

El agente de implementación replica lo existente en el proyecto, no impone estilos nuevos:

- Nombres: `PascalCase` para métodos/propiedades, `camelCase` para parámetros, `_camelCase` para privados.
- Manejo de errores: *exceptions*, *Result types* o *error codes* según el proyecto.
- Logging: `ILogger` / Serilog / Application Insights.
- Tests: xUnit / NUnit / MSTest (los que ya use el repo).
- DI: *constructor injection* / *factory methods*.
- Verificación en capas: **build → warnings → formato → tests**.

## Límites y notas

- Las restricciones **por agente** de archivos ("`pruebas` solo escribe tests", "el coordinador no lee `src/`") viven en el **prompt**: Claude Code no las impone a nivel de subagente. Lo único global y forzado es `settings.json`.
- Los documentos que requieren aprobación (`CLAUDE.md`, `docs/adr/**`, `docs/LECCIONES.md`, `ISA.md`, `PLAN.md`) están en `ask`: cada escritura pide confirmación, además de la aprobación del plan.
- `permissions.allow` de un archivo de proyecto requiere **confiar la carpeta** para aplicar; `deny` y `ask` aplican de inmediato.
- El allowlist `Agent(...)` aplica solo cuando el coordinador es el agente principal.
- No hay hooks ni sandbox: los límites por agente y el "este agente solo escribe acá" dependen del prompt y de la confirmación del usuario. Ver *Endurecimiento recomendado*.
- Los agentes **no** hacen `commit` ni `push`: el versionado es decisión del usuario.

## Validación

```bash
# Verifica que los archivos de agente parseen correctamente
claude plugin validate .claude/agents

# Muestra claves/reglas rechazadas y estado de la configuración
claude doctor
```

## Filosofía

> Evidencia antes que afirmaciones · Validación antes que delegación · Autorización antes que ejecución · Estado conocido antes que reintento · Simplicidad antes que abstracción · Una decisión humana explícita antes de una acción irreversible.

## Licencia

[MIT](LICENSE) © 2026 Luis Matias Kühn
