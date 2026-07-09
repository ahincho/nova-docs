# ADR-011: Composite Actions y Reusable Workflows

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
`nova-devops` tiene 8 reusable workflows. Se necesita reducir duplicacion entre los 15+ repos y encapsular pasos comunes (setup Java, import GPG, gather version info). GitHub Actions ofrece dos mecanismos: reusable workflows (jobs completos) y composite actions (steps reutilizables).

## Decision
Implementar **ambos mecanismos** con regla de uso clara:

| Mecanismo | Cuando usar | Ejemplo |
|---|---|---|
| **Reusable workflow** | Pipeline completo (build + test + publish) | `reusable-build-gradle.yml` |
| **Composite action** | Setup o transformacion reutilizable | `nova-setup-java` |

**7 composite actions** en `nova-devops/.github/actions/`:

Sprint 1 (NOVA-SEMVER-08) — 3 de setup:
1. `nova-setup-java` — Setup Java 25 + Gradle/Maven cache
2. `nova-setup-node` — Setup Node.js 20 + npm + commitlint
3. `nova-setup-gpg` — Import GPG key desde inputs (skip si no hay secrets)

Sprint 5 (NOVA-SEMVER-26) — 4 restantes:
4. `nova-gather-facts` — Recolectar version, branch, commit SHA
5. `nova-publish-aggregator` — Switch interno segun `inputs.registry`
6. `nova-configure-gradle-cache` — Habilitar GitHub Actions Build Cache
7. `nova-validate-build` — Verificar pre-requisitos (Java version, no secrets commiteados)

**14 reusable workflows** propuestos (8 existentes + 6 nuevos):
- P0: `reusable-commitlint.yml`, `reusable-release-please.yml`, `reusable-publish-maven-central.yml`, `reusable-publish-multi-registry.yml`, `reusable-publish-gradle-maven-central.yml`
- P1: `reusable-build-matrix.yml`, `reusable-owasp-check.yml`, `reusable-changelog.yml`, `reusable-publish-*-nexus.yml`
- P2: `reusable-native-image-gradle.yml`, `reusable-sbom.yml`, `reusable-coverage-badge.yml`

**Versionado de actions:** `@v1` en produccion, `@v1.2.3` para maxima estabilidad, `@main` solo en desarrollo.

**Nota tecnica importante:** Las composite actions NO tienen acceso al contexto `secrets.*` de GitHub Actions. Los secrets se pasan como `inputs` desde el workflow que invoca la action.

## Consecuencias
### Positivas
- Elimina duplicacion entre 15 repos
- Consistencia en setup de entornos (Java, Node, GPG)
- Cambio centralizado: una actualizacion en `nova-devops` se propaga a todos los repos consumidores
- Separacion clara de responsabilidades entre workflows (orquestacion) y actions (pasos atomicos)

### Negativas
- Dependencia en `nova-devops` como repo central; si se rompe, afecta a todos los repos
- Versionado de actions requiere disciplina (tags semanticos, no romper contratos)
- Debugging mas complejo al tener logica distribuida entre actions y workflows

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 5.3, 5.4, 5.5, 5.6
