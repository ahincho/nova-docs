# ADR-010: GitHub Actions Cache para Build

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Necesitamos reducir tiempos de CI de minutos a segundos. Gradle ofrece 3 niveles de cache. Se evaluaron backends remotos para Build Cache:

- **Develocity (Gradle Enterprise)** — solución oficial, pero de costo elevado.
- **S3/GCS/Azure Blob** — requiere cuenta cloud adicional.
- **Nexus/Artifactory** — complejidad operativa innecesaria.
- **GitHub Actions Cache** — integrado, sin costo adicional.

## Decision
Usar **GitHub Actions Cache exclusivamente** (via `gradle/actions/setup-gradle@v4`) como backend de Build Cache remoto. Tres capas de cache:

| Capa | Que cachea | Config |
|---|---|---|
| Dependencies cache | JARs externos | `setup-java` con `cache: 'gradle'` (ya implementado) |
| Local Build Cache | Outputs de tasks | `org.gradle.caching=true` en `gradle.properties` |
| Remote Build Cache | Outputs compartidos entre runners | `gradle/actions/setup-gradle@v4` |
| Configuration Cache | Scripts evaluados | `org.gradle.configuration-cache=true` |

### Politica de lectura/escritura

| Evento | Lee cache | Escribe cache |
|---|---|---|
| `push` a `main` | Si | Si (popula) |
| `pull_request` | Si | No (read-only) |
| `workflow_dispatch` | Si | Configurable |

### Alternativas rechazadas

- **Develocity:** costo elevado ($$$), overkill para open source.
- **S3/GCS/Azure:** requiere cuenta cloud adicional, inconsistente con stack GitHub-only.
- **Nexus/Artifactory:** complejidad operativa innecesaria.
- **Self-hosted:** innecesario con GitHub Actions gratis.

### Config recomendada

En `gradle.properties`:

```properties
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.parallel=true
org.gradle.workers.max=4
```

### Migracion futura

Si el proyecto crece a escala empresarial, se puede migrar a Develocity sin cambiar el codigo de los proyectos (solo la action de CI).

## Consecuencias
### Positivas
- Zero costo operativo, incluido en GitHub Actions.
- Consistencia con stack GitHub-only (sin servicios externos).
- Builds subsequentes mucho mas rapidos tras el primer cache hit.

### Negativas
- Primer build del dia es lento (cache miss).
- Limite de 10 GB de cache por repo.
- No permite compartir cache cross-organization.

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.1
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.2
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.3
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.4
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.5
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.9.6
