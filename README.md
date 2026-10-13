# nestjs-clean-architecture-skill
Reusable Claude Code skill for NestJS projects using Clean Architecture, interactors, contracts, drivers and dependency injection.

## Uso

Copiá la carpeta `.claude/skills/nestjs-clean-architecture/` completa a la misma ruta dentro de tu proyecto NestJS. En Claude Code, invocá:

```text
/nestjs-clean-architecture
```

Por ejemplo:

```text
/nestjs-clean-architecture Revisá la arquitectura de este proyecto y señalá violaciones de responsabilidades sin modificar archivos.
```

También podés pedir que implemente una funcionalidad, cree un módulo o refactorice un flujo siguiendo esta metodología.

## Contenido

- [SKILL.md](.claude/skills/nestjs-clean-architecture/SKILL.md): procedimiento operativo y reglas DO/DON'T.
- [architecture.md](.claude/skills/nestjs-clean-architecture/architecture.md): estructura, dependencias, composición e incorporación de tecnologías.
- [naming-conventions.md](.claude/skills/nestjs-clean-architecture/naming-conventions.md): nombres e imports.
- [responsibilities.md](.claude/skills/nestjs-clean-architecture/responsibilities.md): ubicación de responsabilidades y límites entre capas.
- [examples.md](.claude/skills/nestjs-clean-architecture/examples.md): ejemplos reales autocontenidos.
- [checklist.md](.claude/skills/nestjs-clean-architecture/checklist.md): validación de implementaciones y revisiones.
- [inconsistencies.md](.claude/skills/nestjs-clean-architecture/inconsistencies.md): excepciones del modelo que no deben copiarse como reglas.

## Alcance

Las instrucciones están en español y son independientes del dominio. No requieren acceso al repositorio modelo ni a sus servicios. Los ejemplos conservan nombres originales únicamente como referencia; las reglas operativas se adaptan al proyecto destino.

La Skill distingue patrones observados, generalizaciones e inconsistencias. No exige instalar todas las tecnologías mencionadas ni reemplazar la arquitectura de un proyecto completo para realizar un cambio localizado.
