# atl-SymAssist

Repositorio de memoria y configuración del agente ATL para el proyecto **SymAssist**.

## Estructura

```
.atl/
├── memory/          # Snapshots por ticket/sesión de trabajo
├── skills/          # Skills del agente reutilizables
├── AGENTS.md        # Instrucciones del agente
├── skill-registry.md
└── pull_request_template.md
```

## Convención de commits

```
feat: TICKET-123 descripción corta
fix:  TICKET-456 corrección de X
docs: actualización de AGENTS.md
snapshot: estado previo a refactor de Y
```

## Rollback

Para volver a un estado anterior:

```bash
# Ver historial
git log --oneline

# Rollback de un archivo específico
git checkout <commit-hash> -- memory/archivo.md

# Rollback completo (crea rama desde punto anterior)
git checkout -b rollback/<ticket> <commit-hash>
```
