# ADR-008: GitHub Packages como Registry Principal

## Estado
Aceptada (implementada)
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Se evaluaron 4 opciones de registry para publicar artefactos Java:

- **GitHub Packages** — integrado con GitHub, autenticacion nativa.
- **Maven Central** — registry publico universal, requiere firma GPG y registro en Sonatype.
- **Sonatype OSSRH** — staging previo a Maven Central, proceso complejo.
- **Nexus/Artifactory self-hosted** — control total, pero requiere infraestructura propia.

## Decision
**GitHub Packages** (`maven.pkg.github.com`) como registry principal para MVP y desarrollo.

- Ya implementado en 8 reusable workflows existentes.
- Autenticacion nativa con `GITHUB_TOKEN` (sin secretos adicionales).
- URL: `https://maven.pkg.github.com/ahincho/<repo-name>`

### Visibilidad del paquete

Parametrizable por repo via `vars.NOVA_PACKAGE_VISIBILITY`:

| Variable | Default | Valores |
|---|---|---|
| `NOVA_PACKAGE_VISIBILITY` | `public` | `public` \| `private` |

Restriccion: repo publico = paquete publico (no se puede override).

### Decisiones asociadas

- Repos personales de `ahincho`: publicos.
- Repos de empresa: publicos (por ahora).
- Enterprise/pago futuro: podria ser privado.

## Consecuencias
### Positivas
- Zero config adicional para publicar desde GitHub Actions.
- Buen rendimiento para descarga y publicacion.
- Inmutabilidad de versiones publicadas (no se puede sobrescribir un artefacto).

### Negativas
- Consumidores externos necesitan configurar el registry de GitHub Packages en su build (no es automatico como Maven Central).

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 3.1
- `docs/java/06-semantic-versioning-en-java.md` Seccion 3.1.1
