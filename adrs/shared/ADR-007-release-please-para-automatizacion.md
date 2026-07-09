# ADR-007: Release Please para Automatizacion de Releases

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Se evaluaron 3 estrategias para release automation en Java:

- **Estrategia A:** `net.nemerosa.versioning` solo — no decide major/minor/patch automaticamente.
- **Estrategia B:** `semantic-release` orquestando Gradle — mezcla Node.js runtime con Java builds.
- **Estrategia C:** `release-please` de Google — declarativo, no mezcla stacks, multi-repo friendly.

## Decision
Adoptar **release-please** (Estrategia C) combinado con `net.nemerosa.versioning` para derivacion de version en build-time.

### Flujo

1. Developer hace commits con Conventional Commits.
2. `release-please` detecta commits en `main`, abre un **PR de release** con bump + changelog.
3. Reviewer aprueba el PR.
4. Al merge, `release-please` crea tag `vX.Y.Z` + GitHub Release con notas auto-generadas.
5. Workflow de publish se dispara automaticamente.

### Config por repo

```json
// .release-please-config.json
{
  "packages": {
    ".": {
      "release-type": "java",
      "package-name": "nova-java-spring-boot-mask-utils",
      "bump-minor-pre-major": true
    }
  }
}
```

### Workflow centralizado en `nova-devops`

```yaml
# reusable-release-please.yml
uses: googleapis/release-please-action@v4
```

### Plugin de versioning en Gradle

`net.nemerosa.versioning` 4.0.1 agregado a los 10 repos Gradle con config:

```kotlin
versioning {
    releaseMode = "snapshot"
    displayMode = "snapshot"
    dirty = { it }
    releaseBuild = false
}
```

## Consecuencias
### Positivas
- No mezcla stacks (Node.js no invade el runtime de Java).
- Revision humana obligatoria antes de publicar (critico por inmutabilidad de Maven Central).
- Multi-repo nativo, cada repo tiene su propia config.

### Negativas
- Requiere aprobar PR manualmente (no es full-automatic como semantic-release).
- Un paso extra en el flujo de desarrollo.

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 4.3
- `docs/java/06-semantic-versioning-en-java.md` Seccion 6
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.1
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.6
