# Architecture Decision Records (ADRs) — Nova Platform

Registro de decisiones arquitectonicas y tecnicas del meta-framework **Nova**.

## Estructura

```
docs/adrs/
  shared/      Decisiones cross-stack (Java + NestJS)
  java/        Decisiones especificas del stack Java
  nest/        Decisiones especificas del stack NestJS
```

## Convenciones

- **Aceptada:** Decision tomada y en vigor.
- **Aceptada (con concern):** Vigente pero con un punto debil documentado.
- **Propuesta:** Documentada, pendiente de ejecucion o aprobacion.
- **Pendiente:** Placeholder, no desarrollada todavia.
- **Deprecada:** Ya no aplica.
- **Reemplazada por ADR-XXX:** Sustituida por otra decision.

## ADRs Compartidos (`shared/`)

Decisiones aplicables a Java y NestJS.

| # | ADR | Estado | Tema |
|---|---|---|---|
| 001 | [Arquitectura del Meta-Framework en 5 Niveles](shared/ADR-001-arquitectura-meta-framework-cinco-niveles.md) | Aceptada | Arquitectura |
| 004 | [Namespace `pe.edu.nova`](shared/ADR-004-namespace-pe-edu-nova.md) | Aceptada | Estructura |
| 006 | [Conventional Commits y Semantic Versioning](shared/ADR-006-conventional-commits-y-semantic-versioning.md) | Aceptada | Versioning |
| 007 | [release-please para Automatizacion de Releases](shared/ADR-007-release-please-para-automatizacion.md) | Aceptada | Versioning |
| 008 | [GitHub Packages como Registry Principal](shared/ADR-008-github-packages-como-registry-principal.md) | Aceptada | Publishing |
| 009 | [Estrategia Multi-Registry](shared/ADR-009-estrategia-multi-registry.md) | Aceptada | Publishing |
| 010 | [GitHub Actions Cache para Build Performance](shared/ADR-010-github-actions-cache-para-build.md) | Aceptada | Performance |
| 011 | [Composite Actions y Reusable Workflows](shared/ADR-011-composite-actions-y-reusable-workflows.md) | Aceptada | CI/CD |
| 012 | [Estandares de Calidad y Testing](shared/ADR-012-estandares-de-calidad-testing.md) | Aceptada | Calidad |
| 014 | [Observabilidad: Four Golden Signals](shared/ADR-014-observabilidad-four-golden-signals.md) | Aceptada | Observabilidad |

## ADRs Java (`java/`)

Decisiones especificas del stack Java (Spring Boot, Quarkus, Micronaut).

| # | ADR | Estado | Tema |
|---|---|---|---|
| 002 | [Gradle 9.x como Build System Principal](java/ADR-002-gradle-como-build-system-principal.md) | Aceptada | Build System |
| 003 | [Java 25 como Version Objetivo](java/ADR-003-java-25-como-version-objetivo.md) | Aceptada* | Build System |
| 005 | [Multi-Repo con BOM Coordinador](java/ADR-005-multi-repo-con-bom-coordinador.md) | Aceptada | Estructura |
| 013 | [Firma GPG Preparada pero Diferida](java/ADR-013-firma-gpg-preparada-diferida.md) | Propuesta | Seguridad |
| 015 | [Librerias Puras sin Dependencias de Framework](java/ADR-015-librerias-puras-sin-dependencias-framework.md) | Aceptada | Arquitectura |

*ADR-003 tiene un concern abierto sobre soportar Java 21 LTS como minimo.

## ADRs NestJS (`nest/`)

Decisiones especificas del stack NestJS. Stack fuera de alcance actual (Sprint 0+ solo Java).

| # | ADR | Estado | Tema |
|---|---|---|---|
| 016 | [Node.js 22 LTS como Version Objetivo](nest/ADR-016-node-version-objetivo.md) | Pendiente | Build System |
| 017 | [pnpm como Package Manager](nest/ADR-017-pnpm-package-manager.md) | Pendiente | Build System |
| 018 | [NestJS 10.x como Framework Backend](nest/ADR-018-nestjs-framework.md) | Pendiente | Framework |
| 019 | [TypeScript 5.x en Modo Estricto](nest/ADR-019-typescript-estricto.md) | Pendiente | Lenguaje |
| 020 | [ORM para Persistencia](nest/ADR-020-orm-persistencia.md) | Pendiente | Persistencia |
| 021 | [Jest como Framework de Testing](nest/ADR-021-jest-testing.md) | Pendiente | Testing |
| 022 | [ESLint + Prettier + Husky](nest/ADR-022-eslint-prettier-husky.md) | Pendiente | Calidad |
| 023 | [Swagger/OpenAPI](nest/ADR-023-swagger-openapi.md) | Pendiente | Documentacion |

## Formato

Cada ADR sigue el template:

```
# ADR-NNN: Titulo

## Estado
## Fecha
## Contexto
## Decision
## Consecuencias (Positivas / Negativas)
## Referencias
```
