# ADR-005: Multi-Repo con BOM Coordinador

## Estado
Aceptada
**Scope:** `java` (Java stack)

## Fecha
2026-07-08

## Contexto
Con 15 repos Java independientes, se evaluo consolidar en un mono-repo o mantener multi-repo. Cada modulo tiene ciclos de release diferentes. Se necesitaba una estrategia que permitiera autonomia por modulo sin perder la coherencia de versiones del stack completo. 19 repos totales: 15 Java + 4 NestJS (NestJS fuera de alcance de este roadmap).

## Decision
Mantener **multi-repo**. Cada repo se versiona independientemente con semver. `nova-bom` es el coordinador que centraliza las versiones de todas las librerias y starters via `<dependencyManagement>`.

Estrategia de versionado por nivel:

| Nivel | Versionado |
|---|---|
| 1 (libs) | Independiente, semver estricto por repo |
| 2 (starters) | Coordinado con BOM |
| 3 (meta-starter) | Coordinado con BOM |
| 4 (BOM) | `0.x.0`, coordina todo el stack |
| 5 (plugin/archetype) | Independiente |

`release-please` soporta multi-repo nativo. Cada repo tiene su `.release-please-config.json`. El workflow reusable se centraliza en `nova-devops`.

## Consecuencias
### Positivas
- Autonomia por modulo: cada equipo puede hacer releases sin bloquear a otros.
- Releases independientes con PRs mas pequenos y focalizados.
- Paralelismo de equipos: multiples repos pueden evolucionar en paralelo.

### Negativas
- Coordinacion de versiones requiere el BOM como fuente de verdad centralizada.
- Mas repos que gestionar (19 totales), aumentando la carga operativa.
- Mitigado con workflows reusables centralizados en `nova-devops`.

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 10.5
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.3
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.6
