# ADR-001: Arquitectura de Meta-Framework en Cinco Niveles

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Nova Platform es un meta-framework Java que necesita una arquitectura modular que permita reutilizar codigo entre Spring Boot, Quarkus y Micronaut sin acoplamiento al framework. Se requiere una separacion clara de responsabilidades donde las librerias de logica pura no dependan de ningun framework especifico, mientras que las capas superiores se encarguen de la integracion, coordinacion de versiones y tooling de desarrollo.

## Decision
Adoptar una jerarquia de 5 niveles donde cada nivel depende solo de niveles inferiores. Nivel 1 no puede importar de Nivel 2+.

- **Nivel 1 — Librerias puras:** `mask-utils`, `date-utils`, `mapper-utils`, `api-standard`, `observability-utils`. Zero dependencias de framework (`org.springframework.*`, `io.quarkus.*`, `io.micronaut.*`). Solo Java puro.
- **Nivel 2 — Starters/Conectores:** `observability-starter`, `commons-starter` (contiene `mask-starter` + `api-standard-starter`). Conectan las librerias puras con un framework especifico via auto-configuracion.
- **Nivel 3 — Meta-Starter:** `nova-spring-boot-starter`. Agrega todos los starters de Nivel 2 en una sola dependencia transitiva.
- **Nivel 4 — BOM + Parent:** `nova-bom` (centraliza versiones), `nova-spring-boot-parent` (configuracion compartida de build). El BOM es el coordinador de versiones de todo el stack.
- **Nivel 5 — Build Tooling:** `nova-spring-boot-gradle-plugin` (convention plugin), `nova-spring-boot-archetype` (scaffolding). Herramientas de desarrollo, no APIs.

## Consecuencias
### Positivas
- Reutilizacion multi-framework: las librerias de Nivel 1 funcionan con cualquier framework sin modificacion.
- Testing independiente por nivel: cada capa se puede probar de forma aislada.
- Versionado granular: cada modulo evoluciona a su propio ritmo sin forzar releases en cadena.

### Negativas
- Mas repos que gestionar (15 Java), lo que incrementa la complejidad operativa.
- Coordinacion de versiones via BOM requiere disciplina y procesos claros de release.

## Referencias
- `docs/java/01-conceptos-tecnicos-meta-frameworks.md` Seccion 2
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.3
