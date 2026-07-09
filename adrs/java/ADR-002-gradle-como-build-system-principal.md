# ADR-002: Gradle como Build System Principal

## Estado
Aceptada
**Scope:** `java` (Java stack)

## Fecha
2026-07-08

## Contexto
Habia una mezcla de Maven y Gradle sin logica clara. 7 repos usaban Gradle, 6 Maven, 2 sin build. Esta inconsistencia generaba duplicacion en workflows de CI/CD, dificultaba la estandarizacion de plugins y configuraciones, y aumentaba la carga cognitiva al mantener dos ecosistemas de build en paralelo. Se necesitaba estandarizacion para simplificar CI/CD y reducir duplicacion en workflows.

## Decision
Adoptar **Gradle 9.x** como build system principal para TODOS los repos excepto 3 que permanecen en Maven por estandar de la industria:

- `nova-bom` — BOM (packaging `pom`, estandar Maven)
- `nova-java-spring-boot-parent` — Parent POM (concepto inherente de Maven)
- `nova-java-spring-boot-archetype` — Maven Archetype (sin equivalente Gradle)

3 repos fueron migrados de Maven a Gradle:

- `mask-utils` (libreria pura, dificultad baja)
- `observability-utils` (libreria pura, dificultad trivial)
- `starter` (meta-starter, dificultad media)

**Wrapper version:** Gradle 9.2.0

## Consecuencias
### Positivas
- Un solo conjunto de workflows de CI para la mayoria de repos (12 de 15).
- Plugins Gradle modernos: build cache, configuration cache, dependency locking.
- DSL Kotlin type-safe para configuracion de builds.

### Negativas
- BOM/Parent/Archetype requieren workflows Maven separados (3 de 15).
- Curva de aprendizaje para contribuidores familiarizados solo con Maven.

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Contexto (linea 5)
- `docs/java/03-siguientes-pasos-y-mejoras.md` Seccion 2.1
