# Directorio `.agent/skills/_shared`

> Esta es una convención personalizada para compartir convenciones y reglas entre agentes en proyectos específicos.

---

## ¿Qué es gentle-ai?

Gentle-ai es un **configurador de ecosistema** que potencia agentes de coding (Claude Code, Cursor, OpenCode, Windsurf, etc.) añadiendo:

| Componente | Descripción |
|------------|-------------|
| **Engram** | Memoria persistente entre sesiones |
| **SDD** | Spec-Driven Development workflow (9 fases) |
| **Skills** | Biblioteca de skills curados para coding |
| **Context7** | Servidor MCP para documentación live de frameworks |
| **Persona** | Modos de comportamiento: Gentleman, neutral, o custom |
| **GGA** | Gentleman Guardian Angel — switcher de providers de AI |

---

## Estructura de Skills por Agente

Cada agente tiene su propia estructura de directorios para skills:

| Agente | Path de Skills |
|--------|---------------|
| Claude Code | `~/.claude/` |
| OpenCode | `~/.config/opencode/skills/` |
| Cursor | `~/.cursor/skills/` |
| Windsurf | `~/.codeium/windsurf/skills/` |
| Codex | `~/.codex/skills/` |

> **Nota**: Los skills de SDD vienen incluidos en el binario de gentle-ai. Los skills de coding específicos (React, Angular, .NET, etc.) están en el repositorio [Gentleman-Programming/Gentleman-Skills](https://github.com/Gentleman-Programming/Gentleman-Skills).

---

## Propósito del Directorio `_shared`

El directorio `.agent/skills/_shared` es una **convención local** (no estándar de gentle-ai) para compartir archivos entre el agente principal y sub-agentes en un proyecto específico.

### ¿Para qué sirve?

- Archivos de convenciones de código (naming, style guides)
- Reglas de arquitectura específicas del proyecto
- Documentos de contexto global
- Prompts reutilizables

### Diferencia con el ecosistema Gentleman

| Aspecto | `.agent/skills/_shared` (local) | Gentle-ai (ecosistema) |
|---------|----------------------------------|------------------------|
| **Alcance** | Proyecto específico | Global al agente |
| **Skills estándar** | No incluidos | SDD skills + Gentleman-Skills |
| **Propósito** | Convenciones custom | Workflow completo + memoria |

---

## Skills SDD Incluidos

Gentle-ai instala automáticamente estos skills de SDD:

### SDD (Spec-Driven Development)

| Skill | Descripción |
|-------|-------------|
| `sdd-init` | Bootstrap SDD en un proyecto |
| `sdd-explore` | Investigar codebase antes de un cambio |
| `sdd-propose` | Crear proposal con intent, scope, approach |
| `sdd-spec` | Escribir especificaciones con escenarios |
| `sdd-design` | Diseño técnico con decisiones de arquitectura |
| `sdd-tasks` | Descomponer en tareas de implementación |
| `sdd-apply` | Implementar siguiendo specs y diseño |
| `sdd-verify` | Validar que la implementación coincide con specs |
| `sdd-archive` | Sync specs y archivar cambio |
| `judgment-day` | Review paralelo con dos judges independientes |

### Foundation

| Skill | Descripción |
|-------|-------------|
| `go-testing` | Patrones de testing en Go |
| `skill-creator` | Crear nuevos skills para agentes |
| `branch-pr` | Workflow de PR con conventional commits |
| `issue-creation` | Workflow de issues con templates |

---

## Skills en Uso (Local)

Esta es mi colección de skills de convenciones y reglas locales. A medida que las vaya creando o modificando las iré agregando:

### .NET

| Skill | Descripción |
|-------|-------------|
| [csharp-coding-conventions.md 1.0.0 — 2026-03-24](./DotNet/.agent/skills/_shared/csharp-coding-conventions.md) | Convenciones de código para C# |
| [dotnet-architecture-rules.md 1.0.0 — 2026-03-29](./DotNet/.agent/skills/_shared/dotnet-architecture-rules.md) | Reglas de arquitectura para .NET |

### Agregar Nuevos Lenguajes

Para agregar skills de otro lenguaje:

```
.
└── .agent/
    └── skills/
        └── _shared/
            ├── python/
            │   └── python-conventions.md
            ├── javascript/
            │   └── js-style-guide.md
            └── rust/
                └── rust-coding-standards.md
```

---

## Recursos Oficiales

- **Repo principal**: [Gentleman-Programming/gentle-ai](https://github.com/Gentleman-Programming/gentle-ai)
- **Gentleman-Skills**: [Gentleman-Programming/Gentleman-Skills](https://github.com/Gentleman-Programming/Gentleman-Skills)
- **Documentación**: [docs/](https://github.com/Gentleman-Programming/gentle-ai/tree/main/docs)

---

## Notas

- El directorio `.agent/skills/_shared` es **personalizado** — no es creado por gentle-ai por defecto
- Los directorios estándar de gentle-ai son `.engram/` (memoria) y la carpeta de skills del agente configurado
- Para proyectos multi-lenguaje, se recomienda mantener una estructura paralela bajo `_shared/`
