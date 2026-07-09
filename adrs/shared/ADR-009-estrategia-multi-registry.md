# ADR-009: Estrategia Multi-Registry

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
El proyecto eventualmente necesitara publicar a Maven Central para consumidores publicos. Algunos modulos podrian necesitar Nexus on-premise. Se necesita una estrategia gradual que permita escalar sin romper lo existente.

## Decision
Adoptar **multi-registry gradual** en 3 etapas:

| Etapa | Registry | Razon |
|---|---|---|
| **MVP / desarrollo** | GitHub Packages | Ya implementado, sin friccion |
| **Beta** | GitHub Packages + Nexus on-premise (opcional) | Cache interno si se necesita |
| **Produccion** | Maven Central (via Sonatype OSSRH) | Discovery universal, sin auth para consumidores |

### Implementacion tecnica

En `build.gradle.kts`:

```kotlin
publishing {
    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/ahincho/...")
        }
        // maven { name = "SonatypeOSS" ... }  // futuro
    }
}
```

Workflows parametrizados: input `registry: github-packages | maven-central | nexus-custom`.

### Bloqueantes para Maven Central

- Firma GPG obligatoria (NOVA-SEMVER-29, backlog).
- Namespace `pe.edu.nova` no registrado en Sonatype.
- Metadata completa en POM (licencia, developers, SCM).

## Consecuencias
### Positivas
- Migracion progresiva sin romper nada existente.
- Workflows preparados para multiples destinos desde el inicio.

### Negativas
- Complejidad incremental en workflows al agregar registries.
- Maven Central requiere GPG + Sonatype account (proceso burocrático).

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 3.5
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.8
- `docs/java/06-semantic-versioning-en-java.md` Seccion 10.3
